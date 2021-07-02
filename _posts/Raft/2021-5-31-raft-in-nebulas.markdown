---
layout: "post"
title: "raft in markdown"
date: "2020-09-10 14:15"
---

## 代码结构



- api
- example
- pkg
- raft
- rafthttp
  传输层代码，
#### raftnode
#### raftrelay
#### serverpb
#### statemachine
#### wal

## StateMachine

当日志提交之后，由StateMachine负责数据的持久化。

接口定义位于文件**/raftnode/state_machine.go**， 目前仅实现基于BoltDB的应用状态机（位于文件**/statemachine/state_machine.go**。
