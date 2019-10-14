---
title: fabric_example
categories:
  - fabric
date: 2019-4-11 15:05:29
tags:
---

fabric-examples学习笔记

由于docker images下载的速度实在太… 这个时间就先看看examples吧

版本release-1.1



chaincode

- Example-01

    func main() {
    	err := shim.Start(new(SimpleChaincode))
    	if err != nil {
    		fmt.Printf("Error starting Simple chaincode: %s", err)
    	}
    }

第一个例子就详细看一下结构，后面的都类似。

main方法里shim.Start(new(结构体))是入口，所有代码实现的部分都是实现这个结构体。这个结构体需要实现的方法

-     Init(stub shim.ChaincodeStubInterface) pb.Response
-     Invoke(stub shim.ChaincodeStubInterface) pb.Response

在这个例子中，有两个账户及其余额，Init方法为设置两个账户的名字及余额。Invoke方法实现的是从A账户转账到B账户。顾名思义的话，Init为初始化方法，Invoke猜测应该是使用反射，可实现为分发的功能，分发到其他方法中，实现具体相应功能。例如

    func (t *SimpleChaincode) Invoke(stub shim.ChaincodeStubInterface) pb.Response {
    	function, args := stub.GetFunctionAndParameters() //获取调用的方法名及其参数
    	if function == "invoke" {
    		// Make payment of X units from A to B
    		return t.invoke(stub, args)  //调用相应方法
    	} else if function == "delete" {
    		// Deletes an entity from its state
    		return t.delete(stub, args)  //调用相应方法
    	} else if function == "query" {
    		// the old "Query" is now implemtned in invoke
    		return t.query(stub, args)  //调用相应方法
    	}

    	return shim.Error("Invalid invoke function name. Expecting \"invoke\" \"delete\" \"query\"")
    }

好吧其实这是example-02的代码了  - . -



- Example-02

功能上来说的话，01是两个固定的账户，02是账本上的两个账户的操作。第一，操作增多了，有查询删除，实现为在invoke中进行分发，代码如上…第二，牵涉到账本了，看似像一个key-value的数据存储，代码如下

    //存
    err = stub.PutState(A, []byte(strconv.Itoa(Aval)))
    //取
    Avalbytes, err := stub.GetState(A)

猜测肯定是数据库方面的了，更深入的之后再看。尤其是stub，shim.ChaincodeStubInterface。这个对象是所有跟上层接轨的接口，其实现需要查看。这里先略过



- Example -03

这个例子中，好像重点在下面这个代码

    func (t *SimpleChaincode) query(stub shim.ChaincodeStubInterface, args []string) pb.Response {
      ···
      // Write the state to the ledger - this put is illegal within Run
    	err = stub.PutState(A, []byte(strconv.Itoa(Aval)))
      ···
    }

意思好像是在query方法中不能进行putState。延伸一下的话，是只有在原始invoke方法中才能进行putState，修改数据库。当然啦，这个小写的invoke方法也是可以实现的



- Example-04

这个例子为example了在链码中调用其他链码，主要api为

    response := stub.InvokeChaincode(chainCodeToCall, invokeArgs, channelID)

- Example-05

这个例子也是调用其他链码，但是说明了调用的链码和当前链码不是同一个channel上需要指明channel。这个不是很明白。

使用InvokeChaincode方法，传入的参数依次是 被调用的链码名称，调用参数(包括调用方法和参数)， channel name

看4，5test并没有看出来4 5调用的差别...



