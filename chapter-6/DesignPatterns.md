# Chapter 6. Design Patterns

设计模式是解决一个复现问题的可重用方案；定义的很宽它会跨越一个应用的多个领域。但是它经常会和一系列众所周知的面向对象模式产生关联。有本叫《Design Patterns: Elements of Reusable Object-Oriented Software, Pearson Education》的书在九十年代就很出名，它由赫赫有名的四人帮（GoF）所作：Erich Gamma, Richard Helm, Ralph Johnson, 和 John Vlissides。我们将经常提及一些传统的设计模式，或者是 GoF 设计模式。

在 JavaScript 中一系列面向对象设计模式不像经典的面向对象那样直接正式。正如我们所知的，JavaScript 是一门多范式面向对象基于原型拥有动态类型的语言；它把函数作为第一类公民，允许函数式编程。这些特性使 JavaScript 变得十分灵活，它赋给开发者强大的能力的同时也带来了破碎的编程风格、约定、技术，以及最终的生态模式。在 JavaScript 中有太多的办法来完成一件事。一个明显的佐证就是 JavaScript 中大量的框架库；也许其它语言都不曾有过这么多，特别是 Node.js 的出现更给 JavaScript 带来了全新的可能性。

在这种环境下，传统的设计模式也被 JavaScript 所影响。这里有很多方法也可由传统的设计模式所实现。在一些事例中，它们甚至不太可能，因为 JavaScript 不有真正的类或抽象接口。但每个设计模式原本的出发点和解决问题的关键概念没有变化。

在本章，我们将探索一些应用到 Node.js 和它设计哲学中的最重要的 GoF 设计模式，从而从另一个角度重新看待它们的重要性。在这些传统的设计模式中，我们也会找到一些因为 JavaScript 生态而没那么“传统”的设计模式。

我们将在本章探索一下设计模式：

* 工厂
* 揭露构造器
* 代理
* 装饰者
* 适配器
* 策略
* 状态
* 模版
* 中间件
* 命令

## 工厂

我们以可能在 Node.js 中最普遍最简单的设计模式开始： 工厂模式。

### 一个创建对象的通用接口

我们已经知道在 JavaScript 中函数式范例是纯面向对象的首选。因为它简约，可用，较小的表面积。尤其是在创建一个新对象实例时。实际上，通过调用工厂而不是直接在原型上使用 new 和 Object.create 来创建一个新对象是如此的方便和灵活，原因有几点：

首先，工厂可以让我们分离对象的创建和实现；工厂本质上包装了给我们更可控更灵活的一个新实例。在工厂内部，我们可以借助闭包、原型、new、Object.create() ，甚至基于特定的条件来返回一个不同的实例而创建一个新的实例。对工厂的消费者来说如何创建一个实例完全是不可知的。真相是通过 new 关键字我们把我们的代码绑定到一个指定的对象上，因为在 JavaScript 中我们可以很灵活，甚至可以说是自由。我们来看一个创建 Image 对象的例子：

````JavaScript
function createImage(name){
  return new Image(name)
}

const image = createImage('photo.jpg')
````

createImage 工厂看起来完全多此一举；为什么不直接用 new 操作符创建一个 Image 对象呢？就像下面这样：

````JavaScript
const image = new Image(name)
````

正如我们前面提及的，使用 new 绑定我们的代码到一个特别类型的的对象上；拿前面的例子说指 Image 对象类型。因为一个工厂拥有更多的灵活性；假设我们想去重构 Image 类，把它分割成更小的类，支持每一个图片格式。如果我们我们只暴露一个工厂来创建新的图片，我们可以这样做：

````JavaScript
function createImage(name) {
  if(name.match(/\.jpeg$/)) {
    return new JpegImage(name);
  } else if(name.match(/\.gif$/)) {
    return new GifImage(name);
  } else if(name.match(/\.png$/)) {
    return new PngImage(name);
  } else {
    throw new Exception('Unsupported format');
  }
}
````

我们的工厂也允许我们不暴露创造对象的构造器，并保护它们被修改或扩展。在 Node.js 中我们可以通过暴露一个工厂来保持构造器私有化。

### 强制封装机制

因为闭包的存在，工厂也可以用于强制封装。我们直到在 JavaScript 中我们没有入口级修饰符（例如我们没有私有变量），所以强制封装的唯一方法是通过函数作用域和闭包。工厂可以直接封装私有变量：


````JavaScript
function createPerson(name){
  const privateProperties = {}
  const person = {
    setName: name => {
      if (!name) throw new Error('A person must have a name')
      privateProperties.name = name
    },
    getName: () => {
      return privateProperties.name
    }
  }

  person.setName(name)
  return person
}
````

在代码内，我们运用闭包来创建了两个对象：一个被工厂返回的公共接口 person 对象，另一个是不可被外部访问到的只通过 person 对象提供接口暴露 privateProperties 对象。这样我们会确保 person 的 name 属性永不为空。

### 构建一个简单的代码探查器

我们通过构造一个简单的代码探查器来理解工厂，它有两个属性：

* start 方法触发探查 session
* end 方法终止 session 并记录日志

````JavaScript
// profiler.js

class Profiler {
  constructor(label) {
    this.label = label;
    this.lastTime = null;
  }

  start() {
    this.lastTime = process.hrtime();
  }

  end() {
    const diff = process.hrtime(this.lastTime);
    console.log(
      `Timer "${this.label}" took ${diff[0]} seconds and ${diff[1]}
        nanoseconds.`
    );
  }
}
````

我们使用了默认的时间方法来保存时间到 current time 上，然后在 end 方法调用时计算过去的时间并打印。
