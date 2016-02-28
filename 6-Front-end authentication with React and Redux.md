#Trello clone with Phoenix and React (第六章节)

这篇文章属于基于Phoenix Framework 和React的Trello系列    

#前端用户登录

后端的作用是处理登录请求，让我们转移到前端，看看如何建立和发送这些请求，以及如何使用返回数据，并允许访问私有路由。

##路由文件

继续之前，让我们再次看看React路由文件：

```javascript
// web/static/js/routes/index.js

import { IndexRoute, Route }        from 'react-router';
import React                        from 'react';
import MainLayout                   from '../layouts/main';
import AuthenticatedContainer       from '../containers/authenticated';
import HomeIndexView                from '../views/home';
import RegistrationsNew             from '../views/registrations/new';
import SessionsNew                  from '../views/sessions/new';
import BoardsShowView               from '../views/boards/show';
import CardsShowView               from '../views/cards/show';

export default (
  <Route component={MainLayout}>
    <Route path="/sign_up" component={RegistrationsNew} />
    <Route path="/sign_in" component={SessionsNew} />

    <Route path="/" component={AuthenticatedContainer}>
      <IndexRoute component={HomeIndexView} />

      <Route path="/boards/:id" component={BoardsShowView}>
        <Route path="cards/:id" component={CardsShowView}/>
      </Route>
    </Route>
  </Route>
);
```

在第四章中，`AuthenticatedContainer`是为了防止未授权用户访问卡片内容，除非用户合法登录并返回有 **jwt**token。

#视图组件

现在我们需要创建`SessionsNew`组件，用于用户登录时需要呈现的表单：

```javascript
// web/static/js/views/sessions/new.js
import React, {PropTypes}   from 'react';
import { connect }          from 'react-redux';
import { Link }             from 'react-router';

import { setDocumentTitle } from '../../utils';
import Actions              from '../../actions/sessions';

class SessionsNew extends React.Component {
  componentDidMount() {
    setDocumentTitle('Sign in');
  }

  _handleSubmit(e) {
    e.preventDefault();

    const { email, password } = this.refs;
    const { dispatch } = this.props;

    dispatch(Actions.signIn(email.value, password.value));
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

  render() {
    return (
      <div className='view-container sessions new'>
        <main>
          <header>
            <div className="logo" />
          </header>
          <form onSubmit={::this._handleSubmit}>
            {::this._renderError()}
            <div className="field">
              <input ref="email" type="Email" placeholder="Email" required="true" defaultValue="john@phoenix-trello.com"/>
            </div>
            <div className="field">
              <input ref="password" type="password" placeholder="Password" required="true" defaultValue="12345678"/>
            </div>
            <button type="submit">Sign in</button>
          </form>
          <Link to="/sign_up">Create new account</Link>
        </main>
      </div>
    );
  }
}

const mapStateToProps = (state) => (
  state.session
);

export default connect(mapStateToProps)(SessionsNew);
```


基本功能是呈现表单和当表单确认时候调用`signIn`动作creator。同时，也讲连接`store`，获取通过`session` reducer更新的`store`，便于显示用户验证错误。

##action creator

下面是用户交互，让我们创建sessions相关的action creator:

```javascript
// web/static/js/actions/sessions.js

import { routeActions }                   from 'redux-simple-router';
import Constants                          from '../constants';
import { Socket }                         from '../phoenix';
import { httpGet, httpPost, httpDelete }  from '../utils';

function setCurrentUser(dispatch, user) {
  dispatch({
    type: Constants.CURRENT_USER,
    currentUser: user,
  });

  // ...
};

const Actions = {
  signIn: (email, password) => {
    return dispatch => {
      const data = {
        session: {
          email: email,
          password: password,
        },
      };

      httpPost('/api/v1/sessions', data)
      .then((data) => {
        localStorage.setItem('phoenixAuthToken', data.jwt);
        setCurrentUser(dispatch, data.user);
        dispatch(routeActions.push('/'));
      })
      .catch((error) => {
        error.response.json()
        .then((errorJSON) => {
          dispatch({
            type: Constants.SESSIONS_ERROR,
            error: errorJSON.error,
          });
        });
      });
    };
  },

  // ...
};

export default Actions;
```

`signIn`功能，将产生一个`POST`请求，参数包括`email`和用户提供的`password`。如果后端验证成功，将存储返回的`jwt`token到`localStorage`，并发送`currentUser` **JSON**到`store`。如果出现任何用户验证错误，都将发送错误反馈到用户登录表单。

##The reducer

让我们创建 `session reducer`:

```javascript
// web/static/js/reducers/session.js

import Constants from '../constants';

const initialState = {
  currentUser: null,
  error: null,
};

export default function reducer(state = initialState, action = {}) {
  switch (action.type) {
    case Constants.CURRENT_USER:
      return { ...state, currentUser: action.currentUser, error: null };

    case Constants.SESSIONS_ERROR:
      return { ...state, error: action.error };

    default:
      return state;
  }
}
```

这里我们就不说那么多了。很显然，我们需要修改`authenticated`container，就可以获取新`state`:

##验证container

