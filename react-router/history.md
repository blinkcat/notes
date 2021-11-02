# history

版本 [v5.0.3](https://github.com/remix-run/history/tree/v5.0.3)

封装了[History API](https://developer.mozilla.org/zh-CN/docs/Web/API/History_API)，提供了三种类型的 history。

## Usage

```js
// Create your own history instance.
import { createBrowserHistory } from "history";
let history = createBrowserHistory();

// ... or just import the browser history singleton instance.
import history from "history/browser";

// Alternatively, if you're using hash history import
// the hash history singleton instance.
// import history from 'history/hash';

// Get the current location.
let location = history.location;

// Listen for changes to the current location.
let unlisten = history.listen(({ location, action }) => {
  console.log(action, location.pathname, location.state);
});

// Use push to push a new entry onto the history stack.
history.push("/home", { some: "state" });

// Use replace to replace the current entry in the stack.
history.replace("/logged-in");

// Use back/forward to navigate one entry back or forward.
history.back();

// To stop listening, call the function returned from listen().
unlisten();
```

## How it works

这个库的在设计上参考了策略模式，三种 history 的实现大通小异，且对外暴露了一致的接口。除了 `MemoryHistory`， 其他两个底层都是基于浏览器提供的[History API](https://developer.mozilla.org/zh-CN/docs/Web/API/History_API)

### BrowserHistory

使用了这种 history，每次的路由变化都伴随着 url 的变化。

调用`createBrowserHistory`函数，会返回一个 history 对象。

```ts
export function createBrowserHistory(
  options: BrowserHistoryOptions = {}
): BrowserHistory {
  // ...
  let history: BrowserHistory = {
    get action() {
      // ...
    },
    get location() {
      // ...
    },
    createHref,
    push,
    replace,
    go,
    back() {
      // ...
    },
    forward() {
      // ...
    },
    listen(listener) {
      // ...
    },
    block(blocker) {
      // ...
    },
  };

  return history;
}
```

这个对象里面有我们常用的一些方法，但要注意这个对象和原生的`window.history`还是不同的。

在这个函数运行时，首先会做一些初始化工作

```ts
let { window = document.defaultView! } = options;
let globalHistory = window.history;
// ...
window.addEventListener(PopStateEventType, handlePop);

let action = Action.Pop;
let [index, location] = getIndexAndLocation();
let listeners = createEvents<Listener>();
let blockers = createEvents<Blocker>();

if (index == null) {
  index = 0;
  globalHistory.replaceState({ ...globalHistory.state, idx: index }, "");
}
```

通过 window 拿到浏览器的 history 对象，监听`popstate`事件。

定义`action`，这个 action 代表了 location 变化的类型，一共有三种，初始是 `Action.Pop`

```ts
export enum Action {
  /**
   * A POP indicates a change to an arbitrary index in the history stack, such
   * as a back or forward navigation. It does not describe the direction of the
   * navigation, only that the current index changed.
   *
   * Note: This is the default action for newly created history objects.
   */
  Pop = "POP",

  /**
   * A PUSH indicates a new entry being added to the history stack, such as when
   * a link is clicked and a new page loads. When this happens, all subsequent
   * entries in the stack are lost.
   */
  Push = "PUSH",

  /**
   * A REPLACE indicates the entry at the current index in the history stack
   * being replaced by a new one.
   */
  Replace = "REPLACE",
}
```

然后获取当前的 location 和它对应的 index，注意这里的 location 不是浏览器中的 location。

```ts
export interface Path {
  pathname: Pathname;

  search: Search;

  hash: Hash;
}

export interface Location extends Path {
  state: any;

  key: Key;
}
```

接着是调用`createEvents`定义了两个订阅器，`listeners`和`blockers`。

```ts
function createEvents<F extends Function>(): Events<F> {
  let handlers: F[] = [];

  return {
    get length() {
      return handlers.length;
    },
    push(fn: F) {
      handlers.push(fn);
      return function () {
        handlers = handlers.filter((handler) => handler !== fn);
      };
    },
    call(arg) {
      handlers.forEach((fn) => fn && fn(arg));
    },
  };
}
```

createEvents 在实现上很简洁，容易理解。

回过头继续看 createBrowserHistory 函数，如果当前 location 的序号为空(`index == null`)，那就替换掉，将序号设为 0。

接着就是定义了一堆函数，然后聚集成一个 history 对象返回。

我们可以从几个比较常用的方法看起，先看`history.push`方法

```ts
export type Pathname = string;
export type Search = string;
export type Hash = string;
// ...
export interface Path {
  pathname: Pathname;

  search: Search;

  hash: Hash;
}
// ...
export type PartialPath = Partial<Path>;
// ...
export type To = string | PartialPath;
// ...
function push(to: To, state?: any) {
  let nextAction = Action.Push;
  let nextLocation = getNextLocation(to, state);
  function retry() {
    push(to, state);
  }

  if (allowTx(nextAction, nextLocation, retry)) {
    let [historyState, url] = getHistoryStateAndUrl(nextLocation, index + 1);

    // TODO: Support forced reloading
    // try...catch because iOS limits us to 100 pushState calls :/
    try {
      globalHistory.pushState(historyState, "", url);
    } catch (error) {
      // They are going to lose state here, but there is no real
      // way to warn them about it since the page will refresh...
      window.location.assign(url);
    }

    applyTx(nextAction);
  }
}
```

参数中，to 参数必传，且是一个字符串或者是 path 对象。state 参数选传。

nextAction 定义为`Action.Push`，nextLocation 需要通过计算得出，

```ts
const readOnly: <T extends unknown>(obj: T) => T = __DEV__
  ? (obj) => Object.freeze(obj)
  : (obj) => obj;

// state defaults to `null` because `window.history.state` does
function getNextLocation(to: To, state: any = null): Location {
  return readOnly<Location>({
    pathname: location.pathname,
    hash: "",
    search: "",
    ...(typeof to === "string" ? parsePath(to) : to),
    state,
    key: createKey(),
  });
}
```

getNextLocation 函数中的返回是一个冻结过的对象，这说明 location 语义上是一个 immutable 对象。如果 to 参数是字符串，会先被解析成一个 PartialPath 对象

```ts
export interface Path {
  pathname: Pathname;

  search: Search;

  hash: Hash;
}
// ...
export type PartialPath = Partial<Path>;
```
