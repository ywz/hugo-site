---
title: "亿级数据量分布式入库实践（Oracle）"
date: 2020-10-11T19:36:56+08:00
tags: [分布式,storm,akka,oracle,性能]
---

## 需求描述
某电网公司某网省采集系统，每天早上上班前需要把前一天冻结的电表数据入库完成，入库效率是重要的考核指标。

### 数据量
- 电表数大约 3000 万，需要入库的表有十几个，入库的数据量大约在 2 亿左右。
- 除此之外，每小时也有三千万左右的数据需要入库。

### 时间限制
任务高峰在凌晨 1 时 - 5 时（4 小时内）

## 架构
### 数据架构图
![数据架构图](/posts/images/base.png)
- 数据采集层：收集各种规约的报文，发送到 Kafka
- 数据入库层：接收 Kafka 的报文，解析后入 Oracle，入 Oracle 成功后，入 HBase。由于历史原因，程序用 [Apache Storm](https://storm.apache.org/) 编写。
- 数据层：会被频繁访问的档案类数据和近 3 个月的量测数据会放在 Oracle，历史数据保存在 HBase。
- 数据应用层：基于 HBase 中的数据开展电量计算、漏抄估算、数据挖掘和机器学习等业务
<!--more-->
### 拓扑图
![拓扑图](/posts/images/topo.png)
- Spout：接收收 Kafka 的报文，过滤非法报文。
- 解析 Bolt：批量解析报文并校验解析结果，过程中会从 Redis 获取档案信息。
- 入库 Bolt：批量入库。

## 性能瓶颈
程序第一版完成用了很短的时间，经过漫长的调优过程，总结了如下几个对性能影响较大的问题点。

### 解析
程序接收到的报文为 JSON 格式，报文种类有 100 多种，接收到报文以后要判断报文种类，并针对报文种类进行解析，如果有每条报文解析需要 1 毫秒，2 亿条报文解析需要 55.6 小时，离期待的速度相差很远，所以解析速度最好要达到每毫秒 20 条以上。 
``` text
200000000/1000/60/60 = 55.6 小时
ms        s    m  h
```

#### 应对方案  
报文解析没有用代码解析，而是做成配置（存到 Oracle），用到了开源的 [json-path](https://github.com/json-path/JsonPath) 工具，每种报文有一条对应的配置，直接通过 JSON 路径可以获取报文里的值，这样做的好处。

1. 应用更加灵活，支持新报文只需要加一条配置
2. 程序可复用性高

测试时发现 json-path 效率非常低，并且在业务高峰时会发生 CPU 占用率高，内存溢出等问题。最终自己实现了一个简版的 json-path，功能比开源版本要少，但效率至少提升了 10 倍。

同时因为使用了分布式程序，解析处理可以分摊给 N 台机器，利用并行处理来提高解析效率。

### 阻塞
初版程序使用了 Storm 的 Ack 机制，如果有入库行为发生失败(fail)，会触发重试，导致这以后的数据都无法被处理。

#### 应对方案  
1. 关闭 Storm 的 Ack 功能，表之间的入库不会相互影响。
2. 入库批处理增加超时时间，提交超时程序可以继续执行。
``` java
package java.sql;
public interface Statement extends Wrapper, AutoCloseable {
    /**
     * Sets the number of seconds the driver will wait for a
     * <code>Statement</code> object to execute to the given number of seconds.
     *By default there is no limit on the amount of time allowed for a running
     * statement to complete. If the limit is exceeded, an
     * <code>SQLTimeoutException</code> is thrown.
     * A JDBC driver must apply this limit to the <code>execute</code>,
     * <code>executeQuery</code> and <code>executeUpdate</code> methods.
     * <p>
     * <strong>Note:</strong> JDBC driver implementations may also apply this
     * limit to {@code ResultSet} methods
     * (consult your driver vendor documentation for details).
     * <p>
     * <strong>Note:</strong> In the case of {@code Statement} batching, it is
     * implementation defined as to whether the time-out is applied to
     * individual SQL commands added via the {@code addBatch} method or to
     * the entire batch of SQL commands invoked by the {@code executeBatch}
     * method (consult your driver vendor documentation for details).
     *
     * @param seconds the new query timeout limit in seconds; zero means
     *        there is no limit
     * @exception SQLException if a database access error occurs,
     * this method is called on a closed <code>Statement</code>
     *            or the condition {@code seconds >= 0} is not satisfied
     * @see #getQueryTimeout
     */ 
    void setQueryTimeout(int seconds) throws SQLException;
}
```

### 锁表
入库报文的种类有上百种之多，初版程序针对每一种报文类型，都有自己的入库 SQL 文，所以进程间对同一条数据的入库操作容易触发行锁。

#### 应对方案
1. 改进 Storm 的分组策略自己编写自定义分组（[CustomStreamGrouping](https://storm.apache.org/releases/2.1.0/javadocs/org/apache/storm/grouping/CustomStreamGrouping.html)），保证同一张表的同一条数据的所有入库 SQL 在一个线程进行。
2. 改进配置结构，针对同一个表的所有入库操作都尽量使用同一个 SQL 提交。（如下图）  
![锁表](/posts/images/sql.png)

### 批量入库失败
为了提高效率，Oracle 入库时采用批量提交，所以同一批次如果有一条失败，会导致该批次全部数据入库失败。

#### 应对方案
![重试](/posts/images/retry.png)  
批量提交失败后，会触发逐条重试，重试入库再次失败的数据会记录到【失败记录表】。

### Redis
每条报文对应的档案信息需要从 Redis 获取，每次 Redis io 需要零点几毫秒到几毫秒不等，所以每收到一条报文访问一次 Redis 显然行不通。

#### 应对方案
解析 Bolt 接收到报文不会马上解析，满足下面两个条件才会开始批量解析，集中利用资源减少 IO 次数可以大幅提升解析效率。
1. 满足约定数量会触发批量处理。
2. 如果长时间没有接收到新报文，无法满足约定数量，到达约定时间后也会触发批量处理。

### 序列化
由于 Storm 的 Spout 和 Bolt、Bolt 和 Bolt 间的消息传输需要对消息做序列化，初版程序消息体积比较大，会有资源占用高的问题。

#### 应对方案
减小序列化对象的体积。

### Oracle
针对 Oracle 的优化如下。
1. 入库 Bolt 采用批量提交的策略。
1. 减少更新操作，能用插入用插入。
2. 合理选择连接池，推荐 [HikariCP](https://github.com/brettwooldridge/HikariCP)。
2. 根据表的数据量设置每天或者每月自动分区。
3. 对于数据量特别大的表，直接插入或更新操作性能会严重不足，对应方案为建一张没有索引的临时表，Storm 入库入临时表（没有索引，数据量小，所以提交速度非常快），定时用存储过程将数据搬迁到正式表，经过验证，可以满足需求。


## 监控

但由于历史原因，我们用的是 0.9 版本的 Storm，Storm UI 还没有 Metirc 功能，最终选择了 [Prometheus](https://prometheus.io/) + [pushgateway](https://github.com/prometheus/pushgateway) + [Grafana](https://grafana.com/) 做成了监控功能。

效果图如下(测试系统的截图)，现场运维可以通过监控画面监控入库状态、排除问题，优化程序。  

![grafana](/posts/images/h2o_grafana.png)

