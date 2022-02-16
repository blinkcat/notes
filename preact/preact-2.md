# Preact Diff

因为 diff 的过程复杂，所以我们单独开一章来讲解 diff。

## Main idea

既然是在比较(diff)新旧 VNode Tree 的过程中更新 Dom Tree。那么这个过程必定会遍历整个 newVNode 中的节点，和 oldVNode 中对应的节点做比较。如果存在，说明可能是发生了修改。如果不存在，说明是新增操作，这时候需要根据情况 appendChild 或者是 insertChild。而 oldVNode 中没有被对比到的节点都认为被删除了，执行 removeChild 操作。

## diff

首先看 diff 函数的入参，有 9 个之多。

```js
export function diff(
  parentDom,
  newVNode,
  oldVNode,
  globalContext,
  isSvg,
  excessDomChildren,
  commitQueue,
  oldDom,
  isHydrating
) {
  // ...
}
```

配合 render 中调用 diff 的方式，我们来看看每个入参的用途。这对我们了解后面的 diff 过程很有帮助。

```js
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
```

- parentDom 代表父 Dom 元素
- newVNode 新的 VNode Tree
- oldVNode 旧的 VNode Tree，第一个渲染时是一个空对象 EMPTY_OBJ
- globalContext 表示 Context 对象，默认是 EMPTY_OBJ
- isSvg 当前元素是否表示一个 svg
- excessDomChildren 表示将要更新的宿主 dom 下的子元素
- commitQueue 渲染完成后，需要执行的函数队列
- oldDom 在 diff 时 oldVNode 关联的 Dom 节点
- isHydrating 是否是在 hydrate 过程中

replaceNode 是 render 函数的第三个参数。如果传入了这个参数，那么 excessDomChildren 就是\[replaceNode\],oldDom 就是 replaceNode。

diff 自身是处理单个 newVNode 的过程，首先会根据 newVNode 的类型(newType)来决定处理方式。

如果是函数类型，那么这个 newVNode 必然代表一个 Component。

如果`newVNode._original === oldVNode._original`那么当前的 newVNode 可能代表一个`string`、`number`或是`bigint`。还可能代表一个独立的 PreactElement 片段，

```jsx
const element = (
  <div>
    <Comp />
  </div>
);
```

因为代表一个片段的 VNode，它的`_original`的值是不会变的。

最后一种情况，newVNode 代表原生的 Dom 元素，这时它的 type 类型是字符串。

```js
function diff() {
  let tmp,
    newType = newVNode.type;
  // ...
  try {
    outer: if (typeof newType == "function") {
      // ...
    } else if (
      excessDomChildren == null &&
      newVNode._original === oldVNode._original
    ) {
      // ...
    } else {
      // ...
    }
  } catch (e) {
    // ...
  }
}
```

我们先看第一种情况，newVNode 表示一个 Component。

首先会尝试获取当前的 Context 对象，class 声明的组件或者 context.Consumer 都可以有 contextType 这个静态属性。这个属性存放着 context 对象。而 globalContext 默认为空对象，在 diff 的过程中，如果组件存在 getChildContext 方法，就会将这个方法的返回值拷贝到 globalContext 中。而我们的 context.Provider 组件就有这个方法，因此 tmp.\_id 为 key，Provider 实例为 vaule 就会被复制到 globalContext 中。这时我们就可以通过 provider.props.value 获取到 context 中存放的值。

```js
let tmp,
  newType = newVNode.type;
// ...
try {
  outer: if (typeof newType == "function") {
    tmp = newType.contextType;
    let provider = tmp && globalContext[tmp._id];
    let componentContext = tmp
      ? provider
        ? provider.props.value
        : tmp._defaultValue
      : globalContext;
    //   ...
    if (c.getChildContext != null) {
      globalContext = assign(assign({}, globalContext), c.getChildContext());
    }
    // ...
  }
  // ...
} catch {
  // ...
}
```

