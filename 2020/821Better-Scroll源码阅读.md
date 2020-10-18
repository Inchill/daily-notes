## 前言

阅读 `Better-Scroll` 源码时，我是在 `node_modules` 里面开始的。版本并不是最新，是 1.12.6。其中 `src` 存放了源代码，其目录结构如下：

```
|——src 
| |—— scroll : 核心功能
| |    |____ core : 处理滚动三大事件、滚动动画有关的函数。
| |    |____ event : 监听滚动时派发的操作事件 (发布/订阅者模式)
| |    |____ init : 初始化：参数配置、DOM 绑定事件、调用其他扩展插件
| |    |____ pulldown : 下拉刷新功能
| |    |____ pullup : 上拉加载更多功能
| |    |____ scrollbar : 生成滚动条、滚动时实时计算大小与位置
| |    |____ snap : 扩展的轮播图及一系列操作
| |    |____ wheel : 扩展的picker插件操作
| |—— util 工具库
| |    |____ debug : 发出警告
| |    |____ dom : 关于 dom 元素操作（缓存、浏览器兼容、绑定、事件、属性...）
| |    |____ ease : 贝塞尔曲线
| |    |____ lang : 获取时间戳、拷贝函数
| |    |____ momentum : 动量公式
| |    |____ raf : 定时器 requestAnimationFrame
| |——index.js 导入全部文件，调用初始化函数
```

在后续源码阅读中，我会在源码里加上注释，便于更少的文字性描述，使得阅读体验更好。

## `index.js`

`index.js` 文件只做了一件事，对外暴露 `BScroll` 构造函数，这个构造函数接收 2 个参数，分别是 `el` 即需要绑定的 DOM 元素和 `options` 配置项。

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

### `_init()`方法

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
// 判断 scroll 滑动结束后相对于开始滑动位置的方向（左右)，-1 表示从左向右滑，1 表示从右向左滑，0 表示没有滑动。
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

4. **调用 `_addDOMEvents()` 方法添加原生事件监听回调**

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

接下来看一下 `_handleDOMEvents(eventOperation)` 方法：

```js
BScroll.prototype._handleDOMEvents = function (eventOperation) {
  let target = this.options.bindToWrapper ? this.wrapper : window
  eventOperation(window, 'orientationchange', this)  
  eventOperation(window, 'resize', this)

  if (this.options.click) {
    eventOperation(this.wrapper, 'click', this, true)
  }

  if (!this.options.disableMouse) {
    eventOperation(this.wrapper, 'mousedown', this)
    eventOperation(target, 'mousemove', this)
    eventOperation(target, 'mousecancel', this)
    eventOperation(target, 'mouseup', this)
  }

  if (hasTouch && !this.options.disableTouch) {
    eventOperation(this.wrapper, 'touchstart', this)
    eventOperation(target, 'touchmove', this)
    eventOperation(target, 'touchcancel', this)
    eventOperation(target, 'touchend', this)
  }

  eventOperation(this.scroller, style.transitionEnd, this)
}
```

按上述定义，第三个参数是 fn，这里为什么传了个对象进去？MDN 是这样说的，listener 必须是一个实现了 EventListener 接口的对象，或者是一个函数。BScroll 原型上有事件处理函数 `handleEvent`:

```js
BScroll.prototype.handleEvent = function (e) {
  switch (e.type) {
    case 'touchstart':
    case 'mousedown':
      this._start(e)
      if (this.options.zoom && e.touches && e.touches.length > 1) {
        this._zoomStart(e)
      }
      break
    case 'touchmove':
    case 'mousemove':
      if (this.options.zoom && e.touches && e.touches.length > 1) {
        this._zoom(e)
      } else {
        this._move(e)
      }
      break
    case 'touchend':
    case 'mouseup':
    case 'touchcancel':
    case 'mousecancel':
      if (this.scaled) {
        this._zoomEnd(e)
      } else {
        this._end(e)
      }
      break
    case 'orientationchange':
    case 'resize':
      this._resize()
      break
    case 'transitionend':
    case 'webkitTransitionEnd':
    case 'oTransitionEnd':
    case 'MSTransitionEnd':
      this._transitionEnd(e)
      break
    case 'click':
      if (this.enabled && !e._constructed) {
        if (!preventDefaultException(e.target, this.options.preventDefaultException)) {
          e.preventDefault()
          e.stopPropagation()
        }
      }
      break
    case 'wheel':
    case 'DOMMouseScroll':
    case 'mousewheel':
      this._onMouseWheel(e)
      break
  }
}
```

