# 智能合约部署完整流程分析

## 场景：部署 SimpleStorage 合约

---

## 1. 合约源码

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

### 编译产物

```javascript
// ABI (Application Binary Interface)
abi = [
  {
    "inputs": [],
    "name": "retrieve",
    "outputs": [{"internalType": "uint256", "name": "", "type": "uint256"}],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [{"internalType": "uint256", "name": "num", "type": "uint256"}],
    "name": "store",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  }
]

// Bytecode (EVM 字节码)
bytecode = '0x608060405234801561000f575f80fd5b506101438061001d5f395ff3fe...'
```

---

## 2. 部署命令

```javascript
c = web3.eth.contract(abi)
cInstance = c.new({
  data: bytecode,
  gas: 200000,
  gasPrice: 1,
  from: "0xb25b5c941951eeec822f502a2198cd442241b83a"
})
```

---

## 3. 部署过程详解

### Step 1: 构造合约创建交易

#### 3.1 交易结构

```javascript
Transaction {
  from:     "0xb25b5c941951eeec822f502a2198cd442241b83a",  // 部署者地址
  to:       null,  // ← 关键！to 为 null 表示这是合约创建交易
  value:    0,     // 不转账
  gas:      200000,    // Gas Limit
  gasPrice: 1 wei,     // Gas Price
  data:     "0x608060405234801561000f575f80fd...",  // 完整的 bytecode
  nonce:    X,     // 部署者的 nonce（假设为 5）
  v, r, s:  [签名数据]
}
```

**关键点：**
- **`to` 字段为 `null`**：这告诉 EVM 这是一个合约创建交易，而不是普通转账或合约调用。
- **`data` 字段包含 bytecode**：完整的合约字节码。

#### 3.2 部署者账户状态（交易前）

```
Address: 0xb25b5c941951eeec822f502a2198cd442241b83a

Account State:
┌─────────────────────────────────────┐
│ Nonce:       5  ← 将用于计算合约地址 │
│ Balance:     10 ETH                 │
│ StorageRoot: EmptyRoot              │
│ CodeHash:    EmptyHash              │
└─────────────────────────────────────┘
```

---

### Step 2: 计算新合约的地址

**合约地址计算公式（CREATE）：**

```
合约地址 = Keccak256(RLP([sender_address, sender_nonce]))[12:]
```

**具体计算：**

```python
sender = 0xb25b5c941951eeec822f502a2198cd442241b83a
nonce = 5

# RLP 编码
rlp_encoded = RLP([sender, nonce])

# 计算 Keccak256 哈希
hash = Keccak256(rlp_encoded)
# 假设结果为：
# 0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef

# 取后 20 字节（去掉前 12 字节）
contract_address = hash[12:]
# 结果：0x1234567890abcdef1234567890abcdef12345678
```

**示例结果：**
```
新合约地址 = 0x1234567890abcdef1234567890abcdef12345678
```

**注意：**
- 合约地址在交易执行前就可以确定（通过 sender + nonce）。
- 这保证了合约地址的唯一性和可预测性。

---

### Step 3: 交易执行过程（在 EVM 中）

#### 3.1 初始化新合约账户

```
创建新账户：0x1234567890abcdef1234567890abcdef12345678

初始状态：
┌─────────────────────────────────────┐
│ Nonce:       1  ← 合约账户初始 nonce │
│ Balance:     0  ← 没有转账           │
│ StorageRoot: EmptyRoot ← 空存储      │
│ CodeHash:    EmptyHash ← 暂时为空    │
└─────────────────────────────────────┘
```

#### 3.2 执行 Bytecode（构造函数）

**Bytecode 分为两部分：**

```
Bytecode = [Initialization Code] + [Runtime Code]
           ─────────────────────   ──────────────
           部署时执行一次            永久存储，每次调用时执行

完整 Bytecode:
0x608060405234801561000f575f80fd5b506101438061001d5f395ff3fe...
   ─────────────────────────────────   ───────────────────────
   Initialization Code (constructor)   Runtime Code
```

**执行流程：**

1. **EVM 加载 Initialization Code**
   - 执行构造函数逻辑（本例中构造函数为空）
   - 可能初始化一些状态变量
   - 消耗 Gas

