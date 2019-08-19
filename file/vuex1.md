
### 一. vuex

先来看看vuex的几个核心概念。

#### 1. State

Vuex 使用单一状态树——是的，用一个对象就包含了全部的应用层级状态。至此它便作为一个“唯一数据源 (SSOT)”而存在。这也意味着，
每个应用将仅仅包含一个 store 实例。单一状态树让我们能够直接地定位任一特定的状态片段，在调试的过程中也能轻易地取得整个当前应用状态的快照。

```
// 创建一个 Counter 组件
const Counter = {
  template: `<div>{{ count }}</div>`,
  computed: {
    count () {
      return store.state.count
    }
  }
}
```

#### 2. Getter

有时候我们需要从 store 中的 state 中派生出一些状态，这时候就需要用到Getter了。

声明：

```
const store = new Vuex.Store({
  state: {
    todos: [
      { id: 1, text: '...', done: true },
      { id: 2, text: '...', done: false }
    ]
  },
  getters: {
    doneTodos: state => {
      return state.todos.filter(todo => todo.done)
    },

    doneTodosCount: (state, getters) => {
        return getters.doneTodos.length
    },

    getTodoById: (state) => (id) => {
        return state.todos.find(todo => todo.id === id)
      }
  }
})
```


访问：

```
store.getters.doneTodos // -> [{ id: 1, text: '...', done: true }]

store.getters.doneTodosCount // -> 1

store.getters.getTodoById(2) // -> { id: 2, text: '...', done: false }
```

#### 3. Mutation

```
const store = new Vuex.Store({
  state: {
    count: 1
  },
  mutations: {
    increment (state) {
      // 变更状态
      state.count++
    }
  }
})

//使用
store.commit('increment', 10)
```

#### 4. Action
类似于 mutation，不同在于：

Action 提交的是 mutation，而不是直接变更状态。
Action 可以包含任意异步操作。

```
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

//分发
store.dispatch('increment')
```

异步：
```
actions: {
  actionA ({ commit }) {
    return new Promise((resolve, reject) => {
      setTimeout(() => {
        commit('someMutation')
        resolve()
      }, 1000)
    })
  },

  async actionB ({ dispatch, commit }) {
    await dispatch('actionA') // 等待 actionA 完成
    commit('gotOtherData', await getOtherData())
  }
}

```

### 二. vuex内部原理

Vuex 全局维护着一个对象，使用到了单例设计模式。在这个全局对象中，所有属性都是响应式的，任意属性进行了改变，都会造成使用到该属性的组件进行更新。
并且只能通过 commit 的方式改变状态，实现了单向数据流模式。

1. Vue.use(Vuex);

调用这段代码的时候会直接调用Vuex的install方法。

install方法

```
/*暴露给外部的插件install方法，供Vue.use调用安装插件*/
export function install (_Vue) {
  if (Vue) {
    /*避免重复安装（Vue.use内部也会检测一次是否重复安装同一个插件）*/
    if (process.env.NODE_ENV !== 'production') {
      console.error(
        '[vuex] already installed. Vue.use(Vuex) should be called only once.'
      )
    }
    return
  }
  /*保存Vue，同时用于检测是否重复安装*/
  Vue = _Vue
  /*将vuexInit混淆进Vue的beforeCreate(Vue2.0)或_init方法(Vue1.0)*/
  applyMixin(Vue)
}

```
install方法会避免重复安装Vuex，再来看看applyMixin方法：

2. vuexInit

```
export default function (Vue) {
  const version = Number(Vue.version.split('.')[0])

  if (version >= 2) {
    const usesInit = Vue.config._lifecycleHooks.indexOf('init') > -1
    Vue.mixin(usesInit ? { init: vuexInit } : { beforeCreate: vuexInit })
  } else {
    // override init and inject vuex init procedure
    // for 1.x backwards compatibility.
    const _init = Vue.prototype._init
    Vue.prototype._init = function (options = {}) {
      options.init = options.init
        ? [vuexInit].concat(options.init)
        : vuexInit
      _init.call(this, options)
    }
  }

  /**
   * Vuex init hook, injected into each instances init hooks list.
   */

  function vuexInit () {
    const options = this.$options
    // store injection
    if (options.store) {
      this.$store = options.store
    } else if (options.parent && options.parent.$store) {
      this.$store = options.parent.$store
    }
  }
}
```

方法判断了一下当前vue的版本，当vue版本>=2的时候，就在Vue上添加了一个全局mixin，要么在init阶段，要么在beforeCreate阶段。
Vue上添加的全局mixin会影响到每一个组件。mixin的各种混入方式不同，同名钩子函数将混合为一个数组，因此都将被调用。
并且，混合对象的钩子将在组件自身钩子之前。

