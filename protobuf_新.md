

# 快速tutorial

https://protobuf.dev/getting-started/gotutorial/

repeated：Think of repeated fields as dynamically sized arrays.

当前目录结构：

```shell
tutorial_xxx/
  addressbook.proto
```



编译proto文件：

```shell
tutorial_xxx > protoc -I=. --go_out=. addressbook.proto
```



addressbook.proto生成规则

```protobuf
syntax = "proto3";
package tutorial; // 与go代码一点关系没有, 也与文件名一点关系没有

import "google/protobuf/timestamp.proto";

option go_package = "/tutorialpb"; // 关键是这里
```

会生成以下的结构：

```powershell
tutorial_xxx/
  addressbook.proto
  tutorialpb/
    addressbook.pb.go // 生成的go文件, package就是tutorialpb
```



如果 option go_package = "xxx/tutorialpb"; 会生成以下结构：

```powershell
tutorial_xxx/
  addressbook.proto
  xxx/
    tutorialpb/
      addressbook.pb.go // 生成的go文件, package还是tutorialpb
```



# proto3

官网: https://protobuf.dev/programming-guides/proto3/

```protobuf
syntax = "proto3";

message SearchRequest {
  // 每一个叫field, 定义了3个field
  string query = 1;
  
  // field type: int32
  // field name: page_number
  // field number: 2
  int32 page_number = 2;
  int32 results_per_page = 3;
}
```

field number有限制：

* 当前message中必须唯一；

* 19000到19999 是保留号；

  

field number有特点：

* 1-15号留给频繁使用的字段，因为1-15号使用1个字节编码，而16-2047号用2个字节；







