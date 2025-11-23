# Geth启动流程解析 - 第五篇：节点启动与服务运行

## 1. 引言

在完成了Node和Ethereum等核心服务的创建之后，Geth需要启动这些服务使其开始工作。本文将深入分析Geth如何启动节点以及各个服务是如何运行的。

## 2. 节点启动流程

节点启动的入口函数：

```go
// cmd/geth/main.go
func startNode(ctx *cli.Context, stack *node.Node, backend ethapi.Backend, isConsole bool) {
	// 1. 启动 Node
	utils.StartNode(ctx, stack, isConsole)

	// 2. 解锁账户
	unlockAccounts(ctx, stack)

	// 注册钱包事件
	events := make(chan accounts.WalletEvent, 16)
	stack.AccountManager().Subscribe(events)

	// 创建直连本地 geth node 的 rpc 客户端
	rpcClient := stack.Attach()
	ethClient := ethclient.NewClient(rpcClient)

	go func() {
		// 打开所有注册的钱包
		for _, wallet := range stack.AccountManager().Wallets() {
			if err := wallet.Open(""); err != nil {
				log.Warn("Failed to open wallet", "url", wallet.URL(), "err", err)
			}
		}
		// 监听所有钱包事件，直到服务终止
		for event := range events {
			// 这里响应钱包事件
			// 新钱包：尝试打开目标钱包
			// 钱包打开：获取目标钱包状态，生成钱包派生路径（leger硬件钱包和其它钱包不同），调用SelfDerive进行自动账户发现
			// 钱包丢弃：关闭目标钱包
			// ...
		}
	}()

	// Spawn a standalone goroutine for status synchronization monitoring,
	// close the node when synchronization is complete if user required.
	// 生成一个独立的goroutine，用来监控同步进度，当同步完成后退出服务（配置ExitWhenSyncedFlag参数时）
	if ctx.Bool(utils.ExitWhenSyncedFlag.Name) {
		go func() {
			// 这里是通过订阅node对象的downloader.DoneEvent事件实现的，
			// 接收到这个事件时候，会调用node.Close()结束节点服务
			sub := stack.EventMux().Subscribe(downloader.DoneEvent{})
			defer sub.Unsubscribe()
			// ...
		}()
	}

	// 在矿工模式下启动挖矿服务
	if ctx.Bool(utils.MiningEnabledFlag.Name) {
		// 矿工模式不支持轻节点
		if ctx.String(utils.SyncModeFlag.Name) == "light" {
			utils.Fatalf("Light clients do not support mining")
		}
		ethBackend, ok := backend.(*eth.EthAPIBackend)
		if !ok {
			utils.Fatalf("Ethereum service not running")
		}
		// 矿工节点使用矿工费作为txpool的门限值，不接收低于矿工费要求的交易
		gasprice := flags.GlobalBig(ctx, utils.MinerGasPriceFlag.Name)
		ethBackend.TxPool().SetGasTip(gasprice)

		// 这里启动矿工节点挖矿模式
		if err := ethBackend.StartMining(); err != nil {
			utils.Fatalf("Failed to start mining: %v", err)
		}
	}
}
```

## 3. Node.Start()详解

stack.Start()是启动过程的核心：

```go
	// 1. 启动 Node
	utils.StartNode(ctx, stack, isConsole)
```

它不是单纯的直接调用node的Start方法，而是包装了一层逻辑在外面，如下：

```go
// cmd/utils/cmd.go
func StartNode(ctx *cli.Context, stack *node.Node, isConsole bool) {
	// 这里是调用node的Start方法
	if err := stack.Start(); err != nil {
		Fatalf("Error starting protocol stack: %v", err)
	}
	// 下面是启动了一个go协程，实现优雅退出和可用磁盘监控
	go func() {
		sigc := make(chan os.Signal, 1)
		signal.Notify(sigc, syscall.SIGINT, syscall.SIGTERM)
		defer signal.Stop(sigc)

		// 计算最小可接受的空闲磁盘空间
		minFreeDiskSpace := 2 * ethconfig.Defaults.TrieDirtyCache // Default 2 * 256Mb
		if ctx.IsSet(MinFreeDiskSpaceFlag.Name) {
			minFreeDiskSpace = ctx.Int(MinFreeDiskSpaceFlag.Name)
		} else if ctx.IsSet(CacheFlag.Name) || ctx.IsSet(CacheGCFlag.Name) {
			minFreeDiskSpace = 2 * ctx.Int(CacheFlag.Name) * ctx.Int(CacheGCFlag.Name) / 100
		}
		// 这里启动单独的可用磁盘空间监控
		if minFreeDiskSpace > 0 {
			// 详细代码不放上了，就是监测到磁盘不足，使用退出信号强制退出，避免数据损坏
			// sigc <- syscall.SIGTERM
			go monitorFreeDiskSpace(sigc, stack.InstanceDir(), uint64(minFreeDiskSpace)*1024*1024)
		}

		// 这里定义了优雅退出函数
		shutdown := func() {
			log.Info("Got interrupt, shutting down...")
			go stack.Close()
			for i := 10; i > 0; i-- {
				<-sigc
				if i > 1 {
					log.Warn("Already shutting down, interrupt more to panic.", "times", i-1)
				}
			}
			debug.Exit() // 退出是强制刷新trace信息和CPU优化信息
			debug.LoudPanic("boom")
		}

		if isConsole {
			// 这里接收控制台模式下的退出信号SIGTERM 
			for {
				sig := <-sigc
				if sig == syscall.SIGTERM {
					shutdown()
					return
				}
			}
		} else {
			// 接收退出信号（SIGINT或SIGTERM ），执行优雅退出
			<-sigc
			shutdown()
		}
	}()
}
```

