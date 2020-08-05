---
title: defi-compond
categories:
  - eth
date: 2020-08-05 11:22:38
tags:
---



## Defi Compound 



就CToken.sol实现的借贷合约的中基本运算进行简单源码分析。

此处不包括Comptroller相关逻辑，Comptroller合约主要是用于校验用户的操作，比如是否有足够的抵押品进行借贷等等，（借贷即挖矿的实现中，分发comp代币也在Comptroller合约里）



### 简单参数

- `reserveFactor`, 值在[0, 1]之间，提供流动性可获得利率=还款利率 \* `reserveFactor`

### 全局变量

- `borrowIndex`,  当前区块高度的借款利率
- `totalBorrows`, 所有借款(含借款利息)
- `totalReserves`, 所有流动性存款获得的利息吗？
- `accrualBlockNumber`,  更新到区块高度

在进行存储/赎回/借款/还款/清算...的第一步，就是更新这几个变量，这几个变量的值将会影响到利息等。

```
//更新的算法
 blockDelta = currentBlockNumber - accrualBlockNumber

//(区块间隔内)总借款利率 = 区块间隔 * 当前借款利率；相当于是每个区块收取利率。这里就是这段区块内收取的总利率
 simpleInterestFactor = borrowRate * blockDelta

//区块间隔内累计借款利息 = (区块间隔内)总借款利率 * 总共借款；这段区块内收取的总利息
 interestAccumulated = simpleInterestFactor * totalBorrows
 
 //总共借款 += 区块间隔内累计借款利息
 totalBorrows = interestAccumulated + totalBorrows
 
 //总共存储利息 += 区块间隔内累计借款利息 * 存储利率 
 totalReserves = interestAccumulated * reserveFactor + totalReserves
 
 //借款利率 += (区块间隔内)总借款利率 * borrowIndex
 borrowIndex = simpleInterestFactor * borrowIndex + borrowIndex
 
 //更新区块高度
 accrualBlockNumber = currentBlockNumber
```



##### 存款/铸币

- 更新全局变量

- 兑换率计算(兑换率的理解可见下方存储利息的理解处)

  ```
   exchangeRate = (totalCash + totalBorrows - totalReserves) / totalSupply
   //totalCash，资金池合约拥有的原币种数量
   //totalSupply,资金池合约发行的币种数量
  ```

- 假设存款/铸币数量为x,  调用对应的方法将x对应的资产转移到本合约，

- 其次铸币给到对应的用户地址，铸币的数量为`x / exchangeRate`

- 更新`totalSupply` +=  铸币的数量



##### 赎回

- 更新全局变量
- 兑换率计算`exchangeRate = (totalCash + totalBorrows - totalReserves) / totalSupply`
- 假设传入的是本资金池的代币数量`redeemTokensIn`，那么要赎回的原资产数量为`redeemAmount = redeemTokensIn * exchangeRateCurrent`
- 调用相关方法原资产从本合约转移到用户地址
- 假设传入的是原资产数量`redeemAmountIn`，要么对应到本合约资产代币数量为`redeemTokensIn = redeemAmountIn / exchangeRate`;要计算本合约资产代币数量，是为了修改`totalSupply`等
- 更新`totalSupply` -= 赎回的本合约代币数量(`redeemTokensIn`); 更新用户账户余额 -= 赎回的本合约代币数量(`redeemTokensIn`);



通过兑换率的计算方式，可以简单推测，

- 当存入后，假设资金池总存入不变，totalBorrows增多，那么赎回时收获的利息就会增多



##### 借款

- 更新全局变量

- 更新借款人的总欠款=（如已有欠款，欠款+利息） + 新借款数量

- 更新借款人的`interestIndex`为`borrowIndex`

- 更新全局总借款`totalBorrows` +=  新借款数量

- 调用相关方法将借款数量资产从本合约转出到用户地址

  

