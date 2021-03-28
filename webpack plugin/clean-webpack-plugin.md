# clean-webpack-plugin

版本 [64c424162507ed97a9fe4209d5ebfb1176f50860](https://github.com/johnagan/clean-webpack-plugin/commit/64c424162507ed97a9fe4209d5ebfb1176f50860)

每次构建用来清除 output 指定文件夹中的内容

在 webpack5 中已经内置 clean 功能，不再需要这个 plugin

## How it works

首先要定位用户配置的 output 的位置，这个可以直接从`compiler`对象中找到。

```js
this.outputPath = compiler.options.output.path;
```

第一个需求，在首次 build 前删除 output 文件夹中的文件，这里选择监听的是[compiler.hooks.emit](https://webpack.js.org/api/compiler-hooks/#emit)，也就是将 `assets` 输出到 `output`文件夹之前。

这里算是个绝佳的事件点，如果在更靠前的位置，则不能保证 build 过程一定会走到`emit`这一步。

```js
this.cleanOnceBeforeBuildPatterns = Array.isArray(
  options.cleanOnceBeforeBuildPatterns
)
  ? options.cleanOnceBeforeBuildPatterns
  : ["**/*"];
```

默认是删除所有文件，这里要注意删除操作的上下文，就是之前得到的`outputPath`。

```js
removeFiles(patterns: string[]) {
	// ...
	const deleted = delSync(patterns, {
		// ...
		// Change context to build directory
		cwd: this.outputPath,
		// ...
	});
}
```

然后直接删除就是了。

第二个需求是在 build 完成后，删除一些多余的文件。注意这里的 build 可能是在 watch 模式下。

这里选择的时间点是[done](https://webpack.js.org/api/compiler-hooks/#done)。这个时候`assets`已经被 emit 到`output`文件夹，也是个很好的时间点。如果我们没有配置文件 hash，那么同名的会直接覆盖。即使有 hash，我们也只需要删除`stale`状态的文件。

我们通过`stats`对象拿到本次的`assets`

```js
const assets =
  stats.toJson(
    {
      assets: true,
    },
    true
  ).assets || [];
```

以`asset.name`为 key，来判断文件是否改变。每次的`assets`文件名存在`currentAssets`中，留着判断`stale`状态的文件。

```js
const assetList = assets.map((asset: { name: string }) => {
  return asset.name;
});

const staleFiles = this.currentAssets.filter((previousAsset) => {
  const assetCurrent = assetList.includes(previousAsset) === false;

  return assetCurrent;
});

this.currentAssets = assetList.sort();
```

接着收集所有不再需要的文件进行删除操作

```js
const removePatterns = [];

/**
 * Remove unused webpack assets
 */
if (this.cleanStaleWebpackAssets === true && staleFiles.length !== 0) {
  removePatterns.push(...staleFiles);
}

/**
 * Remove cleanAfterEveryBuildPatterns
 */
if (this.cleanAfterEveryBuildPatterns.length !== 0) {
  removePatterns.push(...this.cleanAfterEveryBuildPatterns);
}

if (removePatterns.length !== 0) {
  this.removeFiles(removePatterns);
}
```

总体来说还是属于 plugin 的简单应用。

## Details

提供 dryRun 功能，对这种带有危险的删除操作很有用。

提供 verbose 功能，可以打印出详细的日志。

提供 dangerouslyAllowCleanPatternsOutsideProject 参数，防止随意删除项目外的文件。
