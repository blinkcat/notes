# express

版本[4.17.1](https://github.com/expressjs/express/tree/4.17.1)

## 整体结构

作为一个 web application framework， express 已经有超过十年的历史了，是 node 世界中最优秀的 framework 之一。其自身集成了 web 开发中必要的功能，易上手，且可以通过添加 middleware 扩展功能。

在结构上可以分为：

1. Application
2. Router
3. Route
4. Layer
5. Middleware

其中 middleware 贯穿始终，是 express 最核心的部分。

## Application

Application 代表整个应用层。

```js
const express = require("express");
const app = expess();
```

当我们 require express 的时候，引入的这个 express 其实是个叫做 `createApplication` 函数。

```js
/**
 * Expose `createApplication()`.
 */
exports = module.exports = createApplication;
/**
 * Create an express application.
 *
 * @return {Function}
 * @api public
 */
function createApplication() {
  var app = function (req, res, next) {
    app.handle(req, res, next);
  };
  mixin(app, EventEmitter.prototype, false);
  // ...
  return app;
}
```

这个函数执行后，返回了另一个函数 app，app 函数本身继承了 `EventEmitter`，并且自身带有一些常用的方法，比如 `app.use`，`app.listen`，`app.(get|post|put|...)`等。

接着，我们会通过 `app.use` 引入中间件，`app.verb` 定义路由，最后调用 `app.listen` 来监听端口。在调用 `app.listen` 的时候，真正的 server 才会被创建。

```js
app.listen = function listen() {
  var server = http.createServer(this);
  return server.listen.apply(server, arguments);
};
```

`http.createServer` 中传入了 this，也就是上面的 app 函数。每当有请求进来时，app 函数就会执行，传入和这次请求相关的 req 和 res 。这里还有一个 next 参数，是为了实现 Application 嵌套。app 函数内部会执行 `app.handle`，这里是执行所有 middleware 和 route 的入口。

```js
app.handle = function handle(req, res, callback) {
  var router = this._router;
  // ...
  router.handle(req, res, done);
};
```

`app.handle` 内部把任务指给了 `router.handle`。这里的 router 是 Router 的实例，在 app 内部维护。

实际上 `app.use`，`app.route`，`app.param` 等，内部都调用 router 上的方法来执行具体任务。

```js
app.lazyrouter = function lazyrouter() {
  if (!this._router) {
    this._router = new Router({
      caseSensitive: this.enabled("case sensitive routing"),
      strict: this.enabled("strict routing"),
    });

    this._router.use(query(this.get("query parser fn")));
    this._router.use(middleware.init(this));
  }
};
```

可以看到，router 被创建后，还挂载了两个 middleware，一个用来解析 querystring，一个用来扩展原生的 req，res，在这两个对象上添加一些方法和属性。

继续看 `app.use`，和 `app.verb`。

`app.use` 用来添加全局范围的 middleware，这里 middleware 被分为两类。一类是普通的 middleware，一类是 express app。

middleware 是拥有

```js
function (req, res, next) {
	// ...
}
```

这样签名的函数。前面介绍的 createApplication 函数中 app 函数的定义满足这个签名，所以也可以当做是一个 middleware。

```js
app.use = function use(fn) {
  // ...
  var fns = flatten(slice.call(arguments, offset));
  // setup router
  this.lazyrouter();
  var router = this._router;
  fns.forEach(function (fn) {
    // non-express app
    if (!fn || !fn.handle || !fn.set) {
      return router.use(path, fn);
    }
    // ...
    // restore .app property on req and res
    router.use(path, function mounted_app(req, res, next) {
      var orig = req.app;
      fn.handle(req, res, function (err) {
        setPrototypeOf(req, orig.request);
        setPrototypeOf(res, orig.response);
        next(err);
      });
    });

    // mounted an app
    fn.emit("mount", this);
  }, this);

  return this;
};
```

对这种特殊的 middleware，app.use 中用了一个 `mounted_app` 函数包了起来，然后再调用 `router.use`。为什么需要用一个函数包起来呢？仔细观察这个函数，它的主要逻辑都在传递给下个 app 的 next 参数中。也就是下面这个函数。

```js
function (err) {
	setPrototypeOf(req, orig.request);
    setPrototypeOf(res, orig.response);
    next(err);
}
```

这个函数主要的工作就是复原 req，res 对象的原型，因为在进入子 app 中的时候，子 app 会把 req，res 的原型分别指向自己的 request，response 对象。在执行完所有子 app 中定义的 middleware 后，会调用上面这个函数，退到父 app 中来，重新指向父 app 的 request，response。

最后有一句 `fn.emit("mount", this)`，触发了下面的函数

```js
this.on("mount", function onmount(parent) {
  // inherit trust proxy
  if (
    this.settings[trustProxyDefaultSymbol] === true &&
    typeof parent.settings["trust proxy fn"] === "function"
  ) {
    delete this.settings["trust proxy"];
    delete this.settings["trust proxy fn"];
  }

  // inherit protos
  setPrototypeOf(this.request, parent.request);
  setPrototypeOf(this.response, parent.response);
  setPrototypeOf(this.engines, parent.engines);
  setPrototypeOf(this.settings, parent.settings);
});
```

很简单，利用了原型链来实现子 app 继承父 app 的一些设置。

接下来看看 app 中另一类比较常用的方法，`app.(get|post|put|...)`。

```js
methods.forEach(function (method) {
  app[method] = function (path) {
    if (method === "get" && arguments.length === 1) {
      // app.get(setting)
      return this.set(path);
    }

    this.lazyrouter();

    var route = this._router.route(path);
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});
```

`methods`是一个数组，里面是一些 http verb，如 get、post、put、head 之类的。每个 verb，app 中都有一个对应的方法。在内部通过`this._router.route`，创建一个`route`来添加定义的路由。

## Router

Router 可以算是整个框架的核心，是实际处理 middleware 和 route 的部分。先看 Router 函数

```js
function router(req, res, next) {
  router.handle(req, res, next);
}
```

是不是和上面 app 的定义一样，也是一个 middleware。这就让 Router 可以挂载到 app 上。

先来看`router.use`，这个函数用来在某一组路由上挂载 middleware。在`app.use`内部也是调用了它。

```js
proto.use = function use(fn) {
  // ...
  var callbacks = flatten(slice.call(arguments, offset));
  // ...
  for (var i = 0; i < callbacks.length; i++) {
    var fn = callbacks[i];
    // ...
    var layer = new Layer(
      path,
      {
        sensitive: this.caseSensitive,
        strict: false,
        end: false,
      },
      fn
    );
    layer.route = undefined;
    this.stack.push(layer);
  }
  return this;
};
```

函数内部创建了一个 Layer 对象来表示这个 middleware，并且压入`this.stack`数组中。

再来看看`router.verb`

```js
// create Router#VERB functions
methods.concat("all").forEach(function (method) {
  proto[method] = function (path) {
    var route = this.route(path);
    route[method].apply(route, slice.call(arguments, 1));
    return this;
  };
});
```

内部直接调用了`this.route`，

```js
proto.route = function route(path) {
  var route = new Route(path);

  var layer = new Layer(
    path,
    {
      sensitive: this.caseSensitive,
      strict: this.strict,
      end: true,
    },
    route.dispatch.bind(route)
  );

  layer.route = route;

  this.stack.push(layer);
  return route;
};
```

再看`this.route`方法，内部创建了一个 Route 对象，一个 Layer 对象，然后也将 Layer 对象压入`this.stack`。如果和上面的`route.use`对比一下，可以发现有两处明显的不同。一处是在构造 Layer 对象的时候，上面只是简单的传入 fn，而下面传入的是`route.dispatch.bind(route)`，后面会解释。还有一处是下面会设置`layer.route = route`，来表示这一个 Layer 对象代表的是路由。

Router 将代表 middleware 和 route 的 Layer 对象都压入一个 stack 中，然后按顺序执行。所以这里应该叫 queue 更准确点。

为了使下面的代码更容易理解，先看看 Layer 内部是如何实现的。Layer 对象主要负责三件事，

一是判断当前 path 是否和 middleware，route 定义的路由规则是否匹配，如果匹配，还会解析 param，然后保存起来。

```js
Layer.prototype.match = function match(path) {
  var match;
  if (path != null) {
    // ...
    // match the path
    match = this.regexp.exec(path);
  }
  if (!match) {
    this.params = undefined;
    this.path = undefined;
    return false;
  }
  // store values
  this.params = {};
  this.path = match[0];
  var keys = this.keys;
  var params = this.params;
  for (var i = 1; i < match.length; i++) {
    var key = keys[i - 1];
    var prop = key.name;
    var val = decode_param(match[i]);

    if (val !== undefined || !hasOwnProperty.call(params, prop)) {
      params[prop] = val;
    }
  }
  return true;
};
```

二是执行正常运行的 middleware 和 route

```js
Layer.prototype.handle_request = function handle(req, res, next) {
  var fn = this.handle;

  if (fn.length > 3) {
    // not a standard request handler
    return next();
  }

  try {
    fn(req, res, next);
  } catch (err) {
    next(err);
  }
};
```

三是处理 middleware 和 route 出错的情况

```js
Layer.prototype.handle_error = function handle_error(error, req, res, next) {
  var fn = this.handle;

  if (fn.length !== 4) {
    // not a standard error handler
    return next(error);
  }

  try {
    fn(error, req, res, next);
  } catch (err) {
    next(err);
  }
};
```

了解完 Layer 以后，我们再来看看`router.handle`，这块应该是 express 中最难理解的地方了。为了简化理解，我们只从 middleware 执行这个方向来解读这块代码。

```js
/**
 * Dispatch a req, res into the router.
 * @private
 */

proto.handle = function handle(req, res, out) {
  var self = this;
  var idx = 0;
  // ...

  // middleware and routes
  var stack = self.stack;

  // manage inter-router variables
  var parentParams = req.params;
  var parentUrl = req.baseUrl || "";
  var done = restore(out, req, "baseUrl", "next", "params");

  // ...

  next();

  function next(err) {
    var layerError = err === "route" ? null : err;
    // ...
    // signal to exit router
    if (layerError === "router") {
      setImmediate(done, null);
      return;
    }

    // no more matching layers
    if (idx >= stack.length) {
      setImmediate(done, layerError);
      return;
    }

    // get pathname of request
    var path = getPathname(req);

    if (path == null) {
      return done(layerError);
    }

    // find next matching layer
    var layer;
    var match;
    var route;

    while (match !== true && idx < stack.length) {
      layer = stack[idx++];
      match = matchLayer(layer, path);
      route = layer.route;

      if (typeof match !== "boolean") {
        // hold on to layerError
        layerError = layerError || match;
      }

      if (match !== true) {
        continue;
      }

      if (!route) {
        // process non-route handlers normally
        continue;
      }

      if (layerError) {
        // routes do not match with a pending error
        match = false;
        continue;
      }

      var method = req.method;
      var has_method = route._handles_method(method);

      // build up automatic options response
      if (!has_method && method === "OPTIONS") {
        appendMethods(options, route._options());
      }

      // don't even bother matching route
      if (!has_method && method !== "HEAD") {
        match = false;
        continue;
      }
    }

    // no match
    if (match !== true) {
      return done(layerError);
    }

    // store route for dispatch on change
    if (route) {
      req.route = route;
    }

    // Capture one-time layer values
    req.params = self.mergeParams
      ? mergeParams(layer.params, parentParams)
      : layer.params;
    var layerPath = layer.path;

    // this should be done for the layer
    self.process_params(layer, paramcalled, req, res, function (err) {
      if (err) {
        return next(layerError || err);
      }

      if (route) {
        return layer.handle_request(req, res, next);
      }

      trim_prefix(layer, layerError, layerPath, path);
    });
  }

  function trim_prefix(layer, layerError, layerPath, path) {
    // ...
    if (layerError) {
      layer.handle_error(layerError, req, res, next);
    } else {
      layer.handle_request(req, res, next);
    }
  }
};
```

删减过的代码，还是很长。

首先先看这个函数最后一个入参 out，这个入参表示一个执行链条的出口。在函数内部又被包装成

`done = restore(out, req, "baseUrl", "next", "params")`

接着看`next`函数，这个函数会递归执行，依次处理 `this.stack`中的 Layer 对象。

如果入参 err 不为 null，或者已经处理完全部的 Layer 对象，执行 done 退出。

```js
// signal to exit router
if (layerError === "router") {
  setImmediate(done, null);
  return;
}

// no more matching layers
if (idx >= stack.length) {
  setImmediate(done, layerError);
  return;
}
```

然后进入一个 while 循环，来找出 path 匹配的 Layer。之前说 Layer 有两种，一种是 middleware，一种是 route。如果是 route，还要判断请求的 method 是否匹配。接着往下看，跳过其他的代码，直接看`self.process_params`最后一个入参。这块在执行时，会有三种情况

当 err 非空时，继续递归，把错误往后传递。

```js
if (err) {
  return next(layerError || err);
}
```

如果当前是一个 route，处理这个 Layer 对象

```js
if (route) {
  return layer.handle_request(req, res, next);
}
```

最后如果是 middleware，根据是否有 layerError 来执行

```js
if (layerError) {
  layer.handle_error(layerError, req, res, next);
} else {
  layer.handle_request(req, res, next);
}
```

要注意的是，next 要么是直接被调用，要么是传入 Layer 的方法中。只有当 next 被调用，这条执行链才会往下走。剩下的 Layer 对象才会被处理。

还有一个难理解的点是执行过程中的错误处理，当发生错误的时候，会通过 next 函数往后传。如果后面有 middleware 可以处理，那么可能会恢复正常执行过程。如果没有，那么最终会到一个最外层的错误处理。

```js
// final handler
var done =
  callback ||
  finalhandler(req, res, {
    env: this.get("env"),
    onerror: logerror.bind(this),
  });
```

在错误未得到处理时，path 匹配 route 会被跳过。

## Route

Route 这部分的代码稍微简单点

```js
function Route(path) {
  this.path = path;
  this.stack = [];

  debug("new %o", path);

  // route handlers for various http methods
  this.methods = {};
}
```

Route 代表的是某个具体 path 的路由，可以有多个 method，同一个 method 有多个回调。Route 内部也维护了一个`this.stack`。

先看下`route.verb`的实现

```js
methods.forEach(function (method) {
  Route.prototype[method] = function () {
    var handles = flatten(slice.call(arguments));

    for (var i = 0; i < handles.length; i++) {
      var handle = handles[i];
      // ...
      var layer = Layer("/", {}, handle);
      layer.method = method;

      this.methods[method] = true;
      this.stack.push(layer);
    }

    return this;
  };
});
```

也是创建了一个 Layer 对象，压入自己的`this.stack`中。那么这个 stack 是如何和 Router 中的 stack 连接起来的呢？

还记得之前提到过的 `router.route`方法吗

```js
proto.route = function route(path) {
  var route = new Route(path);

  var layer = new Layer(
    path,
    {
      sensitive: this.caseSensitive,
      strict: this.strict,
      end: true,
    },
    route.dispatch.bind(route)
  );

  layer.route = route;

  this.stack.push(layer);
  return route;
};
```

它在创建 Layer 对象的时候，传入了`route.dispatch.bind(route)`。当外层递归处理 Layer 对象时，遇到匹配的 Route，就会执行传入的这个函数。关键就在这里，这个函数连接了两个 stack。

```js
Route.prototype.dispatch = function dispatch(req, res, done) {
  var idx = 0;
  var stack = this.stack;
  if (stack.length === 0) {
    return done();
  }
  var method = req.method.toLowerCase();
  // ...
  next();
  function next(err) {
    // signal to exit route
    if (err && err === "route") {
      return done();
    }

    // signal to exit router
    if (err && err === "router") {
      return done(err);
    }

    var layer = stack[idx++];
    if (!layer) {
      return done(err);
    }

    if (layer.method && layer.method !== method) {
      return next(err);
    }

    if (err) {
      layer.handle_error(err, req, res, next);
    } else {
      layer.handle_request(req, res, next);
    }
  }
};
```

`dispatch`的实现简单明了，最后一个入参 done 是上层 Router 中的 next 函数，也是 `dispatch`内部递归的出口。里面的这个 next 方法就很好理解了，递归处理完内部的 Layer 对象。

## conclusion

总体上看下来，express 的代码还是比较精简的，实现 middleware 的方式巧妙。

## 2022-03-15 重读

版本 [4.17.3](https://github.com/expressjs/express/tree/4.17.3)

距离上次阅读已经有一年多的时间了，大部分内容都已经忘记了。这次重读的起因是在查看[webpack-dev-server](./webpack-dev-server.md)的源码时，发觉 express 的中间件机制的扩展性非常强，这让 express 在这么多年中依然能够在众多竞争者中保留一块领地。

一个框架总是可以从中发现某些固定角色，在一个流程中，各个角色各司其职，有序的发生交互，推动这个流程进行下去。

在 express 中，我们有 app、router、route、request、response 以及各种 middleware(中间件)。整个框架的流程就是监听端口，接收一个`http.IncomingMessage`对象和一个`http.ServerResponse`对象，简称为 req 和 res。req 中携带了请求信息，而 res 则是用来向请求方回复消息的。在 express 中，这两个对象会经过一个个符合条件的 middleware 处理，最终由某个路由或是某个中间件来回复请求，结束整个流程。这样的处理方式符合职责链模式。

在接收到请求时，req 和 res 都是 node 原生对象。express 通过原型链给二者添加了很多实用的方法和属性。

```js
// lib/middleware/init.js
var setPrototypeOf = require("setprototypeof");
// ...
setPrototypeOf(req, app.request);
setPrototypeOf(res, app.response);
```

在我们创建 express 应用的时候，会调用一个工厂方法，这个方法返回一个 app 函数，这个函数符合中间件的签名。而它的各种方法都是通过[复制](https://github.com/component/merge-descriptors)的方式获得。这里和上面的 req、res 的做法不同，个人感觉应该是设计上的马虎。

```js
// lib/express.js
var EventEmitter = require("events").EventEmitter;
var mixin = require("merge-descriptors");
// ...
function createApplication() {
  var app = function (req, res, next) {
    app.handle(req, res, next);
  };
  mixin(app, EventEmitter.prototype, false);
  mixin(app, proto, false);
  // ...
  app.init();
  return app;
}
```

再看一下路由 router 的创建方式，也是如出一辙。不过这次添加方法的方式又变成了原型链。

```js
// lib/router/index.js
var setPrototypeOf = require("setprototypeof");
// ...
var proto = (module.exports = function (options) {
  var opts = options || {};

  function router(req, res, next) {
    router.handle(req, res, next);
  }

  // mixin Router class functions
  setPrototypeOf(router, proto);

  router.params = {};
  router._params = [];
  router.caseSensitive = opts.caseSensitive;
  router.mergeParams = opts.mergeParams;
  router.strict = opts.strict;
  router.stack = [];

  return router;
});
```

两者的处理逻辑都交由自身的`handle`方法，但是在 app 的内部，它的 handle 方法也是交由隐式创建的 router 对象处理的。

```js
app.handle = function handle(req, res, callback) {
  var router = this._router;
  // ...
  router.handle(req, res, done);
};
```

之前谈到中间件的签名，有两种，第二种的入参多了一个 error，专门用来处理错误。最后一个 next 是触发下个路由或者中间件的函数。

```js
(req, res, next) => {
  // ...
};

(error, req, res, next) => {
  // ...
};
```

这个函数在 router 的 handle 方法中定义，在这里我们执行所有匹配的中间件，包括路由。

```js
// lib/router/index.js
proto.handle = function handle(req, res, out) {
  // ...
  next();
  // ...
  function next(err) {
    // ...
    if (err) {
      return next(layerError || err);
    }

    if (route) {
      return layer.handle_request(req, res, next);
    }

    trim_prefix(layer, layerError, layerPath, path);
    // ...
  }

  function trim_prefix(layer, layerError, layerPath, path) {
    // ...
    if (layerError) {
      layer.handle_error(layerError, req, res, next);
    } else {
      layer.handle_request(req, res, next);
    }
  }
};
```

这个函数中信息比较丰富，首先我们的中间件和路由处理函数并不是直接保存在 router 中，而是包装成`Layer`对象，这个对象提供了方法来判断当前的请求 url 和自身代表的路由或者中间件是否匹配。因为我们在定义这二者的时候，都会和某个 path 对应起来，默认的 path 是`'/'`。

其次是这个`out`入参，它代表这处理流程的出口。router 对象中所有的中间件和路由都存在一个`stack`数组中。而每个特定路由对应的处理函数由一个`Route`对象负责。一个路由可以有不同的处理方法，例如`GET`、`POST`、`PUT`等，每个处理方法又可以定义多个函数。对此，Route 对象也是将这些函数封装成`Layer`对象，存在自己的`stack`数组中。

同样的它的处理方式和 router 中的一样。注意这里的`done`入参，

```js
// lib/router/route.js
Route.prototype.dispatch = function dispatch(req, res, done) {
  var idx = 0;
  var stack = this.stack;
  if (stack.length === 0) {
    return done();
  }
  // ...
  next();
  function next(err) {
    // signal to exit route
    if (err && err === "route") {
      return done();
    }

    // signal to exit router
    if (err && err === "router") {
      return done(err);
    }

    var layer = stack[idx++];
    if (!layer) {
      return done(err);
    }

    if (layer.method && layer.method !== method) {
      return next(err);
    }

    if (err) {
      layer.handle_error(err, req, res, next);
    } else {
      layer.handle_request(req, res, next);
    }
  }
};
```

这个 done 函数就是上面提到的 router.handle 中提供的 next 函数。表示执行完对应路由的处理流程后，再回到上层的 router 对象的处理流程中。我们从路由的定义中，可以证明这点。在创建 Layer 对象的过程中，传入的执行函数是`route.dispatch.bind(route)`。

```js
// lib/router/index.js
proto.route = function route(path) {
  var route = new Route(path);

  var layer = new Layer(
    path,
    {
      sensitive: this.caseSensitive,
      strict: this.strict,
      end: true,
    },
    route.dispatch.bind(route)
  );

  layer.route = route;

  this.stack.push(layer);
  return route;
};
```

所以整个中间件和路由的处理流程大致如下，

```shell
app--router--
            |
            middileware1
            |
            middileware2
            |
            route--
                  |
                  fn1
                  |
                  fn2
                  |
            | --  done
            |
            middileware3
            |
            middileware4
```

最后注意的一点是在整个流程中的错误处理。在 router.handle 已经 route.dispatch 中的 next 函数都有个`err`入参，这个入参是相邻的两个中间件直接沟通的方式之一。目前可以传递错误对象、`'route'`以及`'router'`。如果是错误对象，那么会跳过所有的 3 个参数的中间件或者路由，直到遇到 4 个入参的中间件。
