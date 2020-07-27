---
title: cosmos-abci
categories:
  - cosmos
date: 2019-11-27 19:34:24
tags:
---



### Cosmos abci

记录一下cosmos-sdk与tendermint-abci的衔接。

#### tendermint

+ rpc请求的路由在`rpc/core/routes.go`

  ```
  var Routes = map[string]*rpc.RPCFunc{
  	// subscribe/unsubscribe are reserved for websocket events.
  	"subscribe":       rpc.NewWSRPCFunc(Subscribe, "query"),
  	"unsubscribe":     rpc.NewWSRPCFunc(Unsubscribe, "query"),
  	"unsubscribe_all": rpc.NewWSRPCFunc(UnsubscribeAll, ""),
  
  	// info API
  	"health":               rpc.NewRPCFunc(Health, ""),
  	"status":               rpc.NewRPCFunc(Status, ""),
  	"net_info":             rpc.NewRPCFunc(NetInfo, ""),
  	"blockchain":           rpc.NewRPCFunc(BlockchainInfo, "minHeight,maxHeight"),
  	"genesis":              rpc.NewRPCFunc(Genesis, ""),
  	"block":                rpc.NewRPCFunc(Block, "height"),
  	"block_results":        rpc.NewRPCFunc(BlockResults, "height"),
  	"commit":               rpc.NewRPCFunc(Commit, "height"),
  	"tx":                   rpc.NewRPCFunc(Tx, "hash,prove"),
  	"tx_search":            rpc.NewRPCFunc(TxSearch, "query,prove,page,per_page"),
  	"validators":           rpc.NewRPCFunc(Validators, "height"),
  	"dump_consensus_state": rpc.NewRPCFunc(DumpConsensusState, ""),
  	"consensus_state":      rpc.NewRPCFunc(ConsensusState, ""),
  	"consensus_params":     rpc.NewRPCFunc(ConsensusParams, "height"),
  	"unconfirmed_txs":      rpc.NewRPCFunc(UnconfirmedTxs, "limit"),
  	"num_unconfirmed_txs":  rpc.NewRPCFunc(NumUnconfirmedTxs, ""),
  
  	// tx broadcast API
  	"broadcast_tx_commit": rpc.NewRPCFunc(BroadcastTxCommit, "tx"),
  	"broadcast_tx_sync":   rpc.NewRPCFunc(BroadcastTxSync, "tx"),
  	"broadcast_tx_async":  rpc.NewRPCFunc(BroadcastTxAsync, "tx"),
  //...
  }
  ```

+ 接收到交易会通过`CheckTx`进入交易池，在`mempool.clist_mempool.go`

   ```
  func (mem *CListMempool) CheckTx(tx types.Tx, cb func(*abci.Response)) (err error) {
  	return mem.CheckTxWithInfo(tx, cb, TxInfo{SenderID: UnknownPeerID})
  }
  
  func (mem *CListMempool) CheckTxWithInfo(tx types.Tx, cb func(*abci.Response), txInfo TxInfo) (err error) {
  	mem.proxyMtx.Lock()
  	// use defer to unlock mutex because application (*local client*) might panic
  	defer mem.proxyMtx.Unlock()
  
  	var (
  		memSize  = mem.Size()
  		txsBytes = mem.TxsBytes()
  		txSize   = len(tx)
  	)
    //...省略多行代码
    //CheckTxAsync最终调用的是abci.CheckTx
  	reqRes := mem.proxyAppConn.CheckTxAsync(abci.RequestCheckTx{Tx: tx})
  	//设置回调来决定要不要添加到交易池中
  	reqRes.SetCallback(mem.reqResCb(tx, txInfo.SenderID, txInfo.SenderP2PID, cb))
  
  	return nil
  }
  
  //设置的回调，即当abci.CheckTx返回时，会回调这个方法，重要的是调用resCbFirstTime
  func (mem *CListMempool) reqResCb(
  	tx []byte,peerID uint16,peerP2PID p2p.ID,	externalCb func(*abci.Response),
  ) func(res *abci.Response) {
  	return func(res *abci.Response) {
  	 //...
  		mem.resCbFirstTime(tx, peerID, peerP2PID, res)
  	//..
  	}
  }
  
  //resCbFirstTime会决定交易会不会进入到交易池中
  func (mem *CListMempool) resCbFirstTime(
  	tx []byte, peerID uint16, peerP2PID p2p.ID, res *abci.Response) {
  	switch r := res.Value.(type) {
  	case *abci.Response_CheckTx:
  		var postCheckErr error
  		if mem.postCheck != nil {
  			postCheckErr = mem.postCheck(tx, r.CheckTx)
  		}
  		//abci.CheckTx方法返回错误就不会添加到交易池中
  		if (r.CheckTx.Code == abci.CodeTypeOK) && postCheckErr == nil {
  			memTx := &mempoolTx{
  				height:    mem.height,
  				gasWanted: r.CheckTx.GasWanted,
  				tx:        tx,
  			}
  			memTx.senders.Store(peerID, true)
  			mem.addTx(memTx)
      }
      //...
  	default:
  		// ignore other messages
  	}
  }
  
  ```

