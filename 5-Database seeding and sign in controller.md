#Trello clone with Phoenix and React (第五章节)

这篇文章属于基于Phoenix Framework 和React的Trello系列    

> 1. [介绍和架构选择](1-Intro and selected stack.md) <br/>
> 2. [Phoenix Framework 项目设置](2-Phoenix Framework project setup.md)  <br/>
> 3. [User模型和JWT权限设置](3-The User model and JWT auth.md) <br/>
> 4. [前端使用React 和 Redux实现用户注册](4-Front-end for sign up with React and Redux.md) <br/>
> 5. [数据库初始化和用户登录控制器](5-Database seeding and sign in controller.md)<br/>
> 6. [基于React和Redux的前端验证](6-Front-end authentication with React and Redux.md) <br/>
> 7. [sockets和channels 配置](7-Setting up sockets and channels.md)<br/>
> 8. [展示和创建新Board](8-Listing and creating new boards.md)<br/>
> 9. 即将推出 <br/>


#用户登录

在前两章，我们已经完成了用户注册和创建新用户。在这一章，我们将看到如何数据库seed。同时，我们将添加一些必须的功能，让用户也可以使用email和密码登录。最后，我们创建一个机制，可以从用户认证token中获得用户数据。

##数据库seed

如果你已经拥有 **Rails**经验，将会发现在 **Phoenix**中的数据库seed和Rails十分相似。要做到这一点：我们仅需要`seedx.exs`文件：

```elixir
# priv/repo/seeds.exs

alias PhoenixTrello.{Repo, User}

[
  %{
    first_name: "John",
    last_name: "Doe",
    email: "john@phoenix-trello.com",
    password: "12345678"
  },
]
|> Enum.map(&User.changeset(%User{}, &1))
|> Enum.each(&Repo.insert!(&1))

```


在这个文件中，我们需要向数据库插入应用程序需要的初始化数据。如果想添加其他用户，只需要在列表中添加即可，然后运行下面命令：

```
$ mix run priv/repo/seeds.exs
```

##用户登录控制器

在创建控制器之前，我们需要修改`router.ex`文件：

```elixir
# web/router.ex

defmodule PhoenixTrello.Router do
  use PhoenixTrello.Web, :router

  #...

  pipeline :api do
    # ...

    plug Guardian.Plug.VerifyHeader
    plug Guardian.Plug.LoadResource
  end

  scope "/api", PhoenixTrello do
    pipe_through :api

    scope "/v1" do
      # ...

      post "/sessions", SessionController, :create
      delete "/sessions", SessionController, :delete

      # ...
    end
  end

  #...
end
```


