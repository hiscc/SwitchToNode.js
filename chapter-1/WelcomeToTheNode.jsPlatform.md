# Welcome to the Node.js Platform

开发者在 Node.js 平台及其生态环境中的一些经验总结出一系列准则和设计模式； 其中最奇怪的一项大概就是 Node.js 的天生异步以及它所带来的编程风格， 最简单的体现就是对回掉的严重运用。 在我们首次探究这些准则及设计模式时这些东西非常重要， 不仅仅是为了写出正确的代码， 而且还能在我们处理更多更复杂的问题上采取有效的设计策略。

从另一方面来看这也描绘了 Node.js 自己的哲学理念。 实实在在地探究 Node.js 而非简单地学习一门新技术； 这也会让我们拥抱文化和社区。 我们将见识到它是怎样对我们设计的方程式和组件以及对我们与社区间的交流产生巨大的影响，

还有就是最新版本的 Node.js 值得我们去学习， 从而了解更多 ES6 带来的新特性。 去拥抱这些新语法和新功能对于写出清晰可读的代码来说非常重要， 并且这种设计模式将贯穿本书。

在本章， 我们将学习如下主题：

1. Node.js 哲学理念
1. Node.js 第六版和 ES2015
1. 响应模式 -- Node.js 的核心机制异步架构


## Node.js 哲学理念

每个平台都有由一系列被社区所接受的准则和指南构成的设计哲学， 这由一些影响着平台的意识形态演变而来， 即方程式是如何被开发设计的。  一些准则来自技术本身， 一些来自生态平台， 一些只是社区趋势， 其他则由不同的意识形态的演变而来。 在 Node.js 中， 一些准则直接来自于它的创造者， Ryan Dahl； 来自那些核心贡献者； 来自社区领袖； 一些准则则遗传自 JavaScript 文化或是 Unix 设计哲学。

这些规则不是强制执行它们也需要看情况运用； 但是， 在我们设计方程式的时候会提供给我们非常有用灵感。

### 精简核心

Node.js 的核心本身构建在一些准则之上； 最小化功能便是其一， 留出所谓的用户区来， 即模块的生态环境。 这些准则便会对  Node.js 文化产生强大的冲击， 因为这给了社区在解决问题时试验和快速迭代的自由， 而非在紧密的控制与稳定核心上被强制慢慢演化。 保持核心功能的精简不仅仅在维护上很便利， 而且在项目生态的演变上也会产生积极的文化效应。

### 精简模块

Node.js 使用了一种模块功能的概念来作为程序的基础结构。 由构建方程式的块和重用的库构成了包（package）。 在 Node.js 中， 一个最基本的准则是最小模块原则， 不单以代码量来衡量， 作用范围是最重要的衡量指标。

这项准则源于 Unix 设计哲学， 具体对应下面两个准则：
* “Small is beautiful.”
* “Make each program do one thing well.”

Node.js 把这些概念带到一个全新的境界。 在 npm 的帮助下， Node.js 通过分离每个包自己一系列的依赖来解决依赖地狱。 Node 的方式在实际上需要极度高的重用性， 方程式凭此由若干高度集成的小模块构成。 尽管这些被认为不切实际或者在其他平台完全不可行， 在 Node.js 内却鼓励这种实践。 因此少于 100 行代码的函数在 npm 内很常见。

除了可用性极高外， 小模块也有如下好处：
* 易于理解使用
* 测试简单维护
* 在浏览器内共享

更小更集中的模块让每个人都可以分享或重用甚至是一小块代码； 这把不重复自己的准则推向了新高度。

### 小比表面积

为了精简， Node.js 模块通常会暴露出最小功能。 这会提高 API 的可用性也就是说 API 会更清晰。 大多数情况下模块使用者也不需要扩展模块。

在 Node.js 中只暴露出一个入口功能然后把次级功能设定为属性的设计模式很常见。 这将使功能更加清晰简单。

