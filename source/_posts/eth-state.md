---
title: eth_state
categories:
  - eth
date: 2019-10-14 14:54:57
tags:
---

#### state trie的存储模型



state/Trie接口

```
// Trie is a Ethereum Merkle Trie.
type Trie interface {
	TryGet(key []byte) ([]byte, error)
	TryUpdate(key, value []byte) error
	TryDelete(key []byte) error
	Commit(onleaf trie.LeafCallback) (common.Hash, error)
	Hash() common.Hash
	NodeIterator(startKey []byte) trie.NodeIterator
	GetKey([]byte) []byte // TODO(fjl): remove this when SecureTrie is removed
	Prove(key []byte, fromLevel uint, proofDb ethdb.Putter) error
}
```

Trie接口中定义的方法有Get、Put等操作/查询数据的方法，除此之外，Trie作为数据结构，在进行持久化时需要自定义持久化的数据结构，Commit方法则为进行持久化时的操作。

Trie接口的实现主要是SecureTrie。cachedTrie结构中包含了匿名成员SecureTrie，所以cachedTrie也算Trie接口的实现。SecureTrie位于trie包下，内部是trie/Trie结构体(非接口)的封装，trie/Trie结构体是Trie数据结构的具体实现。



state/Database接口

```
// Database wraps access to tries and contract code.
type Database interface {
	// OpenTrie opens the main account trie.
	OpenTrie(root common.Hash) (Trie, error)

	// OpenStorageTrie opens the storage trie of an account.
	OpenStorageTrie(addrHash, root common.Hash) (Trie, error)

	// CopyTrie returns an independent copy of the given trie.
	CopyTrie(Trie) Trie

	// ContractCode retrieves a particular contract's code.
	ContractCode(addrHash, codeHash common.Hash) ([]byte, error)

	// ContractCodeSize retrieves a particular contracts code's size.
	ContractCodeSize(addrHash, codeHash common.Hash) (int, error)

	// TrieDB retrieves the low level trie database used for data storage.
	TrieDB() *trie.Database
}
```

state/Database接口，提供了从数据库读出数据一系列方法，主要是读出数据并组成Trie树数据结构。同时还提供了获取db的接口以供直接操作db。

trie.Database是底层数据库的直接封装实现，增删改查一系列方法。这里使用的是leveldb，leveldb采用key-value的形式，相关知识暂时不展开了。





trie包下的Trie是树的具体实现，Database是数据库的具体实现。

state包下的Trie接口是接口(= =)..  Database主要是围绕state进行的读取接口。其具体实现也是对trie/Trie和trie/Database的封装。cachingDB另外有一层缓存实现。





#### 组织模型

statedb是key-value结构，value类型为stateObject，最后持久化到数据库的只有stateObject中的data，statedb封装了Database和Trie，可直接进行数据操作。从数据库的key-value来看，存储的value为stateObject中的data(Account类型)，组织trie树的key为addr，从数据操作上来看，state可直接进行账户的操作(setBalance、getBalance等等)，其实操作的都是对应stateObject中的data，最后Commit的时候持久化到数据库。中间还涉及别的很多东西，例如缓存、快照等，以供事务处理。

statedb内部封装的操作是可以直接存储数据库的，stateObject中data类型为account，内部直接封装了value的序列化和反序列化。如果想存储非account类型的数据呢，statedb也提供了相关方法`SetState`和`GetState`

```
var (
	stateAddr = common.BytesToAddress([]byte{1, 1, 1, 1, 2})
	key  = common.BytesToHash([]byte("names_service"))
)
func TestSingleTrie(t *testing.T) {
	db := ethdb.NewMemDatabase()
	state, _ := state.New(common.Hash{}, state.NewDatabase(db))
	state.SetState(stateAddr, key, common.BytesToHash([]byte{0, 0, 0}))
	state.IntermediateRoot(false)
	hash := state.GetState(stateAddr, key)
	if hash != common.BytesToHash([]byte{0, 0, 0}) {
		t.Error("unexpected err")
	}
}
```



setState的value的类型为hash。也就是说setState只能存储一个哈希值。如果想存储别的东西，可以直接使用trie/database相关接口进行存储，然后将存储的key作为setState的value，查询时先getState获取key，然后再查询value，这样就可以进行索引到存储的value了。如智能合约代码的存储，stateObject.data中只存储了合约代码的hash，合约代码存放在另外的地方。

trie/Database下提供的直接存储/查询接口为

```
//查询
func (db *Database) Node(hash common.Hash) ([]byte, error)
//存储
func (db *Database) InsertBlob(hash common.Hash, blob []byte)

```



智能合约代码的存储/读取就采用了以上两个方法

```
// ContractCode retrieves a particular contract's code.
func (db *cachingDB) ContractCode(addrHash, codeHash common.Hash) ([]byte, error) {
	code, err := db.db.Node(codeHash)
	if err == nil {
		db.codeSizeCache.Add(codeHash, len(code))
	}
	return code, err
}

func (s *StateDB) Commit(deleteEmptyObjects bool) (root common.Hash, err error) {
  ...
     // Write any contract code associated with the state object
     if stateObject.code != nil && stateObject.dirtyCode {
		  s.db.TrieDB().InsertBlob(common.BytesToHash(stateObject.CodeHash()), stateObject.code)
		  stateObject.dirtyCode = false
	  }
   ...
}
```



以下示例为存储额外的Trie

```
func TestSingleTrie(t *testing.T) {
	db := ethdb.NewMemDatabase()
	state, _ := state.New(common.Hash{}, state.NewDatabase(db))
	state.SetState(stateAddr, key, common.BytesToHash([]byte{0, 0, 0}))
	state.IntermediateRoot(false)
	hash := state.GetState(stateAddr, key)
	if hash != common.BytesToHash([]byte{0, 0, 0}) {
		t.Error("unexpected err")
	}


	err = trie.TryUpdate([]byte("anny"), []byte("0xccccc"))
	if err != nil {
		t.Error("update err, ", err)
	}
	t.Log(trie.Hash())
	trie.Commit(nil)
	t.Log(trie.Hash())
	err = trie.TryUpdate([]byte("anny"), []byte("0xccccb"))
	if err != nil {
		t.Error("update err, ", err)
	}
	err = trie.TryUpdate([]byte("jim"), []byte("0xbbbbbc"))
	if err != nil {
		t.Error("update err, ", err)
	}
	trie.Commit(nil)
	t.Log(trie.Hash())
	state.SetState(stateAddr, key, trie.Hash())
	hash = state.GetState(stateAddr, key)
	if hash != trie.Hash() {
		t.Error("unexpected err")
	}
	trie1, err := state.Database().OpenStorageTrie(common.Hash{}, hash)
	if err != nil {
		t.Error("unexpected err", err)
	}
	value, err := trie1.TryGet([]byte("anny"))
	if err != nil {
		t.Error("unexpected err", err)
	}
	if string(value) != "0xccccb" {
		t.Error("unexpected err")
	}
}
```
