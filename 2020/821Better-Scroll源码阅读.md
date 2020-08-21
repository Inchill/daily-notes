## 前言

阅读 Better-Scroll 源码时，我是在 node_modules 里面开始的。版本并不是最新，是 1.12.6。其中 src 存放了源代码，其目录结构如下：

```
src
├─scroll
├─util
├─index.js
```

## index.js

index.js 文件只做了一件事，对外暴露 BScroll 构造函数，这个构造函数接收 2 个参数，分别是 el 需要绑定的 DOM 元素和 options 配置项。

```js
function BScroll(el, options) {
  this.wrapper = typeof el === 'string' ? document.querySelector(el) : el
  if (!this.wrapper) {
    warn('Can not resolve the wrapper DOM.')
  }
  this.scroller = this.wrapper.children[0]
  if (!this.scroller) {
    warn('The wrapper need at least one child element to be scroller.')
  }
  // cache style for better performance
  this.scrollerStyle = this.scroller.style

  this._init(el, options)
}
```

在构造函数里也可以清晰地看到，第一步是获取包裹的父容器元素。如果用户传递的是一个字符串，将会调用 document.querySelector 方法获取 DOM 实例；如果用户传递的是已经获取到的 DOM 实例，将会直接进行赋值操作。 

官方文档说的是，传入一个 DOM 容器元素，对其第一个子元素添加滚动效果，因此函数里将 DOM 容器元素的第一个子元素赋值给了 scroller 这个变量。同时对滚动样式进行了初始化操作，将 scroller 的 style 样式添加到 BScroll 实例上。

