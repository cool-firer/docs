iterator

```go
&mergedIterator{
  iters:  [
    
    // 0层的
    &indexedIterator{
      
      index: &indexIter{
				blockIter: &blockIter{
          tr: &table.Reader,
          block: b,
          blockReleaser: bReleaser,
          key:             make([]byte, 0),
          dir:             dirSOI,
          riStart:         0,
          riLimit:         b.restartsLen,
          offsetStart:     0,
          offsetRealStart: 0,
          offsetLimit:     b.restartsOffset,
        },
        
        tr: &table.Reader{
          fd: {文件类型, 文件编号},
          reader: &iStorageReader{
            c: 指向session.stor,
            storage.Reader(内嵌): &fileWrap{
              File(内嵌): os.OpenFile指向,
              fs: fileStorage,
              fd: fd
            },
          }, // end reader
          
          cache: &cache.NamespaceGetter{
            Cache: t.bcache 指向session.tops.bcache, 
            NS: uint64(f.fd.Num),
          }, // end cache
,
          bpool: t.bpool,
          o: s.o.Options,
          cmp: o.GetComparer(),
          verifyChecksum: true,
          
          // 读取了footer内容, 得到meta index
          metaBH: blockHandle{ offset位置, length长度 },
          indexBH: blockHandle{ offset位置, length长度 },
          dataEnd: int64(r.metaBH.offset), // meta的位置就是数据结束的位置
          
        }, // end tr
        
				slice: nil,
				fillCache: true,
			}, 
      
      strict: false
    },

    
    // 非0层的
    
  ],
	cmp:	cmp,
	strict: false,
	keys:   make([][]byte, len(iters)),
}
```

