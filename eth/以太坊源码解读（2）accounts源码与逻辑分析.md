# Accounts源码分析与逻辑结构1

总所周知以太坊在比特币的基础上加以引用与改进，比特币使用UTXO来表示状态的转移，而以太坊使用账来表示状态的转移。

### accounts包实现了以太坊客户端的钱包和账户管理

##### 在以太坊网络中存在两种账户：

外部账户EOA：一般是属于个人或者用户的账户，被私钥控制没有任何代码与之相关

内部账户CA：给智能合约分配的账户，被合约代码控制，且与合约关联

在源码core/state/state_object.go文件下，账户定义如下：

```go
// Account is the Ethereum consensus representation of accounts.
// These objects are stored in the main account trie.
type Account struct {
   Nonce    uint64
   Balance  *big.Int
   Root     common.Hash // merkle root of the storage trie
   CodeHash []byte
}
```

Nonce：如果是EOA账户表示发送交易的序号，如果为CA账户，则Nonce表示合约创建的序号

Balance：表示账户的余额，该账户地址对应的账户余额

Root：存储merkle树的根，如果为EOA账户root为nil

CodeHash：账户绑定的EVM Code，如果为EOA则CodeHash为nil

##### 钱包interface，是指包含了一个或多个账户的软件钱包或者硬件钱包:

```go
type Wallet interface {
 // URL 用来获取这个钱包可以访问的规范路径。它会被上层使用用来从所有的后端的钱包来排序。
   URL() URL
    
 // 用来返回一个文本值用来标识当前钱包的状态。同时也会返回一个error用来标识钱包遇到的任何错误。
   Status() (string, error)
    
  //Open初始化对钱包实例的访问。如果你open了一个钱包，你必须close它。
   Open(passphrase string) error
    
 // Close 释放由Open方法占用的任何资源。           
   Close() error
    
  // Accounts用来获取钱包发现了账户列表。对于分层次的钱包，这个列表不会详尽的列出所有的账号，而是只包   //含在帐户派生期间明确固定的帐户。
   Accounts() []Account
    
  //包含返回帐户是否属于此特定钱包的一部分。
   Contains(account Account) bool
    
 //Derive尝试在指定的派生路径上显式派生出分层确定性帐户。如果pin为true，派生帐户将被添加到钱包的跟踪  //帐户列表中。
   Derive(path DerivationPath, pin bool) (Account, error)
    
 //SelfDerive设置一个基本帐户导出路径，从中钱包尝试发现非零帐户，并自动将其添加到跟踪帐户列表中。
   SelfDerive(base DerivationPath, chain ethereum.ChainStateReader)
    
 // SignHash 请求钱包来给传入的hash进行签名。
   SignHash(account Account, hash []byte) ([]byte, error)
    
  // SignTx 请求钱包对指定的交易进行签名。
   SignTx(account Account, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error)
    
 //SignHashWithPassphrase请求钱包使用给定的passphrase来签名给定的hash
   SignHashWithPassphrase(account Account, passphrase string, hash []byte) ([]byte, error)

   // SignHashWithPassphrase请求钱包使用给定的passphrase来签名给定的
   SignTxWithPassphrase(account Account, passphrase string, tx *types.Transaction, chainID *big.Int) (*types.Transaction, error)
}
```

 

##### 后端Backend，Backend是一个钱包提供器。可以包含一批账号。他们可以根据请求签署交易

```go
type Backend interface {
   // Wallets获取当前能够查找到的钱包
   Wallets() []Wallet

  // 订阅创建异步订阅，以便在后端检测到钱包的到达或离开时接收通知。
   Subscribe(sink chan<- WalletEvent) event.Subscription
}
```

##### manager.go:Manager是一个包含所有东西的账户管理工具。可以和所有的Backends来通信来签署交易

```go
type Manager struct {
    // 当前注册的后端索引
	backends map[reflect.Type][]Backend 
    
    // 钱包更新订阅所有后端
	updaters []event.Subscription 
    
   // 钱包更新订阅接收器
	updates  chan WalletEvent           
    
   // 缓存所有钱包从所有注册后端
	wallets  []Wallet          
    
   // 通知到达/离开的钱包事件
	feed event.Feed 
	
    //退出数据管道 错误信息
	quit chan chan error
    
    //读写互斥锁
	lock sync.RWMutex
}
```

### newAccount源码解读

