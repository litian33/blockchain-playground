
---

# 1. 概览

* **Logs（事件/日志）**：由 EVM `LOGn` 指令产生、随交易 receipt 一起保存的结构，包含 `address`、`topics[]` 和 `data`，用于事件检索与链下索引。
* **Bloom（logs bloom）**：每个区块头包含的 2048-bit（256 字节）布隆滤波器，汇总了该区块所有 receipt 中的 logs 的地址与 topics，用来快速判定某区块是否可能包含匹配的日志（存在假阳性，但无假阴性）。

---

# 2. Logs 的结构与产生时机（执行阶段）

### 2.1. Log 的典型结构（`types.Log`）

（字段示意，实际请参考源码 `core/types`）

* `Address` — 事件发出合约地址
* `Topics`  — indexed 字段的哈希数组（最多 4 项）
* `Data`    — 非 indexed 的原始数据（ABI 编码）
* `BlockHash` / `BlockNumber` / `TxHash` / `TxIndex` / `Index`（在 receipt 中回填）

> logs 在 EVM 执行期间由 `LOG0..LOG4` 指令产生并收集到交易执行上下文中，最终被放入交易的 **Receipt** 中（receipt.Logs）。在 geth 源码中，receipt 的 logs 是在执行完交易之后由 StateDB / EVM 取出并赋值给 receipt 的。

### 2.2. 生成时机（高层流程）

1. EVM 执行交易，contract 发出 `LOG` → 日志累积到当前 tx 的执行上下文。
2. 执行完成，构建 `Receipt`：包含 `status`、`cumulativeGasUsed`、以及 `Logs` 列表。`receipt.Logs` 会被填充；随后通过 `types.CreateBloom`（或相等逻辑）为该 receipt 计算一个短布隆（receipt 层或 block 层的组合），最终 block.header.Bloom = 合并所有 receipts 的 bloom。

---

# 3.在哪里存储 Logs / Receipts / Bloom

### 3.1. Receipts（包括 logs）

* **存储位置**：LevelDB（chaindata）中的 Canonical 存储（`rawdb.WriteReceipts` / `ReadReceiptsRLP` 等接口）。也就是说，**logs 实际上是作为 receipt 的一部分写进数据库的**，并由 blockHash / blockNumber 作为键的索引项关联。

### 3.2. Block header 的 Bloom（256 字节）

* `Header.Bloom` 字段在 block header 内并随 header 一起写入 LevelDB（header 的 RLP 存储）。该 bloom 是将该区块所有 receipt（及其 logs）的地址与 topics 按规则 hash 后合入的 2048-bit 布隆。

### 3.3. “bloombits” 索引（用于跨区块快速查询）

* **为什么需要**：单看 header.Bloom 可以快速判断单个区块“是否可能”包含匹配 logs，但当用户用 `eth_getLogs` 查询几十万区块范围时，逐个读取 header 并测试 bloom 仍然昂贵。
* **Geth 的加速做法**：实现了一个叫 **bloombits** / bloom bitset 的预计算索引（`core/bloombits` 包），把区块范围分段并为每段预计算“位流”（block 篇幅的 bit-slices），从而能用位运算快速定位可能包含结果的 block 子集，然后再读取那些块的 receipts 逐一精确匹配。这个索引保存在节点数据目录（chaindata 的相应位置 / 专门的索引文件）并会在需要时构建或更新。使用这个索引可以大幅减少需要去读 receipts 的块数。

---

# 4. Bloom 的实现细节（如何计算/合并）

### 4.1. Bloom 的位宽与位映射

* **位宽**：2048 bits（= 256 bytes）。`types.Bloom` 在源码中定义为 256 字节。
* **加入元素的方式**：对一个 `address` 或 `topic` 做哈希（或直接取它的前若干位），提取若干位位置（通常 3 个位）并把这些位设为 1（CreateBloom 的实现会对每个 log 的 address 和每个 topic 都调用 `add` 操作）。`CreateBloom` 遍历所有 receipts 的 logs，把每个 log 的 address 与 topics 的 bits 加入 bloom。

### 4.2. Block bloom 的生成

* 对区块中每个交易的 receipt 先生成 receipt bloom（或者直接对所有 logs 累加），然后把这些 bloom 合并（按位 OR）得到 block.header.Bloom。`NewBlock` / block 构造路径在写区块 header 时就会设置 header.Bloom。

