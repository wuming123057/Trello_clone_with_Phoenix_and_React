#Trello clone with Phoenix and React (第八章节)

这篇文章属于基于Phoenix Framework 和React的Trello系列    

#卡片展示和创建

现在我们已经完成用户注册和权限管，同时连接到Socket，并加入到channels。我们将进入下一阶段：让用户展示和创建自己的卡片。

##卡片迁移

首先，我们需要创建迁移和模型，如下所示：

```
$ mix phoenix.gen.model Board boards board_id:references:board name:string
```

将会创建新的类似如下所示的迁移文件：

```elixir
# priv/repo/migrations/20151224093233_create_board.exs

defmodule PhoenixTrello.Repo.Migrations.CreateBoard do
  use Ecto.Migration

  def change do
    create table(:boards) do
      add :name, :string, null: false
      add :user_id, references(:users, on_delete: :delete_all), null: false

      timestamps
    end

    create index(:boards, [:user_id])
  end
end
```

新建数据表表名是boards，包含id和timestamps字段，以及name字段，name字段是外键，关联于users表。注意如果用户被删除了，数据库将删除这个用户所有的卡片。同时为这个表添加索引，便于查询，和name添加空约束。

完成这些修改后，运行它：

```
$ mix ecto.migrate
```

#Board模型

让我们看看Board模型：

```elixir
# web/models/board.ex

defmodule PhoenixTrello.Board do
  use PhoenixTrello.Web, :model

  alias __MODULE__

  @derive {Poison.Encoder, only: [:id, :name, :user]}

  schema "boards" do
    field :name, :string
    belongs_to :user, User

    timestamps
  end

  @required_fields ~w(name user_id)
  @optional_fields ~w()

  @doc """
  Creates a changeset based on the `model` and `params`.

  If no params are provided, an invalid changeset is returned
  with no validation performed.
  """
  def changeset(model, params \\ :empty) do
    model
    |> cast(params, @required_fields, @optional_fields))
  end
end
```

到现在还没有重要的需要提醒的事情，我们仅需要更新User模型，并关联到owned_boards：

```elixir
# web/models/user.ex

defmodule PhoenixTrello.User do
  use PhoenixTrello.Web, :model

  # ...

  schema "users" do
    # ...

    has_many :owned_boards, PhoenixTrello.Board

    # ...
  end

  # ...
end
```

为什么是owned_boards？用户创建的卡片和用户添加其他人的卡片是有区别的，现在不用考虑这些，以后我们会更深入地讨论。

##BoardController

为了实现新的boards，我们将更新路由文件，并添加必要的入口，便于处理各种请求。

```elixir
# web/router.ex

defmodule PhoenixTrello.Router do
  use PhoenixTrello.Web, :router

  # ...

  scope "/api", PhoenixTrello do
    # ...

    scope "/v1" do
      # ...

      resources "boards", BoardController, only: [:index, :create]
    end
  end

  # ...
end
```

在添加boards资源时，仅创建了index和create动作，BoardController将会处理这些请求：

```
$ mix phoenix.routes
board_path  GET     /api/v1/boards    PhoenixTrello.BoardController :index
board_path  POST    /api/v1/boards    PhoenixTrello.BoardController :create
```
让我们创建新的控制器：

```elixir
# web/controllers/board_controller.ex

defmodule PhoenixTrello.BoardController do
  use PhoenixTrello.Web, :controller

  plug Guardian.Plug.EnsureAuthenticated, handler: PhoenixTrello.SessionController

  alias PhoenixTrello.{Repo, Board}

  def index(conn, _params) do
    current_user = Guardian.Plug.current_resource(conn)

    owned_boards = current_user
      |> assoc(:owned_boards)
      |> Board.preload_all
      |> Repo.all

    render(conn, "index.json", owned_boards: owned_boards)
  end

  def create(conn, %{"board" => board_params}) do
    current_user = Guardian.Plug.current_resource(conn)

    changeset = current_user
      |> build_assoc(:owned_boards)
      |> Board.changeset(board_params)

    case Repo.insert(changeset) do
      {:ok, board} ->
        conn
        |> put_status(:created)
        |> render("show.json", board: board )
      {:error, changeset} ->
        conn
        |> put_status(:unprocessable_entity)
        |> render("error.json", changeset: changeset)
    end
  end
end

```