接着是根据情况，复用旧的组件实例，或是创建新的实例。在创建新的函数式组件时，初始化了一个空的 Component 类实例，并且将它的 render 方法用 doRender 函数替代。其实就是把这个函数当成了 render 方法用。这样就在内部统一了两种组件。新创建的组件实例上还会加一些属性，比如 props、state、context 等。同时如果使用了外层的 Context，还要订阅这个 provider。

```js
// Get component and set it to `c`
if (oldVNode._component) {
  c = newVNode._component = oldVNode._component;
  clearProcessingException = c._processingException = c._pendingError;
} else {
  // Instantiate the new component
  if ("prototype" in newType && newType.prototype.render) {
    // @ts-ignore The check above verifies that newType is suppose to be constructed
    newVNode._component = c = new newType(newProps, componentContext); // eslint-disable-line new-cap
  } else {
    // @ts-ignore Trust me, Component implements the interface we want
    newVNode._component = c = new Component(newProps, componentContext);
    c.constructor = newType;
    c.render = doRender;
  }
  if (provider) provider.sub(c);

  c.props = newProps;
  if (!c.state) c.state = {};
  c.context = componentContext;
  c._globalContext = globalContext;
  isNew = c._dirty = true;
  c._renderCallbacks = [];
}
// ...
/** The `.render()` method for a PFC backing instance. */
function doRender(props, state, context) {
  return this.constructor(props, context);
}
```

在组件创建完毕后，可以开始调用各个生命周期的钩子了。在 render 方法执行之前的钩子都会立即调用，比如`getDerivedStateFromProps`、`componentWillMount`、`componentWillReceiveProps`等待。而执行之后才需要调用的方法会放到组件自身的`_renderCallbacks`队列中，等待最终所有 diff 完成后在 commitRoot 函数中执行。

```js
if (c.componentDidMount != null) {
  c._renderCallbacks.push(c.componentDidMount);
}
// ...
if (c.componentDidUpdate != null) {
  c._renderCallbacks.push(() => {
    c.componentDidUpdate(oldProps, oldState, snapshot);
  });
}
```

这里有个比较特殊的钩子是`shouldComponentUpdate`方法。如果它存在，并且返回 false，那么在执行一些复制工作后退出 diff 函数。它的子孙 VNode 也不需要再 diff 了。

```js
// ...
if (
	(!c._force &&
		c.shouldComponentUpdate != null &&
		c.shouldComponentUpdate(
			newProps,
			c._nextState,
			componentContext
		) === false) ||
	newVNode._original === oldVNode._original
) {
	c.props = newProps;
	c.state = c._nextState;
	// More info about this here: https://gist.github.com/JoviDeCroock/bec5f2ce93544d2e6070ef8e0036e4e8
	if (newVNode._original !== oldVNode._original) c._dirty = false;
	c._vnode = newVNode;
	newVNode._dom = oldVNode._dom;
	newVNode._children = oldVNode._children;
	newVNode._children.forEach(vnode => {
		if (vnode) vnode._parent = newVNode;
	});
	if (c._renderCallbacks.length) {
		commitQueue.push(c);
	}

	break outer;
}
// ...
```

最后就是执行 render 方法了。render 方法返回的 VNode 就是当前 newVNode 的子 VNode。需要继续进行 diff 操作，所以传入 diffChildren 函数继续处理。

```js
// ...
tmp = c.render(c.props, c.state, c.context);
// ...
let isTopLevelFragment =
  tmp != null && tmp.type === Fragment && tmp.key == null;
let renderResult = isTopLevelFragment ? tmp.props.children : tmp;

diffChildren(
  parentDom,
  Array.isArray(renderResult) ? renderResult : [renderResult],
  newVNode,
  oldVNode,
  globalContext,
  isSvg,
  excessDomChildren,
  commitQueue,
  oldDom,
  isHydrating
);
// ...
```

最后，如果组件的\_renderCallbacks 不为空，就将它放入 commitQueue 队列中。整个 diff 结束后会执行其中的函数。

