---
title: eos_code
categories:
  - eos
date: 2019-10-14 14:44:30
tags:
---

### EOS 源码学习之项目结构分析(plugins)

整体的代码位置主要位于plugins、libraries。

**eos项目采用了插件管理的结构**。



#### 入口

programs的文件夹结构如下，在[eos官方wiki](https://github.com/EOSIO/eos/wiki/Programs-&-Tools)中可以查看具体每个目录的含义及作用。这里仅备注几个示例。

```
.. eos/programs:

├── CMakeLists.txt
├── cleos //命令行工具，如果把eos比作操作系统的话，cleos就类似终端作用  
├── debug_node
├── eosio-abigen
├── eosio-applesedemo
├── eosio-launcher
├── genesis
├── keosd  //钱包
├── nodeos  //节点核心进程
└── snapshot
```

要启动一个节点，需要用到的便是nodeos，cleos中可使用的REST API便是nodeos中暴露出来的。那么，我们看看nodeos。

源代码main.cpp中，代码仅有105行。主要看看main方法吧（不想看代码可以略过代码部分 = =），通过代码可以看到，main方法作用包括设置版本、工作路径等基本信息，以及log的初始化，另外就是startup方法看起来就是关键！

```
int main(int argc, char** argv)
{
   try {
      app().set_version(eosio::nodeos::config::version);
      auto root = fc::app_path();
      app().set_default_data_dir(root / "eosio/nodeos/data" );
      app().set_default_config_dir(root / "eosio/nodeos/config" );
      if(!app().initialize<chain_plugin, http_plugin, net_plugin, producer_plugin>(argc, argv))
         return -1;
      initialize_logging();
      ilog("nodeos version ${ver}", ("ver", eosio::utilities::common::itoh(static_cast<uint32_t>(app().version()))));
      ilog("eosio root is ${root}", ("root", root.string()));
      //重要的代码!!!
      app().startup();
      app().exec();
   } catch (const fc::exception& e) {
   } .....(此处省略多个catch)
   return 0;
}
```

app()位置位于

```
./libraries/appbase/application.cpp
//app()方法返回application的实例(对c++的类的机制不了解，这个实例像是静态对象，反正就是通过返回的实例可以调用相关方法吧，包括startup这个看起来就跟关键的方法)
```

进入到application.cpp中，例举相关方法

- initialize_..: 初始化插件，功能包括调用各个插件设置命令行参数，以及初始化方法，还有app自身对应初始化操作

- startup:  循环各插件，调用各插件的startup方法
- exec:  表示完全看不懂= =!!。猜测一下，应该是监听io事件，例如结束进程系列..

从这里可以看出，主要的具体的重要的内容都在各插件内部。



#### 插件

eos项目采用了插件管理的结构。插件的目录如下

```
.. eos/plugins:

├── CMakeLists.txt
├── account_history_api_plugin
├── account_history_plugin
├── chain_api_plugin
├── chain_plugin
├── eosio-make_new_plugin.sh
├── faucet_testnet_plugin
├── http_plugin
├── mongo_db_plugin
├── net_api_plugin
├── net_plugin
├── producer_plugin
├── template_plugin
├── txn_test_gen_plugin
├── wallet_api_plugin
└── wallet_plugin
```

完全可以通过命名来理解各插件的含义。要分析的话，直接从相应plugins入手。逐一观察..


### eos/nodeos学习



#### main.cpp

```
      //设置版本
      app().set_version(eosio::nodeos::config::version);
      auto root = fc::app_path();
      //设置工作路径
      app().set_default_data_dir(root / "eosio/nodeos/data" );
      app().set_default_config_dir(root / "eosio/nodeos/config" );
      //单节点需要的插件包括以下四个
      if(!app().initialize<chain_plugin, http_plugin, net_plugin, producer_plugin>(argc, argv))
         return -1;

      initialize_logging();
      ilog("nodeos version ${ver}", ("ver", eosio::nodeos::config::itoh(static_cast<uint32_t>(app().version()))));
      ilog("eosio root is ${root}", ("root", root.string()));
      app().startup();
      app().exec();


```



主要是配置和app的一些调用。然后转到app中。

```
./libraries/appbase/application.cpp
```

- app:  返回application的实例（实例只是声明，木有初始化，对c++类机制不太了解，需进一步了解，目前就理解成这个实例就可访问application::method方法）

- initialize_..: 初始化插件，功能包括调用各个插件设置命令行参数，以及初始化方法，还有app自身对应初始化操作

- startup:  循环各插件，调用各插件的startup方法

- exec:  表示完全看不懂= =!!。猜测一下，应该是监听io事件，例如结束进程系列..



那么针对单个节点启动必要的4个插件，进行逐个学习。

首先是插件父方法：（在application中调用的方法名直接是start_up，没有plugin前缀，这个问题有待解决..）

- set_program_options //设置命令行参数

- plugin_initialize //初始化操作

- plugin_startup  //启动

- plugin_shutdown  //退出





#### Chain_plugin





#### Producer_plugin



```
 // dpos共识  地方 -> get_scheduled_producer的定义在chain_controller
 auto scheduled_producer = chain.get_scheduled_producer( slot );
   // we must control the producer scheduled to produce the next block.
   if( _producers.find( scheduled_producer ) == _producers.end() )
   {
      capture("scheduled_producer", scheduled_producer);
      return block_production_condition::not_my_turn;
   }
```





```go
//chain_controller的一些相关producer方法

//大意猜测更新新区块，包括新区块和签名新区块的producer
update_signing_producer(const producer_object& signing_producer, const signed_block& new_block)

//根据新的producers更新db表？疑问是代码中只有create，没有删除操作
update_or_create_producers( const producer_schedule_type& producers)

//好像这个是关键！详细看一下，对方法的解释为，从投票排名中取出前m个producer,并排除block_signing_key是null的那些，m为配置中的producer_count。然而代码似乎没全..
_calculate_producer_schedule() {
  //get_global_properties返回为global_property_object对象，new_active_producers为对象的熟悉
  //据观察，new_active_producers的赋值目前只出现在wasm的api中，那是不是就可以得出结论new_active_producers是所有的参选producers..这个方法(_calculate_producer_schedule)要做的就是从所有参选者中按排名取前m个，然后去掉block_signing_key是null的那些
   producer_schedule_type schedule = get_global_properties().new_active_producers;
   const auto& hps = _head_producer_schedule();
   schedule.version = hps.version;
   if( hps != schedule )
      ++schedule.version;
   return schedule;
}

//没看懂..好像是给所有producers怎么搞了一下认证(更新到_db某表)
_update_producers_authority()

//返回当前区块头的producer
head_block_producer()

//根据account_name/ownername获取对于producer
get_producer(const account_name& ownername)


//这个方法也蛮重要的，根据slot_num获取已计划好的producer.其中，slot_num始终对应于未来的某一时刻；另外，对于slot_num的值，相对当前多少区块的间隔，例如，slot_num == 1 则为下一个producer, slot_num == 2，则为在一个区块间隔后的下一个producer。注意的是slot_num代表的是区块间隔，非producer间隔
get_scheduled_producer(uint32_t slot_num) {
   const dynamic_global_property_object& dpo = get_dynamic_global_properties();
  //注意点1，最后用来计算位置的值是一个当前的绝对位置 + slot_num  ？具体回头再看
   uint64_t current_aslot = dpo.current_absolute_slot + slot_num;
   ...
  //位置对producer*重复次数的积取模
   auto index = current_aslot % (number_of_active_producers * config::producer_repetitions);
}

//计算producer过去生产的参与率，代码未详
producer_participation_rate()



//大概看了以上方法，有点乱。现在还不能连成线，只是点
关键
update_last_irreversible_block
```




### producer源码学习

代码位置为..eos/plugins/producer_plugin/

文件夹结构如下，producer_plugin.hpp为方法的声明， producer_plugin.cpp为具体实现

```
├── CMakeLists.txt
├── include
│   └── eosio
│       └── producer_plugin
│           └── producer_plugin.hpp
└── producer_plugin.cpp

```



虽然producer_plugin.cpp的代码也只有不到400行，但节省时间和资源，就挑出块相关的说吧。方法调用如下

```
plugin_startup -> schedule_production_loop(到没到出块时间，到了就出块,这个方法不断循环) -> block_production_loop -> maybe_produce_block
```

maybe_produce_block方法是最后决定是不是出块的。以下主要解释maybe_produce_block方法

- 首先会检查区块是否同步到最新。

- 接下来这段代码会判断是否到达出块时间。

  ```
   uint32_t slot = chain.get_slot_at_time( now );
     if( slot == 0 )
     {
        capture("next_time", chain.get_slot_time(1));
        return block_production_condition::not_time_yet;
     }
  ```

- 然后就选择出块producer了。

  ```
  auto scheduled_producer = chain.get_scheduled_producer( slot );
  ```

完了判断这个scheduled_producer是不是自己，是的话，再检查点别的(例如私钥啊等)，最后就出块了。

那么，关键代码就是get_scheduled_producer方法。get_scheduled_producer的追踪要到chain_controller.cpp文件中了。位置位于./eos/libriaries/chain/chain_controller.cpp。以下代码未做特殊说明，都位于chain_controller.cpp中。

首先看get_scheduled_producer的代码，可以看到，③通过从db中读出global_property_object对象gpo（下面会有介绍），gpo.active_producers为当前活跃的producers，然后④⑤为算得当前索引下标index，最后根据index在gpo.active_producers中取出相应producer即为返回值。下标计算的方式就是数学算式index=当前区块位置 % （producers总数 * 每个producer重复出多少块）。最后再除以每个producer重复出多少块。比较不理解的是当前区块位置(代码中的current_aslot)的计算方式。

```
//具体代码
/*
* 1. slot_num始终对应于未来的某一时刻
* 2.对于slot_num的值，相对当前多少区块的间隔，
*     例如，slot_num == 1 则为下一个区块的producer,
*          slot_num == 2，则为在一个区块间隔后的下一个producer。slot_num即为区块间隔数
* 3.使用方法get_slot_time和get_slot_at_time作为slot_num和时间戳的转换
* 4.slot_num == 0，返回EOS_NULL_PRODUCER
*/
account_name chain_controller::get_scheduled_producer(uint32_t slot_num)const
{
   const dynamic_global_property_object& dpo = get_dynamic_global_properties(); //①
   uint64_t current_aslot = dpo.current_absolute_slot + slot_num; //②
   const auto& gpo = _db.get<global_property_object>(); //③
   auto number_of_active_producers = gpo.active_producers.producers.size();
   auto index = current_aslot % (number_of_active_producers * config::producer_repetitions);  //④
   index /= config::producer_repetitions; //⑤
   FC_ASSERT( gpo.active_producers.producers.size() > 0, "no producers defined" );

   return gpo.active_producers.producers[index].producer_name;
}
```



关于global\_property\_object主要属性有active\_producers、new\_active\_producers、pending\_active\_producers。

active\_producers是当前轮的producers, pending\_active\_producers为满足排名等要求的producers, new\_active\_producers为所有可投票(或者是所有备选)的producer。

在controller_chain.cpp中有如下几个方法：

```
//从投票排名中取出前m个producer,并排除block_signing_key是null的那些，m为配置中的producer_count。然而代码似乎没全.
_calculate_producer_schedule() {...}

//更新global_properties
update_global_properties(){...}

//上面提到过的有可能更新active_producers的方法
update_last_irreversible_block(){...}

 /*
 *  After applying all transactions successfully we can update
 *  the current block time, block number, producer stats, etc
 */
_finalize_block(){...}
```

主要通过上面列举的方法介绍active\_producers、new\_active\_producers、pending\_active\_producers是如何关联的。下面一段如果觉得各个方法调用比较蒙的话，可以直接看三个加粗句子，可以了解到active\_producers、new\_active\_producers、pending\_active\_producers之间的关联。

在\_finalize\_block调用的时候，首先调用update\_global\_properties，在update\_global\_properties方法中调用了_calculate\_producer\_schedule方法，calculate\_producer\_schedule方法实现的是**从new\_active\_producers中选出符合条件的producers**，然后回到update\_global\_properties，**根据选出的producers,更新pending\_active\_producers**，然后回到\_finalize\_block方法，调用update\_global\_properties之后调用update_last_irreversible_block方法，**通过pending_active_producers适当更新active\_producers**。

然后，new\_active\_producers是通过wasm的api进行更新。这样就缕通了，通过投票客户端，进行投票，并控制更新new\_active\_producers参数选手，和相应票数。然后根据new\_active\_producers和票数决定blocker producers。

那么，blocker producers的顺序呢？?? 不知道是被我疏忽了还是真的没看到。



producer源码中，可以看到，上层逻辑在 producer_plugin.cpp中，基本的dpos等逻辑还是在chain_controller.cpp中。
