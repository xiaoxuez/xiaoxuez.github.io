---
title: eth_code_tx_event
categories:
  - eth
date: 2019-10-14 14:47:58
tags:
---

#### ApplyTransaction

```
// ApplyTransaction attempts to apply a transaction to the given state database
// and uses the input parameters for its environment. It returns the receipt
// for the transaction, gas used and an error if the transaction failed,
// indicating the block was invalid.
//应用交易到state database中
func ApplyTransaction(config *params.ChainConfig, bc ChainContext, author *common.Address, gp *GasPool, statedb *state.StateDB, header *types.Header, tx *types.Transaction, usedGas *uint64, cfg vm.Config) (*types.Receipt, uint64, error) {
	//构造交易消息
	msg, err := tx.AsMessage(types.MakeSigner(config, header.Number))
	if err != nil {
		return nil, 0, err
	}

	//EVM进行交易处理，返回为处理结果、消耗gas、是否失败、err，logs是在evm处理时产生的
	// Create a new context to be used in the EVM environment
	context := NewEVMContext(msg, header, bc, author)
	// Create a new environment which holds all relevant information
	// about the transaction and calling mechanisms.
	vmenv := vm.NewEVM(context, statedb, config, cfg)
	// Apply the transaction to the current state (included in the env)
	_, gas, failed, err := ApplyMessage(vmenv, msg, gp)
	if err != nil {
		return nil, 0, err
	}


	//根据处理结果构建返回数据
	// Update the state with pending changes
	var root []byte
	if config.IsByzantium(header.Number) {
		statedb.Finalise(true)
	} else {
		root = statedb.IntermediateRoot(config.IsEIP158(header.Number)).Bytes()
	}
	*usedGas += gas

	// Create a new receipt for the transaction, storing the intermediate root and gas used by the tx
	// based on the eip phase, we're passing whether the root touch-delete accounts.
	receipt := types.NewReceipt(root, failed, *usedGas)
	receipt.TxHash = tx.Hash()
	receipt.GasUsed = gas
	// if the transaction created a contract, store the creation address in the receipt.
	if msg.To() == nil {
		receipt.ContractAddress = crypto.CreateAddress(vmenv.Context.Origin, tx.Nonce())
	}
	// Set the receipt logs and create a bloom for filtering
	receipt.Logs = statedb.GetLogs(tx.Hash())
	receipt.Bloom = types.CreateBloom(types.Receipts{receipt})

	return receipt, gas, err
}

```



```
func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
	if evm.vmConfig.NoRecursion && evm.depth > 0 {
		return nil, gas, nil
	}

	// Fail if we're trying to execute above the call depth limit
	if evm.depth > int(params.CallCreateDepth) {
		return nil, gas, ErrDepth
	}
	// Fail if we're trying to transfer more than the available balance
	if !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value) {
		return nil, gas, ErrInsufficientBalance
	}

	var (
		to       = AccountRef(addr)
		snapshot = evm.StateDB.Snapshot()
	)
	if !evm.StateDB.Exist(addr) {
		precompiles := PrecompiledContractsHomestead
		if evm.ChainConfig().IsByzantium(evm.BlockNumber) {
			precompiles = PrecompiledContractsByzantium
		}
		if precompiles[addr] == nil && evm.ChainConfig().IsEIP158(evm.BlockNumber) && value.Sign() == 0 {
			// Calling a non existing account, don't do anything, but ping the tracer
			if evm.vmConfig.Debug && evm.depth == 0 {
				evm.vmConfig.Tracer.CaptureStart(caller.Address(), addr, false, input, gas, value)
				evm.vmConfig.Tracer.CaptureEnd(ret, 0, 0, nil)
			}
			return nil, gas, nil
		}
		evm.StateDB.CreateAccount(addr)
	}
	evm.Transfer(evm.StateDB, caller.Address(), to.Address(), value)
	// Initialise a new contract and set the code that is to be used by the EVM.
	// The contract is a scoped environment for this execution context only.
	contract := NewContract(caller, to, value, gas)
	contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))

	// Even if the account has no code, we need to continue because it might be a precompile
	start := time.Now()

	// Capture the tracer start/end events in debug mode
	if evm.vmConfig.Debug && evm.depth == 0 {
		evm.vmConfig.Tracer.CaptureStart(caller.Address(), addr, false, input, gas, value)

		defer func() { // Lazy evaluation of the parameters
			evm.vmConfig.Tracer.CaptureEnd(ret, gas-contract.Gas, time.Since(start), err)
		}()
	}
	ret, err = run(evm, contract, input, false)

	// When an error was returned by the EVM or when setting the creation code
	// above we revert to the snapshot and consume any gas remaining. Additionally
	// when we're in homestead this also counts for code storage gas errors.
	if err != nil {
		evm.StateDB.RevertToSnapshot(snapshot)
		if err != errExecutionReverted {
			contract.UseGas(contract.Gas)
		}
	}
	return ret, contract.Gas, err
}
```









#### handleMsg

处理接收到的消息码

- ```
  StatusMsg
  ```

  收到这个消息说明握手失败

- ```
  GetBlockHeadersMsg
  ```

  查询区块头请求，回复区块头信息

  ```
  BlockHeadersMsg
  ```

  区块头信息的回复

- ```
  GetBlockBodiesMsg
  ```

  查询区块请求

  ```
  BlockBodiesMsg
  ```

  区块请求查询的回复，

- ```
  GetNodeDataMsg
  ```

- ```
  NodeDataMsg
  ```

- ```
  GetReceiptsMsg
  ```

- ```
  ReceiptsMsg
  ```

- ```
  NewBlockHashesMsg
  ```

- ```
  NewBlockMsg
  ```

- ```
  TxMsg
  ```



#### Start

这四个goroutine 基本上就在不停的做广播区块、广播交易，同步到区块、同步到交易，再广播区块、广播交易。

- txBroadcastLoop

  ```
  func (self *ProtocolManager) txBroadcastLoop() {
      for {
          select {
          case event := <-self.txCh:
              self.BroadcastTx(event.Tx.Hash(), event.Tx)

          // Err() channel will be closed when unsubscribing.
          case <-self.txSub.Err():
              return
          }
      }
  }
  ```

  core/tx_pool.go 产生新的交易的时候会send self.txCh，这时候会激活
  self.BroadcastTx(event.Tx.Hash(), event.Tx)

  ```
  func (pm *ProtocolManager) BroadcastTx(hash common.Hash, tx *types.Transaction) {
      // Broadcast transaction to a batch of peers not knowing about it
      peers := pm.peers.PeersWithoutTx(hash)
      //FIXME include this again: peers = peers[:int(math.Sqrt(float64(len(peers))))]
      for _, peer := range peers {
          peer.SendTransactions(types.Transactions{tx})
      }
      log.Trace("Broadcast transaction", "hash", hash, "recipients", len(peers))
  }

  ```

  向缓存的没有这个交易hash的网络节点广播此次交易。

- minedBroadcastLoop

  收到miner.go 里面NewMinedBlockEvent 挖到新区块的事件通知，激活self.BroadcastBlock(ev.Block, true)

  ```
  // Mined broadcast loop
  func (self *ProtocolManager) minedBroadcastLoop() {
      // automatically stops if unsubscribe
      for obj := range self.minedBlockSub.Chan() {
          switch ev := obj.Data.(type) {
          case core.NewMinedBlockEvent:
              self.BroadcastBlock(ev.Block, true)  // First propagate block to peers
              self.BroadcastBlock(ev.Block, false) // Only then announce to the rest
          }
      }
  }

  ```

  ```
  func (pm *ProtocolManager) BroadcastBlock(block *types.Block, propagate bool) {
      hash := block.Hash()
      peers := pm.peers.PeersWithoutBlock(hash)

      // If propagation is requested, send to a subset of the peer
      if propagate {
          // Calculate the TD of the block (it's not imported yet, so block.Td is not valid)
          var td *big.Int
          if parent := pm.blockchain.GetBlock(block.ParentHash(), block.NumberU64()-1); parent != nil {
              td = new(big.Int).Add(block.Difficulty(), pm.blockchain.GetTd(block.ParentHash(), block.NumberU64()-1))
          } else {
              log.Error("Propagating dangling block", "number", block.Number(), "hash", hash)
              return
          }
          // Send the block to a subset of our peers
          transfer := peers[:int(math.Sqrt(float64(len(peers))))]
          for _, peer := range transfer {
              peer.SendNewBlock(block, td)
          }
          log.Trace("Propagated block", "hash", hash, "recipients", len(transfer), "duration", common.PrettyDuration(time.Since(block.ReceivedAt)))
          return
      }
      // Otherwise if the block is indeed in out own chain, announce it
      if pm.blockchain.HasBlock(hash, block.NumberU64()) {
          for _, peer := range peers {
              peer.SendNewBlockHashes([]common.Hash{hash}, []uint64{block.NumberU64()})
          }
          log.Trace("Announced block", "hash", hash, "recipients", len(peers), "duration", common.PrettyDuration(time.Since(block.ReceivedAt)))
      }
  }

  ```

  如果propagate为true 向网络节点广播整个挖到的block，为false 只广播挖到的区块的hash值和number值。广播的区块还包括这个区块打包的所有交易。

