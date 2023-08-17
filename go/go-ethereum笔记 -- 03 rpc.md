net.Pipe()

```go
clientConn, serverConn := net.Pipe()

```

创建内存管道。



  s, err := newSession(stor, o)

```go
&session{
  stor: &iStorage{
			Storage(内嵌): &fileStorage{
											mu Mutex
											path: "XXX_DATA/chain_data/geth/chaindata",
											readOnly: false,
											flock:    &unixFileLock{f: 指向LOCK},
											logw:     logw, // log文件指针 打开 chaindata/LOG
											logSize:  logSize, // log文件大小, 每写一次会累加
											slock: &fileStorageLock{fs: 指向自己}
			},
			read:0,
			write:0
	},

	storLock: 指向stor.slock,
	refCh:     make(chan *vTask),
	relCh:     make(chan *vTask),
	deltaCh:   make(chan *vDelta),
	abandon:   make(chan int64),
	fileRefCh: make(chan chan map[int64]int),
	closeC:    make(chan struct{}),

  icmp: &iComparer{ ucmp: bytesComparer{} },
  
	o: &cachedOptions{
		Options(内嵌): &opt.Options{
			Filter:      filter.NewBloomFilter(10),
			DisableSeeksCompaction: true,
			OpenFilesCacheCapacity: 16
			BlockCacheCapacity: 8MiB
			WriteBuffer: 4MiB,
			Strict: StrictJournalChecksum | 
							StrictBlockChecksum | 
							StrictCompaction | 
							StrictReader,
			Comparer: &iComparer{ ucmp: bytesComparer{} }, // 同s指向
			Filter: &iFilter{ Filter: 就是外层的BoolmFilter(10) },
		}, // end Options

		compactionExpandLimit []int:
			[ 2MiB*25, 剩余6个一样的 ]

		compactionGPOverlaps  []int:
			[ 2MiB*10, 剩余6个一样的 ]

		compactionSourceLimit []int:
			[ 2MiB*1, 剩余6个一样的 ]

		compactionTableSize   []int:
			[ 2MiB*1, 剩余6个一样的 ]

		compactionTotalSize   []int64:
			[ 10MiB*10^0, 10MiB*10^1, ..., 10MiB*10^6 ]
	}, // end o

	// newTableOps(s) 后添加
	tops: &tOps{
		s:            指回外层session,
		noSync:       false,
		evictRemoved: false,
		cache: &Cache{
							mHead: unsafePointer指向 &mNode{
								buckets: make([]unsafe.Pointer, 16),
									[ &mBucket{}, &mBucket{}, ... ] 16个全部指向空的
								mask: 16 - 1,
								growThreshold: 16 * 32,
								shrinkThreshold: 0,
							},
							cacher:  &lru{ 
												capacity: 16,
												used: 0,
												recent: lruNode{ next: 指向自己, prev:, 指向自己},
              },
		},
		bcache: &Cache{ // 跟cache一样的结构, 只是lru容量不同
							mHead: unsafePointer指向 &mNode{
								buckets: make([]unsafe.Pointer, 16),
									[ &mBucket{}, &mBucket{}, ... ] 16个全部指向空的
								mask: 16 - 1,
								growThreshold: 16 * 32,
								shrinkThreshold: 0,
							},
							cacher:  &lru{ 
												capacity: 8MiB,
												used: 0,
												recent: lruNode{ next: 指向自己, prev:, 指向自己},
              },
		},
		bpool: &BufferPool{
						baseline: [ 4KiB/4, 4KiB/2, 4KiB, 4KiB*2 4KiB*4 ], // int[5]数组
						pool: [6]sync.Pool[
							sync.Pool{
                New: func() interface{} { return new([]byte) },
							},
							... 6个一样的sync.Pool函数
 					]
		},
	}, // end tops

  // newVersion后, 原子 由初始值 + 1
  ntVersionId: 1, // 0 -> 1
  vmu sync.Mutex
  
  // setVersion后添加
  stVersion: &version{
              s: 指外层session, 
              id: 0, // 每新建一个version, 都从ntVersionId里拿
    					ref: 1, // incref里增加
  },

} // end session
```

