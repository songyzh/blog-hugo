---
title: "JavaScript的继承总结"
date: "2020-05-24T12:00:06+08:00"
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
  phone: ""
  facebook: ""
  twitter: ""
  github: "https://github.com/happyElina"

tags:
  - 前端
  - JavaScript
categories:
  - 技术
series:
  -

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1gekw11opskj312w0jgq5b.jpg"

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

##### js的继承总结

###### 本质都是原型链，设置子类的prototype，应用的时候实例属性定义在实例上，通用方法定义在原型链上，达到较好的复用和扩展效果。

###### 方法一 继承父类的实例

```
function Father() {
    this.property = true; // 实例属性
}

Father.prototype.getFatherValue = function(){
    return this.property; // 通用方法
}

function Son(){
    this.sonProperty = false;
}

Son.prototype = new Father();//继承

Son.prototype.getSonProperty = function(){
    return this.sonProperty
}

var instance = new Son();

instance.getFatherValue()
```

###### 方法二 object.create

此方法是《你不知道的js》所推荐的方法，已经被广泛使用，此方法可以避免 new Father() 有可能会带来的负面作用

```
function Father() {
    this.property = true; // 实例属性
}

Father.prototype.getFatherValue = function(){
    console.log('father')
    return this.property; // 通用方法
}

function Son(){
   Father.apply(this,arguments)
   // 子类借用父类的构造方法 实现继承实例属性

}
// 注意是 Father.prototype
Son.prototype =  Object.create(Father.prototype,{
    constructor: {
      value: Father; // 弥补这种方法的不足
    }
})

// 或者添加一句
// Son.constructor = Father

Son.prototype.getSonProperty = function(){
    return this.sonProperty
}

var instance = new Son();

instance.getFatherValue()
```

##### 方法三 es6 class

本质还是原型链的语法糖，但是对于面向对象开发友好

```
class Father {
  constructor(name) {
    this.name = name; // 增加到实例上
  }

  // 这里声明的普通方法都添加到原型上
  drink() {
    console.log('father drink')
  }
}

class Son extends Father { // 关键是extends
  constructor(name) { // 如果没有constructor 默认使用父类的
    // 这里的super指向的是父类 默认里面的this就是子类
    super(name); // Animal.call(this)
  }
  drink() {
    // super = Super.prototype 这个super不能单独打印
    super.drink();
    console.log('son drink')
  }

}
let son = new Son();
son.drink()
// super 指向有两种可能 在constructor 和 static中指向的是父类
// 在子类的原型方法中指向的是父类原型
// 静态方法 就是通过类来调用的方法
```

##### 方法四 Object.setPrototypeOf

原理 方法设置一个指定的对象的原型 ( 即, 内部[[Prototype]]属性），mdn 从性能考虑不建议 [链接](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf)

```
// 就是直接修改 __proto__
function Father() {
    this.property = true; // 实例属性
}

Father.prototype.getFatherValue = function(){
    console.log(123)
    return this.property; // 通用方法
}

function Son(){
    this.sonProperty = false;
}

Object.setPrototypeOf(Son.prototype,Father.prototype)

Son.prototype.getSonProperty = function(){
    return this.sonProperty
}

var instance = new Son();

instance.getFatherValue()
```