---
title: Kibana Sample Logs Data Dashboard
category: 技术
tags: ["Kibana", "ES", "Elasticsearch"]
summary: Add Kibana sample logs data dashboard
reward: true
---

因为`map3-node-edge`从原来的`map3-edge-dashboard`移到`docker-map3/map3-edge`并开源了。过程中做了优化处理，去掉了几个`docker container`，其中包括`metricbeat`和`heartbeat`。在处理总的`Kibana dashboard`过程中，不小心将几个索引数据删除掉了，导致`[Logs] Web Traffic`这个Dashboard用不了。

我首选想到的办法，居然是傻傻地照着原来的设置一个个地加回来，并且因为现在只剩下`filebeat`这个数据源，我天真地以为`kibana_sample_logs_data`这个索引数据是从其他两个已经是删除的beat来的，有些字段还对不上。

折腾一番后，查了一下`kibana_sample_logs_data`发现这个数据是可以手动添加了。操作后，最终发现整个`[Logs] Web Traffic`这个Dashboard，就是点几下就可以出来，根本不需要自己去配置。哎，业务不精，真的是害死人。

添加方法:

* 去到kibana首页（我使用的是KibanaCloud,首页的url为：app/kibana#/home）
* 在`Add sample data`这里，点击下方的链接（`Load a data set and a Kibana dashboard`）
* 跳转后，在列表里面的`Sample web logs`点击按钮
* 等待处理完成，就可以在`Dashboard`这一个链接页面里面看到有`[Logs] Web Traffic`这个Dashboard