---
title: defi-aragon
categories: 
  - eth
date: 2020-11-12 14:37:48
tags:
---

### aragon

这个DAO的工具来说，还是很好用的。各方面都封装得很好；

主要由kernel + acl + apps组成；

#### kernel

简单来说， kernel负责存储各个app的地址（app-id => address），app的部署方式为(proxy delegatecall), app的proxy中的`delegatecall(implement, msg.data)`中的implement将会到kernel中读取对应app-id的base逻辑的合约地址。

### acl

acl负责完成权限管理。app的一切修改数据的操作都需要具有对应的权限才可以。如一个transfer方法的签名为

```
function transfer(address _token, address _to, uint256 _value)
        external
        authP(TRANSFER_ROLE, arr(_token, _to, _value))
				{}
```

`authP`便是权限查询。

一个权限 Permission 主要由下面几个概念来组成

- `manager`管理者，管理者可将该权限授予某实体地址，以及取消某实体地址的该权限
- `role` 权限名称，如`TRANSFER_ROLE`
- `app`，该权限作用的app
- `entity`,拥有该权限的实体

创建一个permission时，就需要包含这几个信息；如 `app = vault, role = TRANSFER_ROLE`，则代表该权限是要使用(transfer)vault中的资产权限；创建(create)完成一个permission后，manager可通过`grant`来对权限进行授予或取消；

#### app