vuexInit方法中，当options中有store，则直接可以通过this.$store来引用，如果没有的话，并且实例有parent，就把parent的$store
挂载到当前实例上，这样，我们在Vue的组件中就可以通过this.$store.xxx访问Vuex的各种数据和状态了。

3. Store构造函数

如下代码，我们调用了Store的构造函数。

```
export default new Vuex.Store({
    state,
      mutations
      actions,
      getters,
      modules: {
        ...
      },
      plugins,
      strict: false
});
```

Store构造函数的代码如下：

```
constructor (options = {}) {
    // Auto install if it is not done yet and `window` has `Vue`.
    // To allow users to avoid auto-installation in some cases,
    // this code should be placed here. See #731
    /*
      在浏览器环境下，如果插件还未安装（!Vue即判断是否未安装），则它会自动安装。
      它允许用户在某些情况下避免自动安装。
    */
    if (!Vue && typeof window !== 'undefined' && window.Vue) {
      install(window.Vue)
    }

    if (process.env.NODE_ENV !== 'production') {
      assert(Vue, `must call Vue.use(Vuex) before creating a store instance.`)
      assert(typeof Promise !== 'undefined', `vuex requires a Promise polyfill in this browser.`)
      assert(this instanceof Store, `Store must be called with the new operator.`)
    }

    const {
      /*一个数组，包含应用在 store 上的插件方法。这些插件直接接收 store 作为唯一参数，可以监听 mutation（用于外部地数据持久化、记录或调试）或者提交 mutation （用于内部数据，例如 websocket 或 某些观察者）*/
      plugins = [],
      /*使 Vuex store 进入严格模式，在严格模式下，任何 mutation 处理函数以外修改 Vuex state 都会抛出错误。*/
      strict = false
    } = options

    /*从option中取出state，如果state是function则执行，最终得到一个对象*/
    let {
      state = {}
    } = options
    if (typeof state === 'function') {
      state = state()
    }

    // store internal state
    /* 用来判断严格模式下是否是用mutation修改state的 */
    this._committing = false
    /* 存放action */
    this._actions = Object.create(null)
    /* 存放mutation */
    this._mutations = Object.create(null)
    /* 存放getter */
    this._wrappedGetters = Object.create(null)
    /* module收集器 */
    this._modules = new ModuleCollection(options)
    /* 根据namespace存放module */
    this._modulesNamespaceMap = Object.create(null)
    /* 存放订阅者 */
    this._subscribers = []
    /* 用以实现Watch的Vue实例 */
    this._watcherVM = new Vue()

    // bind commit and dispatch to self
    /*将dispatch与commit调用的this绑定为store对象本身，否则在组件内部this.dispatch时的this会指向组件的vm*/
    const store = this
    const { dispatch, commit } = this
    /* 为dispatch与commit绑定this（Store实例本身） */
    this.dispatch = function boundDispatch (type, payload) {
      return dispatch.call(store, type, payload)
    }
    this.commit = function boundCommit (type, payload, options) {
      return commit.call(store, type, payload, options)
    }

    // strict mode
    /*严格模式(使 Vuex store 进入严格模式，在严格模式下，任何 mutation 处理函数以外修改 Vuex state 都会抛出错误)*/
    this.strict = strict

    // init root module.
    // this also recursively registers all sub-modules
    // and collects all module getters inside this._wrappedGetters
    /*初始化根module，这也同时递归注册了所有子modle，收集所有module的getter到_wrappedGetters中去，this._modules.root代表根module才独有保存的Module对象*/
    installModule(this, state, [], this._modules.root)

    // initialize the store vm, which is responsible for the reactivity
    // (also registers _wrappedGetters as computed properties)
    /* 通过vm重设store，新建Vue对象使用Vue内部的响应式实现注册state以及computed */
    resetStoreVM(this, state)

    // apply plugins
    /* 调用插件 */
    plugins.forEach(plugin => plugin(this))

    /* devtool插件 */
    if (Vue.config.devtools) {
      devtoolPlugin(this)
    }
  }
```

this._committing 表示提交状态，作用是保证对 Vuex 中 state 的修改只能在 mutation 的回调函数中，而不能在外部随意修改state。
this._actions 用来存放用户定义的所有的 actions。this._mutations 用来存放用户定义所有的 mutatins。this._wrappedGetters 用来存放用户定义的所有 getters。
this._modules 用来存储用户定义的所有modulesthis._modulesNamespaceMap 存放module和其namespace的对应关系。
this._subscribers 用来存储所有对 mutation 变化的订阅者。
this._watcherVM 是一个 Vue 对象的实例，主要是利用 Vue 实例方法 $watch 来观测变化的。这些参数后面会用到，我们再一一展开。

代码里面也有很多注释，方便理解。

核心的方法是 installModule。

由于Vuex是单例模式，只有一个对象存放所有的状态，所以增加了一个模块的概念，一个单例下面可以是状态等等，也可以是各个模块，每个模块有自己的状态。
调用installModule代码如下：

