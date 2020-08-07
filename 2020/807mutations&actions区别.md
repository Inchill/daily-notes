## mutations

mutations必须是同步函数，看个例子：

```js
mutations: {
  someMutation(state) {
    api.callAsyncMethod(() => {
      state.count++;
    })
  }
}
```

我们都知道任何回调函数中进行的状态改变都是无法追踪的,  devtools会对mutations的每一条提交做记录,记录上一次提交之前和提交之后的状态,在上面的例子中无法实现捕捉状态,因为在执行mutations时,内部回调函数还没有执行,
所以此时无法捕捉状态.在 Vuex 中，mutation 都是同步事务.

## actions

vuex肯定不甘示弱,为了解决mutations只有同步的问题,提出了actions(异步),专门用来解决mutations只有同步无异步的问题.

Action 类似于 mutation，不同在于：

- Action 提交的是 mutation，而不是直接变更状态。

- Action 可以包含任意异步操作。

让我们来注册一个简单的 action：

```js
const store = new Vuex.Store({
  state: {
    count: 0
  },
  mutations: {
    increment (state) {
      state.count++
    }
  },
  actions: {
    increment (context) {
      context.commit('increment')
    }
  }
})
```

Action 函数接受一个与 store 实例具有相同方法和属性的 context 对象，因此你可以调用 context.commit 提交一个 mutation，或者通过 context.state 和 context.getters 来获取 state 和 getters。

实践中，我们会经常用到 ES2015 的 参数解构 来简化代码（特别是我们需要调用 commit 很多次的时候）：

```js
actions: {
  increment ({ commit }) {
    commit('increment')
  }
}
```

### 分发actions

在使用中分发actions，然后在actions函数中使用异步操作：

```js
actions: {
  incrementAsync ({ commit }) {
    setTimeout(() => {
      commit('increment')
    }, 1000)
  }
}
```

Actions 支持同样的载荷方式和对象方式进行分发：

```js
// 以载荷形式分发
store.dispatch('incrementAsync', {
  amount: 10
})

// 以对象形式分发
store.dispatch({
  type: 'incrementAsync',
  amount: 10
})
```
