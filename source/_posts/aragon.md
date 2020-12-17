---
title: aragon
categories:
  - eth
date: 2020-12-17 15:20:12
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





### Dot Voting app 选项脚本分析

首先简单恶补一下方法数据参数(abi)编码规则。dot app的代码分析可跳到示例4

- 任何数据的单元都为32字节
- 数据类型分为动态类型和非动态类型，string, array等非固定长度的数据类型为动态类型；动态类型需额外存储数据长度
- abi编码为方法签名(4字节)+数据，以下省略方法签名

#### 示例1

```
function slice(uint32[2])
```

测试数据为 `[2]uint32{1, 2}`

分析，`uint32[2]` 虽然是个数组，但是数组长度固定，数组中的数据长度也固定，所以不需要额外的数据来表示长度。

编码结果(hex)为`00000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000002`

可以看到1和2的编码长度单元都是32bytes(hex长度64)

后续为了方便查看，我以32bytes换行展示。



#### 示例2

```
function slice(uint32[])
```

测试数据为 `[]uint32{1, 2}`

分析，这里做了一点小改动，将数组改成了任意长度。所以这里类型更改为动态类型，需要额外的数据来表示长度。

编码结果(hex)为

```
0000000000000000000000000000000000000000000000000000000000000020
0000000000000000000000000000000000000000000000000000000000000002
0000000000000000000000000000000000000000000000000000000000000001
0000000000000000000000000000000000000000000000000000000000000002
```

可以看到，数据中多了`0x20`和`0x02`, `0x02`可以理解是数据的长度为2，那么`0x20` 代表什么呢。

存在动态类型的编码时，会增加动态类型数据的位置偏移索引值，这个`0x20`就代表数据开始于当前位置再往后偏移`0x20`的位置，也就是向后偏移32bytes，是不是就刚好是数据1开始的位置。

`0x20`是怎么得出来的呢。

##### 编码时，会按照参数顺序依次编码，遇到动态类型，就编码位置偏移索引值，遇到非动态类型，就编码数据，最后，将动态类型的数据再依次编码。

那么位置偏移索引值怎么计算呢？

按照上面说的，先将非动态类型的数据编码后，最后编码动态类型的数据，并且遇到动态类型，会编码一个位置索引值(32bytes)，所以`第一个动态类型的位置索引值 = 动态类型参数的个数 * 32 + 非动态类型数据的具体长度（以32为单元的长度） `。如果存在第二个动态类型参数呢，`第二个动态类型的位置索引值 = 第一个动态类型的位置索引值 + 第一个动态类型数据的具体长度`。

简单看一下代码

```go
// Pack performs the operation Go format -> Hexdata
func (arguments Arguments) Pack(args ...interface{}) ([]byte, error) {
	// Make sure arguments match up and pack them
	abiArgs := arguments
	if len(args) != len(abiArgs) {
		return nil, fmt.Errorf("argument count mismatch: %d for %d", len(args), len(abiArgs))
	}
	// variable input is the output appended at the end of packed
	// output. This is used for strings and bytes types input.
	var variableInput []byte

	// input offset is the bytes offset for packed output
	inputOffset := 0  //按照参数类型计算数据长度，也就是位置索引值
	for _, abiArg := range abiArgs {
		inputOffset += getTypeSize(abiArg.Type) //如果类型是动态，TypeSize为32
	}
	var ret []byte
	for i, a := range args {
		input := abiArgs[i]
		// pack the input  编码数据
		packed, err := input.Type.pack(reflect.ValueOf(a))
		if err != nil {
			return nil, err
		}
		// check for dynamic types  数据如果是动态类型
		if isDynamicType(input.Type) {
			// set the offset  将位置索引值编码
			ret = append(ret, packNum(reflect.ValueOf(inputOffset))...)
			// calculate next offset  位置索引值偏移到该动态数据后面，如果存在下一个动态数据，位置就在这
			inputOffset += len(packed)
			// append to variable input  动态类型的数据先放在一个数组里
			variableInput = append(variableInput, packed...)
		} else {
			// append the packed value to the input 先编码非动态类型数据
			ret = append(ret, packed...)
		}
	}
	// append the variable input at the end of the packed input 最后append动态类型数据
	ret = append(ret, variableInput...)

	return ret, nil
}

```



这种编码方式倒是很神奇，通过读取位置索引值，就能索引到数据真正的地方。



#### 示例3

```
function slice(uint32[] a, uint32 b)
```

测试数据为`[]uint32{1, 2}, 2`

编码结果(hex)为

```
	0000000000000000000000000000000000000000000000000000000000000040 //类型占位和偏移
	0000000000000000000000000000000000000000000000000000000000000002
	0000000000000000000000000000000000000000000000000000000000000002
	0000000000000000000000000000000000000000000000000000000000000001
	0000000000000000000000000000000000000000000000000000000000000002

```

按照编码规则，