- syncer

  ```
  func (pm *ProtocolManager) syncer() {
      // Start and ensure cleanup of sync mechanisms
      pm.fetcher.Start()
      defer pm.fetcher.Stop()
      defer pm.downloader.Terminate()

      // Wait for different events to fire synchronisation operations
      forceSync := time.NewTicker(forceSyncCycle)
      defer forceSync.Stop()

      for {
          select {
          case <-pm.newPeerCh:
              // Make sure we have peers to select from, then sync
              if pm.peers.Len() < minDesiredPeerCount {
                  break
              }
              go pm.synchronise(pm.peers.BestPeer())

          case <-forceSync.C:
              // Force a sync even if not enough peers are present
              go pm.synchronise(pm.peers.BestPeer())

          case <-pm.noMorePeers:
              return
          }
      }
  }


  ```

  pm.fetcher.Start()启动 fetcher，辅助同步区块数据

  当P2P server执行 ProtocolManager 的p2p.Protocol 的Run指针的时候会send pm.newPeerCh，这时候选择最优的网络节点（TD 总难度最大的）启动pm.synchronise(pm.peers.BestPeer()) goroutine。

  ```
  // synchronise tries to sync up our local block chain with a remote peer.
  func (pm *ProtocolManager) synchronise(peer *peer) {
      // Short circuit if no peers are available
      if peer == nil {
          return
      }
      // Make sure the peer's TD is higher than our own
      currentBlock := pm.blockchain.CurrentBlock()
      td := pm.blockchain.GetTd(currentBlock.Hash(), currentBlock.NumberU64())

      pHead, pTd := peer.Head()
      if pTd.Cmp(td) <= 0 {
          return
      }
      // Otherwise try to sync with the downloader
      mode := downloader.FullSync
      if atomic.LoadUint32(&pm.fastSync) == 1 {
          // Fast sync was explicitly requested, and explicitly granted
          mode = downloader.FastSync
      } else if currentBlock.NumberU64() == 0 && pm.blockchain.CurrentFastBlock().NumberU64() > 0 {
          // The database seems empty as the current block is the genesis. Yet the fast
          // block is ahead, so fast sync was enabled for this node at a certain point.
          // The only scenario where this can happen is if the user manually (or via a
          // bad block) rolled back a fast sync node below the sync point. In this case
          // however it's safe to reenable fast sync.
          atomic.StoreUint32(&pm.fastSync, 1)
          mode = downloader.FastSync
      }
      // Run the sync cycle, and disable fast sync if we've went past the pivot block
      if err := pm.downloader.Synchronise(peer.id, pHead, pTd, mode); err != nil {
          return
      }
      if atomic.LoadUint32(&pm.fastSync) == 1 {
          log.Info("Fast sync complete, auto disabling")
          atomic.StoreUint32(&pm.fastSync, 0)
      }
      atomic.StoreUint32(&pm.acceptTxs, 1) // Mark initial sync done
      if head := pm.blockchain.CurrentBlock(); head.NumberU64() > 0 {
          // We've completed a sync cycle, notify all peers of new state. This path is
          // essential in star-topology networks where a gateway node needs to notify
          // all its out-of-date peers of the availability of a new block. This failure
          // scenario will most often crop up in private and hackathon networks with
          // degenerate connectivity, but it should be healthy for the mainnet too to
          // more reliably update peers or the local TD state.
          go pm.BroadcastBlock(head, false)
      }
  }


  ```

  如果最优的网络节点的TD值大于本地最新区块的TD值，调用pm.downloader.Synchronise(peer.id, pHead, pTd, mode)进行同步。同步完成后再屌用go pm.BroadcastBlock(head, false)，把自己最新的区块状态广播出去。

- txsyncLoop

  ```
  func (pm *ProtocolManager) txsyncLoop() {
      var (
          pending = make(map[discover.NodeID]*txsync)
          sending = false               // whether a send is active
          pack    = new(txsync)         // the pack that is being sent
          done    = make(chan error, 1) // result of the send
      )

      // send starts a sending a pack of transactions from the sync.
      send := func(s *txsync) {
          // Fill pack with transactions up to the target size.
          size := common.StorageSize(0)
          pack.p = s.p
          pack.txs = pack.txs[:0]
          for i := 0; i < len(s.txs) && size < txsyncPackSize; i++ {
              pack.txs = append(pack.txs, s.txs[i])
              size += s.txs[i].Size()
          }
          // Remove the transactions that will be sent.
          s.txs = s.txs[:copy(s.txs, s.txs[len(pack.txs):])]
          if len(s.txs) == 0 {
              delete(pending, s.p.ID())
          }
          // Send the pack in the background.
          s.p.Log().Trace("Sending batch of transactions", "count", len(pack.txs), "bytes", size)
          sending = true
          go func() { done <- pack.p.SendTransactions(pack.txs) }()
      }

      // pick chooses the next pending sync.
      pick := func() *txsync {
          if len(pending) == 0 {
              return nil
          }
          n := rand.Intn(len(pending)) + 1
          for _, s := range pending {
              if n--; n == 0 {
                  return s
              }
          }
          return nil
      }

      for {
          select {
          case s := <-pm.txsyncCh:
              pending[s.p.ID()] = s
              if !sending {
                  send(s)
              }
          case err := <-done:
              sending = false
              // Stop tracking peers that cause send failures.
              if err != nil {
                  pack.p.Log().Debug("Transaction send failed", "err", err)
                  delete(pending, pack.p.ID())
              }
              // Schedule the next send.
              if s := pick(); s != nil {
                  send(s)
              }
          case <-pm.quitSync:
              return
          }
      }
  }

  ```

  当从网络节点同步过来最新的交易数据后，本地也会把新同步下来的交易数据广播给网络中的其他节点。



#### fetch

fetcher是用来辅助同步区块数据的，记录各个区块头和区块体的同步状态，但它并不做真正下载区块数据的事情，下载的事情交由downloader来做。那fetcher具体是怎么工作的呢？
我们先看看pm.handleMsg 在收到 NewBlockHashesMsg广播通知的处理代码：

```
case msg.Code == NewBlockHashesMsg:
        var announces newBlockHashesData
        if err := msg.Decode(&announces); err != nil {
            return errResp(ErrDecode, "%v: %v", msg, err)
        }
        // Mark the hashes as present at the remote node
        for _, block := range announces {
            p.MarkBlock(block.Hash)
        }
        // Schedule all the unknown hashes for retrieval
        unknown := make(newBlockHashesData, 0, len(announces))
        for _, block := range announces {
            if !pm.blockchain.HasBlock(block.Hash, block.Number) {
                unknown = append(unknown, block)
            }
        }
        for _, block := range unknown {
            pm.fetcher.Notify(p.id, block.Hash, block.Number, time.Now(), p.RequestOneHeader, p.RequestBodies)
        }


```

从广播通知里会获取到一个newBlockHashesData的列表。newBlockHashesData只包括block的hash值和block的number值。
然后每个newBlockHashesData调用pm.fetcher.Notify(p.id, block.Hash, block.Number, time.Now(), p.RequestOneHeader, p.RequestBodies)方法，除了传入block的hash值和block的number值，还需要传入当前的时间戳，peer.go的两个函数指针。

