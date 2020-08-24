---
title: go-admin 一个 Golang 写的管理后台框架
category: 技术
tags: ["Golang", "go-admin"]
summary: go-admin 一个 Golang 写的管理后台框架
reward: true
---
近期需要做一个管理后台，这边的项目基本都是用`golang`来写的，如果是要自己做模板权限管理这些重头开始写，有点不现实，所以就搜索了一下看看有没有第三方实现的后台管理框架。

搜索一来，发现，只有两个，一个是[qor/admin](https://github.com/qor/admin)，另外一个是[GoAdminGroup/go-admin](https://github.com/GoAdminGroup/go-admin)。号称都支持多个`golang`的流行框架。`qor/admin`这个点看起来已经是比较长的时间没有更新了，`GoAdminGroup/go-admin`在我搜索到的时候，还没有发布正式版。

按照两个项目的官方示例尝试了一下，`qor/admin` 的`gin`示例居然运行不起来，报错了。[`GoAdminGroup/go-admin`](https://github.com/GoAdminGroup/example)就完美运行了，并且比`qor/admin`好用得多了（虽然还没发布正式版）。

套用一下官方的介绍及特性：
```text
`GoAdmin` 是一个基于`golang`的数据可视化管理平台搭建框架，*提供了一整套视觉ui给`golang`程序调用*，并内置了一个sql关系数据库管理后台插件。 

以前我们写一个管理后台，我们需要至少一个后台工程师，一个前端工程师，花费至少一周时间才能搭建完成，搭建完成后我们需要部署前端代码和后端代码，搭建环境就让我们头疼。 而现在有了`GoAdmin`，我们可以不需要前端工程师，我们的后端工程师甚至也不需要太懂得前端的知识，即可在半个小时内构建起一个完善的后台管理系统，但需要先花一点点时间了解掌握`GoAdmin`。 如果你的需求不复杂，只是简单的增删改查，那么你所需要的只是一些golang文件，就可以运行起一个后台程序。而所有文件，包括前端代码都将打包成一个二进制文件，非常利于分发与部署。

有同学就说虽然不需要前端，可是现在`Adminlte`的主题有些陈旧不够好看。很兴奋告诉大家，更多的超好看的主题正在路上，完全能够满足大部分个性化的需求，覆盖不同的设计风格，只要市面上能看到的好看的管理后台界面，我们都将逐步制作成为主题，只需要在配置中一个设置即可拥有。

总而言之，可以理解`GoAdmin`是基于`golang`的`grafana`，一个为生产效率而生的框架。
```
```text
* 内置完善的rbac权限系统
* 支持多个web框架接入
* 本地化支持
* 整个系统可以编译成一个二进制文件
* 提供多个插件（开发中）
* 多个好看的ui主题（目前支持Adminlte，更多主题开发中）
```

`GoAdminGroup/go-admin`支持`beego`、`chi`、`echo`、`gin`、`fasthttp`等待这些框架。`ORM`支持的是`gorm`。

尝试了下来，决定用`GoAdminGroup/go-admin`处理后台这一块，基本上，用自带的`admincli`生成一下对应的数库表处理文件，然后根据自己的需求修改一下就可以直接使用，真的是很好用。

现在`GoAdminGroup/go-admin`才发布`v1.0.0`，功能其实就对应着后台的增删改查，还有一些统计功能。未来看看可以不可以加入到`GoAdminGroup/go-admin`的开发中吧。