# ソースコード戦略

## ステートフルなシングルトンは避ける

クライアント専用のコードを書くとき、私たちはコードが毎回新鮮なコンテキストで評価されるという事実に慣れています。しかし、 Node.js サーバーは長時間実行されるプロセスです。私たちのコードがプロセスに要求されるとき、それは一度評価されメモリにとどまります。つまりシングルトンのオブジェクトを作成したとき、それは全ての受信リクエスト間でシェアされると言うことです。

基本的な例としては、私たちは **リクエストごとに新しいルート Vue インスタンスを作成します**。それは各ユーザがそれぞれのブラウザでアプリの新鮮なインスタンスを使用することに似ています。もし私たちが複数のリクエストをまたいでインスタンスを共有すると、それは容易にクロスリクエスト状態の汚染につながるでしょう。

そのため、直接アプリのインスタンスを作成するのではなく、各リクエストで繰り返し実行される新しいアプリのインスタンスを作成するファクトリ関数を公開する必要があります：

```js
// app.js
const Vue = require('vue')
module.exports = function createApp (context) {
  return new Vue({
    data: {
      url: context.url
    },
    template: `<div data-segment-id="282501">The visited URL is: {{ url }}</div>`
  })
}
```

そして私たちのサーバーコードはこうなります：

```js
// server.js
const createApp = require('./app')
server.get('*', (req, res) => {
  const context = { url: req.url }
  const app = createApp(context)
  renderer.renderToString(app, (err, html) => {
    // handle error...
    res.end(html)
  })
})
```

同じルールがルータ、ストア、イベントバスのインスタンスに適用されます。モジュールから直接エクスポートしアプリにインポートするのでは無く、 `createApp` で新しいインスタンスを作成し、ルート Vue インスタンスから注入する必要があります。

> `{ runInNewContext: true }` でバンドルレンダラを使用するとき、その制約を取り除くことが可能です。しかし各リクエストに対して新しい VM コンテキストを作成する必要があるため、いくらか重大なパフォーマンスコストがかかります。

## ビルドステップの紹介

これまでは、同じ Vue アプリケーションをクライアントへ配信する方法を論じてはいませんでした。これを行うには、webpack を使用して Vue アプリケーションをバンドルする必要があります。実際、webpack を使用して Vue アプリケーションをサーバーにバンドルしたいと思っているのはおそらく次の理由によるものです。

- 典型的な Vue アプリケーションは webpack と `vue-loader` によってビルドされ、 `file-loader` 経由でのファイルのインポートや`css-loader 経由でCSSをインポートなどの多くの webpack 固有の機能は Node.jsで直接動作しないため。`
- Node.jsの最新バージョンはES2015の機能を完全にサポートしていますが、古いブラウザに対応するためにクライアントサイドのコードをトランスパイルする必要があります。これはビルドステップにも再び関係します。

従って基本的な考え方は webpack を使用してクライアントとサーバー両方をバンドルすることです。サーバーバンドルはサーバーによって SSR のために要求され、クライアントバンドルは静的なマークアップのためにブラウザに送信されます。

![architecture](https://cloud.githubusercontent.com/assets/499550/17607895/786a415a-5fee-11e6-9c11-45a2cfdf085c.png)

セットアップの詳細については次のセクションで議論されます。今のところ、ビルドのセットアップが分かっていると仮定すると、webpack を有効にして Vue アプリコードを書くことが可能になっています。

## Webpackによるコード戦略

webpack を使用してサーバーとクライアントのアプリケーションを処理しているので、ソースコードの大部分はユニバーサルに書かれており、すべての webpack の機能にアクセスできます。 同時に、ユニバーサルなコードを書くときに留意すべき[いくつかのことがあります。](./universal.md)

シンプルなプロジェクトは以下のようになります：

```bash
src
├── components
│   ├── Foo.vue
│   ├── Bar.vue
│   └── Baz.vue
├── App.vue
├── app.js # universal entry
├── entry-client.js # runs in browser only
└── entry-server.js # runs on server only
```

### `app.js `

`app.js` はアプリケーションのユニバーサルエントリーです。クライアントアプリケーションでは、このファイルにルート Vue インスタンスを作成し、DOM に直接マウントします。しかし、SSRの場合は責務はクライアント専用のエントリファイルに映されます。`app.js` はシンプルに `createApp` 関数をエクスポートします:

```js
import Vue from 'vue'
import App from './App.vue'
// export a factory function for creating fresh app, router and store
// instances
export function createApp () {
  const app = new Vue({
    // the root instance simply renders the App component.
    render: h => h(App)
  })
  return { app }
}
```

### `entry-client.js`:

クライアントエントリは単にアプリケーションを作成しそれをDOMにマウントします：

```js
import { createApp } from './app'
// client-specific bootstrapping logic...
const { app } = createApp()
// this assumes App.vue template root element has id="app"
app.$mount('#app')
```

### `entry-server.js`:

サーバーエントリはレンダリングごとに繰り返し呼び出すことができる関数をデフォルトでエクスポートします。現時点では、アプリケーションインスタンスを作成して返す以外のことはほとんど行いませんが、後でサーバーサイドのルートマッチングとデータプリフェッチロジックを実行します。The server entry uses a default export which is a function that can be called repeatedly for each render. At this moment, it doesn't do much other than creating and returning the app instance - but later when we will perform server-side route matching and data pre-fetching logic here.

```js
import { createApp } from './app'
export default context => {
  const { app } = createApp()
  return app
}
```
