# retry

[master 分支](https://github.com/tim-kos/node-retry)

b316bfc196a89504fa348bcb97b20eb55509a295

在 js 处理异步任务时，有时会遇到需要重试的情况。所谓的重试，就是在首次请求失败的情况下，隔一段时间继续请求。面对这个需求，最常见的处理方式就是利用 setTimeout 函数，设置定时任务。大多数时候这种方式可以满足当前的需求。但是，当需要复用重试逻辑时，或者是重试需要满足一些条件，比如说最大重试次数、最长重试时间、每次重试时间要做到指数退避(Exponential Backoff)等等需求。直接使用 setTimeput 就会使得代码变得冗长，失去语义，难以理解。

这时候，就需要把重试操作抽象出来，提取为一个库。retry 就做了这样的事情。

## 思路

首先，重试大体上可以分为两步：

1. 尝试执行异步任务(执行)
2. 如果失败了，过一会回到 1 继续执行(重新执行)

执行任务很容易，只是一个函数调用。但是我们不能把重试逻辑直接塞入函数调用中，这样就会破坏执行任务这个抽象操作。然而 retry 这个库是基于 callback，在实现重试时，还是侵入到执行异步任务的代码中。

先看个例子

```js
var dns = require("dns");
var retry = require("../lib/retry");
var operation = retry.operation(opts);

operation.attempt(function (currentAttempt) {
  dns.resolve(address, function (err, addresses) {
    if (operation.retry(err)) {
      return;
    }

    cb(operation.mainError(), operation.errors(), addresses);
  });
});
```

首先初始化一个 operation，用来控制需要重试的任务。接着将异步任务用 attempt 方法包起来，用于在 retry 方法中重新执行。

在源码中，初始化 operation 时会先创建一个时间数组，存着每次重试等待的时间。接着再将其作为参数，创建一个 RetryOperation 对象，具体的逻辑由这个对象完成。

```js
exports.operation = function (options) {
  var timeouts = exports.timeouts(options);
  return new RetryOperation(timeouts, {
    forever: options && options.forever,
    unref: options && options.unref,
    maxRetryTime: options && options.maxRetryTime,
  });
};
```

重试时间这块，使用了指数退避规则，有兴趣可以看看这篇[文章](http://dthain.blogspot.com/2009/02/exponential-backoff-in-distributed.html)。

```js
exports.createTimeout = function (attempt, opts) {
  var random = opts.randomize ? Math.random() + 1 : 1;

  var timeout = Math.round(
    random * opts.minTimeout * Math.pow(opts.factor, attempt)
  );
  timeout = Math.min(timeout, opts.maxTimeout);

  return timeout;
};
```

attempt 这块很简单，直接执行传入的异步任务，关键逻辑在 retry 这块。如果将 error 传入 retry，那么会根据条件来决定是否重试，以及什么时候重试。这是 retry 方法代码片段，返回 false 表示不需要重试，返回 true 表示需要。在三种情况下不需要重试：

1. 没有错误
2. 从第一次执行开始到现在的时间，超过了设置的最大重试时间
3. 超过了最大重试次数

```js
if (!err) {
  return false;
}
var currentTime = new Date().getTime();
if (err && currentTime - this._operationStart >= this._maxRetryTime) {
  this._errors.unshift(new Error("RetryOperation timeout occurred"));
  return false;
}

this._errors.push(err);

var timeout = this._timeouts.shift();
if (timeout === undefined) {
  if (this._cachedTimeouts) {
    // retry forever, only keep last error
    this._errors.splice(0, this._errors.length - 1);
    this._timeouts = this._cachedTimeouts.slice(0);
    timeout = this._timeouts.shift();
  } else {
    return false;
  }
}
```

当需要重试时，就将重试操作放入定时器中。

```js
var self = this;
this._timer = setTimeout(function () {
  self._attempts++;

  if (self._operationTimeoutCb) {
    self._timeout = setTimeout(function () {
      self._operationTimeoutCb(self._attempts);
    }, self._operationTimeout);

    if (self._options.unref) {
      self._timeout.unref();
    }
  }

  self._fn(self._attempts);
}, timeout);
```

主要的思路就是这些。不难发现整个库的设计存在两个缺陷：

1. 侵入了业务代码
2. 基于回调，而不是 promise

## 改造

在 retry 库的基础上，有多个封装后的版本。其中一个比较不错的是 [async-retry](https://github.com/vercel/async-retry)。

先看下 demo,也是直接将异步任务包起来，返回一个新的可重试版本的 promise，且没有侵入业务逻辑。这是 promise 带来的好处。

```js
const getRandomTitle = async () =>
  retry(async () => {
    const res = await fetch("https://en.wikipedia.org/wiki/Special:Random");
    const text = await res.text();
    const $ = cheerio.load(text);
    return $("h1").text();
  });

// eslint-disable-next-line no-console
getRandomTitle().then(console.log);
```

再来看看实现，async-retry 的代码只有 62 行。

```js
var retrier = require("retry");

function retry(fn, opts) {
  function run(resolve, reject) {
    var options = opts || {};
    var op;

    // Default `randomize` to true
    if (!("randomize" in options)) {
      options.randomize = true;
    }

    op = retrier.operation(options);

    function bail(err) {
      reject(err || new Error("Aborted"));
    }

    function onError(err, num) {
      if (err.bail) {
        bail(err);
        return;
      }

      if (!op.retry(err)) {
        reject(op.mainError());
      } else if (options.onRetry) {
        options.onRetry(err, num);
      }
    }

    function runAttempt(num) {
      var val;

      try {
        val = fn(bail, num);
      } catch (err) {
        onError(err, num);
        return;
      }

      Promise.resolve(val)
        .then(resolve)
        .catch(function catchIt(err) {
          onError(err, num);
        });
    }

    op.attempt(runAttempt);
  }

  return new Promise(run);
}
```

窍门是内部创建了一个 promise，直接返回。利用执行包起来的 fn 来决定返回的 promise 是 resolve 还是 reject。重试的逻辑封装在 onError 函数中，只有执行 fn 出错了，才会进入 onError。另外，在执行 fn 时，会传入 bail 函数，用来终止重试。在调用 bail 后，返回的 promise 会立即 reject。之后的代码不能再抛出错误，否则又会进入 onError。

```js
await retry(
  async (bail) => {
    // if anything throws, we retry
    const res = await fetch("https://google.com");

    if (403 === res.status) {
      // don't retry upon 403
      bail(new Error("Unauthorized"));
      return;
    }

    const data = await res.text();
    return data.substr(0, 500);
  },
  {
    retries: 5,
  }
);
```
