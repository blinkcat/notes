# loadable-components

版本[v5.15.1](https://github.com/gregberge/loadable-components/tree/v5.15.1)

另一个实现动态加载(dynamic import)的库。不仅可以加载组件，还可以加载 lib。

## Usage

加载组件

```js
import loadable from "@loadable/component";
const OtherComponent = loadable(() => import("./OtherComponent"));
function MyComponent() {
  return (
    <div>
      <OtherComponent />
    </div>
  );
}
```

加载 lib

```js
import loadable from "@loadable/component";
const Moment = loadable.lib(() => import("moment"));
function FromNow({ date }) {
  return (
    <div>
      <Moment fallback={date.toLocaleDateString()}>
        {({ default: moment }) => moment(date).fromNow()}
      </Moment>
    </div>
  );
}
```

甚至是更动态的

```js
import loadable from "@loadable/component";
const AsyncPage = loadable((props) => import(`./${props.page}`));
function MyComponent() {
  return (
    <div>
      <AsyncPage page="Home" />
      <AsyncPage page="Contact" />
    </div>
  );
}
```

当然也是支持服务端渲染的。

## How it works

和前一个库[react-loadable](./react-loadable.md)类似，也是利用了 webpack 对`dynamic import`的支持，将`import()`封装成适合于组件的用法。

从上面的例子来看，主要有两种使用方法，`loadable()`，`loadable.lib()`。

```js
// loadable
export const { loadable, lazy } = createLoadable({
  defaultResolveComponent,
  render({ result: Component, props }) {
    return <Component {...props} />;
  },
});
// loadable.lib
export const { loadable, lazy } = createLoadable({
  onLoad(result, props) {
    if (result && props.forwardedRef) {
      if (typeof props.forwardedRef === "function") {
        props.forwardedRef(result);
      } else {
        props.forwardedRef.current = result;
      }
    }
  },
  render({ result, props }) {
    if (props.children) {
      return props.children(result);
    }

    return null;
  },
});
```

从实现上看，都是通过`createLoadable`这个工厂方法来创建 loadable 函数，差别只在于传参。这表明工厂方法的内部同时对普通组件和非组件(lib)，进行了处理。

```js
// ...
function createLoadable({
  defaultResolveComponent = identity,
  render,
  onLoad,
}) {
  function loadable(loadableConstructor, options = {}) {
    // ...
    const cache = {};
    // ...
    class InnerLoadable extends React.Component {
      // ...
    }
    const EnhancedInnerLoadable = withChunkExtractor(InnerLoadable);
    const Loadable = React.forwardRef((props, ref) => (
      <EnhancedInnerLoadable forwardedRef={ref} {...props} />
    ));

    Loadable.displayName = "Loadable";
    // ...
    return Loadable;
  }
  // ...
  return { loadable, lazy };
}

export default createLoadable;
```

上面是删减过后的代码，`createLoadable`是一个高阶函数，所有的逻辑都封装在了返回的`loadable`函数中。而 createLoadable 只是通过闭包，为 loadable 函数存储了传递进来的三个参数。

再看 loadable 函数，在执行后，最终返回一个`Loadable`组件，这个就是我们最终可以使用的异步加载的组件。

而主要的加载逻辑都在内部的`InnerLoadable`中，特别是它的构造函数

```js
constructor(props) {
  super(props)

  this.state = {
    result: null,
    error: null,
    loading: true,
    cacheKey: getCacheKey(props),
  }

  invariant(
    !props.__chunkExtractor || ctor.requireSync,
    'SSR requires `@loadable/babel-plugin`, please install it',
  )

  // Server-side
  if (props.__chunkExtractor) {
    // This module has been marked with no SSR
    if (options.ssr === false) {
      return
    }

    // We run load function, we assume that it won't fail and that it
    // triggers a synchronous loading of the module
    ctor.requireAsync(props).catch(() => null)

    // So we can require now the module synchronously
    this.loadSync()

    props.__chunkExtractor.addChunk(ctor.chunkName(props))
    return
  }

  // Client-side with `isReady` method present (SSR probably)
  // If module is already loaded, we use a synchronous loading
  // Only perform this synchronous loading if the component has not
  // been marked with no SSR, else we risk hydration mismatches
  if (
    options.ssr !== false &&
    // is ready - was loaded in this session
    ((ctor.isReady && ctor.isReady(props)) ||
      // is ready - was loaded during SSR process
      (ctor.chunkName &&
        LOADABLE_SHARED.initialChunks[ctor.chunkName(props)]))
  ) {
    this.loadSync()
  }
}
```

在`state`的定义中，`cacheKey`看起来比较特别。因为一个异步的组件可以表示多个异步加载过来的 module，可以参考最上面 usage 中的第三个例子。所以在存储已经加载好的 module 时，需要有个 key 来方便存储。

接着是判断当前是否是在服务器端，或者是客户端否需要配合 ssr。

`props.__chunkExtractor`是在服务器端代码中，通过 context 注入的属性，用来收集当前加载的异步组件所属的 chunk name。

而`ctor.requireAsync`，`ctor.chunkName`，`ctor.isReady`等都是通过 babel plugin 生成的方法，用来辅助服务器端渲染。这些在讲到 babel plugin 的时候再细说。

如果是在服务器端，会先异步加载一次，

```js
// We run load function, we assume that it won't fail and that it
// triggers a synchronous loading of the module
ctor.requireAsync(props).catch(() => null);
```

这个函数大概长这样，这是经过 babel 处理过的样子，

```js
  requireAsync: () =>
    import(/* webpackChunkName: "OtherComponent" */
    './OtherComponent'),
```

实际上，在 webpack(target=node)中处理后，变成

```js
__webpack_require__
  .e(/* import() | OtherComponent */ 317)
  .then(__webpack_require__.t.bind(__webpack_require__, 145, 23));

// ...
__webpack_require__.e = (chunkId) => {
  return Promise.all(
    Object.keys(__webpack_require__.f).reduce((promises, key) => {
      __webpack_require__.f[key](chunkId, promises);
      return promises;
    }, [])
  );
};
// ...
__webpack_require__.f.require = function (chunkId, promises) {
  // ...
  installChunk(require("./" + __webpack_require__.u(chunkId)));
  // ...
};
// ...
var installChunk = (chunk) => {
  // ...
  __webpack_require__.m[moduleId] = moreModules[moduleId];
  // ...
};
```

注意即使在 node 端，`__webpack_require__.e` 返回的还是个 promise，这符合`import()`的语法规范。在`__webpack_require__.f.require`中，使用了原生`require`来获取异步的 chunk 脚本，然后调用`installChunk`，将脚本中的 module 存在`__webpack_require__.m`中。

我们接着构造函数往下看，紧接着调用了`this.loadSync()`，

```js
loadSync() {
  // load sync is expecting component to be in the "loading" state already
  // sounds weird, but loading=true is the initial state of InnerLoadable
  if (!this.state.loading) return

  try {
    const loadedModule = ctor.requireSync(this.props)
    const result = resolve(loadedModule, this.props, Loadable)
    this.state.result = result
    this.state.loading = false
  } catch (error) {
    console.error(
      'loadable-components: failed to synchronously load component, which expected to be available',
      {
        fileName: ctor.resolve(this.props),
        chunkName: ctor.chunkName(this.props),
        error: error ? error.message : error,
      },
    )
    this.state.error = error
  }
}
```

注意这里的`ctor.requireSync`，是由 babel plugin 生成的，长这样

```js
  requireSync(props) {
    const id = this.resolve(props)
    if (typeof __webpack_require__ !== 'undefined') {
      return __webpack_require__(id)
    }
    return eval('module.require')(id)
  },
```

`__webpack_require__`会去`__webpack_require__.m`对象中查找需要的 chunk，然后返回。因为我们上面已经通过调用`ctor.requireAsync`将此 chunk 放入到了这个对象中，所以一定可以找到。

注意虽然`ctor.requireAsync`这个方法看起来是个 promise，但是 promise 的初始化是同步的，后续的 then，catch 才是异步的，所以不用担心执行顺序的问题。

接着`ctor.requireSync`往下看，`resolve(loadedModule, this.props, Loadable)`

```js
const identity = (v) => v;

// defaultResolveComponent = identity,

function resolve(module, props, Loadable) {
  const Component = options.resolveComponent
    ? options.resolveComponent(module, props)
    : defaultResolveComponent(module);

  if (options.resolveComponent && !ReactIs.isValidElementType(Component)) {
    throw new Error(
      `resolveComponent returned something that is not a React component!`
    );
  }
  hoistNonReactStatics(Loadable, Component, {
    preload: true,
  });
  return Component;
}
```

首先会对加载进来的 module 做个处理，看是可以直接使用，还是需要找到它的 default 属性。然后判断它是否是合法的 react element，最后调用 hoistNonReactStatics，将组件的静态方法，赋值给 Loadable 组件。

最后更新 state。

再来看 constructor 中关于加载的第二部分，`ctor.isReady`也是用 babel plugin 生成的，

```js
isReady(props) {
  if (typeof __webpack_modules__ !== 'undefined') {
    return !!__webpack_modules__[this.resolve(props)]
  }
  return false
}
// ...
resolve() {
  if (require.resolveWeak) {
    return require.resolveWeak('./OtherComponent')
  }
  return require('path').resolve(__dirname, './OtherComponent')
}
```

其中`__webpack_modules__`就是上面提到的`__webpack_require__.m`，而这个`resolve()`，是获取当前的 module id，也就是 module 在`__webpack_modules__`中的索引。

`ctor.chunkName`是当前这个异步加载的 module 所在的 chunk 名称，`LOADABLE_SHARED.initialChunks`是我们当前已经加载完成的 chunk 集合。

最后，如果发现条件满足，直接同步加载 module，调用`this.loadSync()`。

接下来就到了组件的 render 方法，

```js
render() {
  const {
    forwardedRef,
    fallback: propFallback,
    __chunkExtractor,
    ...props
  } = this.props
  const { error, loading, result } = this.state

  // ...

  if (error) {
    throw error
  }

  const fallback = propFallback || options.fallback || null

  if (loading) {
    return fallback
  }

  return render({
    fallback,
    result,
    options,
    props: { ...props, ref: forwardedRef },
  })
}
```

如果出现错误，直接抛出去，需要用 ErrorBoundary 来处理。如果是加载中，就渲染 fallback。如果成功了，就调用 render 方法，这里的 render 是我们在调用`createLoadable`时传入的参数。

在处理普通组件时，传入的是

```js
render({ result: Component, props }) {
  return <Component {...props} />
}
```

而在处理 lib 的时候，

```js
render({ result, props }) {
  if (props.children) {
    return props.children(result)
  }
  return null
}
```

所以 lib 需要用 render props 的方式使用。

最后到了`componentDidMount`这里，

```js
  componentDidMount() {
    this.mounted = true

    // retrieve loading promise from a global cache
    const cachedPromise = this.getCache()

    // if promise exists, but rejected - clear cache
    if (cachedPromise && cachedPromise.status === STATUS_REJECTED) {
      this.setCache()
    }

    // component might be resolved synchronously in the constructor
    if (this.state.loading) {
      this.loadAsync()
    }
  }
```

这里的`getCache`，`setCache`都是用来做缓存的，

```js
function getCacheKey(props) {
  if (options.cacheKey) {
    return options.cacheKey(props)
  }
  if (ctor.resolve) {
    return ctor.resolve(props)
  }
  return 'static'
}

getCacheKey() {
  return getCacheKey(this.props)
}

getCache() {
  return cache[this.getCacheKey()]
}

setCache(value = undefined) {
  cache[this.getCacheKey()] = value
}
```

我们之前在 Usage 里看到，一个异步组件可能根据传入的 props，加载多个 module。每个加载中的 module 都作为一个 promise 存在 cache 中，每次先去 cache 中查找。

state 中的 loading 默认是 true，如果没有 loadSync，就还是 true，所以需要执行`this.loadAsync()`。

```js
loadAsync() {
  const promise = this.resolveAsync()

  promise
    .then(loadedModule => {
      const result = resolve(loadedModule, this.props, { Loadable })
      this.safeSetState(
        {
          result,
          loading: false,
        },
        () => this.triggerOnLoad(),
      )
    })
    .catch(error => this.safeSetState({ error, loading: false }))

  return promise
}

resolveAsync() {
  const { __chunkExtractor, forwardedRef, ...props } = this.props

  let promise = this.getCache()

  if (!promise) {
    promise = ctor.requireAsync(props)
    promise.status = STATUS_PENDING

    this.setCache(promise)
    // ...
  }

  return promise
}
```

这个`resolveAsync`会先去缓存中查找 promise，如果没有就通过`ctor.requireAsync`创建一个，这个方法我们上面细讲过，返回一个 promise。在客户端，会通过 jsonp 的方法获取 module。然后将这个 promise 存入 cache。

在 promise 成功以后，会更新 state，然后触发`this.triggerOnLoad()`。

```js
triggerOnLoad() {
  if (onLoad) {
    setTimeout(() => {
      onLoad(this.state.result, this.props)
    })
  }
}
```

这个 onLoad 也是调用 createLoadable 时传入的参数。主要是加载 lib 这里，

```js
  onLoad(result, props) {
    if (result && props.forwardedRef) {
      if (typeof props.forwardedRef === 'function') {
        props.forwardedRef(result)
      } else {
        props.forwardedRef.current = result
      }
    }
  }
```

这就涉及到了加载 lib 的另外一种方式，

```js
import loadable from "@loadable/component";
const Moment = loadable.lib(() => import("moment"));
class MyComponent {
  moment = React.createRef();
  handleClick = () => {
    if (this.moment.current) {
      return alert(this.moment.current.default.format("HH:mm"));
    }
  };
  render() {
    return (
      <div>
        <button onClick={this.handleClick}>What time is it?</button>
        <Moment ref={this.moment} />
      </div>
    );
  }
}
```

通过 ref 的方式引用。

组件的主要逻辑就是这样，此外为了配合服务器渲染，还有一个`loadableReady`方法，在客户端渲染时使用，

```js
import { loadableReady } from "@loadable/component";
loadableReady(() => {
  const root = document.getElementById("main");
  hydrate(<App />, root);
});
```

它会确保所有需要的的 chunk 加载完毕后，再调用传递给它的函数进行渲染。

简单的说，第一步，

```js
if (BROWSER) {
  const id = getRequiredChunkKey(namespace);
  const dataElement = document.getElementById(id);
  if (dataElement) {
    requiredChunks = JSON.parse(dataElement.textContent);

    const extElement = document.getElementById(`${id}_ext`);
    if (extElement) {
      const { namedChunks } = JSON.parse(extElement.textContent);
      namedChunks.forEach((chunkName) => {
        LOADABLE_SHARED.initialChunks[chunkName] = true;
      });
    } else {
      // version mismatch
      throw new Error(
        "loadable-component: @loadable/server does not match @loadable/component"
      );
    }
  }
}
```

获取到所有需要的 chunk 名称，存在`requiredChunks`中。

第二步，检查需要的 chunk 是否已经全部加载完毕，

```js
// chunkLoadingGlobal = '__LOADABLE_LOADED_CHUNKS__'
window[chunkLoadingGlobal] = window[chunkLoadingGlobal] || [];
const loadedChunks = window[chunkLoadingGlobal];
const originalPush = loadedChunks.push.bind(loadedChunks);

function checkReadyState() {
  if (
    requiredChunks.every((chunk) =>
      loadedChunks.some(([chunks]) => chunks.indexOf(chunk) > -1)
    )
  ) {
    if (!resolved) {
      resolved = true;
      resolve();
    }
  }
}

loadedChunks.push = (...args) => {
  originalPush(...args);
  checkReadyState();
};

checkReadyState();
```

这里可能有些难理解，只需要记住这里利用了 webpack 自身加载远程 chunk 的方法，每次加载都会调用`checkReadyState`检查条件是否满足。

## SSR

### Usage

```js
import { ChunkExtractor } from "@loadable/server";
// This is the stats file generated by webpack loadable plugin
const statsFile = path.resolve("../dist/loadable-stats.json");
// We create an extractor from the statsFile
const extractor = new ChunkExtractor({ statsFile });
// Wrap your application using "collectChunks"
const jsx = extractor.collectChunks(<YourApp />);
// Render your application
const html = renderToString(jsx);
// You can now collect your script tags
const scriptTags = extractor.getScriptTags(); // or extractor.getScriptElements();
// You can also collect your "preload/prefetch" links
const linkTags = extractor.getLinkTags(); // or extractor.getLinkElements();
// And you can even collect your style tags (if you use "mini-css-extract-plugin")
const styleTags = extractor.getStyleTags(); // or extractor.getStyleElements();
```

主要依赖`ChunkExtractor`这个类，来处理服务端的逻辑。接近 500 行的代码，主要做的两件事情：

第一步，

```js
const ChunkExtractorManager = ({ extractor, children }) => (
  <Context.Provider value={extractor}>{children}</Context.Provider>
)
// ...
collectChunks(app) {
  return <ChunkExtractorManager extractor={this}>{app}</ChunkExtractorManager>
}
// ...
addChunk(chunk) {
  if (this.chunks.indexOf(chunk) !== -1) return
  this.chunks.push(chunk)
}
```

结合我们上面的讲解，这里就是通过同一个 context，将`extractor`属性注入代码中，收集已经加载的组件中需要的 chunk name。

第二步，

配合 webpack 插件生成的 asset，chunk 信息，生成对应的 script，link，style 标签。

## webpack plugin

```js
new LoadablePlugin({ filename: "stats.json", writeToDisk: true });
```

这个插件用来收集所有的 chunk 信息，在 ssr 时可以用来判断需要的 chunk 是否有加载完毕。

有几个地方可以学习的，

```js
// chunkLoadingGlobal = '__LOADABLE_LOADED_CHUNKS__',
compiler.options.output.chunkLoadingGlobal = this.opts.chunkLoadingGlobal;
```

[chunkLoadingGlobal](https://webpack.js.org/configuration/output/#outputchunkloadingglobal)这个配置很少用到，和上面说到的`loadableReady`配合起来，可以在 webpack 加载脚本时做一些检测。

既然要收集 webpack 最终生成的 chunk 信息，需要监听哪些 hook 呢？

```js
// webpack 5
compiler.hooks.make.tap(name, (compilation) => {
  compilation.hooks.processAssets.tap(
    {
      name,
      stage: compiler.webpack.Compilation.PROCESS_ASSETS_STAGE_REPORT,
    },
    () => {
      const asset = this.handleEmit(compilation);
      if (asset) {
        compilation.emitAsset(this.opts.filename, asset);
      }
    }
  );
});
```

compiler 中的 [make](https://webpack.js.org/api/compiler-hooks/#make)，以及 compilation 中的[processAssets](https://webpack.js.org/api/compilation-hooks/#processassets)。

```js
handleEmit = (compilation) => {
  const stats = compilation.getStats().toJson({
    all: false,
    assets: true,
    cachedAssets: true,
    chunks: false,
    chunkGroups: true,
    chunkGroupChildren: true,
    hash: true,
    ids: true,
    outputPath: true,
    publicPath: true,
  });

  stats.generator = "loadable-components";

  // we don't need all chunk information, only a type
  stats.chunks = [...compilation.chunks].map((chunk) => {
    return {
      id: chunk.id,
      files: [...chunk.files],
    };
  });
  // ...
  const result = JSON.stringify(stats, null, 2);
  // ...
  if (this.opts.outputAsset) {
    return {
      source() {
        return result;
      },
      size() {
        return result.length;
      },
    };
  }

  return null;
};
```

在 compilation 对象中可以获取到 stats 对象，进而获得我们想要的信息。如果`this.opts.outputAsset`为 true，返回了一个 webpack source 类型的对象。这是为了配合上面的`compilation.emitAsset`，这样在 webpack 最后的输出信息中，会显示我们的 json 文件。

## Babel plugin

也是用来支持服务端渲染的，会将一行`import()`函数，

```js
import loadable from "@loadable/component";
const OtherComponent = loadable(() => import("./OtherComponent"));
```

变成一个 object 对象

```js
import loadable from "@loadable/component";
const OtherComponent = loadable({
  chunkName() {
    return "OtherComponent";
  },
  isReady(props) {
    if (typeof __webpack_modules__ !== "undefined") {
      return !!__webpack_modules__[this.resolve(props)];
    }
    return false;
  },
  requireAsync: () =>
    import(/* webpackChunkName: "OtherComponent" */ "./OtherComponent"),
  requireSync(props) {
    const id = this.resolve(props);
    if (typeof __webpack_require__ !== "undefined") {
      return __webpack_require__(id);
    }
    return eval("module.require")(id);
  },
  resolve() {
    if (require.resolveWeak) {
      return require.resolveWeak("./OtherComponent");
    }
    return require("path").resolve(__dirname, "./OtherComponent");
  },
});
```

通过增加 webpack magic comment，为包含组件的 chunk 取一个名字。这里其他的函数的用途我们上面都有介绍，这里不再赘述。

出乎意料的是，在生成的过程中，`chunkName`是最麻烦的，因为它还要处理多个异步 module 的情况。这个我们上面有提到过。而其他几个函数则大同小异，有固定的模板。例如`isReady`，我们可以通过[template.ast](https://babeljs.io/docs/en/babel-template#ast-1)方法来直接生成，

```js
function isReadyProperty({ types: t, template }) {
  const statements = template.ast(`
    const key=this.resolve(props)
    if (this.resolved[key] !== true) {
      return false
    }

    if (typeof __webpack_modules__ !== 'undefined') {
      return !!(__webpack_modules__[key])
    }

    return false
  `);

  return () =>
    t.objectMethod(
      "method",
      t.identifier("isReady"),
      [t.identifier("props")],
      t.blockStatement(statements)
    );
}
```

简单直接。
