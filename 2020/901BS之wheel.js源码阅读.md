## 这个插件是干嘛的？

官网上这么说的，wheel 插件，是实现类似 IOS Picker 组件的基石。该插件源码很少，只封装了 3 个函数，接下来一一分析。

## `wheelTo`

```js
BScroll.prototype.wheelTo = function (index = 0) {
  if (this.options.wheel) {
    this.y = -index * this.itemHeight
    this.scrollTo(0, this.y)
  }
}
```

这里的 index 和配置项里的 selectedIndex 是同一个，默认选中第 selectedIndex 项，索引从 0 开始。如果是从第 0 项开始，是不能往下滚动的，所以这里的 index 始终为自然数，因此滚动条卷去的高度始终是小于等于 0 的，保证滚动是从第0项起是向上滚动的。

计算出滚动条卷去的高度后，再调用 scrollTo方法滚动到新的位置。

## `getSelectedIndex`

```js
BScroll.prototype.getSelectedIndex = function () {
  return this.options.wheel && this.selectedIndex
}
```

没啥分析的，就是返回选中项的索引。

## `_initWheel`

```js
BScroll.prototype._initWheel = function () {
  const wheel = this.options.wheel
  if (!wheel.wheelWrapperClass) {
    wheel.wheelWrapperClass = 'wheel-scroll'
  }
  if (!wheel.wheelItemClass) {
    wheel.wheelItemClass = 'wheel-item'
  }
  if (wheel.selectedIndex === undefined) {
    wheel.selectedIndex = 0
    warn('wheel option selectedIndex is required!')
  }
}
```

初始化 `Wheel`，判断相应的类是否存在，不存在则添加默认类名，并且将 selectedIndex 置为 0.
