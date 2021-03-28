# SizeLimitsPlugin

版本 webpack 5.9.0

webpack 内置 plugin，用于检测 assets 是否超过限制

## Usage

[performance](https://webpack.js.org/configuration/performance/)

## How it works

还算是比较简单的一个 plugin

监听的是 [compiler.hooks.afterEmit](https://webpack.js.org/api/compiler-hooks/#afteremit)，这个阶段的`compilation`里面藏着`build`完成的`assets`。

首先获取所有的`assets`

```js
for (const { name, source, info } of compilation.getAssets()) {
  // ...
}
```

一个`asset`对象的结构为

```js
{
	name: 'main.acb2a349d1.js',
	source: SizeOnlySource { _size: 2036 },
	info: {
		immutable: true,
		contenthash: 'acb2a349d1',
		javascriptModule: false,
		size: 2036
	}
}
```

里面含有`size`信息

```js
const size = info.size || source.size();
```

接着，通过`compilation.entrypoints`这个 Map 可以获取到所有的`entry`信息。我们可以求得一个`entry`的 size

```js
const getEntrypointSize = (entrypoint) => {
  let size = 0;
  for (const file of entrypoint.getFiles()) {
    const asset = compilation.getAsset(file);
    if (
      asset &&
      assetFilter(asset.name, asset.source, asset.info) &&
      asset.source
    ) {
      size += asset.info.size || asset.source.size();
    }
  }
  return size;
};
```

主要是`entrypoint.getFiles()`和`compilation.getAsset(file)`这两个 api。

再来就是判断是否需要提示`warning`或`Error`，这里的提示直接继承了`WebpackError`

```js
module.exports = class AssetsOverSizeLimitWarning extends WebpackError {
  constructor(assetsOverSizeLimit, assetLimit) {
    // ...
  }
};
```

然后放到`compilation`对象中

```js
if (hints === "error") {
  compilation.errors.push(...warnings);
} else {
  compilation.warnings.push(...warnings);
}
```

还有一点，是判断是否有异步`chunk`

```js
const someAsyncChunk = find(
  compilation.chunks,
  (chunk) => !chunk.canBeInitial()
);
```

关于两种 [chunk](https://webpack.js.org/concepts/under-the-hood/#chunks)的定义，适当使用异步`chunk`可以减少首次加载的时间，间接提高 performance。
