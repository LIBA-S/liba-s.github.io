---
layout: "post"
title: "raft membership"
date: "2020-09-10 14:15"
---

## 成员变更

两种方式：

1. simple change
2. joint consensus

名词：

Cold：old configuration
Cnew：new configuration

## simple change

步骤：

1. Leader创建**Configuration entry**，复制给Cold。
2. 当Cold中大多数节点接收并持久化了**Configuration entry**，则配置变更成功。

一旦**Configuration entry**被执行，则新的成员配置生效。在Raft paper中，新配置生效的时间点是接收到即生效，工程实现为了简化流程，常实现为配置被执行之后，才开始生效。同时如果当前log中存在未提交的**Configuration entry**，则不能接收新的配置变更请求（防止“脑裂”产生）。


## Joint consensus

前置条件： 一旦**Configuration entry**被本地持久化，该配置即生效，无需等待提交。

**joint consensus**状态同时包括Cold和Cnew：
- log entries同时复制到Cold和Cnew
- Cold和Cnew中的任意节点都可能被选为leader
- **Leader election** 和 **log replication**都需要Cold和Cnew中的大多数同意

流程：

step1. Leader 创建 **Joint config entry**(包括Cold和Cnew)，复制给Cold和Cnew的所有节点。
step2. 当Cold和Cnew中的大多数节点同时Granted，则**Joint config entry**变为已提交，进入**joint consensus**状态。
step3. Leader 创建 **Cnew entry**，复制给Cnew中的所有节点。
step4. 当Cnew中的大多数节点Granted，则**Cnew entry**变为已提交，配置变更完成。
step5. 如果当前Leader不在Cnew中，执行Leader transter，移交权限，不在Cnew中的节点可以安全shut down。

## 安全性分析

1. step1阶段，Leader corruption

new leader可能包含，也可能不包含**Joint config entry**，但是不会出现“脑裂”的情况。假设new leader
包含**Joint config entry**，那么他必须同时得到Cold和Cnew中的大多数节点的投票，这就杜绝了Cold和Cnew中
出现双主的情况；如果new leader不包含**Joint config entry**，那么说明

一旦**Joint config entry**被提交，则说明Cold和Cnew中的大多数节点已经持久化该配置，如果这时Leader crash，new leader肯定会在包含**Joint config entry**的节点中选举产生。

在**Cnew entry**提交之前，当前leader crash，new leader可能从Cold中产生，无法做到Cnew独自服务，故而**Cnew entry**被提交是配置切换完成的事件标志。

不在Cnew中的被移除的节点Cremoved，无法收到new leader的心跳信息(new leader配置为Cnew)，可能会一致超时，并递增自己的term，干扰现有leader(Cremoved节点处于**joint consensus**状态，所以拥有新主信息)。解决方法：如果节点在**election timeout**时间内（自身未超时）收到了VoteRequest信息，则拒绝投票。
