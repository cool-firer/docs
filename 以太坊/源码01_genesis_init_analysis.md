# Genesis Init 完整分析

## 1. Genesis.json 内容

```json
{
  "config": {
    "chainId": 1337,
    "homesteadBlock": 0,
    ...
  },
  "difficulty": "0x1",
  "gasLimit": "0x8000000",
  "alloc": {
    "0x00001": {
      "Code": "0x1234567890abcdef",
      "Storage": { "0x01": "0xabcdef1234567890" },
      "Balance": "0x1000000000000000000"
    },
    "0x00002": {
      "Code": "0x222222222222222",
      "Storage": { "0x01": "0xabcdef1234567890" },
      "Balance": "0x1000000000000000000"
    }
  }
}
```

---

## 2. Account Trie（世界状态树）结构

### 账户地址处理
- 原始地址：`0x00001` → 补齐为 20 字节：`0x0000000000000000000000000000000000000001`
- 原始地址：`0x00002` → 补齐为 20 字节：`0x0000000000000000000000000000000000000002`

### Account Trie 树形结构

```
                    ┌─────────────────┐
                    │   Account Trie  │
                    │      Root       │
                    │  (StateRoot)    │
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    │  Branch Node    │
                    │   (16 branches) │
                    └────┬──────┬─────┘
                         │      │
        ┌────────────────┘      └─────────────────┐
        │                                          │
        ▼                                          ▼
┌───────────────┐                         ┌───────────────┐
│  Leaf Node    │                         │  Leaf Node    │
│  for 0x00001  │                         │  for 0x00002  │
└───────┬───────┘                         └───────┬───────┘
        │                                          │
        │                                          │
        ▼                                          ▼
┌──────────────────────┐                 ┌──────────────────────┐
│ Account Data (RLP)   │                 │ Account Data (RLP)   │
├──────────────────────┤                 ├──────────────────────┤
│ Nonce: 0             │                 │ Nonce: 0             │
│ Balance: 10^18 wei   │                 │ Balance: 10^18 wei   │
│ StorageRoot: [hash]  │────┐            │ StorageRoot: [hash]  │────┐
│ CodeHash: [hash]     │    │            │ CodeHash: [hash]     │    │
└──────────────────────┘    │            └──────────────────────┘    │
                             │                                        │
                             │                                        │
                             ▼                                        ▼
                    ┌─────────────────┐                    ┌─────────────────┐
                    │ Storage Trie    │                    │ Storage Trie    │
                    │  for 0x00001    │                    │  for 0x00002    │
                    └────────┬────────┘                    └────────┬────────┘
                             │                                      │
                             ▼                                      ▼
                    ┌─────────────────┐                    ┌─────────────────┐
                    │  Leaf Node      │                    │  Leaf Node      │
                    │  Key: 0x01      │                    │  Key: 0x01      │
                    │  Value:         │                    │  Value:         │
                    │  0xabcdef...    │                    │  0xabcdef...    │
                    └─────────────────┘                    └─────────────────┘
```

---

## 3. 详细的账户数据结构

### Account 0x00001 的完整状态

```
Address: 0x0000000000000000000000000000000000000001

Account Object (types.StateAccount):
┌─────────────────────────────────────────────────────┐
│ Nonce:        0                                     │
│ Balance:      1000000000000000000 (1 ETH)          │
│ Root:         0x[StorageTrie Root Hash]            │  ← 指向该账户的 Storage Trie
│ CodeHash:     0x[Keccak256(Code)]                  │  ← 合约代码的哈希
└─────────────────────────────────────────────────────┘

Contract Code (存储在 codeDB):
Key:   0x[CodeHash]
Value: 0x1234567890abcdef

Storage Trie for 0x00001:
Key:   Keccak256(0x01) = 0x[hash of 0x01]
Value: 0xabcdef1234567890 (RLP 编码后存储)
```

### Account 0x00002 的完整状态

```
Address: 0x0000000000000000000000000000000000000002

Account Object (types.StateAccount):
┌─────────────────────────────────────────────────────┐
│ Nonce:        0                                     │
│ Balance:      1000000000000000000 (1 ETH)          │
│ Root:         0x[StorageTrie Root Hash]            │  ← 指向该账户的 Storage Trie
│ CodeHash:     0x[Keccak256(Code)]                  │  ← 合约代码的哈希
└─────────────────────────────────────────────────────┘

Contract Code (存储在 codeDB):
Key:   0x[CodeHash]
Value: 0x222222222222222

Storage Trie for 0x00002:
Key:   Keccak256(0x01) = 0x[hash of 0x01]
Value: 0xabcdef1234567890 (RLP 编码后存储)
```

---

## 4. Genesis Block（创世区块）结构

