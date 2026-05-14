- [Storage](#storage)
  - [**它是如何存在 Key-Value 数据库里的？**](#它是如何存在-key-value-数据库里的)
  - [**“256 位字（256-bit words）”是什么意思？**](#256-位字256-bit-words是什么意思)
  - [**为什么读取和修改很贵？**](#为什么读取和修改很贵)
- [Transient Storage](#transient-storage)
  - [constant的本质](#constant的本质)
  - [Transient Storage物理层面](#transient-storage物理层面)
- [Memory](#memory)
  - [**读写特性：字节寻址 vs 字操作**](#读写特性字节寻址-vs-字操作)
  - [布局](#布局)
    - [1、三大金刚](#1三大金刚)
    - [2、内存数组的“浪费”特性](#2内存数组的浪费特性)
    - [3、Memory 体现在合约里](#3memory-体现在合约里)
    - [4、Zero Slot作用](#4zero-slot作用)
- [Transient与Memory对比](#transient与memory对比)
- [Stack](#stack)
  - [体现在合约代码](#体现在合约代码)
  - [重点在变量的类型](#重点在变量的类型)
  - [为什么基础类型不能加 memory](#为什么基础类型不能加-memory)
- [CallData](#calldata)
  - [`calldata` vs `memory` 的执行逻辑对比](#calldata-vs-memory-的执行逻辑对比)
  - [引用声明不能省略](#引用声明不能省略)
- [Message Calls](#message-calls)
  - [A 合约如何调用 B 合约？](#a-合约如何调用-b-合约)
  - [**Gas 的“分配权”与 63/64 规则**](#gas-的分配权与-6364-规则)
  - [调用深度 vs 栈深度](#调用深度-vs-栈深度)
  - [this.B() 中的 this到底指谁](#thisb-中的-this到底指谁)
  - [external、public的真相](#externalpublic的真相)
- [Delegatecall](#delegatecall)
  - [用处](#用处)
    - [**1. 库（Library）**](#1-库library)
    - [**2. 代理模式（Proxy Pattern / 可升级合约）**](#2-代理模式proxy-pattern--可升级合约)
  - [致命危险：存储布局冲突（Storage Collision）](#致命危险存储布局冲突storage-collision)
  - [一个例子：](#一个例子)
  - [与Call总结对比](#与call总结对比)
- [Logs](#logs)
  - [日志的结构：Topics vs Data](#日志的结构topics-vs-data)
  - [例子](#例子)
  - [更详细的讲解](#更详细的讲解)
- [Precompiled Contracts](#precompiled-contracts)
  

# Storage

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.4.16 <0.9.0;

contract SimpleStorage {
    uint storedData;

    function set(uint x) public {
        storedData = x;
    }

    function get() public view returns (uint) {
        return storedData;
    }
}
```

## **它是如何存在 Key-Value 数据库里的？**

以太坊的底层（比如 Geth 客户端）通常使用 **LevelDB** 或 **RocksDB** 这种键值对数据库。但在逻辑层，`Storage` 被抽象为一个巨大的**插槽（Slots）列表**。

- **编译器的角色：** 当你定义 `uint storedData;` 时，Solidity 编译器会给它分配一个编号（即 **Slot 编号**）。因为它是合约里的第一个变量，所以它被分配到 **Slot 0**。

- **Key 的生成：** 实际上存入数据库的 Key 是由**合约地址**和**插槽编号**共同决定的。

- **存储动作：** 当你调用 `set(x)` 时，EVM 执行 `SSTORE` 指令。
  
  - **逻辑上的 Key：** `0`（代表第一个插槽）
  
  - **逻辑上的 Value：** 你传进去的 `x`

在底层的数据库里，它看起来就像这样：

`sha3(合约地址, 插槽0) => x 的值`

---

## **“256 位字（256-bit words）”是什么意思？**

这是很多人最容易困惑的地方。在普通的电脑（如 64 位系统）上，CPU 一次处理 64 位数据。但 **EVM 是一个 256 位的虚拟机**。

- **字长（Word Size）：** 256 位 = 32 字节。

- **Key 的大小：** 每个插槽的“地址”（Key）是 256 位的。这意味着总共有 $2^{256}$ 个插槽。这个数字大到近乎无限，你完全不用担心插槽不够用。

- **Value 的大小：** 每个插槽存储的数据（Value）固定也是 256 位的。

**举个例子：**

在你的 `SimpleStorage` 合约中：

1. 你定义了 `uint`（在 Solidity 中 `uint` 等同于 `uint256`）。

2. 这个变量恰好占满了 **一个完整的插槽（32 字节）**。

3. 如果你定义的是 `uint8`（只占 8 位，即 1 字节），Solidity 为了省钱，会尝试把多个小的变量“挤”进同一个 256 位的插槽里（这叫 **紧凑打包 Packing**），但逻辑上它们依然共享那个 256 位的空间。

## **为什么读取和修改很贵？**

- **读取（SLOAD）：** 就像你要坐电梯去这幢楼的某一层取东西。即使你只取 1 字节，你也得定位到那一层，把整层（32 字节）的数据搬出来。

- **修改（SSTORE）：** 这相当于你要改建这层楼。在区块链上，这意味着所有的矿工/验证者都要在他们的账本上同步这个改建信息，并永远保存下去。

<br />

# Transient Storage

**瞬时存储**，最好的方式是把它想象成一个“只在单次交易中存在的临时硬盘”。

它在结构上和 `Storage` 一模一样（也是 $2^{256}$ 个插槽，每个插槽 256 位），但它的**生命周期**不同：

- **Storage**：数据写进数据库，永久保存。

- **Transient Storage**：数据存在内存的一个特殊区域，**交易一旦结束（成功或失败），数据立刻抹除**。

目前，你需要使用 **内联汇编（Assembly）** 来操作它，使用的是 `tstore` 和 `tload` 指令。

示例代码：重入锁（Reentrancy Guard），这是瞬时存储最经典的应用场景。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract TransientExample {
    // 定义一个插槽位置（类似 Storage 的 Slot 0）
    bytes32 constant REENTRANCY_GUARD_SLOT = 0;

    modifier nonReentrant {
        // 1. 读取瞬时存储。如果值为 1，说明已经在调用中，触发报错
        require(tload(REENTRANCY_GUARD_SLOT) == 0, "Reentrancy attempted");

        // 2. 设置值为 1，表示锁定
        tstore(REENTRANCY_GUARD_SLOT, 1);

        _;

        // 3. 函数结束，解锁：设置回 0
        // 注意：即使你不设置回 0，这笔交易结束后它也会自动变回 0
        tstore(REENTRANCY_GUARD_SLOT, 0);
    }

    function tload(bytes32 slot) internal view returns (uint256 val) {
        assembly {
            val := tload(slot)
        }
    }

    function tstore(bytes32 slot, uint256 val) internal {
        assembly {
            tstore(slot, val)
        }
    }
}
```

特点：

- **跨调用持久（Inter-call Persistence）**：在同一笔交易中，无论你跨越了多少个合约调用（A -> B -> C），只要它们操作的是同一个合约的瞬时存储空间，数据是共享的。

- **低成本（Gas Efficiency）**：
  
  - `SSTORE`（永久存储）之所以贵，是因为要修改硬盘上的状态树。
  
  - `TSTORE` 读写成本（通常为 100 Gas）远低于 storage（首次修改可能需 20,000 Gas），因为它不涉及磁盘 IO，只涉及内存操作。

注意上面的代码，定义的constant是不占用Storage的：

## constant的本质

当你写下： `bytes32 constant REENTRANCY_GUARD_SLOT = 0;`

- **它不在 Storage 里：** 编译器在编译代码时，会把所有的 `REENTRANCY_GUARD_SLOT` 直接替换成数字 `0`。

- **它像“宏替换”：** 这就像你在文档里“查找并替换”。它不占用合约运行时的 Storage 空间，它只是硬编码在合约的代码区（Code Area）里。

所以，这一行代码只是为了代码可读性。你完全可以不写这一行，直接在汇编里写 `tstore(0, 1)`，效果是一模一样的。

## Transient Storage物理层面

它是以内存（RAM）的方式存在的。

在以太坊客户端（如 Geth）的底层实现中：

1. **Storage**：当你操作它时，底层会触发复杂的数据库查询（B-Tree 或 LSM Tree），涉及到从磁盘读取数据。

2. **Transient Storage**：底层实现就是一个简单的 **哈希表（Hash Map）**，这个表存在于节点的 **RAM（内存）** 中。
   
   - 当一笔交易开始，EVM 为这个合约创建一个空的 Map。
   
   - `tstore(0, v)` 就是在 Map 里存入 `{0: v}`。
   
   - **交易一结束，这个 Map 直接从内存中释放（销毁）。**

<br />

# Memory

合约在每次消息调用（message call）时都会获得一个全新的、已清空的内存实例。

## **读写特性：字节寻址 vs 字操作**

- **线性布局：** 内存就像一条无限长的字节数组（Byte Array）。

- **灵活写入：** 你可以只改 1 个字节（8 bits，用 `MSTORE8`），也可以一次改 32 个字节（256 bits，用 `MSTORE`）。

- **读取限制：** 虽然你可以从任何位置开始读，但每次 `MLOAD` 都会吐出 32 个字节（256 bits）。

这是 Solidity 优化中最需要注意的性能点：

- **按字扩展：** 即使你只访问第 33 个字节，EVM 也会直接开辟 64 字节的空间（2 个字）。

- **二次方惩罚（Quadratic Scaling）：** 内存的 Gas 开销不是线性的。
  
  - 前几个 KB 非常便宜。
  
  - 但随着内存占用增加，成本会呈指数级飙升。
  
  - *公式推导：* $Cost(n) = 3 \times words + \frac{words^2}{512}$。
  
  - **结论：** 永远不要在内存里存储超大规模的数组，否则单笔交易的 Gas 可能会直接爆表。

## 布局

### 1、三大金刚

Solidity 把内存最开始的 **128 字节 (0x00 - 0x7f)** 划为了特殊用途区：

1. **Scratch Space (0x00-0x3f)**：前 64 字节。当你计算 `keccak256` 哈希时，需要把数据拼在一起。为了省 Gas，Solidity 就在这儿临时摆放数据。
   
   比如你要算 `keccak256(a, b)`。Solidity 可能会悄悄把 `a` 和 `b` 拷到 `0x00` 地址，算完哈希就完事了。它不打算长期保存

2. **Free Memory Pointer (0x40-0x5f)**：空闲指针指向的位置。

3. **Zero Slot (0x60-0x7f)**：作为空数组或未初始化动态数组的参考点。它必须保持为 0，否则程序会崩溃。

### 2、内存数组的“浪费”特性

- **对齐处理**：为了处理效率，`bytes1[]` 这种数组里的每个元素（本应只占 1 字节）在内存里也会被强行撑大到占 **32 字节**。

- **例外情况**：`bytes` 和 `string` 是紧凑存储的，不会有这种浪费。

不要假设空闲内存指针指向的地方是干净的（全零）。

- **原因**：一些 Solidity 内部操作（比如 ABI 编码）会临时占用 `0x80` 往后的空间来做计算。计算完后，它不需要更新“空闲内存指针”，因为它是临时的。

- **后果**：当你手动通过汇编去 `0x80` 拿内存时，那里可能残留着上一次计算留下的“垃圾”数据。

### 3、Memory 体现在合约里

在 Solidity 代码中，你通常在以下几种情况看到 `memory` 关键字：

1) 情况 A：函数的局部复杂变量

当你处理结构体（struct）、数组或字符串时，必须指定它们存哪。

```solidity
function processData(string memory name) public pure {
    // 这里的 name 就在内存里
    // 如果 name 是 "Alice"，它可能被 Solidity 放在了地址 0x80 处
}
```

2) 情况 B：在函数内创建新数组

```solidity
function createArray() public pure {
    uint[] memory tempArray = new uint[](3); 
    // 这行代码执行时，Solidity 会：
    // 1. 去 0x40 位置看一眼：现在哪儿空着？（假设读到 0x80）
    // 2. 把 tempArray 放在 0x80。
    // 3. 更新 0x40 位置的值：现在 0x80 被占了，下次请从 0x100 开始（32*4 字节后）。
}
```

### 4、Zero Slot作用

这个 Zero Slot（零插槽） 的存在，主要是为了给那些“没被初始化”的动态数组提供一个安全的“空指向”目标。为了理解它，我们需要先看 Solidity 是如何处理动态内存数组的。

**1、背景：动态数组的结构**

在 Solidity 内存中，一个动态数组（比如 `uint[] memory`）的布局是：

1. **前 32 字节**：存储数组的**长度（length）**。

2. **后续空间**：存储具体的**元素**。

   **2、Zero Slot 的核心作用：安全占位**

****当你声明一个动态内存数组但**不给它赋值**（或者分配长度为 0）时，Solidity 需要让这个数组的指针指向某个地方。

- **如果指向 0x00**：那里是暂存空间，数据经常变，很危险。

- **如果指向 0x40**：那是空闲指针，万一不小心往里写数据，整个内存管理就崩溃了。

因此，Solidity 专门划出了 **0x60 - 0x7f** 这 32 字节，并规定：**这里永远只能是 0**。

**它的具体用途包括：**

- **空数组的指向**：一个长度为 0 的动态数组，其指针通常就指向这个 `0x60`。当你访问这个数组的 `.length` 时，EVM 去 `0x60` 读取，正好读到 0。

- **默认初始值**：在某些复杂的内存结构中，未初始化的引用类型会默认指向这里，以确保你即使意外访问，读到的也是“空值（0）”而不是乱码。

**一句话理解 Zero Slot：** 它是 Solidity 内存世界里的“虚无之地”，用来给所有空引用提供一个统一的、合法的、值永远为 0 的归宿。

<br />

# Transient与Memory对比

很多开发者会混淆这两者，因为它们都在 RAM 里，看这张表就能理清：

| **特性**     | **Memory (内存)**          | **Transient Storage (瞬时存储)** |
| ---------- | ------------------------ | ---------------------------- |
| **主要用途**   | 函数内计算、构建参数、返回数据          | 跨函数/跨合约调用共享状态、锁              |
| **作用域**    | **仅限当前调用帧** (Call Frame) | **整个交易** (Transaction)       |
| **数据结构**   | 连续字节序列 (像 Array)         | 键值对插槽 (像 Mapping)            |
| **重置时机**   | 当前子调用返回时                 | 整个交易执行结束时                    |
| **Gas 计算** | 随使用量**二次方**增长            | 固定成本 (**100 Gas** / 操作)      |

在大多数跨合约交互的场景下，Memory 无法替代 Transient Storage。

核心原因在于 **“作用域（Scope）”的物理墙**。

在 EVM 中，**Memory 是“私有”的**。

- 当你从合约 A 调用合约 B 时，EVM 会为合约 B 开辟一块**全新的、空白的**内存。

- 合约 B **无法读取**合约 A 的内存，合约 A 也**无法读取**合约 B 的内存。

举个例子：你想做一个“限时授权”，即在这一笔交易中，合约 A 临时允许合约 B 转走它的代币，但交易结束就作废。

- **用 Memory：** 合约 A 把授权信息存在自己的 Memory 里。当它调用合约 B 时，B 根本看不见这块内存，也就不知道自己被授权了。

- **用 Transient Storage：** 合约 A 将授权信息存入 `TSTORE`。由于瞬时存储在整笔交易中是共享的（只要是同一个合约环境），后续的调用逻辑可以通过 `TLOAD` 轻松读取到这个信号。

<br />

# Stack

EVM 不是寄存器机（register machine），而是**栈机（stack machine）**，因此所有的计算都在一个被称为 **stack（栈）** 的数据区域内进行。它的最大容量为 **1024 个元素**，每个元素是一个 **256 位**（32 字节）的字。

Op Code 是**指令**（动作），而 Stack 是指令操作的**对象**。比如 `ADD` 指令必须从 Stack 拿数字。

因为Op Code最多只能操作栈顶往下的16个元素，所以局部变量不要超过 16 个。

## 体现在合约代码

在合约函数中，简单的局部变量（如 `uint256`, `bool`, `address`）默认是放在 Stack 里的。

**1、案例拆解：代码到 Stack 的转化**

```solidity
function add(uint x) public pure returns (uint) {
    uint y = 10;
    uint z = x + y;
    return z;
}
```

当你调用这个函数时，EVM 内部发生了什么？

1. 参数入栈：调用者传入 `x`（假设是 5），EVM 首先把 `5` 压入 **Stack**。

2. 变量赋值：代码里有 `uint y = 10`。EVM 执行 `PUSH1 10` 指令，把 `10` 压入 Stack。
   
   - *此时栈内情况：[10, 5]（顶端是 10）*

3. 执行逻辑：代码执行 `x + y`。EVM 执行 `ADD` 指令。
   
   - `ADD` 指令会自动弹出栈顶的两个数（10 和 5），算出结果 15。
   
   - 计算结果 `15` 被压回 **Stack**。
   
   - *此时栈内情况：[15]*

4. 变量存储：如果你把结果赋给 `z`，那么此时栈顶的这个 `15` 在逻辑上就代表了变量 `z`。

结论：在这种情况下，`x`, `y`, `z` 全程都在 Stack 里，根本没去过 Memory 或 Storage。

**2、什么时候会用到 Memory？**

既然 Stack 这么好，为什么还要 Memory？

因为 Stack 太窄了，且无法处理“变长”或“大块”的数据。如果你的变量是以下几种，它就会去 **Memory**：

- **Struct（结构体）**

- **Array（数组）**

- **String（字符串）**

```solidity
function handleArray() public pure {
    uint[] memory arr = new uint[](3); // 这个数组的数据存在 Memory
    arr[0] = 1; 
}
```

- 物理层面：数组的实际内容（1, 0, 0）存放在 **Memory** 的 0x80 地址。

- Stack 层面：Stack 里依然会有一个值，但那个值不是数组本身，而是指向内存地址 0x80 的“指针”。

## 重点在变量的类型

在 Solidity 函数内部，如果你声明一个复杂类型（如数组、结构体或字符串），编译器**强迫**你必须指定它存在哪里。

我们可以根据变量的类型和声明方式，把这个规律总结为以下“三部曲”：

**1、简单变量（基础类型）— 默认 Stack**

对于 `uint`, `int`, `bool`, `address`, `bytes1` 到 `bytes32` 这些**基础类型**：

- 它们**永远**不能加 `memory` 关键字。

- 它们在函数内部**默认就是存在 Stack 里的**。

- *例子：* `uint x = 10;` —— 这个 `x` 就在 Stack 上。

**2、复杂变量（引用类型）— 必须手动指定**

对于 `struct`, `array`, `string` 这些**引用类型**：

- 你**不能**不写位置。如果你只写 `uint[] myList;`，编译器会报错，它要求你必须选一个地方。

- 如果你写 `uint[] memory myList;` —— 它存在 **Memory**（且在 Stack 留一个指针）。

- 如果你写 `uint[] storage myList = ...;` —— 这通常是一个**指针**，指向合约原本就有的某个 **Storage** 变量。

**3、特殊的例外：函数的输入参数**

- 在 `external` 函数中，你会看到 `uint[] calldata arr`。

- **Calldata** 是第五个存储区域：它是只读的、不可修改的，存储在交易的输入数据中。它的 Gas 成本比 Memory 还要低。

## 为什么基础类型不能加 memory

这其实是 Solidity 为了帮你省钱（Gas）。

- **Stack** 操作只需要 **3 Gas**（比如 `ADD`, `PUSH`）。

- **Memory** 操作至少需要 **3 Gas**（`MLOAD`/`MSTORE`），而且还要加上刚才说的**内存扩展费（二次方增长那个）**。

<br />

# CallData

Calldata 存储在以太坊节点的**交易池和区块数据**中。当 EVM 执行你的交易时，它并没有把这些数据加载到 Memory里。

**读取方式**：EVM 有专门的指令 `CALLDATALOAD` 和 `CALLDATACOPY`。

- 如果你声明为 `calldata`：EVM 就像拿着一个放大镜，在原始数据包上直接看。

- 如果你声明为 `memory`：EVM 就像拿了一张白纸（Memory），先把数据包里的内容抄写到纸上，然后再看。

## `calldata` vs `memory` 的执行逻辑对比

假设你调用 `deposit([10, 20])`：

**如果声明为 `calldata`：**

1. 节点收到交易，数据包里有 `[10, 20]`。

2. EVM 启动，`amounts` 变量在栈里只是一个**偏移量指针**，指向原始数据包。

3. 当你代码运行到 `amounts[0]` 时，EVM 直接去数据包对应位置读出 `10`。

4. **Gas 消耗极低**：因为没有“搬运”数据的动作。

**如果声明为 `memory`：**

1. 节点收到交易。

2. EVM 启动，第一件事就是**开辟内存空间**。

3. 执行“拷贝”：把数据包里的 `10` 和 `20` 逐个复制到刚开辟的内存地址中。

4. 当你代码运行到 `amounts[0]` 时，是从内存里读的。

5. **Gas 消耗高**：你支付了内存扩展费和数据拷贝费（`MSTORE`）。

## 引用声明不能省略

```solidity
function deposit(uint who, uint[] calldata amounts) external
```

为什么要强制声明 `calldata` 或 `memory`？

如果你尝试写 `function deposit(uint[] amounts) external`，编译器会直接报错：

> *TypeError: Data location must be "storage", "memory" or "calldata" for variable, but none was given.*

**原因如下：** 对于 `uint256` 这种基础类型，EVM 知道它永远占 32 字节，直接放栈（Stack）里就行，所以不需要声明。
但对于 `uint[]`（数组）、`string` 或 `struct` 这种**引用类型**，它们的大小是不确定的。Solidity 必须明确知道：

1. **你是想在原封不动的数据包里读？**（`calldata`：省 Gas，只读）

2. **你是想拷贝一份到内存里，待会儿还要修改它？**（`memory`：费 Gas，可写）

3. **你是想指向合约里现有的某个持久化数组？**（`storage`：只是一个指针）

<br />

# Message Calls

发起Transaction（交易）本质是一个Message Call。

在以太坊中，交易其实只是一个外壳。

- **Transaction**：它是外部世界（EOA 账户）进入 EVM 世界的**唯一入口**。它包含了签名、Nonce 和你付的小费。

- **Top-level Message Call**：一旦交易外壳被节点验证通过，EVM 就会触发一个“顶级消息调用”。

## A 合约如何调用 B 合约？

当 A 合约代码运行到调用 B 的地方时，EVM 会执行类似 `CALL` 的底层指令。

**物理过程如下：**

1. **准备数据**：A 会在自己的 **Memory** 里按照 ABI 规范拼凑出一串字节码（包含 B 的函数选择器和参数）。

2. **发起指令**：A 执行 `CALL` 指令，并把 B 的地址、准备好的内存地址、以及要发送的 Gas 传给 EVM。

3. **环境切换**：EVM 暂停 A 的执行，为 B 开辟全新的 **Memory** 和 **Stack**，并把 A 准备好的数据作为 **Calldata** 传给 B。

4. **执行 B**：B 开始运行。

5. **返回结果**：B 运行完，把结果放入 **Returndata**。

6. **恢复 A**：EVM 回到 A 的上下文，A 从 Returndata 拷贝数据到自己的内存，继续运行。

## **Gas 的“分配权”与 63/64 规则**

- **主动分配**：合约可以控制给子调用多少钱（Gas）。

- **保护机制（63/64）**：这是为了防止节点受到深度调用攻击。当你调用另一个合约时，EVM 强制要求你至少留下 `1/64` 的 Gas 给自己。这意味着即使被调用的合约把 Gas 全跑光了，你手里还剩一点点 Gas 来处理“善后工作”（比如报错处理）。
  
  - *注：这就是为什么实际深度达不到 1024 层的原因。

```solidity
// 不能指定比例，只能写具体的值，或者不指定。

// 不指定 Gas
// EVM 会自动把当前剩余 Gas 的 63/64 转发给 B

// 现代 Solidity 语法 (0.6.x 以后)
B.transfer{gas: 20000}(arg1, arg2);


// 或者使用底层的 call 方式：
(bool success, ) = address(B).call{gas: 20000}(
    abi.encodeWithSignature("transfer(address,uint256)", to, amount)
);
```

## 调用深度 vs 栈深度

很多初学者会把这两个“1024”搞混：

| **特性**    | **栈深度 (Stack Depth)** | **调用深度 (Call Depth)** |
| --------- | --------------------- | --------------------- |
| **限制值**   | 1024 个元素              | 1024 次连续调用            |
| **对象**    | 函数内部的计算数据 (uint, 指针等) | 合约 A -> B -> C -> ... |
| **失败后果**  | 编译失败 或 运行崩溃           | 触发异常，Gas 损失           |
| **开发者避坑** | 局部变量不要超过 16 个         | 尽量用循环代替递归             |

## this.B() 中的 this到底指谁

这是一个非常经典的问题。在 Solidity 中，`this` 永远指的是**当前合约实例的地址**。

但在调用时，写不写 `this` 有天壤之别：

**情况 A：直接调用 `B()`**

- **逻辑**：这被视为内部跳转（类似于其他编程语言的函数调用）。

- **过程**：EVM 只是跳到了当前合约代码的另一个位置。

- **环境**：**不产生**新的 Message Call。内存、栈、`msg.sender` 全部保持不变。

**情况 B：使用 `this.B()`**

- **逻辑**：这被视为一次**外部调用（External Call）**，尽管它是调自己。

- **过程**：A 合约通过 EVM 给“自己”发了一个消息。

- **环境**：**产生**了一个新的 Message Call。
  
  - B 会得到一块**全新的内存**（Freshly cleared instance）。
  
  - `msg.sender` 会变成 A 自己（合约地址）。
  
  - 受 **63/64 Gas 限制**。

## external、public的真相

结合上面的 Message Call 逻辑，你就明白为什么有这两个关键字了：

- **`external`（外部专用）**：
  
  - 这个函数被设计成**只接收** Message Call。
  
  - 它**期望**参数直接待在 `calldata` 区域里，不要往内存里搬。
  
  - 如果你在内部直接调它，编译器会报错；你必须用 `this.f()`，强迫它走一遍 Message Call 流程。

- **`public`（全能型）**：
  
  - 它生成了两套入口：一套给外部（走 Message Call），一套给内部（直接跳转）。
  
  - 为了兼容内部直接调用，它的参数必须支持在 **Memory** 中操作

<br />

# Delegatecall

存在一种特殊类型的消息调用，称为 **delegatecall**。它与普通的消息调用基本相同，唯一的区别在于：目标地址的代码是在**发起调用的合约上下文**（即地址）中执行的，且 `msg.sender` 和 `msg.value` 的值**不会改变**。

这意味着合约可以在运行时从另一个地址动态加载代码。**存储（Storage）、当前地址（address(this)）和余额（Balance）** 仍然指向发起调用的合约，只有代码（Code）是从被调用地址获取的。

 **A. 上下文（Context）的本质**

这是理解 `delegatecall` 的核心。在执行 `delegatecall` 时：

- **身体（我的）**：Storage, Balance, `address(this)` 全是我的。

- **灵魂（对方的）**：只有逻辑代码（Code）是对方的。

- **外界感知**：对于外界来说，`msg.sender` 还是最初的那个人，没有变。
  
  **B. 三个“不变”与一个“变”**

当你（合约 A）对合约 B 执行 `delegatecall` 时：

1. **`msg.sender` 不变**：如果用户调 A，A 调 B。在 B 的代码里，`msg.sender` 依然是用户，而不是 A。

2. **`address(this)` 不变**：在 B 的代码里查询地址，得到的是 A 的地址。

3. **`storage` 还是 A 的**：B 的代码如果写了 `slot[0] = 1`，改的是 **A 合约**的第 0 号插槽。

4. **唯独变了的是 `Code`**：执行的是 B 的逻辑

## 用处

### **1. 库（Library）**

库合约通常没有自己的状态变量（因为即使写了也改不到自己身上）。当你调用 `Library.sort(data)` 时，其实是把排序的逻辑“借”过来操作你自己的 `data` 数组。

### **2. 代理模式（Proxy Pattern / 可升级合约）**

这是 `delegatecall` 最强大的应用：

- **Proxy 合约（本体）**：负责存钱、存数据，地址永远不变。

- **Implementation 合约（逻辑）**：负责写代码。

- **操作**：用户调 Proxy，Proxy 通过 `delegatecall` 把请求转发给逻辑合约。

- **升级**：想升级逻辑？只需要让 Proxy 指向一个新的逻辑合约地址即可。数据全在 Proxy 里，完美保留。

---

## 致命危险：存储布局冲突（Storage Collision）

这是开发者最容易踩的坑。

- **场景**：合约 A 执行 `delegatecall` 合约 B。

- **冲突**：如果 A 的第 0 号插槽存的是 `owner`，而 B 的代码里认为第 0 号插槽存的是 `count`。

- **后果**：B 执行 `count = 100` 的时候，会无意中把 A 的 `owner` 给覆盖掉！

**所以，使用 `delegatecall` 时，两个合约的存储变量声明顺序必须完全一致。**

---

## 一个例子：

假设你的 **Proxy** 合约和 **Implementation** 合约的变量定义顺序不一样：

**Proxy 合约（本体）：**

```solidity
contract Proxy {
    address public implementation; // 存储在 Slot 0
    address public owner;          // 存储在 Slot 1

    function upgrade(address _newImpl) external {
        implementation = _newImpl;
    }

    fallback() external payable {
        // 使用 delegatecall 转发所有请求
        (bool success, ) = implementation.delegatecall(msg.data);
        require(success);
    }
}
```

**Implementation 合约（逻辑）：**

```solidity
contract Implementation {
    uint256 public count; // 在这里，逻辑合约认为 Slot 0 是 count！

    function increment() external {
        count = count + 1; 
    }
}
```

冲突发生了：

1. 用户调用 Proxy 的 `increment()`。

2. Proxy 发现自己没有这个函数，进入 `fallback`，并 `delegatecall` 给 Implementation。

3. Implementation 执行代码：`count = count + 1`。

4. **灾难：** 这里的 `count` 被编译器编译为“修改 Slot 0”。

5. 由于是 `delegatecall`，它修改的是 **Proxy 的 Slot 0**。

6. **结果：** Proxy 合约里的 `implementation` 地址被改成了 `implementation + 1`。下一次调用时，Proxy 将找不到逻辑合约，合约彻底报废！

---

**如何避免冲突？**

为了解决这个问题，社区（如 OpenZeppelin）通常采用以下两种方案：

方案 A：对齐存储布局（Inherited Storage）

确保 Proxy 和 Implementation 继承自同一个包含变量定义的合约，保证 Slot 顺序一模一样。

方案 B：非结构化存储（Unstructured Storage）— **最推荐**

Proxy 不把 `implementation` 地址放在 `Slot 0`，而是放在一个**随机且极远**的插槽里（例如通过哈希计算得出的地址），这样逻辑合约几乎不可能碰到那个位置。

Solidity

```
// Proxy 不再声明 address implementation，而是手动操作底层的槽位
bytes32 internal constant implSlot = keccak256("org.zeppelinos.proxy.implementation");

function _getImplementation() internal view returns (address impl) {
    bytes32 slot = implSlot;
    assembly {
        impl := sload(slot) // 从遥远的哈希位置读取地址
    }
}
```

## 与Call总结对比

| **维度**              | **Call (普通调用)** | **Delegatecall (委托调用)** |
| ------------------- | --------------- | ----------------------- |
| **代码执行环境**          | 被调用者 (B)        | **发起调用者 (A)**           |
| **Storage 影响**      | 修改 B 的存储        | **修改 A 的存储**            |
| **`msg.sender`**    | 变成 A            | **保持为最初的调用者**           |
| **`address(this)`** | B 的地址           | **A 的地址**               |
| **主要用途**            | 跨合约交互、转账        | **库、代理合约（可升级）**         |

<br />

# Logs

Solidity 利用这种被称为 **logs（日志）** 的功能来实现 **events（事件）**。

存日志比存 Storage **便宜得多**。如果你只需要让外部看到某些信息，而不指望合约逻辑去用它，存日志能省下 90% 以上的 Gas。

## 日志的结构：Topics vs Data

在 Solidity 中，你会看到 `indexed` 关键字。这决定了日志如何被索引：

- **Topics（标题/索引）**：
  
  - 被标记为 `indexed` 的参数会存在 Topic 里。
  
  - 搜索速度极快。
  
  - 一个事件最多只能有 **3 个** indexed 参数。

- **Data（内容）**：
  
  - 没有被标记为 `indexed` 的参数存在这里。
  
  - 搜索速度慢，但可以存非常大的数据（比如一段很长的字符串）。

一个日志条目（Log Entry）包含两个主要部分：

**部分 A：Topics (索引部分)**

这是一个长度是4的数组，元素32字节。

- **Topic 0**：事件的签名哈希 `0xddf252ad...`。

- **Topic 1**：`from` 地址。

- **Topic 2**：`to` 地址。

- *由于它们是索引过的，节点可以用布隆过滤器飞速查到它们。*

**部分 B：Data (非索引部分)**

这是一个字节流。

- **内容**：`_amount` 的值（256位整数）。

- *这个部分没有索引，只能在找到日志后进行解析读取。*

## 例子

```solidity
// 定义事件
event Transfer(address indexed from, address indexed to, uint256 value);


// 触发事件
emit Transfer(msg.sender, _to, _amount);
```

当执行到 `emit` 时，编译器会生成对应的 **`LOG0` 到 `LOG4`** 指令。

- **Opcode：** 根据 `indexed` 参数的数量，对应不同的指令：
  
  - 没有 `indexed` 参数：`LOG0`
  
  - 1 个 `indexed` 参数：`LOG1`
  
  - ...
  
  - 3 个 `indexed` 参数：`LOG4`（Topic 0 永远占一个坑位，所以最多只能再有 3 个 `indexed`）。

在我们的例子中，有两个 `indexed` 变量 + 一个事件签名哈希，总共 3 个 Topics，所以使用的 Opcode 是 **`LOG3`**。

**日志在区块里是以什么格式存在的？**

触发后，这些数据**不会**存在合约的 Storage 里。它们被打包进**收据（Receipt）**，最终存储在区块中。

## 更详细的讲解

1. 重新拆解：Log Entry 的物理结构

一个 Log 条目其实是由一个 **Topic 数组** 和一个 **Data 字节流** 组成的。

- **Topics 数组**：最多包含 4 个元素（Topic 0, Topic 1, Topic 2, Topic 3）。

- **关键点**：**数组里的每一个元素，都是固定 32 字节。**

所以，如果你有三个 `indexed` 参数，加上一个事件签名的 Topic 0，整个 Topics 部分总共占用 $4 \times 32 = 128$ 字节。

---

2. 各种类型是怎么塞进 32 字节的？

不同类型的参数在变成 Topic 时，处理方式不同：

**情况 A：刚好 32 字节的类型（如 `uint256`, `address`, `bytes32`）**

- **`uint256`**：它本身就是 32 字节（256 位），所以直接原封不动地塞进一个 Topic 坑位里。

- **`address`**：它是 20 字节，存入 32 字节的 Topic 时，会在左侧补零（Padding），凑齐 32 字节。

**情况 B：超过 32 字节或变长类型（如 `string`, `bytes`, `struct`）**

这就是最神奇的地方了！如果你的 `indexed` 参数是一个很长的字符串（比如 100 字节），它怎么塞进 32 字节的 Topic 呢？

- **答案是：哈希（Hashing）。**

- 如果一个变长类型被标记为 `indexed`，EVM 会先对这个数据执行 `keccak256` 哈希运算，得到一个固定 32 字节的哈希值，然后把这个**哈希值**存入 Topic。

- **后果**：你在日志索引里能看到它的“指纹”，但你无法从 Topic 里还原出原始的字符串内容。

---

3. 举个具体的例子

Solidity

```
event MyEvent(
    address indexed user,   // Topic 1
    uint256 indexed id,     // Topic 2
    string indexed name,    // Topic 3
    uint256 salary          // Data 部分 (非 indexed)
);
```

当你触发 `emit MyEvent(0x123..., 888, "Alice", 5000)` 时，日志长这样：

| **位置**      | **内容**           | **长度** | **说明**            |
| ----------- | ---------------- | ------ | ----------------- |
| **Topic 0** | `0xabcdef...`    | 32 字节  | 事件签名哈希            |
| **Topic 1** | `0x000...123...` | 32 字节  | `user` 地址（左侧补零）   |
| **Topic 2** | `0x000...378`    | 32 字节  | `id` 的十六进制（888）   |
| **Topic 3** | `0x7123...`      | 32 字节  | **"Alice" 的哈希值**  |
| **Data**    | `0x000...1388`   | 32 字节  | `salary` 的值（5000） |

---

<br />

# Precompiled Contracts

目前常见的预编译合约（1 ~ 10）

你可以通过下表看看这些地址都负责什么“重活”：

| **地址 (Hex)** | **功能**               | **用途示例**                       |
| ------------ | -------------------- | ------------------------------ |
| `0x01`       | **ecrecover**        | 验证签名（根据哈希和签名找回公钥地址）            |
| `0x02`       | **SHA2-256**         | 计算 SHA256 哈希                   |
| `0x03`       | **RIPEMD-160**       | 计算 RIPEMD160 哈希                |
| `0x04`       | **identity**         | 原样输出（用于高效拷贝大块内存数据）             |
| `0x05`       | **modexp**           | 大数模幂运算（用于 RSA 验证等）             |
| `0x06`       | **ecAdd**            | 椭圆曲线加法 (BN254)                 |
| `0x07`       | **ecMul**            | 椭圆曲线标量乘法 (BN254)               |
| `0x08`       | **ecPairing**        | 椭圆曲线配对（**ZK-SNARKs 零知识证明的核心**） |
| `0x09`       | **blake2f**          | BLAKE2b 哈希函数运算                 |
| `0x0a`       | **point_evaluation** | 用于 EIP-4844（坎昆升级）的 Blob 数据验证   |

---

如何在 Solidity 里调用它们？

通常你不需要手动去调这些地址，Solidity 已经封装好了。

- **例子：验证签名**
  
  当你写 `ecrecover(hash, v, r, s)` 时，Solidity 底层其实是执行了一个对地址 `0x01` 的 `STATICCALL`。

- **例子：零知识证明**
  
  如果你在做 ZK-Rollup 的合约，你会频繁地用 `call` 指令去敲 `0x08` 的门，因为那是处理零知识证明最划算的方式。
