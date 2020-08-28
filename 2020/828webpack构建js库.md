## Q1:如何将打包后的js库暴露为一个全局变量？

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

## Q2:入口文件使用ES module还是使用CommonJS暴露？

- 使用ES Module：`export default Vue`

这种形式打包后生成的Vue，打印之后发现是这样的:

<img src="https://github.com/Inchill/mini-vue/blob/master/images/window-vue.png">

然后new Vue实例的时候，控制台会报错：

```shell
Uncaught TypeError: Vue is not a constructor
```

因为构造函数变成了 default，所以需要写成这样的格式：`new Vue.default({ //... })`，这样子显然不够好，于是我把入口文件暴露的 Vue 改成了CommonJS格式：`module.exports = Vue`。这样写之后，再次打包，直接 new Vue 实例就可以了。
