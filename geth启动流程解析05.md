# Geth启动流程解析 - 第五篇：节点启动与服务运行

## 引言

在完成了Node和Ethereum等核心服务的创建之后，Geth需要启动这些服务使其开始工作。本文将深入分析Geth如何启动节点以及各个服务是如何运行的。

## 节点启动流程

节点启动的入口函数：

```go
func startNode(ctx *cli.Context, stack *node.Node, backend ethapi.Backend, isConsole bool) {
	// 启动 Node
	utils.StartNode(ctx, stack, isConsole)

	// 解锁账户
	unlockAccounts(ctx, stack)

	// 注册钱包事件
	events := make(chan accounts.WalletEvent, 16)
	stack.AccountManager().Subscribe(events)

	// 创建直连本地 geth node 的 rpc 客户端
	rpcClient := stack.Attach()
	ethClient := ethclient.NewClient(rpcClient)

	go func() {
		// 打开所有钱包
		for _, wallet := range stack.AccountManager().Wallets() {
			if err := wallet.Open(""); err != nil {
				log.Warn("Failed to open wallet", "url", wallet.URL(), "err", err)
			}
		}
		// Listen for wallet event till termination
		for event := range events {
			switch event.Kind {
			case accounts.WalletArrived:
				if err := event.Wallet.Open(""); err != nil {
					log.Warn("New wallet appeared, failed to open", "url", event.Wallet.URL(), "err", err)
				}
			case accounts.WalletOpened:
				status, _ := event.Wallet.Status()
				log.Info("New wallet appeared", "url", event.Wallet.URL(), "status", status)

				var derivationPaths []accounts.DerivationPath
				if event.Wallet.URL().Scheme == "ledger" {
					derivationPaths = append(derivationPaths, accounts.LegacyLedgerBaseDerivationPath)
				}
				derivationPaths = append(derivationPaths, accounts.DefaultBaseDerivationPath)

				event.Wallet.SelfDerive(derivationPaths, ethClient)

			case accounts.WalletDropped:
				log.Info("Old wallet dropped", "url", event.Wallet.URL())
				event.Wallet.Close()
			}
		}
	}()

	// Start auxiliary services if enabled
	if ctx.Bool(utils.MiningEnabledFlag.Name) {
		// Mining only makes sense if a full Ethereum node is running
		if ctx.String(utils.SyncModeFlag.Name) == "light" {
			utils.Fatalf("Light clients do not support mining")
		}
		ethBackend, ok := backend.(*eth.EthAPIBackend)
		if !ok {
			utils.Fatalf("Ethereum service not running")
		}
		// 矿工节点使用矿工费作为txpool的门限值，不接收低于矿工费的交易
		// Set the gas price to the limits from the CLI and start mining
		gasprice := flags.GlobalBig(ctx, utils.MinerGasPriceFlag.Name)
		ethBackend.TxPool().SetGasTip(gasprice)

		// 这里启动矿工节点挖矿模式
		if err := ethBackend.StartMining(); err != nil {
			utils.Fatalf("Failed to start mining: %v", err)
		}
	}
}
```

## Node.Start()详解

stack.Start()是启动过程的核心：

```go
func (n *Node) Start() error {
	n.startStopLock.Lock()
	defer n.startStopLock.Unlock()

	n.stateLock.Lock()
	switch n.state {
	case stateOpening, stateStarted, stateClosed:
		n.stateLock.Unlock()
		return ErrNodeRunning
	}
	n.state = stateOpening
	n.stateLock.Unlock()
	
	// 启动所有注册的服务
	if err := n.startServices(); err != nil {
		n.doClose(nil)
		return err
	}
	
	// 启动P2P服务器
	if err := n.server.Start(); err != nil {
		n.doClose(nil)
		return err
	}
	
	// 启动RPC服务
	if err := n.startRPC(); err != nil {
		n.doClose(nil)
		return err
	}
	
	// 标记为已启动
	n.stateLock.Lock()
	n.state = stateStarted
	n.stateLock.Unlock()
	
	return nil
}
```

### 1. 启动注册的服务

```go
func (n *Node) startServices() error {
	// 实例化所有已注册的服务
	for _, constructor := range n.serviceFuncs {
		// 调用服务构造函数
		ctx := &ServiceContext{
			Config:         *n.config,
			Services:       make(map[reflect.Type]Service),
			EventMux:       n.eventmux,
			AccountManager: n.accman,
		}
		// 复制已创建的服务到context中
		for kind, s := range n.services {
			ctx.Services[kind] = s
		}
		
		// 创建新服务
		service, err := constructor(ctx)
		if err != nil {
			return err
		}
		
		// 获取服务类型
		kind := reflect.TypeOf(service)
		if _, exists := n.services[kind]; exists {
			return &DuplicateServiceError{Kind: kind}
		}
		
		// 保存服务实例
		n.services[kind] = service
	}
	
	// 启动各个服务
	for _, service := range n.services {
		// 启动服务
		if err := service.Start(); err != nil {
			return err
		}
	}
	
	return nil
}
```

对于Ethereum服务，其方法实现如下：

