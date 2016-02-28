#Trello clone with Phoenix and React (第十章节)

这篇文章属于基于Phoenix Framework 和React的Trello系列    

Disclaimer:
This post is written before the Presence functionality and intended to be a small introduction to the basics of the GenServer behaviour.

#Tracking connect board members

Recalling last part, we supplied our users with the ability of adding new members to their boards. When an existing user email was added, the new relationship between users and boards was created and the new user was broadcasted along the channel so his avatar would be displayed to all connected members of the board. At first this is cool, but we can do it much better and useful if we could just highlight the members that are currently online and viewing the board. Let's get started!

##The problem

Before continuing let's first think about what do we want to achieve. So basically we have a board and multiple members that can suddenly visit its url automatically connecting them to the board channel. When this happens, the member's avatar should be displayed without opacity, contrary to offline members avatars which are displayed semitransparent to differentiate them.

![](/images/part10/board_4.jpg)  

When a connected member leaves the board's url, signs out or even closes his browser we want to broadcast again this event to all connected users in the board channel so his avatar gets semitransparent again, reflecting the user is no longer viewing the board. Let's think about some ways we could achieve this and their drawbacks:

1. Managing the connected members list on the front-end in the Redux store. This can sound as a valid approach at first but it will only work for members which are already connected to the board channel. Recently connected users will not have that data on their application state.
2. Using the database to keep track of connected members. This could also be valid, but will force us to constantly be hitting the database to ask for connected members and update it whenever a members connects or leaves, not to mention mixing data with a very specific user behavior.

So where can we store this information so it's accessible to all users in a fast and efficient way? Easy. In a... wait for it... long running stateful process.

##The GenServer behavior

Although long running stateful process might sound a bit intimidating at first, it's a lot more easier to implement than we might expect, thanks to Elixir and it's GensServer behavior.

> A GenServer is a process as any other Elixir process and it can be used to keep state, execute code asynchronously and so on.


Imagine it as a small process running in our server with a map containing the list of connected user ids per board. Something like this:

```elixir
%{
  "1" => [1, 2, 3],
  "2" => [4, 5]
}
```

Now imagine that this process ​had a public interface to init itself and update its state map, for adding or removing boards and connected users. Well, that's basically a GenServer process, and I say basically because it will also have underlying advantages like tracing, error reporting and supervision capabilities.

##The BoardChannel Monitor

So let's create our very basic version of this process which is going to keep track of the list of board connected members:

```elixir
# /lib/phoenix_trello/board_channel/monitor.ex

defmodule PhoenixTrello.BoardChannel.Monitor do
  use GenServer

  #####
  # Client API

  def start_link(initial_state) do
   GenServer.start_link(__MODULE__, initial_state, name: __MODULE__)
  end
end
```

When working with GenServer we have to think both in the external client API functions and the server implementation of them. The first we need to implement is the start_link one, which will really start our GenServer passing the initial state, in our case an empty map, as an argument among the module and the name of the server. We want this process to start when our application starts too, so let's add it to the children list of our application supervision tree:

```elixir
# /lib/phoenix_trello.ex

defmodule PhoenixTrello do
  use Application

  def start(_type, _args) do
    import Supervisor.Spec, warn: false

    children = [
      # ...
      worker(PhoenixTrello.BoardChannel.Monitor, [%{}]),
      # ...
    ]

    # ...
  end
end
```

By doing this, every time our application starts it will automatically call the start_link function we've just created passing the %{} empty map as initial state. If the Monitor happened to break for any reason, the application will also automatically restart it again with a new empty map. Cool, isn't it? Now that we have setup everything let's beging with adding members to the Monitor's state map.

##Handling joining members

For this we need to add both the client function and it's server callback handler:

```elixir
# /lib/phoenix_trello/board_channel/monitor.ex

defmodule PhoenixTrello.BoardChannel.Monitor do
  use GenServer

  #####
  # Client API

  # ...

  def member_joined(board, member) do
   GenServer.call(__MODULE__, {:member_joined, board, member})
  end

  #####
 # Server callbacks

 def handle_call({:member_joined, board, member}, _from, state) do
   state = case Map.get(state, board) do
     nil ->
       state = state
       |> Map.put(board, [member])

       {:reply, [member], state}
     members ->
       state = state
       |> Map.put(board, Enum.uniq([member | members]))

       {:reply, Map.get(state, board), state}
   end
 end
end
```

When calling the member_joined/2 function passing a board and a user, we will internally make a call to the GenServer process with the message {:member_joined, board, member}. Thus why we need to add a server callback handler for it. The handle_call/3 callback function from GenServer receives the request message, the caller, and the current state. So in our case we will try to get the board from the state, and add the user to the list of users for it. In case we don't have that board yet, we'll add it with a new list containing the joined user. As response we will return the user list belonging to the board.

Having this done, where should we call the member_joined method? In the BoardChannel while the user joins:

