# Chapter 7. Wiring Modules

Node.js 的模块系统极大填充了 JavaScript 语言中的断层：缺乏原生的方式来把代码组织到独立的单元中。一个最大的优点就是使用 require 函数把模块组织到一起，简单有力。但是许多 Node.js 新手还是会问：用哪种方式是把组件 X 的实例传入模块 Y 中最好。

有时这种混乱导致在希望找到一种更熟悉的方式将我们的模块链接在一起时对单例模式绝望的追求。换句话说，这可能会过度随意使用依赖注入模式导入任何类型的依赖。模块的写法在 Node.js 中是最具有争议和主观性的话题。有许多思想流派影响着这个领域，但没有一个可以被视为拥有无可争议的真理。每一种实现都有其优缺点并且他们经常最终在同一个应用程序中混合在一起，改编，定制或以其他名称伪装使用。

在本章，我们将分析各种把模块连接起来的实现。并比较它们的优缺点进而帮助我们在简约性、可重用性、拓展性方面进行权衡。

总的来说，我们将展示关于这个主题的几种最重要的模式：

* 硬编码依赖
* 依赖注入
* 服务定位器
* 依赖注入容器

然后我们将探讨一个密切相关的问题，即如何连接插件。这是在连接模块时经常遇到的问题，它们呈现出相同的特征，但是依据的应用上下文略有差异，特别是在一个插件被当作 Node.js 包来发布时。我们将学习创建一个插件化的架构然后聚焦如何把这些插件集成进主要应用的流程内。

本章结尾，令人摸不着头脑的模块连接问题将不再困惑我们。

## 模块和依赖

每一个现代化的应用总结起来就是一些模块的聚合产物，因为随着应用的增长，我们连接这些组件的方式变得异常重要。这不仅仅关系到技术方面，例如拓展性，而且也关系到整个系统的运行。纠结的依赖图是一种负担，它增加了项目的技术债；与此同时，任何代码的改变无论是修改或者拓展都将导致极大的副作用。

最糟糕的是，组件紧密地连接在一起，最后我们只能重构或完全重写这一部分。当然，这并不意味着我们必须从第一个模块就开始过度设计我们的设计，但从一开始就找到一个很好的平衡将产生深远的影响。

Node.js 提供了一个伟大的工具来组织连接模块。它就是 CommonJS 模块系统。但是单独的模块系统无法保证成功；换句话说，它虽然为客户端和依赖间提供了便利，如果使用不当，它可能会引入更紧密的耦合。在这部分，我们将讨论一些 Node.js 中的基本方面。

### Node.js 中最普遍的依赖

在软件架构中，我们称任何影响组件行为或结构的实体、状态，或者数据格式为依赖。例如，一个组件可能使用了另一个组件的服务，依托特定的系统全局状态或者实现一种特定的交流协议来与其它组件交流信息等等情况。依赖的概念非常广泛而且很难被界定。

但是，在 Node.js 中，我们可以立即识别出一种基本的依赖类型，这是最常见且易于识别的; 当然，我们正在讨论模块之间的依赖关系。模块是我们处理和组织代码结构的基本机制；在不依赖模块系统的情况下构建大型应用程序是不合理的。如果恰当的把各类元素组合在一起，将带来巨大的好处。实际上，模块的属性可以这样总结：

* 一个模块更加可读且便于理解，理想情况下它的内举行更高
* 以单一文件呈现，模块易于区分
* 模块可以在不同的应用间重用

模块呈现了信息隐藏的完美粒度级别，而且是仅提供公开组件的公共接口的有效机制（使用 module.exports）。

但是，仅仅通过应用或者库的功能来简单分割模块还远远算不上成功的设计。其中一个谬误将在我们的删除或替换模块时被终结。我们迅速认识到把代码组织进模块内并把它们连接起来将非常重要。同时，就像软件设计中的其它问题一样，在不同的标准间找到平衡才是关键。

### 内聚和耦合

在构建模块时最重要的两点属性就是内聚和耦合。它们可以被运用到软件架构中的任意组件或子系统，所以我们把这两点作为构建 Node.js 模块时的指导方针。