```go
func (s *Ethereum) Start() error {
	// 启动布隆过滤器处理
	s.startBloomHandlers(params.BloomBitsBlocks)
	
	// 启动协议管理器
	if s.protocolManager != nil {
		s.protocolManager.Start()
	}
	
	// 启动下载器
	if s.downloader != nil {
		s.downloader.Start()
	}
	
	// 启动交易池
	s.txPool.Start()
	
	// 启动矿工
	s.miner.Start()
	
	// 启动自动挖矿（如果配置）
	if s.config.Miner.StartAutomatically {
		s.StartMining()
	}
	
	return nil
}
```

### 2. 启动P2P服务器

```go
if err := n.server.Start(); err != nil {
	n.doClose(nil)
	return err
}
```

P2P服务器的启动包括：

1. 启动网络监听
2. 启动节点发现机制
3. 启动各种网络协议处理器

### 3. 启动RPC服务

```go
func (n *Node) startRPC() error {
	// 收集所有API
	apis := n.apis()
	
	// 启动各种RPC服务
	if err := n.startInProc(apis); err != nil {
		return err
	}
	
	if err := n.startIPC(apis); err != nil {
		n.stopInProc()
		return err
	}
	
	if err := n.startHTTP(n.httpEndpoint, apis, n.httpModules, n.httpCors, n.httpVhosts, n.httpTimeouts); err != nil {
		n.stopIPC()
		n.stopInProc()
		return err
	}
	
	if err := n.startWS(n.wsEndpoint, apis, n.wsModules, n.wsOrigins, n.wsExposeAll); err != nil {
		n.stopHTTP()
		n.stopIPC()
		n.stopInProc()
		return err
	}
	
	return nil
}
```

## 账户解锁

如果配置了自动解锁账户，会执行以下操作：

```go
func unlockAccounts(ctx *cli.Context, stack *node.Node) {
	// 获取账户管理器
	am := stack.AccountManager()
	
	// 获取要解锁的账户
	passwords := utils.MakePasswordList(ctx)
	unlocks := strings.Split(ctx.GlobalString(utils.UnlockedAccountFlag.Name), ",")
	for i, account := range unlocks {
		if trimmed := strings.TrimSpace(account); trimmed != "" {
			// 解锁账户
			unlockAccount(ctx, am, trimmed, i, passwords)
		}
	}
}
```

## 挖矿启动

如果配置了自动挖矿，会启动挖矿服务：

```go
func startMining(ctx *cli.Context, stack *node.Node, backend ethapi.Backend) {
	// 检查是否配置了挖矿
	if !ctx.GlobalIsSet(utils.MiningEnabledFlag.Name) {
		return
	}
	
	// 启动挖矿
	if err := backend.StartMining(ctx.GlobalInt(utils.MinerThreadsFlag.Name)); err != nil {
		Fatalf("Failed to start mining: %v", err)
	}
}
```

在Ethereum服务中，StartMining()方法实现如下：

```go
func (s *Ethereum) StartMining(threads int) error {
	// 设置挖矿线程数
	if threads == 0 {
		threads = runtime.NumCPU()
	}
	
	// 配置挖矿参数
	s.miner.SetThreads(threads)
	
	// 设置etherbase
	eb, err := s.Etherbase()
	if err != nil {
		return err
	}
	s.miner.SetEtherbase(eb)
	
	// 启动挖矿
	s.miner.Start()
	return nil
}
```

## 服务运行机制

### 区块链同步

Ethereum服务启动后，会通过协议管理器进行区块同步：

```go
// 在ProtocolManager.Start()中
go pm.syncer()
go pm.txsyncLoop()
```

### 交易池运行

交易池启动后会运行事件处理循环：

```go
// 在TxPool.Start()中
go pool.scheduleReorgLoop()
go pool.feedPendingLogs()
```

### 矿工运行

矿工启动后会运行挖矿循环：

```go
// 在Miner.Start()中
go miner.update()
go worker.mainLoop()
```

## 等待中断信号

最后，主函数调用stack.Wait()等待中断信号：

```go
func (n *Node) Wait() {
	n.stateLock.RLock()
	if n.state != stateStarted {
		n.stateLock.RUnlock()
		return
	}
	n.stateLock.RUnlock()
	
	// 等待中断信号
	<-n.stop
}
```

当收到中断信号时，会执行关闭流程：

```go
func (n *Node) Close() error {
	// 执行关闭操作
	return n.doClose(nil)
}

func (n *Node) doClose(errs []error) error {
	// 停止所有服务
	for _, service := range n.services {
		service.Stop()
	}
	
	// 停止P2P服务器
	n.server.Stop()
	
	// 停止RPC服务
	n.stopWS()
	n.stopHTTP()
	n.stopIPC()
	n.stopInProc()
	
	// 关闭数据库
	if n.database != nil {
		n.database.Close()
	}
	
	// 更新状态
	n.stateLock.Lock()
	n.state = stateClosed
	n.stateLock.Unlock()
	
	return nil
}
```

## 总结

Geth节点启动过程按以下顺序进行：

1. **服务实例化** - 创建所有注册服务的实例
2. **服务启动** - 逐个启动各服务组件
3. **网络启动** - 启动P2P网络服务
4. **RPC启动** - 启动各种RPC接口服务
5. **特殊处理** - 解锁账户、启动挖矿等
6. **等待运行** - 等待中断信号保持运行状态

这个启动流程确保了各个组件按照正确的依赖关系依次启动，形成了一个完整运行的以太坊节点。每个组件都有自己的生命周期管理，使得整个系统具有良好的可维护性和可扩展性。