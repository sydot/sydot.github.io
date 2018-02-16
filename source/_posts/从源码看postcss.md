---
layout: post
title: 从源码看postcss
date: 2018-02-08 14:32:50
tags: 
    - postcss
    - css
    - 源码
comments: true
---
[PostCSS](https://github.com/postcss/postcss)是一个允许使用JS插件转换样式的工具。这些插件可以检查（lint）你的 CSS，支持 CSS Variables 和 Mixins，编译尚未被浏览器广泛支持的先进的 CSS语法，内联图片，以及其它很多优秀的功能。

PostCSS本身只做三件事：
  1. 将CSS解析成抽象语法树(AST树)
  2. 将AST树"传递"给任意数量的插件处理
  3. 将处理完毕的AST树重新转换成字符串

以webpack的配置为例，在 webpack.config.js 里使用 [postcss-loader](https://github.com/postcss/postcss-loader) :

<!-- more -->
```js
module.exports = {
  module: {
    rules: [
      {
        test: /\.css$/,
        exclude: /node_modules/,
        use: [
          {
            loader: 'style-loader',
          },
          {
            loader: 'css-loader',
            options: {
              importLoaders: 1,
            }
          },
          {
            loader: 'postcss-loader'
          }
        ]
      }
    ]
  }
}
```
然后创建 postcss.config.js:
```js
module.exports = {
  plugins: [
    require('precss'),
    require('autoprefixer')
  ]
}
```
这样，postcss就会以解析.css后缀的文件->传递给precss、autoprefixer处理->返回处理完毕的.css文件这样的顺序来工作了

具体来看node_modules->postcss-loader->lib->index.js的处理
```js
const postcss = require('postcss')
...
module.exports = function loader (css, map, meta) {
  ...
  Promise.resolve().then(() => {
    // 读取config
    ...
  }).then((config) => {
    // 处理config
    // plugins = config.plugins || []
    // options = config.options && 其他属性，包括：
    // syntax: 一个提供了语法解释器和 stringifier 的对象。
    // parser: 一个特殊的语法解释器（例如 SCSS）。
    // stringifier: 一个特殊的语法 output 生成器（例如 Midas）。
    // map: source map 选项.
    // from: input 文件名称（大多数运行器自动设置了这个）。
    // to: output 文件名称（大多数运行器自动设置了这个）。
    ...
    return postcss(plugins)
      .process(css, options)
      .then((result) => {
        //处理返回结果
        ...
      })
  }).catch((err) => {
    if (err.file) this.addDependency(err.file)

    return err.name === 'CssSyntaxError' ? cb(new SyntaxError(err)) : cb(err)
  })
}
```
即最重要的一步是把plugins列表传递给postcss()，然后在回调的process()中传入css文件和options选项，进行上述的三个步骤的处理。这个过程可能看[JS API](https://github.com/postcss/postcss/blob/master/README.cn.md#js-api)会更清晰些
```js
const fs = require('fs');
const postcss = require('postcss');
const precss = require('precss');
const autoprefixer = require('autoprefixer');

fs.readFile('src/app.css', (err, css) => {
    postcss([precss, autoprefixer])
        .process(css, { from: 'src/app.css', to: 'dest/app.css' })
        .then(result => {
            fs.writeFile('dest/app.css', result.css);
            if ( result.map ) fs.writeFile('dest/app.css.map', result.map);
        });
});
```
接下来在node_modules->postcss->lib中看下具体是怎么处理的(ES5的源码，并且修改过很多次了，看起来还是有些繁琐的)
```js
// postcss.js
function postcss() {
  for (var _len = arguments.length, plugins = Array(_len), _key = 0; _key < _len; _key++) {
    plugins[_key] = arguments[_key];
  }
  if (plugins.length === 1 && Array.isArray(plugins[0])) {
    plugins = plugins[0];
  }
  return new _processor2.default(plugins); // 就是调用了下面的Processor(plugins)
}
// processor.js
var Processor = function () {
  function Processor() {
    var plugins = arguments.length > 0 && arguments[0] !== undefined ? arguments[0] : [];
    ...
    this.plugins = this.normalize(plugins);
  }
  ...
  Processor.prototype.process = function process(css) {
    var opts = arguments.length > 1 && arguments[1] !== undefined ? arguments[1] : {};
    return new _lazyResult2.default(this, css, opts); // 调用LazyResult(processor, css, opts)
  };
}
// lazy-result.js
var LazyResult = function () {
  function LazyResult(processor, css, opts) {
    ...
    var parser = _parse2.default; // parser = require('./parse');
    // 使用opts中的指定的语法解释器覆盖原有的parser
    if (opts.syntax) parser = opts.syntax.parse;
    if (opts.parser) parser = opts.parser;
    if (parser.parse) parser = parser.parse;

    try {
      root = parser(css, opts); // 调用parse(css, opts)
    } catch (error) {
      this.error = error;
    }
    this.result = new _result2.default(processor, root, opts); // 调用Result(processor, root, opts)
  }
}
```
可以看到在经过parse处理之后得到了一个root对象，其实就是我们想要的AST树了，来看一下解析的过程
```js
// parse.js
function parse(css, opts) {
  if (opts && opts.safe) {
    throw new Error('Option safe was removed. ' + 'Use parser: require("postcss-safe-parser")');
  }

  var input = new _input2.default(css, opts);
  var parser = new _parser2.default(input);
  try {
    parser.parse();
  } catch (e) {
    // 如果opts.parser是css，但读取的是scss/sass/less文件，需要先转为css，才能交给postcss解析
    if (e.name === 'CssSyntaxError' && opts && opts.from) {
      if (/\.scss$/i.test(opts.from)) {
        e.message += '\nYou tried to parse SCSS with ' + 'the standard CSS parser; ' + 'try again with the postcss-scss parser';
      } else if (/\.sass/i.test(opts.from)) {
        e.message += '\nYou tried to parse Sass with ' + 'the standard CSS parser; ' + 'try again with the postcss-sass parser';
      } else if (/\.less$/i.test(opts.from)) {
        e.message += '\nYou tried to parse Less with ' + 'the standard CSS parser; ' + 'try again with the postcss-less parser';
      }
    }
    throw e;
  }

  return parser.root;
}
```
input和parser的过程不展开写了，作用分别为：
1. Input(css, opts)根据opts的from和map属性，返回相应的css文件内容，input.css = css.toString()
2. Parser(input)根据input.css.valueOf()的值，对于每一个字符位置pos上的值，利用String.charCodeAt(pos)判断不同情况并解析

具体的语法解析过程内容太多，以后抽空写吧,返回的结果大致是这个结构
```js
let css = 'a{color:black; b{z-index:2}} c{position:relative}'
parser.root = {
    type: 'root',
    source: { start: { line: line, column: column }, input: this.input },
    raws: ['selector':{ value: value, raw: raw }, 'value':{ value: value, raw: raw }, 'params':{ value: value,raw: raw }],
    nodes: [{
        type: 'rule',
        source,
        raws,
        nodes:[{
            type:'rule',
            source,
            raws,
            nodes:[]
        }, {
        type: 'rule',
        source,
        raws,
        nodes:[]
    }]
}
```
接下来返回LazyResult中，对AST树使用插件处理，得到this.result = Result(processor, root, opts)
```js
// result.js
var Result = function () {
  function Result(processor, root, opts) {
    this.processor = processor;

    this.messages = [];
    
    this.opts = opts;
    
    this.css = undefined;
    
    this.map = undefined;
  }
}
```
可以看出此时的this.result.css并不是我们想要的结果，这是因为在lazyResult中对于同步和异步的插件有不同的处理方式
1. 通过process(css).css调用时默认为同步插件，通过iterator迭代处理，
2. 通过process(css).then(cb)调用时默认为异步插件，通过promise函数递归处理

同步处理时直接返回this.stringify().css
```js
// 通过Object.defineProperty重新定义LazyResult.css的get方法
_createClass(LazyResult, [
  ...
  {
    key: 'css',
    get: function get() {
      return this.stringify().css;
    }
  }]);
LazyResult.prototype.stringify = function stringify() {
  // 只会进行一次stringify过程
  if (this.stringified) return this.result;
  this.stringified = true;

  this.sync();

  var opts = this.result.opts;
  var str = _stringify3.default; // require('./stringfy')
  if (opts.syntax) str = opts.syntax.stringify;
  if (opts.stringifier) str = opts.stringifier;
  if (str.stringify) str = str.stringify;

  var map = new _mapGenerator2.default(str, this.result.root, this.result.opts);
  var data = map.generate();
  // 返回结果
  this.result.css = data[0];
  this.result.map = data[1];

  return this.result;
};
LazyResult.prototype.sync = function sync() {
  // sync和async函数均通过this.processed标记，不会同时处理plugins
  if (this.processed) return this.result;
  this.processed = true;

  if (this.processing) {
    throw new Error('Use process(css).then(cb) to work with async plugins');
  }

  if (this.error) throw this.error;

  for (var _iterator = this.result.processor.plugins, _isArray = Array.isArray(_iterator), _i = 0, _iterator = _isArray ? _iterator : _iterator[Symbol.iterator]();;) {
    var _ref;

    if (_isArray) {
      if (_i >= _iterator.length) break;
      _ref = _iterator[_i++];
    } else {
      _i = _iterator.next();
      if (_i.done) break;
      _ref = _i.value;
    }

    var plugin = _ref;
    // 如果插件中包含了任何异步插件，在sync()中就会抛出错误，所以最好还是进行异步处理
    var promise = this.run(plugin);
    if (isPromise(promise)) {
      throw new Error('Use process(css).then(cb) to work with async plugins');
    }
  }

  return this.result;
};

LazyResult.prototype.run = function run(plugin) {
  // 在result中记录最后一个处理的插件
  this.result.lastPlugin = plugin;

  try {
    return plugin(this.result.root, this.result); // 插件处理时直接修改this.result.root
  } catch (error) {
    this.handleError(error, plugin);
    throw error;
  }
};
```
为了调试更方便，更推荐异步处理的方式
```js
LazyResult.prototype.then = function then(onFulfilled, onRejected) {
  if (!('from' in this.opts)) {
    (0, _warnOnce2.default)('Without `from` option PostCSS could generate wrong ' + 'source map and will not find Browserslist config. ' + 'Set it to CSS file path or to `undefined` to prevent ' + 'this warning.');
  }
  return this.async().then(onFulfilled, onRejected);
};
LazyResult.prototype.async = function async() {
  var _this2 = this;

  if (this.processed) {
    return new Promise(function (resolve, reject) {
      if (_this2.error) {
        reject(_this2.error);
      } else {
        resolve(_this2.stringify());
      }
    });
  }
  if (this.processing) {
    return this.processing;
  }

  this.processing = new Promise(function (resolve, reject) {
    if (_this2.error) return reject(_this2.error);
    // 处理插件
    _this2.plugin = 0;
    _this2.asyncTick(resolve, reject);
  }).then(function () {
    _this2.processed = true;
    // 最后将AST树返回成字符串
    return _this2.stringify();
  });

  return this.processing;
};
LazyResult.prototype.asyncTick = function asyncTick(resolve, reject) {
  var _this = this;
  // 插件处理完毕时返回resolve
  if (this.plugin >= this.processor.plugins.length) {
    this.processed = true;
    return resolve();
  }

  try {
    // this.plugin记录的是未处理的下一个插件编号
    var plugin = this.processor.plugins[this.plugin];

    var promise = this.run(plugin);    

    this.plugin += 1;

    if (isPromise(promise)) {
      promise.then(function () {
        // 如果插件本身返回的是一个Promise，继续处理时的this对象为处理完该插件的_this对象
        _this.asyncTick(resolve, reject);
      }).catch(function (error) {
        _this.handleError(error, plugin);
        _this.processed = true;
        reject(error);
      });
    } else {
      // 否则，各个插件是允许并行处理的
      // 如果没有异步插件，调用.then()其实和sync()的结果是一样的
      this.asyncTick(resolve, reject);
    }
  } catch (error) {
      this.processed = true;
      reject(error);
  }
};
```
最后将this.result.root的结果用stringify字符串化，并返回结果this.result.css、this.result.map

## 总结
重新来看options中传入的各项参数有什么作用：
+ syntax: 用来替代源码中的语法解释器parser和stringifier
+ parser: 用来替代源码中的语法解释器parser，可覆盖syntax.parser
+ stringifier: 用来替代源码中的语法解释器stringifier，可覆盖syntax.parser
+ map: [source map 选项](https://github.com/postcss/postcss/blob/master/docs/source-maps.md)，分别传递给parser和stringifier
+ from: input 文件名称，loader中自动生成，否则需要手动传入
+ to: output 文件名称，loader中自动生成，否则需要手动传入

postcss的源码内容并不算多，过程其实就是parse->run->stringify，本身只要保证css字符串和AST文件之间的相互转换过程是正确处理的即可，而run的过程则是交给插件去进行处理的，所以postcss的意义就在于提供了这样一个平台，让多个css插件无需每次都经过parse和stringify的过程反复处理css文件，从而大大提升了效率~

## 参考文档

[PostCSS README.md](https://github.com/postcss/postcss)

[使用 PostCSS 进行 CSS 处理](https://www.ibm.com/developerworks/cn/web/1604-postcss-css/)