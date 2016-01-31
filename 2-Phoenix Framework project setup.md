#Trello clone with Phoenix and React (第二章节)

这篇文章属于基于Phoenix Framework 和React的Trello系列    

> 1. [介绍和架构选择](1-Intro and selected stack.md) <br/>
> 2. [Phoenix Framework 项目设置](2-Phoenix Framework project setup.md)  <br/>
> 3. [User模型和JWT权限设置](3-The User model and JWT auth.md) <br/>
> 4. [前端使用React 和 Redux实现用户注册](4-Front-end for sign up with React and Redux.md) <br/>
> 5. [数据库初始化和用户注册控制器](5-Database seeding and sign in controller.md)<br/>
> 6. [基于React和Redux的前端验证](6-Front-end authentication with React and Redux.md) <br/>
> 7. [sockets和channels 配置](7-Setting up sockets and channels.md)<br/>
> 8. 即将推出 <br/>


#项目设置


现在我们已经选择了我们的[框架](1-Intro and selected stack.md)，让我们开始创建新的Phoenix程序。在这之前我们的系统已经官方提供的[安装指南](http://www.phoenixframework.org/docs/installation)安装好了[Elixir](http://elixir-lang.org/)和[Phoenix](http://www.phoenixframework.org/)    


##Webpack打包静态资源

对比Ruby on Rails,Phoenix没有自己的资源管理，使用[Brunch](http://brunch.io/)作为其资源打包工具，Brunch使用起来很现代和灵活。如果我们不想用Brunch，也可以不使用Brunch，比如说[Webpack](https://webpack.github.io/)，这是很棒的事情。我以前也没有用过Brunch，因此后面就使用Webpack打包。

Phoenix 因为使用Brunch作为静态资源管理，node.js需要作为Phoenix[可选依赖](http://www.phoenixframework.org/docs/installation#section-node-js-5-0-0-)，同时Webpack也需要node.js，因此必须确保node.js安装正确。

第一步：不使用Brunch创建新的Phoenix项目：

```
$ mix phoenix.new --no-brunch phoenix_trello
  ...
  ...
  ...
$ cd phoenix_trello
```

对，现在我们创建了新的项目，项目没有使用资源打包工具。    

第二步：创建`package.json`文件，并安装Webpack作为dev依赖：

```
#原文使用npm start，查了查资料，应该使用npm init初始化package.json
$ npm init  
  ...
  ...
  ...
$ npm install  webpack --save-dev
```

现在`package.json`文件中可以看到与下面相似的内容：

```json
{
  "name": "phoenix_trello",
  "devDependencies": {
    "webpack": "^1.12.9"
  },
  "dependencies": {

  },
}
```

在项目中我们将使用一系列依赖，在这里就不列出他们，大家可以查看项目仓库的[源文件](https://github.com/bigardone/phoenix-trello/blob/master/package.json)。（备注：大家可以复制这个文件到项目文件夹，然后执行`npm install`就行）

第三步：我们需要添加[webpack.config.js](https://github.com/bigardone/phoenix-trello/blob/master/webpack.config.js)配置文件，便于Webpack打包资源:    


```javascript
'use strict';

var path = require('path');
var ExtractTextPlugin = require('extract-text-webpack-plugin');
var webpack = require('webpack');

// helpers for writing path names
// e.g. join("web/static") => "/full/disk/path/to/hello/web/static"
function join(dest) { return path.resolve(__dirname, dest); }

function web(dest) { return join('web/static/' + dest); }

var config = module.exports = {
  // our application's entry points - for this example we'll use a single each for
  // css and js
  entry: {
    application: [
      web('css/application.sass'),
      web('js/application.js'),
    ],
  },

  // where webpack should output our files
  output: {
    path: join('priv/static'),
    filename: 'js/application.js',
  },

  resolve: {
    extensions: ['', '.js', '.sass'],
    modulesDirectories: ['node_modules'],
  },

  // more information on how our modules are structured, and
  //
  // in this case, we'll define our loaders for JavaScript and CSS.
  // we use regexes to tell Webpack what files require special treatment, and
  // what patterns to exclude.
  module: {
    noParse: /vendor\/phoenix/,
    loaders: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        loader: 'babel',
        query: {
          cacheDirectory: true,
          plugins: ['transform-decorators-legacy'],
          presets: ['react', 'es2015', 'stage-2', 'stage-0'],
        },
      },
      {
        test: /\.sass$/,
        loader: ExtractTextPlugin.extract('style', 'css!sass?indentedSyntax&includePaths[]=' + __dirname +  '/node_modules'),
      },
    ],
  },

  // what plugins we'll be using - in this case, just our ExtractTextPlugin.
  // we'll also tell the plugin where the final CSS file should be generated
  // (relative to config.output.path)
  plugins: [
    new ExtractTextPlugin('css/application.css'),
  ],
};

// if running webpack in production mode, minify files with uglifyjs
if (process.env.NODE_ENV === 'production') {
  config.plugins.push(
    new webpack.optimize.DedupePlugin(),
    new webpack.optimize.UglifyJsPlugin({ minimize: true })
  );
}
```

这里需要指出的是，我们需要两个不同webpack[入口文件](https://webpack.github.io/docs/multiple-entry-points.html)，一个用于javascript，另一个用户stylesheets，都位于 web/static文件夹。我们的输出文件位于`private/static`文件夹。同时，为了使用了S6/7 和 JSX 特性，我们为此使用Babel来设计。

最后一步：当我们启动开发服务器时，告诉Phoenix每次启动Webpack，这样Webpack可以监控我们开发过程中的任何更改，产生主页面需要的最终资源文件。下面是在`config/dev.exs`添加一个监视：

```elixir
# config/dev.exs

config :phoenix_trello, PhoenixTrello.Endpoint,
  http: [port: 4000],
  debug_errors: true,
  code_reloader: true,
  cache_static_lookup: false,
  check_origin: false,
  watchers: [
    node: ["node_modules/webpack/bin/webpack.js", "--watch", "--color"]
  ]

...
```

如果我们现在运行服务器，可以看到Webpakc已经在运行并监视着项目：

```
$ mix phoenix.server
[info] Running PhoenixTrello.Endpoint with Cowboy using http on port 4000
Hash: 93bc1d4743159d9afc35
Version: webpack 1.12.10
Time: 6488ms
              Asset     Size  Chunks             Chunk Names
  js/application.js  1.28 MB       0  [emitted]  application
css/application.css  49.3 kB       0  [emitted]  application
   [0] multi application 40 bytes {0} [built]
    + 397 hidden modules
Child extract-text-webpack-plugin:
        + 2 hidden modules
```

还有一件事需要做，如果我们查看 `private/static/js`目录，可以发现`phoenix.js`文件。这个文件包含了我们需要使用 websockets和channels的一切,因此把这个文件复制到`web/static/js`文件夹中，这样便于我们使用。

##后端基本结构

现在我们可以编写代码了，让我们开始创建后端应用结构，需要以下npm包：

* bourbon , bourbon-neat, Sass 
* history 用于管理history .
* react 和 react-dom.
* redux 和 react-redux 用于状态处理.
* react-router 路由库.
* redux-simple-router 在线变更路由.

我不打算在stylesheets上面浪费更多的时间，后面始终需要修改它们。但是我需要提醒的是，我经常使用[css-burrito](http://css-burrito.com/)创建合适的文件结构来组织通过Sass文件，这是我个人认为非常有用的。

我们需要配置Redux存储，创建如下文件：

```javascript
//web/static/js/store/index.js

import { createStore, applyMiddleware } from 'redux';
import createLogger from 'redux-logger';
import thunkMiddleware from 'redux-thunk';
import reducers from '../reducers';

const loggerMiddleware = createLogger({
  level: 'info',
  collapsed: true,
});

const createStoreWithMiddleware = applyMiddleware(thunkMiddleware, loggerMiddleware)(createStore);

export default function configureStore() {
  return createStoreWithMiddleware(reducers);
}

```

基本上配置store使用两个中间件。
* redux-thunk 用于处理 async 动作.
* redux-logger 用于记录一切动作和浏览器控制台的状态变化

同时，需要传递所有reducer状态，创建下面的文件：

```javascript
//web/static/js/reducers/index.js

import { combineReducers }  from 'redux';
import { routeReducer }     from 'redux-simple-router';
import session              from './session';

export default combineReducers({
  routing: routeReducer,
  session: session,
});
```

开始我们只需要两个reducer，`routeReducer`自动设置路由管理状态变化，`session reducer`如下所示：

```javascript
//web/static/js/reducers/session.js

const initialState = {
  currentUser: null,
  socket: null,
  error: null,
};

export default function reducer(state = initialState, action = {}) {
  return state;
}
```

初始化状态包含`currentUser`对象，这些对象用于用户验证，以及需要连接channels部分的socket，和验证用户过程中出现的问题。

有了这些准备，可以编写`application.js`文件，和渲染Root组件：

```javascript
//web/static/js/application.js

import React                    from 'react';
import ReactDOM                 from 'react-dom';
import createBrowserHistory     from 'history/lib/createBrowserHistory';
import { syncReduxAndRouter }   from 'redux-simple-router';
import configureStore           from './store';
import Root                     from './containers/root';

const store  = configureStore();
const history = createBrowserHistory();

syncReduxAndRouter(history, store);

const target = document.getElementById('main_container');
const node = <Root routerHistory={history} store={store}/>;

ReactDOM.render(node, target);
```

我们创建store和history，并同步他们，便于先前的`routeReduce`很好的工作，在主应用布局中渲染Root组件，将作为一个Redux Provider作用与路由。

```javascript
//web/static/js/containers/root.js

import React              from 'react';
import { Provider }       from 'react-redux';
import { Router }         from 'react-router';
import invariant          from 'invariant';
import { RoutingContext } from 'react-router';
import routes             from '../routes';

export default class Root extends React.Component {
  _renderRouter() {
    invariant(
      this.props.routingContext || this.props.routerHistory,
      '<Root /> needs either a routingContext or routerHistory to render.'
    );

    if (this.props.routingContext) {
      return <RoutingContext {...this.props.routingContext} />;
    } else {
      return (
        <Router history={this.props.routerHistory}>
          {routes}
        </Router>
      );
    }
  }

  render() {
    return (
      <Provider store={this.props.store}>
        {this._renderRouter()}
      </Provider>
    );
  }
}
```


现在定义我们基本的路由文件：

```javascript
//web/static/js/routes/index.js

import { IndexRoute, Route }  from 'react-router';
import React                  from 'react';
import MainLayout             from '../layouts/main';
import RegistrationsNew       from '../views/registrations/new';

export default (
  <Route component={MainLayout}>
    <Route path="/" component={RegistrationsNew} />
  </Route>
);
```

我们的应用将包裹在`MainLayout`组件和`Root`路径中，路径将渲染`registrations`视图。文件最终版本可能有些复杂，后面将涉及到用户验证机制，这会在下一章节讲到。

最后，在主Phoenix应用布局中，我们添加`root`组件渲染的html文件：

```html
<!-- web/templates/layout/app.html.eex -->

<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="description" content="">
    <meta name="author" content="ricardo@codeloveandboards.com">

    <title>Phoenix Trello</title>
    <link rel="stylesheet" href="<%= static_path(@conn, "/css/application.css") %>">
  </head>

  <body>
    <main id="main_container" role="main"></main>
    <!main role="main">
    <!%= render @view_module, @view_template, assigns %>
    <!/main>
    <script src="<%= static_path(@conn, "/js/application.js") %>"></script>
  </body>
</html>
```

注意：link 和script标记，可以参考Webpack打包生产的静态资源。

因为我们是在前端管理路由，需要告诉Phoenix处理任何通过`index`动作产生的http请求，这个动作位于`PageControler`中，用于渲染主布局和`Root`组件：

```elixir
# master/web/router.ex

defmodule PhoenixTrello.Router do
  use PhoenixTrello.Web, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end

  scope "/", PhoenixTrello do
    pipe_through :browser # Use the default browser stack

    get "*path", PageController, :index
  end
end
```

现在就是这样。下一章节将介绍如何创建第一个数据库迁移，`User`模型和创建新用户的所需的所有功能。在此期间，你可以查看运行演示和下载最终的源代码：

[演示](https://phoenix-trello.herokuapp.com/)        [源代码](https://github.com/bigardone/phoenix-trello)

快乐编程吧！
