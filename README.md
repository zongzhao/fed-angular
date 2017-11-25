# Vuex

> 如果你是Vue初学者或者你的应用程序很简单，建议不要过早的应用它。
> 注意：namespaced 仅2.4.0+支持

![](asset/vuex.png)

通过 vuex 入口文件，我们可以发现，对外暴露了Store类、install方法、mapState辅助函数、mapMutations辅助函数、mapGetters辅助函数、mapActions辅助函数、createNamespacedHelpers辅助函数以及当前的版本号version

```javascript
import { Store, install } from './store'
import { mapState, mapMutations, mapGetters, mapActions, createNamespacedHelpers } from './helpers'

export default {
  Store,
  install,
  version: '__VERSION__',
  mapState,
  mapMutations,
  mapGetters,
  mapActions,
  createNamespacedHelpers
}
```

## Store

## 辅助函数

> 在讲辅助函数以前，先了解下这两个基础函数 normalizeNamespace 和 normalizeMap
> 都支持两个参数传入 namespace 和 map 两个参数传入

```javascript
function normalizeMap (map) {
  return Array.isArray(map)
    ? map.map(key => ({ key, val: key }))
    : Object.keys(map).map(key => ({ key, val: map[key] }))
}
```
normalizeMap 函数接受一个对象或者数组，最后都转化成一个数组形式，数组元素是包含 key 和 value 两个属性的对象。


```javascript
function normalizeNamespace (fn) {
  return (namespace, map) => {
    if (typeof namespace !== 'string') {
      map = namespace
      namespace = ''
    } else if (namespace.charAt(namespace.length - 1) !== '/') {
      namespace += '/'
    }
    return fn(namespace, map)
  }
}
```
normalizeMap 函数接受一个回调函数fn，fn函数接受两个参数，最后执行fn并得到其返回结果。


### mapState
```javascript
export const mapState = normalizeNamespace((namespace, states) => {
  const res = {}
  normalizeMap(states).forEach(({ key, val }) => {
    res[key] = function mappedState () {
      let state = this.$store.state
      let getters = this.$store.getters
      if (namespace) {
        const module = getModuleByNamespace(this.$store, 'mapState', namespace)
        if (!module) {
          return
        }
        state = module.context.state
        getters = module.context.getters
      }
      return typeof val === 'function'
        ? val.call(this, state, getters)  // 如果是函数，返回函数执行后的结果
        : state[val]  // 如果不是函数，而是一个字符串，直接在state中读取。
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})
```

mapState(namespace?: string, map: Array<string> | Object): Object
函数执行的结果是返回一个对象，属性名对应于传入的 states 对象或者数组元素。属性值是字符串或者函数，返回相应的 state

```javascript
// 对象
computed: mapState({
  // 传字符串参数 'count' 等同于 `state => state.count`
  countAlias: 'count',

  // 箭头函数可使代码更简练
  count: state => state.count,

  // 为了能够使用 `this` 获取局部状态，必须使用常规函数
  countPlusLocalState (state) {
    return state.count + this.localCount
  }
})

// 数组
computed: mapState(['countAlias', 'count'])
```

### mapGetters
mapGetters(namespace?: string, map: Array<string> | Object): Object
从 store 中的 state 中派生出一些状态，Vuex 允许我们在 store 中定义“getter”（可以认为是 store 的计算属性）。就像计算属性一样，getter 的返回值会根据它的依赖被缓存起来，且只有当它的依赖值发生了改变才会被重新计算。

- 源码解析
```javascript
export const mapGetters = normalizeNamespace((namespace, getters) => {
  const res = {}
  normalizeMap(getters).forEach(({ key, val }) => {
    val = namespace + val
    res[key] = function mappedGetter () {
      if (namespace && !getModuleByNamespace(this.$store, 'mapGetters', namespace)) {
        return
      }
      // 如果在getters中不存在，报错
      if (process.env.NODE_ENV !== 'production' && !(val in this.$store.getters)) {
        console.error(`[vuex] unknown getter: ${val}`)
        return
      }
      // 根据 val 在 getters 对象里找对应的属性值
      return this.$store.getters[val]
    }
    // mark vuex getter for devtools
    res[key].vuex = true
  })
  return res
})
```

