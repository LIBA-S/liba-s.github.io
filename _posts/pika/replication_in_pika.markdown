---
layout: post
title: log in pika
date: '2021-06-10 14:15'
tags: []
---

## 1. 架构

![](/public/images/pika/pika-arch.jpg)

**Replication Manager**位于Pika实例中，负责分片数据的同步。主要包括以下几个层次：
- **服务接口层**
主要负责管理本实例拥有的所有复制组，为上层应用提供复制组CRUD的API接口，同时协调传输层为消息处理层提供数据传输通道
- **消息处理层**
接收来自服务接口层转发的消息，根据当前的部署类型（经典主从模式或集群模式）的不同，分别进行特定的消息处理
- **传输层**
负责对接和本地复制组相关的所有副本实例，传输和接收副本同步所需的所有消息

![](/public/images/pika/replicationManager.jpg)

## 2. 代码分析
### 2.1 设计架构

主要类图设计如下所示：

![](/public/images/pika/replicationArch.svg)

主要组成部分：

**ReplicationManager**： 上层应用（pika server）负责调用相关接口，管理复制组信息（增删查改）。
其中部分接口是经典模式和集群模式共用的，另外存在一些接口需要子类实现。比如成员变更相关的接口，经典
模式并不支持；而SetMaster之类的接口，需要经典模式实现，而集群模式并不支持。该类包含一些基础设施，
主要为Transporter，Executor和ReplicationGroupNode的集合，其中Transporter主要用于和对端传输数
据。Executor本质是一个线程池，当接收到对端发送过来的数据同步请求，会在Executor中进行处理迭代。
ReplicationGroupNode则负责具体的复制组信息的管理。

**ClassicReplManager**：继承自**ReplicationManager**，实现经典模式的特有接口，同时包含一个
**MasterSlaveController**类，该类负责管理经典模式中的主从关系。

**ClusterReplManager**：继承自**ReplicationManager**，实现集群模式的特有接口。包含一个TimerThread
数据成员，TimerThread负责管理复制组的定时器，定时器超时之后，将在Executor中执行相关处理逻辑。

**MasterSlaveController**：负责管理经典模式的主从关系，内部包含一个简单状态机，当节点为从节点
时（上层调用slave of xxx），启动该状态机，负责建立主从关系，并负责链接超时断连之后的重建。上层可
以通过Info接口，得到目前的主从关系。

**ReplicationGroupNode**：特定复制组（table + slot）在本实例中的唯一代表。经典模式中，不存在分片，
则一个DB（redis DB）对应一个ReplicationGroupNode；集群模式中，存在数据分片，每个DB中的每个分片都
各自对应一个ReplicationGroupNode。上层（pika server）可以通过Propose接口，提交一个操作日志。该类
负责将操作日志持久化的磁盘（写入binlog），同时复制给复制组的其他成员。由于经典模式是异步复制，没有
一致性保证，所有不同的部署模式，具体的接口实现不同。

**Transporter**：负责replication数据的交互，每个对端实例（ip + port）对应一个Peer实例。每个Peer
实例有两条链接，一条是本地主动发起的链接（ClientStream），一条是对端主动发起的链接（ServerStream）。
每条链接上的消息类型，根据具体的部署模式（经典/集群）可能不同。

**ClassicRGNode**: 负责经典模式**ReplicationGroupNode**的实现。主要负责：接受上层应用的的日志写入请求
的数据落盘和异步同步给从节点；接受来自对端的日志写入请求并落盘。其中主要的数据成员为：MasterContext，
SlaveContext，ClassicProgressSet和ClassicLogManager。MasterContext和SlaveContext主要负责处理数据
同步消息，其中SlaveContext存在一个状态机，当节点为从节点时生效（并且已经通过**MasterSlaveController**
建立好了主从连接）；ClassicProgressSet主要负责维护当前复制组的各个成员之间日志同步状态。ClassicLogManager
则负责管理日志信息的读取和写入。

**ClusterRGNode**: 负责集群模式**ReplicationGroupNode**的实现。职责和**ClassicRGNode**相同。集群模式
的日志复制能够通过Raft协议，提供强一致性保障。和**ClassicRGNode**相似，也存在负责管理复制组成员同步状态
的ClusterProgressSet，以及负责日志信息读取和写入的ClusterLogManager。另外，还存在负责管理Raft状态机的
一些成员，比如控制Raft状态转换的定时器。

