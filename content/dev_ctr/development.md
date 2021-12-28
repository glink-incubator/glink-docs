---
title: Development
weight: 10
---

{{< toc >}}

## Preparation

在开始开发之前, 先从代码仓库下载Glink源码.

```shell
git clone git@github.com:glink-incubator/glink.git
```

## Building Glink from Source

为从源码构建Glink, 需要安装Maven 3和Java 8. 安装完成后使用如下命令即可进行编译.

```shell
mvn clean paclage -DskipTests
```

## Importing Flink into IntelliJ IDEA

我们使用IntelliJ IDEA开发Glink, 可通过如下步骤在IDEA中打开Glink项目:
1. 打开IDEA, 选择"Open";
2. 选中Glink源代码所在文件夹, 点击"OK"即可打开;
3. 等待IDEA自动导入项目.

Glink在编译时强制进行Checkstyle检查, 不通过则终止编译. 在IDEA中可使用"CheckStyle-IDEA"插件进行检查, 若未安装该插件, 通过如下步骤安装并配置Glink Checkstyle:
1. 选择"File->Settings->Plugins", 在搜索框输入"CheckStyle-IDEA"并搜索, 点击安装该插件;
2. 重启IDEA;
3. 选择"File->Settings->Tools->Checkstyle";
4. 在"Scan Scope"中选择"Only java sources(including tests)";
5. 在"Configuration File"中点击"+";
6. 在弹出框中, "Description"填入"glink", 点击"Browse", 选择"glink/config/checkstyle.xml"文件.
7. 点击"Next"完成即可.

完成上述步骤后, 即可在IDEA底部栏中看到"CheckStyle"选项卡, 点开即可进行扫描.