---
title: Glink SQL on Zepplin
weight: -10
---

{{< toc >}}

Glink SQL可以借助Flink的SQL API直接在Java代码中使用, 不过借助SQL客户端可以免去创建工程和配置依赖的繁琐. 目前开源可用的Flink SQL客户端主要有Flink自带的[SQL Client](https://nightlies.apache.org/flink/flink-docs-master/docs/dev/table/sqlclient/)的和[Apache Zepplin](https://zeppelin.apache.org/). 我们将在Zepplin上使用Glink SQL.

关于Zepplin的安装这里不再赘述, 如有问题可参考[Flink on Zeppelin](https://www.yuque.com/jeffzhangjianfeng/gldg8w), 其中有详细的教程.

## Config Glink SQL on Zepplin

要在Zepplin中使用Glink扩展的SQL函数需要在Flink Interpreter指定UDF Jar路径. 在`flink.udf.jars`配置项中增加`$GLINK_HOME/lib/glink-sql-x.x.x.jar`即可.