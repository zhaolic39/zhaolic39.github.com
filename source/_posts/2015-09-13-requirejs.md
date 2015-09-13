---
layout: post
title: RequireJS中定义js模块的方式
date: 2015-09-13 21:41:06
comments: true
categories: web前端
---
`web`前端项目一大挑战就是烦多的`js`框架与开发的`js`代码。传统的开发方式我们需要在`html`上用`<script>`标签加载`js`文件，并且按照依赖关系按顺序加载。  
[RequireJS](http://requirejs.org/)框架提供了一种管理`js`脚本的方式，让我们在定义模块时就声明它需要依赖的其他模块，在使用`js`模块时让`RequireJS`自动处理它们的依赖关系。  
其他类似的框架还有国产的[SeaJS](http://seajs.org/docs/)，它使用的`CMD`模块定义。

<!-- more -->

## 定义AMD模块
在`RequireJS`框架下定义的模块需要遵循[AMD](https://github.com/amdjs/amdjs-api/wiki/AMD)规范。
定义模块文件`js/lib/module.js`：
```javascript
define(['dep1', 'dep2'], function (dep1, dep2) {
    //Define the module value by returning a value.
    var module = {};
    moduel.doSomething = function(){
    };
    return module;
});

```
* `define`第一个参数为数组，表示该模块依赖的其他模块列表。
* 第二个参数是`function`，形式参数是依赖模块的注入，`function`内便可以使用这些模块，。
* `function`的返回是定义模块的对象。

## 使用AMD模块
首先定义我们需要用到的模块：
```javascript
requirejs.config({
    //默认情况下模块所在目录为js/lib
    baseUrl: 'js/lib',
    //当模块id前缀为app时，他便由js/app加载模块文件
    //这里设置的路径是相对与baseUrl的，不要包含.js
    paths: {
        module: 'module'
    }
});
```
开始业务逻辑：
```javascript
requirejs(['module'],
  function(module) {
    module.doSomething();
});
```
业务代码中我们只是用到了`module`模块，但`RequireJS`框架帮我们自动加载了所需要的其他依赖模块。
## 加载非AMD模块
很多时候项目中需要使用一些老代码，它们没有使用AMD定义。这个时候我们就要使用`shim`加载方式：
```javascript
require.config({
    shim: {
        'jquery': ['jquery']
    }
});
```
这样我们就可以使用`requirejs(['jquery'], function(){})`的方式加载`jquery`了。

参考文章：
* [RequireJS学习笔记](http://www.cnblogs.com/yexiaochai/p/3214926.html)
