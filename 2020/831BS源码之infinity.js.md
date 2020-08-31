## `infinity.js`

该模块用于做无限滚动的，保证能无限滚动一直加载新的内容而不触底，其实本质上就是一个插件。

该插件接收一个参数，即 `better-scroll` 实例，然后 `new` 一个 `InfiniteScroller` 实例，需要传递 `better-scroll` 实例以及一个 `infinity` 配置对象进去，相当于在原有的 `better-scroll` 基础上继续封装。

### `InfiniteScroller` 函数

该函数主要做了两件事：

1. 初始化参数；

2. 注册事件

```js
function InfiniteScroller(scroller, options) {
  // ...
  this.scroller.on('scroll', () => {
    this.onScroll()
  })
  this.scroller.on('resize', () => {
    this.onResize()
  })

  this.onResize()
}
```

主要来看一下 `onScroll` 和 `onResize` 这两个函数。

### `onScroll` 函数

```js
InfiniteScroller.prototype.onScroll = function () {
  const scrollTop = -this.scroller.y    // 屏幕左上角坐标是(0, 0)，向上滚动是沿着y轴负方向进行的，所以这里取负值
  let delta = scrollTop - this.anchorScrollTop    // 计算与上一次滚动锚点的距离差
  if (scrollTop === 0) {
    this.anchorItem = {
      index: 0,
      offset: 0
    }
  } else {
    this.anchorItem = this._calculateAnchoredItem(this.anchorItem, delta)     // 计算锚点项的位置
  }

  this.anchorScrollTop = scrollTop    // 更改新的锚点为当前的scrollTop
  let lastScreenItem = this._calculateAnchoredItem(this.anchorItem, this.wrapperEl.offsetHeight)  // 根据包裹容器的高度计算上一屏最后一个元素的位置

  let start = this.anchorItem.index
  let end = lastScreenItem.index
  if (delta < 0) {     // 向上滚动
    // RUNWAY_ITEMS：要在滚动方向上实例化超出当前视图的项目数。
    start -= RUNWAY_ITEMS
    // RUNWAY_ITEMS_OPPOSITE：以相反的方向实例化超出当前视图的项目数。
    end += RUNWAY_ITEMS_OPPOSITE
  } else {    // 向下滚动
    start -= RUNWAY_ITEMS_OPPOSITE
    end += RUNWAY_ITEMS
  }
  this.fill(start, end)
  this.maybeRequestContent()
}
```

在这个函数里多次调用了 `_calculateAnchoredItem` 函数，该函数接收 2 个参数 —— 滚动前的锚点项和滚动后产生的距离差，返回计算后的当前锚点项。根据 delta 判断滚动方向，确定需要实例话多少个项目数。最后调用两个函数：



