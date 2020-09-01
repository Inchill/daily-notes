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

  let start = this.anchorItem.index    // 开始索引项
  let end = lastScreenItem.index     // 结束索引项
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

在这个函数里多次调用了 `_calculateAnchoredItem` 函数，该函数接收 2 个参数 —— 滚动前的锚点项和滚动后产生的距离差，返回计算后的当前锚点项。根据 delta 判断滚动方向，确定需要实例化多少个项目数。最后调用两个函数：`fill` 和 `maybeRequestContent`，完成附加内容填充和数据请求。

#### `fill`

```js
InfiniteScroller.prototype.fill = function (start, end) {
  this.firstAttachedItem = Math.max(0, start)
  if (!this.hasMore) {
    end = Math.min(end, this.items.length)
  }
  this.lastAttachedItem = end
  this.attachContent()
}
```

`fill` 函数接收两个参数：开始下标和结束下标，以此来确定开始锚点项和结束锚点项的下标，最后调用 `attachContent` 方法添加占位内容。

#### `attachContent`

```js
InfiniteScroller.prototype.attachContent = function () {
  let unusedNodes = this._collectUnusedNodes()     // 收集不再使用的节点
  let tombstoneAnimations = this._createDOMNodes(unusedNodes)     // 显示墓碑节点时的过渡动画（当老节点清除，墓碑节点显示时展示此动画）
  this._cleanupUnusedNodes(unusedNodes)     // 清除调不再使用的节点
  this._cacheNodeSize()     // 缓存每个节点的 offset 宽高（如果有实际内容，则仅缓存高度，而不是占位符）
  let curPos = this._fixScrollPosition()    // 修复滚动条位置
  this._setupAnimations(tombstoneAnimations, curPos)
}
```

`_collectUnusedNodes` 函数是根据 `fill` 函数里确定的开始、结束锚点项来计算不再使用的节点的，在函数内部 for 循环遍历 `items` 数组，根据条件得到不再使用的节点。到此为止，这两个函数的作用是当数据还处于请求阶段时，进行占位，类似于骨架屏。

### `onResize` 函数

```js
InfiniteScroller.prototype.onResize = function () {
  let tombstone = this.options.createTombstone()   // 创建墓碑 DOM 节点
  tombstone.style.position = 'absolute'    
  this.scrollerEl.appendChild(tombstone)     // 将墓碑 DOM 节点添加到滚动元素里
  tombstone.style.display = ''
  this.tombstoneHeight = tombstone.offsetHeight    // 获取盒子最终的高
  this.tombstoneWidth = tombstone.offsetWidth    // 获取盒子最终的宽
  this.scrollerEl.removeChild(tombstone)

  for (let i = 0; i < this.items.length; i++) {
    this.items[i].height = this.items[i].width = 0
  }

  this.onScroll()
}
```

`onResize` 的作用是重新计算尺寸，其中墓碑节点的作用类似于骨架屏，对于什么是骨架屏，可以参考这个

<img src="https://upload-images.jianshu.io/upload_images/5595939-5deb0c45881d8120?imageMogr2/auto-orient/strip|imageView2/2/w/830">。


