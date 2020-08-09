在用webpack打包时出现如下错误：

```js
ERROR in ./client/user/index.ts
Module not found: Error: Can't resolve './router' in 'D:\WebProjects\trading\client\user'
 @ ./client/user/index.ts 4:15-34
```

问题的根源是在导入组件时没有指定文件扩展名。真是惭愧😥，使用脚手架构建项目，组件导入的时候都是不用拓展名的，却又没有深究为什么可以不适用拓展名。因为脚手架生成的配置文件中有对模块文件的扩展名的相关配置：

```js
// webpack.config.js中配置如下
module.exports = {
    resolve: {
        extensions: ['.ts', '.js', '.vue', '.json']
    }
}
```