### 4.3. 性质与局限

* Bloom 允许假阳性（`Test` 可能返回 true，但并不保证存在），但**不会有假阴性**（若 bloom 测试为 false，则确定没有匹配的 address/topic）。因此 bloom 用作**粗过滤**。
* 由于宽度有限（2048 bit）与大量 topics/address 的组合，Bloom 的假阳性率在实际应用（大量事件）中并非很小；Geth 社区也有对其效率和准确率的讨论以及改进建议（甚至讨论过移除或改动 bloom 机制的 EIP）。

---

# 5. `eth_getLogs` / 过滤器的实现思路（Geth）

1. **输入**：地址/主题/从区块到区块范围等筛选条件。
2. **粗查（使用 bloombits / header.Bloom）**：

   * 如果启用了 bloombits 索引，先用它在区块范围内快速筛出那些可能匹配的块（位运算加速）。否则会逐块读取 header.Bloom 并测试（仍然比直接读 receipts 更轻）。
3. **精查（读取 receipts 并逐条匹配）**：对 candidate block 读取 receipts（包含 logs），逐条检查每个 log 的 `topics` 与 `address` 是否与查询条件完全匹配（这一步无假阳性）。若匹配，则返回该 log（或组合相应的 rpc 返回结构）。Receipts 本身是存储在 LevelDB 的，读取接口在 `rawdb` / `core`。

---

# 6. bloombits（位切片索引）的工作原理

* 将区块序列分成固定大小的 **section**（例如 N 个块为一 section）。对于每一 section，预先为每个 bit 位计算一个 bit-stream（即该位在 section 中每个 block 是否为 1），存成紧凑的 bitset（“bloom bits”）。
* 查询时对于给定的 address/topics，可以组合成一个查询 Bloom（hash 得到要测试的若干位），然后直接在每个 section 上按位 AND / OR 快速得到哪些 blocks 可能匹配，定位到相对少量的 blocks，再去读 receipts 做精确匹配。
* 这个预计算索引是相对耗时的（需要在后台扫描构建），但对频繁的大范围查询能极大加速。Geth 内部有 `core/bloombits` 的实现负责这套逻辑。

---

# 7. 源码查找 / 关键函数

* Bloom & types：`core/types`（`BloomByteLength`、`CreateBloom`、`MergeBloom` 等）。
* Receipt 写/读：`core/rawdb` 中的 `WriteReceipts` / `ReadReceiptsRLP` 等。
* 在构建 block 时设置 Bloom：block 构造路径（`core/types` / `core/block` 的 NewBlock 等）调用 `CreateBloom`。
* 日志收集位置：在交易应用流程里，receipt 的 logs 会由 StateDB/EVM 的日志缓冲收集并设置到 `receipt.Logs`（见 `core/state` / `core/vm` 执行路径）。示例见解析文章片段。
* bloombits 索引：`core/bloombits` 包（构建、检索位切片、Matcher 等）。

---

# 8. 最佳实践

1. **Bloom 只是粗过滤**：不要把 bloom 判定当作最终结果；必须在候选块上读取 receipts 并精确匹配。
2. **bloombits 构建有开销**：如果你看到关于 `bloombits` 的错误或升级阻塞，通常是索引构建/重建过程。日志中常见 `bloombits` 相关错误需要等待或重建索引。
3. **假阳性率**：Geth 的 2048-bit 布隆并非总是足够低的假阳性率，对于高频事件或大量 topic 的情况会导致 `eth_getLogs` 需要加载大量 receipts；社区有多个改进/讨论贴。
4. **EIP/规范演变**：有提议（例如 EIP-7668）讨论在未来移除或修改 bloom 的使用（影响长期设计），注意上游改变。

---

# 9. 总结

* Logs 被 **存进 receipts**（LevelDB）；block header 存了一个合并后的 2048-bit bloom（粗过滤）。
* `eth_getLogs` 的查询流程：**bloombits（或 header.Bloom）粗筛 → 读取 candidate blocks 的 receipts → 精确匹配 logs**。bloombits 是 Geth 为跨区块查询做的位索引加速器。
* Bloom 的性质决定了：**能快速排除不可能的块，但不能以其为最终判定（有假阳性）**。