+ `state/execution.go`中的`CreateProposal`方法会将交易池中的交易取出来进行打包组成区块

+ `state/execution.go`中区块组成后，执行交易在这里`execBlockOnProxyApp`方法里。

  ```
  func execBlockOnProxyApp(logger log.Logger, proxyAppConn proxy.AppConnConsensus, block *types.Block, stateDB dbm.DB) (*ABCIResponses, error) {
  	var validTxs, invalidTxs = 0, 0
  
  	txIndex := 0
  	abciResponses := NewABCIResponses(block)
  
  	//设置回调，当abci.DeliverTx执行完后，统计交易成功和失败
  	// Execute transactions and get hash.
  	proxyCb := func(req *abci.Request, res *abci.Response) {
  		if r, ok := res.Value.(*abci.Response_DeliverTx); ok {
  			// TODO: make use of res.Log
  			// TODO: make use of this info
  			// Blocks may include invalid txs.
  			txRes := r.DeliverTx
  			if txRes.Code == abci.CodeTypeOK {
  				validTxs++
  			} else {
  				logger.Debug("Invalid tx", "code", txRes.Code, "log", txRes.Log)
  				invalidTxs++
  			}
  			abciResponses.DeliverTx[txIndex] = txRes
  			txIndex++
  		}
  	}
  	proxyAppConn.SetResponseCallback(proxyCb)
  
  	commitInfo, byzVals := getBeginBlockValidatorInfo(block, stateDB)
  
    //abci.BeginBlock在这里会被调用
  	// Begin block
  	var err error
  	abciResponses.BeginBlock, err = proxyAppConn.BeginBlockSync(abci.RequestBeginBlock{
  		Hash:                block.Hash(),
  		Header:              types.TM2PB.Header(&block.Header),
  		LastCommitInfo:      commitInfo,
  		ByzantineValidators: byzVals,
  	})
  	if err != nil {
  		logger.Error("Error in proxyAppConn.BeginBlock", "err", err)
  		return nil, err
  	}
  
    //abci.DeliverTx在这里会被调用，但注意的是，就算DeliverTx结果返回错误，这里也不会影响交易入块，记录在链上的交易会包含失败的结果
  	// Run txs of block.
  	for _, tx := range block.Txs {
  		proxyAppConn.DeliverTxAsync(abci.RequestDeliverTx{Tx: tx})
  		if err := proxyAppConn.Error(); err != nil {
  			return nil, err
  		}
  	}
  
  	//abci.EndBlock在这里会被调用
  	// End block.
  	abciResponses.EndBlock, err = proxyAppConn.EndBlockSync(abci.RequestEndBlock{Height: block.Height})
  	if err != nil {
  		logger.Error("Error in proxyAppConn.EndBlock", "err", err)
  		return nil, err
  	}
  
  	logger.Info("Executed block", "height", block.Height, "validTxs", validTxs, "invalidTxs", invalidTxs)
  
  	return abciResponses, nil
  }
  
  ```

