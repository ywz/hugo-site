---
title: "PostgreSQL 里操作 JSON"
date: 2020-09-24T22:04:56+08:00
tags: [json,sql,postgresql]
---

### PostgreSQL 的版本

PostgreSQL 9.2 开始支持 JSON 类型，但函数不全，不建议使用，9.5 版本加入了 jsonb\_set 函数，9.6 版本加入了 jsonb\_insert 函数，变更 json 内容变得非常容易，所以建议使用 9.5 以后的版本。

![PostgreSQL](/posts/images/postgresql-logo-1.jpg)

<!--more-->
### json 和 jsonb
> 有两种 JSON 数据类型：json 和 jsonb。它们**几乎**接受完全相同的值集合作为输入。主要的实际区别之一是效率。json 数据类型存储输入文本的精准拷贝，处理函数必须在每次执行时必须重新解析该数据。而 jsonb 数据被存储在一种分解好的二进制格式中，它在输入时要稍慢一些，因为需要做附加的转换。但是 jsonb 在处理时要快很多，因为不需要解析。jsonb 也支持索引，这也是一个令人瞩目的优势。

### 创建表
```sql
create table fu_mp_param
(
	fu_mp_id numeric(16) not null
		constraint fu_mp_param_pkey
			primary key,
	fu_mp_value jsonb not null
)
;

-- 创建 JSONB 的索引
create index idxgin
	on fu_mp_param (fu_mp_value)
;
```

### 插入
```sql
insert into fu_mp_param values(0, '{"key1":"value1"}');
```
Java 实现，两个关键点：
1. 连接参数中必须包含【stringtype=unspecified】
2. 用 PGobject 类型设置 json 或者 jsonb 的值
```java
String url = "jdbc:postgresql://10.4.57.223/postgres?user=postgres&password=neusoft&ssl=false&stringtype=unspecified";
Connection conn = DriverManager.getConnection(URL);

// ---
String sql = "insert into fu_mp_param values(?, ?);";
st = conn.prepareStatement(sql);

st.setInt(1, i);

JSONObject obj = new JSONObject();
PGobject pgObject = new PGobject();
pgObject.setType("json");
pgObject.setValue(obj.toString());
st.setObject(2, pgObject);

st.executeUpdate();
```
### 更新
主要用到 [jsonb\_set](https://www.postgresql.org/docs/9.6/static/functions-json.html) 函数
```sql
update fu_mp_param
set fu_mp_value = jsonb_set(fu_mp_value, '{item}', '9999', false)
where fu_mp_id = 328323;
```
```java
String sql = "update fu_mp_param set fu_mp_value = jsonb_set(fu_mp_value, '{item}', ?, false) where fu_mp_id = ?;";
st = conn.prepareStatement(sql);
st.setString(1, "54321");
st.setInt(2, 328323);
st.executeUpdate();
```

### 检索
```sql
-- 检索 jsonb 中包含 {"set_value":10000} 的数据
select
  fu_mp_id,
  fu_mp_value->'get_time' --提取 [json 中 get_time] 的值 
from fu_mp_param
where fu_mp_value @> '{"set_value":10000}';
```

### 插件
正常通过内部函数取得的结果都是 json(b) 格式，可以用 [postsql](https://github.com/tobyhede/postsql) 对输出结果进行格式化。postsql 依赖 plv8 插件，默认只支持 json 类型，可以自己修改一套给 jsonb 用。下面是一个简单的例子，其它函数的用法参照文档。
```sql
select jsonb_int(fu_mp_value, 'set_value')
from fu_mp_param
where fu_mp_id = 328323;
```
postsql 函数因为用 js 的 v8 引擎实现，效率并不高，所以**不建议**用在 where 条件里。

### 容量
理论上 json(b) 能容纳 1GB 的数据量。（[参考](https://stackoverflow.com/questions/12632871/size-limit-of-json-data-type-in-postgresql)）

### 索引
测试环境：8 核 16G，机械硬盘，数据量一千万。
SQL：
```sql
select *
from fu_mp_param
where fu_mp_value ->> 'fu_mp_id' = '1000';
```
创建索引前：
```plain
starting from 1 in 2 s 719 ms (execution: 2 s 657 ms, fetching: 62 ms)
```
创建索引
```sql
create index bidbtree on fu_mp_param using btree ((fu_mp_value->>'fu_mp_id'));
```
再次执行
```plain
starting from 1 in 31 ms (execution: 0 ms, fetching: 31 ms)
```
[更多关于索引的用法](http://francs3.blog.163.com/blog/static/40576727201452293027868/)

### 其它
上面的例子都是对 json(b) 的简单操作，PostgreSQL 同样支持 json(b) 中的数组，参考：  
[JSON 类型文档](http://postgres.cn/docs/9.6/datatype-json.html)  
[JSON 函数和操作符](http://postgres.cn/docs/9.6/functions-json.html)

oracle 从 12.1.0.2.0 版开始也加入了对 json 类型的支持，说明：  
[JSON Support in Oracle Database 12c Release 1 (12.1.0.2)](https://oracle-base.com/articles/12c/json-support-in-oracle-database-12cr1)  
[Indexing JSON Data in Oracle Database 12c Release 1 (12.1.0.2)](https://oracle-base.com/articles/12c/indexing-json-data-in-oracle-database-12cr1)
