---
layout: post
title: ceph pg state简介
tags: ceph
---

# 一些概念

如果你执行过 ceph health 、 ceph -s 、或 ceph -w 命令，你也许注意到了集群并非总返回 HEALTH OK 。检查完 OSD 是否在运行后，你还应该检查归置组状态。你应该明白，在归置组建立连接时集群不会返回 HEALTH OK ：
1. 刚刚创建了一个存储池，归置组还没互联好；
2. 归置组正在恢复；
3. 刚刚增加或删除了一个 OSD ；
4. 刚刚修改了 CRUSH 图，并且归置组正在迁移；
5. 某一归置组的副本间的数据不一致；
6. Ceph 正在洗刷一个归置组的副本；
7. Ceph 没有足够空余容量来完成回填操作。

如果是前述原因之一导致了 Ceph 返回 HEALTH WARN ，无需紧张。很多情况下，集群会自行恢复；有些时候你得采取些措施。归置组监控的一件重要事情是保证集群起来并运行着，所有归置组都处于 active 状态、并且最好是 clean 状态。用下列命令查看所有归置组状态：

```
ceph pg stat
```

其结果会告诉你归置组运行图的版本号（ vNNNNNN ）、归置组总数 x 、有多少归置组处于某种特定状态，如 active+clean （ y ）。
> vNNNNNN: x pgs: y active+clean; z bytes data, aa MB used, bb GB / cc GB avail

==Note Ceph 同时报告出多种状态是正常的。==

除了归置组状态之外， Ceph 也会报告数据占据的空间（ aa ）、剩余空间（ bb ）和归置组总容量。这些数字在某些情况下是很重要的：

> - 快达到 ``near full ratio`` 或 ``full ratio`` 时；
> - 由于 CRUSH 配置错误致使你的数据没能在集群内分布。

归置组唯一标识符

归置组 ID 包含存储池号（不是存储池名字），后面跟一个点（ . ），然后是归置组 ID 一个十六进制数字。用 ceph osd lspools 可查看存储池号及其名字，默认存储池名字 data 、 metadata 、和 rbd 对应的存储池号分别是 0 、 1 
、 2 。完整的归置组 ID 格式如下：

```
{pool-num}.{pg-id}
```

典型长相：

```
0.1f
```

用下列命令获取归置组列表：

```
ceph pg dump
```

你也可以让它输出到 JSON 格式，并保存到文件：

```
ceph pg dump -o {filename} --format=json
```

要查询某个归置组，用下列命令：

```
ceph pg {poolnum}.{pg-id} query
```

Ceph 会输出成 JSON 格式。

```
{
  "state": "active+clean",
  "up": [
    1,
    0
  ],
  "acting": [
    1,
    0
  ],
  "info": {
    "pgid": "1.e",
    "last_update": "4'1",
    "last_complete": "4'1",
    "log_tail": "0'0",
    "last_backfill": "MAX",
    "purged_snaps": "[]",
    "history": {
      "epoch_created": 1,
      "last_epoch_started": 537,
      "last_epoch_clean": 537,
      "last_epoch_split": 534,
      "same_up_since": 536,
      "same_interval_since": 536,
      "same_primary_since": 536,
      "last_scrub": "4'1",
      "last_scrub_stamp": "2013-01-25 10:12:23.828174"
    },
    "stats": {
      "version": "4'1",
      "reported": "536'782",
      "state": "active+clean",
      "last_fresh": "2013-01-25 10:12:23.828271",
      "last_change": "2013-01-25 10:12:23.828271",
      "last_active": "2013-01-25 10:12:23.828271",
      "last_clean": "2013-01-25 10:12:23.828271",
      "last_unstale": "2013-01-25 10:12:23.828271",
      "mapping_epoch": 535,
      "log_start": "0'0",
      "ondisk_log_start": "0'0",
      "created": 1,
      "last_epoch_clean": 1,
      "parent": "0.0",
      "parent_split_bits": 0,
      "last_scrub": "4'1",
      "last_scrub_stamp": "2013-01-25 10:12:23.828174",
      "log_size": 128,
      "ondisk_log_size": 128,
      "stat_sum": {
        "num_bytes": 205,
        "num_objects": 1,
        "num_object_clones": 0,
        "num_object_copies": 0,
        "num_objects_missing_on_primary": 0,
        "num_objects_degraded": 0,
        "num_objects_unfound": 0,
        "num_read": 1,
        "num_read_kb": 0,
        "num_write": 3,
        "num_write_kb": 1
      },
      "stat_cat_sum": {

      },
      "up": [
        1,
        0
      ],
      "acting": [
        1,
        0
      ]
    },
    "empty": 0,
    "dne": 0,
    "incomplete": 0
  },
  "recovery_state": [
    {
      "name": "Started\/Primary\/Active",
      "enter_time": "2013-01-23 09:35:37.594691",
      "might_have_unfound": [

      ],
      "scrub": {
        "scrub_epoch_start": "536",
        "scrub_active": 0,
        "scrub_block_writes": 0,
        "finalizing_scrub": 0,
        "scrub_waiting_on": 0,
        "scrub_waiting_on_whom": [

        ]
      }
    },
    {
      "name": "Started",
      "enter_time": "2013-01-23 09:35:31.581160"
    }
  ]
}
```

