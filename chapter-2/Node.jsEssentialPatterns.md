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
* 然后， reader2 在事件轮询的循环内创建。 内部的 inconsistentRead() 将使同步的。 所以这个回掉将立刻被调用， 这也就是说所有对 reader2 的监听都是同步调用的。 但是， 我们在创建 reader2 后注册监听器， 所以这些监听器将不会被调用。

### 使用同步 APIs

对上节例子中 inconsistentRead() 函数进行完全同步化便可以修复会出现的问题。 这是因为 Node.js 为基础 I/O 提供了一系列同步风格的 APIs。 例如  fs.readFileSync() ：

````JavaScript
const fs = require('fs');
 const cache = {};
 function consistentReadSync(filename) {
   if(cache[filename]) {
     return cache[filename];
   } else {
     cache[filename] = fs.readFileSync(filename, 'utf8');
     return cache[filename];
   }
}
````

记住把 API 从 CPS 风格转换到直接风格， 或者说是从异步风格转换到同步风格需要转换所有代码的使用风格。 在本例中我们就完全修改了 createFileReader() 的接口。

当然使用同步 API 有以下注意要点：

* 不是对应每个异步 API 都有其同步 API
* 同步 API 将阻塞事件轮询。 它将减慢方程式运行速度

在我们的 consistentReadSync() 函数中， 阻塞事件轮询带来的后果很轻因为同步的 I/O API 调用只在每个个文件名处启动， 缓存好的值将用于后面的调用。 如果我们限制了静态文件的数量然后再使用 consistentReadSync() 将不会对事件轮询产生大的影响。 当然如果我们一次读取很多的文件那就是另一回事了。 使用同步 I/O 操作在 Node.js 中的大多数情况下都不鼓励； 但是在一些情况下同步操作可能会有奇效。 在启动方程式的时候使用同步阻塞 API 来载入配置文件还是很有意义的。

当它们不影响服务器处理其它同时请求的情况下再使用阻塞 API。

### 延时执行

另一个来修复我们在上面出现的问题的可选方案是使用纯异步。 这里会使用一种延时执行的方案而非在事件轮询内立即启动。 在 Node.js 中， 我们可以使用 process.nextTick() 来延时执行直到下一轮的事件轮询。 它的功能非常简单； 它使用了回掉作为一个参数并把它放到事件队列的顶层， 即很多未开始的 I/O 事件并迅速返回。 回掉将在事件轮询再次开始时被调用。

我们这就来试试：

````JavaScript
const fs = require('fs');
   const cache = {};
   function consistentReadAsync(filename, callback) {
     if(cache[filename]) {
       process.nextTick(() => callback(cache[filename]));
     } else {
       //asynchronous function
       fs.readFile(filename, 'utf8', (err, data) => {
         cache[filename] = data;
         callback(data);
       });
     }
}
````

现在， 我们的函数保证会是异步的。

另一个用来延时执行的 API 是 setImmediate()。 它们的目标非常相似， 但是语义不一样。 process.nextTick() 回掉在所有其它 I/O 事件启动时。 而 setImmediate() 回掉是在已经准备好的 I/O 事件队列之后启动的。

## Node.js 回掉环境

在 Node.js 中， 连续传入风格 APIs 和回掉遵循一套一系列公约。 这些约定应用给 Node.js 核心 API 但是也遵循大量的用户模块。所以我们很有必要了解这些公约。

### 回掉在最后

在所有的 Node.js 核心方法中， 标准的公约是当函数接受回掉时在最后一个参数处放置回掉：

````JavaScript
fs.readFile(filename, [options], callback)
````

回掉一般放置在最后一个参数的位置， 这样可读性更高。

### 错误在最前面

在 CPS 中， 错误被作为任何一种结果来传送。 在 Node.js 中， 任何被  CPS 传入的错误都放置在第一个参数的位置， 任何实际的结果都放置在第二个参数处。 如果操作结果没有错误产生， 第一个参数将使 null 或 undefined：

