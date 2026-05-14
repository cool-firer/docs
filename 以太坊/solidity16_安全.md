- [私密信息与随机数](#私密信息与随机数)
  - [“Private” 并不隐私](#private-并不隐私)
  - [随机数的中心化陷阱](#随机数的中心化陷阱)
- [](#)
- [重入漏洞 (Reentrancy)](#重入漏洞-reentrancy)
  - [存在严重漏洞的版本（使用 `call`）](#存在严重漏洞的版本使用-call)
  - [为什么“一调 call 就会发钱”？](#为什么一调-call-就会发钱)
  - [攻击的具体时序（微观视角）](#攻击的具体时序微观视角)
  - [正确的防范版本（检查-效果-交互模式）](#正确的防范版本检查-效果-交互模式)
- [Gas 限制与循环 (Gas Limit and Loops)](#gas-限制与循环-gas-limit-and-loops)
- [发送与接收 Ether (Sending and Receiving Ether)](#发送与接收-ether-sending-and-receiving-ether)
- [调用栈深度 (Call Stack Depth)](#调用栈深度-call-stack-depth)
- [授权代理 (Authorized Proxies)](#授权代理-authorized-proxies)
- [核心陷阱分析](#核心陷阱分析)
  - [1. 永远不要用 `address(this).balance` 做精确逻辑](#1-永远不要用-addressthisbalance-做精确逻辑)
  - [2. “拒绝服务”攻击（DoS with Gas Limit）](#2-拒绝服务攻击dos-with-gas-limit)
  - [3. 提款模式（Pull over Push）—— 必考点](#3-提款模式pull-over-push-必考点)
  - [4. 理解 2300 Gas 的局限性](#4-理解-2300-gas-的局限性)
- [tx.origin vs msg.sender：谁在敲门？](#txorigin-vs-msgsender谁在敲门)
- [算术溢出的“静默杀手”](#算术溢出的静默杀手)
- [Mapping 的“僵尸数据”](#mapping-的僵尸数据)
- [故障安全（Circuit Breaker / Pausable）](#故障安全circuit-breaker--pausable)






# 私密信息与随机数

## “Private” 并不隐私

这是初学者最容易犯的错误。在 Solidity 中，`private` 仅表示“其他合约无法访问该变量”，但**链下所有人都能看到**。

- **分析：** 区块链的状态是透明的。任何人都可以通过扫描以太坊节点的存储插槽（Storage Slots）来读取被标记为 `private` 的数据。

- **教训：** 永远不要在合约中存储未加密的密码、API 密钥或敏感个人信息。

## 随机数的中心化陷阱

- **分析：** 如果你使用 `block.timestamp` 或 `block.difficulty` 作为随机数种子，矿工（或现在的验证者/区块构建者）可以在生成区块前预知结果。如果结果对他们不利，他们可以选择不发布该区块，从而操纵随机性。

- **对策：** 现在的工业标准是使用 **Chainlink VRF**（可验证随机函数）。

# <br />

# 重入漏洞 (Reentrancy)

## 存在严重漏洞的版本（使用 `call`）

这是最典型的重入漏洞代码。由于 `call` 默认会转发所有剩余的 gas，攻击者有足够的资源发动攻击。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.2 <0.9.0;

// 此合约包含严重 Bug - 请勿使用
contract Fund {
    /// @dev 记录份额
    mapping(address => uint) shares;

    /// 提取你的份额
    function withdraw() public {
        // 使用 call 进行转账，默认转发所有 gas
        // (bool success,) 是解构赋值，获取转账是否成功的布尔值
        (bool success,) = msg.sender.call{value: shares[msg.sender]}("");

        // 如果转账成功，才清空余额
        if (success)
            shares[msg.sender] = 0;
    }
}
```

**攻击原理图解：**

1. 攻击者调用 `withdraw`。

2. 合约执行 `msg.sender.call` 发钱。

3. 攻击者是恶意合约，在其 `fallback` 函数中**再次调用** `withdraw`。

4. **关键点：** 此时第一次调用的 `shares[msg.sender] = 0` 还没有被执行！

5. 合约认为攻击者还有钱，再次发钱。重复此过程，直到合约被吸干。



## 为什么“一调 call 就会发钱”？

在以太坊中，当你执行 `(bool success, ) = target.call{value: amount}("")` 时，发生了以下原子操作：

1. **资产转移：** EVM 立即从“发起方合约”的余额中扣除 `amount`，并增加到 `target` 地址的余额中。

2. **控制权转移：** 如果 `target` 是一个合约，EVM 会立即暂停当前合约的执行，转而运行 `target` 合约的代码（通常是 `fallback` 或 `receive` 函数）。

**关键点：** 这是一次**同步调用**。原合约的代码会“卡”在 `call` 这一行，等待 `target` 合约的代码执行完毕。

## 攻击的具体时序（微观视角）

我们用有漏洞的代码作为参考，看看到底发生了什么：

```solidity
// 有漏洞的代码片段
(bool success,) = msg.sender.call{value: shares[msg.sender]}(""); // 第 A 行
if (success)
    shares[msg.sender] = 0;                                     // 第 B 行
```

**时序表：**

- **第 1 步：** 攻击者（恶意合约）调用 `withdraw`。

- **第 2 步：** 你的合约运行到 **第 A 行**：
  
  - **瞬间动作：** 钱从你的合约转入了恶意合约。
  
  - **控制权转移：** 你的合约停在第 A 行，开始运行恶意合约的 `fallback`。

- **第 3 步（攻击核心）：** 恶意合约的 `fallback` 函数里又写了一句 `fundContract.withdraw()`。
  
  - **注意：** 此时你的合约还停在刚才那个 `withdraw` 函数的 **第 A 行**，**第 B 行（清零余额）还没被执行！**

- **第 4 步：** 你的合约开启了**第二个** `withdraw` 调用栈。
  
  - 它检查 `shares[msg.sender]`，发现余额还是原来的数（因为清零动作没被执行）。
  
  - 它再次执行 **第 A 行**，又发了一次钱。

- **第 5 步：** 再次进入恶意合约的 `fallback`……以此类推，形成递归。

直到你的合约钱被抽干，或者 Gas 耗尽，递归才会停止，开始逐层返回执行后面的 **第 B 行**。但那时候清零已经没有意义了，因为钱早就被取光了。



## 正确的防范版本（检查-效果-交互模式）

这是官方推荐的标准写法，通过调整代码顺序彻底杜绝重入。

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.6.2 <0.9.0;

contract Fund {
    /// @dev 记录份额
    mapping(address => uint) shares;

    /// 提取你的份额
    function withdraw() public {
        // 1. 检查 (Checks)
        uint share = shares[msg.sender];

        // 2. 效果 (Effects) - 在进行外部交互前，先修改内部状态（账本）
        // 即使对方在此处重入，由于 shares[msg.sender] 已经是 0，后续调用会直接跳过
        shares[msg.sender] = 0;

        // 3. 交互 (Interactions) - 最后进行外部转账
        (bool success, ) = payable(msg.sender).call{value: share}("");

        // 如果转账失败，回滚所有操作（包括上面的余额清零）
        require(success);
    }
}
```

<br />

# Gas 限制与循环 (Gas Limit and Loops)

没有固定迭代次数的循环（例如取决于存储变量的循环）必须谨慎使用：由于**区块 Gas 限制 (Block Gas Limit)**，交易只能消耗特定数量的 Gas。无论是因为正常业务增长还是被恶意操纵，循环次数可能超过限额，导致合约在某一点彻底锁死（无法执行）。这可能不适用于仅读取数据的 `view` 函数，但如果这些函数被其他合约在链上调用，仍会导致整个交易失败。

# 发送与接收 Ether (Sending and Receiving Ether)

- **无法拒绝的接收：** 目前，合约和外部账户（EOA）都无法阻止别人给它们发 Ether。合约可以拒绝普通的“消息调用”，但有两种方法可以强行把 Ether 转入合约而不触发任何函数：**1. 挖矿奖励受益人；2. 使用 `selfdestruct(x)` 销毁合约并强制转账。**

- **接收逻辑：** 合约收到 Ether 时，会执行 `receive` 或 `fallback`。如果没有这两个函数且不是通过上述强制手段，Ether 会被拒绝。

- **Gas 补贴 (Gas Stipend)：** 执行这些函数时，合约通常只有 2300 Gas。这不足以修改存储状态（注意：此限额可能在未来的硬分叉中改变）。

- **call vs transfer/send：** `addr.call{value: x}("")` 会转发所有剩余 Gas，这虽然灵活，但也带来了**重入攻击**的风险。

- **提款模式 (Withdrawal Pattern)：** 强烈建议不要主动“推送（send）”钱给用户，而是让用户自己来“拉取（withdraw）”钱。这样可以防止接收方通过耗尽 Gas 或故意报错来导致你的合约功能瘫痪。

# 调用栈深度 (Call Stack Depth)

外部调用如果超过 1024 层深度就会失败。恶意行为者可以先消耗掉大部分深度再与你的合约交互，强行让你的调用失败。

- *注：由于 Tangerine Whistle 硬分叉引入了 63/64 规则，此类攻击在现代以太坊中已不实际。*

# 授权代理 (Authorized Proxies)

如果你的合约可以根据用户输入调用任意合约（作为代理），用户就能以该“代理合约”的身份行事。**建议：** 即使是代理合约，也不应拥有任何权限（甚至是针对自身的权限）。

---

# 核心陷阱分析

## 1. 永远不要用 `address(this).balance` 做精确逻辑

**陷阱点：** 很多开发者认为“如果我没写 `receive()` 函数，合约余额就不会变”。 **现实：** 攻击者可以通过 `selfdestruct(victim_address)` 强行把 Ether 塞进你的合约。 **安全策略：** 不要依赖合约当前的余额来判断业务逻辑（例如：`require(address(this).balance == 10 ether)`）。应该使用一个内部变量来记录通过正常路径存入的资金。

## 2. “拒绝服务”攻击（DoS with Gas Limit）

**场景：** 假设你有一个分红合约，循环遍历 1000 个用户并发钱。 **风险：**

1. **坏人加入：** 攻击者可以注册大量的地址进入你的名单。

2. **撑爆 Gas：** 当用户数量达到一定规模，遍历循环所需的 Gas 会超过区块上限。

3. **结果：** `distribute()` 函数永远无法执行成功，钱被永久锁死。 **对策：** 永远避免在循环中进行外部调用，改用**提款模式（Pull over Push）**。

## 3. 提款模式（Pull over Push）—— 必考点

这是官方文档多次强调的**最佳实践**。

- **错误写法（Push）：** 合约主动转账给所有人。如果其中一个接收者是恶意合约，且其 `receive()` 函数里写了 `revert()`，那么整个分红交易都会失败，导致后面的人也领不到钱。

- **正确写法（Pull）：** 合约只更新一个账本（例如 `mapping(address => uint) pendingWithdrawals`）。每个用户需要自己调用 `withdraw()` 来领走属于自己的那份钱。

## 4. 理解 2300 Gas 的局限性

`transfer` 和 `send` 赋予接收方的 Gas 只有 2300。

- **能做什么：** 抛出一个事件（Emit Event）。

- **不能做什么：** 修改合约状态变量、调用其他函数、或者写复杂的逻辑。 **注意：** 虽然这能防止重入，但它也导致了如果接收方合约的逻辑稍微复杂一点点（比如需要记个账），转账就会失败。因此，现代开发中 `call` 配合 `ReentrancyGuard`（防重入锁）变得更为流行。

<br />

# tx.origin vs msg.sender：谁在敲门？

这是以太坊中经典的“钓鱼攻击”原理。

- **msg.sender：** 直接调用者。如果 A 调用 B，B 调用 C，在 C 看来，`msg.sender` 是 B。

- **tx.origin：** 交易的最初发起者（必须是外部账户 EOA）。在 C 看来，`tx.origin` 始终是 A。

- **安全结论：** 如果你使用 `tx.origin` 授权，就意味着你信任你所调用的**任何**外部合约，这在 DeFi 世界里是非常危险的。

# 算术溢出的“静默杀手”

在 Solidity 0.8.0 之前，溢出是不会报错的，这导致了早年许多著名的黑客事件。

- **现在的风险：** 虽然现在默认报错，但开发者为了节省 Gas 有时会使用 `unchecked`。如果逻辑写错，依然会造成严重后果。

- **最佳实践：** 除非是在处理已确认安全的循环变量（如 `i++`），否则尽量避免使用 `unchecked`。

# Mapping 的“僵尸数据”

这是由于 Solidity 存储布局（Storage Layout）决定的。

- `mapping` 的数据是分散存储在哈希计算出的位置上的，并没有一个中心列表记录哪些位置存了东西。

- `delete` 操作本质上是把特定位置重置为 0。但对于 `mapping`，EVM 不知道要去哪些位置执行重置。

- **后果：** 就像文档中展示的，如果你不手动清除每一个 key（如果你知道的话），那些数据就会永远留在区块链的存储空间里。

# 故障安全（Circuit Breaker / Pausable）

文档建议加入“紧急开关”。

- **工业实践：** 绝大多数成熟的 DeFi 协议（如 Aave, Uniswap 的某些版本）都会引入 OpenZeppelin 的 `Pausable` 库。

- **权衡：** 这引入了中心化风险（谁来按下开关？），但在代码运行初期，这通常是保护用户资金的必要恶行。


