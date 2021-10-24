# @rematch/core

版本 [3634f5c2a8cb1a8e98cfc23449c0df55caeec99a](https://github.com/rematch/rematch/tree/3634f5c2a8cb1a8e98cfc23449c0df55caeec99a)

对 redux 做了一层封装，将 action 和 reducer 合并，简化了更新 state 的过程。引入了`Model`结构，对代码组织方式进行强约束，因此对 typescript 友好。同时又可以继承 redux 现有的生态，包括 redux 的 middleware，enhancer。除此之外，还自带插件机制，可扩展能力强。

## Usage

```ts
type PlayersState = {
  players: PlayerModel[];
};

export const players = createModel<RootModel>()({
  state: {
    players: [],
  } as PlayersState,
  reducers: {
    SET_PLAYERS: (state: PlayersState, players: PlayerModel[]) => {
      return {
        ...state,
        players,
      };
    },
  },
  effects: (dispatch) => {
    const { players } = dispatch;
    return {
      async getPlayers(): Promise<any> {
        let response = await fetch("https://www.balldontlie.io/api/v1/players");
        let { data }: { data: PlayerModel[] } = await response.json();
        players.SET_PLAYERS(data);
      },
    };
  },
});
```

首先需要先定义`Model`，这个 model 中有 state、reducers、effects 等属性，分别代表状态，reducer 函数集合，带有副作用的操作。

```ts
import { init, RematchDispatch, RematchRootState } from "@rematch/core";
import { models, RootModel } from "./models";

export const store = init({
  models,
});

export type Store = typeof store;
export type Dispatch = RematchDispatch<RootModel>;
export type RootState = RematchRootState<RootModel>;
```

这里定义了 Store，可以搭配`react-redux`使用。

## How it works

`createModel`的使用方式看起来很奇怪，这是为了更好的利益 typescript 的类型系统。实际上它的代码像这样

```ts
export const createModel: ModelCreator = () => (mo) => mo as any;
```

如果不使用 ts，可以直接定义成一个 Object。

先看下`init`函数，

```ts
export const init = <
  TModels extends Models<TModels>,
  TExtraModels extends Models<TModels> = Record<string, never>
>(
  initConfig?: InitConfig<TModels, TExtraModels>
): RematchStore<TModels, TExtraModels> => {
  const config = createConfig(initConfig || {});
  return createRematchStore(config);
};
```

通过传入的 initConfig，生成一个符合库要求的 config，然后利用这个 config，创建一个 store 返回。

`createConfig`很简单，就是生成一个 config 对象返回，

```ts
export default function createConfig<
  TModels extends Models<TModels>,
  TExtraModels extends Models<TModels>
>(
  initConfig: InitConfig<TModels, TExtraModels>
): Config<TModels, TExtraModels> {
  const storeName = initConfig.name ?? `Rematch Store ${count}`;

  count += 1;

  const config = {
    name: storeName,
    models: initConfig.models || {},
    plugins: initConfig.plugins || [],
    redux: {
      reducers: {},
      rootReducers: {},
      enhancers: [],
      middlewares: [],
      ...initConfig.redux,
      devtoolOptions: {
        name: storeName,
        ...(initConfig.redux?.devtoolOptions ?? {}),
      },
    },
  } as Config<TModels, TExtraModels>;
  // ...
  return config as Config<TModels, TExtraModels>;
}
```

从`config.redux`这个属性可以看出 rematch 和 redux 的关联非常密切。

在得到这个 config 对象后，传入`createRematchStore`，生成一个 store 对象。这是一个漫长的过程，我们一步一步来看。

第一步是创建一个`bag`对象，

```ts
const bag = createRematchBag(config);
```

这个 bag 对象是内部使用的一个对象，里面存储了整个 store 创建流程中的对象。

```ts
export default function createRematchBag<
  TModels extends Models<TModels>,
  TExtraModels extends Models<TModels>
>(config: Config<TModels, TExtraModels>): RematchBag<TModels, TExtraModels> {
  return {
    models: createNamedModels(config.models),
    reduxConfig: config.redux,
    forEachPlugin(method, fn): void {
      config.plugins.forEach((plugin) => {
        if (plugin[method]) {
          fn(plugin[method]!);
        }
      });
    },
    effects: {},
  };
}
```

models 属性存储的是处理过的 config.models，为每个 model 添加了 name、reducers。

```ts
function createNamedModel<TModels extends Models<TModels>>(
  name: string,
  model: Model<TModels>
): NamedModel<TModels> {
  return {
    name,
    reducers: {},
    ...model,
  };
}
```

这些值后面会使用。

bag 中还有一个`forEachPlugin`函数，用来触发 plugin 中对应的钩子。而 effects 对象中是预留放置带副作用的方法的。

第二步，添加一个处理副作用的 middleware，并且触发`createMiddleware`钩子。

```ts
// add middleware for handling effects
bag.reduxConfig.middlewares.push(createEffectsMiddleware(bag));

// collect middlewares from plugins
bag.forEachPlugin("createMiddleware", (createMiddleware) => {
  bag.reduxConfig.middlewares.push(createMiddleware(bag));
});
```

这一个步是收集所有的中间件。

第一个中间件是内部用来处理副作用的

```ts
function createEffectsMiddleware<
  TModels extends Models<TModels>,
  TExtraModels extends Models<TModels>
>(bag: RematchBag<TModels, TExtraModels>): Middleware {
  return (store) =>
    (next) =>
    (action: Action): any => {
      if (action.type in bag.effects) {
        // first run reducer action if exists
        next(action);

        // then run the effect and return its result
        return (bag.effects as any)[action.type](
          action.payload,
          store.getState(),
          action.meta
        );
      }

      return next(action);
    };
}
```

如果一个 action 的 type 在 bag.effects 中，那么会先继续将这个 action 交由下个中间件处理，然后执行事前定义在 effects 对象中的方法。

第三步，开始创建一个 redux store。

```ts
const reduxStore = createReduxStore(bag);
```

要创建 redux store，需要三个参数，第一个是 reducer，第二个是 initialState，第三个是 storeEnhancer。

在`createReduxStore`函数中，我们先要收集所有 model 中的 reducer，然后再将这些 reducer 组合在一起。

```ts
bag.models.forEach((model) => createModelReducer(bag, model));

const rootReducer = createRootReducer<RootState, TModels, TExtraModels>(bag);
```

我们先看`createModelReducer`，它会将 model 中定义的 reducers 对象中方法作为 reducer，将方法名处理后作为 actionName，这个 actionName 之后会当成 action 的 type。

```ts
const modelReducers: ModelReducers<TState> = {};

// build action name for each reducer and create mapping from name to reducer
const modelReducerKeys = Object.keys(model.reducers);
modelReducerKeys.forEach((reducerKey) => {
  const actionName = isAlreadyActionName(reducerKey)
    ? reducerKey
    : `${model.name}/${reducerKey}`;

  modelReducers[actionName] = model.reducers[reducerKey];
});
```

这个`modelReducers`只是一个中间值，还需要通过`combinedReducer`将它变成一个 reducer。

```ts
// select and run a reducer based on the incoming action
const combinedReducer = (
  state: TState = model.state,
  action: Action
): TState => {
  if (action.type in modelReducers) {
    return modelReducers[action.type](
      state,
      action.payload,
      action.meta
      // we use augmentation because a reducer can return void due immer plugin,
      // which makes optional returning the reducer state
    ) as TState;
  }

  return state;
};

const modelBaseReducer = model.baseReducer;

// when baseReducer is defined, run the action first through it
let reducer = !modelBaseReducer
  ? combinedReducer
  : (state: TState = model.state, action: Action): TState =>
      combinedReducer(modelBaseReducer(state, action), action);
```

这里的函数虽然叫 combinedReducer，但其实就是个`switch case`的`map`版本。

最后我们触发`onReducer`钩子，然后将 reducer 存在`bag.reduxConfig.reducers`对象中。

```ts
bag.forEachPlugin("onReducer", (onReducer) => {
  reducer = onReducer(reducer, model.name, bag) || reducer;
});

bag.reduxConfig.reducers[model.name] = reducer;
```

接下来这个`createRootReducer`，就是将`bag.reduxConfig.reducers`对象中的 reducer 函数，通过 redux 中的 combineReducers 合并成一个 rootReducer。

```ts
function mergeReducers<TRootState>(
  reduxConfig: ConfigRedux<TRootState>
): Redux.Reducer<TRootState, Action> {
  const combineReducers = reduxConfig.combineReducers || Redux.combineReducers;

  if (!Object.keys(reduxConfig.reducers).length) {
    return (state: any): TRootState => state;
  }

  return combineReducers(reduxConfig.reducers as Redux.ReducersMapObject);
}
```

在这个过程中，会触发`onRootReducer`钩子。

有了 rootReducer 后，还需要 storeEnhancer。

```ts
const middlewares = Redux.applyMiddleware(...bag.reduxConfig.middlewares);
const enhancers = bag.reduxConfig.devtoolComposer
  ? bag.reduxConfig.devtoolComposer(...bag.reduxConfig.enhancers, middlewares)
  : composeEnhancersWithDevtools(bag.reduxConfig.devtoolOptions)(
      ...bag.reduxConfig.enhancers,
      middlewares
    );
```

先将我们之前收集的中间件合并为一个 enhancer，然后再加入`bag.reduxConfig.enhancers`中添加的 enhancer，通过 redux devtool 提供的方法，或者是 Redux.compose，合并成一个函数。

```ts
function composeEnhancersWithDevtools(
  devtoolOptions: DevtoolOptions = {}
): (...args: any[]) => Redux.StoreEnhancer {
  return !devtoolOptions.disabled &&
    typeof window === "object" &&
    window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__
    ? window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__(devtoolOptions)
    : Redux.compose;
}
```

从`bag.reduxConfig.initialState`中可以得到 initalState，接着将上面的到的 reducer，enhancers 都传入 Redux.createStore 中，

```ts
const createStore = bag.reduxConfig.createStore || Redux.createStore;
const bagInitialState = bag.reduxConfig.initialState;
const initialState = bagInitialState === undefined ? {} : bagInitialState;

return createStore<RootState, Action, any, typeof initialState>(
  rootReducer,
  initialState,
  enhancers
);
```

这样就得到了一个 redux store。

第四步，创建一个`rematchStore`

```ts
let rematchStore = {
  ...reduxStore,
  name: config.name,
  addModel(model: NamedModel<TModels>) {
    validateModel(model);
    createModelReducer(bag, model);
    prepareModel(rematchStore, bag, model);
    reduxStore.replaceReducer(createRootReducer(bag));
    reduxStore.dispatch({ type: "@@redux/REPLACE" });
  },
} as RematchStore<TModels, TExtraModels>;

addExposed(rematchStore, config.plugins);
```

就是对 reduxStore 做了一个扩充。这里的`addExposed`，是将 plugin 中的`exposed`对象里的函数，或是属性值添加到 rematchStore 上，方便 plugin 之间通信。

第五步是更加 model 对象中的 reducers 和 effects，自动创建 action，并且和 dispatch 函数关联起来。

```ts
// generate dispatch[modelName][actionName] for all reducers and effects
bag.models.forEach((model) => prepareModel(rematchStore, bag, model));
```

prepareModel 中会创建一个 modelDispatcher 对象，赋值给 dispatch 函数，然后还会触发`onModel`钩子。

```ts
function prepareModel<
  TModels extends Models<TModels>,
  TExtraModels extends Models<TModels>,
  TModel extends NamedModel<TModels>
>(
  rematchStore: RematchStore<TModels, TExtraModels>,
  bag: RematchBag<TModels, TExtraModels>,
  model: TModel
): void {
  const modelDispatcher = {} as ModelDispatcher<TModel, TModels>;

  // inject model so effects creator can access it
  rematchStore.dispatch[`${model.name}` as keyof RematchDispatch<TModels>] =
    modelDispatcher;

  createDispatcher(rematchStore, bag, model);

  bag.forEachPlugin("onModel", (onModel) => {
    onModel(model, rematchStore);
  });
}
```

modelDispatcher 会在`createDispatcher`函数中进一步加工。在这个函数中，会根据 model 中的 reducers 和 effects 生成 action，然后创建对应的[bindActionCreators](https://redux.js.org/api/bindactioncreators)存入 modelDispatcher 对象中。

```ts
// ...
modelDispatcher[reducerName] = createActionDispatcher(
  rematch,
  model.name,
  reducerName,
  false
);
// ...
bag.effects[`${model.name}/${effectName}`] =
  effects[effectName].bind(modelDispatcher);
// ...
modelDispatcher[effectName] = createActionDispatcher(
  rematch,
  model.name,
  effectName,
  true
);
// ...
const createActionDispatcher = <
  TModels extends Models<TModels>,
  TExtraModels extends Models<TModels>
>(
  rematch: RematchStore<TModels, TExtraModels>,
  modelName: string,
  actionName: string,
  isEffect: boolean
): RematchDispatcher<boolean> => {
  return Object.assign(
    (payload?: any, meta?: any): Action => {
      const action: Action = { type: `${modelName}/${actionName}` };

      if (typeof payload !== "undefined") {
        action.payload = payload;
      }

      if (typeof meta !== "undefined") {
        action.meta = meta;
      }

      return rematch.dispatch(action);
    },
    {
      isEffect,
    }
  );
};
```

注意 effects 中的方法存入了`bag.effects`中。这和我们前面说的中间件可以对应起来。

最后一步，所有的工作已经准备就绪，只需要触发`onStoreCreated`钩子，然后返回 rematchStore。

```ts
bag.forEachPlugin("onStoreCreated", (onStoreCreated) => {
  rematchStore = onStoreCreated(rematchStore, bag) || rematchStore;
});

return rematchStore;
```

最后得到的 rematchStore 可以配合 react-redux 这类的库一起使用。
