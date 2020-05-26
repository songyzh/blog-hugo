---
title: "Golang踩坑总结-使用下标获取字符串的字符"
slug: "golang-pitfall-string-subscript"
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
categories:
  - 技术
series:
  - Golang踩坑总结

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1ger268h4gdj30ix0ajjt5.jpg"

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

    -   使用下标获取字符串的字符时, 可能得到奇怪的字符

    -   ```go
        func main() {

            s := "hello"

            fmt.Printf("%c\n", s[1])

            s = "你好"

            fmt.Printf("%c\n", s[1])

        }

        //============输出===========

        e
        ½
        ```

2.  问题原因

    -   golang是以utf8格式保存字符串的, 字符串的下标操作, 访问的是字节, 而不是字符. `len`函数输出的也是字节数, 如`len("hello")==5`, `len("你好")==6`

3.  避免方式

    -   把字符串转化为`[]rune/[]int32`, 或者使用`range`遍历

    -   ```go
        func main() {

            s := "你好"

            // 强转为[]rune
            fmt.Printf("%c\n", []rune(s)[1])

            fmt.Println()

            // 使用range遍历
            for _, c := range s {
                fmt.Printf("%c\n", c)
            }

        }

        //===============输出=================

        好

        你
        好
        ```