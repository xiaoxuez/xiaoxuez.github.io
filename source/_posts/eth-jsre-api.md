---
title: eth_jsre_api
categories:
  - eth
date: 2019-10-14 14:53:56
tags:
---
## jsre中添加api

otto包，可以直接在go语言中实现js命令。可以在console这种交互模式或者script这种非交互模式中使用

相关的源码分析就省略了。



要添加新的api,首先需要在合适的地方定义具体方法。合适地方..例如backend.go的GetAPIs()为api集合，可在相应的namespace对应的Service中定义，例如在PublicEthereumAPI中添加方法test，访问路径为`eth.test()`

```
func (s *PublicBlockChainAPI) Test(ctx context.Context) error {
   fmt.Println("test")
   return nil
}

```

接下来就是要在js中进行注册，才能定向到上面的方法中。

js中注册有两种方式。

第一种是直接在go文件中添加js静态代码。在web3ext.go中对应位置添加，如

```
const Eth_JS = `
web3._extend({
	property: 'eth',
	methods: [
		new web3._extend.Method({
			name: 'test',
			call: 'eth_test',
			params: 0
		}),
	]
	...
	`
```

这样添加后，直接重新编译即可`make all`



第二种是在js文件中添加代码，在web3.js中对应位置添加，如

```
var methods = function () {
    var test = new Method({
        name: 'test',
        call: 'eth_test',
        params: 0
    });
    var getStorageAt = new Method({
        name: 'getStorageAt',
        call: 'eth_getStorageAt',
        params: 3,
        inputFormatter: [null, utils.toHex, formatters.inputDefaultBlockNumberFormatter]
    });
	...
	 return [
	 	test
	 ]

}
```

修改js文件，要进行编译成go（bindata.go）文件才能生效。

编译deps.go中有go:generate语句，用来重新生成bindata.go文件.

生成bindata.go文件，需要安装go-bindata。安装方法如下。

```
go get github.com/jteeuwen/go-bindata
go install github.com/jteeuwen/go-bindata/go-bindata
```

安装后生成bindata.go文件，可以在IDE中点击生成，也可以在命令行中执行命令生成。

```
cd $GOPATH/src/github.com/ethereum/go-ethereum/internal/jsre/deps
go-bindata -nometadata -pkg deps -o bindata.go bignumber.js web3.js
gofmt -w -s bindata.go
```

最后再进行编译即可`make all`





关于输入参数和输出参数，可在js层就输入的参数进行参数的数据类型转换，如上面示例中的getStorageAt的inputFormatter，参数为null的不会进行操作，utils.toHex应该是把参数encode成hex。formatter应该是构造结构体实例，下面放2个具体的formatter的构造

```
var inputDefaultBlockNumberFormatter = function (blockNumber) {
    if (blockNumber === undefined) {
        return config.defaultBlock;
    }
    return inputBlockNumberFormatter(blockNumber);
};

var inputTransactionFormatter = function (options){

    options.from = options.from || config.defaultAccount;
    options.from = inputAddressFormatter(options.from);

    if (options.to) { // it might be contract creation
        options.to = inputAddressFormatter(options.to);
    }

    ['gasPrice', 'gas', 'value', 'nonce'].filter(function (key) {
        return options[key] !== undefined;
    }).forEach(function(key){
        options[key] = utils.fromDecimal(options[key]);
    });

    return options;
};
```
