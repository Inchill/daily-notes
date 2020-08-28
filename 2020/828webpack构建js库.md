## 如何将打包后的js库暴露为一个全局变量？

配置 `webpack.output.libraryTarget`，`libraryTarget` 有如下参数：

```
"var" | "assign" | "this" | "window" | "self" | "global" | "commonjs" | "commonjs2" | "commonjs-module" | "amd" | "amd-require" | "umd" | "umd2" | "jsonp" | "system"
```

同时还得指明库的名字：library。举例来说，我模拟了 vue 的实现，打包前需要这样配置：

```js
module.exports = {
  // ...
  output: {
    filename: 'vue.mini.js',
    path: path.join(__dirname, 'dist'),
    library: 'Vue',
    libraryTarget: 'window'
  },
  // ...
}
```

然后新建一个 html 页面，引入打包后生成的 `vue.mini.js` 库，在控制台打印 window 对象，可以看到暴露出来的全局变量 Vue。
