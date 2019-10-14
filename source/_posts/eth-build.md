---
title: eth_build
categories:
  - eth
date: 2019-10-14 14:49:01
tags:
---
项目选择为go-ethereum，即geth。（原来geth是go-ethereum的简称..）。geth是以太坊、Whisper（去中心化通信协议）和Swarm（去中心化文件系统）节点的一个实现。用于接入以太坊网络，进行账户管理、交易、挖矿、智能合约相关的操作的以太坊客户端。另外，geth是一种CLI应用。



环境配置

环境为mac

按照[官方文档]( )进行配置就好。采用的是克隆go-ethereum仓库的方式进行配置。

注意的点是，go版本很关键。刚开始下载的go版本就为1.7.+,然后在进行make geth的时候报错了..然后重新brew install go，搞到go版本1.10.+，然后进行make geth还是报错..无奈下载官方建议1.9.+，顺利通过...



make解读，使用make命令时调用makefile文件，查看makefile代码，其实make all/make geth其实就是使用go对cmd/目录下各工具进行代码打包成可执行文件。生成文件位于cmd/bin下。



make完成后将cmd/bin目录添加到环境变量中，方便之后使用。



创建3个目录文件夹，分别存放3个账户数据。 —datadir node1 为指定Data directory for the databases and keystore。创建3个账户

```
xiaoxuez$ ls
node1	node2	node3
xiaoxuez$ geth --datadir node1 account new
```



生成创世块，需要定义.json文件，puppeth工具可以帮忙快速生成创世快文件。puppeth源代码位于cmd/下，make的时候如果只make etheruem的话就只会生成ethereum的可执行文件，make all就都有了~  实在没有的话就单独生成一次。可参照

```
build/env.sh go run build/ci.go install ./cmd/puppeth/
```

回到正题，puppeth的使用直接见命令行

```
$ puppeth
+-----------------------------------------------------------+
| Welcome to puppeth, your Ethereum private network manager |
|                                                           |
| This tool lets you create a new Ethereum network down to  |
| the genesis block, bootnodes, miners and ethstats servers |
| without the hassle that it would normally entail.         |
|                                                           |
| Puppeth uses SSH to dial in to remote servers, and builds |
| its network components out of Docker containers using the |
| docker-compose toolset.                                   |
+-----------------------------------------------------------+

Please specify a network name to administer (no spaces or hyphens, please)
> helloworldprivate

Sweet, you can set this via --network=helloworldprivate next time!

INFO [04-10|13:45:58] Administering Ethereum network           name=helloworldprivate
WARN [04-10|13:45:58] No previous configurations found         path=/Users/xiaoxuez/.puppeth/helloworldprivate

What would you like to do? (default = stats)
 1. Show network stats
 2. Configure new genesis
 3. Track new remote server
 4. Deploy network components
> 2

Which consensus engine to use? (default = clique) #共识选择，可直接选择pow,这里我选择的是poa,(不消耗计算力, 可以用于私链测试开发)
 1. Ethash - proof-of-work
 2. Clique - proof-of-authority
> 2

How many seconds should blocks take? (default = 15)  #5s出一个块
> 5

Which accounts are allowed to seal? (mandatory at least one)  #输入有签名权限的账户
> 0x1186c97871079e86e9adcf11f56b90caa3619ea4
> 0x34c72adc7499cce01530dc27e67810a79fdbd8ea
> 0xb37fe8ba7afbeb5e57139e55e737a34574e91a74
> 0x

Which accounts should be pre-funded? (advisable at least one)  #输入有预留余额的账户
> 0xb37fe8ba7afbeb5e57139e55e737a34574e91a74
> 0x34c72adc7499cce01530dc27e67810a79fdbd8ea
> 0x1186c97871079e86e9adcf11f56b90caa3619ea4
> 0x

Specify your chain/network ID if you want an explicit one (default = random) #输入私链ID, 直接输入回车,已默认随机数作为私链ID
>
INFO [04-10|13:47:49] Configured new genesis block

What would you like to do? (default = stats)
 1. Show network stats
 2. Manage existing genesis
 3. Track new remote server
 4. Deploy network components
> 2

 1. Modify existing fork rules
 2. Export genesis configuration
 3. Remove genesis configuration
> 2

Which file to save the genesis into? (default = helloworldprivate.json)
>
INFO [04-10|13:48:00] Exported existing genesis block

What would you like to do? (default = stats)
 1. Show network stats
 2. Manage existing genesis
 3. Track new remote server
 4. Deploy network components
> ^C
$ ls  #结束后会在当前文件夹下看到导出的.json文件
helloworldprivate.json	node1			node2			node3

```



