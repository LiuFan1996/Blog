# Accounts源码分析与逻辑结构2

### listAccounts源码分析

##### 位于internal/ethapi/api.go中的ListAccounts方法

```go
// ListAccounts will return a list of addresses for accounts this node manages.
func (s *PrivateAccountAPI) ListAccounts() []common.Address {
   addresses := make([]common.Address, 0) // return [] instead of nil if empty
   for _, wallet := range s.am.Wallets() {
      for _, account := range wallet.Accounts() {
         addresses = append(addresses, account.Address)
      }
   }
   return addresses
}
```

通过accounts/account/下的Wallets钱包管理，遍历钱包中存储的账户，将addresses数组返回控制台

### **sendTransaction** 源码分析

##### 在了解sendTransaction之前我们先看一下Transaction 的数据模型

在core/types/tracsaction.go文件下的Transaction结构体

```go
type Transaction struct {
   data txdata
   // caches
   hash atomic.Value
   size atomic.Value
   from atomic.Value
}
```

hash：交易的hash

size：交易数据的大小

from：发起交易的地址

data：交易数据

```go
type txdata struct {
   AccountNonce uint64          `json:"nonce"    gencodec:"required"`
   Price        *big.Int        `json:"gasPrice" gencodec:"required"`
   GasLimit     uint64          `json:"gas"      gencodec:"required"`
   Recipient    *common.Address `json:"to"       rlp:"nil"` // nil means contract creation
   Amount       *big.Int        `json:"value"    gencodec:"required"`
   Payload      []byte          `json:"input"    gencodec:"required"`

   // Signature values
   V *big.Int `json:"v" gencodec:"required"`
   R *big.Int `json:"r" gencodec:"required"`
   S *big.Int `json:"s" gencodec:"required"`

   // This is only used when marshaling to JSON.
   Hash *common.Hash `json:"hash" rlp:"-"`
}
```

AccountNonce：发起者发起的交易总数量

Price：此次交易的gas 价格

GasLimit：此次交易允许消耗的最大gas数 

Recipient：交易接收者的地址

Amount：此次交易的以太币数量

Payload：对应的虚拟机指令

V：签名数据

R：签名数据

S：签名数据

SendTxArgs：交易的一些参数

```go
type SendTxArgs struct {
   From     common.Address  `json:"from"`
   To       *common.Address `json:"to"`
   Gas      *hexutil.Uint64 `json:"gas"`
   GasPrice *hexutil.Big    `json:"gasPrice"`
   Value    *hexutil.Big    `json:"value"`
   Nonce    *hexutil.Uint64 `json:"nonce"`
   // We accept "data" and "input" for backwards-compatibility reasons. "input" is the
   // newer name and should be preferred by clients.
   Data  *hexutil.Bytes `json:"data"`
   Input *hexutil.Bytes `json:"input"`
}
```

##### 了解了交易的数据模型之后，我们再来具体执行的源码：

sendTransaction经过RPC调用后，会调用internal/ethapi/api.go中的sendTransaction方法

```go
// SendTransaction为给定的参数创建一个事务，签名并提交给事务池
func (s *PublicTransactionPoolAPI) SendTransaction(ctx context.Context, args SendTxArgs) (common.Hash, error) {
	//通过From地址构造一个account
   // Look up the wallet containing the requested signer
   account := accounts.Account{Address: args.From}
	//调用账户管理器获得该account的钱包，Find方法会从账户管理系统中对钱包进行遍历，找到包含这个        //account的钱包
   wallet, err := s.b.AccountManager().Find(account)
   if err != nil {
      return common.Hash{}, err
   }

   if args.Nonce == nil {
       //保持地址的互斥锁在签名附近，以防止并发分配同样的方法可以多次使用
       //对于每一个账户，nonce会随着转账的增加而增加，以防止双花攻击
      // Hold the addresse's mutex around signing to prevent concurrent assignment of
      // the same nonce to multiple accounts.
      s.nonceLock.LockAddr(args.From)
      defer s.nonceLock.UnlockAddr(args.From)
   }
	//设置交易参数的默认值
   // Set some sanity defaults and terminate on failure
   if err := args.setDefaults(ctx, s.b); err != nil {
      return common.Hash{}, err
   }
    //利用toTransaction方法创建一笔交易
   // Assemble the transaction and sign with the wallet
   tx := args.toTransaction()
	//此时我们已经创建好了一笔交易，接着我们获取区块链的配置信息，检查是否是EIP155的配置，并获取链ID。
   var chainID *big.Int
   if config := s.b.ChainConfig(); config.IsEIP155(s.b.CurrentBlock().Number()) {
      chainID = config.ChainID
   }
    //对交易进行签名 参数是账户，交易信息 ，链ID
   signed, err := wallet.SignTx(account, tx, chainID)
   if err != nil {
      return common.Hash{}, err
   }
   return submitTransaction(ctx, s.b, signed)
}
```

