---
title: "react不可变数据之immer的使用"
slug: "react-immer"
date: "2020-12-28T22:14:06+08:00"
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
categories:
  - 技术
series:
  -

image: "https://tva1.sinaimg.cn/large/0081Kckwly1gm3xg9bhbbj30xc0gota7.jpg"

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

#### 一 问题： 我们都知道在js 中一个重要的数据类型就是引用数据类型，特征就是相同的引用类型值可以在各个地方被改变：

```js
var objA = {name: 'Alice'};
var objB = objA;
objB.name = "Bob";
console.log(objA.name); // "Bob"
console.log(objB.name); // "Bob"
```



​  上面的简易代码显示了引用类型的这一特征。这个特征在多个页面结构引用同一个数据的时候很方便，但这种公共状态数据可以随意修改也带来的代码的不稳定性和难以维护。没有人知道会有谁在不经意的情况下修改了你们公共的状态数据。今天我们不谈redux这一个通用的前端框架数据流解决方案，只谈谈细粒度更小的不可变数据的实现。

​         我们都知道react 的实现中，数据变化导致页面重新更新Dom 主要是靠setState方法改变组件内的state,调用了setState 方法后就会更新视图，那么当我们setState 改变的属性页面中没有用到的时候，或者传入的属性就是当前的值，setState 会有优化吗？这里做两个demo 来检测一下：

```javascript
// 这里使用crate-react-app 的框架，改变其中的app.js
import React from 'react';
class App extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      list: [{
        todo: 'learn typescript',
      }]
    }
  }

  componentDidMount() {
    setInterval(() => {
        this.setState({
          list: this.state.list, // 值没有改变
        })
      }, 2000);
  }

  render() {
    console.log('render called'); // 这里会一直打印，说明相同的值不会被检测到
    let content = this.state.list.map((item) => {
      return <p>{item.todo}</p >
    });
    return content;
  }
}

export default App;
```



同样的，没有使用的state 属性也会一直导致重新render:

```javascript
import React from 'react';
class App extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      list: [{
        todo: 'learn typescript',
      }]
    }
  }

  componentDidMount() {
    setInterval(() => {
        this.setState({
          notUsed: 'this is a not used property', // 没有在页面中使用到的值
        })
      }, 2000);
  }

  render() {
    console.log('render called'); // 这里会一直打印
    let content = this.state.list.map((item) => {
      return <p>{item.todo}</p >
    });
    return content;
  }
}

export default App;
```

当然，在项目中我们一般不会这样做，使用没有用到的state属性或者传入相同的值，但是两个例子让我们知道，不能完全依赖setState 来构建我们的交互，毕竟渲染Dom 节点是很消耗性能的，复杂的页面和逻辑中需要考虑到这个过程的优化，避免不必要的渲染。

#### 二 之前的解决方案：官网中指出了集中优化方案，使用shouldComponentUpdate 来自定义是否需要重新渲染的逻辑： 

```javascript
shouldComponentUpdate(nextProps, nextState) {
  // todo 这里可以加上对新老 props state 的比较来决定是否需要更新组件
  return true;
}

// 加上这里的判断就不会一直重复渲染，因为两个list 是同一个引用
  shouldComponentUpdate(nextProps, nextState) {
    if (nextState.list === this.state.list) {
      return false;
    }
    return true;
  }
```



​  React.PureComponent 就是基于这一程序帮你写好了新旧props state 的对比，但是他对比的是新老

props state的浅比较，对于复杂数据结构就判断不出来变化了。

​      或者我们也可以使用es6 的结构赋值，或者Object.assign，或者利用数组的concat，JSON.Stringify 等一切能够基于原来数据产生互不干扰的新数据的方法，比如在我们的项目中在对state中复杂数据的遍历处理前，为了不产生意料之外的副作用，我们会使用lodash 的 cloneDeep，基于这个新数据进行处理判断： 

```javascript
  let workerList = _cloneDeep(this.state.list);
  workerList.forEach((element) => {
    // todo ...
  });
  this.setState({ list: workerList });
```