- 首先遇到动态数组，所以需要所有类型占位和，有两个参数，第一个是动态数组，所以占位是32, 第二个是uint32，占位也是32，所以占位和为64。也就是0x40
- 其次，编码动态数组的数据长度，长度为2
- 然后，编码第二个参数，2
- 最后，将动态数据编码，1和2

按照占位和偏移的含义，0x40也就是数组数据开始的地方，向后偏移64 bytes，也就刚好是数组数据1开始的位置。

下面举个多动态数组的例子，也就是DOT Voting的例子吧



#### 示例4

Dot  voting的数据结构中，提供多选项，每个选项的信息可包含一个地址，一个描述性的info，两个id。（id两级我有点不太理解是做什么用的...，以及signal数组也不太清楚



```
function setSignal(
        address[] _addr, //选项地址数组
        uint256[] _signal,
        uint256[] _infoIndices, //每个info信息长度
        string _candidateInfo,  //多个info拼接在一起，按长度可截取出来
        string description,
        uint256[] _level1Id,
        uint256[] _level2Id
    )
```



按照aragon的结构，和Voting一样，交易数据为调用某合约的forward方法，通过执行器调用对应合约的对应方法。

我在aragon上发起了一个[投票](https://rinkeby.client.aragon.org/#/kuafuzhuri1/0x01b79e559f9c2c9b6b50bd24330add6ddac6e78f/vote/1/)，描述为Test Voting, 三个选项，分别为A B C。然后这笔[交易](https://rinkeby.etherscan.io/tx/0x0fa97113007d5f4d4d0b50cbf5e5373a8755d8758def883e59c5d666a7956c6a)的data如下，以这个数据为例介绍。

```
//这里先按照上面介绍的编码形式简单看看数据
MethodID: 0xd948d468 //forward(bytes s)
[0]:  0000000000000000000000000000000000000000000000000000000000000020 //s索引位置为32bytes
[1]:  00000000000000000000000000000000000000000000000000000000000004c0 //bytes的长度
[2]:  0000000101b79e559f9c2c9b6b50bd24330add6ddac6e78f000004a4d5db2c80 // 这里是特定编码 exec
					//00000001 + 调用合约01b79e559f9c2c9b6b50bd24330add6ddac6e78f + 调用方法 d5db2c80，后面就是调用方法的参数，这里是调用dot vote的newVote(bytes _executionScript, string _metadata)
[3]:  0000000000000000000000000000000000000000000000000000000000000040 //两个参数都是动态类型，故总索引值为32 + 32
[4]:  0000000000000000000000000000000000000000000000000000000000000460 //第二个动态类型位置
[5]:  0000000000000000000000000000000000000000000000000000000000000400 //参数1的长度
[6]:  0000000101b79e559f9c2c9b6b50bd24330add6ddac6e78f000003e400000000 // 参数1又是一个exec script,同上，调用方法是 00000000 ?


/**后面就是具体的要解析出多选项的数据了，参照编码方法为function setSignal(
        address[] _addr,
        uint256[] _signal,
        uint256[] _infoIndices,
        string _candidateInfo,
        string description,
        uint256[] _level1Id,
        uint256[] _level2Id
    )**/
[7]:  00000000000000000000000000000000000000000000000000000000000000e0 //参数1位置，7个动态7*32
[8]:  0000000000000000000000000000000000000000000000000000000000000160 //参数2位置
[9]:  00000000000000000000000000000000000000000000000000000000000001e0 //参数3位置
[10]: 0000000000000000000000000000000000000000000000000000000000000260 //参数4位置
[11]: 00000000000000000000000000000000000000000000000000000000000002a0 // ...
[12]: 00000000000000000000000000000000000000000000000000000000000002e0 // ...
[13]: 0000000000000000000000000000000000000000000000000000000000000360 // 参数7位置（位置结束
[14]: 0000000000000000000000000000000000000000000000000000000000000003 //参数1长度（address）
[15]: 0000000000000000000000000000000000000000000000000000000000000000 //address{0}
[16]: 0000000000000000000000000000000000000000000000000000000000000001 //address{1}
[17]: 0000000000000000000000000000000000000000000000000000000000000002 //address{2}
[18]: 0000000000000000000000000000000000000000000000000000000000000003 //参数2长度
[19]: 0000000000000000000000000000000000000000000000000000000000000000 // 参数2数据[0, 0, 0]
[20]: 0000000000000000000000000000000000000000000000000000000000000000
[21]: 0000000000000000000000000000000000000000000000000000000000000000
[22]: 0000000000000000000000000000000000000000000000000000000000000003 //参数3长度
[23]: 0000000000000000000000000000000000000000000000000000000000000001 //参数3数据[1, 1, 1]
[24]: 0000000000000000000000000000000000000000000000000000000000000001
[25]: 0000000000000000000000000000000000000000000000000000000000000001
[26]: 0000000000000000000000000000000000000000000000000000000000000003 //参数4长度为3bytes
[27]: 4142430000000000000000000000000000000000000000000000000000000000 //参数4数据[41,42,43]，即A,B,C
[28]: 000000000000000000000000000000000000000000000000000000000000000b  //参数5长度为b
[29]: 5465737420566f74696e67000000000000000000000000000000000000000000 //参数5数据Test Voting
[30]: 0000000000000000000000000000000000000000000000000000000000000003 //参数6为[0,0,0]
[31]: 0000000000000000000000000000000000000000000000000000000000000000
[32]: 0000000000000000000000000000000000000000000000000000000000000000
[33]: 0000000000000000000000000000000000000000000000000000000000000000
[34]: 0000000000000000000000000000000000000000000000000000000000000003 //参数7位[0,0,0]
[35]: 0000000000000000000000000000000000000000000000000000000000000000
[36]: 0000000000000000000000000000000000000000000000000000000000000000
[37]: 0000000000000000000000000000000000000000000000000000000000000000

//new vote的第二个参数_metadata编码， 数据Test Voting
[38]: 000000000000000000000000000000000000000000000000000000000000000b
[39]: 5465737420566f74696e67000000000000000000000000000000000000000000
```



然后看看合约代码怎么解析出选项的。

```
//_executionScript为上述6到39
function _extractOptions(bytes _executionScript, uint256 _actionId) internal {
    Action storage actionInstance = actions[_actionId];
    //calldataLength 0x03e4, 0x4 为0x00000001, 0x14为合约地址
    uint256 calldataLength = uint256(_executionScript.uint32At(0x4 + 0x14));
    //startOffset为7开始的位置
    uint256 startOffset = 0x04 + 0x14 + 0x04; //0x04 calldata length

    //OPTION_ADDR_PARAM_LOC = 1,firstParamOffset 为0xe0 + 0x20, 即14开始的位置，对应address数据
    uint256 firstParamOffset = _goToParamOffset(OPTION_ADDR_PARAM_LOC, _executionScript);

    //DESCRIPTION_PARAM_LOC = 5,fifthParamOffset为28开始的位置，对应descript数据
    uint256 fifthParamOffset = _goToParamOffset(DESCRIPTION_PARAM_LOC, _executionScript);

    uint256 currentOffset = firstParamOffset;
    require(startOffset + calldataLength == _executionScript.length); // solium-disable-line error-reason

   //optionLength为address数组的长度
    uint256 optionLength = _executionScript.uint256At(currentOffset);

   //currentOffset向后偏移32, 指向address数组开始的位置
    currentOffset = currentOffset + 0x20;

    //开始解析多选项
    _iterateExtraction(_actionId, _executionScript, currentOffset, optionLength);

   //解析description
    uint256 descriptionStart = fifthParamOffset + 0x20;
    uint256 descriptionEnd = descriptionStart + (_executionScript.uint256At(fifthParamOffset));
    actionInstance.description = substring(_executionScript, descriptionStart, descriptionEnd);

}

//将多选项解析出来，  _currentOffset指向address数组开始的位置，_optionLength为address数组的长度
function _iterateExtraction(uint256 _actionId, bytes _executionScript, uint256 _currentOffset, uint256 _optionLength) internal {
    uint256 currentOffset = _currentOffset;
    address currentOption;
    string memory info;
    uint256 infoEnd;
    bytes32 externalId1;
    bytes32 externalId2;
    uint256 idOffset;
    //OPTION_INFO_PARAM_LOC=4, infoStart为参数4数据开始的位置再后偏移32位(长度)，也就是27的位置
    uint256 infoStart = _goToParamOffset(OPTION_INFO_PARAM_LOC,_executionScript) + 0x20;

    emit OptionQty(_optionLength);
    //以address数组的长度作为遍历次数，长度即为多选的长度
    for (uint256 i = 0 ; i < _optionLength; i++) {
        //以32作为单元存储长度20的地址，前12都是0补齐，所以skip 12，将地址读出来
        currentOption = _executionScript.addressAt(currentOffset + 0x0C);
        emit Address(currentOption);

        //infoEnd = 参数4数据真实的位置 + 参数3的第i个值（参数3为info长度
        infoEnd = infoStart + _executionScript.uint256At(currentOffset + (0x20 * 2 * (_optionLength + 1) ));
        //单个选择项的info=_candidateInfo，按_infoIndices数组对应的值作为长度进行截取
        info = substring(_executionScript, infoStart, infoEnd);

        currentOffset = currentOffset + 0x20;

        infoStart = infoEnd;
        //后面就是按索引值取对应的externalId了
        idOffset = _goToParamOffset(EX_ID1_PARAM_LOC, _executionScript) + 0x20 * (i + 1);
        externalId1 = bytes32(_executionScript.uint256At(idOffset));
        idOffset = _goToParamOffset(EX_ID2_PARAM_LOC, _executionScript) + 0x20 * (i + 1);
        externalId2 = bytes32(_executionScript.uint256At(idOffset));

        addOption(_actionId, info, currentOption, externalId1, externalId2);
        }
    }
```