```js
if (c._renderCallbacks.length) {
  commitQueue.push(c);
}
```

## diffElementNodes

紧接上文，如果 newVNode 代表一个原生的 Dom 类型，那么最终会调用 diffElementNodes 函数。

```js
newVNode._dom = diffElementNodes(
  oldVNode._dom,
  newVNode,
  oldVNode,
  globalContext,
  isSvg,
  excessDomChildren,
  commitQueue,
  isHydrating
);
```

diffElementNodes 函数最终返回一个 Dom 原生，赋值给 newVNode.\_dom。这个属性代表这个 VNode 所关联的 Dom。

如果 excessDomChildren 不为空，就尝试从中寻找可复用的 dom 节点。只要是类型相同就行。

如果 dom 还是为空，那么就要根据 newNode.type 来创建一个 Dom 节点。如果 nodeType 为 null，说明这是个文本节点，创建完成以后直接返回，不需要进行后续操作。最后把 excessDomChildren 置为 null，表示接下来都无法再使用 excessDomChildren 中的 dom 节点。

```js
let nodeType = newVNode.type;

if (dom == null) {
  if (nodeType === null) {
    // @ts-ignore createTextNode returns Text, we expect PreactElement
    return document.createTextNode(newProps);
  }

  if (isSvg) {
    dom = document.createElementNS(
      "http://www.w3.org/2000/svg",
      // @ts-ignore We know `newVNode.type` is a string
      nodeType
    );
  } else {
    dom = document.createElement(
      // @ts-ignore We know `newVNode.type` is a string
      nodeType,
      newProps.is && newProps
    );
  }
  // we created a new parent, so none of the previously attached children can be reused:
  excessDomChildren = null;
  // we are creating a new node, so we can assume this is a new subtree (in case we are hydrating), this deopts the hydrate
  isHydrating = false;
}
```

如果 excessDomChildren 不是 null，说明我们复用了里面的某个节点，这时应该将它的子节点赋值给 excessDomChildren，表示下面如果需要调用 diffChildren 处理子节点时，应该从这里寻找可复用的 dom 节点。

如果 dom 不是文本节点，先去看看 props 中是否有`dangerouslySetInnerHTML`这个特殊的属性，它需要单独处理。如果我们有复用 excessDomChildren 中的 dom 节点，那么就要取出它的属性，留作后续的 diffProps 操作。

```js
let oldProps = oldVNode.props;
let newProps = newVNode.props;
// ...
excessDomChildren = excessDomChildren && slice.call(dom.childNodes);

let oldHtml = oldProps.dangerouslySetInnerHTML;
let newHtml = newProps.dangerouslySetInnerHTML;
// During hydration, props are not diffed at all (including dangerouslySetInnerHTML)
// @TODO we should warn in debug mode when props don't match here.
if (!isHydrating) {
  // But, if we are in a situation where we are using existing DOM (e.g. replaceNode)
  // we should read the existing DOM attributes to diff them
  if (excessDomChildren != null) {
    oldProps = {};
    for (i = 0; i < dom.attributes.length; i++) {
      oldProps[dom.attributes[i].name] = dom.attributes[i].value;
    }
  }

  if (newHtml || oldHtml) {
    // Avoid re-applying the same '__html' if it did not changed between re-render
    if (
      !newHtml ||
      ((!oldHtml || newHtml.__html != oldHtml.__html) &&
        newHtml.__html !== dom.innerHTML)
    ) {
      dom.innerHTML = (newHtml && newHtml.__html) || "";
    }
  }
}
```

接着就可以开始更新 Dom 节点的 props。比如事件、className、style 等。

```js
diffProps(dom, newProps, oldProps, isSvg, isHydrating);
```

diffProps 函数只会在这里调用，因为 Component 的 props 和 state 最终都是作用在 Dom 节点的属性和结构上的。

