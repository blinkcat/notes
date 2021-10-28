# redux

master 分支

0dae2d1c6e0bd9a947910ecb223db784018eac8f

## 主要组成部分

1. store 全局唯一权威数据源，内部存储 state，listeners，提供 dispatch 方法
2. reducer 纯函数，接收 state 和 action，返回一个新的 state 或者原 state
3. action 一个普通的 js 对象，用来携带信息，通过 dispatch 方法广播，修改 state
4. middleware 主要用来隔离副作用

总体上是一个采用观察者模式的架构，形成了一个 action -> reducer -> store -> listeners 的单向数据流。

reducer 是纯函数，state 是一个 immutable 对象，而 action 只是一个简单又普通的 js 对象。整体而言实现上就比较简单，唯一难理解的地方就是 middleware 和 storeenhancer。

## middleware

首先要提的就是 compose 函数

```ts
export default function compose(...funcs: Function[]) {
  if (funcs.length === 0) {
    // infer the argument type so it is usable in inference down the line
    return <T>(arg: T) => arg;
  }

  if (funcs.length === 1) {
    return funcs[0];
  }

  return funcs.reduce((a, b) => (...args: any) => a(b(...args)));
}
```

很简单，就是接受一组函数，然后返回一个函数。例如： compose(a,b,c,d) -> (...args)=>a(b(c(d(...args))))。执行这个返回的函数，就等于从右往左执行原函数组，前一个函数执行的结果作为后一个函数的入参。

接着就是 middleware 的概念

```ts
export interface MiddlewareAPI<D extends Dispatch = Dispatch, S = any> {
  dispatch: D;
  getState(): S;
}

export interface Middleware<
  _DispatchExt = {}, // TODO: remove unused component (breaking change)
  S = any,
  D extends Dispatch = Dispatch
> {
  (api: MiddlewareAPI<D, S>): (
    next: D
  ) => (action: D extends Dispatch<infer A> ? A : never) => any;
}
```

一个 middleware 就是一个嵌套了三层的函数。目的是包裹原本的 dispatch，然后返回一个带有特定逻辑的新的 dispatch。

再来就是 applyMiddleware 函数

```ts
// 接收自定义的 middleware 并返回一个 storeEnhancer
export default function applyMiddleware(
  ...middlewares: Middleware[]
): StoreEnhancer<any> {
  return (createStore: StoreEnhancerStoreCreator) => <S, A extends AnyAction>(
    reducer: Reducer<S, A>,
    preloadedState?: PreloadedState<S>
  ) => {
    const store = createStore(reducer, preloadedState);
    // 创建一个会抛出错误的 dispatch ，避免在初始化 middlewares 的时候调用 dispatch
    let dispatch: Dispatch = () => {
      throw new Error(
        "Dispatching while constructing your middleware is not allowed. " +
          "Other middleware would not be applied to this dispatch."
      );
    };

    const middlewareAPI: MiddlewareAPI = {
      getState: store.getState,
      dispatch: (action, ...args) => dispatch(action, ...args),
    };
    const chain = middlewares.map((middleware) => middleware(middlewareAPI));
    // 难点在这里，先用compose函数将 chain，转换为一个函数。
    // 假设 chain=[a, b, c, d]
    // compose(chain) -> (...args)=>a(b(c(d(...args))))
    // 接着将原始的dispatch传入，执行函数，d先执行，返回一个带有d定义的额外逻辑的dispatch函数
    // 再将这个dispatch函数传给c执行，就这样依次执行，最终返回一个包含a,b,c,d所有逻辑的dispatch函数
    dispatch = compose<typeof dispatch>(...chain)(store.dispatch);

    return {
      ...store,
      dispatch,
    };
  };
}
```

经过 applyMiddleware 处理，redux 实现了一个 middleware 的洋葱模型，假设有 4 个 middleware，a,b,c,d。dispatch 一个 action 的时候，action 会这样流动：

a->b->c->d->c->b->a

前提是每个 middleware 的最后一个返回函数中，

```ts
(api: MiddlewareAPI<D, S>): (
    next: D
  ) => (action: D extends Dispatch<infer A> ? A : never) => any; // 这个函数
```

必须在合适的时候调用参数中的 next 方法，才会进入到后一个 middleware 的逻辑中。而唯有最后一个 middleware 的 next 方法，才是原始的 dispatch，才会将 action 送入 reducer 中。

## StoreEnhancer

先看类型定义

```ts
export type StoreEnhancer<Ext = {}, StateExt = never> = (
  next: StoreEnhancerStoreCreator<Ext, StateExt>
) => StoreEnhancerStoreCreator<Ext, StateExt>;
export type StoreEnhancerStoreCreator<Ext = {}, StateExt = never> = <
  S = any,
  A extends Action = AnyAction
>(
  reducer: Reducer<S, A>,
  preloadedState?: PreloadedState<S>
) => Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext;
```

首先 StoreEnhancer 是一个高阶函数，接收一个 StoreEnhancerStoreCreator 返回一个新的 StoreEnhancerStoreCreator，而 StoreEnhancerStoreCreator 执行后返回一个 store。

applyMiddleware 就是一个 StoreEnhancer, 替换 store 的 dispatch 方法后返回 store。

```ts
// ...
const store = createStore(reducer, preloadedState);
// ...
return {
  ...store,
  dispatch,
};
```

在 redux 的 createStore 方法中，可以看到对 storeEnhancer 的处理

```ts
// ...
if (
  (typeof preloadedState === "function" && typeof enhancer === "function") ||
  (typeof enhancer === "function" && typeof arguments[3] === "function")
) {
  throw new Error(
    "It looks like you are passing several store enhancers to " +
      "createStore(). This is not supported. Instead, compose them " +
      "together to a single function."
  );
}
// ...
if (typeof enhancer !== "undefined") {
  if (typeof enhancer !== "function") {
    throw new Error("Expected the enhancer to be a function.");
  }

  return enhancer(createStore)(
    reducer,
    preloadedState as PreloadedState<S>
  ) as Store<ExtendState<S, StateExt>, A, StateExt, Ext> & Ext;
}
```

如果有多个 enhancer，必须先用 compose 函数包在一起。这样在 redux 中，多个 enhancer 也形成了一个洋葱模型。
