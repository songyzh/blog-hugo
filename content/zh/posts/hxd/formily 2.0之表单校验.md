---
title: "formily 2.0之表单校验"
slug: "formily-2-check"
date: "2021-07-25T21:19:06+08:00"
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

image: "https://tva1.sinaimg.cn/large/008i3skNgy1gstifh6umjj31qi0u0769.jpg"

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

formily 2.0之表单校验

#### 前言

表单作为web前端在页面中收集、展示信息的媒介，在web前端领域是不可或缺的一部分，但是由于其使用场景的复杂性，导致表单的一系列需求（比如表单校验，表单联动，表单布局）都是严重耦合在前端代码中的，[formily](https://v2.formilyjs.org/zh-CN/guide)作为阿里的一整套表单解决方案，给出了一个让人欣喜的答案。

本文意在探讨一下在这套复杂解决方案中，它的校验系统是怎么安排的，看的是2.0.0-beta.76版本。

#### 一 、表单校验系统在整体formily 中的位置

要想知道这个问题，首先必要的就是看下官方给出的一张分层架构图：

![formily2](https://tva1.sinaimg.cn/large/008i3skNgy1gskxmpk3pmj31a60rwae5.jpg)

在看完官方大概介绍我们就会了解，formily这一套低前端代码，高业务聚焦的解决方案，核心就是formily/core,这一套核心，实现了路径系统，表单周期，数据视图双向绑定等等表单通用功能，一个完整内聚的view model 可以提供给任意的前端框架！

在图的左下角，就是我们要研究一下的表单校验系统，可以理解为它是formily/core 的一个底层依赖，所有表单校验的相关代码都在这里面，当然，formily/core 里也会向外直接暴露出一些必要的validator的接口以供开发者使用，比如定制的表单校验文案方法`registerValidateLocale`,后面会发现这就是validator/src/registry中的`registerValidateLocale`方法。

#### 二、表单系统校验的主要方法

在看完官方[表单校验示例](https://v2.formilyjs.org/zh-CN/guide/advanced/validate)后，我们大概有以下一些印象：

1. 从使用方法上支持两种方式： Markup Schema 和 JSON Schema，从传参上讲，又别支持简单传参、和x-validator传参、x-validator 数组传参等多种传参方式，根据需要选择，效果都是一样的。

   比如Markup Schema的三种传参（摘取部分代码结构，关注x-validator参数）：

   ```javascript
         <SchemaField.String
           name="required_1"
           title="必填"
           required
           x-component="Input"
           x-decorator="FormItem"
         />
         <SchemaField.String
           name="required_2"
           title="必填"
           x-validator={{ required: true }}
           x-component="Input"
           x-decorator="FormItem"
         />
         <SchemaField.String
           name="required_3"
           title="必填"
           x-validator={[{ required: true }]}
           x-component="Input"
           x-decorator="FormItem"
         />
   ```

   比如 JSON Schema的三种传参（摘取部分schema结构，关注x-validator参数）：

   ```javascript
       max_1: {
         name: 'max_1',
         title: '最大值(>5报错)',
         type: 'number',
         maximum: 5,
         'x-decorator': 'FormItem',
         'x-component': 'NumberPicker',
       },
       max_2: {
         name: 'max_2',
         title: '最大值(>5报错)',
         type: 'number',
         'x-validator': {
           maximum: 5,
         },
         'x-decorator': 'FormItem',
         'x-component': 'NumberPicker',
       },
       max_3: {
         name: 'max_3',
         title: '最大值(>5报错)',
         type: 'number',
         'x-validator': [
           {
             maximum: 5,
           },
         ],
         'x-decorator': 'FormItem',
         'x-component': 'NumberPicker',
       },
   ```

2. 从格式校验来说，支持内置格式校验和自定义格式校验（这里就不区分格式和规则了，分别是`registerValidateFormats` 和 `registerValidateRules`方法），也就是如果你不想麻烦，他有一些写好的校验直接拿来用，如果你有复杂的校验逻辑需要自己写，也可以有接口支持。

   比如使用内置的校验格式来校验电话：

   ```javascript
      <SchemaField.String
           name={'phone'}
           title={'phone格式'}
           format={'phone'}
           required
           x-component="Input"
           x-decorator="FormItem"
         />
         <SchemaField.String
           name={'phone'}
           title={'phone格式'}
           required
           x-validator={'phone'}
           x-component="Input"
           x-decorator="FormItem"
         />
         <SchemaField.String
           name={'phone'}
           title={'phone格式'}
           required
           x-validator={{phone:'phone'}}
           x-component="Input"
           x-decorator="FormItem"
         />
   ```

   需要注册自己的校验格式：

   ```javascript
   import React from 'react'
   import { createForm, registerValidateFormats } from '@formily/core'
   import { createSchemaField } from '@formily/react'
   import { Form, FormItem, Input } from '@formily/antd'

   const form = createForm()

   const SchemaField = createSchemaField({
     components: {
       Input,
       FormItem,
     },
   })

   registerValidateFormats({
     custom_format: /123/,
   })

   export default () => (
     <Form form={form} labelCol={6} wrapperCol={10}>
       <SchemaField>
         <SchemaField.String
           name="global_style_1"
           title="全局注册风格"
           required
           x-validator={{
             format: 'custom_format',
             message: '错误❎',
           }}
           x-component="Input"
           x-decorator="FormItem"
         />
         <SchemaField.String
           name="global_style_2"
           title="全局注册风格"
           required
           x-validator={'custom_format'}
           x-component="Input"
           x-decorator="FormItem"
         />
         <SchemaField.String
           name="global_style_3"
           title="全局注册风格"
           required
           x-validator={['custom_format']}
           x-component="Input"
           x-decorator="FormItem"
         />
         <SchemaField.Number
           name="global_style_4"
           title="全局注册风格"
           required
           x-validator={{
             format: 'custom_format',
             message: '错误❎',
           }}
           x-component="Input"
           x-decorator="FormItem"
         />

         <SchemaField.String
           name="validator_style_1"
           title="局部定义风格"
           required
           pattern={/123/}
           x-component="Input"
           x-decorator="FormItem"
         />
         <SchemaField.String
           name="validator_style_2"
           title="局部定义风格"
           required
           pattern="123"
           x-component="Input"
           x-decorator="FormItem"
         />
         <SchemaField.String
           name="validator_style_3"
           title="局部定义风格"
           required
           x-validator={{
             pattern: /123/,
             message: '错误了❎',
           }}
           x-component="Input"
           x-decorator="FormItem"
         />
         <SchemaField.String
           name="validator_style_4"
           title="局部定义风格"
           required
           x-validator={{
             pattern: '123',
             message: '错误了❎',
           }}
           x-component="Input"
           x-decorator="FormItem"
         />
       </SchemaField>
     </Form>
   )
   ```

   可以看到表单校验的使用方法是灵活多样的，支持场景下的表单校验。

   那么这些校验系统是如何实现的呢？ 在看代码前我们需要看看他的关于表单的类型声明，这样才对他的抽象结构有一定了解[fieldvalidator](https://core.formilyjs.org/zh-CN/api/models/field#fieldvalidator)：

   ```javascript
   //字符串型格式校验器
   type ValidatorFormats =
     | 'url'
     | 'email'
     | 'ipv6'
     | 'ipv4'
     | 'number'
     | 'integer'
     | 'idcard'
     | 'qq'
     | 'phone'
     | 'money'
     | 'zh'
     | 'date'
     | 'zip'
     | (string & {}) //其他格式校验器需要通过registerValidateFormats进行注册

   //对象型校验结果
   interface IValidateResult {
     type: 'error' | 'warning' | 'success' | (string & {})
     message: string
   }
   //对象型校验器
   interface IValidatorRules<Context = any> {
     triggerType?: 'onInput' | 'onFocus' | 'onBlur'
     format?: ValidatorFormats
     validator?: ValidatorFunction<Context>
     required?: boolean
     pattern?: RegExp | string
     max?: number
     maximum?: number
     exclusiveMaximum?: number
     exclusiveMinimum?: number
     minimum?: number
     min?: number
     len?: number
     whitespace?: boolean
     enum?: any[]
     message?: string
     [key: string]: any //其他属性需要通过registerValidateRules进行注册
   }
   //函数型校验器校验结果类型
   type ValidatorFunctionResponse = null | string | boolean | IValidateResult

   //函数型校验器
   type ValidatorFunction<Context = any> = (
     value: any,
     rule: IValidatorRules<Context>,
     ctx: Context
   ) => ValidatorFunctionResponse | Promise<ValidatorFunctionResponse> | null

   //非数组型校验器
   type ValidatorDescription =
     | ValidatorFormats
     | ValidatorFunction<Context>
     | IValidatorRules<Context>

   //数组型校验器
   type MultiValidator<Context = any> = ValidatorDescription<Context>[]

   type FieldValidator<Context = any> =
     | ValidatorDescription<Context>
     | MultiValidator<Context>
   ```



#### 三、 查看表单系统代码结构

前面的内容帮助我们了解了表单提供的功能和表单相关的类型声明，接下来就看看表单这一块是如何布局的core中的，首先我们关注到formily包中有一个validator，毫无疑问这个文件夹里面都是校验相关内容了，从src/index.ts的内容来看包中的内容包含4个部分：

```javascript
export * from './validator'
export * from './parser'
export * from './registry'
export * from './types' // 很明显是类型声明，就是上面我们提前看过的官网文档中给出的，不过包含更多内部的类型
```

在registry.ts中我们看到了一个registry对象：

```javascript
const registry = {
  locales: {
    messages: {},
    language: getBrowserlanguage(),
  },
  formats: {},
  rules: {},
  template: null,
}
```

这个对象就包含了我们所用到的所有规则和格式了，locales是支持多语言设置，template支持模板字符串来对message做一些延伸功能，registry就包含了较为核心的`registerValidateRules`和`registerValidateFormats`，看代码发现就是往上面的registry对象中增加传入的rule 和format，有向对象中增加值，也有从对象中取值，对应的就是`getValidateFormats`,`getValidateRules`方法了，还有对应的设置语言和设置模板引擎方法；

parser.ts中主要是兼容传入的多种格式的参数，以及传入的template模板，通过将传入的IValidatorRules对象转化为ValidatorParsedFunction校验函数以便后续调用;这里的方法应该是处理schema中传入的各类型参数时候调用的；

validator.ts 中主要是调用registry.ts中的方法注册内部的formats,rules以及设置默认语言，提供一个`validate`方法供外部使用。

####  四、在核心中如何接入表单系统

这下陷入了一个问题，如何查看core中怎么调用上面validator文件夹中的内容呢？想到了文档中介绍form实例对象时提出了form有一个验证表单的方法： [form.validate](https://core.formilyjs.org/zh-CN/api/models/form#validate),那么这个方法肯定是跟validator相关的！去core文件夹中找到这个方法！终于在formily/packages/core/src/models/Form.ts中找到了`validate`方法：

```javascript
validate = async (pattern: FormPathPattern = '*') => {
    this.setValidating(true)
    const tasks = []
    this.query(pattern).forEach((field) => {
      if (!isVoidField(field)) {
        tasks.push(field.validate()) // 并没有直接调用validator中的方法
      }
    })
    await Promise.all(tasks)
    this.setValidating(false)
    if (this.invalid) {
      this.notify(LifeCycleTypes.ON_FORM_VALIDATE_FAILED)
      throw this.errors
    }
    this.notify(LifeCycleTypes.ON_FORM_VALIDATE_SUCCESS)
  }
```

从`field.validate()`可以看出并没有直接调用validator中的方法，从而得知其实form实例的信息状态，应该是基于其中每个field的状态来，有直接拿过来的，也有基于所有的来生成一些form独有的状态，毕竟form实例是一个抽象的总体对象，field对象可以是对应页面中某一个表单项的，接着来找field的validate方法(formily/packages/core/src/models/Field.ts)：

```javascript
validate = async (triggerType?: ValidatorTriggerType) => {
    const start = () => {
      this.setValidating(true)
      this.form.notify(LifeCycleTypes.ON_FIELD_VALIDATE_START, this)
    }
    const end = () => {
      this.setValidating(false)
      if (this.valid) {
        this.form.notify(LifeCycleTypes.ON_FIELD_VALIDATE_SUCCESS, this)
      } else {
        this.form.notify(LifeCycleTypes.ON_FIELD_VALIDATE_FAILED, this)
      }
      this.form.notify(LifeCycleTypes.ON_FIELD_VALIDATE_END, this)
    }
    start()
    if (!triggerType) {
      // parseValidatorDescriptions就是前面validator/parser文件的方法
      const allTriggerTypes = parseValidatorDescriptions(this.validator).map(
        (desc) => desc.triggerType
      )
      const results = {}
      for (let i = 0; i < allTriggerTypes.length; i++) {
        const payload = await validateToFeedbacks(this, allTriggerTypes[i])
        each(payload, (result, key) => {
          results[key] = results[key] || []
          results[key] = results[key].concat(result)
        })
      }
      end()
      return results
    }
    const results = await validateToFeedbacks(this, triggerType)
    end()
    return results
  }
```

虽然代码中校验用的是`validateToFeedbacks`,这个方法查看一下就知道里面也是基于validator 中的`validate`方法去封装一层的，添加了field层面的shouldSkipValidate（某些场景需要校验）的逻辑。

至此，我们大概梳理一下formily/core中校验系统的应用，后面如果在代码中校验遇到了一些问题，可以按照这个思路去排查。



参考：

https://v2.formilyjs.org/