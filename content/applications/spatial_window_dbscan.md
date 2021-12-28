---
title: Spatial Window DBSCAN
weight: -70
---

{{< toc >}}

Spatial Window DBSCAN用于实现窗口化的DBSCAN聚类, 即对每个窗口内的数据执行DBSCAN聚类并即时输出聚类结果. 它包含以下几个要素:
1. 无界点数据: 被查询的主体, 将根据指定的窗口长度进行划分, 对于DBSCAN聚类只支持点对象.
2. DBSCAN参数: 包括距离阈值*ε*和最小点数*minPts*.
3. 窗口: 定义窗口的划分方式, 每个窗口内的数据将分别进行DBSCAN聚类.

Glink目前仅在`SpatialDataStream` API中提供了Spatial Window DBSCAN的支持, 以下是一个案例. 该案例将对无界空间点数据进行滑动窗口划分, 窗口长度为5min, 并对每个窗口内的数据执行DBSCAN聚类, 距离阈值为0.15KM, 最小点数为3. 我们在Glink的源代码中提供了一个可直接运行的案例, 具体可参见[WindowDBSCANExample](https://github.com/glink-incubator/glink/blob/master/glink-examples/src/main/java/cn/edu/whu/glink/examples/datastream/WindowDBSCANExample.java).

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

SpatialDataStream<Point2> pointDataStream = new SpatialDataStream<>(
    env, path, new SimpleSTPoint2FlatMapper())
    .assignTimestampsAndWatermarks(WatermarkStrategy
        .<Point2>forMonotonousTimestamps()
        .withTimestampAssigner((p, time) -> p.getTimestamp()));
DataStream<Tuple2<Integer, List<Point2>>> dbscanStream = WindowDBSCAN.dbscan(
    pointDataStream,
    0.15,
    3,
    TumblingEventTimeWindows.of(Time.minutes(5)));
```