```
Genesis Block (types.Block)
┌──────────────────────────────────────────────────────┐
│                    Block Header                      │
├──────────────────────────────────────────────────────┤
│ ParentHash:    0x0000...0000 (32 bytes of zeros)    │
│ UncleHash:     EmptyUncleHash                        │
│ Coinbase:      0x0000...0000                         │
│ Root:          0x[Account Trie Root Hash]  ←────────┼──┐
│ TxHash:        EmptyRootHash (no txs)                │  │
│ ReceiptHash:   EmptyRootHash (no receipts)           │  │
│ Bloom:         [empty bloom filter]                  │  │
│ Difficulty:    0x1                                   │  │
│ Number:        0                                     │  │
│ GasLimit:      0x8000000 (134217728)                │  │
│ GasUsed:       0                                     │  │
│ Time:          0 (Unix timestamp)                    │  │
│ Extra:         []byte                                │  │
│ MixDigest:     0x0000...0000                         │  │
│ Nonce:         0                                     │  │
│ BaseFee:       nil (London fork enabled at 0)       │  │
├──────────────────────────────────────────────────────┤  │
│                    Block Body                        │  │
├──────────────────────────────────────────────────────┤  │
│ Transactions:  []  (empty)                           │  │
│ Uncles:        []  (empty)                           │  │
└──────────────────────────────────────────────────────┘  │
                                                           │
         ┌─────────────────────────────────────────────────┘
         │
         │  Block Header 的 Root 字段指向 Account Trie 的根
         │
         ▼
┌──────────────────────┐
│  Account Trie Root   │
│  (StateRoot)         │
└──────────────────────┘
```

---

## 5. 底层 Key-Value 存储（LevelDB/Pebble）

### 5.1 区块数据存储

```
Key Format: 'h' + BlockNumber (8 bytes) + BlockHash (32 bytes)
Value: RLP(BlockHeader)

Example:
Key:   h00000000000000000x[genesis block hash]
Value: RLP(Genesis Block Header)

---

Key Format: 'b' + BlockNumber (8 bytes) + BlockHash (32 bytes)
Value: RLP(BlockBody) = RLP([Transactions, Uncles])

Example:
Key:   b00000000000000000x[genesis block hash]
Value: RLP([[], []]) ← 空交易和空叔块

---

Key Format: 'H' + BlockHash (32 bytes)
Value: BlockNumber (8 bytes)

Example:
Key:   H0x[genesis block hash]
Value: 0000000000000000 ← block number 0

---

Key Format: 'LastBlock'
Value: BlockHash (32 bytes)

Example:
Key:   LastBlock
Value: 0x[genesis block hash]
```

### 5.2 Account Trie 节点存储

```
Secure Trie: Key = Keccak256(Address)

Account 0x00001:
─────────────────
Trie Path:  Keccak256(0x0000000000000000000000000000000000000001)
           = 0x[hash1]

存储结构：
Key:   [trie node path] (节点路径的哈希)
Value: RLP([
         node type,
         key fragment,
         RLP(Account{
           Nonce: 0,
           Balance: 1000000000000000000,
           Root: 0x[storage trie root],
           CodeHash: 0x[code hash]
         })
       ])

Account 0x00002:
─────────────────
Trie Path:  Keccak256(0x0000000000000000000000000000000000000002)
           = 0x[hash2]

存储结构：
Key:   [trie node path]
Value: RLP([
         node type,
         key fragment,
         RLP(Account{
           Nonce: 0,
           Balance: 1000000000000000000,
           Root: 0x[storage trie root],
           CodeHash: 0x[code hash]
         })
       ])
```

### 5.3 Storage Trie 节点存储

```
Storage Trie for 0x00001:
─────────────────────────
Secure Trie: Key = Keccak256(Storage Key)

Storage Key 0x01:
Trie Path:  Keccak256(0x01) = 0x[storage_hash]

Key:   [storage trie node path]
Value: RLP([
         node type,
         key fragment,
         RLP(0xabcdef1234567890)  ← storage value
       ])

Storage Trie for 0x00002:
─────────────────────────
(结构相同)
```

### 5.4 合约代码存储

```
Code for 0x00001:
─────────────────
Key:   c + [CodeHash]  ← 'c' 前缀 + Keccak256(code)
Value: 0x1234567890abcdef

Code for 0x00002:
─────────────────
Key:   c + [CodeHash]
Value: 0x222222222222222
```

### 5.5 状态根存储（可选，用于快速查找）

```
Key:   s + [StateRoot]  ← 's' 前缀 + state root hash
Value: [metadata about this state]
```

---

## 6. Trie 节点编码详解

### 6.1 Leaf Node 示例（以 Account 0x00001 为例）

