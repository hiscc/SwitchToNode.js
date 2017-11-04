# Asynchronous Control Flow Patterns with Callbacks

从同步编程转换到一个充满连续传入风格和异步接口的平台来一定会充满困惑。 写回掉代码一定会是一种不同的体验特别是在控制流中。 异步代码可能难以预言哪个语句会被程序执行。 所以像是遍历文件、 执行队列任务或是等待一系列操作等操作需要开发者来采取一种新的方式来实现从而避免写出低效不可读的代码。 一个常见的错误就是陷入回掉地狱眼睁睁看着代码在水平方向无限延伸， 嵌套过深导致一个简单的流程变得复杂。

在本章， 我们将看看如何来通过一些法则和模式写出清晰可控的异步代码从而驯服回掉。 我们将看到如何操控流程库， 就像 async 可以明显简化问题来使我们的代码可读性可维护性更高。

## 异步编程的难点

让异步代码变得难以控制很简单。 闭包和原地定义的异步函数可以让我们不必关注其他点。 这完美符合 KISS 原则； 简单并使代码流可控。 不幸的是， 这样以牺牲模块性、 重用性和可维护性将在不久导致对回掉嵌套的失控和激增的函数数量最终失去组织力。 大多数情况下， 创建闭包不是因为功能需要， 所以它带来更多附加的惩罚。 记住我们的代码开始变得笨重， 而能否提前知道这些就是新手和专家的区别了。

## 创建一个简单的网络爬虫

为了解释这个问题， 我们将创建一个小爬虫， 一个由 web 地址取的内容并下载在本地文件命令行方程式：

* request： 一个 HTTP 请求库
* mkdir： 一个递归创建目录的工具

当然我们将时常提起一个本地模块 ./utilities， 它包含一些辅助函数。

我们方程式的核心功能包含在 spider.js 模块内：

````JavaScript
const request = require('request');
const fs = require('fs');
const mkdirp = require('mkdirp');
const path = require('path');
const utilities = require('./utilities');

function spider(url, callback) {
  const filename = utilities.urlToFilename(url);
  fs.exists(filename, exists => {                   //[1]
    if (!exists) {
      console.log(`Downloading ${url}`);
      request(url, (err, response, body) => {       //[2]
        if (err) {
          callback(err);
        } else {
          mkdirp(path.dirname(filename), err => {   //[3]
            if (err) {
              callback(err);
            } else {
              fs.writeFile(filename, body, err => { //[4]
                if (err) {
                  callback(err);
                } else {
                  callback(null, filename, true);
                }
              });
            }
          });
        }
      });
    } else {
      callback(null, filename, false);
    }
  });
}
````

1. 通过当前文件是否已经被创建来检查 URL 是否已经被下载
1. 未创建则继续下载
1. 确定目录下是不是包含已存在的文件
1. 最后把文件写入

为了启动我们的方程式我们在命令行内调用 spider() 函数：

````JavaScript
spider(process.argv[2], (err, filename, downloaded) => {
  if(err) {
       console.log(err);
     } else if(downloaded){
       console.log(`Completed the download of "${filename}"`);
     } else {
       console.log(`"${filename}" was already downloaded`);
     }
});
````

然后我们开始下载文件：``node spider http://www.example.com``

> 此处需要的其它代码如 utilities.js 需从此书官方网站下载

## 回掉地狱

在我们先前定义的 spider() 函数内， 我们实现的算法很直接而且有多层缩进导致可读性很差。

大量的闭包和就地的回掉定义使得代码可读性变差而且难以维护这被称为回掉地狱。 这通常被认为是反模式化的。 典型结构如下：

````JavaScript
asyncFoo( err => {
     asyncBar( err => {
       asyncFooBar( err => {
         //...
       });
  });
});
````

我们可以看到这种结构就像金字塔一样， 也被叫作毁灭金字塔。

最明显的问题就是多层次的嵌套导致可读性变差。 另一个问题是在每个作用域内变量名字的重叠。 回掉内有 err、 err1、 err2、 err3 参数；或者都使用相同的 err 参数。 两种方案都不理想。

## 使用原生 JavaScript

既然我们意识到遇到了回掉地狱， 我们就应该避免它； 但是， 这不是唯一需要担心的点。 实际上， 有很多情形都需要使用特别的技术和模式特别是在我们仅仅使用原生 JavaScript 来解决问题时。 例如在用异步处理一系列集合时， 这不像使用 forEach 遍历数组那么简单， 它需要使用一种类似递归的技术。

在本节， 我们不仅要学习避免回掉地狱而且要去用原生 JavaScript 去实现一些常见的控制流模式。

### 回掉法则

写异步代码时的第一个准则就是不要乱用闭包。 很简单， 但修复回掉地狱一般只需要少用回掉即可。

这里有一些基本准则：

* 你必须在可能时退出。 使用 return, continue, 或者 break， 依赖于上下本， 迅速退出当前的语句而非写下 if...else 语句。 这将是我们的代码变得浅一些。
* 你需要为回掉创建一个命名函数， 写在回掉函数外面并传入过渡结果作为参数。
* 你需要模块化代码。 把代码分割为更小， 重用性更高的函数， 尽可能。

### 运用回掉法则

我们现在就来试试这些法则并运用在前面的代码上：

````JavaScript
if(err) {
   callback(err);
} else {
   //code to execute when there are no errors
}

if(err) {
 return callback(err);
}
//code to execute when there are no errors
````
第一步， 我们要移除错误检查模式。 这将立即返回错误在我们接收到错误后。

用这个简单的技巧， 我们缩减了函数的嵌套深度； 非常简单。

