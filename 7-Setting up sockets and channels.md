#Trello clone with Phoenix and React (第七章节)

这篇文章属于基于Phoenix Framework 和React的Trello系列    

#Sockets and channels

上一章我们已经完成了验证过程，现在我们准备开始有趣的事情。从现在开始，我们将着重依赖Phoenix实时特性，用于前后端连接。用户卡片的任何事件都将反馈给用户，这些改变都将自动显示在用户屏幕上。

We can think of channels more or less as common controllers. But instead of handling a request and returning a response to a single connection, they handle bidirectional events for a given topic which can be broadcasted to multiple connections. To configure them, Phoenix uses socket handlers which authenticates and identifies a socket connection and also defines channel routes that specify which channel is going to handle each request.

##The user socket

When creating a new Phoenix application, it automatically sets a default socket configuration for us:

```elixir
# lib/phoenix_trello/endpoint.ex

defmodule PhoenixTrello.Endpoint do
  use Phoenix.Endpoint, otp_app: :phoenix_trello

  socket "/socket", PhoenixTrello.UserSocket

  # ...
end
```

The UserSocket is also created, but we need to make some changes to it in order to make it handle the right messages:

```elixir
# web/channels/user_socket.ex

defmodule PhoenixTrello.UserSocket do
  use Phoenix.Socket

  alias PhoenixTrello.{Repo, User}

  # Channels
  channel "users:*", PhoenixTrello.UserChannel
  channel "boards:*", PhoenixTrello.BoardChannel

  # Transports
  transport :websocket, Phoenix.Transports.WebSocket
  transport :longpoll, Phoenix.Transports.LongPoll

  # ...
end
```

Basically we are going to have two different channels:

* The UserChannel will handle messages with any topic starting with "users:" and we will use it to inform users about events related to them, for example when they are invited to join a board for instance.
* The BoardChannel will have the most functionality; handling messages for managing boards, lists and cards, informing any user who might be viewing the board in that exact moment about any change.
We also need to implement the connect and id functions which will look like this:

```elixir
# web/channels/user_socket.ex

defmodule PhoenixTrello.UserSocket do
  # ...

  def connect(%{"token" => token}, socket) do
    case Guardian.decode_and_verify(token) do
      {:ok, claims} ->
        case GuardianSerializer.from_token(claims["sub"]) do
          {:ok, user} ->
            {:ok, assign(socket, :current_user, user)}
          {:error, _reason} ->
            :error
        end
      {:error, _reason} ->
        :error
    end
  end

  def connect(_params, _socket), do: :error

  def id(socket), do: "users_socket:#{socket.assigns.current_user.id}"
end
```

When the connect function is called with a token as parameter it will verify it, get the user from the token using the GuardianSerializer we created on part 3, and assign it to the socket so it's available in the channels if we might need it. Furthermore, this will also prevent unauthenticated users from connecting to the socket.

##The user channel

Now that we have set up the socket, let's move on to the UserSocket which is very simple:

```elixir
# web/channels/user_channel.ex

defmodule PhoenixTrello.UserChannel do
  use PhoenixTrello.Web, :channel

  def join("users:" <> user_id, _params, socket) do
    {:ok, socket}
  end
end
```

This channel will allow us to broadcast any user related message from anywhere, handling it from the front-end. In our particular case we'll use it to broadcast a board in which a user has been added as a member so we can add that new board to the user's boards list. We could also use it for displaying notifications about other boards he owns, or whatever you can imagine.

##Connecting to the socket and channel

Before continuing let's recall what we did at the end of the last post... after authenticating the user, whether it was using the sign in form or using a previously stored phoenixAuthToken, we needed to retrieve the currentUser to dispatch it to the Redux store so we could display the user's avatar and name in the header. This looks like a good place to connect to the socket and channel as well, so let's do some refactoring:

```javascript
// web/static/js/actions/sessions.js

import Constants                          from '../constants';
import { Socket }                         from '../phoenix';

// ...

export function setCurrentUser(dispatch, user) {
  dispatch({
    type: Constants.CURRENT_USER,
    currentUser: user,
  });

  const socket = new Socket('/socket', {
    params: { token: localStorage.getItem('phoenixAuthToken') },
  });

  socket.connect();

  const channel = socket.channel(`users:${user.id}`);

  channel.join().receive('ok', () => {
    dispatch({
        type: Constants.SOCKET_CONNECTED,
        socket: socket,
        channel: channel,
      });
  });
};

// ...
```

After dispatching the user we create a new Socket from the Phoenix js library adding the phoenixAuthToken token required to establish the connection, and we call the connect function. We proceed to create a new user channel from the socket and join it. When we receive the ok message from the join message, we dispatch the SOCKET_CONNECTED action to store both the socket and channel into the store:

```javascript
// web/static/js/reducers/session.js

import Constants from '../constants';

const initialState = {
  currentUser: null,
  socket: null,
  channel: null,
  error: null,
};

export default function reducer(state = initialState, action = {}) {
  switch (action.type) {
    case Constants.CURRENT_USER:
      return { ...state, currentUser: action.currentUser, error: null };

    case Constants.USER_SIGNED_OUT:
      return initialState;

    case Constants.SOCKET_CONNECTED:
      return { ...state, socket: action.socket, channel: action.channel };

    case Constants.SESSIONS_ERROR:
      return { ...state, error: action.error };

    default:
      return state;
  }
}
```

The main reason for storing them in the state is because we are going to need them in many places so having them in the state makes them available to components through their props.

Now that the user is authenticated, connected to the socket and joined to his channel, the AuthenticatedContainer will render the HomeIndexView view where we will display all the boards owned by the user as well as the ones he has been invited as a member. In the next post we will cover how to create a new board and invite existing users, using channels to broadcast the resulting data to the involved users. Meanwhile, don't forget to check out the live demo and final source code:

Live demo Source code
Happy coding!