Find方法会从账户管理系统中对钱包进行遍历，找到包含这个account的钱包

```go
func (am *Manager) Find(account Account) (Wallet, error) {
    //将当前的账户管理器进行读锁
   am.lock.RLock()
    //调用栈结束解锁
   defer am.lock.RUnlock()
	//遍历账户管理器的所有钱包
   for _, wallet := range am.wallets {
       //找到这个account对应的钱包
      if wallet.Contains(account) {
          //返回找到的钱包
         return wallet, nil
      }
   }
    //没有找到返回对应错误
   return nil, ErrUnknownAccount
}
```

获得钱包以后对交易参数中的账户nonce进行上锁，以防止双花攻击，然后调用setDefaults方法对交易参数设置一些默认参数值

```go
func (args *SendTxArgs) setDefaults(ctx context.Context, b Backend) error {
    //参数中Gas是否为空
   if args.Gas == nil {
       //为空设置默认值
      args.Gas = new(hexutil.Uint64)
      *(*uint64)(args.Gas) = 90000
   }
    //参数中GasPrice是否为空
   if args.GasPrice == nil {
       //获取建议的市场价格
      price, err := b.SuggestPrice(ctx)
      if err != nil {
         return err
      }
       //设置价格
      args.GasPrice = (*hexutil.Big)(price)
   }
    //参数中Value是否为空
   if args.Value == nil {
      args.Value = new(hexutil.Big)
   }
    //参数中Nonce是否为空 
   if args.Nonce == nil {
       //通过账户获取账户的nonce
      nonce, err := b.GetPoolNonce(ctx, args.From)
      if err != nil {
         return err
      }
       //设置nonce
      args.Nonce = (*hexutil.Uint64)(&nonce)
   }
   if args.Data != nil && args.Input != nil && !bytes.Equal(*args.Data, *args.Input) {
      return errors.New(`Both "data" and "input" are set and not equal. Please use "input" to pass transaction call data.`)
   }
    //如果参数中To为空
   if args.To == nil {
      // Contract creation
      var input []byte
      if args.Data != nil {
         input = *args.Data
      } else if args.Input != nil {
         input = *args.Input
      }
      if len(input) == 0 {
         return errors.New(`contract creation without any data provided`)
      }
   }
   return nil
}
```

toTransaction方法使用SendTxArgs参数创建一笔交易，将新的交易信息返回

```go
func (args *SendTxArgs) toTransaction() *types.Transaction {
    //一个字节数组
   var input []byte
    //如果参数Data不为空
   if args.Data != nil {
       //字节数组的值=Data
      input = *args.Data
       //如果参数Input不为空
   } else if args.Input != nil {
       //字节数组的值=Input
      input = *args.Input
   }
    //这里会对传入的交易信息的to参数进行判断。如果没有to值，那么这是一笔合约转账；而如果有to值，那么就		是发起的一笔转账。最终，代码会调用NewTransaction创建一笔交易信息
   if args.To == nil {
      return types.NewContractCreation(uint64(*args.Nonce), (*big.Int)(args.Value), uint64(*args.Gas), (*big.Int)(args.GasPrice), input)
   }
   return types.NewTransaction(uint64(*args.Nonce), *args.To, (*big.Int)(args.Value), uint64(*args.Gas), (*big.Int)(args.GasPrice), input)
}
```