```
 //查询借款人总共应还款数量如下
 //update BorrowIndex to current index
 recentBorrowBalance = borrower.borrowBalance * market.borrowIndex / borrower.borrowIndex
 //欠款总额 * 全局borrowIndex / 借款时的borrowIndex
 //当全局borrowIndex相对借时涨了，那么欠款也会增加；如果跌了，那么欠款也会减少
```



##### 还款

- 更新全局变量
- 总欠款数量利息为`borrower.borrowBalance * market.borrowIndex / borrower.borrowIndex`
- 调用相关方法将还款数量资产从用户地址转入到本合约
- 更新用户剩余欠款，更新`totalBorrows` -= 还款数量



##### 清算

由清算人触发清算

- 更新全局变量
- 清算人将从自己的账户中偿还清算借款人的所有债务
- 将债务对应的借款人抵押品，转移到清算人账户中



※清算条件

这里假设用户在eth资金池中存款x个ceth（存款可直接作为抵押品），在dai资金池中借出y个dai；

这里提到`抵押因子`为可贷价值为`原抵押价值 * 抵押因子`, `抵押因子`的值在0到1之间；

- 遍历用户参与compound的所有资产，例子中为eth和dai两种
- 分别统计存款可贷价值总和，和欠款(包括利息)价值总和
  - 存款可贷价值的计算方式为`price * 抵押因子 * 兑换率 * 用户拥有的ctoken`
  -  欠款价值的计算方式为`price *  欠款`

- 所以用户的存款可贷价值总和为 `eth价格 * 抵押因子 * ceth兑换率 * x`; 欠款价值总和为`dai价格 * (y+利息)`
- 当用户的存款可贷价值总和 < 欠款价值总和时，将清算用户的抵押品(存款) (一个交易只能清算一种抵押品)





### 计算示例

假设当前为世界最初状态，一切都还没开始，没有存款，没有借款，那么相关参数的值如下

- `exchangeRate` ，假设兑换率初始值为2(当`totalSupply`=0时，使用初始值)
- `borrowIndex`，初始值为1 (代码设置)
- `totalBorrows`，所有借款为0，`totalReserves`也为0 
- `borrowRate`，假设借贷率为0.05(按块计算)
- `reserveFactor`，按照之前的假设，reverse为平台收取的费用，假设`reserveFactor`利率为0.1



1. 现在，用户A存入100eth, 可获得xeth的数量为`100 / 兑换率`，在用户A存入之前，当前是初始状态，`totalSupply`=0，所以兑换率为2。那么用户A存入100eth时可获得`100 / 2 = 50xeth `;  同时，`totalSupply`将为 `0 + 50 = 50`，`totalCash`更新为100

根据兑换率的计算公式为`(totalCash + totalBorrows - totalReserves) / totalSupply`，（`totalCash`为存储原资产eth的总和，`totalSupply`为此合约币xeth铸币总和）。假设没有借款时，`totalBorrows` ,`totalReserves`都为0， 假设用户A赎回50xeth将获得的eth为`50 * 兑换率`, 而兑换率`（100 + 0 - 0) / 50= 2`, 所以A将获得100eth；用户A发现没有发生借款，就没得利息，用户A此时并不打算赎回，他想等赚到利息了再赎回。

​		假设当前区块高度为10，`borrowIndex `按照公式`borrowRate * blockDelta * borrowIndex + borrowIndex `将更新为`（10 * 0.05） * 1 + 1 = 1.5`

2. 用户B在区块高度100处，借走了50个eth; 在此之前没有另外的借款，所以根据公式，`totalBorrows`,`totalReserves`维持不变，分别为0,0; `borrowIndex`按照公式`borrowRate * blockDelta * borrowIndex + borrowIndex `将更新为`90 * 0.05 * 1.5 + 1.5 = 8.25`, 然后在B借走50的时，B借50时的`borrowIndex`为8.25, `totalBorrows`将更新为`50`。

