# webpack-dev-server

版本 [v4.0.0-rc.0](https://github.com/webpack/webpack-dev-server/tree/v4.0.0-rc.0)

配合`webpack`，生成一个小型的服务器，辅助 web 开发，并且支持 HMR。

## Usage

1. 通过 `webpack` 配置中的 [`devServer`](https://webpack.js.org/configuration/dev-server/)属性来配置，然后执行`npx webpack serve`。
2. 通过`Node Api`,

```js
import DevServer from "webpack-dev-server";
import webpack from 'webpack';

const compiler = new webpack({...});
const devServer = new DevServer(compiler, {...});

devServer.listen(3000, err => {...});
```

## How it works

### server 端

`webpack-dev-server`，下面简称`wds`，并非独立完成所有功能，而是通过整合其他功能内聚的中间件，对外提供了一个基于 webpack 的，比较成熟的 web 开发解决方案。

例如

- 在代码编译方面，使用了`webpack-dev-middleware`中间件，提供编译后的静态资源
- 在热更新方面，需要依赖`webpack`提供的`HotModuleReplacementPlugin`插件，以及`webpack/hot`目录下的文件
- 在和浏览器通信方面，直接使用了`websocket`
- ......

在创建`wds`对象时，除了`options`外，还需要传入一个`compiler`对象。这个对象用来获取配置，添加插件，最后传给`webpack-dev-middleware`。接着还会通过`normalizeOptions`函数处理`options`，提供一些默认选项。

然后直接跳到`listen(port, hostname, fn): Server{...}`函数，因为整个应用是通过这个函数来启动的。`port`和`hostname`不必多说，在确定可用的端口后，还会调用`this.initialize()`做初始化，给`wds`加上所有的功能。

```js
  initialize() {
    this.applyDevServerPlugin();

    if (this.options.client && this.options.client.progress) {
      this.setupProgressPlugin();
    }

    this.setupHooks();
    this.setupApp();
    this.setupHostHeaderCheck();
    this.setupDevMiddleware();
    // Should be after `webpack-dev-middleware`, otherwise other middlewares might rewrite response
    this.setupBuiltInRoutes();
    this.setupWatchFiles();
    this.setupFeatures();
    this.createServer();

    killable(this.server);

    if (this.options.setupExitSignals) {
      const signals = ['SIGINT', 'SIGTERM'];

      signals.forEach((signal) => {
        process.on(signal, () => {
          this.close(() => {
            // eslint-disable-next-line no-process-exit
            process.exit();
          });
        });
      });
    }

    // Proxy WebSocket without the initial http request
    // https://github.com/chimurai/http-proxy-middleware#external-websocket-upgrade
    // eslint-disable-next-line func-names
    this.webSocketProxies.forEach(function (webSocketProxy) {
      this.server.on('upgrade', webSocketProxy.upgrade);
    }, this);
  }
```

1. 先把`DevServerPlugin`插件放入当前`webpack`。这个插件干的事情是，
   1. 根据需要，在`webpack entry`入口文件中，插入`socket` 客户端脚本和`webpack`支持热更新的脚本
   2. 通过`webpack.ProvidePlugin`提供`__webpack_dev_server_client__`变量
   3. 如果需要，再加入`webpack.HotModuleReplacementPlugin`植入热更新代码
2. 按需加入`webpack.ProgressPlugin`插件
3. 调用`setupHooks`，监听`compiler`中的`invalid`和`done`两个钩子，并发送对应消息到客户端
4. 调用`setupApp`，创建一个`express` app
5. 调用`setupHostHeaderCheck`，在上面的`express` app 中添加检测`header`属性的中间件
6. 创建`webpack-dev-middleware`中间件，但是没有立即把中间件加入到上面的`express` app 中
7. 调用`setupBuiltInRoutes`创建三个内部路由
8. 调用`setupWatchFiles`，监听一些文件改动，然后发送消息给客户端，请求刷新页面
9. 接着是重头戏，调用`setupFeatures`，添加一堆中间件，依次是：
   1. compression
   2. 调用`this.options.onBeforeSetupMiddleware`，此时可以添加自定义的中间件
   3. 调用`setupHeadersFeature`，添加一些用户自定义的`header`
   4. 将`webpack-dev-middleware`中间件加入到 app 中
   5. 按需添加`proxy`中间件，重复 4
   6. 按需添加`express.static`中间件
   7. 按需添加`connect-history-api-fallback`中间件，重复 4（为啥又要重复 4，因为 5，7 两个中间件比较特殊，可能会改动`req.url`，然后调用下一个中间件，这时候可能还会访问一些`webpack`编译过的资源，比如说脚本，这时候还需要`webpack-dev-middleware`来处理）
   8. ......
10. 创建 http/https 服务器，保存在`this.server`中
11. `killable(this.server)`，可以加速服务器关闭速度
12. 处理接收到`SIGINT`, `SIGTERM`信号，关闭进程
13. `this.webSocketProxies`处理`websocket`代理

`initialize`函数还是做了很多初始化工作的。

紧接着，继续调用`this.createWebSocketServer()`，创建`websocket`服务，每次接收到请求后，向服务器发送消息。

然后按需调用`runBonjour`，调用`logStatus`打印 log，调用回调`fn.call(this.server, error)`。

除此之外，还有几个有用的方法：

```js
  use() {
    // eslint-disable-next-line prefer-spread
    this.app.use.apply(this.app, arguments);
  }
```

用来添加中间件

`close`，用来关闭所有资源，包括服务器，文件监听，`webpack`的`watching`等等。

```js
  invalidate(callback) {
    if (this.middleware) {
      this.middleware.invalidate(callback);
    }
  }
```

用来让本次`webpack`构建失效，并且重新构建

### client 端

`client`端的代码在另外一个文件夹中，单独`build`。

首先会先解析[\_\_resourceQuery](https://webpack.js.org/api/module-variables/#__resourcequery-webpack-specific)

```js
const parsedResourceQuery = parseURL(__resourceQuery);
```

这个`__resourceQuery`是在插入`DevServerPlugin`时，放入`entry`中的。

```js
`${require.resolve("../../client/index.js")}?${webSocketURL}`;
```

`webSocketURL`中包含了`protocol`，`username`，`password`,`hostname`，`port`，`pathname`，以及`logging`。

`client`端会通过这些变量重建一个 websocket url

```js
const socketURL = createSocketURL(parsedResourceQuery);
```

然后创建一个 socket 客户端

```js
socket(socketURL, onSocketMessage);
```

这里有两个重点，先看`onSocketMessage`，它其实是一个 object，包含了客户端处理接收到服务器端特定的`message`后，需要执行的函数。

```js
const onSocketMessage = {
  hot() {
    if (parsedResourceQuery.hot === "false") {
      return;
    }

    options.hot = true;

    log.info("Hot Module Replacement enabled.");
  },
  // ...
  ok() {
    sendMessage("Ok");

    if (options.overlay) {
      hide();
    }

    reloadApp(options, status);
  },
  // ...
  warnings(warnings) {
    log.warn("Warnings while compiling.");

    const strippedWarnings = warnings.map((warning) =>
      stripAnsi(warning.message ? warning.message : warning)
    );

    sendMessage("Warnings", strippedWarnings);

    for (let i = 0; i < strippedWarnings.length; i++) {
      log.warn(strippedWarnings[i]);
    }

    const needShowOverlay =
      typeof options.overlay === "boolean"
        ? options.overlay
        : options.overlay && options.overlay.warnings;

    if (needShowOverlay) {
      show(warnings, "warnings");
    }

    reloadApp(options, status);
  },
};
```

可以看到，在接收到服务器端的消息后，客户端会执行一些函数，例如，在控制台中打印 log、显示或隐藏`overlay`、`reloadApp`(热更新应用或者刷新页面)...

调用`reloadApp`函数时会传入两个变量，它们在之前定义过，

```js
const status = {
  isUnloading: false,
  // TODO Workaround for webpack v4, `__webpack_hash__` is not replaced without HotModuleReplacement
  // eslint-disable-next-line camelcase
  currentHash: typeof __webpack_hash__ !== "undefined" ? __webpack_hash__ : "",
};
// console.log(__webpack_hash__);
const options = {
  hot: false,
  liveReload: false,
  progress: false,
  overlay: false,
};
const parsedResourceQuery = parseURL(__resourceQuery);

if (parsedResourceQuery.logging) {
  options.logging = parsedResourceQuery.logging;
}
```

`reloadApp`会通过`hotEmitter`来做热更新，直接调用`location.reload`来刷新页面。

```js
import hotEmitter from "webpack/hot/emitter.js";

hotEmitter.emit("webpackHotUpdate", status.currentHash);
```

再回到刚才创建`socket`的地方，这里的 socket 实例有两种，websocket 和 sockjs，通过`__webpack_dev_server_client__`进行选择，也就是上面提到的通过`ProvidePlugin`插入的路径。然后就是正常的创建过程。

总得来说，`wds`的大体实现如上。
