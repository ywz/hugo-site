---
title: "分布式程序监控方案"
date: 2020-10-11T20:06:56+08:00
tags: [分布式,监控,storm,akka,prometheus,grafana]
---
为了保证分布式系统的稳定性，可靠性，可运维性。掌控集群的核心性能指标，了解集群的性能表现，了解业务的进展情况，使用监控系统可以帮助我们更快的定位问题和解决问题。在尝试了很多方案后，我们最终选择了 Prometheus + Grafana 作为分布式系统（Akka 或 Storm）的监控方案，下面以 Akka 为例做介绍。

## 开源组件
- [Kamon](https://kamon.io/)  
一组用于监控运行在 JVM 上的应用的工具和框架，针对 Akka 做了特别支持，Akka 应用只需要做简单的整合就可以对 actor，router 和 dispatcher 进行监控 。Kamon 支持多种 reporter 收集和展示测量值。  
Kamon 的文档质量奇差 x_x，如果想深入了解 Kamon 及各个组件的用法，建议去 [Kamon 的 Github](https://github.com/kamon-io/Kamon) 挖。

- [Prometheus](https://prometheus.io/)  
Prometheus几乎已成为监控领域的事实标准，它自带高效的时序数据库存储，可以让单台Prometheus能够高效的处理大量的数据，可以用来灵活的查询各种监控数据以及配置告警规则，很多没有适配的应用也会有第三方exporter帮它去适配Prometheus，所以监控系统我们首选用Prometheus。

- [Grafana](https://grafana.com/)  
一个开箱即用的可视化工具，具有功能齐全的度量仪表盘和图形编辑器，有灵活丰富的图形化选项，可以混合多种风格，支持多个数据源。

## 流程
- Akka 程序集成 Kamon，程序启动后 Kamon 的 [kamon-prometheus](https://github.com/kamon-io/kamon-prometheus)  后端通过 http 协议发布监控数据，Prometheus 定期收集 kamon-prometheus 发布的最新数据，Grafana 负责结果的展示。  

![flow](/posts/images/2020-10-11_184457.png)

- Linux 服务器的各项指标由 [node_exporter](https://github.com/prometheus/node_exporter) 收集，大致流程同上。


## 截图
![grafana](/posts/images/2019-02-14_142746.png)
<!--more-->

## 环境要求
因为 Prometheus 和 Grafana 都由 Go 编写，所以对环境基本没有要求，本文用的是 Centos 7(10.4.43.140)。

__注意：要保证所有服务器时间同步。__

## 安装 Prometheus
下载地址: https://prometheus.io/download/
``` shell
cd /app
tar xvf prometheus-2.6.0.linux-amd64.tar.gz
mv prometheus-2.6.0.linux-amd64 prometheus
rm prometheus-2.6.0.linux-amd64.tar.gz
```

#### 添加 target
target 指发布指标数据的 http 服务的地址，添加新 target 后需要重启 Prometheus。

``` yml
vim prometheus.yml
--- 添加 ---
scrape_configs:
  - job_name: kamon-prometheus
    static_configs:
      - targets: ['localhost:9095']
```

#### 启动
下面是最简单的启动命令，可以利用 `systemd` 或者 [supervisord](http://supervisord.org/) 实现自动启动。下同。

``` shell
cd prometheus/
nohup ./prometheus --config.file=prometheus.yml &
```

#### 参考：测试用 prometheus.yml

``` yml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

  - job_name: kamon-test
    static_configs:
      - targets: ['10.4.57.33:9095']

  - job_name: akka_poc
    static_configs:
      - targets: ['10.4.43.140:9095','10.4.43.140:9096','10.4.43.140:9097','10.4.43.140:9098']

  - job_name: "node140"
    #scrape_interval: "15s"
    #target_groups:
    static_configs:
    - targets: ['10.4.43.140:9100']

  - job_name: "node141"
    static_configs:
    - targets: ['10.4.43.141:9100']

```

## 安装 node_exporter
node_exporter主要负责监控服务器的硬件指标。  
下载地址: https://prometheus.io/download/

``` shell
cd /app
tar xvf node_exporter-0.17.0.linux-amd64.tar.gz
mv node_exporter-0.17.0.linux-amd64 node_exporter
rm node_exporter-0.17.0.linux-amd64.tar.gz
```

#### 启动
``` shell
cd /node_exporter
nohup ./node_exporter &
```

#### target
``` yml
- job_name: "node141"
  static_configs:
  - targets: ['10.4.43.141:9100']
```

## 安装 Grafana
下载地址: https://grafana.com/grafana/download
``` shell
cd /app
tar xvf grafana-5.4.2.linux-amd64.tar.gz
mv grafana-5.4.2 grafana
rm grafana-5.4.2.linux-amd64.tar.gz
```

#### 启动
``` shell
cd grafana/
nohup ./grafana-server &
```

## 整合 Kamon
#### Maven 依赖
这里列的是全部，用不上的可以去掉。

``` xml
<!-- kamon -->
<dependency>
    <groupId>io.kamon</groupId>
    <artifactId>kamon-prometheus_2.12</artifactId>
    <version>1.1.1</version>
</dependency>
<dependency>
    <groupId>io.kamon</groupId>
    <artifactId>kamon-zipkin_2.12</artifactId>
    <version>1.0.0</version>
</dependency>
<dependency>
    <groupId>io.kamon</groupId>
    <artifactId>kamon-akka-remote-2.5_2.12</artifactId>
    <version>1.0.0</version>
</dependency>
<dependency>
    <groupId>io.kamon</groupId>
    <artifactId>kamon-statsd_2.12</artifactId>
    <version>1.0.0</version>
</dependency>
<dependency>
    <groupId>io.kamon</groupId>
    <artifactId>kamon-system-metrics_2.12</artifactId>
    <version>1.0.1</version>
</dependency>
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.9.2</version>
</dependency>
```

#### 配置

修改 kamon.conf，加入下面的内容，根据需求修改。

filter

``` json
kamon.util.filters {
  "akka.tracked-actor" {
    includes = ["**"]
  }

  "akka.tracked-dispatcher" {
    includes = ["**"]
  }

  "akka.tracked-router" {
    includes = ["**"]
  }

  "akka.traced-actor" {
    includes = ["**"]
  }
}
```

kamon-prometheus 的配置

``` json
# ======================================== #
# kamon-prometheus reference configuration #
# ======================================== #

kamon.prometheus {

  # Enable or disable publishing the Prometheus scraping enpoint using a embedded server.
  start-embedded-http-server = yes

  # Enable of disable including tags from kamon.prometheus.environment as labels
  include-environment-tags = no

  buckets {
    default-buckets = [
      10,
      30,
      100,
      300,
      1000,
      3000,
      10000,
      30000,
      100000
    ]

    time-buckets = [
      0.005,
      0.01,
      0.025,
      0.05,
      0.075,
      0.1,
      0.25,
      0.5,
      0.75,
      1,
      2.5,
      5,
      7.5,
      10
    ]

    information-buckets = [
      512,
      1024,
      2048,
      4096,
      16384,
      65536,
      524288,
      1048576
    ]

    # Per metric overrides are possible by specifying the metric name and the histogram buckets here
    custom {
      // example:
      // "akka.actor.processing-time" = [0.1, 1.0, 10.0]
    }
  }

  embedded-server {
    # Hostname and port used by the embedded web server to publish the scraping enpoint.
    hostname = 0.0.0.0
    port = 9095
  }
}
```

9095 是 prometheus 配置里监控的端口号。

#### excludes
如果 akka 程序中启动了上万的 actor，Prometheus 会非常卡。  
解决办法：将此 actor 的指标数据从 kamon 排除。

``` yml
vim kamon.conf
---
# tag:akka-filters:start
kamon.util.filters {

  "akka.tracked-actor" {
    includes = [ "my-app/user/job-manager", "my-app/user/worker-*" ]
    excludes = [ "my-app/system/**", "my-app/user/worker-helper" ]
  }

  "akka.tracked-dispatcher" {
    includes = [ "my-app/akka.actor.default-dispatcher", "my-app/database-dispatcher" ]
  }

  "akka.tracked-router" {
    includes = [ "my-app/user/some-router" ]
  }
}
# tag:akka-filters:end

# tag:akka-actor-groups:start
kamon {
  util.filters {
    "worker-actors" {
      includes = [ "my-app/user/worker-*" ]
    }
  }

  akka.actor-groups = [ "worker-actors" ]
}
# tag:akka-actor-groups:end

# tag:akka-message-tracing:start
kamon.util.filters {
  "akka.traced-actor" {
    includes = [ "my-app/user/job-manager", "my-app/user/worker-*" ]
    excludes = [ "my-app/system/**", "my-app/user/worker-helper" ]
  }
}
# tag:akka-message-tracing:end

kamon.internal-config {
  akka.loglevel = DEBUG
}

kamon.akka {
  ask-pattern-timeout-warning = heavyweight
}
```

#### 添加 reporter

在 Akka 应用的启动类里，加入下面的 reporter。

``` java
// kamon
Config kamonConfig = ConfigFactory.load("kamon.conf").withFallback(Kamon.config());
Kamon.reconfigure(kamonConfig);
Kamon.addReporter(new PrometheusReporter());
// Kamon.addReporter(new ZipkinReporter());
// Kamon.addReporter(new StatsDReporter());
```

#### 配置 AspectJ weaver

AspectJ weaver 的[下载地址](https://search.maven.org/search?q=a:aspectjweaver)。

在 Akka 应用的 JVM 的启动参数加入 javaagent。
``` text
-javaagent:path-to-aspectjweaver.jar
```

## 确认
正常启动 Akka 程序，Prometheus，node_exporter，Grafana 后。

#### kamon-prometheus
用浏览器访问 kamon-prometheus 的 embedded-server (kamon.conf 里配置)，程序启动 60s 后，可以看到指标数据。

#### Prometheus
确认 targets 状态: http://10.4.43.140:9090/targets

接下来可以用 Prometheus 的搜索功能确认是否已经收到了数据。http://10.4.43.140:9090/graph

## Grafana 使用
#### 访问
地址: http://10.4.43.140:3000

初始用户名密码

``` text
[security]
admin_user = admin
admin_password = admin
```

#### 添加 data source
按照向导，添加刚才启动的 Prometheus。

#### 添加 dashboard
文档附带的 json 可以直接导入为 dashboard。

- kamon: kamon_akka_monitor.json
- 服务器监控: node_exporter.json
- JVM 监控: bbj_jvm_overview.json


## 自定义指标
Kamon 提供了自定义指标的接口，可以把我们自己需要监控的指标融合到上面的监控流程。

例如，统计在线终端数，可以用下面的代码在终端 actor 加入对在线终端数的实时统计。

``` java
private final GaugeMetric gaugeMetric = Kamon.gauge("term_online_gm");

gaugeMetric.refine("actor_name", self().path().name()).set(#current_term_count#); // 添加到合适的位置
```

下面是启动 10 个终端 actor Prometheus 收集到的数据：

``` json
term_online_gm{actor_name="$e"} 100.0
term_online_gm{actor_name="$j"} 100.0
term_online_gm{actor_name="$a"} 100.0
term_online_gm{actor_name="$h"} 100.0
term_online_gm{actor_name="$c"} 100.0
term_online_gm{actor_name="$f"} 100.0
term_online_gm{actor_name="$i"} 100.0
term_online_gm{actor_name="$d"} 100.0
term_online_gm{actor_name="$b"} 100.0
term_online_gm{actor_name="$g"} 100.0
```

每个终端 actor 中的在线终端数为 100，在 Grafana 求合得到的在线终端数为 1000。

``` json
"expr": "sum(term_online_gm)"
```

其它可以用的接口

``` java
final Histogram myHistogram = Kamon.histogram("my.histogram");
final Counter myCounter = Kamon.counter("my.counter");

myHistogram.record(42);
myHistogram.record(50);
myCounter.increment();
```

## JVM 监控
jmx_exporter 的[下载地址](https://github.com/prometheus/jmx_exporter)

在 Java 应用的 JVM 的启动参数加入 javaagent。
``` shell
java -javaagent:./jmx_prometheus_javaagent-0.11.0.jar=8080:config.yaml -jar yourJar.jar
```

## Storm 监控
Storm 的分布式形态与 Akka 不同，Bolt 的位置不固定，无法在`prometheus.yml`配置，所以要借助 [pushgateway](https://github.com/prometheus/pushgateway) 服务做一下中转，并且把 Akka 方案中的 Kamon 替换为 pushgateway 的 [client_java](https://github.com/prometheus/client_java/tree/master/simpleclient_pushgateway)，其它组件安装和使1使用可以参考 Akka 方案。  

![storm](/posts/images/2020-10-11_185833.png)

## 附
- [Grafana Getting started](http://docs.grafana.org/guides/getting_started/)
- [Querying Prometheus](https://prometheus.io/docs/prometheus/latest/querying/basics/)