```javascript
// web/static/js/containers/authenticated.js

import React            from 'react';
import { connect }      from 'react-redux';
import Actions          from '../actions/sessions';
import { routeActions } from 'redux-simple-router';
import Header           from '../layouts/header';

class AuthenticatedContainer extends React.Component {
  componentDidMount() {
    const { dispatch, currentUser } = this.props;
    const phoenixAuthToken = localStorage.getItem('phoenixAuthToken');

    if (phoenixAuthToken && !currentUser) {
      dispatch(Actions.currentUser());
    } else if (!phoenixAuthToken) {
      dispatch(routeActions.push('/sign_in'));
    }
  }

  render() {
    const { currentUser, dispatch } = this.props;

    if (!currentUser) return false;

    return (
      <div className="application-container">
        <Header
          currentUser={currentUser}
          dispatch={dispatch}/>

        <div className="main-container">
          {this.props.children}
        </div>
      </div>
    );
  }
}

const mapStateToProps = (state) => ({
  currentUser: state.session.currentUser,
});

export default connect(mapStateToProps)(AuthenticatedContainer);
```

当这个组件mounte后，如果有验证token，在`store`中却不是`currentUser`，会调用`currentUser`动作creator并从后端获取用户数据。让我们添加这部分代码：

```javascript
// web/static/js/actions/sessions.js
// ...

const Actions = {
  // ...

  currentUser: () => {
    return dispatch => {
      httpGet('/api/v1/current_user')
      .then(function(data) {
        setCurrentUser(dispatch, data);
      })
      .catch(function(error) {
        console.log(error);
        dispatch(routeActions.push('/sign_in'));
      });
    };
  },

  // ...
}

// ...
```
如果用户重新加载浏览器或者是没有之前注销再次访问根URL，数据将会覆盖。根据我们前面的步骤，用户登录后，将在state中设置`currentUser`，该组件将呈现正常显示标题组件及其嵌套的子路由。

##标题组件

这个组件将呈现用户图像和名字，以及卡片链接和注销按钮。

```javascript
// web/static/js/layouts/header.js

import React          from 'react';
import { Link }       from 'react-router';
import Actions        from '../actions/sessions';
import ReactGravatar  from 'react-gravatar';

export default class Header extends React.Component {
  constructor() {
    super();
  }

  _renderCurrentUser() {
    const { currentUser } = this.props;

    if (!currentUser) {
      return false;
    }

    const fullName = [currentUser.first_name, currentUser.last_name].join(' ');

    return (
      <a className="current-user">
        <ReactGravatar email={currentUser.email} https /> {fullName}
      </a>
    );
  }

  _renderSignOutLink() {
    if (!this.props.currentUser) {
      return false;
    }

    return (
      <a href="#" onClick={::this._handleSignOutClick}><i className="fa fa-sign-out"/> Sign out</a>
    );
  }

  _handleSignOutClick(e) {
    e.preventDefault();

    this.props.dispatch(Actions.signOut());
  }

  render() {
    return (
      <header className="main-header">
        <nav>
          <ul>
            <li>
              <Link to="/"><i className="fa fa-columns"/> Boards</Link>
            </li>
          </ul>
        </nav>
        <Link to='/'>
          <span className='logo'/>
        </Link>
        <nav className="right">
          <ul>
            <li>
              {this._renderCurrentUser()}
            </li>
            <li>
              {this._renderSignOutLink()}
            </li>
          </ul>
        </nav>
      </header>
    );
  }
}
```

当用户用户点击注销按钮，将调用session动作creator中的`signOut`方法。让我们添加这些代码：

```javascript
// web/static/js/actions/sessions.js
// ...

const Actions = {
  // ...

  signOut: () => {
    return dispatch => {
      httpDelete('/api/v1/sessions')
      .then((data) => {
        localStorage.removeItem('phoenixAuthToken');

        dispatch({
          type: Constants.USER_SIGNED_OUT,
        });

        dispatch(routeActions.push('/sign_in'));
      })
      .catch(function(error) {
        console.log(error);
      });
    };
  },

  // ...
}

// ...
```

它将发送DELETE请求到后端，成功后将从localStorage中移除`phoenixAuthToken`并调用`USER_SIGNED_OUT`动作，重置先前session reducer定义的state中的`currentUser`：

```javascript
// web/static/js/reducers/session.js

import Constants from '../constants';

const initialState = {
  currentUser: null,
  error: null,
};

export default function reducer(state = initialState, action = {}) {
  switch (action.type) {
    // ...

    case Constants.USER_SIGNED_OUT:
      return initialState;

    // ...
  }
}
```

##另外

尽管我们已经完成了用户登录过程，我们还有一个关键功能还没有实现，这也是我们编写的所有功能的核心：用户socket和channels。这个非常重要，我特别留了一章单独讲解它，下一章会仔细这个，`UserSocket`是什么，以及channels如何实现前后端双向连接，实时显示用户改变。同样，别忘记查看运行演示和下载最终的源代码：

[演示](https://phoenix-trello.herokuapp.com/)        [源代码](https://github.com/bigardone/phoenix-trello)
