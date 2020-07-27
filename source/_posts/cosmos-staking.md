---
title: cosmos-staking
categories:
  - cosmos
date: 2019-12-05 11:36:47
tags:
---



### cosmos-staking

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



### 激励

+ Proposal出块概率，假设所有验证者的总权益是100，自己是10，则自己的出块概率为10%

+ 出块奖励

  假设现在验证人的个数是10个，proposal的佣金率是1%，这个区块的投票率是100%。假设出块总奖励时1025.51020408

  + 扣税，2%， 剩余1005

  + proposal根据区块的投票率获得附属奖励(1% ~ 5%)。这是的投票率是100%，所以附属奖励时5%，所以计算每个验证者应该获得为`9x + x + x *5% = 10005`。算得`x=100`

  + 那么proposal验证者所得奖励为`100 + 100 *5%` = 105。105为proposal验证者及其委托人总得奖励，验证者获得按委托资金比例的奖励 + 佣金。假设proposal和委托人组成的委托总金额中比例为2：8，，并且佣金率为1%，那么proposal所获得的奖励为105 * 80% *1% + 105 * 20 % = 21.84。全部委托人所获得的佣金为105 -  21.84 = 83.16，委托人按照委托金比例获得奖励

  + 对于非proposal的奖励为100，假设委托金额比例为2：8，同样，验证者获得奖励为100 * 80% * 1% + 100 * 20% = 20.8，全部委托人获得奖励为79.2

  + 另外，当验证者行为出现不合规时，会进行扣钱，例如验证者出现双重签名会扣除5%总委托金，当验证者不投票时，也会扣除

    > If a validator misses more than 95% of the last 10.000 blocks, they will get slashed by 0.01%

  

另外，目前hub的验证者总共是100个，打算根据时间最后扩展至300。委托人要提现时，需要等待3周，从验证者切换到另一验证者，不需要等待3周





###  State

#### Params

```
type Params struct {
    UnbondingTime time.Duration // time duration of unbonding
    MaxValidators uint16        // maximum number of validators
    MaxEntries    uint16        // max entries for either unbonding delegation or redelegation (per pair/trio)
    BondDenom     string        // bondable coin denomination
}
```

#### Validator

validators可以有以下三种状态

+ 未绑定`Unbonded`,验证人尚不在active集合中，不能签署区块以及获得奖励，他们可以收到委托
+ 绑定`Bonded`，一旦验证者收到了足够的绑定token，他们就会在`EndBlock`时自动加入active集合，并且他们的状态会更新为`Binded`。这个是就可以签署区块和收到奖励了，他们可以继续被委托，当他们犯错时也会被扣钱。委托人要解绑代理(提现)时，需要等到解绑时间`UnbondingTime`(链的特定参数)后，在解绑时间段内，如果验证人犯错同样会被扣除相应绑定的token
+ 未绑定`Unbonding`，当验证者离开active集合的时候，无论是因为自动退出还是被扣钱等，所有委托人的`Unbonding`的开始了，他们必须等待解绑时间`UnbondingTime`才能从`BondedPool`中收到他们的token

Validators对象应首先由OperatorAddr存储和访问，该程序是验证程序操作员的SDK验证程序地址。 每个Validators对象都维护两个附加索引，以便满足slashing和validator-set 更新的所需查找。 还保留了第三个特殊索引（LastValidatorPower），该索引在整个每个块中保持不变，这与前两个索引不同，后者反映了块中的验证程序记录。

- Validators: `0x21 | OperatorAddr -> amino(validator)`
- ValidatorsByConsAddr: `0x22 | ConsAddr -> OperatorAddr`
- ValidatorsByPower: `0x23 | BigEndian(ConsensusPower) | OperatorAddr -> OperatorAddr`
- LastValidatorsPower: `0x11 OperatorAddr -> amino(ConsensusPower)`

`Validators`是主要索引-它确保每个操作员只能有一个关联的验证者，该验证程序的公钥将来可以更改。 委托人可以引用验证程序的不变运算符，而不必担心更改公钥。

`ValidatorByConsAddr`是一个附加索引，该索引启用用于slashing的查找。 当Tendermint报告证据时，它将提供验证者地址，因此需要此map来查找操作。 请注意，`ConsAddr`对应于可以从验证者的`ConsPubKey`派生的地址。

