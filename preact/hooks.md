# Hooks

hooks 是后出的概念，在 preact 的实现中将它作为插件来实现。

## Preact Plugin

preact 的插件系统使用了类似于`事件点`这样的方式来实现。

在代码中，我们可以看到多处尝试执行特定函数的地方。

比如在 diff 函数中，

```js
export function diff() {
  // ...
  // ...
  if ((tmp = options._diff)) tmp(newVNode);
  // ...
  if ((tmp = options._render)) tmp(newVNode);
  //   ...
  if ((tmp = options.diffed)) tmp(newVNode);
}
```

commitRoot 函数中，

```js
export function commitRoot(commitQueue, root) {
  if (options._commit) options._commit(root, commitQueue);
  // ...
}
```

甚至 createVNode 中...

```js
export function createVNode(type, props, key, ref, original) {
  // ...
  if (original == null && options.vnode != null) options.vnode(vnode);
  // ...
}
```

而这些地方执行的可以是我们自定义的函数，用来达到特定的目的。

在 hooks 的实现中，就使用这种方式，重新定义 options 对象中的某些函数，例如，

```js
// ...
let currentComponent;
// ...
let oldBeforeDiff = options._diff;
// ...
options._diff = (vnode) => {
  currentComponent = null;
  if (oldBeforeDiff) oldBeforeDiff(vnode);
};
```

这样在某些时刻，hook 相关的代码就能得到执行。

## Hooks Structure

hooks 虽然是个函数，但是有这严格的使用规则。

> 只能在函数最外层调用 Hook。不要在循环、条件判断或者子函数中调用。

> 只能在 React 的函数组件中调用 Hook。不要在其他 JavaScript 函数中调用。

这些规则其实就暗示了 hooks 的数据结构隐藏在组件的实例中，且强依赖于调用顺序。

其实几乎在每个 hook 中，都会调用`getHookState`函数。这个函数的第一个参数代表当前调用的 hook 的序号，第二个参数 type 表示的是哪种 hook。但是 currentHook 变量的优先级较高，这是因为有些 hook 的实现其实是基于其他的 hook，为了能够区分开来。currentComponent 变量表示当前正在执行 render 的组件，它是通过前面提到的插件得到的。可以看到\_\_hooks 对象存在于组件实例上，有两个属性。\_list 数组存着组件内所有使用到的 hooks 的数据，而\_pendingEffects 数组存着 hooks 定义的副作用，会在合适的时候执行。

```js
function getHookState(index, type) {
  if (options._hook) {
    options._hook(currentComponent, index, currentHook || type);
  }
  currentHook = 0;
  // ...
  const hooks =
    currentComponent.__hooks ||
    (currentComponent.__hooks = {
      _list: [],
      _pendingEffects: [],
    });

  if (index >= hooks._list.length) {
    hooks._list.push({});
  }
  return hooks._list[index];
}
```

getHookState 每次调用都会返回对应顺序的数据。

## 11 hooks

preact 中拥有 react 的全部 hooks，共计 10 个。除此之外，还有一个`useErrorBoundary`hook。而这些 hooks 中，`useReducer`、`useEffect`、`useLayoutEffect`和`useMemo`是被复用最多的。

### useReducer

useState 是基于 useReducer 实现的，所以了解了 useReducer，也就了解了 useState。

首先在 useState 中 currentHook 设置为 1，但是 useReducer 中没有。因为它在调用 getHookState 函数时最后一个参数传入了 2。结合我们前面讲解 getHookState 时提到的，这样就能区分两个组件了。而在 getHookState 中 currentHook 又被置为 0，以保证不会污染其他 hook 调用。

在 useReducer 中，通过 getHookState 拿到当前 hook 的数据，将 reducer 函数存起来。hook 首次执行时，hookState.\_component 还为空。这时会定义 hookState.\_value，也就是我们最后返回的数组。数组第一位是当前的状态，第二个是个 dispatch 函数，用来更新状态。随后将当前组件赋值给 hookState.\_component，保证这块代码只需执行一次。除了这个原因之外，还需要利用这个组件上的 setState 来做更新操作。

如果没有 init 入参，那么 initialState 入参可能是个表示状态的对象，也可能是个函数执行后返回初始状态。如果存在 init 入参，那么 initialState 就是 init 函数的参数，执行后返回初始状态。dispatch 中执行\_reducer 函数后返回新的状态，只有和当前的状态不同，才会更新 hook 的返回值，和对应的组件。注意这里更新组件是调用的 setState，而不是 forceUpdate。传入的参数是一个空对象。这是为了配合后面的 React.memo，希望更新可以被 shouldComponent 方法拦截。而且对于函数式组件来说，它并不需要 state 属性。