+ 以上代码`proxyAppConn`的实现之一为`abci/client/local_client.go`。

  `local_client`中主要是使用以下这个接口

  ```
  type Application interface {
  	// Info/Query Connection
  	Info(RequestInfo) ResponseInfo                // Return application info
  	SetOption(RequestSetOption) ResponseSetOption // Set application option
  	Query(RequestQuery) ResponseQuery             // Query for state
  
  	// Mempool Connection
  	CheckTx(RequestCheckTx) ResponseCheckTx // Validate a tx for the mempool
  
  	// Consensus Connection
  	InitChain(RequestInitChain) ResponseInitChain    // Initialize blockchain w validators/other info from TendermintCore
  	BeginBlock(RequestBeginBlock) ResponseBeginBlock // Signals the beginning of a block
  	DeliverTx(RequestDeliverTx) ResponseDeliverTx    // Deliver a tx for full processing
  	EndBlock(RequestEndBlock) ResponseEndBlock       // Signals the end of a block, returns changes to the validator set
  	Commit() ResponseCommit                          // Commit the state and return the application Merkle root hash
  }
  
  ```

  这个是abci的接口了。`cosmos-sdk`实现的就实现这个接口

  

  #### `cosmos-sdk` 

  在`baseapp/abci.go`里是对`Application`的实现。

  然后交易的处理都在`baseapp/baseapp.go`里。

  ```
  func (app *BaseApp) runTx(mode runTxMode, txBytes []byte, tx sdk.Tx) (result sdk.Result) {
  	//..省略代码n行
  
  	var msgs = tx.GetMsgs()
  	// x/xx/types/msg会实现这个方法
  	if err := validateBasicTxMsgs(msgs); err != nil {
  		return err.Result()
  	}
  
  	if app.anteHandler != nil {
  		var anteCtx sdk.Context
  		var msCache sdk.CacheMultiStore
  
  		// Cache wrap context before anteHandler call in case it aborts.
  		// This is required for both CheckTx and DeliverTx.
  		// Ref: https://github.com/cosmos/cosmos-sdk/issues/2772
  		//
  		// NOTE: Alternatively, we could require that anteHandler ensures that
  		// writes do not happen if aborted/failed.  This may have some
  		// performance benefits, but it'll be more difficult to get right.
  		anteCtx, msCache = app.cacheTxContext(ctx, txBytes)
  
     	//这个方法也在外面定义
  		newCtx, err := app.anteHandler(anteCtx, tx, mode == runTxModeSimulate)
  		if !newCtx.IsZero() {
  			// At this point, newCtx.MultiStore() is cache-wrapped, or something else
  			// replaced by the ante handler. We want the original multistore, not one
  			// which was cache-wrapped for the ante handler.
  			//
  			// Also, in the case of the tx aborting, we need to track gas consumed via
  			// the instantiated gas meter in the ante handler, so we update the context
  			// prior to returning.
  			ctx = newCtx.WithMultiStore(ms)
  		}
  
  		// GasMeter expected to be set in AnteHandler
  		gasWanted = ctx.GasMeter().Limit()
  
  		if err != nil {
  			res := sdk.ResultFromError(err)
  			res.GasWanted = gasWanted
  			res.GasUsed = ctx.GasMeter().GasConsumed()
  			return res
  		}
  
  		msCache.Write()
  	}
  
  	// Create a new Context based off of the existing Context with a cache-wrapped
  	// MultiStore in case message processing fails. At this point, the MultiStore
  	// is doubly cached-wrapped.
  	runMsgCtx, msCache := app.cacheTxContext(ctx, txBytes)
  	//最后是这个方法
  	result = app.runMsgs(runMsgCtx, msgs, mode)
  	result.GasWanted = gasWanted
  
  	// Safety check: don't write the cache state unless we're in DeliverTx.
  	if mode != runTxModeDeliver {
  		return result
  	}
  
  	// only update state if all messages pass
  	if result.IsOK() {
  		msCache.Write()
  	}
  
  	return result
  }
  ```

插件化里，上面牵涉到3个会调插件的地方

```
type Msg interface {

	// Return the message type.
	// Must be alphanumeric or empty.
	Route() string

	// Returns a human-readable string for the message, intended for utilization
	// within tags
	Type() string

	// ValidateBasic does a simple validation check that
	// doesn't require access to any other information.
	ValidateBasic() Error

	// Get the canonical byte representation of the Msg.
	GetSignBytes() []byte

	// Signers returns the addrs of signers that must sign.
	// CONTRACT: All signatures must be present to be valid.
	// CONTRACT: Returns addrs in some deterministic order.
	GetSigners() []AccAddress
}
```

在x/xx/types/msg.go下会实现这个接口，在CheckTx或者DeliverTx的时候都会调用`ValidateBasic`方法

 其次是，anteHandler，如果外面初始化时赋值了，CheckTx或者DeliverTx时就会执行

最后是，runMsgs，runMsg里其实是根据Tx消息的路由，找到对应的Handler进行处理，在x/xx/Handler.go会有具体处理，路由是在实例化app时进行app.AddRouter进行添加进去的。如下gaia代码

