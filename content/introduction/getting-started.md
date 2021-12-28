---
title: Getting Started
weight: -10
---

{{< toc >}}

## Run Example of Glink

1. 在[Release](https://github.com/glink-incubator/glink/releases)页面下载Glinker二进制发布包.
2. 解压发布包.
```shell
tar -zxvf glink-x.x.x-bin.tar.gz
```
3. 运行Glink提供的Spatial Filter示例.
```shell
nc -lk 9999
cd glink-x.x.x/examples
flink run -c cn.edu.whu.glink.examples.datastream.SpatialFilterExample glink-examples-1.0.0.jar 
```
该示例有两个查询几何, `Polygon ((0 5,5 10,10 5,5 0,0 5))`和`Polygon ((5 5,10 10,15 5,10 0,5 5))`, 用于过滤落在这两个多边形外部的点. 它监听`9999`端口, 可在`nc -lk 9999`窗口输入如下数据进行测试.

```text
1,114.35,34.50
2,2,2
```

## Use Glink in Program

1. 下载Glink源代码.
```shell
git clone git@github.com:glink-incubator/glink.git
```

2. 编译并安装Glink依赖到本地仓库.
```shell
mvn clean install -DskipTests
```

3. 在新工程的`pom.xml`中引入Glink依赖.
```xml
<properties>
    <glink.version>x.x.x</glink.version>
</properties>

<dependency>
    <groupId>cn.edu.whu</groupId>
    <artifactId>glink-core</artifactId>
    <version>${glink.version}</version>
</dependency>
<dependency>
    <groupId>cn.edu.whu</groupId>
    <artifactId>glink-sql</artifactId>
    <version>${glink.version}</version>
</dependency>
<dependency>
    <groupId>cn.edu.whu</groupId>
    <artifactId>glink-connector-geomesa</artifactId>
    <version>${glink.version}</version>
    <exclusions>
        <exclusion>
            <artifactId>slf4j-log4j12</artifactId>
            <groupId>org.slf4j</groupId>
        </exclusion>
    </exclusions>
</dependency>
```