```
func (f *Fetcher) Notify(peer string, hash common.Hash, number uint64, time time.Time,
    headerFetcher headerRequesterFn, bodyFetcher bodyRequesterFn) error {
    block := &announce{
        hash:        hash,
        number:      number,
        time:        time,
        origin:      peer,
        fetchHeader: headerFetcher,
        fetchBodies: bodyFetcher,
    }
    select {
    case f.notify <- block:
        return nil
    case <-f.quit:
        return errTerminated
    }
}


```

Notify()方法把传进来的参数拼成一个announce对象，然后send给f.notify。fetcher的loop()主回路里f.notify receive 到这个notification, 进行处理。

```
case notification := <-f.notify:
            // A block was announced, make sure the peer isn't DOSing us
            propAnnounceInMeter.Mark(1)

            count := f.announces[notification.origin] + 1
            if count > hashLimit {
                log.Debug("Peer exceeded outstanding announces", "peer", notification.origin, "limit", hashLimit)
                propAnnounceDOSMeter.Mark(1)
                break
            }
            // If we have a valid block number, check that it's potentially useful
            if notification.number > 0 {
                if dist := int64(notification.number) - int64(f.chainHeight()); dist < -maxUncleDist || dist > maxQueueDist {
                    log.Debug("Peer discarded announcement", "peer", notification.origin, "number", notification.number, "hash", notification.hash, "distance", dist)
                    propAnnounceDropMeter.Mark(1)
                    break
                }
            }
            // All is well, schedule the announce if block's not yet downloading
            if _, ok := f.fetching[notification.hash]; ok {
                break
            }
            if _, ok := f.completing[notification.hash]; ok {
                break
            }
            f.announces[notification.origin] = count
            f.announced[notification.hash] = append(f.announced[notification.hash], notification)
            if f.announceChangeHook != nil && len(f.announced[notification.hash]) == 1 {
                f.announceChangeHook(notification.hash, true)
            }
            if len(f.announced) == 1 {
                f.rescheduleFetch(fetchTimer)
            }


```

1，将收到的不满足条件的通知都丢弃掉，如果在f.fetching 状态列表里和f.completing 状态列表里，也直接返回。接着更新notification.origin 这个节点的announces 数量，添加到f.announced 等待fetch的表里。
2，如果len(f.announced[notification.hash]) == 1 说明f.announced只有这一个通知，则调用f.announceChangeHook。
3，如果len(f.announced) == 1 也说明只有一个通知，则启动fetchTimer的调度。

```
case <-fetchTimer.C:
            // At least one block's timer ran out, check for needing retrieval
            request := make(map[string][]common.Hash)

            for hash, announces := range f.announced {
                if time.Since(announces[0].time) > arriveTimeout-gatherSlack {
                    // Pick a random peer to retrieve from, reset all others
                    announce := announces[rand.Intn(len(announces))]
                    f.forgetHash(hash)

                    // If the block still didn't arrive, queue for fetching
                    if f.getBlock(hash) == nil {
                        request[announce.origin] = append(request[announce.origin], hash)
                        f.fetching[hash] = announce
                    }
                }
            }
            // Send out all block header requests
            for peer, hashes := range request {
                log.Trace("Fetching scheduled headers", "peer", peer, "list", hashes)

                // Create a closure of the fetch and schedule in on a new thread
                fetchHeader, hashes := f.fetching[hashes[0]].fetchHeader, hashes
                go func() {
                    if f.fetchingHook != nil {
                        f.fetchingHook(hashes)
                    }
                    for _, hash := range hashes {
                        headerFetchMeter.Mark(1)
                        fetchHeader(hash) // Suboptimal, but protocol doesn't allow batch header retrievals
                    }
                }()
            }
            // Schedule the next fetch if blocks are still pending
            f.rescheduleFetch(fetchTimer)


```

1，首先遍历f.announced，如果超过了arriveTimeout-gatherSlack这个时间，把这个hash对应在fetcher里面的状态都清了。
这里随机拿这个announces里面任意一个announce，为啥随机取一个呢？因为都是同一个block的hash，这个hash下的哪一个announce都是一样的。
如果发现超时了还没有没有获取到这个hash的block，则把这个announce加到request列表中，同时重新把announce放到f.fetching状态列表。
2，然后遍历request列表，request列表里面的每个网络节点过来的所有的block的hash，都会调用fetchHeader(hash)方法来获取header数据。
这个fetchHeader(hash)方法是pm.fetcher.Notify传进来的，peer.go
里面的一个全局方法。
3， 这时候NewBlockHashesMsg 的fetcher处理就结束了，最后再启动fetchTimer的调度。

三，Fetcher分析， 之FilterHeaders()
fetchHeader(hash)方法，调用了peer.go 里面的全局方法RequestOneHeader(hash common.Hash)  Send给网络节点一个GetBlockHeadersMsg 消息。
然后pm.handleMsg 收到 BlockHashesMsg广播通知

```
case msg.Code == BlockHeadersMsg:
        // A batch of headers arrived to one of our previous requests
        var headers []*types.Header
        if err := msg.Decode(&headers); err != nil {
            return errResp(ErrDecode, "msg %v: %v", msg, err)
        }
        // If no headers were received, but we're expending a DAO fork check, maybe it's that
        if len(headers) == 0 && p.forkDrop != nil {
            // Possibly an empty reply to the fork header checks, sanity check TDs
            verifyDAO := true

            // If we already have a DAO header, we can check the peer's TD against it. If
            // the peer's ahead of this, it too must have a reply to the DAO check
            if daoHeader := pm.blockchain.GetHeaderByNumber(pm.chainconfig.DAOForkBlock.Uint64()); daoHeader != nil {
                if _, td := p.Head(); td.Cmp(pm.blockchain.GetTd(daoHeader.Hash(), daoHeader.Number.Uint64())) >= 0 {
                    verifyDAO = false
                }
            }
            // If we're seemingly on the same chain, disable the drop timer
            if verifyDAO {
                p.Log().Debug("Seems to be on the same side of the DAO fork")
                p.forkDrop.Stop()
                p.forkDrop = nil
                return nil
            }
        }
        // Filter out any explicitly requested headers, deliver the rest to the downloader
        filter := len(headers) == 1
        if filter {
            // If it's a potential DAO fork check, validate against the rules
            if p.forkDrop != nil && pm.chainconfig.DAOForkBlock.Cmp(headers[0].Number) == 0 {
                // Disable the fork drop timer
                p.forkDrop.Stop()
                p.forkDrop = nil

                // Validate the header and either drop the peer or continue
                if err := misc.VerifyDAOHeaderExtraData(pm.chainconfig, headers[0]); err != nil {
                    p.Log().Debug("Verified to be on the other side of the DAO fork, dropping")
                    return err
                }
                p.Log().Debug("Verified to be on the same side of the DAO fork")
                return nil
            }
            // Irrelevant of the fork checks, send the header to the fetcher just in case
            headers = pm.fetcher.FilterHeaders(p.id, headers, time.Now())
        }
        if len(headers) > 0 || !filter {
            err := pm.downloader.DeliverHeaders(p.id, headers)
            if err != nil {
                log.Debug("Failed to deliver headers", "err", err)
            }
        }


```

如果不是硬分叉的daoHeader，同时len(headers) == 1，则执行pm.fetcher.FilterHeaders(p.id, headers, time.Now())方法

```
func (f *Fetcher) FilterHeaders(peer string, headers []*types.Header, time time.Time) []*types.Header {
    log.Trace("Filtering headers", "peer", peer, "headers", len(headers))

    // Send the filter channel to the fetcher
    filter := make(chan *headerFilterTask)

    select {
    case f.headerFilter <- filter:
    case <-f.quit:
        return nil
    }
    // Request the filtering of the header list
    select {
    case filter <- &headerFilterTask{peer: peer, headers: headers, time: time}:
    case <-f.quit:
        return nil
    }
    // Retrieve the headers remaining after filtering
    select {
    case task := <-filter:
        return task.headers
    case <-f.quit:
        return nil
    }
}


```

send 一个filter 到f.headerFilter，fetcher的loop()主回路里f.headerFilter receive 到这个filter，进行处理。

