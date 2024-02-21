# 简介

作者：胡思乱语

内容以0.8.20文档为例：https://docs.soliditylang.org/zh/v0.8.20/contracts.html#abstract-contract

记录一些常见的点。

# 合约

## 继承篇(inheritance)

### 函数重载(override)

1、virtual：表明函数可以被子合约重载，子合约重载时必须用override修饰。

2、重载时，要更严格。

* 可见性external 由 public重载；

* 可变性，nopayable 由 view、pure重载；

  view 由 pure重载；

```javascript
contract Base {
  function foo() virtual external view {}
}
contract Inherited is Base {
  // ok
  function foo() override external/public pure {}
  // ok
  function foo() override external/public view {}
  
  // 报错 overrideing function changes state mutability from 'view' to 'nonpayable'
  // function foo() override external {}
}
```

### 多继承规则

与python的C3线性化相似，但有不同之处。

#### Solidity继承规则

> 您必须按照从 “最接近的基类”（most base-like）到 “最远的继承”（most derived）的顺序来指定所有的基类。 注意，这个顺序与Python中使用的顺序相反。

继承先写最远的基类，再写次远的基类。

```javascript
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.0 <0.9.0;

contract X {}
contract A is X {}
// 这段代码不会编译
contract C is A, X {}

// 这段就正常
contract C is X, A {}
```

#### c3线性化, 即MRO

引用被Python官网收录的文章：https://www.python.org/download/releases/2.3/mro/

先规定以下术语：

* C(B~1~B~2~...B~N~)，代表C继承自B~1~、B~2~、...B~N~。
* L[C(B~1~B~2~...B~N~)] = CC~1~C~2~...C~N~ = M，代表线性化结果。
* head[M] = C，代表第一个元素。
* tail[M] = C~1~...C~N~，代表剩余元素。
* C + (C~1~C~2~...C~N~) = CC~1~C~2~...C~N~，+号的意义。

总的公式：

L[C(B~1~B~2~...B~N~)] = C + merge(L[B~1~],  ... , L[B~N~],  B~1~B~2~...B~N~)

即线性化结果 = 类C本身 + merge(各自父类的线性结果, 序列B~1~B~2~...B~N~)。

merge过程：

1. 取head = L[B~1~]，如果head 在tail[B~2~]、... tail[B~N~]，都找不到，则加在结果M里，并在所有L[B~X~]中删除head。
2. 如果head 在tail[B~2~]、... tail[B~N~]，中其中一个找到了，则取下一个L[B~2~]的head，继续从1开始。

例：

```python
                         6
                         ---
Level 3                 | O |                  (more general)
                      /  ---  \
                     /    |    \                      |
                    /     |     \                     |
                   /      |      \                    |
                  ---    ---    ---                   |
Level 2        3 | D | 4| E |  | F | 5                |
                  ---    ---    ---                   |
                   \  \ _ /       |                   |
                    \    / \ _    |                   |
                     \  /      \  |                   |
                      ---      ---                    |
Level 1            1 | B |    | C | 2                 |
                      ---      ---                    |
                        \      /                      |
                         \    /                      \ /
                           ---
Level 0                 0 | A |                (more specialized)
                           ---
```



L[O] = O

L[D] = DO

L[E] = EO

L[F] = FO

L[B] = L[B(DE)] 

​        = B + merge(L[D], L[E], DE)

​        = B + merge(DO, EO, DE)

​        = B + D + merge(O, EO, E)，运用规则，取D，在剩余list的tail中没有，加入D，删除D。

​        = B + D + E + merge(O, O)，取O，在剩余list的tail中有O，不能加O，取下一list的head E。注意这里取完了E，下一次还是从第一个list取head。                      

​        = B + D + E + O



## 抽象合约(abstract)

1、有方法没有被实现时，合约必须用`abstract`修饰。

```javascript
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.0 <0.9.0;

// 必须用abstract修饰, 不然编译会报错
abstract contract Feline {
  function utterance() public virtual returns (bytes32);
}

// 抽象合约用作基类
contract Cat is Feline {
  function utterance() public pure override returns (bytes32) { return "miaow"; }
}
```

2、合约A继承抽象合约B，没有重写抽象合约B里的函数，那A也得是`abstract`。

3、抽象合约不能实例化。

4、抽象合约可以有正常实现的函数，变量。

```javascript
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.0 <0.9.0;

abstract contract Feline {
  function utterance() public virtual returns (bytes32);
  function say() public pure returns (string memory) {
    return "hi";
  }

  uint256 a;
  function write() public {
    a = 3;
  }
}

function kk() {
  // 报错, cannot instantiate an abstract contract
  // Feline f = new Feline();
}
```



## 接口合约(interface)

相当于ABI，只用来描述函数，比抽象合约有更多限制。

1、只能继承接口合约。

2、不能有构造函数，不能声明状态变量，不能声明修饰器。

3、接口合约中声明的函数都是隐式`virtual`，重载它们时不需要`override`。

4、接口合约中可以定义类型。

```go
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.2 <0.9.0;

interface ParentA {
  // 定义类型
  struct Funder {
    address addr;
    uint amount;
  }
  function test() external returns (uint256);
}

interface ParentB {
  function test() external returns (uint256);
}

interface SubInterface is ParentA, ParentB {
  // 必须重新定义test，以便断言父类的含义是兼容的。
  function test() external override(ParentA, ParentB) returns (uint256);
}

function kk() pure {
  // 访问在接口合约中定义的类型
  ParentA.Funder memory f =  ParentA.Funder({addr: address(0x0), amount: 1});
}
```



# Events事件

```c#
event Transfer(address indexed from, address indexed to, uint256 value);
```

indexed与非indexed的区别？

首先事件是用日志Log来存储，每一条日志都包含两个部分：Topics和Data。

主题Topics长度最多为4，第一个是事件选择器、后面三个是indexed参数值。

剩余的非indexed参数存在Data中。

所以第一点不同就是，个数不一样，最多只能有三个indexed类型参数，非indexed个数不限。

第二点是，indexed参数可以被索引，客户端过滤。非indexed不行。