```elixir
# /web/channels/board_channel.ex

defmodule PhoenixTrello.BoardChannel do
  use PhoenixTrello.Web, :channel

  alias PhoenixTrello.{User, Board, UserBoard, List, Card, Comment, CardMember}
  alias PhoenixTrello.BoardChannel.Monitor

  def join("boards:" <> board_id, _params, socket) do
    current_user = socket.assigns.current_user
    board = get_current_board(socket, board_id)

    connected_users = Monitor.user_joined(board_id, current_user.id)

    send(self, {:after_join, connected_users})

    {:ok, %{board: board}, assign(socket, :board, board)}
  end

  def handle_info({:after_join, connected_users}, socket) do
    broadcast! socket, "user:joined", %{users: connected_users}

    {:noreply, socket}
  end

  # ...
end
```

So when he joins we use the new Monitor to track him, and broadcast through the socket the updated list of users currently in the board. Now we can handle this broadcast in the front-end to update the application state with the new list of connected users:

```javascript
// /web/static/js/actions/current_board.js

import Constants  from '../constants';

const Actions = {

  // ...
  connectToChannel: (socket, boardId) => {
    return dispatch => {
      const channel = socket.channel(`boards:${boardId}`);
      // ...

      channel.on('user:joined', (msg) => {
        dispatch({
          type: Constants.CURRENT_BOARD_CONNECTED_USERS,
          users: msg.users,
        });
      });
    };
  }
}
```

The only thing left is to change the avatar's opacity depending on whether the board member is listed in this array or not:

```javascript
// /web/static/js/components/boards/users.js

export default class BoardUsers extends React.Component {
  _renderUsers() {
    return this.props.users.map((user) => {
      const index = this.props.connectedUsers.findIndex((cu) => {
        return cu.id === user.id;
      });

      const classes = classnames({ connected: index != -1 });

      return (
        <li className={classes} key={user.id}>
          <ReactGravatar className="react-gravatar" email={user.email} https/>
        </li>
      );
    });
  }

  // ...
}
```
Handling member disconnection

The process when a user leaves the board channel is almost the same. Let's first update the Monitor to add the necessary client function and its server callback handler:

```elixir
# /lib/phoenix_trello/board_channel/monitor.ex

defmodule PhoenixTrello.BoardChannel.Monitor do
  use GenServer

  #####
  # Client API

  # ...

  def member_left(board, member) do
    GenServer.call(__MODULE__, {:member_left, board, member})
  end

  #####
  # Server callbacks

  # ...

  def handle_call({:member_left, board, member}, _from, state) do
    new_members = state
      |> Map.get(board)
      |> List.delete(member)

    state = state
      |> Map.update!(board, fn(_) -> new_members end)

    {:reply, new_members, state}
  end
end
```

As you can see, it's almost the same functionality as the member_joined but reversed. It looks for the board in the state and deletes the member, replacing the existing members list with this new one and returning it in the response. As in the join functionality we are also going to call this function from the BoardChannel so let's update it:

```elixir
# /web/channels/board_channel.ex

defmodule PhoenixTrello.BoardChannel do
  use PhoenixTrello.Web, :channel
  # ...

  def terminate(_reason, socket) do
    board_id = Board.slug_id(socket.assigns.board)
    user_id = socket.assigns.current_user.id

    broadcast! socket, "user:left", %{users: Monitor.user_left(board_id, user_id)}

    :ok
  end
end
```

When the connection to the channel terminates, it will broadcast the updated list of members through the socket just like we did before. To terminate the channel connection we will create an action creator that we'll use once the current board view is unmounted, and we also need to add the handler for the user:left broadcast:

```javascript
// /web/static/js/actions/current_board.js

import Constants  from '../constants';

const Actions = {

  // ...

  connectToChannel: (socket, boardId) => {
    return dispatch => {
      const channel = socket.channel(`boards:${boardId}`);
      // ...

      channel.on('user:left', (msg) => {
        dispatch({
          type: Constants.CURRENT_BOARD_CONNECTED_USERS,
          users: msg.users,
        });
      });
    };
  },

  leaveChannel: (channel) => {
    return dispatch => {
      channel.leave();
    };
  },
}
```

Don't forget to update the BoardsShowView component to dispatch the leaveChannel action creator when it unmounts:

```javascript
// /web/static/js/views/boards/show.js

import Actions              from '../../actions/current_board';
// ...

class BoardsShowView extends React.Component {
  // ...

  componentWillUnmount() {
    const { dispatch,  currentBoard} = this.props;

    dispatch(Actions.leaveChannel(currentBoard.channel));
  }

}
 // ...
```
And that's it! To test it just open two different browsers and sign in with a different user on each. Then navigate to the same board wit both and and play around entering and leaving with the other. You'll se his avatar transitioning from semitransparent and back again, which is pretty cool.

I hope you have enjoyed working with GenServer as much as I did the first time. But we have only scratched the surface. GenServer and Supervisors are very powerful tools Elixir offers us, which are completely native and bullet proof, without the need of third party dependencies contrary to Redis, for instance. In the next post we will continue creating lists and cards in realtime with the help of the socket and channels. Meanwhile, don't forget to check out the live demo and final source code:

Live demo Source code
Happy coding!


