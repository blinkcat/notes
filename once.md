# once

版本[v1.4.0](https://github.com/isaacs/once/tree/v1.4.0)

## Demo

```js
var once = require("once");

function load(cb) {
  cb = once(cb);
  var stream = createStream();
  stream.once("data", cb);
  stream.once("end", function () {
    if (!cb.called) cb(new Error("not found"));
  });
}
```

更多用法参见[usage](https://github.com/isaacs/once#usage)

## How it works

[wrappy](./wrappy.md)在之前介绍过。

once 函数为返回的新的函数添加了两个属性，called 和 value。这两个属性在某些时候特别有用，例如

```js
var once = require("once");

function load(cb) {
  cb = once(cb);
  var stream = createStream();
  stream.once("data", cb);
  stream.once("end", function () {
    if (!cb.called) cb(new Error("not found"));
  });
}
```

onceStrict 则是在此基础上，当多次调用时抛出错误。

once.proto 则是为所有的函数都添加了前两个函数的功能。不过似乎没有必要。

> Ironically, the prototype feature makes this module twice as
> complicated as necessary.

```js
var wrappy = require("wrappy");
module.exports = wrappy(once);
module.exports.strict = wrappy(onceStrict);

once.proto = once(function () {
  Object.defineProperty(Function.prototype, "once", {
    value: function () {
      return once(this);
    },
    configurable: true,
  });

  Object.defineProperty(Function.prototype, "onceStrict", {
    value: function () {
      return onceStrict(this);
    },
    configurable: true,
  });
});

function once(fn) {
  var f = function () {
    if (f.called) return f.value;
    f.called = true;
    return (f.value = fn.apply(this, arguments));
  };
  f.called = false;
  return f;
}

function onceStrict(fn) {
  var f = function () {
    if (f.called) throw new Error(f.onceError);
    f.called = true;
    return (f.value = fn.apply(this, arguments));
  };
  var name = fn.name || "Function wrapped with `once`";
  f.onceError = name + " shouldn't be called more than once";
  f.called = false;
  return f;
}
```
