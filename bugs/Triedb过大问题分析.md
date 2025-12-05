

# Triedb过大问题分析

## 问题现象

```log
INFO [12-04|11:29:47.010] Imported new chain segment               number=21,904,766 hash=19b036..820587 blocks=1    txs=7       mgas=0.474    elapsed=6.096ms     mgasps=77.701   snapdiffs=2.77MiB    triedirty=1.22GiB

INFO [12-04|11:30:16.372] Persisted trie from memory database      nodes=8,175,591  size=1.19GiB   time=41.880221408s gcnodes=183,009,819 gcsize=67.02GiB gctime=11m8.39589783s   livenodes=21475 livesize=8.26MiB

ERROR[12-04|11:30:16.442] Dangling trie nodes after full cleanup
```

如上面的日志所示：
1. 第一条日志，正常运行中，triedb 脏数据占内存为1.22GiB；
1. 第二条日志，关机时清理脏数据，triedb gc 统计信息，回收数据67.02GiB，统计回收时长11 分多；
1. 第三条日志，所有内存数据回收完毕后，triedb 存在孤儿节点。


## 核心原因分析

### 1. triedb 清理相关逻辑

从代码中可以看到：
```go
// eth/ethconfig/config.go
// Defaults contains default settings for use on the Ethereum main net.
var Defaults = Config{
    // ...
	// 这里的缓存超时时间默认 1 小时
	TrieTimeout:        60 * time.Minute,
	// ...
}
```

```go
// core/blockchain.go
func NewBlockChain(db ethdb.Database, cacheConfig *CacheConfig, genesis *Genesis, overrides *ChainOverrides, engine consensus.Engine, vmConfig vm.Config, shouldPreserve func(header *types.Header) bool, txLookupLimit *uint64) (*BlockChain, error) {
    // ...
    // 使用TrieTimeout设置缓存超时
	bc.flushInterval.Store(int64(cacheConfig.TrieTimeLimit))
    // ...
}


func (bc *BlockChain) writeBlockWithState(block *types.Block, receipts []*types.Receipt, state *state.StateDB) error {
	// ...
    
	// 如果是 archive 模式, 直接提交缓存
	if bc.cacheConfig.TrieDirtyDisabled {
		return bc.triedb.Commit(root, false)
	}
	// full 模式下，新加区块时，添加状态根引用和 triegc 记录
	bc.triedb.Reference(root, common.Hash{}) 
	// 注意这里记录的区块的优先级是区块高度的负值，就是越新的区块优先级越低，越旧的区块优先级越高
	bc.triegc.Push(root, -int64(block.NumberU64()))


	// 链的前 128 个块不做任何处理，直接缓存
	current := block.NumberU64()
	if current <= TriesInMemory {
		return nil
	}
	var (
        // 获取 triesdb 和预映像的大小
		_, nodes, imgs = bc.triedb.Size() 
		// 这个值是--cache.gc 参数指定的，默认为cache 的25%
		limit = common.StorageSize(bc.cacheConfig.TrieDirtyLimit) * 1024 * 1024
	)
    // 如果缓存大小超过回收阈值，或者预映像大小超过4M，则开始回收
	if nodes > limit || imgs > 4*1024*1024 {
		// 如果 triedb 内存数据大于回收阈值，则每次减小IdealBatchSize大小
		// 比如我设置的triedb 缓存大小为 512M，在内存超限之后，它每次减小 100KB
		bc.triedb.Cap(limit - ethdb.IdealBatchSize)
	}

	// 往前推128 个块
	chosen := current - TriesInMemory
	// 缓存刷新间隔，注意这里，默认是 1 小时
	flushInterval := time.Duration(bc.flushInterval.Load())

	// 如果超过时间限制，则刷新一个 tries 到磁盘
	log.Debug("Syncing state trie to disk", "gcproc", bc.gcproc, "limit", flushInterval)
	if bc.gcproc > flushInterval {
		// 获取Head-128块的Header
		header := bc.GetHeaderByNumber(chosen)
		if header == nil {
			log.Warn("Reorg in progress, trie commit postponed", "number", chosen)
		} else {
			// 如果到了清理阈值，但是超过 2 个周期，还没有被清理，发出告警
			if chosen < bc.lastWrite+TriesInMemory && bc.gcproc >= 2*flushInterval {
				log.Info("State in memory for too long, committing", "time", bc.gcproc, "allowance", flushInterval, "optimum", float64(chosen-bc.lastWrite)/TriesInMemory)
			}
			// 提交一个完整的 tries 到磁盘，并清理 gc 统计数据
			bc.triedb.Commit(header.Root, true)

			// 记录提交时间和高度
			bc.lastWrite = chosen
			bc.gcproc = 0
		}
	}

	// 下面这个逻辑一般会保证清除掉 128 个之前的区块
	for !bc.triegc.Empty() {
		// 取出最高优先级的 tries（其实就是最老的区块）
		root, number := bc.triegc.Pop()
		if uint64(-number) > chosen {
			// 如果最老的区块高度，还没有达到清理阈值，则重新压入堆中，下次再清理
			bc.triegc.Push(root, number)
			break
		}
		// 删除 trie 数据
		bc.triedb.Dereference(root)
	}
	return nil

}
```

