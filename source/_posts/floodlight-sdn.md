---
title: floodlight_sdn
date: 2017-10-14 13:56:22
categories:
- enjoy
---

## SDN基本架构

- 控制层
- 转发层
- 应用层
- 南向接口
- 北向接口

基本含义借"SDN核心技术剖析和实战指南"书的例子来说：

> SDN的架构可以与人的身体做类比：控制层就是人的大脑，负责对人身体的总体管控；转发层的设备是人的四肢，在大脑的控制下进行各种活动；应用层对应的是各种创新的想法，大脑在它们的驱动下对四肢进行指挥以达到其所需的效果；南向接口和北向接口则分别相当于人体内的神经和脑电波，负责各种信号的上传下达

故由上到下的顺序是 应用层 -> 控制层 -> 转发层，南北也因此命名，北向接口为控制层提供给应用层的接口，南向接口为控制层提供给转发层的接口。

#### SDN 交换机

交换机属于转发层，负责具体数据转发处理的设备。

> 其工作原理都是在收到数据包时，将数据包中的某些特征域与设备自身存储的一些表项进行比对，当发现匹配时则按照表项的要求进行相应处理。

言而言之交换机的工作就是对数据包进行匹配和处理，但匹配和处理的定义并非在交换机中实现，而是由控制器统一下发，统称**流表**。简要介绍一下OpenFlow流表的组成: 包头域 + 计数器 + 动作，包头域包括一系列进行匹配的参数，如入端口、源MAC地址，目的MAC地址等，计数器则用于统计数据流量的相关信息，动作用于指示交换机经匹配后的数据包如何处理，如转发。

#### 南向接口

> SDN交换机需要在远程控制器的管控下工作，与之相关的设备状态和控制指令都需要经由SDN的南向接口传达。当前，最知名的南向接口莫过于ONF倡导的OpenFlow协议。

#### 北向接口

> SDN北向接口是通过控制器向上层业务应用开放的接口，其目标是使得业务应用能够便利地调用底层的网络资源和能力。

> 例如REST API就是上层业务应用的开发者比较喜欢的接口形式

上面的概念下面将进一步讨论，但主要记录的是Builder设计模式的易用性和不可变对象的production(Buidler和production都是设计模式），production的对象的使用，代替了强制类型安全编码的原始类型，并且产生的是更多的可读性代码，内置的通配形式，以及最后没有必要去处理消息的长度。(这段话的对比是Pre-v1.0和Floodlight v1.0, v1.1, v1.2，可见readme中这段话的前面代码对比)

所有的连接到Floodloght的switches都包含一个switch描述Openflow版本的工厂。可以有多种switch,所有都是不同版本的Openflow，在那些swicth中，controller会幕后处理低级协议差异。从模块和应用开发人员的角度来看，switch是暴露出来的IOFSwitch, IOFSwitch的方法之一getOFFactory能返回OpenFlowJ-Loxi工厂，适合switch描述的OpenFlow版本。一旦你有了正确的工厂，你就可以通过公共的OpenFlowJ-Loxi暴露的API创建OpenFlow类型和概念。

因此，你在编写FlowMods和其他类型的时候不需要切换Api。例如，假设你想构建一个FlowMod，发送到一个switch上，OFSwitchManager已知的每一个switch都有一个相同版本的OpenFlow工厂的引用。这个引用在于协商switch和controller之间最初的握手。故上述整体过程是，从你的switch引用工厂，创建buidler，构建FlowMod, 并写到switch.忽略OpenFlow版本的话，所有OpenFlow对象的构造器是相同的API，但是你需要知道你可以使用的每一个OpenFlow版本，否则，例如你告诉一个OpenFlow1.0的switch去执行一些类似添加组的操作，而这个操作在1.0中并不支持，OpenFlowJ-Loxi库会使用一个UnsupportedOperationException友好提醒你。

介绍其他一些微妙的为了变得很好变化。例如，许多常见的类型，如交换机交换机datapath ids, openflow ports, ip mac 地址被定义到OpenFlowJ-Loxi库中，通过DatapathId, OFPort, IPv4Address/IPv6Address, and MacAddress。你可以搜索org.projectfloodlight.openflow.types，能发现很多常见的可以被定义到单一位置的类型，像上面的buidlers中的produced对象，所有的类型都是不可变的。

更多新的API在Floodlight v1.2中，参阅OpenFlowJ-Loxi文档和例子
