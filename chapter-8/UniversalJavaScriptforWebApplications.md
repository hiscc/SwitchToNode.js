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

前面的例子定义了一个简单的模块。这里产出一个有 sayHello 方法的对象，这个方法会渲染 mustache 模版并返回给它的调用者。UMD 的目的就是集成环境中的其它模块系统。

1. 所有的代码都被包围在一个匿名的自执行函数内，这很揭露模块模式很像。这个函数接收一个全局命名空间对象（例如在浏览器中是 window 对象）。这主要用于将依赖项注册为全局变量，我们将在稍后看到。第二个参数是模块的 factory，一个返回模块实例并接受其依赖关系作为输入的函数（依赖注入）
1. 我们首先要做的事就是检查一下 AMD 是否在系统内可用。我们通过检查 define 函数作为标识来完成检查。如果发现有，就意味着在系统内 AMD 是可用的，所以我们使用 define 来定义模块。
1. 然后我们通过 module 和 module.exports 对象检查是否是 Node.js 环境。如果是就直接导入模块并赋值给 module.exports。
1. 如果既不是 AMD 环境也不是 Node.js 环境的话，我们使用 root 对象把模块赋值给全局变量。这个 root 对象在浏览器内将是 window 对象。
1. 最后如果包装函数是自调用的，提供 this 对象给 root，并提供我们的模块工厂作为第二个参数。

值得一说的是我们还未使用任何 ES2015 的新特性。因为这可以让我们的代码的适配性更强。

现在，我们来在 Node.js 和浏览器环境内使用我们的 UMD 模块。

首先，创建一个 testserver.js 文件：

````JavaScript
const umdModule = require('./umdModule');
console.log(umdModule.sayHello('Server!'));

````

如果我们执行这个脚本它将输出：

**<h1>Hello <i>Server!</i></h1>**

如果我们想要在浏览器中使用我们的模块，就创建一个 testBrowser.html 文件：

````Html
<html>
  <head>
    <script src="node_modules/mustache/mustache.js"></script>
    <script src="umdModule.js"></script>
  </head>
  <body>
    <div id="main"></div>
    <script>
document.getElementById('main').innerHTML =
         UmdModule.sayHello('Browser!');
    </script>
  </body>
</html>

````

这将在浏览器内输出 **Hello Browser！**

这里我们将 mustache 和 umdModule 作为普通脚本引入，然后创建了一个内联脚本。

##### 对于 UMD 模式的思考

UMD 模式是一个有效简单的技术来创建通用模块。但是我们也知道了它需要大量的模版，这些模版在每个环境下是难以测试的。当我们从头开始编写新模块时，这不是一种惯例；这是不可行和不切实际的，所以在这些情况下，最好将任务留给可以帮助我们自动化流程的工具。其中一个工具是 Webpack，我们将在本章中使用它。

我们也提及的 AMD， CommonJS 和浏览器全局变量不是唯一的模块系统。我们提及的模式覆盖到了大多数的情况，但也需要适配到其它情况。例如 ES2015 的模块规格。

#### ES2015 模块

ES2015 为我们带来了内建的模块系统。在本书内我们还未提及过这种模块系统，因为目前为止的 Node.js 还没有支持。

我们将深入这个模块系统的的细节，因为它有可能在将来成为一种模块语法；ES2015 模块系统不仅仅提出了一种好用的语法而且相比我们讨论的一些模块系统也拥有很多优点。

ES2015 的目标就是融合 CommonJS 和 AMD 模块的优势：

* 像 CommonJS，ES2015 需要特定的语法，对单一导出的偏好，以及对循环依赖的支持。
* 像 AMD，ES2015 提供了异步导入和配置模块导入。

而且，由于清晰的语法，我们可以使用静态分析器执行静态检查和优化等任务。对于实例，我们可以分析脚本依赖树并为浏览器创建打包文件。

今天，你也可以在 Node.js 中使用新的模块语法，采用像 Babel 这样的转换器。实际上，许多开发人员都在倡导它，同时提出自己的解决方案来构建通用 JavaScript 应用程序。通常一个好的想法是面向未来，特别是因为这个功能已经标准化，最终将成为 Node.js 核心的一部分。为简单起见，我们将在本章中全面遵循 CommonJS 语法。


### WebPack 介绍

在编写 Node.js 应用程序时，我们要做的最后一件事是手动添加对模块系统的支持，该模块系统与平台默认提供的模块系统不同。理想情况下，我们使用 require 和 module.exports 来编写模块，然后使用工具把代码转换到一个打包文件中供浏览器使用。幸运的是，WebPack 的出现解决了这个问题。


