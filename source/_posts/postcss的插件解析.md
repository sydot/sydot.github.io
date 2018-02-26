---
layout: post
title: postcss的插件解析
date: 2018-02-25 22:00:43
tags: 
    - postcss
    - css
    - autoprefixer
    - plugin
    - 源码
comments: true
---

postcss对于其插件的解释是这样的：postcss插件是一个从postcss parser中接收并转换CSS AST树的函数。在postcss源码中有专门针对postcss插件的API文档（docs->guidelines->plugin.md）。下面以postcss中最流行的插件Autoprefixer为例，讨论一下postcss对于插件的处理流程。

[Autoprefixer](https://github.com/postcss/autoprefixer)是一个CSS文件的后处理程序，使用[Can I Use](https://caniuse.com/)的数据来自动添加浏览器前缀到CSS规则当中，首先来看一下官网的示例：

```js
a {
    display: flex;
}
```

以上代码会被自动编译为：

```js
a {
    display: -webkit-box;
    display: -webkit-flex;
    display: -ms-flexbox;
    display: flex;
}
```

显然，这种方案比起在预处理器中混入兼容属性的方案更为简洁明了，只需在打包工具下引入对应的插件即可。


<!-- more -->

## 在postcss中使用插件

仍以webpack的配置为例，参考文章[从源码看postcss](https://sydot.github.io/2018/02/08/%E4%BB%8E%E6%BA%90%E7%A0%81%E7%9C%8Bpostcss/)中的示例，通过postcss-loader引入：

```js
// webpack.config.js
module.exports = {
  module: {
    loaders: [
      {
        test:   /\.css$/,
        loader: "style-loader!css-loader!postcss-loader"
      }
    ]
  }
}

// postcss.config.js
module.exports = {
  plugins: [
    require('autoprefixer')
  ]
}
```

在postcss中对于插件的处理无论是同步还是异步，最终都是通过lazy-result.js中的run函数处理的

```js
LazyResult.prototype.run = function run(plugin) {
  this.result.lastPlugin = plugin; // 调用了require('autoprefixer')

  try {
    return plugin(this.result.root, this.result); // this.result.root为postcss解析的AST树
  } catch (error) {
    this.handleError(error, plugin);
    throw error;
  }
};
```

这里的plugin函数并不是postcss自带的，那是怎么来的呢？这要看Autoprefixer的源码（lib->autoprefixer.js）

```js
(function() {
  ...
  postcss = require('postcss');

  module.exports = postcss.plugin('autoprefixer', function() {
    ...
    plugin = function(css, result) {
      var prefixes, ref;
      prefixes = loadPrefixes({
        from: (ref = css.source) != null ? ref.input.file : void 0,
        env: options.env
      });
      // 检测是否需要prefixes
      timeCapsule(result, prefixes);

      if (options.remove !== false) {
        prefixes.processor.remove(css);
      }
      if (options.add !== false) {
        return prefixes.processor.add(css, result);
      }
    };
    plugin.options = options;
    plugin.info = function(opts) {
      return require('./info')(loadPrefixes(opts));
    };
    return plugin;
  });
  ...
}).call(this);
```

原来是通过IIFE的形式输出了一个postcss.plugin函数调用，并且返回了一个plugin方法绑定到this对象上。

```js
// postcss.js
postcss.plugin = function plugin(name, initializer) {
  var creator = function creator() {
    var transformer = initializer.apply(undefined, arguments);
    transformer.postcssPlugin = name;
    transformer.postcssVersion = new _processor2.default().version;
    return transformer;
  };

  var cache = void 0;
  Object.defineProperty(creator, 'postcss', {
    get: function get() {
      if (!cache) cache = creator();
      return cache;
    }
  });

  creator.process = function (css, processOpts, pluginOpts) {
    return postcss([creator(pluginOpts)]).process(css, processOpts);
  };

  return creator;
};
```

可以看出绑定后的this.plugin作为对象和函数大致有以下属性：

```js
this.plugin = {
  options: undefined, // 传入的options
  info: function(opts) {
    return require('./info')(loadPrefixes(opts));
  },
  postcssPlugin: 'autoprefixer', // LazyResult.prototype.handleError()中用到
  postcssVersion: '6.0.14', // LazyResult.prototype.handleError()中用到
  process: function (css, processOpts, pluginOpts) {
    return postcss([creator(pluginOpts)]).process(css, processOpts);
  },
  
  plugin: function(css, result) {
  var prefixes, ref;
  prefixes = loadPrefixes({
    from: (ref = css.source) != null ? ref.input.file : void 0,
    env: options.env
  });
  timeCapsule(result, prefixes);
  if (options.remove !== false) {
    prefixes.processor.remove(css);
  }
  if (options.add !== false) {
    return prefixes.processor.add(css, result);
  }
}
```

此后就可以在postcss中获得该插件处理后的css文件以及插件的有关信息了。

## postcss的公共API

根据约定，postcss插件应该不依赖于任何非公共的属性和方法，而是使用[API文档](http://api.postcss.org/)中的公共API对AST树进行改动




