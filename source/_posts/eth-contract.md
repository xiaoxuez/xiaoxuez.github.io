---
title: eth_contract
categories:
  - eth
date: 2019-10-14 14:51:05
tags:
---

## 以太坊智能合约

从智能合约的代码到使用智能合约，大概包含以下步骤

- 编写智能合约的代码(一般是用Solidity)
- 编译智能合约的代码变成可在EVM上执行的bytecode(binary code)，同时可以通过编译取得智能合约的ABI
- 部署智能合约，实际上是吧bytecode存储在链上(通过一个transaction)，并取得一个专属于这个合约的地址
- 要调用合约，需要把信息发送到这个合约的地址，一样也是通过transaction，以太坊节点会根据输入的信息，选择要执行合约中的哪一个function和要输入的参数

以下，将详细介绍以上步骤。

#### 代码编写及编译部署

#### 智能合约ABI

如果说api，想必都知道是什么，对应的，ABI，application binary interface，顾名思义，同样是接口，但传递的是binary格式的信息。

ABI理解如下

##### Function

- `name`：a string, 方法名
- `type`:  a string，"function", "constructor", or "fallback"，方法类型
- `inputs`:  an array，方法参数，每个参数的格式为
  - `name`：a string，参数名
  - `type`：a string，参数的 data type(e.g. uint256)
  - `components`：an array，如果输入的参数是 tuple(struct) type 才会有这个参数。描述 struct 中包含的参数类型
- `outputs`：an array， 方法返回值，和 `inputs` 使用相同表示方式。如果沒有返回值可忽略，值为 `[]`
- `payable`：`true`，function 是否可收 Ether，预设为 `false`
- `constant`：`true`，function 是否会改写区块链状态，反之为 `false`
- `stateMutability`：a string，其值可能为以下其中之一："pure"（不会读写区块链状态）、"view"（只读不写区块链状态）、"payable" and "nonpayable"（会改区块链状态，且如可收 Ether 为 "payable"，反之为 "nonpayable"）

仔细看会发现 `payable` 和 `constant` 这两个参数所描述的內容，似乎已包含在 `stateMutability` 中。

##### Event

- `name`: a string，event 的名称
- `type`: a string，always "event"
- `inputs`: an array，输入参数，包含：
  - `name`: a string，参数名称
  - `type`: a string，参数的 data type(e.g. uint256)
  - `components`: an array，如果输入参数是 tuple(struct) type 才会有这个参数。描述 struct 中包含的信息类型
  - `indexed`: `true`，如果这个参数被定义为 indexed ，反之为 `false`
- `anonymous`: `true`，如果 event 被定义为 anonymous

更新智能合约状态需要发送 transaction，transaction 需要等待验证，所以更新合约状态是非同步的，无法马上取得返回值。使用 Event 可以在状态更新成功后，将相关信息记录到 Log，并让监听这个 Event 的 DApp 或任何应用这个接口的程序收到通知。每笔 transaction 都有对应的 Log。

所以简单来说，Event 可用來：1. 取得 function 更新合约状态的返回值 2. 也可作为合约另外的存储空间。

Event 的参数分为：有 `indexed`，和其他没有 `indexed` 的。有 `indexed` 的参数可以使用 filter，例如同一个 Event，我可以选择只监听从特定 address 发出来的交易。每笔 Log 的信息同样分为两个部分：Topics（长度最多为 4 的 array） 和 Data。有 `anonymous` 的参数会存储存在 Log 的 Topics，其他的存在 Data。

##### 示例

```
pragma solidity ^0.4.20;
contract SimpleStorage {
    uint public data;
    event Set(address indexed _from, uint value);
    function set(uint x) public {
        data = x;
        Set(msg.sender, x);
    }
}

```

生成的ABI接口为

