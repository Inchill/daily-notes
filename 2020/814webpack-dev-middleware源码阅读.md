# webpack-dev-middleware源码阅读

查看webpack-dev-server的package.json文件得知，当前webpack-dev-middleware版本是3.7.2。

当前webpack-dev-middleware项目根目录包含lib和index.js，这属于源码部分。

## index.js

index.js文件对外暴露了一个函数，这个函数接收两个参数：compiler、opts，最终返回一个包装好的中间件对象。

1. 拷贝配置项

在index.js文件里，定义了一个名为defaults的对象，表示默认配置项，默认配置项默认不写入本地磁盘，这是为了提升响应速度。

```js
const defaults = {
  logLevel: 'info',
  logTime: false,
  logger: null,
  mimeTypes: null,
  reporter,
  stats: {
    colors: true,
    context: process.cwd(),
  },
  watchOptions: {
    aggregateTimeout: 200,
  },
  writeToDisk: false,
};
```

接下来在函数里拷贝配置项：`const options = Object.assign({}, defaults, opts);`

2. createContext函数

createContext接收两个参数：compiler、options，返回一个context上下文对象。

在函数内部，首先创建一个context对象，函数的后续操作就是对这个context对象属性一一赋值。

```js
const context = {
  state: false,
  webpackStats: null,
  callbacks: [],
  options,
  compiler,
  watching: null,
  forceRebuild: false,
};
```

然后定义invalid、run、done、watchRun四个钩子函数，因为传进来了compiler对象，webpack本身提供了很多hooks，所以在函数的最后，执行了以下步骤，将定义的钩子函数挂载到compiler实例上。关于webpack钩子函数详情，可以阅读[compiler钩子](https://www.webpackjs.com/api/compiler-hooks/)

```js
context.rebuild = rebuild;
context.compiler.hooks.invalid.tap('WebpackDevMiddleware', invalid);
context.compiler.hooks.run.tap('WebpackDevMiddleware', invalid);
context.compiler.hooks.done.tap('WebpackDevMiddleware', done);
context.compiler.hooks.watchRun.tap(
  'WebpackDevMiddleware',
  (comp, callback) => {
    invalid(callback);
  }
);
```

注意rebuild并没有带上()，因此这里不会立即执行rebuild函数，只是进行了赋值操作，需要用户调用时触发。

3. watching

在这一步开始进行监听，当文件发生变化时，立即进行构建打包更新资源。

- 先判断是否开启懒加载

当启用lazy.dev-server会仅在请求时进行编译。
这意味着webpack不会监控文件改变，所以称该模式为lazy mode.

> 当在lazy模式下，watchOptions将不会被启用
>
> 如果在CLI下使用，需要确保inline mode被禁用

如果lazy为false，将进行如下操作：

```js
context.watching = compiler.watch(options.watchOptions, (err) => {
  if (err) {
    context.log.error(err.stack || err);
    if (err.details) {
      context.log.error(err.details);
    }
  }
});
```

如果开启了lazy模式，再判断filename是否是字符串，如果配置正确，将设置context的filename以及state为true。

如果配置了将打包资源写入硬盘，将会调用toDisk函数，传入context参数。完成这一步骤后，再调用setFs函数，传入context和compiler对象。这两个函数，都是由lib文件夹下的fs.js工具函数提供的。

4. middleware.js

返回中间件对象时调用了Object.assign函数，第一个参数是目标对象middleware(context)，第二个参数是自定义的对象，包含close、invalidate、waitUntilValid三个函数。我们来重点看下middleware(context):

在middleware.js文件里，对外暴露了middleware(context)这个方法，而这个方法又返回一个函数middleware(req, res, next)，用来模拟请求。

在这个函数里，定义了goNext方法，如果是服务端渲染的，直接返回next函数，否则返回一个Promise，在Promise里执行了自定义的ready函数，这个函数接收context、一个匿名函数以及req。


