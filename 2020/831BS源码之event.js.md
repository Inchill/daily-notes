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

`_events[type]` 是一个二维数组，如果没有注册该类型，将会初始化一个空数组，然后存储一个数组，存储的这个数组包含回调函数、上下文。可以看出，同一个事件类型允许注册多个回调函数，该函数并没有做限制，只要你注册了新的事件，哪怕类型不变，还是会往数组里 `push`。`vue` 里其实也是可以这样的，每一个事件用分号隔开即可。

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

先给回调函数 `magic` 里的 `fn` 赋值，然后注册事件。当事件被触发后，会执行 `magic` 这个回调函数。当 `magic` 回调函数执行时，第一步就是先移除掉注册的事件；第二步就是执行真正的回调函数 `fn`。

这里比较巧妙的一点就是，注册回调函数时，并没有真正地把传递进来的回调函数当作注册时的回调函数，而是内部自定义一个回调函数，在这个自定义回调函数里调用传进来的回调函数。之所以套了一个自定义回调函数，目的是为了移除注册的事件，保证传进来的回调函数只执行一次。

### 移除事件函数

```js
BScroll.prototype.off = function (type, fn) {
  let _events = this._events[type]
  if (!_events) {
    return
  }

  let count = _events.length
  while (count--) {
    if (_events[count][0] === fn || (_events[count][0] && _events[count][0].fn === fn)) {
      _events[count][0] = undefined
    }
  }
}
```

`_events` 是一个二维数组，先通过 `type` 下标获取存储的数组 `_events`，由注册事件函数知，该数组存储的是回调函数和上下文。数组的第一项存储的是回调函数，因此判断第一项是否是 `fn` 并且和传进来的回调函数一致，如果是对应的回调函数就将改值置为 `undefined`。

### 触发事件函数

```js
BScroll.prototype.trigger = function (type) {
  let events = this._events[type]
  if (!events) {
    return
  }

  let len = events.length
  let eventsCopy = [...events]
  for (let i = 0; i < len; i++) {
    let event = eventsCopy[i]
    let [fn, context] = event
    if (fn) {
      fn.apply(context, [].slice.call(arguments, 1))
    }
  }
}
```

该函数做了如下操作：

1. 获取该类型的注册事件数组，如果没有直接 `return`；

2. 拷贝一份注册事件数组并进行遍历，如果当前项有回调函数则执行。