* 内聚：衡量一个组件中功能间的相关性。例如，一个模块只做一件事，这就体现出了这个模块的搞内聚性。一个包含好几种保存对象到数据库的功能 -- saveProduct()、saveInvoice()、saveUser()，它的内聚性就很低。
* 耦合：衡量一个模块依赖多少个其它模块。例如，当一个模块直接从另一个模块内读取数据时，我们说耦合很高。当然，如果两个模块通过全局或共享状态也是耦合很高。换句话说，两个模块只通过传递参数来交流的话就是松耦合。

我们的目标是高内聚与松耦合，这样的模块会更易读，可用性更高也更容易拓展。

### 有状态的模块

在 JavaScript 中，一切都是对象。我们没有像纯接口或这类的抽象概念；它的动态类型已经提供了一种将接口（或策略）与实现（或细节）分离的自然机制。

在 JavaScript 中，我们在将接口与实现分离时遇到是最小的问题；但是，通过简单使用 Node.js 模块系统，我们已经引入了一种硬编码的特殊实现。一般情况下，这没有问题，但是如果我们使用 require 来导入一个暴露状态化实例的模块时，例如像 db 处理、HTTP 服务实例、服务实例、或者是任何无状态的对象，实际上我们都是引用了一个类似单例的东西，因此继承了它的优点和缺点，并增加了一些注意事项。

#### Node.js 中的单例模式

很多 Node.js 新手对于如何在 Node.js 中正确实现单例充满困惑，大部分时间都是为了在应用程序的各个模块之间共享实例。但是，答案比我们想象的简单；使用 module.exports 简单导出一个实例就已经足够拥有一个类似于单例的东西了。

````JavaScript
// 'db.js' module

module.exports = new Database('my-app-db')
````

通过简单导出一个数据库实例，我们已经获取了 db 模块的实例了。因为 Node.js 会在第一次调用 require 方法后缓存模块，并在后面的调用中直接返回这个模块的缓存。例如，我们简单拥有一个 db 共享的实例：

````JavaScript
const db = require('./db')
````

这里需要注意的点是，模块以它的全路径作为键被缓存，因此这里单例只是当前的包的单例。

````JavaScript
//package.json
{
  "name": "mydb",
  "main": "db.js"
}

// file structure
app/
`-- node_modules
    |-- packageA
    |  `-- node_modules
    |      `-- mydb
    `-- packageB
        `-- node_modules
            `-- mydb

````

packageA 和 packageB 都依赖于 mydb 包；app 包依赖于 packageA 和 packageB。但是 packageA 和 packageB 包都将导入各自的 mydb 模块，因为它们的路径是不同的。

从这点来说，我们字面上描述的单例并不存在于 Node.js 中，除非我们使用一个真正的全局变量来保存它：


````JavaScript
global.db = new Database('my-app-db')
````

这将保证实例在整个应用内是唯一且共享，而不像是个包一样根据导入路径而无法保持状态。大多数情况下，我们不需要一个真正的单例，如果真的需要，我们将在后面看到另一个模式来在不同的包之间共享一个实例。

## 连接模块的模式

我们已经讨论了围绕依赖和耦合的一些基本思想，现在来进一步看一些实际的概念。在这部分，我们将呈现一个模块连接的主要模式。毫无疑问，通过有状态的实例来连接模块是应用中最重要的依赖。

### 硬编码的依赖

我们以分析两个模块间最常规的关系即硬编码依赖为开端。在 Node.js 中，通过 require 方法倒入一个模块时就是这种情况。这种方法创建的模块依赖简单有效，我们必须特别关注有状态实例的硬编码依赖关系。

#### 使用硬编码依赖构建一个授权服务

通过下面的图示来分析：

![](images/6.10.png)

这幅图显示了一种典型的层级架构；这就是一个简单的授权系统的结构。 AuthController 接收客户端的收入，在请求中获取到登录信息并进行一些初步验证。然后它依赖 AuthService 来检查这个用户是否和数据库内的用户匹配；这些查询依赖于 db 模块处理。这三个组件连接起来的方式将决定可重用性、可测试性和可维护性的程度。

