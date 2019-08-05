---
title: 使用连续查询及保留策略优化InfluxDB存储空间及内存占用
tags: ["InfluxDB", "时序数据库", "InfluxDB存储优化", "InfluxDB内存优化", "Continuous Queries", "Retention Policies", "Docker"]
categories: ["技术", "数据库"]
summary: 在 `telegraf` 及 `geostat` 启动的时候，初始化连续查询及保留策略优化InfluxDB存储空间及内存占用（InfluxDB需要6.0或以上版本）
reward: true
---

## 前言

现在在开发的[docker-map3/map3-edge](https://github.com/hyperion-hyn/docker-map3/tree/master/map3-edge)志在做一个去中心化的 `map` 支持服务，理想状态下是用最低的资源，就可以启动一个 `map3` 节点，里面包括有反向代理 、缓存、访问日志收集、该节点的系统状态及访客统计信息收集，Docker 镜像的自动更新以及该节点的数据展示。集成了这么多东西，性能优化是很关键的一环。

## 现在在使用的功能

* <a href="https://github.com/hyperion-hyn/caddy" target="_blank">hyperion-hyn/caddy</a> (fork from <a href="https://github.com/caddyserver/caddy" target="_blank">caddyserver/caddy</a>：用于反向代理及做地图的 tile 文件缓存，未来还会加入 `UPNP` 、 `P2P` 及 `keyless` 功能；
* filebeat：用于访问日志上报到 Elasticsearch；
* telegraf：用于收集节点的 CPU 及内存使用情况，将数据记录到 InfluxDB；
* <a href="https://github.com/hyperion-hyn/geostat" target="_blank">hyperion-hyn/geostat</a>：处理访客信息；
* <a href="https://github.com/hyperion-hyn/ouroboros" target="_blank">hyperion-hyn/ouroboros</a>(fork from <a href="https://github.com/pyouroboros/ouroboros" target="_blank">pyouroboros/ouroboros</a>：自动更新在运行的 Docker container 的镜像到最新版；
* InfluxDB：记录节点系统监控信息及该节点访客的统计信息，用于在 Grafana中显示；
* Grafana：显示系统监控信息及该节点访客统计信息。

## InfluxDB 没优化前导致的问题

前文有说过，我们理想的状态下是以最低的配置就可以起一个节点，以 Amazon AWS 免费的 t2.micro 实例类型为例，配置为1 VCPU, 1 GB 内存以及8 GB 存储。要启动这么多功能是一个不小的考验，特别是需要运行一个数据。

在优化前，如果统计的数据量很小（或者是用户选择展示的时间区间内数据比较小），问题并不大。但我们在测试极端情况时，用 AB Test 去刷数据，并且在 Grafana 里面查看。登录到主机里面查看，可以看到 InfluxDB 占用到了 30% 的内存（1GB的内存配置）, 总内存占用会占到70%多，有时候还会到90%多。刷的数据很多的时候，存储也会有问题，毕竟才8 GB 的存储空间。

![没优化前的内存占用](https://bingozai.s3.ap-east-1.amazonaws.com/blog/2019/08/05/influxdb.png)

## 如何优化

### 查找问题
优化存储的话，基本是基于 InfluxDB 的 `Retention Policies` 做处理就可以，我们默认是保留30天的数据。但这个其实数据也挺多的了，还可以再优化。

优化内存占用，首页要确定的是在写 InfluxDB 数据的时候就会占用很多内存还是说在查看的时候数据量比较大占用的内存比较多。

写入 InfluxDB 有两种请求方式
* http
* UDP

我在测试的时候，无论是用http还是UDP，用 `BatchPoints` 的方式快速写入 50K 条数据，内存基本没什么波动。但如果是在 Grafana 后台去展示数据，数据量越多，占用的内存就越高。所以可以确定的是内存占用高是查询太多数据造成的。

### 优化处理资料
查了很多资料，都是说用 `Continuous Queries` + `Retention Policies` 可以优化，但都没有详细的说明。后来我们团队的大佬 [邹光先](https://github.com/zouguangxian)先生 找到了 [Server stats with collectd, InfluxDB and Grafana (with downsampling)](https://pommi.nethuis.nl/collectd-influxdb-grafana-with-downsampling/) 这个有关 InfluxDB 数据下采样处理的文章，根据这个文章提供的方法结合我们的实际情况处理。

### 优化处理
[Server stats with collectd, InfluxDB and Grafana (with downsampling)](https://pommi.nethuis.nl/collectd-influxdb-grafana-with-downsampling/) 的基本策略就是用 `Continuous Queries` + `Retention Policies`。
* `Retention Policies` 保留策略我们是使用以下几种：
    * autogen：`autogen` 默认是永久存储的，我们将它的保留策略改为 `200m`（为什么是200分钟，因为在 Grafana Dashboard 里面如果是选择 Last 3 hours，应用的保留策略是 `autogen` 的，如果是选择 Last 6 hours，应用的保留策略就会变成 `day`）
    * day: 
    * week
    * month
    * year

* `Continuous Queries` 我们是生成了以下几条：
```sql
CREATE CONTINUOUS QUERY \"cq_day\" ON \"$database\" BEGIN SELECT sum(count) as count INTO \"$database\".\"day\".:MEASUREMENT FROM /.*/ GROUP BY time(60s),* END;
CREATE CONTINUOUS QUERY \"cq_week\" ON \"$database\" BEGIN SELECT sum(count) as count INTO \"$database\".\"week\".:MEASUREMENT FROM /.*/ GROUP BY time(300s),* END;
CREATE CONTINUOUS QUERY \"cq_month\" ON \"$database\" BEGIN SELECT sum(count) as count INTO \"$database\".\"month\".:MEASUREMENT FROM /.*/ GROUP BY time(1800s),* END;
CREATE CONTINUOUS QUERY \"cq_year\" ON \"$database\" BEGIN SELECT sum(count) as count INTO \"$database\".\"year\".:MEASUREMENT FROM /.*/ GROUP BY time(21600s),* END;
```

`https://github.com/hyperion-hyn/docker-map3/blob/keyless/map3-edge/docker/geostat/rootfs/optimize-influxdb.sh` 的示例代码，`telegraf` 的也是差不多的
```bash
#!/bin/sh
# refer: https://pommi.nethuis.nl/collectd-influxdb-grafana-with-downsampling/

options=`cat /etc/geostat/geostat.json | grep -E '(port|host|database)' | sed -e 's/"[ ]*:[ ]*"/":"/g' -e 's/^[ ]*//' -e 's/,[ ]*$//'`
database=`printf "$options" | sed -n -e '/database/p' | sed -e 's/^[^:]*://' -e 's/"//g'`
host=`printf "$options" | sed -n -e '/host/p' | sed -e 's/^[^:]*://' -e 's/"//g'`
port=`printf "$options" | sed -n -e '/port/p' | sed -e 's/^[^:]*://' -e 's/"//g'`

if [[ ! -z "$host" -a ! -z "$port" -a ! -z "$database" ]]; then
url="http://$host:$port"

curl --retry 10 --retry-connrefused --retry-delay 1 "$url/ping"

curl -i -XPOST "$url/query" --data-urlencode "q=
CREATE DATABASE $database;
ALTER RETENTION POLICY \"autogen\" on $database DURATION 200m SHARD DURATION 1h;
CREATE RETENTION POLICY \"day\" ON $database DURATION 1d REPLICATION 1;
CREATE RETENTION POLICY \"week\" ON $database DURATION 7d REPLICATION 1;
CREATE RETENTION POLICY \"month\" ON $database DURATION 31d REPLICATION 1;
CREATE RETENTION POLICY \"year\" ON $database DURATION 366d REPLICATION 1;
CREATE CONTINUOUS QUERY \"cq_day\" ON \"$database\" BEGIN SELECT sum(count) as count INTO \"$database\".\"day\".:MEASUREMENT FROM /.*/ GROUP BY time(60s),* END;
CREATE CONTINUOUS QUERY \"cq_week\" ON \"$database\" BEGIN SELECT sum(count) as count INTO \"$database\".\"week\".:MEASUREMENT FROM /.*/ GROUP BY time(300s),* END;
CREATE CONTINUOUS QUERY \"cq_month\" ON \"$database\" BEGIN SELECT sum(count) as count INTO \"$database\".\"month\".:MEASUREMENT FROM /.*/ GROUP BY time(1800s),* END;
CREATE CONTINUOUS QUERY \"cq_year\" ON \"$database\" BEGIN SELECT sum(count) as count INTO \"$database\".\"year\".:MEASUREMENT FROM /.*/ GROUP BY time(21600s),* END;
CREATE RETENTION POLICY \"forever\" ON \"$database\" DURATION INF REPLICATION 1;
"

curl -i -XPOST "$url/write?db=$database&rp=forever" --data-binary "
rp_config,idx=1 rp=\"autogen\",start=0i,end=12000000i,interval=\"10s\" -9223372036854775806
rp_config,idx=2 rp=\"day\",start=12000000i,end=86401000i,interval=\"60s\" -9223372036854775806
rp_config,idx=3 rp=\"week\",start=86401000i,end=604801000i,interval=\"300s\" -9223372036854775806
rp_config,idx=4 rp=\"month\",start=604801000i,end=2678401000i,interval=\"1800s\" -9223372036854775806
rp_config,idx=5 rp=\"year\",start=2678401000i,end=31622401000i,interval=\"21600s\" -9223372036854775806
"
fi

exit 0
```


**特别提醒，优化后的处理，需要的 Grafana 版本在6.x或以上。 Grafana Dashboard 的配置，参考[https://github.com/hyperion-hyn/docker-map3/blob/master/map3-edge/docker/grafana/rootfs/dashboards/default.json](https://github.com/hyperion-hyn/docker-map3/blob/master/map3-edge/docker/grafana/rootfs/dashboards/default.json)**
## 优化后表现

经过优化后，InfluxDB 的内存占用，很好地控制在 10% 以下，在Grafana 后台可以看到 `map3-edge` 的总内存占用（以1 GB 配置为例），维持在 50% 以下。

![优化后的内存占用](https://bingozai.s3.ap-east-1.amazonaws.com/blog/2019/08/05/optimize-influxdb.png)

## 遗留的问题
虽然用 `Continuous Queries` + `Retention Policies` 做了处理，但其实大家都可以看出来如果是在短时间内有大量访问后，用 Last 3 hours （或者是更小的时间）查看，这个等于是没做优化的，内存占用还是会很多，这个暂时无解。

还有就是 `Continuous Queries` 的 GEO 信息处理，是会根据 city 做 group by，所以如果是访问的地区比较分散的话，要生成的数据其实也是比较多。这种情况下，优化得并不是很好。好在我们希望的 `map3-edge` 是在每个城市都有部署，这样根据 city 做 group by 这种在以后节点越来越多后，就不再是一个问题。

## 参考

* [Server stats with collectd, InfluxDB and Grafana (with downsampling)](https://pommi.nethuis.nl/collectd-influxdb-grafana-with-downsampling/) 
* [Downsampling and data retention](https://docs.influxdata.com/influxdb/v1.7/guides/downsampling_and_retention/)