更详细的[文档](https://hack.aragon.org/docs/guides-custom-deploy#adding-a-vault-and-finance-instance)👈。这里列出基本的几个

- Token Manager

  可使用token来分组，配置不同会有3种场景

  - `Membership`， token不可转让，每地址只能拥有1枚
  - `Reputation` ，token不可转让，每地址没有数量限制
  - `Equity`，token可转让，每地址没有数量限制

   Token manager 使用的token是[minime](https://github.com/Giveth/minime), 这个是具有额外功能的erc20, 额外功能包括，克隆方便，保存着地址资产历史，可根据区块高度来查询余额；

- Voting

  通过token进行投票。

  创建voting app可以设置的参数包括

  - `minime token`
  - 投票通过的百分比
  - 最小参与投票的百分比
  - 投票期时长

- Vault 以及 Finance

  vault保管着一些资产，但vault没提供用户接口，需要通过Finance的用户接口来交互操作，Finance中负责提供预算，在一个阶段可提供一笔预算，通过投票可使用这笔预算，预算中的资产将由Vault支付

- Agent

  这是Vault的子类，保管着资产，并且提供与其他合约交互的接口，比如可以将这些资产放到其他的defi合约里赚取收益什么的。



### 基本的部署示例

这里将部署一个投票app，当投票通过后，使用vault中资产的例子。

1. 部署Kernel和ACL

2. 执行`kernel.initialize(acl, rootAddress)`来配置kernel中的acl实例和操作地址，该方法中会执行`acl.initialize(rootAddress)`以及`createPermission(rootAddress, aclAddress, CREATE_PERMISSIONS_ROLE, rootAddress)`创建权限管理；

   初始化时的rootAddress将会具有以及管理createPermission的权限。

3. 部署Voting app，投票通过后，将会调用vault中的transfer方法，所以voting要具有对应的权限。

4. 上面提到rootAddress有管理createPermission的权限，所以首先使用rootAddress将createPermission授权给voting，之后voting再创建transfer权限。

   > Grant the Voting app the ability to call `createPermission()`: `grantPermission(votingAppAddress, aclAddress, CREATE_PERMISSIONS_ROLE)` (must be executed by `rootAddress`)

5. 部署Vault 合约，该合约具有transfer方法

6. 所以现在voting就要创建自己去执行vault的transfer方法的权限

   > Create a new vote via the Voting app to create the `TRANSFER_ROLE` permission: `createPermission(votingAppAddress, vaultAppAddress, TRANSFER_ROLE, votingAppAddress)`

7. 当投票通过后，voting就可以调用vault中任何由 TRANSFER_ROLE 权限修饰的方法了，在这个例子中，只有`transfer`方法

8. Vault中的资产就被Voting控制了，当用户想使用Vault的资产时，在Voting app上创建一个新的执行Vault的transfer的vote，当投票通过时，transfer将会被执行

9. voting app可以取消transfer的权限



#### 详细的部署示例

模板Reputation部署过程(in templates)

```
//create token => in MiniMeTokenFactory.createCloneToken
MiniMeToken newToken = new MiniMeToken(...)    
newToken.changeController(msg.sender);        

//create dao kernal acl..
Kernel dao = Kernel(new KernelProxy(baseKernel));
dao.initialize(baseACL, _root); //kernel的initialize,上述2步骤

//初始化Vault资金池，如果使用了Agent就使用支持Agent的合约
Vault agentOrVault = _useAgentAsVault ? _installDefaultAgentApp(_dao) : _installVaultApp(_dao);

//创建Finance app
Finance finance = _installFinanceApp(_dao, agentOrVault, _financePeriod == 0 ? DEFAULT_FINANCE_PERIOD : _financePeriod);

//创建Token Manager app
TokenManager tokenManager = _installTokenManagerApp(_dao, token, TOKEN_TRANSFERABLE, TOKEN_MAX_PER_ACCOUNT);

//创建Voting app
Voting voting = _installVotingApp(_dao, token, _votingSettings);

//给初始化时的地址mint币，因为整个系统都是基于权限，所以会先创建mint权限给当前，挖完再取消掉权限
_mintTokens(_acl, tokenManager, _holders, _stakes);

//创建相关的权限
_setupPermissions(_acl, agentOrVault, voting, finance, tokenManager, _useAgentAsVault);

```

 

上面创建相关的权限的细节在这里👇

```
function _setupPermissions(
    ACL _acl,
    Vault _agentOrVault,
    Voting _voting,
    Finance _finance,
    TokenManager _tokenManager,
    bool _useAgentAsVault
)
    internal
{
    if (_useAgentAsVault) {
    		//创建voting app具有transfer vault 资产的权限
        _createAgentPermissions(_acl, Agent(_agentOrVault), _voting, _voting);
    }
    //设置finance具有transfer vault 资产的权限
    _createVaultPermissions(_acl, _agentOrVault, _finance, _voting);
    
    //设置由voting来触发finance中EXECUTE_PAYMENTS_ROLE、MANAGE_PAYMENTS_ROLE权限
    _createFinancePermissions(_acl, _finance, _voting, _voting);
    
    //设置voting具有在finance中创建payment的权限，CREATE_PAYMENTS_ROLE权限
    _createFinanceCreatePaymentsPermission(_acl, _finance, _voting, address(this));
    
    //设置voting 具有evmscriptsregistery中的REGISTRY_MANAGER_ROLE、REGISTRY_ADD_EXECUTOR_ROLE权限
    _createEvmScriptsRegistryPermissions(_acl, _voting, _voting);
    
    //设置voting具有voting的MODIFY_QUORUM_ROLE、MODIFY_SUPPORT_ROLE权限
    //以及_tokenManager具有voting的CREATE_VOTES_ROLE权限
    _createVotingPermissions(_acl, _voting, _voting, _tokenManager, _voting);
    
    
    //创建voting app 具有mint和burn token的权限,也就是投票通过后可以mint 和burn token
    _createTokenManagerPermissions(_acl, _tokenManager, _voting, _voting);
}
```



### script分析

- example来自一个由vote发起的普通new vote[交易](https://rinkeby.etherscan.io/tx/0xd707800483616e85adabe969a675b1e8833148f6505d05c3c3b1e398e1b9ef71)；这是通过aragon网页创建的一个普通投票，内容填写的是"mint"

  使用forward进行new vote时，首先要编码要调用的script为new vote，其次，new vote的签名为`function _newVote(bytes _executionScript, string _metadata)`, 需要传入vote通过后的执行script

  ```
  //对data数据进行解析
  
  0x
  d948d468 //forward
  0000000000000000000000000000000000000000000000000000000000000020
  00000000000000000000000000000000000000000000000000000000000000e0
  //script
  00000001 //executorId ，rinkeby测试网上的id好像只有1
  dcc728ad010792caef6b73ab04633a22c4a9eef4 //execute  contract address，这里是vote的合约地址
  000000c4 // call data length, 后面的数据的长度
  d5db2c80 //execute method signature， vote中要调用方法的签名，这里对应是newVote方法
  0000000000000000000000000000000000000000000000000000000000000040
  0000000000000000000000000000000000000000000000000000000000000080
  0000000000000000000000000000000000000000000000000000000000000004
  //script
  00000001 //executorId
  0000000000000000000000000000000000000000
  0000000000000000000000000000000000000000000000000000000000000000000000000000000
  46d696e74 //new vote的第二个参数,ascii为mint(字符)
  00000000000000000000000000000000000000000000000000000000
  ```

- example再看一个通过token manager创建的new vote，当投票通过后，将会给地址转账, [交易](https://rinkeby.etherscan.io/tx/0xf9678d1697a5310e57431fe554dbbf613c1f5a7646107831891fffbe0278bb5f) 

  ```
  0x
  d948d468 //forward
  0000000000000000000000000000000000000000000000000000000000000020
  00000000000000000000000000000000000000000000000000000000000000c0
  00000001 //executorId
  dcc728ad010792caef6b73ab04633a22c4a9eef4 //vote的合约地址
  000000a4 //call data length
  d948d468 //newVote方法
  0000000000000000000000000000000000000000000000000000000000000020
  0000000000000000000000000000000000000000000000000000000000000060
  //new vote 方法参数1，执行脚本
  00000001  //executorId
  ab3eca07408b8aa4db5915b2a12b193b25716c8c //execute contract address token manager的proxy
  00000044 //call data length
  40c10f19 //方法签名 mint(address,uint256)
  000000000000000000000000
  b7e84ea36c789dd576ffe46419245ea28a1e0e88 //地址
  0000000000000000000000000000000000000000000000000000000000000001  //数量1
  ```

  