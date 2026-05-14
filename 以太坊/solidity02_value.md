- [Fixed Point number](#fixed-point-number)
- [Address](#address)
  - [成员](#成员)
- [定长字节数组(Fixed-size byte arrays)](#定长字节数组fixed-size-byte-arrays)
- [Rational and Integer Literal有理数和整数](#rational-and-integer-literal有理数和整数)
  - [数字字面量Number literal](#数字字面量number-literal)
  - [总结口诀](#总结口诀)
- [位运算、幂运算](#位运算幂运算)
- [String Literals字符串字面量](#string-literals字符串字面量)
  - [核心特性拆解](#核心特性拆解)
  - [隐式转换：字符串的“变身”](#隐式转换字符串的变身)
  - [转义字符与十六进制](#转义字符与十六进制)
- [Unicode Literals](#unicode-literals)
- [Enums](#enums)
- [用户自定义值类型 (User-defined Value Types)](#用户自定义值类型-user-defined-value-types)
- [Function](#function)
  - [转换](#转换)
  - [**Solidity 外部函数变量在 ABI 编码（ABI Encoding）时的底层表现**。](#solidity-外部函数变量在-abi-编码abi-encoding时的底层表现)
  - [示例代码的一点疑问](#示例代码的一点疑问)
  - [合约更新时的稳定性](#合约更新时的稳定性)
- [合约升级的一点疑问](#合约升级的一点疑问)
  - [1. 代理合约 A 到底长什么样？](#1-代理合约-a-到底长什么样)
  - [2. 为什么只在 B 里加变量？](#2-为什么只在-b-里加变量)
      - [**发生了什么？**](#发生了什么)





# Fixed Point number

[Types &mdash; Solidity 0.8.36-develop documentation](https://docs.soliditylang.org/en/latest/types.html#fixed-point-numbers)

`ufixedMxN` and `fixedMxN`

这完全就是在瞎搞，编译器都没有实现，就放出文档来。恶心人。写合约的时候别用！

# Address

[Types &mdash; Solidity 0.8.36-develop documentation](https://docs.soliditylang.org/en/latest/types.html#members-of-addresses)

两个address：

- `address`: 20个byte

- `address payable`: Same as `address`, but with the additional members `transfer` and `send`.

## 成员

* ### balance

* ### transfer
  
  payable才有的。向x这个账号转账。
  
  ```solidity
  address payable x = payable(0x123);
  address myAddress = address(this);
  if (x.balance < 10 && myAddress.balance >= 10) x.transfer(10);
  ```

但是transfer方法已经被弃用了。

1. 为什么当初设计 `transfer`？（曾经的英雄）

在 Solidity 早期，转账最怕的是**重入攻击（Reentrancy）**。
为了“保护”开发者，官方推出了 `transfer`，它有一个硬性限制：**只给接收方转发 2300 Gas。**

- **2300 Gas 能干嘛？** 只够接收方打印一个日志（Log），**绝对不够**接收方再调回你的合约（因为转账、改数据都要几万 Gas）。

- **结果**：由于 Gas 锁死了，重入攻击在物理上就不可能发生了。
2. 为什么现在要废弃它？（现在的累赘）

以太坊是一个会“通货膨胀”的系统，但通胀的是 **Opcode 的 Gas 成本**。

- **Gas 成本会变**：随着硬分叉（如柏林升级、伦敦升级），某些指令的 Gas 消耗被调高了。

- **后果**：以前 2300 Gas 能做完的事，现在可能不够了。

- **尴尬局面**：如果一个合约的 `receive()` 函数里稍微多写了一行代码，以前能用 `transfer` 接收转账，升级后就会因为 Gas 不足而**永远无法接收转账**，导致合约直接“变砖”。

**结论：** 官方意识到硬编码 Gas 上限（2300）是一个极其愚蠢的决定，因为它破坏了合约的**未来兼容性**。

* ### send
  
  `send` 是 `transfer` 的“亲兄弟”，但它更原始、更危险。
  
  1. `send` vs `transfer`：谁更硬？
  
  它们两个在 **Gas 限制**上是一模一样的：都只给接收方 **2300 Gas**。
  
  它们的本质区别在于**失败后的态度**：
  
  - **`transfer` (刚烈)**：如果转账失败（比如对方合约拒收，或者 Gas 不够），它直接**原地爆炸**（抛出异常，整个交易回滚）。
  
  - **`send` (圆滑)**：如果转账失败，它**不报错**，只是默默地给你返回一个 `false`。程序会继续往下跑。
  
  > **致命危险：** 如果你写了 `x.send(10);` 但没有检查返回值，万一转账失败了，你的合约会以为转账成功了并继续执行后面的逻辑（比如把商品发给用户），这会导致严重的资金损失。
2. 为什么 `send` 极其危险？（两个陷阱）

文档提到了两个让 `send` 变得不可靠的原因：

**A. 1024 调用栈深度攻击**

我们在之前学过，EVM 的调用深度限制是 1024 层。

- 攻击者可以先递归调用自己 1023 次，然后再触发你的合约。

- 此时，你的合约尝试执行 `send`，因为深度达到了 1024，`send` 会**强制失败**并返回 `false`。

- 如果你没检查这个 `false`，攻击就成功了。

**B. Gas 耗尽**

和 `transfer` 一样，2300 Gas 实在是太少了。只要对方合约稍微多做一点点事情，`send` 就会因为 Out of Gas 失败并返回 `false`。

这里衍生出了一种安全的模式：**“Withdraw”模式是什么？**

文档最后提到了一句：*“use a pattern where the recipient withdraws the Ether”*。

这是智能合约安全中最高级的思想：**从“推送”转为“拉取”（Pull-over-Push）**。

- **Push（推送/坏习惯）**：合约主动把钱 `send` 给用户。如果其中一个用户是恶意合约（拒收），可能会导致你的整个发钱函数（比如发奖金循环）因为其中一人报错而卡死。

- **Pull（拉取/好习惯）**：合约不主动发钱，只是记一笔账：`balances[user] += reward`。用户想要钱的时候，自己来调 `withdraw()` 函数把钱领走。

**这种模式下，就算用户是个恶意合约，他也只能搞死他自己的 `withdraw` 交易，影响不到你的主合约逻辑。**

* ### call
  
  文档建议使用底层的 `call`。它的写法很啰嗦，但它是目前最推荐的：
  
  ```solidity
  // 现代标准写法：
  (bool success, ) = x.call{value: 10}(""); 
  require(success, "Transfer failed.");
  ```

  address(nameReg).call{gas: 1000000, value: 1 ether}(
    abi.encodeWithSignature("register(string)", "MyName")
  );

```
#### **`call` 和 `transfer` 的核心区别：**

| **特性**     | **x.transfer(amount)** | **x.call{value: amount}("")** |
| ---------- | ---------------------- | ----------------------------- |
| **Gas 转发** | **固定 2300**（死扣）        | **转发所有剩余 Gas**（慷慨）            |
| **失败处理**   | 自动 `revert`            | 返回 `false`（需要你手动处理）           |
| **安全性**    | 自动防重入                  | **不防重入**（需要开发者自己加锁）           |

* ### delegatecall、staticcall

这是最容易混淆的地方，我们用一张表和一句话来彻底分清它们：

**Staticcall：安全的“眼睛”**

它是拜占庭硬分叉引入的。它在底层执行时会设置一个“只读标志位”。如果被调用的代码里包含 `SSTORE`（修改存储）或 `LOG`（发事件），EVM 会直接强制报错。这在调用不信任的合约进行查询时非常安全。

用一张表和一句话来彻底分清它们：

| **函数**             | **存储 (Storage)** | **余额 (Balance)** | **msg.sender** | **状态修改权限** | **典型场景**                 |
| ------------------ | ---------------- | ---------------- | -------------- | ---------- | ------------------------ |
| **`call`**         | 属于 **被调用者 B**    | 属于 **被调用者 B**    | 变成 **调用者 A**   | 允许修改       | 普通转账、调用外部合约              |
| **`delegatecall`** | 属于 **调用者 A**     | 属于 **调用者 A**     | **保持原始用户**     | 允许修改       | 代理模式 (Proxy)、库 (Library) |
| **`staticcall`**   | **不允许修改**        | 属于 **被调用者 B**    | 变成 **调用者 A**   | **严禁修改**   | 只读调用 (View/Pure)、安全性检查   |

为什么它们被称为“低级函数”（Low-level）？

文档多次警告“Last Resort”（最后手段），原因有三：

1. **失去类型检查**：普通的 `x.f(123)` 如果你传了字符串，编译器会报错。但 `call(payload)` 只是发送一串字节流，编译器不知道你发的是什么，发错了也不会提醒你。

2. **不处理失败**：正如我们之前聊 `send` 时说的，这些函数不会自动 `revert`。如果被调用者报错了，它们只会返回 `false`，你必须手动处理。

3. **重入风险（Reentrancy）**：当你使用 `call` 时，你实际上把控制权交给了对方。对方可以在返回之前，反过来再次调用你的函数，把你账户里的钱掏空。

<br />

# abi.encode、encodePacked、encodeWithSignature、encodeWithSelector

**abi.encode**

* 严格遵循 ABI 规范。每一个参数都会被填充（Padding）到 **32 字节**。

* **例子**：`abi.encode(uint8(1), uint256(2))` 会生成 64 字节的数据（两个 32 字节块）。

*  `string` 的三段式编码：偏移、长度、内容总共占用32 x 3 = 96字节。



**abi.encodePacked**

* 极度精简。它只取出字符串的原始字节，不加偏移量，不加长度，也不补齐。

* 对于 `"Hello!"`，它只占 **6 字节**。

* 用途：计算 `keccak256` 哈希，节省存储空间。



**abi.encodeWithSignature**

- **用法**：你给它一个字符串形式的函数签名，比如 `"transfer(address,uint256)"`。

- **内部逻辑**：

1. 它先计算 `keccak256("transfer(address,uint256)")`。

2. 取前 4 个字节（即 Selector）。

3. 后面接上用 `abi.encode` 编码后的参数。



 **abi.encodeWithSelector (选择器编码)**

- **用法**：你直接给它已经算好的 4 字节 `selector`。通常配合接口或合约实例使用：`IERC20.transfer.selector`。

- **内部逻辑**：直接把那 4 字节拼在前面，后面接参数编码。

- **优点**：**更安全**。如果你重命名了函数，编译器会自动报错，避免了 `encodeWithSignature` 拼错字符串的问题。



| **函数**                        | **填充到 32 字节？** | **包含 4 字节 Selector？** | **主要用途**                      |
| ----------------------------- | -------------- | --------------------- | ----------------------------- |
| **`abi.encode`**              | **是**          | 否                     | 返回数据给其他合约、标准的跨合约交互。           |
| **`abi.encodePacked`**        | **否**          | 否                     | 计算哈希 `keccak256(...)`、节省 Gas。 |
| **`abi.encodeWithSignature`** | **是**          | **是**                 | 快速构造 `call` 的 payload。        |
| **`abi.encodeWithSelector`**  | **是**          | **是**                 | **最推荐** 的底层调用方式，具备编译时检查。      |

<br />

# receive & fallback

`receive` **必须**是 `payable` 的。如果你写 `receive() external` 而不加 `payable`，编译器根本不会让你过。

```solidity
x.call{value: 10}("")
```

1. `msg.data` 为空时 (纯转账)
- **情况 1：有 `receive() payable`**
  
  - **执行 `receive`**。 (这是最标准的路径)

- **情况 2：没有 `receive`，有 `fallback() payable`**
  
  - **执行 `fallback`**。 (这是兜底路径)

- **情况 3：没有 `receive`，只有 `fallback()` (不带 payable)**
  
  - **报错 Revert**。 (因为没人能收这笔钱)

- **情况 4：有 `receive`，但没加 `payable`**
  
  - **编译不通过**。 (Solidity 规定 `receive` 必须带 `payable`)
2. `msg.data` 不为空时 (带数据的调用)

当 `msg.data` 有内容时，EVM 的第一反应是：“你是来调我某个函数的吧？”。它会直接**跳过** `receive`，因为 `receive` 只管“纯钱”。

- **情况 5：有 `fallback() payable`，且带钱 (`value > 0`)**
  
  - **执行 `fallback`**。 (即使 `msg.data` 匹配不到任何函数，因为它是 `payable` 的，所以能收下这笔钱并执行逻辑)

- **情况 6：只有 `fallback()` (不带 payable)，且带钱 (`value > 0`)**
  
  - **报错 Revert**。 (因为虽然触发了 `fallback`，但它没权限收钱)

- **情况 7：只有 `fallback()` (不带 payable)，但不带钱 (`value == 0`)**
  
  - **执行 `fallback`**。 (没钱，不需要 `payable` 权限，正常执行逻辑)

<br />

# 定长字节数组(Fixed-size byte arrays)

值类型 `bytes1`, `bytes2`, `bytes3`, ..., `bytes32` 持有从 1 到 32 字节的序列。

在不同的存储介质里，排列会不同。

1. 战场一：Storage（持久化存储）—— **会紧凑排列**

在 Storage 中，EVM 的插槽（Slot）是 32 字节。Solidity 编译器会尽可能把多个变量“塞”进同一个插槽来省钱（毕竟写一个新的插槽要 20,000 Gas）。

Solidity

```
contract StorageTest {
    // 占用 Slot 0
    bytes1  public a; // 1 字节
    bytes1  public b; // 1 字节
    bytes30 public c; // 30 字节
    // 合计 32 字节，a, b, c 全部挤在 Slot 0 里！

    // 占用 Slot 1
    bytes32 public d; // 独自占用 Slot 1
}
```

Storage 的紧凑存储：如何“单独”修改？

EVM 物理层面的存储指令只有 `SSTORE`（写一整个 32 字节）。但你在 Solidity 里写 `a = 0x0f;` 是**完全可行**的。

**编译器是怎么做到的？** 当你修改 `a` 时，编译器并不会只改变那 1 字节，它会生成一段汇编逻辑，执行以下“三部曲”：

1. **读取（SLOAD）**：先把这整个 Slot 0（32 字节）读到栈上。

2. **修改（Mask & Or）**：
   
   - 它会用一个**掩码**把 `a` 原来的位置“抠掉”（清零）。
   
   - 然后把你的新值 `0x0f` 用 `OR` 指令放进去。

3. **写回（SSTORE）**：把这个修改后的、全新的 32 字节存回 Slot 0。

**举个具体的位运算例子：** 假设 Slot 原本是 `0x112233...`，你想把第一字节 `11` 改成 `AA`：

- **原始数据**：`0x112233...`

- **掩码清零**：`0x002233...`（通过 `AND 0x00FFFF...` 实现）

- **并入新值**：`0xAA2233...`（通过 `OR 0xAA0000...` 实现）

- **整存**：把 `0xAA2233...` 存入。

**结论**：你可以单独赋值，但底层的 Gas 消耗并不会因为你只改了 1 字节而变便宜。**只要改了 Slot 里的任何部分，都要付整个 Slot 的写费。** 紧凑存储最大的好处是让你在**读**多个变量时只需要一次 `SLOAD`。

**结论：** 在 Storage 中，`bytesN` 会进行 **紧凑打包 (Packing)**。

---

2. 战场二：Stack（栈）—— **不紧凑**

在栈上，EVM 的最小单位就是 **Word（32 字节）**。

- 如果你声明一个 `bytes1 a`，它在栈上依然会占据 **一整个 32 字节的槽位**。

- 栈是用来计算的，为了速度，它不搞紧凑压缩。

**结论：** 栈上 **不紧凑**，每个变量一个坑（32 字节）。

**`bytesN` 是值类型（Value Type）**：

```solidity
bytes3 memory data; // 这种写法是错误的！
```

- 在 Solidity 中，只有引用类型（Reference Types）才需要加 `memory`、`storage` 或 `calldata`。

- 引用类型包括：`string`、动态 `bytes`、`数组`、`struct` 和 `mapping`。

- `bytesN`（如 `bytes32`）和 `uint256`、`bool` 一样，是直接存放在栈上的。

<br />

# Rational and Integer Literal有理数和整数

## 数字字面量Number literal

可以承受任意精度：

```solidity
unit8 = (2**800 + 1) - 2**800 // 尽管超过了256bit，但是是可以的。
```

但有两个例外：

1、**三元运算符的坑：**

- **理想情况**：`255 + 1` $\rightarrow$ 编译器知道结果是 `256`，它能自动适应 `uint16`。

- **三元运算符**：`255 + (true ? 1 : 0)`
  
  - 编译器看到三元运算符时，会尝试给内部结果定性。它会觉得 `(true ? 1 : 0)` 的结果应该是最小的整数类型（即 `uint8`）。
  
  - 于是计算变成了：`uint8(255) + uint8(1)`。
  
  - **结果**：在 `uint8` 里溢出了！变成了 `0`。

2、 **数组下标的坑：**

- `255 + [1, 2, 3][0]`
  
  - 编译器看到数组 `[1, 2, 3]`，会把它推导为 `uint8[3]`。
  
  - `[0]` 取出的值也是 `uint8` 类型。
  
  - **结果**：同上，`uint8(255) + uint8(1)` 发生溢出。

## 总结口诀

1. **字面量全家桶**：只要不碰到变量，它们就是数学上的“理想数字”，算到多大、多精细都没事。

2. **遇到变量就破功**：一旦碰上变量，字面量必须“现形”变成固定大小的整数。

3. **三元和数组是陷阱**：它们会让字面量提前“现形”，导致意外溢出。
   
   最后一个例子：为什么 `2.5 + a + 0.5` 报错？

```solidity
uint128 a = 1;
uint128 b = 2.5 + a + 0.5; // 报错！
```

- **原因**：计算是从左往右执行的。

- **第一步**：`2.5 + a`。
  
  - `2.5` 是一个字面量类型（分数）。
  
  - `a` 是 `uint128`。
  
  - **冲突**：当字面量遇到变量时，它必须转换成变量的类型。但 `uint128` 无法承载 `2.5`（因为它有小数）。编译器找不出一个中间类型能同时兼容这两者，于是报错。

- **修正方法**：
  
  Solidity
  
  ```
  uint128 b = 2.5 + 0.5 + a; // 正确！
  ```
  
  - **原理**：`2.5 + 0.5` 都是字面量，先在内部算成 `3`（无限精度）。
  
  - 然后 `3 + a`。此时 `3` 是整数，可以完美转成 `uint128`。

<br />

# 位运算、幂运算

这两个运算的结果类型都是以左操作数为决定。分几种情况：

1、左操作数是**数字字面量**。会提升到uint256。

```solidity
function test() public pure {
    uint8 exponent = 2; // 右边是一个 uint8 变量

    // 运算：3 ** exponent
    // 左边是字面量 3，右边是变量 exponent
    // 即使 exponent 是 uint8，这个 3 也会被提升为 uint256。
    // 整个表达式的结果类型是 uint256。

    uint256 result = 3 ** exponent; 

    // 如果你尝试赋值给 uint8，可能会报错（因为 uint256 不能直接给 uint8）
    // uint8 smallResult = 3 ** exponent; // 编译器会提示需要显式转换

    uint8 amount = 2;
    // 1 << amount
    // 这里的 1 是字面量，amount 是 uint8 变量。
    // 这里的 1 会被当做 uint256 处理。
    uint256 shifted = 1 << amount; // 结果是 uint256 类型的 4
}
```

2、左操作数是变量。无论右操作数是否是字面量。

```solidity
uint8 a = 2;
// 结果是 uint8 类型
uint8 result = a ** 3; 

// 危险！如果结果超过 uint8 (255)，会直接报错 (0.8.0+)
// uint8 overflow = a ** 8; // 2^8 = 256，报错 Revert


int8 a = 1;
// 结果随左边，是 uint8 类型
uint8 result = a << 2; // 4

// 即使右边字面量很大，结果依然是 uint8
uint8 lost = a << 8; // 结果是 0（因为 uint8 只有 8 位，全移出去了）
```

<br />

# String Literals字符串字面量

在 Solidity 底层，它们本质上就是一串 **字节（Bytes）**。字符串字面量只能包含 ASCII。

## 核心特性拆解

- **引号通用**：单引号 `'foo'` 和双引号 `"foo"` 没区别。

- **自动拼接**：`"foo" "bar"` 会被编译器直接处理成 `"foobar"`。这在写超长字符串（比如 IPFS 哈希或报错信息）时非常有用，可以换行写增加可读性。

- **没有终止符 `\0`**：这和 C 语言不同。`"foo"` 在内存和存储里就是 3 个字节，**不会**在末尾自动加个 `0`。

- **字符集限制**：默认只能包含可见的 **ASCII 字符**（0x20 到 0x7E）。如果你想存中文、韩文或特殊符号，必须使用 **Unicode 字符串**（下一节会讲到）或转义符。

---

## 隐式转换：字符串的“变身”

字符串字面量非常灵活，它可以根据你赋值的对象自动改变身份：

**A. 转换为定长字节数组 (`bytes1` 到 `bytes32`)**

如果字符串很短，你可以直接把它存进 `bytesN`。

```solidity
bytes32 name = "MyName"; 
// 此时 "MyName" 被解释为原始字节流，存入 32 字节插槽，右侧补 0
```

当字符串赋值给 `bytes32` 时，它的排列方式和 `uint256` 完全不同，这一点极其容易搞混：

**1. 字符串转 `bytes32`（左对齐）**

Solidity

```
bytes32 x = "abc";
// 在内存/存储中长这样：
// 0x6162630000000000000000000000000000000000000000000000000000000000
```

- **特点**：数据在**高位**（左边），剩下的右边补 `0`。

**2. 数字转 `uint256`（右对齐）**

Solidity

```
uint256 y = 0xabc;
// 在内存/存储中长这样：
// 0x0000000000000000000000000000000000000000000000000000000000000abc
```

- **特点**：数据在**低位**（右边），左边补 `0`。

**B. 转换为动态字节数组 (`bytes`) 或字符串 (`string`)**

```solidity
string memory s = "hello";
bytes memory b = "hello";
```

---

## 转义字符与十六进制

你可以通过转义符插入无法直接输入的字符：

- **`\xNN`**：插入特定的十六进制字节。比如 `\x41` 就是字母 `A`。

- **`\uNNNN`**：插入 Unicode 字符，编译器会自动把它转成 **UTF-8** 编码的字节序列。

<br />

# Unicode Literals

由于普通的字符串字面量只能包含 ASCII，Solidity 专门提供了一个前缀 `unicode` 来处理非 ASCII 字符（比如中文、表情符号）：

**核心规则：**

- **语法**：`unicode"Hello 🐼"` 或 `unicode'你好'`。

- **编码**：这些字符会被编译为 **UTF-8** 字节流。

- **注意**：`unicode` 前缀不能和 `hex` 前缀混用。

**代码示例：**

```solidity
function testUnicode() public pure {
    // 报错：普通的字符串字面量不支持中文
    // string memory s = "你好"; 

    // 正确：使用 unicode 前缀
    string memory s1 = unicode"你好"; 

    // 看看长度
    // bytes(s1).length 是 6，而不是 2！
    // 因为在 UTF-8 中，一个中文字符通常占 3 个字节。
}
```

---

补充：十六进制字面量 (Hex Literals)

除了 `unicode`，还有一个常用的前缀是 `hex`：

- **语法**：`hex"001122FF"`。

- **特点**：它不是字符串，它是**原始字节**。

- **要求**：必须是偶数个十六进制字符（因为一个字节由两个十六进制位组成）。

- **多个部分**：`hex"0011" hex"2233"` 等同于 `hex"00112233"`。

**它和字符串字面量的区别：**

- `"12"` $\rightarrow$ 存的是 ASCII 码：`0x3132` (2字节)。

- `hex"12"` $\rightarrow$ 存的是原始数据：`0x12` (1字节)。

<br />

# Enums

enum不能超过256个，Solidity会看它看成unit8。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity ^0.8.8;

contract test {
    enum ActionChoices { GoLeft, GoRight, GoStraight, SitStill }
    ActionChoices choice;
    ActionChoices constant defaultChoice = ActionChoices.GoStraight;

    function setGoStraight() public {
        choice = ActionChoices.GoStraight;
    }

    // Since enum types are not part of the ABI, the signature of "getChoice"
    // will automatically be changed to "getChoice() returns (uint8)"
    // for all matters external to Solidity.
    function getChoice() public view returns (ActionChoices) {
        return choice;
    }

    function getDefaultChoice() public pure returns (uint) {
        return uint(defaultChoice);
    }

    function getLargestValue() public pure returns (ActionChoices) {
        return type(ActionChoices).max;
    }

    function getSmallestValue() public pure returns (ActionChoices) {
        return type(ActionChoices).min;
    }
}
```

<br />

# 用户自定义值类型 (User-defined Value Types)

Solidity 0.8.8 引入的一个非常强大的特性。

1. **什么是“零成本抽象” (Zero Cost Abstraction)？**

这是文档里最装深沉的一个词。简单来说：

- **开发时**：它是一个独立的类型，编译器会帮你检查，防止你把“美元”和“日元”搞混。

- **运行时**：它在 EVM 层面会被完全还原回底层的 `uint256`。

- **零成本**：它**不会消耗任何额外的 Gas**。它纯粹是编译器层面的“安全围栏”。

---

2. **核心语法：`wrap` 与 `unwrap`**

要使用这个类型，你必须通过两个专属“传送门”：

- **`C.wrap(V)`**：把原始数据（如 `uint256`）**封印**进自定义类型 `C`。

- **`C.unwrap(C)`**：把自定义类型 `C` **拆封**回原始数据。

> **注意：** 这个自定义类型极其“简陋”。它**没有任何操作符**。你不能用 `+`、`-`、`*`、`/`，甚至连 `==` 都不行。你想算数，必须先 `unwrap` 出来。

**这种类型的特性总结**

1. **ABI 兼容**：在 ABI 接口中，它依然显示为底层的 `uint256`。这意味着前端调用时不需要特殊处理。

2. **不可隐式转换**：你不能写 `UFixed256x18 a = 123;`，必须写 `wrap`。

3. **安全性**：强制开发者明确自己在操作什么。

<br />

# Function

两个成员：

* address

* selector

在 Solidity 里，**函数本身也是一种“值”**。你可以把函数赋值给变量，也可以把函数当作参数传给另一个函数。

```solidity
contract FunctionTest {
    // 定义一个函数变量：接受 uint，返回 uint，内部类型
    function (uint) internal pure returns (uint) op;

    function square(uint x) internal pure returns (uint) {
        return x * x;
    }

    function double(uint x) internal pure returns (uint) {
        return x * 2;
    }

    function setOp(bool useSquare) public {
        if (useSquare) {
            op = square; // 把函数赋值给变量
        } else {
            op = double;
        }
    }

    function runOp(uint x) public view returns (uint) {
        // 像调用普通函数一样调用变量
        return op(x);
    }
}


contract Oracle {
    // 接受一个外部函数作为回调
    function getData(function(uint) external callback) public {
        // 获取数据后，调用这个回调
        callback(42); 
    }
}
```

## 转换

函数类型 $A$ 可以隐式转换为函数类型 $B$，当且仅当：它们的参数类型完全相同、返回类型完全相同、内部/外部属性相同，且 **$A$ 的状态可变性比 $B$ 更严格**。具体规则如下：

- `pure` 函数可以转换为 `view` 和 `non-payable` 函数。

- `view` 函数可以转换为 `non-payable` 函数。

- `payable` 函数可以转换为 `non-payable` 函数。
  
  除此之外，函数类型之间不存在其他转换。

---

就只这样去理解就好了：

view = pure，把pure转成view，view的本意思是可能只读状态，不改。

pure = view，把view转成pure，pure的本意是绝对不读状态，与view的定义相矛盾，所以不能把view转成pure。

---

在 Solidity 中，你**不能**定义一个带 `calldata` 参数的外部函数指针。

```solidity
// ❌ 编译报错！
// 虽然这种函数在合约里满大街都是，但你不能这样定义指针类型
function (string calldata) external ptr; 

// ✅ 必须定义为 memory
function (string memory) external ptr; 
```

`function (string memory) external` 可以同时指向 `calldata` 和 `memory` 两种实现：

```solidity
contract Target {
    // 实现 A: 使用 memory 接收
    function f(string memory _s) external pure {}
    
    // 实现 B: 使用 calldata 接收（更省 Gas）
    function g(string calldata _s) external pure {}
}

contract Caller {
    // 定义一个指针类型，必须用 memory
    function (string memory) external myPtr;

    function test(address _addr) public {
        Target t = Target(_addr);

        // 1. 指向 f (OK)
        myPtr = t.f;
        myPtr("hello");

        // 2. 指向 g (也 OK！)
        // 虽然 g 定义的是 calldata，但因为 ABI 编码一致，
        // 这里的 memory 指针可以完美兼容
        myPtr = t.g;
        myPtr("world");
    }
}
```



## **Solidity 外部函数变量在 ABI 编码（ABI Encoding）时的底层表现**。

简单来说：当你把一个 `external function` 类型的变量传递给合约之外（比如传给 JavaScript 前端，或者通过 `abi.encode` 序列化）时，它不再是一个抽象的“函数”，而是一个 **24 字节长的二进制数据**。

---

1. 为什么是 24 字节？

一个外部函数要被成功调用，必须知道两件事：

1. **去哪找？** —— 合约地址（Address）：**20 字节**。

2. **找谁？** —— 函数签名选择器（Selector）：**4 字节**。

这两者拼在一起：$20 + 4 = 24$ 字节。

---

2. 举个合约例子

我们通过 `abi.encode` 来“解剖”这个变量：

Solidity

```
contract Target {
    function test(uint a) external pure returns (uint) {
        return a;
    }
}

contract Decoder {
    function getRawData() public view returns (bytes memory) {
        // 1. 获取一个外部函数指针
        Target t = new Target();
        function(uint) external returns (uint) ptr = t.test;

        // 2. 将其编码为原始字节
        // 在 Solidity 内部，ptr 是函数类型
        // 但被 abi.encode 之后，它就变成了 bytes24
        return abi.encode(ptr);
    }
}
```

如果你运行 `getRawData()`，你会得到一段类似这样的数据：

`0x000000000000000000000000d9145ece52d3344019e28e8c3241c8a03890a210668d2777...`

**拆解这段 bytes32（ABI 会补位到 32 字节）：**

- 前 20 字节：`d9145ece52d3344019e28e8c3241c8a03890a210`（这是 Target 合约的**地址**）。

- 接下来的 4 字节：`668d2777`（这是 `test(uint256)` 的**函数选择器**）。

- 最后 8 字节：`0000000000000000`（ABI 编码填充的零）。

---

3. 在“Solidity 上下文之外”是什么意思？

这意味着当你在 **JavaScript (ethers.js / web3.js)** 里调用这个返回函数类型的接口时，或者在 **底层汇编 (Yul/Assembly)** 里操作它时，它就是一个普通的 `bytes24`。

**JavaScript 视角：**

如果你在前端通过 ethers.js 调用上面的 `getRawData`：

JavaScript

```
const data = await contract.getRawData();
console.log(data); 
// 输出: "0xd9145ece52d3344019e28e8c3241c8a03890a210668d2777" 
// (一个 24 字节的十六进制字符串)
```

前端并不能直接像调用函数一样 `data()`。前端需要自己拆分：前 40 位是地址，后 8 位是 Method ID。

---

## 示例代码的一点疑问

1. `new Target()` 到底干了什么？

当你在 `Decoder` 合约的代码里写下 `Target t = new Target();` 时，会发生以下一连串动作：

1. **打包字节码**：编译器在编译 `Decoder` 时，会把 `Target` 的完整 **Creation Bytecode（创建字节码）** 像“压缩包”一样塞进 `Decoder` 的字节码里。

2. **发起部署指令**：当这行代码在链上执行时，`Decoder` 合约会向 EVM 发出一个特殊的指令（`CREATE` 指令）。

3. **计算新地址**：EVM 会根据 `Decoder` 的地址和它当时的 `nonce`（交易计数），计算出一个**属于新 Target 合约的地址**。

4. **运行构造函数**：EVM 会在新地址上运行 `Target` 的构造函数，并把 `Target` 的代码存入该地址。

5. **返回引用**：这行代码最后会把这个**新地址**返回给变量 `t`。

**所以，`t` 本质上就是一个地址。**

---

2. “如果要用到代码，不是要先得到地址吗？”

你的直觉是对的：**调用合约必须有地址。** 但在 Solidity 中，获取地址有两种完全不同的方式，取决于你想用“哪一个” Target：

**情况 A：你想用一个“已经存在”的合约**

如果 `Target` 已经在链上部署好了（地址是 `0x123...`），你不需要 `new`，你只需要“强制转型”：

Solidity

```
// 告诉 Solidity：在这个地址上有一个 Target 合约，请按 Target 的 ABI 跟我交互
Target t = Target(0x123...地址); 
t.test(100); // 正常调用
```

 **情况 B：你想“现造一个”全新的合约**

这就是你看到的 `new Target()`。

- **优点**：`Decoder` 拥有这个新 Target 的“绝对控制权”（它是创建者）。

- **缺点**：非常费 Gas，因为你在交易过程中执行了“部署”这个重体力劳动。



## 合约更新时的稳定性

在使用函数类型的值时，一个需要考虑的重要方面是：如果底层代码发生了变化，该值是否仍然有效。

区块链的状态并非完全不可变，有多种方法可以在同一个地址下放置不同的代码：

1. 使用 **加盐合约创建 (Salted contract creation)** 直接部署不同的代码（即 `CREATE2`）。

2. 通过 **DELEGATECALL** 委托给不同的合约（代理模式下的可升级合约是常见例子）。

3. **EIP-7702** 定义的账户抽象。

**内部函数变量（Internal Function Variable）** 只能指向以下范围内的函数：

- 当前合约内定义的函数。

- 继承的父合约中的函数。

- 库（Library）里的函数或“自由函数”（Free Functions，即写在 `contract` 块之外的函数）。

**它绝对不能指向其他合约的函数。**

> **底层逻辑**：因为内部调用（Internal Call）在 EVM 层面是一个 `JUMP` 指令（就像在自己的书里翻页）。你没法 `JUMP` 到别人的书里去，去别人家必须用 `CALL` 指令（外部调用）。

---

为什么“外部函数”可以引用别人的？

当你定义 `function(...) external ptr` 时，它存的是“地址+签名”。

- 你可以写 `ptr = otherContract.someFunction;`

- 这就好比你把别人的名片存在了你的名片夹里，以后你想找他，拨个号（发起外部调用）就行了。

---

如果强行给内部变量赋值“别人的函数”会怎样？

```solidity
contract Other {
    function add(uint x) internal pure returns (uint) { return x + 1; }
}

contract Main {
    function (uint) internal myPtr;

    function test(address _addr) public {
        Other o = Other(_addr);
        // ❌ 编译报错！
        // 因为 Other 里的 add 是 internal 的，Main 根本看不见它。
        // myPtr = o.add; 
    }
}
```

即使 `Other` 里的函数是 `public` 的，你也没法把它赋值给 `internal` 变量：

```solidity
contract Other {
    function add(uint x) public pure returns (uint) { return x + 1; }
}

contract Main {
    function (uint) internal myPtr;

    function test(address _addr) public {
        Other o = Other(_addr);
        // ❌ 依然报错！
        // 类型不匹配：一个是外部引用（名片），一个是内部跳转（行号）。
        // myPtr = o.add; 
    }
}
```

---

# 合约升级的一点疑问

Proxy代理模式。实际应用中，代理合约不会定义Slot，只存一个逻辑合约地址。

## 1. 代理合约 A 到底长什么样？

在实际的工业开发中，**代理合约 A 几乎是一个“空壳”**。它内部通常**不显式写出** `owner`、`balance` 这些业务变量。

如果我们把合约比作人体：

- **代理合约 A** 是“肉体”（提供存放血液、器官的**空间/槽位**）。

- **逻辑合约 B** 是“灵魂/大脑”（提供操作这些器官的**指令**）。

**真实的代理合约 A 代码片段：**

Solidity

```
contract Proxy {
    // A 唯一需要显式存的，就是逻辑合约 B 的地址
    // 但为了不和 B 的业务变量打架，它存在一个极其偏僻的 Slot 里（EIP-1967）
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    // A 没有任何业务变量（没有 owner, 没有 balance）
    // 所有的调用都通过 fallback() 转发给 B
    fallback() external payable {
        _delegate(StorageSlot.getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }
}
```

---

## 2. 为什么只在 B 里加变量？

记住：**虽然变量定义在 B 里，但数据实际存在 A 的插槽（Slot）里。**

当你需要升级时，你的操作步骤是：

1. **写 B_V2**：在 B 的变量列表最后，增加一个 `uint256 public newVar;`。

2. **部署 B_V2**：这会产生一个新的地址，但你不需要告诉用户这个地址。

3. **更新 A 的引用**：调用 A 的管理函数，把存逻辑地址的那个偏僻 Slot 改成 B_V2 的地址。

#### **发生了什么？**

- 当用户访问 **合约 A（地址永远不变）** 调用 `setNewVar()` 时。

- A 通过 `delegatecall` 把 B_V2 的代码拿过来运行。

- B_V2 说：“请把数据存到我的 Slot 2 里”。

- 于是，数据被存进了 **A 的物理存储 Slot 2**。

**结论：** 代理合约 A 就像是一个**通用的储物柜**，它不需要知道里面装的是金条还是银条。只要逻辑合约 B 知道哪层抽屉（Slot）放的是什么，A 就能配合。

<br />