如果 props 中存在 dangerouslySetInnerHTML，这就表示这个 VNode 代表的 Dom 元素不由 preact 控制。否则，还是需要调用 diffChildren 函数来处理子孙 VNode。如果 excessDomChildren 不为 null，那么传给 diffChildren 函数的 oldDom 入参就是 excessDomChildren 中的第一个元素，excessDomChildren 在上面已经被重新赋值过了。

```js
let newHtml = newProps.dangerouslySetInnerHTML;
// ...
if (newHtml) {
  newVNode._children = [];
} else {
  i = newVNode.props.children;
  diffChildren(
    dom,
    Array.isArray(i) ? i : [i],
    newVNode,
    oldVNode,
    globalContext,
    isSvg && nodeType !== "foreignObject",
    excessDomChildren,
    commitQueue,
    excessDomChildren
      ? excessDomChildren[0]
      : oldVNode._children && getDomSibling(oldVNode, 0),
    isHydrating
  );
  // Remove children that are not part of any vnode.
  if (excessDomChildren != null) {
    for (i = excessDomChildren.length; i--; ) {
      if (excessDomChildren[i] != null) removeNode(excessDomChildren[i]);
    }
  }
}
```

在 diffChildren 完成后，还需要清理 excessDomChildren 数组中没被复用的 dom 节点。

最后，还需要单独处理 dom 变量代表的元素的`value`、`checked`属性。

## diffChildren

这块的逻辑涉及到新旧 VNode Tree 和 Dom Tree 共三棵树的比较，逻辑分支较多。diff 和 diffElementNodes 两个地方都会调用到 diffChildren，用来比较下层子 VNode。在比较完成的同时，会进行 Dom Tree 的修改。所以入参中有个 oldDom，表示此次 diff 所关联的真实 dom。

```js
export function diffChildren(
  parentDom,
  renderResult,
  newParentVNode,
  oldParentVNode,
  globalContext,
  isSvg,
  excessDomChildren,
  commitQueue,
  oldDom,
  isHydrating
) {
  // ...
}
```

VNode 的子节点存放在`_children`属性中，一开始为 null。在 diffChildren 中，新 VNode Tree 会通过 renderResult 入参来构建自己的`_children`。这个入参的来源和 VNode 的类型有关，如果是 Component 类型，就是 render 方法的返回值，如果是浏览器原生的元素，就是`props.children`。

循环 renderResult，对 childVNode 进行一些处理后再放入 newParentVNode.\_children 中。在这里，childNode 中的一些私有属性也会被更新，比如这里的`_parent`和`_depth`。它们一般都是在内部使用。

```js
newParentVNode._children = [];

for (i = 0; i < renderResult.length; i++) {
  childVNode = renderResult[i];

  if (childVNode == null || typeof childVNode == "boolean") {
    childVNode = newParentVNode._children[i] = null;
  }
  // ...
  else if (
    typeof childVNode == "string" ||
    typeof childVNode == "number" ||
    // eslint-disable-next-line valid-typeof
    typeof childVNode == "bigint"
  ) {
    childVNode = newParentVNode._children[i] = createVNode(
      null,
      childVNode,
      null,
      null,
      childVNode
    );
  } else if (Array.isArray(childVNode)) {
    childVNode = newParentVNode._children[i] = createVNode(
      Fragment,
      { children: childVNode },
      null,
      null,
      null
    );
  } else if (childVNode._depth > 0) {
    // ...
    childVNode = newParentVNode._children[i] = createVNode(
      childVNode.type,
      childVNode.props,
      childVNode.key,
      null,
      childVNode._original
    );
  } else {
    childVNode = newParentVNode._children[i] = childVNode;
  }
  // ...
  if (childVNode == null) {
    continue;
  }

  childVNode._parent = newParentVNode;
  childVNode._depth = newParentVNode._depth + 1;
}
```