第一个改变是，在`:api` pipeline中添加了连个新的[plug](http://www.phoenixframework.org/docs/understanding-plug)：
*   **VerifyHeader**: 这个plug是用于在`Authorization` header中查找token。
*   **LoadResource**: 如果token存在，则可以用过`Guardian.Plug.current_resource(conn)`让当前资源可用。

同时，我们需要在`/api/v1` scope中添加两个路由，用于创建和销毁用户的会话。这两个路由都位于`SessionController`中，以下是`create`动作：

```elixir
# web/controllers/api/v1/session_controller.ex

defmodule PhoenixTrello.SessionController do
  use PhoenixTrello.Web, :controller

  plug :scrub_params, "session" when action in [:create]

  def create(conn, %{"session" => session_params}) do
    case PhoenixTrello.Session.authenticate(session_params) do
      {:ok, user} ->
        {:ok, jwt, _full_claims} = user |> Guardian.encode_and_sign(:token)

        conn
        |> put_status(:created)
        |> render("show.json", jwt: jwt, user: user)

      :error ->
        conn
        |> put_status(:unprocessable_entity)
        |> render("error.json")
    end
  end

  # ...
end
```

我们将使用`PhoenixTrello.Session`helper模块完成用户认证，并附带接收一些参数。如果都是`:ok`，用户将编码和登录。这些带来`jwt`token，并返回 **JSON**格式用户数据。在这以前，让我们看看`Session helper`模块：

```elixir
# web/helpers/session.ex

defmodule PhoenixTrello.Session do
  alias PhoenixTrello.{Repo, User}

  def authenticate(%{"email" => email, "password" => password}) do
    user = Repo.get_by(User, email: String.downcase(email))

    case check_password(user, password) do
      true -> {:ok, user}
      _ -> :error
    end
  end

  defp check_password(user, password) do
    case user do
      nil -> false
      _ -> Comeonin.Bcrypt.checkpw(password, user.encrypted_password)
    end
  end
end
```


这个模块试图通过email找到用户，并检查给定的密码与用户的加密是否匹配。如果用户存在并且密码正确将返回`{:ok,user}`[元组](http://elixir-lang.org/getting-started/basic-types.html#tuples)。相反，用户不存在或者密码不匹配，将返回`:error`原子。

回到`SessionController`，当验证用户后返回`:error`原子，控制器将呈现`error.json`模板。最后，我们为了两种结果创建了`SessionView`模块：

```elixir
# web/views/session_view.ex

defmodule PhoenixTrello.SessionView do
  use PhoenixTrello.Web, :view

  def render("show.json", %{jwt: jwt, user: user}) do
    %{
      jwt: jwt,
      user: user
    }
  end

  def render("error.json", _) do
    %{error: "Invalid email or password"}
  end
end
```


##已经注册过的用户

之所以还返回用户的 **JSON**表示在登录该应用程序，我们可能需要多种用途，如在程序顶部显示用户名。我们到目前为止已经完成了。但是，如果用户在根路径视图刷新一次浏览器呢？简单的，我们应用状态是通过 **Redux**管理的，在刷新浏览器时会被重置，我们不会有可用的信息，以及可能会造成不必要的错误。我们不想这样，为了解决这个，我们创建了一个新的控制器，负责返回验证数据。

让我们在`router.ex`文件中添加路由：

```elixir
# web/router.ex

defmodule PhoenixTrello.Router do
  use PhoenixTrello.Web, :router

  #...

  scope "/api", PhoenixTrello do
    pipe_through :api

    scope "/v1" do
      # ...

      get "/current_user", CurrentUserController, :show

      # ...
    end
  end

  #...
end
```

同时我们需要`CurrentUserController`：

```elixir
# web/controllers/api/v1/current_user_controller.ex

defmodule PhoenixTrello.CurrentUserController do
  use PhoenixTrello.Web, :controller

  plug Guardian.Plug.EnsureAuthenticated, handler: PhoenixTrello.SessionController

  def show(conn, _) do
    case decode_and_verify_token(conn) do
      { :ok, _claims } ->
        user = Guardian.Plug.current_resource(conn)

        conn
        |> put_status(:ok)
        |> render("show.json", user: user)

      { :error, _reason } ->
        conn
        |> put_status(:not_found)
        |> render(PhoenixTrello.SessionView, "error.json", error: "Not Found")
    end
  end

  defp decode_and_verify_token(conn)  do
    conn
    |> Guardian.Plug.current_token
    |> Guardian.decode_and_verify
  end
end
```


`Guardian.Plug.EnsureAuthenticated`检查是否是先前验证的token，如果不是，则通过`SessionController`的`:unauthenticated`功能处理这个请求。这种方式是为了保护私有控制器， 因此我们需要一些路由只允许验证过的用户访问，我们仅需要添加这个plug到它们的控制器。从请求检索该令牌之后，解码和校验，然后给用户呈现`current_resource`。否则，将呈现出错误。

最后我们在`SessionController`中添加`unauthenticated`处理：

```elixir
# web/controllers/api/v1/session_controller.ex

defmodule PhoenixTrello.SessionController do
  use PhoenixTrello.Web, :controller

  # ...

  def unauthenticated(conn, _params) do
    conn
    |> put_status(:forbidden)
    |> render(PhoenixTrello.SessionView, "forbidden.json", error: "Not Authenticated")
  end
end
```

它将返回`403`禁止状态代码，是个简单的 **JSON**格式错误字符串。这样我们就完成了用户登录和验证相关的后端工作。下一章，我们将转移到前端，以及如何连接到`UserSocket`，这个是所有实时部分的核心。同样，别忘记查看运行演示和下载最终的源代码：

[演示](https://phoenix-trello.herokuapp.com/)        [源代码](https://github.com/bigardone/phoenix-trello)

快乐编程吧！
