

# 环境

go version go1.20.3 darwin/amd64

所有例子来自官方：https://github.com/boltdb/bolt



# 一、基跑

```shell
mkdir boltdbt
cd boltdbt
go mod init boltdbt
go get github.com/boltdb/bolt/...
vi main.go
```

内容：

```go
package main

import (
	"log"

	"github.com/boltdb/bolt"
)

func main() {
	// Open the my.db data file in your current directory.
	// It will be created if it doesn't exist.
	db, err := bolt.Open("my.db", 0600, nil)
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
}
```



# 二、事务

## 读写事务

```go
err := db.Update(func(tx *bolt.Tx) error {
	...
	return nil
})

// 批量读写事务
err := db.Batch(func(tx *bolt.Tx) error {
	...
	return nil
})
```



## 只读事务

```go
err := db.View(func(tx *bolt.Tx) error {
	...
	return nil
})
```



## 手动事务

```go
// Start a writable transaction.
tx, err := db.Begin(true)
if err != nil {
    return err
}
defer tx.Rollback()

// Use the transaction...
_, err := tx.CreateBucket([]byte("MyBucket"))
if err != nil {
    return err
}

// Commit the transaction and check for error.
if err := tx.Commit(); err != nil {
    return err
}
```



# 三、buckets（桶）

桶是装key/value的地方。

## 创建桶

```go
db.Update(func(tx *bolt.Tx) error {
	b, err := tx.CreateBucket([]byte("MyBucket"))
	if err != nil {
		return fmt.Errorf("create bucket: %s", err)
	}
	return nil
})
```

