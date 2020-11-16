---
title: "浏览器performance性能优化"
slug: "evaluate-performance"
date: "2020-11-16T22:06:06+08:00"
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

image: "https://tva1.sinaimg.cn/large/0084UQshgy1gkrd8qjgylj30zq0l4aiu.jpg"

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

在最近的工作中涉及到了页面大数量的计算和渲染，因为没能很好的利用浏览器开发工具中的perfomance，导致上线后性能遭到了一些吐槽，好在及时通过performance profiling 的过程发现了瓶颈所在，这篇文章就是对于谷歌开发工具官网中的教程[evaluate-performance](https://developers.google.com/web/tools/chrome-devtools/evaluate-performance)的一次翻译和学习。

教程是Chrome 59，我本地安装的是86，可能会有一些差异。

### get started

1 [无痕模式](https://support.google.com/chrome/answer/95464) 打开一个浏览器窗口（**⌘ + Shift + n**），无痕模式确保浏览器在一个比较干净的状态下运行，比如说你安装了一些插件，那么这些插件可能会影响你测量性能时候的指标；

2 无痕窗口中打开下面的网址，这就是你将要profile分析的网页，网页中有很多上下移动的方块；

```javascript
	https://googlechrome.github.io/devtools-samples/jank/
```

3 输入 Command+Option+I (Mac) or Control+Shift+I (Windows, Linux) 命令打开 DevTools.

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkpwnwenjvj31tm0u00zq.jpg" alt="屏幕快照 2020-11-15 15.48.02" style="zoom:50%;" />



### simulate a mobile cpu

手机相较于台式和笔记本电脑有相对弱一些的中央处理器，当我们分析一个网页的时候，可以使用CPU Throttling 来模拟手机页面效果。

1 DevTools中点击performan 栏；

2 确保勾选了**Screenshots**；

2 点击performace下的设置，DevTools显示出它如何捕获性能指标相关的设置；

3 这里我选择4x slowdown 来限制cpu 效率是通常4倍的慢速

![1605433693333](https://tva1.sinaimg.cn/large/0081Kckwgy1gkq051v2wmj30yk07wq4f.jpg)

### 	Set up the demo

要创建一个让所有读者都看到相同性能分析效果的demo不太容易，这部分能让你定制自己的demo来确保能看到和教程类似的运行结果。

1 点击**Add 10** 直到蓝色方块明显异动的比之前更慢，高配的机器可能需要20次；

2 点击**Optimize**，蓝色方块应该会明显移动更快和顺畅；

3 点击**Un-Optimize**，蓝色方框应该再次移动的慢且卡顿。

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkq0gegmejj30u00v112q.jpg" alt="屏幕快照 2020-11-15 17.59.14" style="zoom: 33%;" />



### Record runtime performance

当你点击页面的优化版本时，蓝色方块移动更快，原因是什么？两个版本都应该在相同的距离和时间里移动每一个方块，这里用performance 录一段分析资料来分析没有优化时页面的性能瓶颈。

1 点击浏览器performance 中的  **Record** ![Record](https://developers.google.com/web/tools/chrome-devtools/evaluate-performance/imgs/record.png)，devtools 会抓取页面运行时候的性能指标；

![1605434926103](https://tva1.sinaimg.cn/large/0081Kckwgy1gkq0sc0umej30yi09sabj.jpg)

2 等待几秒

![屏幕快照 2020-11-15 18.09.42](https://tva1.sinaimg.cn/large/0081Kckwgy1gkq0sgsko6j31ty0u07hy.jpg)

3 点击**Stop**，devtools停止记录，处理数据，然后把结果展示在p er formance 面板里

![屏幕快照 2020-11-15 18.10.36](https://tva1.sinaimg.cn/large/0081Kckwgy1gkq0slfs9dj31hi0u0qoh.jpg)



### Analyze the results

得到一份性能记录之后，就可以判断当前页面的性能好坏，得到不好的原因。

#####  Analyze frames per second

分析一个页面动画性能的主要指标就是FPS(帧率)，用户们普遍都在帧率时60 的时候比较满意（也就是每秒页面刷新60次）

1 看看我们的FPS 图表，当你看到FPS 上的红条时，说明帧率已经降到了会影响到用户体验的程度，通常来说，绿条越高说明帧率越高；

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkq2vqkhw9j311i0patfb.jpg" alt="1605439295714" style="zoom:50%;" />

2 在FPS 下面还能看到cpu 图表，CPU 图中颜色对应着最下面的summary 图标的颜色，事实是，CPU图表充满了颜色，意味着CPU在录音期间达到了极限。每当你看到CPU长时间超负荷工作时，就暗示着你要想办法让它减少工作量。

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkq2vwqscxj30ug0u0wn0.jpg" alt="1605439339289" style="zoom:50%;" />

3 将鼠标悬浮在**FPS**, **CPU**, 或者 **NET** 上面， DevTool会展示当前页面的内容，将鼠标从左往右移动可以看到录制时候的页面动态情况，这就叫scrubbing（擦洗？这块我也不太理解是个啥意思），这对手动分析动画过程非常有用。

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkq2wppcnfj311a0o0tpj.jpg" alt="屏幕快照 2020-11-15 19.24.06" style="zoom:50%;" />

4 在**Frames**  部分，鼠标悬浮在一个绿色方块上可以看到当时那一帧的帧率（FPS）,每帧可能都低于60帧/秒的目标。(原来以为是我的谷歌版本较新，没有悬浮展示帧率的效果，后来发现是可以将下面的js heap - GPU 那一行右侧往下拖，然后就可以看到了)

![1605439753056](https://tva1.sinaimg.cn/large/0081Kckwgy1gkq32wexifj311c0dmn1g.jpg)

![1605440148104](https://tva1.sinaimg.cn/large/0081Kckwgy1gkq392xi8uj315a0lkjy3.jpg)

通过这个演示，很明显页面的性能不是很好。但是在真实的场景中，它可能不是那么清楚，所以拥有所有这些工具来进行测量是很方便的。



### Bonus: Open the FPS meter

另一个好用的工具就是FPS meter，可以实时提供当前页面的帧率，

1 用Command+Shift+P (Mac) 或者 Control+Shift+P  (Windows, Linux) 打开菜单栏；

2 输入rendering,找到show frames per second ...,点击选中，然后就可以看到页面上的实时帧率

3 关闭也是1步骤，然后选项就是 hide  frames per second ...

这里我的操作和文中的显示的图不一样，查阅了一些没查出原因。。。esc 也并不能退出

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkq3k7h64nj313u0nsafs.jpg" alt="1605440562695" style="zoom:50%;" />

文中的效果：

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkq3w9gmenj310i0moanx.jpg" alt="屏幕快照 2020-11-15 19.58.05" style="zoom:50%;" />

我的效果：

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkq3wjhfrfj30ze0e6jt3.jpg" alt="屏幕快照 2020-11-15 19.58.15" style="zoom:50%;" />

不知为何显出这一堆数字而不是fps..数字还有重叠...

### Find the bottleneck
现在已经衡量和确定了当前的动画效果是有问题的，接下来就找一下原因。

1 注意到summary 栏中，当没有选中某个事件的时候给出了所有活动的一个拆分时间。这个页面主要的时间都用到了rendering,既然性能优化就是一门做更少事情的艺术，我们的目标就是减少rendering 的工作所占用的时间；

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkrbfivbxcj30ss0eq3zu.jpg" alt="1605531157744" style="zoom:50%;" />

2 打开Main  这一栏，DevTools向你展示了主线程随时间变化的活动火焰图。x轴代表记录时间，每块方块代表一个事件。方块越宽意味着事件耗时比较长，y轴表示调用堆栈。当你看到堆栈堆积在一起时，这意味着上面的事件导致了下面的事件；

![1605531332160](https://tva1.sinaimg.cn/large/0081Kckwgy1gkrbg1tn56j31a60ngdns.jpg)

3 记录里有很多数据。通过在上面总览部分单击、按住并拖动鼠标来放大单个动画帧所触发的事件，总览是包含FPS、CPU和NET图表的部分。Main部分和Summary选项卡仅显示记录的选定部分的信息（上图中有选中部分）。
4 注意到Animation Frame Fired事件右上角的红色三角形，当看到这个红色三角形的时候，意味着这个事件可能又一些问题；

5 点击Animation Frame Fired 事件，Summary 栏会给出这个事件的信息。注意到下面的reveal 链接，点击这个链接，DevTools  会高亮显示触发 **Animation Frame Fired** 事件的原因，注意到**app.js:94**链接，点击可以进入源码行；

![1605531730408](https://tva1.sinaimg.cn/large/0081Kckwgy1gkrbh7rkzvj31840u0wng.jpg)

![1605531773790](https://tva1.sinaimg.cn/large/0081Kckwgy1gkrbhqqxj9j31e80iogqv.jpg)

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkrbhqqxj9j31e80iogqv.jpg" alt="1605531773790" style="zoom:50%;" />

6  在 **app.update** 事件下面可以看到很多紫色的事件，如果它们更宽一些的话，看起来每个紫色块右上角都有又个红色三角形，现在点击一个紫色的 **Layout**事件，下面的DevTools会给出这个事件的详情，事实上，这里有一个关于强制回流(布局的另一种说法)的警告。

![1605532856654](https://tva1.sinaimg.cn/large/0081Kckwgy1gkrbxjvgerj31460a6gov.jpg)

<img src="https://tva1.sinaimg.cn/large/0081Kckwgy1gkrbxobwgcj30zg0jw40k.jpg" alt="1605532901843" style="zoom:50%;" />

7 点击上图的**Summary**栏中的app.js:70,可以看到强制回流的那一行代码：

![1605533354785](https://tva1.sinaimg.cn/large/0081Kckwgy1gkrc6gz58jj31360eqq6w.jpg)

至此我们找到了原因（**Figure 13**）：

代码的原因是，每一帧的动画中都改变了每一个方块的style属性，然后立刻访问每个方块在页面中的位置，因为上面刚刚改变了方块的style 属性，浏览器不知道是否每个方块的位置发生了变化，所以不得不进行重排来计算方块的位置，[这里查看如何避免页面同步重新布局](https://developers.google.com/web/fundamentals/performance/rendering/avoid-large-complex-layouts-and-layout-thrashing#avoid_forced_synchronous_layouts)



### Bonus: Analyze the optimized version

使用上面学习的过程和工具，点击示例页面中的**Optimize**，然后在进行一次录制，分析结果，从提高的帧率到减少主部分火焰图中的事件，你可以看到优化版本的应用做了更少的工作，得到了更好的性能。

下面是一些优化页面的链接这里就不多说了，over。
