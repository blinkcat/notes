# babel-plugin-transform-define

版本 [v2.0.0](https://github.com/FormidableLabs/babel-plugin-transform-define/tree/v2.0.0)

功能和 wepback 的[DefinePlugin](https://webpack.js.org/plugins/define-plugin/)类似。可以对代码中的变量、成员访问表达式、typeof 表达式进行替换。

```js
{
  "plugins": [
    ["transform-define", {
      "process.env.NODE_ENV": "production",
      "typeof window": "object",
	  "VERSION": "1.0.0"
    }]
  ]
}

// source code
// 1
if (process.env.NODE_ENV === "production") {
  console.log(true);
}

// 2
VERSION;

window.__MY_COMPANY__ = {
  version: VERSION
};

// 3
typeof window;
typeof window === "object";

// output
// 1
if (true) {
  console.log(true);
}

// 2
"1.0.0";

window.__MY_COMPANY__ = {
  version: "1.0.0"
};

// 3
'object';
true;
```

有三种节点类型需要处理，`MemberExpression`、`Identifier`、`UnaryExpression`分别对应于上面例子中的三种情况。

首先是要寻找到符合条件的节点。

```js
// process.env.NODE_ENV;
MemberExpression(nodePath, state) {
	processNode(state.opts, nodePath, t.valueToNode, memberExpressionComparator);
},
```

通过`state.opts`获取用户的配置参数，传入`processNode`。在这个函数中，会继续调用`getSortedObjectPaths`。

```js
const getSortedObjectPaths = (obj) => {
  if (!obj) {
    return [];
  }

  return traverse(obj)
    .paths()
    .filter((arr) => arr.length)
    .map((arr) => arr.join("."))
    .sort((a, b) => b.length - a.length);
};
```

这个函数通过`traverse`这个库来获取一个`object`的所有可访问路径，然后排序。比如说，

```js
getSortedObjectPaths({ process: { env: { NODE_ENV: "development" } } });

// output
[ "process.env.NODE_ENV", "process.env" "process" ]
```

这个结果将会辅助我们查找到那些需要替换内容。

接着看`processNode`函数，

```js
const replacementKey = find(getSortedObjectPaths(replacements), (value) =>
  comparator(nodePath, value)
);
```

这里的`replacements`是用户传入的参数，而`find`函数是`lodash`中的方法。

> _.find(collection, [predicate=_.identity], [fromIndex=0])
>
> 遍历 collection（集合）元素，返回 predicate（断言函数）第一个返回真值的第一个元素。predicate（断言函数）调用 3 个参数： (value, index|key, collection)。

主要的查找逻辑就在`comparator`函数里，三种类型的节点有不同的比较方法，

```js
const memberExpressionComparator = (nodePath, value) =>
  nodePath.matchesPattern(value);
const identifierComparator = (nodePath, value) => nodePath.node.name === value;
const unaryExpressionComparator = (nodePath, value) =>
  nodePath.node.argument.name === value;
```

基本上都是直接利用`babel`提供的 api。

再回到`processNode`函数，

```js
if (has(replacements, replacementKey)) {
  replaceAndEvaluateNode(
    replaceFn,
    nodePath,
    get(replacements, replacementKey)
  );
}
```

如果找到，就会调用`replaceAndEvaluateNode`函数进行替换。

```js
const replaceAndEvaluateNode = (replaceFn, nodePath, replacement) => {
  nodePath.replaceWith(replaceFn(replacement));

  if (nodePath.parentPath.isBinaryExpression()) {
    const result = nodePath.parentPath.evaluate();

    if (result.confident) {
      nodePath.parentPath.replaceWith(replaceFn(result.value));
    }
  }
};
```

`replaceFn`直接使用了 babel/types 中的`valueToNode`方法。如果发现`parentPath`是二元表达式，会尝试进行计算，主要是处理下面的情况。

```js
// source code
if (process.env.NODE_ENV === "production") {
  console.log(true);
}
// output code
if (true) {
  console.log(true);
}
```

以上就是整个处理流程。