紧接着，要通过 `key`、`type`，从旧 VNode Tree 的子 VNode 中寻找和这个 childVNode 对应的 VNode。首先尝试当前序号的 oldVNode，如果不是，那只能循环查找了。能否找到分别对应这个 childVNode 是本次渲染新增的还是修改的。注意如果能找到，oldChildren[i]会被置为 undefined。剩下的没有被置为 undefined 的 oldVNode 会被 unMount。

```js
let oldChildren = (oldParentVNode && oldParentVNode._children) || EMPTY_ARR;
// ...
// Check if we find a corresponding element in oldChildren.
// If found, delete the array item by setting to `undefined`.
// We use `undefined`, as `null` is reserved for empty placeholders
// (holes).
oldVNode = oldChildren[i];

if (
  oldVNode === null ||
  (oldVNode &&
    childVNode.key == oldVNode.key &&
    childVNode.type === oldVNode.type)
) {
  oldChildren[i] = undefined;
} else {
  // ...
  for (j = 0; j < oldChildrenLength; j++) {
    oldVNode = oldChildren[j];
    // ...
    if (
      oldVNode &&
      childVNode.key == oldVNode.key &&
      childVNode.type === oldVNode.type
    ) {
      oldChildren[j] = undefined;
      break;
    }
    oldVNode = null;
  }
}

oldVNode = oldVNode || EMPTY_OBJ;
```

然后，比较 childVNode 和 oldNode。

```js
diff(
  parentDom,
  childVNode,
  oldVNode,
  globalContext,
  isSvg,
  excessDomChildren,
  commitQueue,
  oldDom,
  isHydrating
);
```

因为是进行的深度优先 diff，所以我们最终可得到`childVNode._dom`。这个\_dom 是真实 Dom 节点，或者是 undefined。因为整棵 VNode Tree 的叶子节点必然是原生的 dom 类型，或者是 null。

```js
newDom = childVNode._dom;
```

根据 diffElementNodes 中的逻辑，如果 newDom 存在，那么它可能是复用的 oldVNode.\_dom，或者是新创建的 Dom 节点。然将它和 oldDom 比较一下，来决定是应该在 parentDom 中 appendChild 还是根据 oldDom 的位置 insertNode，亦或是直接复用 oldDom。这部分逻辑都在 placeChild 函数中。reorderChildren 函数是用来纠正 oldDom，因为在 renderResult 的循环处理中，oldDom 每次都需要对准目前 Dom Tree 中对应的节点。如果存在 pureComponent 这类的组件，它代表的子 VNode Tree 可能不需要 diff。因此 oldDom 要跳过这颗 VNode Tree 对应的 Dom Tree。至于`childVNode._nextDom`和`newParentVNode._nextDom`，它们表示的是在处理这个 component 类型的 VNode 时，对应的 oldDom 的值。详细的我们在讲解 placeChild 函数的时候会提到。

```js
if (newDom != null) {
  if (firstChildDom == null) {
    firstChildDom = newDom;
  }

  if (
    typeof childVNode.type == "function" &&
    childVNode._children === oldVNode._children
  ) {
    childVNode._nextDom = oldDom = reorderChildren(
      childVNode,
      oldDom,
      parentDom
    );
  } else {
    oldDom = placeChild(
      parentDom,
      childVNode,
      oldVNode,
      oldChildren,
      newDom,
      oldDom
    );
  }

  if (typeof newParentVNode.type == "function") {
    // ...
    newParentVNode._nextDom = oldDom;
  }
} else if (
  oldDom &&
  oldVNode._dom == oldDom &&
  oldDom.parentNode != parentDom
) {
  // The above condition is to handle null placeholders. See test in placeholder.test.js:
  // `efficiently replace null placeholders in parent rerenders`
  oldDom = getDomSibling(oldVNode);
}
```

如果 newDom 是 undefined，说明在新的 VNode Tree 中，有 component 返回 null。那么原先表示这个 component 的 子 VNode 已经被 unMount 了。这时候 oldDom 要变成 oldVNode 的下个兄弟 VNode 的\_dom。

