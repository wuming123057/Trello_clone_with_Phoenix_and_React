#Trello clone with Phoenix and React (第一章节)

这篇文章属于基于Phoenix Framework 和React的Trello系列    

[Trello](https://trello.com/) 是我一直喜欢的一款Web应用。从这款应用发布的时候我就使用它，我喜欢它的工作机制——简单而且灵活。每当我学习新技术的时候，自己很喜欢创建一个实际的应用，这样我可以把一切都放在实践中，从中学到实际中可能遇到的问题，以及查找并解决这些问题。当自己开始学习[Elixir](http://elixir-lang.org/)和[Phoenix framework](http://www.phoenixframework.org/)的时候，我很清楚应该把我即将遇到许多的未知问题都放到实践中，编写一个简单但功能完备的Trello，并分享他。

##创建的内容

基本上我们打算编写单页应用，用户可以登录、创建卡片、分享卡片以及用户之间添加列表和卡片。当查看一个卡片，将显示所有连接用户，任何修改将实时的以Trello风格自动反馈到每一个连接用户。

###当前架构

 **Phoenix**使用 **npm** 管理静态资源，可以使用 **Brunch** 或者 **Webpack**构建它们，非常简单的实现前后端分离，同时它们有相同的代码库。以下是后端需要使用的：

* Elixir.
* Phoenix framework.
* Ecto.
* PostgreSQL.

单页面前端将使用以下技术：
* Webpack.
* Sass 用于stylesheets.
* React.
* React router.
* Redux.
* ES6/ES7 JavaScript.

同时，将使用一些Elixir依赖和npm包，我将在使用的时候讨论他们。

###为什么是这个架构？

Elixir是一种基于Erlang的快速而且强大的函数式编程语言，同时拥有非常友好的类似于Ruby的语法。得益于Erlang的虚拟机，Elixir拥有非常强大且专业的并发特性，它可以自动地管理成千上万的并发进程。我是一名Elixir新手，因此还需要学习很多，但是我可以说经过这个练习后我将获得实质上提高。    

我们现在将使用Phoenix这个Elixir语言最流行的Web框架，这个框架不仅可以使用由Rails Web开发带来的部分部件和标准，而且提供了很多很酷的特点，例如上面提到的静态资源管理，更重要的是，box的实时功能可以通过Channels（原文为websockets）很容易实现，而且不需要额外的依赖（相信我，它非常有魅力）。    

另一方面，我们使用了React, react-router 和Redux，因为我喜欢使用这个组合来创建单页面应用和管理他们的状态。作为替代CoffeeScript，今年我开始使用ES6和ES7，这是一个非常好开始使用它的机会

###最终效果

应用包含四个不同页面。首先是用户注册和用户登录画面。    

 ![登录画面](/images/part1/sign-in.jpg)    

主页面包用户的含卡片列表，以及他添加其他用户的卡片列表：    

 ![卡片画面](/images/part1/boards.jpg)     

最后是卡片页面，所有连接的用户都可以看到，同时可以管理列表和卡片。     

  ![卡片内容](/images/part1/show-board.jpg)    

因此，这些就足够聊聊了。就看到这里吧，让我开始这个系列的第二章节。第二章节包含了如何创建一个Phoenix项目，并且如何使用Webpack替代Brunch作为Phoenix静态资源管理，以及后端基本设置。    