之后，是与go s.refLoop() 的交互。

```js

cur Goro                                            refLoop Goro

&vTask{vid: 0, files: nil, created: time.Now()}  ->  s.refCh
                                                     收到, 放在协程栈
																											ref: {
                                                        0 -> vTask,
                                                      }
```

cur goro发完往下走，s.log("xx")：

```go
s.log("log@legend F·NumFile S·FileSize N·Entry C·BadEntry B·BadBlock Ke·KeyError D·DroppedEntry L·Level Q·SeqNum T·TimeElapsed")

// 最终调用
func (fs *fileStorage) Log(str string) {
}

在chaindata/LOG文件写入：
=============== Aug 4, 2023 (CST)(当前时间) ===============
15:30:25.668299 log@legend F·NumFile S·FileSize N·Entry C·BadEntry B·BadBlock Ke·KeyError D·DroppedEntry L·Level Q·SeqNum T·TimeElapsed

```

到此， s := newSession(stor, o) 完成。

<br />

err = s.recover()开始，

当前是空的，先走一下  err = s.create()：

```javascript
fd := storage.FileDesc{
  Type: storage.TypeManifest, 
  Num: 0, // 文件编号, 每次都从session.stNextFileNum加+1, 返回 - 1
}

// s.stor.Create(fd)返回
&fileWrap{
  // 创建文件, 返回 *File
  File: *File 指向 "chaindata/MANIFEST-000000", 
  fs: 指向fileStorage, 
  fd: 拷贝一份fd FileDesc
}

// session改变 open 0 -> 1
&session{
  stor: &iStorage{
			Storage(内嵌): &fileStorage{
											mu Mutex
											path: "XXX_DATA/chain_data/geth/chaindata",
											readOnly: false,
											flock:    &unixFileLock{f: 指向LOCK},
											logw:     logw,
											logSize:  logSize,
											slock: &fileStorageLock{fs: 指向自己},
                      open: 1, // open增加1
			},
	},
}
  

jw := &Writer{
		w: &fileWrap{ // 就是上面的&fileWrap
  			File: *File 指向 "chaindata/MANIFEST-000000", 
  			fs: 指向fileStorage, 
  			fd: storage.FileDesc{
  						Type: storage.TypeManifest, 
  						Num: 0,
				},
		},
		f: w转成flusher接口后的,
}

```

v = s.version()：改动了session:

```javascript
&session{
  stVersion: &version{
              s: 指外层session, 
              id: 0,
    					ref: 2,// 引用 1 -> 2
  },
}
```

 s.fillRecord(rec, true): 填充rec

```javascript
&sessionRecord{
  //    seqNum nextFileNum journalNum comparer  0
	hasRec: 1         1         1          1      0
	nextFileNum: 1 // 从sessions.tNextFileNum 取的值, 当前是1,
  journalNum: 0,
  seqNum: 0,
  comparer: "leveldb.BytewiseComparator",
}
```

  v.fillRecord(rec)：

```javascript
// 当前v没有levels, 不处理, 爽
```

  w, err := jw.Next()：

```javascript
jw := &Writer{
		w: &fileWrap{ // 就是上面的&fileWrap
  			File: *File 指向 "chaindata/MANIFEST-000000", 
  			fs: 指向fileStorage, 
  			fd: storage.FileDesc{
  						Type: storage.TypeManifest, 
  						Num: 0,
				},
		},
		f: w转成flusher接口后的,
    seq: 1, ++,
    i: 0,
    j: 0 + 7(headerSize),
    first: true,
    pending: true,
    buf [blockSize]byte, // blockSize: 32 * 1024 = 4KB
}

返回的w = singleWriter{
  w: 指向jw,
  seq: 1,
}
```

 err = rec.encode(w)：把rec编码进w里，最终也是修改jw：