### 2. 内存清理机制失效

在正常运行时，系统有三个清理机制：

1. **定时清理** - 当`bc.gcproc > flushInterval`时触发：
```go
if bc.gcproc > flushInterval {
    // 提交一个完整的 tries 到磁盘
    bc.triedb.Commit(header.Root, true)
}
```

2. **内存阈值清理** - 当内存超过限制时：
```go
if nodes > limit || imgs > 4*1024*1024 {
    bc.triedb.Cap(limit - ethdb.IdealBatchSize)
}
```

3. **triedirty阈值清理** - 当triedirty超过限制时：
```go
	// 下面这个逻辑一般会保证清除掉 128 个之前的区块
	for !bc.triegc.Empty() {
		// 删除 trie 数据
		bc.triedb.Dereference(root)
	}
```


但在实际运行中，这三个机制可能都未能有效工作，导致triedirty累积到1.22GiB。

### 3. 关闭时的清理不彻底

节点关闭时会尝试清理所有剩余的trie节点：
```go
for !bc.triegc.Empty() {
    triedb.Dereference(bc.triegc.PopItem())
}
if _, nodes, _ := triedb.Size(); nodes != 0 {
    log.Error("Dangling trie nodes after full cleanup")
}
```

但此时仍有节点残留在内存中，表明在运行期间积累的节点引用关系未能被正确解除。

## 技术细节分析

### Trie数据库结构

在hashdb中，节点通过以下方式管理：

1. **引用计数机制** - [cachedNode.parents]跟踪节点被引用次数
2. **flush-list双向链表** - 按时间顺序维护脏节点
3. **脏节点缓存** - `db.dirties`存储所有未提交的节点

### 悬空节点产生的根本原因

1. **引用计数未正确减少** - 当节点不再需要时，其引用计数未能及时减少到0
2. **循环引用** - 可能存在节点间的循环引用，导致无法被垃圾回收
3. **异常退出** - 在某些异常情况下，节点的引用关系未能正确清理

### 根本原因分析
因为在正常情况下这个问题不是每次出现（悬空节点），而 triedb 的内存管理机制是一贯的，所以暂时不考虑处理机制的问题。
还是出现在清理逻辑的触发点可能有问题，导致没有及时触发，缓存一直累积，上面日志中超大的缓存也可以看出这方面的问题。

经过详细研读代码，发现问题出现在`bc.gcproc`的计算逻辑上。
正常情况下，如果`flushInterval`阈值为 1 小时，每隔 1 小时就会执行一次 Commit，但是日志中并没有搜索到相关信息：
```go
logger("Persisted trie from memory database", "nodes", nodes-len(db.dirties)+int(db.flushnodes), "size", storage-db.dirtiesSize+db.flushsize, "time", time.Since(start)+db.flushtime,
		"gcnodes", db.gcnodes, "gcsize", db.gcsize, "gctime", db.gctime, "livenodes", len(db.dirties), "livesize", db.dirtiesSize)

```

```go
// core/blockchain.go
func (bc *BlockChain) insertChain(chain types.Blocks, setHead bool) (int, error) {
    // ...
    // bc.gcproc只统计真实的区块处理时间，通过日志观察，每个区块大约0.05 秒
    bc.gcproc += proctime
    // ...
}
```

结合之前的代码就可以分析出，多久执行一次 Commit 操作：
```go
    1 hour / 0.05 second = 72000
```

也就是需要经过72000 个区块，Triedb 才能被清理一次，这个时间大概是1 天。

如果不清理，Triedb 就会一直占用内存，节点的引用计数也不会清理，另外两种缓存清理机制也无法彻底清理。


## 解决方案建议

### 1. 调整TrieTimeLimit参数

针对每秒一个区块的网络，建议将TrieTimeLimit设置为更短的时间，例如1-2分钟：
```go
TrieTimeout: 1 * time.Minute,
```

只需要修改这一个地方，因为没有启动参数可以指定，只能在代码中修改默认值。

### 2. 修改后效果

会定期打印“Persisted trie from memory database”日志，但是时间间隔还是比较久，原因是链上的区块处理比较快，很多块只有几笔交易，处理时间0.001 秒，这样累积满 1 分钟的时间就被大大推迟。

不过目前已经规避了超大内存不回收，关闭错误的问题。

预计 1500 个块会回收一次，根据链上负载不同，可能会 10000 个块才会回收。

需要考虑优化手段，比如结合时间和块数来综合判断回收时间。

## 总结

该问题的根本原因是Trie数据库中的节点引用关系未能得到及时清理，特别是在高频出块环境下，过长的刷盘间隔导致大量节点在内存中累积。通过调整TrieTimeLimit参数、增强内存监控机制和改进关闭清理流程，可以有效解决这一问题，确保系统的稳定性和数据一致性。