后续子章节详述了常见状态。

1. 存储池在建中(creating)

创建存储池时，它会创建指定数量的归置组。 Ceph 在创建一或多个归置组时会显示 creating ；创建完后，在其归置组的 Acting Set 里的 OSD 将建立互联；一旦互联完成，归置组状态应该变为 active+clean ，意思是 Ceph 客户端可以向归置组写入数据了。
![image](../images/ceph/ceph_pg_states_change.png)

2. 互联建立中(peering)

Ceph 为归置组建立互联时，会让存储归置组副本的 OSD 之间就其中的对象和元数据状态达成一致。 Ceph 完成了互联，也就意味着存储着归置组的 OSD 就其当前状态达成了一致。然而，互联过程的完成并不能表明各副本都有了数据的最新版本。

> 权威历史
Ceph 不会向客户端确认写操作，直到 acting set 里的所有 OSD 都完成了写操作。这样处理保证了从上次成功互联起， acting set 中至少有一个成员确认了每个写操作。
有了各个已确认写操作的精确记录， Ceph 就可以构建和散布一个新的归置组权威历史——一个完整、完全有序的操作集，如果被采用，就能把一个 OSD 的归置组副本更新到最新。

3. 活跃(active)

eph 完成互联后，一归置组状态会变为 active 。 active 状态意味着数据已完好地保存到了主归置组和副本归置组。

4. 整洁(clean)

某一归置组处于 clean 状态时，主 OSD 和副本 OSD 已成功互联，并且没有偏离的归置组。 Ceph 已把归置组中的对象复制了规定次数。

5. 已降级(degraded)

当客户端向主 OSD 写入数据时，由主 OSD 负责把数据副本写入其余副本 OSD 。主 OSD 把对象写入存储器后，在副本 OSD 创建完对象副本并报告给主 OSD 之前，主 OSD 会一直停留在 degraded 状态。

归置组状态可以处于 active+degraded 状态，原因在于一 OSD 即使尚未持有所有对象也可以处于 active 状态。如果一 OSD 挂了， Ceph 会把分配到此 OSD 的归置组都标记为 degraded ；那个 OSD 重生后，它们必须重新互联。然而，客户端仍可以向处于 degraded 状态的归置组写入新对象，只要它还在 active 状态。

如果一 OSD 挂了，且老是处于 degraded 状态， Ceph 会把 down 的 OSD 标记为在集群外（ out ）、并把那个 down 掉的 OSD 上的数据重映射到其它 OSD 。从标记为 down 到 out 的时间间隔由 mon osd down out interval 控制，默认是 300 秒。

归置组也会被降级（ degraded ），因为 Ceph 找不到本应存在于此归置组中的一或多个对象，这时，你不能读写找不到的对象，但仍能访问位于降级归置组中的其它对象。

6. 恢复中(recovering)

