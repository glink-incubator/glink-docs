---
title: Overview
weight: -20
---

{{< toc >}}

Glink (<u>**G**</u>eographic F<u>**link**</u>)是Flink在空间数据处理领域的扩展, 为Flink增加了兼容OGC标准的空间数据类型. Glink在`DataStream` API的基础之上构建了一层`SpatialDataStream` API, 增加了支持空间数据操作, 处理和分析的算子; 并在SQL API上扩展了符合SFA SQL规范的空间处理函数.

## Glink Use Case

Glink主要用于带有空间属性的无界数据(无界空间数据)的流处理, 目前支持的功能如下下:
+ Spatial Filter: 用于对无界空间数据进行空间过滤, 支持任意数量过类型的过滤几何, 同时支持任意类型的空间关系. Glink对常见的"选取一个或多个多边形内的点数据"这一场景进行了优化.
+ Spatial KNN: 用于在无界空间数据的窗口快照上执行KNN.
+ Spatial Join
    + Spatial Dimension Join: 用于无界空间数据与空间维度表的连接, 空间维度表支持以`BroadcastStream`的形式输入, 或存储在外部空间数据引擎(如GeoMesa)中.
    + Spatial Window Join: 用于对两个无界空间数据集进行Window Join.
    + Spatial Interval Join: 用于对两个无界空间数据集进行Interval Join.
+ Spatial Window DBSCAN: 用于在无界空间数据的窗口快照上执行DBSCAN聚类.

## Glink Architecture

Glink在Flink的基础之上增加了Spatial Data Stream Layer, Spatial Stream Processing Layer和Spatial API Layer. 其中, Spatial Data Stream Layer构建在Flink `DataStream` API的基础之上, 提供基于网格的空间分区, 并对外提供空间操作算子; Spatial Stream Processing Layer包含Glink的三种典型应用; Spatial API Layer提供两套使用Glink的 API, Spatial SQL API允许用户直接使用SQL和Glink提供的空间扩展函数进行相关操作, Java API允许用户编写Java代码调用`SpatialDataStream` API执行相应功能。

<p align="center">
    <img src="/media/glink/glink-arch.png" width="80%">
</p>