事实上NewContractCreation也是调用newTransaction，知识to参数为nil

```go
func NewContractCreation(nonce uint64, amount *big.Int, gasLimit uint64, gasPrice *big.Int, data []byte) *Transaction {
   return newTransaction(nonce, nil, amount, gasLimit, gasPrice, data)
}
```

通过newTransaction构建一个完整的Transaction交易数据结构，将数据机构返回

```go
func newTransaction(nonce uint64, to *common.Address, amount *big.Int, gasLimit uint64, gasPrice *big.Int, data []byte) *Transaction {
   if len(data) > 0 {
      data = common.CopyBytes(data)
   }
   d := txdata{
      AccountNonce: nonce,
      Recipient:    to,
      Payload:      data,
      Amount:       new(big.Int),
      GasLimit:     gasLimit,
      Price:        new(big.Int),
      V:            new(big.Int),
      R:            new(big.Int),
      S:            new(big.Int),
   }
   if amount != nil {
      d.Amount.Set(amount)
   }
   if gasPrice != nil {
      d.Price.Set(gasPrice)
   }

   return &Transaction{data: d}
}
```

这里就是填充了交易结构体中的一些参数，来创建一个交易。到这里，一笔交易就已经创建成功了。

回到sendTransaction方法中，此时我们已经创建好了一笔交易，接着我们获取区块链的配置信息，检查是否是EIP155的配置，并获取链ID。

```go
//获取链的一些配置信息
type ChainConfig struct {
   ChainID *big.Int `json:"chainId"` // chainId identifies the current chain and is used for replay protection

   HomesteadBlock *big.Int `json:"homesteadBlock,omitempty"` // Homestead switch block (nil = no fork, 0 = already homestead)

   DAOForkBlock   *big.Int `json:"daoForkBlock,omitempty"`   // TheDAO hard-fork switch block (nil = no fork)
   DAOForkSupport bool     `json:"daoForkSupport,omitempty"` // Whether the nodes supports or opposes the DAO hard-fork

   // EIP150 implements the Gas price changes (https://github.com/ethereum/EIPs/issues/150)
   EIP150Block *big.Int    `json:"eip150Block,omitempty"` // EIP150 HF block (nil = no fork)
   EIP150Hash  common.Hash `json:"eip150Hash,omitempty"`  // EIP150 HF hash (needed for header only clients as only gas pricing changed)

   EIP155Block *big.Int `json:"eip155Block,omitempty"` // EIP155 HF block
   EIP158Block *big.Int `json:"eip158Block,omitempty"` // EIP158 HF block

   ByzantiumBlock      *big.Int `json:"byzantiumBlock,omitempty"`      // Byzantium switch block (nil = no fork, 0 = already on byzantium)
   ConstantinopleBlock *big.Int `json:"constantinopleBlock,omitempty"` // Constantinople switch block (nil = no fork, 0 = already activated)
   EWASMBlock          *big.Int `json:"ewasmBlock,omitempty"`          // EWASM switch block (nil = no fork, 0 = already activated)

   // Various consensus engines
   Ethash *EthashConfig `json:"ethash,omitempty"`
   Clique *CliqueConfig `json:"clique,omitempty"`
}
```

为了保证交易的真实有效，我们需要对交易进行签名，调用SingTx方法对交易签名

```go
func (ks *KeyStore) SignTx(account *Account, tx *Transaction, chainID *BigInt) (*Transaction, error) {
   if chainID == nil { // Null passed from mobile app
      chainID = new(BigInt)
   }
   signed, err := ks.keystore.SignTx(account.account, tx.tx, chainID.bigint)
   if err != nil {
      return nil, err
   }
    //返回签名后的交易信息
   return &Transaction{signed}, nil
}
```

回到sendTransaction方法中这个时候需要提交交易，调用submitTransaction方法会将交易发送给backend进行处理，返回经过签名后的交易的hash值。这里主要是SendTx方法对交易进行处理。

sendTx方法会将参数转给txpool的Addlocal方法进行处理，而AddLocal方法会将该笔交易放入到交易池中进行等待。这里我们看将交易放入到交易池中的方法。