```javascript
jw := &Writer{
		w: &fileWrap{
  			File: *File 指向 "chaindata/MANIFEST-000000", 
  			fs: 指向fileStorage, 
  			fd: storage.FileDesc{
  						Type: storage.TypeManifest, 
  						Num: 0,
				},
		},
		f: w转成flusher接口后的,
    seq: 1,
    i: 0,
      
    // 修改1
    j: 初值7 --> 随着写入, j往下走, j是可写入的下标,
    first: true,
    pending: true,
    // 修改2
    buf: [ // 一个buf就是一个block大小
      		header(7个byte), 
          1(recComparer), 变长
      		len([]byte("下面的字符串")), 变长
      		"leveldb.BytewiseComparator",
            
          2(recJournalNum),
          0,
          
          3(recNextFileNum),
          1,
            
					4(recSeqNum),
          0,
      ],
}

```

 err = jw.Flush()：写入底层文件，会修改jw：

```javascript
jw := &Writer{
		w: &fileWrap{
  			File: *os.File 指向 "chaindata/MANIFEST-000000", 
  			fs: 指向fileStorage, 
  			fd: storage.FileDesc{
  						Type: storage.TypeManifest, 
  						Num: 0,
				},
		},
		f: w转成flusher接口后的, 这里是nil, 因为w、内嵌的File都没有实现flusher接口,
    // 修改1  1 -> 2 自增
    seq: 2,
    i: 0,
    j: 初值7 --> 随着写入, j往下走, j是可写入的下标,
    first: true,
    // 修改2 写入 header
    buf: [
      		4个字节, 写入 7到 j的CRC值, 小端
      		2个字节数据部分长度, 小端
      		1(fullChunkType),
      
          // 这里开始是数据部分
          1(recComparer), 变长
      		len([]byte("下面的字符串")), 变长
      		"leveldb.BytewiseComparator",
            
          2(recJournalNum),
          0,
          
          3(recNextFileNum),
          1,
            
					4(recSeqNum),
          0,
      ],
    
    // 修改3, 填充header后, true -> false
    pending: false,
      
    // 修改4, 写入底层w后
    written: j, 已写入的数据
}
=========================================================================
|   4个字节数据部分CRC   | 2个字节数据部分长度 | 1个字节类型 |    数据部分    |     
========================================================================
                                                                        j的位置
```

此时，manifest-000000文件已经有了内容：

```javascript
4字节CRC 2字节长度 1(fullChunkType) // header, 紧接着是数据
1(comparer) 字符串字节长度 "leveldb.BytewiseComparator"
2(journalNum) 0 3(nextFileNum) 1 4(seqNum) 0
```

<br />

  err = s.stor.SetMeta(fd)：写进chain_data/geth/chaindata/CURRENT.0文件：

```javascript
MANIFEST-000000\n
```

再把CURRENT.0重命名为CURRENT。

然后，作者开始又发神经了，在defer里写业务：

s.recordCommited(rec)：

