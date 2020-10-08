---
title: "vue3.0的Composition-API"
slug: "vue-3.0-Composition-api"
date: "2020-10-08T22:40:06+08:00"
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
  - vue-3.0
categories:
  - 技术
series:
  -

image: "https://tva1.sinaimg.cn/large/007S8ZIlgy1gjib2gd512j30w00ftq2x.jpg"

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

vue3.0已经正式发布了，作为曾经vue2.0的用户当然迫不及待要来认识一下，趁假期有时间赶紧来小试牛刀～

在查看文档和一些demo的过程中发现一下午想掌握所有新货有点好高骛远了，那么就从3.0中备受瞩目也是最常会用到的composition-api 来开始吧。

composition-api 提出的原因是，在vue组件的逻辑中，我们原来的options api 是主张在组件内，根据逻辑来将代码写到对应的位置的，就是如下的结构

```javascript
export default {
  data () {
    return {}
  },

  methods: {},

  computed: {},

  watch: {}
}
```

在这种结构中，随着组件逻辑更加复杂，零落在各个参数中的函数/逻辑 复用困难，且无法和组件的生命周期结合，于是vue3.0 提出了composition-api，可以提取代码逻辑到一个单独的、可以利用生命周期函数的函数中来达到充分的逻辑复用，使我们的代码可以更加灵活的组合。

首先根据官网的[安装指导](https://vue3js.cn/docs/zh/guide/installation.html#cdn)来安装最新版本的vue，然后尝试做一个简易的todo list，要使用composition-api ，意味着我们的组件中不再有data,methods等2.0中的结构，setup函数是composition-api 的入口点，在组件初始化props 和beforeCreate 之间执行一次，setup返回的内容可以直接在组件中使用，那么如何达到数据变化页面更新的效果呢？答案就是在setup中使用vue 提供的各种composition-api 来达到这种目的。

我想实现可以添加待做事项，以及勾选事项代表该项已经做完，并且分别展示待做和已做列表，代码如下：

```javascript
<template>
  <div>
    <div>
      <input placeholder="输入待办" type="text"
      :value="todoText"
      @input="($event)=>{todoText = $event.target.value}">
      <button @click="addItem">添加</button>
    </div>
    <p>待办事项：</p >
    <ul>
      <li v-for="(item,i) in tobeDone" :key="item.text" @click="checkItem(item,i)">
        <input type="checkbox" name="item.text" value="item.text">{{item.text}}<br>
      </li>
    </ul>
    <p>已办完：</p >
    <ul>
      <li v-for="(item) in haveDone" :key="item.text" >
        <span>{{item.text}}</span>
      </li>
    </ul>
  </div>
</template>

<script>
import { ref, reactive, computed } from 'vue'
export default {
  name: 'App',
  setup() {
    let todoText = ref(''); // 返回 {value:''} 适用于简单双向绑定数据 defineproperty 实现
    let todoItems = reactive({list:[]}); // 适用于复杂引用类型双向绑定数据 proxy代理实现

    function addItem() {
      // state  0 未做 1 已做
      todoItems.list.push({text:todoText.value,state:0});
      todoText.value = '';
    }

    function checkItem(item) {
      item.state = item.state=== 0? 1 : 0;
    }

    // composition api 中的computed
    // 筛选已做好的
    let haveDone = computed(()=>{
      return todoItems.list.filter(item => item.state === 1);
    })

    // 筛选未做好的
    let tobeDone = computed(()=>{
      return todoItems.list.filter(item => item.state === 0);
    })

    return { // 返回的内容在组件中都可以直接拿到,不反回会在template中引用时候报错
      todoItems,
      addItem,
      todoText,
      checkItem,
      haveDone,
      tobeDone
    }
  }
}
</script>
```

完成效果如下：

![vue3](https://tva1.sinaimg.cn/large/007S8ZIlgy1gjiac6r7a7g30cl080t9g.gif)

从以上代码中可以看出，在vue3.0 中的组件中，使用composition-api 后逻辑主要集中在setup 中，这不禁让人觉得setup 中的代码不会过于臃肿和复杂吗？当然不会，在实际应用场景中，我们有很多拉取接口数据和在生命周期中某个阶段需要有一些动作的场景，这些都可以从setup 函数中抽离出来形成一个独立的js 模块提供封装好的某一部分逻辑，这些在[官方指导](https://vue3js.cn/docs/zh/guide/composition-api-introduction.html#%E4%BB%80%E4%B9%88%E6%98%AF-composition-api)中都有更多的示例和说明。

总体来说，在做这个demo的过程中，感觉到在应用到vue3.0的时候需要对每一个点去新文档中确认是否有变化，比如这里的v-model 就跟原来的用法有变化，直接用2.0还是无法与composition api 相适应的。另外，整体看来似乎跟react中的函数式组件+钩子函数有一些相似的用法，在编写组件的过程更多的是考虑数据的变化，生命周期函数只是完成功能的一个辅助力量，不像之前在编写组件的过程主线是vue实例的生命周期过程，另外，vue3.0 也提供了更多针对数据的api,例如isRef,isReadonly,isProxy 等等，对不同的场景使用不同的数据处理方式可以得到更好的组件性能。



参考：

https://vue3js.cn/docs/zh/guide/installation.html

https://vue3js.cn/docs/zh/guide/composition-api-introduction.html#%E4%BB%80%E4%B9%88%E6%98%AF-composition-api