```
case filter := <-f.headerFilter:
            // Headers arrived from a remote peer. Extract those that were explicitly
            // requested by the fetcher, and return everything else so it's delivered
            // to other parts of the system.
            var task *headerFilterTask
            select {
            case task = <-filter:
            case <-f.quit:
                return
            }
            headerFilterInMeter.Mark(int64(len(task.headers)))

            // Split the batch of headers into unknown ones (to return to the caller),
            // known incomplete ones (requiring body retrievals) and completed blocks.
            unknown, incomplete, complete := []*types.Header{}, []*announce{}, []*types.Block{}
            for _, header := range task.headers {
                hash := header.Hash()

                // Filter fetcher-requested headers from other synchronisation algorithms
                if announce := f.fetching[hash]; announce != nil && announce.origin == task.peer && f.fetched[hash] == nil && f.completing[hash] == nil && f.queued[hash] == nil {
                    // If the delivered header does not match the promised number, drop the announcer
                    if header.Number.Uint64() != announce.number {
                        log.Trace("Invalid block number fetched", "peer", announce.origin, "hash", header.Hash(), "announced", announce.number, "provided", header.Number)
                        f.dropPeer(announce.origin)
                        f.forgetHash(hash)
                        continue
                    }
                    // Only keep if not imported by other means
                    if f.getBlock(hash) == nil {
                        announce.header = header
                        announce.time = task.time

                        // If the block is empty (header only), short circuit into the final import queue
                        if header.TxHash == types.DeriveSha(types.Transactions{}) && header.UncleHash == types.CalcUncleHash([]*types.Header{}) {
                            log.Trace("Block empty, skipping body retrieval", "peer", announce.origin, "number", header.Number, "hash", header.Hash())

                            block := types.NewBlockWithHeader(header)
                            block.ReceivedAt = task.time

                            complete = append(complete, block)
                            f.completing[hash] = announce
                            continue
                        }
                        // Otherwise add to the list of blocks needing completion
                        incomplete = append(incomplete, announce)
                    } else {
                        log.Trace("Block already imported, discarding header", "peer", announce.origin, "number", header.Number, "hash", header.Hash())
                        f.forgetHash(hash)
                    }
                } else {
                    // Fetcher doesn't know about it, add to the return list
                    unknown = append(unknown, header)
                }
            }
            headerFilterOutMeter.Mark(int64(len(unknown)))
            select {
            case filter <- &headerFilterTask{headers: unknown, time: task.time}:
            case <-f.quit:
                return
            }
            // Schedule the retrieved headers for body completion
            for _, announce := range incomplete {
                hash := announce.header.Hash()
                if _, ok := f.completing[hash]; ok {
                    continue
                }
                f.fetched[hash] = append(f.fetched[hash], announce)
                if len(f.fetched) == 1 {
                    f.rescheduleComplete(completeTimer)
                }
            }
            // Schedule the header-only blocks for import
            for _, block := range complete {
                if announce := f.completing[block.Hash()]; announce != nil {
                    f.enqueue(announce.origin, block)
                }
            }


```

1，遍历headerFilter里面的各个header，如果在 f.fetching状态列表，且不在f.fetched状态列表和 f.completing状态列表，就继续进行过滤，否则塞进unknown队列 发送给filter，FilterHeaders里面task 接收到filter，并作为FilterHeaders的返回值返回。
2，如果发现这个header的number和从f.fetching状态列表取到的announce的number不一样，说明有可能收到一个伪造的区块通知，此时就要把这个可能的伪造节点和可能的伪造的hash抛弃，另可错杀，不能放过。
3，如果本节点已经有这个hash的block，则放弃这个hash。如果这个block里面没有任何交易也没有任何叔区块，则把这个hash放入complete列表同时加入f.completing状态列表，否则放入incomplete列表。
4，在incomplete列表里面，且不在f.completing状态列表里，则加入f.fetched状态列表，启动completeTimer的调度。
5，在complete列表里面，同时也在f.completing状态列表，则调用f.enqueue(announce.origin, block)方法。

```
case <-completeTimer.C:
            // At least one header's timer ran out, retrieve everything
            request := make(map[string][]common.Hash)

            for hash, announces := range f.fetched {
                // Pick a random peer to retrieve from, reset all others
                announce := announces[rand.Intn(len(announces))]
                f.forgetHash(hash)

                // If the block still didn't arrive, queue for completion
                if f.getBlock(hash) == nil {
                    request[announce.origin] = append(request[announce.origin], hash)
                    f.completing[hash] = announce
                }
            }
            // Send out all block body requests
            for peer, hashes := range request {
                log.Trace("Fetching scheduled bodies", "peer", peer, "list", hashes)

                // Create a closure of the fetch and schedule in on a new thread
                if f.completingHook != nil {
                    f.completingHook(hashes)
                }
                bodyFetchMeter.Mark(int64(len(hashes)))
                go f.completing[hashes[0]].fetchBodies(hashes)
            }
            // Schedule the next fetch if blocks are still pending
            f.rescheduleComplete(completeTimer)

```

1，首先遍历f.fetched，hash对应在fetcher里面的状态都清了。
如果发现超时了还没有没有获取到这个hash的block，则把这个announce加到request列表中，同时重新把announce放到f.completing状态列表。
2，然后遍历request列表，request列表里面的每个网络节点过来的所有的block的hash，都会调用fetchBodies(hashes)方法来获取区块body数据。这个fetchBodies(hashes)方法是peer.go里面的一个全局方法。
3， 这时候BlockHashesMsg 的fetcher处理就结束了，最后再启动completeTimer循环调度。

