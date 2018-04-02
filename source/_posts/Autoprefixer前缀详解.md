---
layout: post
title: Autoprefixer前缀详解
date: 2018-04-02 21:07:13
tags: 
    - autoprefixer
    - css
    - book
    - 源码
comments: true
---

最近在写[autoprefixer插件的prefixes详解](https://sydot.gitbooks.io/autoprefixer-prefixes/content/),详细分析了Autoprefixer对于特殊css规则的前缀处理，并且对于浏览器前缀的特殊规则及处理有了一个大体的认识，感兴趣的话可以mark。

在这里贴一下书写规则和示例代码~

## 书写规则

本书遵循统一的书写规则，以源码（[fork的分支](https://github.com/sydot/autoprefixer)）目录下lib/hacks中的单个js文件为一页，对照查看test/cases中的xxx.css,xxx.out.css文件，并解析其对于传入的css规则的前缀处理。

<!-- more -->

## 示例

例如源码中的示例代码example.css,example.out.css就展示了Autoprefixer对于伪元素选择器`::placeholder`这条css规则的处理:

```css
/* example.css */
::placeholder {
    color: gray;
}

/* example.out.css */
::-webkit-input-placeholder {
    color: gray;
}
:-ms-input-placeholder {
    color: gray;
}
::-ms-input-placeholder {
    color: gray;
}
::placeholder {
    color: gray;
}
```

对应的，在lib/hacks目录下就可以在placeholder.js看到处理过程:

```js
// placeholder.js
const Selector = require('../selector');

class Placeholder extends Selector {

static names = ['::placeholder'];

/**
* Add old mozilla to possible prefixes
*/
possible() {
    return super.possible().concat(['-moz- old', '-ms- old']);
}

/**
* Return different selectors depend on prefix
*/
prefixed(prefix) {
    if (prefix === '-webkit-') {
        return '::-webkit-input-placeholder';
    } else if (prefix === '-ms-') {
        return '::-ms-input-placeholder';
    } else if (prefix === '-ms- old') {
        return ':-ms-input-placeholder';
    } else if (prefix === '-moz- old') {
        return ':-moz-placeholder';
    } else {
        return `::${prefix}placeholder`;
    }
    }

}

module.exports = Placeholder;
```

则处理规则为:

* Chrome/Safari/Opera & IE使用前缀`::-webkit-input-`& `::-ms-input-`
* Firefox使用前缀`::-moz-`
* 老版本的IE & Firefox只用一个冒号
