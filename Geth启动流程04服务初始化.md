# Geth启动流程解析 - 第四篇：以太坊核心服务初始化

## 1.引言

在Geth启动过程中，以太坊核心服务（eth.Ethereum）的初始化是最关键的环节之一。这部分负责创建和配置区块链的核心组件，包括区块链本身、交易池、共识引擎等。本文将深入分析这一过程的实现细节。

## 2.Ethereum服务结构

Ethereum服务是以太坊协议的核心实现，定义：

```go
// eth/backend.go
type Ethereum struct {
    // 服务配置参数，细节可参考（Geth启动流程解析 - 第二篇：配置系统详解）
	config *ethconfig.Config

	// 交易池对象
	txPool *txpool.TxPool

    // 区块链对象
	blockchain         *core.BlockChain
	// 处理器对象
	handler            *handler
	// p2p eth协议 候选对象
	ethDialCandidates  enode.Iterator
	// p2p snap协议 候选对象
	snapDialCandidates enode.Iterator
	// 此对象功能待确认
	merger             *consensus.Merger

	// 数据库操作接口
	chainDb ethdb.Database // Block chain database

	eventMux       *event.TypeMux
	// 共识引擎对象（有且仅有一个）
	engine         consensus.Engine
	// 账户管理
	accountManager *accounts.Manager

	// 接收bloom数据请求的Channel
	bloomRequests     chan chan *bloombits.Retrieval
	// bloom索引器（导入区块时使用）
	bloomIndexer      *core.ChainIndexer
	// bloom关闭通道
	closeBloomHandler chan struct{}

	// 实现了 ethapi.Backend 和 tracers.Backend 接口的后端服务对象
	APIBackend *EthAPIBackend

	// 矿工对象啊
	miner     *miner.Miner
	
	gasPrice  *big.Int
	// 矿工地址
	etherbase common.Address

	// chainID
	networkID     uint64
	// 提供网络相关的 RPC 方法
	netRPCService *ethapi.NetAPI

	// p2p服务器对象
	p2pServer *p2p.Server

	// 内存锁对象，避免并发操作，保护 gas price 、 etherbase等
	lock sync.RWMutex

	// 跟踪node关闭历史信息（非正常关闭）
	shutdownTracker *shutdowncheck.ShutdownTracker
}
```

## 3.Ethereum服务创建过程

Ethereum服务的创建：

