# Chapter 8. Universal JavaScript for Web Applications

JavaScript 诞生自 1995 年，它的使命是赋予开发者构建更多动态交互的网站的能力。

从那以后，JavaScript 成长的很快，现在它已经是世界上最著名最广泛的语言之一了。很早以前， JavaScript 还是一门非常简单有限的语言。如今它可以被认为是一种完整的通用语言，甚至可以在浏览器之外用于构建几乎任何类型的应用程序。实际上，JavaScript 现在驱动了前端应用、web 服务、移动应用当然也可以内置到可穿戴设备内。

 JavaScript 的跨平台和跨设备能力引导了一种新的趋势，那就是把代码简化到在同一个项目在不同的环境下都可以运行。对于 Node.js 来说，最有意义的例子就是代码可以在服务端和客户端进行共享。这种追求代码在多个平台的重用以前被称为同构 JavaScript 现在被称为通用 JavaScript。

 在本章节，我们将探索奇妙的通用 JavaScript，尤其是在 web 开发领域，并发现那些可以在客户端和服务端进行共享的工具和技术。

 特别是，我们将学习如何在客户端和服务端使用相同的模块，并学习像 WebPack 和 Babel 这样的工具。我们将选用 React 库和其它著名的模块来构建 web 接口，在服务端和客户端共享状态，最后探索一些有趣的解决方案来使路由和数据检索变得通用。

 在本章最后，我们将写一个基于 React 的单页面应用（SPAs），这个应用的大部分代码可以共享给服务端，这样的应用一致且易于维护。


 ## 与浏览器共享代码

 Node.js 的一大卖点就是基于 JavaScript 和基于 V8 引擎。肃然如此但在服务端和客户端间共享代码也并不是那么简单。为双端开发代码需要极大的努力。因为这两个环境本质上是截然不同的。例如，Node.js 没有 DOM 或者长时间视图，而浏览器里没有文件系统或者开启新进程的能力。而且我们可以在 Node.js 中安全地使用许多 ES2015 的新特性，而在浏览器内则不能，因为浏览器主要的版本依然是 ES5，所以在浏览器内运行 ES5 代码依然是最保险的行为。

 所以为双端开发代码需要把差异缩减到最小。这可以在抽象和模式的帮助下完成，这些抽象和模式使应用程序能够在浏览器和 Node.js 兼容代码之间动态或在构建时切换。

 幸运的是，随着人们对这种新的令人兴奋的可能性越来越感兴趣，许多生态中的库和框架已经开始支持这两个环境了。这种发展也得到了越来越多支持这种新型工作流程工具的支持，这些工具多年来一直在不断完善和完善。这意味着如果我们在 Node.js 中使用一个包，这个包也可能无缝在浏览器中使用。但是，这并不意味着会一帆风顺，我们需要更小心的设计开发。

 在这一部分，我们将探索在为双端开发应用时将遭遇到的基本问题，并提出一些工具和模式来帮助我们完成挑战。

 ### 共享模块

 我们解决的第一个问题就是 Node.js 中的模块系统与浏览器内各种各样的模块系统的匹配问题。另一个问题是在浏览器内我们没有 require 函数或者可以导入模块的文件系统。如果我们想要在浏览器内共享大量代码，我们需要在浏览器内抽象 require 机制。

 #### 通用模块定义

 在 Node.js 中，我们知道 CommonJS 模块是在组件间创建依赖的默认机制。不幸的是，浏览器空间的情况更加分散；

* 我们可能有一个无模块系统的环境，这意味着 globals 是获取模块的主要机制。
* 我们有一个基于异步模块定义（AMD）的加载器的环境。例如， RequireJS。
* 我们可能有一个抽象了 CommonJS 模块系统的环境。

幸运的是，这里有一个叫通用模式定义（UMD）的模式，这个模式可以让我们从模块系统内抽象代码。

##### 创建一个 UMD 模块

UMD 到目前为止还不是一个标准，所以这里可能有很多根据组件需要和模块系统支持的变体。但是这里有一个可能是最流行的变体，它允许我们支持最通用的模块系统，例如， AMD、CommonJS 和浏览器 globals。

````JavaScript
//umdModule.js
(function(root, factory) {                           //[1]
  if(typeof define === 'function' && define.amd) {   //[2]
    define(['mustache'], factory);
  } else if(typeof module === 'object' &&            //[3]
      typeof module.exports === 'object') {
    var mustache = require('mustache');
    module.exports = factory(mustache);
  } else {                                           //[4]
    root.UmdModule = factory(root.Mustache);
  }
}(this, function(mustache) {                         //[5]
  var template = '<h1>Hello <i>{{name}}</i></h1>';
  mustache.parse(template);

  return {
    sayHello:function(toWhom) {
      return mustache.render(template, {name: toWhom});
    }
  };
}));
````