```js
export function useState(initialState) {
  currentHook = 1;
  return useReducer(invokeOrReturn, initialState);
}
export function useReducer(reducer, initialState, init) {
  /** @type {import('./internal').ReducerHookState} */
  const hookState = getHookState(currentIndex++, 2);
  hookState._reducer = reducer;
  if (!hookState._component) {
    hookState._value = [
      !init ? invokeOrReturn(undefined, initialState) : init(initialState),

      (action) => {
        const nextValue = hookState._reducer(hookState._value[0], action);
        if (hookState._value[0] !== nextValue) {
          hookState._value = [nextValue, hookState._value[1]];
          hookState._component.setState({});
        }
      },
    ];

    hookState._component = currentComponent;
  }

  return hookState._value;
}
// ...
function invokeOrReturn(arg, f) {
  return typeof f == "function" ? f(arg) : f;
}
```

### useEffect useLayoutEffect

这两个 hook 几乎一样，唯一的区别就是副作用调用的时机。当首次调用或是 args 依赖发生变化时，它们就会更新存储的数据。不同点是 useEffect 将这个数据存到了`__hooks._pendingEffects`数组。而 useLayoutEffect 将数据存到前面提到的组件自身的\_renderCallbacks 数组，这个数组里的函数会在 diff 完成后立即调用。注意二者都没有将旧的数据从数组中删除，就直接存入新的。

```js
export function useEffect(callback, args) {
  /** @type {import('./internal').EffectHookState} */
  const state = getHookState(currentIndex++, 3);
  if (!options._skipEffects && argsChanged(state._args, args)) {
    state._value = callback;
    state._args = args;

    currentComponent.__hooks._pendingEffects.push(state);
  }
}

export function useLayoutEffect(callback, args) {
  /** @type {import('./internal').EffectHookState} */
  const state = getHookState(currentIndex++, 4);
  if (!options._skipEffects && argsChanged(state._args, args)) {
    state._value = callback;
    state._args = args;

    currentComponent._renderCallbacks.push(state);
  }
}
// ...
function argsChanged(oldArgs, newArgs) {
  return (
    !oldArgs ||
    oldArgs.length !== newArgs.length ||
    newArgs.some((arg, index) => arg !== oldArgs[index])
  );
}
```

useImperativeHandle hook 基于 useLayoutEffect 实现。

```js
export function useImperativeHandle(ref, createHandle, args) {
  currentHook = 6;
  useLayoutEffect(
    () => {
      if (typeof ref == "function") ref(createHandle());
      else if (ref) ref.current = createHandle();
    },
    args == null ? args : args.concat(ref)
  );
}
```

### useMemo

useMemo 的实现也很简单，当 args 依赖变化的时候，执行 factory 函数，缓存返回值，然后这个值。

```js
export function useMemo(factory, args) {
  /** @type {import('./internal').MemoHookState} */
  const state = getHookState(currentIndex++, 7);
  if (argsChanged(state._args, args)) {
    state._value = factory();
    state._args = args;
    state._factory = factory;
  }

  return state._value;
}
```

useCallback、useRef 都是基于 useMemo 实现的。

### useContext

对这个 hook 的理解必须 preact 的 diff 过程有些了解。componet 实例上的 context 属性是在 diff 的过程中添加的。globalContext 对象中存储着以 context.\_id 为 key，context.Provider 组件实例为 value 的数据。而由于函数式组件不存在 contextType 属性，所以 componentContext 也就是 context 的值为 globalContext。

```js
// diff 函数
// ...
let tmp,
  newType = newVNode.type;
// ...
// Necessary for createContext api. Setting this property will pass
// the context value as `this.context` just for this component.
tmp = newType.contextType;
let provider = tmp && globalContext[tmp._id];
let componentContext = tmp
  ? provider
    ? provider.props.value
    : tmp._defaultValue
  : globalContext;
// ...
c.context = componentContext;
```

因此在 useContext 中可以通过 context.\_id 获取到这个 context 的 Provider 组件的实例。进而可以拿到它的 props.value 数据，并接收到 value 变更的通知。

```js
export function useContext(context) {
  const provider = currentComponent.context[context._id];
  // We could skip this call here, but than we'd not call
  // `options._hook`. We need to do that in order to make
  // the devtools aware of this hook.
  /** @type {import('./internal').ContextHookState} */
  const state = getHookState(currentIndex++, 9);
  // The devtools needs access to the context object to
  // be able to pull of the default value when no provider
  // is present in the tree.
  state._context = context;
  if (!provider) return context._defaultValue;
  // This is probably not safe to convert to "!"
  if (state._value == null) {
    state._value = true;
    provider.sub(currentComponent);
  }
  return provider.props.value;
}
```

### useErrorBoundary

