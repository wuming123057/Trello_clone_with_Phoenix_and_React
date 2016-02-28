#Trello clone with Phoenix and React (第七章节)

这篇文章属于基于Phoenix Framework 和React的Trello系列    

#Sockets and channels

上一章我们已经完成了验证过程，现在我们准备开始有趣的事情。从现在开始，我们将着重依赖Phoenix实时特性，用于处理前后端连接。用户卡片相关的任何事件都将反馈给用户，这些改变都将自动显示在用户屏幕上。

我们可以认为channnels或多或少类似于通常的控制器。不同于处理请求和返回单个连接响应，channels用于处理一个特定主题的双向事件，并可以广播给多个连接。，`Phoenix`使用socket句柄配置channels，验证和标识一个socket连接，并定义一个channel路由，便于处理各种请求指定不同的channel。

##user socket

当我们创建一个 **Phoenix**应用，将会自动配置一个默认的socket：

```elixir
# lib/phoenix_trello/endpoint.ex

defmodule PhoenixTrello.Endpoint do
  use Phoenix.Endpoint, otp_app: :phoenix_trello

  socket "/socket", PhoenixTrello.UserSocket

  # ...
end
```

将创建`UserSocket`，我们需要做些修改，便于处理正确的信息：

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

基本上我们需要两种不同的channels：

* `UserChannel`用于处理任何和"users:"相关的信息，以及通知与用户有关的事件，例如：当有人邀请用户加入一个board。
* `BoardChannel`功能是最多的，处理board、list和card相关的信息，通知用户正在查看的board在确切时刻发生的任何改变。

我们还需要实现`connect`和`id`两个功能：

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

当调用`connect`功能的时候，需要一个`token`参数，`connect`功能会验证这个`token`，获取用户的token（使用第三章创建的`GuardianSerializer`），并分配给socket，这样就可以在channels中使用了。此外，这还将阻止未经授权的用户连接到socket。

##user channel

现在我们需要建立socket,转移到`UserSocket`上面，这个非常简单：

```elixir
# web/channels/user_channel.ex

defmodule PhoenixTrello.UserChannel do
  use PhoenixTrello.Web, :channel

  def join("users:" <> user_id, _params, socket) do
    {:ok, socket}
  end
end
```

这个channel将允许我们从任何地方向任何用户广播信息，处理来自前端的信息。在特定的情况下，当用户的board将被其他用户共享，同时其他用户对它做了改变，我们将使用它广播给所有跟这个board有关的用户。我们还用于显示用户所拥有的board发生的改变，或者可以想的到的。

##连接 socket 和 channel

在继续之前，让我们回忆前一章最后有关的，在用户验证之后，不管是使用登录表单还是先前存储的`phoenixAuthToken`，我们都需要返回`currentUser`，并dispathch它到Redux store中，便于在header中显示用户头像和名字。这地方是展示连接socket和channel的好地方，让我们编写它：

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

After dispatching the user we create a new Socket from the Phoenix js library 
从`Phoenix`的js库中创建一个新的`Socket`，并dispatch给用户，然后添加`phoenixAuthToken` token，用于请求建立连接；同时调用`connect`函数。接下来在`socket`中添加新用户channel，并加入。当我们加入后收到`ok`消息，dispatch 这个`SOCKET_CONNECTED`动作，存储socket和channel：

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

在state存储它们，主要原因是我们在后面的许多地方需要他们，便于通过组件的`props`确保在state中可用。

现在，验证过的用户，连接到socket，并加入channel中，`AuthenticatedContainer`将渲染`HomeIndexView`视图，将显示用户所有的board以及共享的用户。下一章，我们将介绍创建新的board和邀请已存在的用户，使用channel向邀请的用户广播数据。同样，别忘记查看运行演示和下载最终的源代码：

[演示](https://phoenix-trello.herokuapp.com/)        [源代码](https://github.com/bigardone/phoenix-trello)