2. **构造函数执行完成后，返回 Runtime Code**
   - Runtime Code 是实际要存储在合约账户中的代码
   - 包含 `set()` 和 `get()` 函数的实现

3. **EVM 将 Runtime Code 存储到新合约账户**

#### 3.3 存储 Runtime Code

```
Runtime Code (提取出来):
0x608060405234801561000f575f80fd5b5060043610610034575f3560e01c806360fe47b1146100385780636d4ce63c14610054575b5f80fd5b610052600480360381019061004d91906100ba565b610072565b005b61005c61007b565b60405161006991906100f4565b60405180910390f35b805f8190555050565b5f8054905090565b5f80fd5b5f819050919050565b61009981610087565b81146100a3575f80fd5b50565b5f813590506100b481610090565b92915050565b5f602082840312156100cf576100ce610083565b5b5f6100dc848285016100a6565b91505092915050565b6100ee81610087565b82525050565b5f6020820190506101075f8301846100e5565b9291505056fea264...

计算 CodeHash:
CodeHash = Keccak256(Runtime Code)
         = 0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890
```

**存储位置：**

```
数据库 Key-Value 存储：

Key:   'c' + CodeHash
       'c' + 0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890
       
Value: Runtime Code (完整的字节码)
       0x608060405234801561000f575f80fd5b5060043610610034...
```

#### 3.4 更新合约账户状态

```
合约账户最终状态：
┌─────────────────────────────────────────────────────┐
│ Address:     0x1234567890abcdef1234567890abcdef12345678 │
├─────────────────────────────────────────────────────┤
│ Nonce:       1                                      │
│ Balance:     0                                      │
│ StorageRoot: 0x56e81f171bcc55a6ff8345e692c0f86e5b48e01b │ ← 空树根
│              (EmptyRoot)                            │
│ CodeHash:    0xabcdef1234567890abcdef1234567890... │ ← 指向代码
└─────────────────────────────────────────────────────┘
```

---

### Step 4: 更新部署者账户状态

```
部署者账户（交易后）：
┌─────────────────────────────────────┐
│ Address: 0xb25b5c941951eeec822f502a2198cd442241b83a │
├─────────────────────────────────────┤
│ Nonce:       6  ← 增加 1            │
│ Balance:     10 ETH - Gas费用       │
│ StorageRoot: EmptyRoot              │
│ CodeHash:    EmptyHash              │
└─────────────────────────────────────┘

Gas 费用计算：
假设实际消耗了 150,000 gas
交易费用 = 150,000 × 1 wei = 150,000 wei = 0.00015 ETH
```

---

### Step 5: 生成交易回执（Receipt）

```
Transaction Receipt {
  transactionHash:  "0x[tx hash]",
  transactionIndex: 0,
  blockNumber:      12345,
  blockHash:        "0x[block hash]",
  from:             "0xb25b5c941951eeec822f502a2198cd442241b83a",
  to:               null,  ← 合约创建交易
  contractAddress:  "0x1234567890abcdef1234567890abcdef12345678", ← 新合约地址！
  gasUsed:          150000,
  cumulativeGasUsed: 150000,
  logs:             [],
  status:           1  ← 成功
}
```

---

## 4. 状态树的变化

### 4.1 Account Trie（世界状态树）

#### 部署前

```
Account Trie
└─ ...
   └─ 0xb25b5c941951eeec822f502a2198cd442241b83a
       ├─ Nonce: 5
       ├─ Balance: 10 ETH
       ├─ StorageRoot: EmptyRoot
       └─ CodeHash: EmptyHash
```

#### 部署后

```
Account Trie (新的 StateRoot)
├─ ...
│  └─ 0xb25b5c941951eeec822f502a2198cd442241b83a (更新)
│      ├─ Nonce: 6  ← 增加
│      ├─ Balance: 9.99985 ETH  ← 减少（支付 Gas）
│      ├─ StorageRoot: EmptyRoot
│      └─ CodeHash: EmptyHash
│
└─ 0x1234567890abcdef1234567890abcdef12345678 (新增！)
    ├─ Nonce: 1
    ├─ Balance: 0
    ├─ StorageRoot: EmptyRoot  ← storedData 初始为 0，空树
    └─ CodeHash: 0xabcdef1234...  ← 指向合约代码
```

