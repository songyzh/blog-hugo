---
title: "Golang交替打印的几种实现方式"
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
  - Goroutine
  - GoChannel
  - GoSync
categories:
  - 技术
series:
  -

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1geome7841kj30jg0adt8y.jpg"

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

提供了Golang交替打印的几个思路: 使用Goroutine, sync包中的WaitGroup和Cond

1.  使用unbuffered channel实现

    -   ```go
        func PrintABUnbufferedChannel() {
            // 使用2个channel实现交替运行
            c1 := make(chan struct{})
            c2 := make(chan struct{})

            // 打印aaa的函数, 对2个channel执行send操作
            PrintA := func() {
                for {
                    c1 <- struct{}{}
                    fmt.Println("aaa")
                    time.Sleep(1 * time.Second)
                    c2 <- struct{}{}
                }
            }

            // 打印bbb的函数, 对2个channel执行receive操作
            PrintB := func() {
                for {
                    <-c2
                    fmt.Println("bbb")
                    time.Sleep(1 * time.Second)
                    <-c1
                }
            }

            // 先打印aaa
            go func() {
                <-c1
            }()

            go PrintA()
            go PrintB()

            time.Sleep(1 * time.Hour)

        }
        ```

2.  使用`sync.Cond`配合`sync.WaitGroup`实现

    -   ```go
        func PrintABCond() {
            // 使用2个cond实现交替运行
            c1 := sync.NewCond(&sync.Mutex{})
            c2 := sync.NewCond(&sync.Mutex{})

            // 使用waitgroup协调顺序, 确保printA, printB先启动, 再发送第一个signal
            var wg sync.WaitGroup

            // 打印aaa的函数, 等待c1, 通知c2
            printA := func() {
                // 确保printA启动早于第一个signal
                wg.Done()
                for {
                    c1.L.Lock()
                    // 等待信号
                    c1.Wait()
                    fmt.Println("aaa")
                    time.Sleep(1 * time.Second)
                    // 发送信号
                    c2.Signal()
                    c1.L.Unlock()
                }
            }

            // 打印bbb的函数, 等待c2, 通知c1
            printB := func() {
                // 确保printB启动早于第一个signal
                wg.Done()
                for {
                    c2.L.Lock()
                    // 等待信号
                    c2.Wait()
                    fmt.Println("bbb")
                    time.Sleep(1 * time.Second)
                    // 发送信号
                    c1.Signal()
                    c2.L.Unlock()
                }
            }

            wg.Add(2)

            // 先打印aaa
            go func() {
                // 等待printA, printB启动后, 再发送第一个signal
                wg.Wait()
                c1.Signal()
            }()

            go printA()
            go printB()

            time.Sleep(1 * time.Hour)
        }
        ```

3.  使用`for-select`语句实现, 与方法1类似

    -   ```go
        func PrintABForSelect() {

            c1 := make(chan struct{})
            c2 := make(chan struct{})

            printAB := func() {
                for {
                    select {
                    case <-c1:
                        fmt.Println("aaa")
                        time.Sleep(1 * time.Second)
                        c2 <- struct{}{}
                    case <-c2:
                        fmt.Println("bbb")
                        time.Sleep(1 * time.Second)
                        c1 <- struct{}{}
                    }
                }
            }

            go func() {
                c1 <- struct{}{}
            }()

            go printAB()
            go printAB()

            time.Sleep(1 * time.Hour)
        }
        ```