```go
// eth/backend.go
// 这里的参数stack即前面创建的node对象，config即上面介绍的eth配置对象
func New(stack *node.Node, config *ethconfig.Config) (*Ethereum, error) {
    // 0. 一些基础合规检查
    // 同步模式只支持 Full 和 Snap
    // gasPrice 大于 0
    // 
    // ...
	// 1. 数据库初始化……

	// 2. 区块配置信息……

	// 组装eth对象
	eth := &Ethereum{
		config:            config,
		// 这个是干什么的，后面需要补充说明(TODO)
		merger:            consensus.NewMerger(chainDb),
		chainDb:           chainDb,
		eventMux:          stack.EventMux(),
		accountManager:    stack.AccountManager(),
		engine:            engine,
		closeBloomHandler: make(chan struct{}),
		networkID:         networkID,
		gasPrice:          config.Miner.GasPrice,
		etherbase:         config.Miner.Etherbase,
		bloomRequests:     make(chan chan *bloombits.Retrieval),
		bloomIndexer:      core.NewBloomIndexer(chainDb, params.BloomBitsBlocks, params.BloomConfirms),
		// 这个是node节点中创建的当前节点的p2p服务
		p2pServer:         stack.Server(),
		shutdownTracker:   shutdowncheck.NewShutdownTracker(chainDb),
	}
	
	// 1. 区块链版本检查 ...

	// 3. 链配置覆盖设置（这里主要是为了给分叉升级准备的，都是一些临时参数，分叉后会被删除）
	var overrides core.ChainOverrides
	if config.OverrideShanghai != nil {
		overrides.OverrideShanghai = config.OverrideShanghai
	}
	if config.OverrideCancun != nil {
		overrides.OverrideCancun = config.OverrideCancun
	}
	if config.OverrideVerkle != nil {
		overrides.OverrideVerkle = config.OverrideVerkle
	}
	// 4. 创建区块链实例
	eth.blockchain, err = core.NewBlockChain(eth.chainDb, eth.chainConfig, eth.engine, ...)
	
	// 这里需要设置共识引擎使用到的一些函数，如StateAt (它依赖blockchain对象)
	if congressEngine, ok := eth.engine.(*congress.Congress); ok {
		congressEngine.SetStateFn(eth.blockchain.StateAt)
	}
	// 启动bloom索引器
	eth.bloomIndexer.Start(eth.blockchain)

	// 5. 创建交易池
	blobPool := blobpool.New(config.BlobPool, eth.blockchain)
	legacyPool := legacypool.New(config.TxPool, eth.blockchain)
	eth.txPool, err = txpool.New(config.TxPool.PriceLimit, eth.blockchain, []txpool.SubPool{legacyPool, blobPool})	
	
	// 设置eth downloader在快速同步时可以使用的缓存大小
	cacheLimit := cacheConfig.TrieCleanLimit + cacheConfig.TrieDirtyLimit + cacheConfig.SnapshotLimit
	if eth.handler, err = newHandler(&handlerConfig{
		Database:       chainDb,
		Chain:          eth.blockchain,
		TxPool:         eth.txPool,
		Merger:         eth.merger,
		Network:        networkID,
		Sync:           config.SyncMode,
		BloomCache:     uint64(cacheLimit),
		EventMux:       eth.eventMux,
		RequiredBlocks: config.RequiredBlocks,
	}); err != nil {
		return nil, err
	}

	// 6. 创建矿工
	eth.miner = miner.New(eth, &config.Miner, eth.blockchain.Config(), eth.EventMux(), eth.engine, ...)
	// 这里设置矿工的初始扩展信息（例如congress共识的初始矿工列表）
	eth.miner.SetExtra(makeExtraData(config.Miner.ExtraData))

	// 设创建后端服务
	eth.APIBackend = &EthAPIBackend{stack.Config().ExtRPCEnabled(), stack.Config().AllowUnprotectedTxs, eth, nil}
	if eth.APIBackend.allowUnprotectedTxs {
		log.Info("Unprotected transactions allowed")
	}
	// 创建gas价格预言机
	gpoParams := config.GPO
	if gpoParams.Default == nil {
		gpoParams.Default = config.Miner.GasPrice
	}
	eth.APIBackend.gpo = gasprice.NewOracle(eth.APIBackend, gpoParams)

	// 设置 DNS 发现迭代器
	dnsclient := dnsdisc.NewClient(dnsdisc.Config{})
	eth.ethDialCandidates, err = dnsclient.NewIterator(eth.config.EthDiscoveryURLs...)
	if err != nil {
		return nil, err
	}
	eth.snapDialCandidates, err = dnsclient.NewIterator(eth.config.SnapDiscoveryURLs...)
	if err != nil {
		return nil, err
	}
	// 创建 RPC 服务实例
	eth.netRPCService = ethapi.NewNetAPI(eth.p2pServer, networkID)

	// 8. 注册服务（向node对象注册，可参考上一篇 Geth启动流程解析 - 第三篇：Node节点创建与初始化）
	stack.RegisterAPIs(eth.APIs())
	stack.RegisterProtocols(eth.Protocols())
	stack.RegisterLifecycle(eth)
	
	// 标记启动，并检查历史上的异常关闭标记(正常关闭会有记录，异常没有，下次启动时可以检测到)
	eth.shutdownTracker.MarkStartup()	
	return eth, nil
}
```

### 3.1. 数据库初始化

首先，Ethereum服务需要打开区块链数据库：

