---
layout: post
title: 从配置看Nuxt.js原理
date: 2018-02-01 16:21:20
tags: 
    - Nuxt-js
    - Vue-js
    - Webpack
comments: true
---
[Nuxt.js](https://zh.nuxtjs.org/guide)是一个基于Vue.js的高度集成的SSR框架，可以通过[这篇文章](https://juejin.im/entry/5a6eb67e5188257340268005)快速理解Vue SSR。Nuxt.js集成了Vue.js,Webpack和Babel，并已经做好了相关配置，基本上是开箱即用

![nuxt简介](从配置看Nuxt-js原理/nuxt-js.png)

<!-- more -->

## 基本配置

[Nuxt.config.js](https://zh.nuxtjs.org/guide/configuration)是Nuxt.js的唯一配置项，所以如果需要特殊配置，只需要修改该文件来覆盖默认配置即可，包括：

+ build---webpack的构建配置
    - analyze---默认false,使用[webpack-bundle-analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer)分析打包文件
    - babel---对Vue组件使用[babel-preset-vue-app](https://github.com/vuejs/babel-preset-vue-app)加载和转义
    - extend---打包时代码用[eslint-loader](https://github.com/webpack-contrib/eslint-loader)清理
    - filenames---打包文件名为vendor.bundle.js
    - loaders---对图片、字体用[url-loader](https://doc.webpack-china.org/loaders/url-loader/)加载
    - plugins---无默认,配置的webpack插件
    - postcss---对css文件用[postcss-loader](http://postcss.org/)处理
    - publicPath---为项目中的所有资源指定统一基础路径为'/_nuxt/'
    - vendor---无默认,在vendor.bundle.js中添加第三方模块
+ cache---默认false,使用[Iru-cache](https://github.com/isaacs/node-lru-cache)缓存组件
+ css---无默认,引入全局的CSS文件、模块、库
+ dev---默认true,用于在编码时判断是开发环境还是生产环境
+ env---无默认,用来配置在客户端和服务端共享的环境变量
+ generate---配置生成静态站点的具体方式
    - dir---生成静态文件的目录名称为'dist'
    - minify---生成静态文件时对html文件用[html-minifier](https://github.com/kangax/html-minifier)压缩
    - routes---生成静态站点时忽略动态路由
+ head---使用[vue-meta](https://github.com/declandewet/vue-meta#recognized-metainfo-properties)配置应用的meta信息
+ loading---页面切换时，使用内置的加载组件显示加载进度条
    - Disable the Progress Bar---显示加载进度条
    - Customize the Progress Bar---进度条颜色为主题色‘#3B8070’
    - Use a Custom Loading Component---无默认,用自定义组件代替默认组件
+ performance---配置性能选项
    - gzip---使用[compression](https://github.com/expressjs/compression)模块压缩应用的所有资源
    - prefetch---在生产环境中使用浏览器的[prefetch](https://www.w3.org/TR/resource-hints/#dfn-prefetch)预加载页面的脚本资源
+ plugins---无默认，引入在plugins/目录下配置的Vue插件
+ rootDir---项目的目录名，node_modules 目录必须在 rootDir 目录内
+ router---[vue-router](https://router.vuejs.org/zh-cn/)的配置
    - base---url的基路径
    - mode---使用路由的[history模式](https://router.vuejs.org/zh-cn/essentials/history-mode.html)完成路由跳转
    - linkActiveClass---全局配置 [<nuxt-link>](https://zh.nuxtjs.org/api/components-nuxt-link) 组件默认的激活类名。
    - scrollBehavior---设置跳转至新页面时的滚动位置
    - middleware---无默认，为每个页面设置默认的[中间件](https://zh.nuxtjs.org/guide/routing#%E4%B8%AD%E9%97%B4%E4%BB%B6)
    - extendRoutes---无默认，扩展路由配置
+ srcDir---默认rootDir，用于在现有项目中集成Nuxt.js时(与node_modules/同级)修改nuxt.js目录名
+ transition---使用Vue.js的[transition](https://vuejs.org/v2/guide/transitions.html#Transition-Modes)组件设置页面切换过渡效果
+ watchers---热重载的事件监听功能
    - chokidar---无默认，在Mac OS X系统上改善Node.js监听文件的能力
    - webpack---热重载的轮询频率

## 示例
以下是[nuxtjs.org](https://github.com/nuxt/nuxt.js)的nuxt.config.js配置,只设置了head,loading,build三项配置，先来看看它是怎么成功运行的
```javascript
module.exports = {
  head: {
    title: 'nuxt-test', // 用脚手架生成时的设置的文件名
    meta: [
      { charset: 'utf-8' },
      { name: 'viewport', content: 'width=device-width, initial-scale=1' },
      { hid: 'description', name: 'description', content: 'Nuxt.js project' }
    ],
    link: [
      { rel: 'icon', type: 'image/x-icon', href: '/favicon.ico' }
    ]
  },
  loading: { color: '#3B8070' },
  build: {
    extend (config, { isDev, isClient }) {
      if (isDev && isClient) {
        config.module.rules.push({
          enforce: 'pre',
          test: /\.(js|vue)$/,
          loader: 'eslint-loader',
          exclude: /(node_modules)/
        })
      }
    }
  }
}
```

## head配置
首先来看head配置，它用到了[vue-meta](https://github.com/declandewet/vue-meta#recognized-metainfo-properties)来最终转换成这样的结果：
```javascript
<meta data-n-head="true" charset="utf-8">
<meta data-n-head="true" name="viewport" content="width=device-width, initial-scale=1">
<meta data-n-head="true" data-hid="description" name="description" content="Nuxt.js project">
<title data-n-head="true">nuxt-test</title>
<link data-n-head="true" rel="icon" type="image/x-icon" href="/favicon.ico">
```
显然data-n-head和data-hid是nuxt.js传入给vue-meta的，在.nuxt/index.js有如下配置
```javascript
import Meta from 'vue-meta'

Vue.use(Meta, {
  keyName: 'head', // vue-meta会在app的'head'属性中寻找对应的meta标签
  attribute: 'data-n-head', // 给所有vue-meta监听的标签加入'data-n-head'属性
  ssrAttribute: 'data-n-head-ssr', // 给已经完成服务器渲染的标签加入'data-n-head-ssr'属性，从而使vue-meta获知
  tagIDKeyName: 'hid' // 检测所有带有'hid'属性的标签,确保他们是唯一的
})
```
这样就说明了额外属性的来源，那么app的'head'属性是从哪里传入的呢?
```javascript
// index.js
import App from './App.js'

async function createApp (ssrContext) {
  const app = {
    ...
    ...App
  }
}
// App.js
export default {
  head: {"title":"nuxt-test","meta":[{"charset":"utf-8"},{"name":"viewport","content":"width=device-width, initial-scale=1"},{"hid":"description","name":"description","content":"Nuxt.js project"}],"link":[{"rel":"icon","type":"image\u002Fx-icon","href":"\u002Ffavicon.ico"}],"style":[],"script":[]},
  ...
}
```
显然，位于nuxt.config.js中的head配置被写入App.js，存储在了app根组件的'head'属性当中，最终渲染时被vue-meta转成meta头

## loading配置
同理，loading配置也是被注入到了components/nuxt-loading.vue的data当中，并被渲染到了App.js的根dom元素下
```javascript
// components/nuxt-loading.vue
<template>
  <div class="nuxt-progress"/>
</template>
import Vue from 'vue'

export default {
  name: 'nuxt-loading',
  data () {
    return {
      percent: 0,
      show: false,
      canSuccess: true,
      duration: 5000,
      height: '2px',
      color: '#3B8070',
      failedColor: 'red',
    }
  },
  ...
}
// App.js
import NuxtLoading from './components/nuxt-loading.vue'

export default {
  render(h, props) {
    const loadingEl = h('nuxt-loading', { ref: 'loading' })
    ...
    return h('div',{
      domProps: {
        id: '__nuxt'
      }
    }, [
      loadingEl,
      transitionEl
    ])
  }
  ...
  components: {
    NuxtLoading
  }
}
```
h是Vue的render()方法中createElement的简写，为[通用惯例](https://github.com/vuejs/babel-plugin-transform-vue-jsx/issues/6),在页面中渲染结果为：
```javascript
<div id="__nuxt">
  <div class="nuxt-progress">
  ...
</div>
```

## build配置
build模块是webpack的构建配置，所以在执行nuxt指令时就已经配置好了，在node_modules->nuxt->lib->builder中可以看到相关的配置

以extend方法为例，在builder->webpack->client.config.js/server.config.js当中可以看到对应的处理
```javascript
// client.config.js
  let config = base.call(this, { name: 'client', isServer: false })
  ...
  if (typeof this.options.build.extend === 'function') {
    const isDev = this.options.dev
    const extendedConfig = this.options.build.extend.call(this, config, {
      get dev() {
        printWarn('dev has been deprecated in build.extend(), please use isDev')
        return isDev
      },
      isDev,
      isClient: true
    })
    // Only overwrite config when something is returned for backwards compatibility
    if (extendedConfig !== undefined) {
      config = extendedConfig
    }
  }
// server.config.js
  let config = base.call(this, { name: 'server', isServer: true })
  ...
  if (typeof this.options.build.extend === 'function') {
    const isDev = this.options.dev
    const extendedConfig = this.options.build.extend.call(this, config, {
      get dev() {
        printWarn('dev has been deprecated in build.extend(), please use isDev')
        return isDev
      },
      isDev,
      isServer: true
    })
    // Only overwrite config when something is returned for backwards compatibility
    if (extendedConfig !== undefined) {
      config = extendedConfig
    }
  }
```
在base.config.js当中定义了config.module.rules,并已经封装好了以下loaders
+ vue-loader
+ babel-loader
+ less-loader
+ sass-loader
+ stylus-loader
+ url-loader
+ file-loader

在extend方法中扩展的eslint-loader只在出口处client.config.js/server.config.js才被加入rules列表当中,这就符合了官网所说的，[extend](https://zh.nuxtjs.org/api/configuration-build)方法会被调用两次，一次在服务端打包构建的时候，另外一次是在客户端打包构建的时候。

最后，build配置项会统一在builder.js当中处理，并传递给webpack
```javascript
  async webpackBuild() {
    debug('Building files...')
    const compilersOptions = []

    // Client
    const clientConfig = clientWebpackConfig.call(this)
    compilersOptions.push(clientConfig)
    
    // Server
    let serverConfig = null
    if (this.options.build.ssr) {
      serverConfig = serverWebpackConfig.call(this)
      compilersOptions.push(serverConfig)
    }
    ...
    this.compilers = compilersOptions.map(compilersOption => {
      const compiler = webpack(compilersOption)
      ...
      return compiler
    })
    ...
  }
```

## 总结
虽然对nuxt.config.js中各项配置处理方法各不相同，但可以看到整体的思路是一致的，基本上是：
写入config配置->调用nuxt指令->传入builder模块->处理->输出.nuxt目录->输出页面
这种方式的优势就在于集成度相当高，前端可以不需要自己搭建一套SSR程序，只需通过配置文件nuxt.config.js就可以管理多个程序组件之间的关系。