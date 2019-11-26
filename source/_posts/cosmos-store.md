---
title: cosmos-store
categories:
  - cosmos
date: 2019-11-26 14:34:13
tags:
---



###  Cosmos state

 在使用时有用到查询存储的值的api，这个api不仅返回了key对应的value，还返回了proof。使用proof可证明这个value的确存储在链上。这是怎么证明的呢？

翻了下源码，大概了解到是怎么证明的了。随手也记录一下。

首先，在cosms-sdk中的设计是插件式，可以据情况使用`x/`包下的插件。

在存储上的话，使用的是`store/rootmulti/store.go`..  ，根据包名猜测下是会有多颗树，然后每颗一个root，多个root又组成一颗树。根据上面的插件，所以就是据情况多个插件，多颗树吧。

#### 源码分析

`store/rootmulti/store.go`

```
type Store struct {
		db           dbm.DB  //用于持久化存储的db
		lastCommitID types.CommitID
		pruningOpts  types.PruningOptions
		storesParams map[types.StoreKey]storeParams //1
		stores       map[types.StoreKey]types.CommitKVStore //2
		keysByName   map[string]types.StoreKey  //3
		lazyLoading  bool

		traceWriter  io.Writer
		traceContext types.TraceContext

		interBlockCache types.MultiStorePersistentCache
}
```

主要看到1、2、3处为map结构。3的话比较明显是key是string类型，value是封装的某Key类型，这个某Key类型刚好是1和2的 key，注意到1的value类型是storeParams，好像只是参数，不是我想要的树结构。但是2的类型`types.CommitKVStore`，看起来有点像。`types.CommitKVStore`是接口类型，实现有`store/transient/store.go`和`store/iavl/store.go`，这里iavl好像很熟悉...gaia中对store实例化默认选择的是iavl。那么`ival`就是我想找的答案啦。



#####  ival

`store/iavl/store.go`里有我想找的tree结构，但...时间限制..我没看实现🤦‍♀️

简单而言，我选择从测试入手... 首先是这里面的tree的存储是带有版本的..相当于会根据版本存储

```
//store/iavl/store_test.go

var (
	treeData = map[string]string{
		"hello": "goodbye",
		"aloha": "shalom",
	}
	nMoreData = 0
)

func newAlohaTree(t *testing.T, db dbm.DB) (*iavl.MutableTree, types.CommitID) {
	tree := iavl.NewMutableTree(db, cacheSize)
	for k, v := range treeData {
		tree.Set([]byte(k), []byte(v))  //"hello" => "goodbye"; "aloha" =>"shalom"
	}
	for i := 0; i < nMoreData; i++ {
		key := cmn.RandBytes(12)
		value := cmn.RandBytes(50)
		tree.Set(key, value)
	}
	hash, ver, err := tree.SaveVersion()
	require.Nil(t, err)
	return tree, types.CommitID{Version: ver, Hash: hash}
}


func TestGetImmutable(t *testing.T) {
	db := dbm.NewMemDB()
	tree, cID := newAlohaTree(t, db)   //初始化tree并且放入一些key-value对，并且saveVersion一次
																		//version=1
	store := UnsafeNewStore(tree, 10, 10)
	//	将tree中的key="hello"的value由"goodbye"更改为"adios"
	require.True(t, tree.Set([]byte("hello"), []byte("adios")))  															
	hash, ver, err := tree.SaveVersion()  //ver再次更新，此时version=2
	cID = types.CommitID{Version: ver, Hash: hash}  //hash => root
	require.Nil(t, err)
	_, err = store.GetImmutable(cID.Version + 1)
	require.Error(t, err)   //查询version=3的存储树，当然是没有的啦，err信息为找不到

	newStore, err := store.GetImmutable(cID.Version - 1)  //查询version=1的存储树，
	require.NoError(t, err)                           //此时hello的value为goodbye
	require.Equal(t, newStore.Get([]byte("hello")), []byte("goodbye"))

	newStore, err = store.GetImmutable(cID.Version)   //查询version=2的存储树
	require.NoError(t, err)													//此时hello的value已更新为adios
	require.Equal(t, newStore.Get([]byte("hello")), []byte("adios"))

  //abci Query接口实现，查询key=hello，version=2的value，并且返回证据
	res := newStore.Query(abci.RequestQuery{Data: []byte("hello"), Height: cID.Version, Path: "/key", Prove: true})
	require.Equal(t, res.Value, []byte("adios"))
	require.NotNil(t, res.Proof)

	require.Panics(t, func() { newStore.Set(nil, nil) })
	require.Panics(t, func() { newStore.Delete(nil) })
	require.Panics(t, func() { newStore.Commit() })
}

```



##### rootmulti

然后再回来看`rootmulti/store`

同样，还是看测试`store/rootmulti/store_test.go`

```
func TestHashStableWithEmptyCommit(t *testing.T) {
	var db dbm.DB = dbm.NewMemDB()
	ms := newMultiStoreWithMounts(db)
	err := ms.LoadLatestVersion()
	require.Nil(t, err)

	commitID := types.CommitID{}
	checkStore(t, ms, commitID, commitID)

	k, v := []byte("wind"), []byte("blows")
  //key=store1下的一棵树store1
	store1 := ms.getStoreByName("store1").(types.KVStore)
	//store1设置k-v
	store1.Set(k, v)

 //ms.Commit, 会调用store1.SaveVersion方法
	cID := ms.Commit()
	require.Equal(t, int64(1), cID.Version)
	hash := cID.Hash
	// make an empty commit, it should update version, but not affect hash
	cID = ms.Commit()
	require.Equal(t, int64(2), cID.Version)
	require.Equal(t, hash, cID.Hash)
}
```

更多测试可在``store/rootmulti/store_test.go`中。



结论是，首先会添加多个key值，每个key一个树，然后这些树的roothash作为rootmulti树的叶子节点，组成树，计算出来的root-hash会作为apphash存储在区块头中。那些小树的叶子节点的值就是存储的具体的内容啦。



证明key-value确实在区块头的方式是，首先证明key-value在某颗小树上(store-key下)，其次计算出小树的roothash，再使用roothash计算出apphash，与区块头的apphash对比即可证明。

在使用的时候，选择的区块高度，证明为到这个区块为止，这个数据是存在的。



最后结论草草结尾，有时间再细化。