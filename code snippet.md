# code snippet

好的代码片段

## nprogress

[see](https://github.com/rstacruz/nprogress/blob/e1a8b7fb6e059085df5f83c45d3c2308a147ca18/nprogress.js#L361-L379)

```js
/**
 * (Internal) Queues a function to be executed.
 */

var queue = (function () {
  var pending = [];

  function next() {
    var fn = pending.shift();
    if (fn) {
      fn(next);
    }
  }

  return function (fn) {
    pending.push(fn);
    if (pending.length == 1) next();
  };
})();
```

[see](https://github.com/rstacruz/nprogress/blob/master/nprogress.js#L154-L177)

```js
/**
 * Increments by a random amount.
 */

NProgress.inc = function (amount) {
  var n = NProgress.status;

  if (!n) {
    return NProgress.start();
  } else if (n > 1) {
    return;
  } else {
    if (typeof amount !== "number") {
      if (n >= 0 && n < 0.2) {
        amount = 0.1;
      } else if (n >= 0.2 && n < 0.5) {
        amount = 0.04;
      } else if (n >= 0.5 && n < 0.8) {
        amount = 0.02;
      } else if (n >= 0.8 && n < 0.99) {
        amount = 0.005;
      } else {
        amount = 0;
      }
    }

    n = clamp(n + amount, 0, 0.994);
    return NProgress.set(n);
  }
};
```
