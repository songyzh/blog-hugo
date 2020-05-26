---
title: "Golang踩坑总结-循环中使用Goroutine中捕获参数"
slug: "golang-pitfall-for-goroutine"
date: "2020-05-24T12:00:06+08:00"
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
  github: "https://github.com/songyzh"
  weibo: "https://weibo.com/u/1686093881"

tags:
  - 后端
  - Golang
  - 踩坑
  - Goroutine
categories:
  - 技术
series:
  - Golang踩坑总结

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1ger23ueqcgj30h00bcmxs.jpg"

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

1.  问题表现

    - goroutine中捕获的循环变量, 都为循环最后的值

    -   ```go
        func main() {

            for i, v := range []string{"a", "b", "c", "d", "e"} {
                // goroutine中捕获循环变量
                go func() {
                    fmt.Printf("index: %v, value: %v\n", i, v)
                }()
            }

            // 此处应该使用workgroup实现, 为了简单使用了sleep
            time.Sleep(1 * time.Second)

        }

        //================输出==============

        index: 4, value: e
        index: 4, value: e
        index: 4, value: e
        index: 4, value: e
        index: 4, value: e
        ```

2.  问题原因

    -   goroutine中捕获的不是"值", 而是"有地址的变量". for循环可能会先结束, 之后各个goroutine才开始执行. 因此得到的是变量的最终值

3.  避免方式

    -   在goroutine启动的函数中, 把变量作为参数捕获

    -   ```go
        func main() {

            for i, v := range []string{"a", "b", "c", "d", "e"} {
                // 把循环变量作为参数传入
                go func(i int, v string) {
                    // i, v是函数内部的局部变量
                    fmt.Printf("index: %v, value: %v\n", i, v)
                }(i, v)
            }

            time.Sleep(1 * time.Second)

        }

        //================输出==============

        index: 0, value: a
        index: 1, value: b
        index: 4, value: e
        index: 3, value: d
        index: 2, value: c
        ```


