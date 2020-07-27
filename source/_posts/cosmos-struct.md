---
title: cosmos-struct
categories:
  - cosmos
date: 2019-12-09 10:48:18
tags:
---



### Cosmos结构梳理

从编写x模块开始吧！

+ 编写`Keeper`，keeper实现与数据库层交互

+ 实现`AppModule`接口

  ```
  // AppModule is the standard form for an application module
  type AppModule interface {
  	AppModuleGenesis
  
  	// registers
  	RegisterInvariants(sdk.InvariantRegistry)
  
  	// routes
  	Route() string
  	NewHandler() sdk.Handler
  	QuerierRoute() string
  	NewQuerierHandler() sdk.Querier
  
  	// ABCI
  	BeginBlock(sdk.Context, abci.RequestBeginBlock)
  	EndBlock(sdk.Context, abci.RequestEndBlock) []abci.ValidatorUpdate
  }
  
  ```

  `AppModule`主要是负责`abci`回调，包括基本的`BeginBlock`/`EndBlock`回调、以及`CheckTx`的回调实现(Msg实现的`ValidateBasic`方法)、`DeliverTx`的回调(Route + Handler组成的路由实现)、`Query`的回调，同理是(Route + Handler组成的路由实现)。

  还有是`AppModuleGenesis`的实现，包括Genesis的一些处理，如配置从Genesis中读出的初始设置。

+  实现`AppModuleBasic`接口

  ```
  type AppModuleBasic interface {
  	Name() string
  	RegisterCodec(*codec.Codec)
  
  	// genesis
  	DefaultGenesis() json.RawMessage
  	ValidateGenesis(json.RawMessage) error
  
  	// client functionality
  	RegisterRESTRoutes(context.CLIContext, *mux.Router)
  	GetTxCmd(*codec.Codec) *cobra.Command
  	GetQueryCmd(*codec.Codec) *cobra.Command
  }
  ```

  实现该接口的使用，是独立使用，不像负责`abci`回调的`AppModule`使用是与Tendermint一起的。

  `AppModuleBasic`接口主要使用在命令行工具、RestApi等地方。

#### 编写好了x模块，就开始初始化app

```
var (
	// default home directories for hub
	DefaultNodeHome = os.ExpandEnv("$HOME/.hub")

	// The module BasicManager is in charge of setting up basic,
	// non-dependant module elements, such as codec registration
	// and genesis verification.
	ModuleBasics = module.NewBasicManager(
		genutil.AppModuleBasic{},
		auth.AppModuleBasic{},
		bank.AppModuleBasic{},
		staking.AppModuleBasic{},
		mint.AppModuleBasic{},
		distr.AppModuleBasic{},
		gov.NewAppModuleBasic(paramsclient.ProposalHandler, distr.ProposalHandler),
		params.AppModuleBasic{},
		crisis.AppModuleBasic{},
		slashing.AppModuleBasic{},
		supply.AppModuleBasic{},
		ibc.AppModuleBasic{},
		user.NewAppModuleBasic(),
	)
)



NewApp(){
  // NOTE: Any module instantiated in the module manager that is later modified must be passed by reference here.
	app.manager = module.NewManager(
		genutil.NewAppModule(app.accountKeeper, app.stakingKeeper, app.BaseApp.DeliverTx),
		auth.NewAppModule(app.accountKeeper),
		bank.NewAppModule(app.bankKeeper, app.accountKeeper),
		crisis.NewAppModule(&app.crisisKeeper),
		supply.NewAppModule(app.supplyKeeper, app.accountKeeper),
		distr.NewAppModule(app.distrKeeper, app.supplyKeeper),
		gov.NewAppModule(app.govKeeper, app.supplyKeeper),
		mint.NewAppModule(app.mintKeeper),
		slashing.NewAppModule(app.slashingKeeper, app.stakingKeeper),
		staking.NewAppModule(app.stakingKeeper, app.accountKeeper, app.supplyKeeper),
		ibc.NewAppModule(app.ibcKeeper),
		user.NewAppModule(app.userKeeper),
	)

	//...
	//配置路由和handler（包括Tx和查询）
	app.manager.RegisterRoutes(app.Router(), app.QueryRouter())
	
	// 配置EndBlock\BeginBlock等回调
	app.SetInitChainer(app.InitChainer)
	app.SetBeginBlocker(app.BeginBlocker)
	//AnteHandler是特殊的实现，CheckTx和DeliverTx一般只会执行到对应的模块中，AnteHandler会在进入对应模块前执行，如验签等操作..
	app.SetAnteHandler(auth.NewAnteHandler(app.accountKeeper, app.supplyKeeper, auth.DefaultSigVerificationGasConsumer))
	app.SetEndBlocker(app.EndBlocker)
}


//ModuleBasics定义在上面，使用在main中
//main
	rootCmd := &cobra.Command{
		Use:               "hub",
		Short:             "Hub Daemon (server)",
		PersistentPreRunE: server.PersistentPreRunEFn(ctx),
	}
	rootCmd.AddCommand(server.StartCmd(ctx, newApp))
	rootCmd.AddCommand(cmd.InitCmd(ctx, cdc, app.ModuleBasics, app.DefaultNodeHome))
	rootCmd.AddCommand(cmd.AddGenesisAccountCmd(ctx, cdc, app.DefaultNodeHome))
	rootCmd.AddCommand(cmd.GenerateTxCmd(ctx, cdc, app.ModuleBasics, app.DefaultNodeHome))
	rootCmd.AddCommand(cmd.CollectGenTxsCmd(ctx, cdc, app.DefaultNodeHome))
	rootCmd.AddCommand(cmd.ValidateGenesisCmd(ctx, cdc, app.ModuleBasics))
```



 