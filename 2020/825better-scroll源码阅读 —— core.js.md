滚动相关的核心代码，主要是手指在屏幕上滑动开始、滑动中、滑动结束三个时间点 better-scroll 都做了哪些事情。

## `_start()`函数处理滑动开始事件

在 `handleEvent()` 函数中，根据监听事件类型的不同，调用了不同的回调函数。

```js
BScroll.prototype._start = function (e) {
  let _eventType = eventType[e.type]
  if (_eventType !== TOUCH_EVENT) {
    if (e.button !== 0) {   //不是鼠标左键，直接退出
      return
    }
  }
  // 如果未开启滚动或者已经销毁直接退出
  if (!this.enabled || this.destroyed || (this.initiated && this.initiated !== _eventType)) {
    return
  }
  this.initiated = _eventType
  
  // 如果需要阻止浏览器默认行为则执行
  if (this.options.preventDefault && !preventDefaultException(e.target, this.options.preventDefaultException)) {
    e.preventDefault()
  }
  // 阻止事件冒泡
  if (this.options.stopPropagation) {
    e.stopPropagation()
  }

  // 初始化数据
  this.moved = false
  // 手指或鼠标的滑动距离
  this.distX = 0
  this.distY = 0
  // 水平、垂直方向滑动距离
  this.directionX = 0
  this.directionY = 0
  // 水平、垂直方向滑动过程中距离
  this.movingDirectionX = 0
  this.movingDirectionY = 0
  this.directionLocked = 0

  // 初始化过渡时间为0
  this._transitionTime()
  // 初始化开始时间
  this.startTime = getNow()

  if (this.options.wheel) {
    this.target = e.target
  }

  // 如果此时处于惯性滚动过程中，停止惯性滚动。
  this.stop()

  let point = e.touches ? e.touches[0] : e

  // // 滚动的开始位置
  this.startX = this.x
  this.startY = this.y
  this.absStartX = this.x
  this.absStartY = this.y
  // 鼠标相对于页面位置
  this.pointX = point.pageX
  this.pointY = point.pageY

  this.trigger('beforeScrollStart')
}
```

## `stop()`函数

```js
BScroll.prototype.stop = function () {
  if (this.options.useTransition && this.isInTransition) {
    this.isInTransition = false
    cancelAnimationFrame(this.probeTimer)
    // 获取当前计算后的位置
    let pos = this.getComputedPosition()
    //根据当前位置，设置页面最终滚动位置。
    this._translate(pos.x, pos.y)
    if (this.options.wheel) {
      this.target = this.items[Math.round(-pos.y / this.itemHeight)]
    } else {
      this.trigger('scrollEnd', {
        x: this.x,
        y: this.y
      })
    }
    this.stopFromTransition = true
  } else if (!this.options.useTransition && this.isAnimating) {
    this.isAnimating = false
    cancelAnimationFrame(this.animateTimer)
    this.trigger('scrollEnd', {
      x: this.x,
      y: this.y
    })
    this.stopFromTransition = true
  }
}
```

如果没有处于惯性滚动过程中，这个方法什么都不会做（我们只考虑使用transition的情况）。

如果处于惯性滚动过程中，会获取元素当前位置，重新调用_translate()设置最终滚动位置为当前位置。为什么说是重新调用？因为由于过渡动画的存在，上一次的滚动还没有结束，即还没有滚动要目标位置，我们现在又重新设置目标位置为当前位置。同时调用stop()前调用了_transitionTime()设置过渡时间为0，所以过渡动画会马上结束。最终的效果就是，当我们手指接触屏幕的时候，原来的惯性滚动马上停止。

## `_move()`函数处理滑动中事件

当手指或鼠标在屏幕上滑动式，better-scroll 做了如下几件事：

1. 计算鼠标或者手指的滑动距离

2. 根据横向纵向滑动距离锁定一个滚动方向（前提freeScroll === false)

3. 根据滑动距离计算滚动距离，并执行滚动

