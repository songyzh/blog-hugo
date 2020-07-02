---
title: "webpack项目打包优化"
slug: ""
date: "2020-07-02T17:00:06+08:00"
description: ""

draft: false

hideToc: false
enableToc: true
enableTocContent: false

author: "东"
authorEmoji: ""
authorImage: ""
authorImageUrl: ""
authorDesc: ""
socialOptions:
  email: "mailto:1619882712@qq.com"
  github: "https://github.com/happyElina"
  weibo: "https://weibo.com/u/2669254565"

tags:
  - 前端
  - webpack
categories:
  - 技术
series:
  - webpack学习总结

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1ggcra0l7cqj30k00a5js3.jpg"

libraries:
  - katex
  - chart
  - flowchartjs
  - msc
  - mathjax
  - mermaid
  - viz
  - wavedrom
---


本文主要是对项目构建优化过程的记录，项目是基于vue3.0脚手架搭件的

首次打包  dist **: 15.1 M**

## 1  发现dist/img/icons 中有很多vue-cli 自带的vue-icon ，将pubic/img/icon 文件夹删除

​    img中的favicon.icon 可以换成自己网站的favicon.icon(4k)

​    再次打包： **15M**

## 2 速度分析

我们的目的是优化打包速度，那肯定需要一个速度分析插件，此时 speed-measure-webpack-plugin 就派上用场了。它的作用如下：

- 分析整个打包总耗时
- 每个 plugin 和 loader 的耗时情况

首先，安装插件

```javascript
npm i -D speed-measure-webpack-plugin
```

然后修改 vue.config.js 配置文件

```javascript
// 导入速度分析插件
const SpeedMeasurePlugin = require("speed-measure-webpack-plugin");
// 实例化插件
const smp = new SpeedMeasurePlugin();
module.exports = {
  configureWebpack: smp.wrap({
    plugins: [
    // 这里是自己项目里需要使用到的其他插件
    // new yourOtherPlugin()
    ]
  })
}
```

打包后总耗时 **38.28** secs ，也可以看到各个loader 打包所耗费时间

## 3 取消生产环境生成map文件

```javascript
devtool: '#eval-source-map',//映射js到原文件
```

由于打包后的js调试不方面，所以应用此，自动映射报错到原文件.

看到生成的js文件件中有很多.js.map 文件，应该在生产环境去除，以加速生产环境构建，缩小体积

```javascript
productionSourceMap: false
```

再次打包：  **25.93 secs**   **dist 4.4M**

## 4 为了防止dist 目录文件越来越多，每次都要手动清楚dist 文件夹，比较麻烦

clean-webpack-plugin 默认每次打包前删除output.path  文件夹

## 5 一直都有的warning

 ```javascript
The following asset(s) exceed the recommended size limit (244 KiB).

This can impact web performance.  以下文件超出了建议的大小

 img/ionicons.a2c4a261.svg (542 KiB)

 css/chunk-vendors.d48681d6.css (583 KiB)

  js/chunk-vendors.7a36829a.js (1.8 MiB)

  js/main.42b88c10.js (527 KiB)

entrypoint size limit: The following entrypoint(s) combined asset size exceeds the recommended limit (244 KiB). This can impact web performance. 入口文件以及相关的资源大小超过了建议的244k

Entrypoints:

  app (2.41 MiB)

​      css/chunk-vendors.d48681d6.css

​      js/chunk-vendors.7a36829a.js

​      css/app.15a328ee.css

​      js/app.cbb3c76a.js
 ```

查看main.ts 觉得引入的内容过多，但是具体的打包情况不清楚，由此引入webpack-bundle-analyzer 插件来进行体积分析

```javascript
npm i -D webpack-bundle-analyzer
```

打包后可以看到node_modules 中的内容达到了5.43M,其他内容1.11M （未压缩）

其中node_modules chunk-vendor.js 中最大的是elementui

发现有很多动态加载的路由组件没有写明 webpackChunkName 导致单独打包了，而且其他动态加载的都叫main,导致打包的动态加载main.JS 过大

 **优化：  将动态加载的模块根据功能块分为 order  list  user**

再次打包 没有过大的main.js 以及一些匿名的chunk ,打包时间**18.69 secs** 但还是有一些警告：

```javascript
asset size limit: The following asset(s) exceed the recommended size limit (244 KiB).

This can impact web performance.

Assets:

  img/ionicons.a2c4a261.svg (542 KiB)

  css/chunk-vendors.d48681d6.css (583 KiB)

  js/chunk-vendors.aefb80b0.js (1.8 MiB)

  js/list~order~user.7ae80867.js (396 KiB)

 warning

entrypoint size limit: The following entrypoint(s) combined asset size exceeds the recommended limit (244 KiB). This can impact web performance.

Entrypoints:

  app (2.41 MiB)

​      css/chunk-vendors.d48681d6.css

​      js/chunk-vendors.aefb80b0.js

​      css/app.15a328ee.css

​      js/app.a09ad1ae.js
```

