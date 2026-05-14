# Balance 变化分析

## 场景：0x00001 的 Balance 从 1 ETH 变为 2 ETH

---

## 1. 初始状态（Genesis Block 刚创建完成）

### Account 0x00001 初始状态
```
Address: 0x0000000000000000000000000000000000000001
Balance: 1000000000000000000 wei (1 ETH)
Nonce:   0
Code:    0x1234567890abcdef
Storage: {0x01: 0xabcdef1234567890}
```

### 初始的 Account Trie 结构

```
                    ┌─────────────────────────┐
                    │   Account Trie Root     │
                    │   Hash: 0xABCD1234...   │  ← 初始 StateRoot
                    └───────────┬─────────────┘
                                │
                       ┌────────┴────────┐
                       │  Branch Node    │
                       │  Hash: 0xBR01   │
                       └────┬──────┬─────┘
                            │      │
               ┌────────────┘      └─────────────┐
               │                                  │
               ▼                                  ▼
    ┌──────────────────┐                ┌──────────────────┐
    │ Leaf: 0x00001    │                │ Leaf: 0x00002    │
    │ Hash: 0xLEAF1    │                │ Hash: 0xLEAF2    │
    ├──────────────────┤                ├──────────────────┤
    │ Nonce: 0         │                │ Nonce: 0         │
    │ Balance: 1 ETH   │ ← 即将改变     │ Balance: 1 ETH   │
    │ Root: 0xST01     │                │ Root: 0xST02     │
    │ CodeHash: 0xC1   │                │ CodeHash: 0xC2   │
    └──────────────────┘                └──────────────────┘
```

### Genesis Block Header
```
Block #0
├─ ParentHash:  0x0000...0000
├─ Root:        0xABCD1234...  ← 当前 StateRoot
├─ Number:      0
├─ GasUsed:     0
└─ ...
```

---

## 2. 执行交易：向 0x00001 转账 1 ETH

### 交易内容
```
Transaction {
  From:     0xSomeAddress
  To:       0x0000000000000000000000000000000000000001
  Value:    1000000000000000000 wei (1 ETH)
  Gas:      21000
  GasPrice: 1 gwei
  Nonce:    X
}
```

---

## 3. 状态变化过程（在内存中的 StateDB）

### Step 1: 加载当前状态
```
StateDB.root = 0xABCD1234...  (从 Genesis Block Header 读取)

加载 Account 0x00001:
┌────────────────────────────────┐
│ stateObject for 0x00001        │
├────────────────────────────────┤
│ address:  0x00001              │
│ balance:  1 ETH (原值)         │
│ nonce:    0                    │
│ code:     0x1234567890abcdef   │
│ storage:  {0x01: 0xabcd...}    │
│ dirtyStorage: {}  (空)         │
│ dirty:    false                │
└────────────────────────────────┘
```

### Step 2: 修改 Balance（在内存中）
```
stateObject.AddBalance(1 ETH)

┌────────────────────────────────┐
│ stateObject for 0x00001        │
├────────────────────────────────┤
│ address:  0x00001              │
│ balance:  2 ETH (新值) ← 改变  │
│ nonce:    0                    │
│ code:     0x1234567890abcdef   │
│ storage:  {0x01: 0xabcd...}    │
│ dirtyStorage: {}  (空)         │
│ dirty:    true  ← 标记为脏     │
└────────────────────────────────┘

注意：此时只是内存中的修改，尚未写入 Trie！
```

### Step 3: 交易执行完成，准备 Commit

```
Block #1 (新区块)
├─ Transactions: [转账交易]
├─ 交易执行完成
└─ 准备计算新的 StateRoot
```

---

## 4. Commit 过程：重新计算 Trie

### Step 4.1: 更新 Account Trie（自底向上）

#### 4.1.1 Account 0x00001 的新数据（RLP 编码）

```
旧的 Account 数据:
RLP([
  Nonce: 0,
  Balance: 1000000000000000000,
  Root: 0xST01,
  CodeHash: 0xC1
])
Hash: 0xLEAF1 (旧的叶子节点哈希)

────────────────────────────────────

新的 Account 数据:
RLP([
  Nonce: 0,
  Balance: 2000000000000000000,  ← 改变了！
  Root: 0xST01,                   (Storage 未变)
  CodeHash: 0xC1                  (Code 未变)
])
Hash: 0xLEAF1_NEW (新的叶子节点哈希) ← 哈希值改变！
```

#### 4.1.2 新的 Leaf Node

```
Leaf Node for 0x00001 (新版本):
┌──────────────────────────────────┐
│ Type: Leaf                       │
│ Key:  [remaining nibbles]        │
│ Value: RLP(Account{              │
│   Nonce: 0,                      │
│   Balance: 2 ETH,  ← 新值        │
│   Root: 0xST01,                  │
│   CodeHash: 0xC1                 │
│ })                               │
│                                  │
│ Node Hash: 0xLEAF1_NEW           │
└──────────────────────────────────┘
```

