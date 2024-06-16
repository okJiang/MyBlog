---
title: DM 数据旅程 01：第一次 start task
date: 2023-01-03 02:58:33
tags:
- DM
- 源码阅读
categories:
- DM 数据旅程
top_img: /img/category/first_dm_job.png
cover: /img/category/first_dm_job.png
---

# 一、概述

本文以 start task 为目的，带着读者从 0 到 1 启动一个数据迁移任务，旨在让读者了解到最基础的 DM 逻辑。本文将直接参照集成测试 [start\_task](https://github.com/pingcap/dm/blob/master/tests/start_task/run.sh#L27-L36) 的过程，从以下几个方面展开：

1. Start dm-master
2. Start dm-worker
3. 绑定 source 和 dm-worker
4. Start task

> 注：为了专注于我们的目的（start task），本文不会对无关代码进行解读

> 大家可使用 [start/stop 流程](https://pingcap.feishu.cn/mindnotes/bmncnqlO5BCrkgxFqabTLaz6EQh#mindmap) 辅助阅读
>
> 由于写这篇的文章的时间是 2021 年 12 月份，所以所有的链接都是原 DM repo 的😂

# 二、start dm-master

1. [./dm-master](https://github.com/pingcap/dm/blob/master/tests/start_task/run.sh#L27)（in [run\_dm\_master](https://github.com/pingcap/dm/blob/master/tests/_utils/run_dm_master)） 启动二进制文件，即调用 [main 函数](https://github.com/pingcap/dm/blob/master/cmd/dm-master/main.go#L35)，其中 [master-server start](https://github.com/pingcap/dm/blob/master/cmd/dm-master/main.go#L69)
2. [go electionNotify](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/server.go#L232)：这个是为了[等待 ](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/election.go#L55)`etcd election`[ 成功](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/election.go#L55)，并在其成功后做⬇️

> DM master 中内嵌了一个 [etcd](https://etcd.io/)，用于存储各种元数据，并且借此保证 DM master 的高可用。后面非常多的数据存储都会用到 etcd。

3. [startLeaderComponent](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/election.go#L71)，其中我们这次只需要关注 [s.scheduler.Start](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/election.go#L173) 中的[go observeWorkerEvent](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/scheduler/scheduler.go#L243)，主要分为两部分

   1. [go WatchWorkerEvent](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/scheduler/scheduler.go#L1617)：该函数通过 etcd client 监听[是否有 workerEvent 出现](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/pkg/ha/keepalive.go#L198)

   2. [handleWorkerEv](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/scheduler/scheduler.go#L1619)：有 workerEvent 出现时，handle it

      1. [handleWorkerOffline](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/scheduler/scheduler.go#L1580)
      2. [handleWorkerOnline](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/scheduler/scheduler.go#L1582)

4. 这个时候，dm-master 等待 workerEvent 到来

# 三、start dm-worker

1. [./dm-worker](https://github.com/pingcap/dm/blob/master/tests/start_task/run.sh#L29)（in [run\_dm\_worker](https://github.com/pingcap/dm/blob/master/tests/_utils/run_dm_worker)）启动二进制文件，即调用 [main 函数](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/cmd/dm-worker/main.go)，其中[ worker-server start](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/cmd/dm-worker/main.go#L89)

2. [JoinMaster](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/cmd/dm-worker/main.go#L78)：先告诉 master，我来了！

   1. worker 先在这 [RegisterWorker](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/join.go#L72)，然后会触发 master 调用 [RegisterWorker](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/server.go#L298)
   2. Master 会调用 [AddWorker](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/server.go#L308)，然后 [PutWorkerInfo](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/scheduler/scheduler.go#L907)，把相应的 key-value [写到 etcd](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/pkg/ha/worker.go#L69) 中
   3. 可以看到写到 etcd 用的是 `clientv3.OpPut(key, value)`，也就是说 kv 要执行 put 操作
   4. 之前的 [go WatchWorkerEvent](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/scheduler/scheduler.go#L1617) 中就监听到有事件来了，并且判断其为 `mvccpb.PUT`[ 类型](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/pkg/ha/keepalive.go#L224)，event 处理之后会通过 [outCh](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/pkg/ha/keepalive.go#L242) 传到 handleWorkerEv 中进行具体的[上线处理](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/scheduler/scheduler.go#L1582)
   5. 刚上线的时候，就会去各种找 source 去 bound，但是现在我们还没有 create source，所以也找不到 source，暂时可以不关注这里

3. Start task 还需要 bound source，那 worker 首先要做的就是 [observeSourceBound](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/server.go#L169)，这里同 [observeWorkerEvent](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/scheduler/scheduler.go#L243) 是类似的：

   1. [go WatchSourceBound](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/server.go#L404)：通过 etcd client 监听[是否有 sourceBound 出现](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/pkg/ha/bound.go#L265)
   2. [handleSourceBound](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/server.go#L406)：上面监听到了之后，则 [operateSourceBound](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/server.go#L582)

4. 接下来，dm-worker 等待 source bound

# 四、operate-source create

> DM 用的命令行工具是 [cobra](https://github.com/spf13/cobra)，有兴趣的读者可深入了解一下

1. 命令行执行 [operate-source create](https://github.com/pingcap/dm/blob/master/tests/start_task/run.sh#L34)（in [test\_prepare](https://github.com/pingcap/dm/blob/master/tests/_utils/test_prepare#L128-L136)），`operate-source` 这个命令在 [NewOperateSourceCmd](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/ctl/ctl.go#L68) 注册，具体实现在 [operateSourceFunc](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/ctl/master/operate_source.go#L39)

2. 读取到该命令后，开始[解析](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/ctl/master/operate_source.go#L89)第一个参数（即 `create`）并[转换](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/ctl/master/operate_source.go#L47-L48)，最后被[打包送](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/ctl/master/operate_source.go#L143-L152)到 master，开始执行 master 的 [OperateSource](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/server.go#L1186) 函数

3. 该函数中，master 会从命令行中给出的配置文件路径

   1. [解析并调整](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/server.go#L1205) source config
   2. [把 source cfg 也存到 etcd 里](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/server.go#L1227)，因为 worker 待会要用
   3. [Try to bound it to a free worker](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/scheduler/scheduler.go#L318-L319)：因为我们是第一次 start task，并且也没有开启 relay 功能（[test](https://github.com/pingcap/dm/blob/master/tests/start_task/conf/source1.yaml#L4) 中是开启了，但本篇文章假设不开启），所以我们就只能 [bound a free worker](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/scheduler/scheduler.go#L1904-L1915) 了。
   4. 最终，通过 [PutSourceBound](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/scheduler/scheduler.go#L1936)，把 SourceBound [通过 etcd client 发送](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/pkg/ha/bound.go#L100)

4. 发送之后，worker 就通过 [go WatchSourceBound](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/server.go#L404) 监听到有 SourceBound 出现，然后进行 [operateSourceBound](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/server.go#L582)

   1. 首先需要[拿到 source cfg](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/server.go#L649)，因为上面的操作都是在 master 执行的，worker 这里并没有 source cfg
   2. Source cfg 也是通过 [etcd](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/pkg/ha/source.go#L83) 拿到的，正好上面存了

5. 之后就可以[开始 subtask 了吧](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/server.go#L658)！

   1. 但是并没有。。。我们还没开始 start task 呢！
   2. 所以 [fetchSubTasksAndAdjust](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/source_worker.go#L396) 并不能拿到 subtask。拿到是空的

6. 那没办法了，继续[等](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/source_worker.go#L422)呗（又是同样的 watch/handle 机制）

   1. [go WatchSubTaskStage](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/source_worker.go#L638)
   2. [handleSubTaskStage](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/source_worker.go#L640)

# 五、start-task

1. 命令行执行 [start-task](https://github.com/pingcap/dm/blob/master/tests/start_task/run.sh#L36)（in [test\_prepare](https://github.com/pingcap/dm/blob/master/tests/_utils/test_prepare#L53-L64)），`start-task` 命令的注册和实现参考 `operate-source`，最后执行 master 的 [StartTask](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/server.go#L404) 函数

2. 直接开始就 [generateSubTask](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/server.go#L426)（`req.Task` 直接传递的就是解析好的 `task.yaml` 字符串，原来在命令的实现中就帮我们解析好啦）。简单的说，就是经过一些 adjust 和 check， 帮助我们生成了 [SubTask](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/config/subtask.go#L184) struct

3. 重点来了，[AddSubTasks](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/server.go#L489) -> [NewSubTaskStage](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/scheduler/scheduler.go#L727)，subTask 终于创建好了，stage=running；再 [put](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/master/scheduler/scheduler.go#L739) 进 etcd，完美。可以看到我们分别把 [SubTaskCfg](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/pkg/ha/ops.go#L91) 和 [SubTaskStage](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/pkg/ha/ops.go#L95) 都 put 进 etcd 了。

4. 那上面就 watch 到 stage 来了，对 SubTaskCfg 进行[处理](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/source_worker.go#L682)，如果我们是要进行 run 的操作，我们还得[先把 cfg 拿出来](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/source_worker.go#L735-L743)，最后 [startSubTask](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/source_worker.go#L716)

5. startSubTask 中，会 [NewSubTask](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/source_worker.go#L481)，再 [runSubTask](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/source_worker.go#L504)。subTask 内部具体的执行组建是由 [unit](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/unit/unit.go#L32-L67) 负责的，所以它会

   1. [initUnits](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/subtask.go#L200)
   2. [st.run](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/subtask.go#L207) 其实也是由 [currentUnit](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/subtask.go#L228) 来 [Process](https://github.com/pingcap/dm/blob/39b5e2098f21260c14373a23069f7e38395d8d7f/dm/worker/subtask.go#L233)

# 六、结语

在 unit Process 后，start-task 就结束啦！是不是还意犹未尽呢？到底有哪些 unit 呢？这些 unit 内部到底是怎么 Process 的呢？在后续的文章中会陆续和大家见面哦。

其实再复读一下全文，我们发现本篇文章并没有太多很难的东西，大部分篇幅都在描述一些「准备活动」，全程用 etcd watch——master 等待 worker 到来、worker 等待 source 到来、source-worker 等待 subtask 到来。等就完事了。

任何建议和反馈都欢迎告诉我。下期再见！
