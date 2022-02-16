# Preact Compat

为了能够[兼容](https://preactjs.com/guide/v10/getting-started#aliasing-react-to-preact)React 而创建的具有和 React 相同 Api 的包。

## Children

实现了和[React.Children](https://zh-hans.reactjs.org/docs/react-api.html#reactchildren)一致的 Api。

大部分代码都很好理解，唯独 mapFn 那里 toChildArray 包裹了两次很奇怪。

```js
export function toChildArray(children, out) {
  out = out || [];
  if (children == null || typeof children == "boolean") {
  } else if (Array.isArray(children)) {
    children.some((child) => {
      toChildArray(child, out);
    });
  } else {
    out.push(children);
  }
  return out;
}
// ...
const mapFn = (children, fn) => {
  if (children == null) return null;
  return toChildArray(toChildArray(children).map(fn));
};

// This API is completely unnecessary for Preact, so it's basically passthrough.
export const Children = {
  map: mapFn,
  forEach: mapFn,
  count(children) {
    return children ? toChildArray(children).length : 0;
  },
  only(children) {
    const normalized = toChildArray(children);
    if (normalized.length !== 1) throw "Children.only";
    return normalized[0];
  },
  toArray: toChildArray,
};
```

## forwardRef

实现了[React.forwardRef](https://zh-hans.reactjs.org/docs/react-api.html#reactforwardref)的功能。先看个例子，

最终 ref.current 指向的是 button 元素。

```js
const FancyButton = React.forwardRef((props, ref) => (
  <button ref={ref} className="FancyButton">
    {props.children}
  </button>
));

// You can now get a ref directly to the DOM button:
const ref = React.createRef();
<FancyButton ref={ref}>Click me!</FancyButton>;
```

再看实现，可以看到好多地方都为了`mobx-react`这个库做了兼容。forwardRef 函数其实是个高阶组件。它返回了一个新组件 Forwarded，并且组件的\_forwarded 属性为 true。在 Forwarded 内部，从 props 中取出 ref，或者使用入参传入的 ref。接着调用 fn，传入取出 ref 属性的 props 和 ref，然后返回执行结果。

```js
export function forwardRef(fn) {
  // We always have ref in props.ref, except for
  // mobx-react. It will call this function directly
  // and always pass ref as the second argument.
  function Forwarded(props, ref) {
    let clone = assign({}, props);
    delete clone.ref;
    ref = props.ref || ref;
    return fn(
      clone,
      !ref || (typeof ref === "object" && !("current" in ref)) ? null : ref
    );
  }

  // mobx-react checks for this being present
  Forwarded.$$typeof = REACT_FORWARD_SYMBOL;
  // mobx-react heavily relies on implementation details.
  // It expects an object here with a `render` property,
  // and prototype.render will fail. Without this
  // mobx-react throws.
  Forwarded.render = Forwarded;

  Forwarded.prototype.isReactComponent = Forwarded._forwarded = true;
  Forwarded.displayName = "ForwardRef(" + (fn.displayName || fn.name) + ")";
  return Forwarded;
}
```

最奇怪的是 props 属性中居然会有 ref，我们从`createElement`函数中可以发现，ref 和 key 都是要从 props 中取出的，最终给组件的的 props 不可能含有 ref。这个其实是故意放进去的。只要是 VNode.type.\_forwarded 为 true，并且 VNode 中存在 ref，都会将 ref 从 VNode 中移到 props 中。这样在 Forwarded 内才能拿到这个 ref，再转发给下层组件。

```js
let oldDiffHook = options._diff;
options._diff = (vnode) => {
  if (vnode.type && vnode.type._forwarded && vnode.ref) {
    vnode.props.ref = vnode.ref;
    vnode.ref = null;
  }
  if (oldDiffHook) oldDiffHook(vnode);
};
```

## memo

实现了[React.memo](https://zh-hans.reactjs.org/docs/react-api.html#reactmemo)。也是一个高阶组件。返回的 Memoed 组件实现了 shouldComponentUpdate 方法。默认使用 shallowDiffers 来比较两次的 props 是否相同。

```js
export function memo(c, comparer) {
  function shouldUpdate(nextProps) {
    let ref = this.props.ref;
    let updateRef = ref == nextProps.ref;
    if (!updateRef && ref) {
      ref.call ? ref(null) : (ref.current = null);
    }

    if (!comparer) {
      return shallowDiffers(this.props, nextProps);
    }

    return !comparer(this.props, nextProps) || !updateRef;
  }

  function Memoed(props) {
    this.shouldComponentUpdate = shouldUpdate;
    return createElement(c, props);
  }
  Memoed.displayName = "Memo(" + (c.displayName || c.name) + ")";
  Memoed.prototype.isReactComponent = true;
  Memoed._forwarded = true;
  return Memoed;
}
// ...
export function shallowDiffers(a, b) {
  for (let i in a) if (i !== "__source" && !(i in b)) return true;
  for (let i in b) if (i !== "__source" && a[i] !== b[i]) return true;
  return false;
}
```

## createPortal

和 React 中的[createPortal](https://zh-hans.reactjs.org/docs/portals.html)差不多，不同之处在于事件冒泡上不能通过 VNode Tree 来传递。

```js
export function createPortal(vnode, container) {
  return createElement(Portal, { _vnode: vnode, _container: container });
}
```

在 Portal 组件内部利用了 render 方法来实现在另一个 dom 下进行渲染。在渲染的时候，并不是直接使用 container 这个 dom 节点。而是新建一个\_temp 对象，模拟一个 dom 节点。这是为了能够支持在 container 这个节点下，同时渲染多个 Portal，而不会导致后面的覆盖前面的。

在 Context 方面，由于 ContextProvider 组件的存在，它将自己的 context 传递给子 VNode。

```js
function Portal(props) {
  const _this = this;
  let container = props._container;

  _this.componentWillUnmount = function () {
    render(null, _this._temp);
    _this._temp = null;
    _this._container = null;
  };
  // ...
  if (props._vnode) {
    if (!_this._temp) {
      _this._container = container;

      // Create a fake DOM parent node that manages a subset of `container`'s children:
      _this._temp = {
        nodeType: 1,
        parentNode: container,
        childNodes: [],
        appendChild(child) {
          this.childNodes.push(child);
          _this._container.appendChild(child);
        },
        // ...
      };
    }
    // Render our wrapping element into temp.
    render(
      createElement(ContextProvider, { context: _this.context }, props._vnode),
      _this._temp
    );
  }
  //   ...
}
// ...
function ContextProvider(props) {
  this.getChildContext = () => props.context;
  return props.children;
}
```

## PureComponent

PureComponent 的实现就更简单了，实现一个默认的 shouldComponentUpdate，对 props 和 state 做浅比较。

```js
export function PureComponent(p) {
  this.props = p;
}
PureComponent.prototype = new Component();
// Some third-party libraries check if this property is present
PureComponent.prototype.isPureReactComponent = true;
PureComponent.prototype.shouldComponentUpdate = function (props, state) {
  return shallowDiffers(this.props, props) || shallowDiffers(this.state, state);
};
```

## render

这部分就比较杂了，需要了解 React 的某些细节，以及 React 周边的第三方库是如何接触涉及到 React 具体实现的一些地方。之后就要着手抹平这些[差异](https://preactjs.com/guide/v10/differences-to-react)。

### Event

首先是事件部分，React 采用了[合成事件](https://zh-hans.reactjs.org/docs/events.html)，而 preact 使用了原生的 dom 事件。

preact/compat 会把大部分 onchange 替换成 oninput，因为在 dom 事件体系中，它两的差别还是挺大的。

> 和 input 事件不一样，change 事件并不是每次元素的 value 改变时都会触发。

可能大部分情况下 oninput 是更好的选择。

```js
// Input types for which onchange should not be converted to oninput.
// type="file|checkbox|radio", plus "range" in IE11.
// (IE11 doesn't support Symbol, which we use here to turn `rad` into `ra` which matches "range")
const onChangeInputType = (type) =>
  (typeof Symbol != "undefined" && typeof Symbol() == "symbol"
    ? /fil|che|rad/i
    : /fil|che|ra/i
  ).test(type);
//   ...
if (
  /^onchange(textarea|input)/i.test(i + type) &&
  !onChangeInputType(props.type)
) {
  i = "oninput";
}
```

还有一些事件名称在 dom 中和 react 中是不同的，

```js
if (/ondoubleclick/i.test(i)) {
  i = "ondblclick";
} else if (
  /^onchange(textarea|input)/i.test(i + type) &&
  !onChangeInputType(props.type)
) {
  i = "oninput";
} else if (/^onfocus$/i.test(i)) {
  i = "onfocusin";
} else if (/^onblur$/i.test(i)) {
  i = "onfocusout";
} else if (/^on(Ani|Tra|Tou|BeforeInp|Compo)/.test(i)) {
  i = i.toLowerCase();
}
```

接着是 event 对象，将原生的 event 对象打扮成 React event 的样子。

```js
let oldEventHook = options.event;
options.event = (e) => {
  if (oldEventHook) e = oldEventHook(e);
  e.persist = empty;
  e.isPropagationStopped = isPropagationStopped;
  e.isDefaultPrevented = isDefaultPrevented;
  return (e.nativeEvent = e);
};

function empty() {}

function isPropagationStopped() {
  return this.cancelBubble;
}

function isDefaultPrevented() {
  return this.defaultPrevented;
}
```

### special props

还有一些特殊的属性和类型，比如说 noscript 标签、defaultValue 和 value、download 属性的 value、class 和 className、svg 中的一些驼峰形式的属性名、select 元素的多选特性等等。这些差异都通过 preact 自己预留的扩展机制来抹平。

```js
export const REACT_ELEMENT_TYPE =
  (typeof Symbol != "undefined" && Symbol.for && Symbol.for("react.element")) ||
  0xeac7;
// ...
let oldVNodeHook = options.vnode;
options.vnode = (vnode) => {
  // ...
  vnode.$$typeof = REACT_ELEMENT_TYPE;

  if (oldVNodeHook) oldVNodeHook(vnode);
};
```

### render hydrate

react 中的 render 函数会清空宿主元素中原本的子节点，且会返回挂载后的组件实例。所以这里会通过 parent.\_children 来判断是否是首次渲染，如果是，就清空宿主中的子节点。

```js
export function render(vnode, parent, callback) {
  // React destroys any existing DOM nodes, see #1727
  // ...but only on the first render, see #1828
  if (parent._children == null) {
    parent.textContent = "";
  }

  preactRender(vnode, parent);
  if (typeof callback == "function") callback();

  return vnode ? vnode._component : null;
}

export function hydrate(vnode, parent, callback) {
  preactHydrate(vnode, parent);
  if (typeof callback == "function") callback();

  return vnode ? vnode._component : null;
}
```

## Suspense

实现的略复杂，且还在试验阶段，所以跳过。

## jsx-runtime

react17 引入的[更新](https://reactjs.org/blog/2020/09/22/introducing-the-new-jsx-transform.html)。在实现上和之前提到的`createElement`大同小异。

```js
let vnodeId = 0;
function createVNode(type, props, key, __source, __self) {
  // We'll want to preserve `ref` in props to get rid of the need for
  // forwardRef components in the future, but that should happen via
  // a separate PR.
  let normalizedProps = {},
    ref,
    i;
  for (i in props) {
    if (i == "ref") {
      ref = props[i];
    } else {
      normalizedProps[i] = props[i];
    }
  }

  const vnode = {
    type,
    props: normalizedProps,
    key,
    ref,
    _children: null,
    _parent: null,
    _depth: 0,
    _dom: null,
    _nextDom: undefined,
    _component: null,
    _hydrating: null,
    constructor: undefined,
    _original: --vnodeId,
    __source,
    __self,
  };

  // If a Component VNode, check for and apply defaultProps.
  // Note: `type` is often a String, and can be `undefined` in development.
  if (typeof type === "function" && (ref = type.defaultProps)) {
    for (i in ref)
      if (typeof normalizedProps[i] === "undefined") {
        normalizedProps[i] = ref[i];
      }
  }

  if (options.vnode) options.vnode(vnode);
  return vnode;
}

export {
  createVNode as jsx,
  createVNode as jsxs,
  createVNode as jsxDEV,
  Fragment,
};
```

## renderToString

用于服务端渲染，依赖于另外一个包，preact-render-to-string。

```js
var renderToString;
try {
  const mod = require("preact-render-to-string");
  renderToString = mod.default || mod.renderToString || mod;
} catch (e) {
  throw Error(
    'renderToString() error: missing "preact-render-to-string" dependency.'
  );
}

module.exports = {
  renderToString: renderToString,
  renderToStaticMarkup: renderToString,
};
```