WebPack 允许我们使用 Node.js 的惯例来编写模块，由于编译的存在，这里将会创建一个包含了我们所有模块的依赖的打包文件供浏览器使用。WebPack 递归检查我们的源文件并查找 require 函数引用，然后把这些引用包含到打包文件内。

#### 探索神奇的 WebPack

我们以我们的 umdModule 为例展示 WebPack 的威力。首先安装 WebPack：

**npm install webpack -g**

-g 标识用于全局安装 WebPack， 然后我们才能使用 WebPack 的一些命令行功能。

然后，创建一个新的项目，并构建一个和 umdModule 相当的模块：

````JavaScript
//sayHello.js
var mustache = require('mustache');
var template = '<h1>Hello <i>{{name}}</i></h1>';
mustache.parse(template);
module.exports.sayHello = function(toWhom) {
 return mustache.render(template, {name: toWhom});
};

````

UMD 模式的简单使用，对吗？现在我们创建一个 main.js：

````JavaScript
window.addEventListener('load', function(){
  var sayHello = require('./sayHello').sayHello;
  var hello = sayHello('Browser!');
  var body = document.getElementsByTagName("body")[0];
  body.innerHTML = hello;
});

````

我们导入 sayHello 模块，然后安装 **npm install mustache**

现在我们来启动： **webpack main.js bundle.js**

这个指令将编译 main 模块并打包所有需要的依赖到一个 bundle.js 文件，在浏览器中引入这个文件。

我们创建一个 HTML 文件测试一下：

````HTML
<html>
  <head>
    <title>Webpack magic</title>
    <script src="bundle.js"></script>
  </head>
    <body>
    </body>
</html>

````

#### 使用 WebPack 的优势

WebPack 的神奇之处不仅仅如此。它还有很多特性。

* WebPack 自动提供了许多兼容浏览器的 Node.js 核心模块。这意味着我们可以在浏览器内使用像 http、assert、events 等等模块。
* 如果我们有一个与浏览器不兼容的模块，我们可以将其从构建中排除，或者将其替换为空对象或替换为另一个提供替代和浏览器兼容实现的模块。这是一个至关重要的功能，我们将有机会在我们即将看到的示例中使用它。
* WebPack 可以为不同的模块生成打包文件。
* WebPack 允许我们使用像加载器或插件对源文件执行额外的操作。这里有一堆我们需要的插件和加载器。
* WebPack 允许我们管理预处理所有的资源，包括 JavaScript、图片、字体、模版。
* 我们可以配置 WebPack 来分割依赖树并组织它到不同的块中。

WebPack 的灵活的配置使很多开发者开始使用它来管理客户端代码。这也让很多客户端的库开始支持 CommonJS 和 npm，开创了一个全新的局面。例如，我们可以这样安装 jQuery： **npm install jquery**。

然后在我们的项目内导入： **const $ = require('jquery')**

你讲对已有如此多的客户端库支持了 CommonJS 感到惊讶不已。

#### 使用 WebPack 和 ES2015

正如我们在前一段说的，WebPack 一个主要的优势就是可以使用加载器和插件在打包前对源代码进行转换。

这本树我们已经使用了 ES2015 标准的很多新特性，我们将继续在通用 JavaScript 应用中继续使用。在这一部分，我们将看看利用 WebPack 的加载器来重写我们使用 ES2015 的模块。假如适当的配置，WebPack 将把 ES2015 编译为浏览器支持 ES5 语言。

首先，我们把我们的模块移动到 src 文件夹下。这将让我们更加易于组织我们的代码。

现在，我们重写我们的模块。 我们 ES2015 版本的 src/sayHello.js 如下：

````JavaScript

const mustache = require('mustache');
const template = '<h1>Hello <i>{{name}}</i></h1>';
mustache.parse(template);
module.exports.sayHello = toWhom => {
  return mustache.render(template, {name: toWhom});
};

````

注意我们使用了 const、let 和箭头函数语法。

我们可以更新我们的 src/main.js 文件到 ES2015。如下：

````JavaScript

window.addEventListener('load', () => {
  const sayHello = require('./sayHello').sayHello;
  const hello = sayHello('Browser!');
  const body = document.getElementsByTagName("body")[0];
  body.innerHTML = hello;
});

````

如何配置 webpack.config.js：

