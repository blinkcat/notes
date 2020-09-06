# fastdom

> Eliminates layout thrashing by batching DOM read/write interactions.

在频繁读写 dom 元素的位置、尺寸信息时，稍有不慎，就会出现布局抖动。可以看这个[demo](http://wilsonpage.github.io/fastdom/examples/animation.html)。

```js
function update(thisTimestamp) {
  timestamp = thisTimestamp;
  for (var m = 0; m < movers.length; m++) {
    mover[moveMethod](m);
  }
  raf = window.requestAnimationFrame(update);
}

var mover = {
  sync: function (m) {
    // Read the top offset, and use that for the left position
    mover.setLeft(movers[m], movers[m].offsetTop);
  },
  async: function (m) {
    // Use fastdom to batch the reads
    // and writes with exactly the same
    // code as the 'sync' routine
    fastdom.measure(function () {
      var top = movers[m].offsetTop;
      fastdom.mutate(function () {
        mover.setLeft(movers[m], top);
      });
    });
  },
  setLeft: function (mover, top) {
    mover.style.transform =
      "translateX( " + (Math.sin(top + timestamp / 1000) + 1) * 500 + "px)";
  },
};
```

先读取，再设置位置信息，或者反过来，都会导致浏览器立即进行[Layout](https://developers.google.com/web/tools/chrome-devtools/rendering-tools#layout)，非常耗时。

## 原理

理论上，将读写操作分离，分别批量执行，就会大幅提升性能。那么如何实现批量读写呢？先从使用方法上看：

```js
fastdom.measure(() => {
  console.log("measure");
});

fastdom.mutate(() => {
  console.log("mutate");
});

fastdom.measure(() => {
  console.log("measure");
});

fastdom.mutate(() => {
  console.log("mutate");
});

// Outputs:
// measure
// measure
// mutate
// mutate
```

将当前 event loop 中交叉的读写操作存储在读写两个队列中。 fastdom.measure() 把读操作存在读队列中，fastdom.mutate 把写操作存在写队列中。这样执行的时候，先执行读队列，再执行写队列。

```js
measure: function(fn, ctx) {
  debug('measure');
  var task = !ctx ? fn : fn.bind(ctx);
  this.reads.push(task); // 加入队列
  scheduleFlush(this);
  return task;
},

mutate: function(fn, ctx) {
  debug('mutate');
  var task = !ctx ? fn : fn.bind(ctx);
  this.writes.push(task); // 加入队列
  scheduleFlush(this);
  return task;
},
```

那么应该什么时候执行呢？理想情况是当前最后一个读写语句执行之后。但是没法判断到哪里才是最后一句。换个思路，我们可以尽量收集当前 event loop 中的读写语句，将执行放入下一次 loop 中。

```js
function scheduleFlush(fastdom) {
  if (!fastdom.scheduled) {
    fastdom.scheduled = true;
    // fastdom.raf -> requestAnimationFrame
    fastdom.raf(flush.bind(null, fastdom));
    debug("flush scheduled");
  }
}

function flush(fastdom) {
  debug("flush");

  var writes = fastdom.writes;
  var reads = fastdom.reads;
  var error;

  try {
    debug("flushing reads", reads.length);
    fastdom.runTasks(reads);
    debug("flushing writes", writes.length);
    fastdom.runTasks(writes);
  } catch (e) {
    error = e;
  }

  fastdom.scheduled = false;

  // If the batch errored we may still have tasks queued
  if (reads.length || writes.length) scheduleFlush(fastdom);

  if (error) {
    debug("task errored", error.message);
    if (fastdom.catch) fastdom.catch(error);
    else throw error;
  }
}
```

这里还有一个问题，我们的读写语句之间一般存在依赖关系，比如

```js
const top = div.offsetTop;
div.style.left = top;
const left = div.offsetLeft;
div.style.left = left + top;
```

这个时候如果一句一句的处理

```js
let top;
fastdom.measure(() => {
  top = div.offsetTop;
});
fastdom.mutate(() => {
  div.style.left = top;
});
const left
fastdom.measure(() => {
  left = div.offsetLeft;
});
fastdom.mutate(() => {
  div.style.left = left + top;
});
// 那么实际执行顺序会变成
top = div.offsetTop;
left = div.offsetLeft; // 这里在取值之前，其实已经进行了赋值

div.style.left = top;
div.style.left = left + top;
// 显然是有问题的
```

可以利用闭包处理顺序问题

```js
const top;
fastdom.measure(() => {
  top = div.offsetTop;
});
fastdom.mutate(() => {
  div.style.left = top;
  const left;
  fastdom.measure(() => {
    left = div.offsetLeft;
  });
  fastdom.mutate(() => {
    div.style.left = left + top;
  });
});
// 顺序变成
top = div.offsetTop;
div.style.left = top;
left = div.offsetLeft;
div.style.left = left + top;
```

虽然顺序正确了，但是可读性下降明显。可见 fastdom 不适用这种场合，真正发挥威力的还是读写语句块之间没有依赖关系，包在循环或递归里面，多次执行的情况。比如：

```js
function setAspectFastDom(div, i) {
  var aspect = 9 / 16;
  var isLast = i === n - 1;

  // READ
  fastdom.measure(function () {
    var h = div.clientWidth * aspect;

    // WRITE
    fastdom.mutate(function () {
      div.style.height = h + "px";

      if (isLast) {
        displayPerf(performance.now() - start);
      }
    });
  });
}
// ...
divs.forEach(setAspectFastDom);
```

在这种情况下，可以大幅提升性能。

## 可取消

fastdom.measure 和 fastdom.mutate 会返回待执行的函数，通过这个返回值可以取消未执行的函数。

```js
clear: function(task) {
  debug('clear', task);
  // task只可能在一个队列中
  return remove(this.reads, task) || remove(this.writes, task);
},

function remove(array, item) {
  var index = array.indexOf(item);
  return !!~index && !!array.splice(index, 1);
}
```

## 可扩展
