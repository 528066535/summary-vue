
    完整的项目地址可以访问我的[github](https://github.com/528066535/vue-project-ssr)
    
## 一. 构建流程

### 构建图

!![构建图](/img/ssr2-1.png)

 图中分三大块，Source表示我们写的代码，会被webpack打包成两份，其中一份（Server Bundle）在Node Server中使用，用来生成第一个HTML并返回给浏览器。
 Browser中拿到的是webpack打包的Client Bundle 用来接管页面剩余的逻辑。
 
 我而我们要写的地方，主要在Source部分，和Webpack打包部分。
 
### app.js
 
app.js 是我们应用程序的「通用 entry」。在纯客户端应用程序中，我们将在此文件中创建根 Vue 实例，并直接挂载到 DOM。但是，对于服务器端渲染(SSR)，
责任转移到纯客户端 entry 文件。app.js 简单地使用 export 导出一个 createApp 函数：
 
 
    import Vue from 'vue'
    import App from './App.vue'

    // 导出一个工厂函数，用于创建新的
    // 应用程序、router 和 store 实例
    export function createApp () {
    const app = new Vue({
        // 根实例简单的渲染应用程序组件。
        render: h => h(App)
    })
    return { app }
    }

### entry-client.js

客户端 entry 只需创建应用程序，并且将其挂载到 DOM 中：

    import { createApp } from './app'

    // 客户端特定引导逻辑……

    const { app } = createApp()

    // 这里假定 App.vue 模板中根元素具有 `id="app"`
    app.$mount('#app')
    
### entry-server.js

服务器 entry 使用 default export 导出函数，并在每次渲染中重复调用此函数。此时，除了创建和返回应用程序实例之外，它不会做太多事情 - 但是稍后我们将在此执行服务器端路由匹配 
(server-side route matching) 和数据预取逻辑 (data pre-fetching logic)。

    import { createApp } from './app'

    export default context => {
        const { app } = createApp()
        return app
    }
    
### 路由

我们首先创建一个文件，在其中创建 router。注意，类似于 createApp，我们也需要给每个请求一个新的 router 实例，所以文件导出一个 createRouter 函数：

    // router.js
    import Vue from 'vue'
    import Router from 'vue-router'
    
    Vue.use(Router)
    
    export function createRouter () {
      return new Router({
        mode: 'history',
        routes: [
          // ...
        ]
      })
    }
    
然后更新 app.js:

    // app.js
    import Vue from 'vue'
    import App from './App.vue'
    import { createRouter } from './router'
    
    export function createApp () {
      // 创建 router 实例
      const router = createRouter()
    
      const app = new Vue({
        // 注入 router 到根 Vue 实例
        router,
        render: h => h(App)
      })
    
      // 返回 app 和 router
      return { app, router }
    }

现在我们需要在 entry-server.js 中实现服务器端路由逻辑 (server-side routing logic)：

    // entry-server.js
    import { createApp } from './app'
    
    export default context => {
      // 因为有可能会是异步路由钩子函数或组件，所以我们将返回一个 Promise，
        // 以便服务器能够等待所有的内容在渲染前，
        // 就已经准备就绪。
      return new Promise((resolve, reject) => {
        const { app, router } = createApp()
    
        // 设置服务器端 router 的位置
        router.push(context.url)
    
        // 等到 router 将可能的异步组件和钩子函数解析完
        router.onReady(() => {
          const matchedComponents = router.getMatchedComponents()
          // 匹配不到的路由，执行 reject 函数，并返回 404
          if (!matchedComponents.length) {
            return reject({ code: 404 })
          }
    
          // Promise 应该 resolve 应用程序实例，以便它可以渲染
          resolve(app)
        }, reject)
      })
    }
    
假设服务器 bundle 已经完成构建（请再次忽略现在的构建设置），服务器用法看起来如下：
 
    // server.js
    const createApp = require('/path/to/built-server-bundle.js')
    
    server.get('*', (req, res) => {

      const context = { url: req.url }
    
      createApp(context).then(app => {
        renderer.renderToString(app, (err, html) => {
          if (err) {
            if (err.code === 404) {
              res.status(404).end('Page not found')
            } else {
              res.status(500).end('Internal Server Error')
            }
          } else {
            res.end(html)
          }
        })
      })
    })

### Data Store

在服务器渲染期间，如果需要依赖一些数据，则在开始渲染之前，就必须先获取到这些数据，并且，在挂在到客户端应用程序之前，需要让客户端的数据和服务器端的数据一致，
否则客户端会因使用与服务器端不同的数据，而导致激活失败。

而为了解决这个问题，我们会用到store，我们会把服务器端获取的数据填充到store中，并且在挂在之前，把这个数据传递到客户端中。

    import Vuex from 'vuex'
    //具体实现代码忽略
    import common from  './common'
    import test from  './pages/test'

    Vue.use(Vuex);

    const store = new Vuex.Store({
        modules: {
            common,
            test
        }
    });

    export function createStore () {
        return store
    }

app.js

    import { createRouter } from './router'
    import { createStore } from './store'
    import main from './App.vue'

    export function createApp (mode) {
        const router = createRouter()
        const store = createStore()

        let app = new Vue({
            router,
            store,
            render: h => h(main)
        });
        return { app, router, store }
    }

另外，我们修改 enter-server.js

    // entry-server.js
    import { createApp } from './app'

    export default context => {
      return new Promise((resolve, reject) => {
        const { app, router, store } = createApp()

        router.push(context.url)

        router.onReady(() => {
          const matchedComponents = router.getMatchedComponents()
          if (!matchedComponents.length) {
            return reject({ code: 404 })
          }

          // 对所有匹配的路由组件调用 `asyncData()`
          Promise.all(matchedComponents.map(Component => {
            if (Component.asyncData) {
              return Component.asyncData({
                store,
                route: router.currentRoute
              })
            }
          })).then(() => {
            // 在所有预取钩子(preFetch hook) resolve 后，
            // 我们的 store 现在已经填充入渲染应用程序所需的状态。
            // 当我们将状态附加到上下文，
            // 并且 `template` 选项用于 renderer 时，
            // 状态将自动序列化为 `window.__INITIAL_STATE__`，并注入 HTML。
            context.state = store.state

            resolve(app)
          }).catch(reject)
        }, reject)
      })
    }

注意官方给出的 context.state = store.state 上的注释。
并且在组件中编写asyncData方法

    <!-- Item.vue -->
    <template>
      <div>{{ item.title }}</div>
    </template>

    <script>
    export default {
      asyncData ({ store, route }) {
        // 触发 action 后，会返回 Promise
        return store.dispatch('fetchItem', route.params.id)
      },
      computed: {
        // 从 store 的 state 对象中的获取 item。
        item () {
          return this.$store.state.items[this.$route.params.id]
        }
      }
    }
    </script>

### 客户端数据预取

在客户端，处理数据预取有两种不同方式：

#### 1.在路由导航之前解析数据：

使用此策略，应用程序会等待视图所需数据全部解析之后，再传入数据并处理当前视图。好处在于，可以直接在数据准备就绪时，传入视图渲染完整内容，但是如果数据预取需要很长时间，
用户在当前视图会感受到"明显卡顿"。因此，如果使用此策略，建议提供一个数据加载指示器 (data loading indicator)。

我们可以通过检查匹配的组件，并在全局路由钩子函数中执行 asyncData 函数，来在客户端实现此策略。注意，在初始路由准备就绪之后，我们应该注册此钩子，这样我们就不必再次获取服务器提取的数据。

#### 2.匹配要渲染的视图后，再获取数据：
此策略将客户端数据预取逻辑，放在视图组件的 beforeMount 函数中。当路由导航被触发时，可以立即切换视图，因此应用程序具有更快的响应速度。然而，传入视图在渲染时不会有完整的可用数据。
因此，对于使用此策略的每个视图组件，都需要具有条件加载状态。


