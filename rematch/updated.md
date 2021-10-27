# @rematch/updated

版本 [v2.1.1](https://github.com/rematch/rematch/tree/%40rematch/updated%402.1.1)

用来记录 effects 中的方法被调用的时间，可以依据这个时间阻止昂贵的 effect 频繁地触发。

## Usage

```ts
// store.js
import updatedPlugin from "@rematch/updated";
import { init } from "@rematch/core";
import * as models from "./models";
init({
  models,
  plugins: [updatedPlugin()],
});
// some-model.js
export const count = {
	...,
    effects: {
      async fetchOne() {},
      async fetchTwo() {},
    }
}
// someView.jsx
const state = store.getState();
// or just connect() on `react-redux`
console.log(state.updated.count.fetchOne);
console.log(state.updated.count.fetchTwo);
```

## How it works

整体的思路是这样，通过`config`钩子，添加一个新的`model`

```ts
const updated = {
	name: updatedModelName,
	state: {} as Record<string, any>,
	reducers: {
		onUpdate: (
			state: UpdatedState<TModels, T>,
			payload: { name: string; action: string }
		): UpdatedState<TModels, T> => ({
			...state,
			[payload.name]: {
				...state[payload.name],
				[payload.action]: config.dateCreator
					? config.dateCreator()
					: new Date(),
			},
		}),
	},
}
// ...
config: {
	models: {
		updated: updated as UpdatedModel<TModels, T>,
	},
}
```

这个时候，新 model 的 state 是空的，因为它的属性需要在`onModel`钩子执行的时候，动态加入。

```ts
onModel({ name }, rematch): void {
	// do not run dispatch on updated model and blacklisted models
	if (avoidModels.includes(name)) {
		return
	}

	const modelActions = rematch.dispatch[name]

	// add empty object for effects
	updated.state[name] = {}
	// ...
}
```

接着需要对 dispatch 中的 effect action 做 monkey patch。

```ts
// map over effects within models
for (const action of Object.keys(modelActions)) {
  if (rematch.dispatch[name][action].isEffect) {
    // copy function
    const originalDispatcher = rematch.dispatch[name][action];

    // replace existing effect with new dispatch
    rematch.dispatch[name][action] = (...props: any): any => {
      const effectResult = originalDispatcher(...props);
      // check if result is a promise
      if (effectResult?.then) {
        effectResult.then((result: any) => {
          // set updated when promise finishes
          rematch.dispatch[updatedModelName].onUpdate({ name, action });
          return result;
        });
      } else {
        // no need to wait for the result, as it's not a promise
        rematch.dispatch[updatedModelName].onUpdate({ name, action });
      }

      return effectResult;
    };
  }
}
```

原来的`originalDispatcher`被替换为一个新的函数，这个函数除了内部会调用 originalDispatcher，
还会额外调用`rematch.dispatch[updatedModelName].onUpdate`来更新 updated 相关的 state。

这种实现方式似乎是一种反模式，使得代码的运行时的逻辑更加复杂。
