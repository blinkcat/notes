# After.js

版本[3.1.3](https://github.com/jaredpalmer/after.js/tree/v3.1.3)

用来做 react 服务端渲染(ssr)。

## Usage

先声明路由

```js
// ./routes.js
import React from "react";
import { asyncComponent } from "@jaredpalmer/after";

export default [
  {
    path: "/",
    exact: true,
    component: asyncComponent({
      loader: () => import("./Home"), // required
      Placeholder: () => <div>...LOADING...</div>, // this is optional, just returns null by default
    }),
  },
  {
    path: "/about",
    exact: true,
    component: asyncComponent({
      loader: () => import("./About"), // required
      Placeholder: () => <div>...LOADING...</div>, // this is optional, just returns null by default
    }),
  },
];
```

客户端代码

```js
import React from "react";
import { hydrate } from "react-dom";
import { BrowserRouter } from "react-router-dom";
import { ensureReady, After } from "@jaredpalmer/after";
import "./client.css";
import routes from "./routes";

ensureReady(routes).then((data) =>
  hydrate(
    <BrowserRouter>
      <After data={data} routes={routes} />
    </BrowserRouter>,
    document.getElementById("root")
  )
);
```

服务端代码

```js
import express from "express";
import { render } from "@jaredpalmer/after";
import routes from "./routes";

const assets = require(process.env.RAZZLE_ASSETS_MANIFEST);
const chunks = require(process.env.RAZZLE_CHUNKS_MANIFEST);

const server = express();
server
  .disable("x-powered-by")
  .use(express.static(process.env.RAZZLE_PUBLIC_DIR))
  .get("/*", async (req, res) => {
    try {
      const html = await render({
        req,
        res,
        routes,
        assets,
        chunks,
      });
      res.send(html);
    } catch (error) {
      console.error(error);
      res.json({ message: error.message, stack: error.stack });
    }
  });

export default server;
```

和`next.js`不同的是，路由不和 pages 文件夹结构挂钩。而是直接采用了一种集中声明的方式。

## How it works

服务端渲染(ssr)有几个要点，

1. 服务端和客户端运行大致相同的代码，保证服务端渲染的 html 和客户端自身渲染出的 html 一致
2. 服务端需要有能力在页面渲染前，拉取到其所需要的数据，这就要求提前知道当前页面的组件

首先，实现这两点需要将同一套代码编译两份，一份给客户端用，一份给服务端(node)用。

其次要有一个方法来获取页面需要的数据，既能在客户端使用，也能在服务器端使用。`after.js`和`next.js`一样，需要页面组件提供一个名叫`getInitialProps`的静态方法，在组件 render 前拉取数据，通过 props 传入组件。但这里还有一个前提，需要事先知道将要渲染的页面组件。这就要求我们要有一个集中式的路由配置。`next.js`自创了一个路由系统，将 pages 文件夹结构转换为对应路由，由此可以预先知道路由配置。而`after.js`的路由直接使用了`react-router`，但是`react-router`从第 4 个版本开始，推荐使用去中心化的方式使用路由，只有在运行时，才能知道当前页面的组件。所以，`after.js`需要一个 route 配置文件，预先提供路由配置。关于这点，`react-router`自己也提供了一个库`react-router-config`来解决这个问题。

### 文件结构

```sh
./packages/after.js/src
├── After.tsx
├── Document.tsx
├── NotFoundComponent.tsx
├── asyncComponent.tsx
├── ensureReady.ts
├── getAssets.ts
├── index.tsx
├── loadInitialProps.tsx
├── render.tsx
├── serializeData.tsx
├── types.ts
└── utils.ts
```

- `After.tsx` 提供一个组件，内部会处理路由，调用页面组件的`getInitialProps`方法
- `Document.tsx` 相当于一个 html 模板
- `asyncComponent` 提供异步路由组件功能
- `ensureReady.ts`用在客户端，确保当前的异步路由组件加载完毕后再开始渲染
- `loadInitialProps.tsx`找到当前的路由组件，调用其`getInitialProps`方法
- `render.tsx`为服务端提供一个 render 方法，用来生成 html 字符串

### Source Code

接下来我们来看代码，

#### _render.tsx_

```js
export interface AfterRenderOptions<T> {
  req: Request;
  res: Response;
  assets: Assets;
  routes: AsyncRouteProps[];
  document?: typeof DefaultDoc;
  chunks: Chunks;
  scrollToTop?: boolean;
  customRenderer?: (
    element: React.ReactElement<T>
  ) => { html: string } | Promise<{ html: string }>;
}

export async function render<T>(options: AfterRenderOptions<T>) {
  const {
    req,
    res,
    routes: pureRoutes,
    assets,
    document: Document,
    customRenderer,
    chunks,
    scrollToTop = true,
    ...rest
  } = options;
  // ...
}
```

在服务端代码中处理请求时，直接调用此方法，可生成当前 url 下的 html。先看传参，

- 头两个是 express 的 Request，Response 对象
- assets 和 chunks 是 webpack build 后的产物，提供一些打包信息
- routes 就是我们的路由配置
- document 是我们的 html 模板，以 react 组件的形式表示
- scrollToTop 用来控制路由跳转后，滚动条的位置
- customRenderer 一个函数，传入 react 元素，生成其 html 字符串

接下来，先处理传入的路由。

```js
const routes = utils.getAllRoutes(pureRoutes);
```

如果传入的路由中没有处理`404`的路由，就自动加上一个默认的。

```js
export function is404ComponentAvailable(
  routes: AsyncRouteProps<any>[]
): AsyncRouteProps<any> | false {
  return (
    routes.find((route) => ["**", "*", "", undefined].includes(route.path)) ||
    false
  );
}

export function getAllRoutes(
  routes: AsyncRouteProps<any>[]
): AsyncRouteProps<any>[] {
  return is404ComponentAvailable(routes)
    ? routes
    : [...routes, { component: NotFoundComponent }];
}
```

从代码里可以看出，404 的 route path 为`**`、`*`、`''`或是没有 path。

在处理完路由后，会尝试取当前路由的需要的数据。

```js
const autoScrollRef = { current: scrollToTop };
const { match, data: initialData } = await loadInitialProps(
	routes,
	url.parse(req.url).pathname as string,
	{
		req,
		res,
		scrollToTop: autoScrollRef,
		...rest,
	}
);
```

我们先看看调用`loadInitialProps`的入参，

- routes 是处理过的带有 404 的路由配置
- 第二个参数我们传入了当前请求 url 的 pathname
- 第三个对象里有 express 的 request，response 对象，控制路由跳转后页面是否需要回到顶部的配置，rest 的对象是外层调用`render`时，额外传入的属性

```js
import { matchPath, RouteProps } from "react-router-dom";
import { AsyncRouteProps, InitialProps, CtxBase } from "./types";
import { isAsyncComponent } from "./utils";

export async function loadInitialProps(
  routes: AsyncRouteProps[],
  pathname: string,
  ctx: CtxBase
): Promise<InitialProps> {
  const promises: Promise<any>[] = [];

  const matchedComponent = routes.find((route: RouteProps) => {
    const match = matchPath(pathname, { ...route, path: route.path || "*" });

    if (match && route.component && isAsyncComponent(route.component)) {
      const component = route.component;

      promises.push(
        component.load
          ? component
              .load()
              .then(() => component.getInitialProps({ match, ...ctx }))
          : component.getInitialProps({ match, ...ctx })
      );
    }

    return !!match;
  });

  return {
    match: matchedComponent,
    data: (await Promise.all(promises))[0],
  };
}
```

通过`matchPath`方法，我们可以匹配到当前 url 对应的路由。注意这里的`promises`数组，如果我们的路由配置是扁平的，即没有子路由的情况下，只会有一个路由被匹配到。在有子路由的情况下，可能有多个路由被匹配到。所以这里用了个数组，存储着所有调用过组件上`getInitialProps`方法返回的 promise，这个方法传入了第三个参数，外加一个 match 对象。

最后的返回值很奇怪，`(await Promise.all(promises))[0]`，只返回第一个数据。这是因为`after.js`目前还不支持嵌套路由。

在`render.tsx`中，会对这个返回的 data 做一些判断处理，

```js
if (initialData) {
const { redirectTo, statusCode } = initialData as {
	statusCode?: number;
	redirectTo?: string;
};

if (statusCode) {
	context.statusCode = statusCode;
}

if (redirectTo) {
	res.redirect(statusCode || 302, redirectTo);
	return;
}
}
```

这个`initialData`就是我们调用`loadInitialProps`后返回的数据

## References

1. [Nested Routing](https://github.com/jaredpalmer/after.js/issues/38)
2. [React Router Config](https://github.com/remix-run/react-router/tree/main/packages/react-router-config)
3. [Server Rendering, Code Splitting, and Lazy Loading with React Router v4](https://medium.com/airbnb-engineering/server-rendering-code-splitting-and-lazy-loading-with-react-router-v4-bfe596a6af70)
