# redux-batched-subscribe

版本 [v0.1.6](https://github.com/tappleby/redux-batched-subscribe/tree/0.1.6)

一个 redux storeEnhancer，可以用来对多个 dispatch action 行为提供 batch 功能。

## Usage

```js
import { createStore, applyMiddleware, compose } from "redux";
import { unstable_batchedUpdates as batchedUpdates } from "react-dom";
import { batchedSubscribe } from "redux-batched-subscribe";

const enhancer = compose(
  applyMiddleware(...middleware),
  batchedSubscribe(batchedUpdates)
);

const store = createStore(reducer, intialState, enhancer);
```

使用的时候需要把 redux-batched-subscribe 放到最后。

## How it works

一般来说，storeEnhancer 都有这样的结构，

```ts
(createStore) =>
  (...args) => {
    const store = createStore(...args);
    // ...
    return {
      ...store,
      // do something
    };
  };
```

然后再 redux 的 createStore 内部会这样调用，

```ts
export default function createStore(reducer, preloadedState, enhancer) {
  // ...
  enhancer(createStore)(reducer, preloadedState);
  // ...
}
```

多个 storeEnhancer 需要通过 redux 中的 compose 函数组合成一个函数，假设有三个 enhancer，那么执行顺序是：

```js
compose(a, b, c);
// a -> b -> c -> b -> a
```

所以说 c 是最先接收到原始的 createStore 函数，并且可以最先覆盖或添加里面的一些方法。鉴于这个 enhancer 的功能是 batch 多个 action，所以配置时，需要放到其他 enhancer 的后面。

我们看下代码，

```js
export function batchedSubscribe(batch) {
  if (typeof batch !== "function") {
    throw new Error("Expected batch to be a function.");
  }
  // ...
  return (next) =>
    (...args) => {
      const store = next(...args);
      const subscribeImmediate = store.subscribe;

      function dispatch(...dispatchArgs) {
        const res = store.dispatch(...dispatchArgs);
        notifyListenersBatched();
        return res;
      }

      return {
        ...store,
        dispatch,
        subscribe,
        subscribeImmediate,
      };
    };
}
```

主要看这里对原始 store 的改动，首先是将原始的 subscribe 改名为 subscribeImmediate。然后新建一个 dispatch，在这个 dispatch 的内部，调用原始的 dispatch，然后调用内部存储的回调函数。

下面的代码是上面中间省略的部分，

```js
let currentListeners = [];
let nextListeners = currentListeners;

function ensureCanMutateNextListeners() {
  if (nextListeners === currentListeners) {
    nextListeners = currentListeners.slice();
  }
}

function subscribe(listener) {
  if (typeof listener !== "function") {
    throw new Error("Expected listener to be a function.");
  }

  let isSubscribed = true;

  ensureCanMutateNextListeners();
  nextListeners.push(listener);

  return function unsubscribe() {
    if (!isSubscribed) {
      return;
    }

    isSubscribed = false;

    ensureCanMutateNextListeners();
    const index = nextListeners.indexOf(listener);
    nextListeners.splice(index, 1);
  };
}

function notifyListeners() {
  const listeners = (currentListeners = nextListeners);
  for (let i = 0; i < listeners.length; i++) {
    listeners[i]();
  }
}

function notifyListenersBatched() {
  batch(notifyListeners);
}
```

listeners 执行的时候，被 batch 包裹起来了。其他的和原始的 createStore 内部实现差不多。

只要了解 storeEnhancer 的基本结构，就能很容易地看明白代码。
