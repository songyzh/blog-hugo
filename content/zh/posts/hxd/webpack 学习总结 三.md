---
title: "webpack 学习总结 三"
slug: "webpack-notes-3"
date: "2020-06-04T21:00:06+08:00"
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
  - "webpack学习总结"

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1gfgmg3rhhpj30m80fgjrv.jpg"

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

本文对于webpack 编译出的内容做分析总结。

## 普通文件引入的打包

```javascript
// src/test.js
module.exports = 'test'
```


```javascript
// src/index.js
let test = require('./test');
console.log(test)
```

打包后的main.js(摘取部分)

```javascript
// 打包入口 ./src/index.js
(function (modules) {
  var installedModules = {};

  function __webpack_require__(moduleId) {
    // 缓存作用，如果已经引入过了，使用引入过的结果
    if (installedModules[moduleId]) {
      return installedModules[moduleId].exports;
    }

    var module = installedModules[moduleId] = {
      i: moduleId,
      l: false,
      exports: {}
    };


    modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
    module.l = true;
    return module.exports;
  }

  return __webpack_require__(__webpack_require__.s = "./src/index.js");
})

({ // 包装成一个模块对象 绝对路径对应这个模块
  "./src/index.js": (function (module, exports, __webpack_require__) {
    // 将代码中的require ->__webpack_require__
t    console.log(test)
  }),
  "./src/test.js": (function (module, exports) {
    module.exports = 'test'
  })

});

```
 在我理解看来，在打包的过程中，每个模块被封装成了一个函数，此时模块代码并没有被执行，而是被放在了打包文件自执行函数的参数modules 中，从这个自执行函数的末尾

```javascript
return  __webpack_require__(__webpack_require__.s = "./src/index.js");
```
 开始，根据入口文件引用的文件逐一运行，原文件中的require 被替代成__webpack_require__，也就是说，webpack打包的文件都没有使用node/es6的引用方法，而是自己实现的__webpack_require__方法。

 从__webpack_require__('./src/index.js')开始，拿到参数modules中被包装的index.js 的内容，遇到__webpack_require__('./src/test.js') 时，运行参数modules中./src/test.js 对应的封装方法，每运行完一个模块，就会为其生成一个module对象添加到installedModules，{ i: moduleId, l: false, exports:{}},保存其路径i，导出的内容exports，标记是否已经加载过l,installedModules 可以起到缓存的作用，如果已经加载过的模块可以直接从installedModules里拿到。

##  不同引入方法的打包
  众所周知，目前比较流行的模块加载方式是nodejs的module.exports/require 以及es6的 import/export,在webpack中这两种打包方式都支持，也就是上文中的__webpack_require__ 两种都支持，尝试通过es6 将上例中的test.js 修改成


```javascript
// src/test.js
export const test = 'test'
```
打包后的main.js(摘取部分)

```javascript
 (function(modules) { // webpackBootstrap
  // The module cache
  var installedModules = {};

  // The require function
  function __webpack_require__(moduleId) {
   if(installedModules[moduleId]) {
    return installedModules[moduleId].exports;
   }
   var module = installedModules[moduleId] = {
    i: moduleId,
    l: false,
    exports: {}
   };
   modules[moduleId].call(module.exports, module, module.exports, __webpack_require__);
   module.l = true;
   return module.exports;
  }

  __webpack_require__.m = modules;

  __webpack_require__.c = installedModules;

  __webpack_require__.d = function(exports, name, getter) {
   if(!__webpack_require__.o(exports, name)) {
    Object.defineProperty(exports, name, { enumerable: true, get: getter });
   }
  };


  __webpack_require__.r = function(exports) {
   if(typeof Symbol !== 'undefined' && Symbol.toStringTag) {
    Object.defineProperty(exports, Symbol.toStringTag, { value: 'Module' });
   }
   Object.defineProperty(exports, '__esModule', { value: true });
  };



  __webpack_require__.o = function(object, property) { return Object.prototype.hasOwnProperty.call(object, property); };

  return __webpack_require__(__webpack_require__.s = "./src/index.js");
 })

 ({

 "./src/index.js":
 (function(module, exports, __webpack_require__) {
  let test = __webpack_require__("./src/test.js");
  console.log(test)
 }),

 "./src/test.js":
 (function(module, __webpack_exports__, __webpack_require__) {
  __webpack_require__.r(__webpack_exports__);
  __webpack_require__.d(__webpack_exports__, "test", function() { return test; });
  const test = 'test'
 })

 });

```
可以看出，与上例中最大的区别是包装test.js 发生了变化，去除了export 关键字，并且添加了

```
__webpack_require__.r(__webpack_exports__);
__webpack_require__.d(__webpack_exports__, "test", function() { return test; });
```
查看__webpack_require__.r方法，发现是向模块运行结果exports中添加标记，标记当前模块__esModule 为true,如果是支持
Symbol的浏览器，也添加一个Symbol值为Module，这种标记是为了后续有需要的地方可以识别此模块的导出方式，__webpack_require__.d 方法则是为当前导出的exports 对象上添加一个值为'test'的test 属性，如此一来，test.js 也有了module.exports = {test: 'test'}的效果。

同理，其他导出/导入组合，webpack 都会对包装函数做不同的处理，以便能统一达到效果。