4. 派发"scroll"事件

5. 判断手指滑动到视窗(visual viewport)边缘，执行_end()

**源码：**

```js
BScroll.prototype._move = function (e) {
  // 未开启滚动或者已销毁则退出
  if (!this.enabled || this.destroyed || eventType[e.type] !== this.initiated) {
    return
  }

  阻止默认事件
  if (this.options.preventDefault) {
    e.preventDefault()
  }
  // 阻止事件冒泡传播
  if (this.options.stopPropagation) {
    e.stopPropagation()
  }

  // 计算滑动距离差值
  let point = e.touches ? e.touches[0] : e
  let deltaX = point.pageX - this.pointX
  let deltaY = point.pageY - this.pointY

  // 获取当前滚动点在页面的坐标
  this.pointX = point.pageX
  this.pointY = point.pageY

  // 累加滚动距离
  this.distX += deltaX
  this.distY += deltaY

  // 计算滚动距离的绝对值
  let absDistX = Math.abs(this.distX)
  let absDistY = Math.abs(this.distY)

  // 获取滚动结束后的时间
  let timestamp = getNow()

  // 只有滑动超过一定时间和距离才会认为是一次有效的滑动
  // We need to move at least momentumLimitDistance pixels for the scrolling to initiate
  if (timestamp - this.endTime > this.options.momentumLimitTime && (absDistY < this.options.momentumLimitDistance && absDistX < this.options.momentumLimitDistance)) {
    return
  }

  // 1. 如果没有开启freeScroll（自由滚动），会根据手指滑动方向，锁定一个滚动方向
  // If you are scrolling in one direction lock the other
  if (!this.directionLocked && !this.options.freeScroll) {
    if (absDistX > absDistY + this.options.directionLockThreshold) {   // 水平滚动距离大于垂直滚动距离，设置滚动方向为水平
      this.directionLocked = 'h' // lock horizontally
    } else if (absDistY >= absDistX + this.options.directionLockThreshold) {    // 同上，设置滚动方向为垂直
      this.directionLocked = 'v' // lock vertically
    } else {
      this.directionLocked = 'n' // no lock     
    }
  }

  // 2. 如果滚动方向为水平方向
  if (this.directionLocked === 'h') {
    if (this.options.eventPassthrough === 'vertical') {   // 阻止垂直方向上的默认事件
      e.preventDefault()
    } else if (this.options.eventPassthrough === 'horizontal') {
      this.initiated = false
      return
    }
    deltaY = 0   // 设置垂直滚动距离差值为0
  } else if (this.directionLocked === 'v') {     // 2. 如果滚动方向为垂直方向
    if (this.options.eventPassthrough === 'horizontal') {   // 阻止水平方向上的默认事件
      e.preventDefault()
    } else if (this.options.eventPassthrough === 'vertical') {
      this.initiated = false
      return
    }
    deltaX = 0    // 设置水平滚动距离差值为0
  }

  // 3. 计算新的滚动坐标
  deltaX = this.hasHorizontalScroll ? deltaX : 0   // 判断是哪个方向的滚动并赋值
  deltaY = this.hasVerticalScroll ? deltaY : 0
  this.movingDirectionX = deltaX > 0 ? DIRECTION_RIGHT : deltaX < 0 ? DIRECTION_LEFT : 0   // 大于0为向右滚动，小于0为向左
  this.movingDirectionY = deltaY > 0 ? DIRECTION_DOWN : deltaY < 0 ? DIRECTION_UP : 0

  let newX = this.x + deltaX
  let newY = this.y + deltaY

  let top = false
  let bottom = false
  let left = false
  let right = false
  // 如果超出包裹父容器wrapper边界减速或者停止滚动
  // Slow down or stop if outside of the boundaries
  const bounce = this.options.bounce
  if (bounce !== false) {   // 如果允许反弹
    top = bounce.top === undefined ? true : bounce.top
    bottom = bounce.bottom === undefined ? true : bounce.bottom
    left = bounce.left === undefined ? true : bounce.left
    right = bounce.right === undefined ? true : bounce.right
  }
  if (newX > this.minScrollX || newX < this.maxScrollX) {
    if ((newX > this.minScrollX && left) || (newX < this.maxScrollX && right)) {
      newX = this.x + deltaX / 3    // 只要比实际滚动的距离小，都可以算阻尼。当超过边界时有一种阻力在回拉，这个距离比实际滑动的距离小
    } else {
      newX = newX > this.minScrollX ? this.minScrollX : this.maxScrollX
    }
  }
  if (newY > this.minScrollY || newY < this.maxScrollY) {
    if ((newY > this.minScrollY && top) || (newY < this.maxScrollY && bottom)) {
      newY = this.y + deltaY / 3   // 只要比实际滚动的距离小，都可以算阻尼。当超过边界时有一种阻力在回拉，这个距离比实际滑动的距离小
    } else {
      newY = newY > this.minScrollY ? this.minScrollY : this.maxScrollY
    }
  }

  // 如果已经停止滚动则继续滚动
  if (!this.moved) {
    this.moved = true
    this.trigger('scrollStart')
  }
   
  // 滚动到新的位置(因为delatX和deltaY大于0，会在边界处产生留白，类似于微信下拉时出现小程序列表一样)
  this._translate(newX, newY)

  // 4. 派发 scroll 事件
  if (timestamp - this.startTime > this.options.momentumLimitTime) {   // 结束时间减去开始时间的差值大于滚动的最小时间时视为一次有效滚动
    // 重置新的滚动起始时间、滚动起始坐标
    this.startTime = timestamp    
    this.startX = this.x
    this.startY = this.y

    if (this.options.probeType === PROBE_DEBOUNCE) {
      this.trigger('scroll', {
        x: this.x,
        y: this.y
      })
    }
  }

  if (this.options.probeType > PROBE_DEBOUNCE) {
    this.trigger('scroll', {
      x: this.x,
      y: this.y
    })
  }

  // 获取元素滚动条到元素左边和顶部的距离（兼容）
  let scrollLeft = document.documentElement.scrollLeft || window.pageXOffset || document.body.scrollLeft
  let scrollTop = document.documentElement.scrollTop || window.pageYOffset || document.body.scrollTop

  let pX = this.pointX - scrollLeft
  let pY = this.pointY - scrollTop
  
  // 如果滑动到4个视窗边缘则滑动结束
  if (pX > document.documentElement.clientWidth - this.options.momentumLimitDistance || pX < this.options.momentumLimitDistance || pY < this.options.momentumLimitDistance || pY > document.documentElement.clientHeight - this.options.momentumLimitDistance
  ) {
    this._end(e)
  }
}
```

