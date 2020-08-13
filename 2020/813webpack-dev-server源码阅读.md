# webpack-dev-server源码阅读

在node_modules下找到webpack-dev-server，截止本文发布前其版本是3.11.0。其中lib存放了webpack-dev-server源码，找到Server.js，这便是核心源码部分。

阅读Server.js可知，webpack-dev-server是用express搭建的服务器，并依赖webpack-dev-middleware作为中间件实现的。源码中引入了以下几个核心部分：

```js
// Server.js

const express = require('express');
const webpack = require('webpack');
const webpackDevMiddleware = require('webpack-dev-middleware');
```

当前版本的webpack-dev-server基于ES6的class语法编写的，这样阅读起来就方便了很多。

## Server构造函数

```js
// Server.js

class Server {
  constructor(compiler, options = {}, _log) {
    if (options.lazy && !options.filename) {
      throw new Error("'filename' option must be set in lazy mode.");
    }

    validateOptions(schema, options, 'webpack Dev Server');

    this.compiler = compiler;
    this.options = options;

    this.log = _log || createLogger(options);

    if (this.options.transportMode !== undefined) {
      this.log.warn(
        'transportMode is an experimental option, meaning its usage could potentially change without warning'
      );
    }

    normalizeOptions(this.compiler, this.options);

    updateCompiler(this.compiler, this.options);

    this.heartbeatInterval = 30000;
    // this.SocketServerImplementation is a class, so it must be instantiated before use
    this.socketServerImplementation = getSocketServerImplementation(
      this.options
    );

    this.originalStats =
      this.options.stats && Object.keys(this.options.stats).length
        ? this.options.stats
        : {};

    this.sockets = [];
    this.contentBaseWatchers = [];

    // TODO this.<property> is deprecated (remove them in next major release.) in favor this.options.<property>
    this.hot = this.options.hot || this.options.hotOnly;
    this.headers = this.options.headers;
    this.progress = this.options.progress;

    this.serveIndex = this.options.serveIndex;

    this.clientOverlay = this.options.overlay;
    this.clientLogLevel = this.options.clientLogLevel;

    this.publicHost = this.options.public;
    this.allowedHosts = this.options.allowedHosts;
    this.disableHostCheck = !!this.options.disableHostCheck;

    this.watchOptions = options.watchOptions || {};

    // Replace leading and trailing slashes to normalize path
    this.sockPath = `/${
      this.options.sockPath
        ? this.options.sockPath.replace(/^\/|\/$/g, '')
        : 'sockjs-node'
    }`;

    if (this.progress) {
      this.setupProgressPlugin();
    }

    this.setupHooks();
    this.setupApp();
    this.setupCheckHostRoute();
    this.setupDevMiddleware();

    // set express routes
    routes(this);

    // Keep track of websocket proxies for external websocket upgrade.
    this.websocketProxies = [];

    this.setupFeatures();
    this.setupHttps();
    this.createServer();

    killable(this.listeningApp);

    // Proxy websockets without the initial http request
    // https://github.com/chimurai/http-proxy-middleware#external-websocket-upgrade
    this.websocketProxies.forEach(function(wsProxy) {
      this.listeningApp.on('upgrade', wsProxy.upgrade);
    }, this);
  }
}
```

我们可以顺着构造函数来一步一步分析webpack-dev-server是如何运行的。

### 什么是lazy mode？

原文如下：

> This option lets you reduce the compilations in lazy mode. By default in lazy mode, every request results in a new compilation. With filename, it's possible to only compile when a certain file is requested.

翻译过来就是：此选项使您可以在惰性模式下减少编译。 缺省情况下，在惰性模式下，每个请求都将导致一个新的编译。 使用文件名，可以仅在请求特定文件时进行编译。

**需要注意的是，filename需要和output.filename一致。**

明白了这些，再来看第一部分逻辑：

```js
if (options.lazy && !options.filename) {
  throw new Error("'filename' option must be set in lazy mode.");
}
```

这里是错误处理，先看配置项是否有错误，有错误就不用再浪费时间执行后续代码了。在开启惰性模式的情况下没有配置filename，就会抛出这个错误。

### 配置项校验

接下来执行的是validateOptions(schema, options, 'webpack Dev Server')函数，这里对传进来的配置项进行校验，其中validateOptions是第三方库提供的：

```js
const validateOptions = require('schema-utils');
const schema = require('./options.json');
```

options.json文件是与Server.js同级的json文件，定义了配置项的格式规范。这一步是对配置文件进行校验。

### 赋值操作

```js
this.compiler = compiler;
this.options = options;

this.log = _log || createLogger(options);
```

在启动webpack-dev-server时，构造函数接收了3个参数，分别是compiler、options、_log。compiler是webpack构建时创建的实例，options则是用户创建的配置项，而_log是一个可选项，如果用户没有传递这个参数，构造函数会调用createLogger(options)函数创建日志实例。

### 更新compiler

在执行这一步之前，运行了一个函数normalizeOptions(this.compiler, this.options)，不过这不是重点，重点是updateCompiler(this.compiler, this.options)函数。这个函数是在utils文件夹下的工具函数，我们来看看这个函数内部具体是怎样的：

inline选项会为入口页面添加“热加载”功能，即代码改变后重新加载页面。这里首先判断如果配置启用了inline模式：

