# zustand

版本[v3.5.12](https://github.com/pmndrs/zustand/tree/v3.5.12)

## Usage

```js
import create from "zustand";

const useStore = create((set) => ({
  bears: 0,
  increasePopulation: () => set((state) => ({ bears: state.bears + 1 })),
  removeAllBears: () => set({ bears: 0 }),
}));

function BearCounter() {
  const bears = useStore((state) => state.bears);
  return <h1>{bears} around here ...</h1>;
}

function Controls() {
  const increasePopulation = useStore((state) => state.increasePopulation);
  return <button onClick={increasePopulation}>one up</button>;
}
```

通过调用 hooks 的方式，使用起来非常简单。api 少，也很容易理解。

首先用`create`创造一个 store(状态集合)，以及一个 hook，用来获取 state。

## How it works

先来看`create`方法

```ts
export default function create<TState extends State>(
  createState: StateCreator<TState> | StoreApi<TState>
): UseStore<TState> {
  const api: StoreApi<TState> =
    typeof createState === "function" ? createImpl(createState) : createState;

  const useStore: any = <StateSlice>(
    selector: StateSelector<TState, StateSlice> = api.getState as any,
    equalityFn: EqualityChecker<StateSlice> = Object.is
  ) => {
    // ...
  };

  Object.assign(useStore, api);
  // ...
  return useStore;
}
```

因为源码是 ts 写的，所以参数都带上详细的类型定义。`create`方法的入参定义表示它可以是一个函数，也可以是一个对象

```js
export interface StoreApi<T extends State> {
  setState: SetState<T>
  getState: GetState<T>
  subscribe: Subscribe<T>
  destroy: Destroy
}
export type StateCreator<T extends State, CustomSetState = SetState<T>> = (
  set: CustomSetState,
  get: GetState<T>,
  api: StoreApi<T>
) => T
```

而返回值的定义

```js
export interface UseStore<T extends State> {
  (): T
  <U>(selector: StateSelector<T, U>, equalityFn?: EqualityChecker<U>): U
  setState: SetState<T>
  getState: GetState<T>
  subscribe: Subscribe<T>
  destroy: Destroy
}
```

表示返回值是一个函数，并且带了四个函数属性。

如果入参 createState 是个函数，那么就传入 createImpl 函数处理

```ts
export default function create<TState extends State>(
  createState: StateCreator<TState>
): StoreApi<TState> {
  let state: TState;
  const listeners: Set<StateListener<TState>> = new Set();

  const setState: SetState<TState> = (partial, replace) => {
    // ...
  };

  const getState: GetState<TState> = () => state;

  const subscribeWithSelector = <StateSlice>(
    listener: StateSliceListener<StateSlice>,
    selector: StateSelector<TState, StateSlice> = getState as any,
    equalityFn: EqualityChecker<StateSlice> = Object.is
  ) => {
    // ...
  };

  const subscribe: Subscribe<TState> = <StateSlice>(
    listener: StateListener<TState> | StateSliceListener<StateSlice>,
    selector?: StateSelector<TState, StateSlice>,
    equalityFn?: EqualityChecker<StateSlice>
  ) => {
    // ...
  };

  const destroy: Destroy = () => listeners.clear();
  const api = { setState, getState, subscribe, destroy };
  state = createState(setState, getState, api);
  return api;
}
```

经过 createImpl 处理后，会返回一个 StoreApi 类型的对象。

首先在内部通过闭包存储了 state 和 listeners，一个是我们所有的状态，一个是我们收集的函数列表。

getState 函数很简单，setState 函数的类型是

```ts
export type State = object;

export type PartialState<
  T extends State,
  K1 extends keyof T = keyof T,
  K2 extends keyof T = K1,
  K3 extends keyof T = K2,
  K4 extends keyof T = K3
> =
  | (Pick<T, K1> | Pick<T, K2> | Pick<T, K3> | Pick<T, K4> | T)
  | ((state: T) => Pick<T, K1> | Pick<T, K2> | Pick<T, K3> | Pick<T, K4> | T);

export type SetState<T extends State> = {
  <
    K1 extends keyof T,
    K2 extends keyof T = K1,
    K3 extends keyof T = K2,
    K4 extends keyof T = K3
  >(
    partial: PartialState<T, K1, K2, K3, K4>,
    replace?: boolean
  ): void;
};
```

第一个参数 partial 可以是个对象，也可以是个函数。第二个参数 replace 表示是否要用 partial 对象或是 partial 函数的返回结果替换当前 state。

```ts
const setState: SetState<TState> = (partial, replace) => {
  // TODO: Remove type assertion once https://github.com/microsoft/TypeScript/issues/37663 is resolved
  // https://github.com/microsoft/TypeScript/issues/37663#issuecomment-759728342
  const nextState =
    typeof partial === "function"
      ? (partial as (state: TState) => TState)(state)
      : partial;
  if (nextState !== state) {
    const previousState = state;
    state = replace
      ? (nextState as TState)
      : Object.assign({}, state, nextState);
    listeners.forEach((listener) => listener(state, previousState));
  }
};
```

在 setState 中，会先算得当前最新的 state，然后和之前的 state 做比较。如果不同，用新的合并旧的，或者替换旧的(根据 replace 入参)。然后逐一执行 listeners 列表中的函数。

destroy 函数很简单，subscribe 函数定义两个重载函数

```ts
export type StateListener<T> = (state: T, previousState: T) => void;
export type StateSliceListener<T> = (slice: T, previousSlice: T) => void;

export interface Subscribe<T extends State> {
  (listener: StateListener<T>): () => void;
  <StateSlice>(
    listener: StateSliceListener<StateSlice>,
    selector?: StateSelector<T, StateSlice>,
    equalityFn?: EqualityChecker<StateSlice>
  ): () => void;
}

const subscribe: Subscribe<TState> = <StateSlice>(
  listener: StateListener<TState> | StateSliceListener<StateSlice>,
  selector?: StateSelector<TState, StateSlice>,
  equalityFn?: EqualityChecker<StateSlice>
) => {
  if (selector || equalityFn) {
    return subscribeWithSelector(
      listener as StateSliceListener<StateSlice>,
      selector,
      equalityFn
    );
  }
  listeners.add(listener as StateListener<TState>);
  // Unsubscribe
  return () => listeners.delete(listener as StateListener<TState>);
};
```

如果没有第二个和第三个入参，直接将传入的 listener 加入 listeners 列表中。

如果有

```ts
const subscribeWithSelector = <StateSlice>(
  listener: StateSliceListener<StateSlice>,
  selector: StateSelector<TState, StateSlice> = getState as any,
  equalityFn: EqualityChecker<StateSlice> = Object.is
) => {
  let currentSlice: StateSlice = selector(state);
  function listenerToAdd() {
    const nextSlice = selector(state);
    if (!equalityFn(currentSlice, nextSlice)) {
      const previousSlice = currentSlice;
      listener((currentSlice = nextSlice), previousSlice);
    }
  }
  listeners.add(listenerToAdd);
  // Unsubscribe
  return () => listeners.delete(listenerToAdd);
};
```

会先通过 selector 函数计算出需要的这部分状态，然后通过闭包保存起来。接着创建一个新的函数 listenerToAdd，用 selector 函数来计算当前需要的状态，然后用 equalityFn 方法和之前的对比，如果有变动就执行传入的 listener 函数。

这四个方法组成的对象会返回。

回到`create`函数中，在有了 StoreApi 对象后，会在 useStore hook 中使用。这个 hook 被我们用来在组件中，组件外获取 state，更新 state，更新组件。

我们先看 useStore 的函数签名

```ts
export type StateSelector<T extends State, U> = (state: T) => U;
export type EqualityChecker<T> = (state: T, newState: T) => boolean;

const useStore: any = <StateSlice>(
  selector: StateSelector<TState, StateSlice> = api.getState as any,
  equalityFn: EqualityChecker<StateSlice> = Object.is
) => {
  // ...
};
```

`StateSlice`是个泛型参数，第一个参数 selector 是一个选择函数，选择需要的状态，默认是`api.getState`，也就是返回所有的状态。equalityFn 是一个函数，用来比较两个参数是否相等，默认是`Object.is`。

在函数内部，先定义一个更新组件的方法

```ts
const [, forceUpdate] = useReducer((c) => c + 1, 0) as [never, () => void];
```

接着，求得当前的 state，将 state 和一些参数用 ref 保存起来，因为这个函数会在组件内部调用，所以可以使用 useRef。

```ts
const state = api.getState();
const stateRef = useRef(state);
const selectorRef = useRef(selector);
const equalityFnRef = useRef(equalityFn);
const erroredRef = useRef(false);

const currentSliceRef = useRef<StateSlice>();
if (currentSliceRef.current === undefined) {
  currentSliceRef.current = selector(state);
}
```

然后，每次 useStore 运行都会拿这个 ref 值和当前值做比对，如果变化了，求取两个变量。

```ts
let newStateSlice: StateSlice | undefined;
let hasNewStateSlice = false;

// The selector or equalityFn need to be called during the render phase if
// they change. We also want legitimate errors to be visible so we re-run
// them if they errored in the subscriber.
if (
  stateRef.current !== state ||
  selectorRef.current !== selector ||
  equalityFnRef.current !== equalityFn ||
  erroredRef.current
) {
  // Using local variables to avoid mutations in the render phase.
  newStateSlice = selector(state);
  hasNewStateSlice = !equalityFn(
    currentSliceRef.current as StateSlice,
    newStateSlice
  );
}
```

接下来是两个 useEffect，第一个用来更新 ref

```ts
// Syncing changes in useEffect.
useIsomorphicLayoutEffect(() => {
  if (hasNewStateSlice) {
    currentSliceRef.current = newStateSlice as StateSlice;
  }
  stateRef.current = state;
  selectorRef.current = selector;
  equalityFnRef.current = equalityFn;
  erroredRef.current = false;
});
```

第二个用来决定何时需要更新组件

```ts
const stateBeforeSubscriptionRef = useRef(state);
useIsomorphicLayoutEffect(() => {
  const listener = () => {
    try {
      const nextState = api.getState();
      const nextStateSlice = selectorRef.current(nextState);
      if (
        !equalityFnRef.current(
          currentSliceRef.current as StateSlice,
          nextStateSlice
        )
      ) {
        stateRef.current = nextState;
        currentSliceRef.current = nextStateSlice;
        forceUpdate();
      }
    } catch (error) {
      erroredRef.current = true;
      forceUpdate();
    }
  };
  const unsubscribe = api.subscribe(listener);
  if (api.getState() !== stateBeforeSubscriptionRef.current) {
    listener(); // state has changed before subscription
  }
  return unsubscribe;
}, []);
```

重点在 listener 函数中，先获取到最新的选中状态，然后和当前的对比，如果有变化，执行 forceUpdate，如果发生错误了，也会执行 forceUpdate。

这里放置`try catch`的原因是为了防止[僵尸子元素](https://react-redux.js.org/api/hooks#stale-props-and-zombie-children)。在 listener 回调中，执行 selector 函数如果出错了，那么可能是因为回调的添加顺序和执行顺序导致的。这时候尝试在更新一次组件，看看能不能解决。同时，`erroredRef.current`置为 true。从上面的代码中可以看到，如果 `erroredRef.current === true`，selector 函数一定会被执行，用来计算 newStateSlice。这时候如果还是报错，那么错误会被直接抛出。

但是`stale props`的问题还是没能完全解决。多个层级的组件更新如果不使用`unstable_batchedUpdates`包装，还是会存在更新顺序的问题，很可能造成意外的报错情况。

```js
return hasNewStateSlice
	? (newStateSlice as StateSlice)
	: currentSliceRef.current
```

最后会将最新的选择状态返回。

### Middleware

这个库也使用了中间件的方式来扩展自身功能

```ts
// Log every time state is changed
const log = (config) => (set, get, api) =>
  config(
    (args) => {
      console.log("  applying", args);
      set(args);
      console.log("  new state", get());
    },
    get,
    api
  );

// Turn the set method into an immer proxy
const immer = (config) => (set, get, api) =>
  config(
    (partial, replace) => {
      const nextState =
        typeof partial === "function" ? produce(partial) : partial;
      return set(nextState, replace);
    },
    get,
    api
  );

const useStore = create(
  log(
    immer((set) => ({
      bees: false,
      setBees: (input) => set((state) => void (state.bees = input)),
    }))
  )
);
```

一开始可能有些难理解，可以把 config 想象成一个具有`StateCreator`类型的函数，接受三个参数，返回一个完整的 state。中间件运作的方式就是在传给 config 的参数上，利用 monkey patch 的方式做扩展。

传给 immer`StateCreator`类型的参数，返回一个新的`StateCreator`类型的函数。将这个函数再传给 log，又返回一个`StateCreator`类型的函数，最后传给 create。
`create(log(immer(...)))`

简言之，这里的中间件就是一个传入或者不传入`StateCreator`函数，返回另外一个`StateCreator`函数的高阶函数。

我们可以看一个简单的

```ts
export const redux =
  <S extends State, A extends { type: unknown }>(
    reducer: (state: S, action: A) => S,
    initial: S
  ) =>
  (
    set: SetState<S>,
    get: GetState<S>,
    api: StoreApi<S> & {
      dispatch?: (a: A) => A;
      devtools?: any;
    }
  ): S & { dispatch: (a: A) => A } => {
    api.dispatch = (action: A) => {
      set((state: S) => reducer(state, action));
      if (api.devtools) {
        api.devtools.send(api.devtools.prefix + action.type, get());
      }
      return action;
    };
    return { dispatch: api.dispatch, ...initial };
  };
```

这个`redux`函数返回了一个`StateCreator`函数，所以也符合中间件的要求，虽然它不需要传入一个`StateCreator`函数。

### Context

虽然不需要 Context，但是这个库也提供 Context 方式的使用方法。

```ts
import create from 'zustand'
import createContext from 'zustand/context'

const { Provider, useStore } = createContext()

const createStore = () => create(...)

const App = () => (
  <Provider createStore={createStore}>
    ...
  </Provider>
)

const Component = () => {
  const state = useStore()
  const slice = useStore(selector)
  ...
}
```

看下代码

```ts
function createContext<TState extends State>() {
  const ZustandContext = reactCreateContext<UseStore<TState> | undefined>(
    undefined
  );
  // ...
  return {
    Provider,
    useStore,
    useStoreApi,
  };
}
```

示例中使用的 Provider 并非简单的 Context.Provider，而是一个新的组件

```ts
const Provider = ({
  initialStore,
  createStore,
  children,
}: {
  /**
   * @deprecated
   */
  initialStore?: UseStore<TState>;
  createStore: () => UseStore<TState>;
  children: ReactNode;
}) => {
  const storeRef = useRef<UseStore<TState>>();

  if (!storeRef.current) {
    if (initialStore) {
      console.warn(
        "Provider initialStore is deprecated and will be removed in the next version."
      );
      if (!createStore) {
        createStore = () => initialStore;
      }
    }
    storeRef.current = createStore();
  }

  return createElement(
    ZustandContext.Provider,
    { value: storeRef.current },
    children
  );
};
```

内部用一个 ref 保存 createStore 执行后的结果，然后连同 children，传入 ZustandContext.Provider。这样 Provider 的 value 就一直不会改变。

而 useStore 也需要根据上下文重新封装

```ts
const useStore: UseContextStore<TState> = <StateSlice>(
  selector?: StateSelector<TState, StateSlice>,
  equalityFn = Object.is
) => {
  // ZustandContext value is guaranteed to be stable.
  const useProviderStore = useContext(ZustandContext);
  if (!useProviderStore) {
    throw new Error(
      "Seems like you have not used zustand provider as an ancestor."
    );
  }
  return useProviderStore(
    selector as StateSelector<TState, StateSlice>,
    equalityFn
  );
};
```

但是这个时候，useStore 身上附带的 api 只能在组件内部使用，等于使用场景被削弱了。

```ts
const useStoreApi = () => {
  // ZustandContext value is guaranteed to be stable.
  const useProviderStore = useContext(ZustandContext);
  if (!useProviderStore) {
    throw new Error(
      "Seems like you have not used zustand provider as an ancestor."
    );
  }
  return useMemo(
    () => ({
      getState: useProviderStore.getState,
      setState: useProviderStore.setState,
      subscribe: useProviderStore.subscribe,
      destroy: useProviderStore.destroy,
    }),
    [useProviderStore]
  );
};
```

所以不是很推荐这样使用。

## 再探 zustand

版本[v4.0.0-rc.1](https://github.com/pmndrs/zustand/tree/v4.0.0-rc.1)

这个版本相比之前的版本，在内部实现上有了相当大的变化。

首先是 useStore 这个 hook 使用了 react 官方的实现，[useSyncExternalStoreWithSelector](./react/useSyncExternalStore.md)。功能上不仅和之前的实现相同，而且还可以省略大量的代码。

如果将 zustand 和 redux 比较，zustand 就像是弱化了 reducer 和 action，并且自带 redux-thunk 的 redux。而 zustand 的中间件就像是 redux 的 middleware 加上 storeEnhancer 的混合体。但是在设计上，redux 追求的是通过 dispatch 某个 action，来实现业务逻辑上的封装、解耦和复用。而 zustand 把 state 和修改 state 的方法都放在一个 object 中，这就导致了几个问题，

1. 多个方法之间不能直接调用对方，难以复用逻辑
2. 子状态和方法容易发生命名冲突

因此，zustand 比较适合小型项目，或者是嵌入到其他 library 中。

## References

1. [TypeScript Guide](https://github.com/pmndrs/zustand/blob/main/docs/typescript.md)
