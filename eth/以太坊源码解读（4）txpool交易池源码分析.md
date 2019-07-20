# 交易池：txpool源码分析

交易池的源码位于：core/tx_pool.go文件

txpool交易池由两部分构成分别是pending和queued组成。主要适用于存放当前提交等待被区块确认提交的交易，本地交易和网络远程交易都有

1、pending：等待执行的交易会被放在pending队列中

2、queued：提交但是不能够执行的交易，放在queue中等待执行

### 通过阅读tx_pool_test.go这个txpool的测试文件源码可以发现txpool主要功能如下：

1、检查交易的信息数据是否合法，包括Gas，余额是否不足，Nonce大小等等

2、检查时间状态，将nonce过高的交易放在queue队列，将可以执行的交易放在		    pending队列

3、在资源有限的情况下（例如当前池满了，或者网络拥堵）,会优先执行GasPrice高的交易

4、如果当前交易的额度大于当前账户的额度，交易会被删除

5、对于相同的account对应的相同nonce的交易只会保存GasPrice高的哪个交易

6、本地交易会使用journal的功能将信息存在本地磁盘

7、如果account没有余额了，那么对应queue队列和pending队列中的交易会被删除

### txpool的数据结构

```go
type TxPool struct {
    //配置信息
   config       TxPoolConfig
    //链配置
   chainconfig  *params.ChainConfig
    //当前的链
   chain        blockChain
    //最低的gas价格
   gasPrice     *big.Int
    //通过txFedd订阅TxPool的消息
   txFeed       event.Feed
    //提供了同时取消多个订阅的功能
   scope        event.SubscriptionScope
    //当有了新的区块的产生会收到消息，订阅区块头消息
   chainHeadCh  chan ChainHeadEvent
    //区块头消息订阅器
   chainHeadSub event.Subscription
    //对事物进行签名处理
   signer       types.Signer
    //读写互斥锁
   mu           sync.RWMutex
	//当前区块链头部的状态
   currentState  *state.StateDB
    //挂起状态跟踪虚拟nonces
   pendingState  *state.ManagedState 
    // 目前交易的费用上限
   currentMaxGas uint64              
	//一套豁免驱逐规则的本地交易
   locals  *accountSet 
    //本地事务日志备份到磁盘
   journal *txJournal 
	//等待队列
   pending map[common.Address]*txList  
    //排队但不可处理的事务
   queue   map[common.Address]*txList   
    //每个已知帐户的最后一次心跳
   beats   map[common.Address]time.Time 
    //所有允许查询的事务
   all     *txLookup   
    //所有按价格排序的交易
   priced  *txPricedList                
	//关闭同步
   wg sync.WaitGroup 
	//家园版本？？
   homestead bool
}
```

TxPool的配置信息包括一下

```go
type TxPoolConfig struct {
    // 默认情况下应视为本地的地址
   Locals    []common.Address 
    // 是否应该禁用本地事务处理
   NoLocals  bool  
    // 本地事务日志，以便在节点重新启动时存活
   Journal   string  
    // 重新生成本地事务日志的时间间隔
   Rejournal time.Duration 
    // 最低gas价格，以接受入池
   PriceLimit uint64
    // 更换//现有交易的最低价格增幅(一次)	
   PriceBump  uint64 
	//每个帐户保证的可执行事务槽数
   AccountSlots uint64 
    //所有帐户的可执行事务槽的最大数量
   GlobalSlots  uint64 
    //每个帐户允许的非可执行事务槽的最大数量
   AccountQueue uint64 
    //所有帐户的非可执行事务槽的最大数量
   GlobalQueue  uint64
	//非可执行事务的最大排队时间
   Lifetime time.Duration 
}
```

### NewTxPool() 构建

```go
func NewTxPool(config TxPoolConfig, chainconfig *params.ChainConfig, chain blockChain) *TxPool {
   //对输入进行消毒，确保不会设定易受影响的天然气价格
   // sanitize检查所提供的用户配置，并更改任何不合理或不可行的配置
    config = (&config).sanitize()
	//创建带有初始设置的交易池
   // Create the transaction pool with its initial settings
   pool := &TxPool{
      config:      config,
      chainconfig: chainconfig,
      chain:       chain,
      signer:      types.NewEIP155Signer(chainconfig.ChainID),
      pending:     make(map[common.Address]*txList),
      queue:       make(map[common.Address]*txList),
      beats:       make(map[common.Address]time.Time),
      all:         newTxLookup(),
      chainHeadCh: make(chan ChainHeadEvent, chainHeadChanSize),
      gasPrice:    new(big.Int).SetUint64(config.PriceLimit),
   }
   pool.locals = newAccountSet(pool.signer)
   for _, addr := range config.Locals {
      log.Info("Setting new local account", "address", addr)
      pool.locals.add(addr)
   }
   pool.priced = newTxPricedList(pool.all)
   pool.reset(nil, chain.CurrentBlock().Header())
	////如果启用了本地事务和日志记录，则从磁盘加载
   // If local transactions and journaling is enabled, load from disk
   if !config.NoLocals && config.Journal != "" {
      pool.journal = newTxJournal(config.Journal)

      if err := pool.journal.load(pool.AddLocals); err != nil {
         log.Warn("Failed to load transaction journal", "err", err)
      }
      if err := pool.journal.rotate(pool.local()); err != nil {
         log.Warn("Failed to rotate transaction journal", "err", err)
      }
   }
   // 从区块链订阅事件
   pool.chainHeadSub = pool.chain.SubscribeChainHeadEvent(pool.chainHeadCh)

   // 启动事件循环并返回
   pool.wg.Add(1)
   go pool.loop()

   return pool
}
```