四，Fetcher分析， 之FilterBodies() ，Enqueue(），
1，fetchBodies(hash)方法，调用了peer.go 里面的全局方法RequestBodies(hashes []common.Hash) Send给网络节点一个GetBlockBodiesMsg 消息。
2，然后pm.handleMsg 会收到 BlockBodiesMsg广播通知。
3，执行 pm.fetcher.FilterBodies(p.id, trasactions, uncles, time.Now())。
接下来就和FilterHeaders()流程类似，一顿啪啪啪验证，一顿啪啪啪改变状态，一顿啪啪啪通道跳转
4，庆幸的是，走完FilterBodies()就完事了，不用在走timer调度，也不用再发网络请求了。
5，在FilterHeaders()和FilterBodies()最后都走到了f.enqueue(announce.origin, block)方法

```
func (f *Fetcher) enqueue(peer string, block *types.Block) {
    hash := block.Hash()

    // Ensure the peer isn't DOSing us
    count := f.queues[peer] + 1
    if count > blockLimit {
        log.Debug("Discarded propagated block, exceeded allowance", "peer", peer, "number", block.Number(), "hash", hash, "limit", blockLimit)
        propBroadcastDOSMeter.Mark(1)
        f.forgetHash(hash)
        return
    }
    // Discard any past or too distant blocks
    if dist := int64(block.NumberU64()) - int64(f.chainHeight()); dist < -maxUncleDist || dist > maxQueueDist {
        log.Debug("Discarded propagated block, too far away", "peer", peer, "number", block.Number(), "hash", hash, "distance", dist)
        propBroadcastDropMeter.Mark(1)
        f.forgetHash(hash)
        return
    }
    // Schedule the block for future importing
    if _, ok := f.queued[hash]; !ok {
        op := &inject{
            origin: peer,
            block:  block,
        }
        f.queues[peer] = count
        f.queued[hash] = op
        f.queue.Push(op, -float32(block.NumberU64()))
        if f.queueChangeHook != nil {
            f.queueChangeHook(op.block.Hash(), true)
        }
        log.Debug("Queued propagated block", "peer", peer, "number", block.Number(), "hash", hash, "queued", f.queue.Size())
    }
}

```

过滤掉太远的区块。并把hash加入到f.queue列表中。
在loop主回路里面遍历f.queue列表，并把列表中的block insert到本地的block chain中。

```
func (f *Fetcher) insert(peer string, block *types.Block) {
    hash := block.Hash()

    // Run the import on a new thread
    log.Debug("Importing propagated block", "peer", peer, "number", block.Number(), "hash", hash)
    go func() {
        defer func() { f.done <- hash }()

        // If the parent's unknown, abort insertion
        parent := f.getBlock(block.ParentHash())
        if parent == nil {
            log.Debug("Unknown parent of propagated block", "peer", peer, "number", block.Number(), "hash", hash, "parent", block.ParentHash())
            return
        }
        // Quickly validate the header and propagate the block if it passes
        switch err := f.verifyHeader(block.Header()); err {
        case nil:
            // All ok, quickly propagate to our peers
            propBroadcastOutTimer.UpdateSince(block.ReceivedAt)
            go f.broadcastBlock(block, true)

        case consensus.ErrFutureBlock:
            // Weird future block, don't fail, but neither propagate

        default:
            // Something went very wrong, drop the peer
            log.Debug("Propagated block verification failed", "peer", peer, "number", block.Number(), "hash", hash, "err", err)
            f.dropPeer(peer)
            return
        }
        // Run the actual import and log any issues
        if _, err := f.insertChain(types.Blocks{block}); err != nil {
            log.Debug("Propagated block import failed", "peer", peer, "number", block.Number(), "hash", hash, "err", err)
            return
        }
        // If import succeeded, broadcast the block
        propAnnounceOutTimer.UpdateSince(block.ReceivedAt)
        go f.broadcastBlock(block, false)

        // Invoke the testing hook if needed
        if f.importedHook != nil {
            f.importedHook(block)
        }
    }()
}


```

首先调用共识引擎的方法f.verifyHeader(block.Header())，验证blockHeader的有效性。
如果没问题就广播出去，告诉全世界我的区块链更新了一个新区块。
然后调用f.insertChain(types.Blocks{block}) 插入本地区块链。
插入成功，最后再广播一次(这是多么的自恋啊)，这次只广播block的hash。

总结
fetcher.go 作为以太坊同步区块的一个辅助类，它的职责就是层层把关，层层过滤，抵制无效的区块进入，杜绝无用的同步请求。这块代码很多很乱，第一次看可能会有点晕，第二次看可能还是很晕，多看几次可能还会晕😄，不过只要知道它做什么就好了。





#### Downloader

一，启动Downloader
ProtocolManager初始化的时候会进行Downloader的初始化：

```
func New(mode SyncMode, stateDb ethdb.Database, mux *event.TypeMux, chain BlockChain, lightchain LightChain, dropPeer peerDropFn) *Downloader {
    if lightchain == nil {
        lightchain = chain
    }

    dl := &Downloader{
        mode:           mode,
        stateDB:        stateDb,
        mux:            mux,
        queue:          newQueue(),
        peers:          newPeerSet(),
        rttEstimate:    uint64(rttMaxEstimate),
        rttConfidence:  uint64(1000000),
        blockchain:     chain,
        lightchain:     lightchain,
        dropPeer:       dropPeer,
        headerCh:       make(chan dataPack, 1),
        bodyCh:         make(chan dataPack, 1),
        receiptCh:      make(chan dataPack, 1),
        bodyWakeCh:     make(chan bool, 1),
        receiptWakeCh:  make(chan bool, 1),
        headerProcCh:   make(chan []*types.Header, 1),
        quitCh:         make(chan struct{}),
        stateCh:        make(chan dataPack),
        stateSyncStart: make(chan *stateSync),
        trackStateReq:  make(chan *stateReq),
    }
    go dl.qosTuner()
    go dl.stateFetcher()
    return dl
}

```

首先初始化Downloader对象的成员，然后启动dl.qosTuner() goroutine计算请求回路时间，启动dl.stateFetcher() goroutine 开启Downloader状态监控。

ProtocolManager收到新的区块消息广播或者有新的P2P网络节点加入的时候会调用ProtocolManager的 synchronise(peer *peer)方法，这时候会调用Downloader的Synchronise(peer.id, pHead, pTd, mode)方法。

Synchronise方法，重置d.queue和d.peers，清空d.bodyWakeCh, d.receiptWakeCh，d.headerCh, d.bodyCh, d.receiptCh，d.headerProcCh。调用d.syncWithPeer(p, hash, td)方法：

```
func (d *Downloader) syncWithPeer(p *peerConnection, hash common.Hash, td *big.Int) (err error) {
    d.mux.Post(StartEvent{})
    defer func() {
        // reset on error
        if err != nil {
            d.mux.Post(FailedEvent{err})
        } else {
            d.mux.Post(DoneEvent{})
        }
    }()
    if p.version < 62 {
        return errTooOld
    }

    log.Debug("Synchronising with the network", "peer", p.id, "eth", p.version, "head", hash, "td", td, "mode", d.mode)
    defer func(start time.Time) {
        log.Debug("Synchronisation terminated", "elapsed", time.Since(start))
    }(time.Now())

    // Look up the sync boundaries: the common ancestor and the target block
    latest, err := d.fetchHeight(p)
    if err != nil {
        return err
    }
    height := latest.Number.Uint64()

    origin, err := d.findAncestor(p, height)
    if err != nil {
        return err
    }
    d.syncStatsLock.Lock()
    if d.syncStatsChainHeight <= origin || d.syncStatsChainOrigin > origin {
        d.syncStatsChainOrigin = origin
    }
    d.syncStatsChainHeight = height
    d.syncStatsLock.Unlock()

    // Ensure our origin point is below any fast sync pivot point
    pivot := uint64(0)
    if d.mode == FastSync {
        if height <= uint64(fsMinFullBlocks) {
            origin = 0
        } else {
            pivot = height - uint64(fsMinFullBlocks)
            if pivot <= origin {
                origin = pivot - 1
            }
        }
    }
    d.committed = 1
    if d.mode == FastSync && pivot != 0 {
        d.committed = 0
    }
    // Initiate the sync using a concurrent header and content retrieval algorithm
    d.queue.Prepare(origin+1, d.mode)
    if d.syncInitHook != nil {
        d.syncInitHook(origin, height)
    }

    fetchers := []func() error{
        func() error { return d.fetchHeaders(p, origin+1, pivot) }, // Headers are always retrieved
        func() error { return d.fetchBodies(origin + 1) },          // Bodies are retrieved during normal and fast sync
        func() error { return d.fetchReceipts(origin + 1) },        // Receipts are retrieved during fast sync
        func() error { return d.processHeaders(origin+1, pivot, td) },
    }
    if d.mode == FastSync {
        fetchers = append(fetchers, func() error { return d.processFastSyncContent(latest) })
    } else if d.mode == FullSync {
        fetchers = append(fetchers, d.processFullSyncContent)
    }
    return d.spawnSync(fetchers)
}

```

首先调用latest, err := d.fetchHeight(p)获取到peer节点最新的区块头,这个方法有点绕，我们来分析一下：

```
func (d *Downloader) fetchHeight(p *peerConnection) (*types.Header, error) {
    p.log.Debug("Retrieving remote chain height")

    // Request the advertised remote head block and wait for the response
    head, _ := p.peer.Head()
    go p.peer.RequestHeadersByHash(head, 1, 0, false)

    ttl := d.requestTTL()
    timeout := time.After(ttl)
    for {
        select {
        case <-d.cancelCh:
            return nil, errCancelBlockFetch

        case packet := <-d.headerCh:
            // Discard anything not from the origin peer
            if packet.PeerId() != p.id {
                log.Debug("Received headers from incorrect peer", "peer", packet.PeerId())
                break
            }
            // Make sure the peer actually gave something valid
            headers := packet.(*headerPack).headers
            if len(headers) != 1 {
                p.log.Debug("Multiple headers for single request", "headers", len(headers))
                return nil, errBadPeer
            }
            head := headers[0]
            p.log.Debug("Remote head header identified", "number", head.Number, "hash", head.Hash())
            return head, nil

        case <-timeout:
            p.log.Debug("Waiting for head header timed out", "elapsed", ttl)
            return nil, errTimeout

        case <-d.bodyCh:
        case <-d.receiptCh:
            // Out of bounds delivery, ignore
        }
    }
}


```

1，调用peer.RequestHeadersByHash(head, 1, 0, false)，给网络节点发送一个GetBlockHeadersMsg的消息
2，然后阻塞住线程，直到收到d.headerCh或者timeout
3，本地节点会收到网络节点的BlockHeadersMsg的消息返回
4，调用downloader.DeliverHeaders(p.id, headers)
5，这时候会把p.id和headers打包发送给d.headerCh
6，这时候select收到d.headerCh，阻塞打开，并返回header内容

syncWithPeer() 方法接着调用 d.findAncestor(p, height)来获取本地节点和网络节点共同的祖先：

```
func (d *Downloader) findAncestor(p *peerConnection, height uint64) (uint64, error) {
    // Figure out the valid ancestor range to prevent rewrite attacks
    floor, ceil := int64(-1), d.lightchain.CurrentHeader().Number.Uint64()

    if d.mode == FullSync {
        ceil = d.blockchain.CurrentBlock().NumberU64()
    } else if d.mode == FastSync {
        ceil = d.blockchain.CurrentFastBlock().NumberU64()
    }
    if ceil >= MaxForkAncestry {
        floor = int64(ceil - MaxForkAncestry)
    }
    p.log.Debug("Looking for common ancestor", "local", ceil, "remote", height)

    // Request the topmost blocks to short circuit binary ancestor lookup
    head := ceil
    if head > height {
        head = height
    }
    from := int64(head) - int64(MaxHeaderFetch)
    if from < 0 {
        from = 0
    }
    // Span out with 15 block gaps into the future to catch bad head reports
    limit := 2 * MaxHeaderFetch / 16
    count := 1 + int((int64(ceil)-from)/16)
    if count > limit {
        count = limit
    }
    go p.peer.RequestHeadersByNumber(uint64(from), count, 15, false)

    // Wait for the remote response to the head fetch
    number, hash := uint64(0), common.Hash{}

    ttl := d.requestTTL()
    timeout := time.After(ttl)

    for finished := false; !finished; {
        select {
        case <-d.cancelCh:
            return 0, errCancelHeaderFetch

        case packet := <-d.headerCh:
            // Discard anything not from the origin peer
            if packet.PeerId() != p.id {
                log.Debug("Received headers from incorrect peer", "peer", packet.PeerId())
                break
            }
            // Make sure the peer actually gave something valid
            headers := packet.(*headerPack).headers
            if len(headers) == 0 {
                p.log.Warn("Empty head header set")
                return 0, errEmptyHeaderSet
            }
            // Make sure the peer's reply conforms to the request
            for i := 0; i < len(headers); i++ {
                if number := headers[i].Number.Int64(); number != from+int64(i)*16 {
                    p.log.Warn("Head headers broke chain ordering", "index", i, "requested", from+int64(i)*16, "received", number)
                    return 0, errInvalidChain
                }
            }
            // Check if a common ancestor was found
            finished = true
            for i := len(headers) - 1; i >= 0; i-- {
                // Skip any headers that underflow/overflow our requested set
                if headers[i].Number.Int64() < from || headers[i].Number.Uint64() > ceil {
                    continue
                }
                // Otherwise check if we already know the header or not
                if (d.mode == FullSync && d.blockchain.HasBlock(headers[i].Hash(), headers[i].Number.Uint64())) || (d.mode != FullSync && d.lightchain.HasHeader(headers[i].Hash(), headers[i].Number.Uint64())) {
                    number, hash = headers[i].Number.Uint64(), headers[i].Hash()

                    // If every header is known, even future ones, the peer straight out lied about its head
                    if number > height && i == limit-1 {
                        p.log.Warn("Lied about chain head", "reported", height, "found", number)
                        return 0, errStallingPeer
                    }
                    break
                }
            }

        case <-timeout:
            p.log.Debug("Waiting for head header timed out", "elapsed", ttl)
            return 0, errTimeout

        case <-d.bodyCh:
        case <-d.receiptCh:
            // Out of bounds delivery, ignore
        }
    }
    // If the head fetch already found an ancestor, return
    if !common.EmptyHash(hash) {
        if int64(number) <= floor {
            p.log.Warn("Ancestor below allowance", "number", number, "hash", hash, "allowance", floor)
            return 0, errInvalidAncestor
        }
        p.log.Debug("Found common ancestor", "number", number, "hash", hash)
        return number, nil
    }
    // Ancestor not found, we need to binary search over our chain
    start, end := uint64(0), head
    if floor > 0 {
        start = uint64(floor)
    }
    for start+1 < end {
        // Split our chain interval in two, and request the hash to cross check
        check := (start + end) / 2

        ttl := d.requestTTL()
        timeout := time.After(ttl)

        go p.peer.RequestHeadersByNumber(check, 1, 0, false)

        // Wait until a reply arrives to this request
        for arrived := false; !arrived; {
            select {
            case <-d.cancelCh:
                return 0, errCancelHeaderFetch

            case packer := <-d.headerCh:
                // Discard anything not from the origin peer
                if packer.PeerId() != p.id {
                    log.Debug("Received headers from incorrect peer", "peer", packer.PeerId())
                    break
                }
                // Make sure the peer actually gave something valid
                headers := packer.(*headerPack).headers
                if len(headers) != 1 {
                    p.log.Debug("Multiple headers for single request", "headers", len(headers))
                    return 0, errBadPeer
                }
                arrived = true

                // Modify the search interval based on the response
                if (d.mode == FullSync && !d.blockchain.HasBlock(headers[0].Hash(), headers[0].Number.Uint64())) || (d.mode != FullSync && !d.lightchain.HasHeader(headers[0].Hash(), headers[0].Number.Uint64())) {
                    end = check
                    break
                }
                header := d.lightchain.GetHeaderByHash(headers[0].Hash()) // Independent of sync mode, header surely exists
                if header.Number.Uint64() != check {
                    p.log.Debug("Received non requested header", "number", header.Number, "hash", header.Hash(), "request", check)
                    return 0, errBadPeer
                }
                start = check

            case <-timeout:
                p.log.Debug("Waiting for search header timed out", "elapsed", ttl)
                return 0, errTimeout

            case <-d.bodyCh:
            case <-d.receiptCh:
                // Out of bounds delivery, ignore
            }
        }
    }
    // Ensure valid ancestry and return
    if int64(start) <= floor {
        p.log.Warn("Ancestor below allowance", "number", start, "hash", hash, "allowance", floor)
        return 0, errInvalidAncestor
    }
    p.log.Debug("Found common ancestor", "number", start, "hash", hash)
    return start, nil
}

```

1，调用peer.RequestHeadersByNumber(uint64(from), count, 15, false)，获取header。这里传入 count和 15，指从本地最高的header往前数192个区块的头，每16个区块取一个区块头。为了后面select收到d.headerCh时加以验证。
2，select收到了headers，遍历header，看是否在本地是否存在这个header，如果有，并且不为空，就说明找到共同的祖先，返回祖先number
3，如果没有找到共同的祖先，再重新从本地的区块链MaxForkAncestry起的一半的位置开始取区块头，一一验证是否跟网络节点返回的header一致，如果有就说明有共同的祖先，并返回，没有的话就返回0.

继续syncWithPeer()方法，找到同步的轴心的pivot，最后把要同步的数据和同步的方法传给d.spawnSync(fetchers)，并执行。d.spawnSync(fetchers)挨个执行传入的同步方法。

二，Downloader同步数据方法
fetchHeaders()，fetchBodies() , fetchReceipts()

```
func (d *Downloader) fetchHeaders(p *peerConnection, from uint64, pivot uint64) error {
    p.log.Debug("Directing header downloads", "origin", from)
    defer p.log.Debug("Header download terminated")

    // Create a timeout timer, and the associated header fetcher
    skeleton := true            // Skeleton assembly phase or finishing up
    request := time.Now()       // time of the last skeleton fetch request
    timeout := time.NewTimer(0) // timer to dump a non-responsive active peer
    <-timeout.C                 // timeout channel should be initially empty
    defer timeout.Stop()

    var ttl time.Duration
    getHeaders := func(from uint64) {
        request = time.Now()

        ttl = d.requestTTL()
        timeout.Reset(ttl)

        if skeleton {
            p.log.Trace("Fetching skeleton headers", "count", MaxHeaderFetch, "from", from)
            go p.peer.RequestHeadersByNumber(from+uint64(MaxHeaderFetch)-1, MaxSkeletonSize, MaxHeaderFetch-1, false)
        } else {
            p.log.Trace("Fetching full headers", "count", MaxHeaderFetch, "from", from)
            go p.peer.RequestHeadersByNumber(from, MaxHeaderFetch, 0, false)
        }
    }
    // Start pulling the header chain skeleton until all is done
    getHeaders(from)

    for {
        select {
        case <-d.cancelCh:
            return errCancelHeaderFetch

        case packet := <-d.headerCh:
            // Make sure the active peer is giving us the skeleton headers
            if packet.PeerId() != p.id {
                log.Debug("Received skeleton from incorrect peer", "peer", packet.PeerId())
                break
            }
            headerReqTimer.UpdateSince(request)
            timeout.Stop()

            // If the skeleton's finished, pull any remaining head headers directly from the origin
            if packet.Items() == 0 && skeleton {
                skeleton = false
                getHeaders(from)
                continue
            }
            // If no more headers are inbound, notify the content fetchers and return
            if packet.Items() == 0 {
                // Don't abort header fetches while the pivot is downloading
                if atomic.LoadInt32(&d.committed) == 0 && pivot <= from {
                    p.log.Debug("No headers, waiting for pivot commit")
                    select {
                    case <-time.After(fsHeaderContCheck):
                        getHeaders(from)
                        continue
                    case <-d.cancelCh:
                        return errCancelHeaderFetch
                    }
                }
                // Pivot done (or not in fast sync) and no more headers, terminate the process
                p.log.Debug("No more headers available")
                select {
                case d.headerProcCh <- nil:
                    return nil
                case <-d.cancelCh:
                    return errCancelHeaderFetch
                }
            }
            headers := packet.(*headerPack).headers

            // If we received a skeleton batch, resolve internals concurrently
            if skeleton {
                filled, proced, err := d.fillHeaderSkeleton(from, headers)
                if err != nil {
                    p.log.Debug("Skeleton chain invalid", "err", err)
                    return errInvalidChain
                }
                headers = filled[proced:]
                from += uint64(proced)
            }
            // Insert all the new headers and fetch the next batch
            if len(headers) > 0 {
                p.log.Trace("Scheduling new headers", "count", len(headers), "from", from)
                select {
                case d.headerProcCh <- headers:
                case <-d.cancelCh:
                    return errCancelHeaderFetch
                }
                from += uint64(len(headers))
            }
            getHeaders(from)

        case <-timeout.C:
            if d.dropPeer == nil {
                // The dropPeer method is nil when `--copydb` is used for a local copy.
                // Timeouts can occur if e.g. compaction hits at the wrong time, and can be ignored
                p.log.Warn("Downloader wants to drop peer, but peerdrop-function is not set", "peer", p.id)
                break
            }
            // Header retrieval timed out, consider the peer bad and drop
            p.log.Debug("Header request timed out", "elapsed", ttl)
            headerTimeoutMeter.Mark(1)
            d.dropPeer(p.id)

            // Finish the sync gracefully instead of dumping the gathered data though
            for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
                select {
                case ch <- false:
                case <-d.cancelCh:
                }
            }
            select {
            case d.headerProcCh <- nil:
            case <-d.cancelCh:
            }
            return errBadPeer
        }
    }
}

```

1，getHeaders()调用peer.RequestHeadersByNumber()方法 获取网络节点的headers。
2，有两种获取方式，首先走的是skeleton方式，从查找到的共同祖先区块+192个区块位置开始，每隔192个区块，获取128个区块头。非skeleton方式，从共同祖先区块开始，获取192个区块头。
3，如果第一种方式获取不到区块头，则执行第二种获取方式，如果第二种方式还是没有获取到区块头的话，直接返回
4，如果是skeleton获取到的，调用fillHeaderSkeleton()方法加入到skeleton header chain
5，然后调整from值，再递归调用getHeaders()方法

```
func (d *Downloader) fillHeaderSkeleton(from uint64, skeleton []*types.Header) ([]*types.Header, int, error) {
    log.Debug("Filling up skeleton", "from", from)
    d.queue.ScheduleSkeleton(from, skeleton)

    var (
        deliver = func(packet dataPack) (int, error) {
            pack := packet.(*headerPack)
            return d.queue.DeliverHeaders(pack.peerId, pack.headers, d.headerProcCh)
        }
        expire   = func() map[string]int { return d.queue.ExpireHeaders(d.requestTTL()) }
        throttle = func() bool { return false }
        reserve  = func(p *peerConnection, count int) (*fetchRequest, bool, error) {
            return d.queue.ReserveHeaders(p, count), false, nil
        }
        fetch    = func(p *peerConnection, req *fetchRequest) error { return p.FetchHeaders(req.From, MaxHeaderFetch) }
        capacity = func(p *peerConnection) int { return p.HeaderCapacity(d.requestRTT()) }
        setIdle  = func(p *peerConnection, accepted int) { p.SetHeadersIdle(accepted) }
    )
    err := d.fetchParts(errCancelHeaderFetch, d.headerCh, deliver, d.queue.headerContCh, expire,
        d.queue.PendingHeaders, d.queue.InFlightHeaders, throttle, reserve,
        nil, fetch, d.queue.CancelHeaders, capacity, d.peers.HeaderIdlePeers, setIdle, "headers")

    log.Debug("Skeleton fill terminated", "err", err)

    filled, proced := d.queue.RetrieveHeaders()
    return filled, proced, err
}


```

a) 把skeleton的headers加入queue.ScheduleSkeleton调度队列，
b) 然后执行d.fetchParts()方法。
d.fetchParts()方法主要做了这几件事情
1，对收到的headers执行d.queue.DeliverHeaders()方法。
2，如果d.queue.PendingHeaders有pending的headers，调用d.peers.HeaderIdlePeers获取到idle的peers
3，调用d.queue.ReserveHeaders把pending的headers储备到idle的peers里面
4，用idle的peers调用p.FetchHeaders(req.From, MaxHeaderFetch)去获取headers
c) 最后执行d.queue.RetrieveHeaders()，获取到filled进去的headers

