---
title: "HBase: ServerCrashProcedure"
tags:
    - HBase
categories: HBase
---

## 可能有用的细节

需要xlock  org.apache.hadoop.hbase.master.procedure.ServerQueue#requireExclusiveLock

## 触发条件

### zk session expire

配置: {zookeeper.znode.parent}/{zookeeper.znode.rs}
默认: /hbase/{cluster name}/rs

此种情况下，如果rs处于非ONLINE状态，不会强制执行ServerCrashProcedure

### HBCK2

```
./hbase hbck -j hbase-hbck2-1.0.0-SNAPSHOT.jar scheduleRecoveries c4-hadoop-tst-st84.bj,55600,1586416554312 c4-hadoop-tst-st85.bj,55600,1586415546993
```

会强制执行 HBCKServerCarshProcedure，大多数情况下行为和ServerCrashProcedure一样，所以如果rs卡在一个状态太久，可能有问题，可以用HBCK2执行SCP强行修复

行为于ServerCrashProcedure不同的地方在于getRegionsOnCrashedServer方法：

如果ServerCrashProcedure.getRegionsOnCrashedServer返回空集合，HBCKServerCarshProcedure会scan读meta表，将meta表上记录的Opening和Opened两种状态的region返回，另外将Closing状态的region改为Close

---

## 状态流转 executeFromState

### 准备工作

1. updateProgress 更新状态
2. 将当前处理的rs加到 DeadServer processing list ，在SCP执行结束之后，才加到 DeadServers list。每个状态都会检查下，是否加进去了。

```
case SERVER_CRASH_START:
case SERVER_CRASH_SPLIT_META_LOGS:
case SERVER_CRASH_DELETE_SPLIT_META_WALS_DIR:
case SERVER_CRASH_ASSIGN_META:
break;
default:
// If hbase:meta is not assigned, yield.
if (env.getAssignmentManager().waitMetaLoaded(this)) {
    throw new ProcedureSuspendedException();
}
```

这里貌似有问题，


![SCP 状态图](/blog/images/SCP/SCP.png)