Ceph 被设计为可容错，可抵御一定规模的软、硬件问题。当某 OSD 挂了（ down ）时，其内的归置组会落后于别的归置组副本；此 OSD 重生（ up ）时，归置组内容必须更新到当前状态；在此期间， OSD 处于 recovering 状态。

恢复并非总是这些小事，因为一次硬件失败可能牵连多个 OSD 。比如一个机柜或房间的网络交换机失败了，这会导致多个主机上的 OSD 落后于集群的当前状态，故障恢复后每一个 OSD 都必须恢复。

Ceph 提供了几个选项来均衡资源竞争，如新服务请求、恢复数据对象和恢复归置组到当前状态。 osd recovery delay start 选项允许一 OSD 在开始恢复进程前，先重启、重建互联、甚至处理一些重放请求；osd recovery threads 选项限制恢复进程的线程数，默认为 1 线程； osd recovery thread timeout 设置线程超时，因为多个 OSD 可能交替失败、重启和重建互联； osd recovery max active 选项限制一 OSD 最多同时接受多少请求，以防它压力过大而不能正常服务； osd recovery max chunk 选项限制恢复数据块尺寸，以防网络拥塞。

7. 回填中(backfilling)

有新 OSD 加入集群时， CRUSH 会把现有集群内的部分归置组重分配给它。强制新 OSD 立即接受重分配的归置组会使之过载，用归置组回填可使这个过程在后台开始。只要回填顺利完成，新 OSD 就可以对外服务了。

在回填运转期间，你可能见到以下几种状态之一： backfill_wait 表明一回填操作在等待时机，尚未开始； backfill 表明一回填操作正在进行； backfill_too_full 表明需要进行回填，但是因存储空间不足而不能完成。某归置组不能回填时，其状态应该是 incomplete 。

Ceph 提供了多个选项来解决重分配归置组给一 OSD （特别是新 OSD ）时相关的负载问题。默认， osd_max_backfills 把双向的回填并发量都设置为 10 ； osd backfill full \ ratio 可让一 OSD 在接近占满率（默认 85% ）时拒绝回填请求，如果一 OSD 拒绝了回填请求，在 osd backfill retry interval 间隔之后将重试（默认 10 秒）； OSD 也能用 osd backfill scan min 和 osd backfill scan max 来管理扫描间隔（默认 64 和 512 ）。

8. 被重映射(remapped)

负责维护某一归置组的 Acting Set 变更时，数据要从旧集合迁移到新的。新的主 OSD 要花费一些时间才能提供服务，所以老的主 OSD 还要持续提供服务、直到归置组迁移完。数据迁移完后，运行图会包含新 acting set 里的主 OSD 。

9. 发蔫(stale)

虽然 Ceph 用心跳来保证主机和守护进程在运行，但是 ceph-osd 仍有可能进入 stuck 状态，它们没有按时报告其状态（如网络瞬断）。默认， OSD 守护进程每半秒（ 0.5 ）会一次报告其归置组、出流量、引导和失败统计状态，此频率高于心跳阀值。如果一归置组的主 OSD 所在的 acting set 没能向监视器报告、或者其它监视器已经报告了那个主 OSD 已 down ，监视器们就会把此归置组标记为 stale 。

启动集群时，会经常看到 stale 状态，直到互联完成。集群运行一阵后，如果还能看到有归置组位于 stale 状态，就说明那些归置组的主 OSD 挂了（ down ）、或没在向监视器报告统计信息。

# 找出故障归置组

如前所述，一个归置组状态不是 active+clean 时未必有问题。一般来说，归置组卡住时 Ceph 的自修复功能往往无能为力，卡住的状态细分为：

- Unclean: 归置组里有些对象的副本数未达到期望次数，它们应该在恢复中；
- Inactive: 归置组不能处理读写请求，因为它们在等着一个持有最新数据的 OSD 回到 up 状态；
- Stale: 归置组们处于一种未知状态，因为存储它们的 OSD 有一阵子没向监视器报告了（由 mon osd report timeout 配置）。

为找出卡住的归置组，执行：

```
ceph pg dump_stuck [unclean|inactive|stale|undersized|degraded]
```

