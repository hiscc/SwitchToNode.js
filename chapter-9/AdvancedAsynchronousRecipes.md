# Chapter 9. Advanced Asynchronous Recipes

几乎我们见过的所有设计模式都被认为是通用并适用于应用的许多不同方面的。但是这里还有一系列的模式聚焦于解决特定的问题；我们可以叫这些模式为「食谱」。因为在真实的烹饪中，我们有一套定义好的步骤去遵循，跟着这些步骤就可以达到我们想要的结果。当然，这不意味着我们不可以根据自己的口味进行调节，但是这套流程很重要。在本章，我们将提供一些流行的诀窍来解决特定的问题，这些问题在我们每天开发 Node.js 时都会遇到：

* 导入异步初始化的模块
* 通过批处理和缓存异步操作，以便在繁忙的应用程序中获得性能提升，只需极少的开发工作
* 运行可以阻塞事件循环的同步 CPU 绑定操作，并削弱 Node.js 处理并发请求的能力

## 导入异步的初始化模块

在第二章中我们讨论了 Node.js 模块系统的基础属性，我们说到 require 方法是同步的，module.exports 不能是异步的。

这就是有很多 npm 包都支持同步 API 的原因，它们为模块直接初始化而不必去寻找替代异步 API 提供了便利。

不幸的是，这有时候行不通；因为不一定都有同步的 API，尤其是那些使用网络的初始化过程，例如，去执行一个握手协议或者取得一个配置参数。就像很多数据库驱动器和为客户端设计的消息队列。

### 权威解决方案

我们来看个例子：一个链接远程数据库的 db 模块。db 模块将在服务器连接并握手后接收请求。这里我们有两个选项：

* 在模块被使用前确保它已被初始化，否则就等待它初始化。我们不得不在调用异步模块时执行这个处理：

````JavaScript

const db = require('aDb'); //The async module

module.exports = function findAll(type, callback) {
 if(db.connected) {  //is it initialized?
   runFind();
 } else {
   db.once('connected', runFind);
 }
 function runFind() {
   db.findAll(type, callback);
 });
};

````

* 使用依赖注入（DI）而非直接导入异步模块。这样，我们可以延缓一个模块的初始化直到它们的异步依赖已经被完全初始化成功。这个技术使用父级组件将管理模块的复杂性转移到了另一个组件上：

````JavaScript
//app.js
//in the module app.js
const db = require('aDb'); //The async module
const findAllFactory = require('./findAll');
db.on('connected', function() {
 const findAll = findAllFactory(db);
});

//in the module findAll.js
module.exports = db => {
 //db is guaranteed to be initialized
 return function findAll(type, callback) {
   db.findAll(type, callback);
 }
}

````

就模版代码的数量来说，我们可以看到第一个选项没有第二个方案好。

当然，在第二个方案也有不好用的时候，正如我们在第七章里说的。在大项目内，它又能会引入过分的复杂度，尤其是手动且异步处理初始化的时候。这些问题将在我们使用 DI 容器的时候有所缓解。

正如我们将要看到的，还有第三种替代方案可以让我们轻松地将模块与其依赖项的初始化状态隔离开来。

### 预初始化队列

使用命令模式和队列可以简单地解藕模块初始化的依赖状态。这个方法将在模块未初始化前保存模块所有的操作直到初始化成功后才执行。

#### 实现一个异步初始化模块

为了演示这个简单有效的技术，我们构建一个测试应用：

````JavaScript
//asyncModule.js

const asyncModule = module.exports;

asyncModule.initialized = false;

asyncModule.initialize = callback => {
  setTimeout(function() {
    asyncModule.initialized = true;
    callback();
  }, 10000);
};

asyncModule.tellMeSomething = callback => {
  process.nextTick(() => {
    if(!asyncModule.initialized) {
      return callback(
        new Error('I dont have anything to say right now')
      );
    }
    callback(null, 'Current time is: ' + new Date());
  });
};

````

在代码内，asyncModule 尝试去展示异步初始化。它暴露一个会延迟 10 秒钟执行的 initialize 方法，并设置 initialized 为 true 然后通知它的回掉函数。tellMeSomething 方法返回当前的事件，如果模块还未初始化则返回一个错误。

下一步就是创建另一个模块：

````JavaScript
//routes.js
const asyncModule = require('./asyncModule');

module.exports.say = (req, res) => {
  asyncModule.tellMeSomething((err, something) => {
   if(err) {
      res.writeHead(500);
      return res.end('Error:' + err.message);
    }
    res.writeHead(200);
    res.end('I say: ' + something);
  });
};

````

处理句调用 tellMeSomething 方法，然后写入到 HTTP 响应内。如我们所见，我们不去检查 asyncModule 的初始化状态，这可能会导致问题。

现在，我们创建一个基本的 HTTP 服务：


````JavaScript
//app.js

const http = require('http');
const routes = require('./routes');
const asyncModule = require('./asyncModule');

