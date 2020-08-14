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

