- [syncing(同步)](#syncing同步)
- [EL层的同步方式](#el层的同步方式)
  - [Snap（默认的同步方式）](#snap默认的同步方式)
  - [Full](#full)
  - [Archive nodes](#archive-nodes)
  - [Light nodes](#light-nodes)
- [CL层的同步](#cl层的同步)
    - [1. 核心区别：你是如何知道“终点”在哪里的？](#1-核心区别你是如何知道终点在哪里的)
      - [**Optimistic Sync (乐观同步)**](#optimistic-sync-乐观同步)
      - [**Checkpoint Sync (检查点同步)**](#checkpoint-sync-检查点同步)
    - [2. 它们下载的是同一种东西吗？](#2-它们下载的是同一种东西吗)










# syncing(同步)

[Sync modes | go-ethereum](https://geth.ethereum.org/docs/fundamentals/sync-modes)

Geth 有多种同步方式，它们在**速度、存储需求、以及信任假设**上有所不同。  
由于现在以太坊已经使用 PoS 共识，因此 Geth 在同步时**必须依赖共识客户端**。



# EL层的同步方式

full node的两种同步方式：Snap、Full

## Snap（默认的同步方式）

过程：从peers综合决定从哪个checkpoint（起始块）开始同步，同步到最新的head（也就是最新的块），同时只维持~128个块的状态（最近128个块的State数据）。更详细的过程：

```bash
1. 从 peers + 共识客户端选择一个 pivot block（checkpoint）

2. 下载区块数据：
   - 所有 headers（用于链结构）
   - pivot 之后的 block bodies / receipts

3. 只下载 pivot block 对应的一份 state（leaf + proof）
   → 验证能还原出 pivot 的 stateRoot

4. 本地重建 StateTrie（从 leaf 构造）

5. 执行 pivot 之后的区块，把 state 推进到最新（通过执行body.transaction).

6. 在这个过程中持续 healing（修复缺失或过期的 state）
```

## Full

**从创世块开始执行每一个区块**，来生成当前状态。

```bash
从 genesis 开始执行所有交易
→ 构建 stateTrie
→ 验证每个 stateRoot

也就是：
genesis
 → block1（执行交易）
 → block2（执行交易）
 → ...
 → head
 → 得到 stateRoot
```

128 个区块大约是 25.6 分钟的历史（按 12 秒一个区块计算）。



**总结**

其实Snap和Full很像，只不过Snap是从其中一个块开始执行，而Full是从genesis。两个都需要重放块的交易，以得到完整的State Trie。



## Archive nodes

全量数据，通过配置gc启动：geth --syncmode full --gcmode archive



## Light nodes

```bash
PoS时代，Geth 不再提供“纯轻节点”
因为：

PoS 架构下：
  ✔ 链验证交给 CL
  ✔ 状态执行交给 EL

所以：
  轻量 = CL 轻 + EL 快速同步（snap）
```

<br />

# CL层的同步

---

🔥 共识层（CL）

| 模式         | 特点     | 信任  |
| ---------- | ------ | --- |
| Full       | 全验证    | 最低  |
| Optimistic | 先下载后验证 | 中   |
| Checkpoint | 信任一个点  | 更高  |

🔥 执行层（EL）

| 模式   | 特点              |
| ---- | --------------- |
| Full | 从 genesis 执行    |
| Snap | 下载 state + 继续执行 |

---

🔥 实际组合（最常见）

```
CL：checkpoint sync 或 optimisticEL：snap sync
```



**无论是 Optimistic Sync 还是 Checkpoint Sync，最终目的都是为了获取 Beacon Blocks（信标链区块），但它们“起步”的方式和对数据的信任机制完全不同。**

你可以把这两种方式看作是进入区块链网络的两种“入场券”。

---

### 1. 核心区别：你是如何知道“终点”在哪里的？

在以太坊合并（The Merge）之后，节点必须先在共识层（Consensus Layer）确定当前的链头（Head），然后执行层（Geth）才能开始工作。

#### **Optimistic Sync (乐观同步)**

- **机制**：节点像传统方式一样，通过 P2P 网络询问邻居（Peers）：“嘿，现在的最新区块在哪？”

- **过程**：它从 Peer 那里拿到一堆区块头。因为它是“乐观”的，所以它**假设**这些区块是正确的，并立即开始下载对应的 Block Bodies。

- **状态**：此时节点处于“乐观状态”。虽然它在下载，但它不敢参与投票（Attestation）或打包区块，因为它还没验证完，万一被邻居骗了呢？

- **验证**：下载完后，它会把交易发给执行层（Geth）去逐一验证。只有验证通过，它才正式脱离“乐观”身份。

#### **Checkpoint Sync (检查点同步)**

- **机制**：节点不再问邻居，而是直接向一个**可信的源**（如信标链浏览器、API 或朋友的节点）索要一个最近的 **Checkpoint（状态快照）**。

- **过程**：
  
  1. 它首先下载这个 Checkpoint 的 **State（状态）**（包括验证者集、余额等）。
  
  2. 一旦有了这个状态，节点就瞬间知道了“真理”：它拥有了受保护的最新合法链头。
  
  3. **下载 Blocks**：它依然会下载 Beacon Blocks，但它是从这个 Checkpoint 开始向后（Sync 过程中）或向前（回填历史数据）下载。

- **优势**：极快。因为它跳过了从创世块（Genesis）开始验证几百万个 slot 的漫长过程。

---

### 2. 它们下载的是同一种东西吗？

**是的，最终下载的内容是一样的。**

无论哪种方式，共识客户端最终都需要获取完整的 **Beacon Blocks**。不同点在于：

| **特性**   | **Optimistic Sync**          | **Checkpoint Sync**            |
| -------- | ---------------------------- | ------------------------------ |
| **信任起点** | 信任 P2P 网络中的大多数（通过验证确认）       | 信任你指定的那个 URL 或快照（主观信任）         |
| **下载内容** | 从上一个已知点到链头的完整区块序列            | 从 Checkpoint 状态开始，仅需回填少量区块即可可用 |
| **安全性**  | 容易受到长程攻击（Long-range attacks） | 免疫长程攻击，因为你已经手动锚定了正确位置          |
| **速度**   | 慢（需要从 Genesis 开始算或者回溯很久）     | 极快（几分钟内即可进入就绪状态）               |


