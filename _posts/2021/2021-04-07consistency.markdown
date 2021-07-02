---
layout: "post"
title: "raft in etcd"
date: "2020-09-10 14:15"
---

## 一致性模式

### 顺序一致性 （Sequential Consistency)

- 应该提供一个“单实例”的视图
- 读返回**最近**的写入值，忽略客户端
- 所有的**后续读**返回相同的值，直到下一次写入

**最近** && **后续度** ： 相同客户端，由**program order**决定（排序）；不同客户端，可以**reorder**

全局视角：
1. 来自同一客户端请求，执行顺序被保留（program order）。
2. 所有客户端观察到相同的全局顺序。

![](/public/images/sequential consistency.png)

### 线性一致性 （linearizablility）

- 应该提供一个“单实例”的视图
- 读返回**最近**的写入值，忽略客户端
- 所有的**后续读**返回相同的值，直到下一次写入

**最近** && **后续度** ： 由**time**决定（排序）

![](/public/images/linearizablility.png)

### 静态一致性 （quiescent consistency）

- 应该提供一个“单实例”的视图
- 读返回**最近**的写入值，忽略客户端
- 所有的**后续读**返回相同的值，直到下一次写入

**最近** && **后续度** ： 相同客户端，由**program order**决定（排序）；不同客户端，非重叠由**time**排序，
重叠，可以**reorder**
