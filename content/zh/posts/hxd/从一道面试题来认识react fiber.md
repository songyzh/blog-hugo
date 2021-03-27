---
title: "从一道面试题来认识react fiber"
slug: "introduce-react-fiber"
date: "2021-03-27T21:55:06+08:00"
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

image: "https://tva1.sinaimg.cn/large/008eGmZEgy1goyt3gbyatj30zk0jqaam.jpg"

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

从一道面试题来认识react fiber



最近在看公司的面试题，其中一道就是基于react fiber 来出的算法题，react fiber 提出来也好多年了，于是趁研究这个题来进一步学习一下fiber的思想。

我们都知道react 是基于虚拟dom 来分析页面中的dom 元素的变化的，组件需要更新的时候，react 需要同步计算虚拟dom 的变化-->对比新旧虚拟dom-->更新真实dom,因为是同步的，所以当一个组件深层嵌套的时候，庞大的同步计算量会让浏览器无暇去处理别的页面交互，比如说相应用户的input 输入等等你原来写好的交互事件，用户会觉得页面明显卡顿。

为了解决这一问题，react 团队就引入了fiber 的概念，简单来说，就是通过类似于[requestIdleCallback](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestIdleCallback) 来实现在浏览器空闲的时候来“计算虚拟dom 的变化-对比新旧虚拟dom”，为了能随时中断遍历虚拟dom 的计算过程，需要将dom结构变化任务（增删改查）梳理成一个链表，链表的好处就是可以随时中断，下次有空闲的时候继续上一次任务的next就可以了，如何根据虚拟DOM树生成这样的任务链表呢？

 于是大神们发明了fiber 结构，这个结构是一个特殊的链表结构，通常我们熟悉的链表每个节点也就一个 prev,一个 next 指向，可以根据需要形成一条线性链表或者双向链表，但是！根据dom 结构生成的virtual dom 是一个树形结构，如何方便快捷的遍历这颗树来生成我们需要最终交付执行的那个任务链表呢？于是fiber 结构被创造出来，一句话来概括就是，fiber 每个节点都有child,sibling,return(也就是我们下面题里的parent)，神奇的这个小改动，可以让遍历树结构生成任务链表（也就是一些文章里的effectList）更加方便快捷！这里先贴一张网上的fiber 结构和fiber 树的形象图：

