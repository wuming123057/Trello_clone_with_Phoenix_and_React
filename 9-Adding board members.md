#Trello clone with Phoenix and React (第九章节)

这篇文章属于基于Phoenix Framework 和React的Trello系列    

#Adding board members

On the last part we created the boards table, the Board model and we also generated the controller which will be in charge of listing and creating new boards for the authenticated users. We also coded the front-end so the boards and the creation form could be displayed. Recalling where we left it, after receiving the successful response from the controller while creating a new board, we wanted to redirect the user to its view so he could see all the details and add more existing users as members. Let's do this!

##The React view component

Before continuing let's take a look at the React routes:

```javascript
// web/static/js/routes/index.js

import { IndexRoute, Route }        from 'react-router';
import React                        from 'react';
import MainLayout                   from '../layouts/main';
import AuthenticatedContainer       from '../containers/authenticated';;
import BoardsShowView               from '../views/boards/show';
// ...

export default (
  <Route component={MainLayout}>
    ...

    <Route path="/" component={AuthenticatedContainer}>
      <IndexRoute component={HomeIndexView} />

      ...

      <Route path="/boards/:id" component={BoardsShowView}/>
    </Route>
  </Route>
);
```

The /boards/:id route is going to be handled by the BoardsShowView component that we need to create:

```javascript
// web/static/js/views/boards/show.js

import React, {PropTypes}   from 'react';
import { connect }          from 'react-redux';

import Actions              from '../../actions/current_board';
import Constants            from '../../constants';
import { setDocumentTitle } from '../../utils';
import BoardMembers           from '../../components/boards/members';


class BoardsShowView extends React.Component {
  componentDidMount() {
    const { socket } = this.props;

    if (!socket) {
      return false;
    }

    this.props.dispatch(Actions.connectToChannel(socket, this.props.params.id));
  }

  componentWillUnmount() {
    this.props.dispatch(Actions.leaveChannel(this.props.currentBoard.channel));
  }

  _renderMembers() {
    const { connectedUsers, showUsersForm, channel, error } = this.props.currentBoard;
    const { dispatch } = this.props;
    const members = this.props.currentBoard.members;
    const currentUserIsOwner = this.props.currentBoard.user.id === this.props.currentUser.id;

    return (
      <BoardMembers
        dispatch={dispatch}
        channel={channel}
        currentUserIsOwner={currentUserIsOwner}
        members={members}
        connectedUsers={connectedUsers}
        error={error}
        show={showUsersForm} />
    );
  }


  render() {
    const { fetching, name } = this.props.currentBoard;

    if (fetching) return (
      <div className="view-container boards show">
        <i className="fa fa-spinner fa-spin"/>
      </div>
    );

    return (
      <div className="view-container boards show">
        <header className="view-header">
          <h3>{name}</h3>
          {::this._renderMembers()}
        </header>
        <div className="canvas-wrapper">
          <div className="canvas">
            <div className="lists-wrapper">
              {::this._renderAddNewList()}
            </div>
          </div>
        </div>
      </div>
    );
  }
}

const mapStateToProps = (state) => ({
  currentBoard: state.currentBoard,
  socket: state.session.socket,
  currentUser: state.session.currentUser,
});

export default connect(mapStateToProps)(BoardsShowView);
```

When it mounts it will connect to the board's channel using the user socket we already created on part 7. When rendering it will first check if the fetching attribute is set to true, if so it will render a spinner while the board's data is still being fetched. As we can see it takes its props from the currentBoard element in the state which is created by the following reducer.

##The reducer and actions creator

As a starting point of the current board state we will only need to store the board data, the channel and the fetching flag:

```javascript
// web/static/js/reducers/current_board.js

import Constants  from '../constants';

const initialState = {
  channel: null,
  fetching: true,
};

export default function reducer(state = initialState, action = {}) {
  switch (action.type) {
    case Constants.CURRENT_BOARD_FETHING:
      return { ...state, fetching: true };

    case Constants.BOARDS_SET_CURRENT_BOARD:
      return { ...state, fetching: false, ...action.board };

    case Constants.CURRENT_BOARD_CONNECTED_TO_CHANNEL:
      return { ...state, channel: action.channel };

    default:
      return state;
  }
}
```

Let's take a look to the current_board actions creator to check how do we connect to the channel and dispatch all the necessary data:

