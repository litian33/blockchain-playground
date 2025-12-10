下面内容基于官方文档整理，适用于在自定义 L1 上用 OP Deployer 部署 OP/Superchain（参考文档: [Optimism Custom Deployments](https://docs.optimism.io/chain-operators/tools/op-deployer/reference/custom-deployments)）。

## 0. 总览
流程分三步：1) Superchain 引导（bootstrap superchain）→ 2) 实现引导（bootstrap implementations，部署 OPCM 工厂和实现）→ 3) 部署 L2（apply）。后续升级用 upgrade 系列命令。

## 1. 前置准备
- L1 RPC 可访问且正常出块。
- 部署者私钥（EOA），角色地址最好用多签/SAFE。
- 选择合约版本定位器 `--artifacts-locator`（如 `tag://op-contracts/v2.0.0`）。
- 为 Superchain 关键角色准备地址：`superchain-proxy-admin-owner`、`protocol-versions-owner`、`guardian`。

## 步骤 1：Superchain Bootstrap
在目标 L1 上部署超级链管理合约（SuperchainConfig、ProtocolVersions、ProxyAdmin）。
```bash
op-deployer bootstrap superchain \
  --l1-rpc-url "<L1 RPC>" \
  --private-key "<部署私钥>" \
  --artifacts-locator "<locator>" \
  --outfile "superchain.json" \
  --superchain-proxy-admin-owner "<角色地址/SAFE>" \
  --protocol-versions-owner "<角色地址/SAFE>" \
  --guardian "<角色地址/SAFE>"
```
输出 `superchain.json`，内含各合约地址。注意：
- 部署者仅负责部署，控制权在角色地址。
- 角色应为智能合约（如 SAFE），便于后续升级。

```shell
op-deployer bootstrap superchain \
  --l1-rpc-url "http://192.168.8.8:9545" \
  --private-key "0x" \
  --artifacts-locator "embedded" \
  --outfile "superchain.json" \
  --superchain-proxy-admin-owner "0xf23F68Bf489A8aBf36484eD2AF7223746303612D" \
  --protocol-versions-owner "0xf23F68Bf489A8aBf36484eD2AF7223746303612D" \
  --guardian "0xf23F68Bf489A8aBf36484eD2AF7223746303612D"
```

## 步骤 2：实现 Bootstrap（含 OPCM）
部署共享实现和 OP Contracts Manager (OPCM) 工厂，使用 CREATE2 可复用地址。
```bash
op-deployer bootstrap implementations \
  --artifacts-locator "<与上一步相同>" \
  --l1-rpc-url "<L1 RPC>" \
  --outfile "impls.json" \
  --mips-version "2" \
  --private-key "<部署私钥>" \
  --protocol-versions-proxy "<来自 superchain.json>" \
  --superchain-config-proxy "<来自 superchain.json>" \
  --upgrade-controller "<superchain-proxy-admin-owner>"
```
输出 `impls.json`，核心是 `opcm` 地址。重要约束：
- 每个 Superchain 对应一套 OPCM（不能跨 L1 共用）。
- 不同合约版本对应不同 OPCM。


```bash
op-deployer bootstrap implementations \
  --artifacts-locator "embedded" \
  --l1-rpc-url "http://192.168.8.8:9545" \
  --outfile "impls.json" \
  --mips-version "7" \
  --private-key "0x" \
  --protocol-versions-proxy "0x03ef4b5fff349d1175162eec637827219c298fcb" \
  --superchain-config-proxy "0x912555f2f82ba63d634b11a173fac93d2401d7ba" \
  --upgrade-controller "0xf23F68Bf489A8aBf36484eD2AF7223746303612D" \
  --superchain-proxy-admin "0x85215d22f988dea813c3d8c78a8ac94b811c4f80" \
  --challenger "0xf23F68Bf489A8aBf36484eD2AF7223746303612D"
```

## 步骤 3：部署 L2（apply）

### 3.1 创建 L2 部署文件
使用下面的命令创建intent.toml文件，注意这里的类型为`custom`，因为这是我们自定义 L1，如果使用通用 L1，这里可以为`standard-overrides`：
```shell
op-deployer init \
  --l1-chain-id 2025 \
  --l2-chain-ids 2026 \
  --workdir .deployer \
  --intent-type custom
```

然后修改生成的intent.toml文件内容，将`opcmAddress`替换为之前生成的地址，并将其他地址修改为自己可以控制的地址。