最自然的处理方式是在 AuthService 内导入 db 模块，然后在 AuthController 内导入 AuthService。这就是我们所讨论的硬编码。

我们在系统中来展示一下我们刚刚讨论的实现，进而实现一个简单的授权服务：

* POST '/login'：它接收一个包含 username 和 password 的 JSON 对象来授权。成功的话就返回一个 JSON Web Token(JWT)，它可以在请求时校验用户身份。
* GET '/checkoutToken'：他将从 GET 请求内获取查询参数并对之校验。

##### db 模块

我们由底层开始构建我们的应用；首先我们需要暴露 levelUp 数据库实例。创建 lib/db.js:


````JavaScript
const level = require('level');
const sublevel = require('level-sublevel');

module.exports = sublevel(
  level('example-db', {valueEncoding: 'json'})
);
````

我们简单创建一个到 LevelDB 数据库的连接并储存在 ./example-db 目录下，然后使用 sublevel 插件对其进行装饰，它增加了对创建和查询数据库的不同部分的支持（可以将其与 SQL 表或 MongoDB 集合进行比较）。模块导出的对象是数据库句柄本身，它是一个有状态实例; 因此，我们正在创造一个单例。

##### authService 模块

既然我们拥有了 db 单例，我们可以用它来实现我们的 lib/authService.js 模块：

````JavaScript
// ...
const db = require('./db');
const users = db.sublevel('users');

const tokenSecret = 'SHHH!';

exports.login = (username, password, callback) => {
  users.get(username, function(err, user) {
      // ...
    });
  };

exports.checkToken = (token, callback) => {
  // ...
  users.get(userData.username, function(err, user) {
    // ...
  });
};

````

authService 模块实现了 login 服务和 checkToken 服务。我们将讨论 db 模块，db 变量包含了一个已经初始化的数据库句柄，我们可以直接在上面处理数据操作。

##### AuthController 模块

````JavaScript
// lib/authController.js
const authService = require('./authService');

exports.login = (req, res, next) => {
  authService.login(req.body.username, req.body.password,
    (err, result) => {
      // ...
    }
  );
};

exports.checkToken = (req, res, next) => {
  authService.checkToken(req.query.token,
    (err, result) => {
      // ...
    }
  );
};

````

authController 模块实现了两个路由来处理 HTTP 请求和响应。

在这个模块内，我们依然硬编码了一个状态模块：authService。是的，authService 也成了一个状态模块因为它直接依赖于 db 模块。这样我们就理解了硬编码的依赖是如何在整个应用内传播的。

##### app 模块

````JavaScript
//app.js

const express = require('express');
const bodyParser = require('body-parser');
const errorHandler = require('errorhandler');
const http = require('http');

const authController = require('./lib/authController');
const app = module.exports = express();
app.use(bodyParser.json());

app.post('/login', authController.login);
app.get('/checkToken', authController.checkToken);

app.use(errorHandler());
http.createServer(app).listen(3000, () => {
  console.log('Express server started');
});

````

##### 启动服务

**node app**

我们通过 curl 命令来调用我们的服务：

**curl -X POST -d '{"username": "alice", "password":"secret"}' http://localhost:3000/login -H "Content-Type: application/json**

上面的代码应该返回一个 token，然后我们把它放在下面的 <TOKEN HERE> 处：

**curl -X GET -H "Accept: application/json" http://localhost:3000/checkToken?token=<TOKEN HERE>**

然后会返回下面的字符串：

**{"ok":"true","user":{"username":"alice"}}**

##### 硬编码依赖的优缺点

我们前面的例子展示了利用 Node.js 中的模块系统来管理在组件间的依赖的方便之处。我们从模块内暴露一个状态化的实例，让 Node.js 来管理它的生命周期，然后从应用的其它部分导入它。结果是一个直观的直观组织，易于理解和调试，每个模块初始化并连接自己，无需任何外部干预。

换句话说，硬编码依赖限制了模块再次连接一个其它的实例，这将降低重用性和可测试性。例如，在组合内重用 authService 和其它数据库实例就不太可能了，因为这个硬编码依赖依赖的是同一个实例。而且独立测试 authService 的任务也是非常艰难的，因为我们不能简单模拟模块使用的数据库。

