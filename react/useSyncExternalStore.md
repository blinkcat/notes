# use-sync-external-store

版本[v18.1.0](https://github.com/facebook/react/tree/v18.1.0/packages/use-sync-external-store)

实际上是 react 18 提供的新 hook [useSyncExternalStore](https://reactjs.org/docs/hooks-reference.html#usesyncexternalstore)。

> useSyncExternalStore is a hook recommended for reading and subscribing from external data sources in a way that’s compatible with concurrent rendering features like selective hydration and time slicing.

## Demo

```js
const state = useSyncExternalStore(subscribe, getSnapshot[, getServerSnapshot]);

// The most basic example simply subscribes to the entire store:
const state = useSyncExternalStore(store.subscribe, store.getSnapshot);

// However, you can also subscribe to a specific field:
const selectedField = useSyncExternalStore(
  store.subscribe,
  () => store.getSnapshot().selectedField,
);
```

## How it works

useSyncExternalStore 需要一个订阅函数 subscribe，来使得外部的 store 可以触发组件的更新。同时为了判断是否有必要更新，还需要一个 getSnapshot 函数来返回当前的 store 的一个快照，通过比较两次的快照来判断 store 是否有变化。

每次渲染时，都会调用一次 getSnapshot 来获取最新的快照。

```js
const value = getSnapshot();
```

创建一个内部的 state，不仅可以保存上次的 value 和 getSnapshot 函数，还可以提供更新组件的函数。

```js
const [{ inst }, forceUpdate] = useState({ inst: { value, getSnapshot } });
```

在这个 hook 中，用最新的 value 和 getSnapshot 更新替换上次的值，留着后面做比较。

```js
useLayoutEffect(() => {
  inst.value = value;
  inst.getSnapshot = getSnapshot;

  // Whenever getSnapshot or subscribe changes, we need to check in the
  // commit phase if there was an interleaved mutation. In concurrent mode
  // this can happen all the time, but even in synchronous mode, an earlier
  // effect may have mutated the store.
  if (checkIfSnapshotChanged(inst)) {
    // Force a re-render.
    forceUpdate({ inst });
  }
}, [subscribe, value, getSnapshot]);
```

这里比较的方式是，通过当前的 getSnapshot 调用后的返回值和上次的 value 值做比较来判断是否相等。这个比较过程包含在 try catch 中是因为 getSnapshot 函数在执行的过程中可能因为[僵尸子元素](https://react-redux.js.org/api/hooks#stale-props-and-zombie-children)的问题出现报错。这种情况下的报错需要尝试恢复，所以在 catch 中直接返回了 true，继续执行。

```js
function checkIfSnapshotChanged(inst) {
  const latestGetSnapshot = inst.getSnapshot;
  const prevValue = inst.value;
  try {
    const nextValue = latestGetSnapshot();
    return !is(prevValue, nextValue);
  } catch (error) {
    return true;
  }
}
```

你可能会好奇为什么在 useLayoutEffect 这个 hook 中还需要检测一次变化，明明在上面已经将最新的 getSnapshot 的调用结果赋给了 value。这是因为这个 hook 的执行时机靠后，在它执行之前可能 store 又有变化了。

在这个 hook 中，开头做了一次检测，原因和上面说的一样。同时创建一个 handleStoreChange 交给 subscribe 函数，保证外部的 store 可以通知当前组件更新。

```js
useEffect(() => {
  // Check for changes right before subscribing. Subsequent changes will be
  // detected in the subscription handler.
  if (checkIfSnapshotChanged(inst)) {
    // Force a re-render.
    forceUpdate({ inst });
  }
  const handleStoreChange = () => {
    // ...
    if (checkIfSnapshotChanged(inst)) {
      // Force a re-render.
      forceUpdate({ inst });
    }
  };
  // Subscribe to the store and return a clean-up function.
  return subscribe(handleStoreChange);
}, [subscribe]);
```

最终返回 value 值。

在 useSyncExternalStore 这个 hook 的基础之上，还有一个更通用的 hook useSyncExternalStoreWithSelector。它额外提供了两个参数 selector 和 isEqual，可以自定义获取更细粒度的快照和更精确的比较。

useSyncExternalStoreWithSelector 内部利用了 selector 和 isEqual 入参生成两个基于 getSnapshot 和 getServerSnapshot 的细粒度选择函数 getSelection 和 getServerSelection。并将这两个函数传给 useSyncExternalStore。

```js
const [getSelection, getServerSelection] = useMemo(() => {
  // ...
}, [getSnapshot, getServerSnapshot, selector, isEqual]);

const value = useSyncExternalStore(subscribe, getSelection, getServerSelection);
// ...
return value;
```

实际上在 useMemo 这里一共做了两层缓存。第一层是在提供给 useMemo 的函数中保存了上次的 snapshot 和 selection，也就是这里的 memoizedSnapshot 和 memoizedSelection。如果发现两次的 snapshot 或是 selection 没有变化，就返回之前的 selection。

```js
useMemo(() => {
  // ...
  let memoizedSnapshot;
  let memoizedSelection;
  const memoizedSelector = (nextSnapshot) => {
    // ...
  };
  // ...
  const getSnapshotWithSelector = () => memoizedSelector(getSnapshot());
  // ...
  return [
    getSnapshotWithSelector,
    // ...
  ];
}, [getSnapshot, getServerSnapshot, selector, isEqual]);
```

当 useMemo 的依赖变化时，会产生新的返回值。这时最终的 selection 还是可能没有变化，例如，当用户这样使用时，selector 在语义上没有变化，只是引用改变了。

```js
useSyncExternalStoreWithSelector(
  subscribe,
  getSnapshot,
  null,
  (state) => state.something
);
```

所以第二次缓存就是在 useMemo 的外侧也要记住上次的 selection。在选择结果没有变化的时候，返回上次的结果，保持最终的结果引用不变。

```js
// Use this to track the rendered snapshot.
const instRef = useRef(null);
let inst;
if (instRef.current === null) {
  inst = {
    hasValue: false,
    value: (null: Selection | null),
  };
  instRef.current = inst;
} else {
  inst = instRef.current;
}

const [getSelection, getServerSelection] = useMemo(
  () => {
    // ...
  },
  [
    // ...
  ]
);
// ...
useEffect(() => {
  inst.hasValue = true;
  inst.value = value;
}, [value]);
```

## Attention

如果使用了 useSyncExternalStore，那么即使更新外部 store 的操作被包在[startTransition](https://reactjs.org/docs/hooks-reference.html#usetransition)中，也无法开启 time slice 功能。只能配合[useDeferredValue](https://reactjs.org/docs/hooks-reference.html#usedeferredvalue)这个 hook，才能开启 time slice。因为只有组件自身的内部状态，即直接通过 useState 或 useReducer 产生的状态，才可以配合 useTransition 使用 time slice。

## References

1. [React v18.0](https://reactjs.org/blog/2022/03/29/react-v18.html)
2. [useTransition() vs useDeferredValue | React 18](https://www.youtube.com/watch?v=lDukIAymutM)
3. [useMutableSource → useSyncExternalStore](https://github.com/reactwg/react-18/discussions/86)
4. [Dan Abramov: Beyond React 16 | JSConf Iceland](https://www.youtube.com/watch?v=nLF0n9SACd4)
5. [Beyond React 16: Time Slicing and Suspense API](https://auth0.com/blog/time-slice-suspense-react16/)
