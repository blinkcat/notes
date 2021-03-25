# koa

版本[2.13.1](https://github.com/koajs/koa/tree/2.13.1)

## 整体结构

koa 是由 express 原班人马打造，在思想上更加成熟。在核心的 middleware 上，引入了洋葱模型的概念，充分利用了新版 js 语法的特性，实现无论是实现还是使用都更加简洁。

## middleware

不同于 express 的回调式风格，koa 依靠了 promise 的特性使得 middleware 的实现和使用更加简单，并且形成了一种洋葱模型。举个例子，

```js
app.use(async (ctx, next) => {
  console.log(1);
  await next();
  console.log(6);
});
app.use(async (ctx, next) => {
  console.log(2);
  await next();
  console.log(5);
});
app.use(async (ctx, next) => {
  console.log(3);
  await next();
  console.log(4);
});
```

打印出来的结果是 1，2，3，4，5，6。

这个例子表明了 koa 的 middleware 的两个特性，

1. 执行顺序上符合洋葱模型，即 1->2->3->2->1，好预测，next 执行完成后，前一个 middleware 一定完成了工作。
2. middleware 的函数签名简单，一致。因为使用了 async，处理执行链上的错误更容易。

下面看看源码，

```js
  use(fn) {
    if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
    if (isGeneratorFunction(fn)) {
      deprecate('Support for generators will be removed in v3. ' +
                'See the documentation for examples of how to convert old middleware ' +
                'https://github.com/koajs/koa/blob/master/docs/migration.md');
      fn = convert(fn);
    }
    debug('use %s', fn._name || fn.name || '-');
    this.middleware.push(fn);
    return this;
  }
```

没什么好说的，就是把 fn 添加到`this.middleware`数组中。

再看看 middleware 在使用前是怎么处理的。

```js
  callback() {
    const fn = compose(this.middleware);

    if (!this.listenerCount('error')) this.on('error', this.onerror);

    const handleRequest = (req, res) => {
      const ctx = this.createContext(req, res);
      return this.handleRequest(ctx, fn);
    };

    return handleRequest;
  }
```

重点在这一行

`const fn = compose(this.middleware);`

`compose`函数对`this.middleware`做了处理后，传给了`this.handleRequest`。

```js
  handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    res.statusCode = 404;
    const onerror = err => ctx.onerror(err);
    const handleResponse = () => respond(ctx);
    onFinished(res, onerror);
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }
```

将`ctx`传给`fnMiddleware`后，返回了一个 promise。上面这些代码看起来都很普通，那么到底是哪里实现了 middleware 呢？关键在`compose`这个函数。

```js
function compose(middleware) {
  if (!Array.isArray(middleware))
    throw new TypeError("Middleware stack must be an array!");
  for (const fn of middleware) {
    if (typeof fn !== "function")
      throw new TypeError("Middleware must be composed of functions!");
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */

  return function (context, next) {
    // last called middleware #
    let index = -1;
    return dispatch(0);
    function dispatch(i) {
      if (i <= index)
        return Promise.reject(new Error("next() called multiple times"));
      index = i;
      let fn = middleware[i];
      if (i === middleware.length) fn = next;
      if (!fn) return Promise.resolve();
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err);
      }
    }
  };
}
```

忽略前面的类型判断，重点就在返回的这个新函数。这个新的匿名函数符合 middleware 的签名，它的作用是把多个 middleware 组合成一个 middleware，然后返回一个 promise，表示这些 middleware 的处理结果。注意里面那个`dispatch`函数，它始终返回一个 promise。从`dispatch(0)`开始递归调用。调用的方式很特别，`return Promise.resolve(fn(context, dispatch.bind(null, i + 1)))`。这一行代码其实包含了很多的巧思在里面，

1. 使用`Promise.resolve`包起来，保证这个返回结果始终是一个 promise。如果`fn`返回一个 promise，则以这个 promise 的结果作为最终结果返回。
2. 回看上面的例子，middleware 入参中的`next`，就是这里传入的`dispatch.bind(null, i + 1)`。由于`dispatch`始终返回 promise，所以我们在 middleware 中如果想调用下个 middleware，必须使用`await next()`或者`next().then()`这种方式。而通过这种方式，我们就可以把多个 middleware 串联起来了，而且还带来了一个好处，错误的捕获和传递变得容易起来，可以完全依靠语言自身支持。

这就是 koa 实现 middleware 的方式。

## context

另外一点和 express 不同的是，koa 使用了 context 来代理 req 和 res。

```js
/**
 * Response delegation.
 */
delegate(proto, "response").method("attachment")...

/**
 * Request delegation.
 */
delegate(proto, "request").method("acceptsLanguages")...
// ...
```

在使用 middleware 的时候传入，这样我们不会直接接触到 req，res，也就不会在上面挂载一些东西，更不会和 node 原生的属性相冲突，相当于 koa 给我们提供了一个窄接口。

## conclusion

koa 本身只保留了很核心的部分，如 middleware，response，request。其他功能包括 router，static，bodyparser 都由额外的中间件提供。相比于 express，koa 更加精简。大量使用 promise 来实现功能，肯定比 callback 的方式更加先进。