其他同步区块数据的方法d.fetchBodies() , d.fetchReceipts() 和fetchHeaders()流程类似，还更简单一些。

三，Downloader同步数据过程
d.processHeaders(), d.processFastSyncContent(latest) , d.processFullSyncContent
1，d.processHeaders() 方法

```
func (d *Downloader) processHeaders(origin uint64, pivot uint64, td *big.Int) error {
    // Keep a count of uncertain headers to roll back
    rollback := []*types.Header{}
    defer func() {
        if len(rollback) > 0 {
            // Flatten the headers and roll them back
            hashes := make([]common.Hash, len(rollback))
            for i, header := range rollback {
                hashes[i] = header.Hash()
            }
            lastHeader, lastFastBlock, lastBlock := d.lightchain.CurrentHeader().Number, common.Big0, common.Big0
            if d.mode != LightSync {
                lastFastBlock = d.blockchain.CurrentFastBlock().Number()
                lastBlock = d.blockchain.CurrentBlock().Number()
            }
            d.lightchain.Rollback(hashes)
            curFastBlock, curBlock := common.Big0, common.Big0
            if d.mode != LightSync {
                curFastBlock = d.blockchain.CurrentFastBlock().Number()
                curBlock = d.blockchain.CurrentBlock().Number()
            }
            log.Warn("Rolled back headers", "count", len(hashes),
                "header", fmt.Sprintf("%d->%d", lastHeader, d.lightchain.CurrentHeader().Number),
                "fast", fmt.Sprintf("%d->%d", lastFastBlock, curFastBlock),
                "block", fmt.Sprintf("%d->%d", lastBlock, curBlock))
        }
    }()

    // Wait for batches of headers to process
    gotHeaders := false

    for {
        select {
        case <-d.cancelCh:
            return errCancelHeaderProcessing

        case headers := <-d.headerProcCh:
            // Terminate header processing if we synced up
            if len(headers) == 0 {
                // Notify everyone that headers are fully processed
                for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
                    select {
                    case ch <- false:
                    case <-d.cancelCh:
                    }
                }
                if d.mode != LightSync {
                    head := d.blockchain.CurrentBlock()
                    if !gotHeaders && td.Cmp(d.blockchain.GetTd(head.Hash(), head.NumberU64())) > 0 {
                        return errStallingPeer
                    }
                }
                if d.mode == FastSync || d.mode == LightSync {
                    head := d.lightchain.CurrentHeader()
                    if td.Cmp(d.lightchain.GetTd(head.Hash(), head.Number.Uint64())) > 0 {
                        return errStallingPeer
                    }
                }
                // Disable any rollback and return
                rollback = nil
                return nil
            }
            // Otherwise split the chunk of headers into batches and process them
            gotHeaders = true

            for len(headers) > 0 {
                // Terminate if something failed in between processing chunks
                select {
                case <-d.cancelCh:
                    return errCancelHeaderProcessing
                default:
                }
                // Select the next chunk of headers to import
                limit := maxHeadersProcess
                if limit > len(headers) {
                    limit = len(headers)
                }
                chunk := headers[:limit]

                // In case of header only syncing, validate the chunk immediately
                if d.mode == FastSync || d.mode == LightSync {
                    // Collect the yet unknown headers to mark them as uncertain
                    unknown := make([]*types.Header, 0, len(headers))
                    for _, header := range chunk {
                        if !d.lightchain.HasHeader(header.Hash(), header.Number.Uint64()) {
                            unknown = append(unknown, header)
                        }
                    }
                    // If we're importing pure headers, verify based on their recentness
                    frequency := fsHeaderCheckFrequency
                    if chunk[len(chunk)-1].Number.Uint64()+uint64(fsHeaderForceVerify) > pivot {
                        frequency = 1
                    }
                    if n, err := d.lightchain.InsertHeaderChain(chunk, frequency); err != nil {
                        // If some headers were inserted, add them too to the rollback list
                        if n > 0 {
                            rollback = append(rollback, chunk[:n]...)
                        }
                        log.Debug("Invalid header encountered", "number", chunk[n].Number, "hash", chunk[n].Hash(), "err", err)
                        return errInvalidChain
                    }
                    // All verifications passed, store newly found uncertain headers
                    rollback = append(rollback, unknown...)
                    if len(rollback) > fsHeaderSafetyNet {
                        rollback = append(rollback[:0], rollback[len(rollback)-fsHeaderSafetyNet:]...)
                    }
                }
                // Unless we're doing light chains, schedule the headers for associated content retrieval
                if d.mode == FullSync || d.mode == FastSync {
                    // If we've reached the allowed number of pending headers, stall a bit
                    for d.queue.PendingBlocks() >= maxQueuedHeaders || d.queue.PendingReceipts() >= maxQueuedHeaders {
                        select {
                        case <-d.cancelCh:
                            return errCancelHeaderProcessing
                        case <-time.After(time.Second):
                        }
                    }
                    // Otherwise insert the headers for content retrieval
                    inserts := d.queue.Schedule(chunk, origin)
                    if len(inserts) != len(chunk) {
                        log.Debug("Stale headers")
                        return errBadPeer
                    }
                }
                headers = headers[limit:]
                origin += uint64(limit)
            }
            // Signal the content downloaders of the availablility of new tasks
            for _, ch := range []chan bool{d.bodyWakeCh, d.receiptWakeCh} {
                select {
                case ch <- true:
                default:
                }
            }
        }
    }
}

```