`_end()`函数处理滑动结束事件

_end()方法主要做了哪些事情？

1. 如果滚动超出边界，重置滚动位置，并退出函数

2. 如果是一次轻拂操作，退出函数

3. 计算整个滑动过程的时间和距离

4. 判断是否满足惯性滚动条件，并计算惯性滚动距离和时间

5. 执行惯性滚动，并退出

**源码：**

```js
BScroll.prototype._end = function (e) {
  if (!this.enabled || this.destroyed || eventType[e.type] !== this.initiated) {
    return
  }
  this.initiated = false

  if (this.options.preventDefault && !preventDefaultException(e.target, this.options.preventDefaultException)) {
    e.preventDefault()
  }
  if (this.options.stopPropagation) {
    e.stopPropagation()
  }

  // 触发触控结束事件
  this.trigger('touchEnd', {
    x: this.x,
    y: this.y
  })

  this.isInTransition = false  // 不再滚动

  // ensures that the last position is rounded(四舍五入)
  let newX = Math.round(this.x)
  let newY = Math.round(this.y)

  let deltaX = newX - this.absStartX
  let deltaY = newY - this.absStartY
  this.directionX = deltaX > 0 ? DIRECTION_RIGHT : deltaX < 0 ? DIRECTION_LEFT : 0
  this.directionY = deltaY > 0 ? DIRECTION_DOWN : deltaY < 0 ? DIRECTION_UP : 0

  // 下拉刷新
  // if configure pull down refresh, check it first
  if (this.options.pullDownRefresh && this._checkPullDown()) {
    return
  }

  // 点击
  // check if it is a click operation
  if (this._checkClick(e)) {
    this.trigger('scrollCancel')
    return
  }
  
  // 超出边界时重置位置
  // reset if we are outside of the boundaries
  if (this.resetPosition(this.options.bounceTime, ease.bounce)) {
    return
  }

  // 滚动到新位置
  this._translate(newX, newY)

  this.endTime = getNow()
  let duration = this.endTime - this.startTime
  let absDistX = Math.abs(newX - this.startX)
  let absDistY = Math.abs(newY - this.startY)

  // flick（轻击）
  if (this._events.flick && duration < this.options.flickLimitTime && absDistX < this.options.flickLimitDistance && absDistY < this.options.flickLimitDistance) {
    this.trigger('flick')
    return
  }

  let time = 0
  // start momentum animation if needed
  if (this.options.momentum && duration < this.options.momentumLimitTime && (absDistY > this.options.momentumLimitDistance || absDistX > this.options.momentumLimitDistance)) {
    let top = false
    let bottom = false
    let left = false
    let right = false
    const bounce = this.options.bounce
    if (bounce !== false) {
      top = bounce.top === undefined ? true : bounce.top
      bottom = bounce.bottom === undefined ? true : bounce.bottom
      left = bounce.left === undefined ? true : bounce.left
      right = bounce.right === undefined ? true : bounce.right
    }
    const wrapperWidth = ((this.directionX === DIRECTION_RIGHT && left) || (this.directionX === DIRECTION_LEFT && right)) ? this.wrapperWidth : 0
    const wrapperHeight = ((this.directionY === DIRECTION_DOWN && top) || (this.directionY === DIRECTION_UP && bottom)) ? this.wrapperHeight : 0
    let momentumX = this.hasHorizontalScroll ? momentum(this.x, this.startX, duration, this.maxScrollX, this.minScrollX, wrapperWidth, this.options)
      : {destination: newX, duration: 0}
    let momentumY = this.hasVerticalScroll ? momentum(this.y, this.startY, duration, this.maxScrollY, this.minScrollY, wrapperHeight, this.options)
      : {destination: newY, duration: 0}
    newX = momentumX.destination
    newY = momentumY.destination
    time = Math.max(momentumX.duration, momentumY.duration)
    this.isInTransition = true
  } else {
    if (this.options.wheel) {
      newY = Math.round(newY / this.itemHeight) * this.itemHeight
      time = this.options.wheel.adjustTime || 400
    }
  }

  let easing = ease.swipe
  if (this.options.snap) {
    let snap = this._nearestSnap(newX, newY)
    this.currentPage = snap
    time = this.options.snapSpeed || Math.max(
      Math.max(
        Math.min(Math.abs(newX - snap.x), 1000),
        Math.min(Math.abs(newY - snap.y), 1000)
      ), 300)
    newX = snap.x
    newY = snap.y

    this.directionX = 0
    this.directionY = 0
    easing = this.options.snap.easing || ease.bounce
  }

  if (newX !== this.x || newY !== this.y) {
    // change easing function when scroller goes out of the boundaries
    if (newX > this.minScrollX || newX < this.maxScrollX || newY > this.minScrollY || newY < this.maxScrollY) {
      easing = ease.swipeBounce
    }
    this.scrollTo(newX, newY, time, easing)
    return
  }

  if (this.options.wheel) {
    this.selectedIndex = Math.round(Math.abs(this.y / this.itemHeight))
  }
  this.trigger('scrollEnd', {
    x: this.x,
    y: this.y
  })
}
```