- Test

    	scc := new(SimpleChaincode)
    	stub := shim.NewMockStub("ex03", scc)
    	res := stub.MockInvoke("1", [][]byte{[]byte("query"), []byte("A"), []byte("345"))
    	if res.Status != shim.OK {
    		fmt.Println("Query failed", string(res.Message))
    		t.FailNow()
    	}



还有个启动两个链码的

    	scc := new(SimpleChaincode)
    	stub := shim.NewMockStub("ex05", scc)

    	ccEx2 := new(ex02.SimpleChaincode)
    	stubEx2 := shim.NewMockStub(chaincodeName, ccEx2)
    	checkInit(t, stubEx2, [][]byte{[]byte("init"), []byte("a"), []byte("222"), []byte("b"), []byte("333")})
    	//在当前模拟peer，添加chaincode
    	stub.MockPeerChaincode(chaincodeName, stubEx2)





看官方文档的介绍，Tutorials似乎提供了4个tutorial，分别为Application开发(使用的node sdk)、网络搭建、链码开发、链码操作。作为入门的话，肯定要都过一遍啦~



网络搭建

示例中包含2组，每组2个peer节点，以及1个单独的order服务

首先clone fabric-example仓库，然后找到下载docker images的地方，下载docker-images官方是使用curl … | bash 的一行命令，下载速度很慢，经过网上很多好心人提示，说curl 那个连接其实是一个脚本，直接用网站打开那个连接，复制粘贴脚本到本地，命名*.sh，然后直接运行就好了，这个脚本大概看一下，有3个方法的调用，2个方法都有下载docker images, 还有1个方法是下载二进制文件，这个二进制文件下载后会保存在当前目录下的bin文件夹下。bin里面的内容如下

    $ ls
    configtxgen		configtxlator		cryptogen		get-docker-images.sh	orderer			peer

然后把这个bin文件夹拷贝到fabric-example下。

    $ cd first-network
    $ ./byfn.sh -m generate

如果bin里面内容缺失，这里就会报各种命令不存在。再则，下载的fabric-example的版本应该和二进制可执行文件的版本一致，不然还是会报错..可直接修改下载脚本里的版本。

    $ ./byfn.sh -m up

其中，对于版本问题是是严谨。各个版本都要一样。另外，上面提到的那个curl 链接的那个脚本，在fabric仓库下其实是有的，script/bootstrap.sh。可直接拷贝粘贴



然后针对byfn脚本具体分析一下整个过程和结构的吧~

☝️，$ ./byfn.sh -m generate的功能。

- 使用cryptogen工具来为节点生成加密证书。这些证书可代表节点的身份，被允许签署/验证身份验证进行实体沟通和交易。
  - cryptogen通过配置文件进行工作，配置文件中定义了各个节点信息。如crypto-config.yaml
        OrdererOrgs:
          - Name: Orderer
            Domain: example.com
            # ---------------------------------------------------------------------------
            # "Specs" - See PeerOrgs below for complete description
            # ---------------------------------------------------------------------------
            Specs:
              - Hostname: orderer
        PeerOrgs:
        # -----------------------------------------------------
        # Org1
        # ----------------------------------------------------
        - Name: Org1
          Domain: org1.example.com
          Template:
              Count: 2
          Users:
              Count: 1
    使用命令为cryptogen generate --config=./crypto-config.yaml，成功后会在当前文件夹下生成crypto-config文件夹,  各个证书以节点角色/Domain/…的文件形式存在
- 使用configtxgen tool来生成一些部署实体所需要的零件。也需要配置文件，配置文件定义了网络模板。主要零件类型有
  - orderer genesis block, orderer服务启动必备，注意的是每个组织的根证书都包含在gensis.block中。
  - channel configuration transaction, 该配置在orderer服务启动创建channel时配置channel。这里就已经形成了channel的读写策略（即哪些实体可以读，哪些实体可以写，下同）。
  - and two anchor peer transactions - one for each Peer Org. 用于指定在channel中，当前peer组织中都有哪些peer结点。这里就已经形成了组织的读写策略。

- 最后就是 up启动网络了，最重要的命令即... docker-compose -f $COMPOSE_FILE...， 查看docker-compose-cli.yaml，定义了6个容器服务，其中最后一个为cli, 执行为./scripts/script.sh。查看这个脚本大概为创建channel、加入channel、升级组织配置、安装部署chaincode、调用chaincode...后面每一步应该都会自己搞一次..这里还是先跳过
  到这里，我就感觉脑子里放不下东西了已经，很尴尬…先跟着向导继续走吧

  嗯,  很难受。

  具体看了一下cli， 大致是将证书，script等都作为数据卷挂载到容器上，然后在容器上调用script操作。channel的加入，似乎是通过各证书验证身份。链码的部署调用都是通过peer chaincode … 命令进行
