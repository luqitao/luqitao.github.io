# ceph-mon简介
---
Ceph monitors(MONs)通过使用一个与集群状态相关的map来跟踪整个集群的健康状况，它监控的对象   包括OSD，MON，PG，CRUSH maps。集群中所有的节点，当它们的每一个状态发生变化后，会上报和共享信息给监控节点。监控程序为每个组件维护一个单独的map对象。Ceph monitor不存储实际数据，这  是OSD的工作。