### 4.2 合约存储（Storage Trie）

```
合约的 Storage Trie（初始为空）
Address: 0x1234567890abcdef1234567890abcdef12345678

Storage State:
(empty)
// 因为 storedData 尚未被 set() 函数赋值，默认为 0
// 在以太坊中，值为 0 的存储槽不占用存储空间（优化）

当调用 set(42) 后：
Storage Trie:
└─ Slot 0: 42  ← storedData 的存储位置
```

---

## 5. 底层数据库存储（LevelDB/Pebble）

### 5.1 合约代码存储

```
Key:   'c' + CodeHash
       'c' + 0xabcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890

Value: Runtime Code
       0x608060405234801561000f575f80fd5b5060043610610034575f3560e01c806360fe47b1146100385780636d4ce63c14610054...
       (完整的 Runtime Bytecode)
```

### 5.2 Account Trie 节点

```
合约账户的 Leaf Node:
─────────────────────────
Key:   [trie node path for contract address]

Value: RLP([
         node type,
         key fragment,
         RLP(Account{
           Nonce: 1,
           Balance: 0,
           Root: EmptyRoot,
           CodeHash: 0xabcdef1234567890...
         })
       ])
```

### 5.3 交易和回执存储

```
Transaction:
────────────
Key:   [transaction hash]
Value: RLP(Transaction)

Receipt:
────────
Key:   [transaction hash] + 'r'
Value: RLP(Receipt)
```

---

## 6. 完整的数据流图