#### 4.1.3 Branch Node 需要重新计算

```
旧的 Branch Node:
[
  branch[0]: 0xLEAF1,      ← 指向旧的 0x00001 叶子
  branch[1]: 0xLEAF2,      ← 0x00002 未变
  branch[2-15]: nil,
  value: nil
]
Hash: 0xBR01 (旧哈希)

────────────────────────────────────

新的 Branch Node:
[
  branch[0]: 0xLEAF1_NEW,  ← 指向新的 0x00001 叶子！
  branch[1]: 0xLEAF2,      ← 0x00002 未变
  branch[2-15]: nil,
  value: nil
]
Hash: 0xBR01_NEW (新哈希) ← 哈希值改变！
```

#### 4.1.4 Account Trie Root 重新计算

```
新的 Account Trie Root:
Hash(Branch Node 0xBR01_NEW) = 0xABCD5678_NEW

StateRoot 改变了！
从 0xABCD1234... → 0xABCD5678_NEW
```

---

## 5. 新的 Account Trie 结构（Commit 后）

```
                    ┌─────────────────────────┐
                    │   Account Trie Root     │
                    │   Hash: 0xABCD5678_NEW  │  ← 新的 StateRoot
                    └───────────┬─────────────┘
                                │
                       ┌────────┴────────┐
                       │  Branch Node    │
                       │  Hash: 0xBR01_NEW│ ← 新哈希
                       └────┬──────┬─────┘
                            │      │
               ┌────────────┘      └─────────────┐
               │                                  │
               ▼                                  ▼
    ┌──────────────────┐                ┌──────────────────┐
    │ Leaf: 0x00001    │                │ Leaf: 0x00002    │
    │ Hash: 0xLEAF1_NEW│ ← 新哈希       │ Hash: 0xLEAF2    │ ← 未变
    ├──────────────────┤                ├──────────────────┤
    │ Nonce: 0         │                │ Nonce: 0         │
    │ Balance: 2 ETH   │ ← 新值！       │ Balance: 1 ETH   │
    │ Root: 0xST01     │ ← 未变         │ Root: 0xST02     │
    │ CodeHash: 0xC1   │ ← 未变         │ CodeHash: 0xC2   │
    └──────────────────┘                └──────────────────┘
            │                                    │
            │ Storage Trie 未变                  │ Storage Trie 未变
            ▼                                    ▼
    ┌──────────────────┐                ┌──────────────────┐
    │ Storage Trie     │                │ Storage Trie     │
    │ Root: 0xST01     │                │ Root: 0xST02     │
    └──────────────────┘                └──────────────────┘
```

---

## 6. 新区块（Block #1）的结构

```
Block #1
┌──────────────────────────────────────────────────────┐
│                    Block Header                      │
├──────────────────────────────────────────────────────┤
│ ParentHash:    0x[Block #0 Hash]                    │ ← 指向 Genesis
│ UncleHash:     EmptyUncleHash                        │
│ Coinbase:      0xMinerAddress                        │
│ Root:          0xABCD5678_NEW  ← 新的 StateRoot!    │
│ TxHash:        0x[Tx Merkle Root]                    │
│ ReceiptHash:   0x[Receipt Merkle Root]               │
│ Bloom:         [log bloom]                           │
│ Difficulty:    0x1                                   │
│ Number:        1  ← 新区块号                         │
│ GasLimit:      0x8000000                             │
│ GasUsed:       21000  ← 转账消耗的 gas               │
│ Time:          [timestamp]                           │
│ Extra:         []byte                                │
│ MixDigest:     ...                                   │
│ Nonce:         ...                                   │
│ BaseFee:       ...                                   │
├──────────────────────────────────────────────────────┤
│                    Block Body                        │
├──────────────────────────────────────────────────────┤
│ Transactions:  [转账交易]                            │
│ Uncles:        []                                    │
└──────────────────────────────────────────────────────┘
```

---

## 7. 底层数据库变化（LevelDB/Pebble）

### 7.1 新增的 Trie 节点

```
新的 Leaf Node for 0x00001:
────────────────────────────
Key:   [trie path hash for 0xLEAF1_NEW]
Value: RLP([
         leaf flag,
         key nibbles,
         RLP(Account{
           Nonce: 0,
           Balance: 2000000000000000000,
           Root: 0xST01,
           CodeHash: 0xC1
         })
       ])

新的 Branch Node:
────────────────────────────
Key:   [trie path hash for 0xBR01_NEW]
Value: RLP([
         branch[0]: 0xLEAF1_NEW,
         branch[1]: 0xLEAF2,
         branch[2-15]: nil,
         value: nil
       ])

新的 Account Trie Root:
────────────────────────────
(Root 节点本身可能也需要写入，取决于实现)
```

### 7.2 新的区块数据

