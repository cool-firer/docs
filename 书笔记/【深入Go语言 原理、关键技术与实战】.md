# 简介

作者：历冰、朱荣鑫、黄迪旋

2023年4月出版



# 第2章 数据结构源码分析

## 数组

值类型，会拷贝。



## 字符串

底层结构：

```go
type StringHeader struct {
  Data uintptr
  Len int
}
```

字节与符文

```go
f := "Golang编程"
len(f) // 12
utf8.RuneCountInString(f) // 8

for _, g := range []byte(f) {
  fmt.Printf("%c", g) // 地有乱码
}

for _, g := range []rune(f) {
  fmt.Printf("%c", g) // 正常打印
}
```

string对象不可变，多个string可以共享同一底层数据，不用复制：

```go
s1 := "12"
s2 := s1
fmt.Println(stringptr(s1), stringptr(s2)) // 相同地址

s3 := "12"
s4 := s3[:]
fmt.Println(stringptr(s3), stringptr(s4)) // 相同地址

func stringptr(s string) uintptr {
  return (*reflect.StringHeader)(unsafe.Pointer(&s)).Data
}
```

一种性能高的string转byte的方法：

```go
func String2Bytes(s string) []byte {
  // string转成StringHeader
  sh := (*reflect.StringHeader)(unsafe.Pointer(&s))
  
  // 新建一个SliceHeader, 直接指向底层数组
  bh := reflect.SliceHeader{
    Data: sh.Data,
    Len: sh.Len,
    Cap: sh.Len,
  }
  
  // 再转成平常用的[]byte
  return *(*[]byte)(unsafe.Pointer(&bh))
```

避免了内存申请和拷贝，不过只能用于[]byte不会发生改变的场景。



# 第3章 Go语言的并发结构

**sync.Once**

内置会有mutex，多个协程中只运行一次。适用第一次初始化的场景。

```go
p.once.Do(func() {
  ...
})
```



**singleflight.Group**

合并多个协程请求，只执行其中一个协程，其余协程可以拿到执行协程的结果。

```go
func main() {
  var g singleflight.Group
  var wg sync.WaitGroup
  wg.Add(5)
  key := "http://www.baidu.com"
  for i := 0; i < 5; i++ {
    go func(j int) {
      v, _, shared := g.Do(key, func() (interface{}, error) {
        return "result"
      })
      fmt.Println("index: %d, val: %v, shared: %v\n", j, v, shared)
      wg.Done()
    }(i)
  }
  wg.Wait()
}

// 5个协程，只会让其中一个协程运行Do, 其他协程等待这个协程返回结果，直接取结果，再往下。
```



**线程安全map的三种实现方案**

1、自定义的RWMutexMap，读加读锁，写加写锁。

```go
type RWMutexMap struct {
  sync.RWMutex
  m map[int]int
}

func (m *RWMutexMap) Get(k int) (int, bool) {
  m.RLock()
  defer m.RUnlock()
  v, existed := m.m[k]
  return v, existed
}

func (m *RWMutexMap) Set(k int, v int) {
  m.Lock()
  defer m.Unlock()
  m.m[k] = v
}
```

2、orcaman的Concurrent-Map方案。采用分片，分散读写map，降低锁冲突。

```go
type ConcurrentMapShared struct {
  items map[string]interface{}
  sync.RWMutex
}

// 定义成切片
type ConCurrentMap []*ConcurrentMapShared

// 获取key对应的分片
func (m ConCurrentMap) Get(key string) *ConcurrentMapShared {
  return m[uint(fnv32(key) % 32)]
}
```

3、原生的sync.Map，内置两个map，读map，写map。读map是原子的，不用锁。写map需要加锁。适合读多写少。

三者比较：

* 只读下，sync.Map(原子读) > RWMutexMap(读锁) > ConCurrentMap(多一步查找分片)。
* 只写下，ConCurrentMap > RWMutexMap > sync.Map。
* 混合读写下，ConCurrentMap与sync.Map都相当。

需要结合实际项目做benchmark。



# 第5章 Go语言协程