asyncModule.initialize(() => {
  console.log('Async module initialized');
});

http.createServer((req, res) => {
  if (req.method === 'GET' && req.url === '/say') {
    return routes.say(req, res);
  }
  res.writeHead(404);
  res.end('Not found');
}).listen(8000, () => console.log('Started'));

````

现在，我们可以启动我们的应用了。在服务开启后，我们可以跳转到 http://localhost:8000/say，看到输出为： Error:I don't have anything to say right now。这就是说异步模块还未被初始化，但我们依然可以继续使用它。基于异步初始化的模块实现，我们可以收到错误提示，甚至可以挂掉整个应用。总的来说，我们必须避免这种情况。大多数情况下，有几个失败的请求是没关系的，而且初始化可能很快，在实践中，这绝不会发生；但是，对于高负载应用程序和设计为自动伸缩的云服务器，这两种假设都可能很快被消除。

#### 用预初始化队列包装模块

为了提高服务的健壮性，我们将使用我们在本部分前提到的设计模式重构我们的应用。我们将把所有在 asyncModule 未初始化前的操作放入队列内，然后一个个执行我们的队列任务。看起来像是状态模式的应用！我们将需要两个状态，一个是模块还未初始化的队列，另一个将在初始化完成后简单委托原生 asyncModule 模块的每一个方法。

我们也没有机会去修改异步模块的代码；所以加入我们的队列层，我们将需要围绕 asyncModule 模块创建一个代理。

我们把这个文件叫作 asyncModuleWrapper.js：


````JavaScript
const asyncModule = require('./asyncModule');

const asyncModuleWrapper = module.exports;

asyncModuleWrapper.initialized = false;
asyncModuleWrapper.initialize = () => {
  activeState.initialize.apply(activeState, arguments);
};

asyncModuleWrapper.tellMeSomething = () => {
  activeState.tellMeSomething.apply(activeState, arguments);
};

````

asyncModuleWrapper 简单委托它的每个方法给当前的活跃状态。我们来看看这两个状态：

````JavaScript
//notInitializedState
const pending = [];
const notInitializedState = {

  initialize: function(callback) {
    asyncModule.initialize(() => {
      asyncModuleWrapper.initalized = true;
      activeState = initializedState;                 //[1]

      pending.forEach(req => {                        //[2]
        asyncModule[req.method].apply(null, req.args);
      });
      pending = [];
      callback();                                     //[3]
    });
  },

  tellMeSomething: callback => {
    return pending.push({
      method: 'tellMeSomething',
      args: arguments
    });
  }
};

````

当 initialize 方法被调用时，我们通过一个回掉代理触发原生 asyncModule 的初始化。这样我们的包装器就知道原生模块何时初始化因此触发以下的操作；

1. 用下一个状态对象更新 activeState 变量。
1. 启动我们先前保存在 pending 中的所有命令。
1. 调用原生回掉。

因为模块这时还未初始化，这个状态的 tellMeSomething 方法创建一个新的命令对象然后把它添加到 pending 操作队列。

这时，当原生 asyncModule 模块还未初始化时，这个模式应该已经很清晰了，我们的包装器简单队列化了所有接收到的请求。然后当初始化完成时，我们执行所有队列内的操作并切换内部状态的 initializedState。最后一行代码： **let initializedState = asyncModule**

毫无疑问，initializedState 对象是原生 asyncModule 模块的引用！实际上，当初始化完成时，我们可以直接安全地路由任何请求到原生模块。

最后我们看到： **let activeState = notInitializedState**

我们可以尝试再次实施我们的测试服务，但是，别忘了替换原生 asyncModule 模块为我们新的 asyncModuleWrapper 对象；我们需要在 app.js 和 routes.js 模块内完成。

做完这个后，如果我们尝试给服务器发送一个请求，我们将在 asyncModule 模块还未初始化成功是不再接收到时报的请求；请求将挂起直到初始化完成，然后再执行。

现在，我们的服务可以立即接收请求了。我们在没有使用 DI 或者冗长的错误检查来确认异步模块的状态了。

### 真实情景

我们刚刚呈现的模式被很多数据库驱动和 ORM 库所使用。最值得一提的就是 [Mongoose](http://mongoosejs.com)，是 MongoDB 的 ORM。Mongoose 就不需要等待数据库连接后才能发送查询，因为每一个操作都被队列化了，在数据库成功连接后才执行。这大大提高了它 API 的可用性。


## 异步化的批处理和缓存

在高载荷应用中，缓存是关键。它几乎被 web 中的任何地方都用到，从像网页、图片、样式表一样的静态资源到像数据库查询内的纯数据。在本部分，我们将学习如何对异步操作进行缓存，如何将高要求的吞吐量转化为我们的优势。

### 实现一个没有缓存或批处理的服务

在我们开始新挑战前，我们来实现一个简单的事例，我们将用它来衡量我们将要实施的各种技术。