```
[{		//自动生成的方法名为data的只读方法，返回data值,"stateMutabTility": "view"代表只读
        "constant": true,  
        "inputs": [],
        "name": "data",
        "outputs": [{"name": "","type": "uint256"}],
        "payable": false,
        "stateMutabTility": "view",
        "type": "function"
    },
    {  //set方法， "stateMutability": "nonpayable",可写方法，但不可收Ether,方法参数为uint256类型
        "constant": false,
        "inputs": [{"name": "x","type": "uint256"}],
        "name": "set",
        "outputs": [],
        "payable": false,
        "stateMutability": "nonpayable",
        "type": "function"
	},
    {
        "anonymous": false,
        "inputs": [{"indexed": true,"name": "_from","type": "address"},{"indexed": false,"name": "value","type": "uint256"}],
        "name": "Set",
        "type": "event"
}]
```

event时间可监听，在web3使用时会有示例

### web3

关于智能合约的调用，通过命令行也是可以的，但binary的拼凑有点繁琐，web3封装的接口调用起来就方便很多。

直接看例子吧。

```
contract hello {
    function say() constant public returns (string) {
        return "Hello World";
    }
}
```

然后部署到了服务器上，返回合约的地址为0x43d03aaeb07e518cd975c2d67b83a7ffea3a5a51

abi为编译产生的abi数组，直接复制粘贴即可。

```
var contract = web3.eth.contract(info.abi).at("0x43d03aaeb07e518cd975c2d67b83a7ffea3a5a51");
var account_one = web3.eth.accounts[0];
var result = contract.say({from: account_one})
console.log(result); //Hello World
```

一个简单的调用就成功了，然后再看contract提供的接口的具体情况。

智能合约方法调用方式有四种

```
//根据方法类型自动决定是使用call方式进行调用还是sendTransaction方式进行调用，call和sendTransaction的区别呢，如果方法是只读的，则不需要入链，直接调用即可，方法是可写的，则需要以交易的方法进行提交入链。参数中，param则是传入给方法的参数，后面添加别的参数
myContractInstance.myMethod(param1 [, param2, ...] [, transactionObject] [, defaultBlock] [, callback]);

//精确使用call方式进行调用
myContractInstance.myMethod.call(param1 [, param2, ...] [, transactionObject] [, defaultBlock] [, callback]);

//精确使用发送交易形式进行调用
myContractInstance.myMethod.sendTransaction(param1 [, param2, ...] [, transactionObject] [, callback]);

//没看明白
// Get the call data, so you can call the contract through some other means
// var myCallData = myContractInstance.myMethod.request(param1 [, param2, ...]);
var myCallData = myContractInstance.myMethod.getData(param1 [, param2, ...]);
// myCallData = '0x45ff3ff6000000000004545345345345..'

```

然后，顺便玩一下event。

合约修改为

```
pragma solidity ^0.4.18;
contract hello {
    string public greeting;
    event Set(address indexed _from, string value);

    function set(string g) public {
        greeting = g;
        Set(msg.sender, g);
    }

    function get() constant public returns (string) {
        return greeting;
    }
}
```

然后完整的调用脚本为

```
var Web3 = require("web3");
var web3 = new Web3();
web3.setProvider(new Web3.providers.HttpProvider("http://localhost:8545"));
if (web3.isConnected()) {
  console.log("connection success!");
} else {
  console.log("fail to connection!");
  return
}


var contract = web3.eth.contract(abi).at(address);
var account_one = web3.eth.accounts[0];
var my_event = contract.Set();
var getStr = function() {
  var str = contract.greeting({from: account_one})
  console.log("the str is: " + str);
}

getStr()
my_event.watch(function(err, result) {
    if (!err) {
        console.log(result);
        getStr()
    } else {
        console.log(err);
    }
    my_event.stopWatching();
});

contract.set("I am Changed!",{from: account_one})
```

查看输出

