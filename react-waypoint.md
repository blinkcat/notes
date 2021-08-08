# React Waypoint

[source](https://github.com/civiccc/react-waypoint/tree/v10.1.0)

用来判断是否滚动到了一个元素，可以用来实现懒加载，无限滚动加载等等功能。

## How It Works

首先聚焦问题，我们只是想判断一个元素是否被滚动到位，而不是判断这个元素真的是否可见，这样可以简化问题。那么正常的思路就是监听滚动容器的 scroll 事件。

先看使用方法

```js
import { Waypoint } from "react-waypoint";

<Waypoint
  onEnter={this._handleWaypointEnter}
  onLeave={this._handleWaypointLeave}
/>;
```

或者可以包裹一个元素

```js
<Waypoint onEnter={this._handleEnter}>
  <div>Some content here</div>
</Waypoint>
```

然后，将`Waypoint`组件放入到一个滚动容器内，并不一定要是直接的父元素。

> React Waypoint positions its boundaries based on the first scrollable ancestor of the Waypoint.

组件会自行寻找满足条件的可滚动元素，当然也可以使用传入的`scrollableAncestor`属性对应的元素。

先看`render()`部分

```js
  render() {
    const { children } = this.props;

    if (!children) {
      // We need an element that we can locate in the DOM to determine where it is
      // rendered relative to the top of its context.
      return <span ref={this.refElement} style={{ fontSize: 0 }} />;
    }

    if (isDOMElement(children) || isForwardRef(children)) {
      const ref = (node) => {
        this.refElement(node);
        if (children.ref) {
          if (typeof children.ref === 'function') {
            children.ref(node);
          } else {
            children.ref.current = node;
          }
        }
      };

      return React.cloneElement(children, { ref });
    }

    return React.cloneElement(children, { innerRef: this.refElement });
  }
```

如果没有`children`，会渲染一个`span`来确定位置。

`this.refElement`可以在构造函数中看到，

```js
  constructor(props) {
    super(props);

    this.refElement = (e) => {
      this._ref = e;
    };
  }
```

目的是获取 DOM 元素，留着后面计算位置用。

如果有`children`，那么就要设法获取`children`的 DOM 元素。这里会分两种情况，一种是`children`本身是 DOM 元素，如 div，可以直接传入`ref`获取。或者是被`React.forwardRef`包装过的`react element`，也可以通过直接传入`ref`获取。但是要记得，如果`children`上已经存在了`ref`属性，这时要先保存旧的，再传入新的，两者一同处理。

再来看`componentDidMount()`，只贴重要的逻辑，

```js
  componentDidMount() {
	//...
	const { children, debug } = this.props;

	// Berofe doing anything, we want to check that this._ref is avaliable in Waypoint
	ensureRefIsUsedByChild(children, this._ref);

	this._handleScroll = this._handleScroll.bind(this);
	this.scrollableAncestor = this._findScrollableAncestor();

	if (process.env.NODE_ENV !== "production" && debug) {
	debugLog("scrollableAncestor", this.scrollableAncestor);
	}

	this.scrollEventListenerUnsubscribe = addEventListener(
	this.scrollableAncestor,
	"scroll",
	this._handleScroll,
	{ passive: true }
	);

	this.resizeEventListenerUnsubscribe = addEventListener(
	window,
	"resize",
	this._handleScroll,
	{ passive: true }
	);

	this._handleScroll(null);
	// ...
}
```

这里主要干了三件件事情，

1. 通过`this._findScrollableAncestor()`找到支持滚动的父元素
2. 在`this.scrollableAncestor`上绑`scroll`事件，在`window`上绑`resize`事件
3. 通过`this._handleScroll(null)`触发一次绑定函数

好的，看到`_handleScroll`，就能发现整个功能的主要逻辑都在这里。

```js
  _handleScroll(event) {
    if (!this._ref) {
      // There's a chance we end up here after the component has been unmounted.
      return;
    }

    const bounds = this._getBounds();
    const currentPosition = getCurrentPosition(bounds);
	const previousPosition = this._previousPosition;
	// ...
	if (currentPosition === INSIDE) {
      onEnter.call(this, callbackArg);
    } else if (previousPosition === INSIDE) {
      onLeave.call(this, callbackArg);
    }
	// ...
}
```

首先调用`_getBounds`，得到定位元素的上下边界，以及容器的上下边界。默认是垂直方向，如果是水平方向，那就是左右边界。然后，调用`getCurrentPosition`，计算得到定位元素处于容器中的位置，上，中，下等。接着和上次的位置做对比，判断位置是否有变化，接着触发`onEnter`或是`onLeave`回调。

整个组件的功能逻辑大致如此。
