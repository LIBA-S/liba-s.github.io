---
layout: "post"
title: "log in pika"
date: "2020-10-15 14:15"
---

## 1. Pika副本模式 (<=3.4.0)

### 1.1 主从模式

当前主从模式的搭建流程为：
1. 独立启动多个Pika instances。
2. 对某些Pika实例，调用 `Slaveof ip port` 命令，则该实例自动转为`slave`，目标实例则成为`master`。

对`master`的写请求，写入本地Binlog和DB后，即可返回处理结果给客户端，而无需等待相关信息同步给`slave`节点。主从复制是异步的，`slave`实例上的信息存在一定延迟，且当`master`宕机后，可能出现数据丢失。

若当前`master`节点宕机停止服务，则需要对`slave`执行`slaveof`命令，移除其对已宕机节点的主从关系。

### 1.2 分片模式

`分片模式`指对用户数据进行分表分片，对数据空间进行Hash切片，以分散业务压力，提高服务质量。每个数据分片采用
`主从模式`搭建副本关系。

当前`分片模式`搭建流程为：
1. 通过`pkcluster addtable`命令, 在Pika实例上创建`table`。
2. 通过`pkcluster addslots`, 在Pika实例上创建分片`slot`。
3. 通过`pkcluster slotsslaveof`命令, 建立Pika实例中分片的副本关系。

对`master`分片的写入请求，处理流程等同于`主从模式`，仍然存在宕机数据丢失风险。

若当前`master`分片所属节点宕机停止服务，则需要对`slave`分片执行`pkcluster slotsslaveof`命令，移除其对已宕机节点相关分片的主从关系。

### 1.3 一致性模式

无论是`主从模式`还是`分片模式`都属于异步复制，一旦主节点宕机，从节点数据极可能不完整，导致数据丢失。而`一致性模式`则提供了一种灵活的一致性保障，通过配置`consensus_level`参数实现，其描述了当前复制组合中，至少有`consensus_level + 1`个节点具有最新的数据。

对`master`分片的写入请求，在写入本地Binlog的同时，需要保证至少还有`consensus_level - 1`个`slave`节点也同步写入了Binlog文件，而后才能写入DB，返回处理结果给客户端。

同样，若当前`master`分片所属节点宕机停止服务，则需要对`slave`分片执行`pkcluster slotsslaveof`命令，移除其对已宕机节点相关分片的主从关系。

和上两种同步模式相比，一致性模式增加了数据的安全性。当`consensus_level = 2`时，保证了同时有3个节点都包含最新的数据，即便其中一个节点宕机了，仍然能从其余两个节点中恢复。

当前一致性模式的主要缺点有：没有`leader election`机制，导致一旦其中某个节点宕机，需要人为指定新主。

## 2. 目标

1. 抽象底层副本复制模块，统一Pika当前的多种副本模式。
2. 支持动态调整复制组成员(`membership change`)。
3. 支持复制组自动选主机制（`leader election`)。


## 3. 方案

`数据一致性`可分为：`强一致`和`弱一致`。Pika当前的`一致性模式`可能会随着副本当前的存活情况动态变化。对于五副本的数据分片，在`consensus_level = 1`情况下：若节点不可用数量小于2，则仍然能提供`强一致`保障；而一旦超过两个节点宕机，则相关数据集就无法保证一致性，这时候若继续提供服务，则可能会导致数据丢失。

`分片模式`和`一致性模式`的差别在于：`分片模式`只有master节点的数据时最新的，其余follower节点无法保证数据是最新的；`一致性模式`能够保证至少有`consensus_level`个节点的数据是最新的。从这个角度来看，我们可以认为`分片模式`和`一致性模式`本质上没有区别，都是`强一致`的系统，只是可用性有差别，**故我们将移除分片模式，而将其作为一致性模式的特殊情况考虑**。

