# co

版本 [4.6.0](https://github.com/tj/co/releases/tag/4.6.0)

自动执行 generator 函数

```js
co(function* () {
  yield; // promise, thunk, array, object, generator
});
```

首先要熟悉 generator 的语法，然后再看源码。源码总共只有两百来行，很容易看懂。

`co`要求`yield`后面只能跟着 `promise`，`thunk`，`array`，`object`，`generaotr`，并且会把这些类型统一转换为`promise`。这是因为`co`使用的自动执行机制利用了`promiselike.then`来递归调用下一步执行。

`co`函数的内部大致为

```js
function co(gen) {
  // ...
  return new Promise(function (resolve, reject) {
    // ...
    onFulfilled();

    function onFulfilled(res) {
      var ret;
      try {
        ret = gen.next(res);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }

    function onRejected(err) {
      var ret;
      try {
        ret = gen.throw(err);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }

    function next(ret) {
      if (ret.done) return resolve(ret.value);
      var value = toPromise.call(ctx, ret.value);
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
      return onRejected();
      //...
    }
  });
}
```

传入 generator 对象或函数，返回一个 promise。

主要的逻辑在`next`函数中，如果 generator 函数执行完毕，那么直接将结果`value`传递出去。否则将`value`转换为`promise`，再利用`value.then(onFulfilled, onRejected)`来继续执行 generator 函数。

这个`onFulfilled`中会调用生成器对象的`next`方法，并将前次的`yield`表达式的结果传入 genertor 函数中，这个结果就是前面那个 promise `value`的 resolved 状态下结果。然后继续调用`next`函数。

而`onRejected`函数的逻辑也很直接，将前面 promise 的 reject 状态下的结果传入 generator 函数，然后再次调用`next`函数。

这样最终实现了 generator 函数的自动执行。

相比于`async/await`，`co`实现了它们大部分的功能，在原理上都差不多。

> async 函数的实现原理，就是将 Generator 函数和自动执行器，包装在一个函数里。

```js
async function fn(args) {
  // ...
}
// 等同于
function fn(args) {
  return spawn(function* () {
    // ...
  });
}
```

这个`spawn`函数，可以这样实现

```js
function spawn(genF) {
  return new Promise((resolve, reject) => {
    const gen = genF();

    function step(nextF) {
      let res;

      try {
        res = nextF();
      } catch (e) {
        return reject(e);
      }

      if (res.done) {
        return resolve(res.value);
      } else {
        Promise.resolve(res.value)
          .then((v) => {
            step(() => gen.next(v));
          })
          .catch((e) => {
            step(() => gen.throw(e));
          });
      }
    }
    next(() => gen.next());
  });
}
```

思路上几乎是一样的。

## references

1. [什么是状态机？](https://zhuanlan.zhihu.com/p/47434856)
2. [Generator 函数的语法](https://es6.ruanyifeng.com/#docs/generator)
3. [Generator 函数的异步应用](https://es6.ruanyifeng.com/#docs/generator-async)
4. [What’s a thunk?!](https://github.com/reduxjs/redux-thunk#whats-a-thunk)
5. [Thunks](https://github.com/tj/co#thunks)
6. [async 函数的实现原理](https://es6.ruanyifeng.com/#docs/async#async-%E5%87%BD%E6%95%B0%E7%9A%84%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86)