作为最后一个考虑因素，重要的是要看到使用硬编码依赖项的大多数缺点都与有状态实例相关联。 这意味着如果我们使用 require 来加载无状态模块，例如工厂，构造函数或一组无状态函数，我们就不会遇到同样的问题。我们仍将与特定实现紧密耦合，但在 Node.js 中，这通常不会影响组件的可重用性，因为它不会引入与特定状态的耦合。

### 依赖注入

依赖注入（DI）模式也许是在软件设计中最被误解的概念。许多人将这个术语与框架和 DI 容器相关联，例如 Spring（用于 Java 和 C＃ ）或者 Pimple（用于 PHP ），但实际上它是一个更简单的概念。依赖注入模式背后的主要思想是由外部实体提供的组件的依赖关系。

这个实体可以是一个客户端组件或一个集中连接所有模块的全局容器。这种实现的主要优势是可以提升解耦，尤其是在一些依赖状态实例的模块上。使用依赖注入，每一个依赖将不会被硬编码进模块，这个依赖来自外部。这意味我们可以配置模块使用任意依赖，因此就可以在不同的上下文中重用。

为了演示这个模式，我们将使用依赖注入模式重构我们先前的授权服务。

#### 使用 DI 重构授权服务

通过 DI 来重构我们的模块，这其中有一个非常简单的窍门：我们将创建一个工厂，它将一组依赖项作为参数而不是将依赖项硬编码到有状态实例。

以 lib/db.js 模块开始：


````JavaScript
// lib/db.js
const level = require('level');
const sublevel = require('level-sublevel');

module.exports = dbName => {
  return sublevel(
    level(dbName, {valueEncoding: 'json'})
  );
};

````

首先我们通过把 db 模块转换成一个工厂函数为开端。现在我们可以创建任意数量的数据库了；这意味着整个模块将是可重用和无状态的了。

然后构建 lib/authService.js：

````JavaScript
// lib/authService.js

const jwt = require('jwt-simple');
const bcrypt = require('bcrypt');

module.exports = (db, tokenSecret) => {
  const users = db.sublevel('users');
  const authService = {};

  authService.login = (username, password, callback) => {
    //...same as in the previous version
  };

  authService.checkToken = (token, callback) => {
    //...same as in the previous version
  };

  return authService;
};

````

现在， authService 模块也是无状态的了；它不再导出特定的实例了，仅仅导出一个工厂函数。最重要的细节在于我们使得 db 依赖注入作为这个工厂函数的参数，移除了前面的硬编码依赖。这个简单的变化使得我们可以把它连接到任意数据库实例来创建新的 authService 模块。

然后是 lib/authController.js 模块：

````JavaScript
// lib/authController.js
module.exports = (authService) => {
  const authController = {};

  authController.login = (req, res, next) => {
    //...same as in the previous version
  };

  authController.checkToken = (req, res, next) => {
    //...same as in the previous version
  };
  return authController;
};
````

authController 模块没有任何硬编码的依赖。

剩下的就是 app.js 了：


````JavaScript
// app.js

// ...
const dbFactory = require('./lib/db');                        //[1]
const authServiceFactory = require('./lib/authService');
const authControllerFactory = require('./lib/authController');

const db = dbFactory('example-db');                           //[2]
const authService = authServiceFactory(db, 'SHHH!');
const authController = authControllerFactory(authService);

app.post('/login', authController.login);                     //[3]
app.get('/checkToken', authController.checkToken);
// ...


````

1. 首先导入我们的 services 工厂；这时它们还是无状态对象。
1. 然后，通过导入的依赖实例化每一个 service。在这个阶段所有的模块被创建和连接。
1. 最后，我们注册 authController 模块的路由。

#### 不同类型的 DI

我们上面实现的工厂注入是 DI 的一种，这里还有一些其它需要提及的事情：

* 构造器注入：在这种类型的 DI 内，依赖在它被创建的时候传入构造器；可能是这样的：

