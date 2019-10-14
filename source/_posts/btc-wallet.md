---
title: btc_wallet
categories:
  - btc
date: 2019-4-14 14:41:55
tags:
---

### 比特币钱包gocoin学习笔记

> <https://github.com/piotrnar/gocoin>
>
> **Gocoin** is a full **Bitcoin** solution written in Go language (golang).
>
> The software architecture is focused on maximum performance of the node and cold storage security of the wallet.
>
> The **wallet** is designed to be used offline. It is deterministic and password seeded. As long as you remember the password, you do not need any backups ever. Wallet can be used without the client, but with the provided **balio** tool instead.
>
> The **client** (p2p node) is an application independent from the **wallet**. It keeps the entire UTXO set in RAM, providing the best block processing performance on the market.



#### blockdb

- cache, 缓存池，新产生block，添加入缓存池，当缓存池满了之后，踢掉最老使用的block
- 写缓存channel，当channel满了之后，flush channel 到db文件中



#### unspent_db

如介绍中提到的，会将所有的utxo都存在内存里。

- map, key为txid, v为txid下的所有的outs。问题是key的长度为8？txid长度为32，只取前8位？不会存在冲突吗
- update为整体修改，即txid下所有的outs。
- 存于文件，程序启动时从文件读取，必要时存入文件(如程序退出时)



#### wallet/db.go

账户系统，addr - utxo - balance

```
var (
	AllBalancesP2KH, AllBalancesP2SH, AllBalancesP2WKH map[[20]byte]*OneAllAddrBal
	AllBalancesP2WSH                                   map[[32]byte]*OneAllAddrBal
)

type OneAllAddrInp [utxo.UtxoIdxLen + 4]byte

type OneAllAddrBal struct {
	Value   uint64 // Highest bit of it means P2SH
	unsp    []OneAllAddrInp
	unspMap map[OneAllAddrInp]bool
}

```

看起来主要是存于内存中，map的key为addr，OneAllAddrInp为txid的前8位(unspend_db中map的key)+n(第几个out)

其中，根据out中script可以判断是什么地址的签名。

```
if out.IsP2KH() {
		copy(uidx[:], out.PKScr[3:23])
		rec = AllBalancesP2KH[uidx]
		if rec == nil {
			rec = &OneAllAddrBal{}
			AllBalancesP2KH[uidx] = rec
		}
} else if out.IsP2SH() {
	copy(uidx[:], out.PKScr[2:22])
	rec = AllBalancesP2SH[uidx]
	if rec == nil {
		rec = &OneAllAddrBal{}
		AllBalancesP2SH[uidx] = rec
	}
} else if out.IsP2WPKH() {
	copy(uidx[:], out.PKScr[2:22])
	rec = AllBalancesP2WKH[uidx]
	if rec == nil {
		rec = &OneAllAddrBal{}
		AllBalancesP2WKH[uidx] = rec
	}
} else if out.IsP2WSH() {
	var uidx [32]byte
	copy(uidx[:], out.PKScr[2:34])
	rec = AllBalancesP2WSH[uidx]
	if rec == nil {
		rec = &OneAllAddrBal{}
		AllBalancesP2WSH[uidx] = rec
	}
} else {
	continue
}



func (out *UtxoTxOut) IsP2KH() bool {
	return len(out.PKScr) == 25 && out.PKScr[0] == 0x76 && out.PKScr[1] == 0xa9 &&
		out.PKScr[2] == 0x14 && out.PKScr[23] == 0x88 && out.PKScr[24] == 0xac
}

func (r *UtxoTxOut) IsP2SH() bool {
	return len(r.PKScr) == 23 && r.PKScr[0] == 0xa9 && r.PKScr[1] == 0x14 && r.PKScr[22] == 0x87
}

func (r *UtxoTxOut) IsP2WPKH() bool {
	return len(r.PKScr) == 22 && r.PKScr[0] == 0 && r.PKScr[1] == 20
}

func (r *UtxoTxOut) IsP2WSH() bool {
	return len(r.PKScr) == 34 && r.PKScr[0] == 0 && r.PKScr[1] == 32
}

```





#### 分叉处理

首先，收到来自peer的区块都会存下来，只是使用trust字段来表面这是主链上的块还是分叉链上的块。在保存到磁盘时同样会保存trust字段。

其次，如果出现分叉了，会统计两条链的算力(这里为什么不是统计区块高度而是难度值让我有点费解)，然后如果分叉链的算力大于主链，会进行主链重置。跟eth的处理类似，首先找到共同的主块(分叉开始的地方)，然后废除的一方进行utxo的undo, 块的删除，然后新链进行utxo的doing。

代码大致位于lib/chain/下，如blockdb.go,chain_accept.go等



分叉算力统计算法

```
// Returns true if b1 has more POW than b2
func (b1 *BlockTreeNode) MorePOW(b2 *BlockTreeNode) bool {
	var b1sum, b2sum float64
	for b1.Height > b2.Height {
		b1sum += btc.GetDifficulty(b1.Bits())
		b1 = b1.Parent
	}
	for b2.Height > b1.Height {
		b2sum += btc.GetDifficulty(b2.Bits())
		b2 = b2.Parent
	}
	for b1 != b2 {
		b1sum += btc.GetDifficulty(b1.Bits())
		b2sum += btc.GetDifficulty(b2.Bits())
		b1 = b1.Parent
		b2 = b2.Parent
	}
	return b1sum > b2sum
}

```





#### 关于地址

之前想通过input中的script直接算出地址，是不可行的。

普通的*P2PKH*的input中签名包含两部分pk+sig，所以拿到是可以通过pk解析出地址的。

但P2SH script的签名是不包含pk的，所以是解析不到的。

普遍而言，还是在out中找好了