在 diffChildren 中，还有对 ref 的处理。新旧 VNode 的 ref 存在 refs 数组中，并且在函数的末尾更新 ref。

```js
// ...
if ((j = childVNode.ref) && oldVNode.ref != j) {
  if (!refs) refs = [];
  if (oldVNode.ref) refs.push(oldVNode.ref, null, childVNode);
  refs.push(j, childVNode._component || newDom, childVNode);
}
// ....
if (refs) {
  for (i = 0; i < refs.length; i++) {
    applyRef(refs[i], refs[++i], refs[++i]);
  }
}
// ...
export function applyRef(ref, value, vnode) {
  try {
    if (typeof ref == "function") ref(value);
    else ref.current = value;
  } catch (e) {
    options._catchError(e, vnode);
  }
}
```

另外，还有对新 VNode Tree 中被删除的旧 VNode 的处理。oldChildren[i]不为空的都需要 unmount，但是还要记得更新`newParentVNode._nextDom`，跳过将要被删除的 dom 节点。

```js
// Remove remaining oldChildren if there are any.
for (i = oldChildrenLength; i--; ) {
  if (oldChildren[i] != null) {
    if (
      typeof newParentVNode.type == "function" &&
      oldChildren[i]._dom != null &&
      oldChildren[i]._dom == newParentVNode._nextDom
    ) {
      // If the newParentVNode.__nextDom points to a dom node that is about to
      // be unmounted, then get the next sibling of that vnode and set
      // _nextDom to it
      newParentVNode._nextDom = getDomSibling(oldParentVNode, i + 1);
    }

    unmount(oldChildren[i], oldChildren[i]);
  }
}
```

接着我们就来看最难理解的`placeChild`函数。

placeChild 函数的职责就是将 newDom 放置到 parentDom 中正确的位置，然后返回更新后的 oldDom。

如果 childVNode.\_nextDom 不为 undefined，说明这个 childVNode 是 component 类型，在处理它的 children 时，更新过它的\_nextDom 属性。因此可以用它的\_nextDom 属性直接更新 oldDom。

```js
function placeChild(
  parentDom,
  childVNode,
  oldVNode,
  oldChildren,
  newDom,
  oldDom
) {
  let nextDom;
  if (childVNode._nextDom !== undefined) {
    // ...
    nextDom = childVNode._nextDom;
    // ...
    childVNode._nextDom = undefined;
  } else if (
    oldVNode == null ||
    newDom != oldDom ||
    newDom.parentNode == null
  ) {
    outer: if (oldDom == null || oldDom.parentNode !== parentDom) {
      parentDom.appendChild(newDom);
      nextDom = null;
    } else {
      // `j<oldChildrenLength; j+=2` is an alternative to `j++<oldChildrenLength/2`
      for (
        let sibDom = oldDom, j = 0;
        (sibDom = sibDom.nextSibling) && j < oldChildren.length;
        j += 2
      ) {
        if (sibDom == newDom) {
          break outer;
        }
      }
      parentDom.insertBefore(newDom, oldDom);
      nextDom = oldDom;
    }
  }
  // ...
  if (nextDom !== undefined) {
    oldDom = nextDom;
  } else {
    oldDom = newDom.nextSibling;
  }

  return oldDom;
}
```

再看第二个分支，条件是新增了 VNode，或者是新的 VNode 和旧的 VNode 里渲染的 dom 节点不一致。第三种情况`newDom.parentNode == null`是只用在使用`Suspense`组件时才会命中的条件。进入第二个分支后，如果 oldDom 是 null，说明这一定是新增了一个 VNode 或是前面的兄弟节点移动到了后面。这时候执行`appendChild`操作，同时 oldDom 保持为 null。至于那个`oldDom.parentNode !== parentDom`，也是在使用`Suspense`组件时，才会命中。当 oldDom 存在，且和 newDom 不同时，可能是顺序发生了变化。这里操作是向下寻找最多一半的兄弟节点，如果存在和 newDom 相同的节点，退出。oldDom 更新为 newDom 的下一个兄弟节点。如果不存在，将 newDom 插入到 oldDom 前面，最终 oldDom 的值保持不变。

