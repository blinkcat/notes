# re-reselect

版本 [v4.0.0](https://github.com/toomuchdesign/re-reselect/tree/v4.0.0)

基于 [reselect](./reselect.md) 改造，主要解决 reselect 缓存空间太小造成的问题。
`createSelectorCreator`保持一致。

```js

```

## Usage

```js
import { createCachedSelector } from "re-reselect";

// Normal reselect routine: declare "inputSelectors" and "resultFunc"
const getUsers = (state) => state.users;
const getLibraryId = (state, libraryName) => state.libraries[libraryName].id;

const getUsersByLibrary = createCachedSelector(
  // inputSelectors
  getUsers,
  getLibraryId,

  // resultFunc
  (users, libraryId) => expensiveComputation(users, libraryId)
)(
  // re-reselect keySelector (receives selectors' arguments)
  // Use "libraryName" as cacheKey
  (_state_, libraryName) => libraryName
);

// Cached selectors behave like normal selectors:
// 2 reselect selectors are created, called and cached
const reactUsers = getUsersByLibrary(state, "react");
const vueUsers = getUsersByLibrary(state, "vue");

// This 3rd call hits the cache
const reactUsersAgain = getUsersByLibrary(state, "react");
// reactUsers === reactUsersAgain
// "expensiveComputation" called twice in total
```

和原来的 reselect 用法差不多，只不过要多传入一个函数或是对象来提供 `cache key`。

## How it works

主要还是在于`createCachedSelector`这个函数，它也是一个工厂方法。

```js
function createCachedSelector(...funcs) {
  return (polymorphicOptions, legacyOptions) => {
    // ...
  };
}
```

第一部分主要处理了参数，

```js
const options =
  typeof polymorphicOptions === "function"
    ? { keySelector: polymorphicOptions }
    : Object.assign({}, polymorphicOptions);

// https://github.com/reduxjs/reselect/blob/v4.0.0/src/index.js#L54
let recomputations = 0;
const resultFunc = funcs.pop();
const dependencies = Array.isArray(funcs[0]) ? funcs[0] : [...funcs];

const resultFuncWithRecomputations = (...args) => {
  recomputations++;
  return resultFunc(...args);
};
funcs.push(resultFuncWithRecomputations);

const cache = options.cacheObject || new defaultCacheCreator();
const selectorCreator = options.selectorCreator || createSelector;
const isValidCacheKey = cache.isValidCacheKey || defaultCacheKeyValidator;

if (options.keySelectorCreator) {
  options.keySelector = options.keySelectorCreator({
    keySelector: options.keySelector,
    inputSelectors: dependencies,
    resultFunc,
  });
}
```

cache 部分使用了默认的`FlatObjectCache`。这个`keySelector`就是比 reselect 多传入的函数，用来确定 cache key，获取存在 cache 中的 selector 函数。

第二部分，也是最核心的部分，包装一个 selector 函数

```js
// Application receives this function
const selector = function (...args) {
  const cacheKey = options.keySelector(...args);

  if (isValidCacheKey(cacheKey)) {
    let cacheResponse = cache.get(cacheKey);

    if (cacheResponse === undefined) {
      cacheResponse = selectorCreator(...funcs);
      cache.set(cacheKey, cacheResponse);
    }

    return cacheResponse(...args);
  }
  console.warn(
    `[re-reselect] Invalid cache key "${cacheKey}" has been returned by keySelector function.`
  );
  return undefined;
};
```

这个 selector 函数根据入参，生成一个 cache key，如果 key 存在于 cache 中，取出来执行，否则调用`selectorCreator`创建一个，然后存起来。

最后一部分没什么，在 selector 函数上加上一些方法，和 reselect 保持一致。

```js
// Further selector methods
selector.getMatchingSelector = (...args) => {
  const cacheKey = options.keySelector(...args);
  // @NOTE It might update cache hit count in LRU-like caches
  return cache.get(cacheKey);
};

selector.removeMatchingSelector = (...args) => {
  const cacheKey = options.keySelector(...args);
  cache.remove(cacheKey);
};

selector.clearCache = () => {
  cache.clear();
};

selector.resultFunc = resultFunc;

selector.dependencies = dependencies;

selector.cache = cache;

selector.recomputations = () => recomputations;

selector.resetRecomputations = () => (recomputations = 0);

selector.keySelector = options.keySelector;

return selector;
```

可以看出，这里最关键的是 cache key 的生成策略，以及 cache 的实现策略。

cache key 的生成一定要抓住入参的特征。

cache 有多种实现可以选择，Fifo、Flat、Lru 等等。

最后，还有一个`createStructuredCachedSelector`函数，和 reselect 中的`createSelectorCreator`保持一致。

```js
import { createStructuredSelector } from "reselect";
import createCachedSelector from "./createCachedSelector";

function createStructuredCachedSelector(selectors) {
  return createStructuredSelector(selectors, createCachedSelector);
}

export default createStructuredCachedSelector;
```

实现上非常简洁，这也是因为 reselect 本身就有很好的扩展性。