```
// connection success!
// the str is:
// { address: '0x8321a32f8b7bbc00a65c9e21df64948cf40bbeba',
//   blockNumber: 785,
//   transactionHash: '0xa90e35845cd844907f251660bce78f5cde2c968876f3a6321fbf38589d7fb476',
//   transactionIndex: 0,
//   blockHash: '0x742eecc3ae508b33ca5dde2ae2a32b8c0c87d321c6ab13a0cc8ad3909e579275',
//   logIndex: 0,
//   removed: false,
//   event: 'Set',
//   args:
//    { _from: '0xc6e1feafcc44ebb1a7b0ecc06770566845eac7ac',
//      value: 'I am Changed!' } }
// the str is: I am Changed!
```

可以看到返回的结果中包括区块哈希等信息，还有参数





#### Event

在合约中定义事件，如

```
event SendMsg(string sender, string receiver, string detail, address indexed reth);
```

注意到indexed关键字。跟event订阅有一定的关系。

大致理解是，event事件犹如日志，在订阅的时候，需要确定要订阅的关键字，这个关键字可以是事件本身，也可以是某个可索引的值，如address值。如订阅某确定address值时，会取得所有含有可索引的这个地址相关的event事件。订阅事件本身的话，就只能取得这个事件的值。

顺便粘贴一段在web3j中event的使用

```
   public static void test() {
        Web3j web3j = Web3j.build(new HttpService("http://localhost:8545"));
        Event event = new Event("SendMsg",
                Arrays.<TypeReference<?>>asList(
                        new TypeReference<Utf8String>(){},
                        new TypeReference<Utf8String>(){},
                        new TypeReference<Utf8String>(){},
                        new TypeReference<Address>(true){}
                ));
        EthFilter filter = new EthFilter(DefaultBlockParameterName.EARLIEST,
                DefaultBlockParameterName.LATEST, "0x495feebd99f645a43aa63edb32d46e057e44e286");
        //订阅的是event事件本身
        filter.addSingleTopic(EventEncoder.encode(event));

        web3j.ethLogObservable(filter).subscribe(log -> {
            System.out.println(log);
            //data参数值，需要进行解码。
            List<Type> results = FunctionReturnDecoder.decode(log.getData(), event.getNonIndexedParameters());
            for (Type type : results) {
                System.out.println(type);
            }

        });

    }

    //输出，例如，当前事件有 SendMsg("susan", "tom", "hi", "0x11a857e4d069d963c0676c53b68e9d571a3e2b26")
    //log输出为Log{removed=false, logIndex='0x0', transactionIndex='0x0', transactionHash='0x31aa0a485f5595fb5b5c959e94dfb511c3512e94589dfd51205c5bd07c6ba4e2', blockHash='0xc68dcd25600857642199f9ebb354b62532f8468191a85ca923c0e03e958c6398', blockNumber='0x157c', address='0x495feebd99f645a43aa63edb32d46e057e44e286', data='...', type='null', topics=[0x444f124b164dc796e3a81a1d90ea60c96a815eac0a67ad1d3d550d30afda9e1f, 0x00000000000000000000000011a857e4d069d963c0676c53b68e9d571a3e2b26]
    //topic为两个，第一个为SendMsg事件本身，第二个为SendMsg事件定义中含有可索引值address,
    //data解码非索引项结果为susan、tom、hi
```



#### Remix

remix是基于浏览器开发的ide

- 加载本地文件夹

  ```
  npm install -g remixd
  remixd -s <absolute-path-to-the-shared-folder>
  //会开启共享文件夹服务，通过ws
  ```

  然后在remix上点击类似于超链接的那个按钮



#### bytes32、string

文档上是这么说的，bytes1 ~bytes32是长度特定的类型，故花费比string和bytes小。



然后记一下一般存pubkey(pubkey除掉0x长度为64，可以用两个bytes32来存)

```
bytes32 pubkey_half_pre;
bytes32 pubkey_half_after;
```



#### 钱💰💰💰💰