Node.js 中的模块大多用来直接使用而非扩展， 也许不够灵活但这简化了功能实现， 加快了维护， 也增加了利用率。

### 简化与实用性

你听过 KISS 法则吗：
> “Simplicity is the ultimate sophistication.” – Leonardo da Vinci

设计简单拒绝大而全的软件是一项最佳实践。 这将有利于快速实现功能容易适配易于理解维护， 这也会推进社区的参与并使软件完善本身。

在 Node.js 中， 这条法则同样适合 JavaScript。 实际上简单的函数， 闭包， 对象遍历代替了类继承。 纯类继承设计常常试图以高精度的计算来复制真实世界而忽略了真实世界内的缺陷和复杂度。事实上我们可能实现这些理解内的复杂度但不是尝试去完全实现完美的软件。

## Node.js 6 和 ES2015

本书将尽量使用新特性， 下面将简单介绍 ES2015 的新特性。

### let 和 const 关键字

基于历史原因 JavaScript 只支持函数作用域和全局作用域来控制变量的生命期和可见性。 就拿 if 语句来说吧， if 内的变量可以在全局内访问到：

````JavaScript
  if (false) {
      var x = "hello";
   }
   console.log(x);
   //undefined
````
在 ES2015 内引入了 let 关键字实现了块作用域：

````JavaScript
  if (false) {
      let x = "hello";
   }
   console.log(x);
   //ReferenceError: x is not defined
````

更有意义的例子就是用于循环了：

````JavaScript
for (let i=0; i < 10; i++) {
     // do something here
   }
   console.log(i);
````

这样我们的代码会更安全， 更易找到错误并避免副作用。

ES2015 引入了 const 关键字。 这个关键字允许我们申明一个常量：

````JavaScript
const x = 'This will never change';
      x = '...';
// TypeError: Assignment to constant variable
````

常量不可改变。 但是搞清楚 const 并不和其他语言内的常量的含义一致。 实际上对于 const 来说， 赋的值不是常量而绑定值是常量：

````JavaScript
const x = {};
x.name = 'John';
````
即 x 是个对象这是不变的， 但对象里的值可变

````JavaScript
x = null; // This will fail
````

const 在保护一些纯量值是很有用。

一项 const 的最佳实践就是引用模块的时候。

````JavaScript
const path = require('path');
// .. do stuff with the path module
let path = './some/path'; // this will fail
````

### 箭头函数

一项最棒的特性当属箭头函数。 箭头函数更加简洁并在处理回掉时更好用：

````JavaScript
const numbers = [2, 6, 7, 8, 1];
const even = numbers.filter(function(x) {
   return x%2 === 0;
});

const numbers = [2, 6, 7, 8, 1];
const even = numbers.filter(x => x%2 === 0);

const numbers = [2, 6, 7, 8, 1];
const even = numbers.filter(x => {
 if (x%2 === 0) {
   console.log(x + ' is even!');
   return true;
} });
````

如果返回值多余一行箭头函数只自动返回第一行而且需要加上大括号， 参数为空也需要在参数处加小括号。

箭头函数还有一点是绑定了父级块的作用域：

````JavaScript
function DelayedGreeter(name) {
    this.name = name;
}
DelayedGreeter.prototype.greet = function() {
 setTimeout( function cb() {
   console.log('Hello ' + this.name);
  }, 500);
};

const greeter = new DelayedGreeter('World');
greeter.greet(); // will print "Hello undefined"



DelayedGreeter.prototype.greet = function() {
 setTimeout( (function cb() {
   console.log('Hello' + this.name);
 }).bind(this), 500);
};

DelayedGreeter.prototype.greet = function() {
 setTimeout( () => console.log('Hello' + this.name), 500);
};
````

这是因为 setTimeout 内的回掉作用域和 greet 的作用域不一致， 以前我们必须使用 bind 重新绑定作用域， 现在只需要使用箭头函数即可。

### 类语法
