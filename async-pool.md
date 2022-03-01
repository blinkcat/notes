# async-pool

版本 [v1.2.0](https://github.com/rxaviers/async-pool/tree/v1.2.0)

用来对执行一组异步任务做频率限制。

## Usage

```js
const timeout = (i) =>
  new Promise((resolve) => setTimeout(() => resolve(i), i));
// asyncPool(poolLimit, array, iteratorFn)
const results = await asyncPool(2, [1000, 5000, 3000, 2000], timeout);
```

## How it works

为了实现这个功能，最初我的想法是利用两个队列存储当前任务和剩余任务。当前任务队列长度按照 limit 值来限制。每当有任务完成，就去剩余任务队列取一个任务过来。这个库也是类似的思路，但是巧妙的地方在于利用了`await Promise.race(xxx)`来执行当前任务，且剩余的任务利用外层 for 循环来依次执行。因为 await 会等待当前 Promise 的结果，自然也就会在当前任务未出现一个结果之前阻塞外层的 for 循环。

```js
let assert, assertType;
const shouldAssert = process.env.NODE_ENV === "development";

if (shouldAssert) {
  ({ assert, assertType } = require("yaassertion"));
}

async function asyncPool(poolLimit, array, iteratorFn) {
  if (shouldAssert) {
    assertType(poolLimit, "poolLimit", ["number"]);
    assertType(array, "array", ["array"]);
    assertType(iteratorFn, "iteratorFn", ["function"]);
  }
  const ret = [];
  const executing = [];
  for (const item of array) {
    const p = Promise.resolve().then(() => iteratorFn(item, array));
    ret.push(p);

    if (poolLimit <= array.length) {
      const e = p.then(() => executing.splice(executing.indexOf(e), 1));
      executing.push(e);
      if (executing.length >= poolLimit) {
        await Promise.race(executing);
      }
    }
  }
  return Promise.all(ret);
}

module.exports = asyncPool;
```

除此之外，还要对 Promise 的语法有深入的理解。比如说这一行，这样是为了保证即使 iteratorFn 函数执行后不返回 promise 实例，整个内部流程也能继续。因为 p 始终是一个 promise。

```js
const p = Promise.resolve().then(() => iteratorFn(item, array));
```

然后是这里，变量 p 是一个 promise，在执行完 then 后，会返回一个新的 promise。这个新的 promise 存在 executing 数组中，当它执行完毕时，会把自己从 executing 数组中删除。

```js
const e = p.then(() => executing.splice(executing.indexOf(e), 1));
executing.push(e);
```

最后是这里，ret 数组存着一堆 promise，这些 promise 就是上面的变量 p 表示的那样，最终返回 iteratorFn 函数的执行结果。promise 的特性就是当它有结果时，状态就不再可变，并且记住当前的结果。所以 Promise.all 会返回所有的执行结果。这里要把 ret 数组放到 Promise.all 中的原因有两个，

1. 当 poolLimit 超过当前剩余任务数组长度时，等于没有限制，可以直接并发执行这些任务
2. 如果当前执行任务队列的长度为达到 poolLimit，那么就不会执行这些任务。所以在最后还要靠 Promise.all 来执行

```js
return Promise.all(ret);
```
