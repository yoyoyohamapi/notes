# The Cost Of JavaScript

[原文地址](https://medium.com/dev-channel/the-cost-of-javascript-84009f51e99e)

### 减少网络层开销

![](https://cdn-images-1.medium.com/max/1600/1*U00XcnhqoczTuJ8NH8UhOw.png)

- **只传输用户需要的代码**：code-splitting
- **最小化代码**：ES5 考虑使用 Uglify，ES2015 及以上考虑使用 [babel-minify](https://github.com/babel/minify) or [uglify-es](https://www.npmjs.com/package/uglify-es)。
- **压缩文件**： [Brotli](https://www.smashingmagazine.com/2016/10/next-generation-server-compression-with-brotli/) 是比 gzip 拥有更高压缩率的方案。
- **去除没有用到的代码**： [tree-shaking](https://webpack.js.org/guides/tree-shaking/)、[Closure Compiler](https://developers.google.com/closure/compiler/) 能够优化整个项目。lodash、Moment.js 这样的库也有类似 [lodash-babel-plugin](https://github.com/lodash/babel-plugin-lodash) 和 Webpack [ContextReplacementPlugin](https://iamakulov.com/notes/webpack-front-end-size-caching/#moment-js) 这样的插件。使用 babel-preset-env & browserlist 来避免引入浏览器中已经存在的特性。使用 [Webpack Bundle Analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer) 来帮助我们了解 bundle 的详情。
- **缓存**：为脚本文件设置生命期（max-age）并使用校验口令（ETag）来避免重新传输脚本文件。Service Worker 的缓存能带来离线体验。Webpack 的 bundle 文件 [hash 策略](filename hashing)可以看看。

