---
title: "react-hooks 初探"
slug: "react-hooks-study"
date: "2020-09-20T18:31:06+08:00"
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
  - react
series:
  -

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1gixglh6hg1j30zk0jqjsh.jpg"

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

#### 前言

​  我们都知道，在用react开发前端应用时，有类组件和函数组件两种形式来编写组件，其中函数组件有自己的状态state，每次调用setState 都会有一个更新组件需求添加到队列中，从而实现页面跟随数据变化而更新。随着前端生态的发展，React 16.8 引入hook 的概念，可以让我们在编写相对于类组件更加轻便灵活的函数式组件的同时，也能拥有组件内部的状态（useState）、生命周期函数（useEffect）等类组件的特性。本文就部分钩子的作用和实现做一些总结和探讨。

####  一 最常使用的useState

首先将官网的demo拿过来：

```javascript
import React, { useState } from 'react';

function Example() {
  // 声明一个叫 "count" 的 state 变量
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p >
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}
```

useState 实现了：

- 传入一个state 初始变量
- 返回state引用和改变state的回调
- 如果state改变 更新组件

在我们理解函数组件的时候，本质上它是一个函数，每当组件更新，就会重新执行一遍函数，如果函数内正常声明的变量，都是会重新声明一遍，跟上一次函数的执行时声明的变量毫无关系，那么如果我们想要每次执行函数时能有一个变量始终基于上一次函数执行后的结果上再更改，就只有一个办法： 在函数外声明一个变量，又因为我们需要监听状态变化来重新渲染，所以需要将更改状态的操作抽离到一个函数中，由此可以在更改状态后添加更新一以及其他一些逻辑，实现一个自己的setState:

```javascript
import React from 'react';
import ReactDOM from 'react-dom';

// hook 相关的变量存储
let hook = {
  memoizedState: null,
}

function useState(initialState) {
  // 这里也可以添加对initialState 是否是函数的判断来支持 react惰性初始 state 
  // 也就是initialState 如果是函数，执行函数返回结果作为initialState
  hook.memoizedState = hook.memoizedState || initialState;
  function setState(state) {
    hook.memoizedState = state;
    // 由于源码中这里关联fiber，简化它门的dispatch重新渲染的逻辑
    render(); // state 变化重新渲染
  }
  return [hook.memoizedState, setState]; // 返回数组便于用户解构和重命名
}

function Example() {
  // 声明一个叫 "count" 的 state 变量
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>You clicked {count} times</p >
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}


function render(){
  ReactDOM.render(
    <Example/>,
    document.getElementById('root')
  );
}
render();

```



#### 二 第二常用的useEffect

useEffect 实现了：

- 默认情况下，effect 将在每轮渲染结束后执行（没有依赖）
- 组件更新后如果依赖的变量数组有变化，执行回调 （有依赖项）

同理，要想比较两次函数中声明的变量是否相同，需要在函数外声明缓存变量来比较，由于是依赖数组，需要有一个循环依次比较数组的每一项：

```javascript
import React, { useState } from 'react';
import ReactDOM from 'react-dom';

// hook 相关的变量存储
// 这里为方便演示，只有一个lastDeps，真实场景中应该为每一次useEffect 分配一个lastDeps/destroy
let hook = {
  lastDeps: null,
  destroy: null // useEffect 返回的销毁函数
}

function useEffect(create,deps) {
  if (!hook.lastDeps) { // 第一次渲染
    hook.destroy = create(); // 缓存useEffect的清除函数
    hook.lastDeps = deps;
  } else {
    let isSame = hook.lastDeps.every((item,index) => {
      return item === deps[index];
    });
    if (!isSame) {
      hook.destroy(); // 在执行新的回调前先执行清除函数
      hook.destroy = create();
      hook.lastDeps = deps;
    }
  }
}

function Example() {
  // 声明一个叫 "count" 的 state 变量
  const [count, setCount] = useState(0);
  const [stable, setStable] = useState(2);
  // 依赖count 或者不添加依赖数组会导致每次组件更新都执行
  useEffect(() => {
    document.title = count;
    return ()=>{
      console.log('clear');
    }
  },[count]) 

  // 依赖一个不变化的变量只执行一次回调
  // useEffect(() => {
  //   console.log('test');
  // },[stable])

  return (
    <div>
      <p>You clicked {count} times</p >
      <button onClick={() => setCount(count + 1)}>
        Click me
      </button>
    </div>
  );
}


function render(){
  ReactDOM.render(
    <Example/>,
    document.getElementById('root')
  );
}
render();

```

