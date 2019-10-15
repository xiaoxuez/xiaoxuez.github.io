---
title: eos
categories:
  - eos
date: 2019-10-14 14:43:53
tags:
---

## EOS生态系统介绍

### EOS目标

- EOS能实现每秒百万级的处理量，而目前比特币是每秒7笔，以太坊是30-40笔。
- EOS将是第一个拥有自己宪法(constitution)的区块链，能实现高度自治。
- EOS将会成为区块链的操作系统，为开发dApp的开发者提供底层模块，降低开发门槛。会导致开发区块链dApp的大潮。并发处理快，没有手续费，会吸引更多普通用户。



### EOS App架构

![架构图](https://upload-images.jianshu.io/upload_images/3866441-75058ee05f7b7f8d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

- IPFS存储

  包含文件的存储，和服务器端程序的存储

- 数据库查询服务

​        除了托管文件之外，块生产者还需要运行能够代表应用程序查询区块链数据库状态的API节点

- 资源限制

  应用程序在块链和接口上都占用传输带宽，CPU和存储空间。 **块链生产者**必须设置访问限制规则，以防用户滥用



### EOS 系统组成

![EOS系统组成](https://github.com/EOSIO/eos/wiki/assets/Single-Host-Testnet-No-keosd.png)

上图为本地单节点组成。

- `keosd`：本地钱包工具。非节点用户存储钱包的进程，可以管理多个含有私钥的钱包并加密。
- `cleos`:本地的命令行工具，通过命令行与真人用户交互，并与节点（nodeos）的 REST 接口通信。是用户或者开发者与节点进程交互的桥梁。
- `nodeos`： EOS 系统的核心进程，也就是所谓的“节点”。



节点运行时可配置相关插件。如

- `producer_plugin`（见证人插件）：见证人必须使用这个插件，普通节点不需要。
- `wallet_plugin`（钱包插件）：使用这个插件就可以省去 keosd 钱包工具。
- `wallet_api_plugin`（钱包接口插件）：给钱包插件提供接口。
- `chain_api_plugin`（区块链接口插件）：提供区块链数据接口。
- `http_plugin`（http 插件）：提供 http 接口。
- `account_history_api_plugin`（账户历史接口）：提供账户历史查询接口。



公共网络下，用户通过 `cleos` 连接到 `nodeos` ， `nodeos` 再连接到区块链网络（其他`nodeos`）。



### 共识算法(BFT-DPOS)

被投票选出的21个超级节点轮流进行出块，每0.5秒生成一个。任何时刻，只有一个生产者被授权产生区块。如果在计划的某个时间内没有成功出块，则跳过该块。如果有一个或更多的区块被跳过，则在区块链上会有0.5s或者更久的空白。一个生产者一次连续出6个块。那么，区块的产生是以126个区块(每个出块者六个区块，乘以21个出块者)为一个周期。在每个出块周期开始时，会根据通证持有人所投票数选出21个区块生产者。被选中的区块生产者的顺序会根据15个及以上的区块生产者的同意，制定出块顺序的安排。

如果出块者错过了一个块，并且在最近24小时内没有产生任何块，则这个出块者将被剔除在考虑范围之外，直到他们通知区块链可以重新开始产生区块。这确保了网络的顺利运行，把被证明为不可靠的区块生产者排除在出块排程之外，通过这一方式使得错过区块的数量最小化



**投票**：根据最新规则，投票将会在官方钱包中进行，每隔 63s ，也就是每产生126个区块进行一次。每个EOS拥有 30 票，可以投给不同的候选者。参与投票的代币将会被锁定一段时间，代币持有者可以设置代币锁定期内的候选人账户和投票数量进行投票。

投票结果，EOS 系统会统计各个候选节点获得的代币数量。其中获得代币数量最多的 21 个节点将被选定为超级节点，次多的 100 个节点被选为备用节点。

**超级节点**：收集、验证网络交易信息，打包到区块中，并广播给其他节点。

在正常情况下，DPOS块链不会经历任何分叉，因为区块生产者并非竞争关系，他们合作产生区块。如果有区块分叉，共识将自动切换到最长链。关于不会分叉的验证，见[验证篇](https://steemit.com/dpos/@dantheman/dpos-consensus-algorithm-this-missing-white-paper)

另外，在传统的DPOS算法上增加了拜占庭容错算法(Byzantin Fault Tolerance) ，所有的出块者都要对所有区块签名，以此来确保在同一时间戳或者同一区块高度上，没有区块生产者能够同时在两个区块上签名。一个区块有了15个区块生产者的签名，该区块就被认为是不可逆的。



### 生态建设

Block.one和Galaxy Digital将通过资本化一个新的3.25亿美元的[http://EOS.IO](https://link.zhihu.com/?target=http%3A//EOS.IO)生态系统基金（“基金”），为未来的投资部署资金。总之不缺钱，而且目前有大量DAPP已经宣布加入EOS，有易用的通用开发模块，基金投资，人气很不错。

![EOS的Dapp生态图](https://pic2.zhimg.com/80/v2-1b8fb0ef655fcc6e3c70a8da6d3a379d_hd.jpg)