### 2.2 代码结构

相关目录结构如下图所示：

![](/public/images/pika/replication代码目录.PNG)

主要包括以下目录：

**replication**：日志复制模块的主目录，包括一些基础设施接口和基类，向上层（pika server）屏蔽经典和集群
模式，通过**ReplicationManager**（pika_repl_manager.h）和**ReplicationGroupNode**（pika_repl_rg_node.h)
提供的接口，提供信息交互的接口。其中，pika_memory_log.h, pika_stable_log.h和pika_log_manager.h定义了
**LogManager**日志信息读取和写入的相关接口。pika_repl_auxiliary.h, pika_repl_io_stream.h, pika_repl_io_thread.h和
pika_repl_transporter.h定义了**Transporter**数据传输相关的接口。pika_repl_progress.h定义了**ProgressSet**
复制组成员同步状态的相关接口。pika_repl_message_executor.h定义了**MessageSender**数据发送相关结构。

**replication/classic**: 经典模式相关逻辑的实现。pika_classic_controller.h实现了**MasterSlaveController**。
pika_classic_io_stream.h定义了经典模式中，数据传输连接的消息处理逻辑，主要负责协议数据包的解析。pika_classic_log_manager.h
则负责实现了**ClassicLogManager**。pika_classic_message_executor.h则负责定制化经典模式的日志发送时的封包逻辑，同时
定义了如何处理接收到的日志复制消息。pika_classic_progress.h定制化了经典模式下的**ProgressSet**。pika_classic_repl_manager.h
定制化了经典模式下的**ReplicationManager**。pika_classic_rg_node.h则定制化了经典模式下的**ReplicationGroupNode**。

**replication/cluster**: 集群模式相关逻辑的实现。和经典模式下的布局相似，也针对**replication**下定义的接口，做了相关的定制化。
pika_cluster_io_stream.h定义了集群模式中，数据传输连接的消息处理逻辑，主要负责协议数据包的解析。pika_cluster_log_manager.h
则负责实现了**ClusterLogManager**。pika_cluster_message_executor.h则负责定制化集群模式的日志发送时的封包逻辑，同时
定义了如何处理接收到的日志复制消息。pika_cluster_progress.h定制化了集群模式下的**ProgressSet**。pika_cluster_repl_manager.h
定制化了集群模式下的**ReplicationManager**。pika_cluster_rg_node.h则定制化了集群模式下的**ReplicationGroupNode**。此外，
pika_cluster_raft_timer.h定义了Raft定时器相关的处理接口。

**storage**：存储binlog相关的存储和检索逻辑。**replication** 模块下的**StableLog**（定义于pika_repl_stable_log.h)通过封装本目
录下的相关接口，实现binlog日志的落盘和读取。

**util**：本目录定义了一些基础设施，包括MPSC无锁队列，定时器，线程池等。

### 2.3 经典模式
#### 2.3.1 主从关系建立
![](/public/images/pika/classic_state_machine.svg)

经典模式下，主要存在两层状态机：
(1) **MasterSlaveController**：建立Pika实例之间的主从关系。通过**ClassicReplManager**提供的SetMaster接口驱动。当成功建立主从关系后，从节点会通过SetMasterForRGNodes接口，使能ClassicRGNode的状态机。

(2) **ClassicRGNode**：建立复制组之间的主从关系。经典模式下，一旦实例之间的主从关系成功建立，从节点的状态机将开始生效，即从节点
实例中的所有复制组都将激活其SlaveContext。每个复制组单独建立自身的主从关系。

**特别说明：**
为了兼容旧版本**sharding**模式中**consensus_level = 0**的情况，也将其实现归类为经典模式。通过**MasterSlaveController**指针是否
为空作区分。和上诉主从关系建立不同，此种情况下不需要保证Pika实例之间的主从关系，所以SetMaster接口可以直接触发第二层状态机。

整个主从关系建立的代码执行路径为：

![](/public/images/pika/classic_master_and_slave.svg)

调用方使能第一层状态机之后，即可返回。后续状态机的迭代在辅助线程中完成。