````JavaScript
const service = new Service(dependencyA, dependencyB)
````

* 属性注入：在这种类型的 DI 内，依赖绑定到一个对象上：

````JavaScript
const service = new Service()
service.dependencyA = anInstanceOfDependencyA
````

属性注入意味着对象在一个不一致的状态下被创建，因为它不连接自己的依赖，所以它的健壮性不高。但是在依赖间出现循环时可能很有用。例如，如果我们有两个组件 A 和 B ，这两个组件都使用工厂或着构造器注入，而且它们互相依赖。我们不能实例化它们其中任意一个，因为它们其中的一个已经被实例化创建。例子如下：

````JavaScript
function Afactory(b) {
  return {
    foo: function() {
      b.say();
    },
    what: function() {
      return 'Hello!';
    }
  }
}

function Bfactory(a) {
  return {
    a: a,
    say: function() {
      console.log('I say: ' + a.what);
    }
  }
}

````

我们只能通过使用属性注入来解决这种僵局。例如，我们可以先创建 B 的实例，然后再实例化 A：

````JavaScript
const b = Bfactory(null)
const a = Afactory(b)
a.b = b
````

#### DI 的优缺点

在使用 DI 实现的授权服务中，我们从依赖实例入手解耦各个模块。结果是我们可以单独重用我们的各个模块了。测试也方便多了。

值得一提的是，我们把依赖的职责从底层切换到的顶层。高阶组件的重用性不如低阶组件，因为随着我们所在的层级越高，每个组件会变得更加具体。

从这个假定出发，我们可以明白常规来说高阶组件拥有被反转的低阶依赖，所以低阶组件只依赖一个接口，因为依赖的定义和实现的所有权在高阶组件上。在我们的授权服务中，所有的依赖被一个顶级组件所实例化并连接起来，app 模块就是哪个顶级组件，这个顶级组件重用性不高，耦合性强。

然而我们需要为解藕和可重用性方面的优势付出代价。一般来说，我们无法在编码时解决依赖关系，这使得理解系统各个组件之间的关系变得更加困难。而且，我们在 app 模块内实例化所有依赖时需要按照特定的顺序；我们不得不手动为整个应用构建依赖树。所以随着应用规模变大这将变得难以管理。

一个可能的解决方案是在多个组件间分割依赖的所有权，而不是把它们集中在一个地方。这样可以大幅度降低依赖管理的复杂度，因为每个组件都拥有其特定的依赖。当然，我们也可以在必要的时候才使用 DI。

我们将在下面的章节中看到如何在复杂架构中简化模块连接的解决方案，这种方案将使用一个 DI 容器来处理所有的依赖职责。

使用 DI 增加了负责度和冗余性，但是依然有很多理由来使用它。我们有责任选择正确的方法，取决于我们想要获得的简单性和可重用性之间的平衡。

### 服务定位器

在前一章节我们学习了 DI 如何通过获取可重用和解藕的模块改变我们连接依赖的方式。另一个具有类似目的的模式就是服务定位器。它的核心原则就是具有一个在模块任何时候需要导入依赖的调解人并集中管理系统间的组件。我们的想法是向服务定位器询问依赖项而不是硬编码依赖。

重要的是要理解通过使用服务定位器，我们引入了对它的依赖，因此我们将它连接到模块的方式决定了它们的耦合程度即可重用性。在 Node.js 中我们可以通过它们连接系统组件的方式来分出三种类型的服务定位器：

* 在服务定位器上硬编码
* 注入服务定位器
* 全局服务定位器

第一个对解藕最不利，因为它使用 require 包含了服务定位器直接的引用。在 Node.js 中，这被称为反模式，因为它引入了与组件的紧密耦合，据称可以提供更好的解藕。在这种情况下，服务定位器显然不会在可重用性方面提供任何价值，只会增加另一层次的间接性和复杂性。

另一方面，一个注入服务定位器被一个组件通过 DI 引用。这样就可以很方便的一次注入整个依赖了。

