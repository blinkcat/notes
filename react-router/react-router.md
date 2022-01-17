# react-router

版本 [v6.2.1](https://github.com/remix-run/react-router/tree/v6.2.1)

react 项目中路由库的首选，历久弥新，值得学习。

## demo

```tsx
import { render } from "react-dom";
import { BrowserRouter, Routes, Route } from "react-router-dom";
// import your route components too

render(
  <BrowserRouter>
    <Routes>
      <Route path="/" element={<App />}>
        <Route index element={<Home />} />
        <Route path="teams" element={<Teams />}>
          <Route path=":teamId" element={<Team />} />
          <Route path="new" element={<NewTeamForm />} />
          <Route index element={<LeagueStandings />} />
        </Route>
      </Route>
    </Routes>
  </BrowserRouter>,
  document.getElementById("root")
);
```

第六个版本变动巨大，首先是架构上回归了中心化的路由，偏向于预先定义好路由结构，且这个结构和实际组件的结构一致。细节上优化了子路由的定义，即可以使用相对路由来定义子路由，无需指定完整路径。统一了路由和组件的关联方式，只需通过 element 属性添加。

## How it works

从项目结构上看，react-router 分为三个独立的 package

- react-router 大部分逻辑、组件都在这里、且和平台无关
- react-router-dom 为 web 服务
- react-router-native 为 react-native 服务

下面两个 package 都依赖于 react-router 这个 package 提供的功能。所以我们先从这个 package 看起。

首先我们先看下 react-router 在使用时的内部的组件结构，

```html
<!-- 或者是 BrowserRouter -->
<MemoryRouter>
  <Router>
    <NavigationContext.Provider>
      <LocationContext.Provider>
        <Routes>
          <RouteContext.Provider>
            <YourComponent>
              <!-- 不一定有 Outlet -->
              <Outlet />
            </YourComponent>
          </RouteContext.Provider>
        </Routes>
      </LocationContext.Provider>
    </NavigationContext.Provider>
  </Router>
</MemoryRouter>
```

如果和上面的 demo 做个比较，会发现实际运行时，少了一些组件，比如说 Route，多了很多的 Context。这个结构也反应了 react-router 的内部逻。我们先从 Context 看起。

### Context

Context 本身的能力或者说职责就是提供跨越组件传递数据的功能，能够为下层组件注入数据，省去通过 props 层层传递的麻烦。所以我们重点关注的是这个 Context 提供了什么数据。

首先是 NavigationContext，它存储了 basename 属性，navigator 对象，这个对象就是[history](./history.md)对象的窄接口，语义上只提供跳转功能。还有一个 static 属性，表示当前是否是在服务端。

```ts
export type Navigator = Pick<History, "go" | "push" | "replace" | "createHref">;

interface NavigationContextObject {
  basename: string;
  navigator: Navigator;
  static: boolean;
}

const NavigationContext = React.createContext<NavigationContextObject>(null!);
```

LocationContext 存储的是 location 对象和 navigationType 属性，这个属性就是 history 中 Action 的别名。

```ts
import { Action as NavigationType } from "history";

interface LocationContextObject {
  location: Location;
  navigationType: NavigationType;
}

const LocationContext = React.createContext<LocationContextObject>(null!);
```

接着是 RouteContext，outlet 是子路由代表的组件，matches 是路由匹配数组，后面会提到。

```ts
interface RouteContextObject {
  outlet: React.ReactElement | null;
  matches: RouteMatch[];
}

const RouteContext = React.createContext<RouteContextObject>({
  outlet: null,
  matches: [],
});
```

有了这三个 Context 的基础，我们可以接着看主要的组件了。

### Component

react-router 中实现了 MemoryRouter，主要用于测试路由。而实际常用的 BrowserRouter、HashRouter 都在 react-router-dom 中实现。不过它们实现的方式几乎一样，所以只需要看 MemoryRouter 的逻辑。

首先通过[history](./history.md)中的 createMemoryHistory 函数创建一个 history 实例。接着通过 history.listen 方法监听路由变化，每次变化时，更新 action 和 location。最后将这些个属性传递给 Router 组件。

```tsx
export function MemoryRouter({
  basename,
  children,
  initialEntries,
  initialIndex,
}: MemoryRouterProps): React.ReactElement {
  let historyRef = React.useRef<MemoryHistory>();
  if (historyRef.current == null) {
    historyRef.current = createMemoryHistory({ initialEntries, initialIndex });
  }

  let history = historyRef.current;
  let [state, setState] = React.useState({
    action: history.action,
    location: history.location,
  });

  React.useLayoutEffect(() => history.listen(setState), [history]);

  return (
    <Router
      basename={basename}
      children={children}
      location={state.location}
      navigationType={state.action}
      navigator={history}
    />
  );
}
```

Router 组件的代码看起来很多，但还是很容易看懂的。

```tsx
export function Router({
  basename: basenameProp = "/",
  children = null,
  location: locationProp,
  navigationType = NavigationType.Pop,
  navigator,
  static: staticProp = false,
}: RouterProps): React.ReactElement | null {
  invariant(
    !useInRouterContext(),
    `You cannot render a <Router> inside another <Router>.` +
      ` You should never have more than one in your app.`
  );

  let basename = normalizePathname(basenameProp);
  let navigationContext = React.useMemo(
    () => ({ basename, navigator, static: staticProp }),
    [basename, navigator, staticProp]
  );

  if (typeof locationProp === "string") {
    locationProp = parsePath(locationProp);
  }

  let {
    pathname = "/",
    search = "",
    hash = "",
    state = null,
    key = "default",
  } = locationProp;

  let location = React.useMemo(() => {
    let trailingPathname = stripBasename(pathname, basename);

    if (trailingPathname == null) {
      return null;
    }

    return {
      pathname: trailingPathname,
      search,
      hash,
      state,
      key,
    };
  }, [basename, pathname, search, hash, state, key]);

  warning(
    location != null,
    `<Router basename="${basename}"> is not able to match the URL ` +
      `"${pathname}${search}${hash}" because it does not start with the ` +
      `basename, so the <Router> won't render anything.`
  );

  if (location == null) {
    return null;
  }

  return (
    <NavigationContext.Provider value={navigationContext}>
      <LocationContext.Provider
        children={children}
        value={{ location, navigationType }}
      />
    </NavigationContext.Provider>
  );
}
```

第一步先判断自己有没有被嵌套使用，也就是这种情况

```tsx
<Router>
  ...
  <Router>...</Router>
  ...
