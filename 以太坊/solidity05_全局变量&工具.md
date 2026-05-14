- [Ether Units](#ether-units)
- [Time Units](#time-units)
- [Block和Transaction](#block和transaction)
  - [开发者必须掌握的“坑”](#开发者必须掌握的坑)
  - [为什么合约可以拿到块的消息?](#为什么合约可以拿到块的消息)
- [ABI Encoding和Decoding](#abi-encoding和decoding)
- [错误处理（Error Handling）](#错误处理error-handling)
- [数学与加密函数](#数学与加密函数)
  - [ecrecover警告](#ecrecover警告)
  - [深度分析：开发者必须掌握的细节](#深度分析开发者必须掌握的细节)
    - [A. `addmod` 和 `mulmod` 的特殊性](#a-addmod-和-mulmod-的特殊性)
    - [B. `keccak256`：以太坊的灵魂](#b-keccak256以太坊的灵魂)
    - [C. `ecrecover` 与签名安全 (核心考点)](#c-ecrecover-与签名安全-核心考点)
    - [D. ERC-7201：存储冲突的终结者](#d-erc-7201存储冲突的终结者)
    - [代码演练：模拟签名校验](#代码演练模拟签名校验)
    - [ecrecover 的签名可变性(Signature Malleability)攻击案例](#ecrecover-的签名可变性signature-malleability攻击案例)
    - [如何防护](#如何防护)
- [合约相关](#合约相关)
- [类型信息](#类型信息)
  - [合约\*\*`type(I).interfaceId`\*\*：包含接口 `I` 的 **EIP-165** 接口标识符（`bytes4`）。该标识符定义为接口内定义的全部函数选择器的异或（XOR）结果（不包括继承的函数）。](#合约typeiinterfaceid包含接口-i-的-eip-165-接口标识符bytes4该标识符定义为接口内定义的全部函数选择器的异或xor结果不包括继承的函数)
  - [接口](#接口)
    - [1. 它是怎么算出来的？（技术本质）](#1-它是怎么算出来的技术本质)
    - [2.代码用法示例](#2代码用法示例)





# Ether Units

**以太币单位（Ether Units）** 数字字面量可以使用 `wei`、`gwei` 或 `ether` 作为后缀来指定以太币的单位。没有后缀的数字默认为 `wei`。

- `assert(1 wei == 1);`

- `assert(1 gwei == 1e9);`

- `assert(1 ether == 1e18);`

# Time Units

在数字字面量后使用 `seconds`、`minutes`、`hours`、`days` 和 `weeks` 后缀可以指定时间单位。`seconds` 是基本单位，单位换算遵循以下规则：

- `1 == 1 seconds`

- `1 minutes == 60 seconds`

- `1 hours == 60 minutes`

- `1 days == 24 hours`

- `1 weeks == 7 days`

使用这些单位进行日历计算时请务必小心，因为由于**闰秒**的存在，并非每一年都是 365 天，甚至并非每一天都是 24 小时。由于闰秒无法预测，精确的日历库必须由外部预言机（Oracle）进行更新。

- **❌ 错误写法：** `uint t = 5; uint duration = t days;` （编译器会报错）

- **✅ 正确写法：** `uint t = 5; uint duration = t * 1 days;`



# Block和Transaction

在全局命名空间中始终存在一些特殊的变量和函数，主要用于提供区块链信息或作为通用的工具函数。

**区块与交易属性**

- `blockhash(uint blockNumber) returns (bytes32)`：指定区块的哈希值——仅限最近的 256 个区块；否则返回零。

- `blobhash(uint index) returns (bytes32)`：当前交易关联的第 `index` 个 blob 的版本化哈希 (EIP-4844)。

- `block.basefee (uint)`：当前区块的基础费用。

- `block.blobbasefee (uint)`：当前区块的 blob 基础费用。

- `block.chainid (uint)`：当前链的 ID。

- `block.coinbase (address payable)`：当前区块矿工的地址。

- `block.difficulty (uint)`：当前区块难度（在 Paris 升级之前的 EVM 版本）。对于后续版本，它是 `block.prevrandao` 的过时别名。

- `block.gaslimit (uint)`：当前区块的 Gas 限额。

- `block.number (uint)`：当前区块号。

- `block.prevrandao (uint)`：由信标链提供的随机数（EVM >= Paris）。

- `block.timestamp (uint)`：当前区块的时间戳（自 Unix 纪元以来的秒数）。

- `gasleft() returns (uint256)`：剩余的 Gas。

- `msg.data (bytes calldata)`：完整的调用数据（calldata）。

- `msg.sender (address)`：消息发送者（当前调用者）。

- `msg.sig (bytes4)`：调用数据的前四个字节（即函数标识符）。

- `msg.value (uint)`：随消息发送的 wei 数量。

- `tx.gasprice (uint)`：交易的 Gas 价格。

- `tx.origin (address)`：交易的原始发送者（完整的调用链源头）。

## 开发者必须掌握的“坑”

这一段文档的信息密度极高，以下是四个最关键的分析点：

**A. `msg.sender` vs `tx.origin`（安全重点！）**

这是 Solidity 开发中最容易混淆且最具危险性的地方。

- **`msg.sender`** 是“谁直接调了我的函数”。

- **`tx.origin`** 是“最初发起这笔交易的人”。

> **案例：** 如果你写 `require(tx.origin == owner)`，黑客可以诱导你调用一个恶意合约 A，合约 A 再调用你的合约。此时 `msg.sender` 是合约 A，但 `tx.origin` 依然是你。合约 A 就能绕过检查掏空你的钱包。**结论：几乎永远不要用 `tx.origin` 做鉴权。**

**B. 随机数的致命陷阱**

文档明确警告不要用 `blockhash` 或 `block.timestamp` 做随机数。

- **原因：** 矿工可以预先计算结果。如果结果对他不利（比如没中奖），他可以选择不打包这个区块，或者在同一个区块内操纵交易顺序。

- **解决方案：** 如果需要真正的随机数，必须使用 **Chainlink VRF**（可验证随机函数）等链外预言机。

**C. `blockhash` 的 256 个区块限制**

很多新手想实现“开奖日期在 1000 个区块后”的功能，并试图用 `blockhash(futureBlock)`。

- **真相：** 到了 1000 个区块后，你只能查到它最近 256 个块的哈希。如果你太晚去“领奖”，`blockhash` 会变成 `0`。这意味着你的逻辑可能会因为返回值为 0 而失效或被利用。

**D. Paris 升级（以太坊合并）的影响**

文档提到了 `block.difficulty` 的变迁。

- 在以太坊合并（The Merge / Paris 升级）后，PoW 的“难度”概念消失了。

- 现在 `block.prevrandao` 取而代之，提供了一个由信标链产生的混合值。虽然比以前更难操纵，但它依然不是完全不可预测的随机数。



## 为什么合约可以拿到块的消息?

用户发起交易，交易入池，然后矿工拿到交易，执行EVM。这个时候是已经产生块了么？

这是一个非常深刻的问题！你触及了区块链底层最核心的运行机制：**状态预判与共识**。

简单直接的回答是：**当 EVM 执行你的交易时，矿工（在 PoS 中称为验证者）正在“试运行”构造一个新的区块。**

我们可以把这个过程拆解开来看：

1. 块信息从哪来？

你问“是不是已经产生块了”，准确地说：**块正在被打包的过程中。**

当矿工从交易池（Mempool）捞起你的交易并准备放入正在构造的“候选块”时，他必须先确定这个块的基础信息：

- **块号 (`block.number`)：** 必然是当前最新块高度 + 1。

- **时间戳 (`block.timestamp`)：** 矿工根据自己的本地时钟设置一个值（必须大于上一个块的时间）。

- **难度/随机数 (`prevrandao`)：** 根据共识协议确定的值。

**EVM 执行时，这些值已经作为环境参数（Environment Variables）注入进去了。** 你的合约代码就像是在读取一个“预填好的表格”。

---

2. 每个矿工产生的块不一样吗？

**是的，完全不一样！** 这正是区块链冲突处理的关键点：

1. **打包内容不同：** 矿工 A 可能选择了交易 1, 2, 3；矿工 B 可能选择了交易 4, 5, 6。

2. **环境参数微差：** 矿工 A 设定的时间戳可能是 `10:00:01`，矿工 B 可能是 `10:00:02`。

3. **结果差异：** 因为 `block.timestamp` 不同，如果你的合约里写了 `if (block.timestamp % 2 == 0)`，那么在矿工 A 的块里你没中奖，在矿工 B 的块里你可能就中奖了。

---

3. 那最后听谁的？（共识机制）

既然每个矿工算的答案可能不一样，为什么区块链账本还能统一？

- **竞争胜出：** 最终只有一个矿工（或验证者）会被协议选中，他的区块会被广播给全网。

- **全网验证：** 其他节点收到这个块后，会按照块里的交易和块信息（比如时间戳）**重新跑一遍 EVM**。

- **强制统一：** 只有当其他节点跑出来的结果（State Root）和这个矿工广播的结果完全一致时，这个块才会被承认。

**结论：** 你的合约拿到的 `block.number` 等信息，其实是那个**最终胜出的矿工**所定义的块信息。

---

4. 为什么这对开发者很重要？

正因为矿工可以“微调”这些块信息（比如把时间戳往后挪几秒，或者决定把哪笔交易排在前面），所以文档才会警告你：

> **不要依赖 `block.timestamp` 做核心随机数。**

因为矿工可以先在本地“试跑”你的交易。如果发现当前时间戳 `10:00:01` 跑出来的结果对他不利（比如他没中奖），他可以把时间戳改成 `10:00:05` 再跑一次，直到跑出对自己有利的结果再发布区块。这就是所谓的 **MEV（矿工可提取价值）** 攻击的一种。

<br />

# ABI Encoding和Decoding

**ABI 编码与解码函数**

- `abi.decode(bytes memory encodedData, (...)) returns (...)`：对给定数据进行 ABI 解码。类型在括号中作为第二个参数给出。示例：`(uint a, uint[2] memory b, bytes memory c) = abi.decode(data, (uint, uint[2], bytes))`。

- `abi.encode(...) returns (bytes memory)`：对给定的参数进行 ABI 编码。

- `abi.encodePacked(...) returns (bytes memory)`：对给定参数进行**紧凑编码（Packed Encoding）**。注意：紧凑编码可能会产生歧义！

- `abi.encodeWithSelector(bytes4 selector, ...) returns (bytes memory)`：从第二个参数开始进行 ABI 编码，并在前面加上给定的 4 字节选择器（selector）。

- `abi.encodeWithSignature(string memory signature, ...) returns (bytes memory)`：等同于 `abi.encodeWithSelector(bytes4(keccak256(bytes(signature))), ...)`。

- `abi.encodeCall(function functionPointer, (...)) returns (bytes memory)`：对函数指针及其参数进行 ABI 编码。它会执行全量类型检查，确保类型与函数签名匹配。结果等同于 `abi.encodeWithSelector(functionPointer.selector, (...))`。

> **注意：** 这些编码函数可用于在不实际调用外部函数的情况下，构造外部调用的数据。此外，`keccak256(abi.encodePacked(a, b))` 是计算结构化数据哈希的一种方法（但要注意，使用不同的参数类型可能会构造出“哈希碰撞”）。

---

**深度分析：开发者必须避开的“雷区”**

这部分函数是用来将 Solidity 的变量转换成字节码（`bytes`），以便在网络上传输。

**A. `abi.encode` vs `abi.encodePacked` (最重要区别)**

这是面试和开发中最常问的问题：

- **`abi.encode` (标准编码)：** 遵循 ABI 规范，会给每个参数填充零到 32 字节。这种方式**非常安全**，不会有歧义，是跨合约调用的标准。

- **`abi.encodePacked` (紧凑编码)：** 只保留数据的原始字节，不填充零。
  
  - **优点：** 节省空间。
  
  - **缺陷 (哈希碰撞)：** 比如 `abi.encodePacked("a", "bc")` 和 `abi.encodePacked("ab", "c")` 结果是一模一样的（都是 `0x616263`）。如果你用它做签名验证，黑客可以利用这个特性伪造签名。

**B. `abi.encodeWithSignature` vs `abi.encodeCall`**

- **`encodeWithSignature`：** 传入字符串如 `"transfer(address,uint256)"`。**缺点是没类型检查**。如果你拼错了单词，编译器不会报错，直到运行时才会失败。

- **`encodeCall` (推荐)：** 这是 0.8.11 引入的。它直接传入函数名（如 `token.transfer`）。如果你传的参数类型不对，**编译阶段就会报错**。这是目前最安全的做法。

**C. `abi.decode` 的妙用**

当你通过 `low-level call` 收到一段 `bytes` 数据时，你需要知道它的结构才能还原数据。这就像是把 JSON 字符串转回对象。

---

**代码示例：如何构造数据**

假设我们要手动构造一个调用另一个合约函数的 Data 字段：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Encoder {
    // 假设我们要调用：transfer(address _to, uint256 _amount)

    // 方式 1：使用 Signature（容易拼错）
    function withSig(address _to, uint256 _amount) public pure returns (bytes memory) {
        return abi.encodeWithSignature("transfer(address,uint256)", _to, _amount);
    }

    // 方式 2：使用 Selector（更底层）
    function withSelector(address _to, uint256 _amount) public pure returns (bytes memory) {
        // bytes4(keccak256("transfer(address,uint256)")) == 0xa9059cbb
        return abi.encodeWithSelector(0xa9059cbb, _to, _amount);
    }

    // 方式 3：计算哈希时的 Packed (注意碰撞)
    function getHash(string memory _a, string memory _b) public pure returns (bytes32) {
        // 通常用于签名验证
        return keccak256(abi.encodePacked(_a, _b));
    }
}
```

**为什么文档提到“哈希碰撞”？**

如果你有两个动态类型（比如两个 `string` 或 `bytes`）连在一起放在 `encodePacked` 里：

1. `abi.encodePacked("AA", "BBB")` -> `AABBB`

2. `abi.encodePacked("AAB", "BB")` -> `AABBB`

两者的哈希值是一样的！这就是为什么文档警告你：**如果你的哈希计算涉及多个动态类型，请使用 `abi.encode` 而不是 `abi.encodePacked`。**

<br />

# 错误处理（Error Handling）

require、assert、revert。

**用处区别：**

- **`require` (预检)：** 检查**外部输入**。比如：用户的余额够不够？传进来的参数对不对？这是正常的业务逻辑判断。

- **`revert` (主动拦截)：** 和 `require` 本质一样，但用于更复杂的逻辑判断（比如 `if...else` 嵌套）。

- **`assert` (崩溃检查)：** 检查**代码内部错误**。如果它触发了，说明你的合约代码逻辑“坏了”，出现了本该绝对不可能发生的情况（比如：扣款后账户余额反而变多了）。

---

**实现区别：**

| **特性**     | **require / revert** | **assert**                                                         |
| ---------- | -------------------- | ------------------------------------------------------------------ |
| **底层操作码**  | `REVERT` (0xfd)      | `INVALID` (0xfe) -> **0.8.0 之前**<br>**`Panic` (0x4e) -> 0.8.0 之后** |
| **Gas 消耗** | **退还** 剩余 Gas        | **退还** 剩余 Gas (0.8.0+)                                             |
| **语义错误类型** | 业务逻辑错误 (Error)       | 内部致命错误 (Panic)                                                     |
| **是否支持消息** | 支持字符串或自定义 Error      | 不支持消息，仅返回错误码                                                       |

---

**性能区别：**

虽然 `require` 退还 Gas，但它携带的**字符串消息**（如 `"Exceeds max supply"`）是需要存储在部署后的字节码里的。字符串越长，部署越贵；触发时消耗的 Gas 也越多。

Solidity 0.8.4+ 推荐写法：

```solidity
error NotEnoughBalance(uint requested, uint available);

function withdraw(uint amount) public {
    if (amount > balance) {
        // 这种方式比 require(..., "string") 省很多 Gas！
        revert NotEnoughBalance(amount, balance);
    }
}
```

assert不如revert有详细的报错信息，对于调用方也更友好。

---

**代码示例：**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GasTest {
    uint public constant MAX_SUPPLY = 100;
    uint public totalSupply;

    function mint(uint _amount) public {
        // 1. 使用 require：检查外部输入。
        // 如果失败，退还 Gas，提示信息。
        require(_amount > 0, "Amount must > 0");
        require(totalSupply + _amount <= MAX_SUPPLY, "Exceeds max supply");

        totalSupply += _amount;

        // 2. 使用 assert：检查内部不变式（Invariants）。
        // 理论上 totalSupply 永远不该超过 100。
        // 如果这里触发了，说明我上面的逻辑（或者合约其他地方）写错了。
        assert(totalSupply <= MAX_SUPPLY);
    }

    function manualRevert(uint _val) public pure {
        // 3. 使用 revert：适合逻辑复杂的场景。
        if (_val % 2 != 0) {
            // 这里可以做很多复杂的逻辑后再决定回滚
            revert("Only even numbers allowed");
        }
    }
}
```

<br />

# 数学与加密函数

Mathematical and Cryptographic Functions

- `addmod(uint x, uint y, uint k) returns (uint)`：计算 `(x + y) % k`。加法过程具有任意精度，**不会**在 $2^{256}$ 处溢出。从 0.5.0 版本开始，断言 `k != 0`。

- `mulmod(uint x, uint y, uint k) returns (uint)`：计算 `(x * y) % k`。乘法过程具有任意精度，**不会**在 $2^{256}$ 处溢出。从 0.5.0 版本开始，断言 `k != 0`。

- `keccak256(bytes memory) returns (bytes32)`：计算输入的 Keccak-256 哈希值。
  
  - *注意：* 曾有的别名 `sha3` 已在 0.5.0 版本移除。

- `sha256(bytes memory) returns (bytes32)`：计算输入值的 SHA-256 哈希。

- `ripemd160(bytes memory) returns (bytes20)`：计算输入值的 RIPEMD-160 哈希。

- `ecrecover(bytes32 hash, uint8 v, bytes32 r, bytes32 s) returns (address)`：从椭圆曲线签名中恢复与公钥关联的地址，出错则返回零。参数对应 ECDSA 签名的值：`r`（前32字节）、`s`（后32字节）、`v`（最后1字节）。返回值为 `address` 而非 `address payable`。



## ecrecover警告

使用 `ecrecover` 时需注意**签名可变性（Signature Malleability）**问题：在不需要私钥的情况下，一个有效的签名可以被转化为另一个不同的有效签名。如果你的业务逻辑依赖签名唯一性，请使用 OpenZeppelin 的 ECDSA 库。



## 深度分析：开发者必须掌握的细节

### A. `addmod` 和 `mulmod` 的特殊性

在普通的 Solidity 运算中，如果你计算 `(x * y) % k`，如果 `x * y` 的结果超过了 $2^{256}-1$，它会先溢出截断，再取模，结果就会错误。

- **黑科技：** 这两个函数在 EVM 内部会使用**双倍精度**（512位）来保存中间结果，因此**绝对不会溢出**。这在处理大数运算、加密算法或 DeFi 协议的精确份额计算时非常重要。

### B. `keccak256`：以太坊的灵魂

虽然它常被称为 SHA-3，但它与官方标准的 FIPS 202 SHA-3 有细微差别，因此在 Solidity 中必须叫 `keccak256`。

- 用途: 几乎所有地方——计算函数选择器、生成唯一的 ID、在映射（Mapping）中定位槽位、以及构建 Merkle Tree。

- 配合: 通常配合 `abi.encodePacked` 使用：`keccak256(abi.encodePacked(a, b))`。

### C. `ecrecover` 与签名安全 (核心考点)

这是实现“元交易”（Meta-transaction）和白名单签名的核心。但文档给出了严重的警告：

1. **签名可变性：** 椭圆曲线具有对称性，对于任何有效的 $(r, s, v)$，都存在另一个合法的 $s'$ 值能通过验证。这意味着同一个签名可以被“改变外貌”后重复提交。
   
   - **对策：** 永远不要用签名原文作为“已使用”的唯一标识符（Nonce），或者使用 OpenZeppelin 封装好的库，它会限制 $s$ 的取值范围。

2. **返回值为 0：** 如果签名无效，`ecrecover` 不会报错，而是返回 `address(0)`。
   
   - **致命错误：** 如果你的代码里写 `require(ecrecover(...) == msg.sender)`，而 `msg.sender` 恰好也是 0（虽然这在现实中几乎不可能，但逻辑上不严谨），就会导致校验绕过。

### D. ERC-7201：存储冲突的终结者

这是相对较新的标准。在编写可升级合约（Proxy Contracts）时，为了防止新旧逻辑合约的存储变量位置冲突，开发者会将变量存在一个极其偏远的、通过 `erc7201("id")` 计算出来的随机位置。这保证了逻辑扩展时的安全性。

### 代码演练：模拟签名校验

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CryptoTool {
    // 1. 安全的大数模乘
    function safeMultiply(uint x, uint y, uint k) public pure returns (uint) {
        return mulmod(x, y, k); // 不用担心 x*y 超过 2^256
    }

    // 2. 获取数据的哈希
    function getHash(string memory _text) public pure returns (bytes32) {
        return keccak256(abi.encodePacked(_text));
    }

    // 3. 简单的签名恢复示例
    function verify(bytes32 _hash, uint8 _v, bytes32 _r, bytes32 _s) public pure returns (address) {
        address signer = ecrecover(_hash, _v, _r, _s);
        // 必须检查返回地址不是 0
        require(signer != address(0), "Invalid signature");
        return signer;
    }
}
```

### ecrecover 的签名可变性(Signature Malleability)攻击案例

将签名本身作为“唯一标识符”就会有问题。

用户 A 想给用户 B 转 100 元，但 A 不想付 Gas 费，于是 A 线下签了一个名发给 B。B 拿到签名后调用合约取钱，由 B 支付 Gas 费。

```solidity
// ❌ 极度危险的代码
function claim(uint256 amount, bytes memory signature) public {
    require(!usedSignatures[signature], "Signature replayed"); // 这里的 Key 是签名原文
    
    address signer = recover(signature); 
    require(signer == admin, "Invalid signer");

    // ... 执行转账
    usedSignatures[signature] = true; 
}
```

- **攻击：** B 第一次用 $(r, s, v)$ 领了钱。然后利用可变性构造出 $(r, s_{new}, v_{new})$。

- **结果：** 此时 `signature` 字节变了，`usedSignatures` 里的 Key 变了，校验绕过！但 `ecrecover` 算出来的地址还是 admin。**双重支付成功。**

### 如何防护

**方法 A：限定 $s$ 的取值范围 (OpenZeppelin 方案)**

以太坊后来规定，合法的 $s$ 必须在曲线阶数的一半以下（Lower-$s$）。如果合约检测到 $s$ 在高位区间，直接拒绝。

```solidity
// 在 ecrecover 之前检查 s
require(uint256(s) <= 0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0, "Invalid s value");
```

**方法 B：使用 Nonce（随机数/计数器）—— 最推荐**

不要用签名本身的哈希来判断是否使用过，而是给每个用户存一个 `nonce`。

```solidity
mapping(address => uint256) public nonces;

function claim(uint256 amount, uint256 nonce, uint8 v, bytes32 r, bytes32 s) public {
    // 1. 将 nonce 放入哈希计算
    bytes32 hash = keccak256(abi.encodePacked(msg.sender, amount, nonce, address(this)));

    // 2. 校验签名
    address signer = ecrecover(hash, v, r, s);
    require(signer == admin, "Invalid signer");

    // 3. 校验 nonce 并自增，这彻底封死了重放可能
    require(nonce == nonces[msg.sender], "Invalid nonce");
    nonces[msg.sender]++; 

    payable(msg.sender).transfer(amount);
}
```

<br />

# 合约相关

- **`this` (当前合约类型)**：指代当前的合约，可以显式转换为 `address`。

- **`super`**：继承层次结构中高一级的合约。

- **`selfdestruct(address payable recipient)`**：销毁当前合约，将其资金发送到指定的地址并结束执行。注意 `selfdestruct` 具有一些继承自 EVM 的特性：
  
  - 接收合约的 `receive` 函数不会被执行（强制转账）。
  
  - 合约仅在交易结束时才真正被销毁，且 `revert` 可能会“撤销”销毁操作。
  
  - 此外，当前合约的所有函数都可以直接调用，包括当前函数。

> **警告：** 从 **EVM >= Cancun（坎昆升级）** 开始，`selfdestruct` 将**仅**向接收者发送账户中的所有以太币，而**不会销毁合约**。然而，如果在创建合约的同一次交易中调用 `selfdestruct`，则保留 Cancun 升级前的行为（即销毁合约、删除数据、代码和账户）。详见 EIP-6780。

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.18;

contract Vulnerable {
    address public owner;

    constructor() {
        owner = msg.sender;
    }

    // 在旧版本，这个函数会删除合约
    // 在新版本 (Cancun+), 钱转走了，合约还在，owner 还是原来的
    function kill() public {
        require(msg.sender == owner, "Not owner");
        selfdestruct(payable(owner));
    }

    // 依然可以被调用！即使执行过 kill()
    function stillAlive() public pure returns (string memory) {
        return "I am still here even after selfdestruct!";
    }
}
```

<br />

# 类型信息

`type(X)` 允许你在编写代码时，像“照镜子”一样获取某个类型在编译器眼中的元数据。目前该功能支持有限(**X只可以是合约或整数类型)**，但未来可能会扩展。

## 合约**`type(I).interfaceId`**：包含接口 `I` 的 **EIP-165** 接口标识符（`bytes4`）。该标识符定义为接口内定义的全部函数选择器的异或（XOR）结果（不包括继承的函数）。

- **`type(C).name`**：合约的名字。

- **`type(C).creationCode`**：包含合约创建字节码（Creation Bytecode）的内存字节数组。常用于内联汇编构建自定义部署逻辑（特别是配合 `create2`）。注意：该属性不能在合约自身或其派生合约中访问，因为它会将字节码嵌入到调用点，从而导致循环引用。
  
  1. 在以太坊中，当你部署一个合约时，你发送的交易数据其实是 `creationCode`。EVM 执行这段代码，运行构造函数（Constructor），最后将生成的 `runtimeCode` 存入区块链状态中。
  
  2. 你不能在合约 `A` 里面写 `type(A).creationCode`。为什么？因为 `creationCode` 包含了合约的所有代码。如果你在代码里引用了 `creationCode`，那么 `creationCode` 就会变大（因为它包含了自己），这就陷入了无限递归。所以，你只能在一个合约里获取**另一个**合约的字节码。
  
  3. 代码示例：
     
     ```solidity
     function deployOther(bytes32 salt) public {
         bytes memory bytecode = type(OtherContract).creationCode;
         // 使用 create2 部署，地址是可预测的
         assembly {
             let addr := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
         }
     }
     ```

- **`type(C).runtimeCode`**：包含合约运行时字节码（Runtime Bytecode）的内存字节数组。这是合约部署后实际存在于链上的代码。同样的循环引用限制也适用于此属性。

## 接口

**`type(I).interfaceId`**：包含接口 `I` 的 **EIP-165** 接口标识符（`bytes4`）。该标识符定义为接口内定义的全部函数选择器的异或（XOR）结果（不包括继承的函数）。

### 1. 它是怎么算出来的？（技术本质）

`interfaceId` 的计算规则非常直观：它是接口中定义的所有函数选择器（Selector）的异或（XOR）结果。

- **函数选择器**：函数签名的哈希前 4 字节。例如 `transfer(address,uint256)` 的选择器是 `0xa9059cbb`。

- **计算公式**：
  
  $$interfaceId = Selector1 \oplus Selector2 \oplus Selector3$

**举个例子：**

假设接口 `IVault` 有两个函数：

1. `deposit()` -> 选择器 `0x12345678`

2. `withdraw()` -> 选择器 `0x87654321`
   
   那么 `type(IVault).interfaceId` 就是 `0x12345678 ^ 0x87654321`。

### 2.代码用法示例

```solidity
interface Ikk {
    function testa(address from, address to) external;
    function ownerOf(uint256 tokenId) external view returns (address);
}

contract A is Ikk {
    // 实现接口定义的函数
    function testa(address from, address to) external override { /* ... */ }
    function ownerOf(uint256 tokenId) external view override returns (address) { /* ... */ }

    // 关键：通过 supportsInterface 告诉外界我支持 Ikk
    function supportsInterface(bytes4 interfaceId) public pure returns (bool) {
        return interfaceId == type(Ikk).interfaceId; 
    }
}
```


