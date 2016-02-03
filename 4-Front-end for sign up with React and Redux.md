#Trello clone with Phoenix and React (第四章节)

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

#用户注册

[上一章节](3-The User model and JWT auth.md)，我们创建了`User`模型，并完成了验证以及加密密码变更，同时更新了路由文件，创建了`RegistrationControlle`以便于处理新用户请求和返回 **JSON**数据以及 **jwt** token验证请求。现在让我们转移到前端这边。

#Recat路由准备

拥有两个公共路由是我们的主要目的，即`/sign_in`和`/sign_up`，任何访问者将能够通过他们登录到应用程序或这注册新的用户帐户。

另一方面我们需要`/`作为根路由，用于显示所有属于用户的卡片，`/boards/:id`路由用于显示所选择的用户开片。访问最后两个路由，用户必须是验证过的，否则，我们将他重定向到注册页面。

让我们更新`react-router`路由文件，如下所示：

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

export default (
  <Route component={MainLayout}>
    <Route path="/sign_up" component={RegistrationsNew} />
    <Route path="/sign_in" component={SessionsNew} />

    <Route path="/" component={AuthenticatedContainer}>
      <IndexRoute component={HomeIndexView} />

      <Route path="/boards/:id" component={BoardsShowView} />
    </Route>
  </Route>
);
```

复杂的部分是`AuthenticatedContainer`,让我们看看:

```javascript
// web/static/js/containers/authenticated.js

import React        from 'react';
import { connect }  from 'react-redux';
import { routeActions } from 'redux-simple-router';

class AuthenticatedContainer extends React.Component {
  componentDidMount() {
    const { dispatch, currentUser } = this.props;

    if (localStorage.getItem('phoenixAuthToken')) {
      dispatch(Actions.currentUser());
    } else {
      dispatch(routeActions.push('/sign_up'));
    }
  }

  render() {
    // ...
  }
}

const mapStateToProps = (state) => ({
  currentUser: state.session.currentUser,
});

export default connect(mapStateToProps)(AuthenticatedContainer);
```

我们主要做的是在这里，当组件增加，检测 **jwt** token是否在当前浏览器本地存储中。后面我们将看到如何设置它，但现在，让我们试想一下，它不存在，这要感谢`redux-simple-router`库将用户重定向到注册页面。

##注册视图组件

一旦发现他没有通过验证，我们将呈现给用户：

```javascript
// web/static/js/views/registrations/new.js

import React, {PropTypes}   from 'react';
import { connect }          from 'react-redux';
import { Link }             from 'react-router';

import { setDocumentTitle, renderErrorsFor } from '../../utils';
import Actions              from '../../actions/registrations';

class RegistrationsNew extends React.Component {
  componentDidMount() {
    setDocumentTitle('Sign up');
  }

  _handleSubmit(e) {
    e.preventDefault();

    const { dispatch } = this.props;

    const data = {
      first_name: this.refs.firstName.value,
      last_name: this.refs.lastName.value,
      email: this.refs.email.value,
      password: this.refs.password.value,
      password_confirmation: this.refs.passwordConfirmation.value,
    };

    dispatch(Actions.signUp(data));
  }

  render() {
    const { errors } = this.props;

    return (
      <div className="view-container registrations new">
        <main>
          <header>
            <div className="logo" />
          </header>
          <form onSubmit={::this._handleSubmit}>
            <div className="field">
              <input ref="firstName" type="text" placeholder="First name" required={true} />
              {renderErrorsFor(errors, 'first_name')}
            </div>
            <div className="field">
              <input ref="lastName" type="text" placeholder="Last name" required={true} />
              {renderErrorsFor(errors, 'last_name')}
            </div>
            <div className="field">
              <input ref="email" type="email" placeholder="Email" required={true} />
              {renderErrorsFor(errors, 'email')}
            </div>
            <div className="field">
              <input ref="password" type="password" placeholder="Password" required={true} />
              {renderErrorsFor(errors, 'password')}
            </div>
            <div className="field">
              <input ref="passwordConfirmation" type="password" placeholder="Confirm password" required={true} />
              {renderErrorsFor(errors, 'password_confirmation')}
            </div>
            <button type="submit">Sign up</button>
          </form>
          <Link to="/sign_in">Sign in</Link>
        </main>
      </div>
    );
  }
}