1，收到从fetchHeaders()方法 中d.headerProcCh发送过来的headers
2，如果是FastSync或者LightSync模式，直接调用lightchain.InsertHeaderChain(chunk, frequency)插入到headerChain。
3，如果是FullSync或者FastSyn模式，调用d.queue.Schedule(chunk, origin)，放入downloader.queue来调度

2，processFastSyncContent() 方法

```
func (d *Downloader) processFastSyncContent(latest *types.Header) error {
    // Start syncing state of the reported head block. This should get us most of
    // the state of the pivot block.
    stateSync := d.syncState(latest.Root)
    defer stateSync.Cancel()
    go func() {
        if err := stateSync.Wait(); err != nil && err != errCancelStateFetch {
            d.queue.Close() // wake up WaitResults
        }
    }()
    // Figure out the ideal pivot block. Note, that this goalpost may move if the
    // sync takes long enough for the chain head to move significantly.
    pivot := uint64(0)
    if height := latest.Number.Uint64(); height > uint64(fsMinFullBlocks) {
        pivot = height - uint64(fsMinFullBlocks)
    }
    // To cater for moving pivot points, track the pivot block and subsequently
    // accumulated download results separatey.
    var (
        oldPivot *fetchResult   // Locked in pivot block, might change eventually
        oldTail  []*fetchResult // Downloaded content after the pivot
    )
    for {
        // Wait for the next batch of downloaded data to be available, and if the pivot
        // block became stale, move the goalpost
        results := d.queue.Results(oldPivot == nil) // Block if we're not monitoring pivot staleness
        if len(results) == 0 {
            // If pivot sync is done, stop
            if oldPivot == nil {
                return stateSync.Cancel()
            }
            // If sync failed, stop
            select {
            case <-d.cancelCh:
                return stateSync.Cancel()
            default:
            }
        }
        if d.chainInsertHook != nil {
            d.chainInsertHook(results)
        }
        if oldPivot != nil {
            results = append(append([]*fetchResult{oldPivot}, oldTail...), results...)
        }
        // Split around the pivot block and process the two sides via fast/full sync
        if atomic.LoadInt32(&d.committed) == 0 {
            latest = results[len(results)-1].Header
            if height := latest.Number.Uint64(); height > pivot+2*uint64(fsMinFullBlocks) {
                log.Warn("Pivot became stale, moving", "old", pivot, "new", height-uint64(fsMinFullBlocks))
                pivot = height - uint64(fsMinFullBlocks)
            }
        }
        P, beforeP, afterP := splitAroundPivot(pivot, results)
        if err := d.commitFastSyncData(beforeP, stateSync); err != nil {
            return err
        }
        if P != nil {
            // If new pivot block found, cancel old state retrieval and restart
            if oldPivot != P {
                stateSync.Cancel()

                stateSync = d.syncState(P.Header.Root)
                defer stateSync.Cancel()
                go func() {
                    if err := stateSync.Wait(); err != nil && err != errCancelStateFetch {
                        d.queue.Close() // wake up WaitResults
                    }
                }()
                oldPivot = P
            }
            // Wait for completion, occasionally checking for pivot staleness
            select {
            case <-stateSync.done:
                if stateSync.err != nil {
                    return stateSync.err
                }
                if err := d.commitPivotBlock(P); err != nil {
                    return err
                }
                oldPivot = nil

            case <-time.After(time.Second):
                oldTail = afterP
                continue
            }
        }
        // Fast sync done, pivot commit done, full import
        if err := d.importBlockResults(afterP); err != nil {
            return err
        }
    }
}

```