```js
if (options.inline !== false) {
  // ...
}
```

如果为true，则执行以下步骤：

1. 定义查找热模块替换的插件

```js
const findHMRPlugin = (config) => {
  if (!config.plugins) {
    return undefined;
  }

  return config.plugins.find(
    (plugin) => plugin.constructor === webpack.HotModuleReplacementPlugin
  );
};
```

2. 遍历compilers

```js
const compilers = [];
const compilersWithoutHMR = [];
let webpackConfig;
if (compiler.compilers) {
  webpackConfig = [];
  compiler.compilers.forEach((compiler) => {
    webpackConfig.push(compiler.options);
    compilers.push(compiler);
    if (!findHMRPlugin(compiler.options)) {
      compilersWithoutHMR.push(compiler);
    }
  });
} else {
  webpackConfig = compiler.options;
  compilers.push(compiler);
  if (!findHMRPlugin(compiler.options)) {
    compilersWithoutHMR.push(compiler);
  }
}
```

由于webpack配置文件可能有多个，比如我们在开发中喜欢配置common、dev、prod配置文件。在这里先定义一个空的webpackConfig数组和compilers数组，再遍历compiler的compilers数组，将options配置项push进去，以及将遍历的每一个compiler保存在数组中。如果当前compiler配置项没有配置inline模式，则push到compilersWithoutHMR数组中。

如果是单文件入口，webpackConfig直接是传进来的options，compilers数组只保存一个compiler即可。同理，未配置inline模式的，push到compilersWithoutHMR数组中。

3. 添加入口文件并自动加载各个模块

```js
addEntries(webpackConfig, options);
compilers.forEach((compiler) => {
  const config = compiler.options;
  compiler.hooks.entryOption.call(config.context, config.entry);

  const providePlugin = new webpack.ProvidePlugin({
    __webpack_dev_server_client__: getSocketClientPath(options),
  });
  providePlugin.apply(compiler);
});
```

ProvidePlugin的使用，会自动加载模块，不用到处import或require。关于ProvidePlugin具体用法，请阅读[ProvidePlugin](https://www.webpackjs.com/plugins/provide-plugin/)

4. 开启热更新

```js
if (options.hot || options.hotOnly) {
  compilersWithoutHMR.forEach((compiler) => {
    // addDevServerEntrypoints above should have added the plugin
    // to the compiler options
    const plugin = findHMRPlugin(compiler.options);
    if (plugin) {
      plugin.apply(compiler);
    }
  });
}
```

### 初始化webpack-dev-server启动所需配置项

```js
this.heartbeatInterval = 30000;
// this.SocketServerImplementation is a class, so it must be instantiated before use
this.socketServerImplementation = getSocketServerImplementation(
  this.options
);

this.originalStats =
  this.options.stats && Object.keys(this.options.stats).length
    ? this.options.stats
    : {};

this.sockets = [];
this.contentBaseWatchers = [];

// TODO this.<property> is deprecated (remove them in next major release.) in favor this.options.<property>
this.hot = this.options.hot || this.options.hotOnly;
this.headers = this.options.headers;
this.progress = this.options.progress;

this.serveIndex = this.options.serveIndex;

this.clientOverlay = this.options.overlay;
this.clientLogLevel = this.options.clientLogLevel;

this.publicHost = this.options.public;
this.allowedHosts = this.options.allowedHosts;
this.disableHostCheck = !!this.options.disableHostCheck;

this.watchOptions = options.watchOptions || {};

// Replace leading and trailing slashes to normalize path
this.sockPath = `/${
  this.options.sockPath
    ? this.options.sockPath.replace(/^\/|\/$/g, '')
    : 'sockjs-node'
}`;
```

### 创建并运行webpack-dev-server

```js
if (this.progress) {
  this.setupProgressPlugin();
}

this.setupHooks();
this.setupApp();
this.setupCheckHostRoute();
this.setupDevMiddleware();

// set express routes
routes(this);

// Keep track of websocket proxies for external websocket upgrade.
this.websocketProxies = [];

this.setupFeatures();
this.setupHttps();
this.createServer();

killable(this.listeningApp);

// Proxy websockets without the initial http request
// https://github.com/chimurai/http-proxy-middleware#external-websocket-upgrade
this.websocketProxies.forEach(function(wsProxy) {
  this.listeningApp.on('upgrade', wsProxy.upgrade);
}, this);
```

setupHooks()负责将invalidPlugin挂载到webpack实例compiler的compiler、invalid、done钩子函数上，setupApp()负责实例化一个express对象。webpack-dev-server的核心部分是采用的webpack-dev-middleware中间件，这个中间件的作用呢，简单总结为以下三点：

1. 通过watch mode，监听资源的变更，然后自动打包;

2. 快速编译，将打包资源存储在内存；

3. 返回中间件，支持express的use格式。

webpack明明可以用watch mode，可以实现一样的效果，但是为什么还需要这个中间件呢？

只依赖webpack的watch mode来监听文件变更，自动打包，每次变更，都将新文件打包到硬盘，就会很慢。

```js
setupDevMiddleware() {
  // middleware for serving webpack bundle
  this.middleware = webpackDevMiddleware(
    this.compiler,
    Object.assign({}, this.options, { logLevel: this.log.options.level })
  );
}
```
