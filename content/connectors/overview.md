---
title: Overview
weight: -30
---

{{< toc >}}

Glink需要与外部存储引擎交互, 以从中摄取数据进行计算或输出计算结果. 在Glink中, 外部存储引擎通常有以下几种用途:
+ 在ETL场景中, 作为数据输入(source), 或计算结果的输出(sink). source端一般是支持流式消费的消息队列或文件, sink端既可以是消息队列和文件(用于下游流计算), 也可以是数据库(用于查询或分析).
+ 借助外部存储引擎的时空索引能力, 进行空间lookup join (spatial lookup join).

然而, 由于Flink目前并不支持空间数据类型, 且无法支持[可注册的自定义类型](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/table/types/#user-defined-data-types), 因此Flink现有的Connector对空间数据类型的支持是有限制的.
+ 对于像Kafka这种无schema的存储引擎, 可以用Glink提供的Spatial SQL将空间数据转换为WKT/WKB进行存储.
+ 对于关系型数据库, 尽管MySQL, PostreSQL都支持空间数据存储, 但是直接用Flink JDBC Connector是无法写入空间数据类型的, 因为这些数据库需要严格的类型定义, 而目前在Flink中无法在`CREATE TABLE` DDL中定义空间数据类型. 当然, 我们可以以WKT/WKB的形式将空间数据存储到关系型数据库中, 遗憾的是这样我们只能简单的存储数据, 无法使用存储引擎的空间索引能力.

不过好在Flink一般在大数据量的场景下使用, 在大数据场景下目前最为流行的时空数据存储引擎是[GeoMesa](https://www.geomesa.org/documentation/stable/), 底层支持各类分布式存储引擎. 为此Glink通过扩展Flink的Connector框架, 提供了GeoMesa Connector, 它支持完备的空间数据类型和空间关系.