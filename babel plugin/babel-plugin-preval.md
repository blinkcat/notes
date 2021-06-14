# babel-plugin-preval

版本 [v5.0.0](https://github.com/kentcdodds/babel-plugin-preval/tree/v5.0.0)

## Usage

[用法](https://github.com/kentcdodds/babel-plugin-preval/tree/v5.0.0#usage)主要有 4 种，

### Template Tag

**before:**

```js
const name = "Bob Hope";
const person = preval`
  const [first, last] = require('./name-splitter')(${name})
  module.exports = {first, last}
`;
```

**after:**

```js
const name = "Bob Hope";
const person = { first: "Bob", last: "Hope" };
```

### import comment

**before:**

```js
import fileList from /* preval */ "./get-list-of-files";
```

**after:**

```js
const fileList = ["file1.md", "file2.md", "file3.md", "file4.md"];
```

### preval.require

**before:**

```js
const fileLastModifiedDate = preval.require("./get-last-modified-date");
```

**after:**

```js
const fileLastModifiedDate = "2017-07-05";
```

## preval file comment (// @preval)

**before**

```js
// @preval

const id = require("./path/identity");
const one = require("./path/one");

const compose = (...fns) => fns.reduce((f, g) => (a) => f(g(a)));
const double = (a) => a * 2;
const square = (a) => a * a;
```

**After:**

```js
module.exports = compose(square, id, double)(one);
```

## How it works

针对四种用法，分别需要处理四种节点

- `Program`
- `TaggedTemplateExpression`
- `ImportDeclaration`
- `CallExpression`

主要的思路是先获取需要预先执行的脚步，通过特殊方式执行，将得到的结果替换原来的节点。

先来看如何处理`Program`，也就是 `preval file comment (// @preval)`这种使用方式。

```js
Program(path, {file: {opts: fileOpts}}) {
const firstNode = path.node.body[0] || {}
const comments = firstNode.leadingComments || []
const isPreval = comments.some(isPrevalComment)

if (!isPreval) {
	return
}

comments.find(isPrevalComment).value = ' this file was prevaled'

const {code: string} = transformFromAst(
	path.node,
	null,
	/* istanbul ignore next (babel 6 vs babel 7 check) */
	/^6\./.test(babel.version)
	? {}
	: {
		babelrc: false,
		configFile: false,
		},
)
const replacement = getReplacement({string, fileOpts, babel})

const moduleExports = {
	...t.expressionStatement(
	t.assignmentExpression(
		'=',
		t.memberExpression(
		t.identifier('module'),
		t.identifier('exports'),
		),
		replacement,
	),
	),
	leadingComments: comments,
}

path.replaceWith(t.program([moduleExports]))
}
```

稍微有点长，但是在知晓了思路后，也并不难理解。

首先是去获取顶部的注释，得到一个注释数组`comments`。在这些注释中查找`// @preval`注释。比对的方法是，

```js
function isPrevalComment(comment) {
  const normalisedComment = comment.value.trim().split(" ")[0].trim();
  return (
    normalisedComment.startsWith("preval") ||
    normalisedComment.startsWith("@preval")
  );
}
```

很好理解，不赘述了。

如果存在这种注释，表明这个文件需要 plugin 来处理。接着通过 babel 中的`transformFromAst`方法，将`ast`转回`code`，这里是为了接下来可以执行这段`code`。于是调用`getReplacement`方法执行，

```js
function getReplacement({ string, fileOpts, args, babel }) {
  const mod = requireFromString({ string, fileOpts, args, babel });
  return objectToAST(mod, { babel, fileOptions: fileOpts });
}
```

在这个函数中，首先调用`requireFromString`函数执行代码，

```js
function requireFromString({
  string: stringToPreval,
  fileOpts: { filename },
  args = [],
}) {
  let mod = requireFromStringOfCode(String(stringToPreval), filename);
  mod = mod && mod.__esModule ? mod.default : mod;

  if (typeof mod === "function") {
    mod = mod(...args);
  } else if (args.length) {
    throw new Error(
      `\`preval.require\`-ed module (${p.relative(
        process.cwd(),
        filename
      )}) cannot accept arguments because it does not export a function. You passed the arguments: ${args.join(
        ", "
      )}`
    );
  }

  return mod;
}
```

最核心的是调用`requireFromStringOfCode`函数，这个函数来自于[require-from-string](https://github.com/floatdrop/require-from-string)。可以将`string`形式的代码，转成`module`返回。

在得到执行后的结果后，还会调用`objectToAST`函数，将结果转成`ast`再返回。

```js
function objectToAST(object, { babel, fileOptions }) {
  const stringified = stringify(object);
  const variableDeclarationNode = babel.template(`var x = ${stringified}`, {
    preserveComments: true,
    placeholderPattern: false,
    ...fileOptions.parserOpts,
    sourceType: "module",
  })();
  return variableDeclarationNode.declarations[0].init;
}
```

首先会将对象序列化，这里要注意对象中如果存在函数，需要特殊处理。

```js
function stringify(object) {
  let str = JSON.stringify(object, stringifyReplacer);
  if (str === undefined) {
    str = "undefined";
  }
  return str.replace(
    /"__FUNCTION_START__(.*?)__FUNCTION_END__"/g,
    functionReplacer
  );
  function stringifyReplacer(key, value) {
    if (typeof value === "function") {
      return `__FUNCTION_START__${value.toString()}__FUNCTION_END__`;
    }
    return value;
  }
  function functionReplacer(match, p1) {
    return p1.replace(/\\"/g, '"').replace(/\\n/g, "\n");
  }
}
```

在调用原生的`JSON.stringify`时，会多传递一个`stringifyReplacer`函数来特别处理函数。具体的处理方式是，调用函数自身的`toString`方法，并在包在两段特殊的标记中。这是为了待会可以在序列化的结果中，继续处理函数字符串。这里对字符串中的函数部分进行了两个操作，

1. 去除掉外围的特殊标记，包括`"`
2. 将其中的`\"`，`\n`替换成真正表示转义的`\"`，`\n`

再回到`objectToAST`函数，在拿到序列化的对象后，调用 babel 的`template`方法，来将这个序列化后的对象转成对应的`ast`，并返回。

`getReplacement`方法返回了我们需要的`ast`，因此，只要稍加处理，变成一个`module.exports=replacement`形式的表达式，再调用`path.replaceWith`就可以替换原来的结构。

另外三种使用方式原理类似，不再赘述。
