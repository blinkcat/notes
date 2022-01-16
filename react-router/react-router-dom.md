# react-router-dom

绝大部分的功能都是直接使用[react-router](./react-router.md)中的。只是添加了几个组件和 hook，专门用于 web 平台。

## How it works

BrowserRouter、HashRouter、HistoryRouter 的实现都和前面提到的 MemoryRouter 十分相似。因此我们会把注意力集中在那几个特有的组件。

第一个是 Link 组件，可以从当前路由跳转到指定的地方。它的内部实现并非简单地调用了 useNavigate，因为它最终渲染出来的是一个 a 标签。像是绑定事件，ref 等都可以传给它。注意 a 标签中同时有 href 属性和 onClick 属性，这样即保证了点击它可以跳转，也保留了右键在新标签页中打开的功能。

```tsx
export const Link = React.forwardRef<HTMLAnchorElement, LinkProps>(
  function LinkWithRef(
    { onClick, reloadDocument, replace = false, state, target, to, ...rest },
    ref
  ) {
    let href = useHref(to);
    let internalOnClick = useLinkClickHandler(to, { replace, state, target });
    function handleClick(
      event: React.MouseEvent<HTMLAnchorElement, MouseEvent>
    ) {
      if (onClick) onClick(event);
      if (!event.defaultPrevented && !reloadDocument) {
        internalOnClick(event);
      }
    }

    return (
      // eslint-disable-next-line jsx-a11y/anchor-has-content
      <a
        {...rest}
        href={href}
        onClick={handleClick}
        ref={ref}
        target={target}
      />
    );
  }
);
```

在处理点击的部分，使用了 useLinkClickHandler，它会在只有鼠标左键点击，并且 target 属性为空或者是`_self`的情况下处理点击事件。如果目标地址和当前地址相同，那么就会使用 replace 的方式跳转。

```ts
export function useLinkClickHandler<E extends Element = HTMLAnchorElement>(
  to: To,
  {
    target,
    replace: replaceProp,
    state,
  }: {
    target?: React.HTMLAttributeAnchorTarget;
    replace?: boolean;
    state?: any;
  } = {}
): (event: React.MouseEvent<E, MouseEvent>) => void {
  let navigate = useNavigate();
  let location = useLocation();
  let path = useResolvedPath(to);

  return React.useCallback(
    (event: React.MouseEvent<E, MouseEvent>) => {
      if (
        event.button === 0 && // Ignore everything but left clicks
        (!target || target === "_self") && // Let browser handle "target=_blank" etc.
        !isModifiedEvent(event) // Ignore clicks with modifier keys
      ) {
        event.preventDefault();

        // If the URL hasn't changed, a regular <a> will do a replace instead of
        // a push, so do the same here.
        let replace =
          !!replaceProp || createPath(location) === createPath(path);

        navigate(to, { replace, state });
      }
    },
    [location, navigate, path, replaceProp, state, target, to]
  );
}
```

接下来是 NavLink 组件，它在 Link 组件的基础上添加了一些功能。首先是它可以自动判断当前的地址和它代表的跳转地址是否一致，如果一致，那么内部的 isActive 为 true。再来，style 属性和 className 属性都可以是一个函数，接收 isActive 参数，生成对应的样式、类名。

```tsx
export const NavLink = React.forwardRef<HTMLAnchorElement, NavLinkProps>(
  function NavLinkWithRef(
    {
      "aria-current": ariaCurrentProp = "page",
      caseSensitive = false,
      className: classNameProp = "",
      end = false,
      style: styleProp,
      to,
      children,
      ...rest
    },
    ref
  ) {
    let location = useLocation();
    let path = useResolvedPath(to);

    let locationPathname = location.pathname;
    let toPathname = path.pathname;
    if (!caseSensitive) {
      locationPathname = locationPathname.toLowerCase();
      toPathname = toPathname.toLowerCase();
    }

    let isActive =
      locationPathname === toPathname ||
      (!end &&
        locationPathname.startsWith(toPathname) &&
        locationPathname.charAt(toPathname.length) === "/");

    let ariaCurrent = isActive ? ariaCurrentProp : undefined;

    let className: string;
    if (typeof classNameProp === "function") {
      className = classNameProp({ isActive });
    } else {
      // ...
      className = [classNameProp, isActive ? "active" : null]
        .filter(Boolean)
        .join(" ");
    }

    let style =
      typeof styleProp === "function" ? styleProp({ isActive }) : styleProp;

    return (
      <Link
        {...rest}
        aria-current={ariaCurrent}
        className={className}
        ref={ref}
        style={style}
        to={to}
      >
        {typeof children === "function" ? children({ isActive }) : children}
      </Link>
    );
  }
);
```

除了这两个组件外，还有一个特别的 hook，useSearchParams。使用它可以操纵 querystring，同时 url 也会相应变化。但是它的内部依赖于原生的[URLSearchParams](https://developer.mozilla.org/zh-CN/docs/Web/API/URLSearchParams)。所以如果支持 IE，还需要使用 polyfill。searchParams 来源于 defaultInit 和当前的 location.search，每当 location.search 变化的时候，searchParams 又会重新计算。需要改变的时候调用 setSearchParams 会引起路由的变化。

```ts
export function useSearchParams(defaultInit?: URLSearchParamsInit) {
  // ...
  let defaultSearchParamsRef = React.useRef(createSearchParams(defaultInit));

  let location = useLocation();
  let searchParams = React.useMemo(() => {
    let searchParams = createSearchParams(location.search);

    for (let key of defaultSearchParamsRef.current.keys()) {
      if (!searchParams.has(key)) {
        defaultSearchParamsRef.current.getAll(key).forEach((value) => {
          searchParams.append(key, value);
        });
      }
    }

    return searchParams;
  }, [location.search]);

  let navigate = useNavigate();
  let setSearchParams = React.useCallback(
    (
      nextInit: URLSearchParamsInit,
      navigateOptions?: { replace?: boolean; state?: any }
    ) => {
      navigate("?" + createSearchParams(nextInit), navigateOptions);
    },
    [navigate]
  );

  return [searchParams, setSearchParams] as const;
}
```

最后还有一个组件 StaticRouter，专门用于服务端渲染。实际上它几乎也没做什么，内部也是渲染了 Router 组件。只不过在 navigator 属性的处理上，禁止了所有的路由跳转操作。因为在服务器端没有客户端的路由系统。

```tsx
export function StaticRouter({
  basename,
  children,
  location: locationProp = "/",
}: StaticRouterProps) {
  // ...
  let staticNavigator = {
    createHref(to: To) {
      return typeof to === "string" ? to : createPath(to);
    },
    push(to: To) {
      throw new Error(
        `You cannot use navigator.push() on the server because it is a stateless ` +
          `environment. This error was probably triggered when you did a ` +
          `\`navigate(${JSON.stringify(to)})\` somewhere in your app.`
      );
    },
    // ...
  };
  return (
    <Router
      basename={basename}
      children={children}
      location={location}
      navigationType={action}
      navigator={staticNavigator}
      static={true}
    />
  );
}
```
