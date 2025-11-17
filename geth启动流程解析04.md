# Geth启动流程解析 - 第四篇：以太坊核心服务初始化

## 引言

在Geth启动过程中，以太坊核心服务（eth.Ethereum）的初始化是最关键的环节之一。这部分负责创建和配置区块链的核心组件，包括区块链本身、交易池、共识引擎等。本文将深入分析这一过程的实现细节。

## Ethereum服务结构

Ethereum服务是以太坊协议的核心实现，定义：

```go
// eth/backend.go
type Ethereum struct {
    // 服务配置参数
	config *ethconfig.Config

	// 交易池对象
	txPool *txpool.TxPool

    // 区块链对象
	blockchain         *core.BlockChain
	handler            *handler
	ethDialCandidates  enode.Iterator
	snapDialCandidates enode.Iterator
	merger             *consensus.Merger

	// 数据库操作接口
	chainDb ethdb.Database // Block chain database

	eventMux       *event.TypeMux
	engine         consensus.Engine
	accountManager *accounts.Manager

	bloomRequests     chan chan *bloombits.Retrieval // Channel receiving bloom data retrieval requests
	bloomIndexer      *core.ChainIndexer             // Bloom indexer operating during block imports
	closeBloomHandler chan struct{}

	APIBackend *EthAPIBackend

	miner     *miner.Miner
	gasPrice  *big.Int
	etherbase common.Address

	networkID     uint64
	netRPCService *ethapi.NetAPI

	p2pServer *p2p.Server

	lock sync.RWMutex // Protects the variadic fields (e.g. gas price and etherbase)

	shutdownTracker *shutdowncheck.ShutdownTracker // Tracks if and when the node has shutdown ungracefully
}
```

## Ethereum服务创建过程

Ethereum服务的创建：

```go
func New(stack *node.Node, config *ethconfig.Config) (*Ethereum, error) {
    // 0. 一些基础合规检查
    // 同步模式只支持 Full 和 Snap
    // gasPrice 大于 0
    // 
    // ...
	// 1. 创建Ethereum实例
	eth := &Ethereum{
		config:            config,
		chainDb:           stack.OpenDatabaseWithFreezer("chaindata", config.DatabaseCache, config.DatabaseHandles, config.DatabaseFreezer, "eth/db/chaindata/", false),
		eventMux:          stack.EventMux(),
		accountManager:    stack.AccountManager(),
		engine:            ethconfig.CreateConsensusEngine(stack, config),
		shutdownTracker:   shutdowntracker.New(),
		networkID:         config.NetworkId,
		gasPrice:          config.Miner.GasPrice,
		etherbase:         config.Miner.Etherbase,
		bloomRequests:     make(chan chan *bloombits.Retrieval),
		bloomIndexer:      core.NewBloomIndexer(stack.OpenDatabase("chainindex", 0, 0, "eth/db/chainindex/", false)),
	}
	
	// 2. 加载创世区块配置
	eth.chainConfig, eth.genesisHash, err = core.SetupGenesisBlock(eth.chainDb, config.Genesis)
	
	// 3. 创建区块链实例
	eth.blockchain, err = core.NewBlockChain(eth.chainDb, eth.chainConfig, eth.engine, ...)
	
	// 4. 创建交易池
	eth.txPool = core.NewTxPool(config.TxPool, eth.chainConfig, eth.blockchain)
	
	// 5. 创建协议管理器
	eth.protocolManager, err = NewProtocolManager(...)
	
	// 6. 创建下载器
	eth.downloader = downloader.New(...)
	
	// 7. 创建矿工
	eth.miner = miner.New(eth, &config.Miner, eth.chainConfig, eth.EventMux(), eth.engine, ...)
	
	// 8. 注册服务
	stack.RegisterProtocols(eth.Protocols())
	stack.RegisterLifecycle(eth)
	
	return eth, nil
}
```

## 数据库初始化

首先，Ethereum服务需要打开区块链数据库：

```go
	chainDb, err := stack.OpenDatabaseWithFreezer("chaindata", config.DatabaseCache, config.DatabaseHandles, config.DatabaseFreezer, "eth/db/chaindata/", false)
	if err != nil {
		return nil, err
	}
    // 解析数据库 schema，默认为 hash，支持 hash 和 path 两种模式
	scheme, err := rawdb.ParseStateScheme(config.StateScheme, chainDb)
	if err != nil {
		return nil, err
	}
    // 在hash模式下，尝试恢复离线数据
	if scheme == rawdb.HashScheme {
		if err := pruner.RecoverPruning(stack.ResolvePath(""), chainDb); err != nil {
			log.Error("Failed to recover state", "error", err)
		}
	}
```