```
// RegisterRoutes registers all module routes and module querier routes
func (m *Manager) RegisterRoutes(router sdk.Router, queryRouter sdk.QueryRouter) {
	for _, module := range m.Modules {
		if module.Route() != "" {
			router.AddRoute(module.Route(), module.NewHandler())
		}
		if module.QuerierRoute() != "" {
			queryRouter.AddRoute(module.QuerierRoute(), module.NewQuerierHandler())
		}
	}
}
```

可以看到最后一句，查询路由也是在这里添加的。所以在查询，也就是调用`Query`的时候，就会走对应的路由。







## 另起一行

### 这里由更新验证者为契机，查看相关的代码，捋下结构。

#### 更新设置验证者的方式

+ genesis里进行配置
+ 发送CreateValidator交易添加验证者，`staking`模块会处理这个交易，并且在`abci.EndBlock`调用时，`staking`模块返回需要更新的验证者集合。

##### genesis里配置验证者的传递

`cosmos`启动时调用`tendermint`的node代码，传入相关配置。

```
//cosmos/server/start.go
func startInProcess(ctx *Context, appCreator AppCreator) (*node.Node, error) {
	// create & start tendermint node
	tmNode, err := node.NewNode(
		cfg,
		pvm.LoadOrGenFilePV(cfg.PrivValidatorKeyFile(), cfg.PrivValidatorStateFile()),
		nodeKey,
		proxy.NewLocalClientCreator(app),
		node.DefaultGenesisDocProviderFunc(cfg),
		node.DefaultDBProvider,
		node.DefaultMetricsProvider(cfg.Instrumentation),
		ctx.Logger.With("module", "node"),
	)
}


//tendermint/node/node.go
func NewNode(config *cfg.Config, privValidator types.PrivValidator,
	nodeKey *p2p.NodeKey, clientCreator proxy.ClientCreator,
	genesisDocProvider GenesisDocProvider, dbProvider DBProvider, metricsProvider MetricsProvider, logger log.Logger, options ...Option) (*Node, error) {

	blockStore, stateDB, err := initDBs(config, dbProvider)
	if err != nil {
		return nil, err
	}

	//state是链的相关状态，包括版本，上一个区块iD，当前验证者集合，下一个验证者集合等
	//genDoc是genesis中包含的链的状态，包括验证者集合等
	//state和genDoc都与db交互
	state, genDoc, err := LoadStateFromDBOrGenesisDocProvider(stateDB, genesisDocProvider)
	if err != nil {
		return nil, err
	}
 //...
}
```

在`tendermint`中，`state`存储在db中的key为`stateKey`，保存了链的当前相关状态。

在`cosmos`的`staking`中也应该存储了验证者集合，一会想得起来我就去看看怎么查询。



##### staking

`staking`是cosmos实现用来选择验证者集合。在`staking`的`handler.go`中可以查看相关操作。

其中通过发送`CreateValidator`交易，新建验证者，并设置相关参数。

```
  //handleMsgCreateValidator
	validator := NewValidator(msg.ValidatorAddress, msg.PubKey, msg.Description)
	commission := NewCommissionWithTime(
		msg.Commission.Rate, msg.Commission.MaxRate,
		msg.Commission.MaxChangeRate, ctx.BlockHeader().Time,
	)
	//设置佣金
	validator, err := validator.SetInitialCommission(commission)
	if err != nil {
		return err.Result()
	}

	validator.MinSelfDelegation = msg.MinSelfDelegation

```



另外，更新验证者是`abci.EndBlock`。

```
// Called every block, update validator set
func EndBlocker(ctx sdk.Context, k keeper.Keeper) []abci.ValidatorUpdate {
	validatorUpdates := k.ApplyAndReturnValidatorSetUpdates(ctx)
	return validatorUpdates
}

func (k Keeper) ApplyAndReturnValidatorSetUpdates(ctx sdk.Context) (updates []abci.ValidatorUpdate) {
	//根据powerIndex逆序排名，power越大，排名越靠前，也就是取前maxValidators位作为验证者
	iterator := sdk.KVStoreReversePrefixIterator(store, types.ValidatorsByPowerIndexKey)
	defer iterator.Close()
	for count := 0; iterator.Valid() && count < int(maxValidators); iterator.Next() {
		..
	}
}
```



