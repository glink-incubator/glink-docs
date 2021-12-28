---
title: Concepts
weight: 0
---

{{< toc >}}

在Glink的源代码或本文档中, 为了实现更好的抽象, 我们引入了一些新的概念和名词, 这里进行集中介绍, 以方便读者更好地理解代码或文档中的内容.

## Unbounded Spatial Data (无界空间数据)

无界数据(Unbounded Data)的概念已经成熟, 无界空间数据是指带有空间属性的无界数据, 即数据中的每条记录都代表空间上的某个几何对象, 并可附带其他相关属性. 需要强调的一点是, 无界空间数据中的每条记录必须至少包含一个空间属性, 其他属性则是可选的. Glink中的`SpatialDataStream`便是一种无界空间数据的具体实现.

## Stream Geometry (流几何)

无界空间数据中的单条记录称为流几何, 在Glink中用JTS的[`Geometry`](https://github.com/locationtech/jts/blob/master/modules/core/src/main/java/org/locationtech/jts/geom/Geometry.java)表示, 记录中的其他属性以Flink `Tuple`的形式存储在`Geometry`的`userData`成员变量中.

## Query Stream (查询几何)

在无界空间数据的处理算子中, 输入的几何参数称为查询几何. 查询几何通常作为Spatial Filter或Spatial Join的条件.

## Spatial Dimension Table (空间维度表)

空间维度表表示带有空间属性的一系列数据集合, 其中的数据记录可以是不变的或缓慢变化的. 这里的空间维度表是一种抽象的概念, 因此其物理形态可以是多样的, 比如在Flink中以`BroadcastDataStream`的形态存在, 或者存储在外部的空间存储引擎(如GeoMesa)中.

## Dimension Geometry (维度几何)

空间维度表中的单条记录称为维度几何.