#### 2.3.2 日志写入
#### 2.3.2.1 主节点
上层首先通过ReplicationGroupID（TableID和SlotID组成）获取到**ClassicRGNode**，接着通过调用**ClassicRGNOde**的Propose接口
提交一个ReplTask任务。首先调用**ClassicLogManager**的AppendLog接口，将操作记录写入binlog，然后迭代自身的点位信息。

```c++
Status ClassicRGNode::Propose(const std::shared_ptr<ReplTask>& task) {
  if (!started_.load(std::memory_order_acquire)) {
    return Status::Corruption("proposal dropped when the node has not been started");
  }
  LogOffset log_offset;
  Status s = master_ctx_.Propose(task, log_offset);
  if (!s.ok()) {
    return s;
  }
  persistent_ctx_.UpdateCommittedOffset(log_offset);
  persistent_ctx_.UpdateAppliedOffset(log_offset, false, true /*ignore_applied_window*/);
  return s;
}

Status MasterContext::Propose(const std::shared_ptr<ReplTask>& task, LogOffset& log_offset) {
  Status s = options_.log_manager->AppendLog(task, &log_offset);
  if (!s.ok()) {
    return s;
  }

  StepProgress(options_.local_id, log_offset, log_offset);
  options_.rm->SignalAuxiliary();
  return s;
}
```

**特别说明**：
经典模式中，我们仍然记录了**PersistentCtx**,引入了集群模式一致性协议中的点位信息：**committed， applied 和 snapshot**。

(1) 引入**committed**目的：本身没有太多意义，旨在同集群模式复用一些相关数据接口/逻辑。

(2) 引入**applied**的目的：使从节点能够具有一定的故障恢复能力。当前实现中，写完binlog之后，写入DB之前宕机，这些数据将丢失。
引入点位信息之后，**applied** 之后的数据在故障恢复阶段都需要再次应用到DB。但这可能会引入一个重复执行的新问题（写完DB，但是更新
点位信息失败）。问题的本质是操作并非原子，后续可以将写入DB和更新点位操作，在事务中完成（需要pika支持事务，点位信息也需要保存在DB引擎中）。

(3) 引入**snapshot**点位的目的：解决全同步之后，可能无法建立主从关系的问题。考虑以下场景：A是主，B是从。由于数据
差距较大，B节点全同步自A，然后B节点宕机，紧接着写入部分数据之后，新加入C节点全同步自A。然后，A节点宕机，C节点选为主，B节点重启，并试图同C
节点建立主从关系，故将自身点位发送给C节点，然而该点位对应的数据在C中可能是以padding填充的无效数据，C节点读取该点位数据，导致读取失败，
进而无法成功建立主从连接。造成该问题的根本原因是，C节点无法判断点位对应的数据是否是padding数据（全同步点位之前的数据以padding填充）。引入
**snapshot** 点位，即可提供该信息，每次从节点试图同步点位信息时，都将首先判断该点位对应信息是否在 **snapshot**之前，如是，则建立全同步。

#### 2.3.2.2 从节点

从节点收到主节点发送过来的日志数据之后，首先通过ReplicationGroupID进行哈希路由，将相同复制组的日志消息放到同一个线程处理。具体的处理逻辑
在**ClassicRGNode**中处理。大致流程为：
(1) 解析数据包，通过**ClassicLogManager**的AppendReplicaLog接口，将日志写入binlog，同时内存中保存一份，等待后续写入DB。
(2) 若数据成功写入binlog，则通过TryToScheduleApplyLog，调用OnApply接口，将日志传递给上层写入DB。
(3) 上层完成写入之后，需要调用Advance接口，更新**applied**点位信息。

处理**snapshot**的相关路径比较相似，主要不同点在于，快照数据是通过Rsync进行传输的，**ClassicRGNode** 在等待快照数据完成过程中，会一直处于
**WaitSnapshotReceived**状态，第一次进入该状态，将包装一个延迟任务，异步等待数据传输完成。一旦快照数据接收完成，任务处理函数将调用Advance报告
快照数据处理结果，进而迭代SlaveContext状态机。

