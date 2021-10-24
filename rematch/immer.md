# @rematch/immer

版本 [v2.1.2](https://github.com/rematch/rematch/tree/%40rematch/immer%402.1.2)

rematch 的 immer 插件

## Usage

```ts
// store.ts
import immerPlugin from "@rematch/immer";
import { init, RematchDispatch, RematchRootState } from "@rematch/core";

export const store = init<RootModel>({
  models,
  // add immerPlugin to your store
  plugins: [immerPlugin()],
});

// todo.ts
import { createModel } from "@rematch/core";
import { RootModel } from "./models";

export const todo = createModel<RootModel>()({
  state: [
    {
      todo: "Learn typescript",
      done: true,
    },
    {
      todo: "Try immer",
      done: false,
    },
  ],
  reducers: {
    done(state) {
      // mutable changes to the state
      state.push({ todo: "Tweet about it", done: false });
      state[1].done = true;
    },
  },
});
```

加入插件后，可以在 reducer 中直接修改 state。

## How it works

```ts
function wrapReducerWithImmer(reducer: Redux.Reducer) {
  return (state: any, payload: any): any => {
    if (state === undefined) return reducer(state, payload);
    return produce(state, (draft: any) => reducer(draft, payload));
  };
}

const immerPlugin = <
  TModels extends Models<TModels>,
  TExtraModels extends Models<TModels> = Record<string, never>
>(
  config?: ImmerPluginConfig
): Plugin<TModels, TExtraModels> => ({
  onReducer(reducer: Redux.Reducer, model: string): Redux.Reducer | void {
    if (
      !config ||
      (!config.whitelist && !config.blacklist) ||
      (config.whitelist && config.whitelist.includes(model)) ||
      (config.blacklist && !config.blacklist.includes(model))
    ) {
      return wrapReducerWithImmer(reducer);
    }

    return undefined;
  },
});

export default immerPlugin;
```

代码不多，在构造函数中可以传入黑名单，白名单。使用的钩子是`onReducer`，在每个 model 的 reducer 创建完成后会触发。在拿到 reducer 后，使用 wrapReducerWithImmer 包装一下。注意函数入参里那个 payload 应该改名为 action 才比较合理。原始的 reducer 被 immer 的 produce 方法包装，以保证在 reducer 内部可以在直接修改 draft 对象后，自动应用于 state。
