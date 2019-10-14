---
title: wallet_connection
date: 2019-10-14 14:31:54
tags:
categories:
- blockchain
---

## wallet connection

练习使用的库是kotlin的<https://github.com/WalletConnect/kotlin-walletconnect-lib>

然后下载下来，导入到as中...曲折而又艰辛。

- 什么gradle和kotlin版本问题

  ```
    classpath 'com.android.tools.build:gradle:3.4.1'
  ```

  在项目build.gradle中加入

- sample:app不是module

  在settings.gradle中修改

  ```
  include ':lib' , ':sample:app'
  ```

- 云镜像

  ```
  allprojects {
      repositories {
          mavenLocal()
          maven { url 'http://maven.aliyun.com/nexus/content/repositories/central' }
          maven { url 'http://maven.aliyun.com/nexus/content/groups/public' }
          maven { url 'http://maven.aliyun.com/nexus/content/repositories/google' }
          maven { url 'http://maven.aliyun.com/nexus/content/repositories/gradle-plugin' }
          maven { url "https://maven.google.com" }
          /**不需要fabric就不需要此项*/
          maven { url 'http://s3.amazonaws.com/fabric-artifacts/public' }
      }
  }
  ~
  ```

   init.gradle为上述内容，添加到~/.gradle目录下

- 正常情况下就能编译好了，如果出现代码版本不兼容的问题，就根据代码提示修改下



然后就能正常运行和编译啦。

粘贴一个test

```
class WalletConnectBridgeRepositoryIntegrationTest {


    /**
     * Integration test that can be used with the wallet connect example dapp
     */
    @Test
    fun approveSession() {
        val client = OkHttpClient.Builder().pingInterval(1000, TimeUnit.MILLISECONDS).build()
        val moshi = Moshi.Builder().build()
        val sessionDir = File("build/tmp/").apply { mkdirs() }
        val sessionStore = FileWCSessionStore(File(sessionDir, "test_store.json").apply { createNewFile() }, moshi)
        //wc:33...为扫码扫出来的结果
        val config = Session.Config.fromWCUri("wc:79870fae-c04f-479e-8241-085e96603f2b@1?bridge=http%3A%2F%2F192.168.20.18%3A8000%2Fapi%2Fv1%2Fws&key=db8147b92d96a62cb1a2181dd3f9076cfd8eceaeba01feb8becfb965754e6f3e")
        val session = WCSession(
            config,
            MoshiPayloadAdapter(moshi),
            sessionStore,
            OkHttpTransport.Builder(client, moshi),
            Session.PeerMeta(name = "WC Unit Test")
        )
        session.addCallback(object : Session.Callback {
            //相关的状态会回调，如建立连接，连接断掉
            override fun onStatus(status: Session.Status) {
                System.out.println("onStatus: $status")
            }
            //网页有请求，会接收到回调
            override fun onMethodCall(call: Session.MethodCall) {

                if (call is Session.MethodCall.SessionRequest) {
                    //Session.MethodCall.SessionRequest授权请求
                    //回复账户地址，代表授权此账户地址
                    session.approve(listOf("0x32C772DCCF8DCddd3e271B213a0027c22C98Fd1e"), 1L)
                } else if (call is Session.MethodCall.SendTransaction) {
                    //Session.MethodCall.SessionRequest发送交易请求
                    // 可拿到call.to, call.from, call.data 等信息，后进行签名交易，并发送，然后回复处理成功
                    session.approveRequest(call.id, "ok")
                } else if(call is Session.MethodCall.Custom) {
                    //自定义方法和参数
                    //根据call.params call.method进行处理后签名交易，并发送，然后回复处理成功
                    session.approveRequest(call.id, "ok")
                }
            }
        })
        //创建连接
        session.init()
        Thread.sleep(2000000)
    }


    @Test
    fun approveSession1() {
        val client = OkHttpClient.Builder().pingInterval(1000, TimeUnit.MILLISECONDS).build()
        val moshi = Moshi.Builder().build()
        val sessionDir = File("build/tmp/").apply { mkdirs() }
        val sessionStore = FileWCSessionStore(File(sessionDir, "test_store.json").apply { createNewFile() }, moshi)
        val key = ByteArray(32).also { Random().nextBytes(it) }.toNoPrefixHexString()
        val topic  = ByteArray(32).also { Random().nextBytes(it) }.toNoPrefixHexString()
        println("topic: " + topic)
        println("key : $key")
        val config = Session.Config(topic, "http://192.168.20.18:5000", key)
        val session = WCSession(
                config,
                MoshiPayloadAdapter(moshi),
                sessionStore,
                OkHttpTransport.Builder(client, moshi),
                Session.PeerMeta(name = "WC Unit Test")
        )
        session.addCallback(object : Session.Callback {
            override fun onStatus(status: Session.Status) {
                System.out.println("onStatus: $status")
            }

            override fun onMethodCall(call: Session.MethodCall) {
                System.out.println("onMethodCall: $call")
            }
        })
        session.offer()
        Thread.sleep(5000)
        println("==========================")
        val id = Random().nextLong()
        println("id: "+ id)
        val p = session.performMethodCall(Session.MethodCall.Custom(id, "tttt",  listOf(1,2,"3",4,"5")), callback = { resp ->
            println(resp)
        })
        println(p)
        Thread.sleep(100000)
        session.kill()
        Thread.sleep(200000)
    }

}

```