第三种被提及的服务定位器直接来自全局作用域。它有和硬编码服务定位器一样缺点，但是因为是全局的，它是个真正的单例因此也可以简单的在包之间共享实例。我们可以在接下来的例子中看到，但一般不使用全局服务定位器。

#### 使用服务定位器重构授权服务

我们将用注入的服务定位器来转换授权服务。第一步要实现服务定位器本身：

````JavaScript
//lib/serviceLocator.js

module.exports = function() {
  const dependencies = {};
  const factories = {};
  const serviceLocator = {};

  serviceLocator.factory = (name, factory) => {     //[1]
    factories[name] = factory;
  };

  serviceLocator.register = (name, instance) => {   //[2]
    dependencies[name] = instance;
  };

  serviceLocator.get = (name) => {                  //[3]
    if(!dependencies[name]) {
      const factory = factories[name];
      dependencies[name] = factory && factory(serviceLocator);
      if(!dependencies[name]) {
        throw new Error('Cannot find module: ' + name);
      }
    }
    return dependencies[name];
  };

  return serviceLocator;
}
````

我们的 serviceLocator 模块是一个返回有三个方法对象的工厂：

* factory 将组件名称与工厂关联。
* register 将组件名称直接与实例关联。
* get 通过组件名称取回组件。如果一个实例已经可用，就简单返回它；相反，它会尝试调用注册的工厂来获取一个新实例。知道通过注入服务定位器当前的模块实例调用模块工厂很重要。这是模式的核心机制，允许我们的系统的依赖图自动和按需构建。

````JavaScript
//lib/db.js
const level = require('level');
const sublevel = require('level-sublevel');

module.exports = (serviceLocator) => {
  const dbName = serviceLocator.get('dbName');

  return sublevel(
    level(dbName, {valueEncoding: 'json'})
  );
}

````

db 模块使用服务定位器来接收输入并返回数据库的名称来实例化。一个值得提及的点是，服务定位器不仅可以返回组件实例而且也可以提供一个带参数的配置项。

````JavaScript
//lib/authService.js

// ...
module.exports = (serviceLocator) => {
  const db = serviceLocator.get('db');
  const tokenSecret = “serviceLocator.get('tokenSecret');

  const users = db.sublevel('users');
  const authService = {};

  authService.login = (username, password, callback) => {
    //...same as in the previous version
  }

  authService.checkToken = (token, callback) => {
    //...same as in the previous version
  }

  return authService;
};

````

authService 模块是一个以服务定位器作为输入的工厂。db 句柄和 tokenSecret 两个依赖通过 get 方法被返回。


````JavaScript
//lib/authController.js

module.exports = (serviceLocator) => {
  const authService = serviceLocator.get('authService');
  const authController = {};

  authController.login = (req, res, next) => {
    //...same as in the previous version
  };

  authController.checkToken = (req, res, next) => {
    //...same as in the previous version
  };

  return authController;
}

// app.js

//...
const svcLoc = require('./lib/serviceLocator')();      //[1]

svcLoc.register('dbName', 'example-db');               //[2]
svcLoc.register('tokenSecret', 'SHHH!');
svcLoc.factory('db', require('./lib/db'));
svcLoc.factory('authService', require('./lib/authService'));
svcLoc.factory('authController', require('./lib/authController'));

const authController = svcLoc.get('authController');   //[3]
app.post('/login', authController.login);
app.all('/checkToken', authController.checkToken);
// ...

````

1. 我们通过调用工厂来实例化一个新的服务定位器
1. 我们针对服务定位器注册配置参数和模块工厂。此刻，所有的依赖还未被实例化；我们只是注册了它们的工厂。
1. 我们从服务定位器导入 authController；这会触发整个应用依赖树实例化。当我们实例化 authController 组件时，服务定位器通过注入自己来调用相关的工厂。然后 authController 工厂将尝试导入 authService 模块，这会返回 db 模块的实例。

我们可以看到服务定位器惰性特性；每一个实例只有在需要时才会创建。另一个重要的实现是：每一个依赖不需要提前手动连接。优点是我们不必提前知道实例化和连接模块的顺序 -- 它们全部自动按需发生。相比简单的 DI 模式这样更加方便了。