> 一个常见错误就是发现错误后不返回
````JavaScript
if(err) {
  callback(err);
} //code to execute when there are no errors.
````
永远别忘记了在回掉执行完成后执行函数将继续执行。 插入一个 return 来阻塞后面的执行非常重要。 具体返回什么其实不重要。 我们还可以这样写：
````JavaScript
callback(...)
return;
````

现在我们来继续修改我们的 spider() 函数， 我们试着找出可重用的代码。 例如， 写一个把字符串传递给文件的函数：

````JavaScript
function saveFile(filename, contents, callback) {
   mkdirp(path.dirname(filename), err => {
     if(err) {
       return callback(err);
     }
     fs.writeFile(filename, contents, callback);
   });
}
````

同样的， 我们可以一个通用函数来下载：

````JavaScript
function download(url, filename, callback) {
   console.log(`Downloading ${url}`);
   request(url, (err, response, body) => {
     if(err) {
       return callback(err);
     }
     saveFile(filename, body, err => {
         if(err) {
           return callback(err);
         }
         console.log(`Downloaded and saved: ${url}`);
         callback(null, body);
     );
   });
}
````

最后我们修改好的 spider() 函数：

````JavaScript
function spider(url, callback) {
     const filename = utilities.urlToFilename(url);
     fs.exists(filename, exists => {
       if(exists) {
         return callback(null, filename, false);
       }
       download(url, filename, err => {
         if(err) {
           return callback(err);
         }
         callback(null, filename, true);
       })
    });
}
````

功能接口不变我们只是变化了代码组织。 通过运用基本原则， 我们大幅度削减了嵌套层级同时增加可读性和可测试性。 实际上， 我们可以暴露出 saveFile() 和 download() 函数， 然后在其它模块内重用同时也更易于测试。

### 序列执行

现在我们开始探索异步控制流模式。 我们以分析序列执行模式开始。

在序列内执行一系列任务意味着运行它们在某一段时间内， 一个完了又一个。 执行顺序很重要而且还必须稳定， 因为队列里任务的结果可能影响下一个执行：

![](images/3.1.png)

这里有些不同此流程的点：

* 在序列内启动一系列已知的任务， 不用链接或传递结果
* 使用任务的输出作为下一个的输入
* 当在每个元素上运行一个异步任务时遍历集合， 一个接一个

序列执行， 尽管在使用直接风格阻塞 API 实现， 但依然是在使用异步 CPS 时经常造成回掉地狱主要原因。

### 在序列内启动一系列已知任务

我们在实现 spider() 函数时已经实现了序列启动。 通过运用简单的法则， 我们已经能在序列执行流内组织一系列已知任务。 按指南后我们这样写：

````JavaScript
function task1(callback) {
   asyncOperation(() => {
     task2(callback);
   });
}
function task2(callback) {
   asyncOperation(result () => {
     task3(callback);
   });
}

function task3(callback) {
   asyncOperation(() => {
     callback(); //finally executes the callback
   });
}

task1(() => {
   //executed when task1, task2 and task3 are completed
   console.log('tasks 1, 2 and 3 executed');
});
````

处理模式展现了每个任务是如何调用下一个通过异步操作的。 模式把重点放在了任务的模块性上， 展现了在处理异步代码时不是必须使用闭包的。


### 序列遍历

在事先知道有多少任务将运行后， 我们先前描述的模式很完美。 这允许我们在序列内硬编码任务调用； 但是如果我们想要在每个项目内执行异步操作呢？ 这样我们就不能硬编码了， 我们必须动态构建。

#### 网络爬虫 2

为了演示序列遍历， 我们再次实现网络爬虫并加一个新功能。 我们现在递归下载一个页面内所有的链接。 为此， 我们需要取出页内所有的链接并为每个链接指定爬虫程序。

第一步就是使用新函数 spiderLinks() 去递归地下载所有的链接。

当然， 不是先检查文件是否存在， 我们现在试着去读取它， 然后爬取它的链接； 这样， 我们将可能先打断下载。 最后就是传递一个新的参数 nesting 帮助我们限制递归的深度：

````JavaScript
function spider(url, nesting, callback) {
  const filename = utilities.urlToFilename(url);

  fs.readFile(filename, 'utf8', (err, body) => {
         if(err) {
           if(err.code ! == 'ENOENT') {
             return callback(err);
           }
           return download(url, filename, (err, body) => {
             if(err) {
               return callback(err);
             }
             spiderLinks(url, body, nesting, callback);
            });
          }
         spiderLinks(url, body, nesting, callback);
  });
}
````

#### 序列爬取链接

现在我们来创建新版本爬虫的核心， spiderLinks() 函数使用一个序列化异步遍历算法来下载所有 HTML 页面。

````JavaScript
function spiderLinks(currentUrl, body, nesting, callback) {
  if(nesting === 0) {
    return process.nextTick(callback);
  }
  const links = utilities.getPageLinks(currentUrl, body);  //[1]
  function iterate(index) {                         //[2]
    if(index === links.length) {
      return callback();
    }
    spider(links[index], nesting - 1, err => {      //[3]
      if(err) {
        return callback(err);
      }

      iterate(index + 1);
    });
  }
  iterate(0);                                         //[4]
}
````

1. utilities.getPageLinks() 函数来获取全部页内链接。 这个函数只返回内部地址的链接（同一种主机名下）
1. 我们用 iterate() 函数遍历所有链接， 并取得下一个链接进去分析。 在本函数内， 我们首先要检查的就是索引值是否等于链接数组的长度， 是的话便调用回掉退出
1. 处理链接。 调用 spider() 函数并降低嵌套层次然后再在操作完成后调用下一次遍历
1. 我们以 iterate(0) 启动遍历
