---
title: cosmos-ibc-store
categories:
  - cosmos
date: 2019-11-25 14:18:49
tags:
---

## Cosmos-ibc存储

cosmos ibc是实现跨链的协议。其中会涉及到一些数据存储，这里对相关存储进行归纳。

存储使用key-value形式。

cosmos-sdk版本为fedekunze/ibc - 94ffaeb9f746d13b68678423d4aee0828f652fd4



+ Client。

| Key                              | Value          | 备注                                                   |
| -------------------------------- | -------------- | ------------------------------------------------------ |
| client/`clientID`/roots/`height` | Hash-root      | 用于merkle树验证的数据结构                             |
| client/`clientID`/consensusState | consensusState | 另一条链的共识状态，包含root、验证节点、区块高度等信息 |
| clients/`clientID`/state         | bool           | clientID是否由于恶意而被冻结(不可用)                   |
| clients/`clientID`/type          | []byte         | 另一条链的共识算法(tendermint)                         |

client用于验证/存储/更新另一条链的信息状态



+ Connection

| Key                            | Value                                   | 备注               |
| ------------------------------ | --------------------------------------- | ------------------ |
| connections/`connectionID`     | Connection(state/clientID/counterParty) | 连接信息           |
| clients/`clientID`/connections | []connectionIDs                         | clientID下的连接们 |
|                                |                                         |                    |

+ Channel

Path = ports/`portID`/channel/`channelID`

| Key                   | Value                                         | 备注     |
| --------------------- | :-------------------------------------------- | -------- |
| Path                  | Channel(state/ordering/counterParty/connHops) | 通道信息 |
| Path/nextSequenceSend | Sequence（uint64）                            |          |
| Path/nextSequenceRecv | Sequence（uint64）                            |          |



+ Packet

