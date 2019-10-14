---
title: web3j
categories:
  - eth
date: 2019-10-14 14:55:36
tags:
---
## web3j的使用



#### 特性

- 通过Java类型的JSON-RPC与Ethereum客户端进行交互
- 支持所有的JSON-RPC方法类型
- 支持所有Geth和Parity方法，用于管理账户和签署交易
- 同步或异步的发送客户端请求
- 可从Solidity ABI文件自动生成Java只能合约功能包



重点是交互的手段是JSON-RPC！

web3j可以监听事件、发送交易等等，发送交易时上链后方才返回结果。即内部也有监听事件。一直很好奇监听事件是怎么做的，实时推送？

简单翻了下源码，并不是推送，而是轮询。

因为是从智能合约接手入web3j，故从合约生成的java类入手，发现需要上链的方法返回的都是TransactionReceipt类型，例如

```
    public RemoteCall<TransactionReceipt> addUser(byte[] name, String addr, byte[] pk_pre, byte[] pk_bk) {
        final Function function = new Function(
                FUNC_ADDUSER,
                Arrays.<Type>asList(new org.web3j.abi.datatypes.generated.Bytes32(name),
                        new org.web3j.abi.datatypes.Address(addr),
                        new org.web3j.abi.datatypes.generated.Bytes32(pk_pre),
                        new org.web3j.abi.datatypes.generated.Bytes32(pk_bk)),
                Collections.<TypeReference<?>>emptyList());
        return executeRemoteCallTransaction(function);
    }
```

从executeRemoteCallTransaction入手，跟踪进去。看到

```
    TransactionReceipt executeTransaction(String data, BigInteger weiValue, String funcName) throws TransactionException, IOException {
        TransactionReceipt receipt = this.send(this.contractAddress, data, weiValue, this.gasProvider.getGasPrice(funcName), this.gasProvider.getGasLimit(funcName));
        if (!receipt.isStatusOK()) {
            throw new TransactionException(String.format("Transaction has failed with status: %s. Gas used: %d. (not-enough gas?)", receipt.getStatus(), receipt.getGasUsed()));
        } else {
            return receipt;
        }
    }
```

原来交易失败，总是抛出not-enough gas异常是在这里抛出的，根据返回receipt的status判断。

再跟着send方法进去，终于进入到transactionManager，离真相很近了。

然后找到

```
   protected TransactionReceipt executeTransaction(BigInteger gasPrice, BigInteger gasLimit, String to, String data, BigInteger value) throws IOException, TransactionException {
        EthSendTransaction ethSendTransaction = this.sendTransaction(gasPrice, gasLimit, to, data, value);
        return this.processResponse(ethSendTransaction);
    }
```

processResponse肯定就是等待返回结果的地方了。然后内部是调用this.transactionReceiptProcessor.waitForTransactionReceipt，再进去就是个抽象类了。找了一下，在TransactionManager中，实现的具体类是PollingTransactionReceiptProcessor。所以..答案就在这里了

```
 private TransactionReceipt getTransactionReceipt(String transactionHash, long sleepDuration, int attempts) throws IOException, TransactionException {
        Optional<TransactionReceipt> receiptOptional = this.sendTransactionReceiptRequest(transactionHash);

        for(int i = 0; i < attempts; ++i) {
            if (receiptOptional.isPresent()) {
                return (TransactionReceipt)receiptOptional.get();
            }

            try {
                Thread.sleep(sleepDuration);
            } catch (InterruptedException var8) {
                throw new TransactionException(var8);
            }

            receiptOptional = this.sendTransactionReceiptRequest(transactionHash);
        }

        throw new TransactionException("Transaction receipt was not generated after " + sleepDuration * (long)attempts / 1000L + " seconds for transaction: " + transactionHash);
    }
```

**这里可以看到，是每隔sleepDuration就轮询查询一次。sleepDuration的值默认为15s, 查询次数默认最多为40。**
