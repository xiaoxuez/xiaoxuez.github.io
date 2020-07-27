---
title: binance-tss
categories:
  - binance
date: 2020-07-27 10:42:55
tags:
---

## Tss源码分析

[Binance tss库](https://github.com/binance-chain/tss-lib)是提供多签的库。这里将简单过下源码。

在`ecdsa`包下的结构为

```
.
├── keygen
├── resharing
└── signing
```

`keygen`为创建多签私钥提供支持；

`resharding`为修改多签中的参与者，重新分配计算私钥(共享公钥不会发生改变)提供支持；

`signing` 为多签提供支持；



再打开这三个包下的文件结构还蛮清晰的，结构都类似；分别都包含`message.go`、`local_party.go`、`round_*`等文件。

### 大体结构

先说结论

- 每个参与者都需要一个partyID来标识身份，并且，算法开始前，需要拿到所有参与者的partyID，并按Key进行排序
- 实现由party和round定义，party统计数据，并调用round。round发出去信息并根据收到别的参与者的消息进行计算变换状态



同样，以测试`Keygen.local_party_test.go`举例。

```
func TestStartRound1Paillier(t *testing.T) {
	setUp("debug")

	pIDs := tss.GenerateTestPartyIDs(1)
	p2pCtx := tss.NewPeerContext(pIDs)
	threshold := 1
	params := tss.NewParameters(p2pCtx, pIDs[0], len(pIDs), threshold)

	out := make(chan tss.Message, len(pIDs))
	lp := NewLocalParty(params, out, nil).(*LocalParty)
	if err := lp.Start(); err != nil {
		assert.FailNow(t, err.Error())
	}
	<-out

	// Paillier modulus 2048 (two 1024-bit primes)
	// round up to 256, it was used to be flaky, sometimes comes back with 1 byte less
	len1 := len(lp.data.PaillierSK.LambdaN.Bytes())
	len2 := len(lp.data.PaillierSK.PublicKey.N.Bytes())
	if len1%2 != 0 {
		len1 = len1 + (256 - (len1 % 256))
	}
	if len2%2 != 0 {
		len2 = len2 + (256 - (len2 % 256))
	}
	assert.Equal(t, 2048/8, len1)
	assert.Equal(t, 2048/8, len2)
}
```

这是一个参与者为1，阈值为1的生成私钥数据测试；

##### 首先`tss.GenerateTestPartyIDs`

多个人参与的多重签名，之前看到的别的多签简单点来说就是将一个私钥分给多个人，每个人拥有一部分参数，那么就像玩拼图一样，这些人应该是有序的，拆分的时候就按顺序分给每个人，组装的时候同样按照顺序进行组装；所以这里也类似，参与者们都是有顺序的。

`GenerateTestPartyIDs`方法如下

```
func GenerateTestPartyIDs(count int, startAt ...int) SortedPartyIDs {
	ids := make(UnSortedPartyIDs, 0, count)
	key := common.MustGetRandomInt(256)
	frm := 0
	i := 0 // default `i`
	if len(startAt) > 0 {
		frm = startAt[0]
		i = startAt[0]
	}
	for ; i < count+frm; i++ {
		ids = append(ids, &PartyID{
			MessageWrapper_PartyID: &MessageWrapper_PartyID{
				Id:      fmt.Sprintf("%d", i+1),  
				Moniker: fmt.Sprintf("P[%d]", i+1), //为了辅助人为识别是哪个参与者
				Key:     new(big.Int).Sub(key, big.NewInt(int64(count)-int64(i))).Bytes(),
			},
			Index: i,
			// this key makes tests more deterministic
		})
	}
	return SortPartyIDs(ids, startAt...)
}

```

方法返回的`SortedPartyIDs`是PartyID有序数组(按照Key的大小进行排序)，{PartyID实例}.Index其实就是该实例所在的顺序；

##### 其次

```
	//前面组装了多个参与者pIDs，并由pIDs组装context，再由context组装params, params还包括m参与者n阈值的参数，out为输出channel
	lp := NewLocalParty(params, out, nil).(*LocalParty)
	if err := lp.Start(); err != nil {
		assert.FailNow(t, err.Error())
	}
	<-out
```

`Start()`方法，为`tss/Party`的实现,主要重写的方法如下

```
type Party interface {
	Start() *Error
	//更新接收到的来自参与者的消息
	UpdateFromBytes(wireBytes []byte, from *PartyID, isBroadcast bool) (ok bool, err *Error)
	//更新接收到的来自参与者的消息
	Update(msg ParsedMessage) (ok bool, err *Error)
	Running() bool
	WaitingFor() []*PartyID
	ValidateMessage(msg ParsedMessage) (bool, *Error)
	StoreMessage(msg ParsedMessage) (bool, *Error)
	FirstRound() Round 
	WrapError(err error, culprits ...*PartyID) *Error
	PartyID() *PartyID
	String() string

	// Private lifecycle methods
	...	
}
```

在`tss`包下`Party`定义的同文件，定义了`BaseStart`方法，大家实现的Party的`Start`方法都是调用的`BaseStart`方法，在`BaseStart`方法中，设置了初始运行的为`FirstRound`返回的Round；

Round同样为接口

```
type Round interface {
	Params() *Parameters
	Start() *Error
	Update() (bool, *Error)
	RoundNumber() int
	CanAccept(msg ParsedMessage) bool
	CanProceed() bool
	NextRound() Round 
	WaitingFor() []*PartyID
	WrapError(err error, culprits ...*PartyID) *Error
}
```

这类似于状态机，一个接一个..`Start`方法计算出某些数据，发向别的参与者，然后等待接收到足够的来自别的参与者的信息后，便进入`NextRound`。在`keygen`的round4中，实现的NextRound如下，当NextRound返回的值==nil 时，便就结束了。

```
func (round *round4) NextRound() tss.Round {
	return nil // finished!
}
```



### 多个参与者

看一下多个参与者如何进行。这里以signing包下的local_party_test中的测试来举例。

代码较多，就直接将说明注释在代码里。

```
func TestE2EConcurrent(t *testing.T) {
	setUp("info")
	threshold := testThreshold

	//从先前产生的testParticipants个私钥文件中读出testThreshold+1个私钥来参与
	//参与多签的私钥存储于数组keys中，参与多签的partiyId存储于signPIDS中
	keys, signPIDs, err := keygen.LoadKeygenTestFixturesRandomSet(testThreshold+1, testParticipants)
	assert.NoError(t, err, "should load keygen fixtures")
	assert.Equal(t, testThreshold+1, len(keys))
	assert.Equal(t, testThreshold+1, len(signPIDs))

	// PHASE: signing
	// use a shuffled selection of the list of parties for this test
	p2pCtx := tss.NewPeerContext(signPIDs)
	parties := make([]*LocalParty, 0, len(signPIDs))

	errCh := make(chan *tss.Error, len(signPIDs))
	//消息通道
	outCh := make(chan tss.Message, len(signPIDs))
	//多签结束通道
	endCh := make(chan SignatureData, len(signPIDs))
	//接收消息的Update
	updater := test.SharedPartyUpdater
	//要签名的数据
	hash, _ := hex.DecodeString(msg)

	//启动testThreshold+1的参与者
	for i := 0; i < len(signPIDs); i++ {
		fmt.Println("parameters party id: ", signPIDs[i])
		params := tss.NewParameters(p2pCtx, signPIDs[i], len(signPIDs), threshold)
		P := NewLocalParty(new(big.Int).SetBytes(hash), params, keys[i], outCh, endCh).(*LocalParty)
		//将参与者的party存储，后续接收到信息时，需传递给对应的party
		parties = append(parties, P)
		go func(P *LocalParty) {
			if err := P.Start(); err != nil {
				errCh <- err
			}
		}(P)
	}

	var ended int32
signing:
	for {
		// fmt.Printf("ACTIVE GOROUTINES: %d\n", runtime.NumGoroutine())
		select {
		case err := <-errCh:
			common.Logger.Errorf("Error: %s", err)
			assert.FailNow(t, err.Error())
			break signing

		case msg := <-outCh:
		  //有消息发出，这里消息有两种，一种是广播，需将消息传递给出自己之外的参与者；一种是定向，需将消息传递给定向的那个参与者
			dest := msg.GetTo()
			fmt.Println(msg.Type(), " from ",  msg.GetFrom(), " to ", msg.GetTo())
			//如果to == nill,就说明是广播消息
			if dest == nil {
				for _, P := range parties {
					if P.PartyID().Index == msg.GetFrom().Index {
						continue
					}
					//将消息传递给对应party
					go updater(P, msg, errCh)
				}
			} else {
				if dest[0].Index == msg.GetFrom().Index {
					t.Fatalf("party %d tried to send a message to itself (%d)", dest[0].Index, msg.GetFrom().Index)
				}
				//定向将消息传递给对应的party
				go updater(parties[dest[0].Index], msg, errCh)
			}

		case sig := <-endCh:
			//到这里签名有结果了，多签就完成了，签名的数据分别是R、S
			fmt.Println("R=>",new(big.Int).SetBytes(sig.R))
			fmt.Println("S=>", new(big.Int).SetBytes(sig.S))
			// fmt.Println("sig=>", hex.EncodeToString(sig.Signature))
			//后面就是校验了
			//因为是本地启动了多个协程作为参与者，且是使用的相同的channel，所以endCh会收到每一个参与者的多签结果
			atomic.AddInt32(&ended, 1)
			fmt.Println("==> end")
			//当所以的参与者都完成了签名
		 if atomic.LoadInt32(&ended) == int32(len(signPIDs)) {
				t.Logf("Done. Received save data from %d participants", ended)
				R := parties[0].temp.bigR
				r := parties[0].temp.rx
				fmt.Printf("sign result: R(%s, %s), r=%s\n", R.X().String(), R.Y().String(), r.String())

				modN := common.ModInt(tss.EC().Params().N)

				// BEGIN check s correctness
				sumS := big.NewInt(0)
				for _, p := range parties {
					sumS = modN.Add(sumS, p.temp.si)
				}
				//这里计算出来的sumS和sig.S是相同的值，
				fmt.Printf("S: %s\n", sumS.String())
				fmt.Println("pub", keys[0].ECDSAPub.X(), keys[0].ECDSAPub.Y())

				// END check s correctness

				// 以ecdsa来校验签名的正确性
				pkX, pkY := keys[0].ECDSAPub.X(), keys[0].ECDSAPub.Y()
				pk := ecdsa.PublicKey{
					Curve: tss.EC(),
					X:     pkX,
					Y:     pkY,
				}
				ok := ecdsa.Verify(&pk, hash, R.X(), sumS)
				assert.True(t, ok, "ecdsa verify must pass")
				t.Log("ECDSA signing test done.")
				// END ECDSA verify
			if atomic.LoadInt32(&ended) == int32(len(signPIDs)) {
				break signing
			}
			 }
		}
	}
}
```



需要注意的是，这个示例中产生的S，有时候需要稍微处理一下才能适用于币安链。

```
			r := new(big.Int).SetBytes(msg.R)
			s := new(big.Int).SetBytes(msg.S)
			if s.Cmp(HalfN) == 1 {
				s = s.Sub(btcec.S256().N, s)
			}
			signature := btcec.Signature{R: r, S: s}
```



另外，在`tss.LocalPartySaveData`结构中的公钥转`secp256k1` 公钥的方式如下

```
	//key keygen.LocalPartySaveData
	y := key.ECDSAPub.Y()
	x := key.ECDSAPub.X()
	var pubkey [33]byte
	copy(pubkey[1:], x.Bytes())
	if y.And(y, big.NewInt(1)).Cmp(big.NewInt(0)) > 1 {
		pubkey[0] = 2
	} else {
		pubkey[0] = 3
	}
	//secp256k1.PubKeySecp256k1(pubkey)
```



