# thread-loader

版本[v3.0.4](https://github.com/webpack-contrib/thread-loader/tree/v3.0.4)

## Demo

**webpack.config.js**

```js
module.exports = {
  module: {
    rules: [
      {
        test: /\.js$/,
        include: path.resolve("src"),
        use: [
          "thread-loader",
          // your expensive loader (e.g babel-loader)
        ],
      },
    ],
  },
};
```

## How it works

众所周知 js 的程序只跑在一个线程上，这就在处理计算密集型任务上捉襟见肘。因为无法有效地运用多核 cpu 的能力。在 webpack 的工作中，有相当一部分是编译/转译原始代码，这部分工作涉及到大量的字符串查找拼接等工作，属于计算密集型，如果能充分利用多核 cpu，效率上会有极大的提升。thread-loader 应运而生。

thread-loader 工作在[pitch](https://webpack.js.org/api/loaders/#pitching-loader)阶段。因此可以主动去利用后面的 loader 来处理文件。如果更进一步，它可以将这些 loader 的工作分配到其他的线程上执行。

```js
import loaderUtils from "loader-utils";
import { getPool } from "./workerPools";

function pitch() {
  const options = loaderUtils.getOptions(this);
  const workerPool = getPool(options);
  // ...
  const callback = this.async();

  workerPool.run(
    {
      // ...
    },
    (err, r) => {
      // ...
      callback(null, ...r.result);
    }
  );
}
```

但是这里所指的线程其实是其他进程的线程，而不是当前进程的线程。也就是说 thread-loader 实际上是通过多进程的方式来实现多任务并行处理。至于为何要这样处理，在这两个 issue 中有讨论，但这不是本文的重点。

1. [https://github.com/webpack-contrib/thread-loader/pull/81#issuecomment-560985487](https://github.com/webpack-contrib/thread-loader/pull/81#issuecomment-560985487)
2. [Consider using worker threads instead of using spawn](https://github.com/webpack-contrib/thread-loader/issues/61)

在 thread-loader 内部，有一个或者多个 WorkerPool，更多情况下是一个。这个 WorkerPool 中有多个 PoolWorker，每个 PoolWorker 都是一个工作进程的代理。这个进程是通过 childProcess.spawn 来创建的子进程。父子进程之间通过管道通信。当 webpack 工作时，某些匹配的文件会通过上面提到的 pitch 函数，在这个函数中会调用 workerPool.run 将工作分发给 WorkerPool 中的 PoolWorker。而 PoolWorker 又会将工作交给它代理的子进程去完成。这就是 thread-loader 整体的思路。

下面我们主要来看下 WorkerPool，PoolWorker 以及子进程和主进程之间的通信规则。

在 WorkerPool 中使用 Set 来存储所有的 PoolWorker。使用[asyncQueue](http://caolan.github.io/async/v3/docs.html#queue)来存储所有的待分配的任务。

```js
export default class WorkerPool {
  constructor(options) {
    // ...
    this.workers = new Set();
    this.activeJobs = 0;
    // ...
    this.poolQueue = asyncQueue(
      this.distributeJob.bind(this),
      options.poolParallelJobs
    );
    // ...
  }
}
```

当 thread-loader 工作时，run 方法会被调用，传入任务相关的数据和回调，然后存入 poolQueue 中等待执行。

```js
export default class WorkerPool {
  // ...
  run(data, callback) {
    // ...
    this.activeJobs += 1;
    this.poolQueue.push(data, callback);
  }
}
```

而 poolQueue 会使用 distributeJob 方法来分配工作。首先从已有的 worker 中挑选比较闲的 bestWorker，但不会立即使用它，而是接着判断它是不是完全的空闲或者是说当前的 worker 数量是否达到上限。如果不满足这些条件，就会再创建一个新 worker。

```js
export default class WorkerPool {
  // ...
  distributeJob(data, callback) {
    // use worker with the fewest jobs
    let bestWorker;
    for (const worker of this.workers) {
      if (!bestWorker || worker.activeJobs < bestWorker.activeJobs) {
        bestWorker = worker;
      }
    }
    if (
      bestWorker &&
      (bestWorker.activeJobs === 0 || this.workers.size >= this.numberOfWorkers)
    ) {
      bestWorker.run(data, callback);
      return;
    }
    const newWorker = this.createWorker();
    newWorker.run(data, callback);
  }
}
```

创建新 worker 的过程很简单，额外传入一个 onJobDone 回调，用来协助 WorkerPool 回收资源。

```js
export default class WorkerPool {
  // ...
  createWorker() {
    // spin up a new worker
    const newWorker = new PoolWorker(
      {
        nodeArgs: this.workerNodeArgs,
        parallelJobs: this.workerParallelJobs,
      },
      () => this.onJobDone()
    );
    this.workers.add(newWorker);
    return newWorker;
  }
  // ...
  onJobDone() {
    this.activeJobs -= 1;
    if (this.activeJobs === 0 && isFinite(this.poolTimeout)) {
      this.timeout = setTimeout(() => this.disposeWorkers(), this.poolTimeout);
    }
  }
}
```

WorkerPool 在两种情况下会开始清理资源，第一种是在进程退出的情况下，监听[exit](https://nodejs.org/docs/latest-v14.x/api/process.html#process_event_exit)事件。第二种是在上面提到的 onJobDone 回调中判断，如果当前没有任务在执行，可以在一段时间后清理所有的 worker。

```js
export default class WorkerPool {
  constructor(options) {
    // ...
    this.terminated = false;
    this.setupLifeCycle();
  }
  // ...
  terminate() {
    if (this.terminated) {
      return;
    }

    this.terminated = true;
    this.poolQueue.kill();
    this.disposeWorkers(true);
  }

  setupLifeCycle() {
    process.on("exit", () => {
      this.terminate();
    });
  }
  // ...
  disposeWorkers(fromTerminate) {
    if (!this.options.poolRespawn && !fromTerminate) {
      this.terminate();
      return;
    }

    if (this.activeJobs === 0 || fromTerminate) {
      for (const worker of this.workers) {
        worker.dispose();
      }
      this.workers.clear();
    }
  }
}
```

接着我们去看看具体的 worker，即 PoolWorker 是如何工作的。

在 PoolWorker 的构造函数中，会创建子进程来处理任务。而父子进程之间的通信靠的是 pipe。在这里一共创建了 4 个 pipe，实际上只有最后两个会用于通信。这两个 pipe 都是 duplex 双工的 stream，可读可写，但是还是将第一个作为 readPipe，第二个作为 writePipe。做了这些准备工作后，可以调用 readNextMessage 方法，不断地通过 pipe 和子进程进行读写交互。

```js
const workerPath = require.resolve("./worker");
// ...
class PoolWorker {
  constructor(options, onJobDone) {
    this.disposed = false;
    // ...
    this.worker = childProcess.spawn(
      process.execPath,
      [].concat(sanitizedNodeArgs).concat(workerPath, options.parallelJobs),
      {
        detached: true,
        stdio: ["ignore", "pipe", "pipe", "pipe", "pipe"],
      }
    );
    // ...
    const [, , , readPipe, writePipe] = this.worker.stdio;
    // ...
    this.readNextMessage();
  }
  // ...
}
```

我们从 readNextMessage 方法的实现中可以看到两点，第一是它会递归的调用自己，不断地去读取信息，但是这个过程明显是异步的。第二是这个读取的过程是通过 readBuffer 函数来实现的。需要连续读取两次才能获得一条完整的信息，第一次读到的是一个整数，表示正文的长度，第二次才是读取正文。因为数据是通过 stream 来传递的流数据，需要自己来确定起始结束的位置。在获取到正文数据并且序列化后交由 onWorkerMessage 函数处理。

```js
class PoolWorker {
  // ...
  readNextMessage() {
    // ...
    this.readBuffer(4, (lengthReadError, lengthBuffer) => {
      // ...
      const length = lengthBuffer.readInt32BE(0);
      // ...
      this.readBuffer(length, (messageError, messageBuffer) => {
        // ...
        const messageString = messageBuffer.toString("utf-8");
        const message = JSON.parse(messageString, reviver);
        // ...
        this.onWorkerMessage(message, (err) => {
          // ...
          this.state = "soon next";
          setImmediate(() => this.readNextMessage());
        });
      });
    });
  }
  // ...
  readBuffer(length, callback) {
    readBuffer(this.readPipe, length, callback);
  }
}
```

readBuffer 是一个很重要的函数，封装了从 pipe(管道)中读取数据的逻辑。这需要你多少要了解些[readable](https://nodejs.org/docs/latest-v14.x/api/stream.html#stream_class_stream_readable)的知识。readChunk 函数每次会尝试读取 length 数量的数据，如果多读取了一部分，就需要将多的那部分放回 pipe 中。最后当读取完成时，还会去除 data 事件的监听，并且暂停 pipe。可以把这个过程想象成从水龙头中取定量的水一样。

```js
export default function readBuffer(pipe, length, callback) {
  if (length === 0) {
    callback(null, Buffer.alloc(0));
    return;
  }

  let remainingLength = length;

  const buffers = [];

  const readChunk = () => {
    const onChunk = (arg) => {
      let chunk = arg;
      let overflow;

      if (chunk.length > remainingLength) {
        overflow = chunk.slice(remainingLength);
        chunk = chunk.slice(0, remainingLength);
        remainingLength = 0;
      } else {
        remainingLength -= chunk.length;
      }

      buffers.push(chunk);

      if (remainingLength === 0) {
        pipe.removeListener("data", onChunk);
        pipe.pause();

        if (overflow) {
          pipe.unshift(overflow);
        }

        callback(null, Buffer.concat(buffers, length));
      }
    };

    pipe.on("data", onChunk);
    pipe.resume();
  };
  readChunk();
}
```

这是 PoolWorker 读数据的逻辑，还有写数据。

在 WorkerPool 中，会通过调用 PoolWorker 的 run 方法来传递任务。然后 run 方法通过 writeJson 方法将任务传递给子进程。从 writeJson 方法中能看出数据传递的格式是先传数据的长度，再传数据本身。

```js
class PoolWorker {
  // ...
  run(data, callback) {
    const jobId = this.nextJobId;
    this.nextJobId += 1;
    this.jobs[jobId] = { data, callback };
    this.activeJobs += 1;
    this.writeJson({
      type: "job",
      id: jobId,
      data,
    });
  }
  // ...
  writeJson(data) {
    const lengthBuffer = Buffer.alloc(4);
    const messageBuffer = Buffer.from(JSON.stringify(data, replacer), "utf-8");
    lengthBuffer.writeInt32BE(messageBuffer.length, 0);
    this.writePipe.write(lengthBuffer);
    this.writePipe.write(messageBuffer);
  }
}
```

在了解了 PoolWorker 中数据的收发机制之后，我们再来看看子进程是如何接收，发送数据的。

子进程的逻辑在 worker.js 文件中，我们可以从中发现很多相似的数据读写逻辑。子进程从文件描述符 3 和 4 中创建读写流，顺序正好和代理子进程的 PoolWorker 中的读写流相反。接着就是利用这两个流和 PoolWorker 通信。

```js
// src/worker.js
// ...
const writePipe = fs.createWriteStream(null, { fd: 3 });
const readPipe = fs.createReadStream(null, { fd: 4 });
// ...
function writeJson(data) {
  writePipeCork();
  process.nextTick(() => {
    writePipeUncork();
  });

  const lengthBuffer = Buffer.alloc(4);
  const messageBuffer = Buffer.from(JSON.stringify(data, replacer), "utf-8");
  lengthBuffer.writeInt32BE(messageBuffer.length, 0);

  writePipeWrite(lengthBuffer);
  writePipeWrite(messageBuffer);
}
// ...
function readNextMessage() {
  readBuffer(readPipe, 4, (lengthReadError, lengthBuffer) => {
    // ...
    const length = lengthBuffer.length && lengthBuffer.readInt32BE(0);
    // ...
    readBuffer(readPipe, length, (messageError, messageBuffer) => {
      // ...
      const messageString = messageBuffer.toString("utf-8");
      const message = JSON.parse(messageString, reviver);

      onMessage(message);
      setImmediate(() => readNextMessage());
    });
  });
}

// start reading messages from main process
readNextMessage();
```

下面我们来看下 PoolWorker 和子进程之间如何处理消息。

PoolWorker 会接收这几种消息，

- job
- loadModule
- resolve
- emitWarning
- emitError

子进程则会接收，

- job
- result
- warmup

反过来说，双方能接收的，都是对方能发送的消息。

首先，poolWorker 会向 subProcess(子进程)发送 job 消息，这个消息中会携带和这个任务相关的数据。

```js
{
	type: 'job',
    id: jobId,
    data,
}
```

subProcess 在接收到这个消息后，会将消息放入它的[asyncQueue](http://caolan.github.io/async/v3/docs.html#queue)中等待执行，这个 asyncQueue 和我们上面在 WorkerPool 中提到的那个一样，可以并发执行任务。

subProcess 依赖[loader-runner](https://github.com/webpack/loader-runner)这个库，接收一些和当前使用的 loader 相关的信息，来执行这些 loader 的工作。

在任务执行的过程中，subProcess 可能会向 poolWorker 发送 [loadModule](https://webpack.js.org/api/loaders/#thisloadmodule) 消息。在得到回信后，需要处理回调，也就是下面代码片段中的 callback 入参。这里的做法是将处理回调的函数存入 callbackMap 对象中，待收到应答消息后再调用。

```js
{
  loadModule: (request, callback) => {
    callbackMap[nextQuestionId] = (error, result) => callback(error, ...result);
    writeJson({
      type: "loadModule",
      id,
      questionId: nextQuestionId,
      request,
    });
    nextQuestionId += 1;
  };
}
```

poolWorker 在接收到 loadModule 消息后，会利用它 data 中的 loadModule 方法来处理，然后将结果通过发送 result 消息来告知 subProcess。注意这里的 data，其实在前面发送 job 消息的时候已经带过去了，为何 subProcess 还需要 poolWorker 的帮助呢？这是因为消息在传递之前需要经过序列化，data 中的函数无法序列化，也就没法传递过去。

```js
// ...
const { request, questionId } = message;
const { data } = this.jobs[id];
data.loadModule(request, (error, source, sourceMap, module) => {
  this.writeJson({
    type: "result",
    id: questionId,
    error: error
      ? {
          message: error.message,
          details: error.details,
          missing: error.missing,
        }
      : null,
    result: [
      source,
      sourceMap,
      // TODO: Serialize module?
      // module,
    ],
  });
});
```

接着，subProcess 还会发送 resolve 消息，

```js
const resolveWithOptions = (context, request, callback, options) => {
  callbackMap[nextQuestionId] = callback;
  writeJson({
    type: "resolve",
    id,
    questionId: nextQuestionId,
    context,
    request,
    options,
  });
  nextQuestionId += 1;
};
```

当 poolWorker 接收到以后，会利用 [data.getResolve](https://webpack.js.org/api/loaders/#thisgetresolve)或者[data.resolve](https://webpack.js.org/api/loaders/#thisresolve)来处理，然后还是通过 result 消息返回处理后的结果。

最终，subProcess 处理完任务以后，需要将结果分成两部分传给 poolWorker。这是因为结果中可能会有 Buffer 类型的数据无法序列化。

```js
const buffersToSend = [];
// ...
writeJson({
  type: "job",
  id,
  error: err && toErrorObj(err),
  result: {
    result: convertedResult,
    cacheable,
    fileDependencies,
    contextDependencies,
    missingDependencies,
    buildDependencies,
  },
  data: buffersToSend.map((buffer) => buffer.length),
});
buffersToSend.forEach((buffer) => {
  writePipeWrite(buffer);
});
```

因此 poolWorker 在处理收到的 job 消息时，还需要继续去读剩下的 Buffer 类型的数据。接着就要将这两块拼接起来，最终交给外部调用者。至此主流程的工作就完成了。

当所有任务都完成时，或者当主进程退出时，WorkerPool 会关闭所有的 poolWorker，而 poolWorker 会
向 subProcess 发送关闭的消息。也就是一条长度为 0 的空消息。

```js
class PoolWorker {
  // ...
  writeEnd() {
    const lengthBuffer = Buffer.alloc(4);
    lengthBuffer.writeInt32BE(0, 0);
    this.writePipe.write(lengthBuffer);
  }
}
```

当 subProcess 读到空消息时，就会主动关闭自己。

```js
function readNextMessage() {
  // ...
  if (length === 0) {
    // worker should dispose and exit
    dispose();
    return;
  }
  // ...
}
// ...
function dispose() {
  terminate();

  queue.kill();
  process.exit(0);
}
```

thread-loader 还有一个 warmup 的功能

> To prevent the high delay when booting workers it possible to warmup the worker pool.
> This boots the max number of workers in the pool and loads specified modules into the node.js module cache.

```js
function warmup(options, requires) {
  const workerPool = getPool(options);

  workerPool.warmup(requires);
}
```

这个功能会预先创建多个 worker，也就是预先创建多个子进程。

```js
export default class WorkerPool {
  // ...
  warmup(requires) {
    while (this.workers.size < this.numberOfWorkers) {
      this.createWorker().warmup(requires);
    }
  }
}
```

并且这些子进程还会预先加载需要的 package。以此来减少启动时间。

> Each worker is a separate node.js process, which has an overhead of ~600ms. There is also an overhead of inter-process communication.

```js
class PoolWorker {
  // ...
  warmup(requires) {
    this.writeJson({
      type: "warmup",
      requires,
    });
  }
}
// ...
switch (type) {
  // ...
  case "warmup": {
    const { requires } = message;
    // load modules into process
    requires.forEach((r) => require(r)); // eslint-disable-line import/no-dynamic-require, global-require
    break;
  }
  // ...
}
// ...
```

## References

1. [Writing a Loader](https://webpack.js.org/contribute/writing-a-loader/)
2. [thread-loader](https://webpack.js.org/loaders/thread-loader/)
3. [Loader Interface](https://webpack.js.org/api/loaders/)