```
// init root module.
// this also recursively registers all sub-modules
// and collects all module getters inside this._wrappedGetters
installModule(this, state, [], this._modules.root)
```

再看看_modules的代码：

```
this._modules = new ModuleCollection(options)
```

我们再来看看 ModuleCollection 类，代码如下：

```
export default class ModuleCollection {
  constructor (rawRootModule) {
    // register root module (Vuex.Store options)
    this.root = new Module(rawRootModule, false)

    // register all nested modules
    if (rawRootModule.modules) {
      forEachValue(rawRootModule.modules, (rawModule, key) => {
        this.register([key], rawModule, false)
      })
    }
  }
  register (path, rawModule, runtime = true) {
    const parent = this.get(path.slice(0, -1))
    const newModule = new Module(rawModule, runtime)
    parent.addChild(path[path.length - 1], newModule)

    // register nested modules
    if (rawModule.modules) {
      forEachValue(rawModule.modules, (rawChildModule, key) => {
        this.register(path.concat(key), rawChildModule, runtime)
      })
    }
  }

  get (path) {
    return path.reduce((module, key) => {
      return module.getChild(key)
    }, this.root)
  }

  addChild (key, module) {
    this._children[key] = module
  }
}

export default class Module {
  constructor (rawModule, runtime) {
    this.runtime = runtime
    this._children = Object.create(null)
    this._rawModule = rawModule
    const rawState = rawModule.state
    this.state = (typeof rawState === 'function' ? rawState() : rawState) || {}
  }
  ...
}
```

构造方法中先定义了实例的root属性，为一个Modlue实例，然后遍历了options离的modules，再依次注册。

get方法的入参path为一个数组，例如['subModule', 'subsubModule'], 这里使用reduce方法，一层一层的取值, this.get(path.slice(0, -1))
取到当前module的父module。然后再调用Module类的addChild方法，将改module添加到父module的_children对象上。

再来看看installModule方法：

```
/*初始化module*/
function installModule (store, rootState, path, module, hot) {
  /* 是否是根module */
  const isRoot = !path.length
  /* 获取module的namespace */
  const namespace = store._modules.getNamespace(path)

  // register in namespace map
  /* 如果有namespace则在_modulesNamespaceMap中注册 */
  if (module.namespaced) {
    store._modulesNamespaceMap[namespace] = module
  }

  // set state
  if (!isRoot && !hot) {
    /* 获取父级的state */
    const parentState = getNestedState(rootState, path.slice(0, -1))
    /* module的name */
    const moduleName = path[path.length - 1]
    store._withCommit(() => {
      /* 将子module设置称响应式的 */
      Vue.set(parentState, moduleName, module.state)
    })
  }

  const local = module.context = makeLocalContext(store, namespace, path)

  /* 遍历注册mutation */
  module.forEachMutation((mutation, key) => {
    const namespacedType = namespace + key
    registerMutation(store, namespacedType, mutation, local)
  })

  /* 遍历注册action */
  module.forEachAction((action, key) => {
    const namespacedType = namespace + key
    registerAction(store, namespacedType, action, local)
  })

  /* 遍历注册getter */
  module.forEachGetter((getter, key) => {
    const namespacedType = namespace + key
    registerGetter(store, namespacedType, getter, local)
  })

  /* 递归安装mudule */
  module.forEachChild((child, key) => {
    installModule(store, rootState, path.concat(key), child, hot)
  })
}
```

我们先看看

```
getNamespace (path) {
    let module = this.root
    return path.reduce((namespace, key) => {
         module = module.getChild(key)
         return namespace + (module.namespaced ? key + '/' : '')
    }, '')
}
```

所以像下面这样定义的store，得到的selectLabelRule的namespace就是'selectLabelRule/'

```
export default new Vuex.Store({
  state,
  actions,
  getters,
  mutations,
  modules: {
    selectLabelRule
  },
  strict: debug
})
```

我们再来看看_withCommit方法：

```
_withCommit (fn) {
  const committing = this._committing
  this._committing = true
  fn()
  this._committing = committing
}
```

this._committing在Store的构造函数里声明过，初始值为false。这里由于我们是在修改 state，Vuex 中所有对 state 的修改都会用
_withCommit函数包装，保证在同步修改 state 的过程中 this._committing 的值始终为true。这样当我们观测 state 的变化时，
如果 this._committing 的值不为 true，则能检查到这个状态修改是有问题的。

我们发现，通过commit（mutation）修改state数据的时候，会再调用mutation方法之前将committing置为true，接下来再通过mutation
函数修改state中的数据，这时候触发$watch中的回调断言committing是不会抛出异常的（此时committing为true）。而当我们直接修改
state的数据时，触发$watch的回调执行断言，这时committing为false，则会抛出异常。

剩下来的下章继续吧。
