---
title: 如何从Hive中获取最大分区数据
date: 2019-12-28 12:41:47
tags:
	- Apache Spark
	- Hive
categories: 数据库
description: 本文介绍了获取Hive最新分区 (partition) 数据的方法和优化小技巧，以及背后的原理。
cover: /images/apache_hive_icon.png
---

# 案例分析
最近在工作中遇到了这样一个场景：我们有一个Hive table，分区 (partition) 按照日期划分。ETL pipeline需要在执行过程中选择日期最近的分区数据。在Spark环境下，最直接的做法是这样的：

```python
most_recent_date = spark.sql(
	"SELECT max(partition_name) from table_name"
).first()[0]

# 当然在实际代码中不要使用 SELECT * 这种语法，这里只是为了方便起见。
result = spark.sql(
	"""
	SELECT * FROM table_name where partition_name = {partition}
	""".format(partition=most_recent_date)
)
```

利用Spark进行数据处理，对3TB/单一partition的数据集，花费时间为117s。在查阅相关资料<sup>[1]</sup>后我发现还有另外一种做法，通过show table的方式拿到所有的partition然后选择最新的partition。PySpark实现方法如下：

```python
most_recent_date = spark.sql(
	"SHOW PARTITIONS table_name"
).rdd.flatMap(
	lambda x: x
).map(
	lambda x: x.replace('partition_name=', '')
).max()

result = spark.sql(
	"""
	SELECT * FROM table_name where partition_name = {partition}
	""".format(partition=most_recent_date)
)
```

用同样的数据集进行测试，花费时间降低到了0.2s。之所以有这么大的改进，原因要从Hive的架构说起。

# Hive架构
![Hive Architecture](/images/hive_arch.png "Hive Architecture")

Hive的架构可以大致分为四层：

1. 用户接口 (CLI，JDBC)。用户可以通过终端交互(CLI)的方式与Hive进行连接，同时Hive也有基于JDBC (Java Database Connectivity)连接至Hive的服务；
2. 驱动引擎 Driver。Hive的驱动引擎包含了解析、编译、优化和执行，生成等过程；
3. 计算层。计算层Hive支持MapReduce，Tez，Spark等多种计算引擎；
4. 存储层。Hive数据存储包括两个方面，一个是元数据的存储，另一个是数据的持久化。

值得注意的是，在客户端与Driver之间，存在着跨语言服务 (thrift server)，集成多种语言帮助用户调用Hive接口。接下来我们主要来看存储层。

## 元数据 (Metadata) 的存储
Hive元数据主要包括表名，列和分区的属性，以及表的属性（内部表还是外部表）等等。数据库、表、分区对应HDFS下的目录。

Hive的元数据默认会存在Hive自带的Derby数据库中。Derby存在的问题是多用户连接操作支持不是很好，并且数据库目录不固定，不方便管理。一般生产环境中我们使用Mysql存储元数据。Mysql与Hive之间通过metastore进行交互。

## 数据存储
![Hive Data Model](/images/hive_data_model.png "Hive Data Model")

Hive的数据存储在HDFS中，基本存储单位为表或者分区。表或者分区在Hive内部被称为SD<sup>[2]</sup> (Storage Descriptor)。SD的基本信息存储在metastore中。Hive到底层数据的查询都会转化成MapReduce或者Tez/Spark的作业。

从以上介绍可以发现一点，当我们使用Hive SHOW PARTITION的时候，查询并不会跑到HDFS，而是**直接与metastore进行交互拿到分区的信息**，省略了MapReduce任务和各种编译、优化的环节，大大提高了查询速度。

# 总结
1. 元数据尽量从metastore中读取，`SELECT * `并不是万金油；
2. Hive适用于大规模数据的ETL，offline query和需要离线精确结果的场景。如果需要交互式adhoc查询最好移步Presto, Impala；

# Reference
* [1] <https://stackoverflow.com/questions/55053218/pyspark-getting-latest-partition-from-hive-partitioned-column-logic> "pyspark - getting Latest partition from Hive partitioned column logic"
* [2] <https://www.jianshu.com/p/4ce73f4d0bd5> "Hive原理及SQL优化"





