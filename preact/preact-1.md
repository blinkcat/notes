# Preact

版本[10.6.5](https://github.com/preactjs/preact/tree/10.6.5)

类 react 框架，使用方式和 react 几乎一致，所以也被用来替换 react，换取体积上的优势。

## How it works

利用虚拟节点(VNode) Tree 表示 Dom Tree，然后比较(diff)新旧 VNode Tree，决定如何更新 Dom Tree。引入了 Component 的概念，一个 Component 表示一块功能内聚的 VNode(子)树。更新则是以 Component 为最小粒度。

## VNode

首先我们要先了解什么是 VNode 和 PreactElement。PreactElement 就是由 VNode 组成的，所以本质上它也是一个 VNode。

从`createElement`函数中可以看出，最终的返回值是通过`createVNode`函数创建的 VNode。

```js
export function createElement(type, props, children) {
  let normalizedProps = {},
    key,
    ref,
    i;
  for (i in props) {
    if (i == "key") key = props[i];
    else if (i == "ref") ref = props[i];
    else normalizedProps[i] = props[i];
  }

  if (arguments.length > 2) {
    normalizedProps.children =
      arguments.length > 3 ? slice.call(arguments, 2) : children;
  }

  // ...

  return createVNode(type, normalizedProps, key, ref, null);
}
```

这个函数很少会直接使用，更常见的方式是通过 JSX，

```jsx
<A>
  <B />
  <div>Hello!</div>
</A>
```

但这只是语法糖而已，最终会通过 Babel 转换成调用 createElement 函数的方式生成 VNode。

而 VNode 其实只是一个普通的 Object 而已，并没有什么特别的。

```ts
let vnodeId = 0;

export function createVNode(type, props, key, ref, original) {
  const vnode = {
    type,
    props,
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
    _original: original == null ? ++vnodeId : original,
  };
  // ...
  return vnode;
}
```

比较常用的`cloneElement`函数也是直接操作 vnode。将新的 props 和 children 复制到 vnode 原先的 props 中。

```js
export function cloneElement(vnode, props, children) {
  let normalizedProps = assign({}, vnode.props),
    key,
    ref,
    i;
  for (i in props) {
    if (i == "key") key = props[i];
    else if (i == "ref") ref = props[i];
    else normalizedProps[i] = props[i];
  }

  if (arguments.length > 2) {
    normalizedProps.children =
      arguments.length > 3 ? slice.call(arguments, 2) : children;
  }

  return createVNode(
    vnode.type,
    normalizedProps,
    key || vnode.key,
    ref || vnode.ref,
    null
  );
}
```

## Component

Component 的定义方式有两种，一种是通过 class 方法，另一种是通过 function 方式。第二种是目前的主流方式。

```js
export function Component(props, context) {
  this.props = props;
  this.context = context;
}

Component.prototype.setState = function (update, callback) {
  // ...
};

Component.prototype.forceUpdate = function (callback) {
  // ...
};

Component.prototype.render = Fragment;

export function Fragment(props) {
  return props.children;
}
```

Component 的初始定义很简单，两个属性 props 和 context，分别表示组件在使用时接收到的属性和上下文对象。两个方法 setState 和 forceUpdate 用来更新组件。还有一个 render 方法，用来决定渲染的内容。默认值是 Fragment。

两种定义组件的方式本质上都是使用了这里的 Component 类，后续我们会详细了解。这里还要说的是 Component 和 VNode 的关系。下面的代码中定义了两个组件 A、B。在通过 JSX 方式的使用过程中，会被 Babel 转换为 createElement 函数调用。这个时候生成的 VNode 的 type 就是这个组件的构造函数，而代表原生的元素的 VNode 的 type 是字符串。可以通过 type 的类型来区分组件和原生的元素。

```jsx
class A extends Component {
  render() {
    return this.props.children
  }
}

function B() {
  return <div />;
}

<A>
  <B />
</A>
// createElement(A, {}, createElement(B, {}))
<div />
// createElement('div')
```

而判断一个对象是否是 preact element 的方法就是看它的构造函数是否为 undefined。除了显式地在自己的结构中定义 constructor 为 undefined，只能通过`Object.create(null)`的方式。

```js
export const isValidElement = (vnode) =>
  vnode != null && vnode.constructor === undefined;
```

这样做是因为[安全](https://github.com/preactjs/preact/issues/2118#issuecomment-553407900)上的考量。因此在 diff 过程中，如果发现 VNode 的 constructor 属性不为 undefined，会拒绝渲染这个 VNode。

```js
// When passing through createElement it assigns the object
// constructor as undefined. This to prevent JSON-injection.
if (newVNode.constructor !== undefined) return null;
```

## Fragment

Fragment 其实就是一个只返回 children 的函数式组件。

```js
export function Fragment(props) {
  return props.children;
}
```

## Context

Context 在使用方式上和普通的组件没太大的区别，

```jsx
const Theme = createContext("light");

function ThemedButton(props) {
  return (
    <Theme.Consumer>
      {(theme) => (
        <button {...props} class={"btn " + theme}>
          Themed Button
        </button>
      )}
    </Theme.Consumer>
  );
}

<Theme.Provider value="dark">
  <ThemedButton />
</Theme.Provider>;
```

通过 createContext 函数创建的 Context 对象有两个组件，一个是 Provider 负责提供上下文的 value，一个是 Consumer 可以随时为下层组件提供最新的 value 值。

```jsx
export let i = 0;

export function createContext(defaultValue, contextId) {
  contextId = "__cC" + i++;

  const context = {
    _id: contextId,
    _defaultValue: defaultValue,
    /** @type {import('./internal').FunctionComponent} */
    Consumer(props, contextValue) {
      // ...
      return props.children(contextValue);
    },
    /** @type {import('./internal').FunctionComponent} */
    Provider(props) {
      if (!this.getChildContext) {
        let subs = [];
        let ctx = {};
        ctx[contextId] = this;

        this.getChildContext = () => ctx;

        this.shouldComponentUpdate = function (_props) {
          if (this.props.value !== _props.value) {
            // ...
            subs.some(enqueueRender);
          }
        };

        this.sub = (c) => {
          subs.push(c);
          let old = c.componentWillUnmount;
          c.componentWillUnmount = () => {
            subs.splice(subs.indexOf(c), 1);
            if (old) old.call(c);
          };
        };
      }

      return props.children;
    },
  };

  // ...
  return (context.Provider._contextRef = context.Consumer.contextType =
    context);
}
```

## Ref

只是一个普通的对象，里面有 current 属性。

```js
export function createRef() {
  return { current: null };
}
```

从上面的代码中可以看出，Consumer 的实现使用了 render props 的方式。而 Provider 虽然是一个函数式组件，但是也实现了`shouldComponentUpdate`方法。这是因为在 preact 中，两种声明方式最终在创建组件的时候都是采用 new xxx(props, context)的方式，函数式组件的函数会被当做 render 方法使用。Provider 中有一个 subs 数组和一个 sub 方法，用来收集使用到 Context 的组件。当 Context 的 value 改变时，shouldComponentUpdate 方法被触发，subs 数组中的组件会进行更新。注意 Provider 自身会通过 getChildContext 方法暴露出去。

## render

有了 VNode 以后，还需要将它表示的 Dom Tree 渲染到实际的页面中。这时候就需要 render 函数。

```js
export function render(vnode, parentDom, replaceNode) {
  // ...
  let oldVNode = isHydrating
    ? null
    : (replaceNode && replaceNode._children) || parentDom._children;

  vnode = ((!isHydrating && replaceNode) || parentDom)._children =
    createElement(Fragment, null, [vnode]);

  // List of effects that need to be called after diffing.
  let commitQueue = [];
  diff(
    parentDom,
    // Determine the new vnode tree and store it on the DOM element on
    // our custom `_children` property.
    vnode,
    oldVNode || EMPTY_OBJ,
    EMPTY_OBJ,
    parentDom.ownerSVGElement !== undefined,
    !isHydrating && replaceNode
      ? [replaceNode]
      : oldVNode
      ? null
      : parentDom.firstChild
      ? slice.call(parentDom.childNodes)
      : null,
    commitQueue,
    !isHydrating && replaceNode
      ? replaceNode
      : oldVNode
      ? oldVNode._dom
      : parentDom.firstChild,
    isHydrating
  );

  // Flush all queued effects
  commitRoot(commitQueue, vnode);
}
```

render 函数接收三个参数，实际上常用的就是前两个。第一个表示 VNode，第二个表示挂载的 Dom 节点。入参的 vnode 要用 Fragment 包一下，因为 vnode 可能为 null，为了统一处理方便。还有一个原因是在 React 中的 render 函数需要返回渲染后的组件实例，如果你渲染的是普通的原生元素，就不存在组件了。在函数内部，会将上次的 VNode 保存在 parentDom.\_children 属性中，当然第一次渲染的时候不存在，所以首次渲染 oldVNode 是 undefined。然后就开始了比较两个 VNode Tree 的过程，这个过程中的操作封装在 diff 函数中。在比较完成后，Dom Tree 也已经更新完毕，这时候就要调用对应的生命周期钩子，如 componentDidMount、setState 的回调函数等。这些都在 commitRoot 函数中完成。

至于第三个参数，参考下面的 demo，target node 被我们渲染的组件更新了。preact 的 render 函数并不会影响除了 replaceNode 之外，parentDom 下面其他的子节点。

```js
// DOM tree before render:
// <div id="container">
//   <div>bar</div>
//   <div id="target">foo</div>
// </div>

import { render } from "preact";

const Foo = () => <div id="target">BAR</div>;

render(
  <Foo />,
  document.getElementById("container"),
  document.getElementById("target")
);

// After render:
// <div id="container">
//   <div>bar</div>
//   <div id="target">BAR</div>
// </div>
```