```javascript
// web/static/js/actions/current_board.js

import Constants  from '../constants';

const Actions = {
  connectToChannel: (socket, boardId) => {
    return dispatch => {
      const channel = socket.channel(`boards:${boardId}`);

      dispatch({ type: Constants.CURRENT_BOARD_FETHING });

      channel.join().receive('ok', (response) => {
        dispatch({
          type: Constants.BOARDS_SET_CURRENT_BOARD,
          board: response.board,
        });

        dispatch({
          type: Constants.CURRENT_BOARD_CONNECTED_TO_CHANNEL,
          channel: channel,
        });
      });
    };
  },

  // ...
};

export default Actions;
```

Just as with the UserChannel, we use the socket to create a new channel identified as boards:${boardId} and we join it, receiving as response the JSON representation of the board, which will be dispatched to the store along with the BOARDS_SET_CURRENT_BOARD action. From now on it will be connected to the channel receiving any change done to the board by any member, refreshing automatically those updates in the screen thanks to React and Redux. But first we need to create the BoardChannel.

##The BoardChannel

Although almost all of the remaining functionality is going to be placed in this module, we are now going to just create a very simple version of it:

```javascript
# web/channels/board_channel.ex

defmodule PhoenixTrello.BoardChannel do
  use PhoenixTrello.Web, :channel
  alias PhoenixTrello.Board

  def join("boards:" <> board_id, _params, socket) do
    board = get_current_board(socket, board_id)

    {:ok, %{board: board}, assign(socket, :board, board)}
  end

  defp get_current_board(socket, board_id) do
    socket.assigns.current_user
    |> assoc(:boards)
    |> Repo.get(board_id)
  end
end
```
The join method gets the current board from the assigned user in the socket, returns it and assigns it to the socket so its available for future messages.

![](/images/part9/board_1.jpg)   

##Board members

Once the board is displayed to the user, the following step is to allow him to add other existing users as members so they can work together on it. To associate boards with other users we have to create a new table to store this relation. Let's jump to the console and run:
```
$ mix phoenix.gen.model UserBoard user_boards user_id:references:users board_id:references:boards
```
We need to update a bit the resulting migration file:

```elixir
# priv/repo/migrations/20151230081546_create_user_board.exs

defmodule PhoenixTrello.Repo.Migrations.CreateUserBoard do
  use Ecto.Migration

  def change do
    create table(:user_boards) do
      add :user_id, references(:users, on_delete: :delete_all), null: false
      add :board_id, references(:boards, on_delete: :delete_all), null: false

      timestamps
    end

    create index(:user_boards, [:user_id])
    create index(:user_boards, [:board_id])
    create unique_index(:user_boards, [:user_id, :board_id])
  end
end
```

Apart from the null constraints, we are going to add a unique index for the user_id and the board_id so a User can't be added twice to the same Board. After running the necessary mix ecto.migrate lets head to the UserBoard model:

```elixir
# web/models/user_board.ex

defmodule PhoenixTrello.UserBoard do
  use PhoenixTrello.Web, :model

  alias PhoenixTrello.{User, Board}

  schema "user_boards" do
    belongs_to :user, User
    belongs_to :board, Board

    timestamps
  end

  @required_fields ~w(user_id board_id)
  @optional_fields ~w()

  def changeset(model, params \\ :empty) do
    model
    |> cast(params, @required_fields, @optional_fields)
    |> unique_constraint(:user_id, name: :user_boards_user_id_board_id_index)
  end
end
```

Nothing unusual about it, but we also need to add this new relationships to the User model:

```elixir
# web/models/user.ex

defmodule PhoenixTrello.User do
  use PhoenixTrello.Web, :model
  # ...

  schema "users" do
    # ...

    has_many :user_boards, UserBoard
    has_many :boards, through: [:user_boards, :board]

    # ...
  end

  # ...
end
```

We have two more relationships, but the one that matters the most to us is the :boards one, which we are going to use for security checks. Let's also add the collection to the Board model:

```elixir
# web/models/board.ex

defmodule PhoenixTrello.Board do
  # ...

  schema "boards" do
    # ...

    has_many :user_boards, UserBoard
    has_many :members, through: [:user_boards, :user]

    timestamps
  end
end
```

By doing these changes now we can differentiate between boards created by a user and boards where the user has been invited to. This is very important because when a user is in the board's view we only want to show the members form if he is the original creator. We also want to automatically add the creator as a member so he gets listed by default, therefore we have to make a small change in the BoardController:

