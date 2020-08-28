## 如何将打包后的js库暴露为一个全局变量？

配置 `webpack.output.libraryTarget`，`libraryTarget` 有如下参数：

```
"var" | "assign" | "this" | "window" | "self" | "global" | "commonjs" | "commonjs2" | "commonjs-module" | "amd" | "amd-require" | "umd" | "umd2" | "jsonp" | "system"
```