```go
	// 打开datadir目录下的数据库对象（如果没有，则会初始化一个）
	// 参数 "eth/db/chaindata/" 表示数据库相关的指标前缀metrics
	chainDb, err := stack.OpenDatabaseWithFreezer("chaindata", config.DatabaseCache, config.DatabaseHandles, config.DatabaseFreezer, "eth/db/chaindata/", false)
	if err != nil {
		return nil, err
	}
	// 解析状态数据库模式，支持默认、hash和path
	scheme, err := rawdb.ParseStateScheme(config.StateScheme, chainDb)
	if err != nil {
		return nil, err
	}
	// 如果为hash模式，启东时会尝试恢复状态数据修剪
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

打开数据库的逻辑：
```go
// node/node.go
// 在data目录下根据指定的name，打开或创建一个数据库， 同时还绑定一个chain freezer（用来存储ancient数据）
// 如果是临时节点，只需要创建内存数据库
func (n *Node) OpenDatabaseWithFreezer(name string, cache, handles int, ancient string, namespace string, readonly bool) (ethdb.Database, error) {
	n.lock.Lock()
	defer n.lock.Unlock()
	if n.state == closedState {
		return nil, ErrNodeStopped
	}
	var db ethdb.Database
	var err error
	if n.config.DataDir == "" {
		// 没有数据目录，则直接使用内存数据库
		db = rawdb.NewMemoryDatabase()
	} else {
		// rawdb.Open打开KV数据库，返回的db对象是抽象封装，底层可以是多个库（比如chain库和ancient库）
		// 也可以是不同类型的库实现，比如leveldb或pebble
		db, err = rawdb.Open(rawdb.OpenOptions{
			Type:              n.config.DBEngine,
			Directory:         n.ResolvePath(name),
			// 这里指定这个参数，就会同时绑定一个freezer db
			AncientsDirectory: n.ResolveAncient(name, ancient),
			Namespace:         namespace,
			Cache:             cache,
			Handles:           handles,
			ReadOnly:          readonly,
		})
	}

	if err == nil {
		// 这个增加一个外层封装，支持自动Close
		db = n.wrapDatabase(db)
	}
	return db, err
}
```

###  3.2. 区块配置信息
```go
	// 加载链配置（从创世区块0加载，如果没有则返回默认的params.MainnetChainConfig）
	chainConfig, err := core.LoadChainConfig(chainDb, config.Genesis)
	if err != nil {
		return nil, err
	}
	// 这里会创建共识引擎实例对象
	// 这里不展开链， 就是根据链配置中的参数信息，选择创建对应的共识引擎对象
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

下面是加载链配置的具体逻辑：
```go
// core/genesis.go
// 从数据库中加载链配置信息，一般为genesis创世时的配置，或者通过--override写入的配置
func LoadChainConfig(db ethdb.Database, genesis *Genesis) (*params.ChainConfig, error) {
	// 读取创世区块哈希
	stored := rawdb.ReadCanonicalHash(db, 0)
	if stored != (common.Hash{}) {
		// 只读取0块中的链配置部分，其它不关注
		storedcfg := rawdb.ReadChainConfig(db, stored)
		if storedcfg != nil {
			// 如果读取到，则直接返回
			return storedcfg, nil
		}
	}

	// 如果没有读取到创世区块的链配置，且指定链创世配置对象参数
	if genesis != nil {
		// 读取不到，也没指定，返回空
		if genesis.Config == nil {
			return nil, errGenesisNoConfig
		}

		// 如果创世区块哈希和参数指定的创世对象不匹配，则报错
		if stored != (common.Hash{}) && genesis.ToBlock().Hash() != stored {
			return nil, &GenesisMismatchError{stored, genesis.ToBlock().Hash()}
		}
		// 直接返回指定参数中的配置
		return genesis.Config, nil
	}

	// 以上条件都不满足的情况下，返回主网默认配置
	return params.MainnetChainConfig, nil
}
```