### 3.1 主从模式
#### 3.1.1 Log replication
日志复制同`一致性模式`采用相同的基础设施实现。详见3.2.1。
#### 3.1.2 Leader election
对于`主从模式`而言，在建立主从关系之前，各个Pika实例身份是对等的，都能独自提供读写服务，直到建立主从关系之后，从节点不可写。主从角色转换，需要调用`slaveof`命令外部触发。**为了兼容Redis经典模式语义，针对`主从模式`不提供自主`leader election`机制**
#### 3.1.3 membership change
针对`主从模式`而言，副本之间需要通过调用`slaveof`命令建立主从关系，副本本身是无状态的，可以无副作用的动态扩展。
#### 3.1.4 集群搭建
同现在无差别。
![](/public/images/classic_state_machine.svg)
### 3.2 一致性模式
`一致性模式`将基于**Raft**数据一致性协议。通过不同的成员角色(`voter`，`learner`)，模拟当前版本的可用性等级（即`consensus_level`)。
- `voter`之间采用同步复制逻辑。`leader election`和`log replication`在`voter`集合中达成大多数同意，即可提交。
- `learner`采用异步复制逻辑，不参与大多数投票，等价于`主从模式`的`slave`。
- `leader`在`voter`中投票产生，`learner`不参与选举。

当前版本的`consensus_level`对应于`voter`的quorum数量。当前版本的`分片模式`(`consensus_level = 0 `)，等价于`一致性模式`中，`number of voter = 1`的情况。

#### 3.2.1 Log replication

Pika实例之间的架构关系如下图所示，`Replicate Manager`负责副本之间的数据同步。

![](/public/images/pika-instance-infra.svg)

`Replicate Manager`的组件如下图所示，可以分为两个层次：`消息处理层`和`数据传输层`。

`消息处理层`是一个中间层，负责处理上层应用和对端实例传递过来的消息。按照信息的来源，可以分为两类：

1. 上层应用传递的客户端操作信息（命令）。
2. 对端实例发送过来的消息。
2.1 kMsgHeartbeat && kMsgHeartbeatResp（心跳消息）
2.2 kMsgAppend && kMsgAppendResp（日志复制消息）
2.3 kMsgSnapshot (快照消息)
2.3 kMsg(Pre)Vote && kMsg(Pre)VoteResp（选举消息）
2.4 kMsgTimeoutNow（强制超时消息）

`数据传输层`负责同对端实例进行日志消息交互。发送来自`消息处理层`的信息到对端，或转发对端传递过来的消息给`消息处理层`进行处理。

![](/public/images/replicateManager.jpg)

`Replicattion Manager`关键类之间的关系，如下图所示。`数据传输层`和`消息处理层`通过MessageReporter和Transporter接口进行交互。

`消息处理层`通过ReplicationManager管理整个实例所有复制组的日志管理，其中每一个复制组唯一对应一个ReplicationGroup实例。
LogStorage接口抽象日志的持久化和检索接口。StateMachine接口需要上层应用实现，每当日志正确处理之后，需要调用
StateMachine接口进行应用的状态更新（执行日志对应的Pika命令，返回命令处理结果给客户端）。而节点状态机模块，则
依赖于当前配置的副本模式；具体而言，`主从模式`对应ClassicNode实现；
`一致性模式`对应ConsensusNode实现。而Progress负责管理复制组对端实例的日志同步状态。

![](/public/images/architecture.svg)
#### 3.2.2 Leader election
节点状态机转换如下图所示：

![](/public/images/consensus_state_machine.svg)

`leader`的选举基于**Raft**协议实现。
#### 3.2.3 Membership change
在`强一致`的前提下，`一致性模式`是有状态的，副本节点需要拥有复制组的视图，即成员关系需要持久化，并且保持一致。
故而在没有Leader的情况下，无法动态调整复制组的成员；在存在Leader的情况下，可以动态调整复制组成员。

支持变更类型：
- AddNode：添加投票节点
- AddLearner：添加学习节点
- RemoveNode：移除投票节点
- PromoteLearner：提升学习节点为投票节点

configuration表示当前复制组的成员配置。
**1. configuration 什么时候生效？**
  当配置信息在大多数节点中成功持久化并提交之后，新的configuration才生效。
**2. 对新集群而言，提供的configuration是否需要持久化？**
  新节点加入新集群时，本地持久化，强制提交并应用configuration。
**3. configuration不满足大多数需求时，但仍然需要提供服务，如何强制reset？**
  假设原本存在5个投票者（A,B,C,D,E)，其中（C,D,E)宕机，只存活(A，B)，此时无法选主。
  这种情况下只能在剩余节点（A,B)上强制提交并应用新的配置(Add A&&B, Remove C&&D&&E)。
  此后，新集群将只包含（A,B)，即可选择新主。
  **注意：任何一个节点都可能缺失已提交的数据，不满足leader completeness。**
  **即可能会造成数据丢失，即便选择了当前log index最新的节点**
  **注意：为了防止脑裂，集群中只能存在一个未提交的ConfigChangeEntry**

