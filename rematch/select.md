# @rematch/select

版本 [v3.1.1](https://github.com/rematch/rematch/tree/%40rematch/select%403.1.1)

用来提供 selector 函数

## Usage

```ts
const model = {
  name: "cart",
  state: [
    {
      price: 42.0,
      amount: 3,
      productId: 2,
    },
  ],
  selectors: (slice, createSelector, hasProps) => ({
    // creates a simple memoized selector based on the cart state
    total() {
      return slice((cart) => cart.reduce((a, b) => a + b.price * b.amount, 0));
    },
    // uses createSelector method to create more complex memoized selector
    totalWithShipping() {
      return createSelector(
        slice, // shortcut for (rootState) => rootState.cart
        (rootState, props) => props.shipping,
        (cart, shipping) =>
          cart.reduce((a, b) => a + b.price * b.amount, shipping)
      );
    },
    // refers to the other selector from this model
    doubleTotal() {
      return createSelector(
        this.totalWithShipping,
        (totalWithShipping) => totalWithShipping * 2
      );
    },
    // accesses selector from a different model
    productsPopularity(models) {
      return createSelector(
        slice, // shortcut for (rootState) => rootState.cart
        models.popularity.pastDay, // gets 'pastDay' selector from 'popularity' model
        (cart, hot) => cart.sort((a, b) => hot[a.productId] > hot[b.productId])
      );
    },
    // uses hasProps function, which returns new selector for each given lowerLimit prop
    expensiveFilter: hasProps(function (models, lowerLimit) {
      return slice((items) => items.filter((item) => item.price > lowerLimit));
    }),
    // uses expensiveFilter selector to create a new selector where lowerLimit is set to 20.00
    wouldGetFreeShipping() {
      return this.expensiveFilter(20.0);
    },
  }),
};
```

> When called as a function, select lazily creates a structuredSelector using the selectors you return in mapSelectToStructure.

```ts
const selection = store.select((models) => ({
  total: models.cart.total,
  eligibleItems: models.cart.wouldGetFreeShipping,
}));
// it can be used as 'mapStateToProps'
connect(selection)(MyComponent);
// or
connect((state) => ({
  contacts: state.contacts.collection,
  ...selection(state),
}))(MyComponent);
```

> select is also an object with a group of selectors for each of your store models. Selectors are regular functions that can be called anywhere within your application.

```ts
const moreThan50 = store.select.cart.expensiveFilter(50.0);

console.log(moreThan50(store.getState()));

const mapStateToProps = (state) => ({
  items: moreThan50(state),
});
```

## How it works

这次的代码涉及到函数嵌套，有些难理解。

首先要对[reselect](../reselect.md)有些了解，它导出了两个特别常用的函数，`createSelector`和`createStructuredSelector`。

- createSelector 一般至少传入两个函数，前面函数的执行结果作为最后一个函数的入参。并且在入参不变的情况下，不再执行最后一个函数，只是返回上次执行的结果。
- createStructuredSelector 大致一样，只不过接受的参数是一个 object，里面的属性值是函数，执行后返回的结果形状和原 object 相同，属性值是原属性值执行后的结果。

从这个插件的使用方法来看，在 model 中添加`selectors`属性，对应一个接收三个参数的函数，这个函数返回一个对象，里面包含一些 selector 函数。

我们可以先从参数看起，

```ts
const selectorFactories =
  typeof model.selectors === "function"
    ? // @ts-ignore
      model.selectors(slice(model), selectorCreator, hasProps)
    : model.selectors;
```

`slice`这个函数不是很好理解，实际上在内部，是先传入了当前的 model，然后使用它的返回结果。

```ts
const sliceState: SelectConfig<TModels, TExtraModels>["sliceState"] =
  config.sliceState || ((state, model) => state[model.name || ""]);
const selectorCreator = config.selectorCreator || createSelector;

const slice =
  (model: Model<TModels>) =>
  (stateOrNext: ExtractRematchStateFromModels<TModels, TExtraModels>) => {
    if (typeof stateOrNext === "function") {
      return selectorCreator(
        (state: ExtractRematchStateFromModels<TModels, TExtraModels>) =>
          sliceState(state, model),
        stateOrNext
      );
    }
    return sliceState(stateOrNext, model);
  };
```

如果接下来再传入的是一个函数，那么返回的是一个被`createSelector`包装过的带缓存功能的函数。这个返回的函数会通过`sliceState`函数获取当前的 model 的 state，然后再传给`stateOrNext`函数执行。

如果传入的不是一个函数，那似乎只有传入 rootState 才有意义。

selectorCreator 默认就是 createSelector。

而`hasProps`比较特别，从上面例子中可以看出，它是在定义的时候直接执行了。

```ts
const hasProps = (inner: any) =>
  function (this: any, models: any) {
    return selectorCreator(
      (props: any) => props,
      (props: any) => inner.call(this, models, props)
    );
  };
```

返回了一个函数，接收 models，这个 models 我们后面会提到。

接着我们再来看 2 个重要的变量。

第一个是`select`函数，

```ts
const makeSelect = () => {
  /**
   * Maps models to structured selector
   * @param  mapSelectToStructure function that gets passed `selectors` and returns an object
   * @param  structuredSelectorCreator=createStructuredSelector if you need to provide your own implementation
   *
   * @return the result of calling `structuredSelectorCreator` with the new selectors
   */
  function select(
    mapSelectToStructure: any,
    structuredSelectorCreator = createStructuredSelector
  ) {
    let func = (state: any, props: any): any => {
      func = structuredSelectorCreator(mapSelectToStructure(select));
      return func(state, props);
    };

    return (state: any, props: any) => func(state, props);
  }

  return select;
};
// ...
const select = makeSelect();
```

makeSelect 执行后返回一个 select 函数，这个函数最后会合入 store 中，也就有了我们上面例子中的 store.select 方法。

这个 select 函数的实现有些不同寻常，它里面的 func 函数在自己的内部又会被重新赋值，而且 select 又被作为参数传入到 mapSelectToStructure 函数中。

看似不合理，但是运行起来一切正常。首先插件会将所有的 model 中的选择函数按照 model 名称分类，赋值给 select 函数。列如`store.select.cart.total`、`store.select.cart.totalWithShipping`等。接着根据注释，mapSelectToStructure 是一个函数，返回一个 object，这个 object 传给 structuredSelectorCreator 函数，默认就是`createStructuredSelector`，根据我们上面介绍的[reselect](../reselect.md)，这个函数又会返回一个函数，赋值给了 func。最后返回`func(state, props)`。

第二个重要的变量是

```ts
const factoryGroup = makeFactoryGroup();
// ...
const makeFactoryGroup = () => {
  let ready = false;
  const factories = new Set();
  return {
    add(added: any) {
      if (!ready) {
        added.forEach((factory: any) => factories.add(factory));
      } else {
        added.forEach((factory: any) => factory());
      }
    },
    finish(factory: any) {
      factories.delete(factory);
    },
    startBuilding() {
      ready = true;
      factories.forEach((factory: any) => factory());
    },
  };
};
```

这块的实现很直接，没啥好说的。接下来才会用到它。

插件内部监听了`onModel`钩子，改造每个有 selectors 属性的 model。

```ts
return {
  // ...
  onModel(model: Model<TModels>) {
    // @ts-ignore
    select[model.name] = {};

    const selectorFactories =
      typeof model.selectors === "function"
        ? // @ts-ignore
          model.selectors(slice(model), selectorCreator, hasProps)
        : model.selectors;

    factoryGroup.add(
      Object.keys(selectorFactories || {}).map((selectorName: string) => {
        validateSelector(selectorFactories, selectorName, model);

        const factory = () => {
          factoryGroup.finish(factory);
          // @ts-ignore
          delete select[model.name][selectorName];
          // @ts-ignore
          select[model.name][selectorName] = selectorFactories[
            selectorName
            // @ts-ignore
          ].call(select[model.name], select);
          // @ts-ignore
          return select[model.name][selectorName];
        };

        // Define a getter for early constructing
        // @ts-ignore
        Object.defineProperty(select[model.name], selectorName, {
          configurable: true,
          get() {
            return factory();
          },
        });

        return factory;
      })
    );
  },
  onStoreCreated(store: RematchStore<TModels, TExtraModels>) {
    factoryGroup.startBuilding();
    // @ts-ignore
    store.select = select;
  },
};
```

如果 `model.selectors`是函数，会传入上面说的那三个函数得到一个 selector 对象。接着就开始遍历这个 selectorFactories 对象。

主要的逻辑都在`factory`函数中，这个函数会被添加到 factoryGroup 内部的 Set 中，最后再`onStoreCreated`钩子中统一执行。但是也可以在此之前，通过获取`select[model.name].selectorName`的方式调用。

`factory`函数执行时，会先将自己从 Set 中移除，然后 delete 在 Object.defineProperty 方法中定义的属性值，再通过执行`selectorFactories[selectorName].call(select[model.name], select)`创建一个新的值，最后返回。特别要注意这里的执行，这说明我们在 model 的 selectors 属性中定义的函数都会传入 select 函数。当然它在语义上不是作为一个函数，而是作为代表多个 model 中 selector 方法的集合。这也就能解释了`hasProps`函数为何要这样使用。

总得来说，这个插件实现地还是很精悍的。
