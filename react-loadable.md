# react-loadable

版本[v5.5.0](https://github.com/jamiebuilds/react-loadable/tree/v5.5.0)

> A higher order component for loading components with dynamic imports.
>
> 一个用来动态加载组件的高阶组件

## Usage

```jsx
import Loadable from "react-loadable";
import Loading from "./my-loading-component";

const LoadableComponent = Loadable({
  loader: () => import("./my-component"),
  loading: Loading,
});

export default class App extends React.Component {
  render() {
    return <LoadableComponent />;
  }
}
```

## How it works

webpack 自身支持 `import()`，每个动态载入的脚步会被编译到一个单独的文件中。当需要时，会使用 jsonp 的方式导入。

由于`import()`方式返回的是一个 promise，不太方便组件使用，所以需要包装一下，这也是这个库的工作之一。另外，在服务端渲染的时候，也需要特殊处理`import()`进来的脚步，在渲染页面的时候，需要在 html 中预先加入这些脚本。这些工作都可以通过这个库来完成。

源码分成三个部分，组件代码，babel 插件代码，webpack 插件代码。

先看组件部分，

```js
function Loadable(opts) {
  return createLoadableComponent(load, opts);
}

function LoadableMap(opts) {
  if (typeof opts.render !== "function") {
    throw new Error("LoadableMap requires a `render(loaded, props)` function");
  }

  return createLoadableComponent(loadMap, opts);
}

Loadable.Map = LoadableMap;
```

`Loadable`在上面的例子中见过，`LoadableMap`用在动态载入多个脚本的情况下，

```js
Loadable.Map({
  loader: {
    Bar: () => import("./Bar"),
    i18n: () => fetch("./i18n/bar.json").then((res) => res.json()),
  },
  render(loaded, props) {
    let Bar = loaded.Bar.default;
    let i18n = loaded.i18n;
    return <Bar {...props} i18n={i18n} />;
  },
});
```

两者内部都是调用`createLoadableComponent`，唯一的区别是第一个参数。我们先看两者都传递的`opts`参数，这对我们理解后面的代码有帮助。

- **opts.loader** 一个函数，返回一个 promise，里面是我们动态加载的组件
- **opts.loading** 一个组件，在加载时，或者出错时显示
- **opts.delay** 在等待多少毫秒后，再显示`loading`
- **opts.timeout** 超时时间
- **opts.render** 用来自定义渲染加载后的组件的函数
- **opts.webpack** 一个可选函数，返回一个表示加载的`module.id`的数组
- **opts.modules** 可选数组，里面是加载的`module`的路径

下面来看下分别传入的第一个参数，

```js
function load(loader) {
  let promise = loader();

  let state = {
    loading: true,
    loaded: null,
    error: null,
  };

  state.promise = promise
    .then((loaded) => {
      state.loading = false;
      state.loaded = loaded;
      return loaded;
    })
    .catch((err) => {
      state.loading = false;
      state.error = err;
      throw err;
    });

  return state;
}
```

很简单，这个`loader`就是我们前面说的`opts.loader`。load 函数最终返回一个`state`对象，表示本次加载状态和加载结果。

```js
function loadMap(obj) {
  let state = {
    loading: false,
    loaded: {},
    error: null,
  };

  let promises = [];

  try {
    Object.keys(obj).forEach((key) => {
      let result = load(obj[key]);

      if (!result.loading) {
        state.loaded[key] = result.loaded;
        state.error = result.error;
      } else {
        state.loading = true;
      }

      promises.push(result.promise);

      result.promise
        .then((res) => {
          state.loaded[key] = res;
        })
        .catch((err) => {
          state.error = err;
        });
    });
  } catch (err) {
    state.error = err;
  }

  state.promise = Promise.all(promises)
    .then((res) => {
      state.loading = false;
      return res;
    })
    .catch((err) => {
      state.loading = false;
      throw err;
    });

  return state;
}
```

loadMap 稍微复杂点，它的参数是一个 object 对象，它也有一个内部的`state`对象，之后也会返回。不同的是，它会遍历 object 中的所有 `opts.loader`函数，传入上面的 load 函数中，然后将结果采用相同的 key 值存入`state.loaded`中。

好了，有了这层理解，我们再来看`createLoadableComponent`函数，它是一个高阶组件，可以分为两部分来看，

```js
function createLoadableComponent(loadFn, options) {
  if (!options.loading) {
    throw new Error("react-loadable requires a `loading` component");
  }

  let opts = Object.assign(
    {
      loader: null,
      loading: null,
      delay: 200,
      timeout: null,
      render: render,
      webpack: null,
      modules: null,
    },
    options
  );

  let res = null;

  function init() {
    if (!res) {
      res = loadFn(opts.loader);
    }
    return res.promise;
  }

  ALL_INITIALIZERS.push(init);

  if (typeof opts.webpack === "function") {
    READY_INITIALIZERS.push(() => {
      if (isWebpackReady(opts.webpack)) {
        return init();
      }
    });
  }

  return class LoadableComponent extends React.Component {
    // ...
  };
}
```

我们先看返回组件之前的准备工作，重心都在这个`init`函数上，这个函数唯一的工作就是调用传入的 load 函数加载组件，将返回的 state 对象赋值给 res，然后返回 promise。注意这个 res 变量的位置，在返回的组件之外，相当于保存在闭包中。然后就是将这个 init 函数存储在两个数组中，`ALL_INITIALIZERS`和`READY_INITIALIZERS`。后一个是有条件的，只有当`opts.webpack`存在时，而且里面还得满足一个条件，init 函数才会被执行。

```js
function isWebpackReady(getModuleIds) {
  if (typeof __webpack_modules__ !== "object") {
    return false;
  }

  return getModuleIds().every((moduleId) => {
    return (
      typeof moduleId !== "undefined" &&
      typeof __webpack_modules__[moduleId] !== "undefined"
    );
  });
}
```

这快的判断很重要，会直接影响你对后续功能的理解。

首先，在 webpack 编译后的代码中`__webpack_modules__`表示一个全局的对象，里面存着所有的 module。key 值可以是这个 module 的路径，也可以是[别的值](https://webpack.js.org/configuration/optimization/#optimizationmoduleids)。

`getModuleIds`是我们上面提到的`opts.webpack`，里面是通过`require.resolveWeak`获取到的 module 的路径，可以和上面的`__webpack_modules__`中的 key 值对应起来。如果一个通过`import()`引入的 chunk 加载进来以后，会在注册后，放入`__webpack_modules__`对象中。所以可以通过上面的条件来判断模块有没有被加载。在做服务器渲染时，当前用到的异步模块都应该是已经被加载完成的。

接着看高阶组件返回的组件，

```js
class LoadableComponent extends React.Component {
  constructor(props) {
    super(props);
    init();

    this.state = {
      error: res.error,
      pastDelay: false,
      timedOut: false,
      loading: res.loading,
      loaded: res.loaded,
    };
  }

  static contextTypes = {
    loadable: PropTypes.shape({
      report: PropTypes.func.isRequired,
    }),
  };

  static preload() {
    return init();
  }

  componentWillMount() {
    this._mounted = true;
    this._loadModule();
  }

  _loadModule() {
    if (this.context.loadable && Array.isArray(opts.modules)) {
      opts.modules.forEach((moduleName) => {
        this.context.loadable.report(moduleName);
      });
    }

    if (!res.loading) {
      return;
    }

    if (typeof opts.delay === "number") {
      if (opts.delay === 0) {
        this.setState({ pastDelay: true });
      } else {
        this._delay = setTimeout(() => {
          this.setState({ pastDelay: true });
        }, opts.delay);
      }
    }

    if (typeof opts.timeout === "number") {
      this._timeout = setTimeout(() => {
        this.setState({ timedOut: true });
      }, opts.timeout);
    }

    let update = () => {
      if (!this._mounted) {
        return;
      }

      this.setState({
        error: res.error,
        loaded: res.loaded,
        loading: res.loading,
      });

      this._clearTimeouts();
    };

    res.promise
      .then(() => {
        update();
      })
      .catch((err) => {
        update();
      });
  }

  componentWillUnmount() {
    this._mounted = false;
    this._clearTimeouts();
  }

  _clearTimeouts() {
    clearTimeout(this._delay);
    clearTimeout(this._timeout);
  }

  retry = () => {
    this.setState({ error: null, loading: true, timedOut: false });
    res = loadFn(opts.loader);
    this._loadModule();
  };

  render() {
    if (this.state.loading || this.state.error) {
      return React.createElement(opts.loading, {
        isLoading: this.state.loading,
        pastDelay: this.state.pastDelay,
        timedOut: this.state.timedOut,
        error: this.state.error,
        retry: this.retry,
      });
    } else if (this.state.loaded) {
      return opts.render(this.state.loaded, this.props);
    } else {
      return null;
    }
  }
}
```

当这个组件被创建的时候，才会去调用 init 函数，加载远程的模块。在渲染之前，也就是`componentWillMount`这个钩子里，会调用`this._loadModule()`，将 res 对象中的结果同步到组件的 state 中。注意这个钩子在服务器端也会执行，所以`_loadModule`函数会先执行`this.context.loadable.report(moduleName)`，这个`moduleName`就是之前提到的`opts.modules`，而这个`this.context.loadable.report`就是`Capture`组件通过 context 传入的函数，用来收集当前已经加载的异步组件所包含的 module 文件路径。

```js
class Capture extends React.Component {
  static propTypes = {
    report: PropTypes.func.isRequired,
  };

  static childContextTypes = {
    loadable: PropTypes.shape({
      report: PropTypes.func.isRequired,
    }).isRequired,
  };

  getChildContext() {
    return {
      loadable: {
        report: this.props.report,
      },
    };
  }

  render() {
    return React.Children.only(this.props.children);
  }
}

Loadable.Capture = Capture;
```

现在我们再看 render 函数，一目了然，根据条件进行渲染，没什么好说的。

除此之外，还有两个方法，分别用在服务器端和客户端，用来确保异步组件加载完成。

```js
function flushInitializers(initializers) {
  let promises = [];

  while (initializers.length) {
    let init = initializers.pop();
    promises.push(init());
  }

  return Promise.all(promises).then(() => {
    if (initializers.length) {
      return flushInitializers(initializers);
    }
  });
}

Loadable.preloadAll = () => {
  return new Promise((resolve, reject) => {
    flushInitializers(ALL_INITIALIZERS).then(resolve, reject);
  });
};

Loadable.preloadReady = () => {
  return new Promise((resolve, reject) => {
    // We always will resolve, errors should be handled within loading UIs.
    flushInitializers(READY_INITIALIZERS).then(resolve, resolve);
  });
};
```

两种分别是确保上面提到的两个数组中的 init 函数执行完毕。这也是 init 函数中为什么要先判断 res 是否已经被赋值的原因，因为它可能被执行两次。

接下来是 babel 插件部分，因为`opts.webpack`和`opts.modules`两个属性在客户端代码中是可选的，在服务器代码是中必须有的，如果全手写，会令人困扰，所以需要 babel 插件来动态写入这些属性。

```js
// 输入
import Loadable from "react-loadable";

const LoadableMyComponent = Loadable({
  loader: () => import("./MyComponent"),
});

const LoadableComponents = Loadable.Map({
  loader: {
    One: () => import("./One"),
    Two: () => import("./Two"),
  },
});

// 输出
import Loadable from "react-loadable";
import path from "path";

const LoadableMyComponent = Loadable({
  loader: () => import("./MyComponent"),
  webpack: () => [require.resolveWeak("./MyComponent")],
  modules: [path.join(__dirname, "./MyComponent")],
});

const LoadableComponents = Loadable.Map({
  loader: {
    One: () => import("./One"),
    Two: () => import("./Two"),
  },
  webpack: () => [require.resolveWeak("./One"), require.resolveWeak("./Two")],
  modules: [path.join(__dirname, "./One"), path.join(__dirname, "./Two")],
});
```

但是实际上，输出的是

```js
import Loadable from "react-loadable";
const LoadableMyComponent = Loadable({
  loader: () => import("./MyComponent"),
  modules: ["./MyComponent"],
  webpack: () => [require.resolveWeak("./MyComponent")],
});
const LoadableComponents = Loadable.Map({
  loader: {
    One: () => import("./One"),
    Two: () => import("./Two"),
  },
  modules: ["./One", "./Two"],
  webpack: () => [require.resolveWeak("./One"), require.resolveWeak("./Two")],
});
```

modules 中的并非是绝对路径，但这并不妨碍后续寻找对应的 module。

最后再来看看 webpack 插件，

```js
// webpack.config.js
import { ReactLoadablePlugin } from "react-loadable/webpack";

export default {
  plugins: [
    new ReactLoadablePlugin({
      filename: "./dist/react-loadable.json",
    }),
  ],
};
```

只需要传入一个 filename，就会生成一个文件，包含我们需要的 chunk，module 信息。

在具体实现上监听了 compiler 的`emit`事件，拿到 chunk 信息后建立索引，

```js
manifest[currentModule.rawRequest].push({ id, name, file, publicPath });
```

注意这里取得是`rawRequest`属性，就是业务代码中直接写在`import()`中的路径，对应上面的`modules`数组。

在服务端渲染时，找到对应 chunk 文件后，就可以预先插入到渲染的 html 中。
