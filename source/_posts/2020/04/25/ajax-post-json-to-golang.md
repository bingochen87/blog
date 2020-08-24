---
title: 用 Ajax Post Json 数据到 Gin
category: 技术
tags: ["Golang", "Gin", "Ajax", "Json"]
summary: 使用 Ajax post json 数据到 Gin，并用 `ctx.ShouldBindJSON`（或者是其他的类似操作） 绑定到 struct 里面。
reward: true
---
### 前言
一直都有用 `go-admin` 作为 `golang` 的后台处理，但升级到稳定版的 `v1.1.x` 后，一直没更新过。近日是在做一个功能，用 `go-admin` 本身的 `CRUD` 并不能满足我想要的，所以只好是自定义页面来处理这个。

但用自定义的页面，就会涉及到一个数据提交时的鉴权、返回json数据的问题，在 `v1.1.x` 里面并没有外部可以调用的 `go-admin` 的鉴权方法，也没有是可以返回 `json` 响应的处理。

查看了最新版的示例，发现这两个需求都已经是已经是有了。所以，将 `go-admin` 升级到最新版（v1.2.9）。

### 处理
升级完成后，需要做一下兼容性的更改，对于我现在在使用的代码，需要更改
```text
1. 现在的 `table` 部分，需要强制使用官方的 `context.Context`
2. 旧版的 `FieldOptions` 接收的是一个 `[]map[string]string`，现在改为内部定义的 `types.FieldOptions` （相应地，会有 `types.FieldOption`），调用变得更简单了
```

处理完兼容性问题后，使用以下代码请求
```javascript
$.ajax({
    type: "post",
    url: '/admin/text/ajax',
    // 1 需要使用JSON.stringify 否则格式为 a=2&b=3&now=14...
    // 2 需要强制类型转换，否则格式为 {"a":"2","b":"3"}
    data: JSON.stringify({
        a: parseInt($('input[name="a"]').val()),
        b: parseInt($('input[name="b"]').val())
    }),
    dataType: "json",
    success: function(data) {
        $('#result').text(data.result);
    }
});
```

理论上，这样的代码是没问题的，但发请求过去后，`go-admin` 的 `context.BindJSON` 并没有正常工作，debug 出来的结果是 form body 里面没有信息。但在 Chrome 里面的网络里面是看到请求已经是正常发送
