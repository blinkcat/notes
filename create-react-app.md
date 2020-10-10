# create-react-app

[d5c0fe287a6bc24c60ee52ca768bdf3a10b937e1](https://github.com/facebook/create-react-app/commit/d5c0fe287a6bc24c60ee52ca768bdf3a10b937e1)

官方维护的 react 脚手架，没看源码之前就觉得灵活性很差，现在开始慢慢支持自定义模板了，但是修改 webpack 配置还是很难。总得来说，源码中还是有不少值得学习的地方。

## Supported Browsers and Features

单独抽出了一个 package [react-app-polyfill](https://github.com/facebook/create-react-app/blob/master/packages/react-app-polyfill/README.md)来维护 polyfill。脚手架本身没有包含 polyfill，需要在 package.json 中手动配置[browserslist](https://create-react-app.dev/docs/supported-browsers-features/#configuring-supported-browsers)，然后根据是否支持 IE，引入对应的包。

> If you are supporting Internet Explorer 9 or Internet Explorer 11 you should include both the ie9 or ie11 and stable modules:

```js
// These must be the first lines in src/index.js
// For IE9:
// import "react-app-polyfill/ie9";
// For IE11:
import "react-app-polyfill/ie11";
import "react-app-polyfill/stable";
```

除此之外，还单独抽出了一个 [babel-preset-react-app](https://github.com/facebook/create-react-app/tree/master/packages/babel-preset-react-app)，里面包含了 [@babel/preset-env](https://www.babeljs.cn/docs/babel-preset-env)。搭配上这个 babel 插件，就可以配合上面说的 browserslist，自动决定要适配的浏览器环境需要哪些方法转换或是 polyfill。

## references

1. [@babel/plugin-transform-runtime 到底是什么？](https://zhuanlan.zhihu.com/p/147083132)
