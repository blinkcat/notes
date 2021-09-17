# loadable-components

版本[v5.15.1](https://github.com/gregberge/loadable-components/tree/v5.15.1)

另一个实现动态加载(dynamic import)的库。不仅可以加载组件，还可以加载 lib。

## Usage

加载组件

```js
import loadable from "@loadable/component";
const OtherComponent = loadable(() => import("./OtherComponent"));
function MyComponent() {
  return (
    <div>
      <OtherComponent />
    </div>
  );
}
```

加载 lib

```js
import loadable from "@loadable/component";
const Moment = loadable.lib(() => import("moment"));
function FromNow({ date }) {
  return (
    <div>
      <Moment fallback={date.toLocaleDateString()}>
        {({ default: moment }) => moment(date).fromNow()}
      </Moment>
    </div>
  );
}
```

甚至是更动态的

```js
import loadable from "@loadable/component";
const AsyncPage = loadable((props) => import(`./${props.page}`));
function MyComponent() {
  return (
    <div>
      <AsyncPage page="Home" />
      <AsyncPage page="Contact" />
    </div>
  );
}
```

当然也是支持服务端渲染的。

## How it works