这里创建出来的 chainDb 实例会作为Ethereum服务的数据库，用于访问区块链数据。

下面还会进行数据库区块链版本兼容性检查：
所谓的区块链版本其实就是区块链对应的数据库存储结构，不管逻辑如何变化，如果最终存储结构没有变化，则这个版本不会发生变化，否则就会触发版本的升级。一般的升级是兼容的。
```go
	bcVersion := rawdb.ReadDatabaseVersion(chainDb)
	var dbVer = "<nil>"
	if bcVersion != nil {
		dbVer = fmt.Sprintf("%d", *bcVersion)
	}
	log.Info("Initialising Ethereum protocol", "network", networkID, "dbversion", dbVer)

	if !config.SkipBcVersionCheck {
		if bcVersion != nil && *bcVersion > core.BlockChainVersion {
            // 如果现有数据版本比代码新，则无法兼容
			return nil, fmt.Errorf("database version is v%d, Geth %s only supports v%d", *bcVersion, params.VersionWithMeta, core.BlockChainVersion)
		} else if bcVersion == nil || *bcVersion < core.BlockChainVersion {
            // 如果数据库版本为空或者小于当前代码版本，则可以进行升级
			if bcVersion != nil { // only print warning on upgrade, not on init
				log.Warn("Upgrade blockchain database version", "from", dbVer, "to", core.BlockChainVersion)
			}
            // 写入升级后的数据版本
			rawdb.WriteDatabaseVersion(chainDb, core.BlockChainVersion)
		}
	}
```

##  区块配置信息
```go
	// 这里加载ChainConfig，附加genesis配置
	chainConfig, err := core.LoadChainConfig(chainDb, config.Genesis)
	if err != nil {
		return nil, err
	}
	// 这里第一次创建共识引擎
	engine, err := ethconfig.CreateConsensusEngine(chainConfig, chainDb)
	if err != nil {
		return nil, err
	}
    // 如果未指定网络ID，则使用链配置的ChainID
	networkID := config.NetworkId
	if networkID == 0 {
		networkID = chainConfig.ChainID.Uint64()
	}
```

## 区块链实例创建

区块链实例是Ethereum服务的核心组件之一：

```go
eth.blockchain, err = core.NewBlockChain(eth.chainDb, eth.chainConfig, eth.engine, ...)
```

函数创建区块链对象：

```go
func NewBlockChain(db ethdb.Database, chainConfig *params.ChainConfig, engine consensus.Engine, ...) (*BlockChain, error) {
	// 1. 创建BlockChain实例
	bc := &BlockChain{
		chainConfig: chainConfig,
		cacheConfig: cacheConfig,
		db:          db,
		triegc:      prque.New(nil),
		stateCache: state.NewDatabaseWithConfig(db, &trie.Config{
			Cache:     cacheConfig.TrieCleanLimit,
			Journal:   cacheConfig.TrieCleanJournal,
			Preimages: cacheConfig.Preimages,
		}),
		quit:          make(chan struct{}),
		chainHeadFeed: new(event.Feed),
		chainSideFeed: new(event.Feed),
		// ...
	}
	
	// 2. 初始化当前头部区块
	bc.currentBlock.Store(bc.LoadLastState())
	
	// 3. 初始化区块验证器
	bc.validator = NewBlockValidator(chainConfig, bc, engine)
	
	// 4. 初始化状态处理器
	bc.processor = NewStateProcessor(chainConfig, bc, engine)
	
	// 5. 初始化布隆过滤器索引器
	bc.bloomIndexer.Start(bc)
	
	return bc, nil
}
```

## 交易池初始化

交易池负责管理待处理的交易：

```go
eth.txPool = core.NewTxPool(config.TxPool, eth.chainConfig, eth.blockchain)
```

创建交易池实例：

