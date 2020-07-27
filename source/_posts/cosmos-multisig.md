---
title: cosmos-multisig
categories:
  - cosmos
date: 2020-01-02 14:33:08
tags:
---



## Cosmos 多重签名



### 创建多签账户

```
gaiacli keys add \
  p1p2p3 \
  --multisig-threshold=2 \
  --multisig=p1,p2,p3
```



```
			var pks []crypto.PubKey
			//参与多签的公钥
			multisigThreshold := viper.GetInt(flagMultiSigThreshold)
			if err := validateMultisigThreshold(multisigThreshold, len(multisigKeys)); err != nil {
				return err
			}

			for _, keyname := range multisigKeys {
				k, err := kb.Get(keyname)
				if err != nil {
					return err
				}
				pks = append(pks, k.GetPubKey())
			}

			// Handle --nosort
			//按地址进行排序
			if !viper.GetBool(flagNoSort) {
				sort.Slice(pks, func(i, j int) bool {
					return bytes.Compare(pks[i].Address(), pks[j].Address()) < 0
				})
			}
	
			pk := multisig.NewPubKeyMultisigThreshold(multisigThreshold, pks)
			if _, err := kb.CreateMulti(name, pk); err != nil {
				return err
			}
```



其实是增加了公钥类型

```
type PubKeyMultisigThreshold struct {
	K       uint            `json:"threshold"`
	PubKeys []crypto.PubKey `json:"pubkeys"`
}

var _ crypto.PubKey = PubKeyMultisigThreshold{}
```



在验签的时候，是调用公钥的方法进行验签。

```
func (pk PubKeyMultisigThreshold) VerifyBytes(msg []byte, marshalledSig []byte) bool {
	var sig Multisignature
	err := cdc.UnmarshalBinaryBare(marshalledSig, &sig)
	if err != nil {
		return false
	}
	size := sig.BitArray.Size()
	// ensure bit array is the correct size
	if len(pk.PubKeys) != size {
		return false
	}
	// ensure size of signature list
	if len(sig.Sigs) < int(pk.K) || len(sig.Sigs) > size {
		return false
	}
	// ensure at least k signatures are set
	if sig.BitArray.NumTrueBitsBefore(size) < int(pk.K) {
		return false
	}
	// index in the list of signatures which we are concerned with.
	sigIndex := 0
	for i := 0; i < size; i++ {
		if sig.BitArray.GetIndex(i) {
			if !pk.PubKeys[i].VerifyBytes(msg, sig.Sigs[sigIndex]) {
				return false
			}
			sigIndex++
		}
	}
	return true
}
```



所以多签是线下完成的。

将多个人的签名会整合到一个签名中。

使用多签签署交易时，命令行如下

```
gaiacli tx send cosmos1570v2fq3twt0f0x02vhxpuzc9jc4yl30q2qned 1000000uatom \
  --from=<multisig_address> \
  --generate-only > unsignedTx.json
```

先产生交易存于文件中。



```
gaiacli tx sign \
  unsignedTx.json \
  --multisig=<multisig_address> \
  --from=p1 \
  --output-document=p1signature.json 
```

p1单独签署交易，签名信息保存于文件中。



```
gaiacli tx sign \
  unsignedTx.json \
  --multisig=<multisig_address> \
  --from=p2 \
  --output-document=p2signature.json
```

p2单独签署交易，签名信息保存于文件中。



```bash
gaiacli tx multisign \
  unsignedTx.json \
  p1p2p3 \
  p1signature.json p2signature.json > signedTx.json
```

将p1的签名与p2的签名进行整合，整合完成后的签名数据保存于文件中



```
gaiacli tx broadcast signedTx.json
```

最后发送交易。