```go
func submitTransaction(ctx context.Context, b Backend, tx *types.Transaction) (common.Hash, error) {
   if err := b.SendTx(ctx, tx); err != nil {
      return common.Hash{}, err
   }
   if tx.To() == nil {
      signer := types.MakeSigner(b.ChainConfig(), b.CurrentBlock().Number())
      from, err := types.Sender(signer, tx)
      if err != nil {
         return common.Hash{}, err
      }
      addr := crypto.CreateAddress(from, tx.Nonce())
      log.Info("Submitted contract creation", "fullhash", tx.Hash().Hex(), "contract", addr.Hex())
   } else {
      log.Info("Submitted transaction", "fullhash", tx.Hash().Hex(), "recipient", tx.To())
   }
   return tx.Hash(), nil
}
```

```go
func (b *EthAPIBackend) SendTx(ctx context.Context, signedTx *types.Transaction) error {
   return b.eth.txPool.AddLocal(signedTx)
}
```

```go
func (pool *TxPool) AddLocal(tx *types.Transaction) error {
   return pool.addTx(tx, !pool.config.NoLocals)
}
```

```go
func (pool *TxPool) addTx(tx *types.Transaction, local bool) error {
   pool.mu.Lock()
   defer pool.mu.Unlock()

   // Try to inject the transaction and update any state
   replace, err := pool.add(tx, local)
   if err != nil {
      return err
   }
   // If we added a new transaction, run promotion checks and return
   if !replace {
      from, _ := types.Sender(pool.signer, tx) // already validated
      pool.promoteExecutables([]common.Address{from})
   }
   return nil
}
```

这里一共有两部操作，第一步操作是调用add方法将交易放入到交易池中，第二步是判断replace参数。如果该笔交易合法并且交易原来不存在在交易池中，则执行promoteExecutables方法，将可处理的交易变为待处理（pending）。

首先看第一步add方法。

```go
func (pool *TxPool) add(tx *types.Transaction, local bool) (bool, error) {
   // If the transaction is already known, discard it
   hash := tx.Hash()
    //判断这个交易hash有么有在交易池中，如果交易池中有这笔交易则返回报错
   if pool.all.Get(hash) != nil {
      log.Trace("Discarding already known transaction", "hash", hash)
      return false, fmt.Errorf("known transaction: %x", hash)
   }
    //调用validateTx判断交易是否合法，如果不合法则返回报错
   // If the transaction fails basic validation, discard it
   if err := pool.validateTx(tx, local); err != nil {
      log.Trace("Discarding invalid transaction", "hash", hash, "err", err)
      invalidTxCounter.Inc(1)
      return false, err
   }
    //判断交易池是否超过容量
   // If the transaction pool is full, discard underpriced transactions
   if uint64(pool.all.Count()) >= pool.config.GlobalSlots+pool.config.GlobalQueue {
      // If the new transaction is underpriced, don't accept it
       //如果超过容量，并且该笔交易的费用低于当前交易池中列表的最小值，则拒绝这一笔交易
      if !local && pool.priced.Underpriced(tx, pool.locals) {
         log.Trace("Discarding underpriced transaction", "hash", hash, "price", tx.GasPrice())
         underpricedTxCounter.Inc(1)
         return false, ErrUnderpriced
      }
       //如果超过容量，并且该笔交易的费用比当前交易池中列表最小值高，那么从交易池中移除交易费用最低的交易，为当前这一笔交易留出空间。
      // New transaction is better than our worse ones, make room for it
      drop := pool.priced.Discard(pool.all.Count()-int(pool.config.GlobalSlots+pool.config.GlobalQueue-1), pool.locals)
      for _, tx := range drop {
         log.Trace("Discarding freshly underpriced transaction", "hash", tx.Hash(), "price", tx.GasPrice())
         underpricedTxCounter.Inc(1)
         pool.removeTx(tx.Hash(), false)
      }
   }
    //如果事务正在替换一个已经挂起的事务，请直接执行
   // If the transaction is replacing an already pending one, do directly
   from, _ := types.Sender(pool.signer, tx) // already validated
    //接着继续调用Overlaps方法检查该笔交易的Nonce值，确认该用户下的交易是否存在该笔交易
   if list := pool.pending[from]; list != nil && list.Overlaps(tx) {
      // Nonce already pending, check if required price bump is met
      inserted, old := list.Add(tx, pool.config.PriceBump)
     //  如果已经存在这笔交易，则删除之前的交易，并将该笔交易放入交易池中，然后返回。
      if !inserted {
         pendingDiscardCounter.Inc(1)
         return false, ErrReplaceUnderpriced
      }
      // New transaction is better, replace old one
      
      if old != nil {
         pool.all.Remove(old.Hash())
         pool.priced.Removed()
         pendingReplaceCounter.Inc(1)
      }
       //在池里添加新的交易信息
      pool.all.Add(tx)
       //添加新交易的价格
      pool.priced.Put(tx)
       //将交易信息写到日志中，只有是在本地账户的情况下
      pool.journalTx(from, tx)

      log.Trace("Pooled new executable transaction", "hash", hash, "from", from, "to", tx.To())

      // We've directly injected a replacement transaction, notify subsystems
      go pool.txFeed.Send(NewTxsEvent{types.Transactions{tx}})

      return old != nil, nil
   }
    //如果不存在，则调用enqueueTx将该笔交易放入交易池中。如果交易是本地发出的，则将发送者保存在交易池的local中
   // New transaction isn't replacing a pending one, push into queue
   replace, err := pool.enqueueTx(hash, tx)
   if err != nil {
      return false, err
   }
    //标记本地地址并记录本地事务
   // Mark local addresses and journal local transactions
   if local {
      if !pool.locals.contains(from) {
         log.Info("Setting new local account", "address", from)
         pool.locals.add(from)
      }
   }
    //记录本地账户的事务交易信息
   pool.journalTx(from, tx)

   log.Trace("Pooled new future transaction", "hash", hash, "from", from, "to", tx.To())
   return replace, nil
}
```