````JavaScript
const path = require('path');

module.exports = {
  entry:  path.join(__dirname, "src", "main.js"),
  output: {
    path: path.join(__dirname, "dist"),
    filename: "bundle.js"
  },
  module: {
    loaders: [
      {
        test: path.join(__dirname, "src"),
        loader: 'babel-loader',
        query: {
          presets: ['es2015']
        }
      }
    ]
  }
};

````

这个文件暴露一个 WebPack 可读取的配置对象。

在这个配置对象内，我们定义了一个 entry 作为 src/mian.js 文件的端点，并设置打包文件为 dist/bundle.js。

现在我们来看看加载器数组。这个可选的数组允许我们指定一系列可以改变源代码的加载器。每个加载器都是一个特定的转换器（例如使用 babel-loader 将 ES2015 转换为 ES5），然后只在当前的文件匹配特定的 test 时才起效。在这个例子中，我们告诉 WebPack 对所有 src 文件夹下的文件使用 babel-loader。

现在我们执行 WebPack 操作 **npm install babel-core babel-loader babel-preset-es2015**， **webpack**

记住在 magic.html 中引用 dist/bundle.js 文件。


### 跨平台开发基础

当我们为不同的平台开发时，遇到的最普遍的问题就是我们不得不面对基于特定的平台提供一个方案的实现了。我们将探索一些面对这些问题时使用的原则和模式。

#### 代码运行时分支

一个最简单最易见的例子就是基于不同的主机平台来提供不同的实现。这要求我们有一种识别出运行时的主机平台的机制，然后通过 if...else 语句动态实现切换。一些通用的实现涉及到检查全局变量。例如，我们可以检查全局 window 是否存在：

````JavaScript
if(typeof window !== "undefined" && window.document) {
  //client side code
  console.log('Hey browser!');
} else {
  //Node.js code
  console.log('Hey Node.js!');
}

````

使用一个运行时分支实现在 Node.js 和浏览器定义间切换是最简单最易见的模式；但是也有一些不便之处：

* 对这两个平台的代码被包含到相同的模块所以也被放入到最后的打包文件内，因此一些不用的代码会增加打包文件的体积。
* 如果大面积使用，这将降低代码可读性，因为业务逻辑将被和跨平台逻辑相混合。
* 依赖于平台使用动态的分支来导入不同的模块将导致所有的模块都被添加到最后的打包文件内。例如，如果我们考虑一下下一个代码片段，clientModule 和 serverModule 将被包含到打包文件中，除非我们在构建时明确地排除它们：

````JavaScript
if(typeof window !== "undefined" && window.document) {
         require('clientModule');
       } else {
         require('serverModule');
       }

````

最后一个不便之处在于打包文件在构建时无法知道运行时变量的值（除非这个变量是一直不变的），所以它们将打包所有模块无论是否会被用到。

这最后一个属性的结果是动态使用变量所需的模块不包含在打包文件中。例如，对于下面的代码，没有模块将被打包：

````JavaScript
moduleList.forEach(function(module) {
  require(module)
})
````

值得注意的是 WebPack 克服了一些限制，在确定的环境下，它可以猜出变量所有可能的值。如果你有这样一段代码：

````JavaScript
function getController(controllerName) {
  return require("./controller/" + controllerName);
}
````

它将导入所有在 controller 文件夹下的模块变量。

#### 构建时代码分支

在这一部分我们将看看如何使用 WebPack 来移除不需要的文件。这样我们就有了更加轻量的打包文件并且避免了意外暴露出属于服务端的敏感代码。

WebPack 还支持插件，这些插件允许我们扩展我们的打包文件处理。去执行一个构建时代码分支，我们可以使用一个两个内建插件： DefinePlugin 和 UglifyJsPlugin。

DefinePlugin 可以帮助我们用自定义代码或变量替代一些代码。而且，UglifyJsPlugin 允许我们压缩我们的代码并移除未使用的语句。

我们在 mian.js 中实现：

````JavaScript
//main.js

if (typeof __BROWSER__ !== "undefined") {
  console.log('Hey browser!');
} else {
  console.log('Hey Node.js!');
}

//webpack.config.js
const path = require('path');
const webpack = require('webpack');

const definePlugin = new webpack.DefinePlugin({
  "__BROWSER__": "true"
});
const uglifyJsPlugin = new webpack.optimize.UglifyJsPlugin({
  beautify: true,
  dead_code: true
});