接下来就是使用.json文件生成创世块了

```
#init:Bootstrap and initialize a new genesis block
$ geth --datadir node1 init helloworldprivate.json
$ geth --datadir node1 --port 30000 --nodiscover --unlock '0' --password ./node1/password console
#启动geth, --port: Network listening port ，指定和其他节点链接所用的端口号
#               --nodiscover: Disables the peer discovery mechanism (manual peer addition)关闭节点发现机制，防止加入有同样初始配置的陌生节点，后续手动配置节点网络，
#               --unlock: Comma separated list of accounts to unlock, 解锁的账户，以逗号分隔，账户要转帐或者部署合约，需要先解锁
#               console: 进入控制台，不加console的话，也可在geth启动后通过geth attach
ipc:node0/geth.ipc来访问
#               --rpc: 表示开启http-rpc服务
                --rpcport: 指定http-rpc服务监听端口，默认为8545
                --ws:
                --wsport:
                --allow-insecure-unlock


```

同样，目前启动3个节点，使用不同端口来模拟多节点，另起终端

```
$ geth --datadir node2 init helloworldprivate.json
$ geth --datadir node2 --port 30001 --nodiscover --unlock '0' --password ./node2/password console

$ geth --datadir node3 init helloworldprivate.json
$ geth --datadir node3 --port 30002 --nodiscover --unlock '0' --password ./node3/password console
```

三个启动时候unlock 的值都是'0',  之前在每个节点文件下都只创建了一个账户，所以解锁的就是各个节点下的账户，即目前三个账户都是解锁。这里的小疑问是要起3个初始块，猜测是相同初始配置的节点才能加入到同一个网络中。

console进入的控制台是一个交互式的JavaScript执行环境，在这里面可以执行JavaScript代码。这个环境里也内置了一些用来操作以太坊的JavaScript对象，例如

- eth：包含一些跟操作区块链相关的方法；
- net：包含一些查看p2p网络状态的方法；
- admin：包含一些与管理节点相关的方法；
- miner：包含启动&停止挖矿的一些方法；
- personal：主要包含一些管理账户的方法；
- txpool：包含一些查看交易内存池的方法；
- web3：包含了以上对象，还包含一些单位换算的方法。



常见命令有：

- personal.newAccount()：创建账户；

- personal.unlockAccount()：解锁账户；

- eth.accounts：枚举系统中的账户；

- eth.getBalance()：查看账户余额，返回值的单位是 Wei（Wei 是以太坊中最小货币面额单位，类似比特币中的`聪`，1 ether = 10^18 Wei）； `web3.fromWei(eth.getBalance(e), "ether") `

  `web3.toWei('1', 'ether')`



- eth.blockNumber：列出区块总数；blockNumber为eth的属性，直接查看即可

- eth.getTransaction()：获取交易；

- eth.getBlock()：获取区块；如eth.getBlock("pending", true).transactions 查看当前待确认交易。或通过区块号查看eth.getBlock(4)

- miner.start()：开始挖矿；

- miner.stop()：停止挖矿；

- web3.fromWei()：Wei 换算成以太币；

- web3.toWei()：以太币换算成 Wei；如amount = web3.toWei(5,'ether')

- txpool.status：交易池中的状态；

- admin.addPeer()：连接到其他节点；



接下来，添加peer节点

```
> admin.peers #查看当前节点链接的节点
> net.peerCount #查看当前节点链接的节点数
```

目前三个节点都是没有链接的，所以admin.peers的结果为[]。



```
> admin.nodeInfo.enode #查看节点地址，地址组成为encode://<节点id>@ip地址：端口
"enode://0480c837fc76671a8d8ee78b74fd04f9439762cc2e2e668b8604dda964c679ec8dd10f8347d7066707a315db8ad5c141e9d5a5b98f3a790660792f4469086f21@[::]:30000?discport=0"
```

查看node1节点地址，在node2, node3的console中使用admin.addPeer，将node1分别添加到node2,node3中

```
> admin.addPeer("enode://0480c837fc76671a8d8ee78b74fd04f9439762cc2e2e668b8604dda964c679ec8dd10f8347d7066707a315db8ad5c141e9d5a5b98f3a790660792f4469086f21@[::]:30000?discport=0")

```

