本篇将覆盖出块流程中最关键的部分：**矿工如何从上一块状态开始，对交易逐条执行、修改状态、生成 receipts、累积 gas、构造最终区块状态根与结果。**

> 📌 **本篇既适用于 PoW、PoA 和 PoS（Engine API）时代的 Geth**，因为执行层逻辑完全通用。
> 📌 **本篇不讨论 TxPool，只关注“从交易被选中进入区块到区块准备完毕”的整个执行流水线。**

---

# 📙 **第三篇：状态转换与 EVM 执行**

---

# 一、执行层的核心职责

执行层（EL）的全部责任可以总结为：

> **给定上一块的 post-state + 交易列表，执行所有交易 → 产生一个确定性的下一块状态与执行结果。**

矿工在本地执行交易时要：

1. 加载 parent block 的状态（StateDB）
2. 设置区块上下文（BlockContext）
3. 逐条执行交易（TxContext）
4. 计算 receipts、gasUsed、logs、状态变化
5. 更新各种计量值（baseFee、blobGas、excessBlobGas）
6. 最后写入状态树（Merkle Trie → root hash）
7. 构造区块 header（Finalize 区块头）

流程结构如下：

```
parent state
    ↓ clone(working state)
BlockContext
    ↓
Tx 1 → 执行 → 状态变化
Tx 2 → 执行 → 状态变化
...
Tx N → 执行 → 状态变化
    ↓
header finalize
    ↓
block payload (未封装/未签名)
```

---

# 二、工作状态的构建：StateDB 初始化流程

矿工构建区块时的第一步：

### **从 parent block 的后状态复制一份可写状态（dirty-trie working state）**

代码位于：

```
core/state/statedb.go
core/state/database.go
miner/worker.go
```

伪代码：

```go
parent := bc.GetBlock(parentHash)
statedb := state.New(parent.Root, db)
```

### 为什么要复制？

* 避免污染真实链状态
* EVM 执行大量反复写入操作
* 需要快照（journal）支持错误回滚

所以 Geth 的 StateDB 具备这些特性：

| 功能                | 意义                            |
| ----------------- | ----------------------------- |
| Journal（undo log） | 执行失败可回滚 nonce/balance/storage |
| Dirty Storage     | 记录被修改的 storage key            |
| Touch Rules       | 在 EIP-158/161 下保证账户存在性规则      |
| Preimages         | 记录 SSTORE key 原像用于调试          |

---

# 三、BlockContext / TxContext 填充逻辑

在 EVM 执行交易之前，需要准备以下上下文：

---

## 📌 **3.1 BlockContext**

代表本区块的全局信息：

| 字段            | 说明                   |
| ------------- | -------------------- |
| Coinbase      | feeRecipient，矿工收益地址  |
| BaseFee       | EIP-1559 baseFee     |
| BlobBaseFee   | EIP-4844 blobbasefee |
| BlockNumber   | 当前区块号                |
| GasLimit      | 区块 gas 上限            |
| Time          | timestamp            |
| PrevRandao    | PoS 的 randomness     |
| ExcessBlobGas | 上一块遗留的 blob gas 负荷   |

BlockContext 在整个区块生命周期内不会变。

---

## 📌 **3.2 TxContext**

每次执行单个交易时构造：

| 字段             | 含义                                  |
| -------------- | ----------------------------------- |
| Origin         | 发送者                                 |
| GasPrice 或 Tip | 动态费率交易（1559）为 tip，Legacy 为 gasPrice |
| BlobFeeCap     | EIP-4844 blob 参数                    |
| AccessList     | EIP-2929 的 warm/cold 管理             |

TxContext 在 **每个交易执行前重建**。

---

# 四、交易执行总流程

下面是区块执行的最关键部分。

## 📌 总体伪代码

```go
func ApplyTransactions(blockCtx, txs):
    receipts := []
    cumulativeGas := 0

    for tx := range txs:
        // 1. 前置检查（nonce/balance/gas）
        if !CheckIntrinsic(tx, statedb):
            skip or drop sender
            continue

        // 2. 执行
        result, gasUsed, err := evm.ApplyTransaction(blockCtx, txContext, tx, statedb)

        if err:
            // invalid tx cannot enter block
            continue

        // 3. 更新区块 gas
        cumulativeGas += gasUsed

        // 4. 生成 receipt
        receipt := buildReceipt(tx, result, cumulativeGas)
        receipts = append(receipts, receipt)
    end

    return receipts
```

---

# 五、关键步骤详解

## **5.1 Intrinsic Gas 验证**