#### 3.2.4 集群搭建

`PikaManager`用户侧接口：
1) 首先调用 **CreateTable [table_name] [partition_num] [consensus_level] [replication_num]** 初始化复制组。
   pika cluster将自动根据`consensus_level`和`replication_num`构建复制组，决定复制组中成员组成（`voter`和`learner`数量）。
2）通过调用 **AddMember [table_name] [partition_id] [voter|learner] [ip_port]** 新增复制组的副本成员。
3）通过调用 **RemoveMember [table_name] [partition_id] ip_port** 移除指定复制组的副本成员。
3）通过调用 **ResetCluster [table_name] [partition_id] ip_port [ip_port ...]** 以指定成员强制重新构建复制组。
   当复制组无法满足quorum的情况下（即投票者多于半数节点无法再提供服务），需要调用。

为了实现`PikaManager`的相关功能，需要实现一组内部API接口。

对于新建的副本结点，节点以`uninitialized`状态启动，无任何信息，无法向外提供服务，对复制组无感知。
等待`PikaManager`通知集群的配置信息(`configuration`)。一旦接受到相应通知，则切换为`follower`状态
进行常规的**Raft状态转移**。

管理员，可以向集群Leader下发以下命令，修改集群成员配置。这些命令将构建
`ConfigChangeEntry`，添加到log storage中，并复制给Followers，一旦这些
日志记录被提交和引用，则新的集群配置即可生效。

```c++
bool add_node(clusterId， nodeId node, int role);
bool remove_node(clusterId, nodeId node);
bool promote_node(clusterId, nodeId node);
bool transfer_leader(transfee node);
```

## 4. 线程模型

Pika处理客户端请求的线程模型示意，如下图所示。

按照线程的功能，主要分为四类：
1. 接收客户端请求的ClientThreadPool。
2. 负责迭代ReplicateManager状态的MsgProcessTheadPool。
3. 负责应用日志的ApplyThreadPool。
4. 负责定时任务的TimerThread（心跳超时，选举超时和切主超时）。

在具体实现中，基于现有的线程模型，ApplyThreadPool可以复用ClientThreadPool。

![](/public/images/leader-thread-model.svg)

## 5. 测试
### 5.1 Transporter
| case  | result  |
|----|----|
| Peer is unreachable  |   |
### 5.2 Log storage
| case  | result  |
|----|----|
| recover unapplied logs   |   |
| get log_term with specific index  |   |
| get first_offset after latest snapshot   |   |
| get last_offset  |   |
### 5.3 Log replication
| case  | result  |
|----|----|
| Logs in leader are purged   |   |
| Logs in leader are matched   |   |
| Logs in leader are conflicted   |   |
| concurrent append entries  |   |
| snapshot with matched data & meta  |   |
| snapshot with unmatched data & meta   |   |
| concurrent snapshots   |   |
| Disk is corruption (no-free, io Error)   |   |
### 5.4 Leader election
| case  | result  |
|----|----|
| election timedout  |   |
| Receive PreVote msg with lower term   |   |
| Receive PreVote msg with higher term  |   |
| PreVoteStage is timedout   |   |
| PreVoteRequest is granted   |   |
| PreVoteRequest is rejected   |   |
| Receive Vote msg with lower term   |   |
| Receive Vote msg with higher term   |   |
| VoteStage is timedout   |   |
| VoteRequest is granted   |   |
| VoteRequest is rejected   |   |

### 5.5 Leadership change
| case  | result  |
|----|----|
| leadership change   |   |
| concurrent leadership change   |   |
| leadership change timedout    |   |

### 5.6 Membership change
| case  | results  |
|----|----|
| add voter |   |
| add learner  |   |
| remove member   |   |
| promote learner   |   |
| concurrent membership change |   |
| reset cluster   |   |

### 5.7 Other raft events
| case  | result  |
|----|----|
| heartbeat timeout   |   |
| peer is unreachable   |   |

### 5.8 Application StateMachine
| case  | results  |
|----|----|
| apply client requests  |   |
| apply leader requests   |   |
| apply confchange entries  |   |
| apply dummy entry (ghost entries)   |   |
| apply self-produced entries   |   |

## 6. 目录信息

| 配置选项  | 含义  | 默认值|
|----|----|----|
| db_path  |  存放rocksdb数据 | 经典模式：./db/db[0-7]/[strings|lists|hash|zest|set] 集群模式: ./db/db[0-7]/partition/[strings]|
