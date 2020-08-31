## `event.js` 文件

该文件在 `better-scroll` 原型上绑定了四个函数，分别是注册事件函数、只执行一次的注册事件函数、移除事件函数以及触发事件函数。

### 注册事件函数

```js
BScroll.prototype.on = function (type, fn, context = this) {
  if (!this._events[type]) {
    this._events[type] = []
  }

  this._events[type].push([fn, context])
}
```

`_events` 是一个二维数组，如果没有注册该类型，将会初始化一个空数组，然后存储一个数组，存储的这个数组包含回调函数、上下文。

### 只执行一次的注册事件函数

```js
BScroll.prototype.once = function (type, fn, context = this) {
  function magic() {
    this.off(type, magic)

    fn.apply(context, arguments)
  }
  // To expose the corresponding function method in order to execute the off method
  magic.fn = fn

  this.on(type, magic)
}
```

先给回调函数 `magic` 里的 `fn` 赋值，然后注册事件。当事件被触发后，会执行 `magic` 这个回调函数。当 `magic` 回调函数执行时，第一步就是先移除掉注册的事件，保证只执行一次；第二步就是执行真正的回调函数 `fn`。

这里比较巧妙的一点就是，注册回调函数时，并没有真正地把传递进来的回调函数当作注册时的回调函数，而是内部自定义一个回调函数，在这个自定义回调函数里调用传进来的回调函数。之所以套了一个自定义回调函数，目的是为了移除注册的事件，保
