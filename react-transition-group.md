# react-transition-group

版本[v4.4.2](https://github.com/reactjs/react-transition-group/tree/v4.4.2)

用来实现简单的转场动画。

## How it works

react-transition-group 导出了 Transition、CSSTransition、TransitionGroup 和 SwitchTransition 等组件。从名称可以看出，这些组件都是以 Transition 组件为基础，添加新的功能。所以我们需要先从 Transition 组件开始看起。

### Transition

Transition 组件是一个平台无关的基础组件，它不会直接去控制子组件的状态和行为。而是在不同时间提供给子组件不同的状态值，由子组件来决定在不同的状态下，应该产生的效果。

如下所示，当 Transition 组件的 in 属性为 true 时，state 先转移到 entering 状态，然后停留 duration 时间，再转移到 entered 状态。当 in 属性为 false 的时候，又会转移到 exiting 状态，同样停留 duration 的时间后转移到 exited 状态。

```js
import { Transition } from "react-transition-group";

const duration = 300;

const defaultStyle = {
  transition: `opacity ${duration}ms ease-in-out`,
  opacity: 0,
};

const transitionStyles = {
  entering: { opacity: 1 },
  entered: { opacity: 1 },
  exiting: { opacity: 0 },
  exited: { opacity: 0 },
};

const Fade = ({ in: inProp }) => (
  <Transition in={inProp} timeout={duration}>
    {(state) => (
      <div
        style={{
          ...defaultStyle,
          ...transitionStyles[state],
        }}
      >
        I'm a fade Transition!
      </div>
    )}
  </Transition>
);
```

除了上面提到的四种状态外，Transition 还有一种 unmounted 状态，表示在 transition 开始前和结束后，是否需要渲染子组件。

因此，Transition 组件的主要逻辑就是决定何时在这几种状态之间切换。in 属性表示是否显示组件，改变传入的值就会触发不同的状态。appear 属性表示子组件首次挂载时，是否显示进入状态。再加上 其他几个属性，以及父级组件的 context 中的属性，共同决定了 Transition 组件的初始状态。这个 context 一般是由 TransitionGroup 组件提供的，用来协调一组 Transition 组件的状态切换。它的 isMounting 属性表示父级 TransitionGroup 组件是否正在加载，在父组件 didMount 以后，isMounting 属性为 false。因此 appear 属性只会在没有父级 TransitionGroup 组件，或者父级组件加载完成后才起作用。

```js
class Transition extends React.Component {
  static contextType = TransitionGroupContext;

  constructor(props, context) {
    super(props, context);

    let parentGroup = context;
    // In the context of a TransitionGroup all enters are really appears
    let appear =
      parentGroup && !parentGroup.isMounting ? props.enter : props.appear;

    let initialStatus;

    this.appearStatus = null;

    if (props.in) {
      if (appear) {
        initialStatus = EXITED;
        this.appearStatus = ENTERING;
      } else {
        initialStatus = ENTERED;
      }
    } else {
      if (props.unmountOnExit || props.mountOnEnter) {
        initialStatus = UNMOUNTED;
      } else {
        initialStatus = EXITED;
      }
    }

    this.state = { status: initialStatus };

    this.nextCallback = null;
  }
  // ...
}
```

当 in 和 appear 都为 true 时，初始状态是 EXITED，接下来它会进入 ENTERING 状态，然后停在 ENTERED 状态。这表现为子组件在首次加载时有一个 appear 的过程。否则初始状态是 ENTERED，表示子组件已经显示完毕。

当 in 为 false 的时候，如果要求子组件不立即加载或是当状态转移到 EXITED 时销毁子组件，初始状态为 UNMOUNTED，这时候不会渲染子组件。其他情况下，初始状态为 EXITED。到这里我们在两种情况下虽然都是 EXITED，但是上面的会将 appearStatus 设置为 ENTERING 作为区分。

在 componentDidMount 钩子中，根据 appearStatus 的不同。上面的 EXITED 状态会开始进入 ENTERING 状态，而下面继续保持 EXITED 状态。

```js
class Transition extends React.Component {
  // ...
  componentDidMount() {
    this.updateStatus(true, this.appearStatus);
  }
  // ...
}
```

而当 in 属性变化时，如果当前的状态还是 UNMOUNTED，就将状态转为 EXITED，使得子组件可以渲染。

```js
class Transition extends React.Component {
  // ...
  static getDerivedStateFromProps({ in: nextIn }, prevState) {
    if (nextIn && prevState.status === UNMOUNTED) {
      return { status: EXITED };
    }
    return null;
  }
  // ...
}
```

在 render 方法中，除了 UNMOUNTED 状态外，其他状态下都能正常渲染子组件。

```js
class Transition extends React.Component {
  // ...
  render() {
    const status = this.state.status;

    if (status === UNMOUNTED) {
      return null;
    }
    // ...
    return (
      // allows for nested Transitions
      <TransitionGroupContext.Provider value={null}>
        {typeof children === "function"
          ? children(status, childProps)
          : React.cloneElement(React.Children.only(children), childProps)}
      </TransitionGroupContext.Provider>
    );
  }
  // ...
}
```

而状态转换的逻辑都在 componentDidUpdate 这个钩子中，在这里决定了 nextStatus 的状态。然后调用 updateStatus 方法进行转换。

```js
class Transition extends React.Component {
  // ...
  componentDidUpdate(prevProps) {
    let nextStatus = null;
    if (prevProps !== this.props) {
      const { status } = this.state;

      if (this.props.in) {
        if (status !== ENTERING && status !== ENTERED) {
          nextStatus = ENTERING;
        }
      } else {
        if (status === ENTERING || status === ENTERED) {
          nextStatus = EXITING;
        }
      }
    }
    this.updateStatus(false, nextStatus);
  }
  // ...
}
```

在 updateStatus 方法内部，会根据 nextStatus 的值执行 enter 系列状态转换或是 exit 系列状态转换，又或者是从 EXITED 状态切换到 UNMOUNTED 状态。

```js
class Transition extends React.Component {
  // ...
  updateStatus(mounting = false, nextStatus) {
    if (nextStatus !== null) {
      // nextStatus will always be ENTERING or EXITING.
      this.cancelNextCallback();

      if (nextStatus === ENTERING) {
        this.performEnter(mounting);
      } else {
        this.performExit();
      }
    } else if (this.props.unmountOnExit && this.state.status === EXITED) {
      this.setState({ status: UNMOUNTED });
    }
  }
  // ...
}
```

在 performEnter 中，会进行从 ENTERING 状态到 ENTERED 状态的转换过程，在这个过程中会调用 onEnter、onEntering、onEntered 回调。这些回调的参数中有一个 maybeAppearing 参数，表示当前的这次回调是否属于首次。

```js
class Transition extends React.Component {
  // ...
  performEnter(mounting) {
    const { enter } = this.props;
    const appearing = this.context ? this.context.isMounting : mounting;
    const [maybeNode, maybeAppearing] = this.props.nodeRef
      ? [appearing]
      : [ReactDOM.findDOMNode(this), appearing];

    const timeouts = this.getTimeouts();
    const enterTimeout = appearing ? timeouts.appear : timeouts.enter;
    // no enter animation skip right to ENTERED
    // if we are mounting and running this it means appear _must_ be set
    if ((!mounting && !enter) || config.disabled) {
      this.safeSetState({ status: ENTERED }, () => {
        this.props.onEntered(maybeNode);
      });
      return;
    }

    this.props.onEnter(maybeNode, maybeAppearing);

    this.safeSetState({ status: ENTERING }, () => {
      this.props.onEntering(maybeNode, maybeAppearing);

      this.onTransitionEnd(enterTimeout, () => {
        this.safeSetState({ status: ENTERED }, () => {
          this.props.onEntered(maybeNode, maybeAppearing);
        });
      });
    });
  }
  // ...
}
```

performExit 也是类似，只不过它执行的是 onExit、onExiting、onExited 回调。

### CSSTransition

Transition 为了保持其平台无关性，只提供几种状态给子组件。CSSTransition 在它的基础上将各种状态转换为 classname，不同状态的切换就是不同的 classname 的切换。

从 render 方法中可以看到其实现方式就是通过这些 Transition 组件提供的回调函数来适时添加移除 classname。

```js
class CSSTransition extends React.Component {
  // ...
  onEnter = (maybeNode, maybeAppearing) => {
    const [node, appearing] = this.resolveArguments(maybeNode, maybeAppearing);
    this.removeClasses(node, "exit");
    this.addClass(node, appearing ? "appear" : "enter", "base");

    if (this.props.onEnter) {
      this.props.onEnter(maybeNode, maybeAppearing);
    }
  };
  // ...
  render() {
    const { classNames: _, ...props } = this.props;

    return (
      <Transition
        {...props}
        onEnter={this.onEnter}
        onEntered={this.onEntered}
        onEntering={this.onEntering}
        onExit={this.onExit}
        onExiting={this.onExiting}
        onExited={this.onExited}
      />
    );
  }
}
```

### TransitionGroup

TransitionGroup 是一个 smart container，负责统筹协调一组 Transition 或者 CSSTransition 组件。

我们先看 render 方法，其中的 children 并非从 props 取得，而是从 state 上得来的。这说明 TransitionGroup 会对它的子组件进行一些处理。

```js
class TransitionGroup extends React.Component {
  // ...
  render() {
    const { component: Component, childFactory, ...props } = this.props;
    const { contextValue } = this.state;
    const children = values(this.state.children).map(childFactory);

    delete props.appear;
    delete props.enter;
    delete props.exit;

    if (Component === null) {
      return (
        <TransitionGroupContext.Provider value={contextValue}>
          {children}
        </TransitionGroupContext.Provider>
      );
    }
    return (
      <TransitionGroupContext.Provider value={contextValue}>
        <Component {...props}>{children}</Component>
      </TransitionGroupContext.Provider>
    );
  }
}
```

这部分的逻辑在 getDerivedStateFromProps 钩子中。

```js
class TransitionGroup extends React.Component {
  // ...
  static getDerivedStateFromProps(
    nextProps,
    { children: prevChildMapping, handleExited, firstRender }
  ) {
    return {
      children: firstRender
        ? getInitialChildMapping(nextProps, handleExited)
        : getNextChildMapping(nextProps, prevChildMapping, handleExited),
      firstRender: false,
    };
  }
  // ...
}
```

在首次渲染的时候，调用 getInitialChildMapping 处理 props.children。然后返回以元素的 key 为键值，处理过的子元素为值的对象赋给 state.children。

```js
export function getChildMapping(children, mapFn) {
  let mapper = (child) =>
    mapFn && isValidElement(child) ? mapFn(child) : child;

  let result = Object.create(null);
  if (children)
    Children.map(children, (c) => c).forEach((child) => {
      // run the map function here instead so that the key is the computed one
      result[child.key] = mapper(child);
    });
  return result;
}
// ...
export function getInitialChildMapping(props, onExited) {
  return getChildMapping(props.children, (child) => {
    return cloneElement(child, {
      onExited: onExited.bind(null, child),
      in: true,
      appear: getProp(child, "appear", props),
      enter: getProp(child, "enter", props),
      exit: getProp(child, "exit", props),
    });
  });
}
```

接下来的渲染，需要通过新的 props.children 和之前的 state.children 做合并操作。针对这个合并后的结果，区分出准备离开的，和新进入的，分别对其的 in 属性设置成对应的布尔值。最后将合并后的结果赋值给 state.children。

```js
export function getNextChildMapping(nextProps, prevChildMapping, onExited) {
  let nextChildMapping = getChildMapping(nextProps.children);
  let children = mergeChildMappings(prevChildMapping, nextChildMapping);

  Object.keys(children).forEach((key) => {
    let child = children[key];

    if (!isValidElement(child)) return;

    const hasPrev = key in prevChildMapping;
    const hasNext = key in nextChildMapping;

    const prevChild = prevChildMapping[key];
    const isLeaving = isValidElement(prevChild) && !prevChild.props.in;

    // item is new (entering)
    if (hasNext && (!hasPrev || isLeaving)) {
      // console.log('entering', key)
      children[key] = cloneElement(child, {
        onExited: onExited.bind(null, child),
        in: true,
        exit: getProp(child, "exit", nextProps),
        enter: getProp(child, "enter", nextProps),
      });
    } else if (!hasNext && hasPrev && !isLeaving) {
      // item is old (exiting)
      // console.log('leaving', key)
      children[key] = cloneElement(child, { in: false });
    } else if (hasNext && hasPrev && isValidElement(prevChild)) {
      // item hasn't changed transition states
      // copy over the last transition props;
      // console.log('unchanged', key)
      children[key] = cloneElement(child, {
        onExited: onExited.bind(null, child),
        in: prevChild.props.in,
        exit: getProp(child, "exit", nextProps),
        enter: getProp(child, "enter", nextProps),
      });
    }
  });

  return children;
}
```

这里的逻辑还有两个细节，首先是合并算法，代码看起来不少，但是逻辑很简单。首先子元素是通过 key 值比较的，有相同的 key 值就是相同的元素。这也是使用 TransitionGroup 的前提。

```js
export function mergeChildMappings(prev, next) {
  prev = prev || {};
  next = next || {};

  function getValueForKey(key) {
    return key in next ? next[key] : prev[key];
  }

  // For each key of `next`, the list of keys to insert before that key in
  // the combined list
  let nextKeysPending = Object.create(null);

  let pendingKeys = [];
  for (let prevKey in prev) {
    if (prevKey in next) {
      if (pendingKeys.length) {
        nextKeysPending[prevKey] = pendingKeys;
        pendingKeys = [];
      }
    } else {
      pendingKeys.push(prevKey);
    }
  }

  let i;
  let childMapping = {};
  for (let nextKey in next) {
    if (nextKeysPending[nextKey]) {
      for (i = 0; i < nextKeysPending[nextKey].length; i++) {
        let pendingNextKey = nextKeysPending[nextKey][i];
        childMapping[nextKeysPending[nextKey][i]] =
          getValueForKey(pendingNextKey);
      }
    }
    childMapping[nextKey] = getValueForKey(nextKey);
  }

  // Finally, add the keys which didn't appear before any key in `next`
  for (i = 0; i < pendingKeys.length; i++) {
    childMapping[pendingKeys[i]] = getValueForKey(pendingKeys[i]);
  }

  return childMapping;
}
```

配合下面两个例子，我们可以了解这个合并算法。首先遍历旧的集合，如果当前的 key 不在新的集合中，就压入 pendingKeys 数组。如果在的话，就将 pendingKeys 数组赋值给以这个 key 为键值的 nextKeysPending 对象，接着清空 pendingKeys 数组。然后开始遍历新的集合，如果当前 key 存在于 nextKeysPending 对象中，就先将这个对象的 key 值存储的数组里的 key 依次放入最终的结果中，接着放入当前 key。最后如果 pendingKeys 数组中还有值，也依次放入最终结果中。

```sh
# 1
# prev:
a b c d
# next:
b c d e
# merged:
a b c d e

# 2
# prev:
a b c d
# next:
b e f g
# merged:
a b e f g c d
```

通过前面的代码我们可以看到，每个被处理过的元素都通过 onExited 绑定了 handleExited 方法，

```js
class TransitionGroup extends React.Component {
  // ...
  handleExited(child, node) {
    let currentChildMapping = getChildMapping(this.props.children);

    if (child.key in currentChildMapping) return;

    if (child.props.onExited) {
      child.props.onExited(node);
    }

    if (this.mounted) {
      this.setState((state) => {
        let children = { ...state.children };

        delete children[child.key];
        return { children };
      });
    }
  }
  // ...
}
```

状态转换到 exited 的元素如果不在当前的 props.children 中，就会被删除。

### SwitchTransition

SwitchTransition 用于两个子元素切换，有两种模式。

- out-in 旧的先走，新的再进来
- in-out 新的先进来，旧的再走

```js
function App() {
  const [state, setState] = useState(false);
  return (
    <SwitchTransition>
      <CSSTransition
        key={state ? "Goodbye, world!" : "Hello, world!"}
        addEndListener={(node, done) =>
          node.addEventListener("transitionend", done, false)
        }
        classNames="fade"
      >
        <button onClick={() => setState((state) => !state)}>
          {state ? "Goodbye, world!" : "Hello, world!"}
        </button>
      </CSSTransition>
    </SwitchTransition>
  );
}
```

SwitchTransition 初始状态是 ENTERED，current 表示当前渲染的子元素。appeared 看起来是表示当前是否在加载中。

```js
class SwitchTransition extends React.Component {
  state = {
    status: ENTERED,
    current: null,
  };

  appeared = false;

  componentDidMount() {
    this.appeared = true;
  }
  // ...
}
```

接着又来到了 getDerivedStateFromProps 钩子，首次渲染时 state.current 被赋值为 props.children。同时 in 属性设置为 true。当子元素变化时，也就是子元素的 key 变化时，状态变为 EXITING。

```js
class SwitchTransition extends React.Component {
  // ...
  static getDerivedStateFromProps(props, state) {
    if (props.children == null) {
      return {
        current: null,
      };
    }

    if (state.status === ENTERING && props.mode === modes.in) {
      return {
        status: ENTERING,
      };
    }

    if (state.current && areChildrenDifferent(state.current, props.children)) {
      return {
        status: EXITING,
      };
    }

    return {
      current: React.cloneElement(props.children, {
        in: true,
      }),
    };
  }
  // ...
}
```

这里最好配合 render 方法的代码一起来看，首次渲染的时候，状态是 ENTERED，这时渲染的子元素就是 current。而当子元素变化时，状态变为 EXITING，这时候会调用 leaveRenders 中的函数来决定接下来渲染的元素。

```js
class SwitchTransition extends React.Component {
  // ...
  render() {
    const {
      props: { children, mode },
      state: { status, current },
    } = this;

    const data = { children, current, changeState: this.changeState, status };
    let component;
    switch (status) {
      case ENTERING:
        component = enterRenders[mode](data);
        break;
      case EXITING:
        component = leaveRenders[mode](data);
        break;
      case ENTERED:
        component = current;
    }

    return (
      <TransitionGroupContext.Provider value={{ isMounting: !this.appeared }}>
        {component}
      </TransitionGroupContext.Provider>
    );
  }
}
```

leaveRenders 中有两种模式对应的函数，modes.out 表示当前的元素 current 先离开，modes.in 表示新的元素先进来，旧的元素先不动。在完成状态转换后，都会将当前的状态转变为 ENTERING。modes.out 还会将 state.current 变成 null。

```js
const leaveRenders = {
  [modes.out]: ({ current, changeState }) =>
    React.cloneElement(current, {
      in: false,
      onExited: callHook(current, "onExited", () => {
        changeState(ENTERING, null);
      }),
    }),
  [modes.in]: ({ current, changeState, children }) => [
    current,
    React.cloneElement(children, {
      in: true,
      onEntered: callHook(children, "onEntered", () => {
        changeState(ENTERING);
      }),
    }),
  ],
};
```

接着在进入 render 方法之前，触发 getDerivedStateFromProps 钩子，在 render 中优惠调用 enterRenders 计算接下来渲染的元素。

```js
const enterRenders = {
  [modes.out]: ({ children, changeState }) =>
    React.cloneElement(children, {
      in: true,
      onEntered: callHook(children, "onEntered", () => {
        changeState(ENTERED, React.cloneElement(children, { in: true }));
      }),
    }),
  [modes.in]: ({ current, children, changeState }) => [
    React.cloneElement(current, {
      in: false,
      onExited: callHook(current, "onExited", () => {
        changeState(ENTERED, React.cloneElement(children, { in: true }));
      }),
    }),
    React.cloneElement(children, {
      in: true,
    }),
  ],
};
```

这里的 modes.out 表示新的元素进来，然后将状态改为 ENTERED，并将 state.current 赋值为新的元素。modes.in 表示让当前的元素离开，新的不动，并且最终状态变为 ENTERED，state.current 变成新的元素。

## Conclusion

react-transition-group 有两点非常值得学习，第一点是它只提供状态变化保证了平台无关性，同时暴露了足够的属性和回调来支持进一步的封装。第二点是在协调多个 transition 的时候，利用了 key 值来识别子元素，获取前后两次的 children 来控制元素离开和进入。
