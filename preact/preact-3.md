# Preact Update

这一章来看看组件的更新方式。

抛开函数式组件的 hooks 不谈。类式组件中存在两个方法来更新组件，`setState`和`forceUpdate`。唯一的区别是 forceUpdate 会跳过 shouldComponentUpdate 方法，强制更新组件。

setState 的代码很简单，回调函数被放入了组件的\_renderCallbacks 数组中，留着后面的 commitRoot 函数处理。然后调用`enqueueRender`函数，传入组件的实例。

```js
Component.prototype.setState = function (update, callback) {
  // only clone state when copying to nextState the first time.
  let s;
  if (this._nextState != null && this._nextState !== this.state) {
    s = this._nextState;
  } else {
    s = this._nextState = assign({}, this.state);
  }

  if (typeof update == "function") {
    // Some libraries like `immer` mark the current state as readonly,
    // preventing us from mutating it, so we need to clone it. See #2716
    update = update(assign({}, s), this.props);
  }

  if (update) {
    assign(s, update);
  }

  // Skip update if updater function returned null
  if (update == null) return;

  if (this._vnode) {
    if (callback) this._renderCallbacks.push(callback);
    enqueueRender(this);
  }
};
```

而 forceUpdate 则是更简单，不同的是将`_force`标识置为 true，表明更新是通过 forceUpdate 发起的。

```js
Component.prototype.forceUpdate = function (callback) {
  if (this._vnode) {
    // Set render mode so that we can differentiate where the render request
    // is coming from. We need this because forceUpdate should never call
    // shouldComponentUpdate
    this._force = true;
    if (callback) this._renderCallbacks.push(callback);
    enqueueRender(this);
  }
};
```

enqueueRender 函数在被调用后，会检查组件的`_dirty`属性，将其置为 true，然后将组件放入 rerenderQueue 中，并且还要检查 process.\_rerenderCount。保证 process 在未处理完 rerenderQueue 中的组件更新之前，不会再次调用。而 process 其实是被异步调用的。

```js
let rerenderQueue = [];
// ...
let prevDebounce;

export function enqueueRender(c) {
  if (
    (!c._dirty &&
      (c._dirty = true) &&
      rerenderQueue.push(c) &&
      !process._rerenderCount++) ||
    prevDebounce !== options.debounceRendering
  ) {
    prevDebounce = options.debounceRendering;
    (prevDebounce || defer)(process);
  }
}
// ...
process._rerenderCount = 0;
```

从这个 defer 函数就可以看出。

```js
const defer =
  typeof Promise == "function"
    ? Promise.prototype.then.bind(Promise.resolve())
    : setTimeout;
```

我们再看 process 函数，它在统一更新之前会对 rerenderQueue 中的组件进行排序。按照`_depth`从小到大的顺序，其实就是先父组件后子组件的顺序更新。全部更新完毕后才会将 process.\_rerenderCount 的值置为 0。这样防止 process 函数多次调用。这里有个注意点，在调用 renderComponent 函数之前，会先判断这个组件的\_dirty 是否为 true。而 renderComponent 中会调用 diff，完成一个 component 的 diff 工作后会再将它的\_dirty 置为 false。这样即使 rerenderQueue 中存在多个相同的组件，也只会对这个组件做一次 diff 操作。这都归功于异步的方式更新组件。

```js
function process() {
  let queue;
  while ((process._rerenderCount = rerenderQueue.length)) {
    queue = rerenderQueue.sort((a, b) => a._vnode._depth - b._vnode._depth);
    rerenderQueue = [];
    // Don't update `renderCount` yet. Keep its value non-zero to prevent unnecessary
    // process() calls from getting scheduled while `queue` is still being consumed.
    queue.some((c) => {
      if (c._dirty) renderComponent(c);
    });
  }
}
process._rerenderCount = 0;
```

到 renderComponent 函数这里才开始真正的更新组件。因为这里会调用 diff 函数。关注点首先在 diff 的入参这里。之前我们看的 setState 方法实际上是修改了组件中的`_nextState`变量，在 diff 函数中，这个变量会在合适的时候赋值给`state`变量。而组件需要更新的原因也是因为自身的 state 变化，或是外部传入的属性变化才需要更新。所以这里的 oldVNode 直接使用了组件当前的 vnode，只不过改变了`_original`。而 oldDom 会优先使用 vnode.\_dom，或是下一个兄弟 VNode 的\_dom。

```js
function renderComponent(component) {
  let vnode = component._vnode,
    oldDom = vnode._dom,
    parentDom = component._parentDom;

  if (parentDom) {
    let commitQueue = [];
    const oldVNode = assign({}, vnode);
    oldVNode._original = vnode._original + 1;

    diff(
      parentDom,
      vnode,
      oldVNode,
      component._globalContext,
      parentDom.ownerSVGElement !== undefined,
      vnode._hydrating != null ? [oldDom] : null,
      commitQueue,
      oldDom == null ? getDomSibling(vnode) : oldDom,
      vnode._hydrating
    );
    commitRoot(commitQueue, vnode);

    if (vnode._dom != oldDom) {
      updateParentDomPointers(vnode);
    }
  }
}
```

在完成更新之后，还需要向上递归更新 VNode 的父 VNode 中的\_dom 指针。但是要判断父节点是不是代表一个 component。

```js
function updateParentDomPointers(vnode) {
  if ((vnode = vnode._parent) != null && vnode._component != null) {
    vnode._dom = vnode._component.base = null;
    for (let i = 0; i < vnode._children.length; i++) {
      let child = vnode._children[i];
      if (child != null && child._dom != null) {
        vnode._dom = vnode._component.base = child._dom;
        break;
      }
    }

    return updateParentDomPointers(vnode);
  }
}
```

总结一下，通过 setState 和 forceUpdate 发起的更新，会使得组件进入 rerenderQueue 队列中等待异步触发 diff。而真正进入 diff 过程后，整个过程是同步进行的。