注意：我们从Guardian添加了 EnsureAuthenticated 插件，因此在这个控制器中只允许有授权的连接。在index动作中，我们从连接中获取当前用户，并从数据库检索相关卡片信息，便于在BoardView展示。几乎同时，在create动作中，我们从user建立了owned_board变更，并插入到数据库，如果一切顺利就渲染卡片。

让我们创建 BoardView：

```elixir
# web/views/board_view.ex

defmodule PhoenixTrello.BoardView do
  use PhoenixTrello.Web, :view

  def render("index.json", %{owned_boards: owned_boards}) do
    %{owned_boards: owned_boards}
  end

  def render("show.json", %{board: board}) do
    board
  end

  def render("error.json", %{changeset: changeset}) do
    errors = Enum.map(changeset.errors, fn {field, detail} ->
      %{} |> Map.put(field, detail)
    end)

    %{
      errors: errors
    }
  end
end
```

##React 视图组件

服务器后端已经准备好处理卡片展示和创建请求，现在让我们着重于前端。在用户登陆后，首先将展示用户的卡片，同时可以点击窗口创建新的卡片，让我们来创建HomeIndexView：

```javascript
// web/static/js/views/home/index.js

import React                from 'react';
import { connect }          from 'react-redux';
import classnames           from 'classnames';

import { setDocumentTitle } from '../../utils';
import Actions              from '../../actions/boards';
import BoardCard            from '../../components/boards/card';
import BoardForm            from '../../components/boards/form';

class HomeIndexView extends React.Component {
  componentDidMount() {
    setDocumentTitle('Boards');

    const { dispatch } = this.props;
    dispatch(Actions.fetchBoards());
  }

  _renderOwnedBoards() {
    const { fetching } = this.props;

    let content = false;

    const iconClasses = classnames({
      fa: true,
      'fa-user': !fetching,
      'fa-spinner': fetching,
      'fa-spin':    fetching,
    });

    if (!fetching) {
      content = (
        <div className="boards-wrapper">
          {::this._renderBoards(this.props.ownedBoards)}
          {::this._renderAddNewBoard()}
        </div>
      );
    }

    return (
      <section>
        <header className="view-header">
          <h3><i className={iconClasses} /> My boards</h3>
        </header>
        {content}
      </section>
    );
  }

  _renderBoards(boards) {
    return boards.map((board) => {
      return <BoardCard
                key={board.id}
                dispatch={this.props.dispatch}
                {...board} />;
    });
  }

  _renderAddNewBoard() {
    let { showForm, dispatch, formErrors } = this.props;

    if (!showForm) return this._renderAddButton();

    return (
      <BoardForm
        dispatch={dispatch}
        errors={formErrors}
        onCancelClick={::this._handleCancelClick}/>
    );
  }

  _renderAddButton() {
    return (
      <div className="board add-new" onClick={::this._handleAddNewClick}>
        <div className="inner">
          <a id="add_new_board">Add new board...</a>
        </div>
      </div>
    );
  }

  _handleAddNewClick() {
    let { dispatch } = this.props;

    dispatch(Actions.showForm(true));
  }

  _handleCancelClick() {
    this.props.dispatch(Actions.showForm(false));
  }

  render() {
    return (
      <div className="view-container boards index">
        {::this._renderOwnedBoards()}
      </div>
    );
  }
}

const mapStateToProps = (state) => (
  state.boards
);

export default connect(mapStateToProps)(HomeIndexView);
```
这里内容很多，让我们一个一个的看：

* 首先我们必须牢记：组件需要连接到sotre，并且属性改变来自于我们创建的卡片reducer。
* When it mounts it will change the document's title to Boards and will dispatch and action creator to fetch the boards on the back-end.
* For now it will just render the owned_boards array in the store and also the BoardForm component.
* Before rendering this two, it will first check if the fetching prop is set to true. If so, it will mean that boards are still being fetched so it will render a spinner. Otherwise it will render the list of boards and the button for adding a new board.
* When clicking the add new board button it will dispatch a new action creator for hiding the button and showing the form.

现在让我们来添加BoardForm 组件：

