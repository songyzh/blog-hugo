---
title: "webpack 学习总结 一"
slug: "webpack-notes-1"
date: "2020-06-01T21:00:06+08:00"
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

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1gfd4lqjr4hj30zk0ic0u1.jpg"

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

webpack 是当前前端项目打包当中非常流行的打包框架，结合丰富的流程钩子函数、插件系统以及nodejs部分功能 给开发者提供了方便配置，易于扩展的打包工具，在学习webpack源码流程(4.43.0)的这段时间收获很多，在此做一些总结。

 本文首先总结下，在webpack中用到的诸多工具项目，他们在实现整体流程中起到了很大的作用。

####  一 Tapable

[Tapable](https://github.com/webpack/tapable)是webpack流程的基础，整个编译过程，各个模块的方法调用都是基于Tapable的事件注册和触发。相对于nodejs 的EventEmitter，Tapable提供了事件触发的更多形式，同步触发，异步并行触发，异步串行触发等等，可以满足不同需求。

在webapck中，最主要的Compiler和Compilation类都继承了
Tapable，从而可以生成自己的一系列钩子函数，这些钩子函数在流程控制中起到承上启下的作用，也为开发者自定义功能提供了灵活的接口。

官网中的例子：

```
const {
 SyncHook,
 AsyncParallelHook,
 } = require("tapable");

class Car {
 constructor() {
  this.hooks = {
   accelerate: new SyncHook(["newSpeed"]),
   brake: new SyncHook(),
   calculateRoutes: new AsyncParallelHook(["source", "target", "routesList"])
  };
 }

 /* ... */
}

const myCar = new Car();

// 相当于给brake钩子添加一个事件（此时并不会执行，可以添加多个）
myCar.hooks.brake.tap("WarningLampPlugin", () => warningLamp.on());

// 触发钩子函数,可以在需要的时机来触发，webpack中正是利用这一点来统一给各流程添加逻辑
 myCar.hooks.brake.call()
```

#### 二 neo-async
[neo-async](https://www.npmjs.com/package/neo-async) 其中的forEach 方法可以方便的进行异步循环，循环结束后执行回调,在webpack中处理模块依赖时需要用到该方法来确保在模块的所有依赖处理完成之后进行下一步。

```
// array
var order = [];
var array = [1, 3, 2];
var iterator = function(num, done) {
  setTimeout(function() {
    order.push(num);
    done();
  }, num * 10);
};
async.each(array, iterator, function(err, res) {
  console.log(res); // undefined
  console.log(order); // [1, 2, 3]
});
```
#### 三 Semaphore
[Semaphore](https://www.npmjs.com/package/semaphore)
是一个控制并发数量的工具，可以设置最大并发量，在任务结束时释放容量，每次take 一个任务时都会查看是否到达容量极限，如果到达会存储任务到队列中，等待容量被释放时再从队列中取出任务执行

```
// 2 clients at a time
var sem = require('semaphore')(2);
var server = require('http').createServer(req, res) {
    res.write("Then good day, madam!");

    sem.take(function() {
        res.end("We hope to see you soon for tea.");
        sem.leave();
    });
});
```
webpack中工厂函数生成多个模块 以及编译模块时都采用了
Semaphore 来控制并发
####  四 利用Object.defineProperty 重写get函数来添加一些内容获取的逻辑


```
let test = {arr:  [1,2,2,3,4,5] }
let res = {}
Object.defineProperty(res, 'arr', {
  get() { // 这里可以自由添加自己需要的逻辑
    return test.arr.filter(item=>item>1);
  }
});
console.log(res.arr)// [2, 2, 3, 4, 5]
```

webpack 中LoaderRunner.js 中就采用了这一技巧对loader进行信息采集