然后有一个问题是addEventListener中不是函数而是提供一个具有handleEvent方法的对象，这么做的好处是什么？我们知道addEventListener中如果提供的是一个函数，那么使用removeEventListener清除这个事件监听时要提供一个完全相同的函数才可以。因为我们这里"add"了很多事件回调，因此在最终清除操作的时候，仅仅是因为每个事件的处理函数不同，我们要重复写很多个"remove"的方法。而如果提供的是一个对象的话，即上面的做法，那么清除事件监听时，只需要把addEventListener改为removeEventListener即可，因此_handleDOMEvents把这个操作写成了一个变量，从而在清除事件监听时，可以重用_handleDOMEvents的代码。我们可以在src/scroll/init.js中找到相应的_removeDOMEvents和removeEvent`方法。

```js
BScroll.prototype._removeDOMEvents = function () {
  let eventOperation = removeEvent
  this._handleDOMEvents(eventOperation)
}
```

```js
export function removeEvent(el, type, fn, capture) {
  el.removeEventListener(type, fn, {passive: false, capture: !!capture})
}
```

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

6. 调用 `_watchTransition` 方法监听是否处于过渡动画

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

- _initDOMObserver函数

```js
BScroll.prototype._initDOMObserver = function () {
  if (typeof MutationObserver !== 'undefined') {    // 如果有定义 MutationObserver
    let timer
    let observer = new MutationObserver((mutations) => {
      // don't do any refresh during the transition, or outside of the boundaries
      if (this._shouldNotRefresh()) {
        return
      }
      let immediateRefresh = false
      let deferredRefresh = false
      for (let i = 0; i < mutations.length; i++) {
        const mutation = mutations[i]
        if (mutation.type !== 'attributes') {
          immediateRefresh = true
          break
        } else {
          if (mutation.target !== this.scroller) {
            deferredRefresh = true
            break
          }
        }
      }
      if (immediateRefresh) {
        this.refresh()
      } else if (deferredRefresh) {
        // attributes changes too often
        clearTimeout(timer)
        timer = setTimeout(() => {
          if (!this._shouldNotRefresh()) {
            this.refresh()
          }
        }, 60)
      }
    })
    const config = {
      attributes: true,
      childList: true,
      subtree: true
    }
    observer.observe(this.scroller, config)

    this.on('destroy', () => {
      observer.disconnect()
    })
  } else {
    this._checkDOMUpdate()
  }
}
```

什么是 `MutationObserver`?

MutationObserver接口提供了监视对DOM树所做更改的能力。它被设计为旧的Mutation Events功能的替代品，该功能是DOM3 Events规范的一部分。

它具有的方法如下：

disconnect()
  
  阻止 MutationObserver 实例继续接收的通知，直到再次调用其observe()方法，该观察者对象包含的回调函数都不会再被调用。
  
observe()

  配置MutationObserver在DOM更改匹配给定选项时，通过其回调函数开始接收通知。
  
takeRecords()

  从MutationObserver的通知队列中删除所有待处理的通知，并将它们返回到MutationRecord对象的新Array中。
  
- _handleAutoBlur函数

```js
BScroll.prototype._handleAutoBlur = function () {
  this.on('scrollStart', () => {
    let activeElement = document.activeElement
    if (activeElement && (activeElement.tagName === 'INPUT' || activeElement.tagName === 'TEXTAREA')) {
      activeElement.blur()
    }
  })
}
```

8. 滚动后重新计算 Better-Scroll

```js
// 发生滚动后重新计算 DOM
this.refresh()

