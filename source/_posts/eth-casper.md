---
title: eth_casper
categories:
  - eth
date: 2019-10-14 14:51:46
tags:
---

## Casper的基本认识



#### Casper是以太坊基于POS的研究项目，Casper项目其实包含两个研究项目

- Casper the Friendly Finality Gadget（FFG）
- Casper the Friendly GHOST: Correct-by-Construction（CBC/TFG）



POS算法实现主要可分为两个方向，如下

- 基于链，如 the Friendly GHOST，（理解中基于链意味可是可分叉的）
- 基于BFT，如Tendermint

Casper TFG是链和BFT二者混合，也是POW+POS混合。其中，POW完成大部分协议，POS定期验证检查点。



[关于casper分片](https://ethfans.org/posts/Vitalik-on-the-first-test-of-sharding),[casper交联](https://ethresear.ch/t/cross-links-between-main-chain-and-shards/1860),[ppt](https://ethfans.org/posts/you-want-to-be-casper-sharding-validator)

#### Casper vs Tendermint

第一个真正提出将BFT研究应用到PoS公有区块链环境中是Jae Kwon，他在2014年创造了**Tendermint**。

**Tendermint**的设计决策确实是把安全性和不可改变性地位放在了灵活性之上。在现实世界上有相当高的可能性是，系统真的会停止运行，参与者将会需要在协议外组织在某种软件上更新后重启系统。

Tendermint的明确属性

可证明的活跃性

安全阈值：1/3的验证者

公有/私有链相容

即时的最终确定性：1-3秒，取决于验证者数量

一致性优先

在弱同步性网络的共识安全

**Casper**的PoS提议机制与Tendermint提议机制最大的区别是相比较伪随机选择领导者，前者的验证者可以基于自己见到的块提出块。

**Casper**提供的一个独特功能是参数化安全阈值。Casper的设计目标是在网络维持PoS低开销的时候能够允许验证者选择自己的容错阈值。

**Casper**对 **Tendermint**的核心优势在于网络随时可以容纳一定数量的验证者。因为Tendermint中的区块在创建的时候需要最终化，所以区块的确认时间应该短一点。为了达到短区块时间，Tendermint PoS能够容纳的验证者数量就需要有个限制。

**Casper**的明确属性

可用性。Casper的节点在它们达成共识之前可以块分杈

异步安全性

生存。Casper的决策可以在部分同步中存活，但是不能在异步中存活

卡特尔阻力。Casper的整个前提是建立在抵制寡头垄断攻击者基础之上，因此不会有任何勾结的验证者可以超越协议

安全性。取决于每个验证者的评估安全阈值