![tree](https://tva1.sinaimg.cn/large/008eGmZEly1gorsyx6g9dj30qy0kcgnt.jpg)



好了，前置背景就介绍到这里，现在我们来看我们的面试题:

**第一问**

与传统的树型结构不同，fiber的每个节点由3个字段组成：

- child - 第一个子节点
- sibling - 下一个兄弟节点
- parent - 父节点

给一棵fiber树，写一个先根序遍历，遍历的内容就是console.log出对应节点的name

给出的测试数据为：

```javascript
var a1 = { name: 'a1', child: null, sibling: null, parent: null };
var b1 = { name: 'b1', child: null, sibling: null, parent: null };
var b2 = { name: 'b2', child: null, sibling: null, parent: null };
var b3 = { name: 'b3', child: null, sibling: null, parent: null };
var c1 = { name: 'c1', child: null, sibling: null, parent: null };
var c2 = { name: 'c2', child: null, sibling: null, parent: null };
var d1 = { name: 'd1', child: null, sibling: null, parent: null };
var d2 = { name: 'd2', child: null, sibling: null, parent: null };

a1.child = b1;
b2.child = c1;
b3.child = c2;
c1.child = d1;

b1.sibling = b2;
b2.sibling = b3;
d1.sibling = d2;

b1.parent = a1;
b2.parent = a1;
b3.parent = a1;
c1.parent = b2;
c2.parent = b3;
d1.parent = c1;
d2.parent = c1;

// 要求
traverse(a1); // 将依次打印出 a1 b1 b2 c1 d1 d2 b3 c2
```

这是根据测试数据画出的解释图：

​ ![tree2](https://tva1.sinaimg.cn/large/008eGmZEly1gort3nywn8j30oz0kwglo.jpg)

看题目是要求先根序遍历，也就是当我们遍历到某个节点的时候优先对这个节点本身作出操作（也就是打印na me）,于是用一个递归来实现：

```javascript
function traverse(node) {
  print(node); // 打印自己的name
  if (node.child != null){
    traverse(node.child) // 根据fiber结构的特点，如果有child 下一个任务就是child
  }
  if (node.sibling != null){
    traverse(node.sibling) // child 结束后下一个任务就是sibling
  }
}

function  print(node) {
  console.log(node.name);
}
```





**第二问**

第一问中实现的traverse过程是一个完整的过程，当树很大的时候，traverse一次耗时比较久，容易导致页面卡顿（例如影响到了动画执行），如果能让traverse过程可以随时中断和恢复，那么就可以分多次执行完成traverse过程，避免页面卡顿。

现在需要扩展刚才的traverse函数：

1. 将traverse从一次性遍历变更为多次遍历，即一个完整的traverse由若干次调用traverseOneNode组成，每次traverseOneNode结束后，判断一下是否可以继续（shouldStop），如果可以则继续下一次traverseOneNode，否则就停下来等待（waitForNextIdle）
2. traverseOneNode只遍历一个节点，但是会返回下一个要遍历的节点

于是我写下了我的答案：

```javascript
var a1 = { name: 'a1', child: null, sibling: null, parent: null };
var b1 = { name: 'b1', child: null, sibling: null, parent: null };
var b2 = { name: 'b2', child: null, sibling: null, parent: null };
var b3 = { name: 'b3', child: null, sibling: null, parent: null };
var c1 = { name: 'c1', child: null, sibling: null, parent: null };
var c2 = { name: 'c2', child: null, sibling: null, parent: null };
var d1 = { name: 'd1', child: null, sibling: null, parent: null };
var d2 = { name: 'd2', child: null, sibling: null, parent: null };

a1.child = b1;
b2.child = c1;
b3.child = c2;
c1.child = d1;

b1.sibling = b2;
b2.sibling = b3;
d1.sibling = d2;

b1.parent = a1;
b2.parent = a1;
b3.parent = a1;
c1.parent = b2;
c2.parent = b3;
d1.parent = c1;
d2.parent = c1;

function shouldStop(deadline) {
  console.log(deadline.timeRemaining());
  if ((deadline.timeRemaining() > 1 || deadline.didTimeout) && node) {
    return false;
  }
  return true;
}

function traverse(deadline) {
  while (node) {
    debugger;//  方便调试每一步是否正确
    var nextNode = traverseOneNode(node);
    node = nextNode;
    if (shouldStop(deadline)) {
      requestIdleCallback(traverse);
      break;
    }
  }
}


/**
 * @param {Node} node
 * @return {Node|null} 下一个将要遍历的节点
 */
function traverseOneNode(node) {
  // todo
  printNode(node);
  if (node.child) {
    return node.child;
  } else if (node.sibling) {
    return node.sibling;
  } else {
    return node.parent;
  }
}

function printNode(node) {
  console.log(node.name);
}

var node = a1;
requestIdleCallback(traverse);
```



上述代码导致了一个死循环。。。因为在c1-d1-d2 这块返回父节点后，依然走的是 判断是否有子，有子返回子，无子返回sibling，所以就导致了c1-d1-d2 -c1-d1-d2 -c1-d1-d2  这样的循环。。。所以在print方法中我添加了一个标记当前node已经遍历过了,然后发现打印一个就标记一个还是无法判断当前应该继续向下走还是向上走。。。于是又各种搜索相关文章，发现思路是如果当前节点的子元素都遍历完了再标记当前节点已完成，就可以判断是要继续向下走还是向上返回父节点，继续遍历父节点的兄弟节点。。。终于经过一番对浏览器的死循环考验之后得出了下面的答案（就只是打印，主要修改了traverseOneNode方法,trasverse 方法也发现网上有更简洁的写法，一起改了）：

```javascript
var a1 = { name: 'a1', child: null, sibling: null, parent: null };
var b1 = { name: 'b1', child: null, sibling: null, parent: null };
var b2 = { name: 'b2', child: null, sibling: null, parent: null };
var b3 = { name: 'b3', child: null, sibling: null, parent: null };
var c1 = { name: 'c1', child: null, sibling: null, parent: null };
var c2 = { name: 'c2', child: null, sibling: null, parent: null };
var d1 = { name: 'd1', child: null, sibling: null, parent: null };
var d2 = { name: 'd2', child: null, sibling: null, parent: null };

a1.child = b1;
b2.child = c1;
b3.child = c2;
c1.child = d1;

b1.sibling = b2;
b2.sibling = b3;
d1.sibling = d2;

b1.parent = a1;
b2.parent = a1;
b3.parent = a1;
c1.parent = b2;
c2.parent = b3;
d1.parent = c1;
d2.parent = c1;

function shouldStop(deadline) {
  if ((deadline.timeRemaining() > 1 || deadline.didTimeout) && node) {
    return false;
  }
  return true;
}

function traverse(deadline) {
  while (node && !shouldStop(deadline)) {
    var nextNode = traverseOneNode(node);
    node = nextNode;
  }
  if (node) {
    requestIdleCallback(traverse);
  }

}


/**
 * @param {Node} node
 * @return {Node|null} 下一个将要遍历的节点
 */
function traverseOneNode(node) {
  if (node.completed) {
    // 遍历完的节点 返回其下一个兄弟节点 或者 父节点
    if (node.sibling) {
      return node.sibling;
    } else if (node.parent) {
      node.parent.completed = 1;
      return node.parent;
    }
  } else {
    printNode(node); // 避免从子节点回来的时候再次打印
  }

  if (node.child) {
    return node.child;
  } else if (node.sibling) {
    node.completed = 1; // 没有child,标记这个节点已经循环过了
    return node.sibling;
  } else {
    node.completed = 1; // 没有child 也没有sibling 这个节点肯定是父亲的唯一/最后一个节点 标记这个节点已经循环过了
    node.parent.completed = 1; // 回到父节点，此时父节点也应该标记循环过了 父节点等所有的子节点遍历后才算遍历完
    return node.parent;
  }
  return null;
}

function printNode(node) {
  console.log(node.name);
}

var node = a1;
requestIdleCallback(traverse);
```



对这个面试题的研究就到这里了，经过这道题更加理解了fiber的理念，整个fiber这一套在react中的实现还是特别复杂和庞大的，这道题只是其中很小的一部分，主要考察的是对这种特殊树结构的处理，对遇到这道题的小伙伴表示摸摸头。。。



参考：

http://www.ayqy.net/blog/dive-into-react-fiber/

https://segmentfault.com/a/1190000018250127

https://juejin.cn/post/6844903582622285831

https://zhuanlan.zhihu.com/p/37095662

https://github.com/JTangming/blog/issues/16

https://blog.logrocket.com/deep-dive-into-react-fiber-internals/

https://www.velotio.com/engineering-blog/react-fiber-algorithm