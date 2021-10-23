# @rematch/core

版本 [@rematch/core@2.1.1](https://github.com/rematch/rematch/tree/%40rematch/core%402.1.1)

对 redux 做了一层封装，将 action 隐藏入 reducer，简化了触发 store 更新 state 的过程。同时引入了`Model`结构，对代码组织方式进行强约束，对 typescript 友好，但是又可以继承 redux 现有的生态，包括 redux 能用的 middleware，enhancer。除此之外，还带有插件机制，可扩展性强。

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

首先需要先定义`Model`，这个 model 中有 state、reducers、effects 等，分别代表状态，reducer 函数集合，带有副作用的操作。

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

## How it works
