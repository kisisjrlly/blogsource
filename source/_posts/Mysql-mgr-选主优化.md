---
title: Mysql-mgr-选主优化
categories:
- distributed system
tags: 
- system
- distribute
- fault tolerance
---

## 选主优化
### 原生选主策略
MGR在单主模式下，当主节点退出集群时会触选主过程，默认的选主策略在选主时会比较节点的server_uuid 和 group_replication_member_weight， 选择哪个secondary作为新的主节点。


首先比较group_replication_member_weight，值最高的成员被选为新的主节点。如果相同，按照server_uuid排序，排序在最前的被选择为主节点。

此外可以通过group_replication_set_as_primary() 函数指定主节点：
```
select group_replication_set_as_primary(server_uuid);
```

查看group_replication_member_weight的值，范围为0-100，默认为50：
```
show variables like 'group_replication_member_weight';
```
设置group_replication_member_weight的值：
```
set global group_replication_member_weight = 200;
```
当想要设置的值小于0时，会自动设置为0，当想要设置的值大于100时，会默认等于100。调整权重后不能自动识别进行主从切换，当新加入的从节点权重较现有的主节点高时也不会发生主从切换，只有当主节点退出重新触发选主时才可以进行。

### 优化思路
在原生选主方式中，希望通过设置group_replication_member_weight参数，希望在发生主从切换时，将硬件配置、性能较好的节点选为主节点，因为理论上这种节点的选择回放速度最快，系统的响应时间最快。但在现实应用场景中，配置较好的节点回放速度不一定更快。因此当发生选主时，可以判断一下集群中所有节点回放速度，选择最快的节点作为主节点。当回放速度相同或者回放过程完成时，再按照原生策略进行选主。

回放速度可以通过gtid反应出来，在同一时刻，从节点中已执行的gtid值较大的节点回放速度加快，因此选主时选则gtid最新的节点为主节点。

通过增添和设置参数 primary_election_self_adaption=ON/OFF控制是否需要根据gtid按照节点执行速度选主，还是按照默认的权重+uuid方式选主。默认为OFF 

新加入集群的节点参数primary_election_self_adaption必须与集群中参数相同。

### 优化细节

MGR中class Group_member_info 及 class Group_member_info_manager 保存组中所有成员的状态信息，MGR中已经回放完成的事务对应的gtid保存在executed_gtid_set中。所有节点中executed_gtid_set值最大的节点回放速度最快。一个节点可以通过以下两种方式获取其他节点的executed_gtid_set：
- method 1：流控时可以得知其他节点的execute_gtid set。但是选主过程中只要最新时刻的gtid。通过流控过程传播的gitd需要不停的更新各个节点的最新gtid，代价较高。如果只在发生选主时才去读取流控得到的最新gtid，此时流控交互 的信息可能并没有发生传播。
- method 2: 选主时进入视图变更阶段，视图变更阶段的前期需要与其他节点交互获取其节点的member_weight，execute_gtid_ set等信息，并且进程选主过程的节点不再接受新的写请求，此时交互的信息即为当前集群中各个节点状态的最新信息。

- 选择方法2的方式获取其他节点的execute_gtid_set。Group_member_info 中添加成员变量primary_election_self_adaption,用来表示当前的选主方式。
- class Group_member_info 中 增加成员函数comparator_group_member_executed_gtid(), 用于选主排序sort函数的仿函数。通过子集的方式判断节点间gitd的大小关系。

### 具体实现

- [primary-elec-optimize](https://github.com/kisisjrlly/mysql-mgr-develop/commit/dbb5e54022c46811d92b61b83830cde815ac9036)






















