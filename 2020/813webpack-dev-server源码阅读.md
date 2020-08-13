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



在启动webpack-dev-server时，构造函数接收了3个参数，分别是compiler、options、_log。compiler是webpack构建时创建的实例，options则是用户创建的配置项，而_log是一个可选项，如果用户没有传递这个参数，构造函数会调用createLogger(options)函数创建日志实例。