警告是因为默认配置中提供了这些限制

[https://cli.vuejs.org/zh/guide/build-targets.html#%E5%BA%94%E7%94%A8](https://cli.vuejs.org/zh/guide/build-targets.html#应用) 中给出了默认的打包策略

@vue\cli-service\lib\config\prod.js  这里是cli中的默认配置，可以根据需要查看

发现  第三方库会被分到一个独立包以便更好的缓存（也就是比较大的chunk-vendors.）,为了更快的加载，决定提取比较大的element

在我们的项目中要引入很多的第三方组件库例如element-ui，还有公司内部的一些组件库，如果我们使用默认配置不做任何修改，vendors缓存组中的内容会很多，页面引用时耗时就会较长，所以将缓存组中的内容进行拆分是很有必要的。

例如我们将element-ui分割成一个独立的chunk，cacheGroups中配置如下：

```javascript
 configureWebpack: smp.wrap({
    plugins: [
        // 这里是自己项目里需要使用到的其他插件
        // new yourOtherPlugin()
        new CleanWebpackPlugin(),
        new BundleAnalyzerPlugin()
    ],
    optimization: { //注意这里要写到configureWebpack: smp.wra 内部
      splitChunks: {
        cacheGroups:{
            elementUI: {
              priority: 20,
              name: "elementUI",
              test: /element-ui/,
              reuseExistingChunk: true,
              chunks: 'initial'
            }
        }
      }
    }
  })
```

再次打包：  都在500k 之内，**用时17.43 secs**  **dist 4.4M** 如果觉得不好，还可以继续抽取某个模块出来

```javascript
dist/js/chunk-vendors.f6665299.js         1187.85 KiB      410.17 KiB

  dist/js/elementUI.8d715862.js             651.54 KiB       161.19 KiB

  dist/js/list~order~user.7ae80867.js       396.42 KiB       109.46 KiB

  dist/js/list.4bae4c4a.js                  73.11 KiB        20.93 KiB

  dist/js/order.d53d6a7b.js                 56.50 KiB        11.95 KiB

  dist/js/user.960bd4f6.js                  52.52 KiB        10.88 KiB

  dist/js/app.7a5d9e33.js                   30.20 KiB        8.92 KiB

  dist/precache-manifest.b90139ea6a8765f    2.58 KiB         0.81 KiB

  806da9c48a2db3e60.js

  dist/service-worker.js                    0.95 KiB         0.54 KiB

  dist/css/chunk-vendors.2451089e.css       347.71 KiB       50.92 KiB

  dist/css/elementUI.cb02cf65.css           235.19 KiB       34.85 KiB

  dist/css/user.59bdb936.css                19.23 KiB        5.49 KiB

  dist/css/order.480f9cfb.css               18.01 KiB        2.18 KiB

  dist/css/list.752f1f66.css                17.29 KiB        2.54 KiB

  dist/css/app.15a328ee.css                 11.20 KiB        5.32 KiB

  dist/css/list~order~user.c0f64a3d.css     4.94 KiB         1.06 KiB
```

关于打包结果的提示： 可以配置：

```javascript
module.exports = {
    //webpack配置
 configureWebpack: {
     //关闭 webpack 的性能提示
     performance: {
      hints:false
     }

     //或者 修改提示阙值
     //警告 webpack 的性能提示
     performance: {
      hints:'warning',
      //入口起点的最大体积
      maxEntrypointSize: 50000000,
      //生成文件的最大体积
      maxAssetSize: 30000000,
      //只给出 js 文件的性能提示
      assetFilter: function(assetFilename) {
       return assetFilename.endsWith('.js');
      }
     }
    }
```

## 6 尝试用compression-webpack-plugin 对代码再次压缩，不过需要配置服务器就没有采用了

```javascript
const productionGzipExtensions = /\.(js|css)$/i;
 new CompressionPlugin({
          filename: '[path].gz[query]',
          algorithm: 'gzip',
          test: productionGzipExtensions,
          threshold: 10240,
          minRatio: 0.8,
          deleteOriginalAssets: true
        }),
```

## 7 提高构建速度

HardSourceWebpackPlugin 为模块提供中间缓存，缓存默认的存放路径是: node_modules/.cache/hard-source。

配置 hard-source-webpack-plugin，首次构建时间没有太大变化，但是第二次开始，构建时间大约可以节约 80%。

```javascript
//webpack.config.js
var HardSourceWebpackPlugin = require('hard-source-webpack-plugin');
module.exports = {
    //...
    plugins: [
        new HardSourceWebpackPlugin()
    ]
}
```

使用后打包时间第一次30多秒，之后都差不多为8、9秒。