````JavaScript
fs.readFile('foo.txt', 'utf8', (err, data) => {
   if(err)
     handleError(err);
   else
     processData(data);
});
````

这是种最佳实践而且易于调试。 另一个需要注意的点就是错误一直是 Error 类型。 这意味着简单的字符串或者数字将不应当作错误对象。

### 传递错误

同步传递错误， throw 语句被认为是直接的风格， 这将会导至直到错误被捕获时才弹出调用栈。

在异步 CPS 中， 简单的传入错误给下个回掉链很合适：

````JavaScript
const fs = require('fs');
function readJSON(filename, callback) {
 fs.readFile(filename, 'utf8', (err, data) => {
   let parsed;
   if(err)
     //propagate the error and exit the current function
     return callback(err);
   try {
     //parse the file contents
     parsed = JSON.parse(data);
   } catch(err) {
     //catch parsing errors
     return callback(err);
   }
   //no errors, propagate just the data
   callback(null, parsed);
 });
};
````

你应该注意到当我们想传递错误时该怎样传递有效结果。 我们使用 return 来传递错误。 这样出现错误时就不会调用下一行的代码了。

### 未捕获的异常

你可能已经看到 readJSON() 函数被用于避免任何异常出现在 fs.readFile() 回掉中了， 我们使用了 try...catch 块来包围 JSON.parse() 。 在异步函数内抛出异常将导致这个异常跳出事件轮询并不会被传递到下一个回掉内。

在 Node.js 内， 这是一种不可恢复的状态， 方程式将简单地打印处错误给 stderr 接口。 我们将去除 try...catch 块而示范：

````JavaScript
const fs = require('fs');
 function readJSONThrows(filename, callback) {
   fs.readFile(filename, 'utf8', (err, data) => {
     if(err) {
       return callback(err);
     }
     //no errors, propagate just the data
     callback(null, JSON.parse(data));
   });
};
````

在我们现在定义的函数内， 就没办法来捕获最终来自 JSON.parse() 的异常了。 如果我们试着解析一个未验证的 JSON 文件：

````JavaScript
readJSONThrows('nonJSON.txt', err => console.log(err));

// SyntaxError: Unexpected token d
//            at Object.parse (native)
//            at [...]
//            at fs.js:266:14
//            at Object.oncomplete (fs.js:107:15)
````

这将导致方程式因为异常突然被终止。

现在我们来追踪一下处理栈， 我们将看到它起始于 fs.js 模块， 确切说是来自读取 fs.readFile() 函数后通过事件轮询返回的结果。 这清晰地告诉我们异常来自我们的回掉然后直接进入事件轮询， 最后被捕获抛出。

这也意味着用 try...catch 块来包围 readJSONThrows() 将不会起作用， 因为在块内的处理栈不同于我们在回掉内的操作：

````JavaScript
try {
     readJSONThrows('nonJSON.txt', function(err, result) {
//... });
   } catch(err) {
     console.log('This will not catch the JSON parsing exception');
}
````

catch 语句将不会接收到 JSON 解析异常， 因为它将返回异常将抛出的栈。 我们只会看到栈在事件轮询处结束而不会看到触发异步操作。

正如以前说的， 方程式在异常到达事件轮询的时候发生崩溃； 但是我们依然可以做一点补救措施， 当发生崩溃时， Node.js 会分发一个特别事件 uncaughtException：

````JavaScript
process.on('uncaughtException', (err) => {
     console.error('This will catch at last the ' +
       'JSON parsing exception: ' + err.message);
     // Terminates the application with 1 (error) as exit code:
     // without the following line, the application would continue
     process.exit(1);
});
````

理解未捕获的异常在离开方程式时的状态不是连续的这点很重要。 例如， 一个未完成的 I/O 请求运行或闭包可能变成不连续的。 这就是为什么经常被建议， 特别是在生产环境内， 在收到未捕获异常后要退出方程式的原因。

## 模块系统和它的模式
