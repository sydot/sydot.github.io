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

```css
a {
    display: flex;
}
```

以上代码会被自动编译为：

```css
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

根据约定，postcss插件应该不依赖于任何非公共的属性和方法，而是使用[API文档](http://api.postcss.org/)中的公共API对AST树进行改动，这些规则定义在postcss源码的container.js当中

例如，在prefixes.processor.add(css, result)当中就用到了postcss当中的walkAtRules函数，对AST树中的所有type类型为'atrule'的规则进行遍历，并逐一处理。

```js
// 停止遍历的条件是return false，不是undefined
css.walkAtRules(function (rule) {
  if (rule.name === 'keyframes') {
    if (!_this.disabled(rule, result)) {
      return keyframes && keyframes.process(rule);
    }
  } else if (rule.name === 'viewport') {
    if (!_this.disabled(rule, result)) {
      return viewport && viewport.process(rule);
    }
  } else if (rule.name === 'supports') {
    if (_this.prefixes.options.supports !== false && !_this.disabled(rule, result)) {
      return supports.process(rule);
    }
  } else if (rule.name === 'media' && rule.params.indexOf('-resolution') !== -1) {
    if (!_this.disabled(rule, result)) {
      return resolution && resolution.process(rule);
    }
  }
  return undefined;
});
```

在文章[postcss的AST解析](https://sydot.github.io/2018/02/16/postcss%E7%9A%84AST%E8%A7%A3%E6%9E%90/)中总结了AST树中的type为'atrule'的node节点的一般规律，即'atrule'节点额外包含name, params属性。因而我们能在callback中判断其具体类型

下面来看一下postcss源码中是怎么获取每一个'atrule'节点的

```js
// container.js
Container.prototype.walkAtRules = function walkAtRules(name, callback) {
  if (!callback) {
    callback = name;
    return this.walk(function (child, i) {
      // child即为一个node节点
      if (child.type === 'atrule') {
        return callback(child, i);
      }
    });
  } else if (name instanceof RegExp) {
    return this.walk(function (child, i) {
      if (child.type === 'atrule' && name.test(child.name)) {
        return callback(child, i);
      }
    });
  } else {
    return this.walk(function (child, i) {
      if (child.type === 'atrule' && child.name === name) {
        return callback(child, i);
      }
    });
  }
};

Container.prototype.walk = function walk(callback) {
  return this.each(function (child, i) {
    var result = callback(child, i);
    // BFS 遍历所有AST的子节点
    if (result !== false && child.walk) {
      result = child.walk(callback);
    }
    return result;
  });
};

Container.prototype.each = function each(callback) {
  if (!this.lastEach) this.lastEach = 0;
  if (!this.indexes) this.indexes = {};

  this.lastEach += 1;
  var id = this.lastEach;
  this.indexes[id] = 0;

  if (!this.nodes) return undefined;

  var index = void 0,
      result = void 0;
  // 遍历一个nodes节点中的所有子节点
  while (this.indexes[id] < this.nodes.length) {
    index = this.indexes[id];
    result = callback(this.nodes[index], index);
    if (result === false) break;

    this.indexes[id] += 1;
  }

  delete this.indexes[id];

  return result;
};
```
postcss提供的walkAtRules,walkDecls,walkRules函数分别对应着AST树中type类型为'atrule','decl','rule'的所有非根节点，为postcss插件对于AST树的访问提供了极大的便利

## 插件处理

上述的walkAtRules函数最终会调用at-rule.js中的process函数来处理匹配到的'atrule'节点

```js
AtRule.prototype.process = function process(node) {
  var parent = this.parentPrefix(node);
  // this.prefixes就是要添加的前缀的集合
  for (var _iterator = this.prefixes, _isArray = Array.isArray(_iterator), _i = 0, _iterator = _isArray ? _iterator : _iterator[Symbol.iterator]();;) {
    var _ref;

    if (_isArray) {
      if (_i >= _iterator.length) break;
      _ref = _iterator[_i++];
    } else {
      _i = _iterator.next();
      if (_i.done) break;
      _ref = _i.value;
    }

    var prefix = _ref;
    // 逐一添加前缀
    if (!parent || parent === prefix) {
      this.add(node, prefix);
    }
  }
};

AtRule.prototype.add = function add(rule, prefix) {
  var prefixed = prefix + rule.name;

  var already = rule.parent.some(function (i) {
    return i.name === prefixed && i.params === rule.params;
  });
  if (already) {
    return undefined;
  }
  // cloned即为添加了前缀的节点
  var cloned = this.clone(rule, { name: prefixed });
  // 在该节点之前添加clone节点
  return rule.parent.insertBefore(rule, cloned);
};
```

## 总结
梳理一下代码流程，首先是在postcss中调用run函数，然后在插件的plugin函数中处理AST树，通过postcss的公共API获取node节点信息，最后插件返回处理完的AST树~