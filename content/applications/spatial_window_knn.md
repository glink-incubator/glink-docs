---
title: Spatial Window KNN
weight: -90
---

{{< toc >}}

Spatial Window KNN用于实现窗口化的KNN查询, 即根据输入的查询点, 实时输出每个时间窗口内距离(目前仅支持欧式距离)查询点最近的K个流几何. 它包含以下几个要素:
1. 无界空间数据: 被查询的主体, 将根据指定的窗口长度进行划分.
2. 查询点: 用来进行查询的空间点.
3. K值: 当前窗口中与查询点距离最近的K个流几何将被保留.
3. 窗口: 定义窗口的划分方式, 每个窗口内的数据将分别进行KNN查询.

Glink目前仅在`SpatialDataStream` API中提供了Spatial Window KNN的支持, 以下是一个案例. 该案例将对无界空间点数据进行滑动窗口划分, 窗口长度为5s, 并对每个窗口内的数据执行KNN查询, 距离空间点`POINT (100.5, 30.5)`最近的3个流几何将被输出. 我们在Glink的源代码中提供了一个可直接运行的案例, 具体可参见[SpatialWindowKNNExample](https://github.com/glink-incubator/glink/blob/master/glink-examples/src/main/java/cn/edu/whu/glink/examples/datastream/SpatialWindowKNNExample.java).

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

Point queryPoint = SpatialDataStream.geometryFactory.createPoint(new Coordinate(100.5, 30.5));
SpatialDataStream<Point> pointDataStream = new SpatialDataStream<>(
    env, pointTextPath, new SimpleSTPointFlatMapper());
pointDataStream.assignTimestampsAndWatermarks(WatermarkStrategy
    .<Point>forMonotonousTimestamps()
    .withTimestampAssigner((point, time) -> {
        Tuple2<String, Long> userData = (Tuple2<String, Long>) point.getUserData();
        return userData.f1;
    }));
DataStream<Point> knnStream = SpatialWindowKNN.knn(
    pointDataStream,
    queryPoint,
    3,
    Double.MAX_VALUE,
    TumblingEventTimeWindows.of(Time.seconds(5)));
```