1，同步最新的状态信息，的到最新的pivot值
2，不停的从d.queue 的result缓存中获取要处理的result数据
3，如果results数据为空，同时pivot也为空的时候，说明同步完成了，并返回
4，根据pivot值和results计算：pivot值对应的result，和pivot值之前的results和pivot值之后的results
5，调用commitFastSyncData把pivot值之前的results 插入本地区块链中，带上收据和交易数据
6，更新同步状态信息后，把pivot值对应的result 调用commitPivotBlock插入本地区块链中，并调用FastSyncCommitHead，记录这个pivot的hash值
7，调用d.importBlockResults把pivot值之后的results插入本地区块链中，这时候不插入区块交易收据数据。

3，processFullSyncContent()方法

```
func (d *Downloader) processFullSyncContent() error {
    for {
        results := d.queue.Results(true)
        if len(results) == 0 {
            return nil
        }
        if d.chainInsertHook != nil {
            d.chainInsertHook(results)
        }
        if err := d.importBlockResults(results); err != nil {
            return err
        }
    }
}

func (d *Downloader) importBlockResults(results []*fetchResult) error {
    // Check for any early termination requests
    if len(results) == 0 {
        return nil
    }
    select {
    case <-d.quitCh:
        return errCancelContentProcessing
    default:
    }
    // Retrieve the a batch of results to import
    first, last := results[0].Header, results[len(results)-1].Header
    log.Debug("Inserting downloaded chain", "items", len(results),
        "firstnum", first.Number, "firsthash", first.Hash(),
        "lastnum", last.Number, "lasthash", last.Hash(),
    )
    blocks := make([]*types.Block, len(results))
    for i, result := range results {
        blocks[i] = types.NewBlockWithHeader(result.Header).WithBody(result.Transactions, result.Uncles)
    }
    if index, err := d.blockchain.InsertChain(blocks); err != nil {
        log.Debug("Downloaded item processing failed", "number", results[index].Header.Number, "hash", results[index].Header.Hash(), "err", err)
        return errInvalidChain
    }
    return nil
}

```

processFullSyncContent方法比较简单：直接获取缓存的results数据，并插入到本地区块链中。

总结：
Downloader看似非常复杂，其实逻辑还好，如果没有light模式，读起来会好很多。其实light模式不太成熟，基本也没什么用。fast模式比full模式逻辑上面多了一个pivot，处理起来就复杂很多。但是fast模式在本地存储了收据数据，大大减少了区块交易验证的时间。如果要更清楚明白fast模式的原理，可以看看以太坊白皮书关于fast模式同步这一部分：[文档](https://github.com/ethereum/go-ethereum/pull/1889)
