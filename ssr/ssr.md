# 如何实现 SSR

首先一个正经的服务端渲染(SSR)是将同构的代码在 node 端运行后生成 html 后返回给客户端。而不是说将客户端代码在 puppeteer 这类的 headless browser 中运行后再将页面的 html 复制出来。

这里我们先区分开发模式和 build 模式。

## 开发模式

如果是在开发模式中，我们需要同时使用两个 webpack compiler，一个负责编译客户端代码，一个复杂编译 node 端代码。没错同一套 webpack 配置在稍加修改后可以编译出能在 node 环境运行的代码。

也就说，客户端的 compiler 负责提供样式、脚本等静态资源，而 node 端的 compiler 负责生成页面并将样式脚本等标签插入到页面中。

所以客户端的 compiler 还需要通过某种方式和 node 端的 compiler 共享编译后的结果，也就是本次客户端编译的 compilation 或是 stats。

通常两个 compiler 的编译并不是并行的，客户端要先编译完成，这样服务端总是可以拿到最新的结果。

有一种做法是利用[webpack-dev-server](https://webpack.js.org/api/webpack-dev-server/)作为服务端，既提供编译后得到的静态资源，也能生成渲染后的 html 页面。根据 wds 的文档，我们需要提供一个[multicompiler](https://webpack.js.org/api/node/#multicompiler)给 wds。这个 multicompiler 中客户端的配置在前，node 端的在后，以保证客户端代码先编译完成。这时候需要将客户端的 stats 或是相关 chunks、assets 等信息保存起来。这个可以通过 [webpack hook](https://webpack.js.org/api/compiler-hooks/#done)来实现，又或者是通过[webpack-manifest-plugin](https://github.com/shellscape/webpack-manifest-plugin)这种插件来输出。总之要确保等会 node 端编译完成以后可以拿到。然后，我们需要给 wds 增加一个[中间件](https://webpack.js.org/configuration/dev-server/#devserversetupmiddlewares)来处理页面访问的请求，在这个中间件里调用 node 端编译好的代码来渲染组件生成静态页面，并且将客户端的脚本样式等插入到页面中返回给用户。

这里还有一点要注意，wds 内部使用了[webpack-dev-middleware](https://github.com/webpack/webpack-dev-middleware)，它调用 webpack 来编译打包并且保证在完成之前能到延迟客户端请求到编译打包完成。但是它默认是在内存中保存这些编译结果的，所以我们需要它把 node 端的代码文件[写到磁盘中](https://github.com/webpack/webpack-dev-middleware#writetodisk)，这样才能确保可以`require`到 node 端的代码。

最后一个问题是 node 端的代码可能在每次编译后都会有更新，但是我们`require`一次以后，会在`require.cache`中产生缓存。这个时候就需要及时的清除缓存，以保证下次`require`到的是最新的代码。或者是考虑使用 node 端的 [hmr](https://github.com/webpack/docs/issues/45)，这样就不需要我们手动清除缓存了。

## build 模式

在这个环节就没那么复杂了，但是依然要先编译打包客户端的包，然后 node 端再介入。这个时候 node 端的打包产物可能最终只导出一个`render`方法，来渲染对应的页面。这样可以避免和特定的服务端框架耦合。

## 静态中心化路由和组件数据预备

试想一下，如果服务端在响应用户请求时，需要根据渲染的页面，提前准备好页面需要的数据。这是一个很常见的需求，可以提升首次渲染的响应速度。那么怎么才能知道当前请求的 url 对应的是哪个页面的组件呢，又该如何知道这个页面组件需要什么数据呢。

针对这两个问题，首先可以用预先定义好的路由来解决组件定位的问题，再商定需要服务端预先准备数据的页面组件可以定义一个静态的`getInitialProps`方法，执行后返回的数据通过 props 传入组件。

## SSR 页面缓存

由于每次响应的页面都是即时渲染的，如果请求过于频繁，会对服务器造成不小的压力。这时就需要一些缓存策略来减轻一些压力。例如可以利用一些[Caching Headers](https://github.com/vercel/next.js/tree/canary/examples/ssr-caching)。
