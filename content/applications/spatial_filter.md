---
title: Spatial Filter
weight: -100
---

{{< toc >}}

Spatial Filter用于进行时间无关的无界空几间数据处理. 一般用于对无界空间数据进行基于空间关系的过滤操作, 比如对轨迹数据流进行过滤, 以保留落在某个指定行政区内的轨迹数据. 它包含三个要素:
1. 无界空间数据: 被过滤的主体.
2. 查询几何: 用来对空间数据流进行过滤的几何图形.
3. 拓扑关系: 指示流几何与查询几何在空间上处于何种拓扑关系时, 该记录将被保留.

Glink在`SpatialDataStream` API中对Spatial Filter提供了完善的支持, 包括支持所有符合OGC标准的几何要素对象以及拓扑关系, 以及支持多个查询几何的场景, 并对此进行了优化. Spatial SQL API中同样提供了对Spatial Filter的支持, 不过存在一定限制.

## SpatialDataStream API

以下是一个使用`SpatialDataStream` API进行Spatial Filter的案例. 该案例对一个无界空间点数据进行过滤, 所有包含在多边形 `POLYGON ((10 10, 10 20, 20 20, 20 10, 10 10))`内的点将被保留, 其他点将被过滤. 我们在Glink的源代码中提供了一个可运行的Spatial Filter案例, 具体可参见[SpatialFilterExample](https://github.com/glink-incubator/glink/blob/master/glink-examples/src/main/java/cn/edu/whu/glink/examples/datastream/SpatialFilterExample.java).

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

Geometry queryGeometry = ... // e.g "POLYGON ((10 10, 10 20, 20 20, 20 10, 10 10))"

SpatialDataStream<Point> pointDataStream = new SpatialDataStream<>(
    env, "localhost", 9999, new SimplePointFlatMapper());
DataStream<Point> resultStream = SpatialFilter.filter(
    pointDataStream, queryGeometry, TopologyType.CONTAINS);
```

Glink提供了完整的拓扑关系模型, 兼容OGC标准, 并在此基础上进行了扩展, Spatial Filter所支持的拓扑关系类型如下. 在 `SpatialDataStream` API 中, 可通过`TopologyType`枚举类进行指定.

| 拓扑关系 | 枚举类型 | 是否 OGC 标准 |
| ---- | ---- | ---- |
| 包含 | CONTAINS | 是 |
| 交叉 | CROSSES | 是 |
| 相离 | DISJOINT | 是 |
| 相等 | EQUAL | 是 |
| 相交 | INTERSECTS | 是 |
| 覆盖 | OVERLAPS | 是 |
| 接触 | TOUCH | 是 |
| 包含于 | WITHIN | 是 |
| 距离包含于 | WITHIN_DISTANCE | 否 |
| 缓冲 | BUFFER | 否 |

需要注意的是, 这里的拓扑关系是流几何对于查询几何而言的.

## Spatial SQL API

Glink SQL提供了多个空间函数, 可用于进行Spatial Filter. 函数列表如下.

| Spatial SQL函数 | 拓扑关系 |
| ---- | ---- |
| ST_Contains | CONTAINS |
| ST_Covers | COVERS |
| ST_Crosses | CROSSES |
| ST_Disjoint | DISJOINT |
| ST_Equals | EQUALS |
| ST_Intersects | INTERSECTS |
| ST_Overlaps | OVERLAPS |
| ST_Touches | TOUCHES |
| ST_Within | WITHIN |

使用Spatial SQL函数可以实现简单的Spatial Filter, 如下案例实现了与上述`SpatialDataStream` API中案例相同的语义.

```sql
CREATE TABLE point_table (
    id STRING,
    dtg TIMESTAMP(0),
    lng DOUBLE NOT NULL,
    lat DOUBLE NOT NULL,
) WITH (
    ...
)

SELECT * FROM point_table 
WHERE ST_Contains(
    ST_PolygonFromText('POLYGON ((10 10, 10 20, 20 20, 20 10, 10 10))'), 
    ST_Point(lng, lat)
);
```

{{< hint info >}}
这里需要注意的是, 尽管在Spatial SQL中我们可以通过串联多个空间关系判断函数实现多个查询几何的Spatial Filter, 但是这毫无疑问是低效的, 我们需要遍历所有的查询几何并进行判断. 相比之下, SpatialDataStream API在多个查询几何的情况下进行了优化, 它首先会为所有的查询几何建立R-tree, 这样在流几何与与查询几何进行空间关系判断时可借助R-tree索引, 从而避免了每次判断都遍历所有查询几何, 这在查询几何数量较多的情况下尤为明显.

在Spatial SQL API中如果要实现包含大量查询几何的Spatial Filter场景, 可借助lookup join功能, 把查询几何存储在外部存储引擎中, 关于如何在Glink中进行spatial lookup join可参考[Spatial Dimension Join](../spatial_join/#spatial-dimension-join).
{{< /hint >}}