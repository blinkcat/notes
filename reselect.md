# reselect

版本 [v4.0.0](https://github.com/reduxjs/reselect/tree/v4.0.0)

提供一种带 cache 的 selector，selector 是 redux 中的概念，是一种用来从 state 对象中提取某些属性值的纯函数。具有如下的签名，

```js
(state) => stateSlice;
```

state 作为参数之一，在得到结果的过程中可能包含一些复杂昂贵的计算。所以要求尽量减少重复计算，缓存之前的计算结果。在入参不变的情况下，直接返回缓存的结果。

## Usage

```js
import { createSelector } from "reselect";

const shopItemsSelector = (state) => state.shop.items;
const taxPercentSelector = (state) => state.shop.taxPercent;

const subtotalSelector = createSelector(shopItemsSelector, (items) =>
  items.reduce((subtotal, item) => subtotal + item.value, 0)
);

const taxSelector = createSelector(
  subtotalSelector,
  taxPercentSelector,
  (subtotal, taxPercent) => subtotal * (taxPercent / 100)
);

const totalSelector = createSelector(
  subtotalSelector,
  taxSelector,
  (subtotal, tax) => ({ total: subtotal + tax })
);

const exampleState = {
  shop: {
    taxPercent: 8,
    items: [
      { name: "apple", value: 1.2 },
      { name: "orange", value: 0.95 },
    ],
  },
};

console.log(subtotalSelector(exampleState)); // 2.15
console.log(taxSelector(exampleState)); // 0.172
console.log(totalSelector(exampleState)); // { total: 2.322 }
```

这个例子展示了`createSelector`函数的使用方法，以及 selector 函数的组合能力。

## How it works

想要方便的缓存数据，闭包是一种选择。整个库也是围绕着闭包来提供缓存入参，缓存计算值的能力。

关键的导出在这一行，

```js
export const createSelector = createSelectorCreator(defaultMemoize);
```

先看这个`defaultMemoize`，它提供了缓存功能

```js
export function defaultMemoize(func, equalityCheck = defaultEqualityCheck) {
  let lastArgs = null;
  let lastResult = null;
  // we reference arguments instead of spreading them for performance reasons
  return function () {
    if (!areArgumentsShallowlyEqual(equalityCheck, lastArgs, arguments)) {
      // apply arguments instead of spreading for performance.
      lastResult = func.apply(null, arguments);
    }

    lastArgs = arguments;
    return lastResult;
  };
}
```

这是一个高阶函数，传入一个函数，然后返回一个可以缓存计算结果的新函数。它在内部会先比较参数是否有变化，然后决定是否要重新执行。

这个默认的判断是否相等的函数`defaultEqualityCheck`很简单，

```js
function defaultEqualityCheck(a, b) {
  return a === b;
}
```

直接用`===`比较两个变量。

而用来比较本次入参是否和上次执行时的入参相等的函数`areArgumentsShallowlyEqual`也算简单

```js
function areArgumentsShallowlyEqual(equalityCheck, prev, next) {
  if (prev === null || next === null || prev.length !== next.length) {
    return false;
  }

  // Do this in a for loop (and not a `forEach` or an `every`) so we can determine equality as fast as possible.
  const length = prev.length;
  for (let i = 0; i < length; i++) {
    if (!equalityCheck(prev[i], next[i])) {
      return false;
    }
  }

  return true;
}
```

先判断之前有没有过缓存，或者是两个 arguments 对象的长度是否一致。如果过了这关，再逐项比较是否相等。

接下来看看这个`createSelectorCreator`做了什么，

```js
export function createSelectorCreator(memoize, ...memoizeOptions) {
  return (...funcs) => {
    let recomputations = 0;
    const resultFunc = funcs.pop();
    const dependencies = getDependencies(funcs);

    const memoizedResultFunc = memoize(function () {
      recomputations++;
      // apply arguments instead of spreading for performance.
      return resultFunc.apply(null, arguments);
    }, ...memoizeOptions);

    // If a selector is called with the exact same arguments we don't need to traverse our dependencies again.
    const selector = memoize(function () {
      const params = [];
      const length = dependencies.length;

      for (let i = 0; i < length; i++) {
        // apply arguments instead of spreading and mutate a local list of params for performance.
        params.push(dependencies[i].apply(null, arguments));
      }

      // apply arguments instead of spreading for performance.
      return memoizedResultFunc.apply(null, params);
    });

    selector.resultFunc = resultFunc;
    selector.dependencies = dependencies;
    selector.recomputations = () => recomputations;
    selector.resetRecomputations = () => (recomputations = 0);
    return selector;
  };
}
```

从函数名可以看出，它是一个工厂函数，返回一个带有特定的缓存策略的`createSelector`函数。因为默认传入的是`defaultMemoize`函数，所以默认的缓存空间是 1。

createSelector 的入参一般为几个函数，或者是一个函数数组，再加最后一个计算函数。所以开头先处理参数，得到了最终用于计算的函数`resultFunc`，以及这个计算函数所有的入参来源`dependencies`。

接着，通过`memoize`函数，也就是默认的`defaultMemoize`函数生成一个缓存版的`resultFunc`函数，
`memoizedResultFunc`。然后再通过 memoize 函数，生成最终返回的`selector`函数。在这个函数里，会先通过`dependencies`函数数组，计算出一组值作为入参，执行`memoizedResultFunc`函数，最后返回执行结果。

总结一下，这里总共做了两次缓存。第一次是为了判断是否需要重新执行 dependencies 函数数组，收集入参。第二次是判断这些入参是否有变化，需不需要再执行 resultFunc 函数。

除此之外，还有一个版本的函数`createStructuredSelector`，

```js
const mySelectorA = (state) => state.a;
const mySelectorB = (state) => state.b;

const structuredSelector = createStructuredSelector({
  x: mySelectorA,
  y: mySelectorB,
});

const result = structuredSelector({ a: 1, b: 2 }); // will produce { x: 1, y: 2 }
```

这实际上可以看做`selectorCreator`函数的变种用法，其实内部也是通过 selectorCreator 函数实现。

```js
export function createStructuredSelector(
  selectors,
  selectorCreator = createSelector
) {
  if (typeof selectors !== "object") {
    throw new Error(
      "createStructuredSelector expects first argument to be an object " +
        `where each property is a selector, instead received a ${typeof selectors}`
    );
  }
  const objectKeys = Object.keys(selectors);
  return selectorCreator(
    objectKeys.map((key) => selectors[key]),
    (...values) => {
      return values.reduce((composition, value, index) => {
        composition[objectKeys[index]] = value;
        return composition;
      }, {});
    }
  );
}
```

可以从代码中看到，先是将对象中的函数收集起来，外加一个将这个函数的执行结果再拼装回去的函数。

总的来说，`reselect`是一个很好的库，在 redux 中可以用来介绍很多不必要的计算，进而减少很多不必要的渲染。但是它还是有一个很大的缺点。前面我们提到它的缓存空间只有 1。这意味着，如果在多个组件中使用同一个 selector 函数，入参很可能不一样，这时候，selector 的缓存可能会一直失效，每次都重新计算。这就需要我们为每个组件提供一个自己的 selector 函数。而这无疑会增加维护成本。