```
┌─────────────────────────────────────────────────────────────┐
│  1. 用户发起合约部署交易                                     │
│     - from: 0xb25b...                                       │
│     - to: null                                              │
│     - data: bytecode                                        │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  2. 计算新合约地址                                           │
│     contract_address = Keccak256(RLP([sender, nonce]))[12:] │
│     = 0x1234567890abcdef1234567890abcdef12345678           │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  3. 创建新账户（在内存 StateDB 中）                          │
│     Address: 0x12345678...                                  │
│     Nonce: 1, Balance: 0                                    │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  4. EVM 执行 Bytecode                                       │
│     ┌────────────────────────────────────┐                 │
│     │ Initialization Code (Constructor)  │                 │
│     │ - 执行构造函数逻辑                  │                 │
│     │ - 返回 Runtime Code                │                 │
│     └────────────┬───────────────────────┘                 │
│                  │                                          │
│                  ▼                                          │
│     ┌────────────────────────────────────┐                 │
│     │ Runtime Code                       │                 │
│     │ - 包含 set() 和 get() 函数         │                 │
│     │ - 将被永久存储                     │                 │
│     └────────────────────────────────────┘                 │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  5. 存储 Runtime Code 到数据库                              │
│     Key: 'c' + CodeHash                                     │
│     Value: Runtime Code bytes                               │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  6. 更新合约账户状态                                         │
│     CodeHash = Keccak256(Runtime Code)                      │
│     StorageRoot = EmptyRoot                                 │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  7. 更新部署者账户状态                                       │
│     Nonce += 1                                              │
│     Balance -= Gas费用                                      │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  8. 重新计算 Account Trie                                   │
│     - 两个账户都被修改/新增                                  │
│     - 计算新的 StateRoot                                    │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  9. 写入新的 Trie 节点到数据库                               │
│     - 合约账户的 Leaf Node                                  │
│     - 部署者账户的 Leaf Node (更新)                         │
│     - 受影响的 Branch Nodes                                 │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  10. 创建新区块                                             │
│      Block Header.Root = 新的 StateRoot                     │
│      Block Body.Transactions = [部署交易]                   │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  11. 写入区块和交易回执                                      │
│      - Block Header & Body                                  │
│      - Transaction                                          │
│      - Receipt (含 contractAddress)                         │
└─────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  12. 部署完成！                                             │
│      合约地址: 0x1234567890abcdef1234567890abcdef12345678  │
│      可以调用 set() 和 get() 函数                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 后续操作：调用合约函数

### 7.1 调用 set(42)

```javascript
// 构造函数调用交易
tx = {
  from: "0xb25b5c941951eeec822f502a2198cd442241b83a",
  to: "0x1234567890abcdef1234567890abcdef12345678",  ← 合约地址
  data: "0x60fe47b1000000000000000000000000000000000000000000000000000000000000002a",
        // ↑ 函数选择器 (set 的签名)  ↑ 参数 42 (十六进制: 0x2a)
  gas: 50000,
  gasPrice: 1
}
```

**执行过程：**

1. **加载合约代码**
   ```
   从数据库读取：
   Key: 'c' + 0xabcdef1234...
   Value: Runtime Code
   ```

2. **EVM 执行 Runtime Code**
   - 解析函数选择器 `0x60fe47b1`（对应 `set(uint256)`）
   - 提取参数 `42`
   - 执行 `storedData = 42`

3. **更新合约存储**
   ```
   Storage Trie (新增):
   └─ Slot 0: 42
   
   合约账户状态更新：
   StorageRoot: 0x[new storage root]  ← 不再是 EmptyRoot
   ```

4. **重新计算 StateRoot**
   - 合约账户的 StorageRoot 变了
   - Account Trie 需要更新
   - 新的 StateRoot 写入新区块

### 7.2 调用 get()（只读）

```javascript
// view 函数，不需要发交易
result = web3.eth.call({
  to: "0x1234567890abcdef1234567890abcdef12345678",
  data: "0x6d4ce63c"  // get() 的函数选择器
})
// 返回: 42
```

**执行过程：**
- 只读操作，不修改状态
- 从 Storage Trie 读取 Slot 0 的值
- 返回 42
- 不消耗 Gas，不需要签名

---

## 8. 关键概念总结

### 8.1 合约地址生成（CREATE）

```
合约地址 = Keccak256(RLP([sender, nonce]))[12:]
```

- **确定性**：相同的 sender + nonce 总是生成相同的地址
- **唯一性**：每次部署 nonce 都会增加，保证地址唯一
- **可预测**：在交易执行前就能计算出合约地址

**另一种方式：CREATE2（EIP-1014）**
```
合约地址 = Keccak256(0xff + sender + salt + Keccak256(init_code))[12:]
```

### 8.2 Bytecode 的两部分

```
┌─────────────────────────┬──────────────────────┐
│  Initialization Code    │    Runtime Code      │
├─────────────────────────┼──────────────────────┤
│ - 只在部署时执行一次     │ - 永久存储在链上     │
│ - 执行构造函数          │ - 每次调用时执行     │
│ - 初始化状态变量        │ - 包含所有函数实现   │
│ - 返回 Runtime Code     │                      │
└─────────────────────────┴──────────────────────┘
```

### 8.3 合约账户 vs 外部账户（EOA）

```
外部账户（EOA）:
├─ 有私钥
├─ CodeHash = EmptyHash
├─ 可以发起交易
└─ 示例: 0xb25b5c941951eeec822f502a2198cd442241b83a

合约账户:
├─ 没有私钥
├─ CodeHash 指向合约代码
├─ 不能主动发起交易（只能被调用）
└─ 示例: 0x1234567890abcdef1234567890abcdef12345678
```

### 8.4 合约存储位置

```
1. 合约代码（Code）:
   - 存储位置: 数据库中单独的 K-V 对
   - Key: 'c' + CodeHash
   - Value: Runtime Bytecode
   - 不可修改（immutable）

2. 合约状态（Storage）:
   - 存储位置: 合约专属的 Storage Trie
   - 每个合约有独立的 Storage Trie
   - 可以通过交易修改
   - Account.Root 指向 Storage Trie 的根

3. 账户信息:
   - 存储位置: Account Trie（全局世界状态）
   - 包含: Nonce, Balance, StorageRoot, CodeHash
```

### 8.5 Gas 消耗

```
合约部署的 Gas 消耗主要来自：
1. 交易基础费用: 21,000 gas
2. 字节码数据费用: 每字节 16 gas (非零) / 4 gas (零)
3. 合约创建费用: 32,000 gas
4. 构造函数执行: 根据逻辑复杂度
5. 状态存储费用: 20,000 gas per slot (SSTORE)