#### 服务定位器的优缺点

服务定位器和依赖注入有很多相似点：它们都将依赖所有权转移到组件外部的实体。但我们连接服务定位器的方式决定了我们整个架构的灵活性。我们选择一个注入的服务定位器而不是硬编码或全局服务定位器来实现我们的示例并不是偶然的。这最后两个变种几乎抵消了这种模式的优势。实际上我们通过在服务定位器实例里面连接它们，而不是通过 require 导入依赖来连接组件。虽然硬编码的服务定位器更方便但就可重用性来说优势不大。

当然，就像 DI 一样，使用一个服务定位器很难确定两个组件间的关系，因为它们在运行时才确定关系。还有，也很难确切知道哪个组件是被导入的。通过声明构造器参数或工厂中的依赖，在 DI 中表达会更清晰一点。但在服务定位器内需要代码审查器或一个确切的语句来解释组件的具体依赖。

最后需要注意的是对于 DI 容器相比，服务定位器是一个不那么正确的实现，因为它共享了和服务注册相同的角色；但是，这里也有很大区别。在服务定位器中，每个组件知道自己明确的依赖；在 DI 容器中，组件对自己的容器一无所知。

两种实现的不同点主要有以下几点：

* 可重用性：依赖于服务定位器的组件重用性更低因为它依赖于服务定位器。
* 可读性：服务定位器混淆了依赖。

就可重用性来说，我们可以说服务定位器模式居于硬编码依赖和 DI 之间。就便捷性和简易性来说，它绝对比手动依赖好，因为我们不必手动控制整个依赖树。

基于这些假定，就组件可重用性和便捷性来说，一个 DI 容器绝对提供了最好的平衡。我们这就来看看。

### 依赖注入容器

将服务定位器转换为依赖注入（DI）容器的步骤并不大，但正如我们已经提到的，它在解耦方面产生了巨大的差异。这个模式下，每个模块不必依赖于服务定位器，它可以简单地表达其对依赖性的需求，DI 容器将无缝地完成剩下的工作。对于这个容器最大的进步就是每个模块可以脱离模块进行重用。

#### 为一个 DI 容器声明一系列依赖

一个 DI 容器本质上是一个添加的新特性的服务定位器：在模块实例化前它可以识别模块的依赖。因此，一个模块必须以某种方法来声明它的依赖。

首先，最流行的方式是注入一系列基于在工厂中使用的参数和构造器的依赖：



````JavaScript
//lib/authService.js

module.exports = (db, tokenSecret) => {
  //...
}
````

正如我们定义的，这个模块将通过我们的 DI 容器传入的名称来实例化。但是为了从函数内读取到参数，我们有必要使用一点点技巧。在 JavaScript 中，我们可以通过序列化一个函数返回它的源码；使用 toString 方法就可以办到。

这种方法的最大问题在于它不能很好地用于缩小，这种做法在客户端JavaScript中广泛使用，其中包括应用特定的代码转换以将源代码的大小减小到最小。许多缩小应用了一种称为名称修改的技术，它基本上重命名了一切局部变量以减少其长度，通常减少为单个字符。一个坏消息是函数参数是本地变量而且经常被其处理过程所影响，这会导致我们描述的声明依赖的机制失效。即使在服务端缩小不是必须的，但是考虑在浏览器内共享 Node.js 模块的代码也很重要。

幸运的是，一个 DI 容器可能使用其它的的技术来获取注入的依赖。这些技术被列在下面：

* 我们可以使用一个附在工厂函数上的特别属性，例如一个确切列出注入依赖的数组：

````JavaScript

module.exports = (a, b) => {}

module.exports._inject = ['db', 'another/dependency']
````

* 我们可以指定一个依赖名称的数组：

````JavaScript
module.exports = ['db', 'another/dependency'， (a, b) => {}]
````

* 我们可以使用一个注释标记添加到每个函数内：

````JavaScript
module.exports = function(a /*db*/, b /*another/dependency*/) {}
````

这些技术都十分主观，所以我们使用其中最流行的实现：使用函数参数作为依赖名称。

#### 通过 DI 容器来重构授权服务