`ValidatorsByPower`是一个附加索引，它提供了一个排序列表或潜在的验证者，以快速确定当前active集。 这里`ConsensusPower`是`validator.Tokens` / 10 ^ 6。 请注意，Jailed为true的所有验证器均未存储在该索引中。

`LastValidatorsPower`是一个特殊的索引，它提供了最后一个块的绑定验证器的历史列表。 该索引在块中保持不变，但在EndBlock中进行的验证器集更新过程中进行更新。

每个验证者的状态都会保存如下

```
type Validator struct {
    OperatorAddress         sdk.ValAddress  // address of the validator's operator; bech encoded in JSON
    ConsPubKey              crypto.PubKey   // the consensus public key of the validator; bech encoded in JSON
    Jailed                  bool            // has the validator been jailed from bonded status?
    Status                  sdk.BondStatus  // validator status (bonded/unbonding/unbonded)
    Tokens                  sdk.Int         // delegated tokens (incl. self-delegation)
    DelegatorShares         sdk.Dec         // total shares issued to a validator's delegators
    Description             Description     // description terms for the validator
    UnbondingHeight         int64           // if unbonding, height at which this validator has begun unbonding
    UnbondingCompletionTime time.Time       // if unbonding, min time for the validator to complete unbonding
    Commission              Commission      // commission parameters
    MinSelfDelegation       sdk.Int         // validator's self declared minimum self delegation
}

type Commission struct {
    CommissionRates
    UpdateTime time.Time // the last time the commission rate was changed
}

CommissionRates struct {
    Rate          sdk.Dec // the commission rate charged to delegators, as a fraction
    MaxRate       sdk.Dec // maximum commission rate which validator can ever charge, as a fraction
    MaxChangeRate sdk.Dec // maximum daily increase of the validator commission, as a fraction
}

type Description struct {
    Moniker          string // name
    Identity         string // optional identity signature (ex. UPort or Keybase)
    Website          string // optional website link
    SecurityContact  string // optional email for security contact
    Details          string // optional details
}
```



#### Delegation

通过将DelegatorAddr（委托人的地址）与ValidatorAddr组合在一起来标识委托，这些委托人在store中的索引如下：

+ Delegation: `0x31 | DelegatorAddr | ValidatorAddr -> amino(delegation)`

利益相关者可以将Coins委托给验证者； 在这种情况下，他们的资金存放在“委托”数据结构中。 它由一位委托人拥有，并与一位验证人的股份（Shares）相关联。 交易的发送者是委托者。

```
type Delegation struct {
    DelegatorAddr sdk.AccAddress
    ValidatorAddr sdk.ValAddress
    Shares        sdk.Dec        // delegation shares received
}
```

#### Delegator Shares

当一个委托Token分配给验证者时，将根据动态汇率向他们分配一定数量的代理股份`Delegator Shares`，该份额是根据委托给验证者的代币总数和迄今为止发行的股票数量计算得出的

`Shares per Token = validator.TotalShares() / validator.Tokens()`

只有收到的股份数量存储在委托实例中。 当委托人然后取消授权时，根据他们当前持有的股份数量和逆汇率计算收到的代币数量：

`Tokens per Share = validator.Tokens() / validatorShares()`

这些股份只是一种会计机制。 它们不是可替代的资产。 这种机制的原因是简化了围绕slashing的记帐。 可以迭代地削减Validators的总绑定令牌，而不是迭代地削减每个委托条目的令牌，从而有效地减少了每个已发行委托人份额的价值。



#### UnbondingDelegation

委托中的shared可以解除绑定，但是必须在一段时间内以UnbondingDelegation的形式存在，如果检测到拜占庭行为，则可以减少共享。

`UnbondingDelegation` are indexed in the store as:

- UnbondingDelegation: `0x32 | DelegatorAddr | ValidatorAddr ->
  amino(unbondingDelegation)`
- UnbondingDelegationsFromValidator: `0x33 | ValidatorAddr | DelegatorAddr ->
  nil`

此处的第一个map用于查询，以查找给定委托人的所有未绑定的委托，而第二个映射用于slashing查找，以查找与需要削减的给定验证器关联的所有无约束的委托。 每次启动取消绑定时，都会创建一个UnbondingDelegation对象。