### add() 方法

验证交易并将其插入到future queue. 如果这个交易是替换了当前存在的某个交易,那么会返回之前的那个交易,这样外部就不用调用promote方法. 如果某个新增加的交易被标记为local, 那么它的发送账户会进入白名单,这个账户的关联的交易将不会因为价格的限制或者其他的一些限制被删除

```go
func (pool *TxPool) add(tx *types.Transaction, local bool) (bool, error) {
    //如果这个交易已经知道，就丢弃他
   hash := tx.Hash()
   if pool.all.Get(hash) != nil {
      log.Trace("Discarding already known transaction", "hash", hash)
      return false, fmt.Errorf("known transaction: %x", hash)
   }
   // 如果交易不能通过基本数据验证，就丢弃它
   if err := pool.validateTx(tx, local); err != nil {
      log.Trace("Discarding invalid transaction", "hash", hash, "err", err)
      invalidTxCounter.Inc(1)
      return false, err
   }
   // 如果交易池已经满了，就丢弃交易费用低的交易
   if uint64(pool.all.Count()) >= pool.config.GlobalSlots+pool.config.GlobalQueue {
      // 如果新的交易费用交易过低就不接受
      if !local && pool.priced.Underpriced(tx, pool.locals) {
         log.Trace("Discarding underpriced transaction", "hash", hash, "price", tx.GasPrice())
         underpricedTxCounter.Inc(1)
         return false, ErrUnderpriced
      }
      // 如果新的交易比旧的交易号好，就添加新的交易
      drop := pool.priced.Discard(pool.all.Count()-int(pool.config.GlobalSlots+pool.config.GlobalQueue-1), pool.locals)
      for _, tx := range drop {
         log.Trace("Discarding freshly underpriced transaction", "hash", tx.Hash(), "price", tx.GasPrice())
         underpricedTxCounter.Inc(1)
         pool.removeTx(tx.Hash(), false)
      }
   }
   // 如果交易正在替换已经挂起的交易，请直接执行
   from, _ := types.Sender(pool.signer, tx) // already validated
   if list := pool.pending[from]; list != nil && list.Overlaps(tx) {
      // 一旦已经挂起，检查是否满足要求的价格上涨
      inserted, old := list.Add(tx, pool.config.PriceBump)
      if !inserted {
         pendingDiscardCounter.Inc(1)
         return false, ErrReplaceUnderpriced
      }
      // 新交易更好，替换掉旧交易
      if old != nil {
         pool.all.Remove(old.Hash())
         pool.priced.Removed()
         pendingReplaceCounter.Inc(1)
      }
      pool.all.Add(tx)
      pool.priced.Put(tx)
      pool.journalTx(from, tx)

      log.Trace("Pooled new executable transaction", "hash", hash, "from", from, "to", tx.To())

      // 我们直接注入了一个新的交易，通知子系统
      go pool.txFeed.Send(NewTxsEvent{types.Transactions{tx}})

      return old != nil, nil
   }
   // New transaction isn't replacing a pending one, push into queue
   replace, err := pool.enqueueTx(hash, tx)
   if err != nil {
      return false, err
   }
   // 标记本地地址并记录本地交易
   if local {
      if !pool.locals.contains(from) {
         log.Info("Setting new local account", "address", from)
         pool.locals.add(from)
      }
   }
   pool.journalTx(from, tx)

   log.Trace("Pooled new future transaction", "hash", hash, "from", from, "to", tx.To())
   return replace, nil
}
```

### validateTx() 方法

使用一致性规则来检查一个交易是否有效,并采用本地节点的一些启发式的限制

