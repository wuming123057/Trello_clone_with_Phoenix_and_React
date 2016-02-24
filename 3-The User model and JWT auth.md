#Trello clone with Phoenix and React (第三章节)

这篇文章属于基于Phoenix Framework 和React的Trello系列    

#用户注册

现在，项目已经完成了[基本项目设置](https://blog.diacode.com/trello-clone-with-phoenix-and-react-pt-2)，我们准备创建`User`数据库迁移和`User`模型。在这一章，我们将看到整个过程以及来宾如何创建一个用户账号。    

##User数据库迁移与User 模型

**Phoenix**使用[Ecto](https://github.com/elixir-lang/ecto)装与数据库相关的任何操作。如果使用过 **Rails**，**Ecto** 有些类似于 **ActiveRecord**，**ActiveRecord**把相似的功能分隔为不同的模块。

在这之前，我们需要创建运行的数据库：

`$ mix ecto.create`

现在，让我们创建新的 **Ecto** 迁移和模型。模型生成命令后跟模块的名字作为参数，模块名称的复数（小写）就是表名称，表的字段需要使用`name:type`语法，让我们运行它：

`$ mix phoenix.gen.model User users first_name:string last_name:string email:string encrypted_password:string`

如果我们查看我们刚刚创建迁移文件，会发现它与 **Rails** 的迁移文件十分类似：

```elixir
# priv/repo/migrations/20151224075404_create_user.exs

defmodule PhoenixTrello.Repo.Migrations.CreateUser do
  use Ecto.Migration

  def change do
    create table(:users) do
      add :first_name, :string, null: false
      add :last_name, :string, null: false
      add :email, :string, null: false
      add :encrypted_password, :string, null: false
      #add :crypted_password, :string, null: false  原文是crypted_password，作者后面添加了user_password_fix
      timestamps
    end

    create unique_index(:users, [:email])
  end
end
```

字段已经添加了 `null` 限制和`email`索引唯一。因为，我喜欢数据库负责数据完整性，而不是依赖于应用开发者。我猜这是个人喜好的事情。

现在迁移文件已经准备好了，让我们运行它创建 `users`表：

`$ mix ecto.migrate`

现在是我们仔细看看 **User** 模型的时候了：

```elixir
# web/models/user.ex

defmodule PhoenixTrello.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :first_name, :string
    field :last_name, :string
    field :email, :string
    field :encrypted_password, :string

    timestamps
  end

  @required_fields ~w(first_name last_name email)
  @optional_fields ~w(encrypted_password)

  def changeset(model, params \\ :empty) do
    model
    |> cast(params, @required_fields, @optional_fields)
  end
end
```

这里可以找到两个主要不同的部分：

*  **schema**部分所有元数据对应于表字段
* **变更** 功能：可以定义所有的验证和转变，这些都可以应用于之前在程序中需要使用的数据。


##变更验证和转换

当用户注册的时候，我们希望添加一些验证过程，因为之前我们已经在表字段上添加了null限制和email唯一约束。我们必须在 `User`模型中反应这些，便于处理非法数据引起的运行错误。同时我们想加密 `encrypted_password`字段，即使我们将使用普通字符串来指定用户的密码，它也将以安全的方式被插入。

首先更新User模型和添加一些验证：    

```elixir
# web/models/user.ex

defmodule PhoenixTrello.User do
  # ...

  schema "users" do
    # ...
    field :password, :string, virtual: true
    # ...
  end

  @required_fields ~w(first_name last_name email password)
  @optional_fields ~w(encrypted_password)

  def changeset(model, params \\ :empty) do
    model
    |> cast(params, @required_fields, @optional_fields)
    |> validate_format(:email, ~r/@/)
    |> validate_length(:password, min: 5)
    |> validate_confirmation(:password, message: "Password does not match")
    |> unique_constraint(:email, message: "Email already taken")
  end
end
```

基本上我们修改了如下内容：

* 添加了虚拟的`password`字段，虚拟字段不会存在到数据库中，却可以和其他字段一样使用。在这里，我们主要用户注册表单。
* 添加`pasword`字段请求。
* 添加`email`格式检测验证。
* 添加检测密码最小长度为5个字符验证，以及密码是否输入同一个值。
* 添加`email`唯一检测验证。

经过这些修改，我们完成了验证。但是在保存数据前，我们需要填充encrypted_password字段。为了完成这，我们需要使用[comeonin](https://github.com/elixircnx/comeonin) 密码加密库，在`mix.exs`文件中添加`comeonin`依赖，并作为应用程序：

```elixir
# mix.exs

defmodule PhoenixTrello.Mixfile do
  use Mix.Project
  # ...

  def application do
    [mod: {PhoenixTrello, []},
     applications: [
       # ...
       :comeonin
       ]
     ]
  end

  #...

  defp deps do
    [
      # ...
      {:comeonin, "~> 2.0"},
      # ...
    ]
  end
end
```

别忘记运行如下命令：

`$ mix deps.get`

现在 **comeonin**已经安装完毕，让我们回到`User`模型，在变更管道中添加一个生成`encrypted_password`字段步骤：

```elixir
# web/models/user.ex

defmodule PhoenixTrello.User do
  # ...

  def changeset(model, params \\ :empty) do
    model
    # ... other validations and contraints
    |> generate_encrypted_password
  end

  defp generate_encrypted_password(current_changeset) do
    case current_changeset do
      %Ecto.Changeset{valid?: true, changes: %{password: password}} ->
        put_change(current_changeset, :encrypted_password, Comeonin.Bcrypt.hashpwsalt(password))
      _ ->
        current_changeset
    end
  end
end
```

在新的函数中，我们首先监测变革是否合法和密码是否改变。如果合法，我们使用 **comeonin**加密密码并放入`encrypted_password`字段中，否则范围变更。

##路由


现在`User`模型已经准备好了，让我们通过修改`router.ex`继续完成注册过程，在这个文件中创建`:api` 管道和我们的第一个路由：

```elixir
# web/router.ex

defmodule PhoenixTrello.Router do
  use PhoenixTrello.Web, :router

  #...

  pipeline :api do
    plug :accepts, ["json"]
  end

  scope "/api", PhoenixTrello do
    pipe_through :api

    scope "/v1" do
      post "/registrations", RegistrationController, :create
    end
  end

  #...
end
```

任何通过`/api/v1/registrations`的`POST`请求，都会在`RegistrationController`中的`create`动作中处理（接受 **JSON**...相当于解释：）

##控制器

在完成控制器之前，让我们想想我们都需要什么。新用户将访问注册页面，填写表格并确认。如果控制器接收到的数据是合法的，我们将向数据库中插入一条新`User`数据，在系统中登记，并返回[jwt](https://en.wikipedia.org/wiki/JSON_Web_Token)验证标记，在前端登陆过程是以 **json**形式返回。这个标记不仅在每次用户验证时需要，而且当用户访问程序的私有页面时需要。

为了处理这个验证和 **jwt**生成器，我们将使用[Guardian](https://github.com/ueberauth/guardian)库，这个库非常好用。仅需要在`mix.exs`文件中添加：

```elixir
# mix.exs

defmodule PhoenixTrello.Mixfile do
  use Mix.Project

  #...

  defp deps do
    [
      # ...
      {:guardian, "~> 0.9.0"},
      # ...
    ]
  end
end
```

在运行`mix deps.get`之后，我们需要配置`config.exs`文件：

```elixir
# config/confg.exs

#...

config :guardian, Guardian,
  issuer: "PhoenixTrello",
  ttl: { 3, :days },
  verify_issuer: true,
  secret_key: <your guardian secret key>,
  serializer: PhoenixTrello.GuardianSerializer
```

同时，我们需要创建`GuardianSerializer`，便于告诉 **Guardian**如何编码和解码用户进出令牌:

```elixir
# lib/phoenix_trello/guardian_serializer.ex

defmodule PhoenixTrello.GuardianSerializer do
  @behaviour Guardian.Serializer

  alias PhoenixTrello.{Repo, User}

  def for_token(user = %User{}), do: { :ok, "User:#{user.id}" }
  def for_token(_), do: { :error, "Unknown resource type" }

  def from_token("User:" <> id), do: { :ok, Repo.get(User, String.to_integer(id)) }
  def from_token(_), do: { :error, "Unknown resource type" }
end
```

现在一切就绪，让我们完成`RegistrationController`:

```elixir
# web/controllers/api/v1/registration_controller.ex

defmodule PhoenixTrello.RegistrationController  do
  use PhoenixTrello.Web, :controller

  alias PhoenixTrello.{Repo, User}

  plug :scrub_params, "user" when action in [:create]

  def create(conn, %{"user" => user_params}) do
    changeset = User.changeset(%User{}, user_params)

    case Repo.insert(changeset) do
      {:ok, user} ->
        {:ok, jwt, _full_claims} = Guardian.encode_and_sign(user, :token)

        conn
        |> put_status(:created)
        |> render(PhoenixTrello.SessionView, "show.json", jwt: jwt, user: user)

      {:error, changeset} ->
        conn
        |> put_status(:unprocessable_entity)
        |> render(PhoenixTrello.RegistrationView, "error.json", changeset: changeset)
    end
  end
end
```

多亏于 **Elixir**[模式匹配](http://elixir-lang.org/getting-started/pattern-matching.html)，在`create`动作中获取`"user"`里面的参数。通过这些参数，我们将创建新的`User`变更并插入到数据库。如果一切顺利，我们将使用 **Guardian**的`encode_and_sign`功能来索取新用户的`jwt` token，并返回用户`json`数据。另外，如果变更非法，将返回错误`json`数据，我们可以在注册表单中看到这些。

##JSON 序列化

**Phoenix**使用[Poison](https://github.com/devinus/poison)作为默认 **JSON**库。因为 **Phoenix**依赖中已经包含了，我们不需要添加。我们仅需要更新`User`模型的时候指定那些字段需要序列化：

```elixir
# web/models/user.ex

defmodule PhoenixTrello.User do
  use PhoenixTrello.Web, :model
  # ...

   @derive {Poison.Encoder, only: [:id, :first_name, :last_name, :email]}

   # ...
 end
```

从现在起，当我们呈现一个用户，或者用户的列表，控制器动作或通道将做出相应，它将只返回那些指定的字段。非常容易!

后端已经为注册新用户准备好了，在下一章节我们将转移到前端并使用 **React**和 **Redux**这些有趣的东西完成注册过程。同样，别忘记查看运行演示和下载最终的源代码：

[演示](https://phoenix-trello.herokuapp.com/)        [源代码](https://github.com/bigardone/phoenix-trello)

快乐编程吧！