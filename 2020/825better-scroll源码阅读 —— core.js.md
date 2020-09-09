滚动相关的核心代码，主要是手指在屏幕上滑动开始、滑动中、滑动结束三个时间点 better-scroll 都做了哪些事情。理解了 core，再来阅读其他插件源码，就轻松多了。

在 `handleEvent()` 函数中，根据监听事件类型的不同，调用了不同的回调函数。

具体可查看初始化这个文件里怎么做的，[链接](https://github.com/Inchill/daily-notes/blob/master/2020/821Better-Scroll%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB.md)。

## `_start()`函数处理滚动开始事件

```js
BScroll.prototype._start = function (e) {
  let _eventType = eventType[e.type]
  if (_eventType !== TOUCH_EVENT) {    // PC端鼠标滚动，值为2
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
  // 滑动方向
  this.directionX = 0
  this.directionY = 0
  // 滑动中方向
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

  // 滚动的开始位置
  this.startX = this.x
  this.startY = this.y
  this.absStartX = this.x      // 注意在 _end 函数里会用到
  this.absStartY = this.y
  // 鼠标或手指相对于页面位置（不是可视区域里的文档部分，而是整个文档）
  this.pointX = point.pageX
  this.pointY = point.pageY

  this.trigger('beforeScrollStart')
}
```

## `stop()`函数

为什么在 `_start` 函数里会出现 `_stop` 函数呢？这是因为当滚动还没有停止时，用户又一次开始了新的滚动。这很好理解，比如我们在查看一个长列表时，可能会不停地滑动，从而触发这个 `_stop` 函数。

```js
BScroll.prototype.stop = function () {
  if (this.options.useTransition && this.isInTransition) {
    this.isInTransition = false
    cancelAnimationFrame(this.probeTimer)
    let pos = this.getComputedPosition()     // 获取当前计算后的位置
    this._translate(pos.x, pos.y)    // 根据当前位置，设置页面最终滚动位置。
    if (this.options.wheel) {      // picker相关，暂不关心
      // ...
    } else {
      this.trigger('scrollEnd', {     // 触发 scrollEnd 事件
        x: this.x,
        y: this.y
      })
    }
    this.stopFromTransition = true     // 惯性滚动被停止的标记，设置为true
  } else if (!this.options.useTransition && this.isAnimating) {    // 暂不关心
    // ...
  }
}
```

如果没有处于惯性滚动过程中，这个方法什么都不会做（我们只考虑使用transition的情况）。

如果处于惯性滚动过程中，会获取元素当前位置，重新调用_translate()设置最终滚动位置为当前位置。为什么说是重新调用？因为由于过渡动画的存在，上一次的滚动还没有结束，即还没有滚动要目标位置，我们现在又重新设置目标位置为当前位置。同时调用stop()前调用了_transitionTime()设置过渡时间为0，所以过渡动画会马上结束。最终的效果就是，当我们手指接触屏幕的时候，原来的惯性滚动马上停止。

## `_move()`函数处理滚动中事件

当手指或鼠标在屏幕上滚动时，better-scroll 做了如下几件事：

1. 计算鼠标或者手指的滚动距离

2. 根据横向纵向滚动距离锁定一个滚动方向（前提freeScroll === false)

3. 根据滚动距离计算滚动距离，并执行滚动

4. 派发"scroll"事件

5. 判断手指滚动到视窗(visual viewport)边缘，执行_end()

**源码：**

