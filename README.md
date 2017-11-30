> 如果你是Vue初学者或者你的应用程序很简单，建议不要过早的应用它。
> 注意：namespaced 仅2.4.0+支持

![](asset/vuex.png)

Vuex 是一个专为 Vue.js 应用程序开发的状态管理模式（至于什么是状态管理模式我就不科普了）。它采用集中式存储管理应用的所有组件的状态，并以相应的规则保证状态以一种可预测的方式发生变化。


Vuex 对于我们怎么组织代码结构没有任何的限制，却强制规定了一系列高级的规则：
- `应用级`的状态集中放在store中。
- 改变状态的位置方式就是提交 mutations, 同步操作。
- 异步的逻辑封装在 action

通过 Vuex 入口文件，我们可以发现，对外暴露的有Store类、install方法、mapState辅助函数、mapMutations辅助函数、mapGetters辅助函数、mapActions辅助函数、createNamespacedHelpers辅助函数以及当前的版本号version。

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

## 基本用法
```javascript
import Vue from 'vue'
import Vuex from 'vuex'
Vue.use(Vuex)
const store = new Vuex.Store({
  state: {
  },
  actions: {
  },
  mutations: {
  },
  getters: {
  },  
  modules: {

  }
})
export default store
```
可以看出，Vuex Store 模板包含了5个关键属性：
- state 定义了应用状态的数据结构，同样可以在这里设置默认的初始状态。
- actions 定义提交触发更改信息的描述，常见的例子有从服务端获取数据，在数据获取完成后会调用store.commit()来调用更改 Store 中的状态。可以在组件中使用dispatch来发出 Actions。
- mutations 是唯一允许更新应用状态的地方。
- getters 允许组件从 Store 中获取数据，并做一定的处理再返回。
- modules 对象允许将单一的 Store 拆分为多个 Store 的同时保存在单一的状态树中。随着应用复杂度的增加，这种拆分能够更好地组织代码


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
normalizeMap这个方法判断了map书否是数组，如果是就将map中每一项转化为{ key: key, val: key }的对象，否则传入的 map 就是一个对象（因为mapstate传入的参数不是数组就是对象），那就调用Object.keys获取map的所有key值，然后key数组再次遍历转化为{ key: key, val: map[key] }这个对象，最后将这个对象数组作为normalizeMap的返回值。


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
这个函数接受一个方法作为参数，返回一个函数，这个函数也就mapState/mapGetters/mapMutations/mapActions。这个函数接收两个参数namespace和map，当namespace不传的时候就将第一个参数（映射的对象）赋给map，或者传入namespace就对其进行类型判断以及格式校验后，返回传入的fn的调用。


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
默认进来state和getters是全局仓库里面的，然后再去判断是否有namespace这个，如果有将对应的那个module中的state和getters覆盖掉全局的那个，最后判断val如果是函数，就直接调用这个函数，并且将前面定义好的state以及getters当做参数传入，如果不是函数则直接取值。
res[key].vuex = true;这行就是提供给插件使用的。

其他的辅助函数也都是类似的，有兴趣的可以深入研究一下。


### mapState

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

// 你可以在组件中使用 this.$store.commit('xxx') 提交 mutation，或者使用 mapMutations 辅助函数将组件中的 methods 映射为 store.commit 调用（需要在根节点注入 store）。
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

```JavaScript
actions: {
  increment (context) {
    context.commit('increment')
  },
  // context = {state, dispatch, commit，getters, rootState}
  actions({state, dispatch, commit，getters, rootState}) {
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

## 总结
### 父子组件通信
通过 this.$children 或者 this.$refs 访问子组件
通过 this.$parent 来访问父组件

### 避免全局操作
- 用 this.$el.querySelector 代替 document.querySelector，不去查询组件外的 DOM
- 由于我们可以在任何组件中通过 this.$store 访问/提交状态，所以我们可以单独定义一个模块，把所有修改全局状态的方法都定义在这里面，组件借助这个模块去修改 store 中的数据。

### 常量替代 Mutation 和 Actions 事件类型
// type.js
```javascript
// mutations
export const SET_APP_LIST = 'SET_APP_LIST'
// actions
export const GET_APP_LIST = 'GET_APP_LIST'

// 组件中使用
import * as types from './type.js'
methods： {
   ...mapMutations({
    update: types.SET_APP_LIST
  }),
  ...mapActions({
    add: types.GET_APP_LIST
  }),
  update(list) {
	this.$store.commit(types.SET_APP_LIST, list)
  }
}
```

### async await
```javascript
actions: {
  async actionA ({ commit }) {
    commit('gotData', await getData())
  },
  // async / await 这个 JavaScript 即将到来的新特性，我们可以像这样组合 action
  async actionB ({ dispatch, commit }) {
    await dispatch('actionA') // 等待 actionA 完成
    commit('gotOtherData', await getOtherData())
  }
}
// 组件内部
methods： {
	actions： async function(){
	   const result = await GetAppList()
       // do something
    }
}
```

### 表单处理
```html
<input v-model="message">
<input type="checkbox" v-model="isVisible">
```

```javascript
// ...
computed: {
  message: {
    get () {
      return this.$store.state.obj.message
    },
    set (value) {
      this.$store.commit('updateMessage', value)
    }
  },

  isVisible: {
	get() {
      return this.$store.filter.params.length ? true : false
    },
    set(value) {
      const params = value ? [1,2,3] : []
      this.$store.commit('updateParams', params)
    }
  }
}
```