</Router>
```

判断方法很简单，只需判断当前是否能获取到 LocationContext 的值。因为它自己渲染的时候会将 children 包在这个 Context 中。

```tsx
export function useInRouterContext(): boolean {
  return React.useContext(LocationContext) != null;
}
```

接着会处理 basename，它的值默认是`/`，调用 normalizePathname 函数对它进行添头去尾。为什么要特别处理 basename 呢，因为接下来它在路由路径处理中是无法忽略的。

```ts
const normalizePathname = (pathname: string): string =>
  pathname.replace(/\/+$/, "").replace(/^\/*/, "/");
```

再接下来就是生成两个 Context 的 value 了。特别要注意 location 对象中的 pathname，它利用了上面我们提到的 basename 对原始的 pathname 进行掐头操作，因为在 basename 之后的路径才是我们需要匹配的。

```ts
function stripBasename(pathname: string, basename: string): string | null {
  if (basename === "/") return pathname;

  if (!pathname.toLowerCase().startsWith(basename.toLowerCase())) {
    return null;
  }

  let nextChar = pathname.charAt(basename.length);
  if (nextChar && nextChar !== "/") {
    // pathname does not start with basename/
    return null;
  }

  return pathname.slice(basename.length) || "/";
}
```

在 Router 组件里，必须要使用 Routes。这个组件里包含了所有的路由匹配逻辑和渲染对应组件的逻辑。

```tsx
export interface RoutesProps {
  children?: React.ReactNode;
  location?: Partial<Location> | string;
}

export function Routes({
  children,
  location,
}: RoutesProps): React.ReactElement | null {
  return useRoutes(createRoutesFromChildren(children), location);
}
```

首先看 createRoutesFromChildren 这个方法，它会递归处理 Routes 组件的子元素，生成一个 RouteObject 对象数组。

```ts
export interface RouteObject {
  caseSensitive?: boolean;
  children?: RouteObject[];
  element?: React.ReactNode;
  index?: boolean;
  path?: string;
}
```

这个数组连同可选的 location 对象一起传给 useRoutes。

useRoutes 这个 hook 是 react-router 的核心，它会根据当前的 location，将 RouteObject 数组转化成 ReactNode 返回。

首先它会判断自己是不是在 Router 组件下面，同样也是利用 Context 来判断。

```tsx
export function useRoutes(
  routes: RouteObject[],
  locationArg?: Partial<Location> | string
): React.ReactElement | null {
  invariant(
    useInRouterContext(),
    // TODO: This error is probably because they somehow have 2 versions of the
    // router loaded. We can help them understand how to avoid that.
    `useRoutes() may be used only in the context of a <Router> component.`
  );
  // ...
}
```

接着会通过 RouteContext 获取到之前的 RouteMatch 数组，也就是之前匹配的内容，当然不一定存在这个数组，这里是为了处理存在于组件中的子路由，后面我们会提到。

```tsx
export function useRoutes(
  routes: RouteObject[],
  locationArg?: Partial<Location> | string
): React.ReactElement | null {
  // ...
  let { matches: parentMatches } = React.useContext(RouteContext);
  let routeMatch = parentMatches[parentMatches.length - 1];
  let parentParams = routeMatch ? routeMatch.params : {};
  let parentPathname = routeMatch ? routeMatch.pathname : "/";
  let parentPathnameBase = routeMatch ? routeMatch.pathnameBase : "/";
  let parentRoute = routeMatch && routeMatch.route;
  // ...
}
```

react-router 对在组件中定义的子路由有个约束，其父路由必须以`/*`结尾，否则匹配不到。且这种子路由也必须用 Routes 组件包起来，这也是我们上面提到的代码的来由。

```tsx
<Routes>
  {/* This route path MUST end with /* because otherwise
          it will never match /blog/post/123 */}
  {/*这里需要写成 blog/* */}
  <Route path="blog" element={<Blog />} />
  <Route path="blog/feed" element={<BlogFeed />} />