```js
BScroll.prototype._move = function (e) {
  ... // 是否开启滚动，是否已销毁的判断，阻止浏览器默认行为操作

  /************1. 下面部分计算滚动距离************/
  let point = e.touches ? e.touches[0] : e
  let deltaX = point.pageX - this.pointX     // 计算滚动距离差值
  let deltaY = point.pageY - this.pointY

  this.pointX = point.pageX     // 缓存本次手指/鼠标坐标，方便下一次使用
  this.pointY = point.pageY

  this.distX += deltaX      // 缓存累加后的总共滚动距离，因为方向不同可能为正可能为负
  this.distY += deltaY

  let absDistX = Math.abs(this.distX)    // 这里取绝对值只是为了方便数值比较
  let absDistY = Math.abs(this.distY)
  /************上面部分计算滑动距离************/
  
  let timestamp = getNow()   // 获取滚动结束后的时间

  // 只有滚动超过一定时间和距离才会认为是一次有效的滚动
  if (timestamp - this.endTime > this.options.momentumLimitTime && (absDistY < this.options.momentumLimitDistance && absDistX < this.options.momentumLimitDistance)) {
    return
  }

  /****************2. 下面部分锁定滚动方向******************************/
  // 如果没有开启freeScroll（自由滚动），会根据手指滑动方向，锁定一个滚动方向
  if (!this.directionLocked && !this.options.freeScroll) {
    if (absDistX > absDistY + this.options.directionLockThreshold) {    // 水平方向绝对滚动距离 > 垂直方向绝对滚动距离 + 用户自定义的滚动阈值距离（用户设置距离阈值为一次有效滚动）
      this.directionLocked = 'h'    // 水平滚动距离大于垂直滚动距离，设置滚动方向为水平
    } else if (absDistY >= absDistX + this.options.directionLockThreshold) {
      this.directionLocked = 'v'    // 同上，设置滚动方向为垂直
    } else {
      this.directionLocked = 'n'    // 不锁定方向    
    }
  }

  if (this.directionLocked === 'h') {     // 1. 如果滚动方向为水平方向
    if (this.options.eventPassthrough === 'vertical') {   
      e.preventDefault()     // 取消滚动锁定方向（垂直方向）的浏览器默认行为
    } else if (this.options.eventPassthrough === 'horizontal') {
      this.initiated = false    // 如果滚动锁定方向和eventPassthrough方向相同，不允许滚动，并直接退出
      return
    }
    deltaY = 0   // 滚动锁定的另一个方向，手指垂直滑动距离重置为0
  } else if (this.directionLocked === 'v') {     // 2. 如果滚动方向为垂直方向
    if (this.options.eventPassthrough === 'horizontal') {   
      e.preventDefault()     // 保留水平方向的原生滚动，去除浏览器默认事件
    } else if (this.options.eventPassthrough === 'vertical') {
      this.initiated = false      // 如果滚动锁定方向和eventPassthrough方向相同，不允许滚动，并直接退出
      return
    }
    deltaX = 0    // 滚动锁定的另一个方向，手指水平滑动距离重置为0
  }
  /****************上面部分锁定滚动方向******************************/

  /************3. 以下部分计算新滚动坐标**********/
  deltaX = this.hasHorizontalScroll ? deltaX : 0   
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
  const bounce = this.options.bounce
  if (bounce !== false) {   // 如果开启了反弹动画（当滚动超过边缘的时候）
    top = bounce.top === undefined ? true : bounce.top
    bottom = bounce.bottom === undefined ? true : bounce.bottom
    left = bounce.left === undefined ? true : bounce.left
    right = bounce.right === undefined ? true : bounce.right
  }
  if (newX > this.minScrollX || newX < this.maxScrollX) {    // newX 大于最小横向滚动值或者小于最大横向滚动值，此时会产生阻尼
    if ((newX > this.minScrollX && left) || (newX < this.maxScrollX && right)) {    // 如果左侧或者右侧开启了反弹动画
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

  if (!this.moved) {
    this.moved = true
    this.trigger('scrollStart')
  }
  
  this._translate(newX, newY)    // 滚动到新的位置
  /***********以上部分计算新的滚动坐标**********/

  /***********下面这部分负责滑动过程中派发"scroll"事件**********/
  if (timestamp - this.startTime > this.options.momentumLimitTime) {   // 结束时间 - 开始时间的差值 > 滚动的最小时间时，开启 momentum 动画
    this.startTime = timestamp    // 缓存开始时间，以便下一次使用
    this.startX = this.x     // 缓存开始坐标
    this.startY = this.y

    if (this.options.probeType === PROBE_DEBOUNCE) {      // 非实时派发 scroll 事件
      this.trigger('scroll', {
        x: this.x,
        y: this.y
      })
    }
  }

  if (this.options.probeType > PROBE_DEBOUNCE) {      // 实时派发 scroll 事件，当probeType = 3 时，不仅在屏幕滑动的过程中，而且在 momentum 滚动动画运行过程中实时派发 scroll 事件
    this.trigger('scroll', {
      x: this.x,
      y: this.y
    })
  }
  /***********上面这部分负责滑动过程中派发"scroll"事件**********/

  /**********下面是如果滑动到视窗的四个边缘执行滑动结束****************/
  let scrollLeft = document.documentElement.scrollLeft || window.pageXOffset || document.body.scrollLeft    // 获取元素滚动条到元素左边和顶部的距离（考虑了兼容）
  let scrollTop = document.documentElement.scrollTop || window.pageYOffset || document.body.scrollTop

  let pX = this.pointX - scrollLeft      // 手指或鼠标相对于页面的坐标（也就是clientX加上水平滚动条的距离，不是相对可视区页面来算的） - 滚动条向左卷去的距离 = 在可视区的坐标
  let pY = this.pointY - scrollTop      // 同上
  
  // 如果滑动到4个视窗边缘则滑动结束（只有在屏幕上快速滑动的距离大于 momentumLimitDistance，才能开启 momentum 动画。）
  if (pX > document.documentElement.clientWidth - this.options.momentumLimitDistance || pX < this.options.momentumLimitDistance || pY < this.options.momentumLimitDistance || pY > document.documentElement.clientHeight - this.options.momentumLimitDistance
  ) {
    this._end(e)
  }
  /**********上面是如果滑动到视窗的四个边缘执行滑动结束****************/
}
```

