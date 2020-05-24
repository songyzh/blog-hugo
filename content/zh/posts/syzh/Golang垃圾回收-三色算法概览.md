---
title: "Golang垃圾回收-三色算法概览"
date: "2019-12-16T12:00:06+09:00"
description: ""

draft: false

hideToc: false
enableToc: true
enableTocContent: false

author: "卓"
authorEmoji: ""
authorImage: ""
authorImageUrl: ""
authorDesc: ""
socialOptions:
  email: "mailto:mailsyzh@gmail.com"
  phone: ""
  facebook: ""
  twitter: ""
  github: "https://github.com/songyzh"

tags:
  - 后端
  - Golang
  - 垃圾回收
categories:
  - 技术
series:
  -

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1geotk98c56j314n0u0dk8.jpg"

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

三色标记-清除算法(tricolor mark-and-sweep algorithm)

-   把heap中的对象, 用***黑色, 灰色, 白色***进行标记
    -   黑色对象: 已经以其为root执行过可达性分析的对象
    -   灰色对象: 需要但还未以其为root执行过可达性分析的对象
    -   白色对象: 从gc root无法被触达的对象, 可能被垃圾回收
-   垃圾回收过程
    1.  垃圾回收开始时, 所有对象都是白色. 所有root对象被标记为灰色. root对象指程序可以直接访问的对象, 包含全局变量, 位于栈上的数据.
    2.  垃圾回收器选择一个灰色对象, 把它标记为黑色, 寻找其可以触达的白色对象, 并把它们标记为灰色. 直到把所有灰色对象都处理过一遍
    3.  此时的白色对象说明无法被触达, 因此可以被垃圾回收
    4.  如果在垃圾回收过程中, 某个灰色对象变得不可触达了, 则它会在***下次***垃圾回收时被清理
-   标记过程中, 执行中的应用程序被称为**mutator**, mutator有一个函数**write barrier**. 当heap中的某个对象被修改了, 说明该对象可被触达, 则**write barrier**会把此对象标记为灰色
-   当某channel无法被触达时, 即使其未被close, 也会被垃圾回收, 清理其占用的资源
-   手动触发GC: `runtime.GC()`, 此操作会阻塞调用者, 并可能阻塞整个程序
-   mark阶段, 标记对象; 没有灰色对象后, 开始sweep阶段
-   go的垃圾回收是与其他goroutine并发执行的