```go
func NewTxPool(config TxPoolConfig, chainconfig *params.ChainConfig, chain BlockChain) *TxPool {
	// 1. 验证配置
	config = (&config).check()
	
	// 2. 创建TxPool实例
	pool := &TxPool{
		config:      config,
		chainconfig: chainconfig,
		chain:       chain,
		minGasPrice: minGasPrice,
		eventFeed:   new(event.Feed),
		signer:      types.LatestSigner(chainconfig),
		events:      events,
		locals:      newAccountSet(),
		journal:     newTxJournal(config.Journal),
		pending:     make(map[common.Address]*txList),
		queue:       make(map[common.Address]*txList),
		beats:       make(map[common.Address]time.Time),
		all:         newTxLookup(),
		reqResetCh:  make(chan *txpoolResetRequest),
		reqPromoteCh: make(chan *accountSet),
		queueTxEventCh: make(chan *types.Transaction),
	}
	
	// 3. 加载本地交易日志
	if config.Journal != "" {
		pool.journal = newTxJournal(config.Journal)
	}
	
	// 4. 启动事件循环
	go pool.loop()
	
	return pool
}
```

## 协议管理器初始化

协议管理器负责处理网络层面的区块和交易同步：

```go
eth.protocolManager, err = NewProtocolManager(...)
```

创建协议管理器：

```go
func NewProtocolManager(config *params.ChainConfig, mode downloader.SyncMode, networkID uint64, 
	eth *Ethereum, engine consensus.Engine, blockchain *core.BlockChain, 
	cacheLimit int, txpool *core.TxPool) (*ProtocolManager, error) {
	
	// 1. 创建ProtocolManager实例
	manager := &ProtocolManager{
		networkID:   networkID,
		eth:         eth,
		txpool:      txpool,
		blockchain:  blockchain,
		chainconfig: config,
		downloader:  downloader.New(...),
		fetcher:     fetcher.New(...),
		peers:       newPeerSet(),
		SubProtocols: []p2p.Protocol{
			{
				Name:    "eth",
				Version: 65,
				Length:  ProtocolLengths[65],
				Run: func(p *p2p.Peer, rw p2p.MsgReadWriter) error {
					return pm.handle(p, rw, 65)
				},
				NodeInfo: func() interface{} {
					return pm.NodeInfo()
				},
				PeerInfo: func(id enode.ID) interface{} {
					return pm.PeersInfo()[id.String()]
				},
			},
		},
	}
	
	return manager, nil
}
```

## 下载器初始化

下载器负责从其他节点下载区块数据：

```go
eth.downloader = downloader.New(...)
```

## 矿工初始化

矿工负责区块打包和共识过程：

```go
eth.miner = miner.New(eth, &config.Miner, eth.chainConfig, eth.EventMux(), eth.engine, ...)
```

创建矿工实例：

```go
func New(eth Backend, config *Config, chainConfig *params.ChainConfig, mux *event.TypeMux, engine consensus.Engine, isLocalBlock func(block *types.Block) bool) *Miner {
	// 1. 创建Miner实例
	miner := &Miner{
		eth:            eth,
		mux:            mux,
		engine:         engine,
		exitCh:         make(chan struct{}),
		startCh:        make(chan common.Address),
		stopCh:         make(chan struct{}),
		worker:         newWorker(config, chainConfig, engine, eth, mux, isLocalBlock, true),
		canStart:       1,
		shouldStart:    0,
	}
	
	// 2. 启动事件循环
	go miner.update()
	
	return miner
}
```

## API后端注册

最后，注册API后端以提供RPC服务：

```go
stack.RegisterAPIs([]rpc.API{
	{
		Namespace: "eth",
		Version:   "1.0",
		Service:   NewPublicEthereumAPI(s),
		Public:    true,
	}, {
		Namespace: "eth",
		Version:   "1.0",
		Service:   NewPublicMinerAPI(s),
		Public:    true,
	}, {
		Namespace: "eth",
		Version:   "1.0",
		Service:   NewPublicNetAPI(s),
		Public:    true,
	}, {
		Namespace: "personal",
		Version:   "1.0",
		Service:   NewPrivateAccountAPI(s),
		Public:    false,
	},
})
```

## 总结

Ethereum服务的初始化过程创建了以太坊节点的所有核心组件：

1. **数据库层** - 提供数据持久化能力
2. **区块链层** - 管理区块和状态数据
3. **交易池层** - 管理待处理交易
4. **网络层** - 处理P2P通信和数据同步
5. **共识层** - 实现共识算法
6. **应用层** - 提供API接口

这些组件协同工作，形成了一个完整的以太坊节点实现。每个组件都有明确的职责边界，通过精心设计的接口进行交互，体现了良好的软件工程实践。