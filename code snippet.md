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
