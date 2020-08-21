## 前言

阅读 `Better-Scroll` 源码时，我是在 `node_modules` 里面开始的。版本并不是最新，是 1.12.6。其中 `src` 存放了源代码，其目录结构如下：

```
src
├─scroll
├─util
├─index.js
```

在后续源码阅读中，我会在源码里加上注释，便于更少的文字性描述，使得阅读体验更好。

## `index.js`

`index.js` 文件只做了一件事，对外暴露 `BScroll` 构造函数，这个构造函数接收 2 个参数，分别是 `el` 需要绑定的 DOM 元素和 `options` 配置项。

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

在构造函数里也可以清晰地看到，第一步是获取包裹的父容器元素。如果用户传递的是一个字符串，将会调用 `document.querySelector` 方法获取对应的 `DOM` 实例；如果用户传递的是已经获取到的 `DOM` 实例，将会直接进行赋值操作。 

官方文档说的是，传入一个 `DOM` 容器元素，对其第一个子元素添加滚动效果，因此函数里将 `DOM` 容器元素的第一个子元素赋值给了 `scroller` 这个变量。同时对滚动样式进行了初始化操作，将 `scroller` 的 `style` 样式添加到 `BScroll` 实例上。

最后一步调用了自身原型上挂载的 `_init` 方法进行真正的初始化。

## `init.js`

`init.js` 对外暴露了 `initMixin` 方法，它接收 `BScroll` 作为参数，然后在 `BScroll` 原型上封装了一系列方法。上面提到的 `_init` 方法就是在这个方法里封装的。

### `_init`方法

`_init`函数进行了如下的一些操作：

1. 调用 `_handleOptions` 方法

由于用户可以自定义配置项，所以通过构造函数传进来的 `options` 配置项可能是不完整的。这很好理解，毕竟用户针对某个特定场景只会使用其中一些配置来完成需求。`_handleOptions` 的第一步操作就是将自身默认配置项和用户自定义配置项进行合并，然后设置 `translateZ`、`options.useTransition`、`options.useTransform`、`options.preventDefault`等变量，具体可以看源码是怎么做的：

```js
BScroll.prototype._handleOptions = function (options) {
  // 合并配置项
  this.options = extend({}, DEFAULT_OPTIONS, options)

  // HMCompositing 是否开启硬件加速，默认开启，这是为了有更好的滚动效果。
  // 如果开启了硬件加速并且支持三维CSS，将 translateZ 设置为 translateZ(0)，否则为空字符串 
  this.translateZ = this.options.HWCompositing && hasPerspective ? ' translateZ(0)' : ''

  // 是否使用 CSS3 transition 动画
  this.options.useTransition = this.options.useTransition && hasTransition
  // 是否使用 CSS3 transform 做位移
  this.options.useTransform = this.options.useTransform && hasTransform

  // 当事件派发后是否阻止浏览器默认行为
  this.options.preventDefault = !this.options.eventPassthrough && this.options.preventDefault

  // If you want eventPassthrough I have to lock one of the axes
  // 是否保留另一个方向上的滚动
  this.options.scrollX = this.options.eventPassthrough === 'horizontal' ? false : this.options.scrollX
  this.options.scrollY = this.options.eventPassthrough === 'vertical' ? false : this.options.scrollY

  // With eventPassthrough we also need lockDirection mechanism
  // 是否支持横向和纵向同时滚动
  this.options.freeScroll = this.options.freeScroll && !this.options.eventPassthrough
  // 当我们需要锁定只滚动一个方向的时候，我们在初始滚动的时候根据横轴和纵轴滚动的绝对值做差，当差值大于 directionLockThreshold 的时候来决定滚动锁定的方向。
  this.options.directionLockThreshold = this.options.eventPassthrough ? 0 : this.options.directionLockThreshold

  // 因为 better-scroll 会阻止原生的 click 事件，我们可以设置 tap 为 true，它会在区域被点击的时候派发一个 tap 事件，你可以像监听原生事件那样去监听它
  if (this.options.tap === true) {
    this.options.tap = 'tap'
  }
}
```

在上述过程中，我对于某些设置产生了一些疑惑：