useErrorBoundary 是 preact 原创的 hook，react 中没有。它给组件实例添加了`componentDidCatch`方法，因此这个组件就变成了一个 ErrorBoundary。之所以可以这么做，是因为在 preact 中，类式组件和函数式组件都是通过`new`初始化的，因此都具有实例。而这个 hook 最终返回的是一个数组，但是当前的错误，和清空错误的函数。

```js
export function useErrorBoundary(cb) {
  /** @type {import('./internal').ErrorBoundaryHookState} */
  const state = getHookState(currentIndex++, 10);
  const errState = useState();
  state._value = cb;
  if (!currentComponent.componentDidCatch) {
    currentComponent.componentDidCatch = (err) => {
      if (state._value) state._value(err);
      errState[1](err);
    };
  }
  return [
    errState[0],
    () => {
      errState[1](undefined);
    },
  ];
}
```

## Execution order of plugins

为了执行`useEffect`和`useLayoutEffect`产生的副作用函数，一共实现了 5 个`事件点`函数。按照执行顺序排列为，`_diff`->`_render`->`unmount`->`diffed`->`_commit`。

最先执行的\_diff 将当前的 currentComponent 清空。

```js
let oldBeforeDiff = options._diff;
// ...
options._diff = (vnode) => {
  currentComponent = null;
  if (oldBeforeDiff) oldBeforeDiff(vnode);
};
```

\_render 会在组件的 render 方法将要执行之前执行。这时会知道当前是哪个组件将会渲染，并把 hook 的序号置为 0。

```js
let oldBeforeRender = options._render;
// ...
options._render = (vnode) => {
  if (oldBeforeRender) oldBeforeRender(vnode);

  currentComponent = vnode._component;
  currentIndex = 0;

  const hooks = currentComponent.__hooks;
  if (hooks) {
    hooks._pendingEffects.forEach(invokeCleanup);
    hooks._pendingEffects.forEach(invokeEffect);
    hooks._pendingEffects = [];
  }
};
```

如果组件不是首次渲染，那么实例上的\_\_hooks 可能不为空。这时候就将\_pendingEffects 数组中对象挨个取出执行。

我们先看 invokeEffect 函数，传入的 hook 其实是我们上面讲解 useEffect 时，push 到\_pendingEffects 数组中的 state 对象。hook.\_value 就是传给 useEffect 的 callback。执行后可能会返回一个清理函数，存在 hook.\_cleanup 中。再回到上面的 invokeCleanup 函数，这里实际上就是取出 hook.\_cleanup，如果是函数就执行。

最后整个\_pendingEffects 置空。这里有个问题是，**为什么要在 render 之前执行这两个函数？**这个问题我们留到后面解答。

```js
function invokeCleanup(hook) {
  // A hook cleanup can introduce a call to render which creates a new root, this will call options.vnode
  // and move the currentComponent away.
  const comp = currentComponent;
  let cleanup = hook._cleanup;
  if (typeof cleanup == "function") {
    hook._cleanup = undefined;
    cleanup();
  }
  currentComponent = comp;
}
function invokeEffect(hook) {
  // A hook call can introduce a call to render which creates a new root, this will call options.vnode
  // and move the currentComponent away.
  const comp = currentComponent;
  hook._cleanup = hook._value();
  currentComponent = comp;
}
```

unmount 会在组件销毁前调用，所以这里也会调用 useEffect 产生的清理函数，如果有的话。但是这里遍历的是 c.\_\_hooks.\_list，而不是上面的 hooks.\_pendingEffects。在首次渲染时，它们都会存储 state，但是当下次渲染时，如果 args 依赖不变，state 对象并不会被添加到\_pendingEffects 数组中。而且就像我们在 options.\_render 中看到的，\_pendingEffects 数组会被清空。所以这里使用了\_\_hooks.\_list，可以保证被卸载的组件中，每个 cleanup 函数都能被执行。

```js
let oldBeforeUnmount = options.unmount;
// ...
options.unmount = (vnode) => {
  if (oldBeforeUnmount) oldBeforeUnmount(vnode);

  const c = vnode._component;
  if (c && c.__hooks) {
    let hasErrored;
    c.__hooks._list.forEach((s) => {
      try {
        invokeCleanup(s);
      } catch (e) {
        hasErrored = e;
      }
    });
    if (hasErrored) options._catchError(hasErrored, c._vnode);
  }
};
```

接着是 diffed 部分，发生在组件完成 diff 后。这个时候应该是计划执行副作用的时候了。这些逻辑都在`afterPaint`函数中。

```js
let afterPaintEffects = [];
// ...
let oldAfterDiff = options.diffed;
// ...
options.diffed = (vnode) => {
  if (oldAfterDiff) oldAfterDiff(vnode);

  const c = vnode._component;
  if (c && c.__hooks && c.__hooks._pendingEffects.length) {
    afterPaint(afterPaintEffects.push(c));
  }
  currentComponent = null;
};
```