```javascript
// web/static/js/components/boards/form.js

import React, { PropTypes } from 'react';
import PageClick            from 'react-page-click';
import Actions              from '../../actions/boards';
import {renderErrorsFor}    from '../../utils';

export default class BoardForm extends React.Component {
  componentDidMount() {
    this.refs.name.focus();
  }

  _handleSubmit(e) {
    e.preventDefault();

    const { dispatch } = this.props;
    const { name } = this.refs;

    const data = {
      name: name.value,
    };

    dispatch(Actions.create(data));
  }

  _handleCancelClick(e) {
    e.preventDefault();

    this.props.onCancelClick();
  }

  render() {
    const { errors } = this.props;

    return (
      <PageClick onClick={::this._handleCancelClick}>
        <div className="board form">
          <div className="inner">
            <h4>New board</h4>
            <form id="new_board_form" onSubmit={::this._handleSubmit}>
              <input ref="name" id="board_name" type="text" placeholder="Board name" required="true"/>
              {renderErrorsFor(errors, 'name')}
              <button type="submit">Create board</button> or <a href="#" onClick={::this._handleCancelClick}>cancel</a>
            </form>
          </div>
        </div>
      </PageClick>
    );
  }
}
```
这是一个很简单的组件。用于渲染表格This is a very simple component. It renders the form and when submitted it dispatches an action creator to create the new board with the supplied name. The PageClick component is an external component I found which detects page clicks outside the wrapper element. In our case we will use it to hide the form and show the Add new board... button again.

##action creators

我们最少需要 3个 action creators：

```javascript
// web/static/js/actions/boards.js

import Constants              from '../constants';
import { routeActions }       from 'react-router-redux';
import { httpGet, httpPost }  from '../utils';
import CurrentBoardActions    from './current_board';

const Actions = {
  fetchBoards: () => {
    return dispatch => {
      dispatch({ type: Constants.BOARDS_FETCHING });

      httpGet('/api/v1/boards')
      .then((data) => {
        dispatch({
          type: Constants.BOARDS_RECEIVED,
          ownedBoards: data.owned_boards
        });
      });
    };
  },

  showForm: (show) => {
    return dispatch => {
      dispatch({
        type: Constants.BOARDS_SHOW_FORM,
        show: show,
      });
    };
  },

  create: (data) => {
    return dispatch => {
      httpPost('/api/v1/boards', { board: data })
      .then((data) => {
        dispatch({
          type: Constants.BOARDS_NEW_BOARD_CREATED,
          board: data,
        });

        dispatch(routeActions.push(`/boards/${data.id}`));
      })
      .catch((error) => {
        error.response.json()
        .then((json) => {
          dispatch({
            type: Constants.BOARDS_CREATE_ERROR,
            errors: json.errors,
          });
        });
      });
    };
  },
};

export default Actions;
```
* fetchBoards: it will first dispatch the BOARDS_FETCHING action type so we can render the spinner previously mentioned. I will also launch the http request to the back-end to retrieve the boards owned by the user which will be handled by the BoardController:index action. When the response is back, it will dispatch the boards to the store.
* showForm: this one is pretty simple and it will just dispatch the BOARDS_SHOW_FORM action to set whether we want to show the form or not.
* create: it will send a POST request to create the new board. If the response is successful then it will dispatch the BOARDS_NEW_BOARD_CREATED action with the created board, so its added to the boards in the store and it will navigate to the show board route. In case there is any error it will dispatch the BOARDS_CREATE_ERROR.

##The reducer

The last piece of the puzzle would be the reducer which is very simple:

```javascript
// web/static/js/reducers/boards.js

import Constants from '../constants';

const initialState = {
  ownedBoards: [],
  showForm: false,
  formErrors: null,
  fetching: true,
};

export default function reducer(state = initialState, action = {}) {
  switch (action.type) {
    case Constants.BOARDS_FETCHING:
      return { ...state, fetching: true };

    case Constants.BOARDS_RECEIVED:
      return { ...state, ownedBoards: action.ownedBoards, fetching: false };

    case Constants.BOARDS_SHOW_FORM:
      return { ...state, showForm: action.show };

    case Constants.BOARDS_CREATE_ERROR:
      return { ...state, formErrors: action.errors };

    case Constants.BOARDS_NEW_BOARD_CREATED:
      const { ownedBoards } = state;

      return { ...state, ownedBoards: [action.board].concat(ownedBoards) };

    default:
      return state;
  }
}
```
Note how we set the fetching attribute to false once we load the boards and how we concat the new board created to the existing ones.

Enough work for today! In the next post we will build the view to show a board and we will also add the functionality for adding new members to it, broadcasting the board to the related users so it appears in their invited boards list that we will also have to add.
同样，别忘记查看运行演示和下载最终的源代码：

[演示](https://phoenix-trello.herokuapp.com/)        [源代码](https://github.com/bigardone/phoenix-trello)