# redux-saga

版本[v1.1.3](https://github.com/redux-saga/redux-saga/tree/v1.1.3)

一个 redux 中间件，用来处理各种副作用。`saga`这个名词是从后端的分布式系统中借鉴过来的，但是用在这里又和在分布式系统中的含义不同。具体可以看看这篇[文章](<https://docs.microsoft.com/en-us/previous-versions/msp-n-p/jj591569(v=pandp.10)?redirectedfrom=MSDN>)。在 redux-saga 中，可以任务 saga 就是一个 generator 函数。

以及官方的定义，

> The mental model is that a saga is like a separate thread in your application that's solely responsible for side effects. redux-saga is a redux middleware, which means this thread can be started, paused and cancelled from the main application with normal redux actions, it has access to the full redux application state and it can dispatch redux actions as well.

## Usage

```js
function* watcher() {
  while (true) {
    const action = yield take(ACTION);
    yield fork(worker, action.payload);
  }
}

function* worker(payload) {
  // ... do some stuff
}
```

## How it works

想要深入了解这个库，首先要了解

- redux, redux middleware
- es6 generator function, coroutine
- long running thread

redux-saga 常见的工作模式是监听一个特定的 redux action，然后执行一个任务，完成后继续监听。这种方式很像是有一个额外的线程。但与线程不同的是，这些任务可以根据我们自己定义的逻辑来执行、暂停、取消。因此将这种处于用户态的类似于线程的东西称之为协程。redux-saga 是基于 es6 中的 generator 来实现对这些任务的调度。

我们先来看 redux-saga 是怎样和 redux 连接的，

```js
// ...
import { createStore, applyMiddleware } from "redux";
import createSagaMiddleware from "redux-saga";

// ...
import { helloSaga } from "./sagas";

const sagaMiddleware = createSagaMiddleware();
const store = createStore(reducer, applyMiddleware(sagaMiddleware));
sagaMiddleware.run(helloSaga);

const action = (type) => store.dispatch({ type });

// rest unchanged
```

首先利用一个工厂函数创建一个 sagaMiddleware，它是一个标准的 redux middleware。然后将它加入到 redux 的 store 中，最后再通过它的 run 方法执行 helloSaga 这个 generator 函数。

从源码中可以 sagaMiddleware 的具体定义，它会将每个收到的 action 都通过 channel 发送给自己内部的系统。这个 channel 可以理解为管道，专门用来通信。此外，还会将 runSaga 绑定首个参数得到 boundRunSaga 函数，这个函数就是最终在 sagaMiddleware.run 中调用的函数，执行传入的 generator 函数。

```js
// packages/core/src/internal/middleware.js
import { stdChannel } from "./channel";
import { runSaga } from "./runSaga";

export default function sagaMiddlewareFactory({
  context = {},
  channel = stdChannel(),
  sagaMonitor,
  ...options
} = {}) {
  let boundRunSaga;
  // ...
  function sagaMiddleware({ getState, dispatch }) {
    boundRunSaga = runSaga.bind(null, {
      ...options,
      context,
      channel,
      dispatch,
      getState,
      sagaMonitor,
    });

    return (next) => (action) => {
      // ...
      const result = next(action); // hit reducers
      channel.put(action);
      return result;
    };
  }

  sagaMiddleware.run = (...args) => {
    // ...
    return boundRunSaga(...args);
  };
  // ...
  return sagaMiddleware;
}
```

接下来我们先不急着去看 runSaga 是如何实现的，而是先看看几个内部使用的辅助模块。

### buffer

专供 channel 使用的缓存区，内部利用一个数组实现了环形缓冲区(ringBuffer)。这个缓存区是有固定的容量的，因此当数据超出容量时，也有可选的处理策略。比如 ON_OVERFLOW_EXPAND，当缓冲区满的时候，会自动将容量扩充两倍。再比如说 ON_OVERFLOW_SLIDE，当缓冲区满了的时候，会从头开始写入。之所以采用环形数组的结构，可能是为了简便和效率。因为 js 的数组使用起来很方便，面对这种 FIFO 的使用顺序，如果是环形结构，就能免去删除头部数据的开销。

```js
function ringBuffer(limit = 10, overflowAction) {
  // ...
}
// ...
export const sliding = (limit) => ringBuffer(limit, ON_OVERFLOW_SLIDE);
export const expanding = (initialSize) =>
  ringBuffer(initialSize, ON_OVERFLOW_EXPAND);
```

### matcher

就是一个工厂方法，接收一个 pattern，这个 pattern 可以是通配符、字符串、函数、数组等等。返回一个函数，这个函数接收一个 input 来判断这个 input 和 pattern 是否能匹配上。

```js
export const string = (pattern) => (input) => input.type === String(pattern);
// ...
export default function matcher(pattern) {
  // prettier-ignore
  const matcherCreator = (
      pattern === '*'            ? wildcard
    : is.string(pattern)         ? string
    : is.array(pattern)          ? array
    : is.stringableFunc(pattern) ? string
    : is.func(pattern)           ? predicate
    : is.symbol(pattern)         ? symbol
    : null
  )
  // ...
  return matcherCreator(pattern);
}
```

### scheduler

scheduler 中有一个任务队列，和一个信号量。导出的两个函数都是用来接收任务的，asap 函数会先将任务放入队列中，然后看情况去执行整个队列中的任务。而 immediately 函数则是会先执行传入的任务，然后再尝试去执行队列中的任务。是否能执行队列中的任务就在于这个 semaphore 信号量的值。

```js
const queue = [];
let semaphore = 0;
// ...
export function asap(task) {
  queue.push(task);

  if (!semaphore) {
    suspend();
    flush();
  }
}
// ...
export function immediately(task) {
  try {
    suspend();
    return task();
  } finally {
    flush();
  }
}
// ...
function suspend() {
  semaphore++;
}
// ...
function release() {
  semaphore--;
}
```

你可能会感到奇怪，为什么这里还需要这个锁机制。这是为了保证所有的任务可以按顺序执行。例如当一个 task 中包含了其他的 task 的时候，可以保证当当前 task 执行完后，才会有机会执行这个嵌套的 task。

### channel

redux-saga 中有两种 channel，multicastChannel 和 channel。

multicastChannel 正如它的名字一样，是个广播类型的通道。take 方法用来存储回调函数，同时还接收一个 matcher 函数，也就是我们上面提到的，可以用来判断回调函数是否应该执行。而 put 方法接收 input，这个 input 会作为符合条件的回调函数来执行。

```js
export function multicastChannel() {
  // ...
  return {
    [MULTICAST]: true,
    put(input) {
      // ...
    },
    take(cb, matcher = matchers.wildcard) {
      // ...
    },
    close,
  };
}
```

表面上 multicastChannel 似乎是一个普通的 event emitter，但是还是有很多的不同之处。put 接收的 input 其实是一个 redux action，每次匹配到的回调在执行前会将自己从内部数组中删除。无论 put 还是 take，当管道处于 closed 状态时都不再执行原有的操作。

redux-saga 的中间件中默认使用的是 multicastChannel，借此实现了`take`的功能。

```js
export function stdChannel() {
  const chan = multicastChannel();
  const { put } = chan;
  chan.put = (input) => {
    if (input[SAGA_ACTION]) {
      put(input);
      return;
    }
    asap(() => {
      put(input);
    });
  };
  return chan;
}
```

还有一种就是普通的 channel，它接收一个缓冲区对象。在 takers 队列为空的时候，这个缓存区对象会用来存放 put 方法中未能被及时处理的 input。若是 takers 队列不为空，就会立即取出一个回调函数来执行。同样的，调用 take 方法时，传入回调函数。如果 buffer 非空，就从中取出 input 作为参数执行。如果没有，就将回调函数存入 takers 队列。

```js
export function channel(buffer = buffers.expanding()) {
  // ...
  let takers = [];
  // ...
  return {
    take,
    put,
    flush,
    close,
  };
}
```

在此基础上还有一个 eventChannel，用来将外部的数据源连入 channel。入参 subscribe 是一个函数，它接收一个函数，用来将 input 传入到内部的 channel 中。

```js
export function eventChannel(subscribe, buffer = buffers.none()) {
  // ...
  let unsubscribe;

  const chan = channel(buffer);
  // ...
  unsubscribe = subscribe((input) => {
    if (isEnd(input)) {
      close();
      return;
    }
    chan.put(input);
  });
  // ...
  return {
    take: chan.take,
    flush: chan.flush,
    close,
  };
}
```

### runSaga

runSaga 是执行所有 saga 的入口。在 sagaMiddleware.run 中就是直接调用了这个函数。这里传入的 saga 是一个[主 saga](https://redux-saga.js.org/docs/advanced/RootSaga)，用来启动所有的子 saga，一般来说这些子 saga 都是并发的，互不干扰。执行 saga 后得到一个 iterator 对象。接着是这个 env 对象，注意里面的 dispatch 函数被包装过。这使得在 put effect 中发送的 action 都会带有 SAGA_ACTION 属性。这就和我们上面提到的 stdChannel 函数对应起来了。如果它接收的 action 没有这个属性，它就会用 asap 函数包裹一个执行 put 函数的函数。保证 runSaga 中的 immediately 中的任务先完成。这是为了保证所有的子 saga 初始化完毕，然后再接收 action。

```js
export function runSaga(
  {
    channel = stdChannel(),
    dispatch,
    getState,
    context = {},
    sagaMonitor,
    effectMiddlewares,
    onError = logError,
  },
  saga,
  ...args
) {
  // ...
  const iterator = saga(...args);
  // ...
  const env = {
    channel,
    dispatch: wrapSagaDispatch(dispatch),
    getState,
    sagaMonitor,
    onError,
    finalizeRunEffect,
  };
  return immediately(() => {
    const task = proc(
      env,
      iterator,
      context,
      effectId,
      getMetaInfo(saga),
      /* isRoot */ true,
      undefined
    );
    // ...
    return task;
  });
}
```

在 immediately 包裹的函数中，调用了 proc 函数来一步步地执行 saga。proc 函数执行后返回一个 task 对象，这个对象我们后面会提到。

### proc

在 proc 函数中，有个 generator 自动执行器，next 函数。这个函数的第二个入参代表执行的过程中是否有错误发生。next 函数会不断的递归执行，直到 generator 函数执行完毕或者中途发生错误或者执行被取消了。如果在递归的过程中，得到的中间结果是 Promise 或是 iterator 对象、[effect](https://redux-saga.js.org/docs/basics/Effect) 对象。就需要对它们进行特殊特殊处理。

```js
export default function proc(
  // ...
  cont
) {
  // ...
  next.cancel = noop;
  // ...
  const mainTask = { meta, cancel: cancelMain, status: RUNNING };
  // ...
  const task = newTask(
    // ...
    cont
  );
}
// ...
function cancelMain() {
  if (mainTask.status === RUNNING) {
    mainTask.status = CANCELLED;
    next(TASK_CANCEL);
  }
}
// ...
if (cont) {
  cont.cancel = task.cancel;
}
next();
return task;
function next(arg, isErr) {
  // ...
}
```

这段逻辑在 runEffect 函数中，可以看到这三种情况下都要把 currCb 传入。以保证在处理完它们以后，调用此函数可以继续 next 函数的执行。

```js
function runEffect(effect, effectId, currCb) {
  if (is.promise(effect)) {
    resolvePromise(effect, currCb);
  } else if (is.iterator(effect)) {
    // resolve iterator
    proc(env, effect, task.context, effectId, meta, /* isRoot */ false, currCb);
  } else if (effect && effect[IO]) {
    const effectRunner = effectRunnerMap[effect.type];
    effectRunner(env, effect.payload, currCb, executingContext);
  } else {
    // anything else returned as is
    currCb(effect);
  }
}
```

因为 redux-saga 支持任务的错误向上传递，取消的信号向下传递。所以 next 函数的入参中有个 isErr，表示执行过程中是否出现错误，currCb 的函数签名和 next 函数是一致的。为了支持取消任务，next 函数和 currCb 函数在每次执行任务过程中都会被赋值一个 cancel 函数，这个函数用来取消子任务。

### task

调用 proc 函数会返回一个 task 对象，这个 task 包含了描述这个 generator 的主任务以及和它关联的通过`fork`生成的任务。

```js
const task = newTask(
  env,
  mainTask,
  parentContext,
  parentEffectId,
  meta,
  isRoot,
  cont
);
```

在 newTask 函数中会使用一个队列来管理所有的任务。这个队列通过 forkQueue 函数生成。当所有子任务都完成的时候，这个任务才算完成。当这个任务取消的时候，所有的子任务都会取消。只要有一个子任务出错，其他所有的任务都会被取消，并且这个任务也被认为出错了。

```js
export default function newTask(
  // ...
  cont = noop
) {
  let status = RUNNING;
  // ...
  const queue = forkQueue(
    mainTask,
    function onAbort() {
      cancelledDueToErrorTasks.push(
        ...queue.getTasks().map((t) => t.meta.name)
      );
    },
    end
  );
  // ...
  function cancel() {
    // ...
  }
  // ...
  function end(result, isErr) {
    // ...
  }
  // ...
  const task = {
    [TASK]: true,
    // ...
    queue,
    // ...
    cancel,
    cont,
    end,
    // ...
    isRunning: () => status === RUNNING,
    // ...
  };
  return task;
}
```

### effects

redux-saga 提供了相当多的 [effects](https://redux-saga.js.org/docs/api#effect-creators)，这些 effects 组合起来使用可以处理绝大部分业务了。

例如下面的 demo，在使用时 effects 都是跟在 yield 关键字后面。因为每个 effect 的调用结果都是一个普通的 object 对象，这个对象描述了这个 effect。

```js
import { fork, call, take, put } from "redux-saga/effects";
import Api from "...";

function* authorize(user, password) {
  try {
    const token = yield call(Api.authorize, user, password);
    yield put({ type: "LOGIN_SUCCESS", token });
    yield call(Api.storeItem, { token });
  } catch (error) {
    yield put({ type: "LOGIN_ERROR", error });
  }
}

function* loginFlow() {
  while (true) {
    const { user, password } = yield take("LOGIN_REQUEST");
    yield fork(authorize, user, password);
    yield take(["LOGOUT", "LOGIN_ERROR"]);
    yield call(Api.clearItem, "token");
  }
}
```

在上面提到的 runEffect 函数中，有处理这些 effects 的逻辑。

```js
if (effect && effect[IO]) {
  const effectRunner = effectRunnerMap[effect.type];
  effectRunner(env, effect.payload, currCb, executingContext);
}
```

我们可以挑几个常见的 effect 来看看，比如说 take、put、call、fork 等。

因为 take 可以接收的参数有很多种，我们先看最常见的，接收一个 action.type。

在 take 中调用了 makeEffect，生成一个 object 对象。

```js
// packages/core/src/internal/io.js
const makeEffect = (type, payload) => ({
  [IO]: true,
  // this property makes all/race distinguishable in generic manner from other effects
  // currently it's not used at runtime at all but it's here to satisfy type systems
  combinator: false,
  type,
  payload,
});
// ...
export function take(patternOrChannel = "*", multicastPattern) {
  // ...
  if (is.pattern(patternOrChannel)) {
    return makeEffect(effectTypes.TAKE, { pattern: patternOrChannel });
  }
  // ...
}
```

根据这个对象的 type 属性，从 effectRunnerMap 中找到对应的处理函数。承载我们业务逻辑的 generator 函数实际上在执行的时候只产生了这些对象，然后在 redux 中间件中才会进行实际的处理。这样的设计使得我们的 generator 函数可以成为一个纯函数，方便于单元测试。

```js
// packages/core/src/internal/effectRunnerMap.js
const effectRunnerMap = {
  [effectTypes.TAKE]: runTakeEffect,
  // ...
};
// ...
function runTakeEffect(env, { channel = env.channel, pattern, maybe }, cb) {
  const takeCb = (input) => {
    if (input instanceof Error) {
      cb(input, true);
      return;
    }
    if (isEnd(input) && !maybe) {
      cb(TERMINATE);
      return;
    }
    cb(input);
  };
  try {
    channel.take(takeCb, is.notUndef(pattern) ? matcher(pattern) : null);
  } catch (err) {
    cb(err, true);
    return;
  }
  cb.cancel = takeCb.cancel;
}
```

runTakeEffect 函数中的 env 就是我们在 runSaga 函数中见到的，

```js
const env = {
  channel,
  dispatch: wrapSagaDispatch(dispatch),
  getState,
  sagaMonitor,
  onError,
  finalizeRunEffect,
};
```

由于我们只看入参是 action.type 的情况，所以这里的 channel 是 env 提供的 multicastChannel 的实例。而调用入参 cb 可以间接调用前面提到的 next 函数，让主任务中的 generator 函数继续执行。takeCb 对 cb 函数进行了简单的封装，区分了出错的情况，取消任务的情况以及正常的情况。channel.take 操作将 takeCb 放入内部的数组中，等待合适的 action 来触发这个它。触发的地方就在最上面的 sagaMiddlewareFactory 函数里的 redux 中间件逻辑中。每次有 action 发送的时候，都会调用 channel.put(action)。

接着我们再看看 call effect，这个 effect 会调用传入的函数，并阻塞后面的操作。

```js
function getFnCallDescriptor(fnDescriptor, args) {
  // ...
  return { context, fn, args };
}
// ...
export function call(fnDescriptor, ...args) {
  // ...
  return makeEffect(effectTypes.CALL, getFnCallDescriptor(fnDescriptor, args));
}
```

在 runCallEffect 中，传入 call 的函数会被执行，根据返回值的类型决定了不同的处理方式。如果返回值是 iterator 对象，说明传入 call 的函数是一个 generator 函数。在这里我们继续调用 proc 函数处理这个 iterator 对象。注意我们将 cb 也传入了 proc，cb 函数连接了两个 generator 函数，以支持执行权交换、错误传递、取消信号传递。

```js
function runCallEffect(env, { context, fn, args }, cb, { task }) {
  // catch synchronous failures; see #152
  try {
    const result = fn.apply(context, args);

    if (is.promise(result)) {
      resolvePromise(result, cb);
      return;
    }

    if (is.iterator(result)) {
      // resolve iterator
      proc(
        env,
        result,
        task.context,
        currentEffectId,
        getMetaInfo(fn),
        /* isRoot */ false,
        cb
      );
      return;
    }

    cb(result);
  } catch (error) {
    cb(error, true);
  }
}
```

前面说了 call 的执行是会阻塞后面的操作的，而 fork 不会。

```js
export function fork(fnDescriptor, ...args) {
  // ...
  return makeEffect(effectTypes.FORK, getFnCallDescriptor(fnDescriptor, args));
}
```

所以 runForkEffect 的实现也略微复杂。不管传入的函数执行后返回的是什么，都要将这个结果包装成一个 taskIterator，传入 proc 函数中，得到一个子 task。注意这次调用 proc 最后一个参数是 undefined。因为 fork 的任务是和主任务是并发执行的关系，不会阻塞主任务继续运行。但是子任务还是要依附于主任务，所以后面的操作会把子任务加入到主任务内部的队列中。然后继续执行主任务。

```js
function runForkEffect(
  env,
  { context, fn, args, detached },
  cb,
  { task: parent }
) {
  const taskIterator = createTaskIterator({ context, fn, args });
  const meta = getIteratorMetaInfo(taskIterator, fn);

  immediately(() => {
    const child = proc(
      env,
      taskIterator,
      parent.context,
      currentEffectId,
      meta,
      detached,
      undefined
    );

    if (detached) {
      cb(child);
    } else {
      if (child.isRunning()) {
        parent.queue.addTask(child);
        cb(child);
      } else if (child.isAborted()) {
        parent.queue.abort(child.error());
      } else {
        cb(child);
      }
    }
  });
  // Fork effects are non cancellables
}
```

接着就是常用的 put，最常见的用法是用来发送 action。

```js
export function put(channel, action) {
  // ...
  if (is.undef(action)) {
    action = channel;
    // `undefined` instead of `null` to make default parameter work
    channel = undefined;
  }
  return makeEffect(effectTypes.PUT, { channel, action });
}
```

在 runPutEffect 中，整个过程被包裹在 asap 函数中。这样 put 的 effect 可能并不会立即发生。这应该也是要确保所有的 fork 产生的任务都能初始化完毕，再依次执行 put effect。传给 put 的 action 最终会通过 redux 的 dispatch 发送出去。

```js
function runPutEffect(env, { channel, action, resolve }, cb) {
  /**
   Schedule the put in case another saga is holding a lock.
   The put will be executed atomically. ie nested puts will execute after
   this put has terminated.
   **/
  asap(() => {
    let result;
    try {
      result = (channel ? channel.put : env.dispatch)(action);
    } catch (error) {
      cb(error, true);
      return;
    }

    if (resolve && is.promise(result)) {
      resolvePromise(result, cb);
    } else {
      cb(result);
    }
  });
  // Put effects are non cancellables
}
```

select 是另一个常用的且和 redux 强关联的 effect，

```js
export const identity = (v) => v;
// ...
export function select(selector = identity, ...args) {
  // ...
  return makeEffect(effectTypes.SELECT, { selector, args });
}
```

runSelectEffect 的实现很简单，就是利用你传入的 selector 函数传入 redux 的 state 来计算出你想要的值。这里最好能使用 reselect 这样的库来缓存结果，减少多次调用中重复计算的次数。

```js
function runSelectEffect(env, { selector, args }, cb) {
  try {
    const state = selector(env.getState(), ...args);
    cb(state);
  } catch (error) {
    cb(error, true);
  }
}
```

前面提到的都是实现了单个功能的 effect，但是经常会有多个 effects 需要并行或是竞争的情况。这个时候就需要 all、race effect 了。

我们先看 all，

```js
export function all(effects) {
  const eff = makeEffect(effectTypes.ALL, effects);
  eff.combinator = true;
  return eff;
}
```

runAllEffect 会将多个 effects 转成对应的任务，并发执行，最后再将结果汇总返回。

```js
function runAllEffect(env, effects, cb, { digestEffect }) {
  const effectId = currentEffectId;
  const keys = Object.keys(effects);
  if (keys.length === 0) {
    cb(is.array(effects) ? [] : {});
    return;
  }

  const childCallbacks = createAllStyleChildCallbacks(effects, cb);
  keys.forEach((key) => {
    digestEffect(effects[key], effectId, childCallbacks[key], key);
  });
}
```

和 all 类似，race 也是作用于一组 effects。只不过它只取最快执行完毕的 effect 的结果。

```js
export function race(effects) {
  const eff = makeEffect(effectTypes.RACE, effects);
  eff.combinator = true;
  return eff;
}
```

runRaceEffect 和 runAllEffect 最大的区别就是只需要有一个 effect 有结果就行。

```js
function runRaceEffect(env, effects, cb, { digestEffect }) {
  // ...
}
```

除此之外，还会有一些进阶的用法，比如可以令连续发生的请求进入队列，然后顺序执行。

```js
import { take, actionChannel, call, ... } from 'redux-saga/effects'

function* watchRequests() {
  // 1- Create a channel for request actions
  const requestChan = yield actionChannel('REQUEST')
  while (true) {
    // 2- take from the channel
    const {payload} = yield take(requestChan)
    // 3- Note that we're using a blocking call
    yield call(handleRequest, payload)
  }
}

function* handleRequest(payload) { ... }
```

actionChannel 也是一种特殊的 effect，可以缓存接下来的请求。它的两个入参分别表示执行的条件和缓冲区实例。

```js
export function actionChannel(pattern, buffer) {
  // ...
  return makeEffect(effectTypes.ACTION_CHANNEL, { pattern, buffer });
}
```

runChannelEffect 中会将包装后的消费者 taker 函数加入到 env.channel 中，这个 env.channel 就是我们前面提到的 multiChannel。在 taker 函数中，只要传入的 action 不代表结束，就会重新将自己加入 env.channel 中。这是因为在 multiChannel 中，每个消费者被触发后都会从内部的数组中删除，这个行为是为了同时满足 take effect 的单次阻塞和一直阻塞(take 在无限循环中阻塞)。接着执行的 chan.put 会将 action 传入我们自己创建的 channel 中。这里还将 channel 的 close 函数中执行了 taker.cancel 函数，保证在关闭 channel 后能将 taker 函数从 env.channel 的内部删除。最后这个 channel 通过 cb 函数返回主任务。

```js
function runChannelEffect(env, { pattern, buffer }, cb) {
  const chan = channel(buffer);
  const match = matcher(pattern);

  const taker = (action) => {
    if (!isEnd(action)) {
      env.channel.take(taker, match);
    }
    chan.put(action);
  };

  const { close } = chan;

  chan.close = () => {
    taker.cancel();
    close();
  };

  env.channel.take(taker, match);
  cb(chan);
}
```

从上面的 demo 可以看到，这个返回的 channel 会作为 take effect 的参数。我们再次看向 runTakeEffect 函数的代码。这时候我们的 channel 获取到的 takeCb 内部是调用了 cb，将执行权交还给了主任务。

```js
function runTakeEffect(env, { channel = env.channel, pattern, maybe }, cb) {
  const takeCb = (input) => {
    if (input instanceof Error) {
      cb(input, true);
      return;
    }
    if (isEnd(input) && !maybe) {
      cb(TERMINATE);
      return;
    }
    cb(input);
  };
  try {
    channel.take(takeCb, is.notUndef(pattern) ? matcher(pattern) : null);
  } catch (err) {
    cb(err, true);
    return;
  }
  cb.cancel = takeCb.cancel;
}
```

actionChannel 本质上还是连接的 redux 的数据流，如果我们需要自定义外部数据流，就需要使用 eventChannel。它的实现依赖于 channel，但不是一个 effect。在构造 eventChannel 时，我们需要将外部数据源的数据通过 channel 的 put 函数，也就是下面例子中的 emitter，传入到 channel 内部。最后返回一个取消订阅的函数。

```js
import { take, put, call } from "redux-saga/effects";
import { eventChannel, END } from "redux-saga";

function countdown(secs) {
  return eventChannel((emitter) => {
    const iv = setInterval(() => {
      secs -= 1;
      if (secs > 0) {
        emitter(secs);
      } else {
        // this causes the channel to close
        emitter(END);
      }
    }, 1000);
    // The subscriber must return an unsubscribe function
    return () => {
      clearInterval(iv);
    };
  });
}

export function* saga() {
  const chan = yield call(countdown, value);
  try {
    while (true) {
      // take(END) will cause the saga to terminate by jumping to the finally block
      let seconds = yield take(chan);
      console.log(`countdown: ${seconds}`);
    }
  } finally {
    console.log("countdown terminated");
  }
}
```

接着，我们在使用的时候还是把 eventChannel 放入 take 中获取传出来的数据。take 的内部逻辑我们在 actionChannel 哪里提过了。

如果我们仔细看，可以发现上面两种使用 channel 的方式大同小异。都是利用 take effect 接收信息，唯一的差别在于在不同的地方调用 channel.put 发送消息。因此 channel 也可以用于不同的 saga 之间传递信息。

看下面的例子，watchRequests 和 3 个 handleRequest 之间共享了一个 channel。handleRequest 中使用了 take，因此在获取到 channel 中的消息之前都会处于阻塞状态。但是当 watchRequests 中向 channel 传递消息时，handleRequest 在收到消息后变会执行下面的工作。

```js
import { channel } from "redux-saga";
import { take, fork } from "redux-saga/effects";

function* watchRequests() {
  // create a channel to queue incoming requests
  const chan = yield call(channel);

  // create 3 worker 'threads'
  for (var i = 0; i < 3; i++) {
    yield fork(handleRequest, chan);
  }

  while (true) {
    const { payload } = yield take("REQUEST");
    yield put(chan, payload);
  }
}

function* handleRequest(chan) {
  while (true) {
    const payload = yield take(chan);
    // process the request
  }
}
```

### helpers

除了上面提到的那些 effects、channel，redux-saga 还提供了一些便捷的函数，减少我们重复的工作。比如说 takeEvery、takeLatest 等。

我们先来看 takeEvery 的使用方式，当接收到多个信号时，会并行开启多个任务。

```js
import { takeEvery } from `redux-saga/effects`

function* fetchUser(action) {
// ...
}

function* watchFetchUser() {
  yield takeEvery('USER_REQUESTED', fetchUser)
}
```

相当于下面的写法，

```js
import { take, fork } from "redux-saga/effects";

function* watchFetchUser() {
  while (true) {
    const action = yield take("USER_REQUESTED");
    yield fork(fetchUser, action);
  }
}
```

再看看 takeEvery 的实现，在内部定义了一些状态对象，然后调用 fsmIterator 后返回。这个 fsm 前缀全称是 finite state machine(有限状态机)。最终返回的是一个 iterator 对象，我们定义好的状态在这个返回的 iterator 对象的调用中不断轮转。

```js
export default function takeEvery(patternOrChannel, worker, ...args) {
  const yTake = { done: false, value: take(patternOrChannel) };
  const yFork = (ac) => ({ done: false, value: fork(worker, ...args, ac) });

  let action,
    setAction = (ac) => (action = ac);

  return fsmIterator(
    {
      q1() {
        return { nextState: "q2", effect: yTake, stateUpdater: setAction };
      },
      q2() {
        return { nextState: "q1", effect: yFork(action) };
      },
    },
    "q1",
    `takeEvery(${safeName(patternOrChannel)}, ${worker.name})`
  );
}
```

takeLatest 也是同样的套路，只不过在逻辑上需要取消之前的请求，只保留最新的请求。所以多了一个状态

```js
export default function takeLatest(patternOrChannel, worker, ...args) {
  const yTake = { done: false, value: take(patternOrChannel) };
  const yFork = (ac) => ({ done: false, value: fork(worker, ...args, ac) });
  const yCancel = (task) => ({ done: false, value: cancel(task) });

  let task, action;
  const setTask = (t) => (task = t);
  const setAction = (ac) => (action = ac);

  return fsmIterator(
    {
      q1() {
        return { nextState: "q2", effect: yTake, stateUpdater: setAction };
      },
      q2() {
        return task
          ? { nextState: "q3", effect: yCancel(task) }
          : { nextState: "q1", effect: yFork(action), stateUpdater: setTask };
      },
      q3() {
        return {
          nextState: "q1",
          effect: yFork(action),
          stateUpdater: setTask,
        };
      },
    },
    "q1",
    `takeLatest(${safeName(patternOrChannel)}, ${worker.name})`
  );
}
```

我们再来看些复杂的例子，debounce 和 throttle。

debounce 可以确保在一段时间内事件连续触发时，只响应最后的事件。

```js
import { call, put, debounce } from `redux-saga/effects`

function* fetchAutocomplete(action) {
  const autocompleteProposals = yield call(Api.fetchAutocomplete, action.text)
  yield put({type: 'FETCHED_AUTOCOMPLETE_PROPOSALS', proposals: autocompleteProposals})
}

function* debounceAutocomplete() {
  yield debounce(1000, 'FETCH_AUTOCOMPLETE', fetchAutocomplete)
}
```

实际上相当于这样写，

```js
function* debounceAutocomplete() {
  while (true) {
    let action = yield take("FETCH_AUTOCOMPLETE");
    while (true) {
      const { debounced, lastestAction } = yield race({
        debounced: delay(1000),
        latestAction: take("FETCH_AUTOCOMPLETE"),
      });

      if (debounced) {
        yield fork(fetchAutocomplete, action);
        break;
      }

      action = latestAction;
    }
  }
}
```

有了这个参考，debounce 的实现代码会更容易看懂。总共三个状态，每个状态对应的操作其实都和上面的代码能对应起来。

```js
export default function debounceHelper(
  delayLength,
  patternOrChannel,
  worker,
  ...args
) {
  let action, raceOutput;

  const yTake = { done: false, value: take(patternOrChannel) };
  const yRace = {
    done: false,
    value: race({
      action: take(patternOrChannel),
      debounce: delay(delayLength),
    }),
  };
  const yFork = (ac) => ({ done: false, value: fork(worker, ...args, ac) });
  const yNoop = (value) => ({ done: false, value });

  const setAction = (ac) => (action = ac);
  const setRaceOutput = (ro) => (raceOutput = ro);

  return fsmIterator(
    {
      q1() {
        return { nextState: "q2", effect: yTake, stateUpdater: setAction };
      },
      q2() {
        return { nextState: "q3", effect: yRace, stateUpdater: setRaceOutput };
      },
      q3() {
        return raceOutput.debounce
          ? { nextState: "q1", effect: yFork(action) }
          : {
              nextState: "q2",
              effect: yNoop(raceOutput.action),
              stateUpdater: setAction,
            };
      },
    },
    "q1",
    `debounce(${safeName(patternOrChannel)}, ${worker.name})`
  );
}
```

throttle 可以确保一段时间内只响应一次事件。

```js
import { call, put, throttle } from `redux-saga/effects`

function* fetchAutocomplete(action) {
  const autocompleteProposals = yield call(Api.fetchAutocomplete, action.text)
  yield put({type: 'FETCHED_AUTOCOMPLETE_PROPOSALS', proposals: autocompleteProposals})
}

function* throttleAutocomplete() {
  yield throttle(1000, 'FETCH_AUTOCOMPLETE', fetchAutocomplete)
}
```

和下面写法等价，

```js
function* throttleAutocomplete() {
  const throttleChannel = yield actionChannel(
    "FETCH_AUTOCOMPLETE",
    buffers.sliding(1)
  );

  while (true) {
    const action = yield take(throttleChannel);
    yield fork(fetchAutocomplete, action);
    yield delay(1000);
  }
}
```

而 throttle 的实现也是有四个状态，对应了这四部操作。

最后一个常用的方法是 retry,

```js
import { put, retry } from "redux-saga/effects";
import { request } from "some-api";

function* retrySaga(data) {
  try {
    const SECOND = 1000;
    const response = yield retry(3, 10 * SECOND, request, data);
    yield put({ type: "REQUEST_SUCCESS", payload: response });
  } catch (error) {
    yield put({ type: "REQUEST_FAIL", payload: { error } });
  }
}
```

由于 retry 多了个处理错误的步骤，所以它的实现和前面有些不同。在 q1 状态中，有个 errorState 指向 q10，用来实现重试的逻辑。当重试了指定次数后还是失败时，错误就会抛出，被 catch 部分捕获。

```js
export default function retry(maxTries, delayLength, fn, ...args) {
  let counter = maxTries;

  const yCall = { done: false, value: call(fn, ...args) };
  const yDelay = { done: false, value: delay(delayLength) };

  return fsmIterator(
    {
      q1() {
        return { nextState: "q2", effect: yCall, errorState: "q10" };
      },
      q2() {
        return { nextState: qEnd };
      },
      q10(error) {
        counter -= 1;
        if (counter <= 0) {
          throw error;
        }
        return { nextState: "q1", effect: yDelay };
      },
    },
    "q1",
    `retry(${fn.name})`
  );
}
```

## Monitor

由于 redux-saga 高度封装抽象，使得我们可以利用各种 effects 很轻松地处理业务流程。但是也因为这种密封性，使得在流程中不利于观察内部流程。因此在 sagaMiddlewareFactory 函数中可以传入一个 [sagaMonitor](https://redux-saga.js.org/docs/api#sagamonitor) 对象来监听各种关键事件。

首先在每次有 redux ation 发出时，会触发 actionDispatched 事件。

```js
if (sagaMonitor && sagaMonitor.actionDispatched) {
  sagaMonitor.actionDispatched(action);
}
```

在所有的 saga 启动之前，会触发 rootSagaStarted 事件。

```js
sagaMonitor.rootSagaStarted({ effectId, saga, args });
```

当有 effect 发生时，触发 effectTriggered 事件。

```js
env.sagaMonitor &&
  env.sagaMonitor.effectTriggered({ effectId, parentEffectId, label, effect });
```

当 effect 被成功处理时，触发 effectResolved 事件。如果是 fork、spawn effect，第二个参数就是 task 对象。

```js
env.sagaMonitor.effectResolved(effectId, res);
// ...
sagaMonitor.effectResolved(effectId, task);
```

当 effect 处理失败时，触发 effectRejected 事件。

```js
env.sagaMonitor.effectRejected(effectId, res);
```

当 effect 被取消时，触发 effectCancelled 事件。

```js
env.sagaMonitor && env.sagaMonitor.effectCancelled(effectId);
```

通过监听这些事件，我们可以大致了解 redux-saga 在运行时的内部情况。

## Error Stack

当有错误抛出时，如果能将详细的错误信息打印出来对于开发和调试都是相当友好的。

如果再加上 babel-plugin-redux-saga 这个 babel 插件，就可以提供更完整的位置信息。默认 stack 如下，

```js
The above error occurred in task throwAnErrorSaga
    created by errorInCallAsyncSaga
    created by takeEvery(ACTION_ERROR_IN_CALL_ASYNC, errorInCallAsyncSaga)
    created by rootSaga
```

使用了 babel 插件后，可以看到每行错误后面跟着对应文件的位置。

```js
The above error occurred in task throwAnErrorSaga  src/sagas/index.js?16
    created by errorInCallAsyncSaga  src/sagas/index.js?25
    created by takeEvery(ACTION_ERROR_IN_CALL_ASYNC, errorInCallAsyncSaga)
    created by rootSaga  src/sagas/index.js?78
```

这是因为在 babel 编译代码的过程中，插件会将我们的 saga，也就是 generator 函数加上 filename 和 lineNumber 信息，例如

```js
// input
function* effectHandler() {}
// output
function * effectHandler(){}
	Object.defineProperty(effectHandler, "@@redux-saga/LOCATION", {
        value: { fileName: ..., lineNumber: ... }
    })
```

包括 yield 后的 effect，也加上这些信息。

```js
// input
yield call(smthelse)
// output
yield Object.defineProperty(call(smthelse), "@@redux-saga/LOCATION", {
    	value: { fileName: ..., lineNumber: ... }
	})
```

当 saga 和 effect 对象上带有这些位置信息后，我们只需要用 getMetaInfo 就可以取出这些信息。

```js
export function getMetaInfo(fn) {
  return {
    name: fn.name || "anonymous",
    location: getLocation(fn),
  };
}

export function getLocation(instrumented) {
  return instrumented[SAGA_LOCATION];
}
```

为了能够记录下 saga 出错时的调用栈，我们得手动将这些信息保存起来。

```js
let crashedEffect = null;
const sagaStack = [];

export const addSagaFrame = (frame) => {
  frame.crashedEffect = crashedEffect;
  sagaStack.push(frame);
};

export const clear = () => {
  crashedEffect = null;
  sagaStack.length = 0;
};
// ...
export const setCrashedEffect = (effect) => {
  crashedEffect = effect;
};
```

当错误发生时，会将这个错误向上传递的过程中被记录下来，直到遇到主任务时，也就是 task.isRoot 为 true 时将栈交给 env.onError 执行，默认是将这些信息在控制台中打印出来。然后清空栈信息。

```js
// packages/core/src/internal/proc.js
function digestEffect(effect, parentEffectId, cb, label = "") {
  // ...
  function currCb(res, isErr) {
    // ...
    if (isErr) {
      sagaError.setCrashedEffect(effect);
    }
    // ...
  }
  // ...
}

// packages/core/src/internal/newTask.js
export default function newTask(
  // ...
  meta,
  isRoot,
  cont = noop
) {
  // ...
  function end(result, isErr) {
    if (!isErr) {
      // ...
    } else {
      status = ABORTED;
      sagaError.addSagaFrame({
        meta,
        cancelledTasks: cancelledDueToErrorTasks,
      });

      if (task.isRoot) {
        const sagaStack = sagaError.toString();
        // ...
        sagaError.clear();
        env.onError(result, { sagaStack });
      }
    }
  }
  // ...
}
```

## Conclusion

整个 redux-saga 的工作流程大致如此。实际上这个库的实现相当精密，紧凑，还有好多细节我们没讲到。比如说它的整体构思，将多个业务流程抽象成一个常驻运行的线程、任务。这些任务之间并不孤立，可以通过 channel 通信。主任务和子任务之间像一棵树一样，因此也容易实现错误向上传递和取消信号向下传递。再比如为了方便测试，每个 effect 都只是一个普通的对象，需要通过中间件来解释执行。这样就避免了副作用代码难以 mock 的问题，同时 generator 自身的特性允许暂停函数或向函数中传递任意值，更加方便了测试工作。