`_end()`函数处理滚动结束事件

_end()方法主要做了哪些事情？

1. 如果滚动超出边界，重置滚动位置，并退出函数

2. 如果是一次轻拂操作，退出函数

3. 计算整个滑动过程的时间和距离

4. 判断是否满足惯性滚动条件，并计算惯性滚动距离和时间

5. 执行惯性滚动，并退出

**源码：**

```js
BScroll.prototype._end = function (e) {
  // ... 是否开启滚动，是否已销毁的判断，阻止浏览器默认行为操作
  
  this.trigger('touchEnd', {
    x: this.x,
    y: this.y
  })

  this.isInTransition = false  // 设置过渡标志为false，停止滚动

  // ensures that the last position is rounded(四舍五入)
  let newX = Math.round(this.x)
  let newY = Math.round(this.y)

  let deltaX = newX - this.absStartX     // 在 _start 函数里初始化为 this.x
  let deltaY = newY - this.absStartY
  this.directionX = deltaX > 0 ? DIRECTION_RIGHT : deltaX < 0 ? DIRECTION_LEFT : 0
  this.directionY = deltaY > 0 ? DIRECTION_DOWN : deltaY < 0 ? DIRECTION_UP : 0

  // 下拉刷新，暂不关心
  if (this.options.pullDownRefresh && this._checkPullDown()) {
    return
  }

  // 点击事件，暂不关心
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

  /***************下面这部分主要是_end()方法核心********************/
  this.endTime = getNow()
  let duration = this.endTime - this.startTime
  let absDistX = Math.abs(newX - this.startX)      // 这里取绝对值用于后续比较判断是否是一次轻拂事件
  let absDistY = Math.abs(newY - this.startY)

  // flick（只有用户在屏幕上滑动的距离小于 flickLimitDistance ，才算一次轻拂）
  if (this._events.flick && duration < this.options.flickLimitTime && absDistX < this.options.flickLimitDistance && absDistY < this.options.flickLimitDistance) {
    this.trigger('flick')
    return
  }

  let time = 0
  // start momentum animation if needed（如果开启了 momentum 动画）
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
      // ... picker 相关，暂不关心
    }
  }

  let easing = ease.swipe
  if (this.options.snap) {
    // ... snap 配置，暂不关心
  }

  if (newX !== this.x || newY !== this.y) {
    // change easing function when scroller goes out of the boundaries
    if (newX > this.minScrollX || newX < this.maxScrollX || newY > this.minScrollY || newY < this.maxScrollY) {
      easing = ease.swipeBounce
    }
    this.scrollTo(newX, newY, time, easing)
    return
  }
  /***************上面这部分主要是_end()方法核心********************/

  if (this.options.wheel) {
    // ... picker 相关，暂不关心
  }
  this.trigger('scrollEnd', {
    x: this.x,
    y: this.y
  })
}
```