Intrinsic Gas 包含：

* base cost（21000）
* data cost

  * 0 byte → 4 gas
  * 非零 byte → 16 gas
* create tx cost（32000）
* blob gas（EIP-4844）

如果 intrinsicGas > tx.gasLimit → 交易直接拒绝。

---

## **5.2 Nonce 验证与递增**

矿工必须保证：

```
tx.nonce == statedb.GetNonce(sender)
```

否则：

* nonce 太小：跳过（重复/已执行）
* nonce 太大：整个账号的高 nonce tx 都不能执行（必须连续）

执行成功后：

```
statedb.IncNonce(sender)
```

---

## **5.3 扣除最大 gas 预付费**

交易在进入 EVM 前，必须预付：

```
maxGasCost = tx.gasLimit * tx.effectiveGasPrice
```

Geth 会：

```
subBalance(sender, maxGasCost)
```

执行后退回剩余 gas：

```
refund := (gasLimit - gasUsed) * tx.effectiveGasPrice
addBalance(sender, refund)
```

---

## **5.4 EVM 调用与状态修改**

执行主体位于：

```
core/vm/evm.go
core/vm/instructions
```

执行过程包括：

1. CREATE / CALL / SELFDESTRUCT / LOG
2. SLOAD / SSTORE（冷/热访问不同 gas）
3. balance/nonce/storage 读写
4. 合约代码执行（字节码 → opcodes）

所有修改都先写入：

* account state（nonce/balance/codeHash）
* storage trie dirty nodes

若执行失败（Out-of-Gas、REVERT、INVALID）：

* 状态回滚到 pre-tx 状态
* tx 不包含进区块

---

## **5.5 Gas Refund 规则**

EIP-3529 后：

* 最多退回 `gasUsed / 5`
* SELFDESTRUCT 不再返还大量 gas
* SSTORE 清零仍有小额度返还

---

# 六、Merkle Trie 更新（状态树写入）

所有 dirty storage 和账户更新会在 finalize 阶段写入底层 database，计算：

* `stateRoot`
* `transactionsRoot`
* `receiptsRoot`
* `logsBloom`

写入时 Geth 会：

1. 遍历 dirty nodes
2. 生成 RLP 编码
3. 哈希后写入 LevelDB
4. 清理 cached trie node

这是构造区块 header 的关键一步。

---

# 七、Block 内部计量规则（BaseFee / BlobGas）

矿工在执行交易时必须更新多个链级计量值。

---

## **7.1 BaseFee 计算（EIP-1559）**

根据公式：

```
baseFee_{n+1} = baseFee_n * (1 + gasUsed/targetGas) +/- delta
```

Geth 在构造 header 前计算新 baseFee。

---

## **7.2 BlobGas 逻辑（EIP-4844）**

主要包括：

* blobGasUsed
* excessBlobGas（上块遗留）
* blobBaseFee（按负荷自动调整）

计算方式参见：

```
EIP-4844 blob gas equations
```

Geth 在 finalize 阶段更新这些字段。

---

# 八、Receipts、Logs、Bloom 生成

每个成功执行的交易都产生一个 receipt：

字段：

| 字段                | 说明          |
| ----------------- | ----------- |
| status            | 成功/失败       |
| cumulativeGasUsed | 累积 gas      |
| logs              | 事件日志        |
| bloom             | 日志 bloom 位图 |
| blobGasUsed       | (若有)        |

所有 receipts 会构建成一个 Merkle Patricia Trie（receiptTrie）。

Bloom Filter 是 256 byte 的 bitmap，每个 log 的 topics/address 都会设置相应 bit。

---

# 九、Finalize 阶段（构造 header）

执行完所有交易后，矿工需要构建最终区块头：

```
header.StateRoot
header.TxRoot
header.ReceiptsRoot
header.GasUsed
header.GasLimit
header.Timestamp
header.ExcessBlobGas
header.BaseFee
header.BlobGasUsed
header.LogsBloom
```

之后返回给共识层进行封装 Seal。

---

# 十、本篇总结：矿工（执行层）的本质

执行层（Miner / Engine）的核心逻辑：

> **将所有交易在一个可写状态中模拟执行，记录每个状态变化，最终输出一个 deterministic 的下一块状态。**

它不是简单地：

* 执行交易
* 拼合区块

而是一个高度完整的 **状态机驱动系统**：

* 在工作状态上执行 EVM
* 严格遵守 gas、nonce、blob 限制
* 构造状态树
* 构造 receipts、bloom、root
* 最终构成可以交给共识层封装的区块

---