```javascript
省的上去翻，当前rec是：
&sessionRecord{
  //    seqNum nextFileNum journalNum comparer  0
	hasRec: 1         1         1          1      0
	nextFileNum: 1 // 从sessions.tNextFileNum 取的值, 当前是1,
  journalNum: 0,
  seqNum: 0,
  comparer: "leveldb.BytewiseComparator",
}
  
会修改sessoin里的：
&session: {
  stJournalNum: 0, // 从rec里取来
  stSeqNum: 0, // 从rec里取来
  manifestFd: storage.FileDesc{ // 拷贝一个
  							Type: storage.TypeManifest, 
  							Num: 0,
							},
  manifestWriter: &fileWrap{ // 把 w 赋给
  								File: *File 指向 "chaindata/MANIFEST-000000", 
  								fs: 指向fileStorage, 
  								fd: 拷贝一份fd FileDesc
	},
  manifest: &Writer{ // 把 jw 赋给
		w: &fileWrap{ // 就是 manifestWriter指向的 fileWrap
  			File: *os.File 指向 "chaindata/MANIFEST-000000", 
  			fs: 指向fileStorage, 
  			fd: storage.FileDesc{
  						Type: storage.TypeManifest, 
  						Num: 0,
				},
		},
		f: w转成flusher接口后的, 这里是nil, 因为w、内嵌的File都没有实现flusher接口,
    seq: 2,
    i: 0,
    j: j是可写入的下标,
    first: true,
    buf: [
      		4个字节, 写入 7到 j的CRC值, 小端
      		2个字节数据部分长度, 小端
      		1(fullChunkType),
      
          // 这里开始是数据部分
          1(recComparer), 变长
      		len([]byte("下面的字符串")), 变长
      		"leveldb.BytewiseComparator",
            
          2(recJournalNum),
          0,
          
          3(recNextFileNum),
          1,
            
					4(recSeqNum),
          0,
      ],
    pending: false,
    written: j, 已写入的数据
  }

}
```

到此 s.create()完成，创建写入了manfiest文件、current文件。



开始openDB(s)：

```go
先新建一个db
db := &DB{
		s: s,
		seq: s.stSeqNum, 0
		// MemDB  这里看起来就是 MemTable了, 应该是个skiplist
		memPool: make(chan *memdb.DB, 1),
		// Snapshot
		snapsList: list.New(),
    
		// Write
		batchPool:    sync.Pool{New: newBatch},
		writeMergeC:  make(chan writeMerge),
		writeMergedC: make(chan bool),
		writeLockC:   make(chan struct{}, 1),
		writeAckC:    make(chan error),
    
		// Compaction
		tcompCmdC:   make(chan cCmd),
		tcompPauseC: make(chan chan<- struct{}),
		mcompCmdC:   make(chan cCmd),
		compErrC:    make(chan error),
		compPerErrC: make(chan error),
		compErrSetC: make(chan error),
		// Close
		closeC: make(chan struct{}),
}
```

调用    if err := db.recoverJournal()：恢复日志：

```javascript
先调用stor.List()，遍历 get/chaindata/下的所有文件, 找出journal文件:
<< 文件命名规则：文件编号.log、文件编号.ldb/文件编号.sst >>
rawFds, err := db.s.stor.List(storage.TypeJournal)
现在取不到，因为是空的，rawFds = []。


再调用	if _, err := db.newMem(0)：
	fd: storage.FileDesc{
    		Type: storage.TypeJournal, 
        Num: db.s.allocFileNum() 1, // 此时 s.stNextFileNum是2
  }
	
此时db增加的字段有：
&db: {
  journal: &Writer{
		w: &fileWrap{
    		File: *os.File指针 "chaindata/000001.log", 创建了
    		fs: 指向fileStorage, 同时 fileStorage.open++
    		fd: storage.FileDesc{
    					Type: storage.TypeJournal, 
        			Num: db.s.allocFileNum() 1, // 此时 s.stNextFileNum是2
  			}
  	},
		f: w转成flusher接口后的,
  },
  journalWriter: &fileWrap{ // 就是上面的w
    		File: *os.File指针 "chaindata/000001.log", 创建了
    		fs: 指向fileStorage
    		fd: fd
  	},
  journalFd: fd,
  frozenMem: nil,
  mem: &memDB{
    db: 指回db
    DB: &DB(内嵌){
      cmp:       指向s.icmp,
      rnd:       rand.New(rand.NewSource(0xdeadbeef)),
			maxHeight: 1,
			kvData:    make([]byte, 0, 4MiB), // 从s.options里取的
			nodeData:  make([]int, 16),
        [ 3 ] = 12
      prevNode:  [12]int,
      // put的时候
      kvSize: len(key) + len(value),
      n: ++, // n标识有几个结点(entry)
		 },
      
     ref: 2
	},
  fronzenSeq: 0,
}

```

  rec.setJournalNum(db.journalFd.Num 1)：

  rec.setSeqNum(db.seq 0)  ：

