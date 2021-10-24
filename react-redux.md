# react-redux

版本 [v7.2.5](https://github.com/reduxjs/react-redux/tree/v7.2.5)

绑定 react 和 redux，当 redux 的 store 有变动时，更新相应的 react component。

## How it works

我们主要看 hooks 版本，hoc 版本应该会慢慢被淘汰。

首先是`Provider`，作为一个全局的上下文，为各个 api 提供 redux store。

```js
function Provider({ store, context, children }) {
  // ...
}
```

本质上是一个组件，props 中还可以接受自定义的`context`。

第一部分是生成 provider 的 value

```js
const contextValue = useMemo(() => {
  const subscription = createSubscription(store);
  subscription.onStateChange = subscription.notifyNestedSubs;
  return {
    store,
    subscription,
  };
}, [store]);
```

`store`一般不会变化，所以这个 useMemo 只会执行一次。最后返回 store 和 subscription 组成的对象。这个 subscription 通过`createSubscription`函数创建。这个 subscription 贯穿了整个库的逻辑，所以我们有必要先了解下。

```js
export function createSubscription(store, parentSub) {
  let unsubscribe;
  let listeners = nullListeners;
  // ...
  const subscription = {
    addNestedSub,
    notifyNestedSubs,
    handleChangeWrapper,
    isSubscribed,
    trySubscribe,
    tryUnsubscribe,
    getListeners: () => listeners,
  };

  return subscription;
}
```

subscription 是一个众多方法的集合，它在内部维护了一个双向的链表，用来记录回调函数。

通过`trySubscribe`方法来初始化一个称为`listeners`的双向链表，

```js
function handleChangeWrapper() {
  if (subscription.onStateChange) {
    subscription.onStateChange();
  }
}

function addNestedSub(listener) {
  trySubscribe();
  return listeners.subscribe(listener);
}

function trySubscribe() {
  if (!unsubscribe) {
    unsubscribe = parentSub
      ? parentSub.addNestedSub(handleChangeWrapper)
      : store.subscribe(handleChangeWrapper);

    listeners = createListenerCollection();
  }
}
```

这里可以发现存在父子 subscription 的概念，两者都在内部有一个双向链表。并且通过`parentSub.addNestedSub`方法，将子 subscription 上的`onStateChange`函数存在父 subscription 的链表里。

`createListenerCollection`函数用来创建一个双向链表

```js
function createListenerCollection() {
  const batch = getBatch();
  let first = null;
  let last = null;

  return {
    clear() {
      first = null;
      last = null;
    },

    notify() {
      batch(() => {
        let listener = first;
        while (listener) {
          listener.callback();
          listener = listener.next;
        }
      });
    },

    get() {
      let listeners = [];
      let listener = first;
      while (listener) {
        listeners.push(listener);
        listener = listener.next;
      }
      return listeners;
    },

    subscribe(callback) {
      let isSubscribed = true;

      let listener = (last = {
        callback,
        next: null,
        prev: last,
      });

      if (listener.prev) {
        listener.prev.next = listener;
      } else {
        first = listener;
      }

      return function unsubscribe() {
        if (!isSubscribed || first === null) return;
        isSubscribed = false;

        if (listener.next) {
          listener.next.prev = listener.prev;
        } else {
          last = listener.prev;
        }
        if (listener.prev) {
          listener.prev.next = listener.next;
        } else {
          first = listener.next;
        }
      };
    },
  };
}
```

基本上都是在操作链表引用。subscribe 用来在链表末尾添加一个`listener`对象，这个对象有这样的结构。

```js
{
	callback,
    next: null,
    prev: last,
}
```

unsubscribe 就是将这个对象从链表中移除。特别是的 notify 函数，它会将所有的 callback 都放在`batch`函数中执行。这个 batch 函数是`react-dom`中的`unstable_batchedUpdates`函数。可以将多次 state 更新合并为一次。所以这个链表中维护的回调函数都是用来更新 state 的。而且，从`trySubscribe`函数的实现中可以发现，每个链表里存的都是子 subscription 的 onStateChange 函数。

再回到`Provider`组件的第一部分中，创建了一个根 subscription，然后将这个根 subscription 的 onStateChange 属性赋值为它的 notifyNestedSubs 属性。

这个 notifyNestedSubs 是 subscription 对象中的一个函数，用来执行内部链表中所有的回调。

```js
function notifyNestedSubs() {
  listeners.notify();
}
```

现在这个根 subscription 的 onStateChange 也就是 notifyNestedSubs，绑定在 store 的回调数组中了。也就是说只要 state 发生变化，就会调用 notifyNestedSubs，根 subscription 中链表保存的回调函数就会执行。

而在第二部分代码中，

```js
const previousState = useMemo(() => store.getState(), [store]);

useIsomorphicLayoutEffect(() => {
  const { subscription } = contextValue;
  subscription.trySubscribe();

  if (previousState !== store.getState()) {
    subscription.notifyNestedSubs();
  }
  return () => {
    subscription.tryUnsubscribe();
    subscription.onStateChange = null;
  };
}, [contextValue, previousState]);
```

先获取一次 state 对象，因为 store 通常不会改变，所以这里也只会获取一次。

useIsomorphicLayoutEffect 很简单，就是在`useLayoutEffect`和`useEffect`中挑一个。

```js
export const useIsomorphicLayoutEffect =
  typeof window !== "undefined" &&
  typeof window.document !== "undefined" &&
  typeof window.document.createElement !== "undefined"
    ? useLayoutEffect
    : useEffect;
```

在它的内部，根 subscription 被初始化，subscription.notifyNestedSubs 被添加到 store 的回调数组中，响应每次 state 的改变。接着如果在执行这次 effect 的过程中，state 发生变化，就必须立即调用一次 subscription.notifyNestedSubs，来使根 subscription 的双向链表中所有的回调执行一次。最后返回一个 clear 函数。

接下来进入 react-redux 的 hooks api。

第一个是`useReduxContext`

```js
export function useReduxContext() {
  const contextValue = useContext(ReactReduxContext);

  if (process.env.NODE_ENV !== "production" && !contextValue) {
    throw new Error(
      "could not find react-redux context value; please ensure the component is wrapped in a <Provider>"
    );
  }

  return contextValue;
}
```

顾名思义，它直接使用了默认的`ReactReduxContext`。

第二个是`useStore`，

```js
export function createStoreHook(context = ReactReduxContext) {
  const useReduxContext =
    context === ReactReduxContext
      ? useDefaultReduxContext
      : () => useContext(context);
  return function useStore() {
    const { store } = useReduxContext();
    return store;
  };
}

export const useStore = /*#__PURE__*/ createStoreHook();
```

注意这里有个`createStoreHook`工厂方法，可以用来自定义 useStore，只需要传入你自己的上下文。默认使用的是 ReactReduxContext，然后从中获取 store 对象返回。

第三个是`useDispatch`，也是同样的思路

```js
export function createDispatchHook(context = ReactReduxContext) {
  const useStore =
    context === ReactReduxContext ? useDefaultStore : createStoreHook(context);

  return function useDispatch() {
    const store = useStore();
    return store.dispatch;
  };
}

export const useDispatch = /*#__PURE__*/ createDispatchHook();
```

传入 context，可以用`createDispatchHook`定制你自己的 hook，默认使用 ReactReduxContext。如果是默认的 context，就会使用上面提到的默认的 useStore，得到 store 对象，返回 dispatch 属性值。

最后一个是 useSelector，还是同样的思路，只不过这里边还包含绑定组件更新函数的逻辑。

```js
export function createSelectorHook(context = ReactReduxContext) {
  const useReduxContext =
    context === ReactReduxContext
      ? useDefaultReduxContext
      : () => useContext(context);
  return function useSelector(selector, equalityFn = refEquality) {
    // ...
    const { store, subscription: contextSub } = useReduxContext();

    const selectedState = useSelectorWithStoreAndSubscription(
      selector,
      equalityFn,
      store,
      contextSub
    );

    useDebugValue(selectedState);

    return selectedState;
  };
}

export const useSelector = /*#__PURE__*/ createSelectorHook();
```

这里的逻辑主要都包含在`useSelectorWithStoreAndSubscription`中。

在这个函数中，首先会创建一些变量，

```js
const [, forceRender] = useReducer((s) => s + 1, 0);

const subscription = useMemo(
  () => createSubscription(store, contextSub),
  [store, contextSub]
);

const latestSubscriptionCallbackError = useRef();
const latestSelector = useRef();
const latestStoreState = useRef();
const latestSelectedState = useRef();
```

有组件的更新函数，一个新的 subscription，以及一些用来记住上次执行时的相关变量值的 ref。

注意这里的 createSubscription 传入了父 subscription，也就是我们在 `Provider`中定义的那个。

接着是一个 try catch 包裹起来的代码快，

```js
const storeState = store.getState();
let selectedState;

try {
  if (
    selector !== latestSelector.current ||
    storeState !== latestStoreState.current ||
    latestSubscriptionCallbackError.current
  ) {
    const newSelectedState = selector(storeState);
    // ensure latest selected state is reused so that a custom equality function can result in identical references
    if (
      latestSelectedState.current === undefined ||
      !equalityFn(newSelectedState, latestSelectedState.current)
    ) {
      selectedState = newSelectedState;
    } else {
      selectedState = latestSelectedState.current;
    }
  } else {
    selectedState = latestSelectedState.current;
  }
} catch (err) {
  if (latestSubscriptionCallbackError.current) {
    err.message += `\nThe error may be correlated with this previous error:\n${latestSubscriptionCallbackError.current.stack}\n\n`;
  }

  throw err;
}
```

如果在这次组件 render 过程中，传入的 selector，store 中的 state 有变化，或者是上次执行 selector 函数出错了，就需要重新执行 selector 函数。将最终得到的结果赋值给 selectedState。如果出现错误，捕获后做个处理再抛出。

再来一段是在组件渲染后，浏览器完成绘制之后，更新一堆变量的过程

```js
useIsomorphicLayoutEffect(() => {
  latestSelector.current = selector;
  latestStoreState.current = storeState;
  latestSelectedState.current = selectedState;
  latestSubscriptionCallbackError.current = undefined;
});
```

在这段之后才是重头戏，绑定组件的更新函数到 subscription 中。

```js
useIsomorphicLayoutEffect(() => {
  function checkForUpdates() {
    try {
      const newStoreState = store.getState();
      // Avoid calling selector multiple times if the store's state has not changed
      if (newStoreState === latestStoreState.current) {
        return;
      }

      const newSelectedState = latestSelector.current(newStoreState);

      if (equalityFn(newSelectedState, latestSelectedState.current)) {
        return;
      }

      latestSelectedState.current = newSelectedState;
      latestStoreState.current = newStoreState;
    } catch (err) {
      // we ignore all errors here, since when the component
      // is re-rendered, the selectors are called again, and
      // will throw again, if neither props nor store state
      // changed
      latestSubscriptionCallbackError.current = err;
    }

    forceRender();
  }

  subscription.onStateChange = checkForUpdates;
  subscription.trySubscribe();

  checkForUpdates();

  return () => subscription.tryUnsubscribe();
}, [store, subscription]);
```

`checkForUpdates`是当前组件的更新函数。将它赋值给 subscription.onStateChange，然后调用 subscription.trySubscribe。根据我们上面的讲解，这个 checkForUpdates 函数最终被存入了父 subscription，也就是在 Provider 中创建的那个 subscription 的内部链表中。

然后立即调用一次 checkForUpdates 尝试更新组件，因为在这个时间段内，state 可能有更新。最后返回取消绑定的函数。

在 checkForUpdates 函数中，会判断当前的 state 有没有发生变化，已经当前的 selectedState 有没有发生变化，如果有变化就更新。在执行 selector 函数的过程中，如果出错了，只会默默的记录下来，不会阻断更新。这是为了不受[僵尸子元素](https://react-redux.js.org/api/hooks#stale-props-and-zombie-children)的影响。

如果看过[zustand](./zustand.md)的源码，就会发现它们之间的绑定相关逻辑几乎一样。但是 react-redux 这里为什么要搞一个多层级的 subscription 呢？这看起来似乎多此一举。这就要考虑到 hoc 方式的 api 了。

为了防止[stale props](https://react-redux.js.org/api/hooks#stale-props-and-zombie-children)，主要原因是子组件先渲染完成，先于父组件绑定更新函数，在 state 变化时，子组件的更新函数先执行，这就可能造成子组件使用了旧的 props。所以在`connect`中，会为每一个被包裹的组件添加一个新的上下文，在这个上下文中添加新的 subscription，这样就形成了一个多层级的 subscription 结构。子组件的更新函数存在父 subscription 中，需要通过父 subscription 调用 notifyNestedSubs 才能执行更新。这就可以让父组件在更新自身后再执行子组件更新，因为这时候子组件的 props 肯定是最新的。

这段的逻辑在`connectAdvanced.js`文件中

```js
function captureWrapperProps(
  lastWrapperProps,
  lastChildProps,
  renderIsScheduled,
  wrapperProps,
  actualChildProps,
  childPropsFromStoreUpdate,
  notifyNestedSubs
) {
  // ...

  // If the render was from a store update, clear out that reference and cascade the subscriber update
  if (childPropsFromStoreUpdate.current) {
    childPropsFromStoreUpdate.current = null;
    notifyNestedSubs();
  }
}
// ...
const checkForUpdates = () => {
  // ...
  // If the child props haven't changed, nothing to do here - cascade the subscription update
  if (newChildProps === lastChildProps.current) {
    if (!renderIsScheduled.current) {
      notifyNestedSubs();
    }
  } else {
    // Save references to the new child props.  Note that we track the "child props from store update"
    // as a ref instead of a useState/useReducer because we need a way to determine if that value has
    // been processed.  If this went into useState/useReducer, we couldn't clear out the value without
    // forcing another re-render, which we don't want.
    lastChildProps.current = newChildProps;
    childPropsFromStoreUpdate.current = newChildProps;
    renderIsScheduled.current = true;

    // If the child props _did_ change (or we caught an error), this wrapper component needs to re-render
    forceComponentUpdateDispatch({
      type: "STORE_UPDATED",
      payload: {
        error,
      },
    });
  }
};
```

总得来说，还是推荐使用 hook api。
