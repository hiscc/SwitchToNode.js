# Node.js Essential Patterns

去拥抱天生异步的 Node.js 非常重要， 特别是对那些不经常写异步代码语言的开发者。

在异步编程中我们习惯去假想一系列连续的操作来解决问题。 因为每一步操作都是阻塞的， 这意味着只有当前操作完成后才能执行下一步操作。

在异步编程中的一些操作， 例如读取文件或完成网络请求可以在后台启动。 当一个调用被启动时， 里一个调用会非常快地被启动甚至是在前一个操作还未完成时， 整个方程式应该在异步操作完成后出现适当的响应。

Node.js 提供了一系列工具和设计模式来处理这种异步代码。 在本章我们将要学习最重要的异步模式： 回掉和事件分发。

## 回掉模式

回掉是响应器模式的实体化处理， 它为 Node.js 带来了特别的编程风格。 回掉函数在操作完成后被调用。 回掉会替代经常同步调用的 ``return``。 函数是第一类对象它可以被当作值、 参数、 或者返回另一个函数或者被存储到数据结构内。 另一个实现回掉的理想结构式闭包。 在闭包内我们可以访问到函数的创建环境也可以不管回掉是否被调用时维持异步操作的上下文环境。

## 连续传递风格

在 JavaScript 中， 回掉可以被当作参数传递给另一个函数， 当操作结束后调用。 在函数编程内这被称作连续传递风格 continuation-passing style (CPS) 。 这是个普遍概念不经常与异步操作相联系。 实际上， 结果是被传送给另一个回掉函数了而非由调用者返回。

## 同步连续传递风格

````JavaScript
function add(a, b) {
     return a + b;
}
````

在这里， 结果被调用者用 ``return`` 直接返回； 这也叫作直接风格， 这是在同步编程中返回一个结果最普遍的方式。 而连续传递风格的函数是这样的：

````JavaScript
function add(a, b, callback) {
     callback(a + b);
}
````

``add()`` 函数是一个同步 CPS 函数， 这个函数将在回掉完成调用时返回一个值。  下面是示范：

````JavaScript
console.log('before');
add(1, 2, result => console.log('Result: ' + result));
console.log('after');
````

因为 ``add()`` 是同步的， 前面的代码将返回：

````JavaScript
before
Result: 3
after
````

## 异步连续传递风格
现在， 我们来看看异步下的 ``add()`` ：

````JavaScript
function additionAsync(a, b, callback) {
     setTimeout(() => callback(a + b), 100);
}
````

我们使用 ``setTimeout()`` 来模拟异步调用。

````JavaScript
console.log('before');
additionAsync(1, 2, result => console.log('Result: ' + result));
console.log('after');

before
after
Result: 3
````
因为 ``setTimeout()`` 触发一个异步操作， 它将不等待回掉被触发， 而是迅速返回， 把控制权交还给 ``additionAsync()``， 然后再回到它的调用者。 这在 Node.js 中至关重要， 因为它把控制权交还给了事件轮询当异步请求被触发时， 这样就允许一个队列处理一个新的事件。

![](images/2.1.png)

当异步操作完成时， 操作执行继续从回掉开始。 操作执行将从事件轮询开始， 所以这有一个新的栈。 由于闭包， 维持回掉函数的上下文已经不重要了， 尽管回掉在不同时不同点被调用了。

同步函数一直阻塞直到完成它的操作。 异步函数立即返回而且结果被放入在未来一个事件轮询内的处理程序中。

## 非阻塞连续传递风格的回掉

这里有些状况让我们以为一个函数是异步的或是使用了连续传递风格； 一般不是这样的， 我们来看看：

````JavaScript
const result = [1, 5, 7].map(element => element - 1);
console.log(result); // [0, 4, 6]
````

很清楚， 回掉只用来遍历数组内的元素， 而不是被传入操作结果。 这是用直接风格的同步操作。

## 同步还是异步？

我们已经明白了指令的顺序是如何根据功能同步或异步的性质而发生变化的。 这将影响整个方程式的流程。 下面就分析了这两种模式的利弊。

### 一个难以预测的函数

一个最危险的情况是一个 API 在某种情况下是同步的而在其它情况下却是异步的：

````JavaScript
const fs = require('fs');
 const cache = {};
 function inconsistentRead(filename, callback) {
   if(cache[filename]) {
     //invoked synchronously
     callback(cache[filename]);
   } else {
     //asynchronous function
     fs.readFile(filename, 'utf8', (err, data) => {
       cache[filename] = data;
       callback(data);
     });
   }
}
````

处理函数使用了 ``cache`` 变量来储存不同文件读取的结果。 这只是个例子， 缓存逻辑不是最优的。

### 解除束缚

现在来看看不可预测函数的用处：

````JavaScript
function createFileReader(filename) {
   const listeners = [];
   inconsistentRead(filename, value => {
     listeners.forEach(listener => listener(value));
   });
   return {
     onDataReady: listener => listeners.push(listener)
   };
}
````

当处理函数被调用时它创建了一个新的对象， 我们将可以建立一个文件读取操作。 所有的监听操作将在读取操作完成的时候一次调用。

````JavaScript
const reader1 = createFileReader('data.txt');
   reader1.onDataReady(data => {
     console.log('First call data: ' + data);
     //...sometime later we try to read again from
     //the same file
     const reader2 = createFileReader('data.txt');
     reader2.onDataReady( data => {
       console.log('Second call data: ' + data);
     });
});

// First call data: some data
````

如你所见第二个回掉将永不执行：
* 在创建 reader1 时， 我们的 inconsistentRead() 是异步的， 因为因为这里没有缓存结果可用。 因此， 我们一直在注册我们的监听器
* 然后， reader2 在事件轮询的循环内创建。 内部的 inconsistentRead() 将使同步的。 所以这个回掉将立刻被调用， 这也就是说所有
