# babel-loader

版本 [8.2.2](https://github.com/babel/babel-loader/tree/v8.2.2)

利用 babel 将匹配的文件转成 支持 es5 的文件

## How it works

想要这个 loader 的工作原理需要先了解很多其他知识

1. [babel 配置](https://babeljs.io/docs/en/options)
2. [@babel/core](https://babeljs.io/docs/en/babel-core)
3. [Writing a Loader](https://webpack.js.org/contribute/writing-a-loader/)
4. [Loader Interface](https://webpack.js.org/api/loaders/)

在对以上的知识有个大致了解后，就会发现这个 loader 内部真正做编译工作的只有 2 行代码

```js
const babel = require("@babel/core");
// ...
const transform = promisify(babel.transform); // 1
// ...
result = await transform(source, options); // 2
```

就是直接调用`babel.transform`而已。

其他大部分的代码是用来处理参数，loader 的参数，babel 的参数没什么可说的，下面着重聊两点，一个是 loader 提供的自定义功能，还有一个是如何实现缓存优化。

### Customized Loader

[文档](https://github.com/babel/babel-loader/tree/v8.2.2#customized-loader)

用户通过提供三个可选函数，可以定制一些参数，或者修改最终编译完成的代码。

第一个是`customOptions`函数，我们直接看源码中如何使用

```js
if (overrides && overrides.customOptions) {
  const result = await overrides.customOptions.call(this, loaderOptions, {
    source,
    map: inputSourceMap,
  });
  customOptions = result.custom;
  loaderOptions = result.loader;
}
```

这个函数接收 loaderOptions 返回一个自定义参数 customOptions，后面两个方法会用到。以及一个新的 loaderOptions。

第二个是`config`函数，

```js
if (overrides && overrides.config) {
  options = await overrides.config.call(this, config, {
    source,
    map: inputSourceMap,
    customOptions,
  });
}
```

接收一个 babel config 对象，返回一个新的对象。

最后一个是`result`函数，

```js
if (overrides && overrides.result) {
  result = await overrides.result.call(this, result, {
    source,
    map: inputSourceMap,
    customOptions,
    config,
    options,
  });
}
```

用来修改编译后的 code。

### 缓存优化

webpack 在 watch 模式下，可能会频繁触发 babel-loader。我们肯定希望只有在文件有变化的时候才利用 babel 重新编译，如果没有变化，就使用上次编译的结果。

babel-loader 提供了[三个参数](https://github.com/babel/babel-loader/tree/v8.2.2#options)来定制内部的缓存优化功能

- cacheDirectory
- cacheIdentifier
- cacheCompression

```js
if (cacheDirectory) {
  result = await cache({
    source,
    options,
    transform,
    cacheDirectory,
    cacheIdentifier,
    cacheCompression,
  });
} else {
  result = await transform(source, options);
}
```

可以看到，如果 cacheDirectory 不为空，就会启用缓存功能。

当第一次编译时，没有缓存，直接通过 babel 编译，然后将结果保存到临时文件夹的文件中。接下来，通过检查缓存文件是否存在，来判断是否需要重新编译。来看代码，

```js
const handleCache = async function (directory, params) {
  const {
    source,
    options = {},
    cacheIdentifier,
    cacheDirectory,
    cacheCompression,
  } = params;

  const file = path.join(directory, filename(source, cacheIdentifier, options));

  try {
    // No errors mean that the file was previously cached
    // we just need to return it
    return await read(file, cacheCompression);
  } catch (err) {}

  const fallback =
    typeof cacheDirectory !== "string" && directory !== os.tmpdir();

  // Make sure the directory exists.
  try {
    await makeDir();
  } catch (err) {
    if (fallback) {
      return handleCache(os.tmpdir(), params);
    }

    throw err;
  }

  // Otherwise just transform the file
  // return it to the user asap and write it in cache
  const result = await transform(source, options);

  try {
    await write(file, cacheCompression, result);
  } catch (err) {
    if (fallback) {
      // Fallback to tmpdir if node_modules folder not writable
      return handleCache(os.tmpdir(), params);
    }

    throw err;
  }

  return result;
};
```

在这个缓存策略中，文件名作为 key，文件名的生成方式如下，

```js
const file = path.join(directory, filename(source, cacheIdentifier, options));
```

其中，`directory`就是 cacheDirectory，`source`是文件内容，`cacheIdentifier`是参数中定义的标识，`options`是传给 babel 的配置参数。

`cacheIdentifier`的默认生成方式如下：

```js
cacheIdentifier = JSON.stringify({
	options,
	"@babel/core": transform.version,
	"@babel/loader": version,
}),
```

最后，filename 这个函数结合这些个内容生成一个 md4 文件名，为什么用 md4？因为要考虑到生成速度。

```js
const filename = function (source, identifier, options) {
  const hash = crypto.createHash("md4");

  const contents = JSON.stringify({ source, options, identifier });

  hash.update(contents);

  return hash.digest("hex") + ".json";
};
```

这时候，如果文件内容有变，那么生成的文件名一定是新的，读取这个文件一定会报错，这时候就认为是没有缓存

```js
try {
  // No errors mean that the file was previously cached
  // we just need to return it
  return await read(file, cacheCompression);
} catch (err) {}
```

这里直接省略了判断文件是否存在的操作，应该也是为了速度，减少文件 io 操作。

如果没有缓存，就意味着要走 babel 编译，然后再把结果保存起来

```js
try {
  await write(file, cacheCompression, result);
} catch (err) {
  if (fallback) {
    // Fallback to tmpdir if node_modules folder not writable
    return handleCache(os.tmpdir(), params);
  }

  throw err;
}
```

注意保存的文件内容是编译后的结果，而生成文件名用的是源码内容。

`cacheCompression` 表示是否需要使用 gzip 压缩文件。因为这里没有删除文件的操作，这个缓存文件夹里的内容会越来越多。不删除，有两个好处

- 即使 webpack 重启，也可以享受到上次的缓存
- 简化缓存策略的实现方式，即不用考虑清除失效缓存
