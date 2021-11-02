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

getNextLocation 函数中的返回是一个冻结过的对象，这说明 location 语义上是一个 immutable 对象。如果 to 参数是字符串，会先被解析成一个 PartialPath 对象。

```ts
export interface Path {
  pathname: Pathname;

  search: Search;

  hash: Hash;
}
// ...
export type PartialPath = Partial<Path>;
```

key 值是一个临时生成的随机值。然后将这些值合并成 location 对象。

```ts
function createKey() {
  return Math.random().toString(36).substr(2, 8);
}
// ...
export interface Location extends Path {
  state: any;
  key: Key;
}
```

再回到 push 函数，将这些准备好的值传入`allowTx`函数，判断当前是否可以进行 push 操作，

```ts
function allowTx(action: Action, location: Location, retry: () => void) {
  return (
    !blockers.length || (blockers.call({ action, location, retry }), false)
  );
}
```

因为 blockers 对象就是我们前面提到的通过 createEvents 函数创建的对象，里面存储着一些函数，用来决定当前的路由是否可以跳转。

如果可以，调用 getHistoryStateAndUrl 函数，获取 state 和 url。这里需要传入前面计算得到的 nextLocation，已经当前 location 的序号加 1。

```ts
function getHistoryStateAndUrl(
  nextLocation: Location,
  index: number
): [HistoryState, string] {
  return [
    {
      usr: nextLocation.state,
      key: nextLocation.key,
      idx: index,
    },
    createHref(nextLocation),
  ];
}
// ...
function createHref(to: To) {
  return typeof to === "string" ? to : createPath(to);
}
// ...
export function createPath({
  pathname = "/",
  search = "",
  hash = "",
}: PartialPath) {
  return pathname + search + hash;
}
```

在 getHistoryStateAndUrl 函数中，生成新的 state 和 url。调用原生 history 对象的 pushState 方法，将当前路由变成 url，并存储 state。最后调用 applyTx 函数，更新 action，index，location 后，执行 listeners 对象中的回调。

```ts
function applyTx(nextAction: Action) {
  action = nextAction;
  [index, location] = getIndexAndLocation();
  listeners.call({ action, location });
}
```

这些回调通过 history.listen 方式传入

```ts
{
  // ...
  listen(listener) {
    return listeners.push(listener);
  }
  // ...
}
```

而前面提到的 blockers 对象的中回调则是通过 history.block 的方式传入

```ts
block(blocker) {
  let unblock = blockers.push(blocker);

  if (blockers.length === 1) {
    window.addEventListener(BeforeUnloadEventType, promptBeforeUnload);
  }

  return function () {
    unblock();

    // Remove the beforeunload listener so the document may
    // still be salvageable in the pagehide event.
    // See https://html.spec.whatwg.org/#unloading-documents
    if (!blockers.length) {
      window.removeEventListener(BeforeUnloadEventType, promptBeforeUnload);
    }
  };
}
// ...
function promptBeforeUnload(event: BeforeUnloadEvent) {
  // Cancel the event.
  event.preventDefault();
  // Chrome (and legacy IE) requires returnValue to be set.
  event.returnValue = '';
}
```

在这个函数中，还会监听 beforeunload 事件。

其他常用的方法，如 history.replace 和 history.push 的实现差不多，只不过调用的是 globalHistory.replaceState。history.go 则是直接调用了 globalHistory.go。

现在来看下前面提到的用来监听`popstate`事件的方法，

```ts
let blockedPopTx: Transition | null = null;
function handlePop() {
  if (blockedPopTx) {
    blockers.call(blockedPopTx);
    blockedPopTx = null;
  } else {
    let nextAction = Action.Pop;
    let [nextIndex, nextLocation] = getIndexAndLocation();

    if (blockers.length) {
      if (nextIndex != null) {
        let delta = index - nextIndex;
        if (delta) {
          // Revert the POP
          blockedPopTx = {
            action: nextAction,
            location: nextLocation,
            retry() {
              go(delta * -1);
            },
          };

          go(delta);
        }
      } else {
        // Trying to POP to a location with no index. We did not create
        // this location, so we can't effectively block the navigation.
        warning(
          false,
          // TODO: Write up a doc that explains our blocking strategy in
          // detail and link to it here so people can understand better what
          // is going on and how to avoid it.
          `You are trying to block a POP navigation to a location that was not ` +
            `created by the history library. The block will fail silently in ` +
            `production, but in general you should do all navigation with the ` +
            `history library (instead of using window.history.pushState directly) ` +
            `to avoid this situation.`
        );
      }
    } else {
      applyTx(nextAction);
    }
  }
}
```

这个事件主要用来配合 history.block 函数，实现阻塞路由变化。

> 需要注意的是调用 history.pushState()或 history.replaceState()不会触发 popstate 事件。只有在做出浏览器动作时，才会触发该事件，如用户点击浏览器的回退按钮（或者在 Javascript 代码中调用 history.back()或者 history.forward()方法）

当首次触发 popstate 事件时，blockedPopTx 为 null，因此会执行 else 中的代码。获取当前地址的 index 和 location，如果存在事先订阅的 blocker 函数，就会生成一个对象，赋值给 blockedPopTx，然后先跳回到原来的地址。这时候因为是调用了 go 函数，也就是内部调用了原生 history 对象的 go 方法，这时候又会触发 popstate 事件。这个时候再执行绑定函数，blockedPopTx 有值了，是否要跳转就留给了 blocker 函数决定了。

### HashHistory

和 BrowserHistory 提供的接口一致，但是是通过 hash 来实现路由。所以额外监听了`hashchange`事件

```ts
window.addEventListener(HashChangeEventType, () => {
  let [, nextLocation] = getIndexAndLocation();

  // Ignore extraneous hashchange events.
  if (createPath(nextLocation) !== createPath(location)) {
    handlePop();
  }
});
```

### MemoryHistory

MemoryHistory 比较特殊，主要用在 test 环境或 node 环境，这时候没有原生的 history 对象，因为它是属于浏览器提供的 api。

所以在 MemoryHistory 的内部，定义了一个 location 数组来存储所有的路由。

```ts
let { initialEntries = ["/"], initialIndex } = options;
let entries: Location[] = initialEntries.map((entry) => {
  let location = readOnly<Location>({
    pathname: "/",
    search: "",
    hash: "",
    state: null,
    key: createKey(),
    ...(typeof entry === "string" ? parsePath(entry) : entry),
  });

  warning(
    location.pathname.charAt(0) === "/",
    `Relative pathnames are not supported in createMemoryHistory({ initialEntries }) (invalid entry: ${JSON.stringify(
      entry
    )})`
  );

  return location;
});
let index = clamp(
  initialIndex == null ? entries.length - 1 : initialIndex,
  0,
  entries.length - 1
);
```

就是这个 entries 变量，而 index 序号是数组的下标，指向当前的 location。

相应的，push 函数内部也就变成了操作数组，添加新的 location

```ts
if (allowTx(nextAction, nextLocation, retry)) {
  index += 1;
  entries.splice(index, entries.length, nextLocation);
  applyTx(nextAction, nextLocation);
}
```

replace 函数也是

```ts
if (allowTx(nextAction, nextLocation, retry)) {
  entries[index] = nextLocation;
  applyTx(nextAction, nextLocation);
}
```

这三种 history 都是服务于另外一个库，react-router。
