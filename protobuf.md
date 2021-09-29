#### 以下内容是proto3的

- [以下内容是proto3的](#以下内容是proto3的)
- [安装编码器](#安装编码器)
- [编绎proto文件](#编绎proto文件)
- [举个例子](#举个例子)
  - [新建文件 addressbook.proto](#新建文件-addressbookproto)
  - [跑编译](#跑编译)
- [package和option go_package](#package和option-go_package)
- [消息go对应](#消息go对应)
  - [命名字段](#命名字段)
  - [标量字段](#标量字段)
  - [消息字段](#消息字段)
  - [重复字段](#重复字段)
  - [map字段](#map字段)
  - [oneof字段](#oneof字段)
  - [枚举字段](#枚举字段)
- [字段数字(Field Numbers)](#字段数字field-numbers)
- [repeated、optional、required、enum](#repeatedoptionalrequiredenum)
- [自定义字段类型](#自定义字段类型)


#### 安装编码器

https://developers.google.com/protocol-buffers/docs/downloads

下载protoc-3.18.0-osx-x86_64.zip，解压得到bin/protoc

```shell
vi ~/.bash_profile 
export PATH=/Users/demon/Desktop/own/protoc-3.18.0/bin:$PATH

source一下

安装go插件: 成功会在$GOBIN目录下有protoc-gen-go
go install google.golang.org/protobuf/cmd/protoc-gen-go
```



#### 编绎proto文件

```shell
protoc -I=$SRC_DIR --go_out=$DST_DIR $SRC_DIR/addressbook.proto

-I: import的路径
--go_out: 生成的go文件目录
最后是源pb文件
```



#### 举个例子

##### 新建文件 addressbook.proto

```protobuf
syntax = "proto3";
package tutorial;

option go_package = "core/common/serial"; // 会自动生成目录

message Person {
  string name = 1;
  int32 id = 2;
  string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    string number = 1;
    PhoneType type = 2;
  }

  repeated PhoneNumber phones = 4;
  // google.protobuf.Timestamp last_updated = 5;
}

message AddressBook {
  repeated Person people = 1;
}
```

```shell
当前目录结构:
demon@demondeMacBook-Pro protobuftest $ tree
.
└── core
    └── common
        └── serial
            └── addressbook.proto
```

##### 跑编译

```shell
protoc -I=. --go_out=. ./core/common/serial/addressbook.proto
```

```shell
生成的文件:
demon@demondeMacBook-Pro protobuftest $ tree
.
└── core
    └── common
        └── serial
            ├── addressbook.pb.go
            └── addressbook.proto
```

会根据go_package在 go_out指定的目录下创建对应的目录

注意, 在生成的addressbook.pb.go文件里都会有三个字段:

```protobuf
type Person struct {
	state         protoimpl.MessageState
	sizeCache     protoimpl.SizeCache
	unknownFields protoimpl.UnknownFields
 ...
}
```



#### package和option go_package

.proto文件里 option go_package学名叫Go import path，与 package没有啥联系, package只用于名空间。



#### 消息go对应

```protobuf
单消息:
	message Foo {}
对应生成Foo结构体，指针*Foo 实现proto.Message接口


嵌套消息:
message Foo {
  message Bar {
  }
}
对应生成Foo and Foo_Bar
```



##### 命名字段

官方建议.proto消息体内字段用小写+下线线命名，比如：

```protobuf
message InboundHandlerConfig {
  string tag = 1;
  int32 receiver_settings = 2;
  int32 proxy_settings = 3;
}
```

对应生成的go结构体会是首字母大写+驼峰，比如:

```protobuf
foo_bar_baz -> FooBarBaz
_my_field_name_2 -> XMyFieldName_2
```

##### 标量字段

```protobuf
int32 foo = 1 对应生成的go->  Foo int32 且 带有GetFoo()方法

```

##### 消息字段

```protobuf
message Bar {}

message Baz {
  Bar foo = 1;
}

对应生成的go结构体:
type Baz struct {
  Foo *Bar
}

```

##### 重复字段

```protobuf
message Bar {}

message Baz {
  repeated Bar foo = 1;
}

对应生成的go结构体:
type Baz struct {
  Foo  []*Bar
}
```

##### map字段

```protobuf
message Bar {}

message Baz {
  map<string, Bar> foo = 1;
}
对应生成的go结构体:
type Baz struct {
  Foo map[string]*Bar
}
```

##### oneof字段

```protobuf
package account;
message Profile {
  oneof avatar {
    string image_url = 1;
    bytes image_data = 2;
  }
}
```

生成规则:

字段名以isMessageName_MyField命名，这里是isProfile_Avatar，是个interface。

为每个oneof生成一个结构体, 每结构体者实现isProfile_Avatar接口。

```go
type Profile struct {
  Avatar isProfile_Avatar `protobuf_oneof:"avatar"`
}

type Profile_ImageUrl struct {
  ImageUrl string
}
type Profile_ImageData struct {
  ImageData []byte
}
```

设置值就这样:

```go
p1 := &account.Profile{
  Avatar: &account.Profile_ImageUrl{"http://example.com/image.png"},
}

// imageData is []byte
imageData := getImageData()
p2 := &account.Profile{
  Avatar: &account.Profile_ImageData{imageData},
}
```

判断类型就这样：

```go
switch x := m.Avatar.(type) {
case *account.Profile_ImageUrl:
  // Load profile image based on URL
  // using x.ImageUrl
case *account.Profile_ImageData:
  // Load profile image based on bytes
   // using x.ImageData
case nil:
  // The field is not set.
default:
  return fmt.Errorf("Profile.Avatar has unexpected type %T", x)
}
```

细品一下吧。

##### 枚举字段

消息内定义的枚举:

```protobuf
message SearchRequest {
  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  Corpus corpus = 1;
  ...
}

对应生成的go类型:
type SearchRequest_Corpus int32

并生成一堆常量:
const (
  SearchRequest_UNIVERSAL SearchRequest_Corpus = 0
  SearchRequest_WEB       SearchRequest_Corpus = 1
  SearchRequest_IMAGES    SearchRequest_Corpus = 2
  SearchRequest_LOCAL     SearchRequest_Corpus = 3
  SearchRequest_NEWS      SearchRequest_Corpus = 4
  SearchRequest_PRODUCTS  SearchRequest_Corpus = 5
  SearchRequest_VIDEO     SearchRequest_Corpus = 6
)
```



包级枚举:

```protobuf
enum Foo {
  DEFAULT_BAR = 0;
  BAR_BELLS = 1;
  BAR_B_CUE = 2;
}

对应生成的go类型:
type Foo int32

生成的常量:
const (
  Foo_DEFAULT_BAR Foo = 0
  Foo_BAR_BELLS   Foo = 1
  Foo_BAR_B_CUE   Foo = 2
)
```

**从生成规则可以看出，不同的enum，同一范围内字段名称不能相同，不然会冲突编译不过。**

除此之外，还会生成两个map，名字到值，值到名字，不重要。







```protobuf
message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3;
}
```



#### 字段数字(Field Numbers)

1. 【1, 15】只会占用1个字节空间（存储数字+类型），频繁使用的字段应该被安排到这个区间上;
2. 【16, 2047】占2个字节;
3. 可安排的区间为【1, 2^29 -1 】;
4. 【19000, 19999】为系统保留区间，应用里要避开使用;



#### repeated、optional、required、enum

历史原因, repeated的编码并没那么高效，新代码可以加上这个来让编码更高效：

```protobuf
repeated int32 samples = 4 [packed=true];

```

谷歌内部的一些工程师认为, required的坏处大于好处，optional和repeated更通用，所以我们也老实的不用required吧



optional设置默认值, 如果没有设置, 那就是一个系统默认值

```protobuf
optional int32 result_per_page = 3 [default = 10];
```



enum可以定义在消息体内, 也可以在消息体外

```protobuf
message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3 [default = 10];
  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  optional Corpus corpus = 4 [default = UNIVERSAL];
}
```



#### 自定义字段类型

```protobuf
message SearchResponse {
  repeated Result result = 1;
}

message Result {
  required string url = 1;
  optional string title = 2;
  repeated string snippets = 3;
}
```



从其他文件import

```protobuf
import "myproject/other_protos.proto";
```

编码程序搜寻导入的目录： `-I`/`--proto_path`



