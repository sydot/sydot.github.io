---
layout: post
title: 从Nuxt目录看Nuxt-js原理
date: 2018-02-05 15:30:20
tags: 
    - Nuxt-js
    - Vue-js
    - Vue-router
comments: true
---
[Nuxt.js](https://zh.nuxtjs.org/guide)是一个基于Vue.js的高度集成的SSR框架，可以通过[这篇文章](https://juejin.im/entry/5a6eb67e5188257340268005)快速理解Vue SSR。Nuxt.js集成了Vue.js,Webpack和Babel，并已经做好了相关配置，基本上是开箱即用

![nuxt简介](从Nuxt目录看Nuxt-js原理/nuxt-js.png)

Nuxt的npm包（node_modules->nuxt->lib）主要包含了两部分：builder和core，分别输出{Builder, Generator}和{Nuxt, Module, Renderer, Options, Utils}方法。从下面的代码可以看出，[上篇文章](https://sydot.github.io/2018/02/01/%E4%BB%8E%E9%85%8D%E7%BD%AE%E7%9C%8BNuxt-js%E5%8E%9F%E7%90%86/)中提到的nuxt.config.js配置项就是首先被传给core中的Nuxt类，然后将生成的实例传递给Builder类，并最终输出到.nuxt目录当中的。

<!-- more -->
```javascript
const { Nuxt, Builder } = require('nuxt')

let config = require('./nuxt.config.js')
config.dev = (process.env.NODE_ENV !== 'production')

let nuxt = new Nuxt(config)

if (config.dev) {
  new Builder(nuxt).build()
}
```

Nuxt的[目录结构](https://zh.nuxtjs.org/guide/directory-structure)如下：
+ assets---用于组织未编译的静态资源如 LESS、SASS 或 JavaScript，会被Webpack编译处理
+ components---用于组织应用的Vue.js组件，不会被Nuxt扩展
+ layouts---用于组织应用的布局组件，在布局文件中添加 <nuxt/> 组件用于显示页面的主体内容。
+ middleware---目录用于存放应用的中间件，将在每个路由改变时被调用
+ pages---用于组织应用的路由及视图，该目录下的.vue文件会被Nuxt扩展，且生成对应的路由
+ plugins---用于组织那些需要在根vue.js应用实例化之前需要运行的 Javascript 插件
+ static---用于存放应用的静态文件，不会被Webpack编译处理
+ store---目录用于组织应用的Vuex状态树文件

其中assets,components,middleware,plugins,static,store均不是项目运行必要的，只有layouts,pages是必要的。在这里分析一下pages生成[路由](https://zh.nuxtjs.org/guide/routing#%E5%8A%A8%E6%80%81%E8%B7%AF%E7%94%B1)的过程

## pages
页面组件实际上是Vue组件，只不过Nuxt.js为这些组件添加了一些特殊的[配置项](https://zh.nuxtjs.org/api)。以[官网示例](https://zh.nuxtjs.org/guide/routing#%E5%8A%A8%E6%80%81%E8%B7%AF%E7%94%B1)中的动态路由为例，对于如下的目录结构
```
pages/
--| _slug/
-----| comments.vue
-----| index.vue
--| users/
-----| _id.vue
--| index.vue
```
在builder.js中将所有pages路径下的.vue/.js文件传入createRoutes当中处理
```javascript
  if (this._nuxtPages) {
    // Use nuxt.js createRoutes bases on pages/
    const files = {};
    (await glob('pages/**/*.{vue,js}', {
      cwd: this.options.srcDir,
      ignore: this.options.ignore
    })).forEach(f => {
      const key = f.replace(/\.(js|vue)$/, '')
      if (/\.vue$/.test(f) || !files[key]) {
        files[key] = f
      }
    })
    templateVars.router.routes = createRoutes(
      Object.values(files),
      this.options.srcDir
    )
  } else {
    templateVars.router.routes = this.options.build.createRoutes(
      this.options.srcDir
    )
  } 
```
得到
```javacript
files = {
  'pages/index': 'pages/index.vue',
  'pages/_slug/comments': 'pages/_slug/comments.vue',
  'pages/_slug/index': 'pages/_slug/index.vue',
  'pages/users/_id': 'pages/users/_id.vue'
}
Object.values(files) = ['pages/index.vue', 'pages/_slug/comments.vue', 'pages/_slug/index.vue', 'pages/users/_id.vue']
```
在common/utils.js中处理files列表，并得到对应的route对象，传入cleanChildrenRoutes中生成最终的路由配置表
```javascript
exports.createRoutes = function createRoutes(files, srcDir) {
  let routes = []
  files.forEach(file => {
    let keys = file
      .replace(/^pages/, '')
      .replace(/\.(vue|js)$/, '')
      .replace(/\/{2,}/g, '/')
      .split('/')
      .slice(1)
    let route = { name: '', path: '', component: r(srcDir, file) }
    let parent = routes
    keys.forEach((key, i) => {
      route.name = route.name
        ? route.name + '-' + key.replace('_', '')
        : key.replace('_', '')
      route.name += key === '_' ? 'all' : ''
      route.chunkName = file.replace(/\.(vue|js)$/, '')
      let child = _.find(parent, { name: route.name })
      if (child) {
        child.children = child.children || []
        parent = child.children
        route.path = ''
      } else {
        if (key === 'index' && i + 1 === keys.length) {
          route.path += i > 0 ? '' : '/'
        } else {
          route.path += '/' + (key === '_' ? '*' : key.replace('_', ':'))
          if (key !== '_' && key.indexOf('_') !== -1) {
            route.path += '?'
          }
        }
      }
    })
    // Order Routes path
    parent.push(route)
    parent.sort((a, b) => {
      // Order: /static, /index, /:dynamic
      // Match exact route before index: /login before /index/_slug
      ...
    }
  })
  return cleanChildrenRoutes(routes)
}
```
上述路由被处理为：
```javacript
routes = [{
  chunkName: 'pages/index',
  component: 'pages/index.vue',
  name: 'index',
  path: '/'
}, {
  chunkName: 'pages/users/_id',
  component: 'pages/users/_id.vue',
  name: 'users-id',
  path: '/users/:id?'
}, {
  chunkName: 'pages/_slug/index',
  component: 'pages/_slug/index.vue',
  name: 'slug-index',
  path: '/:slug?'
}, {
  chunkName: 'pages/_slug/comments',
  component: 'pages/_slug/comments.vue',
  name: 'slug-comments',
  path: '/:slug?/comments'
}]
```
cleanChildrenRoutes()最终处理如下，返回结果即为templateVars.router.routes = routes
```javascript
function cleanChildrenRoutes(routes, isChild = false) {
  let start = -1
  let routesIndex = []
  routes.forEach(route => {
    if (/-index$/.test(route.name) || route.name === 'index') {
      // 去除'slug-index'中的'-index',将带有index的route记录在routesIndex中
      let res = route.name.split('-')
      let s = res.indexOf('index')
      start = start === -1 || s < start ? s : start
      routesIndex.push(res)
    }
  })
  routes.forEach(route => {
    route.path = isChild ? route.path.replace('/', '') : route.path
    if (route.path.indexOf('?') > -1) {
      let names = route.name.split('-')
      let paths = route.path.split('/')
      if (!isChild) {
        paths.shift()
      } // clean first / for parents
      // 对于原本name中带有'index'的route，去除具有相同路径的route的path中的'?'
      // 去除'/:slug?'、'/:slug?/comments'中的'?'，但不去除'/users/:id?'当中的'?'
      routesIndex.forEach(r => {
        let i = r.indexOf('index') - start //  children names
        if (i < paths.length) {
          for (let a = 0; a <= i; a++) {
            if (a === i) {
              paths[a] = paths[a].replace('?', '')
            }
            if (a < i && names[a] !== r[a]) {
              break
            }
          }
        }
      })
      route.path = (isChild ? '' : '/') + paths.join('/')
    }
    route.name = route.name.replace(/-index$/, '')
    if (route.children) {
      if (route.children.find(child => child.path === '')) {
        delete route.name
      }
      route.children = cleanChildrenRoutes(route.children, true)
    }
  })
  return routes
}
```
得到与[官网示例](https://zh.nuxtjs.org/guide/routing#%E5%8A%A8%E6%80%81%E8%B7%AF%E7%94%B1)一致的路由配置表
```javacript
routes = [{
  chunkName: 'pages/index',
  component: 'pages/index.vue',
  name: 'index',
  path: '/'
}, {
  chunkName: 'pages/users/_id',
  component: 'pages/users/_id.vue',
  name: 'users-id',
  path: '/users/:id?'
}, {
  chunkName: 'pages/_slug/index',
  component: 'pages/_slug/index.vue',
  name: 'slug',
  path: '/:slug'
}, {
  chunkName: 'pages/_slug/comments',
  component: 'pages/_slug/comments.vue',
  name: 'slug-comments',
  path: '/:slug/comments'
}]
```
templateVars最终被写入了.nuxt文件当中
```javascript
// Interpret and move template files to .nuxt/
await Promise.all(
  // templatesFiles是最后生成的.nuxt文件目录
  templatesFiles.map(async ({ src, dst, options, custom }) => {
    // Add template to watchers
    this.options.build.watch.push(src)
    // Render template to dst
    const fileContent = await readFile(src, 'utf8')
    let content
    try {
      const template = _.template(fileContent, {
        imports: {
          serialize,
          hash,
          r,
          wp,
          wChunk,
          resolvePath: this.nuxt.resolvePath.bind(this.nuxt),
          resolveAlias: this.nuxt.resolveAlias.bind(this.nuxt),
          relativeToBuild: this.relativeToBuild
        }
      })
      content = template(
        Object.assign({}, templateVars, {
          options: options || {},
          custom,
          src,
          dst
        })
      )
    } catch (err) {
      /* istanbul ignore next */
      throw new Error(`Could not compile template ${src}: ${err.message}`)
    }
    const path = r(this.options.buildDir, dst)
    // Ensure parent dir exits
    await mkdirp(dirname(path))
    // Write file
    await writeFile(path, content, 'utf8')
  })
)
```
由templateVars传入的router.routes替换了lib->app->router.js中的模板字符串(templateVars有一个属性appPath: './App.js',即读取了npm包中app目录下的文件)
```javascript
const _components = []
const _routes = recursiveRoutes(router.routes, '\t\t', _components)
...
export function createRouter () {
  return new Router({
    mode: '<%= router.mode %>',
    base: '<%= router.base %>',
    linkActiveClass: '<%= router.linkActiveClass %>',
    linkExactActiveClass: '<%= router.linkExactActiveClass %>',
    scrollBehavior,
    routes: [
<%= _routes %>
    ],
    <% if (router.parseQuery) { %>parseQuery: <%= serialize(router.parseQuery).replace('parseQuery(', 'function(') %>,<% } %>
    <% if (router.stringifyQuery) { %>stringifyQuery: <%= serialize(router.stringifyQuery).replace('stringifyQuery(', 'function(') %>,<% } %>
    fallback: <%= router.fallback %>
  })
}
```
最终我们可以在.nuxt/router.js中看到由pages目录结构自动生成vue-router模块的路由配置

## 总结
可以看出nuxt.js的处理过程基本上是：
读取pages下所有文件->生成files文件列表->处理动态、嵌套路由->整理、生成routes列表->去除不必要的'index'和'?'
->存储在templateVars.router.routes中->写入模板代码router.js当中->输出.nuxt文件夹