```go
func (pool *TxPool) validateTx(tx *types.Transaction, local bool) error {
   // 拒绝超过32KB的事务，以防止DOSS攻击
   if tx.Size() > 32*1024 {
      return ErrOversizedData
   }
   // 交易不能是负的。这可能永远不会发生使用RLP解码
   // 但如果使用RPC创建交易，则可能发生交易。
   if tx.Value().Sign() < 0 {
      return ErrNegativeValue
   }
   // 确保交易使用的Gas不超过当前块限制的GasLimit
   if pool.currentMaxGas < tx.Gas() {
      return ErrGasLimit
   }
   // 确保交易签名正确
   from, err := types.Sender(pool.signer, tx)
   if err != nil {
      return ErrInvalidSender
   }
   // 在我们自己的最低接受Gas价格下放弃非本地交易
   local = local || pool.locals.contains(from) 
    //即使交易从网络到达，帐户也可能是本地的
   if !local && pool.gasPrice.Cmp(tx.GasPrice()) > 0 {
      return ErrUnderpriced
   }
   // 确保交易符合即时的nonce 也就是当前的交易的nonce必须要等与当前账户的	//nonce
   if pool.currentState.GetNonce(from) > tx.Nonce() {
      return ErrNonceTooLow
   }
   // 发起交易的一方应该有足够的资金来支付费用，判断余额是否足够
   // cost == V + GP * GL
   if pool.currentState.GetBalance(from).Cmp(tx.Cost()) < 0 {
      return ErrInsufficientFunds
   }
   intrGas, err := IntrinsicGas(tx.Data(), tx.To() == nil, pool.homestead)
   if err != nil {
      return err
   }
   if tx.Gas() < intrGas {
      return ErrIntrinsicGas
   }
   return nil
}
```

### loop() 方法

是txPool的一个goroutine.也是主要的事件循环.等待和响应外部区块链事件以及各种报告和交易驱逐事件

```go
func (pool *TxPool) loop() {
    //等待组计数器减一
   defer pool.wg.Done()

   // 启动统计报表和交易退出提示符
   var prevPending, prevQueued, prevStales int
	
   report := time.NewTicker(statsReportInterval)
   defer report.Stop()

   evict := time.NewTicker(evictionInterval)
   defer evict.Stop()

   journal := time.NewTicker(pool.config.Rejournal)
   defer journal.Stop()

   // 跟踪交易重组的前一个头标头
   head := pool.chain.CurrentBlock()

   // 不断等待和应对各种事件
   for {
      select {
      // 处理链头事件
      case ev := <-pool.chainHeadCh:
         if ev.Block != nil {
            pool.mu.Lock()
            if pool.chainconfig.IsHomestead(ev.Block.Number()) {
               pool.homestead = true
            }
            pool.reset(head.Header(), ev.Block.Header())
            head = ev.Block

            pool.mu.Unlock()
         }
      //由于系统停止而取消订阅
      case <-pool.chainHeadSub.Err():
         return

      // 处理统计报表刻度
      case <-report.C:
         pool.mu.RLock()
         pending, queued := pool.stats()
         stales := pool.priced.stales
         pool.mu.RUnlock()

         if pending != prevPending || queued != prevQueued || stales != prevStales {
            log.Debug("Transaction pool status report", "executable", pending, "queued", queued, "stales", stales)
            prevPending, prevQueued, prevStales = pending, queued, stales
         }

      // 处理非活动帐户事务退出
      case <-evict.C:
         pool.mu.Lock()
         for addr := range pool.queue {
            // 从退出机制中跳过本地事务
            if pool.locals.contains(addr) {
               continue
            }
            // 任何时间足够长的非本地交易信息都应该被清除
            if time.Since(pool.beats[addr]) > pool.config.Lifetime {
               for _, tx := range pool.queue[addr].Flatten() {
                  pool.removeTx(tx.Hash(), true)
               }
            }
         }
         pool.mu.Unlock()

      // 处理本地交易日志的轮换
      case <-journal.C:
         if pool.journal != nil {
            pool.mu.Lock()
            if err := pool.journal.rotate(pool.local()); err != nil {
               log.Warn("Failed to rotate local tx journal", "err", err)
            }
            pool.mu.Unlock()
         }
      }
   }
}
```

### promoteTx() 方法

把某个交易加入到pending 队列. 这个方法假设已经获取到了锁

```go
func (pool *TxPool) promoteTx(addr common.Address, hash common.Hash, tx *types.Transaction) bool {
   // 尝试将交易插入到等待挂起队列中
   if pool.pending[addr] == nil {
      pool.pending[addr] = newTxList(true)
   }
   list := pool.pending[addr]

   inserted, old := list.Add(tx, pool.config.PriceBump)
   if !inserted {
      // 如果旧的交易更好就丢弃这个交易
      pool.all.Remove(hash)
      pool.priced.Removed()

      pendingDiscardCounter.Inc(1)
      return false
   }
   // 如果新的交易更好就丢弃以前的任何事务并标记此事务
   if old != nil {
      pool.all.Remove(old.Hash())
      pool.priced.Removed()

      pendingReplaceCounter.Inc(1)
   }
   // Failsafe to work around direct pending inserts (tests)
   if pool.all.Get(hash) == nil {
      pool.all.Add(tx)
      pool.priced.Put(tx)
   }
   // Set the potentially new pending nonce and notify any subsystems of the new tx
   pool.beats[addr] = time.Now()
   pool.pendingState.SetNonce(addr, tx.Nonce()+1)

   return true
}
```