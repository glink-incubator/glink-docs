---
title: Glink SQL最佳实践 - GeoMesa结合应用
type: posts
date: 2020-01-08
tags:
  - 最佳实践
---

原文链接: [Glink SQL最佳实践 - GeoMesa结合应用](https://liebing.org.cn/2021/02/20/glink_sql_best_practice-geomesa-application/)

----

[GeoMesa](https://www.geomesa.org/documentation/stable/index.html)已经成为时空数据存储领域重要的索引中间件, [京东城市时空数据引擎JUST](https://just.urban-computing.cn/)和阿里云的[HBase Ganos](https://help.aliyun.com/knowledge_detail/87287.html)均是在GeoMesa的基础上扩展而来. GeoMesa采用键值存储, 支持多种类型的存储后端, 如HBase, Kafka, Redis等. 相对于PostgreSQL+PostGIS这种基于R-tree索引的关系型存储, GeoMesa的存储方案更容易与HBase等现有的分布式数据库相结合, 从而直接利用底层数据库的分布式特性, 更适合时空大数据的存储以及实时场景的应用. 

为在时空流计算中利用GeoMesa的高效写入和时空查询能力, Glink扩展Flink SQL Connector框架形成了Flink GeoMesa SQL Connector(简称GeoMesa SQL Connector), 支持使用Flink SQL读写GeoMesa. 本文通过实际的应用案例, 讲述如何在Flink SQL中使用GeoMesa. 在流计算中Flink+GeoMesa主要有以下两种使用场景:
+ **时空数据管道 & ETL**: 以GeoMesa作为时空数据存储引擎, 通过Flink SQL构建实时的时空数据ETL管道, 将时空数据从文件, Kafka等数据源导入到GeoMesa;
+ **Spatial temporal join**: 将维表存储在GeoMesa中, 通过Flink SQL进行流表与维表的**空间join**.

本文需要glink-0.1.2及以上版本, 可在zepplin中运行, 关于Glink及zepplin的安装配置参考[Glink文档](https://liebingyu.gitbook.io/glink/install). 为方便复现笔者提供了可直接运行的[glink-geomesa.zpln](https://github.com/traj-explorer/glink/blob/master/glink-examples/src/main/zepplin/glink-geomesa.zpln), 下载后可直接在zepplin打开运行.

<!--more-->

## 时空数据管道 & ETL
在IoT等行业, 产生的大量时空数据一般会接入到Kafka, 之后经过清洗, 转换, 增强存入时空数据库, 这就需要建设时空数据ETL管道. Flink在实时数据ETL管道建设中已经起到了重要作用. 然而Flink不支持空间数据类型, 同时也缺乏与空间数据库, 如GeoMesa等的Connector. 为此Glink增加了GeoMesa SQL Connector, 支持与GeoMesa进行交互, 方便了时空数据ETL管道的建设.

在Glink中, 所有空间数据类型均用WKT格式的`STRING`类型表示, 同时通过Connector参数`geomesa.spatial.fields`指定空间类型字段和表示的几何类型. GeoMesa SQL Connector在写入GeoMesa时会将WKT转化为实际的几何对象. 下面通过一个实际的案例讲述如何利用Glink构建时空数据ETL管道.

在该案例中我们将CSV文件中的数据通过Flink SQL导入到GeoMesa中. CSV文件中每行代表一个空间点, 总共包含四列, 每列的含义是: 点ID, 点生成时间, 经度, 纬度. 以下是一个简单的案例.

```shell
1,2008-02-02 13:30:40,116.31412,70.89454
2,2008-02-02 13:30:44,116.31412,39.89454
3,2008-02-02 13:30:45,116.32674,39.89577
4,2008-02-02 13:30:49,116.31412,39.89454
```

{{< hint info >}}
这里将CSV文件作为source只是为了方便, source可以是Kafka, MySQL等Flink支持的任意组件.
{{< /hint >}}

首先创建source table, DDL语句如下.

```sql
CREATE TABLE csv_table (
    id STRING,
    dtg TIMESTAMP(0),
    lng DOUBLE,
    lat DOUBLE
) WITH (
    'connector' = 'filesystem',
    'path' = '/path/to/file',
    'format' = 'csv'
);
```

然后创建GeoMesa sink table.

```sql
CREATE TABLE geomesa_table (
    id STRING,
    dtg TIMESTAMP(0),
    point STRING,
    PRIMARY KEY (id) NOT ENFORCED
) WITH (
    'connector' = 'geomesa',
    'geomesa.data.store' = 'hbase',
    'geomesa.schema.name' = 'geomesa_table',
    'geomesa.spatial.fields' = 'point:Point',
    'hbase.catalog' = 'geomesa_table'
);
```

最后通过如下语句即可实现将数据从CSV文件导入GeoMesa, 完成数据管道的构建.

```sql
INSERT INTO geomesa_table 
    SELECT id, dtg, ST_AsText(ST_Point(lng, lat)) FROM csv_table;
```

## Spatial Temporal Join
在流计算中, 流表与维表的关联是一项重要的基础功能. Flink可通过temporal join实现流表与维表的关联. 然而, 目前Flink的temporal join只支持等值join, 对于时空数据而言, 通常需要基于流表与维表中对象的空间关系进行join. 为此, Glink抽象出了spatial temporal join, 支持基于空间关系的temporal join, 目前Glink的spatial temporal join支持**距离join**, **相交join**和**包含join**.

{% colorquote info %}
的processing time temporal join. event time temporal join需要支持chengelog模式数据源, 无法在GeoMesa中实现.
{% endcolorquote %}

spatial temporal join具有大量的应用场景, 比如:
1. 地理围栏应用, 流表中每条记录表示行人或车辆的轨迹点, 维表存储在GeoMesa中, 每条记录都是一个由多边形表示的地理围栏. 为了判断流表中的轨迹点是否出入了某个地理围栏, 可以将流表与维表做一个包含join, 若某个轨迹点被包含在某个多边形围栏中, 则这两条记录会执行join.
2. 订单调度应用, 流表中每条记录都表示一个订单, 包含订单送达目的地的经纬度坐标, 维表存储在GeoMesa中, 每条记录都是由经纬度点表示的仓库位置. 为了获取与订单位置在某个距离范围内的仓库, 可以将流表与维表做一个距离join, 若订单位置与仓库位置在距离范围内, 则这两条记录会执行join.

在Glink中可以通过`geomesa.temporal.join.predict`这一Connector参数指定进行何种类型的空间join:
+ `R:<distance>`表示距离join, 流表中空间对象与维表中空间对象距离在`distance`之内的记录都会被join, `distance`的单位为米;
+ `I`表示相交join, 流表中空间对象与维表中空间对象在空间上相交的记录都会被join;
+ `+C`表示正相交join, 流表中空间对象若在空间上包含维表中空间对象, 则两条记录会被join;
+ `-C`表示负相交join, 维表中空间对象若在空间上包含流表中空间对象, 则两条记录会被join.

下面通过具体的案例讲述如何使用Glink进行spatial temporal join.

### 相交/包含join
我们通过地理围栏应用讲述如何在Glink中进行相交join或包含join. 在地理围栏应用中, 流表中的一条记录通常是行人或车辆的轨迹点, 包含一些非空间属性及轨迹点的经纬度坐标. 维表中的一条记录通常代表一块地理区域, 包含一些非空间属性及地理区域的空间范围(由多边形表示). 在本例中, 流表来自CSV文件, 维表存储在GeoMesa中. 通过相交join或负相交join可以实现轨迹点与地理围栏的关联.

首先定义流表, DDL语句如下.

```sql
CREATE TABLE csv_point (
    id STRING,
    dtg TIMESTAMP(0),
    lng DOUBLE,
    lat DOUBLE,
    proctime AS PROCTIME())
WITH (
    'connector' = 'filesystem',
    'path' = '/path/to/csv', 
    'format' = 'csv'
);
```

GeoMesa维表的定义语句如下, `geomesa.temporal.join.predict`用于指定空间join的类型, 在地理围栏应用中使用`I`和`-C`可以达到相同的结果. 但是使用I有更高的效率.

```sql
CREATE TABLE geomesa_area (
    id STRING,
    dtg TIMESTAMP(0),
    geom STRING,
    PRIMARY KEY (id) NOT ENFORCED)
WITH (
    'connector' = 'geomesa',
    'geomesa.data.store' = 'hbase',
    'geomesa.schema.name' = 'restricted_area',
    'geomesa.spatial.fields' = 'geom:Polygon',
    'geomesa.temporal.join.predict' = 'I',
    'hbase.zookeepers' = 'localhost:2181',
    'hbase.catalog' = 'restricted_area'
);
```

通过如下语句进行spatial temporal join.

```sql
SELECT A.id AS point_id, A.dtg, ST_AsText(ST_Point(A.lng, A.lat)) AS point, B.id AS area_id
    FROM csv_point AS A
    LEFT JOIN geomesa_area FOR SYSTEM_TIME AS OF A.proctime AS B
    ON ST_AsText(ST_Point(A.lng, A.lat)) = B.geom;
```

### 距离join
我们通过订单调度应用讲述如何在Glink中进行距离join. 在订单调度应用中, 流表中每条记录都表示一个订单, 包含相关的非空间属性及订单送达目的地的经纬度坐标; 维表存储在GeoMesa中, 每条记录都是由经纬度点表示的仓库位置. 在订单调度应用中, 通常需要为每个订单关联某个距离范围内的仓库, 用于订单的分发调度. 这可以通过Glink的距离join实现.

首先定义流表, DDL语句如下.

```sql
CREATE TABLE csv_order (
    id STRING,
    dtg TIMESTAMP(0),
    lng DOUBLE,
    lat DOUBLE,
    proctime AS PROCTIME())
WITH (
    'connector' = 'filesystem',
    'path' = '/path/to/csv',
    'format' = 'csv'
);
```

然后定义GeoMesa维表.

```sql
CREATE TABLE geomesa_warehouse (
    id STRING,
    geom STRING,
    PRIMARY KEY (id) NOT ENFORCED)
WITH (
    'connector' = 'geomesa',
    'geomesa.data.store' = 'hbase',
    'geomesa.schema.name' = 'warehouse_point',
    'geomesa.spatial.fields' = 'geom:Point',
    'geomesa.temporal.join.predict' = 'R:400000',
    'hbase.zookeepers' = 'localhost:2181',
    'hbase.catalog' = 'warehouse_point'
);
```

最后通过如下语句即可实现距离join.

```sql
SELECT A.id AS order_id, A.dtg, ST_AsText(ST_Point(A.lng, A.lat)) AS order_point, B.id AS warehouse_id
    FROM csv_order AS A
    LEFT JOIN geomesa_warehouse FOR SYSTEM_TIME AS OF A.proctime AS B
    ON ST_AsText(ST_Point(A.lng, A.lat)) = B.geom;
```

## 参考
[1] [Flink SQL最佳实践 - HBase结合应用](https://lb-yu.github.io/2021/01/19/Flink-SQL%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5-HBase%E7%BB%93%E5%90%88%E5%BA%94%E7%94%A8/)