```
Block #1 Header:
────────────────────────────
Key:   h00000000000000010x[block #1 hash]
Value: RLP(Block #1 Header)

Block #1 Body:
────────────────────────────
Key:   b00000000000000010x[block #1 hash]
Value: RLP([[转账交易], []])

Block Number → Hash:
────────────────────────────
Key:   H0x[block #1 hash]
Value: 0000000000000001

Latest Block:
────────────────────────────
Key:   LastBlock
Value: 0x[block #1 hash]  ← 更新为新区块
```

### 7.3 旧数据仍然保留（用于历史查询）

```
旧的 Trie 节点依然存在：
- 0xLEAF1 (旧的 0x00001 叶子) ← 仍在数据库中
- 0xBR01 (旧的 Branch) ← 仍在数据库中
- 0xABCD1234... (旧的 Root) ← 仍在数据库中

这样可以通过 Block #0 的 StateRoot 重建旧状态！
```

---

## 8. 变化传播路径（哈希链）

```
Balance 改变
    ↓
Account Object 改变 (RLP 编码后的内容变了)
    ↓
Leaf Node 哈希改变 (0xLEAF1 → 0xLEAF1_NEW)
    ↓
Branch Node 哈希改变 (0xBR01 → 0xBR01_NEW)
    ↓
Account Trie Root 改变 (0xABCD1234 → 0xABCD5678_NEW)
    ↓
Block Header.Root 改变
    ↓
Block Hash 改变
```

**关键点：任何状态的微小改变，都会向上传播，最终改变 Block Hash！**

---

## 9. 对比图：变化前后

### Before (Genesis Block)

```
Block #0
├─ StateRoot: 0xABCD1234...
└─ Account Trie
    ├─ Branch (0xBR01)
    │   ├─ Leaf 0x00001 (0xLEAF1)
    │   │   └─ Balance: 1 ETH
    │   └─ Leaf 0x00002 (0xLEAF2)
    │       └─ Balance: 1 ETH
    └─ Storage Tries (未变)
```

### After (Block #1)

```
Block #1
├─ ParentHash: [Block #0 Hash]
├─ StateRoot: 0xABCD5678_NEW  ← 改变！
└─ Account Trie
    ├─ Branch (0xBR01_NEW)  ← 新节点
    │   ├─ Leaf 0x00001 (0xLEAF1_NEW)  ← 新节点
    │   │   └─ Balance: 2 ETH  ← 新值
    │   └─ Leaf 0x00002 (0xLEAF2)  ← 未变，复用
    │       └─ Balance: 1 ETH
    └─ Storage Tries (未变，复用)
```

---

## 10. 重要概念总结

### 10.1 Copy-on-Write（写时复制）

```
- 当 0x00001 的 balance 改变时，只创建新的节点
- 0x00002 的节点没有改变，直接复用旧节点
- Storage Trie 未变，直接复用

新 Trie 节点：
✓ Leaf 0x00001 (新)
✓ Branch Node (新)
✓ Root (新)

复用的旧节点：
✓ Leaf 0x00002 (复用)
✓ Storage Trie for 0x00001 (复用)
✓ Storage Trie for 0x00002 (复用)
✓ Code (复用)
```

### 10.2 历史状态可追溯

```
想查询 Block #0 时的 0x00001 余额？
1. 读取 Block #0 Header.Root = 0xABCD1234...
2. 从这个 Root 开始遍历 Account Trie (旧版本)
3. 找到旧的 Leaf 0xLEAF1
4. 读取 Balance = 1 ETH

想查询 Block #1 时的 0x00001 余额？
1. 读取 Block #1 Header.Root = 0xABCD5678_NEW
2. 从这个 Root 开始遍历 Account Trie (新版本)
3. 找到新的 Leaf 0xLEAF1_NEW
4. 读取 Balance = 2 ETH
```

### 10.3 Merkle Proof（状态证明）

```
证明 Block #1 中 0x00001 的 Balance = 2 ETH：

需要提供：
1. 0x00001 的 Leaf Node (0xLEAF1_NEW)
2. Branch Node (0xBR01_NEW)
3. 从 Branch 到 Root 的路径

验证过程：
1. 验证 Leaf Node 包含 Balance = 2 ETH
2. Hash(Leaf) = 0xLEAF1_NEW ✓
3. Hash(Branch[含 0xLEAF1_NEW]) = 0xBR01_NEW ✓
4. Hash(Branch) = StateRoot ✓

如果验证通过，则证明 Balance = 2 ETH 是正确的！
```

---

## 11. 关键要点

1. **状态改变 → StateRoot 改变**
   - Balance 变化会导致整条哈希链向上传播
   - 最终 Block Header.Root 改变

2. **Copy-on-Write 优化**
   - 只有变化的节点需要创建新版本
   - 未变化的节点可以复用，节省存储空间

3. **历史可追溯**
   - 旧的 Trie 节点仍然保留在数据库中
   - 通过旧的 StateRoot 可以重建历史状态

4. **完整性保证**
   - 任何数据篡改都会导致哈希不匹配
   - Merkle Proof 可以高效验证状态

5. **存储效率**
   - 只存储变化的节点（增量存储）
   - 不同版本的 Trie 共享未变化的子树
