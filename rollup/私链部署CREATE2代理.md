forge在部署合约时，默认会使用位于 0x4e59b44847b379578588920ca78fbf26c0b4956c 地址上的CREATE2代理合约。

这个合约在主流公链上已经存在，可以直接使用，但是在自己搭建的开发环境或私链上不存在，使用forge时会有一些问题，为了达到和主网同样的体验，需要自己在私链上部署相同的合约。

## 简化版本

1. Geth启动参数增加如下参数，不然未绑定链ID的交易无法发送：

```shell
--rpc.allow-unprotected-txs
```

2. 给这个地址充值手续费

```shell
0x3fab184622dc19b6109349b94811493bf2a45362
```

3. 发送签名交易
```shell
curl --location 'http://127.0.0.1:9545' \
--header 'Content-Type: application/json' \
--data '{
	"jsonrpc":"2.0",
	"method":"eth_sendRawTransaction",
	"params":["0xf8a58085174876e800830186a08080b853604580600e600039806000f350fe7fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffe03601600081602082378035828234f58015156039578182fd5b8082525050506014600cf31ba02222222222222222222222222222222222222222222222222222222222222222a02222222222222222222222222222222222222222222222222222222222222222"],
	"id":1
}'
```

4. 检查合约状态
> 如果返回不为0x0，则表明合约部署成功
```shell
curl --location 'http://127.0.0.1:9545' \
--header 'Content-Type: application/json' \
--data '{
	"jsonrpc":"2.0",
	"method":"eth_getCode",
	"params":[
		"0x4e59b44847b379578588920ca78fbf26c0b4956c", 
		"latest"
	],
	"id":1
}'
```


## 原始版本

来源于开源项目：https://github.com/Arachnid/deterministic-deployment-proxy