我们来看个例子，假设有两颗子树，新的 VNode Tree children 顺序发生了变化。

```js
//  	old          	new
		<div>			<div>
oldDom->	<A/>			<D/> <-newDom
			<B/>			<C/>
			<C/>			<B/>
			<D/>			<A/>
		</div>			</div>
```

1. oldDom 指向 A，newDom 指向 D。A!=D，A 向下查找一半兄弟节点，没找到 D。所以将 D 插入到 A 的前面。这时候顺序为 D-A-B-C，oldDom 还是指向 A。因为外层的 for 循环处理 renderResult，newDom 指向了 C。
2. 继续处理，从 A 向下找一半，还是没找到 C。所以将 C 插入到 A 的前面。顺序为，D-C-A-B。
3. oldDom 指向 A，newDom 指向 B。这次在下方可以找到 B 了。oldDom 更新为 B 的下个兄弟节点，因为 B 是最后一个节点，所以 oldDom 为 null。
4. newDom 现在指向 A，因为 oldDom 是 null，所以执行 appendChild 操作，将 A 添加到父节点的尾部。最终结果是 D-C-B-A。

所以整个的操作为两次 insertBefore，一次 appendChild。

接着我们来看下`reorderChildren`函数，只有当前 VNode 是不需要 diff 的 pureComponent 时才会调用这个函数。它的职责是更新子 VNode 的 parent，更新 oldDom。

```js
// ...
if (
  typeof childVNode.type == "function" &&
  childVNode._children === oldVNode._children
) {
  childVNode._nextDom = oldDom = reorderChildren(childVNode, oldDom, parentDom);
}
// ...
function reorderChildren(childVNode, oldDom, parentDom) {
  // Note: VNodes in nested suspended trees may be missing _children.
  let c = childVNode._children;
  let tmp = 0;
  for (; c && tmp < c.length; tmp++) {
    let vnode = c[tmp];
    if (vnode) {
      // We typically enter this code path on sCU bailout, where we copy
      // oldVNode._children to newVNode._children. If that is the case, we need
      // to update the old children's _parent pointer to point to the newVNode
      // (childVNode here).
      vnode._parent = childVNode;

      if (typeof vnode.type == "function") {
        oldDom = reorderChildren(vnode, oldDom, parentDom);
      } else {
        oldDom = placeChild(parentDom, vnode, vnode, c, vnode._dom, oldDom);
      }
    }
  }

  return oldDom;
}
```

可以看到 reorderChildren 中很明显的递归结构，如果是 Component 类型，进入递归。如果是原生类型，执行 placeChild。最终的效果是使得 oldDom 跳过整个 pureComponent 的 dom 结构。我们来看两个例子。
如果组件内部渲染出一个 div，那么 oldDom 指向这个 div，vnode.\_dom 也是指向这个 div。最终 oldDom=newDom.nextSibling。

```js
// pureComponent render
<div />
```

如果组件内部是一个复杂的结构。首先是一个 fragment，进入 reorderChildren 递归，循环处理四个子 VNode。这时 oldDom 指向 A 渲染出的 div。A 进入 reorderChildren，然后对 A 中 div 执行 placeChild 操作。这时候返回的 oldDom 是 A 下面的 div。接着对这个 div 执行 placeChild 操作，oldDom 变成 p 节点，下面的 B 同理，最终 oldDom 指向 B 中的 p 节点后面的兄弟节点。

```js
// pureComponent render
<>
  <A />
  <div />
  <p />
  <B />
</>

// 实际反应在dom tree中的结构
// 其他节点...
<div/>
<div/>
<p/>
<p/>
// 其他节点...
```

最后我们再来看下`unmount`是如何实现的。首先更新 ref，然后调用组件的`componentWillUnmount`钩子，接着递归处理子 VNode，最后执行删除 dom 操作。