module.exports = {
  entry:  path.join(__dirname, "src", "main.js"),
  output: {
    path: path.join(__dirname, "dist"),
    filename: "bundle.js"
  },
  plugins: [definePlugin, uglifyJsPlugin]
};

````

DefinePlugin 插件允许我们用动态代码或常量值对特定的代码进行替换。我们把 __BROWSER__ 替换为 true。

第二个插件用于混淆和最小化打包文件内的 JavaScript 代码。 dead_code 选项用于移除所有没有用到的代码。


````JavaScript
//mian.js
if (true) {
  console.log('Hey browser!');
} else {
  console.log('Hey Node.js!');
}

console.log('Hey browser!')
````

现在我们的 mian.js 文件就是这样的了。beautify：true 选项用于避免移除所有的空格。

#### 模块交换

大多数时间，我们已经知道在构建时包含哪些代码了。这意味着我们可以在构建时对一个模块进行替换。这养就会生成更轻量的打包文件了，因为我们排除了不需要的模块。

我们将构建一个暴露函数 alert 的模块。我们将有两种不同的实现，一个是服务器端一个用于浏览器端：

````JavaScript
//alertServer.js
module.exports = console.log

//alertBrowser.js
module.exports = alert
````

代码很简单。如你所见，我们仅仅是使用了默认的函数。他们都接受一个字符串作为参数。

现在在写我们的通用 main.js 文件：

````JavaScript
//main.js
const alert = require('./alertServer');
alert('Morning comes whether you set the alarm or not!');

````

然后配置：

````JavaScript
//webpack.config.js
const path = require('path');
const webpack = require('webpack');

const moduleReplacementPlugin =
  new webpack.NormalModuleReplacementPlugin(/alertServer.js$/,
    './alertBrowser.js');

module.exports = {
  entry:  path.join(__dirname, "src", "main.js"),
  output: {
    path: path.join(__dirname, "dist"),
    filename: "bundle.js"
  },
  plugins: [moduleReplacementPlugin]
};

````

我们使用 NormalModuleReplacementPlugin 插件，第一个参数是一个正则表达式。第二个参数是资源路径。在构建时，如果一个资源被匹配到，它将被第二个参数里的文件替代。

在我们的事例中， alertBrowser 将替代 alertServer 模块。

我们也可以用 npm 上的库进行替换，比如用 toastr 替换浏览器原生的 alert。

**npm install jQuery toastr**

现在我们可以使用 toastr 重写我们的 alertBrowser 模块。

````JavaScript
const toastr = require('toastr')
module.exports = toastr.info
````

toastr.info 函数接收一个字符串作为参数，并在浏览器的右上方展示一个信息框。

我们的 WebPack 配置文件依然不变，但是这次它将自动解决新的依赖树，包括 jQuery 和 toastr。

还有，服务器端版本的 main.js 文件依然不变。

由于 WebPack 和模块替换插件，我们可以在不同平台间更容易处理这些问题。我们可以聚焦在基于平台的模块，然后使用模块替换插件并打包到一起。


#### 跨平台开发的设计模式

既然我们知道了如何在 Node.js 和浏览器代码之间进行切换，剩下就是如何集成我们的设计模式了，如何创建一个部分组件可以变换的组件。

* 策略和模版：这两个模式可能是在浏览器内共享代码最有用的模式了。他们的目的是定义一个通用的算法，并允许其中的部分可以变换，而这正是我们需要的！在跨平台开发中，这些模式允许我们共享组件中平台无关的部分，同时对哪些基于平台的部分使用不同的策略或者模版进行更改。
* 适配器：这个模式可能在我们替换整个组件时很有用。
* 代理：当代码想同时运行在两个平台上时。远程代理模式正好派上用场：想象一下如果我们像在浏览器端获取到 fs 对象的情况。我们就可以在客户端创建一个 fs 对象并代理服务器端 fs 模块的所有请求，使用 Ajax 或者 Web Socket 来交换命令或者返回值。
* 观察者：观察者在组件间分发接收事件时提供了天然的抽象。在跨平台开发中，这意味着我们可以用浏览器特定的实现代替发射体，而不改变监听器，反之亦然。
* DI 和 服务定位器：这两个模式都可以用在注入的模块实现上。

正如我们所看到的，我们掌握的模式库非常强大，但最强大的武器仍然是开发人员选择最佳方法并使其适应手头的具体问题的能力。 在下一节中，我们将把我们学到的东西付诸实践，利用我们目前所见的一些概念和模式。

## React 介绍
