# CDN引入UI组件

对于一个应用来说，如果资源体积过大，那么势必会影响到性能。所以我们会绞尽脑汁想着去提升应用的性能，而CDN就是一种手段。对于一些提供了CDN的第三方库，我们完全可以不用通过`npm`的形式安装到项目里，而是以CDN方式引入。这样做无疑可以减少资源打包体积，而且应用响应速度更快。

之前我有做过对UI组件CDN引入的尝试，不过除了样式是成功了，组件库倒是没成功过。这反而激起了我的兴趣，所以我特意花费了时间来探索如何更优雅地使用CDN。

在探索过程中，遇到了不少问题，所以我一一记录下了这些问题，方便日后需要。

## externals

webpack目前是前端主流打包工具，对于CDN引入的第三方库，可以通过配置externals。externals作用是防止将某些 import 的包(package)打包到 bundle 中，而是在运行时(runtime)再去从外部获取这些扩展依赖(external dependencies)。

对于网络上更多的是使用CDN来引入Element-UI，所以很容易搜到它对外暴露的是ELEMENT这个全局对象，在externals中配置如下：

```js
// webpack.common.js

module.exports = {
  externals: {
    'vue': 'Vue',
    'element-ui': 'ELEMENT'
  }
}
```

在上述配置中，key可以在项目中通过import的形式引入，而value就是映射的全局对象。那么如何确定第三方库对外暴露的全局名称是什么样的呢？可以在浏览器中打开引入这些第三方库的html文件，在控制台里打印window这个对象，就能看到第三方库对外暴露的名称，然后再在externals里配置。

**注意：对于基于vue开发的UI组件库，vue也必须通过CDN引入，而且注意顺序在组件库之前。这是为了保证UI组件库运行时，vue实例已经存在。**

## CDN结合webpack

我在index.html里CDN引入了vue和at-ui组件库，并对其进行了测试：

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
  <script src="https://cdn.jsdelivr.net/npm/vue@2.6.11"></script>
  <!-- 引入样式 -->
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/at-ui-style/css/at.min.css">
  <!-- 引入组件库 -->
  <script src="https://cdn.jsdelivr.net/npm/at-ui/dist/at.min.js"></script>
</head>
<body>
  <div id="app">
    <at-button type="primary">click me</at-button>
  </div>

  <script type="text/javascript">
    new Vue({
      el: '#app'
    })
  </script>
</body>
</html>
```

这样做是能正常显示按钮的，于是我继续推进，想在vue文件里使用它们，于是做了如下配置:

```js
// index.js
import Vue from 'vue';
import App from './App.vue';

new Vue({
  el: '#app',
  render: h => h(App)
})
```

```js
// webpack.common.js

const path = require('path');
const vueLoaderPlugin = require('vue-loader/lib/plugin');

module.exports = {
  entry: {
    index: path.join(__dirname, '../src/index.js')
  },
  output: {
    filename: '[name]-[hash:6].js',
    path: path.join(__dirname, './dist')
  },
  module: {
    rules: [
      {
        test: /\.vue$/,
        use: 'vue-loader'
      }
    ]
  },
  plugins: [
    new vueLoaderPlugin()
  ],
  externals: {
    'vue': 'Vue',
    'at-ui': 'at'
  },
  resolve: {
    extensions: ['.js']
  }
}
```

```html
// App.vue

<template>
  <div id="app-instance">
    hello
    <at-button type="primary">click me</at-button>
  </div>
</template>

<script>
export default {
  
}
</script>

<style>

</style>
```

然后启动webpack，很遗憾，页面显示的是项目结构，而不是期望的vue实例。这是为什么呢？最后意识到是没有指定模版html文件，所以继续配置webpack.common.js:

```js
// webpack.common.js

const path = require('path');
const vueLoaderPlugin = require('vue-loader/lib/plugin');
const { CleanWebpackPlugin } = require('clean-webpack-plugin');
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
  entry: {
    index: path.join(__dirname, '../src/index.js')
  },
  output: {
    filename: '[name]-[hash:6].js',
    path: path.join(__dirname, './dist')
  },
  module: {
    rules: [
      {
        test: /\.vue$/,
        use: 'vue-loader'
      }
    ]
  },
  plugins: [
    new CleanWebpackPlugin(),
    new HtmlWebpackPlugin({
        filename: 'index.html',
        template: path.join(__dirname, '../src/assets/index.html'),
        chunks: ['index']
    }),
    new vueLoaderPlugin()
  ],
  externals: {
    'vue': 'Vue',
    'at-ui': 'at'
  },
  resolve: {
    extensions: ['.js']
  }
}
```

再次启动webpack，发现输出文件里存在index.html了，浏览器也能显示按钮了，大功告成！！