合约部署之后生成合约地址，地址同样可以作为普通地址使用，即转账系列。可以给合约地址转账，见[文档](https://solidity-cn.readthedocs.io/zh/develop/contracts.html#fallback)

需定义一个未命名、没有参数也没有返回值函数,且为payable。如

```
function () payable {}
```

在ico中，转账给到合约地址，合约返回一定数量token给到转账者，就可以利用这个函数进行。



或者呢，合约的某方法如果使用了payable修饰的话，也代表这个方法可以接受ether, 即msg.value，钱会存到合约地址中，通过this.balance即可获取余额。



在合约中，向某地址发送ether, 使用address.transfer、address.send即可。



#### 合约交互

现有一合约

```
pragma solidity ^0.4.18;

contract UserTest {
    struct User {
        string name;
        address addr;
        string pubkey;
        bool isValue;
    }
    mapping(string => User)  users;
    event UserChange(string uname, address addr, uint value);

    function addUser(string uname, address addr, string pubkey)  public {
         require(
           !users[uname].isValue,
           "name already exist"
       );
       users[uname] = User({
           name: uname,
           addr: addr,
           pubkey: pubkey,
           isValue: true
       });
       UserChange(uname, addr, msg.value);
    }

    function userExist(string uname) constant external returns (bool) {
        return users[uname].isValue;
    }

}
```

然后在另一合约中调用userExist方法,首先部署上述合约，合约地址如0x123456789

```
pragma solidity ^0.4.18;

contract UserTest {
	//要调用的方法声明需要跟原来一样
    function userExist(string uname) constant external returns (bool);
}
contract ExternalTest {

    address userInstance = 0x123456789;
    UserTest user = UserTest(userInstance); //强制类型转换

    function userCheck(string uname) public returns (bool){
        return user.userExist(uname);
    }
}
```



#### mapping

mapping是不能遍历的，需要借助别的数据结构。

```
library IterableMapping
{
  struct itmap
  {
    mapping(uint => IndexValue) data;
    KeyFlag[] keys;
    uint size;
  }
  struct IndexValue { uint keyIndex; uint value; }
  struct KeyFlag { uint key; bool deleted; }
  function insert(itmap storage self, uint key, uint value) returns (bool replaced)
  {
    uint keyIndex = self.data[key].keyIndex;
    self.data[key].value = value;
    if (keyIndex > 0)
      return true;
    else
    {
      keyIndex = self.keys.length++;
      self.data[key].keyIndex = keyIndex + 1;
      self.keys[keyIndex].key = key;
      self.size++;
      return false;
    }
  }
  function remove(itmap storage self, uint key) returns (bool success)
  {
    uint keyIndex = self.data[key].keyIndex;
    if (keyIndex == 0)
      return false;
    delete self.data[key];
    self.keys[keyIndex - 1].deleted = true;
    self.size --;
  }
  function contains(itmap storage self, uint key) returns (bool)
  {
    return self.data[key].keyIndex > 0;
  }
  function iterate_start(itmap storage self) returns (uint keyIndex)
  {
    return iterate_next(self, uint(-1));
  }
  function iterate_valid(itmap storage self, uint keyIndex) returns (bool)
  {
    return keyIndex < self.keys.length;
  }
  function iterate_next(itmap storage self, uint keyIndex) returns (uint r_keyIndex)
  {
    keyIndex++;
    while (keyIndex < self.keys.length && self.keys[keyIndex].deleted)
      keyIndex++;
    return keyIndex;
  }
  function iterate_get(itmap storage self, uint keyIndex) returns (uint key, uint value)
  {
    key = self.keys[keyIndex].key;
    value = self.data[key].value;
  }
}

// How to use it:
contract User
{
  // Just a struct holding our data.
  IterableMapping.itmap data;
  // Insert something
  function insert(uint k, uint v) returns (uint size)
  {
    // Actually calls itmap_impl.insert, auto-supplying the first parameter for us.
    IterableMapping.insert(data, k, v);
    // We can still access members of the struct - but we should take care not to mess with them.
    return data.size;
  }
  // Computes the sum of all stored data.
  function sum() returns (uint s)
  {
    for (var i = IterableMapping.iterate_start(data); IterableMapping.iterate_valid(data, i); i = IterableMapping.iterate_next(data, i))
    {
        var (key, value) = IterableMapping.iterate_get(data, i);
        s += value;
    }
  }  
}
```







#### 以太坊虚拟机(EVM)

以太坊虚拟机(EVM)是智能合约的运行环境。它是一个完全独立的沙盒，合约代码在EVM内部运行，对外是完全隔离的，甚至不同合约之间也只有有限的访问权限

#### 账户

- 以太坊中有两种不同类型但是共享同一地址空间的账户：`外部账户`由一对公私钥控制，`合约账户`由账户内部的合约代码控制。
- 外部账户的地址是由公钥（经过hash运算）决定的，而合约账户的地址在此合约被创建的时候决定的（由合约创建者的地址和发送到此合约地址的交易数决定，这就是所谓的“nonce”）
- 不管是哪种类型的账户，EVM的处理方式是一样的
- 每个账户都有一个持久的key-value类型的存储，把256字节的key映射到256字节的value
- 此外，每个账户都有以“Wei”为单位，在交易过程中会被修改的资产(balance)信息

#### 交易

- 交易是一个从账户发往另一个账户（可以是同一个账户或者是special zero-account）的消息。它包含二进制数据（交易相关的数据）和 Ether。
- 如果目标账户包含代码，代码会被执行，交易相关的数据将作为参数
- 如果目标账户是地址为0的账户`zero-account`, 交易会创建一个新的合约。如上文提到的，合约地址不是一个地址为0的地址，而是一个由交易发送者和交易数来决定的地址。这样的一笔（到zero-account）交易的相关参数会被转化为EVM字节码
  然后被执行，输出结果就是被永久存储的合约代码。这意味着为了创建一个合约，并不需要发送真实的合约代码，代码可以被自动创建

#### gas

- 创建之后，每笔交易都需要一定数量的gas，用于限制交易所消耗的工作量，即交易是需要付出代价的（避免DDoS攻击）。EVM执行交易的过程中，gas会按一个特殊规则逐渐减少
- 费用的多少是由交易发起者设置，至少需要从发起账户支付`gas_price * gas`用费。如果交易执行完毕费用还有剩余的，将退回到发起账户。
- 如果交易完成之前费用耗尽，将会抛出一个`out-of-gas`的异常，所有的修改都会被回滚

更多关于gas的理解和讨论可以[戳这里](http://bitshuo.com/topic/5857774a2a482b0d339aab99/)

#### storage,memory,stack

- 每个账户都有一个持久的内存空间，称之为`storage`,`storage`以key-value形式存储，256字节的key映射到256字节value，合约内部不可能枚举`storage`(内部元素)，读取或者修改`storage`操作消耗都很大(原文是 It is not possible to enumerate storage from within a contract and it is comparatively costly to read and even more so, to modify storage. )。 合约只能读取和修改自己的`storage`里的数据。
- 第二种内存空间称之为`memory`,里面存储着每个消息调用时合约创建的实例。`memory`是线型的，可以以字节级别来处理，但是限制为256字节宽度，写入可以是8或256字节宽度。当读取或写入一个预先未触发的指令的时候会消耗`memory`的空间，消耗空间的同时，必须支付gas。`memory`消耗的越多，需要的gas越多（按平方级增长）
- EVM不是一个注册的机器而是一个堆栈机器，所以所有的计算指令都在`stack`空间里面执行。`stack`最多只能容纳1024个长度不超过256字节的指令元素。只能用下述方法，从顶部访问`stack`：可以拷贝最顶部的16个元素中的一个到`stack`的最顶部，或者将最顶部的那个元素与其下面的16个元素之一互换。所有其它操作从`stack`最顶部取出两个（或一个，或更多，取决于操作）元素，然后把结果push到`stack`顶端。当然将`stack`中的元素移到`memory`或者`storage`也是可以的，但是不能直接访问`stack`中间的元素（必须从头部开始访问）

#### 指令集合

- EVM的指令集合控制的很小，这样可以避免错误的执行引发问题。所有的指令都是操作最基本的数据类型，256字节。而且都是最常见的逻辑，算法，字节和比较运算。有条件或无条件的跳转都可以。此外，合约可以访问当前区块的属性，比如区块编号和时间戳。

#### 消息调用

- 合约之间可以通过消息调用的方式进行相互调用或者另一个给另一个无合约账户（外部账户）转币。消息调用很像交易，两者都有源账户，目标账户，数据（data payload），`Ether`,费用和返回数据。实际上每笔交易都由一个可创建更多调用的顶级调用组成。
- 合约可以决定内部消息调用的时候发送多少手续费，保留多少。如果在内部消息调用的时候抛出`out-of-gas`异常（或者其它异常），这个会被一个错误值标记，放到`stack`顶部。如此，只有和消息一起发出的手续费才会被消耗。在`Solidity`中这种情形默认会引发一个异常，以便异常“冒泡”到`stack`最顶端
- 如上所述，被调用的合约会接收到一个刚创建的`memory`实例，并且可以访问调用参数，调用参数被存储在一个被为`calldata`的隔离的区域。执行完毕后，被调用的合约将返回数据存储在调用合约预先创建的内存中。
- 调用被限制在1024深度，这意味着复杂的操作应尽量使用循环代替递归调用。

#### 代理调用/调用代码和库

- 存在一种称为`delegatecall`的特殊的多样性的消息调用，which is identical to a message call apart from the fact that the code at the target address is executed in the context of the calling contract and msg.sender and msg.value do not change their values.
- 这意味着合约可以在运行的时候动态的从另一个地址加载代码。存储、当前地址和资产仍然和调用的合约相关联，只有代码来自被调用的地址。
- 这样可以实现Solidity库的特性:反复使用的库代码可以被应用到合约的`storage`来实现复杂的数据结构。

#### 日志

It is possible to store data in a specially indexed data structure that maps all the way up to the block level. This feature called logs is used by Solidity in order to implement events. Contracts cannot access log data after it has been created, but they can be efficiently accessed from outside the blockchain. Since some part of the log data is stored in bloom filters, it is possible to search for this data in an efficient and cryptographically secure way, so network peers that do not download the whole blockchain (“light clients”) can still find these logs.

#### Create

- 合约甚至可以使用特殊的`opcode`创建其它的合约,`create`消息调用和普通的消息调用区别在于，`create`消息调用的data字段会被执行，执行结果以代码的形式存储，调用者可在stack上接接到新合约账户的地址


- 示例一

```
pragma solidity ^0.4.18;

contract ClassifyStorage {
    mapping(address => string) details;
    event AddDetailEvent(address classify, string detail);

    function addDetail(address classify, string detail) public {
        details[classify] = strConcat(details[classify], detail);
        AddDetailEvent(classify, detail);
    }

    function getSpecificDetails(address classify) constant public returns (string) {
        return details[classify];
    }

    function strConcat(string _a, string _b) internal returns (string){
        bytes memory _ba = bytes(_a);
        if(_ba.length == 0) {
            return _b;
        }
        bytes memory _bb = bytes(_b);
        string memory ret = new string(_ba.length + _bb.length + 1);
        bytes memory bret = bytes(ret);
        uint k = 0;
        for (uint i = 0; i < _ba.length; i++)bret[k++] = _ba[i];
        bret[k++] = ",";
        for (i = 0; i < _bb.length; i++) bret[k++] = _bb[i];
        return string(ret);
   }  
}

```