归根结底，对于state 中的这个list ,直接遍历修改是非常不美观，不符合react 设计流程的，也会引起代码的不好阅读和困惑，明明人家提供的、建议的是setState 为何要直接修改这个引用类型的状态呢？

#### 三 开始使用immer

对于不可变数据的需求，终于引出了今天想来说一说的immer，相对于API更加丰富的Immutablejs,immer 似乎上手成本更低，接下来引入immer:

```javascript
npm install immer
```

index.js:

```javascript
import produce from "immer"

const baseState = [
    {
        todo: "Learn typescript",
        done: true
    },
    {
        todo: "Try immer",
        done: false
    }
]

// immer 中主要使用produce方法来基于原来的数据生成新的immutable数据
const nextState = produce(baseState, draftState => {
    // 这里的draftState 可以理解成baseState 的一个浅引用
    // 注意这个函数里操作的只是draftState
    draftState.push({todo: "Tweet about it"});
    draftState[1].done = true;
});

console.log(JSON.stringify(baseState));
// 可以看到原数据不受影响
// [{"todo":"Learn typescript","done":true},{"todo":"Try immer","done":false}]

console.log(JSON.stringify(nextState));
// 新增的数据在原数据基础上作出produce第二个函数中的变更
// [{"todo":"Learn typescript","done":true},{"todo":"Try immer","done":true},{"todo":"Tweet about it"}]

console.log(baseState === nextState);
// false 新旧对象并不相等

console.log(baseState[0] === nextState[0]); // true
// 神奇的是没有变更的数据还是原来的数据！这肯定比上面粗暴的deepClone 在处理大数据的时候性能高，因为他是只修改了变化的部分
console.log(baseState[1] === nextState[1]); // false

```

因此上面的deepClone 可以替换成:

```javascript
 let workerList = produce(this.state.list,(draft)=>{
    // todo...对draft的操作 再也不用担心大数据deepClone的性能消耗
 })
 this.setState({ list: workerList });
```

当我们给produce 传递的第一个参数是函数，就是用于产生一个科里化函数，[在官网可以看到更多例子](https://immerjs.github.io/immer/docs/curried-produce)，immer 基于原对象产生新对象的功能也使得他可以更便利的编写[reducer](https://immerjs.github.io/immer/docs/example-reducer),感兴趣的可以去看看。

#### 四 什么魔法造成了神奇的draft操作函数

​    秘诀就是proxy 代理，vue3.0 为了解决Object.defineProperty 的性能和不能为新增属性写代理的问题就采用了Proxy，Proxy 实际上重载（overload）了点运算符，即用自己的定义覆盖了语言的原始定义：

```javascript
var obj = new Proxy({}, {
  get: function (target, key, receiver) {
    console.log(`getting ${key}!`);
    return Reflect.get(target, key, receiver);
  },
  set: function (target, key, value, receiver) {
    console.log(`setting ${key}!`);
    return Reflect.set(target, key, value, receiver);
  }
});

obj.count = 1
//  setting count!
++obj.count
//  getting count!
//  setting count!
//  2
```



在produce方法中的draft 就是这样的一个代理，当我们设置值的时候，会走immer中定义好的set,获取值的时候也是同理。 produce(base, recipe, patchListener) ，produce 接收三个参数，正常来说 base 是原数据，recipe 是用户执行修改逻辑的地方，patchListener 是用户接收 patch 数据然后做一些自定义操作的地方，patchListener 我暂时没有用到过。

- **调用`createProxy`生成 draft 供用户使用，createProxy就是生成上面的代理**
- **执行用户传入的 recipe，拦截读写操作，走到 proxy 内部的 getter/setter**
- **调用`processResult`解析组装最后的结果返回给用户**

在每一个对象中还定义了modified 属性来标记是否被更改过，如果取值的时候发现modified为false，则直接返回原来的值（proxy中的）。

具体的代码在这个架构上更加细致和繁琐，在这里不多赘述。

全文完。



参考文档：

https://zh-hans.reactjs.org/docs/optimizing-performance.html

https://immerjs.github.io/immer/docs/introduction

https://juejin.cn/post/6844903782145327118

https://zhuanlan.zhihu.com/p/34691516