---
layout: post
title: Autoprefixer解析
date: 2018-03-04 13:51:02
tags: 
    - css
    - autoprefixer
    - plugin
    - 源码
comments: true
---
在上篇文章[postcss的插件解析](https://sydot.github.io/2018/02/25/postcss%E7%9A%84%E6%8F%92%E4%BB%B6%E8%A7%A3%E6%9E%90/)中以Autoprefixer为例，介绍了postcss对于插件的处理流程。这次主要分析Autoprefixer的工作原理。

[Autoprefixer](https://github.com/postcss/autoprefixer)是一个CSS文件的后处理程序，使用[Can I Use](https://caniuse.com/)的数据来自动添加浏览器前缀到CSS规则当中，首先来看一下示例代码：

```css
:fullscreen a {
    display: flex
}
```

以上代码会被自动编译为：

```css
:-webkit-full-screen a {
    display: -webkit-box;
    display: flex
}
:-moz-full-screen a {
    display: flex
}
:-ms-fullscreen a {
    display: -ms-flexbox;
    display: flex
}
:fullscreen a {
    display: -webkit-box;
    display: -ms-flexbox;
    display: flex
}
```
可以看到Autoprefixer为了兼容各浏览器所做的工作：给选择器（例如`:fullscreen`和`::selector`）、属性（例如`flex`）、函数（例如`calc()`）以及@规则（例如`@supports`和`@keyframes`）等加上对应的前缀，并插入到当前节点之前。此外，Autoprefixer还负责清除不必要的浏览器前缀等工作。

<!--more -->

## 整体分析
Autoprefixer运行的主函数是autoprefixer.js中的plugin函数，代码如下：
```js
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
```
在没有传入options选项，且Autoprefixer对css文件有加工的情况下，相当于如下代码：
```js
plugin = function(css, result) {
  var prefixes, ref;
  prefixes = loadPrefixes({
    from: css.source.input.file,
    env: options.env
  });
  // 清除不需要的前缀
  prefixes.processor.remove(css);
  // 增添需要的前缀
  return prefixes.processor.add(css, result);
};
```

显然，这里的loadPrefixes函数创建了一个包含所有需要的浏览器前缀列表，以及在processor属性中用于加工AST树的各类回调函数的prefixes对象，随后调用该对象中的remove、add等函数对传入的css AST树进行处理。

```js
loadPrefixes = function(opts) {
  var browsers, key, stats;
  stats = options.stats;
  // 根据browserslist(https://github.com/ai/browserslist)获取的需要兼容的浏览器列表
  browsers = new Browsers(module.exports.data.browsers, reqs, opts, stats);
  // 根据当前配置得到一个key值
  key = browsers.selected.join(', ') + JSON.stringify(options);
  // 缓存该key值对应的prefixes对象
  return cache[key] || (cache[key] = new Prefixes(module.exports.data.prefixes, browsers, options));
};

module.exports.data = {
  browsers: require('caniuse-db/data.json').agents,
  prefixes: require('../data/prefixes')
};
```

## 获取前缀

首先来看传入Prefixes函数中的第一个参数，即`require('../data/prefixes')`,该函数会返回一个result对象，存储每个css名称对应的prefixes列表，这里以`:fullscreen`为例介绍其流程：

```js
// prefixes.js
(function() {
  ...
  fullscreen = require('caniuse-db/features-json/fullscreen.json');

  feature(fullscreen, function(browsers) {
    return prefix(':fullscreen', {
      selector: true,
      browsers: browsers,
      feature: 'fullscreen'
    });
  });

  feature = function(data, opts, callback) {
    var browser, match, need, ref, ref1, support, version, versions;
    if (!callback) {
      // callback = opts, opts = {}
      ref = [opts, {}], callback = ref[0], opts = ref[1];
    }
    match = opts.match || /\sx($|\s)/;
    need = [];
    ref1 = data.stats;
    for (browser in ref1) {
      versions = ref1[browser];
      for (version in versions) {
        support = versions[version];
        if (support.match(match)) {
          // 例如"ie 11"
          need.push(browser + ' ' + version);
        }
      }
    }
    return callback(sort(need));
  };
  ...
}).call(this);
```

其中`require('caniuse-db/features-json/fullscreen.json')`请求的就是针对`:fullscreen`的caniuse参数，其stats属性包含了所有浏览器版本中对`:fullscreen`的支持情况：

```json
"stats":{
  "ie":{
    "5.5":"n",
    "6":"n",
    "7":"n",
    "8":"n",
    "9":"n",
    "10":"n",
    "11":"a x #3"
  },
  "edge":{
    "12":"a #3",
    "13":"a #3",
    "14":"a #3",
    "15":"a #3",
    "16":"a #3",
    "17":"a #3"
  },
  ...
}
```
其中`n`表示No support，`a`表示Almost supported，`x`表示Requires prefi(**x**) to work，`#3`表示对应的注释编号。在这里显然只需通过正则表达式`/\sx($|\s)/`检测出含有`x`字段的版本名称即可。这样，我们给callback函数传入的就是所有支持`:fullscreen`的浏览器版本的已排序列表。

```js
(function() {
  ...
  result = {};

  prefix = function() {
    var data, i, j, k, len, name, names, results;
    // names = ['fullscreen'], data = {selector: true, browsers: browsers, feature: 'fullscreen'}
    names = 2 <= arguments.length ? slice.call(arguments, 0, j = arguments.length - 1) : (j = 0, []), data = arguments[j++];
    results = [];
    for (k = 0, len = names.length; k < len; k++) {
      name = names[k];
      // result['fullscreen'] = {}
      result[name] = {};
      results.push((function() {
        var results1;
        results1 = [];
        for (i in data) {
          // result['fullscreen'][selector] = true
          // result['fullscreen'][browsers] = browsers
          // result['fullscreen'][feature] = 'fullscreen'
          results1.push(result[name][i] = data[i]);
        }
        return results1;
      })());
    }
    return results;
  };
  module.exports = result;
  ...
}).call(this);
```

现在返回到lib/prefixes.js函数当中，上面返回的result对象就是下面的this.data对象

```js
function Prefixes(data1, browsers, options) {
  var ref;
  this.data = data1;
  this.browsers = browsers;
  this.options = options != null ? options : {};
  ref = this.preprocess(this.select(this.data)), this.add = ref[0], this.remove = ref[1];
  this.transition = new Transition(this);
  this.processor = new Processor(this);
}
Prefixes.prototype.select = function(list) {
  var add, all, data, name, notes, selected;
  selected = {
    add: {},
    remove: {}
  };
  for (name in list) {
    data = list[name];
    // 所有支持的浏览器版本
    add = data.browsers.map(function(i) {
      var params;
      params = i.split(' ');
      return {
        browser: params[0] + ' ' + params[1],
        note: params[2]
      };
    });
    ...
    // 配置中需要支持的浏览器版本的所有前缀
    add = add.filter((function(_this) {
      return function(i) {
        return _this.browsers.isSelected(i.browser);
      };
    })(this)).map((function(_this) {
      return function(i) {
        var prefix;
        prefix = _this.browsers.prefix(i.browser);
        if (i.note) {
          return prefix + ' ' + i.note;
        } else {
          return prefix;
        }
      };
    })(this));
    add = this.sort(utils.uniq(add));
    ...
    // 所有支持的浏览器版本的所有前缀
    all = data.browsers.map((function(_this) {
      return function(i) {
        return _this.browsers.prefix(i);
      };
    })(this));
    all = utils.uniq(all);
    ...
    if (add.length) {
      // selected.add['fullscreen']是配置对应浏览器版本的所有前缀
      selected.add[name] = add;
      if (add.length < all.length) {
        // selected.remove['fullscreen']是不在配置中的其他前缀
        selected.remove[name] = all.filter(function(i) {
          return add.indexOf(i) === -1;
        });
      }
    } else {
      selected.remove[name] = all;
    }
  }
  return selected;
};
```
最后在preprocess函数中返回ref数组，存储在this.add和this.remove当中
```js
Prefixes.prototype.preprocess = function(selected) {
  var a, add, j, k, l, len, len1, len2, len3, len4, len5, len6, m, n, name, o, old, olds, p, prefix, prefixed, prefixes, prop, props, ref, ref1, ref2, ref3, remove, selector, value, values;
  add = {
    selectors: [],
    '@supports': new Supports(Prefixes, this)
  };
  ref = selected.add;
  for (name in ref) {
    prefixes = ref[name];
    if (name === '@keyframes' || name === '@viewport') {
      add[name] = new AtRule(name, prefixes, this);
    } else if (name === '@resolution') {
      add[name] = new Resolution(name, prefixes, this);
    } else if (this.data[name].selector) {
      // 处理:fullscreen这样的选择器
      add.selectors.push(Selector.load(name, prefixes, this));
    } else {
      // 处理flex这样的属性
      props = this.data[name].props;
      if (props) {
        value = Value.load(name, prefixes, this);
        ...
    }
  }
  // remove = { selectors: [] } 同理
  ...
  return [add, remove];
}
```
最后得到add.selectors为一个数组，保存了该规则所有需要添加的前缀名称
```js
add = {
  selectors: [{
    name: 'fullscreen',
    prefixes: '-webkit-', '-moz-', '-ms',
    ...
  },
  ...]
}
```

## 获取方法
由于Selector函数是extend在Prefixer函数上的，所以上面调用的hack、load方法都是调用了Prefixer上的方法
```js
// prefixes.js
Selector = require('./selector');
Selector.hack(require('./hacks/fullscreen'));
Selector.hack(require('./hacks/placeholder'));

// prefixer.js
Prefixer.hack = function(klass) {
  var j, len, name, ref, results;
  this.hacks || (this.hacks = {});
  ref = klass.names;
  results = [];
  for (j = 0, len = ref.length; j < len; j++) {
    name = ref[j];
    // this.hacks[':fullscreen'] = require('./hacks/fullscreen')
    results.push(this.hacks[name] = klass);
  }
  return results;
};
Prefixer.load = function(name, prefixes, all) {
  var klass, ref;
  klass = (ref = this.hacks) != null ? ref[name] : void 0;
  if (klass) {
    return new klass(name, prefixes, all);
  } else {
    return new this(name, prefixes, all);
  }
};

// hacks/fullscreen.js
Fullscreen.names = [':fullscreen'];

Fullscreen.prototype.prefixed = function(prefix) {
  if ('-webkit-' === prefix) {
    return ':-webkit-full-screen';
  } else if ('-moz-' === prefix) {
    return ':-moz-full-screen';
  } else {
    return ":" + prefix + "fullscreen";
  }
};
```
所以，调用Selector.load(name, prefixes, this)实际上调用了new Fullscreen(name, prefixes, all)，而Fullscreen函数又是extend在Selector函数上的，覆盖了Selector上的prefixed方法

## 添加前缀

最后看一下在processor.prototype.add函数中如何将上述前缀添加上去（walkRules的原理见[postcss的插件解析](https://sydot.github.io/2018/02/25/postcss%E7%9A%84%E6%8F%92%E4%BB%B6%E8%A7%A3%E6%9E%90/)）。

```js
// processor.js
css.walkRules((function(_this) {
  return function(rule) {
    var j, len, ref, results, selector;
    if (_this.disabled(rule)) {
      return;
    }
    ref = _this.prefixes.add.selectors;
    results = [];
    // 遍历selector数组，检查是否与该条css规则相匹配
    for (j = 0, len = ref.length; j < len; j++) {
      selector = ref[j];
      results.push(selector.process(rule, result));
    }
    return results;
  };
})(this));

// prefixer.js
Prefixer.prototype.process = function(node) {
  var added, j, k, len, len1, parent, prefix, prefixes, ref;
  if (!this.check(node)) {
    return;
  }
  // 匹配则添加对应的前缀
  parent = this.parentPrefix(node);
  prefixes = [];
  ref = this.prefixes;
  for (j = 0, len = ref.length; j < len; j++) {
    prefix = ref[j];
    if (parent && parent !== utils.removeNote(prefix)) {
      continue;
    }
    prefixes.push(prefix);
  }
  added = [];
  for (k = 0, len1 = prefixes.length; k < len1; k++) {
    prefix = prefixes[k];
    if (this.add(node, prefix, added.concat([prefix]))) {
      added.push(prefix);
    }
  }
  return added;
};
// selector.js
Selector.prototype.add = function(rule, prefix) {
  var cloned, prefixeds;
  prefixeds = this.prefixeds(rule);
  if (this.already(rule, prefixeds, prefix)) {
    return;
  }
  // cloned即为添加了前缀的节点
  cloned = this.clone(rule, {
    selector: prefixeds[prefix]
  });
  // 在该节点之前添加clone节点
  return rule.parent.insertBefore(rule, cloned);
};

Selector.prototype.prefixeds = function(rule) {
  var i, len, prefix, prefixeds, ref;
  if (rule._autoprefixerPrefixeds) {
    return rule._autoprefixerPrefixeds;
  }
  prefixeds = {};
  ref = this.possible();
  for (i = 0, len = ref.length; i < len; i++) {
    // 获取转换后的prefixeds数组
    prefix = ref[i];
    prefixeds[prefix] = this.replace(rule.selector, prefix);
  }
  return rule._autoprefixerPrefixeds = prefixeds;
};

Selector.prototype.replace = function(selector, prefix) {
  return selector.replace(this.regexp(), '$1' + this.prefixed(prefix));
};
```
因而在add函数中实际添加的前缀是hack中替换的前缀`:-webkit-full-screen`，`:-moz-full-screen`以及未替换的前缀`-ms`

## 总结
梳理一下代码流程：首先是loadPrefixes函数根据caniuse的数据和传入的config参数得到需要添加的前缀——add对象和需要删除的前缀——remove对象，然后在add和remove函数中逐条匹配AST树上的css规则并进行处理，最后在该条规则之前，或是添加一个添加了前缀的克隆节点，或是删除匹配到的有多余前缀的节点。
