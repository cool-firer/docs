李晓均编著 2019出版



# 第2章 Go语言编码基础

## 2.1 数据类型

浮点数分为float32、float64两种，表示的最大值从math包获取：

math.MaxFloat32 大约是3.4e38、math.MaxFloat64 大约是1.8e308。

float32可心提供小数点后6位的精度，float64是后15位精度。

<br />

Go的引用类型有：指针、切片、字典、通道。



## 2.2 变量

在Go语言中，如果导入的包未使用，就不能通过编译。如果只想调用包中的init函数，可以用"_"来作为这个包的名字：

```go
// 可以编译通过
package main
import (
	_ "fmt"
	"time"
)

func main() {
	_ = time.Now
}
```

<br />

当一个变量被var声明之后，如果没指定值，就初始化为零值。

| 类型                                          | 零值  |
| :-------------------------------------------- | ----- |
| integer                                       | 0     |
| float                                         | 0.0   |
| bool                                          | false |
| string                                        | ""    |
| pointer interface function map slice channedl | nil   |



## 2.5 字符串

双引号：转义字符会被替换。

反引号：不会替换。

```go
str := "Hello World!\n Hello Gopher!\n"
// 输出
Hello World!
Hello Gopher!

str := `Hello World!\n Hello Gopher!\n`
// 输出
Hello World!\n Hello Gopher!\n
```

len(str)获取的是字节长度，注意与rune的区别。

字符串拼接：

1. 直接使用+，或 +=

   ```go
   str := "Beginning of the string " + "second"
   s := "hel"
   s += "world"
   ```

   每次+都会产生一个新的字符串，如果加的多，会产生很多临时字符串，性能较差。

2. fmt.Sprintf()

   ```go
   fmt.Sprintf("%d:%s", 2018, "年)
   ```

   内部使用[]byte实现，不会像+产生临时字符串，内部有逻辑复杂判断，还用到了接口，性能一般。

3. strings.Join()

   ```go
   strings.Join([]string{"hello", "world"}, ","}
   ```

   Join会先根据字符串数据长度，申请一个对应大小的内存再填入。

4. bytes.Buffer

   ```go
   var buffer bytes.Buffer
   buffer.WriteString("hello")
   buffer.WriteString("world")
   fmt.Println(buffer.String())
   ```

   比较理想，当成可变字符使用，对内存的增长有优化。

5. Strings.Builder

   ```go
   var bl strings.Builder
   b1.WriteString("ABC")
   fmt.Println(b1.String())
   ```

   跟bytes.Buffer一样，性能相差无几。非线程安全。



# 第4章 代码结构化与项目管理

## 4.1 包的概念

每个目录下可以有多个.go文件，这些文件只能属于同一个包。

init()函数不能被其他函数调用。

import 使用目录名作为包的路径，而不是包名，如 import "math/big"，导入的是src/math/big目录下的包。程序代码中使用的big.Int，bit指的就是包名，只是目录名与包名一致而已。

gomodules是go1.11才有的。

GO111MODULE=off 从GOPATH和vendor文件夹找包；

GO111MODULE=on 忽略GOPATH和vendor，只根据go.mod下载依赖；

GO111MODULE=auto 目录下有go.mod就开启。

gomodule下载的包是放在$GOPATH/pkg/mod/下。



# 第6章 type关键字

type关键字在原有类型基础上构造出一个新类型，不会拥有原类型的方法。

```go
type A struct { Face int }
func (a A) f() {}

type Aa A
func main() {
  var s A = A{ Face: 1}
  s.f()
  
  var sa Aa = Aa{ Face: 2 }
  sa.f() // 这行会报错 sa.f undefined (type Aa has no field or method f)
```

<br />

type关键字类型别名，则拥有原类型的方法。

```go
type A struct { Face int }
func (a A) f() {}

type Aa = A
func main() {
  var s A = A{ Face: 1}
  s.f()
  
  var sa Aa = Aa{ Face: 2 }
  sa.f() // 正常运行

```



# 第8章函数

函数签名由形参、返回值以及它们的类型构成。

函数重载是指可以定义同名的函数，只要形参或返回值不同。Go不允许重载。

new(S)：为S类型的变量分配内存，并初始化，返回变量地址，是一个指针。

变参函数：

```go
func Greeting(who ...string) {
  for k, v := range who {
    fmt.Println(, v)
  }
}

s := []string{"Jan", "KK"}
// 这里要打散传入
Greeting(s...)
```



# 第9章 结构体和接口

new(T)：为T类型的变量分配内存，并初始化，返回变量地址，是一个指针。

var t T：也给t分配内存，并零值化内存，这t是类型T。

new(Type)和&Type{}是等价的。



结构构可以引用自身指针来定义。

```go
type Element struct {
	next, prev *Element
}
```

<br />

## 9.1.4 嵌入与聚合