</Routes>;

function Blog() {
  return (
    // 这里需要Routes
    <Routes>
      <Route path="post/:id" element={<Post />} />
    </Routes>
  );
}
```

接下来这部分是真的要来根据 location 匹配路由了。首先是确定 location，从入参传进来的优先，其次是从 Context 中获取的。如果是传进来的 location，还会多一个要求，它的 pathname 必须以父路的 pathnameBase 打头。

```tsx
export function useRoutes(
  routes: RouteObject[],
  locationArg?: Partial<Location> | string
): React.ReactElement | null {
  // ...
  let locationFromContext = useLocation();

  let location;
  if (locationArg) {
    let parsedLocationArg =
      typeof locationArg === "string" ? parsePath(locationArg) : locationArg;

    invariant(
      parentPathnameBase === "/" ||
        parsedLocationArg.pathname?.startsWith(parentPathnameBase),
      `When overriding the location using \`<Routes location>\` or \`useRoutes(routes, location)\`, ` +
        `the location pathname must begin with the portion of the URL pathname that was ` +
        `matched by all parent routes. The current pathname base is "${parentPathnameBase}" ` +
        `but pathname "${parsedLocationArg.pathname}" was given in the \`location\` prop.`
    );

    location = parsedLocationArg;
  } else {
    location = locationFromContext;
  }

  let pathname = location.pathname || "/";
  let remainingPathname =
    parentPathnameBase === "/"
      ? pathname
      : pathname.slice(parentPathnameBase.length) || "/";
  let matches = matchRoutes(routes, { pathname: remainingPathname });
  // ...
}
```

接着我们会求得 remainingPathname，之所以不直接用 pathname 还是考虑到了嵌套路由的情况，需要去除父路由的 pathnameBase。接着连同 routes 传递给 matchRoutes 函数。

在 matchRoutes 函数中，我们还是会先处理 pathname，因为这个函数是 export 出去的，不仅是内部使用。所以也需要排除 basename 的影响。

```ts
export function matchRoutes(
  routes: RouteObject[],
  locationArg: Partial<Location> | string,
  basename = "/"
): RouteMatch[] | null {
  let location =
    typeof locationArg === "string" ? parsePath(locationArg) : locationArg;

  let pathname = stripBasename(location.pathname || "/", basename);

  if (pathname == null) {
    return null;
  }

  let branches = flattenRoutes(routes);
  rankRouteBranches(branches);

  let matches = null;
  for (let i = 0; matches == null && i < branches.length; ++i) {
    matches = matchRouteBranch(branches[i], pathname);
  }

  return matches;
}
```

接着将 routes 原封不动传入 flattenRoutes 函数，这个函数从名字上看，似乎是将嵌套的 RouteObject 数组打平，其实完全不是这样。准确地说，它是用来根据当前的路由定义，遍历出当前所有可能的路由路线。

```sh
    /
  a   b/*
:id    :id

1. /
2. /, a
3. /, a, :id
4. /, b/*
5. /, b/*, :id
```

假设我们定义的路由用上面那棵 path 树来表示，那么所有能匹配到的路由线路有 5 种。flattenRoutes 函数就是找到这些路线，然后以特定结构返回。

我们来看下代码，虽然有点长，但是并不是很复杂。直接对 routes 数组做循环操作，将 RouteObject 对象封装成 RouteMeta 对象。注意这个 relativePath 属性，它直接取自 route.path，如果 route 没有 path 属性就为空字符串，例如 [Index Route](https://reactrouter.com/docs/en/v6/getting-started/tutorial#index-routes) 就不需要 path。如果这个 path 以`/`开头，说明是个绝对路径，必须以它的父路由的 path 开头，然后我们再去掉这个父路由的前缀，毕竟需要的是相对路径。

```ts
interface RouteMeta {
  relativePath: string;
  caseSensitive: boolean;
  childrenIndex: number;
  route: RouteObject;
}

function flattenRoutes(
  routes: RouteObject[],
  branches: RouteBranch[] = [],
  parentsMeta: RouteMeta[] = [],
  parentPath = ""
): RouteBranch[] {
  routes.forEach((route, index) => {
    let meta: RouteMeta = {
      relativePath: route.path || "",
      caseSensitive: route.caseSensitive === true,
      childrenIndex: index,
      route,
    };

    if (meta.relativePath.startsWith("/")) {
      invariant(
        meta.relativePath.startsWith(parentPath),
        `Absolute route path "${meta.relativePath}" nested under path ` +
          `"${parentPath}" is not valid. An absolute child route path ` +
          `must start with the combined path of all its parent routes.`
      );

      meta.relativePath = meta.relativePath.slice(parentPath.length);
    }

    // ...
  });

  // ...
}
```

除此之外，RouteMeta 对象中还保留了 childrenIndex，也就是它在上级路由中的序号，这个值会留着以后排序的时候使用。

接着生成一个 path 路径，就是将 parentPath 和 relativePath 做合并。然后再生成一个 routesMeta 数组。这两个值都会作为参数传入对子路由的 flattenRoutes 操作中。但是注意最下方的 branches.push 操作，父路由的 path 和 routesMeta 数组和子路由的是分别被加入到 branches 数组中的。在递归遍历的过程中，只有 branches 这个参数的引用是保持不变的。所以最终得到了所有可能的路径。

```tsx
function flattenRoutes(
  routes: RouteObject[],
  branches: RouteBranch[] = [],
  parentsMeta: RouteMeta[] = [],
  parentPath = ""
): RouteBranch[] {
  routes.forEach((route, index) => {
    // ...
    let path = joinPaths([parentPath, meta.relativePath]);
    let routesMeta = parentsMeta.concat(meta);

    // Add the children before adding this route to the array so we traverse the
    // route tree depth-first and child routes appear before their parents in
    // the "flattened" version.
    if (route.children && route.children.length > 0) {
      invariant(
        route.index !== true,
        `Index routes must not have child routes. Please remove ` +
          `all child routes from route path "${path}".`
      );

      flattenRoutes(route.children, branches, routesMeta, path);
    }

    // Routes without a path shouldn't ever match by themselves unless they are
    // index routes, so don't add them to the list of possible branches.
    if (route.path == null && !route.index) {
      return;
    }

    branches.push({ path, score: computeScore(path, route.index), routesMeta });
  });

  return branches;
}
```

在遍历路径的过程中，有一个 computeScore 操作，来计算这个路由路径的分数。这个分数留着后面的排序用。实际上这是为了配合 v6 版本的新功能。比如说当前需要匹配的路径是`/teams/new`，但是在路由配置中，`teams/:teamId`比`teams/new`靠前，但是依然是后面的被匹配到，因为后面的路由更贴合当前的路径。

```tsx
<Route path="teams/:teamId" element={<Team />} />
<Route path="teams/new" element={<NewTeamForm />} />
```

这个功能的实现就是依靠这里的打分操作，更详细的路由得分更高。首先 path 会被分段，根据每一段的字符串类型打分，累加。分段的数量是基础分，动态的字段得 3 分，例如:id 这样的。而静态的字段可以得到 10 分，所以说上面那种情况下，后面的路由分数就会很高。

```tsx
const paramRe = /^:\w+$/;
const dynamicSegmentValue = 3;
const indexRouteValue = 2;
const emptySegmentValue = 1;
const staticSegmentValue = 10;
const splatPenalty = -2;
const isSplat = (s: string) => s === "*";

function computeScore(path: string, index: boolean | undefined): number {
  let segments = path.split("/");
  let initialScore = segments.length;
  if (segments.some(isSplat)) {
    initialScore += splatPenalty;
  }

  if (index) {
    initialScore += indexRouteValue;
  }

  return segments
    .filter((s) => !isSplat(s))
    .reduce(
      (score, segment) =>
        score +
        (paramRe.test(segment)
          ? dynamicSegmentValue
          : segment === ""
          ? emptySegmentValue
          : staticSegmentValue),
      initialScore
    );
}
```

在求得 RouteBranch 对象数组后，就会利用它的分数进行排序，分数高的排在前面。

```tsx
interface RouteBranch {
  path: string;
  score: number;
  routesMeta: RouteMeta[];
}

function rankRouteBranches(branches: RouteBranch[]): void {
  branches.sort((a, b) =>
    a.score !== b.score
      ? b.score - a.score // Higher score first
      : compareIndexes(
          a.routesMeta.map((meta) => meta.childrenIndex),
          b.routesMeta.map((meta) => meta.childrenIndex)
        )
  );
}
```

但是如果分数相同的话，这个时候会根据它们在父路由中的位置来排序，靠前的就放前面。

```ts
function compareIndexes(a: number[], b: number[]): number {
  let siblings =
    a.length === b.length && a.slice(0, -1).every((n, i) => n === b[i]);

  return siblings
    ? // If two routes are siblings, we should try to match the earlier sibling
      // first. This allows people to have fine-grained control over the matching
      // behavior by simply putting routes with identical paths in the order they
      // want them tried.
      a[a.length - 1] - b[b.length - 1]
    : // Otherwise, it doesn't really make sense to rank non-siblings by index,
      // so they sort equally.
      0;
}
```

在排好序后，就要开启一个循环来真正的做匹配了，直到有一条路径被选中。

```ts
for (let i = 0; matches == null && i < branches.length; ++i) {
  matches = matchRouteBranch(branches[i], pathname);
}
```

所有的匹配逻辑都在这个 matchRouteBranch 函数里。在这里进行分段的匹配，首先从入参的 RouteBranch 对象中取出 RouteMeta 数组，对这个数组做循环操作，每次循环过程中，根据上次的 matchedPathname，计算出这次需要匹配的 remainingPathname。调用 matchPath 函数做一个匹配操作，如果得到了一个 PathMatch 对象，就记录下来。如果得到的是 null，说明当前的 RouteBranch 对象不能够匹配当前的路径，直接返回 null。这样根据上面的循环操作，继续进行 matchRouteBranch 操作。

```ts
function matchRouteBranch<ParamKey extends string = string>(
  branch: RouteBranch,
  pathname: string
): RouteMatch<ParamKey>[] | null {
  let { routesMeta } = branch;

  let matchedParams = {};
  let matchedPathname = "/";
  let matches: RouteMatch[] = [];
  for (let i = 0; i < routesMeta.length; ++i) {
    let meta = routesMeta[i];
    let end = i === routesMeta.length - 1;
    let remainingPathname =
      matchedPathname === "/"
        ? pathname
        : pathname.slice(matchedPathname.length) || "/";
    let match = matchPath(
      { path: meta.relativePath, caseSensitive: meta.caseSensitive, end },
      remainingPathname
    );

    if (!match) return null;

    Object.assign(matchedParams, match.params);

    let route = meta.route;

    matches.push({
      params: matchedParams,
      pathname: joinPaths([matchedPathname, match.pathname]),
      pathnameBase: joinPaths([matchedPathname, match.pathnameBase]),
      route,
    });

    if (match.pathnameBase !== "/") {
      matchedPathname = joinPaths([matchedPathname, match.pathnameBase]);
    }
  }

  return matches;
}
```

这里的 matchPath 函数返回一个 PathMatch 对象，表示匹配成功的结果。

```ts
/**
 * A PathMatch contains info about how a PathPattern matched on a URL pathname.
 */
