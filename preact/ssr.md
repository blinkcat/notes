# SSR

在服务器端使用[preact-render-to-string](https://github.com/preactjs/preact-render-to-string/)这个库将组件渲染成字符串。在客户端使用`hydrate`函数替代`render`函数渲染组件。

## preact-render-to-string

### demo

```js
import render from "preact-render-to-string";
import { h } from "preact";

const App = <div class="foo">content</div>;

console.log(render(App));
// <div class="foo">content</div>
```

JSX 片段被渲染成了字符串。

### How it works

通过前几篇文章的讲解，我们知道了 VNode 的结构就是一个普通的 Object。每个 VNode 有`type`属性标识它的类型。如果是 Component，那么 type 就是这个 Component 的构造函数、或是它自己。如果是原生的 dom 元素，那么 type 就是字符串，对应着节点的名称。而属性都在 props 中。在服务端渲染的时候，我们只需要首次渲染出的结果，也就是说只会让组件执行`render`方法，其他的生命周期钩子除了`getDerivedStateFromProps`和`componentWillMount`外都不会执行，包括`useEffect`和`useLayoutEffect`这两个带副作用的 hook 也不会执行。在得到渲染的结果后，也就是得到了个 VNode Tree 以后，就可以进行递归遍历了，将 VNode 用`<VNode.type ...VNode.props />`这种形式的字符串表示。最终我们会得到对应这个 VNode Tree 的 html。

第一个难题是如何让上面提到的那些方法都不执行呢？

首先是通过 preact 预留的扩展机制。就是引入的这个 options 对象，在 preact 运行的过程中会在各个关键时刻引用它内部的属性。`options.__s`就是`options._skipEffects`，将它设置为 true，那么在`useEffect`和`useLayoutEffect`中就不会执行传入的 callback 函数。

其次是不调用`commitRoot`函数，也就不会执行剩下的那些生命周期钩子和回调。`options.__c`就是`options._commit`，将 EMPTY_ARR 传进去，然后再把它清空。这么做没什么意义。

```js
import { options } from "preact";
// ...
function renderToString(vnode, context, opts) {
  context = context || {};
  opts = opts || {};

  // Performance optimization: `renderToString` is synchronous and we
  // therefore don't execute any effects. To do that we pass an empty
  // array to `options._commit` (`__c`). But we can go one step further
  // and avoid a lot of dirty checks and allocations by setting
  // `options._skipEffects` (`__s`) too.
  const previousSkipEffects = options.__s;
  options.__s = true;

  const res = _renderToString(vnode, context, opts);

  // options._commit, we don't schedule any effects in this library right now,
  // so we can pass an empty queue to this hook.
  if (options.__c) options.__c(vnode, EMPTY_ARR);
  EMPTY_ARR.length = 0;
  options.__s = previousSkipEffects;
  return res;
}
```

preact 的代码中有些变量被改成了更节约空间的名字，就像这里的`options.__s`。详细可见 preact 项目根目录下的`mangle.json`文件。

接着就开始进行 VNode 转 html 流程了，也就是`_renderToString`这个函数做的事情。这个处理流程可以说是前面 diff 过程的翻版了。根据不同 VNode 类型，有不同的处理方式。如果是`null`和`boolean`直接返回空字符串。

```js
if (vnode == null || typeof vnode === "boolean") {
  return "";
}
```

如果是字符串，直接转义特殊字符后返回。

```js
// #text nodes
if (typeof vnode !== "object") {
  return encodeEntities(vnode);
}
// ...
const ENCODED_ENTITIES = /[&<>"]/;

export function encodeEntities(input) {
  const s = String(input);
  if (!ENCODED_ENTITIES.test(s)) {
    return s;
  }
  return s
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;");
}
```

如果是数组类型，需要循环处理。

```js
if (Array.isArray(vnode)) {
  let rendered = "";
  for (let i = 0; i < vnode.length; i++) {
    if (pretty && i > 0) rendered += "\n";
    rendered += _renderToString(
      vnode[i],
      context,
      opts,
      inner,
      isSvgMode,
      selectValue
    );
  }
  return rendered;
}
```

如果是组件，这里的情况就有些复杂了。如果在需要`shallow render`的场景下，比如说 Snapshot test，我们只需要处理到组件这个层次，不再处理组件中可能渲染出的内容。例如下面的例子。

```jsx
const Foo = () => <div>foo</div>;
const App = (
  <div class="foo">
    <Foo />
  </div>
);

console.log(shallow(App));
// <div class="foo"><Foo /></div>
```

所以这个时候会获取组件名作为 nodeName，然后当成原生的 dom 类型处理。

```js
let nodeName = vnode.type;
// ...
if (opts.shallow && (inner || opts.renderRootComponent === false)) {
  nodeName = getComponentName(nodeName);
}
// ...
```

在一般情况下，组件本身不需要显示在最终的结果中。只有它渲染出的内容，才需要继续处理。因此我们需要让组件执行 render 方法，或者对于函数式组件来说，执行组件本身。当然这个过程并不只是简单的 render，还有处理 Context、state、生命周期钩子以及调用 preact options 中的一些函数等。

```js
if (typeof nodeName === "function") {
  // ...
  if (!nodeName.prototype || typeof nodeName.prototype.render !== "function") {
    // ...
    // stateless functional components
    rendered = nodeName.call(vnode.__c, props, cctx);
  } else {
    // ...
    rendered = c.render(c.props, c.state, c.context);
  }
  // ...
  return _renderToString(
    rendered,
    context,
    opts,
    opts.shallowHighOrder !== false,
    isSvgMode,
    selectValue
  );
}
```

接下来，就开始处理 props。这里看起来跳过了代表原生元素的 VNode 的处理，实际上没有。原生元素的 VNode.type 就是个字符串，可以直接用来拼接。

```js
	let s = '<' + nodeName,
```

循环处理 props 中的属性，然后将处理后的结果拼接到`s`中。有些属性比较特殊，需要特别处理，比如`style`、`children`、`dangerouslySetInnerHTML`等等。

```js
if (props) {
  let attrs = Object.keys(props);
  //   ...
  for (let i = 0; i < attrs.length; i++) {
    let name = attrs[i],
      v = props[name];
    // ...
    s += ` ${name}="${encodeEntities(v)}"`;
  }
}
```

在处理完 props 以后，我们得到的字符串`s`类似于这样`<nodeName a="b" c="d"`，还缺少`>`或是`/>`。这取决于这个 dom 有没有子元素。此外，如果入参中`opts.pretty`属性是 true，则要求输出的字符串是格式化过的。每一行前面要有合适的缩进。如果当前的`s`中有属性存在多行的情况，需要`>`再下一行显示。类似于下面这样。

```jsx
<Wrap
	a={1}
	b={
		Object {
			"a": 1,
			"b": false,
			"c": [Function f]
		}
	}
>
```

所以这里就需要做出判断，首先尝试去掉 s 中的一个换行和若干个空白字符得到 sub。如果 sub 不等于 s，并且 sub 中不再含有换行，就将 sub 赋值给 s。否则，如果 s 中有换行，s 的末尾加上一个换行。最后 s 再加上`>`。

```js
// ...
let pretty = opts.pretty,
  indentChar = pretty && typeof pretty === "string" ? pretty : "\t";
// ...
// account for >1 multiline attribute
if (pretty) {
  let sub = s.replace(/\n\s*/, " ");
  if (sub !== s && !~sub.indexOf("\n")) s = sub;
  else if (pretty && ~s.indexOf("\n")) s += "\n";
}

s += ">";
```

整个逻辑关键点就在于 s 本身是否存在换行。其实是有可能存在的，看我们上面的那个例子，某些属性打印出来时，是有换行的。那又为何是使用 replace 呢，因为如果在 replace 以后就不存在换行了，说明只有一个属性，且这个属性只有一行，所以不需要再加换行了。

```jsx
<Wrap
	b={
		Object {
			"a": 1,
			"b": false,
			"c": [Function f]
		}
	}
>

<Wrap a={1}>
```

接下来就是处理子节点了。代表原生元素的 VNode 的子节点都藏在 props.children 中。但是如果遇到 props.dangerouslySetInnerHTML 属性，优先级会降低。

如果 dangerouslySetInnerHTML 属性存在，且不为空。复杂的地方还是在格式化这里。如果 html 的长度超过 40、或者存在换行又或者存在`<`字符。那么就对它进行换行加缩进。`indent`函数会将传入的字符串中的换行符替换成换行加缩进符。这个函数很关键，后面我们还会遇到它。

```js
if (html) {
  // if multiline, indent.
  if (pretty && isLargeString(html)) {
    html = "\n" + indentChar + indent(html, indentChar);
  }
  s += html;
}
// ...
export let indent = (s, char) =>
  String(s).replace(/(\n+)/g, "$1" + (char || "\t"));

export let isLargeString = (s, length, ignoreLines) =>
  String(s).length > (length || 40) ||
  (!ignoreLines && String(s).indexOf("\n") !== -1) ||
  String(s).indexOf("<") !== -1;
```

接着处理 props.children，明显要比上面的复杂多了。将 propChildren 打平。然后循环处理其中的 VNode 后将得到的字符串放入 pieces 数组中。如果这些字符串中有满足`isLargeString`函数的条件的，hasLarge 为 true。进而导致每个字符串都要再处理一遍。可以看到代码中最后处理的过程中又调用了`indent`函数，这样会使得原先的`\n\t`变成`\n\t\t`，或是`\n\t\t`变成`\n\t\t\t`等等。这样在递归的过程中每行就形成了正确的缩进，很巧妙。

```js
let children;
// ...
if (propChildren != null && getChildren((children = []), propChildren).length) {
  let hasLarge = pretty && ~s.indexOf("\n");
  let lastWasText = false;

  for (let i = 0; i < children.length; i++) {
    let child = children[i];

    if (child != null && child !== false) {
      // ...
      let ret = _renderToString(
        child,
        context,
        opts,
        true,
        childSvgMode,
        selectValue
      );
      // ...
    }
  }
  if (pretty && hasLarge) {
    for (let i = pieces.length; i--; ) {
      pieces[i] = "\n" + indentChar + indent(pieces[i], indentChar);
    }
  }
}
```

然后就是根据条件，判断最后应该加上`/>`还是`</nodeName>`。

```js
if (pieces.length || html) {
  s += pieces.join("");
} else if (opts && opts.xml) {
  return s.substring(0, s.length - 1) + " />";
}

if (isVoid && !children && !html) {
  s = s.replace(/>$/, " />");
} else {
  if (pretty && ~s.indexOf("\n")) s += "\n";
  s += `</${nodeName}>`;
}

return s;
```

最后，这个库还有一个方法`renderToJsxString`，内部调用了`renderToString`函数，带入了默认的配置。

```js
let defaultOpts = {
  attributeHook,
  jsx: true,
  xml: false,
  functions: true,
  functionNames: true,
  skipFalseAttributes: true,
  pretty: "  ",
};

function renderToJsxString(vnode, context, opts, inner) {
  opts = assign(assign({}, defaultOpts), opts || {});
  return renderToString(vnode, context, opts, inner);
}
```

最大的不同就是这个`attributeHook`。有了它，在生成字符串时，如果遇到 Object、Array 或是 Function 这类的可以将其中的值打印出来。类似于这样，

```jsx
<Wrap
	b={
		Object {
			"a": 1,
			"b": false,
			"c": [Function f]
		}
	}
/>
```

这个 attributeHook 之所以能起作用有两点原因，一是我们前面在处理 props 的时候，留了可以扩展的地方，这个扩展方式和前面文章谈到的 preact 使用`options`对象做扩展是一个思路。

```js
if (props) {
  let attrs = Object.keys(props);
  // ...
  for (let i = 0; i < attrs.length; i++) {
    let name = attrs[i],
      v = props[name];
    // ...
    let hooked =
      opts.attributeHook &&
      opts.attributeHook(name, v, context, opts, isComponent);
    if (hooked || hooked === "") {
      s += hooked;
      continue;
    }
    // ...
  }
}
```

第二点是，引入了[pretty-format](https://github.com/facebook/jest/tree/main/packages/pretty-format)这个库。它可以将一些对象类型转成格式化后的字符串，甚至可以自定义如何序列化某一种类型的对象。比如说我们的组件的属性其实可以传入 JSX 片段。这就需要自定义的序列化方式，在遇到 JSX 的属性时，使用`renderToString`函数。

```js
let preactPlugin = {
  test(object) {
    return (
      object &&
      typeof object === "object" &&
      "type" in object &&
      "props" in object &&
      "key" in object
    );
  },
  print(val, print, indent) {
    return renderToString(val, preactPlugin.context, preactPlugin.opts, true);
  },
};

let prettyFormatOpts = {
  plugins: [preactPlugin],
};
// ...
value = prettyFormat(value, prettyFormatOpts);
```

## hydrate

在客户端代码中，需要使用`hydrate`函数来渲染组件。其内部还是调用`render`函数，但是入参有变化。

```js
export function hydrate(vnode, parentDom) {
  render(vnode, parentDom, hydrate);
}
```

而在 render 函数内部，isHydrating 为 true，这样传给 diff 函数的 oldVNode 为 null，excessDomChildren 为 slice.call(parentDom.childNodes)，oldDom 为 parentDom.firstChild。

```js
export function render(vnode, parentDom, replaceNode) {
  // ...
  let isHydrating = typeof replaceNode === "function";
  //   ...
  let oldVNode = isHydrating
    ? null
    : (replaceNode && replaceNode._children) || parentDom._children;
  // ...
  diff(
    // ...
    !isHydrating && replaceNode
      ? [replaceNode]
      : oldVNode
      ? null
      : parentDom.firstChild
      ? slice.call(parentDom.childNodes)
      : null,
    // ...
    !isHydrating && replaceNode
      ? replaceNode
      : oldVNode
      ? oldVNode._dom
      : parentDom.firstChild
    // ...
  );
  // ...
}
```

hydrate 的影响主要集中在`diffElementNodes`和`diffProps`中。

在 diffElementNodes 中，如果没有找到可复用的 dom，那么接下来会将 isHydrating 改成 false，意味着这个 dom 代表的子树会进行完整的 diff 流程。因为如果是在 hydrate 状态下，是不会进行 diffProps 操作的。除此之外，如果文本节点的内容在两端渲染后有差异，也是会进行修复的。

```js
function diffElementNodes(
  dom,
  // ...
  isHydrating
) {
  // ...
  let nodeType = newVNode.type;
  // ...
  if (dom == null) {
    // ...
    // we created a new parent, so none of the previously attached children can be reused:
    excessDomChildren = null;
    // we are creating a new node, so we can assume this is a new subtree (in case we are hydrating), this deopts the hydrate
    isHydrating = false;
  }
  if (nodeType === null) {
    // During hydration, we still have to split merged text from SSR'd HTML.
    if (oldProps !== newProps && (!isHydrating || dom.data !== newProps)) {
      dom.data = newProps;
    }
  } else {
    //   ...
    if (!isHydrating) {
      // ...
    }
    diffProps(dom, newProps, oldProps, isSvg, isHydrating);
    // ...
    // (as above, don't diff props during hydration)
    if (!isHydrating) {
      // ...
    }
  }
}
```

虽然在 diffElementNodes 函数中确实调用了 diffProps，但我们可以从 diffProps 函数中看到，在 hydrate 状态下，只会处理事件绑定。

```js
export function diffProps(dom, newProps, oldProps, isSvg, hydrate) {
  let i;

  // ...

  for (i in newProps) {
    if (
      (!hydrate || typeof newProps[i] == "function") &&
      i !== "children" &&
      i !== "key" &&
      i !== "value" &&
      i !== "checked" &&
      oldProps[i] !== newProps[i]
    ) {
      setProperty(dom, i, newProps[i], oldProps[i], isSvg);
    }
  }
}
```

总的来说，在 hydrate 模式下，preact 只会对当前已有的 dom tree 进行事件绑定，不会对节点属性差异做检查。在遇到文本节点内容差异时会进行修正，或是客户端渲染的结果和当前某些 dom 节点完全无法匹配时，会对这个局部进行 diff 操作。事实上这也和 React 中的逻辑相同。

> 与 render() 相同，但它用于在 ReactDOMServer 渲染的容器中对 HTML 的内容进行 hydrate 操作。React 会尝试在已有标记上绑定事件监听器。

> React 希望服务端与客户端渲染的内容完全一致。React 可以弥补文本内容的差异，但是你需要将不匹配的地方作为 bug 进行修复。在开发者模式下，React 会对 hydration 操作过程中的不匹配进行警告。但并不能保证在不匹配的情况下，修补属性的差异。由于性能的关系，这一点非常重要，因为大多是应用中不匹配的情况很少见，并且验证所有标记的成本非常昂贵。

> 如果单个元素的属性或者文本内容，在服务端和客户端之间有无法避免差异（比如：时间戳），则可以为元素添加 suppressHydrationWarning={true} 来消除警告。这种方式只在一级深度上有效，应只作为一种应急方案（escape hatch）。请不要过度使用！除非它是文本内容，否则 React 仍不会尝试修补差异，因此在未来的更新之前，仍会保持不一致。