- 使用方式
```javascript
getters: {
  // state 作为其第一个参数
  doneTodos: state => {
    return state.todos.filter(todo => todo.done)
  },
  // 其他 getter 作为第二个参数
  doneTodosCount: (state, getters) => {
    return getters.doneTodos.length
  },
  // 通过让 getter 返回一个函数，来实现给 getter 传参
  getTodoById: (state, getters) => (id) => {
    return state.todos.find(todo => todo.id === id)
  }
}
// 数组
computed: mapGetters([
  'doneTodosCount',
  'anotherGetter',
  // ...
])
// 对象
computed: mapGetters({
  // 映射 `this.doneCount` 为 `store.getters.doneTodosCount`
  doneCount: 'doneTodosCount'
})
```

### mapMutations
mapMutations(namespace?: string, map: Array<string> | Object): Object
修改 state 的唯一方法是提交 mutation，mutation 都有一个字符串的 事件类型 (type) 和 一个 回调函数 (handler)

- 源码
```javascript
export const mapMutations = normalizeNamespace((namespace, mutations) => {
  const res = {}
  normalizeMap(mutations).forEach(({ key, val }) => {
    res[key] = function mappedMutation (...args) {
      let commit = this.$store.commit
      if (namespace) {
        const module = getModuleByNamespace(this.$store, 'mapMutations', namespace)
        if (!module) {
          return
        }
        commit = module.context.commit
      }
      return typeof val === 'function'
        ? val.apply(this, [commit].concat(args))
        : commit.apply(this.$store, [val].concat(args))
    }
  })
  return res
})
```

- 使用方式
```javascript
mutations: {
  increment (state) {
    // 变更状态
    state.count++
  },
  // 提交载荷
  increment1 (state, n) {
    state.count += n
  },
  // 在大多数情况下，载荷应该是一个对象，这样可以包含多个字段并且记录的 mutation 会更易读
  increment1 (state, payload) {
    state.count += payload.amount
  }
}

你可以在组件中使用 this.$store.commit('xxx') 提交 mutation，或者使用 mapMutations 辅助函数将组件中的 methods 映射为 store.commit 调用（需要在根节点注入 store）。
methods: {
  ...mapMutations([
    'increment', // 将 `this.increment()` 映射为 `this.$store.commit('increment')`

    // `mapMutations` 也支持载荷：
    'incrementBy' // 将 `this.incrementBy(amount)` 映射为 `this.$store.commit('incrementBy', amount)`
  ]),
  ...mapMutations({
    add: 'increment' // 将 `this.add()` 映射为 `this.$store.commit('increment')`
  })
}
```


### mapActions
mapActions(namespace?: string, map: Array<string> | Object): Object
Action 提交的是 mutation，而不是直接变更状态。
Action 可以包含任意异步操作。

- 源码
```javascript
export const mapActions = normalizeNamespace((namespace, actions) => {
  const res = {}
  normalizeMap(actions).forEach(({ key, val }) => {
    res[key] = function mappedAction (...args) {
      let dispatch = this.$store.dispatch
      if (namespace) {
        const module = getModuleByNamespace(this.$store, 'mapActions', namespace)
        if (!module) {
          return
        }
        dispatch = module.context.dispatch
      }
      return typeof val === 'function'
        ? val.apply(this, [dispatch].concat(args))
        : dispatch.apply(this.$store, [val].concat(args))
    }
  })
  return res
})
```
- 使用方式
```JavaScript
actions: {
  increment (context) {
    context.commit('increment')
  },
  async actionA ({ commit }) {
    commit('gotData', await getData())
  },
  // async / await 这个 JavaScript 即将到来的新特性，我们可以像这样组合 action
  async actionB ({ dispatch, commit }) {
    await dispatch('actionA') // 等待 actionA 完成
    commit('gotOtherData', await getOtherData())
  }
}

methods: {
  ...mapActions([
    'increment', // 将 `this.increment()` 映射为 `this.$store.dispatch('increment')`

    // `mapActions` 也支持载荷：
    'incrementBy' // 将 `this.incrementBy(amount)` 映射为 `this.$store.dispatch('incrementBy', amount)`
  ]),
  ...mapActions({
    add: 'increment' // 将 `this.add()` 映射为 `this.$store.dispatch('increment')`
  })
}
```

## 命名空间
