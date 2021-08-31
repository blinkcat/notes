# webpack-dev-middleware

[v5.0.0](https://github.com/webpack/webpack-dev-middleware/tree/v5.0.0)

express 风格的中间件，用来编译代码并提供编译完成后的 asset 访问链接。

## Usage

```js
const webpack = require("webpack");
const middleware = require("webpack-dev-middleware");
const compiler = webpack({
  // webpack options
});
const express = require("express");
const app = express();

app.use(
  middleware(compiler, {
    // webpack-dev-middleware options
  })
);

app.listen(3000, () => console.log("Example app listening on port 3000!"));
```

## How it works

从调用方式上看，底层还是依赖于`webpack`的编译能力。只是结合了`web`开发的场景，提供了一些比较有用的功能，例如，默认不会把文件真的写入磁盘，而是放到内存中、当有请求过来时，直到编译成功后才会响应。

来看源码，以下将`webpack-dev-middleware`简称为`wdm`。

首先，代码内部维护了一个上下文

```js
const context = {
  state: false,
  stats: null,
  callbacks: [],
  options,
  compiler,
  watching: null,
};
```

- `state`表示当前编译是否成功
- `stats`是编译成功后的`stats`对象
- `callbacks`中通常存放着处理 http 请求的回调
- `options` 是 wdm 自身的配置
- `compiler`是传入的 webpack compiler 对象
- `watching`是执行 webpack.watch 后的返回对象

接着调用`setupHooks(context)`，绑定`compiler`中的一些关键事件

```js
context.compiler.hooks.watchRun.tap("webpack-dev-middleware", invalid);
context.compiler.hooks.invalid.tap("webpack-dev-middleware", invalid);
(context.compiler.webpack
  ? context.compiler.hooks.afterDone
  : context.compiler.hooks.done
).tap("webpack-dev-middleware", done);
```

> **invalid** Executed when a watching compilation has been invalidated.
>
> **done** Executed when the compilation has completed.

当 invalid 的时候，改变上下文对象中对应的属性。

```js
function invalid() {
  if (context.state) {
    context.logger.log("Compilation starting...");
  }

  // We are now in invalid state
  // eslint-disable-next-line no-param-reassign
  context.state = false;
  // eslint-disable-next-line no-param-reassign, no-undefined
  context.stats = undefined;
}
```

而当 done 的时候，除了更新`context`对象中的属性，还会执行所有的回调。

```js
function done(stats) {
  // We are now on valid state
  // eslint-disable-next-line no-param-reassign
  context.state = true;
  // eslint-disable-next-line no-param-reassign
  context.stats = stats;

  // Do the stuff in nextTick, because bundle may be invalidated if a change happened while compiling
  process.nextTick(() => {
    // ...
    // eslint-disable-next-line no-param-reassign
    context.callbacks = [];

    // Execute callback that are delayed
    callbacks.forEach((callback) => {
      callback(stats);
    });
  });
}
```

注意这些回调是放在 process.nextTick 中执行的。

> process.nextTick() adds callback to the "next tick queue". This queue is fully drained after the current operation on the JavaScript stack runs to completion and before the event loop is allowed to continue.

接着往下，这里决定是否将编译后的文件写入磁盘。

```js
if (options.writeToDisk) {
  setupWriteToDisk(context);
}
```

`writeToDisk`可以是一个布尔型，也可以是一个函数。

```js
export default function setupWriteToDisk(context) {
  const compilers = context.compiler.compilers || [context.compiler];

  for (const compiler of compilers) {
    compiler.hooks.emit.tap("DevMiddleware", (compilation) => {
      if (compiler.hasWebpackDevMiddlewareAssetEmittedCallback) {
        return;
      }

      compiler.hooks.assetEmitted.tapAsync(
        "DevMiddleware",
        (file, info, callback) => {
          // ...
        }
      );

      compiler.hasWebpackDevMiddlewareAssetEmittedCallback = true;
    });
  }
}
```

`wdm`也可以处理`MultiCompiler`，所以这里第一行有个处理 compilers 数组的操作。接下来针对每个 compiler 进行两个事件监听。

> **emit** Executed right before emitting assets to output dir.
>
> **assetEmitted** Executed when an asset has been emitted. Provides access to information about the emitted asset, such as its output path and byte content.

主要的文件信息都在第二个监听中可以获得，第一个监听用来去除重复监听的情况。

接下来开始处理文件系统，使得输出的文件到内存中，而不是磁盘中，加快写入读取速度。

```js
setupOutputFileSystem(context);
```

这个方法其实很简单，

```js
import { createFsFromVolume, Volume } from "memfs";

export default function setupOutputFileSystem(context) {
  let outputFileSystem;

  if (context.options.outputFileSystem) {
    // ...
  } else {
    outputFileSystem = createFsFromVolume(new Volume());
    // TODO: remove when we drop webpack@4 support
    outputFileSystem.join = path.join.bind(path);
  }

  const compilers = context.compiler.compilers || [context.compiler];

  for (const compiler of compilers) {
    // eslint-disable-next-line no-param-reassign
    compiler.outputFileSystem = outputFileSystem;
  }

  // eslint-disable-next-line no-param-reassign
  context.outputFileSystem = outputFileSystem;
}
```

利用`memfs`这个库，和 webpack 预留的`outputFileSystem`属性来实现这个功能。注意这里还是有对`MultiCompiler`的处理。

再往下就是开启 webpack 的 watch 模式了，

```js
context.watching = context.compiler.watch(watchOptions, (error) => {
  if (error) {
    // TODO: improve that in future
    // For example - `writeToDisk` can throw an error and right now it is ends watching.
    // We can improve that and keep watching active, but it is require API on webpack side.
    // Let's implement that in webpack@5 because it is rare case.
    context.logger.error(error);
  }
});
```

直到这里，都没有遇到任何中间件相关的代码，其实这部分代码都在这一行。

```js
const instance = middleware(context);
```

middleware 这个函数其实是这里导出的`wrapper`函数，

```js
export default function wrapper(context) {
  return async function middleware(req, res, next) {
    const acceptedMethods = context.options.methods || ["GET", "HEAD"];

    // fixes #282. credit @cexoso. in certain edge situations res.locals is undefined.
    // eslint-disable-next-line no-param-reassign
    res.locals = res.locals || {};

    if (!acceptedMethods.includes(req.method)) {
      await goNext();
      return;
    }

    ready(context, processRequest, req);

    async function goNext() {
      // ...
    }

    async function processRequest() {
      // ...
    }
  };
}
```

没错，`wrapper`是一个高阶函数。

middleware 默认只处理 get、head 请求。我们按顺序看，先看 ready 函数做了什么，

```js
export default function ready(context, callback, req) {
  if (context.state) {
    return callback(context.stats);
  }

  const name = (req && req.url) || callback.name;

  context.logger.info(`wait until bundle finished${name ? `: ${name}` : ""}`);

  context.callbacks.push(callback);
}
```

很简单，如果当前编译已经完成，直接执行回调，否则就存在回调队列中。

接着看 goNext 函数，也比较简单

```js
async function goNext() {
  if (!context.options.serverSideRender) {
    return next();
  }

  return new Promise((resolve) => {
    ready(
      context,
      () => {
        // eslint-disable-next-line no-param-reassign
        res.locals.webpack = { devMiddleware: context };

        resolve(next());
      },
      req
    );
  });
}
```

如果不需要支持服务端渲染（options.serverSideRender），直接执行下个中间件。否则就把 context 对象放到`res.local`中传递个下个中间件。

最后是 processRequest 这个函数，

```js
async function processRequest() {
  const filename = getFilenameFromUrl(context, req.url);
  // ...
  let content;

  if (!filename) {
    await goNext();
    return;
  }

  try {
    content = context.outputFileSystem.readFileSync(filename);
  } catch (_ignoreError) {
    await goNext();
    return;
  }
  // ...
  // Express API
  if (res.send) {
    res.send(content);
  }
  // Node.js API
  else {
    res.setHeader("Content-Length", content.length);

    if (req.method === "HEAD") {
      res.end();
    } else {
      res.end(content);
    }
  }
}
```

在这里，先通过 req.url，调用`getFilenameFromUrl`函数，来获取存在内存中的对应这个请求 url 的文件名。当不存在时，跳过下面的处理，跳到下个中间件去。如果存在，就读取处理，发送给客户端。

整个 middleware 的功能大体如上。

最后，`wdm`还会在这个中间件函数上添加一些实用的方法，

```js
// API
instance.getFilenameFromUrl = (url) => getFilenameFromUrl(context, url);

instance.waitUntilValid = (callback = noop) => {
  ready(context, callback);
};

instance.invalidate = (callback = noop) => {
  ready(context, callback);

  context.watching.invalidate();
};

instance.close = (callback = noop) => {
  context.watching.close(callback);
};

instance.context = context;
```

有了上面的讲解，这些方法的含义不言自明。
