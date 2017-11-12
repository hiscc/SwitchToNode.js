# Asynchronous Control Flow Patterns with ES2015 and Beyond

前一章我们学习了用回掉处理异步代码并理解了回掉地狱的产生。 在处理回掉地狱的方法上还有很多可选方案。

在本章， 我们将探索一下 promise 和 generators 还有 async await。

这些新方案处理起异步控制流将更加简单， 我们将试着理解它们并运用。

## Promise

Continuation Passing Style (CPS) 连续传入模式并不是处理异步代码唯一的方法。 在 JavaScript 生态中提供了很多可选方案， 其中 promise 已经被 ECMAScript 2015 纳入标准并在 Node.js 4 中可用。

### 什么是 promise

promise 是中抽象概念它允许函数返回一个 promise 对象， 而 promise 对象包含着异步操作的最终结果。 在 promise 内， 当异步操作没有完成时叫作 pending； 完成则叫 fulfilled； 失败则叫 rejected。 一旦 promise 处于 fulfilled 或 rejected 时， 就表明 promise 结束了。

我们使用 then() 方法来接收 fulfilled 或 rejected 值：

````JavaScript
promise.then([onFulfilled], [onRejected])
````

onFulfilled() 函数将接收成功的结果， onRejected() 将接收失败的结果。 它们都是可选的。


````JavaScript
asyncOperation(arg, (err, result) => {
  if (err) {
    //handle error
  }
  //do stuff with result
});

asyncOperation(arg)
  .then(result => {
    //do stuff with result
  }, err => {
    //handle error
  });
````

我们至关重要的 then() 方法会同步返回另一个 promise。 如果 fulfilled 或 rejected 会返回值， then() 方法将这样接收：

* Fulfill with x if x is a value
* Fulfill with the fulfillment value of x if x is a promise or a thenable
* Reject with the eventual rejection reason of x if x is a promise or a thenable

> A thenable is a promise-like object with a then() method. This term is used to indicate a promise that is foreign to the particular promise implementation in use.

这个特性允许我们启动链式 promise 然后我们简单聚合整理异步操作。 当然如果我们不处理 onFulfilled() 或 onRejected()， promise 的结果值将自动传递到下一个链式 promise。 这允许我们在整个 promise 链内传递错误， 直到出现 onRejected() 处理程序：

````JavaScript
asyncOperation(arg)
  .then(result1 => {
    //returns another promise
    return asyncOperation(arg2);
  })
  .then(result2 => {
    //returns a value
    return 'done';
  })
  .then(undefined, err => {
    //any error in the chain is caught here
  });
````

![](images/4.1.png)

promise 内另一个重要的属性是 onFulfilled() 和 onRejected() 函数， 它们是被异步调用的。

如果在 onFulfilled() 或 onRejected() 处理程序内有  throw 语句抛出的异常， promise 将在 then() 方法内自动 reject。 这点比 CPS 更好， 这意味着 promise 将自动穿越链式调用并把错误传递出来。

### Promises/A+ 实现

在 JavaScript 和 Node.js 中， 下面的库实现了 Promises/A+ 标准：

* Bluebird (https://npmjs.org/package/bluebird)
* Q (https://npmjs.org/package/q)
* RSVP (https://npmjs.org/package/rsvp)
* Vow (https://npmjs.org/package/vow)
* When.js (https://npmjs.org/package/when)
* ES2015 promises

它们之间最重要的区别就是部分实现的特性不同。

ES2015 promises 提供的 APIs:

Constructor (new Promise(function(resolve, reject) {})): 这会创建一个新的 promise， 并基于函数传入的参数进行处理。

* resolve(obj): 这将以 fulfillment 值来解决一个 promise， obj 是一个值的话。
* reject(err): 这将以一个 err 来 reject promise。

Static methods of the Promise object:

* Promise.resolve(obj): 这将创建一个在 then() 方法后或一个值的 promise。
* Promise.reject(err): 这将创建一个以 err 为原因的 reject
* Promise.all(iterable):  This creates a promise that fulfills with an iterable of fulfillment values when every item in the iterable object fulfills, and rejects with the first rejection reason if any item rejects. Each item in the iterable object can be a promise, a generic thenable, or a value.
* Promise.race(iterable): This returns a promise that resolves or rejects as soon as one of the promises in the iterable resolves or rejects, with the value or reason from that promise.

Methods of a promise instance:

* promise.then(onFulfilled, onRejected): 这是 promise 的必要方法。
* promise.catch(onRejected): 这个是 promise.then(undefined, onRejected) 的语法糖。

### promise 风格的 Node.js 函数

在 JavaScript 内， 不是所有的异步函数和库都支持了 promise。 大多数情况下， 我们必须做些转换， 这个转换称为 promisification。

我们使用 Promise 对象提供的构建器来生成一个新函数 promisify() 并放在 utilities.js 模块内：

````JavaScript
module.exports.promisify = function(callbackBasedApi) {
  return function promisified() {
    const args = [].slice.call(arguments);
    return new Promise((resolve, reject) => { //[1]
      args.push((err, result) => {            //[2]
        if (err) {
          return reject(err);                 //[3]
        }
        if (arguments.length <= 2) {          //[4]
          resolve(result);
        } else {
          resolve([].slice.call(arguments, 1));
        }
      });
      callbackBasedApi.apply(null, args);     //[5]
    });
  }
};
````

处理函数返回另一个叫 promisified() 的函数， 它代表 promise 版本的 callbackBasedApi：

1. promisified() 函数使用 Promise 构建器创建一个新的 promise 并立即返回给它的调用者。
1. 在 Promise 构建器内， 传入 callbackBasedApi 回掉， 我们简单把它加到参数列表 (args) 并提供给 promisified() 函数。
1. 在这个回掉里， 如果我们收到一个错误， 立即 reject promise。
1. 如果没有接收到错误， 我们以单个值或值数组来 resolve promise， 这取决去有多少结果被传入回掉内
1. 最后， 我们以参数列表来简单地调用 callbackBasedApi。

### 序列执行

我们来把新的技术用到爬虫内：

````JavaScript
const utilities = require('./utilities');
const request = utilities.promisify(require('request'));
const mkdirp = utilities.promisify(require('mkdirp'));
const fs = require('fs');
const readFile = utilities.promisify(fs.readFile);
const writeFile = utilities.promisify(fs.writeFile);

function download(url, filename) {
  console.log(`Downloading ${url}`);
  let body;
  return request(url)
    .then(response => {
      body = response.body;
      return mkdirp(path.dirname(filename));
    })
    .then(() => writeFile(filename, body))
    .then(() => {
      console.log(`Downloaded and saved: ${url}`);
      return body;
    });
}

 spider(process.argv[2], 1)
     .then(() => console.log('Download complete'))
     .catch(err => console.log(err));
````

最重要的是我们通过 readFile() 为 promise 注册了 onRejected() 函数以防页面还未被下载。 
