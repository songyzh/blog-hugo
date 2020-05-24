---
title: "Golang sync包用法小结"
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
  phone: ""
  facebook: ""
  twitter: ""
  github: "https://github.com/songyzh"

tags:
  - 后端
  - Golang
  - GoSync
categories:
  - 技术
series:
  -

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1gepo1j18koj30m80bc40e.jpg"

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

-   sync包提供传统的内存访问的同步机制

-   WaitGroup

    -   等待若干goroutine执行完毕

    -   ```go
        // 注意: 如果wg是在闭包环境中使用, 由于闭包中捕获的是变量本身, 因此直接使用wg变量即可. 但如果是下面代码这样, 作为函数的参数调用, 则需要传引用来保证使用的是同一个wg

        // 传入公用的wg对象
        hello := func(wg *sync.WaitGroup, id int) {
            // 执行完成后, 内部值-1
            defer wg.Done()
            fmt.Printf("Hello from %v!\n", id)
        }

        // 定义goroutine数量
        const numGreeters = 5

        // 初始化waitgroup
        var wg sync.WaitGroup

        // 一次性增加内部值, 也可以在for语句中逐个增加
        wg.Add(numGreeters)

        for i := 0; i < numGreeters; i++ {
            // wg.Add(1) 如果使用逐个增加的方式
            go hello(&wg, i+1)
        }

        // 阻塞, 直到内部值变为0
        wg.Wait()

        //===========输出===============
        // 顺序不确定
        Hello from 1!
        Hello from 5!
        Hello from 2!
        Hello from 3!
        Hello from 4!
        ```

-   Mutex and RWMutex

    -   保护关键区域(guard critical sections)

    -   Mutex

        -   ```go
            var count int

            var lock sync.Mutex

            // 下面两个函数都是线程安全的
            increment := func() {
                lock.Lock()
                // 使用defer, 确保锁被释放
                defer lock.Unlock()

                count++
                fmt.Printf("Incrementing: %d\n", count)
            }
            decrement := func() {
                lock.Lock()
                defer lock.Unlock()

                count--
                fmt.Printf("Decrementing: %d\n", count)
            }
            ```

    -   RWMutex

        -   可以同时有任意多个读锁, 或一个写锁

        -   读锁是共享的, 写锁是排他的

        -   ```go
            var mu sync.RWMutex
            var balance int

            func Balance() int {
                mu.RLock() // readers/shared lock
                defer mu.RUnlock()
                return balance
            }

            func Deposit(amount int) {
                mu.Lock() // // writer/exclusive lock
                defer mu.Unlock()
                balance += amount
            }
            ```

-   Cond

    -   在某个点等待某事件(event)的发生, 在此之前保持阻塞/挂起状态, 使其他goroutine可以执行

    -   用于notify的方法

        -   Signal: 通知等待最久的goroutine, 这个功能也可以用channel实现
            -   runtime维护了一个FIFO保存等待通知的goroutine, 因此会通知等待最久的goroutine
        -   Broadcast: 通知所有goroutine, 这个功能用channel不好实现

    -   性能比使用channel实现好很多

    -   ```go
        // 假设我们有一个固定len为2的queue, 我们想加入10个元素进去. 我们希望queue一有多余的空间, 就能通知我们, 让我们插入新的元素

        // 创建一个cond, 需要传入一个sync.Locker来保护关键区域
        c := sync.NewCond(&sync.Mutex{})

        // 初始化queue
        queue := make([]interface{}, 0, 10)

        // dequeue函数
        removeFromQueue := func(delay time.Duration) {
            // sleep delay的时间
            time.Sleep(delay)
            // 使用lock保护关键区域
            c.L.Lock()
            // dequeue
            queue = queue[1:]
            fmt.Println("Removed from queue")
            // unlock, 离开关键区域
            c.L.Unlock()
            // 通知等待最久的goroutine, 也可以用broadcast来通知所有
            c.Signal()
        }
        for i := 0; i < 10; i++ {
            // 进入关键区域
            c.L.Lock()
            // 需要使用for循环来检查条件. 因为接到通知并不一定意味着我们在等待的事情已经发生, 所以需要再次检查条件
            for len(queue) == 2 {
                // 在内部会先调用unlock, 然后等待. 接到通知继续执行后, 会调用lock
                c.Wait()
            }
            fmt.Println("Adding to queue")
            // enqueue
            queue = append(queue, struct{}{})
            // 启动goroutine, 1s后执行dequeue
            go removeFromQueue(1 * time.Second)
            // 离开关键区域
            c.L.Unlock()
        }
        ```

-   Once

    -   每个sync.Once的对象, 只能调用1次`Do`方法, 之后的对`Do`的调用无效.

    -   ```go
        var count int
        increment := func() { count++ }
        decrement := func() { count-- }

        var once sync.Once
        // 第一次调用Do方法, 有效
        once.Do(increment)
        // 此次调用Do方法无效
        once.Do(decrement)

        fmt.Printf("Count: %d\n", count)
        // count为1, once对象只能执行一次DO方法
        ```

-   Pool

    -   创建一个对象池, 其中的对象可以复用(比如数据库连接), 且是线程安全的

    -   ```go
        myPool := &sync.Pool{
            // 生成新对象
            New: func() interface{} {
                fmt.Println("Creating new instance.")
                return struct{}{}
            },
        }

        // pool是空的, 因此会创建新对象
        myPool.Get()

        // 上面创建的对象没有被放回, 因此现在pool中还是空的
        // 由于pool是空的, 因此会创建新对象
        instance := myPool.Get()

        // 放回对象, 经常配合defer使用
        myPool.Put(instance)

        // 由于pool中有对象, 因此不会创建新对象
        myPool.Get()

        //============输出============
        Creating new instance.
        Creating new instance.
        ```