```
type UnbondingDelegation struct {
    DelegatorAddr sdk.AccAddress             // delegator
    ValidatorAddr sdk.ValAddress             // validator unbonding from operator addr
    Entries       []UnbondingDelegationEntry // unbonding delegation entries
}

type UnbondingDelegationEntry struct {
    CreationHeight int64     // height which the unbonding took place
    CompletionTime time.Time // unix time for unbonding completion
    InitialBalance sdk.Coin  // atoms initially scheduled to receive at completion
    Balance        sdk.Coin  // atoms to receive at completion
}
```

#### Redelegation

委托的绑定代币价值可以立即从源验证者重新委派给其他验证者（目标验证器）。 但是，当发生这种情况时，必须在“重新授权”对象中对其进行跟踪，如果它们的令牌导致源验证者提交的拜占庭式错误，则可以削减其份额。

`Redelegation` are indexed in the store as:

- Redelegations: `0x34 | DelegatorAddr | ValidatorSrcAddr | ValidatorDstAddr -> amino(redelegation)`
- RedelegationsBySrc: `0x35 | ValidatorSrcAddr | ValidatorDstAddr | DelegatorAddr -> nil`
- RedelegationsByDst: `0x36 | ValidatorDstAddr | ValidatorSrcAddr | DelegatorAddr -> nil`

第一个map用于查询，以查找给定委托人的所有重新授权。 第二个map用于基于ValidatorSrcAddr的slashing，而第三个map用于基于ValidatorDstAddr的slashing。

每次重新授权时都会创建一个重新授权对象。 为防止“重新授权跳跃”，在以下情况下可能不会发生重新授权：

+ （重新）委托人已经进行了另一个不成熟的重新委托，其目的是验证者的目的地（我们称其为Validator X）
+ 并且，（重新）委托人正在尝试创建新的委托人，其中此新委托人的源验证人是Validator-X。



```go
type Redelegation struct {
    DelegatorAddr    sdk.AccAddress      // delegator
    ValidatorSrcAddr sdk.ValAddress      // validator redelegation source operator addr
    ValidatorDstAddr sdk.ValAddress      // validator redelegation destination operator addr
    Entries          []RedelegationEntry // redelegation entries
}

type RedelegationEntry struct {
    CreationHeight int64     // height which the redelegation took place
    CompletionTime time.Time // unix time for redelegation completion
    InitialBalance sdk.Coin  // initial balance when redelegation started
    Balance        sdk.Coin  // current balance (current value held in destination validator)
    SharesDst      sdk.Dec   // amount of destination-validator shares created by redelegation
}
```

#### Queues

所有的队列对象都根据时间戳进行排序，首先将任何队列中使用的时间四舍五入到最接近的纳秒，然后进行排序。 使用的可排序时间格式是对RFC3339Nano的略微修改，并使用格式字符串“ 2006-01-02T15：04：05.000000000”。 值得注意的是这个形式

- right pads all zeros
- drops the time zone info (uses UTC)

在所有情况下，存储的时间戳都表示队列元素的成熟时间。

##### UnbondingDelegationQueue

For the purpose of tracking progress of unbonding delegations the unbonding
delegations queue is kept.

- UnbondingDelegation: `0x41 | format(time) -> []DVPair`

```go
type DVPair struct {
    DelegatorAddr sdk.AccAddress
    ValidatorAddr sdk.ValAddress
}
```

##### RedelegationQueue

For the purpose of tracking progress of redelegations the redelegation queue is
kept.

- UnbondingDelegation: `0x42 | format(time) -> []DVVTriplet`

```go
type DVVTriplet struct {
    DelegatorAddr    sdk.AccAddress
    ValidatorSrcAddr sdk.ValAddress
    ValidatorDstAddr sdk.ValAddress
}
```

##### ValidatorQueue

For the purpose of tracking progress of unbonding validators the validator
queue is kept.

- ValidatorQueueTime: `0x43 | format(time) -> []sdk.ValAddress`

The stored object as each key is an array of validator operator addresses from
which the validator object can be accessed.  Typically it is expected that only
a single validator record will be associated with a given timestamp however it is possible
that multiple validators exist in the queue at the same location.