我修改后的内容如下：
```ini
configType = "custom"
l1ChainID = 2025
fundDevAccounts = false
l1ContractsLocator = "embedded"
l2ContractsLocator = "embedded"
opcmAddress = "0x3827c776bc9f803c221d8b39a9b8ba1ab48dbde3"

[[chains]]
  id = "0x00000000000000000000000000000000000000000000000000000000000007ea"
  baseFeeVaultRecipient = "0xf23F68Bf489A8aBf36484eD2AF7223746303612D"
  l1FeeVaultRecipient = "0xf23F68Bf489A8aBf36484eD2AF7223746303612D"
  sequencerFeeVaultRecipient = "0xf23F68Bf489A8aBf36484eD2AF7223746303612D"
  eip1559DenominatorCanyon = 250
  eip1559Denominator = 50
  eip1559Elasticity = 10
  gasLimit = 60000000
  operatorFeeScalar = 0
  operatorFeeConstant = 0
  [chains.roles]
    l1ProxyAdminOwner = "0xf23F68Bf489A8aBf36484eD2AF7223746303612D"
    l2ProxyAdminOwner = "0xf23F68Bf489A8aBf36484eD2AF7223746303612D"
    systemConfigOwner = "0xf23F68Bf489A8aBf36484eD2AF7223746303612D"
    unsafeBlockSigner = "0xf23F68Bf489A8aBf36484eD2AF7223746303612D"
    batcher = "0xf23F68Bf489A8aBf36484eD2AF7223746303612D"
    proposer = "0xf23F68Bf489A8aBf36484eD2AF7223746303612D"
    challenger = "0xf23F68Bf489A8aBf36484eD2AF7223746303612D"
```

### 3.2 设置环境变量
```shell
# Create .env file
cat << 'EOF' > .env
# Your L1 RPC URL (e.g., from Alchemy, Infura)
L1_RPC_URL=https://192.168.8.8:9545

# Private key for deployment.
# Get this from your self-custody wallet, like Metamask.
PRIVATE_KEY=0x
EOF
```

然后执行如下命令使之生效：
```shell
source .env
```

### 3.3 部署 L1
使用下面命令部署 L1 层合约：
```shell
op-deployer apply \
  --workdir .deployer \
  --l1-rpc-url $L1_RPC_URL \
  --private-key $PRIVATE_KEY
```

命令执行完成之后会在.deployer目录下生成state.json，则表示初始化成功。

### 3.4 生成创世配置

使用如下命令生成创世配置，以后的步骤会用到：
```shell
# Generate genesis and rollup configs
op-deployer inspect genesis --workdir .deployer <YOUR_CHAIN_ID> > .deployer/genesis.json
op-deployer inspect rollup --workdir .deployer <YOUR_CHAIN_ID> > .deployer/rollup.json
```

执行如下：
```shell
op-deployer inspect genesis --workdir .deployer 2026 > .deployer/genesis.json
op-deployer inspect rollup --workdir .deployer 2026 > .deployer/rollup.json
```

然后就大功告成，部署文件生成，可以开始正式部署节点服务了！

下一步参考官方文档即可：https://docs.optimism.io/chain-operators/tutorials/create-l2-rollup/op-geth-setup

## 升级（可选）
- 先用对应版本的 `bootstrap implementations` 为新版本部署 OPCM。
- 再用 `op-deployer upgrade <version>` 生成升级所需 calldata（不直接上链）。
- 将生成的 calldata 由链的 owner（通常是 SAFE）通过 `DELEGATECALL` 方式执行到 OPCM。

示例升级命令：
```bash
op-deployer upgrade <version> \
  --config <upgrade-config.json> \
  --l1-rpc-url "<L1 RPC>"
```
`upgrade-config.json` 需包含：
```
{
  "prank": "<链的 owner/SAFE>",
  "opcm": "<新 OPCM 地址>",
  "chainConfigs": [
    {
      "systemConfigProxy": "<该链的 system config proxy>",
      "proxyAdmin": "<该链的 proxy admin>",
      "absolutePrestate": "<32 字节哈希>"
    }
  ]
}
```

## 验证与注意事项
- 始终使用与 bootstrap 相同的 `artifacts-locator`，否则版本不匹配。
- 部署/升级后用 RPC `eth_getCode` 核验关键代理/实现是否有字节码。
- OPCM 必须与正确的 Superchain 绑定；调用错误的 OPCM 会导致版本或归属错误。
- 角色地址建议用 SAFE，部署者 EOA 不再保有控制权。
- 如果交易未落链，检查：节点是否出块、私钥余额、chainId 是否匹配、是否需要自动确认/自动挖矿。
