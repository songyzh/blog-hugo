---
title: "Golang踩坑总结-循环中使用闭包捕获参数"
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
  - 踩坑
  - 闭包
categories:
  - 技术
series:
  - Golang踩坑总结

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1geon92s5ajj31e40p0dji.jpg"

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

    - 闭包中捕获的循环变量, 都为循环最后的值

    -   ```go
        func main() {
            // 保存函数闭包
            var s []func()

            for i, v := range []string{"a", "b", "c", "d", "e"} {
                s = append(s, func() {
                    // 捕获i, v, 保存在闭包中
                    fmt.Printf("index: %v, value: %v\n", i, v)
                })
            }

            for _, f := range s {
                f()
            }
        }

        // ====================输出===========================

        index: 4, value: e
        index: 4, value: e
        index: 4, value: e
        index: 4, value: e
        index: 4, value: e
        ```

2.  问题原因

    -   闭包中捕获的不是"值", 而是"有地址的变量". 最终执行时, 根据变量寻址, 得到的是变量最后的值

3.  避免方式

    -   在循环中, 重新声明要捕获的变量. 由于是单个循环的局部变量, 在其作用域结束前, 会进行evaluate

    -   ```go
        func main() {
            // 保存函数闭包
            var s []func()

            for i, v := range []string{"a", "b", "c", "d", "e"} {
                // 重新声明要捕获的变量
                i, v := i, v
                s = append(s, func() {
                    // 捕获i, v, 保存在闭包中
                    fmt.Printf("index: %v, value: %v\n", i, v)
                })
            }

            for _, f := range s {
                f()
            }
        }

        // ====================输出===========================

        index: 0, value: a
        index: 1, value: b
        index: 2, value: c
        index: 3, value: d
        index: 4, value: e
        ```

