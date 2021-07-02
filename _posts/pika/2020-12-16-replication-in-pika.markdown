---
layout: "post"
title: "replication in pika"
date: "2020-10-15 14:15"
---

## 背景和目标
### 背景
已知`分片模式`将采用`Raft`协议实现，副本数据复制的一致性，通过加入`Learner`角色，实现强一致性
下的可用性控制。
所以`分片模式`需要实现`log replication`，`leader election`和`membership change`三大功能。
### 目标
在兼容当前`经典模式`的前提下，尽可能利用`分片模式`的基础设施，即Raft相关实现的基础设施。
## 方案
在兼容`经典模式`语义的前提下，无法实现`leader election`和`membership change`功能。所以我们能够利用的基础设施即`log replication`。

### log replication

`经典模式`和`分片模式`日志复制处理流程可以完全兼容：投递提案，等待提案提交，日志应用。

两者唯一不同的地方，即是主从间数据传输的方式：
- `经典模式`仍然兼容现有的“MetaSync，TrySync，BinlogSync和TryDBSync”等数据交互接口，及其配套的数据传输格式（InnerMessage PB），以及相关的Follower sync state。
- `分片模式`则采用Raft paper的传输方式，维持**leader replicate to follower**的数据传输方向。
  非`leader`主动发起的请求，只包含**超时选举**和**客户端请求代理**。

为了实现上述目标，需要做到：
- `leader`的**数据准备**和**数据传输**两个阶段解耦合，日志**数据准备**阶段可以复用。
- 能够实现不同的**数据传输**逻辑，同时维护相关状态。可以通过配置，定制数据传输的方式。

### 架构

![](/public/images/architecture.svg)

`RG`代表整个复制组，负责协调管理当前数据分片（`经典模式`可认为只有一个分片）的所有日志复制工作。

`LogManager`模块负责维护日志文件的读写，以及相关状态的持久化工作。
`ProgressSet`模块负责跟踪`follower`节点的日志同步状态。
`Transporter`模块负责连接建立和数据传输。
上层应用通过`StateMachine`获取已提交而待应用的日志，更新DB数据。

以上模块作为**基础设施**，可以被`经典模式`和`分片模式`复用。

而`经典模式`和`分片模式`各自拥有自己的**Context**，维护各自的日志同步状态机，通过重写`StepRGMachine`，迭代各自状态机。
两种模式具有相同的日志复制角色（**leader**，**follower**和**learner**），但具有不同的状态函数。
比如，`经典模式`的**follower**负责发起同步请求；而`分片模式`的**follower**则被动接受**leader**的日志信息。

### 优缺点

优点：两种模式处理流程相互独立，仅复用相关基础设施。便于理解和分析。