```c++
void ClassicRGNode::HandleBinlogSyncResponse(const PeerID& peer_id,
                                             const std::vector<int>* index,
                                             InnerMessage::InnerResponse* response) {
  if (slave_ctx_.HandleBinlogSyncResponse(peer_id, index, response)) {
    TryToScheduleApplyLog(log_manager_->last_offset());
  }
}
bool SlaveContext::HandleBinlogSyncResponse(const PeerID& peer_id,
                                            const std::vector<int>* index,
                                            InnerMessage::InnerResponse* response) {
  SetReplLastRecvTime(slash::NowMicros());

  bool append_success = false;

  ...

  for (size_t i = 0; i < index->size(); ++i) {
    InnerMessage::InnerResponse::BinlogSync* binlog_res = response->mutable_binlog_sync((*index)[i]);
    // if pika are not current in BinlogSync state, we drop remain write binlog task
    if (ReplCode() != ReplStateCode::kBinlogSync) {
      return append_success;
    }
    if (ReplSession() != binlog_res->session_id()) {
      SetReplCode(ReplStateCode::kTrySync);
      return append_success;
    }
    // empty binlog treated as keepalive packet
    if (binlog_res->binlog().empty()) {
      continue;
    }
    std::string binlog = std::move(*(binlog_res->release_binlog()));
    BinlogItem::Attributes binlog_attributes;
    if (!PikaBinlogTransverter::BinlogAttributesDecode(binlog, &binlog_attributes)) {
      SetReplCode(ReplStateCode::kTrySync);
      return append_success;
    }
    // Append to local log storage.
    std::string binlog_data = binlog.erase(0, storage::BINLOG_ITEM_HEADER_SIZE);
    Status s = options_.log_manager->AppendReplicaLog(peer_id, std::move(binlog_attributes),
                                                      std::move(binlog_data));
    if (!s.ok()) {
      options_.rm->ReportLogAppendError(options_.group_id);
      SetReplCode(ReplStateCode::kTrySync);
      return append_success;
    }
    if (!append_success) {
      append_success = true;
    }
  }

  LogOffset ack_end;
  if (only_keepalive) {
    ack_end = LogOffset();
  } else {
    LogOffset productor_status;
    // Reply Ack to master immediately
    options_.log_manager->GetProducerStatus(&productor_status.b_offset.filenum,
                                            &productor_status.b_offset.offset,
                                            &productor_status.l_offset.term,
                                            &productor_status.l_offset.index);
    ack_end = productor_status;
    ack_end.l_offset.term = pb_end.l_offset.term;
  }

  SendBinlogSyncRequest(ack_start, ack_end);
  return append_success;


void ClassicRGNode::TryToScheduleApplyLog(const LogOffset& committed_offset) {
  ...
  Status s = ScheduleApplyLog(updated_committed_offset);
}

Status ClassicRGNode::ScheduleApplyLog(const LogOffset& committed_offset) {
  // logs from PurgeLogs goes to InternalApply in order
  std::vector<MemLog::LogItem> logs;
  Status s = log_manager_->PurgeMemLogsByOffset(committed_offset, &logs);
  if (!s.ok()) {
    return Status::NotFound("committed offset not found " + committed_offset.ToBinlogString());
  }
  for (const auto& log : logs) {
    persistent_ctx_.PrepareUpdateAppliedOffset(log.GetLogOffset());
  }
  fsm_caller_->OnApply(std::move(logs));
  return Status::OK();
}

Status ClassicRGNode::Advance(const Ready& ready) {
  if (ready.db_reloaded && slave_ctx_.ReplCode() == ReplStateCode::kWaitSnapshotReceived) {
    if (!ready.db_reloaded_error) {
      persistent_ctx_.ResetPerssistentContext(ready.snapshot_offset, ready.applied_offset);
      meta_storage_->ResetOffset(ready.snapshot_offset, ready.applied_offset);
      log_manager_->Reset(ready.snapshot_offset);
    }
    // After Snapshot reloaded, we should step the state_machine of slave context.
    slave_ctx_.HandleSnapshotSyncCompleted(!ready.db_reloaded_error);
  } else if (!ready.applied_offset.Empty()) {
    persistent_ctx_.UpdateAppliedOffset(ready.applied_offset);
  }
  return Status::OK();
}
```
### 2.3 集群模式
