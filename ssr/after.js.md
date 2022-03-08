# after.js

版本[v3.2.0](https://github.com/jaredpalmer/after.js/tree/v3.2.0)

用来实现服务端渲染(SSR)

## Usage

首先是定义 routes，

```js
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

客户端，

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

if (module.hot) {
  module.hot.accept();
}
```

服务端，

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

## How it works

首先我们要了解服务端渲染的步骤，

1. 根据请求的 url，找到对应的路由
2. 如果存在路由能匹配这个 url，还需要得到这个路由代表的组件
3. 如果这个组件是异步加载的，需要手动触发加载
4. 在组件加载完毕后，如果它含有`getInitialProps`方法，就代表需要服务端先调用这个方法，将结果通过 props 传递给组件
5. 将组件渲染成字符串返回
6. 由于请求的 url 可能没有路由可以处理，需要有个 404 路由来兜底，并且服务端的 status code 为 404
7. 如果请求的 url 对应的路由是一个 Redirect，那么不会渲染组件而是直接返回 301 跳转

正因为服务端需要通过 url 来判断渲染哪些组件，所以需要预先定义一个静态的、中心化的路由。这一点在`next.js`中也是如此，只不是是通过文件路径来间接定义路由。而在客户端代码中，需要先等待异步路由加载完成，以及当前组件的`getInitialProps`方法执行完成得到数据以后才能开始渲染工作。在服务端通过`render`方法来渲染，得到对应的 html 字符串。在两端渲染的过程中，路由数组 routes 都被作为参数传入。

下面我们先来看看 render 方法。内部调用了`renderApp`函数来渲染，注意它的返回结果。如果 redirect 不为空，说明当前 url 需要跳转到另外一个 url。这时候调用了`res.redirect`，因为内部的服务端使用了 express 所以说在 res 对象上是存在这个方法的。但是在这里调用有些不妥，最后又返回了 html。联系我们上面 demo 中的服务端代码，这个 html 又被当做`res.send`的参数，整个过程很混乱。实际上如果 redirect 存在，那么 html 一定是空的，且没有意义。因为 redirect 的意思就是让浏览器再去访问另外一个地址，`res.redirect`在调用后已经将这个 res 流断开了，后面再调用`res.send`没有意义。再者 render 方法中调用 res 中的方法超出了它的职责范围。甚至可以说这个 render 方法都不应该存在，我们可以直接在服务端代码中调用`renderApp`函数。

```js
export const render = async <T extends any>(
  params: Omit<AfterRenderOptions<T>, 'ssg'>
) => {
  const { res } = params;
  const { redirect, statusCode, html } = await renderApp({
    ...params,
    ssg: false,
  });

  if (redirect) {
    res.redirect(statusCode, redirect);
  }

  res.status(statusCode);

  return html;
};
```

接下来看看这个 renderApp 函数，它是如何在服务端渲染组件的。

首先检查传入的路由中有没有处理 404 的路由，其实就是看有没有路由的 path 是这个数组中的值`["**", "*", "", undefined]`。如果没有，就在末尾添加一个。接着从当前的 url 中解析出 pathname 来匹配路由。最后调用 loadInitialProps 来加载组件和组件需要的数据。

```ts
export async function renderApp<T>(
  options: AfterRenderAppOptions<T>
): Promise<RenderResult> {
  // ...
  const routes: AsyncRouteProps[] = utils.getAllRoutes(pureRoutes);
  // ...
  const pathname: string = url.parse(req.url).pathname as string;
  // finds related component for the current path (request url)
  //  and calls component.getInitialProps({ match,...ctx })
  const { match, data: initialData } = await loadInitialProps(
    pathname,
    routes,
    ctx
  );
  // ...
}
// ...
export function getAllRoutes(
  routes: AsyncRouteProps<any>[]
): AsyncRouteProps<any>[] {
  return is404ComponentAvailable(routes)
    ? routes
    : [...routes, { component: NotFoundComponent }];
}
// ...
export function is404ComponentAvailable(
  routes: AsyncRouteProps<any>[]
): AsyncRouteProps<any> | false {
  return (
    routes.find((route) => ["**", "*", "", undefined].includes(route.path)) ||
    false
  );
}
```

loadInitialProps 函数的实现并不复杂，通过`react-router-dom`中的`matchPath`来匹配 routes 中的路由。然后调用其对应组件的 load 方法和 getInitialProps 方法。

```js
import { matchPath, RouteProps } from "react-router-dom";
// ...
export async function loadInitialProps(
  pathname: string,
  routes: AsyncRouteProps[],
  ctx: CtxBase
): Promise<InitialProps> {
  const promises: Promise<any>[] = [];

  const matchedComponent = routes.find((route: RouteProps) => {
    // matchPath dont't accept undefined path property
    // in <Switch> componet all Child <Route> components
    // have a path prop with value of "/", { path: "/" }
    // https://github.com/ReactTraining/react-router/blob/master/packages/react-router/modules/Router.js#L12
    // we get arround this problem by adding { path: "*" }
    // to route that don't have path property
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

但是这里有个问题，在实际的应用中，我们的路由并不是扁平的。也就是存在嵌套路由的情况，所以这里的处理方式其实是不对的。正确的做法是通过[React Router Config](https://github.com/remix-run/react-router/tree/v5.3.0/packages/react-router-config)这个库中的`matchRoutes`方法来匹配到所有的路由，然后逐一处理这些路由中的组件。最后我们调用各个组件的 getInitialProps 方法会得到一组值，可以将这些值合并成一个对象，通过 props 传给各个组件。这也是[umi](https://umijs.org/docs)中的做法。但是在这里只是返回了首个匹配到的路由以及数据。

在得到了路由所需的数据后，取出其中 redirectTo 和 statusCode，当然不一定会存在。但是如果 redirectTo 存在的话，整个流程就可以提前结束了，因为这个值代表需要浏览器自行跳转到这个地址，也就没必要再渲染当前的组件了。

```js
export async function renderApp<T>(
  options: AfterRenderAppOptions<T>
): Promise<RenderResult> {
  // ...
  // <StaticRouter /> context object, which get mutated by React Router
  // and it contains information about statusCode and target <Redirect /> component target url (if any)
  // https://github.com/ReactTraining/react-router/blob/master/packages/react-router/docs/api/StaticRouter.md#context-object
  const context: StaticRouterContext = {};

  // here we will check result of the getInitialProps
  // and see if it contains redirectTo or statusCode properties
  // we will mutate context if we got redirectTo or statusCode in initialData
  if (initialData) {
    const { redirectTo, statusCode } = initialData as RedirectWithStatuCode;

    if (statusCode) {
      context.statusCode = statusCode;
    }

    // if we got redirectTo from getInitalProps
    // we don't waste server resources by rendering the react tree
    // so we return early
    if (redirectTo) {
      return {
        html: '',
        data: initialData,
        redirect: redirectTo,
        statusCode: statusCode || 302,
      };
    }
  }
  // ...
}
```

接下来就是将得到数据通过 props 传给组件，然后渲染成字符串。也就是这个 renderPage 函数做的事情。内部也是调用`react-dom/server`中的 renderToString 方法来渲染。组件的外层用 StaticRouter 包起来，它可以在服务端使用。同时还传入了 context 对象，如果在组件中有 Redirect 组件被渲染了，那么这个对象中记录一个 url，表示这个 Redirect 组件的跳转地址。

```js
import { After } from "./After";
import * as ReactDOMServer from "react-dom/server";
import { matchPath, StaticRouter, RouteProps } from "react-router-dom";
// ...
const context: StaticRouterContext = {};

// ...
const renderPage = async (fn = modPageFn) => {
  // By default, we keep ReactDOMServer synchronous renderToString function
  const defaultRenderer = (element: React.ReactElement<T>) => ({
    html: ReactDOMServer.renderToString(element),
  });
  const renderer = customRenderer || defaultRenderer;
  const asyncOrSyncRender = renderer(
    <StaticRouter location={req.url} context={context}>
      {fn(After)({ routes, data, transitionBehavior: "blocking" })}
    </StaticRouter>
  );

  const renderedContent = await asyncOrSyncRender;
  const helmet = Helmet.renderStatic();

  return { helmet, ...renderedContent };
};
// ...
const modPageFn = function <Props>(Page: React.ComponentType<Props>) {
  return function RenderAfter(props: Props) {
    return <Page {...props} />;
  };
};
```

renderPage 还有一个入参函数，默认是 modPageFn。这个函数看起来是一个高阶组件，但是使用方式却是当成函数在用。它存在的目的是方便自定义渲染组件。例如在配合 redux 使用时可以这样调用：

```js
const page = await renderPage((App) => (props) => (
  <Provider store={store}>
    <App {...props} />
  </Provider>
));
```

这个 renderPage 是在 Document 中被调用的，这个 Document 是我们的页面模板，后面会讲到。如果 context 对象中存在 url，那么当前的渲染流程也要提前结束了。

```js
const Doc = Document || DefaultDoc;
// ...
const { html, ...docProps } = await Doc.getInitialProps({
  req,
  res,
  assets,
  renderPage,
  data,
  helmet: Helmet.renderStatic(),
  match: reactRouterMatch,
  scripts,
  styles,
  scrollToTop: autoScrollRef,
  ...rest,
});

// if we got a <Redirect /> in during render of the react tree
// we redirect the user and we don't waste server resources
if (context.url) {
  return {
    html: "",
    data: initialData,
    redirect: context.url,
    statusCode: context.statusCode || 302,
  };
}
```

最后我们将所有的数据，包括前面渲染完组件得到的 html 片段，通过 props 传入 Document 组件中，再通过`renderToStaticMarkup`方法渲染一次生成最终完整的 html 字符串返回。

```js
const props: DocumentProps<RenderPageResult> = {
  assets,
  data,
  scripts,
  styles,
  match: reactRouterMatch,
  ...rest,
  ...docProps,
  html,
};

// we render <Docuemnt /> which is our app shell
const doc = ReactDOMServer.renderToStaticMarkup(
  <__AfterContext.Provider value={props}>
    <Doc {...props} />
  </__AfterContext.Provider>
);

const page = `<!doctype html>${doc}`;

return {
  html: page,
  data: initialData,
  redirect: "",
  statusCode: context.statusCode || 200,
};
```

这个 Document 组件长这样，代表一个完整的 html。我们的 renderPage 方法会在它的 getInitialProps 方法中被调用，得到的 html 片段会放到 AfterRoot 组件中。然后组件所需要的数据会放入全局的`window.__SERVER_APP_STATE__`中，方便客户端使用。

```js
export class Document extends React.Component<DocumentProps> {
  static getInitialProps = async ({ renderPage }: DocumentgetInitialProps) => {
    const page = await renderPage();
    return { ...page };
  };

  render() {
    const { helmet } = this.props;
    // get attributes from React Helmet
    const htmlAttrs = helmet.htmlAttributes.toComponent();
    const bodyAttrs = helmet.bodyAttributes.toComponent();

    return (
      <html {...htmlAttrs}>
        <head>
          <meta httpEquiv="X-UA-Compatible" content="IE=edge" />
          <meta charSet="utf-8" />
          <meta name="viewport" content="width=device-width, initial-scale=1" />
          {helmet.title.toComponent()}
          {helmet.meta.toComponent()}
          {helmet.link.toComponent()}
          <AfterStyles />
        </head>
        <body {...bodyAttrs}>
          <AfterRoot />
          <AfterData />
          <AfterScripts />
        </body>
      </html>
    );
  }
}

export const AfterRoot: React.FC = () => {
  const { html } = useAfterContext();
  return <div id="root" dangerouslySetInnerHTML={{ __html: html }} />;
};

export const AfterData: React.FC<{ data?: object }> = ({ data }) => {
  const { data: contextData } = useAfterContext();
  return (
    <script
      defer
      dangerouslySetInnerHTML={{
        __html: `window.__SERVER_APP_STATE__ =  ${serialize({
          ...(data || contextData),
        })}`,
      }}
    />
  );
};
```

整个服务端的渲染逻辑就是这样，下面我们看看客户端的代码。

首先是 ensureReady，它会主动加载当前 url 匹配的路由对应的异步组件。然后返回(window as any).\_\_SERVER_APP_STATE\_\_，这个值是上面提到的 Document 组件中留在页面中的全局变量。

```js
/**
 * This helps us to make sure all the async code is loaded before rendering.
 */
export async function ensureReady(
  routes: AsyncRouteProps[],
  pathname?: string
) {
  await Promise.all(
    routes.map(route => {
      const match = matchPath(pathname || window.location.pathname, route);
      if (
        match &&
        route &&
        route.component &&
        isLoadableComponent(route.component) &&
        route.component.load
      ) {
        return route.component.load();
      }
      return undefined;
    })
  );

  return Promise.resolve(
    (window as any).__SERVER_APP_STATE__ as Promise<any>[]
  );
}
```

接着要将这个数据连同 routes 数组传给 After 组件。这个组件有两个职责，

1. 将 routes 数组渲染成对应的路由组件
2. 当路由变化时调用新页面组件的`getInitialProps`方法获取组件需要的数据

组件的初始 state 记录了首次渲染时页面需要的数据，前一个路由和当前路由，以及当前是否在加载。

```js
export const After = withRouter(Afterparty);

class Afterparty extends React.Component<AfterpartyProps, AfterpartyState> {
  state = {
    data: this.props.data.initialData,
    previousLocation: null,
    currentLocation: this.props.location,
    isLoading: false,
  };
  // ...
}
```

当路由变化时，Afterparty 组件的 getDerivedStateFromProps 静态方法会更新自己的 state。因为它被`withRouter`包起来，所以会实时得到最新的 location。但是这里的代码在更新 previousLocation 的时候似乎有问题，previousLocation 的值应该一直是 state.currentLocation。

```js
class Afterparty extends React.Component<AfterpartyProps, AfterpartyState> {
  //...
  static getDerivedStateFromProps(
    props: AfterpartyProps,
    state: AfterpartyState
  ) {
    const currentLocation = props.location;
    const previousLocation = state.currentLocation;

    const navigated = currentLocation !== previousLocation;
    if (navigated) {
      return {
        previousLocation: state.previousLocation || previousLocation,
        currentLocation,
        isLoading: true,
      };
    }

    return null;
  }
  // ...
}
```

按照声明周期的顺序，接下来会调用 render 方法。我们可以看到 render 就是简单的将 routes 渲染成对应的路由组件。这里和上面提到的`loadInitialProps`方法有这同样的问题，就是不能处理嵌套路由。所以还是应该使用[React Router Config](https://github.com/remix-run/react-router/tree/v5.3.0/packages/react-router-config)中的 `renderRoutes`函数来渲染路由，或者使用递归的方式将整个路由都渲染出来。

注意 Switch 组件上传入了 location，这和 transitionBehavior 有关系。transitionBehavior 有两个值 blocking 和 instant。blocking 表示在`getInitialProps`加载完成之前一直渲染旧的页面，而 instant 相反，不需要等待数据加载完成，立即渲染新的页面。

```js
class Afterparty extends React.Component<AfterpartyProps, AfterpartyState> {
  // ...
  render() {
    const { previousLocation, data, isLoading } = this.state;
    const { location: currentLocation, transitionBehavior } = this.props;
    const initialData = this.prefetcherCache[currentLocation.pathname] || data;

    const instantMode = isInstantTransition(transitionBehavior);

    // when we are in the instant mode we want to pass the right location prop
    // to the <Route /> otherwise it will render previous matche component
    const location = instantMode
      ? currentLocation
      : previousLocation || currentLocation;

    return (
      <Switch location={location}>
        {initialData?.statusCode === 404 && (
          <Route component={this.NotfoundComponent} path={location.pathname} />
        )}
        {initialData?.redirectTo && <Redirect to={initialData.redirectTo} />}
        {getAllRoutes(this.props.routes).map((r, i) => (
          <Route
            key={`route--${i}`}
            path={r.path}
            exact={r.exact}
            render={(props) =>
              React.createElement(r.component, {
                ...initialData,
                history: props.history,
                match: props.match,
                prefetch: this.prefetch,
                location,
                isLoading,
              })
            }
          />
        ))}
      </Switch>
    );
  }
}
```

再接着就是 componentDidUpdate 钩子，一旦路由有变动，就会去加载页面的数据。

```js
class Afterparty extends React.Component<AfterpartyProps, AfterpartyState> {
  // ...
  componentDidUpdate(_prevProps: AfterpartyProps, prevState: AfterpartyState) {
    const navigated = prevState.currentLocation !== this.state.currentLocation;
    if (navigated) {
      // ...
      // in ssg mode we don't call component.getInitialProps
      // instead we fetch the page-data.json file
      const loadData = ssg ? loadStaticProps : loadInitialProps;

      loadData(location.pathname, routes, ctx)
        .then((res) => res.data)
        .then((data: InitialData) => {
          // if user moved to a new page at the time we were fetching data
          // for the previous page, we ignore data of the previous page
          if (this.state.currentLocation !== location) return;

          // in blocked mode, first we fetch the data and then we scroll to top
          if (isBloackedMode && isAllowedToScroll) {
            window.scrollTo(0, 0);
          }

          this.setState({ previousLocation: null, data, isLoading: false });
        })
        .catch((e: Error) => {
          // @todo we should more cleverly handle errors???
          console.log(e);
        });
    }
  }
  // ...
}
```

这里还会根据 ssg 变量来判断需要选择哪个加载方法，loadStaticProps 专门用在 ssg 模式，也就是生成静态页面的模式。

## References

1. [Nested Routing](https://github.com/jaredpalmer/after.js/issues/38)
2. [React Router Config](https://github.com/remix-run/react-router/tree/v5.3.0/packages/react-router-config)
3. [Server Rendering, Code Splitting, and Lazy Loading with React Router v4](https://medium.com/airbnb-engineering/server-rendering-code-splitting-and-lazy-loading-with-react-router-v4-bfe596a6af70)
4. [react-router-dom](https://v5.reactrouter.com/web/guides/quick-start)
5. [[Feature]: v6 SSR strategy seems to require client/server duplication](https://github.com/remix-run/react-router/issues/8286)
6. [How to HMR on server side?](https://github.com/webpack/docs/issues/45)
7. [Creating target: "node" module with runtimeChunk: "single" doesn't work](https://github.com/webpack/webpack/issues/8156)
8. [commonjs output clashes with chunk splitting](https://github.com/webpack/webpack/issues/7392)
9. [Question: How does one use **non_webpack_require**?](https://github.com/webpack/webpack/issues/1021)
10. [Server-Side Rendering Caching Headers](https://github.com/vercel/next.js/tree/canary/examples/ssr-caching)
