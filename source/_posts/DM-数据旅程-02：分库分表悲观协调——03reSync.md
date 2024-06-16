---
title: DM 数据旅程 02：分库分表悲观协调——03reSync
date: 2023-01-03 04:58:33
tags:
- DM
- 源码阅读
categories:
- DM 数据旅程
top_img: https://tidb-blog.oss-cn-beijing.aliyuncs.com/media/unnamed-1672404522042.png
cover: https://tidb-blog.oss-cn-beijing.aliyuncs.com/media/unnamed-1672404522042.png
---

# 一、概述

在分库分表同步的过程中，不同的 DDL 之间，还会穿插着各种各样的 DML 语句，这些 DML 语句要如何处理呢？今天就来看看它们的处理过程——reSync。

本节先介绍 reSync 的总流程，然后分为四个部分介绍 reSync：

- 一阶段：reSync 之前
- 开启 reSync
- 二阶段：reSync 之后
- 关闭 reSync

> 本节内容皆参考 [DM v6.0](https://github.com/pingcap/tiflow/tree/release-6.0)，对现在而言，有可能已过时，欢迎大家提出意见～

# 二、Overview

1. 进入 lock 阶段后，在第一次 sync 的时候，会跳过被 active 影响到的 targetTable 对应的 DML
2. 在 Lock resolved 之后，会回到第一次进入 lock 的 binlog Location 点，进行 reSync，把之前 skip 的 DML 重新 sync，这个阶段会跳过上次执行过的 DML。

![](https://tidb-blog.oss-cn-beijing.aliyuncs.com/media/unnamed-1672404522042.png)

## 原理

两种 DML 互相隔离，不会互相影响。

# 三、过程

## 1、一阶段

### Skip

在 handleRowsEvent 的时候，[判断该 event 是否需要 skip](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2368)：

1. 通过 targetTable 得到对应的 group

2. 通过 sourceTable 得到对应的 activeDDLItem，

   1. 下图中 ID3 即不需要 skip

   2. 如果已经收到 activeDDLItem

      1. 如果该 DML 在 active 前面，不需要 skip
      2. 如果该 DML 在 active 后面，则 skip

![](https://tidb-blog.oss-cn-beijing.aliyuncs.com/media/unnamed-1672404521513.png)

## 2、开始 reSync

[reSync 信号](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2970-L2975)：把 targetTable 和 binlog location 信息传递给主逻辑中，[重定向 streamer](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L1843-L1864)。

### ShardingReSync

```go
// ShardingReSync represents re-sync info for a sharding DDL group.

type ShardingReSync struct {

    currLocation   binlog.Location // current DDL's binlog location, initialize to first DDL's location

    latestLocation binlog.Location // latest DDL's binlog location

    targetTable    *filter.Table

    allResolved    bool

}
```

用到的 `ShardingReSync` 结构体：

- currLocation：用于重定向
- latestLocation：用于判断 reSync 是否结束
- targetTable：用于判断该 DML 是否需要 reSync
- allResolved：与关闭 reSync 有关

## 3、二阶段

1. [判断是否 skip](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2333-L2337)
2. 更新 `shardingReSync.currLocation` 并判断是否要结束 reSync

- [XIDEvent](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2125-L2143)
- [RotateEvent](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2285-L2300)
- [RowsEvent](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2328-L2332)
- [QueryEvent](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2629-L2645)

## 4、关闭 reSync

如果当前 binlog location 已经越过了 shardingReSync.lastestLocation，则重定向回去。根据是否 allResolved 有两种重定向方式，但貌似没有什么区别？

# 四、总结

经过 reSync 之后，悲观协调就结束了。reSync 的过程不是很复杂，主要包括两种操作：

- 重定向
- Skip

接下来将会开始乐观协调的学习（希望😭
