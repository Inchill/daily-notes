pullup 插件为 BetterScroll 扩展上拉加载的能力。

关于 pullup 的使用，可以查阅官方文档[pullup](https://better-scroll.github.io/docs/zh-CN/plugins/pullup.html#%E4%BD%BF%E7%94%A8)。

`pullup.js` 在 better-scroll 原型上绑定了 6 个方法，下面将会一一分析：

## `_initPullUp`

```js
BScroll.prototype._initPullUp = function () {
  // must watch scroll in real time(必须实时监测滚动事件)
  this.options.probeType = PROBE_REALTIME

  this.pullupWatching = false
  this._watchPullUp()
}
```

这个函数没什么好说的，设置了
