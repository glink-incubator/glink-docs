---
title: Spatial Join
weight: -80
---

{{< toc >}}

Glink提供了三种类型的Spatial Join, 分别是用于流-维Join的Spatial Dimension Join, 以及用于双流Join的Spatial Window Join和Spatial Interval Join. 下面将分别介绍.

## Spatial Dimension Join

Spatial Dimension Join用于流-维Join, 可以实现无界空间数据中流几何与空间维度表中维度几何的空间连接. 比如现在有一个轨迹数据流, 以及相应的区县级行政区划(每个区/县的地理范围都对应一个地理多边形), Spatial Dimension Join可将二者进行空间连接, 每个轨迹点与空间上包含其的行政区连接.

Glink在`SpatialDataStream` API以及Spatial SQL API中均提供了对Spatial Dimension Join的支持, 不过目前实现方式有所区别. 在`SpatialDataStream`中, Glink将空间维度表以`BraodcastDataStream`的形式表示, 在内存中为空间维度表建立了R-tree索引, 吞吐量较高; 在Spatial SQL中, 借助lookup join语法实现, 空间维度表必须存储在外部引擎中, 目前仅支持GeoMesa HBase Datastore, 由于每个流几何的连接都需要访问外部存储, 其吞吐量相对较低, 仍存在进一步优化的空间.

### SpatialDataStream API

以下是一个使用`SpatialDataStream` API进行Spatial Dimension Join的案例. 该案例对一个无界空间点数据`pointDataStream`与一个空间维度表`spatialDataStream`进行连接, 当维度几何在空间上包含流几何时(由`TopologyType.N_CONTAINS`指定), 即将二者进行连接操作. 

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

SpatialDataStream<Point> pointDataStream = new SpatialDataStream<>(
    env, "localhost", 8888, new SimplePointFlatMapper());
BroadcastSpatialDataStream<Geometry> spatialDataStream = new BroadcastSpatialDataStream<>(
    env, "localhost", 9999, new WKTGeometryBroadcastMapper());

SpatialDimensionJoin.join(
    pointDataStream,
    spatialDataStream,
    TopologyType.N_CONTAINS,
    Tuple2::new,
    new TypeHint<Tuple2<Point, Geometry>>() { });
```

### Spatial SQL API

在Spatial SQL中, Glink借助Flink的[lookup join](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/table/sql/queries/joins/#lookup-join)语法实现了Spatial Dimension Join. 由于lookup join必须借助外部存储, 目前Glink仅支持GeoMesa作为外部存储, 具体可见[GeoMesa SQL Connector](../../connectors/geomesa).

关于使用Spatial SQL实现Spatial Dimension Join的案例可参考[DDL中指定空间连接谓词](../../connectors/geomesa/#ddl中指定空间连接谓词).

## Spatial Window Join

Spatial Window Join支持两个无界空间数据的连接, 在时间维度上与Flink的[window join](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/operators/joining/#window-join)有相同的语义. 以下案例对两个无界空间点数据进行空间连接, 在每个5s的滑动窗口内, 只要两个无界空间数据中的流几何距离在1KM以内即会进行连接.

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

SpatialDataStream<Point> pointSpatialDataStream1 =
    new SpatialDataStream<>(env, "localhost", 9000, new SimpleSTPointFlatMapper())
        .assignTimestampsAndWatermarks(WatermarkStrategy
        .<Point>forBoundedOutOfOrderness(Duration.ZERO)
        .withTimestampAssigner((event, time) -> ((Tuple2<String, Long>) event.getUserData()).f1));
SpatialDataStream<Point> pointSpatialDataStream2 =
    new SpatialDataStream<>(env, "localhost", 9001, new SimpleSTPointFlatMapper())
        .assignTimestampsAndWatermarks(WatermarkStrategy
        .<Point>forBoundedOutOfOrderness(Duration.ZERO)
        .withTimestampAssigner((event, time) -> ((Tuple2<String, Long>) event.getUserData()).f1));

DataStream<Tuple2<Point, Point>> windowJoinStream = SpatialWindowJoin.join(
    pointSpatialDataStream1,
    pointSpatialDataStream2,
    TopologyType.WITHIN_DISTANCE.distance(1),
    TumblingEventTimeWindows.of(Time.seconds(5)));
```

## Spatial Interval Join

Spatial Interval Join同样支持两个无界空间数据的连接, 在时间维度上与Flink的[interval join](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/datastream/operators/joining/#interval-join)有相同的语义. 以下案例对两个无界空间点数据进行连接, 对一个无界空间数据中的任意一个流几何, 连接另一个无界空间点数据中在前后5s时间窗口内的数据.

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

SpatialDataStream<Point> pointSpatialDataStream1 =
    new SpatialDataStream<>(env, "localhost", 8888, new SimpleSTPointFlatMapper())
        .assignBoundedOutOfOrdernessWatermarks(Duration.ZERO, 1);
SpatialDataStream<Point> pointSpatialDataStream2 =
    new SpatialDataStream<>(env, "localhost", 9999, new SimpleSTPointFlatMapper())
        .assignBoundedOutOfOrdernessWatermarks(Duration.ZERO, 1);

DataStream<Tuple2<Point, Point>> joinStream = SpatialIntervalJoin.join(
    pointSpatialDataStream1,
    pointSpatialDataStream2,
    TopologyType.WITHIN_DISTANCE.distance(1),
    Time.seconds(-5),
    Time.seconds(5));
```