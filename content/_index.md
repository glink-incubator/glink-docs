---
title: 
description: Glink
geekdocNav: false
geekdocAlign: center
geekdocAnchor: false
geekdocHiddenTocTree: true
---

# Glink - A Spatial Extension of Apache Flink

<!-- markdownlint-capture -->
<!-- markdownlint-disable MD033 -->

<!-- <span class="badge-placeholder">[![Build Status](https://img.shields.io/drone/build/thegeeklab/hugo-geekdoc?logo=drone&server=https%3A%2F%2Fdrone.thegeeklab.de)](https://drone.thegeeklab.de/thegeeklab/hugo-geekdoc)</span>
<span class="badge-placeholder">[![Hugo Version](https://img.shields.io/badge/hugo-0.83-blue.svg)](https://gohugo.io)</span> -->
<span class="badge-placeholder">[![GitHub release](https://img.shields.io/github/v/release/glink-incubator/glink)](https://github.com/glink-incubator/glink/releases/latest)</span>
<span class="badge-placeholder">[![GitHub contributors](https://img.shields.io/github/contributors/glink-incubator/glink)](https://github.com/glink-incubator/glink/graphs/contributors)</span>
<span class="badge-placeholder">[![License: Apache](https://img.shields.io/github/license/glink-incubator/glink)](https://github.com/glink-incubator/glink/blob/master/LICENSE)</span>
<span class="badge-placeholder">[![GitHub starts](https://img.shields.io/github/stars/glink-incubator/glink)](https://github.com/glink-incubator/glink/stargazers)</span>
<span class="badge-placeholder">[![GitHub forks](https://img.shields.io/github/forks/glink-incubator/glink)](https://github.com/glink-incubator/glink/network/members)</span>

<!-- markdownlint-restore -->

Glink is an extension of [Apache Flink](https://flink.apache.org/) in the field of spatial data. It adds spatial processing operators and supports spatial data types that conform to the OGC standard. Glink can make spatial stream data processing as simple as writing a "WordCount" program.

{{< button size="large" relref="introduction/getting-started/" >}}Getting Started{{< /button >}}

## Feature overview

{{< columns >}}

### Spatial Filter

Filter unbounded spatial data and optimize scenes with complex polygons as filter conditions.

<--->

### Spatial Window KNN

Perform a KNN query on the spatial data in each window snapshot.

<--->

### Spatial Join

Spatial join for unbounded spatial data, support stream and table join and double stream join.

{{< /columns >}}


{{< columns >}}

### Spatial Window DBSCAN

Perform DBSCAN clustering on the spatial data in each window snapshot, and output the clustering results in real time.

<--->

<--->

{{< /columns >}}