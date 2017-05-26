# データのプリフェッチとステート

## データストア

SSR をしているとき、基本的にはアプリケーションの「スナップショット」をレンダリングしています、したがって、アプリケーションがいくつかの非同期データに依存している場合においては、**それらのデータを、レンダリング処理を開始する前にプリフェッチして解決する必要があります**。

もうひとつの重要なことは、クライアントサイドでアプリケーションがマウとされる前に、クライアントサイドで同じデータを利用可能である必要があるということです。そうしないと、クライアントサイドが異なるステートを用いてレンダリングしてしまい、ハイドレーションが失敗してしまいます。

この問題に対応するため、フェッチされたデータはビューコンポーネントの外でも存続している必要があります。つまり特定の用途のデータストアもしくは "ステート・コンテナ" に入っている必要があります。サーバーサイドではレンダリングする前にデータをプリフェッチしてストアの中に入れることができます。さらにシリアライズして HMTL にステートを埋め込みます。クライアントサイドのストアは、アプリケーションをマウントする前に、埋め込まれたステートを直接取得できます。

このような用途として、公式のステート管理ライブラリである [Vuex](https://github.com/vuejs/vuex/) を使っています。では `store.js` ファイルをつくって、そこに id に基づく item を取得するコードを書いてみましょう:

```js
// store.js
import Vue from 'vue'
import Vuex from 'vuex'
Vue.use(Vuex)
// Assume we have a universal API that returns Promises
// and ignore the implementation details
import { fetchItem } from './api'
export function createStore () {
  return new Vuex.Store({
    state: {
      items: {}
    },
    actions: {
      fetchItem ({ commit }, id) {
        // return the Promise via store.dispatch() so that we know
        // when the data has been fetched
        return fetchItem(id).then(item => {
          commit('setItem', { id, item })
        })
      }
    },
    mutations: {
      setItem (state, { id, item }) {
        Vue.set(state.items, id, item)
      }
    }
  })
}
```

そして `app.js` を更新します:

```js
// app.js
import Vue from 'vue'
import App from './App.vue'
import { createRouter } from './router'
import { createStore } from './store'
import { sync } from 'vuex-router-sync'
export function createApp () {
  // create router and store instances
  const router = createRouter()
  const store = createStore()
  // sync so that route state is available as part of the store
  sync(store, router)
  // create the app instance, injecting both the router and the store
  const app = new Vue({
    router,
    store,
    render: h => h(App)
  })
  // expose the app, the router and the store.
  return { app, router, store }
}
```

## ロジックとコンポーネントとの結び付き

ではデータをプリフェッチするアクションをディスパッチするコードはどこに書けばよいでしょうか？

フェッチする必要があるデータはアクセスしたルートによって決まります。またそのルートによってどのコンポーネントがレンダリングされるかも決まります。実のところ、与えられたルートに必要とされるデータは、そのルートでレンダリングされるコンポーネントに必要とされるデータでもあるのです。したがって、データをフェッチするロジックはルートコンポーネントの中に書くのが自然でしょう。

ルートコンポーネントではカスタム静的関数 `asyncData` が利用可能です。この関数はそのルートコンポーネントがインスタンス化される前に呼び出されるため `this` にアクセスできないことを覚えておいてください。ストアとルートの情報は引数として渡される必要があります:

```html
<!-- Item.vue -->
<template>
  <div data-segment-id="282437">{{ item.title }}</div>
</template>
<script>
export default {
  asyncData ({ store, route }) {
    // return the Promise from the action
    return store.dispatch('fetchItem', route.params.id)
  },
  computed: {
    // display the item from store state.
    items () {
      return this.$store.state.items[this.$route.params.id]
    }
  }
}
</script>
```

## サーバーサイドのデータ取得

`entry-server.js` において `router.getMatchedComponents()` を使ってルートにマッチしたコンポーネントを取得できます。そしてコンポーネントが `asyncData` を利用可能にしていればそれを呼び出すことができます。そしてレンダリングのコンテキストに解決したステートを付属させる必要があります。

```js
// entry-server.js
import { createApp } from './app'
export default context => {
  return new Promise((resolve, reject) => {
    const { app, router, store } = createApp()
    router.push(context.url)
    router.onReady(() => {
      const matchedComponents = router.getMatchedComponents()
      if (!matchedComponents.length) {
        reject({ code: 404 })
      }
      // call asyncData() on all matched route components
      Promise.all(matchedComponents.map(Component => {
        if (Component.asyncData) {
          return Component.asyncData({
            store,
            route: router.currentRoute
          })
        }
      })).then(() => {
        // After all preFetch hooks are resolved, our store is now
        // filled with the state needed to render the app.
        // When we attach the state to the context, and the `template` option
        // is used for the renderer, the state will automatically be
        // serialized and injected into the HTML as window.__INITIAL_STATE__.
        context.state = store.state
        resolve(app)
      }).catch(reject)
    }, reject)
  })
}
```

`template` を使うと `context.state` は自動的に最終的な HTML に `window.__INITIAL__` というかたちのステートとして埋め込まれます。クライアントサイドでは、アプリケーションがマウントされる前に、ストアがそのステートを取得します:

```js
// entry-client.js
const { app, router, store } = createApp()
if (window.__INITIAL_STATE__) {
  store.replaceState(window.__INITIAL_STATE__)
}
```

## クライアントサイドのデータ取得

クライアントサイドではデータ取得について 2つの異なるアプローチがあります:

1. **ルートのナビゲーションの前にデータを解決する:**

この方法では、アプリケーションは、遷移先のビューが必要とするデータが解決されるまで、現在のビューを保ちます。良い点は遷移先のビューがデータの準備が整い次第、フルの内容をダイレクトにレンダリングできることです。しかしながら、データの取得に時間がかかるときは、ユーザーは現在のビューで「固まってしまった」と感じてしまうでしょう。そのため、この方法を用いるときにはローディング・インジケーターを表示させることが推奨されます。

この方法は、クライアントサイドでマッチするコンポーネントをチェックし、グローバルなルートのフック内で `asyncData` 関数を実行することにより実装できます。重要なことは、このフックは初期ルートが ready になった後に登録するということです。そうすれば、サーバーサイドで取得したデータをもう一度無駄に取得せずに済みます。

```js
  // entry-client.js
  // ...omitting unrelated code
  router.onReady(() => {
    // Add router hook for handling asyncData.
    // Doing it after initial route is resolved so that we don't double-fetch
    // the data that we already have. Using router.beforeResolve() so that all
    // async components are resolved.
    router.beforeResolve((to, from, next) => {
      const matched = router.getMatchedComponents(to)
      const prevMatched = router.getMatchedComponents(from)
      // we only care about none-previously-rendered components,
      // so we compare them until the two matched lists differ
      let diffed = false
      const activated = matched.filter((c, i) => {
        return diffed || (diffed = (prevMatched[i] !== c))
      })
      if (!activated.length) {
        return next()
      }
      // this is where we should trigger a loading indicator if there is one
      Promise.all(activated.map(c => {
        if (c.asyncData) {
          return c.asyncData({ store, route: to })
        }
      })).then(() => {
        // stop loading indicator
        next()
      }).catch(next)
    })
    app.$mount('#app')
  })
```

1. **マッチするビューがレンダリングされた後にデータを取得する:**

この方法ではビューコンポーネントの `beforeMount` 関数内にクライアントサイドでデータを取得するロジックを置きます。こうすればルートのナビゲーションが発火したらすぐにビューを切り替えられます。そうすればアプリケーションはよりレスポンスが良いと感じられるでしょう。しかしながら、遷移先のビューはレンダリングした時点ではフルのデータを持っていません。したがって、この方法を使うコンポーネントの各々がローディング中か否かの状態を持つ必要があります。

この方法はクライアントサイド限定のグローバルな mixin で実装できます:

```js
  Vue.mixin({
    beforeMount () {
      const { asyncData } = this.$options
      if (asyncData) {
        // assign the fetch operation to a promise
        // so that in components we can do `this.dataPromise.then(...)` to
        // perform other tasks after data is ready
        this.dataPromise = asyncData({
          store: this.$store,
          route: this.$route
        })
      }
    }
  })
```

これら 2つの方法のどちらを選ぶかは、究極的には異なる UX のどちらを選ぶかの判断であり、構築しようとしているアプリケーションの実際のシナリオに基づいて選択されるべきものです。しかし、どちらの方法を選択したかにかかわらず、ルートコンポーネントが再利用されたとき（つまりルートは同じだがパラメーターやクエリが変わったとき。例えば `user/1` から `user/2`) へ変わったとき）には `asyncData` 関数は呼び出されるようにすべきです。これはクライアントサイド限定のグローバルな mixin でハンドリングできます:

```js
Vue.mixin({
  beforeRouteUpdate (to, from, next) {
    const { asyncData } = this.$options
    if (asyncData) {
      asyncData({
        store: this.$store,
        route: to
      }).then(next).catch(next)
    } else {
      next()
    }
  }
})
```

---

ふぅ、コードが長いですね。これはどうしてかというと、ユニバーサルなデータ取得は、大抵の場合、サーバーサイドでレンダリングするアプリケーションの最も複雑な問題であり、また、今後、スムーズに開発を進めていくための下準備をしているためです。一旦ひな形が準備できてしまえば、あとは、それぞれのコンポーネントを記述していく作業は、実際のところ実に楽しいものになるはずです。
