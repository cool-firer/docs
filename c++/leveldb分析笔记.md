# 编译&使用

git地址：https://github.com/google/leveldb

```shell
git clone --recurse-submodules https://github.com/google/leveldb.git
```

<br />

## 编译出库

```shell
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release .. && cmake --build .
```

在build目录出现libleveldb.a静态库文件。

<br />

## 写个test.cpp

```c++
#include <iostream>
#include <leveldb/db.h>
#include <string>

using namespace std;

int main() {
  leveldb::DB* db;
  leveldb::Options opt;
  leveldb::Status status;
  // 写业务
  // ...
  return 0;
}
```

<br />

## 编译运行

```shell
g++ test.cpp -o testcpp -I/Users/luke/Desktop/test/leveldb/include -L/Users/luke/Desktop/test/leveldb/build -lleveldb -std=c++11

-I:头文件目录
-L:静态库目录
-l:静态库名字
```





# 源码记录

LEVELDB_EXPORT

在class、struct前面都有一个`LEVELDB_EXPORT`，如：

```c++
struct LEVELDB_EXPORT Options { ... }

class LEVELDB_EXPORT Snapshot { ... }
```

[作用说明](https://stackoverflow.com/questions/70814746/what-does-it-mean-class-leveldb-export-status-in-leveldb): 一些链接器不导出所有的标识符，所以需要这样导出。（没看懂）