```
Leaf Node 结构：
[
  flag,              ← 节点类型标记（带终止符）
  key,               ← 剩余的 key 路径（nibbles）
  value              ← 账户数据（RLP 编码）
]

具体编码：
[
  0x20,              ← leaf flag (0x20 = 32 = compact encode with terminator)
  [remaining nibbles of Keccak256(address)],
  RLP([
    Nonce: 0,
    Balance: 0x0de0b6b3a7640000,  ← 1 ETH in hex
    Root: 0x[storage root hash],
    CodeHash: 0x[code hash]
  ])
]
```

### 6.2 Branch Node（如果有多个账户需要分叉）

```
Branch Node 结构：
[
  branch[0],         ← 指向子节点的哈希或内联节点
  branch[1],
  ...
  branch[15],
  value              ← 如果该分支本身是终点，存储值
]

在我们的例子中，如果 0x00001 和 0x00002 的 key 路径
在某个 nibble 位置不同，会创建 branch node 分叉。
```

---

## 7. 完整的数据流图

```
┌─────────────────────────────────────────────────────────────┐
│                    Genesis.json                             │
└────────────────────────────┬────────────────────────────────┘
                             │
                             │ geth init
                             ▼
┌─────────────────────────────────────────────────────────────┐
│              构建 StateDB (内存中的状态)                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Account 0x00001                                      │  │
│  │  - Balance: 1 ETH                                   │  │
│  │  - Code: 0x1234567890abcdef                         │  │
│  │  - Storage: {0x01: 0xabcdef1234567890}              │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Account 0x00002                                      │  │
│  │  - Balance: 1 ETH                                   │  │
│  │  - Code: 0x222222222222222                          │  │
│  │  - Storage: {0x01: 0xabcdef1234567890}              │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────┘
                             │
                             │ Commit()
                             ▼
┌─────────────────────────────────────────────────────────────┐
│            计算 Storage Trie Roots                          │
│  - Storage Trie for 0x00001 → Root Hash A                  │
│  - Storage Trie for 0x00002 → Root Hash B                  │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│        写入 Storage Trie 节点到数据库                       │
│  Key: [path hash]  Value: RLP(trie node)                   │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│           计算 Account Trie Root                            │
│  - 插入 Account 0x00001 (with Storage Root A)              │
│  - 插入 Account 0x00002 (with Storage Root B)              │
│  → Account Trie Root = StateRoot                           │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│        写入 Account Trie 节点到数据库                       │
│  Key: [path hash]  Value: RLP(trie node)                   │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│              写入合约代码到数据库                           │
│  Key: c + CodeHash  Value: Code Bytes                      │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│             构建 Genesis Block                              │
│  Header.Root = StateRoot (Account Trie Root)               │
│  Header.Number = 0                                          │
│  Header.Difficulty = 0x1                                    │
│  Body = {Txs: [], Uncles: []}                              │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│        写入 Genesis Block 到数据库                          │
│  - Block Header (h + number + hash)                        │
│  - Block Body (b + number + hash)                          │
│  - Block Number → Hash mapping (H + hash)                  │
│  - LastBlock pointer                                        │
└─────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│                  初始化完成！                               │
│         链已就绪，可以开始出块                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. 关键哈希值说明

### StateRoot (Account Trie Root)
这是 Genesis Block Header 中的 `Root` 字段，代表整个世界状态的唯一标识。
任何账户的余额、代码、存储的改变都会导致这个值变化。

### StorageRoot (每个账户的 Storage Trie Root)
每个智能合约账户都有自己的 Storage Trie，存储该合约的所有状态变量。
StorageRoot 存储在账户对象中。

### CodeHash
合约代码的 Keccak256 哈希值，用于在数据库中索引合约代码。
外部账户（EOA）的 CodeHash 为空哈希（Keccak256("")）。

---

## 9. 总结

1. **两层 Trie 结构**：
   - 外层：Account Trie（世界状态）
   - 内层：每个合约账户有独立的 Storage Trie

2. **Genesis Init 过程**：
   - 解析 genesis.json
   - 构建内存状态
   - 计算所有 Trie 的根哈希
   - 写入所有节点到数据库
   - 创建并存储 Genesis Block

3. **数据隔离**：
   - 不同账户的数据通过 Account Trie 隔离
   - 同一账户的不同存储槽通过 Storage Trie 组织
   - 合约代码通过 CodeHash 单独存储

4. **哈希链接**：
   - Block Header.Root → Account Trie Root
   - Account.Root → Storage Trie Root
   - Account.CodeHash → Contract Code

这种设计保证了：
- **完整性**：任何数据改变都会反映在根哈希上
- **可验证性**：可以通过 Merkle Proof 验证任何数据
- **历史状态**：可以通过 StateRoot 重建任意历史状态
