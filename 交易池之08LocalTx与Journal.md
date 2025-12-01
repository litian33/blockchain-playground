
---

## **专题八：Local Transactions 与 Journal**

### **1. 概述**

Geth TxPool 不仅管理来自网络或 RPC 的普通交易，还需要支持 **本地用户提交的交易**（Local Tx）。本地交易通常指节点自身生成、签名的交易，这些交易具有以下特点：

1. **高优先级**：应立即加入 Pending 队列，优先打包。
2. **持久化**：在节点重启后可以恢复，不丢失。
3. **隔离管理**：不受一般 P2P 驱逐策略直接影响，但仍受全局资源限制控制。

为实现这些目标，Geth 引入了 **Journal 机制**，将本地交易序列化存储到磁盘，在节点启动时读取恢复。

---

### **2. 本地交易（Local Tx）设计**

#### **2.1 生命周期**

1. 用户通过 RPC 或 CLI 提交本地交易。
2. TxPool 的 `AddLocal` 方法接收交易：

   * 立即验证合法性（签名、Nonce、余额）。
   * 将交易标记为 LocalTx。
   * 写入 Journal 持久化日志。
   * 添加到 Pending 队列或晋升 queue。
3. 在区块上链或驱逐时：

   * Local Tx 默认不会被普通驱逐策略立即移除。
   * 如果本地节点上链成功，Local Tx 会被清理。

#### **2.2 优先级**

* Local Tx 在 Pending 队列中具有高优先级。
* 可确保即使存在网络交易拥塞，本地用户提交的交易也能及时打包。

---

### **3. Journal 持久化机制**

Journal 是 Geth 维护本地交易状态的磁盘日志，实现了**事务安全、节点重启恢复**。

#### **3.1 文件位置**

* 默认路径：`chaindata/txpool/journal.rlp`
* 文件格式：**RLP 列表**，每条记录包含：

  1. 交易的 RLP 编码。
  2. LocalTx 标识（可选元数据）。
  3. 时间戳（用于策略或驱逐）。

#### **3.2 写入流程**

```go
// core/txpool/journal.go
func (j *Journal) Append(tx *types.Transaction) error {
    // 将 tx RLP 编码
    encoded := tx.MarshalBinary()
    // 写入文件末尾
    _, err := j.file.Write(encoded)
    if err != nil { return err }
    // 刷盘保证持久化
    return j.file.Sync()
}
```

* 每次 Local Tx 添加时写入 Journal。
* Journal 写入是顺序追加（append-only），便于快速恢复。

#### **3.3 恢复流程**

节点启动时：

```go
func (j *Journal) Replay(pool *TxPool) error {
    // 读取 journal 文件
    for each txRLP in journalFile {
        tx := decodeRLP(txRLP)
        // 添加到 pool 的 LegacyPool 或 BlobPool
        pool.AddLocal(tx)
    }
}
```

* Replay 后，本地交易恢复到 Pending 队列，确保重启不中断本地交易生命周期。

#### **3.4 清理策略**

* Local Tx 成功上链后，从 Journal 中删除。
* 如果交易被驱逐（如超过最大 pending 或 queue 容量），Journal 可以保留或清理（可配置）。

---

### **4. Local Tx 与子池关系**

| 特性     | LegacyPool    | BlobPool             |
| ------ | ------------- | -------------------- |
| 本地交易支持 | ✅             | ✅（需要磁盘存储 Blob 数据）    |
| 持久化机制  | Journal (RLP) | Journal + Blob 数据文件  |
| 优先级    | 高优先级 Pending  | 高优先级 Pending（索引指向磁盘） |
| 替换策略   | 支持高 Gas 替换    | 支持高 BlobFee 替换       |

* Local Tx 在 `AddLocal` 入口添加时，会自动选择对应子池。
* LegacyPool 内存管理，BlobPool 磁盘 + 内存索引。
* 上链后，TxPool 调用 `Reset` 会清理 Local Tx 对应的 Pending/Queue 条目，同时 Journal 删除。

---

### **5. 核心方法与文件**

| 文件                                     | 方法             | 功能                            |
| -------------------------------------- | -------------- | ----------------------------- |
| `core/txpool/txpool.go`                | `AddLocal(tx)` | 入口方法，将本地交易路由到子池并写入 Journal    |
| `core/txpool/journal.go`               | `Append(tx)`   | 写入 RLP 本地交易日志                 |
| `core/txpool/journal.go`               | `Replay(pool)` | 节点启动恢复 Local Tx               |
| `core/txpool/legacypool/legacypool.go` | `addLocal(tx)` | LegacyPool 内部添加 Local Tx 流程   |
| `core/txpool/blobpool/blobpool.go`     | `addLocal(tx)` | BlobPool 内部添加 Local BlobTx 流程 |

---

### **6. 小结**

1. **Local Tx 与普通交易分离管理**，保证用户本地交易高优先级执行。
2. **Journal 机制提供持久化**，顺序写入 RLP 日志文件，可在节点重启后恢复。
3. **Pending / Queue / 子池协调**：Local Tx 可以立即进入 Pending 或 queue，仍遵循非 Local Tx 的晋升逻辑。
4. **与 BlobPool 集成**：支持 EIP-4844 类型本地交易，磁盘 + 内存索引机制保证资源安全。
5. **GC 与清理**：本地交易上链或被驱逐后，Journal 同步清理，确保不会无限累积。

