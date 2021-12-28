---
title: GeoMesa SQL Connector
weight: -20
---

{{< toc >}}

Glink对Flink的SQL Connector进行了扩展, 实现了GeoMesa SQL Connector. 这里介绍了如何设置和使用GeoMesa SQL Connector. 我们在GeoMesa 3.1+版本上进行了测试.

## Dependencies

为了使用GeoMesa SQL Connector, 需要添加如下依赖.

| Glink dependency |
| ----------- |
| glink-connector-geomesa-x.x.x.jar |
| glink-sql-x.x.x.jar |

最简单的方法是将`$GLINK_HOME/lib`目录中的上述两个Jar包复制到`$FLINK_HOME/lib`目录下.

## DDL

使用GeoMesa SQL Connector时, 在DDL中有几点需要特别注意, 这里进行详细说明.

### DDL中定义空间数据类型

由于Flink当前无法支持可注册的自定义类型, 因此我们无法在DDL中直接定义空间数据类型. 在GeoMesa SQL Connector中可以WKT/WKB形式表示空间数据类型, 并在`WITH`参数中用`geomesa.spatial.fields`指明, 其格式为: `<field name>:<field type>`, 多个字段由","分隔. 其中`<field type>`支持[Spatial Data Types](#spatial-data-types)中的所有GeoMesa Type.

如下DDL定义了GeoMesa中的一个表, 用于存储T-Drive数据, 其中`point2`字段为WKT格式的`STRING`类型, 并且在`WITH`参数中将其指定为`Point`空间类型. 这样, 如果将其作为source, 那么GeoMesa SQL Connector从GeoMesa取出数据时会将`point2`字段的数据类型从`Point`转化为WKT格式的`STRING`; 如果将其作为sink, 那么GeoMesa SQL Connector将数据写入GeoMesa时会将WKT格式的`STRING`转化为`Point`. 

```sql
CREATE TABLE GeoMesa_TDrive (
    `pid` STRING,
    `time` TIMESTAMP(0),
    `point2` STRING,
    PRIMARY KEY (pid) NOT ENFORCED
) WITH (
    'connector' = 'geomesa',
    'geomesa.data.store' = 'hbase',
    'geomesa.schema.name' = 'geomesa-test',
    'geomesa.spatial.fields' = 'point2:Point',
    'hbase.catalog' = 'test-sql'
);
```

### DDL中指定空间连接谓词

目前Flink仅支持等值连接, 因此在Glink中使用GeoMesa进行lookup join时, 我们必须为每个连接指定其空间关系. 我们通过在`WITH`参数中增加`geomesa.temporal.join.predict`选项来实现. `geomesa.temporal.join.predict`目前支持以下选项:
+ `R:<distance>`:表示维度几何与流几何距离小于`distance`米即符合空间连接条件;
+ `I`: 表示流几何与维度几何相交即符合空间连接条件;
+ `+C`: 表示流几何包含维度几何即符合空间连接条件;
+ `-C`: 表示维度几何包含流几何即符合空间连接条件.

以下是一个使用GeoMesa SQL Connector进行lookup join的案例, 它读取CSV文件中的点数据, 将其与GeoMesa中的多边形数据进行空间连接, 如果点被包含在某个多边形中, 则将二者进行连接. 注意`'geomesa.temporal.join.predict' = 'I'`这一行, 它指定了空间连接的类型为包含(对于多边形与点之间的空间关系, 包含和相交是一样的), 这样在DQL的连接条件`ON ST_AsText(ST_Point(A.lng, A.lat)) = B.geom`中, Glink就会将`=`理解为两侧的几何类型相交.

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

CREATE TABLE GeoMesa_Area (
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

SELECT A.id AS point_id, A.dtg, ST_AsText(ST_Point(A.lng, A.lat)) AS point, B.id AS area_id
    FROM csv_point AS A
    LEFT JOIN GeoMesa_Area FOR SYSTEM_TIME AS OF A.proctime AS B
    ON ST_AsText(ST_Point(A.lng, A.lat)) = B.geom;
```

## How to use GeoMesa table

上述案例中已经使用过GeoMesa表, 这里再用一个完整的例子进一步阐述. 在这个例子中, 我们实现的功能是使用Glink将CSV中的点数据导入到GeoMesa中, 它几乎是一个完整的空间数据ETL案例了.

首先创建CSV source table.

```sql
CREATE TABLE CSV_TDrive (
    `pid` STRING,
    `time` TIMESTAMP(0),
    `lng` DOUBLE,
    `lat` DOUBLE
) WITH (
    'connector' = 'filesystem',
    'path' = '/path/to/csv',
    'format' = 'csv'
);
```

然后创建GeoMesa sink table.

```sql
CREATE TABLE Geomesa_TDrive (
    `pid` STRING,
    `time` TIMESTAMP(0),
    `point2` STRING,
    PRIMARY KEY (pid) NOT ENFORCED
) WITH (
    'connector' = 'geomesa',
    'geomesa.data.store' = 'hbase',
    'geomesa.schema.name' = 'geomesa-test',
    'geomesa.spatial.fields' = 'point2:Point',
    'hbase.catalog' = 'test-sql'
);
```

最后执行以下DML语句即可执行ETL作业.

```sql
INSERT INTO Geomesa_TDrive 
    SELECT `pid`, `time`, ST_AsText(ST_Point(lng, lat)) FROM CSV_TDrive;
```

## Connector Options

GeoMesa SQL Connector支持通过WITH参数的方式对GeoMesa客户端的相关参数进行配置. 其中有些参数是引擎无关的, 有些参数是不同后端存储引擎的可选配置, 具体如下.

### GeoMesa

以下配置是不受GeoMesa后端存储引擎所影响的.

| Option | Required | Description |
| ---- | ---- | ---- |
| connector | required | 连接器类型, 对于GeoMesa SQL Connector而言固定为geomesa |
| geomesa.data.store | required | GeoMesa Data Store类型, 目前支持hbase |
| geomesa.schema.name | required | GeoMesa Schema名称 |
| geomesa.spatial.fields | optional | 空间类型字段, 当包含空间字段时必须指定, 否则空间类型将无法正确解析, 格式: <field name>:<field type>, 多个字段间由","分隔 |
| geomesa.temporal.join.predict | optional | 指定lookup join的空间关系谓词, 符合关系的记录将被join:<br>R:<distance>表示维表中空间对象与流表中空间对象距离小于distance米;<br>I表示流表中空间对象与维表中空间对象相交;<br>+C表示流表中空间对象包含维表中空间对象;<br>-C表示维表中空间对象包含流表中空间对象. |

### HBase Data Store

以下参数是GeoMesa以HBase作为后端存储引擎时可选的配置参数. GeoMesa SQL Connector支持GeoMesa HBase Data Store的所有配置参数, 关于各个参数的具体含义, 参见[Geomesa文档](https://www.geomesa.org/documentation/stable/user/hbase/usage.html#hbase-data-store-parameters).

| Option | Required |
| ---- | ---- |
| hbase.catalog | optional |
| hbase.zookeepers | optional |
| hbase.coprocessor.url | optional |
| hbase.config.paths | optional |
| hbase.config.xml | optional |
| hbase.connections.reuse | optional |
| hbase.remote.filtering | optional |
| hbase.security.enabled | optional |
| hbase.coprocessor.threads | optional |
| hbase.ranges.max-per-extended-scan | optional |
| hbase.ranges.max-per-coprocessor-scan | optional |
| hbase.coprocessor.arrow.enable | optional |
| hbase.coprocessor.bin.enable | optional |
| hbase.coprocessor.density.enable | optional |
| hbase.coprocessor.stats.enable |  optional |
| hbase.coprocessor.yield.partial.results | optional |
| hbase.coprocessor.scan.parallel | optional |

{{< hint info >}}
注意: 当`geomesa.data.store`为hbase时必须指定`hbase.catalog`
{{< /hint >}}

## Data Type Mapping

Flink SQL的数据类型并不与GeoMesa完全兼容. 对于基础数据类型而言GeoMesa SQL Connector进行了最大程度的适配; 对于空间数据类型, 由于Flink目前尚未支持可注册的结构化类型, 因此在Flink SQL中所有空间数据类型均由WKT/WKB格式的SRING/BINARY类型表示, 且必须使用`geomesa.spatial.fields`这一`WITH`参数指定具体类型, Geomesa SQL Connector在写入时会使用`org.locationtech.jts.io.WKTReader`/`org.locationtech.jts.io.WKBReader`进行转换. 详细的数据类型对应关系如下. Flink SQL数据类型参见[Flink文档](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/table/types/). GeoMesa数据类型参见[GeoMesa文档](https://www.geomesa.org/documentation/stable/user/datastores/attributes.html).

### Basic Data Types

| Flink SQL Type | GeoMesa Type | Java Type | Indexable |
| ---- | ---- | ---- | ---- |
| CHAR / VARCHAR / STRING | String | java.lang.String | Yes |
| BOOLEAN | Boolean | java.lang.Boolean | Yes |
| BINARY / VARBINARY / BYTES | Bytes | byte[] | No | 
| TINYINT / SMALLINT / INT | Integer | java.lang.Integer | Yes |
| BIGINT | Long | java.lang.Long | Yes |
| FLOAT | Float | java.lang.Float | Yes |
| DOUBLE | Double | java.lang.Double | Yes |
| DATE / TIME | Date | java.util.Date | Yes |
| TIMESTAMP | Timestamp | java.sql.Timestamp | Yes |
| DECIMAL | Not supported | | |
| ARRAY | Not supported | | |
| MAP / MULTISET | Not supported | | |
| Row | Not supported | | |

### Spatial Data Types

所有空间数据类型在Glink SQL中均由WKT/WKB格式的STRING/ARRAY类型表示.

| GeoMesa Type | Java Type | Indexable |
| ---- | ---- | ---- |
| Point | org.locationtech.jts.geom.Point | Yes |
| LineString | org.locationtech.jts.geom.LineString | Yes |
| Polygon | org.locationtech.jts.geom.Polygon | Yes |
| MultiPoint | org.locationtech.jts.geom.MultiPoint | Yes |
| MultiLineString | org.locationtech.jts.geom.MultiLineString | Yes |
| MultiPolygon | org.locationtech.jts.geom.MultiPolygon | Yes |
| GeometryCollection | org.locationtech.jts.geom.GeometryCollection | Yes |
| Geometry | org.locationtech.jts.geom.Geometry | Yes |