包含匿名字段叫嵌入或内联。包含类型和名称叫聚合。

```go
type Human struct {}

type Person1 struct {
  Human // 内嵌
}

type Person2 struct {
  *Human // 内嵌
}

type Person3 struct {
  hu Human // 聚合
}
```

## 9.2.3 类型断言

Value, ok := varI.(T)，varI必须是一个接口变量，否则报错。



# 第10章 方法

## 10.2 指针方法与值方法

结构体可以调用指针方法+值方法。

```go
package main

type T struct { Name string }

// 把它想成：M1(T t)
func (t T) M1() {
	t.Name = "name1"
}

// 把它想成：M2(T& t)
func (t *T) M2() {
	t.Name = "name2"
}

func main() {
  t1 := T{"t1"}           t2 := &T{"t2"}
  // t1                   // t2
	fmt.Println(t1.Name)
  // 相当于 M1(t1)         // 相当于 M1(*t2) 先圈起来, 再通过拷贝赋值给M1     
  t1.M1()                 t2.M1()
  
  // t1                   // t2
	fmt.Println(t1.Name)
  // 相当于 M2(t1)         // 相当于 M2(*t2) 先圈起来, 再引用给M2
  t1.M2()                 t2.M2()
  
  // name2                // name2
	fmt.Println(t1.Name)
}
```

T可以调用 func(t T)、func(t *T)

*T可调用 func(t T)、func(t *T)

但区别是：值变量拥有值方法集，指针变量拥有值方法集 + 指针方法集。这里看不出这句话的区别，得跟接口结合看。

<br />

```go
type T struct { Name string }
type Inf interface {
  M1()
  M2()
}

// 看这这句, 要下意识反应
// 值、指针实现了M1方法
func (t T) M1() { t.Name = "name1" }

// 看到这句, 就要反应，只有指针实现了M2方法
func (t *T) M2() { t.Name = "name2" }

func main() {
  var t1 T = T{"t1"}
  t1.M1()
  t2.M2()
  
  // 这句报错了, T并没有实现接口Inf
  // 改成 var t2 inf = &t1就Ok
  var t2 Inf = t1
  t2.M1()
  t2.M2()
}
```

上例规则：指针方法实现接口，只要指向类型的指针才能实现接口；

值方法实现接口，类型值和指针都实现接口；



# 第14章 系统标准库

## 14.1.1 反射

一个接口类型的变量存储了两个信息：(实际变量的值value，实际变量的类型type)。

即 var a interface{} = <value, type>。

## 14.2 unsafe包

这个包只提供两个类型，三个函数：

```go
type ArbitraryType int
type Pointer *ArbitraryType // Pointer是个指针

func AlignOf(x ArbitraryType) uintptr // 返回变量对齐字节数量
func Offsetof(x ArbitraryType) uintptr  // 偏移量
func Sizeof(x ArbitraryType) uintptr // 变量在内存中占用的字节数
```

unsafe.Point = void *，万能指针；uintptr是内置类型，也是个指针。

unsafe.Point与uintptr区别：

1. Point用于将不同类型转换成指针，不可以进行指针运算；
2. uintptr可用于指针运算；
3. Point可以与普通指针相互转换，包括uintptr；

```go
import ( "fmt" "unsafe" )
type V struct {
  b byte
  i int32
  j int64
}

var v *V = new(V)
unsafe.Sizeof(*v) // 结构体占用的字节: 16

// 拿到i的地址: 
var i *int32 = (*int32)(
unsafe.Pointer( 
  uintptr(unsafe.Pointer(v)) 
  + 
  uintptr(unsafe.Sizeof(byte(0)) * 4) 
)
)
```

一句话，Pointer作为中转，在struct中，对齐值是按成员最大来。

## 14.7 文件操作与I/O

三种读写文件方式：

1. os.File

   ```go
   import ( "os" )
   fi, err := os.Open("./1.txt")
   defer fi.Close()
   
   buf := make([]byte, 1024)
   for {
     n, err := fi.Read(buf)
     if err != nil && err != io.EOF {
       panic(err)
     }
     if 0 == n {
       break
     }
   }
   ```

2. iotuil包

   ```go
   import ( "io/ioutil" )
   fi, err := os.Open("./1.txt")
   ioutil.ReadAll(fi)
   ```

3. bufio包

   ```go
   fi, err := os.Open("1.txt")
   r := bufio.newReader(fi)
   buf := make([]byte, 1024)
   for {
     n, err := r.Read(buf)
     if err != nil && err != io.EOF {
       panic(err)
     }
     if 0 == n {
       break
     }
   }
   ```

   

# 第16章数据格式与存储

go struct --------> 编码(encode、Marshal)     -----------> bytes，用来写入文件、传输网络

go struct <-------- 解码(decode、Unmarshal) ------------ bytes

