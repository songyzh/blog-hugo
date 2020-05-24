---
title: "Golang defer语句用法小结"
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
categories:
  - 技术
series:
  -

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1gequsxb9r0j30hq0dbmy7.jpg"

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

-   程序执行到defer语句的时候, 被defer的函数的***实参***会在此时被求值

    -   ```go
        func a() {
            i := 0
            // 被defer的函数实参会在此时被求值, 这里为0
            defer fmt.Println(i)
            // i自增, 但不会影响defer的函数
            i++
            return
        }
        //===========输出===========
        0
        ```

-   被defer的函数调用, 执行顺序是"后进先出"

-   defer的函数调用会被压入一个栈中。当外层函数返回时，被推迟的函数会按照后进先出的顺序调用。

-   如果函数的返回值是***命名的***, defer语句中的函数可以读取或修改该值

    -   ```go
        // 修改命名的返回值
        func c() (i int) {
            // 在defer语句中自增
            defer func() { i++ }()
            // 这里i被设为1
            return 1
        }
        //=============输出============
        2
        ```

    -   ```go
        // 返回值是匿名的, 无法被defer修改
        func c() int {
            i := 1
            defer func() { i++ }()
            return i
        }
        //=============输出============
        1
        ```


-   实参估值和修改返回变量的例子

    -   ```go
        func test() (x int) {
            // defer语句的实参会被立刻估值, 因此实参x==0
            defer func(n int) {
                // 实参x估值为0, 因此形参n==0
                fmt.Printf("in defer x as parameter: x = %d\n", n)
                // 这里的x是外面的x, 因此为9
                fmt.Printf("in defer x after return: x = %d\n", x)
            }(x)

            x = 7
            // x==9
            return 9
        }
        // =============执行结果==============
        in defer x as parameter: x = 0
        in defer x after return: x = 9
        ```

-   最好在获取资源之后, 立刻调用defer

-   defer在***其所在的函数***的末尾执行, 因此在for循环中使用时要注意

    -   ```go
        // 有问题的写法
        for _, filename := range filenames {
            f, err := os.Open(filename)
            if err != nil {
                return err
            }
            // 释放资源的操作直到所在函数末尾才执行, 如果文件很多, 可能导致文件描述符用尽
            defer f.Close()
            // 处理文件
        }
        ```


    -   ```go
        // 改进写法
        for _, filename := range filenames {
            if err := doFile(filename); err != nil {
                return err
            }
        }

        // 抽取处理单个文件的逻辑, 此函数返回时即可关闭文件
        func doFile(filename string) error {
            f, err := os.Open(filename)
            if err != nil {
                return err
            }
            defer f.Close()
            // 处理文件
        }
        ```


