### go delve使用记录



#### 不优化build

```go
// -N: 禁止优化
// -l: 禁止内联
env CGO_ENABLED=0 go build -gcflags="all=-N -l" -o ./v2ray
```



#### 调试

```shell
 dlv exec ./v2ray -- -config ./config.json
```



在使用过程中发现，list命令总是会弹出警告:

Warning: listing may not match stale executable



显示的函数行也不准，google一番无果。