再使用admin.peers查看的时候，可以看到node1链接上2个节点（node2, node3）,node2和node3各自链接上1个节点（node1）

也可使用配置文件的形式添加节点，在节点的datadir下新建static-node.json, 将需要连接的节点地址写入，如：

```
[
"enode://0480c837fc76671a8d8ee78b74fd04f9439762cc2e2e668b8604dda964c679ec8dd10f8347d7066707a315db8ad5c141e9d5a5b98f3a790660792f4469086f21@[::]:30000?discport=0",
"enode://79671ba7d09a4e6d16a64128e2cb24f544ec8740084e1ceb1abcac7b1c2b8a37dae812653360caf30ceff782265b124c1e6ce1d547dc6854bafb85596795751c@[::]:30001?discport=0",
"enode://3237c4eaf0dbf55b459eef14c4ecffe987b5179e482399b282bc5ab3d1559dd8461a9d39460810bf0d8c276fb13a9950eca5c38890dbf11d9dccdb1b61231a78@[::]:30002?discport=0"
]
```

或者直接在geth启动行中添加 - -bootnodes enode:….

那么在geth启动的时候会自动链接。链接到的节点就互相发现组成以太坊私链网络了



当组成网络之后，就可以开始挖矿了。在某个节点下，调用miner.start()

```
> miner.start()
INFO [04-10|17:59:10] Transaction pool price threshold updated price=18000000000
INFO [04-10|17:59:10] Starting mining operation
null
> INFO [04-10|17:59:10] Commit new mining work                   number=1 txs=2 uncles=0 elapsed=1.058ms
INFO [04-10|17:59:10] Successfully sealed new block            number=1 hash=3b7e10…38d85e
INFO [04-10|17:59:10] 🔨 mined potential block                  number=1 hash=3b7e10…38d85e
INFO [04-10|17:59:10] Commit new mining work                   number=2 txs=0 uncles=0 elapsed=285.156µs
INFO [04-10|17:59:15] Successfully sealed new block            number=2 hash=1bb861…fbcb78
```

正常情况是这个样子..  不正常的情况，其一需要先解锁账户

```
WARN [04-10|17:55:49] Block sealing failed                     err="authentication needed: password or unlock"
> personal.unlockAccount(eth.accounts[0])
```

其二，如下

```
> miner.start()

null
```

返回null, 是不得行的~，解决方式如下

```
> miner.setEtherbase(eth.accounts[0])
true
> miner.start()
```



在某个节点下，提交一个交易，如,  to为账户地址，查看账户地址为eth.accounts

```
> eth.sendTransaction({from: eth.accounts[0], to: "b6deeb049de5fb90345b5b84dd1faf2d56c6929e", value: web3.toWei(200, "ether")})
```



如果miner没有关掉的话，几秒后就会有含有交易的block产出,  txs为block包含的交易数

```
INFO [04-10|17:59:15] Imported new chain segment               blocks=2 txs=1 mgas=0.042 elapsed=1.348ms  mgasps=31.157 number=2 hash=1bb861…fbcb78 cache=2.47kB
```



挖矿，挖到一个区块之后就停止挖矿可使用

```
> miner.start(1);admin.sleepBlocks(1);miner.stop();
```

当然，还有可能是网速原因… 过几分钟再试就好了。





智能合约

```
brew install solc
```

…..





```
geth attach http://0.0.0.0:1112



"/Applications/Ethereum Wallet.app/Contents/MacOS/Ethereum Wallet" --rpc http://localhost:8545



    --rpcapi="db,eth,net,web3,personal,web3" --rpc --rpcaddr 0.0.0.0 -rpcport 1112 --mine



eth.sendTransaction({from:eth.accounts[0], data:"0x0100f862a00000000000000000000000000000000000000000000000000000000000000000a00000000000000000000000000000000000000000000000000000000000000000809471562b71999873db5b286df957af199ec94617f78568656c6c6f80808080"})



```





编译android

```
make android

./build/env.sh xgo --go=latest --dest=build/bin -v --targets=android-23/arm64 ./cmd/geth

glide mirror set golang.org/x/text  /home/users/qiangmzsx/var/golang/golang.org/x/text



```





```
eth.sendTransaction({from: eth.accounts[0], to: "0x71562b71999873DB5b286dF957af199Ec94617F7", value: 200000})
```
