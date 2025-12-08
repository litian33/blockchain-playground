---


> 适用版本：ethpandaops/ethereum-genesis-generator
> 适用场景：本地私链 / EIP 测试网 / 分叉研发 / 自定义 EL+CL 联动链
> 文档目标：为研究以太坊分叉、共识、Genesis 生成机制提供完整技术说明。

---

# 目录

1. [基础链配置](#基础链配置)
2. [助记词与验证者配置](#助记词与验证者配置)
3. [时间与 Slot / Genesis 参数](#时间与-slot--genesis-参数)
4. [Fork 分叉配置（Phase0 → Electra → Future）](#fork-分叉配置phase0--electra--future)
5. [提款与退出配置](#提款与退出配置)
6. [共识层链参数（Churn / Follow Distance / Sharding）](#共识层链参数churn--follow-distance--sharding)
7. [Gossip / BLS / DASC / DA 相关参数](#gossip--bls--dasc--da-相关参数)
8. [Blob / 数据可用性（Deneb / Electra）](#blob--数据可用性deneb--electra)
9. [Fulu / BPO（Blob Pricing Override）](#fulu--bpo）blob-pricing-override)
10. [Genesis 预置账户与合约](#genesis-预置账户与合约)
11. [ENR / Shadow Fork 支持](#enr--shadow-fork-支持)
12. [同步与区块请求范围](#同步与区块请求范围)

---

# 基础链配置

### `PRESET_BASE`

* **用途**：指定共识层 preset（对应以太坊官方的配置预设）。
* **常用值**：`mainnet` / `minimal` / `local`
* **推荐**：PoC / 本地测试使用 `local`
* **作用范围**：仅 CL（Beacon 链）

---

### `CHAIN_ID`

* **用途**：指定执行层链 ID（EIP-155 保护 replay）。
* **注意**：EL 和 CL 会共享（CL 通过 execution payload）。

---

### `DEPOSIT_CONTRACT_ADDRESS`

* **用途**：Beacon 链识别的存款合约地址。
* **用法**：本地链通常设 0 地址，generator 自动生成。

---

### `DEPOSIT_CONTRACT_BLOCK`

* **用途**：CL 读取存款合约的起始 EL 区块。
* **典型值**：`0x00` 或真实区块哈希（shadow-fork 模式）

---

# 助记词与验证者配置

### `EL_AND_CL_MNEMONIC`

* **用途**：批量生成：

  * 执行层 genesis 账户（开发用于 premine）
  * 共识层验证者密钥（validator keystore）
* **注意**：此助记词可以完整控制整个链。

---

### `NUMBER_OF_VALIDATORS`

* **用途**：初始验证者数量（会为每个生成 deposit data）。
* **典型本地链**：3~50 个。

---

### `MIN_GENESIS_ACTIVE_VALIDATOR_COUNT`

* **用途**：控制创世前 Beacon 链的最低激活验证者数量。
* **作用**：触发 genesis 条件。

---

### `CL_ADDITIONAL_VALIDATORS`

* **用途**：手动追加额外验证者（弃用场景较多）。

---

### `VALIDATOR_BALANCE`

* **用途**：每个验证者的初始余额（单位 Gwei）。
* **默认**：32000000000（= 32 ETH）

---

# 时间与 Slot / Genesis 参数

### `SLOT_DURATION_IN_SECONDS`

* **用途**：决定 slot 长度（以太坊主网固定为 12s）。
* **修改影响**：影响 epoch、奖励计算、同步速度。

---

### `GENESIS_TIMESTAMP`

* **用途**：最终生成 genesis.ssz 时的时间戳。
* **注意**：必须在当前时间之后才能启动 CL。

---

### `GENESIS_DELAY`

* **用途**：generator 在生成 genesis 时增加的延迟时间。
* **典型用途**：确保 CL 启动前 genesis timestamp 不早于当前时间。

---

### `GENESIS_GASLIMIT`

* **用途**：执行层 genesis block gaslimit。
* **影响**：区块容量、测试合约部署过程。

---

### `GENESIS_DIFFICULTY`

* **用途**：兼容旧版本客户端，PoS 下为 0。

---

### `SECONDS_PER_ETH1_BLOCK`

* **用途**：CL 跟踪 EL 时估算区块时间，用于 follow distance 等计算。

---

# Fork 分叉配置（Phase0 → Electra → Future）

以太坊分叉由两个参数控制：

* `*_FORK_VERSION`：分叉后的 fork digest，影响 epoch 计算、签名域。
* `*_FORK_EPOCH`：分叉启用 epoch。

例如：

### `ALTAIR_FORK_VERSION` / `ALTAIR_FORK_EPOCH`

* Altair 是 PoS 时代第一个升级，主要增强惩罚、轻客户端支持。

---

### `BELLATRIX_FORK_VERSION` / `BELLATRIX_FORK_EPOCH`

* Bellatrix 是 Merge 前 EL+CL 接口引入的分叉。

---

### `TERMINAL_TOTAL_DIFFICULTY`

* Merge 的触发条件，如果设为 `0`，generator 会自动处理。

---

### `CAPELLA_*`

* 启用提款（Withdrawal）功能。

---

### `DENEB_*`

* EIP-4844（Blob 交易）所在分叉。

---

### `ELECTRA_*`

* 新时代升级，优化 validator churn、DA 参数。

---

### `FULU_*`

* 未来可能的升级（尚未在主网使用）。

---

### `GLOAS_*`, `EIP7805_*`, `EIP7441_*`

* 未来 fork 配置。
* epoch 设置为 `18446744073709551615` 表示“永不启用”。

---

# 提款与退出配置

### `WITHDRAWAL_TYPE`

* `0x00`: BLS withdraw
* `0x01`: ETH1 地址 withdraw（推荐）

---

### `WITHDRAWAL_ADDRESS`

* 验证者提款地址。

---

### `MIN_VALIDATOR_WITHDRAWABILITY_DELAY`

* 从激活到可提款的延迟。

---

# 共识层链参数（Churn / Follow Distance / Sharding）

### `ETH1_FOLLOW_DISTANCE`

* CL 跟随 EL 的距离（主网为 2048）。

---

### `CHURN_LIMIT_QUOTIENT`  / `MAX_PER_EPOCH_ACTIVATION_CHURN_LIMIT`

* 控制验证者每 epoch 的最大激活/退出数量。

---

### `EJECTION_BALANCE`

* 低于此余额，验证者被强制退出（单位 Gwei）。

---

### `SHARD_COMMITTEE_PERIOD`

* 旧的分片机制遗留参数。

---

# Gossip / BLS / DASC / DA 相关参数

这些 BPS（basis point）值控制 gossip “截止时间窗口”。

### 示例：

### `ATTESTATION_DUE_BPS_GLOAS`

* 表示 attestation 在 slot 中多少百分比时间前必须广播。

### `VIEW_FREEZE_CUTOFF_BPS`

* 在视图冻结前的广播截止点。

用途：

* 研究未来分叉行为
* 调整 gossip 网络容忍度
* 测试时间窗口边界条件

---

# Blob / 数据可用性（Deneb / Electra）

这些参数直接影响：

* EIP-4844 blob 的数量上限
* DA 网络带宽
* 块生成大小
* basefee 调整曲线

### 核心参数：

### `MAX_BLOBS_PER_BLOCK_ELECTRA`

* Electra 时代每个区块最大 blob 数量。

---

### `TARGET_BLOBS_PER_BLOCK_ELECTRA`

* 控制 blob basefee 调整的目标值。

---

### `BASEFEE_UPDATE_FRACTION_ELECTRA`

* basefee 调整的斜率。

---

### `DATA_COLUMN_SIDECAR_SUBNET_COUNT`

* DASC sidecar gossip 子网数量。

---

### `MAX_REQUEST_BLOB_SIDECARS_ELECTRA`

* 同步时最大 blob sidecar 请求数量。

---

# Fulu / BPO（Blob Pricing Override）

BPO 是 generator 的扩展，允许模拟多阶段的 blob 价格机制替换。

* `BPO_X_EPOCH`：阶段启用 epoch
* `BPO_X_MAX_BLOBS`：每块最大 blob 数
* `BPO_X_TARGET_BLOBS`：basefee 调整目标
* `BPO_X_BASE_FEE_UPDATE_FRACTION`：basefee 调整速度

典型用途：

* 研究激进调整策略
* 压力测试 DA 网络
* 比较 Electra 与未来 DA 模型

---

# Genesis 预置账户与合约

### `EL_PREMINE_ADDRS`

* 格式为 JSON，定制执行层预分配账户余额。
* 示例：

  ```json
  {
    "0xabc...": { "balance": "100ETH" }
  }
  ```

---

### `ADDITIONAL_PRELOADED_CONTRACTS`

* 预部署合约，支持：

  * code
  * storage
  * nonce
  * balance

用途：

* 部署系统合约
* Fork 主网状态
* 测试 Merge 前后过渡效果

---

# ENR / Shadow Fork 支持

### `BEACON_STATIC_ENR`

* 手动固定 ENR（用于多节点 P2P 引导）。

---

### `SHADOW_FORK_RPC`

* Shadow fork 上游 RPC（从实际网络复制状态）。

---

### `SHADOW_FORK_FILE`

* 从文件导入最新区块。

用途：

* 主网/holesky 对照链
* 回放攻击
* 压测 validator 流程

---

# 同步与区块请求范围

### `MIN_EPOCHS_FOR_DATA_COLUMN_SIDECARS_REQUESTS`

* DA 同步请求范围下限。

---

### `MIN_EPOCHS_FOR_BLOCK_REQUESTS`

* 区块同步请求的最小 epoch 范围。

---