// 是否配置slide
if (!this.options.snap) {
  this.scrollTo(this.options.startX, this.options.startY)
}

// 开启 BS 滚动
this.enable()
```

```js
BScroll.prototype.enable = function () {
  this.enabled = true
}
```

重点看一下 `refresh()` 方法：

```js
BScroll.prototype.refresh = function () {
  const isWrapperStatic = window.getComputedStyle(this.wrapper, null).position === 'static'
  // 获取包裹父容器的宽高
  let wrapperRect = getRect(this.wrapper)
  this.wrapperWidth = wrapperRect.width
  this.wrapperHeight = wrapperRect.height

  // 获取滚动子元素(包裹父容器的第一个子元素)的宽高
  let scrollerRect = getRect(this.scroller)
  this.scrollerWidth = Math.round(scrollerRect.width * this.scale)
  this.scrollerHeight = Math.round(scrollerRect.height * this.scale)

  // scroller相对于包裹父元素wrapper的横纵轴坐标
  this.relativeX = scrollerRect.left
  this.relativeY = scrollerRect.top

  // 如果wrapper不具有定位属性
  if (isWrapperStatic) {
    this.relativeX -= wrapperRect.left
    this.relativeY -= wrapperRect.top
  }

  // 初始化滚动元素scroller的最小横纵滚动距离
  this.minScrollX = 0
  this.minScrollY = 0

  // 这个配置是为了做 picker 组件用的，默认为 false
  const wheel = this.options.wheel
  if (wheel) {
    this.items = this.scroller.children
    // 如果滚动元素有子元素，计算每个子元素的高度
    this.options.itemHeight = this.itemHeight = this.items.length ? this.scrollerHeight / this.items.length : 0
    if (this.selectedIndex === undefined) {
      this.selectedIndex = wheel.selectedIndex || 0
    }
    this.options.startY = -this.selectedIndex * this.itemHeight
    this.maxScrollX = 0
    this.maxScrollY = -this.itemHeight * (this.items.length - 1)
  } else {
    // 滚动元素横向能滚动的最大距离等于wrapper宽度减去滚动元素宽度
    this.maxScrollX = this.wrapperWidth - this.scrollerWidth
    if (!this.options.infinity) {
      this.maxScrollY = this.wrapperHeight - this.scrollerHeight
    }
    if (this.maxScrollX < 0) {
      this.maxScrollX -= this.relativeX
      this.minScrollX = -this.relativeX
    } else if (this.scale > 1) {
      this.maxScrollX = (this.maxScrollX / 2 - this.relativeX)
      this.minScrollX = this.maxScrollX
    }
    if (this.maxScrollY < 0) {
      this.maxScrollY -= this.relativeY
      this.minScrollY = -this.relativeY
    } else if (this.scale > 1) {
      this.maxScrollY = (this.maxScrollY / 2 - this.relativeY)
      this.minScrollY = this.maxScrollY
    }
  }

  this.hasHorizontalScroll = this.options.scrollX && this.maxScrollX < this.minScrollX
  this.hasVerticalScroll = this.options.scrollY && this.maxScrollY < this.minScrollY

  if (!this.hasHorizontalScroll) {
    this.maxScrollX = this.minScrollX
    this.scrollerWidth = this.wrapperWidth
  }

  if (!this.hasVerticalScroll) {
    this.maxScrollY = this.minScrollY
    this.scrollerHeight = this.wrapperHeight
  }

  this.endTime = 0
  this.directionX = 0
  this.directionY = 0
  this.wrapperOffset = offset(this.wrapper)

  this.trigger('refresh')

  !this.scaled && this.resetPosition()
}
```

