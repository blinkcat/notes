# swr

版本[2.0.0-beta.3](https://github.com/vercel/swr/tree/2.0.0-beta.3)

swr 是一个用于请求数据的 react hook。

> “SWR” 这个名字来自于 stale-while-revalidate：一种由 HTTP RFC 5861 推广的 HTTP 缓存失效策略。这种策略首先从缓存中返回数据（过期的），同时发送 fetch 请求（重新验证），最后得到最新数据。

## Demo

```js
import useSWR from "swr";

function Profile() {
  const { data, error } = useSWR("/api/user/123", fetcher);

  if (error) return <div>failed to load</div>;
  if (!data) return <div>loading...</div>;

  // 渲染数据
  return <div>hello {data.name}!</div>;
}
```

这是一个非常简单的 demo，只需要一次调用就可以完成请求数据的工作。

## How it works

swr 最主要的 api 就这一个。从 api 的入参和返回值的命名中，多少也暴露出 swr 的一些[特性](https://swr.vercel.app/zh-CN#%E7%89%B9%E6%80%A7)。例如 key，暗示了 swr 会将请求结果缓存起来、fetcher 是一个由用户定义的请求方法，不仅能给 swr 带来扩展能力，也能使得 swr 可以实现平台无关性。

```js
const { data, error, isValidating, mutate } = useSWR(key, fetcher, options);
```

而[options](https://swr.vercel.app/zh-CN/docs/options#%E9%80%89%E9%A1%B9)这个 object 中的属性非常之多，可以按照自己的业务需要配置不同的属性，这让 swr 具有很大的灵活性。

### Options

下面我们先从 swr 的配置部分讲起。

swr 的配置有两种。

- 全局配置，通过 SWRConfig 组件的 value 属性，传递到所有的 hooks 中。
- 局部配置，在调用 hook 时，通过第三个参数传入。正如上面的 api 所示。

```js
<SWRConfig value={options}>
  <Component />
</SWRConfig>
```

SWRConfig 组件在内部使用了 React Context 向下传递配置。为了支持嵌套使用，在组件内部会将自身的 value 属性和上层 SWRConfig 组件提供的配置做一个合并。

```js
const extendedConfig = mergeConfigs(useContext(SWRConfigContext), value);
```

其次需要特别处理 swr 的 cache，保证相同的 cache 在多个组件和 hook 中可以作为一个单例。这里的 cache 指的是 swr 内部用来保存相同 key 值的数据请求的地方。默认的 defaultCache 是一个原生的 Map 实例。利用 SWRConfig 组件的 provider 属性，我们可以自定义一个新的 cache，应用在所有的子 hook 中。

在 useState hook 里，通过 initCache 函数来初始化 cache。这个函数是幂等的，同一个 cache 多次调用，也只会进行一次初始化。依赖于 useState 的特性，我们可以保证 cacheContext 数组的稳定性。也间接地保证了最终的配置中，cache 和 mutate 属性的稳定。

```js
// Should not use the inherited provider.
const provider = value && value.provider;

// Use a lazy initialized state to create the cache on first access.
const [cacheContext] = useState(() =>
  provider
    ? initCache(provider(extendedConfig.cache || defaultCache), value)
    : UNDEFINED
);

// Override the cache if a new provider is given.
if (cacheContext) {
  extendedConfig.cache = cacheContext[0];
  extendedConfig.mutate = cacheContext[1];
}

// Unsubscribe events.
useIsomorphicLayoutEffect(() => {
  if (cacheContext) {
    cacheContext[2] && cacheContext[2]();
    return cacheContext[3];
  }
}, []);

return createElement(
  SWRConfigContext.Provider,
  mergeObjects(props, {
    value: extendedConfig,
  })
);
```

在 useIsomorphicLayoutEffect 中，尝试对 cache 做一些工作，并返回一个清理的函数。但是 cacheContext 数组的第 2、3 项目只有在首次初始化 cache 的时候才存在。

最后将合并后的配置交给 SWRConfigContext.Provider。

在 useSWR hook 中，我们也需要考虑对多个 config 进行合并，因为单个 hook 中也可传入 config。这个工作是由 withArgs 函数来完成的。它也是采用和 SWRConfig 组件中一样的合并函数 mergeObjects，将上层的 config 和本地的 config 做一个合并。

```ts
export const withArgs = <SWRType>(hook: any) => {
  return function useSWRArgs(...args: any) {
    // Get the default and inherited configuration.
    const fallbackConfig = useSWRConfig();

    // Normalize arguments.
    const [key, fn, _config] = normalize<any, any>(args);

    // Merge configurations.
    const config = mergeConfigs(fallbackConfig, _config);

    // ...
  } as unknown as SWRType;
};

export const useSWRConfig = <
  T extends Cache = Map<string, State>
>(): FullConfiguration<T> => {
  return mergeObjects(defaultConfig, useContext(SWRConfigContext));
};
```

而我们所直接使用的 useSWR 是经过 withArgs 包装过的。所以不需要关心最终的 config 的来源。

```ts
export default withArgs<SWRHook>(useSWRHandler);
```

### Cache

接着我们再聊聊 swr 中的 cache，大多数 swr 的功能都是围绕 cache 来建设的，可以说 cache 是重中之重。

从类型上看，cache 至少要有读、写、删的能力。cache 中存的是可以代表一个请求的 key(一般是请求的 url)，和对应的包装过的请求结果(类型如下方的 State 所示)。

```ts
export interface Cache<Data = any> {
  get(key: Key): State<Data> | undefined;
  set(key: Key, value: State<Data>): void;
  delete(key: Key): void;
}

export type State<Data = any, Error = any> = {
  data?: Data;
  error?: Error;
  isValidating?: boolean;
  isLoading?: boolean;
};
```

前面提到过我们可以通过 SWRConfig 组件中的 provider 属性引入新的 cache。这就说明 swr 实际使用的 cache 可能有多个。这些 cache 都会作为 key 存在 SWRGlobalState 这个 weakMap 中。对应的 value 是 GlobalState 类型，具体的属性的作用我们后面会提到。

```ts
export const SWRGlobalState = new WeakMap<Cache, GlobalState>();

export type GlobalState = [
  Record<string, RevalidateCallback[]>, // EVENT_REVALIDATORS
  Record<string, [number, number]>, // MUTATION: [ts, end_ts]
  Record<string, [any, number]>, // FETCH: [data, ts]
  ScopedMutator, // Mutator
  (key: string, value: any, prev: any) => void, // Setter
  (key: string, callback: (current: any, prev: any) => void) => () => void // Subscriber
];
```

initCache 函数不仅会将传入的 cache 存进 SWRGlobalState 中，还会为这个 cache 创建一个 observer 模型。当 cache 有写入操作时，通知所有的订阅者。这里的回调函数是按照 key 分类的，key 相当于是事件名，setter 操作类似于触发 key 事件。

```ts
export const initCache = <Data = any>(
  provider: Cache<Data>,
  options?: Partial<ProviderConfiguration>
) => {
  // ...
  const subscriptions: Record<string, ((current: any, prev: any) => void)[]> =
    {};
  const subscribe = (
    key: string,
    callback: (current: any, prev: any) => void
  ) => {
    const subs = subscriptions[key] || [];
    subscriptions[key] = subs;

    subs.push(callback);
    return () => {
      subs.splice(subs.indexOf(callback), 1);
    };
  };
  const setter = (key: string, value: any, prev: any) => {
    provider.set(key, value);
    const subs = subscriptions[key];
    if (subs) {
      for (let i = subs.length; i--; ) {
        subs[i](value, prev);
      }
    }
  };
  // ...
};
```

除了通过 setter 修改 cache，还可以通过 mutate 函数。这个函数也会作为 initCache 函数的返回值暴露给用户。

```ts
const mutate = internalMutate.bind(UNDEFINED, provider) as ScopedMutator<Data>;
```

然后，就可以准备将 cache 存入 SWRGlobalState 中，这个 provider 变量就是 initCache 的入参，类型是 Cache。EVENT_REVALIDATORS 用来存储请求的 key 对应的重新请求数据的方法。

它后面的两个 object 存储的是请求的 key 对应的 mutation 操作和 fetch 操作的相关信息。mutation 就是我们上面提到的 mutate 函数调用，fetch 就是调用请求方法请求数据，然后写入 cache。因此如果是相同的 key，这两个操作都会操作同一个 cache。利用这些信息可以对多个相同 key 的 多个 fetch 操作进行去重，或者解决有相同 key 的 mutate 和 fetch 的冲突问题。

```ts
const EVENT_REVALIDATORS = {};
// ...
SWRGlobalState.set(provider, [
  EVENT_REVALIDATORS,
  {},
  {},
  mutate,
  setter,
  subscribe,
]);
```

在默认情况下，swr 需要在浏览器处于两种状态时重新请求数据，更新缓存。

- 浏览器页面可见性发生变化，document 对象触发 visibilitychange 事件。或者页面重新获得焦点，window 对象触发 focus 事件
- 浏览器断网后重新连接网络，window 对象触发 online 事件

因此在这里将 EVENT_REVALIDATORS 中的回调和这些事件做绑定。

```ts
const releaseFocus = opts.initFocus(
  setTimeout.bind(
    UNDEFINED,
    revalidateAllKeys.bind(
      UNDEFINED,
      EVENT_REVALIDATORS,
      revalidateEvents.FOCUS_EVENT
    )
  )
);
// ...
const revalidateAllKeys = (
  revalidators: Record<string, RevalidateCallback[]>,
  type: RevalidateEvent
) => {
  for (const key in revalidators) {
    if (revalidators[key][0]) revalidators[key][0](type);
  }
};
```

既然有绑定，必须也要有解绑。

```ts
unmount = () => {
  releaseFocus && releaseFocus();
  releaseReconnect && releaseReconnect();
  // When un-mounting, we need to remove the cache provider from the state
  // storage too because it's a side-effect. Otherwise when re-mounting we
  // will not re-register those event listeners.
  SWRGlobalState.delete(provider);
};
```

最后如果是第一次处理这个 cache，即 SWRGlobalState 中原先并没有存入它。initCache 函数返回

```ts
return [provider, mutate, initProvider, unmount];
```

否则返回

```ts
return [provider, (SWRGlobalState.get(provider) as GlobalState)[3]];
```

也就是结果中只有 provider 和 mutate。

接下来我们可以进入 useSWR 这个 hook 的内部逻辑。由于它的逻辑比较复杂，我们需要分成四个部分来讲解。

第一个部分是 hook 中的准备工作。

### Preparations

首先我们需要确定一个请求的 key 值。

serialize 函数可以来处理传入的\_key，得到本次请求的 key 值。

```ts
export const useSWRHandler = <Data = any, Error = any>(
  _key: Key,
  fetcher: Fetcher<Data> | null,
  config: typeof defaultConfig & SWRConfiguration<Data, Error>
) => {
  // ...
  // `key` is the identifier of the SWR `data` state, `keyInfo` holds extra
  // states such as `error` and `isValidating` inside,
  // all of them are derived from `_key`.
  // `fnArg` is the argument/arguments parsed from the key, which will be passed
  // to the fetcher.
  const [key, fnArg] = serialize(_key);
  // ...
};
```

通过 key，我们可以从 cache 中得到它对应的读、写、订阅函数。

```ts
const [getCache, setCache, subscribeCache] = createCacheHelper<Data>(
  cache,
  key
);
```

第二个部分我们关注的是当请求结果变化时，useSWR 如何通知组件更新。

### Update component

答案是使用了 react 官方的[useSyncExternalStoreWithSelector](./react/useSyncExternalStore.md) hook。它需要一个订阅变化的函数，一个获取当前状态的函数，一个接收当前状态返回需要关注的子状态的 selector 函数，以及判断两次渲染过程中，关注的子状态是否有变化的函数。接着它会在每次的渲染后来控制组件的更新。

第一个入参利用了上面第一部分提到的 subscribeCache 函数，订阅 cache[key]值的变化。

```ts
// Get the current state that SWR should return.
const cached = useSyncExternalStoreWithSelector(
  useCallback(
    (callback: () => void) =>
      subscribeCache(key, (current: State<Data, any>) => {
        stateRef.current = current;
        callback();
      }),
    // eslint-disable-next-line react-hooks/exhaustive-deps
    [cache, key]
  ),
  getSnapshot,
  getSnapshot,
  selector,
  isEqual
);
```

getSnapshot 也是利用上面第一部分提到的 getCache 函数。

```ts
const getSnapshot = useCallback(getCache, [cache, key]);
```

selector 中关于 shouldStartRequest 的判断似乎没有必要。返回值只和 snapshot 本身有关。

```ts
const selector = (snapshot: State<Data, any>) => {
  const shouldStartRequest = (() => {
    if (!key) return false;
    if (!fetcher) return false;
    // If `revalidateOnMount` is set, we take the value directly.
    if (!isUndefined(revalidateOnMount)) return revalidateOnMount;
    // If it's paused, we skip revalidation.
    if (getConfig().isPaused()) return false;
    if (suspense) return false;
    return true;
  })();
  if (!shouldStartRequest) return snapshot;
  if (isEmptyCache(snapshot)) {
    return {
      isValidating: true,
      isLoading: true,
    };
  }
  return snapshot;
};
```

最有技术含量的还是在 isEqual 这里，它会比较两次的状态，判断是否有必要更新。所以减少不必要的更新可以直接地提升渲染效率。通常情况下，我们至少会浅比较两次的状态，即比较它们是否包含了相同的属性。但是依靠引用来判断不够精准。如果要更进一步，可以通过深比较来判断。在 isEqual 中确实也用了深比较，但是它在此基础上只会比较实际使用到的属性。

```ts
const isEqual = useCallback(
  (prev: State<Data, any>, current: State<Data, any>) => {
    let equal = true;
    for (const _ in stateDependencies) {
      const t = _ as keyof StateDependencies;
      if (!compare(current[t], prev[t])) {
        if (t === "data" && isUndefined(prev[t])) {
          if (!compare(current[t], fallback)) {
            equal = false;
          }
        } else {
          equal = false;
        }
      }
    }
    return equal;
  },
  // eslint-disable-next-line react-hooks/exhaustive-deps
  [cache, key]
);
// ...
const stateDependencies = useRef<StateDependencies>({}).current;
// ...
export interface StateDependencies {
  data?: boolean;
  error?: boolean;
  isValidating?: boolean;
  isLoading?: boolean;
}
```

我们用 stateDependencies 记录实际会用到的属性。方法是在 hook 的返回数据中，我们使用 get 方法，这样可以记录哪个数据是用户实际使用到的。

```ts
return {
  // ...
  get data() {
    stateDependencies.data = true;
    return returnedData;
  },
  // ...
};
```

> 在 Vercel 的实际应用中，这个优化减少了约 60% 的重新渲染。

接着我们来看第三部分，创建 revalidate 函数来发送请求，更新 cache。

### Revalidate

revalidate 函数有两百多行，我们慢慢分析。

通过 setCache 来更新 cache 可以触发对应回调。这个我们在上面已经提到过了。

当需要请求数据时，需要在 FETCH 对象中存着本次请求的 Promise，以及序号。这个 FETCH 是 SWRGlobalState 中存储的关于当前 cache 的第三个对象。用来记录请求，对多个相同的请求进行去重。如果在 FETCH 中已经存在了，一般就不需要再发起请求，而是可以复用之前的。除非 opts.dedupe 指明为 false，表示不需要去重。

```ts
const currentFetcher = fetcherRef.current;
// ...
// If there is no ongoing concurrent request, or `dedupe` is not set, a
// new request should be initiated.
const shouldStartNewRequest = !FETCH[key] || !opts.dedupe;
// ...
if (shouldStartNewRequest) {
  // ...
  // Start the request and save the timestamp.
  // Key must be truthy if entering here.
  FETCH[key] = [currentFetcher(fnArg as DefinitelyTruthy<Key>), getTimestamp()];
  // ...
}
// ...
```

getTimestamp 返回的并不是时间戳，而只是一个自增的序号而已。这个序号可以用来区分一段时间内相同 key 的多个请求。

```ts
// Global timestamp.
let __timestamp = 0;

export const getTimestamp = () => ++__timestamp;
```

在请求完成之后，还需处理竞争冲突问题。

```ts
// Wait until the ongoing request is done. Deduplication is also
// considered here.
[newData, startAt] = FETCH[key];
newData = await newData;
```

第一种情况是两个请求之间的冲突，这可能发生在用户指明不需要去重的情况或是在执行 mutate 函数的过程中。先发起的请求让位于后发起的请求。

```ts
// If there're other ongoing request(s), started after the current one,
// we need to ignore the current one to avoid possible race conditions:
//   req1------------------>res1        (current one)
//        req2---------------->res2
// the request that fired later will always be kept.
// The timestamp maybe be `undefined` or a number
if (!FETCH[key] || FETCH[key][1] !== startAt) {
  if (shouldStartNewRequest) {
    if (callbackSafeguard()) {
      getConfig().onDiscarded(key);
    }
  }
  return false;
}
```

第二种情况是 fetch 和 mutate 之间的冲突。因为二者都是异步修改同一个 cache[key]，可能存在执行顺序交错的情况。只要存在这种情况，就要停止更新 data，确保 mutate 操作优先。mutationInfo 数组记录的是 mutate 操作开始和结束的序号。如果还处于 mutate 过程中，mutationInfo[1]为 0；

```ts
// If there're other mutations(s), overlapped with the current revalidation:
// case 1:
//   req------------------>res
//       mutate------>end
// case 2:
//         req------------>res
//   mutate------>end
// case 3:
//   req------------------>res
//       mutate-------...---------->
// we have to ignore the revalidation result (res) because it's no longer fresh.
// meanwhile, a new revalidation should be triggered when the mutation ends.
const mutationInfo = MUTATION[key];
if (
  !isUndefined(mutationInfo) &&
  // case 1
  (startAt <= mutationInfo[0] ||
    // case 2
    startAt <= mutationInfo[1] ||
    // case 3
    mutationInfo[1] === 0)
) {
  finishRequestAndUpdateState();
  if (shouldStartNewRequest) {
    if (callbackSafeguard()) {
      getConfig().onDiscarded(key);
    }
  }
  return false;
}
```

经过这两步验证后，可以安全更新 cache 了。关于错误处理和重试暂且按下不表。

### Auto revalidation

书接上文，有了 revalidate 函数以后，我们可以在恰当的时机获取数据，更新 cache。大概有五种情况下需要更新 cache。

1. revalidateEvents.FOCUS_EVENT 浏览器中当前页面重获焦点
2. revalidateEvents.RECONNECT_EVENT 浏览器重新联网
3. revalidateEvents.MUTATE_EVENT 设置 mutate 操作之后，重新请求数据
4. 组件首次加载时
5. 定时轮询

注意这里的 softRevalidate，它表示在重新刷新 cache 的过程中需要去除重复请求。唯独第三种情况不需要去重，这是因为 mutate 函数内部尝试删除冲突的请求，并且 revalidate 函数内部也会处理冲突。总之为了确保 mutate 操作优先。

```ts
const WITH_DEDUPE = { dedupe: true };
// ...
const softRevalidate = revalidate.bind(UNDEFINED, WITH_DEDUPE);

// Expose revalidators to global event listeners. So we can trigger
// revalidation from the outside.
let nextFocusRevalidatedAt = 0;
const onRevalidate = (type: RevalidateEvent) => {
  if (type == revalidateEvents.FOCUS_EVENT) {
    const now = Date.now();
    if (
      getConfig().revalidateOnFocus &&
      now > nextFocusRevalidatedAt &&
      isActive()
    ) {
      nextFocusRevalidatedAt = now + getConfig().focusThrottleInterval;
      softRevalidate();
    }
  } else if (type == revalidateEvents.RECONNECT_EVENT) {
    if (getConfig().revalidateOnReconnect && isActive()) {
      softRevalidate();
    }
  } else if (type == revalidateEvents.MUTATE_EVENT) {
    return revalidate();
  }
  return;
};
// ...
// Trigger a revalidation.
if (shouldDoInitialRevalidation) {
  if (isUndefined(data) || IS_SERVER) {
    // Revalidate immediately.
    softRevalidate();
  } else {
    // Delay the revalidate if we have data to return so we won't block
    // rendering.
    rAF(softRevalidate);
  }
}
```

代表前三种情况的 onRevalidate 函数将会被添加到 EVENT_REVALIDATORS 中。在前面的内容中，我们提到过 EVENT_REVALIDATORS 会被绑定到对应的事件上，或者在 mutate 函数中被使用。

```ts
const unsubEvents = subscribeCallback(key, EVENT_REVALIDATORS, onRevalidate);
```

最后一种轮询就是通过 setTimeout，无需赘言。

最终，通过从 useSyncExternalStoreWithSelector 中返回的 cached，我们可以将得到的数据返回。

```ts
const cached = useSyncExternalStoreWithSelector();
// ...
const cachedData = cached.data;
const data = isUndefined(cachedData) ? fallback : cachedData;
const error = cached.error;
// ...
const returnedData = keepPreviousData
  ? isUndefined(cachedData)
    ? laggyDataRef.current
    : cachedData
  : data;
// ...
const isValidating = cached.isValidating || defaultValidatingState;
const isLoading = cached.isLoading || defaultValidatingState;
// ...
return {
  mutate: boundMutate,
  get data() {
    stateDependencies.data = true;
    return returnedData;
  },
  get error() {
    stateDependencies.error = true;
    return error;
  },
  get isValidating() {
    stateDependencies.isValidating = true;
    return isValidating;
  },
  get isLoading() {
    stateDependencies.isLoading = true;
    return isLoading;
  },
} as SWRResponse<Data, Error>;
```

### Mutation

mutate 函数可以让我们从外部更新 cache。它有两个来源，useSWR 和 useSWRConfig。两处都是通过 internalMutate 函数来生成 mutate 函数。

mutate 具体的用法可以看[这里](https://swr.vercel.app/zh-CN/docs/mutation)。可以说它的功能是非常强大的。

我们直接看代码，internalMutate 函数的第三，四个入参可以有类型多种类型。依靠这两个多变的入参可以实现[多种功能](https://swr.vercel.app/zh-CN/docs/mutation)。

```ts
export const internalMutate = async <Data>(
  ...args: [
    Cache,
    Key,
    undefined | Data | Promise<Data | undefined> | MutatorCallback<Data>,
    undefined | boolean | MutatorOptions<Data>
  ]
): Promise<Data | undefined> => {
  // ...
};
```

在 internalMutate 内部需要一个重新发起请求更新缓存的方法 startRevalidate，它依靠着 EVENT_REVALIDATORS 中存储的回调来实现这个目的。关于 EVENT_REVALIDATORS 我们上面已经提到过了。在 startRevalidate 内部首先删除了 FETCH 中对应的 key，这也同我们上面提到的 mutate 优先是一个道理。

```ts
const revalidate = options.revalidate !== false;
// ...
const [EVENT_REVALIDATORS, MUTATION, FETCH] = SWRGlobalState.get(
  cache
) as GlobalState;

const revalidators = EVENT_REVALIDATORS[key];
const startRevalidate = () => {
  if (revalidate) {
    // Invalidate the key by deleting the concurrent request markers so new
    // requests will not be deduped.
    delete FETCH[key];
    if (revalidators && revalidators[0]) {
      return revalidators[0](revalidateEvents.MUTATE_EVENT).then(
        () => get().data
      );
    }
  }
  return get().data;
};
```

如果第三个入参存在，表示这次的 mutate 操作要直接改动 cache。我们在 MUTATION 中存入一个数组，表示起讫序号，用于解决同一时间段内多个操作之间的冲突。

```ts
const beforeMutationTs = getTimestamp();
MUTATION[key] = [beforeMutationTs, 0];
```

如果第三个参数是一个 Promise 或者是一个返回 Promise 的函数，就可能会产生冲突。如果两个 mutate 操作冲突了，确保后面的优先。

```ts
// `data` is a promise/thenable, resolve the final data first.
if (data && isFunction((data as Promise<Data>).then)) {
  // This means that the mutation is async, we need to check timestamps to
  // avoid race conditions.
  data = await(data as Promise<Data>).catch((err) => {
    error = err;
  });

  // Check if other mutations have occurred since we've started this mutation.
  // If there's a race we don't update cache or broadcast the change,
  // just return the data.
  if (beforeMutationTs !== MUTATION[key][0]) {
    if (error) throw error;
    return data;
  } else if (error && hasOptimisticData && rollbackOnError) {
    // Rollback. Always populate the cache in this case but without
    // transforming the data.
    populateCache = true;
    data = originalData;
    set({ data: originalData });
  }
}
```

最后在 MUTATION 中记录下完成的序号。尝试调用 startRevalidate，抛出错误或者返回结果。

```ts
// Reset the timestamp to mark the mutation has ended.
MUTATION[key][1] = getTimestamp();

// Update existing SWR Hooks' internal states:
const res = await startRevalidate();

// Throw error or return data
if (error) throw error;
return populateCache ? res : data;
```

### Middleware

> 中间件是 SWR 1.0 中新增的一个功能，它让你能够在 SWR hook 之前和之后执行代码。

swr 的[中间件](https://swr.vercel.app/zh-CN/docs/middleware)给它带来了强大的扩展功能，在内部实现上非常简单。

```ts
export const withArgs = <SWRType>(hook: any) => {
  // ...
  // Apply middleware
  let next = hook;
  const { use } = config;
  if (use) {
    for (let i = use.length; i--; ) {
      next = use[i](next);
    }
  }

  return next(key, fn || config.fetcher, config);
};
```

而它的中间件定义的方式也与 redux 的中间件类似，其实和 express 的中间件本质上也一样。

```ts
function myMiddleware(useSWRNext) {
  return (key, fetcher, config) => {
    // hook 运行之前...

    // 处理下一个中间件，如果这是最后一个，则处理 `useSWR` hook。
    const swr = useSWRNext(key, fetcher, config);

    // hook 运行之后...
    return swr;
  };
}
```

我们就挑选两个官方的中间件来看下，一个简单的，一个复杂的。

首先是这个 useImmutable，它不会在窗口聚焦时、浏览器恢复网络连接时、存在陈旧数据时尝试重新验证 cache。

```ts
import useSWR, { Middleware } from "swr";
import { withMiddleware } from "swr/_internal";

export const immutable: Middleware = (useSWRNext) => (key, fetcher, config) => {
  // Always override all revalidate options.
  config.revalidateOnFocus = false;
  config.revalidateIfStale = false;
  config.revalidateOnReconnect = false;
  return useSWRNext(key, fetcher, config);
};

export default withMiddleware(useSWR, immutable);
```

withMiddleware 的实现并不复杂。仅仅是将 immutable 这个中间件插入到 config.use 数组中。

```ts
// Create a custom hook with a middleware
export const withMiddleware = (
  useSWR: SWRHook,
  middleware: Middleware
): SWRHook => {
  return <Data = any, Error = any>(
    ...args:
      | [Key]
      | [Key, Fetcher<Data> | null]
      | [Key, SWRConfiguration | undefined]
      | [Key, Fetcher<Data> | null, SWRConfiguration | undefined]
  ) => {
    const [key, fn, config] = normalize(args);
    const uses = (config.use || []).concat(middleware);
    return useSWR<Data, Error>(key, fn, { ...config, use: uses });
  };
};
```

第二个中间件 useSWRInfinite 就比较复杂了，甚至需要专门一页篇幅来描述它的[使用方法](https://swr.vercel.app/zh-CN/docs/pagination#useswrinfinite)。

```ts
import useSWRInfinite from 'swr/infinite'

// ...
const { data, error, isValidating, mutate, size, setSize } = useSWRInfinite(
  getKey, fetcher?, options?
)
```

它和原先的 useSWR 相比，有不少的变化，

> getKey 函数是 useSWRInfinite 和 useSWR 之间的主要区别。它接受当前页的索引以及上一页的数据。因此可以很好地支持基于索引和基于游标的分页 API

其次它的返回值中多了一些属性，size、setSize，并且 data 变成了包含多个页面数据的数组。原先的 useSWR 只能满足于单个 api 请求，而分页的需求是要求一系列连续的相关的 api 组合在一起形成一个整体。这个设计就很好的满足了分页的需求。size 表示即将请求和返回的页面数，setSize 用来设置需要请求的页面数。例如我们使用 setSize(n) 的时候，useSWRInfinite 会连续地去取这 10 页数据。

具体的实现是，当前有多少个页面数据同时存在时，就得准备多少个相对应的 cache，因为每个页面对应的 url 肯定不同，即 key 不同。其次还得记录一下当前的页面数之类的 meta 信息，还得在页面数变化时更新组件。

infiniteKey 这个特殊值作为 key，在 cache 中存储着当前页面数。结合我们前面学到的知识，它配合着[useSyncExternalStore](./react/useSyncExternalStore.md)在页面数变化时，也就是调用 setSize 时，更新组件。

```ts
const INFINITE_PREFIX = "$inf$";
// ...
const getFirstPageKey = (getKey: SWRInfiniteKeyLoader) => {
  return serialize(getKey ? getKey(0, null) : null)[0];
};
// ...
// The serialized key of the first page. This key will be used to store
// metadata of this SWR infinite hook.
let infiniteKey: string | undefined;
try {
  infiniteKey = getFirstPageKey(getKey);
  if (infiniteKey) infiniteKey = INFINITE_PREFIX + infiniteKey;
} catch (err) {
  // Not ready yet.
}
const [get, set, subscribeCache] = createCacheHelper<
  Data,
  SWRInfiniteCacheValue<Data, any>
>(cache, infiniteKey);

const getSnapshot = useCallback(() => {
  const size = isUndefined(get().$len) ? initialSize : get().$len;
  return size;
  // eslint-disable-next-line react-hooks/exhaustive-deps
}, [cache, infiniteKey, initialSize]);
useSyncExternalStore(
  useCallback(
    (callback: () => void) => {
      if (infiniteKey)
        return subscribeCache(infiniteKey, () => {
          callback();
        });
      return () => {};
    },
    // eslint-disable-next-line react-hooks/exhaustive-deps
    [cache, infiniteKey]
  ),
  getSnapshot,
  getSnapshot
);
// ...
// Extend the SWR API
const setSize = useCallback(
  (arg: number | ((size: number) => number)) => {
    // ...
    set({ $len: size });
    // ...
    return mutate(resolvePagesFromCache(size));
  },
  // `cache` and `rerender` isn't allowed to change during the lifecycle
  // eslint-disable-next-line react-hooks/exhaustive-deps
  [infiniteKey, resolvePageSize, mutate, cache]
);
```

但这只是页面数的变化，在 setSize 还会接着调用 mutate 函数触发多个页面数据的变化。

我们原先的 useSWRNext 被用来获取页面数据，并且是根据当前的 size，串行地获取 size 个页面的数据。当然不是一定会去重新请求数据，而是根据各种条件来判断，得出 shouldFetchPage 变量来决定是使用旧的数据还是重新请求。

```ts
// Actual SWR hook to load all pages in one fetcher.
const swr = useSWRNext<Data[], Error>(
  infiniteKey,
  async () => {
    // ...
    // return an array of page data
    const data: Data[] = [];
    const pageSize = resolvePageSize();
    // ...
    for (let i = 0; i < pageSize; ++i) {
      // ...
      const [getSWRCacahe, setSWRCache] = createCacheHelper<
        Data,
        SWRInfiniteCacheValue<Data, any>
      >(cache, pageKey);
      // Get the cached page data.
      let pageData = getSWRCacahe().data as Data;
      // ...
      if (fn && shouldFetchPage) {
        pageData = await fn(pageArg);
        setSWRCache({ ...getSWRCacahe(), data: pageData });
      }
      data.push(pageData);
      // ...
    }
    // ...
    // return the data
    return data;
  },
  config
);
```

上面提到的 setSize 中会调用 mutate 函数，而 mutate 中会调用 swr.mutate 函数来修改数据然后重新更新。

```ts
const mutate = useCallback(
  // eslint-disable-next-line func-names
  function (
    data?: undefined | Data[] | Promise<Data[]> | MutatorCallback<Data[]>,
    opts?: undefined | boolean | MutatorOptions<Data[]>
  ) {
    // ...
    // Default to true.
    const shouldRevalidate = options.revalidate !== false;
    // ...
    return arguments.length ? swr.mutate(data, shouldRevalidate) : swr.mutate();
  },
  // swr.mutate is always the same reference
  // eslint-disable-next-line react-hooks/exhaustive-deps
  [infiniteKey, cache]
);
```

最后返回所有的数据。注意这里也是用了 get 方法。原因我们上面提过，注释中也写得很明白。

```ts
// Use getter functions to avoid unnecessary re-renders caused by triggering
// all the getters of the returned swr object.
return {
  size: resolvePageSize(),
  setSize,
  mutate,
  get data() {
    return swr.data;
  },
  get error() {
    return swr.error;
  },
  get isValidating() {
    return swr.isValidating;
  },
  get isLoading() {
    return swr.isLoading;
  },
};
```

可以说 swr 的中间件设计给其带来了相当大的扩展性。
