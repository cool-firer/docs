- [Reference Types](#reference-types)
- [calldata的坑](#calldata的坑)
  - [不能分配（Allocate）calldata 类型](#不能分配allocatecalldata-类型)
  - [必须要赋值给自己的古怪操作](#必须要赋值给自己的古怪操作)
- [Arrays](#arrays)
  - [内存数组 (Memory Array) 的致命限制](#内存数组-memory-array-的致命限制)
  - [bytes和string](#bytes和string)
  - [bytes.concat 和 string.concat](#bytesconcat-和-stringconcat)
  - [数组字面量Array Literals](#数组字面量array-literals)
  - [数组元素的悬空引用](#数组元素的悬空引用)
- [Array Slices数组切片](#array-slices数组切片)
- [Structs](#structs)
- [Mapping类型](#mapping类型)
  - [遍历](#遍历)




# Reference Types

引用类型包括：**结构体 (Structs)**、**数组 (Arrays)** 和 **映射 (Mappings)**。

如果你使用引用类型，**必须显式指定该类型存储的数据区域**：

* **`storage`**：存储状态变量的位置，其生命周期与合约的生命周期一致。最贵，数据永久保存。
- **`memory`**：其生命周期仅限于一次外部函数调用。较便宜，函数跑完数据就没了。

- **`calldata`**：包含函数参数的特殊数据位置。最便宜，数据不可修改。

**涉及数据位置变更**的赋值或类型转换总是会触发自动拷贝（Copy）操作; 而**在同一数据位置内**的赋值，仅在存储（`storage`）类型的某些情况下会发生拷贝。

**规则 A：跨位置赋值 = 全力拷贝（消耗大量 Gas）**

当你把数据从一个地方搬到另一个地方，Solidity 会在目标位置创建一个**全新的完整副本**。

- `storage` $\rightarrow$ `memory`：拷贝一份。

- `memory` $\rightarrow$ `storage`：拷贝一份。

 **规则 B：`storage` 内部的赋值 = 创建指针**

当你把一个状态变量（已经在 `storage` 里）赋值给函数内的一个 `storage` 局部变量时，它**不会**产生副本，而是产生一个**指针**。

Solidity

```
uint[] public myArray;

function test() public {
    uint[] storage ptr = myArray; // ptr 只是 myArray 的一个“外号”
    ptr.push(1); // 修改 ptr，实际上直接修改了 myArray！
}
```

**规则 C：`memory` 内部的赋值 = 始终创建引用**

对于 `memory` 对象，赋值操作从不拷贝数据。两个变量会指向内存中的同一个位置。



# calldata的坑

## 不能分配（Allocate）calldata 类型

- **`memory` 可以“分配”**：你可以凭空在内存里造一个新数组。
  
  Solidity
  
  ```
  // ✅ 运行正常：在内存里申请了一块新空间，存了 10 个 uint
  uint[] memory a = new uint[](10); 
  ```

- **`calldata` “不能分配”**：你**不能**在代码里凭空造一个 `calldata` 数组。

```solidity
// ❌ 编译报错！你不能用 new 创建 calldata
uint[] calldata b = new uint[](10); 
```

但可以声明，并且在使用之前必须要赋值：

```solidity
function process(uint[] calldata inputs) external {
    // 这里的 inputs 就是 calldata 类型
    // 虽然你不能 new 一个 calldata，但你可以把 inputs 传给另一个函数
    someOtherFunction(inputs); 
}

// 在使用前没有赋值
function myFunc(uint[] calldata inputs) external pure {
    uint[] calldata temp; // 声明了
    uint x = temp[0];     // ❌ 报错！temp 指向哪里？编译器不知道，这会导致执行崩溃。
}

// 使用前赋值
function myFunc(uint[] calldata inputs) external pure {
    uint[] calldata temp; 
    temp = inputs;        // ✅ 赋值：现在 temp 指向了入参 inputs 的位置
    uint x = temp[0];     // 正常使用
}


// 还可用在返回值上。这个函数的意思是：我从传入的一堆数据里，挑一段原封不动地传回去
function getSlice(uint[] calldata inputs) external pure returns (uint[] calldata) {
    // 你必须给返回变量赋值，通常是赋值为输入的一部分或全部
    return inputs[1:3]; 
}
```

## 必须要赋值给自己的古怪操作

```solidity
// 文档中提到的“控制流复杂导致编译器检测不到初始化”，指的是下面这种场景：

function complexLogic(bool flag, uint[] calldata data) external pure {
    uint[] calldata target;

    if (flag) {
        target = data;
    } else {
        // 假设这里漏写了 target = ...
    }

    // 编译器会在这里报警：你可能没给 target 赋值！
    // 即使你确信逻辑上没问题，编译器也可能因为不够聪明而卡住。
    
    // 骚操作：
    target = target; // 👈 这一行让编译器觉得“哦，它初始化过了”，从而闭嘴。
}
```

<br />

# Arrays

数组可以具有编译时固定的长度，也可以是动态长度。

元素类型为 `T`、固定长度为 `k` 的数组类型写作 `T[k]`；动态长度的数组写作 `T[]`。

例如，一个由 5 个 `uint` 动态数组组成的数组写作 `uint[][5]`。这种记法与某些其他语言相比是**反向的**。

数组元素可以是任何类型，包括映射（mapping）或结构体（struct）。

越界访问数组会导致断言失败（Panic）。方法 `.push()` 和 `.push(value)` 可用于在动态数组末尾添加新元素，其中 `.push()` 会添加一个零初始化的元素并返回其引用。



## 内存数组 (Memory Array) 的致命限制

文档最后提到的 `Note` 非常关键：

- **Storage 数组**：可以用 `.push()`，长度会自动变长，就像 Python 的 List。

- **Memory 数组**：**长度是死的！**

```solidity
function test() public pure {
    // ✅ 在内存里开辟一个长度为 3 的空间
    uint[] memory a = new uint[](3);

    // ❌ 报错！内存数组没有 push 方法
    // a.push(4); 

    // ❌ 报错！也不能直接修改长度属性
    // a.length = 4; 
}
```

**为什么？** 因为内存管理为了效率，是连续分配的。如果你想让数组变长，可能会覆盖掉内存中紧随其后的其他变量。所以 Solidity 规定：内存数组一旦 `new` 出来，大小就固定了。



## bytes和string

1、`bytes` 和 `string` 类型的变量是特殊的数组。`bytes` 类似于 `bytes1[]`，但它在 `calldata` 和 `memory` 中被紧密打包（Packed）。`string` 等同于 `bytes`，但不允许使用长度（length）或索引（index）访问。

2、应该优先使用 `bytes` 而不是 `bytes1[]`，因为前者更便宜。在 `memory` 中使用 `bytes1[]` 会在元素之间添加 31 字节的填充（Padding）。

3、`string` 本质上是一个被禁用了 `.length` 和索引访问的 `bytes`。如果你想数一个 `string` 有多长，你得先把它强转成 `bytes`：

```solidity
string memory s = "hello";
// uint len = s.length; // ❌ 报错
uint len = bytes(s).length; // ✅ 成功
```



## bytes.concat 和 string.concat

* `string.concat` 连接任意数量的 `string` 值。该函数返回一个包含所有参数内容的单个 `string memory` 数组，且不包含填充（Padding）。如果你想使用其他无法隐式转换为 `string` 的参数类型，需要先将它们转换为 `string`。

* 同理，`bytes.concat` 函数可以连接任意数量的 `bytes` 或 `bytes1` ... `bytes32` 值。该函数返回一个包含参数内容的单个 `bytes memory` 数组，不含填充。如果你想使用字符串参数或其他无法隐式转换为 `bytes` 的类型，需要先将其转换为 `bytes` 或 `bytes1...bytes32`。

**A. 告别 "ABI 拼接时代"**

在老版本里，我们要拼两个字符串 `a` 和 `b`，通常写成： `string(abi.encodePacked(a, b))` 虽然好用，但 `abi.encodePacked` 的本意是用来计算哈希的，用来拼字符串总有种“拿扳手当锤子使”的感觉。`string.concat` 专门为拼接设计，语义更清晰，编译器优化也更好。

 **B. 无填充 (Without Padding)**

这是最关键的 Gas 优化点。

- 如果使用 `abi.encode`，它会强制每个参数占用 32 字节（不够的补 0）。

- `concat` 函数会计算所有参数的精确长度，开辟一块刚好够大的内存空间，然后把数据一个接一个地紧密排布。



## 数组字面量Array Literals

数组字面量是包含在一个或多个表达式中、用方括号括起来并以逗号分隔的列表（例如 `[1, a, f(3)]`）。数组字面量的类型确定规则如下：

1. 它始终是一个**固定长度的内存数组 (Fixed-size memory array)**，其长度等于表达式的数量。

2. 数组的**基类类型 (Base type)** 取决于列表中**第一个表达式**的类型，且要求所有其他表达式都能隐式转换为该类型。如果无法转换，则会引发类型错误。

3. 仅仅存在一个所有元素都能转换到的类型是不够的，必须**其中一个元素本身就是该类型**。

**第一个元素决定一切**

这是最容易翻车的地方。

- `[1, 2, 256]` $\rightarrow$ 报错！因为第一个元素 `1` 被推断为 `uint8`，但 `256` 放不下。

- `[uint(1), 2, 256]` $\rightarrow$ 成功！因为第一个元素变成了 `uint256`，后面的 `2` 和 `256` 都能塞进去。

**固定长度内存数组不能赋值给动态长度内存数组**。以下代码无法通过编译：

```solidity
uint[] memory x = [uint(1), 3, 4]; // 错误：uint[3] 无法转为 uint[]
```

**“固定”与“动态”的鸿沟**

这是一个非常反直觉的点。在 JavaScript 里，`let x = [1,2,3]` 以后可以随便 `push`。但在 Solidity 里：

- `[1, 2, 3]` 的类型是 `uint8[3]`。它的长度被**写死**在了类型里。

- `uint[]` 是动态数组。
  在底层，`uint[3]` 和 `uint[]` 在内存中的布局是完全不同的。`uint[3]` 只有数据，而 `uint[]` 的开头还要存一个长度信息。**这就是为什么你不能直接把 `[1, 2, 3]` 赋值给 `uint[]`。**

<br />

## 数组元素的悬空引用

例子1：

```solidity
contract C {
    uint[][] s;

    function f() public {
        // Stores a pointer to the last array element of s.
        uint[] storage ptr = s[s.length - 1];
        // Removes the last array element of s.
        s.pop();
        // Writes to the array element that is no longer within the array.
        ptr.push(0x42);
        // Adding a new element to ``s`` now will not add an empty array, but
        // will result in an array of length 1 with ``0x42`` as element.
        s.push();
        assert(s[s.length - 1][0] == 0x42);
    }
}
```

**深度分析：为什么“删掉”了还能写进去？**

这是理解 EVM 存储布局的关键。

- **存储是“虚拟”的**：EVM 的 `storage` 并不是物理硬盘上的文件，而是一个巨大的键值对空间（$2^{256}$ 个插槽）。

- **Pop 的本质**：`pop()` 只是修改了数组的 `length` 计数器，并把对应的槽位抹成 0。

- **指针的本质**：当你写 `uint[] storage ptr = s[last]` 时，`ptr` 实际上已经计算出了那个元素在存储中的**绝对物理地址**（Slot 编号）。

**发生了什么鬼故事？**

1. 执行 `pop()` 时，该地址的内容确实被清空了。

2. 但是，局部变量 `ptr` 依然记着那个物理地址。

3. 当你通过 `ptr.push(0x42)` 写入时，你直接绕过了数组 `s` 的长度检查，强行修改了那个物理地址的内容。

4. **最关键的一步**：当你再次 `s.push()` 时，Solidity 为了省 Gas，认为“反正我之前删过，那里应该是空的（0）”，所以它**不会重新初始化**那个槽位。

5. 结果：你之前的“非法写入”被当成了“合法新数据”。

---

例子2：

```solidity
contract C {
    uint[] s;
    uint[] t;
    constructor() {
        // Push some initial values to the storage arrays.
        s.push(0x07);
        t.push(0x03);
    }

    function g() internal returns (uint[] storage) {
        s.pop();
        return t;
    }

    function f() public returns (uint[] memory) {
        // The following will first evaluate ``s.push()`` to a reference to a new element
        // at index 1. Afterwards, the call to ``g`` pops this new element, resulting in
        // the left-most tuple element to become a dangling reference. The assignment still
        // takes place and will write outside the data area of ``s``.
        (s.push(), g()[0]) = (0x42, 0x17);
        // A subsequent push to ``s`` will reveal the value written by the previous
        // statement, i.e. the last element of ``s`` at the end of this function will have
        // the value ``0x42``.
        s.push();
        return s;
    }
}
```

**深度分析：元组赋值的“拆解”过程**

在 Solidity 中，`(a, b) = (v1, v2)` 并不是瞬间完成的。编译器必须先搞清楚 `a` 应该指向哪，`b` 应该指向哪，然后才开始填值。

**代码中的“连环追尾”事件：**

1. **左侧第一项 `s.push()`**：数组 `s` 长度从 1 变 2。编译器拿到了一个指向 `s[1]` 地址的“入场券”。

2. **左侧第二项 `g()[0]`**：
   
   - 调用 `g()`。
   
   - **关键点**：`g` 里面执行了 `s.pop()`。数组 `s` 长度从 2 缩回 1。刚才那个 `s[1]` 地址在逻辑上已经“作废”了。
   
   - `g` 返回了 `t` 的引用，所以这一项最后指向 `t[0]`。

3. **开始赋值**：
   
   - 将 `0x42` 填入 `s[1]` 的地址（由于 `s.push()` 已经给出了地址，即使它被 pop 了，值还是写进去了）。
   
   - 将 `0x17` 填入 `t[0]`。

**结果**：你通过元组赋值，在 `s` 的长度之外成功“走私”了一个数据 `0x42`。

在区块链开发中，**越酷、越简洁的代码往往越危险**。

- **危险写法（追求简洁）**：
  
  ```solidity
  (s.push(), g()[0]) = (0x42, 0x17); // 逻辑耦合严重，求值顺序依赖性强
  ```

- **安全写法（老老实实）**：
  
  将复杂操作拆分成独立的行，可以确保每一行的状态变更在下一行开始前已经完全确定，彻底消除由于求值顺序导致的“副作用”。

```solidity
s.push() = 0x42;
uint[] storage target = g();
target[0] = 0x17;
```

<br />

# Array Slices数组切片

数组切片（array slices）是对数组中一段连续区域的“视图（view）”。

它的写法是：x[start:end]

* 其中 `start` 和 `end` 都是表达式，结果必须是 `uint256` 类型（或者可以隐式转换成 `uint256`）。

* 切片的第一个元素是：x[start]
- 最后一个元素是：x[end−1]

如果：

- `start > end`
- 或 `end > 数组长度`

则会抛出异常。

**目前数组切片只支持 `calldata` 数组。在 ABI 解码函数参数中的二级数据时非常有用：

```solidity
    function forward(bytes calldata payload) external {
        bytes4 sig = bytes4(payload[:4]);
        // Due to truncating behavior, bytes4(payload) performs identically.
        // bytes4 sig = bytes4(payload);
        if (sig == bytes4(keccak256("setOwner(address)"))) {
            address owner = abi.decode(payload[4:], (address));
            require(owner != address(0), "Address of owner cannot be zero.");
        }
        (bool status,) = client.delegatecall(payload);
        require(status, "Forwarded call failed.");
    }
```

<br />

# Structs

官方的这个例子特别好，涵盖了structs的一些坑点。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.2 <0.9.0;

// 定义一个包含两个字段的新类型
// 在合约外定义 struct，意味着它可以被多个合约共享
struct Funder {
    address addr;
    uint amount;
}

contract CrowdFunding {
    // struct 也可以定义在合约内部
    // 这样它只在当前合约和派生合约中可见
    struct Campaign {
        address payable beneficiary;
        uint fundingGoal;
        uint numFunders;
        uint amount;

        mapping(uint => Funder) funders;
    }
    mapping(uint => Campaign) campaigns;

    function newCampaign(
        address payable beneficiary,
        uint goal
    )
        public
        returns (uint campaignID)
    {
        campaignID = numCampaigns++;

        // 不能这样写：
        // campaigns[campaignID] = Campaign(beneficiary, goal, 0, 0)

        // 因为右边会创建一个 memory struct
        // 而这个 struct 里面包含 mapping

        Campaign storage c = campaigns[campaignID];

        c.beneficiary = beneficiary;
        c.fundingGoal = goal;
    }

    function contribute(uint campaignID) public payable {

        Campaign storage c = campaigns[campaignID];

        // 创建一个临时 memory struct
        // 用给定值初始化
        // 然后复制到 storage

        // 也可以写成：
        // Funder(msg.sender, msg.value)

        c.funders[c.numFunders++] =
            Funder({
                addr: msg.sender,
                amount: msg.value
            });

        c.amount += msg.value;
    }

    function checkGoalReached(uint campaignID)
        public
        returns (bool reached)
    {
        Campaign storage c = campaigns[campaignID];

        if (c.amount < c.fundingGoal)
            return false;

        uint amount = c.amount;

        c.amount = 0;

        (bool success, ) =
            c.beneficiary.call{value: amount}("");

        return success;
    }
}
```

* struct 不能直接包含自己类型的成员，但struct 可以作为 mapping 的 value type，或者包含动态数组。
  
  ```solidity
  struct Node {
      Node next; // ❌ 不允许
  }
  
  
  struct Node {
      Node[] children; // ✅
  }
  数组本身只保存：length + pointer
  固定大小。真正元素在别处。
  ```

* 为什么不能这样写：campaigns[campaignID] = Campaign(beneficiary, goal, 0, 0)
  
  ```solidity
  会创建：memory struct
  而Campaign内部包含：mapping(uint => Funder)
  但问题来了：mapping 不能存在于 memory，这是 Solidity 的底层限制。
  mapping只能存在 storage。
  ```

* 隐藏一个安全点：Checks-Effects-Interactions
  
  先：1、修改状态  2、再 external call。避免 reentrancy。

<br />

# Mapping类型

mapping 只能存在于storage。

```solidity
mapping(KeyType => ValueType)
```

1、KeyType 可以是：

- 任意内置 value type
- `bytes`
- `string`
- contract type
- enum type

但不能是：

- mapping
- struct
- array
- 其他复杂类型

2、ValueType 可以是：

- 任意类型
- 包括：
  - mapping
  - array
  - struct

3、mapping 不存储 key，只使用keccak256(key) 来查找 value。

4、不能作为public/external 函数参数、public/external 返回值。internal/private就可以。

```solidity
// ❌ 非法。
// 因为public/external 函数需要 ABI。
// uint address bytes array struct 都可以 ABI encode。 但 mapping 不行。
function f(mapping(address => uint) storage m) public 


// ✅ 合法。
// 因为这只是EVM的JUMP，不涉及ABI encode/decode
function f(mapping(address=>uint) storage m) internal

```

## 遍历

原生的mapping不支持遍历。只能自己实现一套数据结构。

原理就是存储一下所有的key，官方文档的例子就可以。


