---
title: fabric_pbft
categories:
  - fabric
date: 2019-4-14 15:06:52
tags:
---

#### fabric consensus pbft VS neo dbft



#### pbft

**pbft分为3个阶段，pre-prepare, prepare, commit**。

一直很好奇为什么是3个阶段，为什么这三个阶段就能保证拜占庭呢？

- 首先， pre-prepare阶段是主节点发给各个从节点的，pre-prepare消息中包括共识消息、序号等
- 从节点收到pre-prepare消息后，发其他节点广播prepare消息，这是为了告诉别人自己收到的消息是什么
- 从节点收到有效的prepare消息足够多(f+1)，代表有f+1个节点收到的消息是一样的，故**证明消息的真实可信性**。发出commit，commit为自身签名消息，代表自己同意消息。
- 只要有1个节点收到的commit消息数量有f+1，则可以确认消息成功。



#### dbft

dbft中，d是授权的意思，授权拜占庭，这里忽略验证人的选择，仅仅就共识过程进行讨论。

**共识过程只有两个阶段，prepare, prepare-response**

- 首先，prepare阶段，主节点发给各个从节点的，prepare消息中包括共识消息、序号等
- 从节点收到prepare消息后，就消息进行验证判断，如同意消息，则发送prepare-response消息
- 从节点收集prepare-response消息，如接收到的消息数量达到阈值，则确认共识成功



二者对比起来，很明显，neo的dbft缺少了验证消息真实可信性，即无法判断主节点发送给大家的prepare消息是相同的还是不同的。当然，因为回复的response是对消息的签名，是否可以从签名的特性出发，判断对方签名的数据是否与自身签名的数据是相同的。这一点，在代码里来看，暂时还没看到具体解释。