```javascript
rec: &sessionRecord{
  //    seqNum nextFileNum journalNum comparer  0
	hasRec: 1         0         1          0      0
  journalNum: 1,
  seqNum: 0,
}
```

  if err := db.s.commit(rec, false)：

```javascript
// 取一下version	
v := s.version()

nv = &version{
  s: session, 
  id: 2, 从s.ntVersionId增加取 s.ntVersionId变为3
  levels: make([]tFiles, 0),
  cLevel: -1,
  cScore: -1,
}
  
  
err = s.flushManifest(rec)
修改rec:
  //    seqNum nextFileNum journalNum comparer  0
	hasRec: 1         1         1          0      0
  journalNum: 1,
  seqNum: 0,
  nextFileNum: 2,
}
 
```

再次写入manfiest。





Batch：

```go
&Batch{
  data: [ kt, len(key), key, len(value), value ]
  index: [ 
    { keyPos: key在data的位置, valuePos: value在Data的位置, kt },
  ],
  internalLen: len(key) + len(value) + 8
}
```



t.create(0)：tWriter：

```go
w = &tWriter{
	t:  tOps,
  fd: FileDesc{ Type: storage.TypeTable, Num: t.s.allocFileNum()},
	w:  &fileWrap{
    File: *os.OpenFile 创建打开 "000002.ldb", 
    fs: fs, 
    fd: fd
  },
	tw: &Writer{
		writer:          w,
		cmp:             o.GetComparer(),
		filter:          o.GetFilter(), // filter.NewBloomFilter(10)
		compression:     o.GetCompression(), // SnappyCompression
		blockSize:       o.GetBlockSize(), // 4KiB
		comparerScratch: make([]byte, 0),
		bpool:           pool,
    
    scratch          [50]byte,
    
		dataBlock: blockWriter{
      buf: Buffer{buf: [], },
      restartInterval: 16, // 16个kv 1个restart
      scratch: scratch[20:],
      prevKey: nil,
      nEntries: 0,
    },
    restarts []uint32: [ 保存restart在buf的下标 ]
    
    indexBlock: blockWriter{
      buf:
      restartInterval: 1,
      scratch: scratch[20:],
      prevKey: nil,
      nEntries: 0,
    },
    
    filterBlock: filterWriter{
      generator: &bloomFilterGenerator{
        n: int(f), // 10
        k: 6, // hash函数个数
        keyHashes []uint32, // add key 时append
      },
      baseLg: 11,
      nKeys: 0, // add key时++,
      offsets []uint32,
      buf: util.Buffer{
        buf []byte
        off int
      },
    },
    
    nEntries: 0, // append时++
    pendingBH: nil, blockHandle struct {
      offset: w.offset旧值
      length: 写入的量,
    }
    
    offset: 0, // writeBlock时 + 已写入的量
    
	}, // end tw
  
  first: nil,
  last: nil,
} // end writer
```



bytes.Compare()测试

```go
package main

import ( "bytes" "fmt" )

func main() {
	var a, b []byte
	b = []byte{'a'}
  bytes.Compare(a, b) // -1 a < b 空与有值比较

  a = []byte{'a'}
  b = []byte{'b'}
  bytes.Compare(a, b) // -1 a < b
  
 	a = []byte{'a', 'b'}
	b = []byte{'b'}
  bytes.Compare(a, b) // -1 a < b
  
  a = []byte{'a', 'b'}
	b = []byte{'a', 'b', 'a'}
  bytes.Compare(a, b) // -1 a < b
}


```





读取目录下的文件：

```go
	dir, err := os.Open(fs.path)
	if err != nil {
		return FileDesc{}, err
	}
	names, err := dir.Readdirnames(0) // 读取目录下所有文件
	// 要close
	if ce := dir.Close(); ce != nil {
		fs.log(fmt.Sprintf("close dir: %v", ce))
	}
```

