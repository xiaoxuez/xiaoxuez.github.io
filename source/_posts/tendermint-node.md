---
title: tendermint-node
categories:
  - cosmos
date: 2019-12-18 15:38:39
tags:
---



+ Transport 节点的tcp连接
+ Switch p2p连接

Transport作为Switch的底层

```
func (sw *Switch) acceptRoutine() {
	for {
		//等待有节点接入，接入后会返回p，否则会一直等着
		p, err := sw.transport.Accept(peerConfig{
			chDescs:      sw.chDescs,
			onPeerError:  sw.StopPeerForError,
			reactorsByCh: sw.reactorsByCh,
			metrics:      sw.metrics,
			isPersistent: sw.isPeerPersistentFn(),
		})
		...
		if err := sw.addPeer(p); err != nil {
			//断开连接
			sw.transport.Cleanup(p)
		}
	}		
```





+ node.Start

  ```
//start rpc
n.startRPC()

//transport开启监听端口等待节点tcp接入
n.transport.Listen(*addr)

//p2p服务开启，当transport有节点接入时，会进入switch处理
n.sw.Start()

//sw.Start()内容是
// OnStart implements BaseService. It starts all the reactors and peers.
func (sw *Switch) OnStart() error {
	// Start reactors
	for _, reactor := range sw.reactors {
		err := reactor.Start()
		if err != nil {
			return errors.Wrapf(err, "failed to start %v", reactor)
		}
	}

	// Start accepting Peers.
	go sw.acceptRoutine()

	return nil
}
  ```

//sw的reactor包括

+ mempoolReactor  //for gossipping transactions，广播交易
+ brReactor   //for fast-syncing
+ consensusReactor //for participating in the consensus, 共识
+ evidenceReactor



### consensusReactor

```
// OnStart implements BaseService by subscribing to events, which later will be
// broadcasted to other peers and starting state if we're not in fast sync.
func (conR *ConsensusReactor) OnStart() error {
	conR.Logger.Info("ConsensusReactor ", "fastSync", conR.FastSync())

	// start routine that computes peer statistics for evaluating peer quality
	go conR.peerStatsRoutine()

	conR.subscribeToBroadcastEvents()

	if !conR.FastSync() {
		err := conR.conS.Start()
		if err != nil {
			return err
		}
	}

	return nil
}
```

