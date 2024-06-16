---
title: DM 数据旅程 02：分库分表悲观协调——02Lock -> Resolve Lock
date: 2023-01-03 03:58:33
tags:
- DM
- 源码阅读
categories:
- DM 数据旅程
top_img: https://tidb-blog.oss-cn-beijing.aliyuncs.com/media/unnamed-1672403283666.png
cover: https://tidb-blog.oss-cn-beijing.aliyuncs.com/media/unnamed-1672403283666.png
---

# 一、概述

介绍了与悲观协调有关的各个数据结构之后，接下来将介绍一个 DM 系统从接受到第一条分表 DDL 开始，到所有该 DDL 对应的 Lock resolved 的全过程。

> 本节内容皆参考 [DM v6.0](https://github.com/pingcap/tiflow/tree/release-6.0)，对现在而言，有可能已过时，欢迎大家提出意见～

# 二、Overview

假设现在起了 master 和两个 Worker，两个 Worker 分别绑定两个 Source（s1，s2），每个 Source 有两个分表（t1，t2），这四个表会 route 到一个 target table（tarTbl）中。在 DDL 到来之前，两个 worker 会创建好 ShardingGroupKeeper，里面只有一个 target table，两个 source table。

![](https://tidb-blog.oss-cn-beijing.aliyuncs.com/media/unnamed-1672403283187.png)

两个 Worker 先后收到两个 Source 四个分表的同一条 DDL，表示为：

- DDL1：表示对 s1.t1 的 DDL。
- DDL2：表示对 s2.t1 的 DDL。
- DDL3：表示对 s1.t2 的 DDL。
- DDL4：表示对 s2.t2 的 DDL。

![](https://tidb-blog.oss-cn-beijing.aliyuncs.com/media/unnamed-1672403283666.png)

# 三、具体过程

> 本小节 worker 对 DDL 的处理从 [handleQueryEventPessimistic](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2853) 开始，如果不知道在此之前的 DDL 做过哪些处理，请期待后续文章😁

接下来将对上图中的步骤一一介绍

## Step 1

1. worker1 收到来自 source1 的 DDL1，会直接 [TrySync](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2883)，这里返回了一大堆的参数，其实有用的就只是 `synced`，简单来说，里面只做了一件事：[AddItem](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/sharding_group.go#L215)。下面详细介绍一下 TrySync

> 本节，假设该 DDL 不是 CreateTableStmt，如果是的话，也很简单，每次都是 remain <=0，synced = true，但是只有第一次发送该 create table 的 worker 会 syncDDL。

### TrySync

想要知道 worker 在悲观协调的时候做了什么，必须要搞懂 TrySync 是怎么运作的。在此之前，我们来重新理解一下 ShardingGroupKeeper：

![](https://tidb-blog.oss-cn-beijing.aliyuncs.com/media/unnamed-1672403283187.png)

单独拿出一个 ShardingGroup 举例（颜色代表 DDL 的种类）：

#### Example

worker 收到了以下 DDL

- 依次收到了 table1 橙色、黄色、绿色三条 DDL
- 收到了 table2 橙色
- 这个时候 `activeIdx = 0`，`remain = 1`，因为只有 table3 没有收到 active DDL 了

![](https://tidb-blog.oss-cn-beijing.aliyuncs.com/media/unnamed-1672403283679.png)

- 如果在这个时候收到了 table3 橙色 DDL，`remain = 0`，处理了这条 DDL 之后，`active ++`

![image.png](https://tidb-blog.oss-cn-beijing.aliyuncs.com/media/image-1672713793518.png)

- 如果这个时候收到 table2 绿色 DDL，则会[报错](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/sharding-meta/shardmeta.go#L197-L199)，悲观协调要求同一个 source table 中 DDL 必须有序

![](https://tidb-blog.oss-cn-beijing.aliyuncs.com/media/unnamed-1672403283186.png)

- 如果 group 中所有的 DDL 都 resolved 了。则会[重置](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/sharding-meta/shardmeta.go#L243-L246)一下 meta

![](https://tidb-blog.oss-cn-beijing.aliyuncs.com/media/unnamed-1672403283692.png)

总结：[AddItem](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/sharding_group.go#L215) 就是上面图中把 DDLItem 放进 ShardingSequence 的过程

## Step 2 与 Step 1 相同

## Step 3

由于 tarTbl 只有两个 source table：s1.t1 和 s1.t2，这时 [TrySync](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2883) 后，[synced = true](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2948)，说明该 worker ready

## Step 4

### worker1

- [PutInfo](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2985)：发送 ready info 给 master（后面会详细介绍这个函数）
- 之后会被阻塞，[等待 master 发送 operation](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2992)。

### Master

- [watchInfoPut](https://github.com/pingcap/tiflow/blob/release-6.0/dm/dm/master/shardddl/pessimist.go#L177)：成功监听到了
- handleInfoPut：master 也开始 [TrySync](https://github.com/pingcap/tiflow/blob/release-6.0/dm/dm/master/shardddl/pessimist.go#L474)，master 的 TrySync 相比 worker 中的要简单很多，每一个 Info 直接相当于该 source/worker 已 ready，所以直接统计是否所有 ready 即可
- 由于这里是第一个 worker 发 info 给 master，所以需要[新建 Lock](https://github.com/pingcap/tiflow/blob/release-6.0/dm/pkg/shardddl/pessimism/keeper.go#L49)，新建 Lock 过程中会获取该 task 所有的 source，并存到 lock 中，这样就知道还有多少 source 没 ready。显然还有 source2 没 ready。
- [继续等待](https://github.com/pingcap/tiflow/blob/release-6.0/dm/dm/master/shardddl/pessimist.go#L481-L485)

## Step 5 与 Step 3 相同

## Step 6

- 前面与 Step 4 相同，但是这时所有 source 都 ready 了，[synced = true](https://github.com/pingcap/tiflow/blob/release-6.0/dm/dm/master/shardddl/pessimist.go#L486)

## Step 7

### master

- [putOpForOwner](https://github.com/pingcap/tiflow/blob/release-6.0/dm/dm/master/shardddl/pessimist.go#L599)：发送 [exec](https://github.com/pingcap/tiflow/blob/release-6.0/dm/dm/master/shardddl/pessimist.go#L604) operation to owner(worker1)

### Worker1

- [得到 master 的 exec operation](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/shardddl/pessimist.go#L120-L124)

- [NewDDLJob](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L3042-L3043) 并 handle

- [addJob](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L1057)：发送 job 给 DDLJobCh

- syncDDL：之前 worker 起的协程接受到该 job

  - [拿到刚刚得到的 operation](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L1324)，并且不会被跳过，因为 ignore = false

## Step 8

- [同步到下游](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L1344)

## Step 9

### Worker1

- [DoneOperationDeleteInfo](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L1384)：发送 done operation 给 master 并删除 info

## Step 10

### Master

- [接受 done operation](https://github.com/pingcap/tiflow/blob/release-6.0/dm/dm/master/shardddl/pessimist.go#L509)
- [markDone](https://github.com/pingcap/tiflow/blob/release-6.0/dm/dm/master/shardddl/pessimist.go#L529)
- [putOpsForNonOwner](https://github.com/pingcap/tiflow/blob/release-6.0/dm/dm/master/shardddl/pessimist.go#L553)：给其他所有 non-owner 发送 skip info（skip done 是为了防止把 done 覆盖掉）

### Worker2

- 接受到了 no exec(skip) 的 operation，和 Step 7 类似，但是在 syncDDL 的时候会[被 skip](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L1325-L1328)

## Step 11 和 Step 9 相同

# 四、总结

本节介绍了在悲观协调过程中，遇到第一条 DDL 语句，生成 Lock，到收到所有 DDL 语句，Lock 解除的过程。

经过本节的学习，我们已经完全学会了 DM 悲观协调的过程。是不是也没那么复杂😏，但是在这个过程中，我们只知道 DDL 的处理方式，这个过程中的 DML 会怎么办呢？下一章揭晓。

# 五、疑似 bug

1. [initShardingGroups](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L552-L559) 中对 ShardingGroup.sources 进行了初始化，这里包含了该 targetTable 对应的所有 sourceTables
2. 在 [TrySync](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2883) 的时候，却使用第一个 `sourceTable[0]`，来标记所有的 `DDL[]`，如果 `len(DDL[]) != 1`，则 remain 永远不可能为 0。因为 sourceTables 来源于 ddlInfo。而 DDLInfo 只存了[第一条 ddl 的 ddlInfo](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2738-L2743)，实际上对于每一个 Split 之后的 DDL，都会 [genDDLInfo](https://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2694)。
3. 但是奇怪的是，代码中标注了[不支持 multi-table DDL](http://ErrSyncerUnitDDLOnMultipleTable)，这个 error 在 pessimist/optimist mode 中都用到了。但是在现在看 pessimist 代码中，又看到其为 multi-table DDL 实现了部分相关逻辑。。。**有毒**
4. 比如：

- https\://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/syncer.go#L2957-L2960
- https\://github.com/pingcap/tiflow/blob/release-6.0/dm/syncer/sharding-meta/shardmeta.go#L35
- 。。。

