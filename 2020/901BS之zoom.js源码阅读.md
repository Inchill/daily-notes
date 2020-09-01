zoom 插件为 BetterScroll 提供缩放功能，除了提供了一个 zoomTo 方法，其余的都是私有方法。

## `_initZoom` 初始化

```js
BScroll.prototype._initZoom = function () {
  const {start = 1, min = 1, max = 4} = this.options.zoom   // 设置初始缩放比例、最大及最小缩放比例
  this.scale = Math.min(Math.max(start, min), max)    // 保证缩放比在设定范围内
  this.setScale(this.scale)    // 调用函数设置缩放比
  this.scrollerStyle[style.transformOrigin] = '0 0'
}
```

## `_zoomStart` 缩放开始

```js
BScroll.prototype._zoomStart = function (e) {
  // 获取触点对象
  const firstFinger = e.touches[0]
  const secondFinger = e.touches[1]
  // 计算触点间的x、y绝对值差值
  const deltaX = Math.abs(firstFinger.pageX - secondFinger.pageX)
  const deltaY = Math.abs(firstFinger.pageY - secondFinger.pageY)

  // 获取触点间的距离
  this.startDistance = getDistance(deltaX, deltaY)
  this.startScale = this.scale

  let {left, top} = offsetToBody(this.wrapper)   // 获取元素在 body 里到左边、顶部的距离

  // 缩放原点的 x，y 坐标，相当于缩放元素的左右顶点
  this.originX = Math.abs(firstFinger.pageX + secondFinger.pageX) / 2 + left - this.x
  this.originY = Math.abs(firstFinger.pageY + secondFinger.pageY) / 2 + top - this.y

  this.trigger('zoomStart')   // 触发自定义钩子函数，用户可自定义回调函数
}
```

这里提一下 `offsetToBody` 这个函数，其内部具体实现如下：

```js
export function offsetToBody(el) {
  let rect = el.getBoundingClientRect()

  return {
    left: -(rect.left + window.pageXOffset),     // 滚动条卷去的距离 + 元素到当前视口左边的距离 = 元素在滚动元素内的实际位置
    top: -(rect.top + window.pageYOffset)
  }
}
```

Element.getBoundingClientRect() 方法返回元素的大小及其相对于视口的位置，是一个集合，该集合包含left, top, right, bottom, x, y, width, 和 height 只读属性。

<img src="https://mdn.mozillademos.org/files/15087/rect.png">

**返回参数说明：**

rectObject.top：元素上边到视窗上边的距离;

rectObject.right：元素右边到视窗左边的距离;

rectObject.bottom：元素下边到视窗上边的距离;

rectObject.left：元素左边到视窗左边的距离;

[链接：](https://www.jianshu.com/p/824eb6f9dda4)

详情阅读[MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/getBoundingClientRect)。

在此函数里，用到了 `window.pageXOffset`，它是 scrollX 属性的别名。

pageX：触点相对于HTML文档左边沿的的X坐标. 和 clientX 属性不同, 这个值是相对于整个html文档的坐标, 和用户滚动位置无关. 因此当存在水平滚动的偏移时, 这个值包含了水平滚动的偏移.

## `_zoom` 缩放中

```js
BScroll.prototype._zoom = function (e) {
  if (!this.enabled || this.destroyed || eventType[e.type] !== this.initiated) {
    return
  }

  if (this.options.preventDefault) {
    e.preventDefault()
  }

  if (this.options.stopPropagation) {
    e.stopPropagation()
  }

  const firstFinger = e.touches[0]
  const secondFinger = e.touches[1]
  const deltaX = Math.abs(firstFinger.pageX - secondFinger.pageX)
  const deltaY = Math.abs(firstFinger.pageY - secondFinger.pageY)
  const distance = getDistance(deltaX, deltaY)
  let scale = distance / this.startDistance * this.startScale     // 计算相对于初始时的缩放比

  this.scaled = true   // 设置缩放标志为 true

  const {min = 1, max = 4} = this.options.zoom

  if (scale < min) {
    scale = 0.5 * min * Math.pow(2.0, scale / min)
  } else if (scale > max) {
    scale = 2.0 * max * Math.pow(0.5, max / scale)
  }

  const lastScale = scale / this.startScale

  const x = this.startX - (this.originX - this.relativeX) * (lastScale - 1)
  const y = this.startY - (this.originY - this.relativeY) * (lastScale - 1)

  this.setScale(scale)

  this.scrollTo(x, y, 0)
}
```

## `_zoomEnd` 缩放结束

```js
BScroll.prototype._zoomEnd = function (e) {
  if (!this.enabled || this.destroyed || eventType[e.type] !== this.initiated) {
    return
  }

  if (this.options.preventDefault) {
    e.preventDefault()
  }

  if (this.options.stopPropagation) {
    e.stopPropagation()
  }

  this.isInTransition = false
  this.isAnimating = false
  this.initiated = 0

  const {min = 1, max = 4} = this.options.zoom

  const scale = this.scale > max ? max : this.scale < min ? min : this.scale

  this._zoomTo(scale, this.originX, this.originY, this.startScale)

  this.trigger('zoomEnd')
}
```

## `_zoomTo` 缩放到某个位置

```js
BScroll.prototype._zoomTo = function (scale, originX, originY, startScale) {
  this.scaled = true

  const lastScale = scale / (startScale || this.scale)
  this.setScale(scale)

  this.refresh()

  let newX = Math.round(this.startX - (originX - this.relativeX) * (lastScale - 1))
  let newY = Math.round(this.startY - (originY - this.relativeY) * (lastScale - 1))

  if (newX > this.minScrollX) {
    newX = this.minScrollX
  } else if (newX < this.maxScrollX) {
    newX = this.maxScrollX
  }

  if (newY > this.minScrollY) {
    newY = this.minScrollY
  } else if (newY < this.maxScrollY) {
    newY = this.maxScrollY
  }

  if (this.x !== newX || this.y !== newY) {
    this.scrollTo(newX, newY, this.options.bounceTime)
  }

  this.scaled = false
}
```

## `zoomTo` 对外暴露的API

```js
BScroll.prototype.zoomTo = function (scale, x, y) {
  let {left, top} = offsetToBody(this.wrapper)
  let originX = x + left - this.x
  let originY = y + top - this.y
  this._zoomTo(scale, originX, originY)
}
```
