---
title: "Golang避免使用struct tag"
slug: "golang-reduce-json-db-tag"
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
categories:
  - 技术
series:
  -

image: "https://tva1.sinaimg.cn/large/006tNbRwgy1g9m3dglipwj30dw092acv.jpg"

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

最近在用Golang搭建博客后端时, 遇到一个问题: 数据在从mysql到接口输出的json转化中, 需要做字段映射. 常规的方法是写db tag和json tag, 自动进行转换. 这种方式有点麻烦, 尤其struct和字段多起来的时候. 我的场景是mysql字段为snake_case, struct为CamelCase, json字段为snake_case, 就想着用代码来完成转换



代码示例如下

```go
// 博客文章struct
type Post struct {
    Content   string
    Cover     string
    CreatedAt time.Time
    FamilyID  int
    ID        int
    Slug      string
    Status    int
    Title     string
    UpdatedAt time.Time
}


// struct和json互转
var keyMatchRegex = regexp.MustCompile(`"\w+":`)
var wordBarrierRegex = regexp.MustCompile(`([a-z])([A-Z])`)
// 重写Response类型的MarshalJSON方法
func (resp Response) MarshalJSON() ([]byte, error) {
    // 创建此类型, 防止递归
    type Response_ Response
    marshalled, err := json.Marshal(Response_(resp))
    // 正则替换
    converted := keyMatchRegex.ReplaceAllFunc(
        marshalled,
        func(match []byte) []byte {
            return bytes.ToLower(wordBarrierRegex.ReplaceAll(
                match,
                []byte(`${1}_${2}`),
            ))
        },
    )
    return converted, err
}


// struct和db字段互转
var DB *sqlx.DB

func init(){
    // 使用sqlx包与数据库交互
    DB = sqlx.MustConnect("mysql", "db connection")
    // 支持自定义MapperFunc, 使用strcase包中的ToSnake
    DB.MapperFunc(strcase.ToSnake)
    DB = DB.Unsafe()
}
```