详情见[归置组子系统](http://ceph-doc.imaclouds.com/rados/operations/control/#placement-group-subsystem)，关于排除卡住的归置组见[排除归置组错误](http://ceph-doc.imaclouds.com/rados/troubleshooting/troubleshooting-pg/#troubleshooting-pg-errors)。

# 定位对象

要把对象数据存入 Ceph 对象存储，一 Ceph 客户端必须：

1. 设置对象名
2. 指定一存储池

Ceph 客户端索取最新集群运行图、并用 CRUSH 算法计算对象到归置组的映射，然后计算如何动态地把归置组映射到 OSD 。要定位对象，只需要知道对象名和存储池名字，例如：

```
ceph osd map {poolname} {object-name}
```

# 练习：定位一个对象

反正是练习，我们先创建一个对象。给 rados put 命令指定一对象名、一个包含数据的测试文件路径、和一个存储池名字，例如：

```
rados put {object-name} {file-path} --pool=data
rados put test-object-1 testfile.txt --pool=data
```

用下列命令确认 Ceph 对象存储已经包含此对象：

```
rados -p data ls
```

现在可以定位对象了：

```
ceph osd map {pool-name} {object-name}
ceph osd map data test-object-1
```

Ceph 应该输出对象的位置，例如：

```
osdmap e537 pool 'data' (0) object 'test-object-1' -> pg 0.d1743484 (0.4) -> up [1,0] acting [1,0]
```

要删除测试对象，用 rados rm 即可，如：

```
rados rm test-object-1 --pool=data
```

随着集群的运转，对象位置会动态改变。 Ceph 动态重均衡的优点之一，就是把你从人工迁移中解救了，详情见[体系结构](http://docs.openfans.org/ceph/ceph4e2d658765876863/ceph-1/architecture301067b667843011/copy_of_architecture301067b667843011)。

# PLACEMENT GROUP STATES

When checking a cluster’s status (e.g., running ceph -w or ceph -s), Ceph will report on the status of the placement groups. A placement group has one or more states. The optimum state for placement groups in the placement group map is active + clean.

1. Creating

Ceph is still creating the placement group.

2. Active

Ceph will process requests to the placement group.

3. Clean

Ceph replicated all objects in the placement group the correct number of times.

4. Down

A replica with necessary data is down, so the placement group is offline.

5. Replay

The placement group is waiting for clients to replay operations after an OSD crashed.

6. Splitting

Ceph is splitting the placement group into multiple placement groups. (functional?)

7. Scrubbing

Ceph is checking the placement group for inconsistencies.

8. Degraded

Ceph has not replicated some objects in the placement group the correct number of times yet.

9. Inconsistent

Ceph detects inconsistencies in the one or more replicas of an object in the placement group (e.g. objects are the wrong size, objects are missing from one replica after recovery finished, etc.).

10. Peering

The placement group is undergoing the peering process

11. Repair

Ceph is checking the placement group and repairing any inconsistencies it finds (if possible).

12. Recovering

Ceph is migrating/synchronizing objects and their replicas.

13. Backfill

Ceph is scanning and synchronizing the entire contents of a placement group instead of inferring what contents need to be synchronized from the logs of recent operations. Backfill is a special case of recovery.

14. Wait-backfill

The placement group is waiting in line to start backfill.

15. Backfill-toofull

A backfill operation is waiting because the destination OSD is over its full ratio.

16. Incomplete

Ceph detects that a placement group is missing information about writes that may have occurred, or does not have any healthy copies. If you see this state, try to start any failed OSDs that may contain the needed information or temporarily adjust min_size to allow recovery.

17. Stale

The placement group is in an unknown state - the monitors have not received an update for it since the placement group mapping changed.

18. Remapped

The placement group is temporarily mapped to a different set of OSDs from what CRUSH specified.

19. Undersized

The placement group fewer copies than the configured pool replication level.

20. Peered

The placement group has peered, but cannot serve client IO due to not having enough copies to reach the pool’s configured min_size parameter. Recovery may occur in this state, so the pg may heal up to min_size eventually.