### 3.3. 区块链实例创建

区块链实例是Ethereum服务的核心组件之一：

```go
eth.blockchain, err = core.NewBlockChain(eth.chainDb, eth.chainConfig, eth.engine, ...)
```

函数创建区块链对象(TODO 这里只列出大概，后面还要单独介绍区块链对象)：

```go
// core/blockchain.go
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

### 3.4.交易池初始化

交易池负责管理待处理的交易(TODO 后面还要有交易池专题)：

目前有两种交易池，传统交易池和blob交易池，分别初始化并统一放到eth.txPool
```go
	// blob交易需要文件存储
	if config.BlobPool.Datadir != "" {
		config.BlobPool.Datadir = stack.ResolvePath(config.BlobPool.Datadir)
	}
	blobPool := blobpool.New(config.BlobPool, eth.blockchain)

	if config.TxPool.Journal != "" {
		config.TxPool.Journal = stack.ResolvePath(config.TxPool.Journal)
	}
	legacyPool := legacypool.New(config.TxPool, eth.blockchain)

	eth.txPool, err = txpool.New(config.TxPool.PriceLimit, eth.blockchain, []txpool.SubPool{legacyPool, blobPool})
```

创建交易池实例：

```go
// 交易池用来收集、排序、过滤 网络过来的所有交易
// core/txpool/txpool.go
func New(gasTip uint64, chain BlockChain, subpools []SubPool) (*TxPool, error) {
	// 获取当前区块头，统一子池的开始时间
	head := chain.CurrentBlock()

	pool := &TxPool{
		subpools:     subpools,
		reservations: make(map[common.Address]SubPool),
		quit:         make(chan chan error),
		term:         make(chan struct{}),
		sync:         make(chan chan error),
	}
	for i, subpool := range subpools {
		// 初始化每个子池，接口都一样
		if err := subpool.Init(gasTip, head, pool.reserver(i, subpool)); err != nil {
			for j := i - 1; j >= 0; j-- {
				subpools[j].Close()
			}
			return nil, err
		}
	}
	// 启动交易池住循环
	go pool.loop(head, chain)
	return pool, nil
}
```

### 3.5. 矿工初始化

矿工负责区块打包和共识过程：

```go
	// 6. 创建矿工
	eth.miner = miner.New(eth, &config.Miner, eth.blockchain.Config(), eth.EventMux(), eth.engine, ...)
	// 这里设置矿工的初始扩展信息（例如congress共识的初始矿工列表）
	eth.miner.SetExtra(makeExtraData(config.Miner.ExtraData))
```

创建矿工实例：

```go
func New(eth Backend, config *Config, chainConfig *params.ChainConfig, mux *event.TypeMux, engine consensus.Engine, isLocalBlock func(header *types.Header) bool) *Miner {
	miner := &Miner{
		mux:     mux,
		eth:     eth,
		engine:  engine,
		exitCh:  make(chan struct{}),
		startCh: make(chan struct{}),
		stopCh:  make(chan struct{}),
		// 矿工的主要逻辑主要在worker中实现（TODO 这里也需要专题介绍）
		worker:  newWorker(config, chainConfig, engine, eth, mux, isLocalBlock, true),
	}
	miner.wg.Add(1)
	// 启动矿工循环
	go miner.update()
	return miner
}
```

## 4. 总结

Ethereum服务的初始化过程创建了以太坊节点的所有核心组件：

1. **数据库层** - 提供数据持久化能力
2. **区块链层** - 管理区块和状态数据
3. **交易池层** - 管理待处理交易
4. **网络层** - 处理P2P通信和数据同步
5. **共识层** - 实现共识算法
6. **应用层** - 提供API接口

这些组件协同工作，形成了一个完整的以太坊节点实现。每个组件都有明确的职责边界，通过精心设计的接口进行交互，体现了良好的软件工程实践。

革命尚未成功，同志仍需努力！
