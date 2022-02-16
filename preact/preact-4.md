# preact ErrorBoundary

根据文档中的定义

> 错误边界是一种 React 组件，这种组件可以捕获发生在其子组件树任何位置的 JavaScript 错误，并打印这些错误，同时展示降级 UI，而并不会渲染那些发生崩溃的子组件树。错误边界可以捕获发生在整个子组件树的渲染期间、生命周期方法以及构造函数中的错误。

catch error 主要存在于组件创建、执行 render 方法、生命周期钩子的地方。

1. diff 函数中，内部执行组件 render 方法，一部分生命周期钩子。

```js
export function diff(
  // ...
  newVNode,
  oldVNode
  // ...
) {
  // ...
  try {
    // ...
  } catch (e) {
    // ...
    options._catchError(e, newVNode, oldVNode);
  }
}
```

2. commitRoot 内部执行了`componentDidMount`、`componentDidUpdate`等方法。

```js
export function commitRoot(commitQueue, root) {
  // ...
  commitQueue.some((c) => {
    try {
      // ...
    } catch (e) {
      options._catchError(e, c._vnode);
    }
  });
}
```

3. applyRef 更新 ref 的地方。

```js
export function applyRef(ref, value, vnode) {
  try {
    // ...
  } catch (e) {
    options._catchError(e, vnode);
  }
}
```

4. unmount 执行组件`componentWillUnmount`钩子的地方。

```js
export function unmount(vnode, parentVNode, skipRemove) {
  // ...
  if (r.componentWillUnmount) {
    try {
      r.componentWillUnmount();
    } catch (e) {
      options._catchError(e, parentVNode);
    }
  }
  // ...
}
```

当捕获到错误时，按照定义会向上寻找 ErrorBoundary。

> 如果一个 class 组件中定义了 static getDerivedStateFromError() 或 componentDidCatch() 这两个生命周期方法中的任意一个（或两个）时，那么它就变成一个错误边界。当抛出错误后，请使用 static getDerivedStateFromError() 渲染备用 UI ，使用 componentDidCatch() 打印错误信息。

```js
export function _catchError(error, vnode) {
  /** @type {import('../internal').Component} */
  let component, ctor, handled;

  for (; (vnode = vnode._parent); ) {
    if ((component = vnode._component) && !component._processingException) {
      try {
        ctor = component.constructor;

        if (ctor && ctor.getDerivedStateFromError != null) {
          component.setState(ctor.getDerivedStateFromError(error));
          handled = component._dirty;
        }

        if (component.componentDidCatch != null) {
          component.componentDidCatch(error);
          handled = component._dirty;
        }

        // This is an error boundary. Mark it as having bailed out, and whether it was mid-hydration.
        if (handled) {
          return (component._pendingError = component);
        }
      } catch (e) {
        error = e;
      }
    }
  }

  throw error;
}
```

首先会检查组件是否定义了`getDerivedStateFromError`或是`componentDidCatch`。然后检查组件的`_dirty`属性，判断这个组件是否对错误有反应，也就是说是否会通过更新的方式处理错误。如果这个组件会更新，那就将组件的`_pendingError`属性设置为组件自己。如果不处理，那么就把这个错误再抛出去。

在 diff 函数中有两段代码。`_pendingError`会赋值给`_processingException`，然后在 diff 结束的时候二者都被清空，表示错误已经被处理了。

```js
if (oldVNode._component) {
  c = newVNode._component = oldVNode._component;
  clearProcessingException = c._processingException = c._pendingError;
}
// ...
if (clearProcessingException) {
  c._pendingError = c._processingException = null;
}
```

但是在\_catchError 函数中，有个条件判断，只有在这个 ErrorBoundary 不包含\_processingException 属性的时候才会处理错误。如果在 ErrorBoundary 因为处理错误而产生的 diff 中再次出错，那么这个错误会跳过这个 ErrorBoundary 再向上传递。

## Recover from errors

让我们再回到 diff 函数，看看它是怎么尝试从错误中恢复的。如果上层没有 ErrorBoundary 处理 diff 过程中的错误，显然会导致整个应用崩溃。而 ErrorBoundary 所做的处理就是更新自己，重新开启 diff 过程。

先看 catch 语句捕获到错误时，diff 函数怎么保存一些必要的信息。首先是 newVNode.\_original 置为 null。这句的逻辑和 Suspense 组件有关。接着当首次渲染的时候，excessDomChildren 可能有值，那么就把 oldDom 保存在 newVNode 的\_dom 属性中，\_hydrating 属性设置成一个 boolean 类型。excessDomChildren 数组中如果有 oldDom，就把它的位置用 null 代替。因为既然我们在这里处理了错误，那么后面的程序还可以执行，最终 excessDomChildren 数组中不为 null 的位置表示的 dom 都会被删除。

```js
// diff/index.js
function diff(
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
  // If the previous diff bailed out, resume creating/hydrating.
  if (oldVNode._hydrating != null) {
    isHydrating = oldVNode._hydrating;
    oldDom = newVNode._dom = oldVNode._dom;
    // if we resume, we want the tree to be "unlocked"
    newVNode._hydrating = null;
    excessDomChildren = [oldDom];
  }
  // ...
  try {
    // ...
  } catch (e) {
    newVNode._original = null;
    // if hydrating or creating initial tree, bailout preserves DOM:
    if (isHydrating || excessDomChildren != null) {
      newVNode._dom = oldDom;
      newVNode._hydrating = !!isHydrating;
      excessDomChildren[excessDomChildren.indexOf(oldDom)] = null;
      // ^ could possibly be simplified to:
      // excessDomChildren.length = 0;
    }
    options._catchError(e, newVNode, oldVNode);
  }
}
```

当 ErrorBoundary 处理的错误，说明新的更新开始了，原来的 newVNode 就是这个 diff 中的 oldVNode。这时它的\_hydrating 不为 null，就可以开始恢复 isHydrating、oldDom、excessDomChildren 等变量，继续 diff 流程。但是这里有个问题，如果这个报错的组件是一个被`memo`函数包裹的组件，在下次的 diff 中它不会被 diff，最终也就不会被渲染出来。详细可以看[这里](https://github.com/preactjs/preact/issues/3449)。