3. 在区块高度200处，我们来计算所有的利息；假设在此期间也并没有发生任何借款/还款/存款/赎回等操作。

   首先将更新全局变量们；距离区块高度100到现在为止，经历了100个区块；

   这100个区块的每货币借贷的利率为`100 * borrowRate = 100 * 0.05= 5`

   总共有50个借贷的货币，那么50个货币产生的利息为`50 * 5 = 250`

   `totalBorrows = totalBorrows + 利息 `，`totalBorrows`将更新为`50 + 250 = 300`

   `totalReserves = totalReserves + reserveFactor * 利息 `，  `totalReserves`将更新为`0 + 250 * 0.1 = 25`

   `borrowIndex `按照公式将更新为`5 * 8.25 + 8.25 = 49.5`（复利？）

    

   用户B在此时要偿还债务，他欠款利息为`欠款总额 * 全局borrowIndex / 借款时的borrowIndex `，按照公式套入计算为`50 * 49.5 / 8.25 = 300`。也就是说B将偿还`300 + 50 = 350eth`才能还清债务。

    用户A在此时选择赎回，此时的兑换率为`(100 + 300 - 25)/50=7.5`; 那么A将获得`50xeth * 7.5= 375eth`

    

   然后我们再来看看结论，在B还清债务后，池子里总共有`50 + 350 = 400`个eth，A赎回时拿走`375`个`eth`; `xeth`完全销毁，池子里剩余`400 - 375 = 25`eth, 刚好就是`totalReserves`的数量。完全吻合。

   虽然计算过程很复杂，甚至有点看不懂，但感觉巨神奇。



### 理解公式

##### 存储利息的理解

```
exchangeRate = (totalCash + totalBorrows - totalReserves) / totalSupply
```

为了说明方便，我们假设这是供应eth资金池，供应eth, 产生ceth。

```
//存入eth n个，可获得m个ceth，其中
m = n / exchangeRate

//赎回时，可获得的eth = m * exchangeRate
```

1. 首先

   假设初始exchangeRate=2，那么A存入100eth,将获得50ceth(100 / 2);此后，`exchangeRate =100/50 `

   假设C再存入100eth，那么C将获得的ceth为50(100 / 2),此后，`exchangeRate = (100 + 100)/ (50 + 50)`

   可以看到，当`x / y = z; `时，`(x + nz)/(y + n) `还是等于z; 可以通过简单的数学公式来验算；

```
x / y = z;
x = y * z;
x + m = (y + n)*z //当分子分母分别加上 m n，要达到左右依旧相等，那么m = nz
```

**结论：**也就是说，存入eth后，`exchangeRate`将不会发生变化，只有借贷产生利息，才会影响`exchangeRate`的值

其中，`totalCash` + `totalBorrows`  = `资金池中存入的总供应` + `借贷产生的利息`

那么`(totalCash + totalBorrows - totalReserves)`为除了市场(合约本身/中间商)收取的`Reserves`之外，所有的eth总量(包括借出去的和借利息)；

2. 其次 

   假设，当存入n个eth时`exchangeRate = (eth总供应 + 利息）/ ceth总量 `; 获得m个ceth

   当赎回`exchangeRate1 = (eth总供应1 + 利息1）/ ceth总量1 `，现在要赎回m个ceth,可获得的eth为？      

```
//可获得的eth为
m * exchangeRate1 = m *  (eth总供应1 + 利息1）/ ceth总量1
									= n / [(eth总供应 + 利息）/ ceth总量] * [(eth总供应1 + 利息1）/ ceth总量1]
									= n * [ceth总量 / (eth总供应 + 利息)] * [(eth总供应1 + 利息1）/ ceth总量1]
									//根据1的结论，可变式为
									= n * [ceth总量 / (eth总供应 + 利息)] * {[(eth总供应 + 利息) / ceth总量] + [(利息1 - 利息) / ceth总量1]}
								  = n + n * [ceth总量 / (eth总供应 + 利息)] * [(利息1 - 利息) / ceth总量1]
								  = n + m * [(利息1 - 利息) / ceth总量1
```

**结论：**也就是说，当赎回时，<font color='red'>至少可获得当时存入的n个eth，以及获得从存入时到赎回这期间产生的利息，利息的分配方式乘以拥有的m个ceth占ceth总量的比例</font>；

