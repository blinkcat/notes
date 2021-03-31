# sentry-webpack-plugin

版本 [1.14.2](https://github.com/getsentry/sentry-webpack-plugin/tree/v1.14.2)

用来上传 sourcemap 到 sentry 服务器

## How It Works

主要是依靠`@sentry/cli`来上传文件，那么监听的事件点必然是选择`compiler.hooks.afterEmit`，然后利用`cli`对象来收集文件然后上传，直接了当。

其中有一个需求是需要在全局对象中留下一个变量`SENTRY_RELEASE`，表示 release 的版本号。这个版本号是通过`cli`异步获得的一个值。

这里的做法是在`entry`中插入一个空文件`sentry-webpack.module.js`

```js
const SENTRY_MODULE = path.resolve(__dirname, "sentry-webpack.module.js");
// ...
options.entry = this.injectEntry(options.entry, SENTRY_MODULE);
```

这里`injectEntry`的实现还是很值得借鉴的，兼容了 webpack3、webpack4+

然后再插入一个`loader`

```js
const SENTRY_LOADER = path.resolve(__dirname, 'sentry.loader.js');
// ...
injectRule(rules) {
	const rule = {
		test: /sentry-webpack\.module\.js$/,
		use: [
		{
			loader: SENTRY_LOADER,
			options: {
			releasePromise: this.release,
			},
		},
		],
	};

	return (rules || []).concat([rule]);
}
```

这个`loader`专门用来处理那个空文件，在里面插入脚本。

```js
module.exports = function sentryLoader(content, map, meta) {
  const { releasePromise } = this.query;
  const callback = this.async();
  releasePromise.then((version) => {
    const sentryRelease = `(typeof window !== 'undefined' ? window : typeof global !== 'undefined' ? global : typeof self !== 'undefined' ? self : {}).SENTRY_RELEASE={id:"${version}"};`;
    callback(null, sentryRelease, map, meta);
  });
};
```

这是一个[异步 loader](https://webpack.js.org/api/loaders/#asynchronous-loaders)，这个`releasePromise`就是上面配置传入的`this.release`

```js
this.release = this.getReleasePromise();
// ...
getReleasePromise() {
return (this.options.release
	? Promise.resolve(this.options.release)
	: this.cli.releases.proposeVersion()
)
	.then(version => `${version}`.trim())
	.catch(() => undefined);
}
```

就是调用`cli`的到的一个 promise，最终得到了想要的结果。

算是一个巧妙的 plugin，在参数方面，也有`slient`，`dryrun`这种参数方便使用。