```js
for (i = oldChildrenLength; i--; ) {
  if (oldChildren[i] != null) {
    // ...
    unmount(oldChildren[i], oldChildren[i]);
  }
}
// ...
export function unmount(vnode, parentVNode, skipRemove) {
  let r;
  if (options.unmount) options.unmount(vnode);

  if ((r = vnode.ref)) {
    if (!r.current || r.current === vnode._dom) applyRef(r, null, parentVNode);
  }

  if ((r = vnode._component) != null) {
    if (r.componentWillUnmount) {
      try {
        r.componentWillUnmount();
      } catch (e) {
        options._catchError(e, parentVNode);
      }
    }

    r.base = r._parentDom = null;
  }

  if ((r = vnode._children)) {
    for (let i = 0; i < r.length; i++) {
      if (r[i]) {
        unmount(r[i], parentVNode, typeof vnode.type != "function");
      }
    }
  }

  if (!skipRemove && vnode._dom != null) removeNode(vnode._dom);

  // Must be set to `undefined` to properly clean up `_nextDom`
  // for which `null` is a valid value. See comment in `create-element.js`
  vnode._dom = vnode._nextDom = undefined;
}
```

## diffProps

用来将 props 的变动反应在 Dom 节点上。先根据 oldProps，将 dom 中对应的属性值设置为 null。然后根据 newProps，将 dom 中对应的属性设置为新的值。

```js
export function diffProps(dom, newProps, oldProps, isSvg, hydrate) {
  let i;

  for (i in oldProps) {
    if (i !== "children" && i !== "key" && !(i in newProps)) {
      setProperty(dom, i, null, oldProps[i], isSvg);
    }
  }

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

一般 dom 上常见的 props 有 style、className、事件等。我们单独来看看事件是如何处理的。

类似于`onClick`、`onChange`这类事件都会存在`dom._listeners`对象中。绑定的时候，绑定函数是`eventProxy`或`eventProxyCapture`。触发事件的时候，根据事件名，直接在这个对象中查找处理函数。这样做的好处是保证了绑定函数引用的稳定，方便解绑。当更新绑定函数的时候，只需要更新对象中对应的属性。

```js
// ...
if (name[0] === "o" && name[1] === "n") {
  useCapture = name !== (name = name.replace(/Capture$/, ""));

  // Infer correct casing for DOM built-in events:
  if (name.toLowerCase() in dom) name = name.toLowerCase().slice(2);
  else name = name.slice(2);

  if (!dom._listeners) dom._listeners = {};
  dom._listeners[name + useCapture] = value;

  if (value) {
    if (!oldValue) {
      const handler = useCapture ? eventProxyCapture : eventProxy;
      dom.addEventListener(name, handler, useCapture);
    }
  } else {
    const handler = useCapture ? eventProxyCapture : eventProxy;
    dom.removeEventListener(name, handler, useCapture);
  }
}
// ...
function eventProxy(e) {
  this._listeners[e.type + false](options.event ? options.event(e) : e);
}

function eventProxyCapture(e) {
  this._listeners[e.type + true](options.event ? options.event(e) : e);
}
```

## commitRoot

在所有的 diff 工作完成后，就要处理那些还未执行的钩子以及回调函数了。例如`componentDidMount`、`componentDidUpdate`、setState 的 callback 等等。它们都被放到了组件的`_renderCallbacks`数组里面。

因此，commitRoot 函数的工作就是将这些函数依次执行。

```js
export function commitRoot(commitQueue, root) {
  if (options._commit) options._commit(root, commitQueue);

  commitQueue.some((c) => {
    try {
      // @ts-ignore Reuse the commitQueue variable here so the type changes
      commitQueue = c._renderCallbacks;
      c._renderCallbacks = [];
      commitQueue.some((cb) => {
        // @ts-ignore See above ts-ignore on commitQueue
        cb.call(c);
      });
    } catch (e) {
      options._catchError(e, c._vnode);
    }
  });
}
```
