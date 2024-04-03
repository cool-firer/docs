作者：卡山 - 因德拉西里

2021年1月版



# 第2章 开始使用gRPC

**1、下载安装pb编译器**:  https://github.com/protocolbuffers/protobuf/releases/tag/v26.1.

现在最新版本v26(2024/3/25日发布)

```shell
> protoc --version
libprotoc 3.7.1

> which protoc
/usr/local/bin/protoc
```



v3.12.0（2020/5/16日发布)

下载protoc-3.12.0-osx-x86_64.zip，解压，直接覆盖原来的。

```shell
> protoc --version
libprotoc 3.12.0
```



**2、安装gRPC库**

```shell
> go get -u google.golang.org/grpc
```

最新版本是v1.62.1 (2025/5/5日发布) 当前用的这个

v1.28.0 (2020/3/10日发布)



**3、安装go插件**

```shell
> go install google.golang.org/protobuf/cmd/protoc-gen-go@latest

> go get google.golang.org/grpc/cmd/protoc-gen-go-grpc
> go install google.golang.org/grpc/cmd/protoc-gen-go-grpc
```



**4、当前目录结构**

```shell
productinfo/
  ecommerce/
    ProductInfo.proto
  go.mod
  go.sum
  product_service.go
  main.go
```

进入ecommer目录，编译：

```shell
ecommerce > protoc -I=. --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative ProductInfo.proto
```

直接就会在当前下生成pb.go、grpc.pb.go文件。