总结:add方法执行流程

1.判断这个交易hash有么有在交易池中，如果交易池中有这笔交易则返回报错

2.调用validateTx判断交易是否合法，如果不合法则返回报错

3.判断交易池是否超过容量

4.如果超过容量，并且该笔交易的费用低于当前交易池中列表的最小值，则拒绝这一笔交易

5.如果超过容量，并且该笔交易的费用比当前交易池中列表最小值高，那么从交易池中移除交易费用最低的交易，为当前这一笔交易留出空间。

6.接着继续调用Overlaps方法检查该笔交易的Nonce值，确认该用户下的交易是否存在该笔交易

7.如果已经存在这笔交易，则删除之前的交易，并将该笔交易放入交易池中，然后返回。

8.如果不存在，则调用enqueueTx将该笔交易放入交易池中。如果交易是本地发出的，则将发送者保存在交易池的local中

9.返回执行结果 true /false和错误信息

validateTx方法执行的逻辑

```go
func (pool *TxPool) validateTx(tx *types.Transaction, local bool) error {
   // Heuristic limit, reject transactions over 32KB to prevent DOS attacks
   if tx.Size() > 32*1024 {
      return ErrOversizedData
   }
   // Transactions can't be negative. This may never happen using RLP decoded
   // transactions but may occur if you create a transaction using the RPC.
   if tx.Value().Sign() < 0 {
      return ErrNegativeValue
   }
   // Ensure the transaction doesn't exceed the current block limit gas.
   if pool.currentMaxGas < tx.Gas() {
      return ErrGasLimit
   }
   // Make sure the transaction is signed properly
   from, err := types.Sender(pool.signer, tx)
   if err != nil {
      return ErrInvalidSender
   }
   // Drop non-local transactions under our own minimal accepted gas price
   local = local || pool.locals.contains(from) // account may be local even if the transaction arrived from the network
   if !local && pool.gasPrice.Cmp(tx.GasPrice()) > 0 {
      return ErrUnderpriced
   }
   // Ensure the transaction adheres to nonce ordering
   if pool.currentState.GetNonce(from) > tx.Nonce() {
      return ErrNonceTooLow
   }
   // Transactor should have enough funds to cover the costs
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

validateTx会验证一笔交易的以下几个特性：
​    1.首先验证这笔交易的大小，如果大于32kb则拒绝这笔交易，这样主要是为了防止DDOS攻击。
​    2.接着验证转账金额。如果金额小于0则拒绝这笔交易。
​    3.这笔交易的gas不能超过交易池的gas上限。
​    4.验证这笔交易的签名是否合法。
​    5.如果这笔交易不是来自本地并且这笔交易的gas小于当前交易池中的gas，则拒绝这笔交易。
​    6.当前用户的nonce如果大于这笔交易的nonce，则拒绝这笔交易。
​    7.当前用户的余额是否充足，如果不充足则拒绝该笔交易。
​    8.验证这笔交易的固有花费，如果小于交易池的gas，则拒绝该笔交易。
以上就是在进行交易验证时所需验证的参数。这一系列的验证操作结束后，回到addTx的第二步。
会判断replace。如果replace是false，则会执行promoteExecutables方法。

```go
func (pool *TxPool) promoteExecutables(accounts []common.Address) {
   // Track the promoted transactions to broadcast them at once
   var promoted []*types.Transaction

   // Gather all the accounts potentially needing updates
   if accounts == nil {
      accounts = make([]common.Address, 0, len(pool.queue))
      for addr := range pool.queue {
         accounts = append(accounts, addr)
      }
   }
   // Iterate over all accounts and promote any executable transactions
   for _, addr := range accounts {
      list := pool.queue[addr]
      if list == nil {
         continue // Just in case someone calls with a non existing account
      }
      // Drop all transactions that are deemed too old (low nonce)
      for _, tx := range list.Forward(pool.currentState.GetNonce(addr)) {
         hash := tx.Hash()
         log.Trace("Removed old queued transaction", "hash", hash)
         pool.all.Remove(hash)
         pool.priced.Removed()
      }
      // Drop all transactions that are too costly (low balance or out of gas)
      drops, _ := list.Filter(pool.currentState.GetBalance(addr), pool.currentMaxGas)
      for _, tx := range drops {
         hash := tx.Hash()
         log.Trace("Removed unpayable queued transaction", "hash", hash)
         pool.all.Remove(hash)
         pool.priced.Removed()
         queuedNofundsCounter.Inc(1)
      }
      // Gather all executable transactions and promote them
      for _, tx := range list.Ready(pool.pendingState.GetNonce(addr)) {
         hash := tx.Hash()
         if pool.promoteTx(addr, hash, tx) {
            log.Trace("Promoting queued transaction", "hash", hash)
            promoted = append(promoted, tx)
         }
      }
      // Drop all transactions over the allowed limit
      if !pool.locals.contains(addr) {
         for _, tx := range list.Cap(int(pool.config.AccountQueue)) {
            hash := tx.Hash()
            pool.all.Remove(hash)
            pool.priced.Removed()
            queuedRateLimitCounter.Inc(1)
            log.Trace("Removed cap-exceeding queued transaction", "hash", hash)
         }
      }
      // Delete the entire queue entry if it became empty.
      if list.Empty() {
         delete(pool.queue, addr)
      }
   }
   // Notify subsystem for new promoted transactions.
   if len(promoted) > 0 {
      go pool.txFeed.Send(NewTxsEvent{promoted})
   }
   // If the pending limit is overflown, start equalizing allowances
   pending := uint64(0)
   for _, list := range pool.pending {
      pending += uint64(list.Len())
   }
   if pending > pool.config.GlobalSlots {
      pendingBeforeCap := pending
      // Assemble a spam order to penalize large transactors first
      spammers := prque.New(nil)
      for addr, list := range pool.pending {
         // Only evict transactions from high rollers
         if !pool.locals.contains(addr) && uint64(list.Len()) > pool.config.AccountSlots {
            spammers.Push(addr, int64(list.Len()))
         }
      }
      // Gradually drop transactions from offenders
      offenders := []common.Address{}
      for pending > pool.config.GlobalSlots && !spammers.Empty() {
         // Retrieve the next offender if not local address
         offender, _ := spammers.Pop()
         offenders = append(offenders, offender.(common.Address))

         // Equalize balances until all the same or below threshold
         if len(offenders) > 1 {
            // Calculate the equalization threshold for all current offenders
            threshold := pool.pending[offender.(common.Address)].Len()

            // Iteratively reduce all offenders until below limit or threshold reached
            for pending > pool.config.GlobalSlots && pool.pending[offenders[len(offenders)-2]].Len() > threshold {
               for i := 0; i < len(offenders)-1; i++ {
                  list := pool.pending[offenders[i]]
                  for _, tx := range list.Cap(list.Len() - 1) {
                     // Drop the transaction from the global pools too
                     hash := tx.Hash()
                     pool.all.Remove(hash)
                     pool.priced.Removed()

                     // Update the account nonce to the dropped transaction
                     if nonce := tx.Nonce(); pool.pendingState.GetNonce(offenders[i]) > nonce {
                        pool.pendingState.SetNonce(offenders[i], nonce)
                     }
                     log.Trace("Removed fairness-exceeding pending transaction", "hash", hash)
                  }
                  pending--
               }
            }
         }
      }
      // If still above threshold, reduce to limit or min allowance
      if pending > pool.config.GlobalSlots && len(offenders) > 0 {
         for pending > pool.config.GlobalSlots && uint64(pool.pending[offenders[len(offenders)-1]].Len()) > pool.config.AccountSlots {
            for _, addr := range offenders {
               list := pool.pending[addr]
               for _, tx := range list.Cap(list.Len() - 1) {
                  // Drop the transaction from the global pools too
                  hash := tx.Hash()
                  pool.all.Remove(hash)
                  pool.priced.Removed()

                  // Update the account nonce to the dropped transaction
                  if nonce := tx.Nonce(); pool.pendingState.GetNonce(addr) > nonce {
                     pool.pendingState.SetNonce(addr, nonce)
                  }
                  log.Trace("Removed fairness-exceeding pending transaction", "hash", hash)
               }
               pending--
            }
         }
      }
      pendingRateLimitCounter.Inc(int64(pendingBeforeCap - pending))
   }
   // If we've queued more transactions than the hard limit, drop oldest ones
   queued := uint64(0)
   for _, list := range pool.queue {
      queued += uint64(list.Len())
   }
   if queued > pool.config.GlobalQueue {
      // Sort all accounts with queued transactions by heartbeat
      addresses := make(addressesByHeartbeat, 0, len(pool.queue))
      for addr := range pool.queue {
         if !pool.locals.contains(addr) { // don't drop locals
            addresses = append(addresses, addressByHeartbeat{addr, pool.beats[addr]})
         }
      }
      sort.Sort(addresses)

      // Drop transactions until the total is below the limit or only locals remain
      for drop := queued - pool.config.GlobalQueue; drop > 0 && len(addresses) > 0; {
         addr := addresses[len(addresses)-1]
         list := pool.queue[addr.address]

         addresses = addresses[:len(addresses)-1]

         // Drop all transactions if they are less than the overflow
         if size := uint64(list.Len()); size <= drop {
            for _, tx := range list.Flatten() {
               pool.removeTx(tx.Hash(), true)
            }
            drop -= size
            queuedRateLimitCounter.Inc(int64(size))
            continue
         }
         // Otherwise drop only last few transactions
         txs := list.Flatten()
         for i := len(txs) - 1; i >= 0 && drop > 0; i-- {
            pool.removeTx(txs[i].Hash(), true)
            drop--
            queuedRateLimitCounter.Inc(1)
         }
      }
   }
}
```

在promoteExecutable中有一个promoteTx方法，这个方法是将交易防区pending区方法中。在promoteTx方法中，最后一步执行的是一个Send方法。
这个Send方法会同步将pending区的交易广播至它所连接到的节点，并返回通知到的节点的数量。
然后被通知到的节点继续通知到它添加的节点，继而广播至全网。

至此，发送交易就结束了。此时交易池中的交易等待挖矿打包处理

