#Trello clone with Phoenix and React (第八章节)

这篇文章属于基于Phoenix Framework 和React的Trello系列    

#Listing and creating boards

Now that we have covered the important aspects of user registration and authentication management as well as connecting to the socket and joining channels, we are ready to move on to the next level and let the user list and create his own boards.

##The Board migration

First we need to create the migration and model. To do that, just run:

```
$ mix phoenix.gen.model Board boards board_id:references:board name:string
```

This will generate our new migration file which will look something similar to this:

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

The new table called boards will have, apart from its id and timestamps fields, a name field and a foreign key to the users table. Note that we are relying on the database to delete the boards belonging to a user if the user is deleted. It also adds an index to the user_id to speed up things, and a null constraints to the name.

Having finished modifying the migration file, we need to run it:

```
$ mix ecto.migrate
```

#The Board model

Let's take a look at the Board model:

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

For now there's nothing important to mention yet, but we need to update the User model to add its related owned boards:

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

Why owned_boards? To differentiate between the boards created by the user and the ones he’s been added by other users, but let’s don’t worry about this right now, we will dive into it more deeply later on.

##The BoardController

So to create new boards we are going to need to update the routes file to add the necessary entry to handle the requests:

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

We've added the boards resource with only the index and create actions so the BoardController will handle this requests:

```
$ mix phoenix.routes
board_path  GET     /api/v1/boards    PhoenixTrello.BoardController :index
board_path  POST    /api/v1/boards    PhoenixTrello.BoardController :create
```
Let's create the new controller:

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

Note that we are adding the EnsureAuthenticated plug from Guardian so only authenticated connections are permitted in this controller. In the index action we get the current user from the connection and retrieve his owned board from the database so we can render them using the BoardView. Almost the same happens in the create action, we build a owned_board changeset from the user and insert it into the database, rendering the board as response if everything goes as expected.

Let's create the BoardView:

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

##The React view component

Now that the back-end is ready for handling listing boards requests and also their creation, we are going to focus on the front-end. After the user signs in the first thing we want to show him is the list of his boards and the form for creating a new one, so let's create the HomeIndexView:

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
Many things are going on here so let's check them one by one:

* First of all we have to keep in mind that this component is connected to the store and will receive its props from the resulting changes by the boards reducer that we'll create in short.
* When it mounts it will change the document's title to Boards and will dispatch and action creator to fetch the boards on the back-end.
* For now it will just render the owned_boards array in the store and also the BoardForm component.
* Before rendering this two, it will first check if the fetching prop is set to true. If so, it will mean that boards are still being fetched so it will render a spinner. Otherwise it will render the list of boards and the button for adding a new board.
* When clicking the add new board button it will dispatch a new action creator for hiding the button and showing the form.

Now let's add the BoardForm component:

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
This is a very simple component. It renders the form and when submitted it dispatches an action creator to create the new board with the supplied name. The PageClick component is an external component I found which detects page clicks outside the wrapper element. In our case we will use it to hide the form and show the Add new board... button again.

##The action creators

So we basically need three action creators:

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

Enough work for today! In the next post we will build the view to show a board and we will also add the functionality for adding new members to it, broadcasting the board to the related users so it appears in their invited boards list that we will also have to add. Meanwhile, don't forget to check out the live demo and final source code:

Live demo Source code
Happy coding!