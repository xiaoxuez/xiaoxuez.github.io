---
title: btc_data_dat
categories:
  - btc
date: 2019-4-14 15:16:51
tags:
---



## btc dat数据重组

Bitcoin core同步的数据是dat文件和index文件夹，index文件夹里是ldb 

dat是向各个节点同步的数据，各个区块之间不保证有序。ldb里保存了如何去dat文件里查询区块/交易等的信息。 



具体维基解释 https://en.bitcoin.it/wiki/Bitcoin_Core_0.11_(ch_2):_Data_Storage 



其中ldb中区块信息为 

 ```
  'b' + 32-byte block hash -> block index record. Each record stores: 

       * The block header 

       * The height. 

       * The number of transactions. 

       * To what extent this block is validated. 

       * In which file, and where in that file, the block data is stored. 

       * In which file, and where in that file, the undo data is stored. 

 ```

实现，重组为有序的dat文件 

```
/*
@Time : 2019/4/4 下午6:57
@Author : xiaoxuez
*/

package main

import (
   "encoding/binary"
   "encoding/hex"
   "errors"
   "flag"
   "fmt"
   "io"
   "os"
   "path"
   "sync"

   "git.wokoworks.com/blockchain/btc-explorer/db"
   "git.wokoworks.com/blockchain/btc-explorer/log"
)

type readBlockChain struct {
   height    uint64
   blockHash [32]byte
}

var indexLDB *db.LDBDatabase
var blSourceDir string
var blDirtyDir string

func main() {
   log.InitHandler(log.Root(), log.LvlTrace, "logs/reorder.log")
   var err error
   flag.StringVar(&blSourceDir, "old", "/Users/xiaoxuez/btc_test_db/mainnet/", "xx")
   flag.StringVar(&blDirtyDir, "new", "/Users/xiaoxuez/btc_test_db/reorder", "xx")
   flag.Parse()
   //bmdb, err = db.NewLDBDatabase("/Users/xiaoxuez/btc_test_db/mainnet/index", 0, 0)
   indexLDB, err = db.NewLDBDatabase(path.Join(blSourceDir, "index"), 0, 0)
   if err != nil {
      panic(err)
   }
   //len must < 24
   var data = []readBlockChain{
      {
         height:    0,
         blockHash: hexToHash("00000000000003a20def7a05a77361b9657ff954b2f2080e135ea6f5970da215"), // 20万-1
      },
      {
         height:    20 * 10000,
         blockHash: hexToHash("0000000000000009c2e82d884ec07b4aafb64ca3ef83baca2b6b0b5eb72c8f02"), // 25万-1
      },
      //{
      // height:    25 * 10000,
      // blockHash: hexToHash("000000000000000067ecc744b5ae34eebbde14d21ca4db51652e4d67e155f07e"), // 30-1
      //},
      //{
      // height:    30 * 10000,
      // blockHash: hexToHash("000000000000000002045664f89a1077d0c6c0aaa6dd89b485208cf92d6bbd30"), // 35-1
      //},
      //{
      // height:    35 * 10000,
      // blockHash: hexToHash("0000000000000000030034b661aed920a9bdf6bbfa6d2e7a021f78481882fa39"), // 40-1
      //},
      //{
      // height:    40 * 10000,
      // blockHash: hexToHash("0000000000000000024c4a35f0485bab79ce341cdd5cc6b15186d9b5b57bf3da"), // 45-1
      //},
      //{
      // height:    45 * 10000,
      // blockHash: hexToHash("0000000000000000007962066dcd6675830883516bcf40047d42740a85eb2919"), // 50
      //},
      //{
      // height:    50 * 10000,
      // blockHash: hexToHash("00000000000000000013dad60a42a3401a8f37ca02f1c00ac5923e674566a3ae"), // 55
      //},
   }
   if len(data) > 24 {
      log.Error("parallel number must < 24", "actual ", len(data))
      return
   }
   var wg sync.WaitGroup
   for index, v := range data {
      wg.Add(1)
      go func(index int, v readBlockChain) {
         rwBlocks(v.blockHash[:], v.height, fmt.Sprint(index), func() { wg.Done() })
      }(index, v)
   }
   wg.Wait()
}

func rwBlocks(startBlock []byte, endHeight uint64, prefix string, callback func()) {
   defer callback()
   fileCache := make(map[uint64]*os.File, 0)
   blockHash := startBlock
   reverWriter := NewReverseWriter(blDirtyDir, 138*1024*1024, prefix)
   defer reverWriter.Flush()
   count := 0
   for {
      content, height, err := rwBlock(fileCache, blockHash)
      if err != nil {
         log.Error("rw block", "err", err.Error())
         return
      }
      blockHash = content[12:44]
      err = reverWriter.Insert(content)
      if err != nil {
         fmt.Println("writer err", err)
         return
      }
      if height == endHeight {
         log.Info("finish to save block height", "height", height)
         return
      }
      if height%3000 == 0 {
         log.Trace("====>", "height", height, "parent block hash", hashToHex(blockHash))
      }
      count++
   }
   log.Info("write file end", "count", count)
}

func rwBlock(fileCache map[uint64]*os.File, block []byte) ([]byte, uint64, error) {
   value, err := indexLDB.Get(append([]byte("b"), block...))
   if err != nil {
      return []byte{}, 0, errors.New(fmt.Sprintf("%s%s", err.Error(), hashToHex(block)))
   }
   bm := parseBlockMeta(value)
   file, ok := fileCache[bm.file]
   if !ok {
      file, err = os.Open(idx2fname(blSourceDir, uint32(bm.file)))
      if err != nil {
         return []byte{}, 0, err
      }
      fileCache[bm.file] = file
   }
   file.Seek(int64(bm.dataPos-8), io.SeekStart)
   res, err := readBlockFromFile(file)

   if err != nil {
      return []byte{}, 0, err
   }
   binary.LittleEndian.PutUint32(res[:4], uint32(bm.height))
   return res, bm.height, err
}

func readBlockFromFile(f *os.File) (res []byte, e error) {
   var buf [4]byte
   _, e = f.Read(buf[:])
   _, e = f.Read(buf[:])
   if e != nil {
      return
   }
   le := uint32(lsb2uint(buf[:]))
   if le < 81 {
      e = errors.New(fmt.Sprintf("Incorrect block size %d", le))
      return
   }

   res = make([]byte, le+8)
   copy(res[4:8], buf[:])
   _, e = f.Read(res[8:])
   if e != nil {
      return
   }
   return
}
func idx2fname(dir string, fidx uint32) (s string) {
   if fidx == 0xffffffff {
      return "blk99999.dat"
   }
   return fmt.Sprintf("%s/blk%05d.dat", dir, fidx)
}
func lsb2uint(lt []byte) (res uint64) {
   for i := 0; i < len(lt); i++ {
      res |= uint64(lt[i]) << uint(i*8)
   }
   return
}

type blockMeta struct {
   version uint64
   height  uint64
   status  uint64
   nTx     uint64
   file    uint64
   dataPos uint64
   undoPos uint64
}

func parseBlockMeta(raw []byte) (bm blockMeta) {
   var pos uint64
   var i uint64
   bm.version, i = readRawData(raw[pos:])
   pos += i
   bm.height, i = readRawData(raw[pos:])
   pos += i
   bm.status, i = readRawData(raw[pos:])
   pos += i
   bm.nTx, i = readRawData(raw[pos:])
   pos += i
   bm.file, i = readRawData(raw[pos:])
   pos += i
   bm.dataPos, i = readRawData(raw[pos:])
   pos += i
   bm.undoPos, i = readRawData(raw[pos:])
   return
}

func readRawData(raw []byte) (uint64, uint64) {
   var n uint64
   var pos uint64
   for {
      data := raw[pos]
      pos += 1
      n = (n << 7) | (uint64(data) & 0x7f)
      if data&0x80 == 0 {
         return n, pos
      }
      n += 1
   }
}

func hashToHex(hash []byte) (s string) {
   for i := 0; i < 32; i++ {
      s += fmt.Sprintf("%02x", hash[31-i])
   }
   return
}

func hexToHash(s string) (res [32]byte) {
   d, e := hex.DecodeString(s)
   if e != nil {
      return
   }
   if len(d) != 32 {
      return
   }
   for i := 0; i < 32; i++ {
      res[31-i] = d[i]
   }
   return
}

type ReverseWriter struct {
   Dir          string
   CacheMaxSize int
   len          int
   cache        []byte
   seek         int
   index        int
   Prefix       string
}

func NewReverseWriter(dir string, cacheSize int, prefix string) *ReverseWriter {
   rw := &ReverseWriter{Dir: dir, CacheMaxSize: cacheSize, Prefix: prefix}
   rw.init()
   return rw
}

func (rw *ReverseWriter) init() {
   rw.len = 0
   rw.cache = make([]byte, rw.CacheMaxSize)
   rw.seek = rw.CacheMaxSize
}

func (rw *ReverseWriter) Insert(content []byte) error {
   if rw.len+len(content) > rw.CacheMaxSize {
      err := rw.writeToFile()
      if err != nil {
         return err
      }
      rw.index++
      rw.init()
   }
   copy(rw.cache[rw.seek-len(content):rw.seek], content)
   rw.seek = rw.seek - len(content)
   rw.len += len(content)
   return nil
}

func (rw *ReverseWriter) writeToFile() error {
   if rw.len == 0 {
      return nil
   }
   writer, err := os.OpenFile(fmt.Sprintf("%s/%sblk%05d.dat", rw.Dir, rw.Prefix, rw.index), os.O_RDWR|os.O_CREATE|os.O_TRUNC, 0666)
   if err != nil {
      return err
   }
   _, err = writer.Write(rw.cache[rw.seek:])
   return err
}

func (rw *ReverseWriter) Flush() error {
   err := rw.writeToFile()
   rw.index++
   rw.init()
   return err
}


重组完成之后，进行更名操作
#!/usr/bin/env bash

# shell is funny

read -p "Are you sure:(y/n) " an
if [[ ${an} != "y" ]]; then exit; fi

cd /Users/woko/bitcoin

function rename_2() {
    index=0
    for name in $(ls | grep .dat) ; do
        if [[ -f ${name} ]]; then
            rename=$(echo ${index}|awk '{printf("blk%05d.dat",$0)}')
            echo ${name} " ==> " ${rename}
            mv ${name} ${rename}
            ((index=index+1))
        fi
    done
}

function rename_1() {
    count=0
    for (( i = 0; i < 10; ++i )); do
        index=$(ls ${count}*.dat | wc -l)
        for name in $(ls ${count}*.dat) ; do
            if [[ -f ${name} ]]; then
                ((index=index-1))
                rename=$(echo "${count} ${index}"|awk '{printf("%02dblk%05d.dat",$1,$2)}')
                echo ${name} " ==> " ${rename}
                mv ${name} ${rename}
            fi
        done
        ((count=count+1))
        echo "--------------"
    done
}

rename_1
echo "=============="
rename_2

```

