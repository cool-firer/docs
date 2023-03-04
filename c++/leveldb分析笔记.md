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

## LEVELDB_EXPORT

在class、struct前面都有一个`LEVELDB_EXPORT`，如：

```c++
struct LEVELDB_EXPORT Options { ... }

class LEVELDB_EXPORT Snapshot { ... }
```

[作用说明](https://stackoverflow.com/questions/70814746/what-does-it-mean-class-leveldb-export-status-in-leveldb): 一些链接器不导出所有的标识符，所以需要这样导出。（没看懂）





## sizeof和alignof区别

sizeof：对象、类占用的内存大小 字节

alignof：对齐的大小 字节

```c++
#include <iostream>
using namespace std;

struct A {
	int avg;
	int avg2;
	double c;
	A(int a, int b) {}
};

struct B {
	int avg;
	int avg2;
	char c;
};

int main() {
	cout<<"sizeof(A):"<<sizeof(A)<<endl;		// 16
	cout<<"alignof(A):"<<alignof(A)<<endl;	// 8

	cout<<"sizeof(B):"<<sizeof(B)<<endl; 		// 12
	cout<<"alignof(B):"<<alignof(B)<<endl;	// 4
}
```



std::aligned_storage

定义在头文件<type_traits> https://en.cppreference.com/w/cpp/types/aligned_storage

```c++
template< std::size_t Len, std::size_t Align = /*default-alignment*/ >
```



# struct ::flock

```c++
int LockOrUnlock(int fd, bool lock) {
  errno = 0;
  struct ::flock file_lock_info;
  std::memset(&file_lock_info, 0, sizeof(file_lock_info));
  file_lock_info.l_type = (lock ? F_WRLCK : F_UNLCK);
  file_lock_info.l_whence = SEEK_SET;
  file_lock_info.l_start = 0;
  file_lock_info.l_len = 0;  // Lock/unlock entire file.
  return ::fcntl(fd, F_SETLK, &file_lock_info);
}
```



# std::string

clear(): 清空字符串。

resize(kHeader): 设置为长度为n个字符，多了截断，少了加空字符。

data(): 返回char* 指针。

size(): 返回字节长度。

push_back(): 往后追加一个字符, 且增加长度。

append(): 追加一个字符串





# 报错集锦

does not refer to a value

alignof(struct A) 语句报错，编译时加上std11即可：g++ -o sizeof sizeof.cpp --std=c++11

<br />

