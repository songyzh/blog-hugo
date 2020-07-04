---
title: "Golang踩坑总结-把slice传入函数"
slug: "golang-pitfall-slice-as-argument"
date: "2020-07-04T21:00:06+08:00"
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

1.  问题表现: 把slice传入函数并修改, 所做的append操作在函数外会丢失

    -   ```go
        func main() {
        	// 初始化slice
        	s := make([]int, 0, 10)
        	s = append(s, 1, 2, 3)
        	// 打印当前slice的信息
        	header := (*reflect.SliceHeader)(unsafe.Pointer(&s))
        	fmt.Printf("main start - %v, %#v\n", s, header)
        	// 把slice传入函数
        	helper(s)
        	// 打印当前slice的信息
        	header = (*reflect.SliceHeader)(unsafe.Pointer(&s))
        	fmt.Printf("main end - %v, %#v\n", s, header)
        }

        func helper(s []int) {
        	// 打印刚传入时slice的信息
        	header := (*reflect.SliceHeader)(unsafe.Pointer(&s))
        	fmt.Printf("helper start - %v, %#v\n", s, header)
        	// 向slice中插入数据
        	s = append(s, 4, 5)
        	// 打印插入数据后slice的信息
        	header = (*reflect.SliceHeader)(unsafe.Pointer(&s))
        	fmt.Printf("helper end - %v, %#v\n", s, header)
        }

        // ===================输出======================
        // 可以看到, 即使slice的地址保持不变, 在helper函数中做的修改还是丢失了.
        main start - [1 2 3], &reflect.SliceHeader{Data:0xc00001c0a0, Len:3, Cap:10}
        helper start - [1 2 3], &reflect.SliceHeader{Data:0xc00001c0a0, Len:3, Cap:10}
        helper end - [1 2 3 4 5], &reflect.SliceHeader{Data:0xc00001c0a0, Len:5, Cap:10}
        main end - [1 2 3], &reflect.SliceHeader{Data:0xc00001c0a0, Len:3, Cap:10}
        ```

2.  问题原因
    -   因为golang总是传值, slice在传入helper函数后, helper函数中的s变量是一个新的变量, append操作修改的是此局部变量的Len值. 而main函数中的s变量, 其Len值始终没有改变.
    -   这里需要注意, 如果在helper中修改了slice中的值, 比如`s[0] = 100`, 这个会反映在main函数中的s变量上. 因为两个s变量共用一个底层array
3.  避免方式

    -   如果想在函数中修改原始的slice, 可以传递引用, 或是返回新的slice. 传递引用的代码如下

    -   ```go
        func main() {
        	// 初始化slice
        	s := make([]int, 0, 10)
        	s = append(s, 1, 2, 3)
        	// 打印当前slice的信息
        	header := (*reflect.SliceHeader)(unsafe.Pointer(&s))
        	fmt.Printf("main start - %v, %#v\n", s, header)
        	// 把slice的引用传入函数
        	helper(&s)
        	// 打印当前slice的信息
        	header = (*reflect.SliceHeader)(unsafe.Pointer(&s))
        	fmt.Printf("main end - %v, %#v\n", s, header)
        }

        // 接收引用作为参数
        func helper(s *[]int) {
        	// 打印刚传入时slice的信息
        	header := (*reflect.SliceHeader)(unsafe.Pointer(s))
        	fmt.Printf("helper start - %v, %#v\n", s, header)
        	// 向slice中插入数据
        	*s = append(*s, 4, 5)
        	// 打印插入数据后slice的信息
        	header = (*reflect.SliceHeader)(unsafe.Pointer(s))
        	fmt.Printf("helper end - %v, %#v\n", s, header)
        }

        //==================输出=====================
        main start - [1 2 3], &reflect.SliceHeader{Data:0xc00001c0a0, Len:3, Cap:10}
        helper start - &[1 2 3], &reflect.SliceHeader{Data:0xc00001c0a0, Len:3, Cap:10}
        helper end - &[1 2 3 4 5], &reflect.SliceHeader{Data:0xc00001c0a0, Len:5, Cap:10}
        main end - [1 2 3 4 5], &reflect.SliceHeader{Data:0xc00001c0a0, Len:5, Cap:10}
        <!-- ``` -->