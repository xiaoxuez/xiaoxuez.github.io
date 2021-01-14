---
title: eth-contracts-projects
categories:
  - eth
date: 2020-12-17 15:20:57
tags:
---
### 流动性池子兑换系列

- Uniswap

  初始化池子时设定x y币种的数量，之后将按照x * y = k，乘积不变的方式进行兑换；价格的波动主要因为兑换时手续费的产生；

  例如当前x10个，y20个，k=10 * 20 =200；

  用户A要兑换y, 没有手续费的情况下，使用x 5个可兑换的y = 20 - (200 / 15) = 6.77777，兑换后x 15个，y 13.33333个，相乘还是200 ；但是存在手续费的时候，5个x可兑换的y将会减少，假设只能兑换6个y；那么兑换后x15,y14,相乘等于210，K将变大； 在当前这样的情况，再使用5个x去兑换y,就算没有手续费，也只能兑换 4.5个y；也就是说y的价格涨了….

  手续费将会分摊给提供流动性的用户；

  相当于是买得更多的那个币种的价格将增加，并且Uniswap规定x y中其中一个必须是ETH。也就是 **Uniswap的交易合约中的ETH和A代币的相对供应理解为A代币的供需关系，这种供需关系决定了A代币与ETH之间的汇率**。

- Balancer

  跟Uniswap比起来，uniswap像是balancer的特殊情况；balancer支持池子中的币种为多个，且不规定必须有ETH

  另外Balancer，对池子中提供流动性的用户将会额外派发BAL代币奖励，不过仅限于是token是被包含在白名单中的，细节可参考自[这里](https://forum.balancer.finance/t/proposing-the-token-whitelist-for-bal-distribution/55)

- Curve.fi

  稳定币版的uniswap；提供稳定币的兑换；它基于稳定币的特性，提供了低滑点和低手续费。比如，使用DAI兑换USDC， 对比uniswap上的兑换，将换到更多的USDC；

  整个使用的话，在curve上会有许多个池子，比如compound的稳定币池，yfi的稳定币池子等.. 然后将稳定币放入其中一个池子，将会获得流动性凭证某LP代币，然后再将LP代币放入对应的mint池，就能获得CRV奖励了；

  在curve.fi上提供的c(compond)/y(yfi)池等，可以放入原生的DAI，也可以放入cDAI/yDAI(在存款时选择deposit wrapped)；如果放入的是原生的DAI,Curve会自动将你的DAI转换成cDAI或yDAI；

  这里贴一段转换的代码

  ```
  @public
  @nonreentrant('lock')
  def add_liquidity(uamounts: uint256[N_COINS], min_mint_amount: uint256):
      tethered: bool[N_COINS] = TETHERED
      amounts: uint256[N_COINS] = ZEROS

      for i in range(N_COINS):
          uamount: uint256 = uamounts[i]

          if uamount > 0:
              # Transfer the underlying coin from owner
              if tethered[i]:
                  USDT(self.underlying_coins[i]).transferFrom(
                      msg.sender, self, uamount)
              else:
                  assert_modifiable(ERC20(self.underlying_coins[i])\
                      .transferFrom(msg.sender, self, uamount))

              # Mint if needed, 调用ytoken合约存储，
              ERC20(self.underlying_coins[i]).approve(self.coins[i], uamount)
              yERC20(self.coins[i]).deposit(uamount)
              amounts[i] = yERC20(self.coins[i]).balanceOf(self)
              ERC20(self.coins[i]).approve(self.curve, amounts[i])
  		//并将获得的ytoken, 存入合约中
      Curve(self.curve).add_liquidity(amounts, min_mint_amount)

      tokens: uint256 = ERC20(self.token).balanceOf(self)
      assert_modifiable(ERC20(self.token).transfer(msg.sender, tokens))

  ```

  那么，exchange的时候，如果exchange的是原始token（如DAI），会依次调用ytoken合约的`deposit`和`withdraw`方法将一个原始token存储进去，再换出另外一个原始token



  另外，在deposit的时候，通过锁仓一部分crv可以加速收益，最多可以加速2.5%（这个锁多少个能达到最大的算法暂时我还没看），然后锁仓可以获得ve.crv，ve.crv可以给gov里的proposal投票，持有ve.crv 2500颗可以提proposal，不过好像是线性减少，锁仓一次获得的ve.crv在时间流逝下可以投的会越来越少(ve.crv数量变少吗)。

### 复合资产

- Synthetic

  超额抵押SNX，以获得债务sUSD，复合资产包括sUSD之外，还包括sBTC, iBTC， i开头做空币主要是通过反向价格实现，通过chainlink的价格预言机，来整体计算债务池的价值变化。个人债务将按照比例挂钩总债务池，当总债务池变多，个人债务也将变多。更多细节可以在专门介绍Synthetix的那篇文章里看到。

- UMA

  参考可见[连接](https://www.zilian8.com/384427.html)

  大概总结，通过抵押ETH可获得UMA合约的erc20（到期时锚定美元价格，感觉像是期货），在到期之间的时间可以自由买卖。





### Farm

- sushi swap

  在uniswap的流动性证明放入池子中，可以挖出sushi币。主要是uniswap没有发币，大家也愿意追求免费的更多利益，所以sushi swap目前的量还是蛮多的。

- Yam

  番薯币。YAM 是一个由社区孵化的实验性协议，本质上是在 **AMPL 弹性供应机制（reBase）**上进行改进，弹性供应机制意味着随时根据市场情况进行**通胀或通缩**，旨在将每个 **YAM 代币的价格维持为 1 美元**。

  YAM 将每隔 **12** 小时 reBase 一次，凌晨 4 点一次，下午 4 点一次。

  但要注意的是，和 AMPL 的弹性供应模式不同，**YAM 每次供应通胀的一部分（约 10％）将被用于购买 yCRV**（一种以美元计价的高收益稳定币），并将其添加到 Yam 资金库中，并通过 Yam 社区链上去中心化治理进行控制，进一步支持价格稳定。

  目前是通过 staking 各支持的代币来挖 YAM 代币。支持的代币有**COMP、LEND、LINK、MKR、SNX、WETH、YFI 和 ETH/APML Uniswap v2 LP** 代币等



### 其余DEFI

- PieDao

  与etfs的结合实现，dao使用的aragon

- BarnBridge

  CDO产品，CDO(担保债务凭证)是把所有的可能的现金流打包在一起，并且进行重新包装，再以产品的形式投放到市场的凭证。

   链闻的相关[介绍](https://www.chainnews.com/articles/073409789038.htm)👈

  其中的风险分级，举例来说，把 1 ETH 的所有权拆分，分成高风险和低风险两个级别。假设 ETH 当前价格为 \$ 1000，然后降至 \$ 900，则高风险的部分承担的损失比例更高。相反，如果当前 1 ETH 的价格为 ​\$1000，然后升至 \$1100，高风险的部分会获得更高的收益。



- 1inch

  链上聚合交易协议。帮你做交易拆分，选择最低滑点进行交易。

  他接了很多的链上交易所，比如uniswap,balancer等等，然后你要兑换的时候，他会帮你选择最低滑点的方式去选择对应的交易所进行兑换，也就是让你尽可能减少兑换产生的损失。



- DefiDollar

  稳定币。

  会提供别的稳定币的池子，你可以选择放别的稳定币进去换得这个dUSD。比如有个池子是TUSD和DAI组成，你放100个TUSD和100个DAI，就可以获得200dUSD，然后合约会将TUSD和DAI放在Avae上赚取收益，以及放在Balancer池子里赚取收益，赚取的收益会进入earn池，池中的收益会给到持有dUSD的人，以及当TUSD和DAI的价格低于1美元的时候，将会购买更多的TUSD和DAI放入池子中，维持dUSD的价格为1；当dUSD的价格高于1美元的时候，大家也可以选择将dUSD换回TUSD和DAI。池子里也提供TUSD和DAI的转换，当TUSD和DAI存在价格差异，会刺激大家做对应的转换来维持。



- Empty Set Dollar（空集美元）

  稳定币。

  基本Rebase，又不同于Rebase;  ESD通过发行和赎回债务来负面地调整其供应，而不是通过修改用户钱包中持有的代币数量来进行调整。他的系统中会有一个coupons币，当esd的价格低于1时，就可以使用esd去购买coupons，这个操作会燃烧esd（供应量减少）。



- harvest

  与YFI类似，且相关策略也是直接调用YFI



- Aave

  - 闪电贷，无需抵押品的贷款，只要在一个交易（区块）内完成借贷还款即可。

    ```
    //code in LendingPool
    flashLoan(receiver, amount, ...) {
      //贷款所需手续费fee
      //balanceBeforeLoan 借贷前本合约余额
      //transfer amount from this to receiver，贷款金额转到接收合约地址中
      //receiver.executeOperation()，合约收到钱后，可执行相关代码，最后把钱和手续费还清
      //balanceBeforeLoan + fee == balanceAfterLoan 判断是否还清贷款
    }
    ```

    主要是通过在借贷合约方法中注入外部调用来完成闪电贷。







### 跨链

- Ren

  封装别的资产到以太坊上，实现跨链价值转移。如`wBTC`



### DAO

- aragon



### 工具

- chainlink

  价格预言机合约；

- The Graph

  数据服务，The Graph提供了检索以太坊上event数据的服务；

- API3

  跟chainlink类似，也是致力于解决外部价格的问题。不同的是，chainlink主要由几个主要节点提供服务，这几个主要节点去调用数据提供方的数据，赚取巨大收益。但是数据提供方在这个过程中是没有任何收益的，甚至他们不知道有人利用自己的接口去赚钱..所以在这里，API3从数据提供方出发，提供比较简单部署的方式，不用部署自己的以太坊节点，比如使用infura的接口等，让数据提供方能直接提供数据到以太坊中；并且整个过程会更透明。



### 保险

- Cover



### 其他

+ Tornado 

  以太坊链上的匿名转账协议,使用智能合约接受各用户的 ETH 存款起到混币功能，用户可以通过指定不同地址提币来实现匿名转账。

  ​	

  



------



阶段性工作

#### yearn vs harvest

例如DAI yfi的Vault策略是

1. 先放到yfi的DAI池(普通池)，合约拿到yDAI，
2. 使用yDAI再放Curve的yDAi池，拿到ycrv
3. 使用ycrv可挖crv，外部调用提现挖出来的crv去卖回DAI，再将DAI放回1步骤中

harvest的策略，和YFI是相同的，也是先放到yfi的DAI池... 再到Curve的y池，再卖crv

目前看起来的区别是，卖crv得到的DAI，harvest会将30%用于购买FARM(他的代币)，作为奖励池