```elixir
# web/controllers/api/v1/board_controller.ex

defmodule PhoenixTrello.BoardController do
  use PhoenixTrello.Web, :controller
  #...

  def create(conn, %{"board" => board_params}) do
    current_user = Guardian.Plug.current_resource(conn)

    changeset = current_user
      |> build_assoc(:owned_boards)
      |> Board.changeset(board_params)

    if changeset.valid? do
      board = Repo.insert!(changeset)

      board
      |> build_assoc(:user_boards)
      |> UserBoard.changeset(%{user_id: current_user.id})
      |> Repo.insert!

      conn
      |> put_status(:created)
      |> render("show.json", board: board )
    else
      conn
      |> put_status(:unprocessable_entity)
      |> render("error.json", changeset: changeset)
    end
  end
end
```

Note how we build the new UserBoard association and insert it after previously checking if the board is valid.

##The board members component

This component will display all the board's members avatars and the form to add new ones:

![](/images/part9/board_3.jpg)   

As you can see, thanks to the previous change in the BoardController, the owner will be displayed as the only member for now. Let's see how this component will look like:

```javascipt
// web/static/js/components/boards/members.js

import React, {PropTypes}       from 'react';
import ReactGravatar            from 'react-gravatar';
import classnames               from 'classnames';
import PageClick                from 'react-page-click';
import Actions                  from '../../actions/current_board';

export default class BoardMembers extends React.Component {
  _renderUsers() {
    return this.props.members.map((member) => {
      const index = this.props.connectedUsers.findIndex((cu) => {
        return cu === member.id;
      });

      const classes = classnames({ connected: index != -1 });

      return (
        <li className={classes} key={member.id}>
          <ReactGravatar className="react-gravatar" email={member.email} https/>
        </li>
      );
    });
  }

  _renderAddNewUser() {
    if (!this.props.currentUserIsOwner) return false;

    return (
      <li>
        <a onClick={::this._handleAddNewClick} className="add-new" href="#"><i className="fa fa-plus"/></a>
        {::this._renderForm()}
      </li>
    );
  }

  _renderForm() {
    if (!this.props.show) return false;

    return (
      <PageClick onClick={::this._handleCancelClick}>
        <ul className="drop-down active">
          <li>
            <form onSubmit={::this._handleSubmit}>
              <h4>Add new members</h4>
              {::this._renderError()}
              <input ref="email" type="email" required={true} placeholder="Member email"/>
              <button type="submit">Add member</button> or <a onClick={::this._handleCancelClick} href="#">cancel</a>
            </form>
          </li>
        </ul>
      </PageClick>
    );
  }

  _renderError() {
    const { error } = this.props;

    if (!error) return false;

    return (
      <div className="error">
        {error}
      </div>
    );
  }

  _handleAddNewClick(e) {
    e.preventDefault();

    this.props.dispatch(Actions.showMembersForm(true));
  }

  _handleCancelClick(e) {
    e.preventDefault();

    this.props.dispatch(Actions.showMembersForm(false));
  }

  _handleSubmit(e) {
    e.preventDefault();

    const { email } = this.refs;
    const { dispatch, channel } = this.props;

    dispatch(Actions.addNewMember(channel, email.value));
  }

  render() {
    return (
      <ul className="board-users">
        {::this._renderUsers()}
        {::this._renderAddNewUser()}
      </ul>
    );
  }
}
```

Basically it will loop through its members prop displaying their avatars. It will also display the add new button if the current user happens to be the owner of the board. When clicking this button the form will be shown, prompting the user to enter a member email and calling the addNewMember action creator when the form is submitted.

##The addNewMember action creator

From now on, instead of using controllers to create and retrieve the necessary data for our React front-end we will move this responsibility into the BoardChannel so any change can be broadcasted to every joined user. Having this in mind let's add the necessary action creators:

```javascript
// web/static/js/actions/current_board.js

import Constants  from '../constants';

const Actions = {
  // ...

  showMembersForm: (show) => {
    return dispatch => {
      dispatch({
        type: Constants.CURRENT_BOARD_SHOW_MEMBERS_FORM,
        show: show,
      });
    };
  },

  addNewMember: (channel, email) => {
    return dispatch => {
      channel.push('members:add', { email: email })
      .receive('error', (data) => {
        dispatch({
          type: Constants.CURRENT_BOARD_ADD_MEMBER_ERROR,
          error: data.error,
        });
      });
    };
  },

  // ...

}

export default Actions;
```
The showMembersForm will make the form show or hide, easy as pie. The tricky part comes when we want to add the new member with the email provided by the user. Instead of making the typical http request we've been doing so far, we push the message members:add to the channel with the email as parameter. If we receiver an error we will dispatch it so it's displayed in the screen. Why aren't we handling the case for a success response? Because we are going to take a different approach, broadcasting the result to all the connected members.

