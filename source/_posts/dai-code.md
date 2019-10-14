---
title: dai_code
categories:
  - eth
date: 2019-10-14 14:52:07
tags:
---

## dai结构及源码解析

![合约调用图](<https://user-images.githubusercontent.com/5028/58217074-3ec06880-7d55-11e9-91e2-c9bec7172bc4.png>)

结构图如上所示，vat左边的主要是**管理控制修改**vat及vat右边各合约各**参数**的合约，红线file为调用各合约中file方法，也就是修改各合约中相应参数。如vat中file为P427L-9Y552-5433E-8DSR3-58Z68

```
    function file(bytes32 what, uint data) external note auth {
        if (what == "Line") Line = data;
        else revert();
    }
    function file(bytes32 ilk, bytes32 what, uint data) external note auth {
        if (what == "spot") ilks[ilk].spot = data;
        else if (what == "line") ilks[ilk].line = data;
        else if (what == "dust") ilks[ilk].dust = data;
        else revert();
    }
```

vat是CDP的主要逻辑合约。

cat为清算触发合约，通过调用cat.bite方法，当满足清算条件就会触发清算。具体逻辑稍后提及。

vow为结算合约，通过vow的结算条件可触发flop和flap。

flip为抵押物拍卖合约，拍卖由cat触发，拍卖逻辑稍后提及。

flop和flap分别为债务拍卖和盈余拍卖，由vow触发。



### vat

变量

```
//抵押物id对应的参数
mapping (bytes32 => Ilk)                       public ilks;  
//抵押物id => 个人地址 => 个人债务
mapping (bytes32 => mapping (address => Urn )) public urns;  
//抵押物id => 个人地址 => 个人抵押物总值
mapping (bytes32 => mapping (address => uint)) public gem;  
//个人地址 => 含有的dai，统计作用
mapping (address => uint256)                   public dai;  
//清算地址 => 清算总值， 统计作用
mapping (address => uint256)                   public sin;

 struct Urn {
        uint256 ink;   // 锁定的抵押物值
        uint256 art;   // 借贷的dai
 }
 //以抵押物id为个体
 struct Ilk {
        uint256 Art;   // 总借贷dai
        uint256 rate;  // 可借贷比率
        uint256 spot;  // 价格安全线
        uint256 line;  // 总借贷上线
        uint256 dust;  // 总借贷下限
 } //rate和spot的作用主要是满足ink * spot >= art * rate可借贷，不然就不能借贷或者需要清算了。

```

主要讲讲frob方法吧，这个方法可谓是精短又强悍！

```
//bytes32 i ilk标识， address u cdp所有者，v 抵押物所有者，w 获得dai的所有者，dink 参与的抵押物，dart参与的债务
function frob(bytes32 i, address u, address v, address w, int dink, int dart) external note {
    // system is live
    require(live == 1);

    Urn memory urn = urns[i][u];
    Ilk memory ilk = ilks[i];
    // ilk has been initialised
    require(ilk.rate != 0);

    urn.ink = add(urn.ink, dink);
    urn.art = add(urn.art, dart);
    ilk.Art = add(ilk.Art, dart);

    int dtab = mul(ilk.rate, dart);
    uint tab = mul(urn.art, ilk.rate);
    debt     = add(debt, dtab); //dai总量

    //dart <=0, 赎回抵押物？ dart >0, 借贷

    // either debt has decreased, or debt ceilings are not exceeded
    require(either(dart <= 0, both(mul(ilk.Art, ilk.rate) <= ilk.line, debt <= Line)));
    // urn is either less risky than before, or it is safe
    require(either(both(dart <= 0, dink >= 0), tab <= mul(urn.ink, ilk.spot)));
    // urn is either more safe, or the owner consents
    require(either(both(dart <= 0, dink >= 0), wish(u, msg.sender)));
    // collateral src consents
    require(either(dink <= 0, wish(v, msg.sender)));
    // debt dst consents
    require(either(dart >= 0, wish(w, msg.sender)));
    // urn has no debt, or a non-dusty amount
    require(either(urn.art == 0, tab >= ilk.dust));

    gem[i][v] = sub(gem[i][v], dink);
    dai[w]    = add(dai[w],    dtab);
    urns[i][u] = urn;
    ilks[i]    = ilk;
}
```

在以上这个方法中，包含了很多种用法，包括新增cdp、偿还等等，例如当dart>0的时候，就是借贷，当dart<0的时候，偿还。

首先，当借贷/增加抵押物时，抵押物为dink >=0, 贷款dart >0。贷款价值为dart * rate，抵押物价值为dink * spot。借贷需要满足的条件如下

- 贷款价值 < 债务上限line, 且该抵押物的总贷款价值<Line上限

- 贷款价值 <= 抵押物价值，这里理解一下rate，当rate > 1，其实就是说可贷的dart其实是小于真正抵押物价值的，用来调控风险。
- u为调用者，或者调用者是u的下层授权管理（u授权x以操作其cdp）
- 贷款价值 > 债务下限 dust

当条件判断完毕后，进行扣除/赎回抵押物(gem)，借出/偿还dai。

### cat

```
 function bite(bytes32 ilk, address urn) external returns (uint id) {
        VatLike.Ilk memory i = vat.ilks(ilk);
        VatLike.Urn memory u = vat.urns(ilk, urn);

        require(live == 1);
        //触发清算要求，个人抵押品 * 具有安全边际的抵押品价格 < 个人债务 * 稳定债务乘数(稳定债务)
        //即抵押品价格下降到一定程度后，触发清算
        require(mul(u.ink, i.spot) < mul(u.art, i.rate));
        //取个人抵押品和任何一次清算事件所涵盖的固定债务数量的小值
        uint lot = min(u.ink, ilks[ilk].lump);
        //取个人债务 和 抵押品 * 债务 / 抵押品的 小值
        uint art = min(u.art, mul(lot, u.art) / u.ink);
        //债务 * 稳定债务乘数(稳定债务)
        uint tab = mul(art, i.rate);

        require(lot <= 2**255 && art <= 2**255);
        //清算，将个人抵押品减掉，个人债务也减掉,将减掉的抵押品转移到本地址下(gem)
        vat.grab(ilk, urn, address(this), address(vow), -int(lot), -int(art));
        //结算？
        vow.fess(tab);
        //触发拍卖
        id = Kicker(ilks[ilk].flip).kick({ urn: urn
                                         , gal: address(vow)
                                         , tab: rmul(tab, ilks[ilk].chop) //所需债务: dai债务 * 清算罚款 / 10 ** 27
                                         , lot: lot //抵押品
                                         , bid: 0
                                         });

        emit Bite(ilk, urn, lot, art, tab, ilks[ilk].flip, id);
    }
```



##### example

补充，借贷的第一步，会有一个join合约，就是帮忙保管币/代币。调用join方法，会将币/代币转移到本合约地址下，然后本合约会调用vat.slip方法，如

```
    function join(address usr, uint wad) external note {
        require(int(wad) >= 0);
        vat.slip(ilk, usr, int(wad)); //gem[ilk][usr] = wad
        require(gem.transferFrom(msg.sender, address(this), wad));
    }
```

测试参数

```
vat.init("gold");
gemA = new GemJoin(address(vat), "gold", address(gold));
vat.file("gold", "spot", ray(1 ether)); //ray: 10 ** 9
vat.file("gold", "line", rad(1000 ether)); //rad: 10 ** 27
vat.file("Line",         rad(1000 ether));
cat.file("gold", "chop", ray(1 ether));
```



```
address urn = address(this);
//转移币
gemA.join(urn,                             500 ether);
vat.file("gold", 'spot', ray(2.5 ether));
vat.frob("gold", me, me, me, 40 ether, 100 ether);
//假设rate为 (10 ** 9)
//借贷判断， 40 * (10 ** 18) * 2.5 * (10 ** 9) >=? 100 * (10 ** 18) * (10 ** 9),满足条件，可借贷
//vat.urn[gold][me]= ink(40)， art(100)

 vat.file("gold", 'spot', ray(2 ether));  //抵押物价格下降
 cat.file("gold", "chop", ray(1.1 ether));
 cat.file("gold", "lump", 30 ether);

 //清算判断
 uint auction = cat.bite("gold", address(this));
 //40 * (10 ** 18) * 2 * (10 ** 9) >=? 100 * (10 ** 18) * (10 ** 9) 很明显，不满足条件，触发清算
 //清算lot=min(u.ink, ilks[ilk].lump) = 30 ether
 // art = min(u.art, mul(lot, u.art) / u.ink); = 75 ether
 //此时vat.urn[gold][me] = ink(10 ether), art(25 ether)
 //						tab = art * rate
 //触发拍卖kick中tab为rmul(tab, ilks[ilk].chop)， (75 * (10 ** 18) * (10 ** 9)) * 1.1 * (10 ** 18) / 10 ** 27 = 82.5 * (10 ** 18) 即85 ether
```
