---
title: chainlink
categories:
  - eth
date: 2020-12-17 15:20:19
tags:
---

## Chainlink

chainlink提供在以太坊上与外界沟通的功能。

出发点大概是要往合约中更新数据，使用多个节点一起更新，达到最少个数的阈值后，便取这些节点提交上来的中位数作为本次更新的结果。

然后再进行抽象化实现通用性，通用包括节点的数据从哪里来，请求数据的方式等；

### V0.4

在v0.4的版本里，采用的是，需要有专门一个人发起交易来请求新数据。

oracle为节点操作地址（节点监听请求触发获得数据并提交新数据交易），`Job ID`为数据请求时不同方式的适配器。比如在Rinkeby上`GttpGetJsonParseEthBytes32`的`Job ID`为`b0bde308282843d49a3a8d2dd2464af1`[参考自](https://docs.chain.link/docs/testnet-oracles)；在发起请求的时候，`Job ID`是`b0bde308282843d49a3a8d2dd2464af1`就代表要请求的数据将使用`Http Get`的方式，且返回数据的解析方式为`Json`，具体的数据将会为`string`，随后发起交易时转为bytes32格式；



通过继承`ChainlinkClient`合约，可以[自定义请求](https://docs.chain.link/docs/request-and-receive-data)，如想要的天气等等，只要在合约里定义请求的路径，方式，返回数据path等等，再挑选适配的节点Oracle地址及对应的Job ID。就可以完成了。



每个节点都会有一个oracle地址，以及支持的Job ID,  Job ID通用的就是上面的示例，也有特定的，比如[CoinMarketCap ETH-USD](https://market.link/jobs/edd1b403-0273-45ce-9e17-49b9e7d90d90)，这些特定的Job ID就不用特定Request的url等。



在[Chainlink Market](https://market.link/search/nodes)中，可以看到全部的节点oracle 地址，以及他们支持的Job ID



#### 示例

在[官网示例对AUD/USD](https://feeds.chain.link/aud-usd)里，Aggregator合约[地址](https://etherscan.io/address/0x05cf62c4ba0ccea3da680f9a8744ac51116d6231#readContract)，某一个节点的`oracle`[地址](https://etherscan.io/address/0x29e3b3c76e7ae0d681bf1a6bcee1c0e7d17dbaa9); ，Request的时候需要遍历参与的节点地址们，给每一个节点都发一个Request（一次更新请求在oracle中会存在多个Request记录）。

- 在某个时刻，有人发起了一笔`requestRateUpdate`的[交易](https://etherscan.io/tx/0x1a8d9ef0971b884c76e630f29da1456842325f4cf582bc99c0a4e8d521c14efe)

  通过查看这个交易的 event事件，本次的RequestId有许多个，（RequestID是由Job Id/节点地址/<font color=red>回调方法</font>签名等多个信息的Hash），相当于一个节点会有一个RequestId,依次分别为

  - `0xb74533b2e6e5f876ad70dd2bcffc6c9e1a283a8a38d0f8adac74679fbe906ac5`
  - `0x6c0ee538c9b2cde6850449fff6ecc54d6f72215addcc10b3e793851f04d9452e`
  - `0xf3b2f5647cbd18aae04e2bb96866dea4ed29b6835f71768e91dd61bf205abfba`

​          等...



- 在一个节点的`oracle`的合约交易中，找到了他回复数据的这条[交易](https://etherscan.io/tx/0xcd0b8368cac6072b8bb0339a9f111e5c721038717ffb533616545a2658a7adeb)

  回复数据将调用`fulfillOracleRequest`方法，通过InputData可以看到回复的RequestId刚好是我们上面的第一个ID，在这个方法中，将会回调上面提到的标红的<font color=red>回调方法</font>



- 在回调方法里，其实就是统计各个节点回复上来的数据。当回复的个数超过阈值后，就根据这些数据去一个合适的值(如中位数)来作为本次更新的结果；



大致这样就算完成了。



#### V0.6

在0.6的版本里，可以有许多人来发起新的一轮，当然节点们自然也可以，新的一轮的规则可以是基于时间等，每次新的一轮开始节点们再针对新的一轮进行数据的更新。

这个更新采用的是FluxAggrator，相关的[资料](https://medium.com/@chainlinkgod/scaling-chainlink-in-2020-371ce24b4f31)。

但是这个里面，oracle是操作的节点地址，v0.4里的指的是节点的oracle合约地址；既没有oracle合约地址，也没有job id,  似乎不可以通用。