##The BoardChannel

Having this said let's add the underlying message handler to the BoardChannel

``elixir
# web/channels/board_channel.ex

defmodule PhoenixTrello.BoardChannel do
  # ...

  def handle_in("members:add", %{"email" => email}, socket) do
    try do
      board = socket.assigns.board
      user = User
        |> Repo.get_by(email: email)

      changeset = user
      |> build_assoc(:user_boards)
      |> UserBoard.changeset(%{board_id: board.id})

      case Repo.insert(changeset) do
        {:ok, _board_user} ->
          broadcast! socket, "member:added", %{user: user}

          PhoenixTrello.Endpoint.broadcast_from! self(), "users:#{user.id}", "boards:add", %{board: board}

          {:noreply, socket}
        {:error, _changeset} ->
          {:reply, {:error, %{error: "Error adding new member"}}, socket}
      end
    catch
      _, _-> {:reply, {:error, %{error: "User does not exist"}}, socket}
    end
  end

  # ...
end
```

Phoenix channels handle incoming messages using the handle_in function and Elixir's powerful pattern matching to handle incoming messages. In our case the message name will be members:add, and it will be also be expecting an email parameter which will be matched to the corresponding variable. It will get the assigned board in the channel, find the user by his email and create a new UserBoard with both of them. If everything goes fine it will broadcast the message member:added to all the available connections passing the added user. Now let's take a closer look to this:

PhoenixTrello.Endpoint.broadcast_from! self(), "users:#{user.id}", "boards:add", %{board: board}
By doing this, it will be broadcasting the message boards:add along with the board to the UserChannel of the added member so the board suddenly appears in his invited boards list. This means we can broadcast any message to any channel from anywhere, which is awesome and brings a new bunch of possibilities and fun.

To handle the member:added message in the front-end we have to add a new handler to the channel where it will dispatch the added member to the store:

```javascript
// web/static/js/actions/current_board.js

import Constants  from '../constants';

const Actions = {
  // ...

  connectToChannel: (socket, boardId) => {
    return dispatch => {
      const channel = socket.channel(`boards:${boardId}`);

      // ...

      channel.on('member:added', (msg) => {
        dispatch({
          type: Constants.CURRENT_BOARD_MEMBER_ADDED,
          user: msg.user,
        });
      });

      // ...
    }
  },
};

export default Actions;
```

And we have to do exactly the same for the boards:add, but dispatching the board:

```javascript
// web/static/js/actions/sessions.js

export function setCurrentUser(dispatch, user) {
  channel.on('boards:add', (msg) => {
    // ...

    dispatch({
      type: Constants.BOARDS_ADDED,
      board: msg.board,
    });
  });
};
Finally, we need to update the reducers so both the new member and the new board are added into the application state:

// web/static/js/reducers/current_board.js

export default function reducer(state = initialState, action = {}) {
  // ...

  case Constants.CURRENT_BOARD_MEMBER_ADDED:
    const { members } = state;
    members.push(action.user);

    return { ...state, members: members, showUsersForm: false };
  }

  // ...
}
// web/static/js/reducers/boards.js

export default function reducer(state = initialState, action = {}) {
  // ...

  switch (action.type) {
    case Constants.BOARDS_ADDED:
      const { invitedBoards } = state;

      return { ...state, invitedBoards: [action.board].concat(invitedBoards) };
  }

  // ...
}
```
Now the new member's avatar will appear in the list, and he will have access to the board and the necessary permissions to add and update new lists and cards.

![](/images/part9/board_4.jpg)  

If we recall the BoardMembers component we previously described, the className of the avatar depends on wether the member id exists in the connectedUsers list prop or not. This list stores all the ids of the currently connected members to the board's channel. To create and handle this list we will be using a longtime running stateful Elixir process, but we will do this on the next part. Meanwhile, don't forget to check out the live demo and final source code:

Live demo Source code
Happy coding!