const mapStateToProps = (state) => ({
  errors: state.registration.errors,
});

export default connect(mapStateToProps)(RegistrationsNew);
```

对于这个组件，不能说太多，当它加载的时候，改变了`document`标题，同时呈现注册表单并调用注册登记action creator的结果。

##Action creator

当前面的表单确认后，我们想把数据发送到服务器并处理：

```javascript
// web/static/js/actions/registrations.js

import { pushPath }  from 'redux-simple-router';
import Constants     from '../constants';
import { httpPost }  from '../utils';

const Actions = {};

Actions.signUp = (data) => {
  return dispatch => {
    httpPost('/api/v1/registrations', {user: data})
    .then((data) => {
      localStorage.setItem('phoenixAuthToken', data.jwt);

      dispatch({
        type: Constants.CURRENT_USER,
        currentUser: data.user,
      });

      dispatch(pushPath('/'));
    })
    .catch((error) => {
      error.response.json()
      .then((errorJSON) => {
        dispatch({
          type: Constants.REGISTRATIONS_ERROR,
          errors: errorJSON.errors,
        });
      });
    });
  };
};

export default Actions;
```


当`RegistrationsNew`组件调用action creator传递表单数据，新的POST请求发送到服务器。请求被`Phoenix`路由过滤，并被`RegistrationController`处理（上一章节创建的）。如果成功，将返回`jwt` token，并存储到`localStorage`中，创建的用户数据将传递到`CURRENT_USER`动作中，最后重定向用户到根路由页面。相反，注册数据有任何错误，将产生`REGISTRATIONS_ERROR`动作与错误，这样我们就可以在形式向他们展示给用户。

为了处理这些`http`请求，我们将使用[isomorphic-fetch](https://github.com/matthew-andrews/isomorphic-fetch)包处理文件，其中一些有助于完成这个目的。

```javascript
// web/static/js/utils/index.js

import React        from 'react';
import fetch        from 'isomorphic-fetch';
import { polyfill } from 'es6-promise';

export function checkStatus(response) {
  if (response.status >= 200 && response.status < 300) {
    return response;
  } else {
    var error = new Error(response.statusText);
    error.response = response;
    throw error;
  }
}

export function parseJSON(response) {
  return response.json();
}

export function httpPost(url, data) {
  const headers = {
    Authorization: localStorage.getItem('phoenixAuthToken'),
    Accept: 'application/json',
    'Content-Type': 'application/json',
  }

  const body = JSON.stringify(data);

  return fetch(url, {
    method: 'post',
    headers: headers,
    body: body,
  })
  .then(checkStatus)
  .then(parseJSON);
}

// ...
```

##Reducers

最后的步骤是处理这些带reducers动作返回的结果，因此我们创建了程序需要的状态树。首先，让我们看看`session reducer`，用于设置`currentUser`:

```javascript
// web/static/js/reducers/session.js

import Constants from '../constants';

const initialState = {
  currentUser: null,
};

export default function reducer(state = initialState, action = {}) {
  switch (action.type) {
    case Constants.CURRENT_USER:
      return { ...state, currentUser: action.currentUser };

    default:
      return state;
  }
}
```

在这种情况下，任何注册错误都会更新状态，同时显示给用户。让我们添加注册`reducer`:

```javascript
// web/static/js/reducers/registration.js


import Constants from '../constants';

const initialState = {
  errors: null,
};

export default function reducer(state = initialState, action = {}) {
  switch (action.type) {
    case Constants.REGISTRATIONS_ERROR:
      return {...state, errors: action.errors};

    default:
      return state;
  }
}
```

注意：当我们从`utils`文件中调用`renderErrorsFor`函数用于显示错误：

```javascript
// web/static/js/utils/index.js

// ...

export function renderErrorsFor(errors, ref) {
  if (!errors) return false;

  return errors.map((error, i) => {
    if (error[ref]) {
      return (
        <div key={i} className="error">
          {error[ref]}
        </div>
      );
    }
  });
}
```

这就是所有的注册过程。下一章节，我们将看到已存在用户如何登录，并且访问个人部分。同样，别忘记查看运行演示和下载最终的源代码：

[演示](https://phoenix-trello.herokuapp.com/)        [源代码](https://github.com/bigardone/phoenix-trello)

快乐编程吧！