- 为什么开启了硬件加速并且支持三维CSS，将 translateZ 设置为 translateZ(0)？

[传送门](https://segmentfault.com/q/1010000007962353)

参考文章：[一篇文章说清浏览器解析和CSS（GPU）动画优化](https://segmentfault.com/a/1190000008015671)

2. 初始化参数

```js
// init private custom events
this._events = {}

// scroll 横纵轴坐标
this.x = 0
this.y = 0
// 判断 scroll 滑动结束后相对于开始滑动位置的方向（左右）
this.directionX = 0
this.directionY = 0
```

3. 调用 `setScale(1)` 方法

```js
BScroll.prototype.setScale = function (scale) {
  this.lastScale = isUndef(this.scale) ? scale : this.scale
  this.scale = scale
}
```

在设置缩放比例之前，会保留上一次的缩放比，然后再设置当前缩放比。

4. 调用 `_addDOMEvents()` 方法注册事件

```js
BScroll.prototype._addDOMEvents = function () {
  let eventOperation = addEvent
  this._handleDOMEvents(eventOperation)
}
```

`addEvent` 变量是 `util` 目录下的 `dom.js` 文件里定义的，我们来看一下这个变量是如何定义的：

`dom.js` 文件对外暴露的 `addEvent` 其实是一个函数：

```js
export function addEvent(el, type, fn, capture) {
  el.addEventListener(type, fn, {passive: false, capture: !!capture})
}
```

使用这个函数需要传递 4 个参数：元素对象、事件类型、事件处理函数及 `capture`，然后函数内部注册了监听事件。在第三个参数中，`passive: true` 表示会调用 `preventDefault()`，`capture` 表示事件处理函数会在该类型的事件捕获阶段传播到该元素时触发。

5. 调用 `_initExtFeatures()` 方法添加高级选项配置

```js
BScroll.prototype._initExtFeatures = function () {
  // 这个配置是为了做 slide 组件用的，默认为 false，如果开启则需要配置一个 Object
  if (this.options.snap) {
    this._initSnap()
  }
  // 是否开启滚动条，默认为 false
  if (this.options.scrollbar) {
    this._initScrollbar()
  }
  // 是否开启上拉加载功能，默认为 false
  if (this.options.pullUpLoad) {
    this._initPullUp()
  }
  // 是否开启上拉加载功能，默认为 false
  if (this.options.pullDownRefresh) {
    this._initPullDown()
  }
  // 这个配置是为了做 picker 组件用的，默认为 false，如果开启则需要配置一个 Object
  if (this.options.wheel) {
    this._initWheel()
  }
  // 是否开启了鼠标滚动条功能
  if (this.options.mouseWheel) {
    this._initMouseWheel()
  }
  // 是否开启了缩放功能
  if (this.options.zoom) {
    this._initZoom()
  }
  // 是否开启了无限虚拟长列表滚动
  if (this.options.infinity) {
    this._initInfinite()
  }
}
```

6. 调用 `_watchTransition` 方法

```js
BScroll.prototype._watchTransition = function () {
  // 这里因为版本比较低，考虑了兼容性问题，因此做了一下判断
  if (typeof Object.defineProperty !== 'function') {
    return
  }
  let me = this
  let isInTransition = false
  let key = this.useTransition ? 'isInTransition' : 'isAnimating'
  Object.defineProperty(this, key, {
    get() {
      return isInTransition
    },
    set(newVal) {
      isInTransition = newVal
      // fix issue #359
      let el = me.scroller.children.length ? me.scroller.children : [me.scroller]
      let pointerEvents = (isInTransition && !me.pulling) ? 'none' : 'auto'
      for (let i = 0; i < el.length; i++) {
        el[i].style.pointerEvents = pointerEvents
      }
    }
  })
}
```

7. 判断是否支持插件和自动模糊

这两个选项是根据用户来配置的，源码如下：

```js
// 是否通过插件支持
if (this.options.observeDOM) {
  this._initDOMObserver()
}

// 根据配置自动模糊
if (this.options.autoBlur) {
  this._handleAutoBlur()
}
```


