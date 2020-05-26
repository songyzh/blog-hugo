---
title: "Golang踩坑总结-接口值是否等于nil"
slug: "golang-pitfall-interface-value-nil"
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

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1ger1z2llizj30go0go75m.jpg"

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

    - 具体类型的nil值, 赋值给接口值变量后, 被判定不为nil

    -   ```go
        func main() {

            // *bytes.Buffer, 零值为nil
            var b *bytes.Buffer

            if b == nil {
                fmt.Println("外面的b等于nil")
            } else {
                fmt.Println("外面的b不等于nil")
            }

            f := func(b io.Writer) {
                if b == nil {
                    fmt.Println("里面的b等于nil")
                } else {
                    fmt.Println("里面的b不等于nil")
                }
            }

            // 把b传入函数
            f(b)

        }

        //===========输出===============

        外面的b等于nil
        里面的b不等于nil
        ```

2.  问题原因

    -   golang中的接口值, 除了有自己的类型type外, 还有***动态类型***(dynamic type)和***动态值***(dynamic value). ***接口值如果要被判断为nil, 需要动态类型和动态值都为nil***. 可以通过fmt的"%T", "%v"观察动态类型和动态值

3.  打印动态类型和动态值

    -   ```go
        func main() {

            var b *bytes.Buffer

            fmt.Printf("外面的b类型为%T\n", b)
            fmt.Printf("外面的b值为%v\n", b)

            if b == nil {
                fmt.Println("外面的b等于nil")
            } else {
                fmt.Println("外面的b不等于nil")
            }

            fmt.Println("")

            f := func(b io.Writer) {
                fmt.Printf("里面的b类型为%T\n", b)
                fmt.Printf("里面的b值为%v\n", b)
                if b == nil {
                    fmt.Println("里面的b等于nil")
                } else {
                    fmt.Println("里面的b不等于nil")
                }
            }

            f(b)

        }

        //==============输出=================

        外面的b类型为*bytes.Buffer
        外面的b值为<nil>
        外面的b等于nil

        里面的b类型为*bytes.Buffer
        里面的b值为<nil>
        里面的b不等于nil
        ```