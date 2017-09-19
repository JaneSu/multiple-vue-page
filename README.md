最近一直在研究使用vue做出来一些东西，但都是SPA的单页面应用，但实际工作中，单页面并不一定符合业务需求，所以这篇我就来说说怎么开发多页面的Vue应用，以及在这个过程会遇到的问题。


# 准备工作

在本地用`vue-cli`新建一个项目，这个步骤vue的官网上有，我就不再说了。

这里有一个地方需要改一下，在执行`npm install`命令之前，在`package.json`里添加一个依赖,后面会用到。

![](http://onsmpwfeh.bkt.clouddn.com/15058154261893.jpg)

# 修改webpack配置

这里展示一下我的项目目录

```
├── README.md
├── build
│   ├── build.js
│   ├── check-versions.js
│   ├── dev-client.js
│   ├── dev-server.js
│   ├── utils.js
│   ├── vue-loader.conf.js
│   ├── webpack.base.conf.js
│   ├── webpack.dev.conf.js
│   └── webpack.prod.conf.js
├── config
│   ├── dev.env.js
│   ├── index.js
│   └── prod.env.js
├── package.json
├── src
│   ├── assets
│   │   └── logo.png
│   ├── components
│   │   ├── Hello.vue
│   │   └── cell.vue
│   └── pages
│       ├── cell
│       │   ├── cell.html
│       │   ├── cell.js
│       │   └── cell.vue
│       └── index
│           ├── index.html
│           ├── index.js
│           ├── index.vue
│           └── router
│               └── index.js
└── static
```

在这一步里我们需要改动的文件都在`build`文件下，分别是：

- utils.js
- webpack.base.conf.js
- webpack.dev.conf.js
- webpack.prod.conf.js

我就按照顺序放出完整的文件内容，然后在做修改或添加的位置用注释符标注出来：

## utils.js文件

```javascript
// utils.js文件

var path = require('path')
var config = require('../config')
var ExtractTextPlugin = require('extract-text-webpack-plugin')

exports.assetsPath = function (_path) {
    var assetsSubDirectory = process.env.NODE_ENV === 'production' ?
        config.build.assetsSubDirectory :
        config.dev.assetsSubDirectory
    return path.posix.join(assetsSubDirectory, _path)
}

exports.cssLoaders = function (options) {
    options = options || {}

    var cssLoader = {
        loader: 'css-loader',
        options: {
            minimize: process.env.NODE_ENV === 'production',
            sourceMap: options.sourceMap
        }
    }

    // generate loader string to be used with extract text plugin
    function generateLoaders(loader, loaderOptions) {
        var loaders = [cssLoader]
        if (loader) {
            loaders.push({
                loader: loader + '-loader',
                options: Object.assign({}, loaderOptions, {
                    sourceMap: options.sourceMap
                })
            })
        }

        // Extract CSS when that option is specified
        // (which is the case during production build)
        if (options.extract) {
            return ExtractTextPlugin.extract({
                use: loaders,
                fallback: 'vue-style-loader'
            })
        } else {
            return ['vue-style-loader'].concat(loaders)
        }
    }

    // https://vue-loader.vuejs.org/en/configurations/extract-css.html
    return {
        css: generateLoaders(),
        postcss: generateLoaders(),
        less: generateLoaders('less'),
        sass: generateLoaders('sass', { indentedSyntax: true }),
        scss: generateLoaders('sass'),
        stylus: generateLoaders('stylus'),
        styl: generateLoaders('stylus')
    }
}

// Generate loaders for standalone style files (outside of .vue)
exports.styleLoaders = function (options) {
    var output = []
    var loaders = exports.cssLoaders(options)
    for (var extension in loaders) {
        var loader = loaders[extension]
        output.push({
            test: new RegExp('\\.' + extension + '$'),
            use: loader
        })
    }
    return output
}

/* 这里是添加的部分 ---------------------------- 开始 */

// glob是webpack安装时依赖的一个第三方模块，还模块允许你使用 *等符号, 例如lib/*.js就是获取lib文件夹下的所有js后缀名的文件
var glob = require('glob')
// 页面模板
var HtmlWebpackPlugin = require('html-webpack-plugin')
// 取得相应的页面路径，因为之前的配置，所以是src文件夹下的pages文件夹
var PAGE_PATH = path.resolve(__dirname, '../src/pages')
// 用于做相应的merge处理
var merge = require('webpack-merge')


//多入口配置
// 通过glob模块读取pages文件夹下的所有对应文件夹下的js后缀文件，如果该文件存在
// 那么就作为入口处理
exports.entries = function () {
    var entryFiles = glob.sync(PAGE_PATH + '/*/*.js')
    var map = {}
    entryFiles.forEach((filePath) => {
        var filename = filePath.substring(filePath.lastIndexOf('\/') + 1, filePath.lastIndexOf('.'))
        map[filename] = filePath
    })
    return map
}

//多页面输出配置
// 与上面的多页面入口配置相同，读取pages文件夹下的对应的html后缀文件，然后放入数组中
exports.htmlPlugin = function () {
    let entryHtml = glob.sync(PAGE_PATH + '/*/*.html')
    let arr = []
    entryHtml.forEach((filePath) => {
        let filename = filePath.substring(filePath.lastIndexOf('\/') + 1, filePath.lastIndexOf('.'))
        let conf = {
            // 模板来源
            template: filePath,
            // 文件名称
            filename: filename + '.html',
            // 页面模板需要加对应的js脚本，如果不加这行则每个页面都会引入所有的js脚本
            chunks: ['manifest', 'vendor', filename],
            inject: true
        }
        if (process.env.NODE_ENV === 'production') {
            conf = merge(conf, {
                minify: {
                    removeComments: true,
                    collapseWhitespace: true,
                    removeAttributeQuotes: true
                },
                chunksSortMode: 'dependency'
            })
        }
        arr.push(new HtmlWebpackPlugin(conf))
    })
    return arr
}
/* 这里是添加的部分 ---------------------------- 结束 */
```

## webpack.base.conf.js 文件

```javascript
// webpack.base.conf.js 文件

var path = require('path')
var utils = require('./utils')
var config = require('../config')
var vueLoaderConfig = require('./vue-loader.conf')

function resolve(dir) {
  return path.join(__dirname, '..', dir)
}

module.exports = {
  /* 修改部分 ---------------- 开始 */
  entry: utils.entries(),
  /* 修改部分 ---------------- 结束 */
  output: {
    path: config.build.assetsRoot,
    filename: '[name].js',
    publicPath: process.env.NODE_ENV === 'production' ?
      config.build.assetsPublicPath :
      config.dev.assetsPublicPath
  },
  resolve: {
    extensions: ['.js', '.vue', '.json'],
    alias: {
      'vue$': 'vue/dist/vue.esm.js',
      '@': resolve('src'),
      'pages': resolve('src/pages'),
      'components': resolve('src/components')
    }
  },
  module: {
    rules: [{
      test: /\.vue$/,
      loader: 'vue-loader',
      options: vueLoaderConfig
    },
    {
      test: /\.js$/,
      loader: 'babel-loader',
      include: [resolve('src'), resolve('test')]
    },
    {
      test: /\.(png|jpe?g|gif|svg)(\?.*)?$/,
      loader: 'url-loader',
      options: {
        limit: 10000,
        name: utils.assetsPath('img/[name].[hash:7].[ext]')
      }
    },
    {
      test: /\.(woff2?|eot|ttf|otf)(\?.*)?$/,
      loader: 'url-loader',
      options: {
        limit: 10000,
        name: utils.assetsPath('fonts/[name].[hash:7].[ext]')
      }
    }
    ]
  }
}
```

## webpack.dev.conf.js 文件

```javasctipt
var utils = require('./utils')
var webpack = require('webpack')
var config = require('../config')
var merge = require('webpack-merge')
var baseWebpackConfig = require('./webpack.base.conf')
var HtmlWebpackPlugin = require('html-webpack-plugin')
var FriendlyErrorsPlugin = require('friendly-errors-webpack-plugin')

// add hot-reload related code to entry chunks
Object.keys(baseWebpackConfig.entry).forEach(function (name) {
  baseWebpackConfig.entry[name] = ['./build/dev-client'].concat(baseWebpackConfig.entry[name])
})

module.exports = merge(baseWebpackConfig, {
  module: {
    rules: utils.styleLoaders({ sourceMap: config.dev.cssSourceMap })
  },
  // cheap-module-eval-source-map is faster for development
  devtool: '#cheap-module-eval-source-map',
  plugins: [
    new webpack.DefinePlugin({
      'process.env': config.dev.env
    }),
    // https://github.com/glenjamin/webpack-hot-middleware#installation--usage
    new webpack.HotModuleReplacementPlugin(),
    new webpack.NoEmitOnErrorsPlugin(),
    // https://github.com/ampedandwired/html-webpack-plugin
    /* 注释这个区域的文件 ------------- 开始 */
    // new HtmlWebpackPlugin({
    //   filename: 'index.html',
    //   template: 'index.html',
    //   inject: true
    // }),
    /* 注释这个区域的文件 ------------- 结束 */
    new FriendlyErrorsPlugin()

    /* 添加 .concat(utils.htmlPlugin()) ------------------ */
  ].concat(utils.htmlPlugin())
})
```

## webpack.prod.conf.js 文件

```javascript
var path = require('path')
var utils = require('./utils')
var webpack = require('webpack')
var config = require('../config')
var merge = require('webpack-merge')
var baseWebpackConfig = require('./webpack.base.conf')
var CopyWebpackPlugin = require('copy-webpack-plugin')
var HtmlWebpackPlugin = require('html-webpack-plugin')
var ExtractTextPlugin = require('extract-text-webpack-plugin')
var OptimizeCSSPlugin = require('optimize-css-assets-webpack-plugin')

var env = config.build.env

var webpackConfig = merge(baseWebpackConfig, {
  module: {
    rules: utils.styleLoaders({
      sourceMap: config.build.productionSourceMap,
      extract: true
    })
  },
  devtool: config.build.productionSourceMap ? '#source-map' : false,
  output: {
    path: config.build.assetsRoot,
    filename: utils.assetsPath('js/[name].[chunkhash].js'),
    chunkFilename: utils.assetsPath('js/[id].[chunkhash].js')
  },
  plugins: [
    // http://vuejs.github.io/vue-loader/en/workflow/production.html
    new webpack.DefinePlugin({
      'process.env': env
    }),
    new webpack.optimize.UglifyJsPlugin({
      compress: {
        warnings: false
      },
      sourceMap: true
    }),
    // extract css into its own file
    new ExtractTextPlugin({
      filename: utils.assetsPath('css/[name].[contenthash].css')
    }),
    // Compress extracted CSS. We are using this plugin so that possible
    // duplicated CSS from different components can be deduped.
    new OptimizeCSSPlugin({
      cssProcessorOptions: {
        safe: true
      }
    }),
    // generate dist index.html with correct asset hash for caching.
    // you can customize output by editing /index.html
    // see https://github.com/ampedandwired/html-webpack-plugin

    /* 注释这个区域的内容 ---------------------- 开始 */
    // new HtmlWebpackPlugin({
    //   filename: config.build.index,
    //   template: 'index.html',
    //   inject: true,
    //   minify: {
    //     removeComments: true,
    //     collapseWhitespace: true,
    //     removeAttributeQuotes: true
    //     // more options:
    //     // https://github.com/kangax/html-minifier#options-quick-reference
    //   },
    //   // necessary to consistently work with multiple chunks via CommonsChunkPlugin
    //   chunksSortMode: 'dependency'
    // }),
    /* 注释这个区域的内容 ---------------------- 结束 */

    // split vendor js into its own file
    new webpack.optimize.CommonsChunkPlugin({
      name: 'vendor',
      minChunks: function (module, count) {
        // any required modules inside node_modules are extracted to vendor
        return (
          module.resource &&
          /\.js$/.test(module.resource) &&
          module.resource.indexOf(
            path.join(__dirname, '../node_modules')
          ) === 0
        )
      }
    }),
    // extract webpack runtime and module manifest to its own file in order to
    // prevent vendor hash from being updated whenever app bundle is updated
    new webpack.optimize.CommonsChunkPlugin({
      name: 'manifest',
      chunks: ['vendor']
    }),
    // copy custom static assets
    new CopyWebpackPlugin([{
      from: path.resolve(__dirname, '../static'),
      to: config.build.assetsSubDirectory,
      ignore: ['.*']
    }])
    /* 该位置添加 .concat(utils.htmlPlugin()) ------------------- */
  ].concat(utils.htmlPlugin())
})

if (config.build.productionGzip) {
  var CompressionWebpackPlugin = require('compression-webpack-plugin')

  webpackConfig.plugins.push(
    new CompressionWebpackPlugin({
      asset: '[path].gz[query]',
      algorithm: 'gzip',
      test: new RegExp(
        '\\.(' +
        config.build.productionGzipExtensions.join('|') +
        ')$'
      ),
      threshold: 10240,
      minRatio: 0.8
    })
  )
}

if (config.build.bundleAnalyzerReport) {
  var BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin
  webpackConfig.plugins.push(new BundleAnalyzerPlugin())
}

module.exports = webpackConfig
```

至此，webpack的配置就结束了。

但是还没完啦，下面继续。


# 文件结构 

```
├── src
│   ├── assets
│   │   └── logo.png
│   ├── components
│   │   ├── Hello.vue
│   │   └── cell.vue
│   └── pages
│       ├── cell
│       │   ├── cell.html
│       │   ├── cell.js
│       │   └── cell.vue
│       └── index
│           ├── index.html
│           ├── index.js
│           ├── index.vue
│           └── router
│               └── index.js
```

`src`就是我所使用的工程文件了，`assets`,`components`,`pages`分别是静态资源文件、组件文件、页面文件。

前两个就不多说，主要是页面文件里，我目前是按照项目的模块分的文件夹，你也可以按照你自己的需求调整。然后在每个模块里又有三个内容：vue文件，js文件和html文件。这三个文件的作用就相当于做spa单页面应用时，根目录的`index.html`页面模板，src文件下的`main.js`和`app.vue`的功能。

原先，入口文件只有一个main.js,但现在由于是多页面，因此入口页面多了，我目前就是两个：index和cell，之后如果打包，就会在`dist`文件下生成两个HTML文件：`index.html`和`cell.html`(可以参考一下单页面应用时，打包只会生成一个index.html,区别在这里)。

cell文件下的三个文件，就是一般模式的配置，参考index的就可以，但并不完全相同。

# 特别注意的地方

## cell.js

在这个文件里，按照写法，应该是这样的吧：

```javascript
import Vue from 'Vue'
import cell from './cell.vue'

new Vue({
    el:'#app'，// 这里参考cell.html和cell.vue的根节点id，保持三者一致
    teleplate：'<cell/>',
    components:{ cell }
})
```

这个配置在运行时（npm run dev）会报错

```
[Vue warn]: You are using the runtime-only build of Vue where the template compiler is not available. Either pre-compile the templates into render functions, or use the compiler-included build.
(found in <Root>)
```

网上的解释是这样的：

> 运行时构建不包含模板编译器，因此不支持 template 选项，只能用 render 选项，但即使使用运行时构建，在单文件组件中也依然可以写模板，因为单文件组件的模板会在构建时预编译为 render 函数。运行时构建比独立构建要轻量30%，只有 17.14 Kb min+gzip大小。
上面一段是官方api中的解释。就是说，如果我们想使用template，我们不能直接在客户端使用npm install之后的vue。

也给出了相应的修改方案：

```
resolve: { alias: { 'vue': 'vue/dist/vue.js' } }
```

这里是修改`package.json`的resolve下的vue的配置，很多人反应这样修改之后就好了，但是我按照这个方法修改之后依然报错。然后我就想到上面提到的`render`函数，因此我的修改是针对`cell.js`文件的。

```javascript
import Vue from 'Vue'
import cell from './cell.vue'

/* eslint-disable no-new */
new Vue({
  el: '#app',
  render: h => h(cell)
})

```

这里面我用`render`函数取代了组件的写法，在运行就没问题了。

## 页面跳转

既然是多页面，肯定涉及页面之间的互相跳转，就按照我这个项目举例，从index.html文件点击a标签跳转到cell.html。

我最开始写的是：

```html
 <!-- index.html -->
<a href='../cell/cell.html'></a>
```

但这样写，不论是在开发环境还是最后测试，都会报404，找不到这个页面。

改成这样既可：

```html
 <!-- index.html -->
<a href='cell.html'></a>
```

这样他就会自己找`cell.html`这个文件。

## 打包后的资源路径

执行`npm run build`之后，打开相应的html文件你是看不到任何东西的，查看原因是找不到相应的js文件和css文件。

这时候的文件结构是这样的：

```
├── dist
│   ├── js
│   ├── css
│   ├── index.html
│   └── cell.html
```

查看index.html文件之后会发现资源的引用路径是：

    /dist/js.........
    
这样，如果你的dist文件不是在根目录下的，就根本找不到资源。

方法当然也有啦，如果你不嫌麻烦，就一个文件一个文件的修改路径咯，或者像我一样偷懒，修改`config`下的`index.js`文件。具体的做法是：

```js
build: {
    env: require('./prod.env'),
    index: path.resolve(__dirname, '../dist/index.html'),
    assetsRoot: path.resolve(__dirname, '../dist'),
    assetsSubDirectory: 'static',
    assetsPublicPath: '/',
    productionSourceMap: true,
    // Gzip off by default as many popular static hosts such as
    // Surge or Netlify already gzip all static assets for you.
    // Before setting to `true`, make sure to:
    // npm install --save-dev compression-webpack-plugin
    productionGzip: false,
    productionGzipExtensions: ['js', 'css'],
    // Run the build command with an extra argument to
    // View the bundle analyzer report after build finishes:
    // `npm run build --report`
    // Set to `true` or `false` to always turn it on or off
    bundleAnalyzerReport: process.env.npm_config_report
  },

```

将这里面的

    assetsPublicPath: '/',
    
改成

    assetsPublicPath: './',
    
酱紫，配置文件资源的时候找到的就是相对路径下的资源了，在重新`npm run build`看看吧。