以上的例子只是为了理解useEffect 原理而写的例子，在真实场景中，官网中指出：**与 `componentDidMount`、`componentDidUpdate` 不同的是，在浏览器完成布局与绘制之后，传给 `useEffect` 的函数会延迟调用**，官网还指出： **为防止内存泄漏，清除函数会在组件卸载前执行**，所以在上面的例子中，create和destroy 函数的执行时机是不准确的，在react机制中应该是有更加复杂的调度。

#### 三 通常常用的useCallback

官网中的解释是**把内联回调函数及依赖项数组作为参数传入 `useCallback`，它将返回该回调函数的 memoized 版本，该回调函数仅在某个依赖项改变时才会更新,当你把回调函数传递给经过优化的并使用引用相等性去避免非必要渲染（例如 `shouldComponentUpdate`）的子组件时,它将非常有用**，当我们在函数组件中声明一个函数，那么每次执行渲染都会声明这个新的函数，如果此函数是传递给子组件的一个属性，那么每次函数更新都会传递给子组件全新的函数，在使用memo包装子组件的时候结合useCallback 可以有效减少子组件不必要的重新渲染：

```javascript
import React, { useState ,memo} from 'react';
import ReactDOM from 'react-dom';

let hook = {
  lastCallback: null,
  lastDeps: null,
}

function useCallback(cb,deps) {
  if (!hook.lastCallback) { // 第一次渲染
    hook.lastCallback = cb;
    hook.lastDeps = deps;
    return cb;
  } else {
    let isSame = hook.lastDeps.every((item,index) => {
      return item === deps[index];
    })
    if (!isSame) {
      hook.lastCallback = cb;
      hook.lastDeps = deps;
      return cb;
    } else {
      return hook.lastCallback;
    }
  }
}

function Child(props) {
  console.log('child render');
  return (
    <div>
      <br />
      <span onClick={props.handleClick}> I am child</span>
    </div>
  )
}

Child=memo(Child); // memo 包装的组件在属性不变时不再重新渲染

function Example() {
  // 声明一个叫 "count" 的 state 变量
  const [count, setCount] = useState(0);
  const [count2, setCount2] = useState(0);

  // 这样写每次传递给Child 组件的handleClick 都是全新的函数 
  // 点击add count2 -> 组件刷新 ->handleClick是新的函数 ->Child 组件props变化重新渲染
  // let handleClick = () => {
  //   setCount(count + 1);
  // }

  // 点击add count2 -> 组件刷新 ->handleClick是旧的函数 ->Child 组件不会重新渲染
  let handleClick = useCallback(() => {
     setCount(count + 1);
  },[count])

  return (
    <div>
      <p>count {count}</p >
      <button onClick={() => setCount(count + 1)}>
        add count
      </button>
      <br />
       <p>count2 {count2}</p >
      <button onClick={() => setCount2(count2 + 1)}>
        add count2
      </button>
      <Child handleClick={handleClick}/>
    </div>
  );
}

function render(){
  ReactDOM.render(
    <Example/>,
    document.getElementById('root')
  );
}
render();

```

总结来说，优化都是靠函数外声明的变量来缓存值，并且在每次调用函数时做优化判断，在应用钩子函数的时候，要想达到理想的效果，需要尽可能理清在渲染过程的这些动作，避免引起更多bug 的情况。



参考  https://zh-hans.reactjs.org/docs/hooks-state.html

​         https://www.jianshu.com/p/61d6193e04da

​        https://www.lagou.com/lgeduarticle/103675.html