了解了账户的结构以后，我们来看看和账户有关的代码，因为以太坊源码的分离性，数据结构的定义和工具方法逻辑实现比较分离，在整个流程的执行中或调用多层。

首先当用户在console也就是控制台输入personal.newAccount()会创建一个新的账户这个命令的执行流程如下：
1）执行internal/ethapi/api.go文件中的NewAccount方法，返回账户地址

```go
func (s *PrivateAccountAPI) NewAccount(password string) (common.Address, error) {
   acc, err := fetchKeystore(s.am).NewAccount(password)
   if err == nil {
      return acc.Address, nil
   }
   return common.Address{}, err
}
```

internal/ethapi/api.go文件中的NewAccount方法调用fetchKeystore方法从帐户管理器检索加密的密钥存储库获取keystore。

```go
func fetchKeystore(am *accounts.Manager) *keystore.KeyStore {
   return am.Backends(keystore.KeyStoreType)[0].(*keystore.KeyStore)
}
```

internal/ethapi/api.go文件中的NewAccount方法获取到keystore后通过keystore调用accounts/keystore/keystore.go中的NewAccoun方法获取account,并将这个账户添加到keystore中，返回account

```go
func (ks *KeyStore) NewAccount(passphrase string) (accounts.Account, error) {
   _, account, err := storeNewKey(ks.storage, crand.Reader, passphrase)
   if err != nil {
      return accounts.Account{}, err
   }
   // Add the account to the cache immediately rather
   // than waiting for file system notifications to pick it up.
   ks.cache.add(account)
   ks.refreshWallets()
   return account, nil
}

```

调用storeNewKey方法创建一个新的账户，生成一对公私钥，通过私钥以及地址构建一个账户

```go
func storeNewKey(ks keyStore, rand io.Reader, auth string) (*Key, accounts.Account, error) {
   key, err := newKey(rand)
   if err != nil {
      return nil, accounts.Account{}, err
   }
   a := accounts.Account{Address: key.Address, URL: accounts.URL{Scheme: KeyStoreScheme, Path: ks.JoinPath(keyFileName(key.Address))}}
   if err := ks.StoreKey(a.URL.Path, key, auth); err != nil {
      zeroKey(key.PrivateKey)
      return nil, a, err
   }
   return key, a, err
}

```

Key的生成函数，通过椭圆曲线加密生成的私钥，生成Key

```go
func newKey(rand io.Reader) (*Key, error) {
   privateKeyECDSA, err := ecdsa.GenerateKey(crypto.S256(), rand)
   if err != nil {
      return nil, err
   }
   return newKeyFromECDSA(privateKeyECDSA), nil
}
```

生成公钥和私钥对,`ecdsa.GenerateKey(crypto.S256(), rand)` 以太坊采用了椭圆曲线数字签名算法（ECDSA）生成一对公私钥，并选择的是secp256k1曲线

```GO
func GenerateKey(c elliptic.Curve, rand io.Reader) (*PrivateKey, error) {
   k, err := randFieldElement(c, rand)
   if err != nil {
      return nil, err
   }

   priv := new(PrivateKey)
   priv.PublicKey.Curve = c
   priv.D = k
   priv.PublicKey.X, priv.PublicKey.Y = c.ScalarBaseMult(k.Bytes())
   return priv, nil
}
```

```GO
func randFieldElement(c elliptic.Curve, rand io.Reader) (k *big.Int, err error) {
   params := c.Params()
   b := make([]byte, params.BitSize/8+8)
   _, err = io.ReadFull(rand, b)
   if err != nil {
      return
   }

   k = new(big.Int).SetBytes(b)
   n := new(big.Int).Sub(params.N, one)
   k.Mod(k, n)
   k.Add(k, one)
   return
}
```

以太坊使用私钥通过 ECDSA算法推导出公钥，继而经过 Keccak-256 单向散列函数推导出地址

```GO
func newKeyFromECDSA(privateKeyECDSA *ecdsa.PrivateKey) *Key {
   id := uuid.NewRandom()
   key := &Key{
      Id:         id,
      Address:    crypto.PubkeyToAddress(privateKeyECDSA.PublicKey),
      PrivateKey: privateKeyECDSA,
   }
   return key
}
```

地址代币以太坊的20位地址hash

```go
// Address represents the 20 byte address of an Ethereum account.
type Address [AddressLength]byte
```

整个过程可以总结为：

从前控制台传入创建账户命令

首先创建随机私钥

通过私钥导出公钥

通过公私钥导出地址