export interface PathMatch<ParamKey extends string = string> {
  /**
   * The names and values of dynamic parameters in the URL.
   */
  params: Params<ParamKey>;
  /**
   * The portion of the URL pathname that was matched.
   */
  pathname: string;
  /**
   * The portion of the URL pathname that was matched before child routes.
   */
  pathnameBase: string;
  /**
   * The pattern that was used to match.
   */
  pattern: PathPattern;
}
```

这个对象会被转化为一个 RouteMatch 对象，存在 matches 数组中，留给后面的渲染函数使用。

```ts
/**
 * A RouteMatch contains info about how a route matched a URL.
 */
export interface RouteMatch<ParamKey extends string = string> {
  /**
   * The names and values of dynamic parameters in the URL.
   */
  params: Params<ParamKey>;
  /**
   * The portion of the URL pathname that was matched.
   */
  pathname: string;
  /**
   * The portion of the URL pathname that was matched before child routes.
   */
  pathnameBase: string;
  /**
   * The route object that was used to match.
   */
  route: RouteObject;
}
```

matchPath 还是有必要讲一下的，但在此之前，先要提一下 compilePath 函数。这个函数在 matchPath 中被调用，用来将一段 path 变成一段正则表达式。例如：

| options                                       | reg                             | paramNames |
| --------------------------------------------- | ------------------------------- | ---------- |
| {path:'/', caseSensitive:false, end:false}    | /^\\/(?:\\b\|\\/\|$)/i          | []         |
| {path:'/', caseSensitive:true, end:true}      | /^\\/\\/\*$/                    | []         |
| {path:'a', caseSensitive:false, end:false}    | /^\\/a(?:\\b\|\\/\|$)/i         | []         |
| {path:':id', caseSensitive:false, end:false}  | /^\\/([^\\/]+)(?:\\b\|\\/\|$)/i | ['id']     |
| {path:'b/\*', caseSensitive:false, end:false} | /^\\/b(?:\\/(.+)\|\\/\*)$/i     | ['*']      |

而且路由 path 中的动态部分会被收集在 paramNames 数组中。

在 matchPath 函数中，通过这个正则可以获取到对应的 matchedPathname。当然如果返回 null 则说明不匹配，直接返回。然后就是求解这个 params 数组了，它表示的是路由路径中动态部分的值，比如说`*`，`id`。还有一个属性 pathnameBase，一般情况下只是将 matchedPathname 去掉末尾的`/`。但是有一种情况，路由的 path 是`b/*`，被匹配的 pathname 为`/b/2`，这个时候得到的 matchedPathname 就是`/b/2`，但是 pathnameBase 却为`/b`。这是因为，pathname 的后半段`/2`是留给子路由匹配的，所以下次匹配的时候 matchRouteBranch 函数要通过去掉父路由的 pathnameBase 来获取 remainingPathname，再和下一个 RouteMeta 对象匹配。

```ts
export function matchPath<
  ParamKey extends ParamParseKey<Path>,
  Path extends string
>(
  pattern: PathPattern<Path> | Path,
  pathname: string
): PathMatch<ParamKey> | null {
  if (typeof pattern === "string") {
    pattern = { path: pattern, caseSensitive: false, end: true };
  }

  let [matcher, paramNames] = compilePath(
    pattern.path,
    pattern.caseSensitive,
    pattern.end
  );

  let match = pathname.match(matcher);
  if (!match) return null;

  let matchedPathname = match[0];
  let pathnameBase = matchedPathname.replace(/(.)\/+$/, "$1");
  let captureGroups = match.slice(1);
  let params: Params = paramNames.reduce<Mutable<Params>>(
    (memo, paramName, index) => {
      // We need to compute the pathnameBase here using the raw splat value
      // instead of using params["*"] later because it will be decoded then
      if (paramName === "*") {
        let splatValue = captureGroups[index] || "";
        pathnameBase = matchedPathname
          .slice(0, matchedPathname.length - splatValue.length)
          .replace(/(.)\/+$/, "$1");
      }

      memo[paramName] = safelyDecodeURIComponent(
        captureGroups[index] || "",
        paramName
      );
      return memo;
    },
    {}
  );

  return {
    params,
    pathname: matchedPathname,
    pathnameBase,
    pattern,
  };
}
```

如果再回到上面的 matchRouteBranch 函数，可以发现 matchPath 返回值中的 pathname 和 pathnameBase 还需要和 matchedPathname 做一次合并。

```ts
matches.push({
  params: matchedParams,
  pathname: joinPaths([matchedPathname, match.pathname]),
  pathnameBase: joinPaths([matchedPathname, match.pathnameBase]),
  route,
});
```

最终在这些工作完成之后，我们得到了一个 RouteMatch 数组 matches。做完一些校验，将 matches 交给\_renderMatches 函数生成对应的 react 组件。

```ts
export function useRoutes(
  routes: RouteObject[],
  locationArg?: Partial<Location> | string
): React.ReactElement | null {
  // ...
  if (__DEV__) {
    warning(
      parentRoute || matches != null,
      `No routes matched location "${location.pathname}${location.search}${location.hash}" `
    );

    warning(
      matches == null ||
        matches[matches.length - 1].route.element !== undefined,
      `Matched leaf route at location "${location.pathname}${location.search}${location.hash}" does not have an element. ` +
        `This means it will render an <Outlet /> with a null value by default resulting in an "empty" page.`
    );
  }

  return _renderMatches(
    matches &&
      matches.map((match) =>
        Object.assign({}, match, {
          params: Object.assign({}, parentParams, match.params),
          pathname: joinPaths([parentPathnameBase, match.pathname]),
          pathnameBase:
            match.pathnameBase === "/"
              ? parentPathnameBase
              : joinPaths([parentPathnameBase, match.pathnameBase]),
        })
      ),
    parentMatches
  );
}
```

在深入\_renderMatches 函数之前，我们先看看 RouteMatch 对象的结构。

```tsx
interface RouteMatch<ParamKey extends string = string> {
  params: Params<ParamKey>;
  pathname: string;
  pathnameBase: string;

  route: {
    caseSensitive?: boolean;
    children?: RouteObject[];
    element?: React.ReactNode;
    index?: boolean;
    path?: string;
  };
}
```

再来看下\_renderMatches 函数的代码。reduceRight 就是从数组右侧向左侧累进。每个组件都用 RouteContext 包起来了，children 默认是 match.route.element，如果不存在就用 Outlet 替代。value 部分会保存上次的组件和 matches 数组，这会方便我们处理子路由和实现一些 hooks。

```tsx
function _renderMatches(
  matches: RouteMatch[] | null,
  parentMatches: RouteMatch[] = []
): React.ReactElement | null {
  if (matches == null) return null;

  return matches.reduceRight((outlet, match, index) => {
    return (
      <RouteContext.Provider
        children={
          match.route.element !== undefined ? match.route.element : <Outlet />
        }
        value={{
          outlet,
          matches: parentMatches.concat(matches.slice(0, index + 1)),
        }}
      />
    );
  }, null as React.ReactElement | null);
}
```

为何需要从数组的右边处理呢？因为数组的顺序是从顶层路由到底层路由，我们最终一定是先渲染顶层路由代表的组件，然后根据这个组件中的 Outlet 组件的位置，渲染子路由代表的组件。

下面就说说 Outlet 组件，它代表子路由组件的入口。参考上面的\_renderMatches 函数中，它从 RouteContext 中获取到下一层的子组件，包在 OutletContext 中渲染出来。

```tsx
export function Outlet(props: OutletProps): React.ReactElement | null {
  return useOutlet(props.context);
}

export function useOutlet(context?: unknown): React.ReactElement | null {
  let outlet = React.useContext(RouteContext).outlet;
  if (outlet) {
    return (
      <OutletContext.Provider value={context}>{outlet}</OutletContext.Provider>
    );
  }
  return outlet;
}
```

如果每一个父路由中都有这个 Outlet 组件，那么就可以渲染出子路由表示的组件。

还有一个组件 Navigate，可以跳转到对应的路由，在它的内部直接调用了 useNavigate 这个 hook。

```tsx
export function Navigate({ to, replace, state }: NavigateProps): null {
  // ...
  let navigate = useNavigate();
  React.useEffect(() => {
    navigate(to, { replace, state });
  });

  return null;
}
```

所以我们下面直接进入 hooks 部分。

### Hook

第一个 hook，useHref 用来计算出目标的完整 url。在它的内部又调用了 useResolvedPath。

```tsx
export interface Path {
  pathname: Pathname;
  search: Search;
  hash: Hash;
}

export declare type To = string | Partial<Path>;

export function useHref(to: To): string {
  // ...
  let { basename, navigator } = React.useContext(NavigationContext);
  let { hash, pathname, search } = useResolvedPath(to);

  let joinedPathname = pathname;
  if (basename !== "/") {
    let toPathname = getToPathname(to);
    let endsWithSlash = toPathname != null && toPathname.endsWith("/");
    joinedPathname =
      pathname === "/"
        ? basename + (endsWithSlash ? "/" : "")
        : joinPaths([basename, pathname]);
  }

  return navigator.createHref({ pathname: joinedPathname, search, hash });
}
```

不得已，我们还是要先看 useResolvedPath。首先从 RouteContext 中取出 RouteMatch 数组，然后取出 pathnameBase 属性进行序列化，这是为了配合下面的 useMemo hook，方便确定是否需要重新计算。

```tsx
export function useResolvedPath(to: To): Path {
  let { matches } = React.useContext(RouteContext);
  let { pathname: locationPathname } = useLocation();

  let routePathnamesJson = JSON.stringify(
    matches.map((match) => match.pathnameBase)
  );

  return React.useMemo(
    () => resolveTo(to, JSON.parse(routePathnamesJson), locationPathname),
    [to, routePathnamesJson, locationPathname]
  );
}
```

接着在 useMemo hook 中又调用了 resolveTo 函数。这个函数有些复杂，因为我们提供的目标地址可能是绝对地址。

- /a/b/c

也可能是相对地址，甚至是只有 search 或 hash 部分的地址。

- ./a/b
- ../a/b
- ?a=b

因此对相对地址来说，需要先要确定基准地址，也就是说当前是相对于谁来定位。

如果 toPathname 是空的，那么就认为它是相对于当前的 url。否则就要根据 routePathnames 数组来求得基准地址 from。这种情况下，指针指向数组尾部，如果 toPathname 头部有一个`..`，指针就往前移动一次。最后得到新的 to.pathname 和 from 值。再将这两个值交给 resolvePath 函数处理。

```ts
function resolveTo(
  toArg: To,
  routePathnames: string[],
  locationPathname: string
): Path {
  let to = typeof toArg === "string" ? parsePath(toArg) : toArg;
  let toPathname = toArg === "" || to.pathname === "" ? "/" : to.pathname;
  // ...
  let from: string;
  if (toPathname == null) {
    from = locationPathname;
  } else {
    let routePathnameIndex = routePathnames.length - 1;

    if (toPathname.startsWith("..")) {
      let toSegments = toPathname.split("/");

      // Each leading .. segment means "go up one route" instead of "go up one
      // URL segment".  This is a key difference from how <a href> works and a
      // major reason we call this a "to" value instead of a "href".
      while (toSegments[0] === "..") {
        toSegments.shift();
        routePathnameIndex -= 1;
      }

      to.pathname = toSegments.join("/");
    }

    // If there are more ".." segments than parent routes, resolve relative to
    // the root / URL.
    from = routePathnameIndex >= 0 ? routePathnames[routePathnameIndex] : "/";
  }

  let path = resolvePath(to, from);
  // ...
}
```

有了目标地址和基准地址以后，resolvePath 函数就可以求得最终的地址了。

再回到 useHref hook，如果 basename 不为`/`，我们还要拼上 basename。最终通过 createHref 函数将 Path 对象变成一个 url。

接着我们来看 useNavigate，通过它我们可以获取一个函数，进行路由跳转。其实和上面的 useRef 如出一辙，首先通过 NavigationContext 获取 navigator 对象，然后根据入参 to 的类型以及 options 中的属性，决定调用 navigator 中的`go`、`replace`或是`push`方法来实现跳转。

```ts
export function useNavigate(): NavigateFunction {
  // ...
  let { basename, navigator } = React.useContext(NavigationContext);
  let { matches } = React.useContext(RouteContext);
  let { pathname: locationPathname } = useLocation();

  let routePathnamesJson = JSON.stringify(
    matches.map((match) => match.pathnameBase)
  );

  let activeRef = React.useRef(false);
  React.useEffect(() => {
    activeRef.current = true;
  });

  let navigate: NavigateFunction = React.useCallback(
    (to: To | number, options: NavigateOptions = {}) => {
      warning(
        activeRef.current,
        `You should call navigate() in a React.useEffect(), not when ` +
          `your component is first rendered.`
      );

      if (!activeRef.current) return;

      if (typeof to === "number") {
        navigator.go(to);
        return;
      }

      let path = resolveTo(
        to,
        JSON.parse(routePathnamesJson),
        locationPathname
      );

      if (basename !== "/") {
        path.pathname = joinPaths([basename, path.pathname]);
      }

      (!!options.replace ? navigator.replace : navigator.push)(
        path,
        options.state
      );
    },
    [basename, navigator, routePathnamesJson, locationPathname]
  );

  return navigate;
}
```

另外还有一个比较常用的 useParams，它的实现更加简单，只是通过 RouteContext 拿到当前的 RouteMatch 对象，取出里面的 params 对象。

```ts
export function useParams<
  ParamsOrKey extends string | Record<string, string | undefined> = string
>(): Readonly<
  [ParamsOrKey] extends [string] ? Params<ParamsOrKey> : Partial<ParamsOrKey>
> {
  let { matches } = React.useContext(RouteContext);
  let routeMatch = matches[matches.length - 1];
  return routeMatch ? (routeMatch.params as any) : {};
}
```

至此，react-router 中的大部分逻辑我们都探索过了。

---

## 2022-01-17 补充

关于 useMatch 的类型声明，如果这么用，`useMatch("/a/b/:id/:name")`可以发现返回值类型为`PathMatch<"id" | "name"> | null`，非常聪明。

关键就是这个[ParamParseSegment](https://www.typescriptlang.org/play?ts=4.4.2#code/C4TwDgpgBACghgJzgW3ggzhAYnAlgGwgBMoBeKAbyggQQHsEAuKYBAV2gF8BuAWACgBoSLEQo0mADwBlCAHNkEAHbBqAD2DKi6KOla4lcgHxkBUKAHoLUAMIALCAGMA1lAcJouAGYt30CGq4ejpwUF4MAO6IJOj4cOh2UAa+0HoIBnIAdGZQsgrKqgGaStpQAAYAJBQGXjRQADIQXsB5iiqcFlU1dQBKuHJ2LfJtwJxlOeZWUACSPsB+SSFhkdG6cQkANClKUHDAmshgqsB0UGCImNRwjonouETQdHMOE5bW4QhRCDHrdtn85nMAH5REhUBcIJJGs1WgUTEUtDpuggGk1gD0IOg2PhgK9gaDxBDJH0BkN8ip4RpEUklLUUSTBhisTi8YCQdD0ZjsYUqSUdGkMqzAW8Zs9oIRmrp7p5+WxHI5MegvNj8CAzhCSPFduqwVt5sooDcnK5vELAVN9VB0qSpQ9DXAdgAjVJyhXoJUqtXnDDEXY6CIQfD4TKiqCOujzW2Ys2TayOB1h6DezBEPUOHaWjzM1RBbVsJS4Og7J4pFgRU53B7oGMigAUHjgRGYACJwnRm1AAD5QZuOxDNgCU-2FI6gIIZnOz6mKpQFhjHqOaTO5XagE+XOKgzA5G9xANH5uss1LEtUlZlUCUEd0rsVyqDXo1fu13pQaYNRpcNam3igRdVpbWoMUb2k6Lryneno6pcWqhK+yAhseuB7geh6ge+GYOFaXKbrmABWbB6Fa-SDFsgaXMhADkSxeHg+BsB4w6oVua4kZOK4Inyuj6PO45sbuLFoISPo4AQxBmswQngiJdHifuwpTMelqniBRD3EolGqMmngqKccFiMgGFQARRGfq4mZsayUznkxwqSQZEiQhOsIUtO1LIqxpK7mafFeThPIzvyPFyAu67+YJDkQqJhBEBJBLSZg0VyQpR5ih41GXqcHxfD88R2EZZlJM8uaYOSZ7AIgwD+shiRwK8UyOHQ+BFohPjIVARB0JiRnzLmuahEQIBKCguCOLowwFFs6CnJm4VBPV1gmccWEeMgeAlHUJaWnOWRQAA8vqnxBBAWxUTRdEMRAtksS5AXUmUjBdLSvQQGtBgZGMoWvetGQRWCjlJUQfCCPwwjQFJ6AyBNKhuVxO0mKQORSY5UNlZSgXcekhgLaG23Q6o8ZgTeEHuveAHaSQBgnNqmCqCWO1QPgyE0HA+DoBsOP6ph0BZiuuZyHQv3U862prWAf7PHQlw7SEOjOBAIAhCUONgFLdyOgB+YPF4Bi+jLz6WgAbqzHDoNdIJoMAuCs8STgMEQkjI0St1GFNwVGEYONKVhpUjKBV6qCLWIkx6D7QcQRkBpRHjGYRqgC0LdA43YcCG9AoQC3QJDa3AK60UGfYuBLNPBVAAC0Jg7SCYvXfZCBWzbGKNd8kg7W7WPGEYwMCAEqv12E+aOFbRZQIREAALJ7DcMiw7OwVbDAs86BDMge7W8HoMw0gDpJlA5B4wAMTsFCcM+MDA5wAgCGPk-ADctatnQdAWH2CAv3AABeg7AzfU92A-bYLCMFfu-L+A4f6YFvvfR+z9gGICAX2MBECJ5-wfowQBcC36IO-tfSBqDmzoKfgg+BcCkECCAA)的定义，相当的巧妙，但是也很复杂。

## References

- [Parse params in TypeScript #8030](https://github.com/remix-run/react-router/pull/8030)
- [Template Literal Types](https://www.typescriptlang.org/docs/handbook/2/template-literal-types.html)
- [Conditional Types](https://www.typescriptlang.org/docs/handbook/2/conditional-types.html)
