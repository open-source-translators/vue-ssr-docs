# 层叠样式(CSS)管理

在 `*.vue` 单文件组件中使用 `<style></style>` 标签是推荐的CSS管理方式。 有如下好处:

- Collocated, 可设置组件范围内CSS有效 
- 可以使用CSS预处理或后处理
- 开发时热替换

更重要的,`vue-loader`内部使用的 loader(加载器) `vue-style-loader`, 有支持服务端渲染的特殊功能：

- 一致的客户端与服务端开发体验.
- 使用 `bundleRenderer`时，自动临界CSS.

  如果在服务端渲染中使用了, 组件的CSS能被收集起来而且HTML行内( 当使用 `template`选项时，自动处理). 在客户端，当组件第一次使用时， `vue-style-loader` 将检查服务端是是否已有此组件的内联CSS - 如果没有，CSS将被动态得注入到`<style></style>` 标签中.

- 通用CSS抽取.

  安装支持使用 [`extract-text-webpack-plugin`](https://github.com/webpack-contrib/extract-text-webpack-plugin) 来抽取主chunk中的 CSS 到分离的 CSS 文件( 用 `template`自动注入), 这能让按文件单个进行缓存. 当有大量共享CSS时，推荐这样做。

  异步组件的CSS将像 JavaScript 字符串一样把置留在行内，由 `vue-style-loader` 处理.

## 启用CSS提取

使用 `vue-loader`的 `extractCSS` 配置从 `*.vue` 文件中抽取 CSS。(版本号要求 `vue-loader>=12.0.0`):

```js
// webpack.config.js
const ExtractTextPlugin = require('extract-text-webpack-plugin')
// CSS extraction should only be enabled for production
// so that we still get hot-reload during development.
const isProduction = process.env.NODE_ENV === 'production'
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          // enable CSS extraction
          extractCSS: isProduction
        }
      },
      // ...
    ]
  },
  plugins: isProduction
    // make sure to add the plugin!
    ? [new ExtractTextPlugin({ filename: 'common.[chunkhash].css' })]
    : []
}
```

注意上边代码的配置仅作用于`*.vue`文件中的样式，但可以使用 `<style src="./foo.css"></style>` 把外部引入到 Vue 组件中.

如果期望从JavaScript导入CSS, 如. `import 'foo.css'`, 你需要配置相应的loader(加载器):

```js
module.exports = {
  // ...
  module: {
    rules: [
      {
        test: /\.css$/,
        // important: use vue-style-loader instead of style-loader
        use: isProduction
          ? ExtractTextPlugin.extract({
              use: 'css-loader',
              fallback: 'vue-style-loader'
            })
          : ['vue-style-loader', 'css-loader']
      }
    ]
  },
  // ...
}
```

## 从依赖导入样式

从NPM依赖导入CSS时需要注意以下几点：

1. 服务端构建后，导入的样式不应该在外部了。
2. 如果同时使用 CSS 抽取 + `CommonsChunkPlugin`, `extract-text-webpack-plugin`的vendor 抽取 ，如果要抽取的CSS同时在vendors chunk中，运行时会报错。想正常运行，须避免CSS文件在vendor chunk中。 示例客户端webpack配置如下:

```js
  module.exports = {
    // ...
    plugins: [
      // it is common to extract deps into a vendor chunk for better caching.
      new webpack.optimize.CommonsChunkPlugin({
        name: 'vendor',
        minChunks: function (module) {
          // a module is extracted into the vendor chunk when...
          return (
            // if it's inside node_modules
            /node_modules/.test(module.context) &&
            // do not externalize if the request is a CSS file
            !/\.css$/.test(module.request)
          )
        }
      }),
      // extract webpack runtime & manifest
      new webpack.optimize.CommonsChunkPlugin({
        name: 'manifest'
      }),
      // ...
    ]
  }
```