本例估算:
- 基础: 21,000
- 数据: ~150 bytes × 16 = 2,400
- 创建: 32,000
- 总计: ~55,400 gas (实际可能更高)
```

---

## 9. 完整示例：从部署到调用

### 时间线

```
Block #12345:
─────────────
│ 部署交易:
│   - from: 0xb25b...
│   - to: null
│   - data: bytecode
│   - Result: 创建合约 0x12345678...
│   - StateRoot: 0xAABBCC...
│
│ 部署者 Nonce: 5 → 6

Block #12346:
─────────────
│ 调用 set(42):
│   - from: 0xb25b...
│   - to: 0x12345678...
│   - data: 0x60fe47b1...0000002a
│   - Result: storedData = 42
│   - StateRoot: 0xDDEEFF...  ← 改变了
│
│ 部署者 Nonce: 6 → 7
│ 合约 StorageRoot: EmptyRoot → 0xABCD...

Block #12347:
─────────────
│ 调用 get() (只读，不上链):
│   - 直接从状态读取
│   - 返回: 42
```

---

## 10. 常见问题

### Q1: 为什么合约地址是确定的？
**A:** 因为使用 sender + nonce 计算，这两个值在交易执行前就确定了。

### Q2: 合约代码存储在哪里？
**A:** 存储在数据库中，Key 为 `'c' + CodeHash`，不在 Trie 中（为了效率）。

### Q3: 如果构造函数失败会怎样？
**A:** 
- 交易回滚
- 合约不会被创建
- Gas 仍然被消耗
- 部署者 Nonce 仍然增加

### Q4: 合约能否修改自己的代码？
**A:** 
- 不能直接修改（代码是 immutable 的）
- 但可以通过 DELEGATECALL 实现代理模式（upgradeable contracts）
- 或使用 SELFDESTRUCT + CREATE2 重新部署

### Q5: 为什么要分 Initialization Code 和 Runtime Code？
**A:**
- **节省存储空间**：构造函数只执行一次，不需要永久保存
- **安全性**：构造函数逻辑不会在后续调用中被触发
- **效率**：Runtime Code 更小，调用时加载更快

---

## 11. 实验：手动部署合约

```bash
# 1. 启动 Geth 开发模式
geth --dev --http --http.api eth,web3,personal,net --http.corsdomain "*"

# 2. 连接控制台
geth attach http://localhost:8545

# 3. 解锁账户
personal.unlockAccount(eth.accounts[0])

# 4. 部署合约
var bytecode = "0x608060405234801561000f575f80fd5b506101438061001d5f395ff3fe..."
var tx = eth.sendTransaction({
  from: eth.accounts[0],
  data: bytecode,
  gas: 200000
})

# 5. 获取交易回执
var receipt = eth.getTransactionReceipt(tx)
var contractAddress = receipt.contractAddress
console.log("合约地址:", contractAddress)

# 6. 调用 set(42)
var setData = "0x60fe47b1" + "000000000000000000000000000000000000000000000000000000000000002a"
eth.sendTransaction({
  from: eth.accounts[0],
  to: contractAddress,
  data: setData,
  gas: 50000
})

# 7. 调用 get()
var getData = "0x6d4ce63c"
var result = eth.call({
  to: contractAddress,
  data: getData
})
console.log("storedData =", parseInt(result, 16))  // 输出: 42
```

---

## 总结

**合约部署的核心流程：**

1. ✅ 构造 `to = null` 的交易，`data` 包含完整 bytecode
2. ✅ 使用 `sender + nonce` 计算合约地址（确定性）
3. ✅ EVM 执行 Initialization Code（构造函数）
4. ✅ 提取 Runtime Code，存储到数据库（Key = `'c' + CodeHash`）
5. ✅ 创建新合约账户，设置 CodeHash 指向存储的代码
6. ✅ 更新 Account Trie，计算新的 StateRoot
7. ✅ 写入区块，返回交易回执（含 contractAddress）

**关键数据结构：**
- **Account Trie**: 全局状态树，包含所有账户
- **Storage Trie**: 每个合约独立的存储树
- **Code Storage**: 单独存储合约代码（K-V 对）
- **Transaction Receipt**: 包含部署后的合约地址

**与普通转账的区别：**
- 普通转账：`to` 为目标地址，只修改两个账户的 Balance
- 合约部署：`to` 为 null，创建新账户，存储代码，执行构造函数