至于node.Start()的详细逻辑，在本系列文章的 Geth启动流程解析 - 第三篇：Node节点创建与初始化 有介绍，可以参考，这里不再赘述。

## 4. 账户解锁

如果配置了自动解锁账户，会执行以下操作：

```go
func unlockAccounts(ctx *cli.Context, stack *node.Node) {
	// 获取命令行参数--unlock指定的需要自动解锁的地址列表
	var unlocks []string
	inputs := strings.Split(ctx.String(utils.UnlockedAccountFlag.Name), ",")
	for _, input := range inputs {
		if trimmed := strings.TrimSpace(input); trimmed != "" {
			unlocks = append(unlocks, trimmed)
		}
	}
	// 没指定就直接返回
	if len(unlocks) == 0 {
		return
	}

	// 检查命令行allow-insecure-unlock参数和rpc是否同时启用（禁止同时启用，有安全风险）
	if !stack.Config().InsecureUnlockAllowed && stack.Config().ExtRPCEnabled() {
		utils.Fatalf("Account unlock with HTTP access is forbidden!")
	}
	backends := stack.AccountManager().Backends(keystore.KeyStoreType)
	if len(backends) == 0 {
		log.Warn("Failed to unlock accounts, keystore is not available")
		return
	}
	ks := backends[0].(*keystore.KeyStore)
	// 这行代码会从命令行参数--password指定的密码文件中读取钱包文件的解密密码（可能会有多行，和要解锁的地址顺序对应）
	passwords := utils.MakePasswordList(ctx)
	for i, account := range unlocks {
		// 解锁钱包账户
		unlockAccount(ks, account, i, passwords)
	}
}
```

## 5.挖矿启动

本系列文章就不深入挖矿机制了，只需要直到这里会开始启动挖矿进程（和交易池、共识配合打包出块），后面会有专题分析（TODO）。


## 6. 服务运行机制

这里也不深入介绍，运行过程中涉及到多个功能模块的配合，这里简单理出服务运行过程中的主要内容：

- P2P会进行区块的同步和交易的广播；
- TxPool管理内存中的交易；
- RPC接收外部交易请求和查询请求；
- 矿工负责读取TxPool中的交易和共识一起打包出块；


## 7. 服务关闭

最后，主函数调用stack.Wait()等待中断信号：

```go
// Wait blocks until the node is closed.
func (n *Node) Wait() {
	<-n.stop
}
```

当收到中断信号时，会执行关闭流程：

```go
// 执行关闭操作
func (n *Node) Close() error {
	n.startStopLock.Lock()
	defer n.startStopLock.Unlock()

	n.lock.Lock()
	state := n.state
	n.lock.Unlock()
	switch state {
	case initializingState:
		// 这个是异常情况，未启动成功就关闭
		return n.doClose(nil)
	case runningState:
		// 运行态关闭，正常情况，需要释放一些资源
		var errs []error
		// 停止node内部的多个服务
		if err := n.stopServices(n.lifecycles); err != nil {
			errs = append(errs, err)
		}
		return n.doClose(errs)
	case closedState:
		// 重复关闭，这里不应该进入
		return ErrNodeStopped
	default:
		panic(fmt.Sprintf("node is in unknown state %d", state))
	}
}

func (n *Node) stopServices(running []Lifecycle) error {
	// 停止RPC服务
	n.stopRPC()

	// 停止多个Lifecycle生命周期
	failure := &StopError{Services: make(map[reflect.Type]error)}
	for i := len(running) - 1; i >= 0; i-- {
		if err := running[i].Stop(); err != nil {
			failure.Services[reflect.TypeOf(running[i])] = err
		}
	}

	// 停止P2P服务
	n.server.Stop()

	if len(failure.Services) > 0 {
		return failure
	}
	return nil
}

// 然后是最终收尾
func (n *Node) doClose(errs []error) error {
	// 关闭数据库，需要获取锁进行操作（和OpenDatabase*互斥）
	n.lock.Lock()
	n.state = closedState
	// 这里执行真正的数据库Close逻辑
	errs = append(errs, n.closeDatabases()...)
	n.lock.Unlock()

	// 关闭账户管理器
	if err := n.accman.Close(); err != nil {
		errs = append(errs, err)
	}
	// 删除临时的keystore目录
	if n.keyDirTemp {
		if err := os.RemoveAll(n.keyDir); err != nil {
			errs = append(errs, err)
		}
	}

	// 释放datadir目录锁
	n.closeDataDir()

	// 释放n.Wait锁
	close(n.stop)

	// 检查错误信息
	switch len(errs) {
	case 0:
		return nil
	case 1:
		return errs[0]
	default:
		return fmt.Errorf("%v", errs)
	}
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

本系列暂时草草收尾，感觉写的很乱，改天再重新整理一下。
