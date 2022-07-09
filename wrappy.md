# wrappy

版本[wrappy](https://github.com/npm/wrappy/tree/v1.0.2)

wrappy 接收一个函数 fn，这个 fn 可以接收一个 cb 函数，一般然后返回一个新的函数。wrappy 执行后返回一个 wrapper 函数，这个 wrapper 函数不仅拥有 fn 的静态属性，而且也可以接收一个 cb 函数。当传入一个 cb 函数给 wrapper 执行后，其内部执行 fn(cb)，然后返回执行结果。如果这个结果是一个函数，那么这个函数也拥有 cb 的静态属性。

## Demo

```js
var wrappy = require("wrappy");

// var wrapper = wrappy(wrapperFunction)

// make sure a cb is called only once
// See also: http://npm.im/once for this specific use case
var once = wrappy(function (cb) {
  var called = false;
  return function () {
    if (called) return;
    called = true;
    return cb.apply(this, arguments);
  };
});

function printBoo() {
  console.log("boo");
}
// has some rando property
printBoo.iAmBooPrinter = true;

var onlyPrintOnce = once(printBoo);

onlyPrintOnce(); // prints 'boo'
onlyPrintOnce(); // does nothing

// random property is retained!
assert.equal(onlyPrintOnce.iAmBooPrinter, true);
```

## How it works

源码相当精简。wrappy 函数的主要工作就是保证入参函数和返回值函数的静态属性可以保留。具体的业务逻辑还是在第一个参数 fn 函数中。

```js
// Returns a wrapper function that returns a wrapped callback
// The wrapper function should do some stuff, and return a
// presumably different callback function.
// This makes sure that own properties are retained, so that
// decorations and such are not lost along the way.
module.exports = wrappy;
function wrappy(fn, cb) {
  if (fn && cb) return wrappy(fn)(cb);

  if (typeof fn !== "function") throw new TypeError("need wrapper function");

  Object.keys(fn).forEach(function (k) {
    wrapper[k] = fn[k];
  });

  return wrapper;

  function wrapper() {
    var ret = fn.apply(this, arguments);
    var cb = arguments[arguments.length - 1];
    if (typeof ret === "function" && ret !== cb) {
      Object.keys(cb).forEach(function (k) {
        ret[k] = cb[k];
      });
    }
    return ret;
  }
}
```