通过 afterPaint 函数中的执行条件， 可以看出它也被设计成了只能执行一次。从 afterNextFrame 函数中可以发现，这些副作用函数也是批量的异步执行。且最终执行时候是放在 setTimeout 中的，算是宏任务。前面有章我们提到的 setState 或 forceUpdate 是通过 Promise.prototype.then 触发的更新操作，算是微任务，执行的优先级更高。

```js
const RAF_TIMEOUT = 100;
let prevRaf;
// ...
function afterPaint(newQueueLength) {
  if (newQueueLength === 1 || prevRaf !== options.requestAnimationFrame) {
    prevRaf = options.requestAnimationFrame;
    (prevRaf || afterNextFrame)(flushAfterPaintEffects);
  }
}
// ...
let HAS_RAF = typeof requestAnimationFrame == "function";

function afterNextFrame(callback) {
  const done = () => {
    clearTimeout(timeout);
    if (HAS_RAF) cancelAnimationFrame(raf);
    setTimeout(callback);
  };
  const timeout = setTimeout(done, RAF_TIMEOUT);

  let raf;
  if (HAS_RAF) {
    raf = requestAnimationFrame(done);
  }
}
```

接着我们再看下`flushAfterPaintEffects`函数，在这里 afterPaintEffects 会从头开始处理。这是因为子组件会先完成 diff，触发 options.diffed，即先被 push 进 afterPaintEffects 数组。所以整个副作用函数执行的顺序是从内向外。首先判断 component.\_parentDom 是否存在，如果不存在，说明这个组件已经被卸载了。然后下面的执行就和我们在 options.\_render 中看到的一样。

```js
function flushAfterPaintEffects() {
  let component;
  while ((component = afterPaintEffects.shift())) {
    if (!component._parentDom) continue;
    try {
      component.__hooks._pendingEffects.forEach(invokeCleanup);
      component.__hooks._pendingEffects.forEach(invokeEffect);
      component.__hooks._pendingEffects = [];
    } catch (e) {
      component.__hooks._pendingEffects = [];
      options._catchError(e, component._vnode);
    }
  }
}
```

到这里，再联系一下 options.\_render 的逻辑，就可以回答上面提到的那个问题了。我们可以捋一下逻辑，

- 首次渲染前组件中没有 currentComponent.\_\_hooks，所以 options.\_render 中的那段逻辑不会执行

- 接着在 render 的过程中，useEffect 执行，currentComponent.\_\_hooks 被创建，并且\_pendingEffects 数组中被压入 state。这时 state 中只是保存了 callback 和 args，并未执行。

- 在 render 之后，diff 结束后，flushAfterPaintEffects 函数执行。这时候\_pendingEffects 数组中的对象上并没有\_cleanup 属性，所以在 invokeCleanup 函数中，没什么会发生。但是接着执行 invokeEffect 函数，代表副作用的 callback 被执行，可能返回 cleanup 函数，保存在了 state 对象中。然后\_pendingEffects 数组变为空数组。

- 如果组件再次更新，因为\_pendingEffects 已经变为空数组，所以 options.\_render 中的那段逻辑虽然会执行，但是没什么效果。

- 接着 render 过程中\_pendingEffects 数组可能还会收集到一些 state。最后在执行 flushAfterPaintEffects 函数的过程中，这些 state 如果有保存了的上次的 cleanup 函数，会被执行。然后 callback 函数再次执行，又返回新的 cleanup 函数被保存起来。

整个过程似乎没 options.\_render 中那段逻辑什么事。但是 flushAfterPaintEffects 是异步执行的。如果在它执行之前，组件又有更新需求，因为 setState 引发的更新是微任务，所以优先级更高。这个时候 diff 过程开始，options.\_render 先于 flushAfterPaintEffects 函数执行。这样就保证了 useEffect 产生的副作用能在下次 render 之前执行完。

最后是\_commit，这里会执行 useLayoutEffect 产生的副作用。因为这个 hook 的副作用执行的时机和`componetDidMount `这类的钩子一样，所以直接存在了 component.\_renderCallbacks。会在 diff 完成后同步执行，所以也不需要像\_pendingEffects 那样费事。因为是同步执行，所以在下次渲染之前肯定能完成这次渲染产生的副作用。

```js
let oldCommit = options._commit;
// ...
options._commit = (vnode, commitQueue) => {
  commitQueue.some((component) => {
    try {
      component._renderCallbacks.forEach(invokeCleanup);
      component._renderCallbacks = component._renderCallbacks.filter((cb) =>
        cb._value ? invokeEffect(cb) : true
      );
    } catch (e) {
      commitQueue.some((c) => {
        if (c._renderCallbacks) c._renderCallbacks = [];
      });
      commitQueue = [];
      options._catchError(e, component._vnode);
    }
  });

  if (oldCommit) oldCommit(vnode, commitQueue);
};
```
