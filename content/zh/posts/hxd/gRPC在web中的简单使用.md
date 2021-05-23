---
title: "gRPC在web中的简单使用"
slug: "grpc-web-frontend"
date: "2021-05-23T18:55:06+08:00"
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

image: "https://tva1.sinaimg.cn/large/008i3skNgy1gqslcasmdrj30g207sq36.jpg"

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

gRPC在web中的简单使用



通常我们在web中请求后端数据用的是XMLHttpRequest  或者fetch, 随着前端业务的复杂性增加，还有抽离专门的service文件用于请求接口，model文件用于封装数据状态；

gRPC是一种不同于这些途径和后端交互的一种新形式，RPC（Remote Procedure Call）叫做远程过程调用，是一种在本地调用远程函数的技术思想，gRPC是谷歌基于这一思想实现的框架。[什么是gRPC](https://juejin.cn/post/6844903897723568135)可以看这里，gRPC应用范围很广，支持多种语言多种平台，本文主要对web中的使用根据他的文档做一个简单介绍，主要看的是[grpcWEB](https://grpc.io/docs/platforms/web/basics/).

### 一：定义接口proto文件

使用gRPC首先我们需要定义一个前后端都要遵循的接口文档api.proto，我理解成这个就是一个对于前后端分别如何实现这个函数的一个说明文档, 比如下面的就是一个函数SayHello，参数是SayHelloReq类型的，有一个属性是name,返回值是SayHelloRsp的，返回值中有一个属性是reply

```javascript
service api {
  rpc SayHello (SayHelloReq) returns (SayHelloRsp) {}
}

message SayHelloReq {
  string name = 1;
}

message SayHelloRsp {
  string reply = 1;
}
```

### 二：后端实现接口文件

这里可以用任何一种语言来实现接口，比如文档里的例子就是用node 来实现的，我这里因为时间紧张请后端朋友帮忙实现了需要的接口

### 三： 修改代理

普通的Nginx 代理这里无法使用，可以参考[文档](https://grpc.io/docs/platforms/web/basics/#configure-the-envoy-proxy)来修改代理

### 四：生成文件

我们的第一步是写了一个协议文件来定义函数，但是具体的实现我们需要通过gRPC提供的工具来编译出具体的函数体

安装protoc 命令

```javascript
brew install protobuf
```

 安装 gRPC-Web protoc plugin

```javascript
$ cd grpc-web
$ sudo make install-plugin
```

 生成api_pb.js 和api_grpc_web_pb.js

```javascript
protoc --js_out=import_style=commonjs:. --grpc-web_out=import_style=commonjs,mode=grpcwebtext:. protobuf/api.proto
```

### 五：编写web客户端代码

 将上面生成的api_pb.js 和api_grpc_web_pb.js放到src文件夹中，新建个client.js:

```javascript
const {SayHelloReq, SayHelloRsp} = require('./api_pb.js');
const {apiClient} = require('./api_grpc_web_pb.js');

var echoService = new apiClient('http://127.0.0.1:50000');

var request = new SayHelloReq();
request.setName('hxd');

echoService.sayHello(request, {}, function(err, response) {
  console.log('123');
  console.log(response.getReply());
});
```

 我这里为了演示就在react项目的app.js  中   import './grpc/client.js'，将项目跑起来后可以看到

![image-20210523155717261](https://tva1.sinaimg.cn/large/008i3skNgy1gqsf2igwjtj31380gwtga.jpg)

返回的内容会经过proto buff 编码所以看起来是一个二进制的内容

![image-20210523154917960](https://tva1.sinaimg.cn/large/008i3skNgy1gqseu75wmoj31340fq77q.jpg)

可以看到该接口会将传入的名字拼接到hello ${name} ,have a nice day 中然后返回。

可以看到gRPC这种前后端交互是与我们之前一直所用的接口请求方式有很大的改变，前后端只要根据相同的ptoto来进行开发，也不用现在的接口文档了，而且这种数据传输方式完全看不到内容，更加安全可靠，基于http2.0 的传输方式也提高的很多的效率。

缺点就是接口每次修改还需要编译这些文件，很繁琐，看不到后端返回前端也更不好调试，我就是还得打印才能看到返回值。

参考：

https://grpc.io/docs/platforms/web/basics/

https://juejin.cn/post/6844903897723568135
