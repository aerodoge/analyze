# 区块链开发工程师详细学习计划

> **目标**：3个月系统学习，达到95%面试通过率
> **学习时间**：工作日每天2-3小时，周末每天4-6小时
> **总时长**：约200小时

---

## 学习原则

1. **理解优于记忆**：不要死记硬背，理解原理
2. **动手实践**：每个知识点都要写代码验证
3. **循序渐进**：从简单到复杂，不跳步
4. **及时复习**：每周末复习本周内容
5. **画图总结**：用架构图和流程图总结知识

---

# 第一阶段：区块链基础 + 密码学（Week 1-2）

**目标**：建立区块链基础认知，掌握核心密码学原理

## Week 1: 区块链基础理论

### Day 1（周一）：区块链概念入门

**学习内容（2小时）**：

1. **区块链是什么**（30分钟）
   - 阅读：Bitcoin白皮书前3页
   - 理解：去中心化、不可篡改、共识
   - 画图：区块链数据结构

2. **区块结构**（60分钟）
   ```go
   // 动手实践：实现简单的区块结构
   type Block struct {
       Index        int64
       Timestamp    int64
       Data         string
       PrevHash     string
       Hash         string
       Nonce        int64
   }

   // TODO: 实现Hash计算函数
   func (b *Block) CalculateHash() string {
       // 使用SHA256计算
   }
   ```

3. **创世区块**（30分钟）
   ```go
   // 实现创世区块
   func CreateGenesisBlock() *Block {
       return &Block{
           Index:     0,
           Timestamp: time.Now().Unix(),
           Data:      "Genesis Block",
           PrevHash:  "0",
       }
   }
   ```

**作业**：
- [ ] 实现Block结构体（包含Hash计算）
- [ ] 实现创世区块创建函数
- [ ] 理解为什么需要PrevHash

**检查点**：
- [ ] 能用自己的话解释什么是区块链
- [ ] 能画出区块链数据结构图
- [ ] 代码能运行并输出正确的哈希

**参考资料**：
- Bitcoin白皮书：https://bitcoin.org/bitcoin.pdf
- blockchain-study-guide.md 第一部分 1.1节

---

### Day 2（周二）：哈希与链式结构

**学习内容（2.5小时）**：

1. **哈希函数特性**（45分钟）
   - 确定性：相同输入→相同输出
   - 不可逆：无法从哈希反推原文
   - 雪崩效应：输入微小变化→哈希巨变
   - 碰撞抗性：几乎不可能找到两个相同哈希

   ```go
   // 实践：观察雪崩效应
   func TestAvalancheEffect() {
       data1 := "Hello, Blockchain"
       data2 := "Hello, Blockchain!"  // 只差一个字符

       hash1 := sha256.Sum256([]byte(data1))
       hash2 := sha256.Sum256([]byte(data2))

       fmt.Printf("Hash1: %x\n", hash1)
       fmt.Printf("Hash2: %x\n", hash2)
       // 观察：两个哈希完全不同
   }
   ```

2. **实现区块链**（75分钟）
   ```go
   type Blockchain struct {
       Blocks []*Block
   }

   func (bc *Blockchain) AddBlock(data string) {
       prevBlock := bc.Blocks[len(bc.Blocks)-1]
       newBlock := &Block{
           Index:     prevBlock.Index + 1,
           Timestamp: time.Now().Unix(),
           Data:      data,
           PrevHash:  prevBlock.Hash,
       }
       newBlock.Hash = newBlock.CalculateHash()
       bc.Blocks = append(bc.Blocks, newBlock)
   }

   // 验证区块链完整性
   func (bc *Blockchain) IsValid() bool {
       for i := 1; i < len(bc.Blocks); i++ {
           currentBlock := bc.Blocks[i]
           prevBlock := bc.Blocks[i-1]

           // 检查当前区块哈希
           if currentBlock.Hash != currentBlock.CalculateHash() {
               return false
           }

           // 检查链接
           if currentBlock.PrevHash != prevBlock.Hash {
               return false
           }
       }
       return true
   }
   ```

3. **测试不可篡改性**（30分钟）
   ```go
   func TestImmutability() {
       bc := NewBlockchain()
       bc.AddBlock("Block 1")
       bc.AddBlock("Block 2")
       bc.AddBlock("Block 3")

       fmt.Println("原链有效:", bc.IsValid())  // true

       // 尝试篡改
       bc.Blocks[1].Data = "Hacked Block 1"
       fmt.Println("篡改后有效:", bc.IsValid())  // false

       // 解释：为什么返回false？
   }
   ```

**作业**：
- [ ] 实现完整的Blockchain结构
- [ ] 实现AddBlock和IsValid方法
- [ ] 测试篡改检测

**检查点**：
- [ ] 能解释哈希函数的4个特性
- [ ] 能说明为什么修改历史区块会被检测到
- [ ] 代码能正确检测篡改

---

### Day 3（周三）：工作量证明（PoW）

**学习内容（3小时）**：

1. **PoW原理**（60分钟）
   - 理解：挖矿 = 寻找满足难度的Nonce
   - 难度：Hash必须以N个0开头
   - 示例：难度4 → Hash以"0000"开头

   ```go
   // 实现PoW
   func (b *Block) Mine(difficulty int) {
       target := strings.Repeat("0", difficulty)

       for {
           hash := b.CalculateHashWithNonce()

           if strings.HasPrefix(hash, target) {
               b.Hash = hash
               fmt.Printf("挖矿成功！Nonce: %d\n", b.Nonce)
               break
           }

           b.Nonce++

           // 每10万次打印进度
           if b.Nonce%100000 == 0 {
               fmt.Printf("尝试了 %d 次...\n", b.Nonce)
           }
       }
   }
   ```

2. **难度调整**（45分钟）
   ```go
   // 目标：每个区块10秒
   func (bc *Blockchain) AdjustDifficulty() int {
       if len(bc.Blocks) < 2 {
           return bc.Difficulty
       }

       lastBlock := bc.Blocks[len(bc.Blocks)-1]
       prevBlock := bc.Blocks[len(bc.Blocks)-2]

       timeTaken := lastBlock.Timestamp - prevBlock.Timestamp
       targetTime := int64(10)  // 10秒

       if timeTaken < targetTime/2 {
           // 太快，增加难度
           return bc.Difficulty + 1
       } else if timeTaken > targetTime*2 {
           // 太慢，降低难度
           if bc.Difficulty > 1 {
               return bc.Difficulty - 1
           }
       }

       return bc.Difficulty
   }
   ```

3. **性能测试**（45分钟）
   ```go
   // 测试不同难度的挖矿时间
   func BenchmarkMining() {
       for difficulty := 1; difficulty <= 6; difficulty++ {
           block := &Block{
               Index:     1,
               Timestamp: time.Now().Unix(),
               Data:      "Test Block",
           }

           start := time.Now()
           block.Mine(difficulty)
           elapsed := time.Since(start)

           fmt.Printf("难度 %d: %v, Nonce: %d\n",
               difficulty, elapsed, block.Nonce)
       }
   }

   // 预期结果：
   // 难度 1: ~1ms
   // 难度 2: ~10ms
   // 难度 3: ~100ms
   // 难度 4: ~1s
   // 难度 5: ~10s
   // 难度 6: ~100s
   ```

**作业**：
- [ ] 实现PoW挖矿函数
- [ ] 实现难度调整算法
- [ ] 测试不同难度的性能

**检查点**：
- [ ] 能解释PoW如何防止双花攻击
- [ ] 能计算给定难度的预期挖矿时间
- [ ] 理解为什么难度需要动态调整

**面试问题准备**：
- Q: 为什么比特币是10分钟，以太坊是12秒？
- A: 权衡安全性和确认速度。时间越短，分叉概率越高...

---

### Day 4（周四）：交易与数字签名

**学习内容（2.5小时）**：

1. **非对称加密基础**（45分钟）
   ```go
   // 生成密钥对
   import (
       "crypto/ecdsa"
       "crypto/elliptic"
       "crypto/rand"
   )

   func GenerateKeyPair() (*ecdsa.PrivateKey, *ecdsa.PublicKey) {
       privateKey, err := ecdsa.GenerateKey(elliptic.P256(), rand.Reader)
       if err != nil {
           panic(err)
       }
       return privateKey, &privateKey.PublicKey
   }

   // 私钥 → 公钥 → 地址
   func PublicKeyToAddress(pubKey *ecdsa.PublicKey) string {
       // 简化版：实际使用Keccak256
       pubKeyBytes := append(pubKey.X.Bytes(), pubKey.Y.Bytes()...)
       hash := sha256.Sum256(pubKeyBytes)
       return fmt.Sprintf("%x", hash[:20])  // 取前20字节
   }
   ```

2. **交易结构**（60分钟）
   ```go
   type Transaction struct {
       From      string
       To        string
       Amount    float64
       Timestamp int64
       Signature []byte
   }

   // 签名交易
   func (tx *Transaction) Sign(privateKey *ecdsa.PrivateKey) error {
       // 1. 计算交易哈希
       txHash := tx.Hash()

       // 2. 使用私钥签名
       r, s, err := ecdsa.Sign(rand.Reader, privateKey, txHash)
       if err != nil {
           return err
       }

       // 3. 编码签名
       tx.Signature = append(r.Bytes(), s.Bytes()...)
       return nil
   }

   // 验证签名
   func (tx *Transaction) Verify(publicKey *ecdsa.PublicKey) bool {
       // 1. 分离r和s
       r := new(big.Int).SetBytes(tx.Signature[:32])
       s := new(big.Int).SetBytes(tx.Signature[32:])

       // 2. 验证
       txHash := tx.Hash()
       return ecdsa.Verify(publicKey, txHash, r, s)
   }
   ```

3. **UTXO模型 vs 账户模型**（45分钟）
   ```go
   // UTXO模型（比特币）
   type UTXO struct {
       TxID   string
       Index  int
       Amount float64
       Owner  string
   }

   // 示例：Alice有10 BTC（来自3个UTXO）
   utxos := []UTXO{
       {TxID: "tx1", Index: 0, Amount: 3, Owner: "Alice"},
       {TxID: "tx2", Index: 1, Amount: 5, Owner: "Alice"},
       {TxID: "tx3", Index: 0, Amount: 2, Owner: "Alice"},
   }
   // Alice要给Bob 7 BTC：
   // Input: tx1:0 (3) + tx2:1 (5) = 8 BTC
   // Output: Bob (7) + Alice找零 (1)

   // 账户模型（以太坊）
   type Account struct {
       Address string
       Balance float64
       Nonce   uint64
   }

   // 示例：直接扣减余额
   func Transfer(from, to *Account, amount float64) {
       from.Balance -= amount
       to.Balance += amount
       from.Nonce++  // 防止重放攻击
   }
   ```

**作业**：
- [ ] 实现交易签名和验证
- [ ] 对比UTXO和账户模型的优缺点
- [ ] 测试签名验证流程

**检查点**：
- [ ] 能解释为什么需要数字签名
- [ ] 能说出UTXO和账户模型的区别
- [ ] 理解Nonce的作用（防重放）

---

### Day 5（周五）：Merkle Tree

**学习内容（2.5小时）**：

1. **Merkle Tree原理**（60分钟）
   ```
   画图理解：

         Root Hash
            |
      +-----+-----+
      |           |
   Hash(A,B)  Hash(C,D)
      |           |
   +--+--+     +--+--+
   |     |     |     |
   A     B     C     D

   特性：
   1. 修改任何叶子节点 → Root Hash改变
   2. 验证只需O(log n)个哈希
   ```

2. **实现Merkle Tree**（90分钟）
   ```go
   type MerkleNode struct {
       Left  *MerkleNode
       Right *MerkleNode
       Data  []byte
       Hash  []byte
   }

   type MerkleTree struct {
       Root *MerkleNode
   }

   func NewMerkleNode(left, right *MerkleNode, data []byte) *MerkleNode {
       node := &MerkleNode{}

       if left == nil && right == nil {
           // 叶子节点
           hash := sha256.Sum256(data)
           node.Data = data
           node.Hash = hash[:]
       } else {
           // 中间节点
           prevHashes := append(left.Hash, right.Hash...)
           hash := sha256.Sum256(prevHashes)
           node.Hash = hash[:]
       }

       node.Left = left
       node.Right = right

       return node
   }

   func NewMerkleTree(data [][]byte) *MerkleTree {
       var nodes []*MerkleNode

       // 创建叶子节点
       for _, d := range data {
           node := NewMerkleNode(nil, nil, d)
           nodes = append(nodes, node)
       }

       // 构建树
       for len(nodes) > 1 {
           var level []*MerkleNode

           for i := 0; i < len(nodes); i += 2 {
               if i+1 < len(nodes) {
                   node := NewMerkleNode(nodes[i], nodes[i+1], nil)
                   level = append(level, node)
               } else {
                   // 奇数个节点，复制最后一个
                   node := NewMerkleNode(nodes[i], nodes[i], nil)
                   level = append(level, node)
               }
           }

           nodes = level
       }

       return &MerkleTree{Root: nodes[0]}
   }
   ```

3. **Merkle Proof**（30分钟）
   ```go
   // 证明某交易存在于区块中
   type MerkleProof struct {
       Hashes  [][]byte
       Index   int
   }

   func (mt *MerkleTree) GenerateProof(data []byte) *MerkleProof {
       // 实现：生成从叶子到根的路径
       // 返回兄弟节点的哈希
   }

   func VerifyProof(data []byte, proof *MerkleProof, root []byte) bool {
       hash := sha256.Sum256(data)

       for _, siblingHash := range proof.Hashes {
           // 按顺序哈希
           combined := append(hash[:], siblingHash...)
           h := sha256.Sum256(combined)
           hash = h
       }

       return bytes.Equal(hash[:], root)
   }
   ```

**作业**：
- [ ] 实现完整的Merkle Tree
- [ ] 实现Merkle Proof生成和验证
- [ ] 理解为什么比特币使用Merkle Tree

**检查点**：
- [ ] 能画出Merkle Tree结构
- [ ] 能解释轻节点如何验证交易（SPV）
- [ ] 理解Merkle Proof的O(log n)复杂度

---

### 周末实践（Week 1）：构建完整的简易区块链

**项目：SimpleCoin**（6小时）

```go
// 目标：实现一个包含以下功能的区块链：
// 1. PoW共识
// 2. 交易签名
// 3. Merkle Tree
// 4. 余额查询

package main

type SimpleCoin struct {
    blockchain *Blockchain
    accounts   map[string]*Account
    mempool    []*Transaction
}

// 功能清单
// [ ] 创建账户（生成密钥对）
// [ ] 转账（创建并签名交易）
// [ ] 挖矿（打包交易到区块）
// [ ] 查询余额
// [ ] 验证区块链完整性

func main() {
    // 1. 初始化
    sc := NewSimpleCoin()

    // 2. 创建账户
    alice := sc.CreateAccount("Alice")
    bob := sc.CreateAccount("Bob")

    // 3. 初始分配（创世区块）
    sc.InitialAllocation(alice.Address, 100)

    // 4. 转账
    tx := sc.CreateTransaction(alice, bob.Address, 30)
    sc.AddToMempool(tx)

    // 5. 挖矿
    sc.Mine(alice.Address)  // Alice挖矿获得奖励

    // 6. 查询余额
    fmt.Printf("Alice: %.2f\n", sc.GetBalance(alice.Address))
    fmt.Printf("Bob: %.2f\n", sc.GetBalance(bob.Address))

    // 7. 验证
    fmt.Printf("区块链有效: %v\n", sc.IsValid())
}
```

**实现步骤**：

1. **创建项目结构**（30分钟）
   ```
   simplecoin/
   ├── main.go
   ├── blockchain.go
   ├── block.go
   ├── transaction.go
   ├── account.go
   ├── merkle.go
   └── pow.go
   ```

2. **实现核心功能**（4小时）
   - 区块链管理
   - 交易处理
   - PoW挖矿
   - 账户管理

3. **测试与调试**（1.5小时）
   - 单元测试
   - 集成测试
   - 边界情况测试

**检查点**：
- [ ] 代码能运行
- [ ] 转账功能正常
- [ ] 挖矿能获得奖励
- [ ] 篡改能被检测

**提交到GitHub**：
- 创建仓库：simplecoin
- 写好README.md（包含设计思路）
- 这个项目可以在面试时展示

---

## Week 2: 以太坊架构基础

### Day 6（周一）：以太坊概述

**学习内容（2小时）**：

1. **以太坊 vs 比特币**（45分钟）
   ```
   对比表：
   ┌──────────────┬────────────┬────────────┐
   │    特性      │  Bitcoin   │ Ethereum   │
   ├──────────────┼────────────┼────────────┤
   │ 定位         │ 数字货币   │ 世界计算机  │
   │ 脚本语言     │ Script     │ Solidity   │
   │ 图灵完备     │ ✗          │ ✓          │
   │ 状态模型     │ UTXO       │ Account    │
   │ 出块时间     │ 10分钟     │ 12秒       │
   │ 共识         │ PoW        │ PoS        │
   │ 智能合约     │ 受限       │ 完整支持    │
   └──────────────┴────────────┴────────────┘
   ```

2. **以太坊架构图**（45分钟）
   ```
   画出以太坊分层架构：

   ┌────────────────────────────┐
   │    DApp Layer (应用层)      │
   │   - DeFi, NFT, DAO...      │
   ├────────────────────────────┤
   │    Smart Contract Layer    │
   │   - Solidity合约           │
   ├────────────────────────────┤
   │    EVM Layer (执行层)       │
   │   - 虚拟机、Gas机制         │
   ├────────────────────────────┤
   │    Consensus Layer (共识层) │
   │   - PoS (Merge后)          │
   ├────────────────────────────┤
   │    Network Layer (网络层)   │
   │   - P2P、DevP2P            │
   ├────────────────────────────┤
   │    Storage Layer (存储层)   │
   │   - State、Blockchain DB   │
   └────────────────────────────┘
   ```

3. **安装Geth**（30分钟）
   ```bash
   # 安装go-ethereum
   git clone https://github.com/ethereum/go-ethereum.git
   cd go-ethereum
   make geth

   # 启动开发网络
   geth --dev --http --http.api eth,net,web3,personal

   # 另开终端，连接geth console
   geth attach http://localhost:8545

   # 测试基本命令
   > eth.accounts
   > eth.blockNumber
   > eth.getBalance(eth.accounts[0])
   ```

**作业**：
- [ ] 画出以太坊架构图（手绘或工具）
- [ ] 安装Geth并启动开发网络
- [ ] 熟悉geth console基本命令

**检查点**：
- [ ] 能说出以太坊的5大创新（vs 比特币）
- [ ] Geth能正常运行
- [ ] 能用geth console查询余额

---

### Day 7（周二）：账户与状态

**学习内容（2.5小时）**：

1. **账户模型**（60分钟）
   ```go
   // 以太坊账户结构
   type Account struct {
       Nonce    uint64      // 交易计数
       Balance  *big.Int    // 余额（wei）
       Root     common.Hash // 存储树根（合约）
       CodeHash []byte      // 代码哈希（合约）
   }

   // 两种账户
   // EOA（Externally Owned Account）：
   // - 由私钥控制
   // - 无代码
   // - 可以发起交易

   // CA（Contract Account）：
   // - 由代码控制
   // - 有存储
   // - 不能主动发起交易

   // 示例
   func ExampleAccounts() {
       // EOA
       privateKey, _ := crypto.GenerateKey()
       address := crypto.PubkeyToAddress(privateKey.PublicKey)

       eoaAccount := &Account{
           Nonce:    0,
           Balance:  big.NewInt(1000000000000000000),  // 1 ETH
           Root:     common.Hash{},
           CodeHash: crypto.Keccak256(nil),  // 空代码
       }

       // CA
       contractAccount := &Account{
           Nonce:    1,  // 创建的合约数量
           Balance:  big.NewInt(0),
           Root:     storageRootHash,  // 合约存储
           CodeHash: contractCodeHash,
       }
   }
   ```

2. **世界状态（World State）**（60分钟）
   ```go
   // 世界状态 = 所有账户的映射
   type WorldState map[common.Address]*Account

   // 状态转换函数
   func ApplyTransaction(
       state WorldState,
       tx *Transaction,
   ) (WorldState, *Receipt, error) {
       // 1. 验证
       if !ValidateTransaction(tx, state) {
           return state, nil, ErrInvalidTx
       }

       // 2. 扣除Gas预付费
       sender := tx.From()
       state[sender].Balance -= tx.GasLimit * tx.GasPrice

       // 3. 执行交易
       if tx.To == nil {
           // 合约创建
           contractAddr := CreateContract(state, tx)
           receipt := &Receipt{ContractAddress: contractAddr}
           return state, receipt, nil
       } else {
           // 转账或合约调用
           state[*tx.To].Balance += tx.Value
           state[sender].Balance -= tx.Value
       }

       // 4. 退还剩余Gas
       // 5. 矿工奖励

       return state, receipt, nil
   }
   ```

3. **实践：简易状态管理**（30分钟）
   ```go
   // 实现一个简单的StateDB
   type SimpleStateDB struct {
       accounts map[common.Address]*Account
       dirty    map[common.Address]struct{}
   }

   func (s *SimpleStateDB) GetBalance(addr common.Address) *big.Int {
       if acc, ok := s.accounts[addr]; ok {
           return acc.Balance
       }
       return big.NewInt(0)
   }

   func (s *SimpleStateDB) SetBalance(addr common.Address, amount *big.Int) {
       if _, ok := s.accounts[addr]; !ok {
           s.accounts[addr] = &Account{}
       }
       s.accounts[addr].Balance = amount
       s.dirty[addr] = struct{}{}
   }

   func (s *SimpleStateDB) Transfer(from, to common.Address, amount *big.Int) error {
       fromBalance := s.GetBalance(from)
       if fromBalance.Cmp(amount) < 0 {
           return ErrInsufficientBalance
       }

       s.SetBalance(from, new(big.Int).Sub(fromBalance, amount))
       s.SetBalance(to, new(big.Int).Add(s.GetBalance(to), amount))

       return nil
   }
   ```

**作业**：
- [ ] 实现SimpleStateDB
- [ ] 实现Transfer方法
- [ ] 测试余额不足的情况

**检查点**：
- [ ] 能区分EOA和CA
- [ ] 能解释Nonce的作用
- [ ] 理解状态转换函数

---

### Day 8（周三）：交易与Gas

**学习内容（3小时）**：

1. **交易结构详解**（75分钟）
   ```go
   // 以太坊交易
   type Transaction struct {
       Nonce    uint64          // 发送者交易序号
       GasPrice *big.Int        // Gas价格（wei/gas）
       GasLimit uint64          // Gas限制
       To       *common.Address // 接收地址（nil=合约创建）
       Value    *big.Int        // 转账金额
       Data     []byte          // 交易数据

       // 签名（EIP-155）
       V *big.Int
       R *big.Int
       S *big.Int
   }

   // Gas计算
   func CalculateGas(tx *Transaction) uint64 {
       gas := uint64(21000)  // 基础gas

       // 数据gas
       for _, b := range tx.Data {
           if b == 0 {
               gas += 4  // 零字节
           } else {
               gas += 68  // 非零字节
           }
       }

       return gas
   }

   // 交易成本
   func TxCost(tx *Transaction) *big.Int {
       return new(big.Int).Mul(
           new(big.Int).SetUint64(tx.GasLimit),
           tx.GasPrice,
       )
   }

   // 示例
   func ExampleGas() {
       // 简单转账
       tx1 := &Transaction{
           GasLimit: 21000,
           GasPrice: big.NewInt(20000000000),  // 20 Gwei
       }
       cost1 := TxCost(tx1)
       fmt.Printf("转账成本: %s wei = %s ETH\n",
           cost1, weiToEth(cost1))
       // 输出: 0.00042 ETH

       // 复杂合约调用
       tx2 := &Transaction{
           GasLimit: 100000,
           GasPrice: big.NewInt(20000000000),
           Data:     make([]byte, 100),
       }
       cost2 := TxCost(tx2)
       fmt.Printf("合约调用成本: %s ETH\n", weiToEth(cost2))
       // 输出: 0.002 ETH
   }
   ```

2. **EIP-1559（新Gas机制）**（60分钟）
   ```go
   // EIP-1559交易
   type DynamicFeeTx struct {
       ChainID    *big.Int
       Nonce      uint64
       GasTipCap  *big.Int  // 小费上限（给矿工）
       GasFeeCap  *big.Int  // 总费用上限
       Gas        uint64
       To         *common.Address
       Value      *big.Int
       Data       []byte
   }

   // Gas费用计算
   func CalcBaseFee(parentGasUsed, parentGasLimit uint64, parentBaseFee *big.Int) *big.Int {
       // 目标gas使用率：50%
       targetGasUsed := parentGasLimit / 2

       if parentGasUsed == targetGasUsed {
           return parentBaseFee
       }

       if parentGasUsed > targetGasUsed {
           // 使用超过50%，增加baseFee（最多12.5%）
           delta := new(big.Int).Sub(
               big.NewInt(int64(parentGasUsed)),
               big.NewInt(int64(targetGasUsed)),
           )
           delta.Mul(delta, parentBaseFee)
           delta.Div(delta, big.NewInt(int64(targetGasUsed)))
           delta.Div(delta, big.NewInt(8))

           return new(big.Int).Add(parentBaseFee, delta)
       } else {
           // 使用低于50%，降低baseFee
           delta := new(big.Int).Sub(
               big.NewInt(int64(targetGasUsed)),
               big.NewInt(int64(parentGasUsed)),
           )
           delta.Mul(delta, parentBaseFee)
           delta.Div(delta, big.NewInt(int64(targetGasUsed)))
           delta.Div(delta, big.NewInt(8))

           return new(big.Int).Sub(parentBaseFee, delta)
       }
   }

   // 用户实际支付
   func EffectiveGasPrice(tx *DynamicFeeTx, baseFee *big.Int) *big.Int {
       // effectiveGasPrice = min(baseFee + tip, gasFeeCap)
       tip := tx.GasTipCap
       price := new(big.Int).Add(baseFee, tip)

       if price.Cmp(tx.GasFeeCap) > 0 {
           return tx.GasFeeCap
       }
       return price
   }
   ```

3. **实践：交易池（Mempool）**（45分钟）
   ```go
   type TxPool struct {
       pending map[common.Address][]*Transaction
       queued  map[common.Address][]*Transaction
       mu      sync.RWMutex
   }

   func (pool *TxPool) Add(tx *Transaction) error {
       pool.mu.Lock()
       defer pool.mu.Unlock()

       // 验证
       if err := pool.validateTx(tx); err != nil {
           return err
       }

       sender := tx.From()

       // 按nonce排序添加
       pool.pending[sender] = append(pool.pending[sender], tx)
       pool.sortByNonce(pool.pending[sender])

       return nil
   }

   func (pool *TxPool) GetPending(maxGas uint64) []*Transaction {
       var txs []*Transaction
       gasUsed := uint64(0)

       // 按Gas价格排序
       sorted := pool.sortByGasPrice()

       for _, tx := range sorted {
           if gasUsed+tx.GasLimit > maxGas {
               break
           }
           txs = append(txs, tx)
           gasUsed += tx.GasLimit
       }

       return txs
   }
   ```

**作业**：
- [ ] 实现Gas计算函数
- [ ] 实现简单的TxPool
- [ ] 理解EIP-1559的动态baseFee

**检查点**：
- [ ] 能计算交易的Gas成本
- [ ] 能解释为什么有Gas机制
- [ ] 理解EIP-1559的优势

---

### Day 9（周四）：智能合约入门

**学习内容（2.5小时）**：

1. **Solidity基础**（90分钟）
   ```solidity
   // SimpleStorage.sol
   pragma solidity ^0.8.0;

   contract SimpleStorage {
       uint256 private value;

       event ValueChanged(uint256 oldValue, uint256 newValue);

       function set(uint256 newValue) public {
           uint256 oldValue = value;
           value = newValue;
           emit ValueChanged(oldValue, newValue);
       }

       function get() public view returns (uint256) {
           return value;
       }
   }

   // 部署与调用
   // 1. 编译
   //    solc --bin --abi SimpleStorage.sol

   // 2. 部署（使用Go）
   func DeployContract(client *ethclient.Client, auth *bind.TransactOpts) {
       address, tx, instance, err := DeploySimpleStorage(auth, client)
       if err != nil {
           log.Fatal(err)
       }

       fmt.Printf("合约地址: %s\n", address.Hex())
       fmt.Printf("交易哈希: %s\n", tx.Hash().Hex())
   }
   ```

2. **合约字节码**（45分钟）
   ```
   // Solidity → Bytecode → EVM

   Solidity代码:
   value = 123;

   ↓ 编译

   字节码:
   60 7b    PUSH1 0x7b  (123)
   60 00    PUSH1 0x00  (storage slot 0)
   55       SSTORE      (存储)

   ↓ EVM执行

   状态变更:
   storage[0] = 123
   ```

3. **实践：Go调用合约**（45分钟）
   ```go
   // 使用go-ethereum调用合约
   package main

   import (
       "github.com/ethereum/go-ethereum/ethclient"
       "github.com/ethereum/go-ethereum/accounts/abi/bind"
   )

   func main() {
       // 1. 连接节点
       client, _ := ethclient.Dial("http://localhost:8545")

       // 2. 加载合约
       address := common.HexToAddress("0x...")
       instance, _ := NewSimpleStorage(address, client)

       // 3. 调用get（只读，不需要gas）
       value, _ := instance.Get(nil)
       fmt.Printf("当前值: %d\n", value)

       // 4. 调用set（需要交易）
       auth := getAuth()  // 获取授权
       tx, _ := instance.Set(auth, big.NewInt(456))

       fmt.Printf("交易已发送: %s\n", tx.Hash().Hex())

       // 5. 等待确认
       receipt, _ := bind.WaitMined(context.Background(), client, tx)
       fmt.Printf("交易已确认，区块: %d\n", receipt.BlockNumber)
   }
   ```

**作业**：
- [ ] 写一个简单的Solidity合约
- [ ] 编译并查看字节码
- [ ] 用Go部署和调用合约

**检查点**：
- [ ] 能写基本的Solidity合约
- [ ] 理解合约编译过程
- [ ] 能用Go与合约交互

---

### Day 10（周五）：EVM基础

**学习内容（3小时）**：

1. **EVM架构**（75分钟）
   ```
   EVM组成部分：

   ┌──────────────────────────────┐
   │         EVM                  │
   ├──────────────────────────────┤
   │  1. Stack (栈)               │
   │     - 最大1024项             │
   │     - 每项256位              │
   ├──────────────────────────────┤
   │  2. Memory (内存)            │
   │     - 字节数组               │
   │     - 按需扩展               │
   ├──────────────────────────────┤
   │  3. Storage (存储)           │
   │     - 键值对                 │
   │     - 持久化                 │
   ├──────────────────────────────┤
   │  4. Program Counter (PC)     │
   │     - 指向当前指令           │
   ├──────────────────────────────┤
   │  5. Gas                      │
   │     - 剩余gas                │
   └──────────────────────────────┘
   ```

2. **操作码（Opcodes）**（75分钟）
   ```go
   // EVM操作码
   const (
       // 0x0: 停止和算术
       STOP       = 0x00
       ADD        = 0x01
       MUL        = 0x02
       SUB        = 0x03
       DIV        = 0x04

       // 0x10: 比较和位操作
       LT         = 0x10
       GT         = 0x11
       EQ         = 0x14
       AND        = 0x16
       OR         = 0x17
       XOR        = 0x18

       // 0x20: SHA3
       SHA3       = 0x20

       // 0x30: 环境信息
       ADDRESS    = 0x30
       BALANCE    = 0x31
       CALLER     = 0x33
       CALLVALUE  = 0x34

       // 0x50: 栈操作
       POP        = 0x50
       MLOAD      = 0x51
       MSTORE     = 0x52
       SLOAD      = 0x54
       SSTORE     = 0x55
       JUMP       = 0x56
       JUMPI      = 0x57

       // 0x60-0x7f: PUSH
       PUSH1      = 0x60
       PUSH2      = 0x61
       // ...
       PUSH32     = 0x7f

       // 0xf0: 合约操作
       CREATE     = 0xf0
       CALL       = 0xf1
       RETURN     = 0xf3
       DELEGATECALL = 0xf4
       CREATE2    = 0xf5
       SELFDESTRUCT = 0xff
   )

   // Gas成本
   var gasCosts = map[byte]uint64{
       ADD:    3,
       MUL:    5,
       SUB:    3,
       DIV:    5,
       SLOAD:  2100,   // 昂贵
       SSTORE: 20000,  // 非常昂贵
       SHA3:   30,
       CALL:   700,
   }
   ```

3. **简单EVM实现**（60分钟）
   ```go
   type SimpleEVM struct {
       stack   *Stack
       memory  []byte
       storage map[common.Hash]common.Hash
       pc      uint64
       gas     uint64
       code    []byte
   }

   func (evm *SimpleEVM) Run() error {
       for evm.pc < uint64(len(evm.code)) {
           op := evm.code[evm.pc]

           // 扣除gas
           if evm.gas < gasCosts[op] {
               return ErrOutOfGas
           }
           evm.gas -= gasCosts[op]

           switch op {
           case ADD:
               a := evm.stack.Pop()
               b := evm.stack.Pop()
               evm.stack.Push(new(big.Int).Add(a, b))

           case MUL:
               a := evm.stack.Pop()
               b := evm.stack.Pop()
               evm.stack.Push(new(big.Int).Mul(a, b))

           case SSTORE:
               key := evm.stack.Pop()
               value := evm.stack.Pop()
               evm.storage[common.BigToHash(key)] = common.BigToHash(value)

           case SLOAD:
               key := evm.stack.Pop()
               value := evm.storage[common.BigToHash(key)]
               evm.stack.Push(value.Big())

           case PUSH1:
               evm.pc++
               evm.stack.Push(big.NewInt(int64(evm.code[evm.pc])))

           // ... 更多操作码
           }

           evm.pc++
       }

       return nil
   }
   ```

**作业**：
- [ ] 实现简单的EVM（支持10个基本操作码）
- [ ] 运行一段简单的字节码
- [ ] 理解Gas扣除机制

**检查点**：
- [ ] 能解释EVM的5大组件
- [ ] 能说出常见操作码的Gas成本
- [ ] 理解为什么SSTORE最贵

---

### 周末实践（Week 2）：EVM模拟器

**项目：MiniEVM**（6小时）

**目标**：实现一个支持基本操作的EVM模拟器

```go
// 功能清单
// [ ] Stack实现（push/pop/peek）
// [ ] Memory实现（read/write/expand）
// [ ] Storage实现（load/store）
// [ ] 20个基本操作码
// [ ] Gas计算和限制
// [ ] 运行Solidity编译的字节码

package main

type MiniEVM struct {
    stack   *Stack
    memory  *Memory
    storage map[common.Hash]common.Hash
    pc      uint64
    gas     uint64
    gasLimit uint64
    code    []byte
    context Context
}

type Context struct {
    Address   common.Address
    Caller    common.Address
    Value     *big.Int
    GasPrice  *big.Int
    BlockNumber uint64
}

// 示例：运行简单的加法程序
func main() {
    // 字节码：PUSH1 5, PUSH1 3, ADD, STOP
    code := []byte{0x60, 0x05, 0x60, 0x03, 0x01, 0x00}

    evm := NewMiniEVM(code, 100000)
    err := evm.Run()

    result := evm.stack.Pop()
    fmt.Printf("结果: %d\n", result)  // 8
}
```

**实现步骤**：

1. **Stack实现**（1小时）
2. **Memory实现**（1小时）
3. **基本操作码**（2小时）
4. **测试与调试**（1小时）
5. **运行真实字节码**（1小时）

**检查点**：
- [ ] 能正确执行算术操作
- [ ] 能处理存储读写
- [ ] Gas计算正确

---

## 第一阶段总结与检查

### Week 1-2 学习成果检查清单

**理论知识**：
- [ ] 能解释区块链的不可篡改性原理
- [ ] 能说出PoW的工作流程
- [ ] 理解UTXO和账户模型的区别
- [ ] 能画出Merkle Tree结构
- [ ] 理解以太坊的状态模型
- [ ] 能解释Gas机制的目的
- [ ] 理解EVM的基本架构

**实践能力**：
- [ ] 实现了SimpleCoin项目
- [ ] 实现了MiniEVM项目
- [ ] 能部署和调用智能合约
- [ ] 能运行Geth节点

**代码量统计**：
- SimpleCoin: ~500行
- MiniEVM: ~300行
- 练习代码: ~200行
- **总计**: ~1000行Go代码

### 下一阶段预习

**Week 3-4重点**：
1. Go-Ethereum源码阅读
2. StateDB深度分析
3. MPT数据结构
4. Transaction Pool实现

**准备工作**：
- [ ] Clone go-ethereum仓库
- [ ] 配置好IDE（推荐VSCode + Go插件）
- [ ] 安装调试工具

---

**继续到第二阶段 →**

（由于文档过长，我会在你确认后继续写剩余阶段）

你觉得这个详细程度如何？需要我继续写后面的阶段吗？

---

# 第二阶段：Go-Ethereum源码深度学习（Week 3-4）

**目标**：深入理解以太坊核心实现，掌握StateDB、MPT、TxPool

## Week 3: StateDB + MPT源码分析

### Day 11（周一）：环境搭建 + 源码导读

**学习内容（2.5小时）**：

1. **Go-Ethereum源码结构**（60分钟）
   ```bash
   # 1. Clone仓库
   cd ~/go/src
   git clone https://github.com/ethereum/go-ethereum.git
   cd go-ethereum

   # 2. 查看目录结构
   tree -L 1 -d

   # 核心目录说明
   go-ethereum/
   ├── accounts/      # 账户管理、钱包
   ├── cmd/          # 命令行工具（geth、clef等）
   ├── consensus/    # 共识引擎（ethash、clique、beacon）
   ├── core/         # ⭐核心逻辑（区块链、状态、EVM）
   │   ├── state/    # ⭐⭐ 状态管理（重点）
   │   ├── types/    # ⭐⭐ 核心数据类型
   │   ├── vm/       # ⭐⭐ EVM实现
   │   └── rawdb/    # 数据库操作
   ├── eth/          # 以太坊协议实现
   ├── ethdb/        # 数据库接口
   ├── p2p/          # ⭐ P2P网络
   ├── trie/         # ⭐⭐ MPT实现（重点）
   └── crypto/       # ⭐ 密码学

   # 3. 编译
   make geth
   ```

2. **设置调试环境**（45分钟）
   ```json
   // VSCode launch.json
   {
       "version": "0.2.0",
       "configurations": [
           {
               "name": "Debug Geth",
               "type": "go",
               "request": "launch",
               "mode": "debug",
               "program": "${workspaceFolder}/cmd/geth",
               "args": [
                   "--dev",
                   "--http",
                   "--http.api", "eth,net,web3,debug",
                   "--verbosity", "4"
               ]
           }
       ]
   }
   ```

3. **第一个源码阅读任务**（45分钟）
   ```go
   // 阅读 core/types/block.go
   // 任务：理解Block结构

   // 位置：core/types/block.go:60
   type Block struct {
       header       *Header        // 区块头
       uncles       []*Header      // 叔块
       transactions Transactions   // 交易列表

       // 缓存
       hash atomic.Value
       size atomic.Value

       // Td用于区块链总难度
       td *big.Int

       // 接收时间
       ReceivedAt   time.Time
       ReceivedFrom interface{}
   }

   // 阅读任务：
   // 1. 找到Header结构定义（同文件约第70行）
   // 2. 理解为什么需要缓存hash
   // 3. 找到Hash()方法的实现
   // 4. 理解ReceivedAt的作用
   ```

**作业**：
- [ ] 成功编译geth
- [ ] 配置好调试环境
- [ ] 阅读Block和Header结构
- [ ] 在源码中标注注释

**检查点**：
- [ ] 能启动geth --dev模式
- [ ] 能用VSCode调试geth
- [ ] 理解Block的基本结构

---

### Day 12（周二）：StateDB核心概念

**学习内容（3小时）**：

1. **StateDB架构**（75分钟）
   ```go
   // 阅读 core/state/statedb.go

   // StateDB管理整个世界状态
   type StateDB struct {
       db         Database              // 底层数据库
       prefetcher *triePrefetcher       // Trie预取器
       trie       Trie                  // 账户trie（世界状态）

       // 快照
       snaps         *snapshot.Tree     // 快照树
       snap          snapshot.Snapshot   // 当前快照

       // 缓存
       stateObjects      map[common.Address]*stateObject
       stateObjectsPending map[common.Address]struct{}
       stateObjectsDirty   map[common.Address]struct{}

       // 数据库错误
       dbErr error

       // 日志
       logs    map[common.Hash][]*types.Log
       logSize uint

       // 预编译合约
       precompiles map[common.Address]PrecompiledContract

       // Journal记录状态变更（用于回滚）
       journal        *journal
       validRevisions []revision
       nextRevisionId int

       // 并发控制
       lock sync.Mutex
   }

   // 重点理解：
   // 1. 两级缓存：stateObjects（内存） + trie（持久化）
   // 2. journal机制：可以回滚状态变更
   // 3. snapshot优化：加速状态访问
   ```

2. **stateObject详解**（60分钟）
   ```go
   // core/state/state_object.go

   type stateObject struct {
       address  common.Address
       addrHash common.Hash // address的keccak哈希
       data     types.StateAccount  // 账户数据
       db       *StateDB

       // 写缓存
       dirtyStorage Storage // 修改过的存储
       originStorage Storage // 原始存储（用于回滚）

       // 缓存标志
       dirtyCode bool
       fakeStorage bool

       // 合约代码
       code Code
   }

   // types.StateAccount定义在 core/types/state_account.go
   type StateAccount struct {
       Nonce    uint64
       Balance  *big.Int
       Root     common.Hash  // 存储trie根
       CodeHash []byte
   }

   // 关键方法分析
   func (s *stateObject) GetState(key common.Hash) common.Hash {
       // 1. 先查dirty缓存
       if value, dirty := s.dirtyStorage[key]; dirty {
           return value
       }

       // 2. 再查origin缓存
       if value, cached := s.originStorage[key]; cached {
           return value
       }

       // 3. 最后查trie
       value, err := s.getTrie().GetStorage(s.address, key.Bytes())
       if err != nil {
           s.db.setError(err)
           return common.Hash{}
       }

       // 4. 缓存结果
       s.originStorage[key] = value
       return value
   }

   func (s *stateObject) SetState(key, value common.Hash) {
       // 记录变更到journal（支持回滚）
       s.db.journal.append(storageChange{
           account:  &s.address,
           key:      key,
           prevalue: s.GetState(key),
       })

       // 更新dirty缓存
       s.dirtyStorage[key] = value
   }
   ```

3. **实践：追踪一笔转账**（45分钟）
   ```go
   // 任务：在源码中设置断点，追踪转账流程

   // 入口：core/state_processor.go:ApplyTransaction()
   func ApplyTransaction(...) (*types.Receipt, error) {
       // 断点1：这里
       msg, err := TransactionToMessage(tx, signer, header.BaseFee)

       // 断点2：状态转换
       result, err := ApplyMessage(evm, msg, gp)

       // 断点3：创建收据
       receipt := &types.Receipt{...}

       return receipt, err
   }

   // core/state_transition.go:TransitionDb()
   func (st *StateTransition) TransitionDb() (*ExecutionResult, error) {
       // 断点4：扣除Gas预付费
       st.state.SubBalance(sender, mgval)

       // 断点5：执行转账
       if contractCreation {
           ret, _, st.gasRemaining, vmerr = st.evm.Create(sender, st.data, st.gasRemaining, st.value)
       } else {
           st.state.SetNonce(sender.Address(), st.state.GetNonce(sender.Address())+1)
           ret, st.gasRemaining, vmerr = st.evm.Call(sender, st.to(), st.data, st.gasRemaining, st.value)
       }

       // 断点6：退还Gas
       st.refundGas(params.RefundQuotient)

       return &ExecutionResult{...}, nil
   }

   // 运行测试：
   // 1. 在geth console中发起转账
   // 2. 观察断点执行流程
   // 3. 查看stateDB的变化
   ```

**作业**：
- [ ] 完整阅读statedb.go（约1500行）
- [ ] 理解stateObject的缓存机制
- [ ] 使用调试器追踪一笔转账

**检查点**：
- [ ] 能画出StateDB的架构图
- [ ] 理解两级缓存的作用
- [ ] 知道journal如何支持回滚

**面试问题准备**：
- Q: 以太坊如何实现状态回滚？
- A: 通过journal机制记录每次状态变更，回滚时反向执行...

---

### Day 13（周三）：MPT数据结构（上）

**学习内容（3小时）**：

1. **Trie接口**（45分钟）
   ```go
   // trie/trie.go

   // Trie是Merkle Patricia Trie的实现
   type Trie struct {
       root  node              // 根节点
       owner common.Hash       // 拥有者（用于并发控制）

       // 缓存
       unhashed int            // 未哈希的节点数
       db       *Database      // 后端数据库

       // 追踪
       tracer *tracer
   }

   // 节点类型
   type node interface {
       cache() (hashNode, bool)
       encode(w rlp.EncoderBuffer)
   }

   // 4种节点类型
   type (
       // 全节点：17个子节点（0-f + value）
       fullNode struct {
           Children [17]node
           flags    nodeFlag
       }

       // 短节点：路径压缩
       shortNode struct {
           Key   []byte
           Val   node
           flags nodeFlag
       }

       // 哈希节点：引用其他节点
       hashNode []byte

       // 值节点：叶子节点
       valueNode []byte
   )

   // 画图理解节点类型：
   /*
   示例：插入 "apple", "app", "apply"

   1. 插入 "apple":
      root -> shortNode("apple") -> valueNode("A")

   2. 插入 "app":
      root -> shortNode("app") -> fullNode
                                   ├─ [empty] -> valueNode("B")
                                   └─ [l] -> shortNode("e") -> valueNode("A")

   3. 插入 "apply":
      root -> shortNode("app") -> fullNode
                                   ├─ [empty] -> valueNode("B")
                                   └─ [l] -> fullNode
                                              ├─ [e] -> valueNode("A")
                                              └─ [y] -> valueNode("C")
   */
   ```

2. **插入操作详解**（90分钟）
   ```go
   // trie/trie.go:TryUpdate()
   func (t *Trie) TryUpdate(key, value []byte) error {
       k := keybytesToHex(key)  // 转hex编码
       if len(value) != 0 {
           _, n, err := t.insert(t.root, nil, k, valueNode(value))
           if err != nil {
               return err
           }
           t.root = n
       } else {
           _, n, err := t.delete(t.root, nil, k)
           if err != nil {
               return err
           }
           t.root = n
       }
       return nil
   }

   // insert的核心逻辑
   func (t *Trie) insert(n node, prefix, key []byte, value node) (bool, node, error) {
       if len(key) == 0 {
           if v, ok := n.(valueNode); ok {
               return !bytes.Equal(v, value.(valueNode)), value, nil
           }
           return true, value, nil
       }

       switch n := n.(type) {
       case *shortNode:
           // 情况1：key完全匹配
           matchlen := prefixLen(key, n.Key)
           if matchlen == len(n.Key) {
               dirty, nn, err := t.insert(n.Val, append(prefix, key[:matchlen]...), key[matchlen:], value)
               if !dirty || err != nil {
                   return false, n, err
               }
               return true, &shortNode{n.Key, nn, nodeFlag{dirty: true}}, nil
           }

           // 情况2：部分匹配，需要分裂
           branch := &fullNode{flags: nodeFlag{dirty: true}}
           var err error

           // 分裂点的子节点
           _, branch.Children[n.Key[matchlen]], err = t.insert(nil, append(prefix, n.Key[:matchlen+1]...), n.Key[matchlen+1:], n.Val)
           if err != nil {
               return false, nil, err
           }

           // 新key的子节点
           _, branch.Children[key[matchlen]], err = t.insert(nil, append(prefix, key[:matchlen+1]...), key[matchlen+1:], value)
           if err != nil {
               return false, nil, err
           }

           // 如果有公共前缀，包装在shortNode中
           if matchlen == 0 {
               return true, branch, nil
           }

           return true, &shortNode{key[:matchlen], branch, nodeFlag{dirty: true}}, nil

       case *fullNode:
           // 插入到对应的子节点
           dirty, nn, err := t.insert(n.Children[key[0]], append(prefix, key[0]), key[1:], value)
           if !dirty || err != nil {
               return false, n, err
           }
           n.Children[key[0]] = nn
           n.flags.dirty = true
           return true, n, nil

       case nil:
           // 空节点，创建新的shortNode
           return true, &shortNode{key, value, nodeFlag{dirty: true}}, nil

       case hashNode:
           // 需要从数据库加载节点
           rn, err := t.resolveHash(n, prefix)
           if err != nil {
               return false, nil, err
           }
           dirty, nn, err := t.insert(rn, prefix, key, value)
           if !dirty || err != nil {
               return false, rn, err
           }
           return true, nn, nil

       default:
           panic(fmt.Sprintf("%T: invalid node: %v", n, n))
       }
   }
   ```

3. **手动模拟插入**（45分钟）
   ```go
   // 练习：在纸上画出插入过程

   // 初始：空trie
   root = nil

   // 插入 "do" -> "verb"
   root = shortNode{
       Key: hex("do"),
       Val: valueNode("verb"),
   }

   // 插入 "dog" -> "puppy"
   root = shortNode{
       Key: hex("do"),
       Val: fullNode{
           Children: [17]node{
               nil, nil, ...,
               hex('g'): shortNode{
                   Key: hex(""),
                   Val: valueNode("puppy"),
               },
               ...,
               valueNode("verb"),  // 第16个位置存原来的"do"
           },
       },
   }

   // 画出完整的树形结构
   // 理解路径压缩的作用
   ```

**作业**：
- [ ] 阅读trie.go的insert方法
- [ ] 在纸上模拟3-4次插入操作
- [ ] 理解shortNode和fullNode的转换

**检查点**：
- [ ] 能画出MPT的4种节点
- [ ] 理解路径压缩的原理
- [ ] 知道何时shortNode转fullNode

---

### Day 14（周四）：MPT数据结构（下）

**学习内容（2.5小时）**：

1. **查询操作**（60分钟）
   ```go
   // trie/trie.go:TryGet()
   func (t *Trie) TryGet(key []byte) ([]byte, error) {
       value, newroot, didResolve, err := t.tryGet(t.root, keybytesToHex(key), 0)
       if err == nil && didResolve {
           t.root = newroot
       }
       return value, err
   }

   func (t *Trie) tryGet(origNode node, key []byte, pos int) (value []byte, newnode node, didResolve bool, err error) {
       switch n := origNode.(type) {
       case nil:
           return nil, nil, false, nil

       case valueNode:
           return n, n, false, nil

       case *shortNode:
           if len(key)-pos < len(n.Key) || !bytes.Equal(n.Key, key[pos:pos+len(n.Key)]) {
               return nil, n, false, nil
           }
           value, newnode, didResolve, err = t.tryGet(n.Val, key, pos+len(n.Key))
           if err == nil && didResolve {
               n = n.copy()
               n.Val = newnode
           }
           return value, n, didResolve, err

       case *fullNode:
           value, newnode, didResolve, err = t.tryGet(n.Children[key[pos]], key, pos+1)
           if err == nil && didResolve {
               n = n.copy()
               n.Children[key[pos]] = newnode
           }
           return value, n, didResolve, err

       case hashNode:
           child, err := t.resolveHash(n, key[:pos])
           if err != nil {
               return nil, n, true, err
           }
           value, newnode, _, err := t.tryGet(child, key, pos)
           return value, newnode, true, err

       default:
           panic(fmt.Sprintf("%T: invalid node: %v", origNode, origNode))
       }
   }

   // 复杂度分析：
   // - 最坏情况：O(log₁₆ n) 其中n是键值对数量
   // - 实际情况：由于路径压缩，通常更快
   ```

2. **Commit提交机制**（60分钟）
   ```go
   // trie/trie.go:Commit()
   func (t *Trie) Commit(collectLeaf bool) (common.Hash, *trienode.NodeSet, error) {
       // 1. 哈希所有节点
       hasher := newHasher(collectLeaf)
       defer returnHasherToPool(hasher)

       // 2. 递归哈希
       root, nodes := hasher.hash(t.root, true)

       // 3. 更新root
       t.root = root

       return common.BytesToHash(root.(hashNode)), nodes, nil
   }

   // hasher.hash() 的核心逻辑
   func (h *hasher) hash(n node, force bool) (hashed node, cached node) {
       switch n := n.(type) {
       case *shortNode:
           // 递归哈希子节点
           collapsed, cached := h.hashShortNodeChildren(n)

           // 编码节点
           h.tmp.Reset()
           if err := rlp.Encode(&h.tmp, collapsed); err != nil {
               panic(err)
           }

           // 如果编码后<32字节，直接嵌入父节点
           if len(h.tmp) < 32 && !force {
               return collapsed, cached
           }

           // 否则，计算哈希
           return h.hashData(h.tmp), cached

       case *fullNode:
           // 递归哈希所有子节点
           collapsed, cached := h.hashFullNodeChildren(n)

           // 编码并哈希
           h.tmp.Reset()
           rlp.Encode(&h.tmp, collapsed)
           if len(h.tmp) < 32 && !force {
               return collapsed, cached
           }
           return h.hashData(h.tmp), cached

       default:
           return n, n
       }
   }

   // 关键优化：
   // 1. 小于32字节的节点直接嵌入（减少数据库查询）
   // 2. 使用对象池复用hasher（减少GC）
   // 3. RLP编码压缩数据
   ```

3. **Merkle Proof**（30分钟）
   ```go
   // trie/proof.go:Prove()
   func (t *Trie) Prove(key []byte, proofDb ethdb.KeyValueWriter) error {
       key = keybytesToHex(key)
       nodes := []node{}

       // 收集从根到叶子的路径
       tn := t.root
       for len(key) > 0 && tn != nil {
           switch n := tn.(type) {
           case *shortNode:
               if len(key) < len(n.Key) || !bytes.Equal(n.Key, key[:len(n.Key)]) {
                   return nil
               }
               tn = n.Val
               key = key[len(n.Key):]
               nodes = append(nodes, n)

           case *fullNode:
               tn = n.Children[key[0]]
               key = key[1:]
               nodes = append(nodes, n)

           case hashNode:
               var err error
               tn, err = t.resolveHash(n, nil)
               if err != nil {
                   return err
               }

           default:
               panic(fmt.Sprintf("%T: invalid node: %v", tn, tn))
           }
       }

       // 将节点写入proofDb
       hasher := newHasher(false)
       defer returnHasherToPool(hasher)

       for i, n := range nodes {
           n, _ = hasher.proofHash(n)
           if hash, ok := n.(hashNode); ok || i == 0 {
               enc := nodeToBytes(n)
               if !ok {
                   hash = hasher.hashData(enc)
               }
               proofDb.Put(hash, enc)
           }
       }

       return nil
   }

   // 验证Proof
   func VerifyProof(rootHash common.Hash, key []byte, proofDb ethdb.KeyValueReader) (value []byte, err error) {
       // 从proof重建路径并验证
       key = keybytesToHex(key)
       wantHash := rootHash

       for i := 0; ; i++ {
           buf, _ := proofDb.Get(wantHash[:])
           if buf == nil {
               return nil, fmt.Errorf("proof node %d (hash %064x) missing", i, wantHash)
           }

           n, err := decodeNode(wantHash[:], buf)
           if err != nil {
               return nil, err
           }

           // 继续沿着key走
           switch n := n.(type) {
           case *shortNode:
               if len(key) < len(n.Key) || !bytes.Equal(n.Key, key[:len(n.Key)]) {
                   return nil, nil
               }
               wantHash = n.Val.(hashNode)
               key = key[len(n.Key):]

           case *fullNode:
               wantHash = n.Children[key[0]].(hashNode)
               key = key[1:]

           case valueNode:
               return n, nil
           }
       }
   }
   ```

**作业**：
- [ ] 实现简单的MPT Proof生成
- [ ] 实现Proof验证
- [ ] 理解proof的大小（O(log n)）

**检查点**：
- [ ] 理解Commit的哈希过程
- [ ] 知道<32字节优化的作用
- [ ] 能生成和验证Merkle Proof

---

### Day 15（周五）：Transaction Pool

**学习内容（2.5小时）**：

1. **TxPool架构**（75分钟）
   ```go
   // core/txpool/txpool.go

   type TxPool struct {
       config       Config
       chainconfig  *params.ChainConfig
       chain        blockChain
       gasPrice     *big.Int
       txFeed       event.Feed
       scope        event.SubscriptionScope
       signer       types.Signer

       mu          sync.RWMutex
       currentState  *state.StateDB    // 当前状态
       pendingNonces *noncer           // Pending nonce
       currentMaxGas uint64             // 当前区块gas限制

       locals  *accountSet             // 本地账户
       journal *journal                // 持久化journal

       pending map[common.Address]*list  // 可执行交易
       queue   map[common.Address]*list  // 未来交易（nonce不连续）

       all     *lookup                  // 所有交易的索引
       priced  *pricedList              // 按价格排序的交易

       wg sync.WaitGroup
   }

   // 重要概念：
   // pending：nonce连续，可以立即执行的交易
   // queue：nonce不连续，等待前面的交易

   // 示例：
   // 当前账户nonce: 5
   // 交易A nonce=5 → pending
   // 交易B nonce=6 → pending（如果A在）
   // 交易C nonce=8 → queue（等待nonce=7）
   ```

2. **添加交易流程**（60分钟）
   ```go
   // core/txpool/txpool.go:add()
   func (pool *TxPool) add(tx *types.Transaction, local bool) (replaced bool, err error) {
       // 1. 验证交易
       if err := pool.validateTx(tx, local); err != nil {
           return false, err
       }

       // 2. 检查是否已存在
       hash := tx.Hash()
       if pool.all.Get(hash) != nil {
           return false, ErrAlreadyKnown
       }

       // 3. 检查是否可以替换
       from, _ := types.Sender(pool.signer, tx)
       if list := pool.pending[from]; list != nil && list.Overlaps(tx) {
           // 检查Gas价格是否更高
           inserted, old := list.Add(tx, pool.config.PriceBump)
           if !inserted {
               return false, ErrReplaceUnderpriced
           }
           if old != nil {
               pool.all.Remove(old.Hash())
               pool.priced.Removed(1)
           }
           pool.all.Add(tx, local)
           pool.priced.Put(tx, local)
           return old != nil, nil
       }

       // 4. 新交易，检查nonce
       if pool.pendingNonces.get(from) == tx.Nonce() {
           // Nonce连续，加入pending
           pool.enqueueTx(hash, tx, local, true)
       } else {
           // Nonce不连续，加入queue
           pool.enqueueTx(hash, tx, local, false)
       }

       // 5. 尝试提升queue中的交易
       pool.promoteExecutables([]common.Address{from})

       return false, nil
   }

   // validateTx验证交易
   func (pool *TxPool) validateTx(tx *types.Transaction, local bool) error {
       // 1. 检查交易大小
       if tx.Size() > txMaxSize {
           return ErrOversizedData
       }

       // 2. 检查交易金额
       if tx.Value().Sign() < 0 {
           return ErrNegativeValue
       }

       // 3. 检查Gas
       if tx.Gas() < params.TxGas {
           return ErrIntrinsicGas
       }

       // 4. 检查Gas价格
       if !local && tx.GasPrice().Cmp(pool.gasPrice) < 0 {
           return ErrUnderpriced
       }

       // 5. 检查发送者余额
       from, err := types.Sender(pool.signer, tx)
       if err != nil {
           return ErrInvalidSender
       }

       if pool.currentState.GetBalance(from).Cmp(tx.Cost()) < 0 {
           return ErrInsufficientFunds
       }

       // 6. 检查nonce
       if pool.currentState.GetNonce(from) > tx.Nonce() {
           return ErrNonceTooLow
       }

       return nil
   }
   ```

3. **交易排序与选择**（45分钟）
   ```go
   // core/txpool/txpool.go:Pending()
   func (pool *TxPool) Pending(enforceTips bool) map[common.Address][]*types.Transaction {
       pool.mu.Lock()
       defer pool.mu.Unlock()

       pending := make(map[common.Address][]*types.Transaction)
       for addr, list := range pool.pending {
           txs := list.Flatten()

           // 如果强制最低tip，过滤
           if enforceTips {
               filtered := []*types.Transaction{}
               for _, tx := range txs {
                   if tx.EffectiveGasTipValue(pool.priced.urgent.baseFee) >= pool.config.PriceLimit {
                       filtered = append(filtered, tx)
                   }
               }
               txs = filtered
           }

           if len(txs) > 0 {
               pending[addr] = txs
           }
       }
       return pending
   }

   // 矿工选择交易（按Gas价格）
   func selectTransactions(pending map[common.Address][]*types.Transaction, gasLimit uint64) []*types.Transaction {
       // 1. 所有pending交易
       var txs []*types.Transaction
       for _, list := range pending {
           txs = append(txs, list...)
       }

       // 2. 按Gas价格排序（从高到低）
       sort.Slice(txs, func(i, j int) bool {
           return txs[i].GasPrice().Cmp(txs[j].GasPrice()) > 0
       })

       // 3. 贪心选择
       var selected []*types.Transaction
       gasUsed := uint64(0)

       for _, tx := range txs {
           if gasUsed+tx.Gas() > gasLimit {
               continue
           }
           selected = append(selected, tx)
           gasUsed += tx.Gas()
       }

       return selected
   }
   ```

**作业**：
- [ ] 阅读txpool.go的add方法
- [ ] 理解pending和queue的区别
- [ ] 实现简单的交易排序算法

**检查点**：
- [ ] 能画出TxPool的架构图
- [ ] 理解nonce检查的重要性
- [ ] 知道为什么需要Gas价格排序

---

### 周末实践（Week 3）：简化版StateDB实现

**项目：MiniStateDB**（6小时）

**目标**：实现一个包含StateDB + MPT的简化版本

```go
// 项目结构
ministatedb/
├── statedb.go      # StateDB实现
├── state_object.go # 账户对象
├── trie.go         # 简化的MPT
├── database.go     # 内存数据库
└── main_test.go    # 测试

// 功能清单
// [ ] StateDB基本操作（GetBalance/SetBalance）
// [ ] stateObject缓存机制
// [ ] 简化的MPT（支持插入/查询）
// [ ] Commit提交机制
// [ ] 状态回滚（journal）
```

**实现要点**：

1. **StateDB核心**（2小时）
   ```go
   type MiniStateDB struct {
       db           *Database
       trie         *Trie
       stateObjects map[common.Address]*stateObject
       dirty        map[common.Address]struct{}
       journal      *journal
   }

   func (s *MiniStateDB) GetBalance(addr common.Address) *big.Int {
       obj := s.getStateObject(addr)
       if obj != nil {
           return obj.Balance
       }
       return big.NewInt(0)
   }

   func (s *MiniStateDB) SetBalance(addr common.Address, amount *big.Int) {
       obj := s.getOrCreateStateObject(addr)
       obj.setBalance(amount)
   }

   func (s *MiniStateDB) Commit() (common.Hash, error) {
       // 1. 提交所有stateObject
       for addr := range s.dirty {
           obj := s.stateObjects[addr]
           data := obj.encode()
           s.trie.Update(addr.Bytes(), data)
       }

       // 2. 提交trie
       root := s.trie.Hash()

       // 3. 清空缓存
       s.dirty = make(map[common.Address]struct{})

       return root, nil
   }
   ```

2. **简化的MPT**（2.5小时）
   ```go
   type Trie struct {
       root node
       db   *Database
   }

   func (t *Trie) Update(key, value []byte) {
       // 实现简化的插入逻辑
       // 只支持shortNode和valueNode
   }

   func (t *Trie) Get(key []byte) []byte {
       // 实现查询
   }

   func (t *Trie) Hash() common.Hash {
       // 计算根哈希
   }
   ```

3. **测试**（1.5小时）
   ```go
   func TestStateDB(t *testing.T) {
       db := NewMiniStateDB()

       addr1 := common.HexToAddress("0x01")
       addr2 := common.HexToAddress("0x02")

       // 测试1：设置余额
       db.SetBalance(addr1, big.NewInt(100))
       assert.Equal(t, big.NewInt(100), db.GetBalance(addr1))

       // 测试2：转账
       db.SetBalance(addr1, big.NewInt(70))
       db.SetBalance(addr2, big.NewInt(30))

       // 测试3：提交
       root1, _ := db.Commit()

       // 测试4：修改后再提交
       db.SetBalance(addr1, big.NewInt(50))
       root2, _ := db.Commit()

       // root1 != root2
       assert.NotEqual(t, root1, root2)
   }
   ```

**检查点**：
- [ ] StateDB能正确管理账户
- [ ] MPT能计算正确的根哈希
- [ ] 相同状态产生相同根哈希

---

## Week 4: 区块管理 + EVM深入

### Day 16（周一）：BlockChain管理

**学习内容（2.5小时）**：

1. **BlockChain结构**（60分钟）
   ```go
   // core/blockchain.go

   type BlockChain struct {
       chainConfig *params.ChainConfig
       db          ethdb.Database
       snaps       *snapshot.Tree

       // 区块链状态
       genesisBlock *types.Block
       currentBlock atomic.Pointer[types.Header]

       stateCache   state.Database
       validator    Validator
       prefetcher   Prefetcher
       processor    Processor

       // 缓存
       headerCache   *lru.Cache[common.Hash, *types.Header]
       blockCache    *lru.Cache[common.Hash, *types.Block]
       txLookupCache *lru.Cache[common.Hash, *rawdb.LegacyTxLookupEntry]

       // 侧链缓存
       futureBlocks *lru.Cache[common.Hash, *types.Block]

       quit    chan struct{}
       wg      sync.WaitGroup
       running atomic.Bool

       // 链重组
       chainmu      *syncx.ClosableMutex
       chainHeadFeed event.Feed

       // 同步相关
       scope event.SubscriptionScope
   }
   ```

2. **插入区块流程**（75分钟）
   ```go
   // core/blockchain.go:InsertChain()
   func (bc *BlockChain) InsertChain(chain types.Blocks) (int, error) {
       // 1. 验证区块
       for i, block := range chain {
           if err := bc.validator.ValidateBody(block); err != nil {
               return i, err
           }
       }

       // 2. 执行区块
       for i, block := range chain {
           receipts, logs, usedGas, err := bc.processor.Process(
               block,
               bc.stateCache,
               bc.vmConfig,
           )
           if err != nil {
               return i, err
           }

           // 3. 验证状态
           if err := bc.validator.ValidateState(
               block,
               bc.stateCache,
               receipts,
               usedGas,
           ); err != nil {
               return i, err
           }

           // 4. 写入数据库
           rawdb.WriteBlock(bc.db, block)
           rawdb.WriteReceipts(bc.db, block.Hash(), receipts)

           // 5. 更新链头
           bc.writeHeadBlock(block)
       }

       return len(chain), nil
   }

   // 处理区块
   func (p *StateProcessor) Process(
       block *types.Block,
       statedb *state.StateDB,
       cfg vm.Config,
   ) ([]*types.Receipt, []*types.Log, uint64, error) {
       var (
           receipts    types.Receipts
           usedGas     = new(uint64)
           header      = block.Header()
           blockHash   = block.Hash()
           blockNumber = block.Number()
           allLogs     []*types.Log
           gp          = new(GasPool).AddGas(block.GasLimit())
       )

       // 处理每笔交易
       for i, tx := range block.Transactions() {
           msg, err := TransactionToMessage(tx, types.MakeSigner(p.config, header.Number, header.Time), header.BaseFee)
           if err != nil {
               return nil, nil, 0, err
           }

           statedb.SetTxContext(tx.Hash(), i)

           receipt, err := applyTransaction(msg, p.config, gp, statedb, blockNumber, blockHash, tx, usedGas, vmConfig)
           if err != nil {
               return nil, nil, 0, err
           }

           receipts = append(receipts, receipt)
           allLogs = append(allLogs, receipt.Logs...)
       }

       // 区块奖励
       accumulateRewards(p.config, statedb, header, block.Uncles())

       return receipts, allLogs, *usedGas, nil
   }
   ```

3. **链重组（Reorg）**（45分钟）
   ```go
   // 当发现更长的链时触发重组
   func (bc *BlockChain) reorg(oldBlock, newBlock *types.Block) error {
       var (
           newChain    types.Blocks
           oldChain    types.Blocks
           commonBlock *types.Block
       )

       // 1. 找到公共祖先
       if oldBlock.NumberU64() > newBlock.NumberU64() {
           for ; oldBlock != nil && oldBlock.NumberU64() != newBlock.NumberU64(); oldBlock = bc.GetBlock(oldBlock.ParentHash(), oldBlock.NumberU64()-1) {
               oldChain = append(oldChain, oldBlock)
           }
       } else {
           for ; newBlock != nil && newBlock.NumberU64() != oldBlock.NumberU64(); newBlock = bc.GetBlock(newBlock.ParentHash(), newBlock.NumberU64()-1) {
               newChain = append(newChain, newBlock)
           }
       }

       // 2. 同时向前，找到分叉点
       for {
           if oldBlock.Hash() == newBlock.Hash() {
               commonBlock = oldBlock
               break
           }
           oldChain = append(oldChain, oldBlock)
           newChain = append(newChain, newBlock)

           oldBlock = bc.GetBlock(oldBlock.ParentHash(), oldBlock.NumberU64()-1)
           newBlock = bc.GetBlock(newBlock.ParentHash(), newBlock.NumberU64()-1)
       }

       // 3. 回滚旧链
       for _, block := range oldChain {
           bc.rollbackBlock(block)
       }

       // 4. 应用新链
       for i := len(newChain) - 1; i >= 0; i-- {
           bc.insertBlock(newChain[i])
       }

       log.Info("Chain reorg", "old", len(oldChain), "new", len(newChain))
       return nil
   }
   ```

**作业**：
- [ ] 阅读blockchain.go的InsertChain方法
- [ ] 理解链重组的触发条件
- [ ] 画出reorg的流程图

**检查点**：
- [ ] 理解区块插入的验证流程
- [ ] 知道什么情况下会触发reorg
- [ ] 理解为什么需要公共祖先

**面试问题准备**：
- Q: 以太坊如何处理分叉？
- A: 通过链重组（reorg），选择总难度最高的链...

---

### **Day 17：EVM执行引擎深度分析**

**学习目标**：
- 理解EVM指令执行流程
- 掌握Stack、Memory、Storage的操作
- 分析Gas消耗计算

**学习内容**：

1. **EVM执行架构**：
   ```go
   // core/vm/evm.go
   type EVM struct {
       Context      BlockContext
       TxContext    TxContext
       StateDB      StateDB
       depth        int
       chainConfig  *params.ChainConfig
       chainRules   params.Rules
       interpreter  *EVMInterpreter
   }

   func (evm *EVM) Call(caller ContractRef, addr common.Address,
                        input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
       // 1. 检查调用深度
       if evm.depth > int(params.CallCreateDepth) {
           return nil, gas, ErrDepth
       }

       // 2. 检查余额
       if !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value) {
           return nil, gas, ErrInsufficientBalance
       }

       // 3. 创建快照（用于回滚）
       snapshot := evm.StateDB.Snapshot()

       // 4. 转账
       evm.Context.Transfer(evm.StateDB, caller.Address(), addr, value)

       // 5. 执行合约代码
       contract := NewContract(caller, AccountRef(addr), value, gas)
       contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))

       ret, err = evm.interpreter.Run(contract, input, false)

       // 6. 处理错误回滚
       if err != nil {
           evm.StateDB.RevertToSnapshot(snapshot)
           if err != ErrExecutionReverted {
               gas = 0
           }
       }

       return ret, contract.Gas, err
   }
   ```

2. **解释器执行循环**：
   ```go
   // core/vm/interpreter.go
   func (in *EVMInterpreter) Run(contract *Contract, input []byte, readOnly bool) (ret []byte, err error) {
       // 初始化
       var (
           op      OpCode
           mem     = NewMemory()
           stack   = newstack()
           pc      = uint64(0)
           cost    uint64
           pcCopy  uint64
           gasCopy uint64
       )

       contract.Input = input

       // 主执行循环
       for {
           // 1. 获取操作码
           op = contract.GetOp(pc)

           // 2. 获取操作函数
           operation := in.cfg.JumpTable[op]

           // 3. 检查栈深度
           if sLen := stack.len(); sLen < operation.minStack {
               return nil, &ErrStackUnderflow{stackLen: sLen, required: operation.minStack}
           } else if sLen > operation.maxStack {
               return nil, &ErrStackOverflow{stackLen: sLen, limit: operation.maxStack}
           }

           // 4. 计算Gas消耗
           cost, err = operation.gasCost(in.evm, contract, stack, mem, memorySize)
           if err != nil || !contract.UseGas(cost) {
               return nil, ErrOutOfGas
           }

           // 5. 扩展内存（如果需要）
           if memorySize > 0 {
               mem.Resize(memorySize)
           }

           // 6. 执行操作
           res, err := operation.execute(&pc, in, contract, mem, stack)
           if err != nil {
               return nil, err
           }

           // 7. 处理返回
           if operation.returns {
               return res, nil
           }

           pc++
       }
   }
   ```

3. **典型指令实现示例**：
   ```go
   // core/vm/instructions.go

   // ADD指令
   func opAdd(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
       x, y := scope.Stack.pop(), scope.Stack.peek()
       y.Add(&x, y)
       return nil, nil
   }

   // MUL指令
   func opMul(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
       x, y := scope.Stack.pop(), scope.Stack.peek()
       y.Mul(&x, y)
       return nil, nil
   }

   // MSTORE指令（写入内存）
   func opMstore(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
       mStart, val := scope.Stack.pop(), scope.Stack.pop()
       scope.Memory.Set32(mStart.Uint64(), &val)
       return nil, nil
   }

   // SSTORE指令（写入存储）
   func opSstore(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
       loc := scope.Stack.pop()
       val := scope.Stack.pop()
       interpreter.evm.StateDB.SetState(scope.Contract.Address(), loc.Bytes32(), val.Bytes32())
       return nil, nil
   }

   // SLOAD指令（读取存储）
   func opSload(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
       loc := scope.Stack.peek()
       val := interpreter.evm.StateDB.GetState(scope.Contract.Address(), loc.Bytes32())
       loc.SetBytes(val.Bytes())
       return nil, nil
   }
   ```

4. **Gas计算示例**：
   ```go
   // SSTORE的Gas计算（EIP-2200）
   func gasSStore(evm *EVM, contract *Contract, stack *Stack, mem *Memory, memorySize uint64) (uint64, error) {
       var (
           y, x    = stack.Back(1), stack.Back(0)
           current = evm.StateDB.GetState(contract.Address(), x.Bytes32())
       )

       // 判断存储槽的状态
       value := common.Hash(y.Bytes32())

       if current == value { // 重新写入相同的值
           return params.SloadGasEIP2200, nil
       }

       original := evm.StateDB.GetCommittedState(contract.Address(), x.Bytes32())
       if original == current {
           if original == (common.Hash{}) { // 从零值写入非零值
               return params.SstoreSetGasEIP2200, nil
           }
           return params.SstoreResetGasEIP2200, nil
       }

       return params.SloadGasEIP2200, nil
   }
   ```

**实践任务**：
写一个简单的字节码解释器：
```go
package main

import (
    "fmt"
    "math/big"
)

type SimpleEVM struct {
    stack   []*big.Int
    memory  []byte
    storage map[string]*big.Int
    pc      int
    code    []byte
    gas     uint64
}

func NewSimpleEVM(code []byte, gas uint64) *SimpleEVM {
    return &SimpleEVM{
        stack:   make([]*big.Int, 0),
        memory:  make([]byte, 0),
        storage: make(map[string]*big.Int),
        code:    code,
        gas:     gas,
    }
}

func (evm *SimpleEVM) Run() error {
    for evm.pc < len(evm.code) {
        opcode := evm.code[evm.pc]

        switch opcode {
        case 0x01: // ADD
            if len(evm.stack) < 2 {
                return fmt.Errorf("stack underflow")
            }
            a := evm.stack[len(evm.stack)-1]
            b := evm.stack[len(evm.stack)-2]
            evm.stack = evm.stack[:len(evm.stack)-2]
            evm.stack = append(evm.stack, new(big.Int).Add(a, b))
            evm.gas -= 3

        case 0x60: // PUSH1
            evm.pc++
            val := new(big.Int).SetBytes([]byte{evm.code[evm.pc]})
            evm.stack = append(evm.stack, val)
            evm.gas -= 3

        case 0x00: // STOP
            return nil
        }

        evm.pc++

        if evm.gas <= 0 {
            return fmt.Errorf("out of gas")
        }
    }
    return nil
}
```

**作业**：
- [ ] 阅读core/vm/evm.go的Call方法
- [ ] 理解interpreter.go的执行循环
- [ ] 分析5个常见指令的实现
- [ ] 实现ADD、MUL、PUSH1、POP指令

**检查点**：
- [ ] 理解EVM的执行流程
- [ ] 知道Stack、Memory、Storage的区别
- [ ] 理解Gas计算原理
- [ ] 能解释SSTORE为什么Gas消耗高

**面试问题准备**：
- Q: EVM执行合约调用的完整流程是什么？
- A: 1) 检查调用深度 2) 检查余额 3) 创建状态快照 4) 转账 5) 加载合约代码 6) 解释器执行 7) 错误时回滚

---

### **Day 18：预编译合约与特殊指令**

**学习目标**：
- 理解预编译合约的作用
- 掌握ecrecover、sha256等预编译实现
- 分析CREATE、CREATE2指令

**学习内容**：

1. **预编译合约架构**：
   ```go
   // core/vm/contracts.go

   // 预编译合约接口
   type PrecompiledContract interface {
       RequiredGas(input []byte) uint64  // 计算Gas消耗
       Run(input []byte) ([]byte, error) // 执行逻辑
   }

   // 预编译合约映射表
   var PrecompiledContractsBerlin = map[common.Address]PrecompiledContract{
       common.BytesToAddress([]byte{1}): &ecrecover{},      // 0x01
       common.BytesToAddress([]byte{2}): &sha256hash{},     // 0x02
       common.BytesToAddress([]byte{3}): &ripemd160hash{},  // 0x03
       common.BytesToAddress([]byte{4}): &dataCopy{},       // 0x04
       common.BytesToAddress([]byte{5}): &bigModExp{},      // 0x05
       common.BytesToAddress([]byte{6}): &bn256AddIstanbul{},    // 0x06
       common.BytesToAddress([]byte{7}): &bn256ScalarMulIstanbul{}, // 0x07
       common.BytesToAddress([]byte{8}): &bn256PairingIstanbul{},   // 0x08
       common.BytesToAddress([]byte{9}): &blake2F{},        // 0x09
   }
   ```

2. **ecrecover实现**（最常用）：
   ```go
   // core/vm/contracts.go
   type ecrecover struct{}

   func (c *ecrecover) RequiredGas(input []byte) uint64 {
       return params.EcrecoverGas // 3000 gas
   }

   func (c *ecrecover) Run(input []byte) ([]byte, error) {
       const ecRecoverInputLength = 128

       // 1. 填充输入到128字节
       input = common.RightPadBytes(input, ecRecoverInputLength)

       // 2. 解析参数
       // hash: input[0:32]
       // v: input[32:64]
       // r: input[64:96]
       // s: input[96:128]

       r := new(big.Int).SetBytes(input[64:96])
       s := new(big.Int).SetBytes(input[96:128])
       v := input[63] - 27

       // 3. 验证签名参数
       if !allZero(input[32:63]) || !crypto.ValidateSignatureValues(v, r, s, false) {
           return nil, nil
       }

       // 4. 恢复公钥
       pubKey, err := crypto.Ecrecover(input[:32], append(input[64:128], v))
       if err != nil {
           return nil, nil
       }

       // 5. 返回地址（公钥哈希的后20字节）
       return common.LeftPadBytes(crypto.Keccak256(pubKey[1:])[12:], 32), nil
   }
   ```

3. **CREATE2指令实现**：
   ```go
   // core/vm/instructions.go
   func opCreate2(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
       var (
           endowment    = scope.Stack.pop()
           offset, size = scope.Stack.pop(), scope.Stack.pop()
           salt         = scope.Stack.pop()
           input        = scope.Memory.GetCopy(int64(offset.Uint64()), int64(size.Uint64()))
           gas          = scope.Contract.Gas
       )

       // 1. 计算Gas消耗
       if interpreter.evm.chainRules.IsEIP150 {
           gas -= gas / 64
       }
       scope.Contract.UseGas(gas)

       // 2. 计算新合约地址
       // address = keccak256(0xff ++ sender ++ salt ++ keccak256(init_code))
       codeHash := crypto.Keccak256Hash(input)
       address := crypto.CreateAddress2(scope.Contract.Address(), salt.Bytes32(), codeHash.Bytes())

       // 3. 创建合约
       res, addr, returnGas, suberr := interpreter.evm.Create2(scope.Contract, input, gas, &endowment, address)

       // 4. 处理返回
       if suberr != nil {
           scope.Stack.push(new(uint256.Int))
       } else {
           scope.Stack.push(new(uint256.Int).SetBytes(addr.Bytes()))
       }
       scope.Contract.Gas += returnGas

       return res, nil
   }

   // crypto/crypto.go
   func CreateAddress2(b common.Address, salt [32]byte, inithash []byte) common.Address {
       return common.BytesToAddress(Keccak256([]byte{0xff}, b.Bytes(), salt[:], inithash)[12:])
   }
   ```

4. **DELEGATECALL实现**：
   ```go
   func opDelegateCall(pc *uint64, interpreter *EVMInterpreter, scope *ScopeContext) ([]byte, error) {
       var (
           gas      = scope.Stack.pop()
           toAddr   = common.Address(scope.Stack.pop().Bytes20())
           inOffset = scope.Stack.pop()
           inSize   = scope.Stack.pop()
           retOffset= scope.Stack.pop()
           retSize  = scope.Stack.pop()
       )

       // 获取输入数据
       input := scope.Memory.GetPtr(int64(inOffset.Uint64()), int64(inSize.Uint64()))

       // 关键：保持原始调用者和value
       ret, returnGas, err := interpreter.evm.DelegateCall(
           scope.Contract,
           toAddr,
           input,
           gas.Uint64(),
       )

       // 处理返回数据
       if err != nil {
           scope.Stack.push(new(uint256.Int))
       } else {
           scope.Stack.push(new(uint256.Int).SetUint64(1))
           scope.Memory.Set(retOffset.Uint64(), retSize.Uint64(), ret)
       }

       scope.Contract.Gas += returnGas
       return ret, nil
   }
   ```

**实践任务**：
实现一个简单的预编译合约：
```go
// 简单的哈希预编译
type simpleHash struct{}

func (c *simpleHash) RequiredGas(input []byte) uint64 {
    return uint64(len(input)/32)*10 + 60
}

func (c *simpleHash) Run(input []byte) ([]byte, error) {
    return crypto.Keccak256(input), nil
}

// 测试CREATE2地址计算
func TestCreate2Address() {
    sender := common.HexToAddress("0x1234567890123456789012345678901234567890")
    salt := [32]byte{1, 2, 3}
    initCode := []byte{0x60, 0x80, 0x60, 0x40} // PUSH1 0x80 PUSH1 0x40

    codeHash := crypto.Keccak256(initCode)
    address := crypto.CreateAddress2(sender, salt, codeHash)

    fmt.Printf("CREATE2 Address: %s\n", address.Hex())
}
```

**作业**：
- [ ] 阅读所有预编译合约的实现
- [ ] 理解ecrecover的签名恢复原理
- [ ] 对比CREATE和CREATE2的区别
- [ ] 分析DELEGATECALL的安全风险

**检查点**：
- [ ] 知道9个预编译合约的地址和功能
- [ ] 理解CREATE2的地址计算公式
- [ ] 理解DELEGATECALL与CALL的区别
- [ ] 能解释为什么delegatecall有安全风险

**面试问题准备**：
- Q: CREATE和CREATE2有什么区别？
- A: CREATE地址由sender和nonce决定，CREATE2由sender、salt和init_code哈希决定，CREATE2可以预测地址且不依赖nonce

---

### **Day 19：Gas优化技巧**

**学习目标**：
- 掌握Gas消耗计算规则
- 学习常见的Gas优化技巧
- 分析真实合约的Gas优化案例

**学习内容**：

1. **Gas消耗表**：
   ```go
   // params/protocol_params.go
   const (
       GasQuickStep   uint64 = 2       // ADD, SUB, MUL等
       GasFastestStep uint64 = 3       // PUSH, DUP, SWAP等
       GasFastStep    uint64 = 5       // LT, GT, EQ等
       GasMidStep     uint64 = 8       // ADDMOD, MULMOD等
       GasSlowStep    uint64 = 10      // SIGNEXTEND等
       GasExtStep     uint64 = 20      // SHA3等

       SstoreSetGasEIP2200        uint64 = 20000  // 从零写非零
       SstoreResetGasEIP2200      uint64 = 5000   // 改变非零值
       SloadGasEIP2200            uint64 = 800    // 读取存储

       CallGasEIP150              uint64 = 700    // CALL基础消耗
       CallValueTransferGas       uint64 = 9000   // 转账额外消耗
       CallNewAccountGas          uint64 = 25000  // 创建新账户

       LogGas                     uint64 = 375    // LOG基础
       LogTopicGas                uint64 = 375    // 每个topic
       LogDataGas                 uint64 = 8      // 每字节data

       MemoryGas                  uint64 = 3      // 每32字节内存
   )
   ```

2. **Storage优化技巧**：
   ```solidity
   // ❌ 差：多次SSTORE
   contract BadStorage {
       uint256 public a;
       uint256 public b;
       uint256 public c;

       function update(uint256 _a, uint256 _b, uint256 _c) external {
           a = _a;  // 20000 gas (首次) 或 5000 gas
           b = _b;  // 20000 gas (首次) 或 5000 gas
           c = _c;  // 20000 gas (首次) 或 5000 gas
       }
   }

   // ✅ 好：打包存储
   contract GoodStorage {
       struct Data {
           uint64 a;
           uint64 b;
           uint64 c;
           uint64 d;
       }
       Data public data;  // 打包到一个slot

       function update(uint64 _a, uint64 _b, uint64 _c, uint64 _d) external {
           data = Data(_a, _b, _c, _d);  // 只需一次SSTORE！
       }
   }

   // ✅ 好：使用mapping而不是数组
   contract OptimizedMapping {
       mapping(uint256 => uint256) public balances;  // Gas友好

       // 而不是:
       // uint256[] public balances;  // 需要边界检查，Gas高
   }
   ```

3. **Memory vs Calldata优化**：
   ```solidity
   // ❌ 差：使用memory（需要复制）
   function processBad(uint256[] memory data) external pure returns (uint256) {
       uint256 sum = 0;
       for (uint256 i = 0; i < data.length; i++) {
           sum += data[i];
       }
       return sum;
   }

   // ✅ 好：使用calldata（只读，无复制）
   function processGood(uint256[] calldata data) external pure returns (uint256) {
       uint256 sum = 0;
       for (uint256 i = 0; i < data.length; i++) {
           sum += data[i];
       }
       return sum;
   }

   // ✅ 好：缓存storage到memory
   contract CachingExample {
       uint256[] public largeArray;

       function sumBad() external view returns (uint256) {
           uint256 sum = 0;
           for (uint256 i = 0; i < largeArray.length; i++) {
               sum += largeArray[i];  // 每次SLOAD = 800 gas
           }
           return sum;
       }

       function sumGood() external view returns (uint256) {
           uint256[] memory cached = largeArray;  // 一次性加载
           uint256 sum = 0;
           for (uint256 i = 0; i < cached.length; i++) {
               sum += cached[i];  // 从memory读，便宜
           }
           return sum;
       }
   }
   ```

4. **循环优化**：
   ```solidity
   // ❌ 差：未优化的循环
   function loopBad(uint256[] memory data) public pure returns (uint256) {
       uint256 sum = 0;
       for (uint256 i = 0; i < data.length; i++) {  // 每次读.length
           sum += data[i];
       }
       return sum;
   }

   // ✅ 好：缓存length
   function loopGood(uint256[] memory data) public pure returns (uint256) {
       uint256 sum = 0;
       uint256 len = data.length;  // 缓存
       for (uint256 i = 0; i < len; ) {
           sum += data[i];
           unchecked { ++i; }  // 避免overflow检查（Solidity 0.8+）
       }
       return sum;
   }

   // ✅ 最佳：使用assembly
   function loopBest(uint256[] memory data) public pure returns (uint256 sum) {
       assembly {
           let len := mload(data)
           let dataPtr := add(data, 0x20)
           for { let i := 0 } lt(i, len) { i := add(i, 1) } {
               sum := add(sum, mload(add(dataPtr, mul(i, 0x20))))
           }
       }
   }
   ```

5. **事件优化**：
   ```solidity
   // ✅ indexed参数可以过滤，但消耗更多Gas
   event Transfer(
       address indexed from,     // 可以过滤，+375 gas
       address indexed to,       // 可以过滤，+375 gas
       uint256 value             // 不能过滤，但便宜
   );

   // 权衡：最多3个indexed参数
   event Swap(
       address indexed user,
       address indexed tokenIn,
       address indexed tokenOut,
       uint256 amountIn,   // 如果也indexed，会超过3个限制
       uint256 amountOut
   );
   ```

6. **实际Gas对比测试**：
   ```go
   // 测试代码
   func TestGasOptimization(t *testing.T) {
       // 创建测试环境
       db := rawdb.NewMemoryDatabase()
       state, _ := state.New(common.Hash{}, state.NewDatabase(db), nil)

       // 测试SSTORE
       addr := common.HexToAddress("0x1")

       // 情况1：零 -> 非零
       startGas1 := uint64(1000000)
       state.SetState(addr, common.Hash{}, common.HexToHash("0x1"))
       gas1 := startGas1 - state.GetGas()
       fmt.Printf("零->非零 SSTORE: %d gas\n", gas1)  // ~20000

       // 情况2：非零 -> 非零
       startGas2 := uint64(1000000)
       state.SetState(addr, common.Hash{}, common.HexToHash("0x2"))
       gas2 := startGas2 - state.GetGas()
       fmt.Printf("非零->非零 SSTORE: %d gas\n", gas2)  // ~5000

       // 情况3：非零 -> 零（有退款）
       startGas3 := uint64(1000000)
       state.SetState(addr, common.Hash{}, common.Hash{})
       gas3 := startGas3 - state.GetGas()
       refund := state.GetRefund()
       fmt.Printf("非零->零 SSTORE: %d gas (退款: %d)\n", gas3, refund)
   }
   ```

**实践任务**：
分析Uniswap V2的Gas优化技巧：
```solidity
// UniswapV2Pair.sol的优化示例
function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external {
    // ✅ 使用局部变量缓存storage
    uint112 _reserve0 = reserve0;  // gas savings
    uint112 _reserve1 = reserve1;

    // ✅ 使用require而不是多个if
    require(amount0Out > 0 || amount1Out > 0, 'UniswapV2: INSUFFICIENT_OUTPUT_AMOUNT');
    require(amount0Out < _reserve0 && amount1Out < _reserve1, 'UniswapV2: INSUFFICIENT_LIQUIDITY');

    // ... 其他逻辑 ...

    // ✅ 最后才更新storage
    reserve0 = uint112(balance0);
    reserve1 = uint112(balance1);
}
```

**作业**：
- [ ] 记住常见操作的Gas消耗
- [ ] 对比memory和calldata的Gas差异
- [ ] 实现一个Gas优化前后的对比测试
- [ ] 分析一个真实DeFi合约的Gas优化

**检查点**：
- [ ] 知道SSTORE的不同场景Gas消耗
- [ ] 理解storage packing的原理
- [ ] 掌握calldata vs memory的选择
- [ ] 能够识别常见的Gas浪费模式

**面试问题准备**：
- Q: 如何优化Solidity合约的Gas消耗？
- A: 1) Storage打包 2) 使用calldata 3) 缓存storage到memory 4) 优化循环 5) 使用unchecked 6) 合理使用indexed事件

---

### **Day 20：交易执行与状态转换**

**学习目标**：
- 理解交易的完整生命周期
- 掌握StateTransition的实现
- 分析交易Receipt的生成

**学习内容**：

1. **交易执行入口**：
   ```go
   // core/state_processor.go
   type StateProcessor struct {
       config *params.ChainConfig
       bc     *BlockChain
       engine consensus.Engine
   }

   func (p *StateProcessor) Process(block *types.Block, statedb *state.StateDB,
                                     cfg vm.Config) (types.Receipts, []*types.Log, uint64, error) {
       var (
           receipts    types.Receipts
           usedGas     = new(uint64)
           header      = block.Header()
           blockHash   = block.Hash()
           blockNumber = block.Number()
           allLogs     []*types.Log
           gp          = new(GasPool).AddGas(block.GasLimit())
       )

       // 处理区块中的每笔交易
       for i, tx := range block.Transactions() {
           statedb.Prepare(tx.Hash(), i)
           receipt, err := applyTransaction(p.config, p.bc, nil, gp, statedb, header, tx, usedGas, cfg)
           if err != nil {
               return nil, nil, 0, fmt.Errorf("could not apply tx %d [%v]: %w", i, tx.Hash().Hex(), err)
           }
           receipts = append(receipts, receipt)
           allLogs = append(allLogs, receipt.Logs...)
       }

       // 执行区块奖励
       p.engine.Finalize(p.bc, header, statedb, block.Transactions(), block.Uncles(), receipts)

       return receipts, allLogs, *usedGas, nil
   }
   ```

2. **applyTransaction实现**：
   ```go
   // core/state_processor.go
   func applyTransaction(config *params.ChainConfig, bc ChainContext, author *common.Address,
                         gp *GasPool, statedb *state.StateDB, header *types.Header,
                         tx *types.Transaction, usedGas *uint64, cfg vm.Config) (*types.Receipt, error) {
       // 1. 转换为Message
       msg, err := tx.AsMessage(types.MakeSigner(config, header.Number), header.BaseFee)
       if err != nil {
           return nil, err
       }

       // 2. 创建EVM上下文
       context := NewEVMBlockContext(header, bc, author)
       vmenv := vm.NewEVM(context, vm.TxContext{}, statedb, config, cfg)

       // 3. 应用状态转换
       result, err := ApplyMessage(vmenv, msg, gp)
       if err != nil {
           return nil, err
       }

       // 4. 更新状态
       *usedGas += result.UsedGas

       // 5. 创建Receipt
       receipt := &types.Receipt{Type: tx.Type(), PostState: root, CumulativeGasUsed: *usedGas}
       if result.Failed() {
           receipt.Status = types.ReceiptStatusFailed
       } else {
           receipt.Status = types.ReceiptStatusSuccessful
       }
       receipt.TxHash = tx.Hash()
       receipt.GasUsed = result.UsedGas

       // 6. 如果是合约创建，记录合约地址
       if msg.To() == nil {
           receipt.ContractAddress = crypto.CreateAddress(vmenv.TxContext.Origin, tx.Nonce())
       }

       // 7. 记录日志
       receipt.Logs = statedb.GetLogs(tx.Hash(), blockHash)
       receipt.Bloom = types.CreateBloom(types.Receipts{receipt})
       receipt.BlockHash = blockHash
       receipt.BlockNumber = header.Number
       receipt.TransactionIndex = uint(statedb.TxIndex())

       return receipt, nil
   }
   ```

3. **StateTransition核心逻辑**：
   ```go
   // core/state_transition.go
   type StateTransition struct {
       gp         *GasPool
       msg        Message
       gas        uint64
       gasPrice   *big.Int
       gasFeeCap  *big.Int
       gasTipCap  *big.Int
       initialGas uint64
       value      *big.Int
       data       []byte
       state      vm.StateDB
       evm        *vm.EVM
   }

   func (st *StateTransition) TransitionDb() (*ExecutionResult, error) {
       // 1. 检查nonce
       if msg := st.msg; msg.Nonce() != st.state.GetNonce(msg.From()) {
           return nil, fmt.Errorf("%w: address %v, tx: %d state: %d",
               ErrNonceTooLow, msg.From().Hex(), msg.Nonce(), st.state.GetNonce(msg.From()))
       }

       // 2. 购买Gas
       if err := st.preCheck(); err != nil {
           return nil, err
       }

       // 3. 扣除Gas费用
       st.state.SubBalance(msg.From(), mgval)
       st.gas += st.initialGas

       // 4. EIP-1559: 销毁BaseFee
       if st.evm.ChainConfig().IsLondon(st.evm.Context.BlockNumber) {
           st.state.AddBalance(st.evm.Context.Coinbase, new(big.Int).Mul(new(big.Int).SetUint64(st.gasUsed()), st.evm.Context.BaseFee))
       }

       // 5. 执行交易
       var (
           ret   []byte
           vmerr error
       )
       if contractCreation {
           ret, _, st.gas, vmerr = st.evm.Create(sender, st.data, st.gas, st.value)
       } else {
           st.state.SetNonce(msg.From(), st.state.GetNonce(sender.Address())+1)
           ret, st.gas, vmerr = st.evm.Call(sender, st.to(), st.data, st.gas, st.value)
       }

       // 6. 计算退款
       refund := st.gasUsed() / params.RefundQuotient
       if refund > st.state.GetRefund() {
           refund = st.state.GetRefund()
       }
       st.gas += refund

       // 7. 退还剩余Gas
       remaining := new(big.Int).Mul(new(big.Int).SetUint64(st.gas), st.gasPrice)
       st.state.AddBalance(msg.From(), remaining)

       // 8. 矿工获得小费
       effectiveTip := st.gasPrice
       if st.evm.ChainConfig().IsLondon(st.evm.Context.BlockNumber) {
           effectiveTip = cmath.BigMin(st.gasTipCap, new(big.Int).Sub(st.gasFeeCap, st.evm.Context.BaseFee))
       }
       st.state.AddBalance(st.evm.Context.Coinbase, new(big.Int).Mul(new(big.Int).SetUint64(st.gasUsed()), effectiveTip))

       return &ExecutionResult{
           UsedGas:    st.gasUsed(),
           Err:        vmerr,
           ReturnData: ret,
       }, nil
   }
   ```

4. **Receipt布隆过滤器**：
   ```go
   // core/types/bloom9.go
   func CreateBloom(receipts Receipts) Bloom {
       var bin Bloom
       for _, receipt := range receipts {
           for _, log := range receipt.Logs {
               bin.Add(log.Address.Bytes())
               for _, topic := range log.Topics {
                   bin.Add(topic.Bytes())
               }
           }
       }
       return bin
   }

   func (b Bloom) Add(data []byte) {
       // Bloom Filter的3个哈希函数
       for i := 0; i < 6; i += 2 {
           // KeccakState哈希
           keccak := crypto.Keccak256(data)

           // 提取位位置
           v := (uint(keccak[i+1]) + (uint(keccak[i]) << 8)) & 0x7ff

           // 设置位
           b[BloomByteLength-1-v/8] |= 1 << (v % 8)
       }
   }

   // 快速检查日志是否可能存在
   func (b Bloom) Test(test []byte) bool {
       for i := 0; i < 6; i += 2 {
           keccak := crypto.Keccak256(test)
           v := (uint(keccak[i+1]) + (uint(keccak[i]) << 8)) & 0x7ff
           if b[BloomByteLength-1-v/8]&(1<<(v%8)) == 0 {
               return false
           }
       }
       return true
   }
   ```

**实践任务**：
模拟完整的交易执行：
```go
func SimulateTransaction() {
    // 1. 准备状态
    db := rawdb.NewMemoryDatabase()
    stateDB, _ := state.New(common.Hash{}, state.NewDatabase(db), nil)

    // 2. 创建账户
    sender := common.HexToAddress("0x1234")
    receiver := common.HexToAddress("0x5678")
    stateDB.SetBalance(sender, big.NewInt(1000000000000000000))  // 1 ETH
    stateDB.SetNonce(sender, 0)

    // 3. 创建交易
    tx := types.NewTransaction(
        0,                                      // nonce
        receiver,                               // to
        big.NewInt(100000000000000000),        // value: 0.1 ETH
        21000,                                  // gas limit
        big.NewInt(1000000000),                // gas price: 1 Gwei
        nil,                                    // data
    )

    // 4. 执行交易
    msg, _ := tx.AsMessage(types.HomesteadSigner{}, nil)
    gp := new(GasPool).AddGas(tx.Gas())

    // 5. 应用状态转换
    evm := vm.NewEVM(vm.BlockContext{}, vm.TxContext{Origin: sender}, stateDB, params.TestChainConfig, vm.Config{})
    result, _ := ApplyMessage(evm, msg, gp)

    // 6. 检查结果
    fmt.Printf("Gas Used: %d\n", result.UsedGas)
    fmt.Printf("Sender Balance: %s\n", stateDB.GetBalance(sender))
    fmt.Printf("Receiver Balance: %s\n", stateDB.GetBalance(receiver))
}
```

**作业**：
- [ ] 阅读state_processor.go的Process方法
- [ ] 理解StateTransition的完整流程
- [ ] 分析Receipt的生成逻辑
- [ ] 实现一个简单的交易模拟器

**检查点**：
- [ ] 理解交易执行的8个步骤
- [ ] 知道Gas退款的计算规则
- [ ] 理解Bloom Filter的作用
- [ ] 能解释EIP-1559的费用分配

**面试问题准备**：
- Q: 以太坊交易执行的完整流程是什么？
- A: 1) 验证nonce 2) 购买Gas 3) 扣除Gas费 4) 执行EVM 5) 计算退款 6) 退还剩余Gas 7) 矿工收小费 8) 生成Receipt

---

### **Day 21：周末项目 - Mini区块链节点**

**项目目标**：
整合本周所学，实现一个简化的区块链节点，包括：
- StateDB + MPT存储
- EVM执行引擎
- 交易处理
- 区块验证

**项目结构**：
```
mini-blockchain/
├── state/
│   ├── statedb.go       # 状态数据库
│   └── mpt.go           # Merkle Patricia Trie
├── vm/
│   ├── evm.go           # EVM执行器
│   ├── instructions.go  # 指令集
│   └── stack.go         # 栈实现
├── core/
│   ├── block.go         # 区块结构
│   ├── transaction.go   # 交易结构
│   ├── processor.go     # 交易处理器
│   └── blockchain.go    # 区块链管理
└── main.go
```

**核心代码实现**：

```go
// state/statedb.go
package state

import (
    "math/big"
    "github.com/ethereum/go-ethereum/common"
)

type Account struct {
    Nonce    uint64
    Balance  *big.Int
    CodeHash common.Hash
    Root     common.Hash  // storage root
}

type StateDB struct {
    trie     *MPT
    accounts map[common.Address]*Account
    storage  map[common.Address]map[common.Hash]common.Hash
}

func NewStateDB() *StateDB {
    return &StateDB{
        trie:     NewMPT(),
        accounts: make(map[common.Address]*Account),
        storage:  make(map[common.Address]map[common.Hash]common.Hash),
    }
}

func (s *StateDB) GetBalance(addr common.Address) *big.Int {
    acc := s.getAccount(addr)
    if acc == nil {
        return big.NewInt(0)
    }
    return acc.Balance
}

func (s *StateDB) SetBalance(addr common.Address, amount *big.Int) {
    acc := s.getOrCreateAccount(addr)
    acc.Balance = new(big.Int).Set(amount)
}

func (s *StateDB) GetState(addr common.Address, key common.Hash) common.Hash {
    if s.storage[addr] == nil {
        return common.Hash{}
    }
    return s.storage[addr][key]
}

func (s *StateDB) SetState(addr common.Address, key, value common.Hash) {
    if s.storage[addr] == nil {
        s.storage[addr] = make(map[common.Hash]common.Hash)
    }
    s.storage[addr][key] = value
}

func (s *StateDB) Commit() common.Hash {
    // 提交所有账户到MPT
    for addr, acc := range s.accounts {
        data := encodeAccount(acc)
        s.trie.Update(addr.Bytes(), data)
    }
    return s.trie.Hash()
}

// vm/evm.go
package vm

import (
    "math/big"
    "mini-blockchain/state"
)

type EVM struct {
    StateDB *state.StateDB
    caller  common.Address
    value   *big.Int
    gas     uint64
}

func NewEVM(stateDB *state.StateDB) *EVM {
    return &EVM{StateDB: stateDB}
}

func (evm *EVM) Call(caller, to common.Address, input []byte, gas uint64, value *big.Int) ([]byte, uint64, error) {
    // 1. 转账
    if value.Sign() > 0 {
        callerBalance := evm.StateDB.GetBalance(caller)
        if callerBalance.Cmp(value) < 0 {
            return nil, gas, fmt.Errorf("insufficient balance")
        }
        evm.StateDB.SetBalance(caller, new(big.Int).Sub(callerBalance, value))
        evm.StateDB.SetBalance(to, new(big.Int).Add(evm.StateDB.GetBalance(to), value))
    }

    // 2. 获取代码
    code := evm.StateDB.GetCode(to)
    if len(code) == 0 {
        return nil, gas, nil  // EOA账户
    }

    // 3. 执行代码
    contract := NewContract(caller, to, value, gas)
    contract.Code = code
    contract.Input = input

    return evm.Run(contract)
}

func (evm *EVM) Run(contract *Contract) ([]byte, uint64, error) {
    var (
        stack = NewStack()
        mem   = NewMemory()
        pc    = uint64(0)
    )

    for pc < uint64(len(contract.Code)) {
        op := OpCode(contract.Code[pc])

        switch op {
        case PUSH1:
            pc++
            stack.Push(new(big.Int).SetBytes([]byte{contract.Code[pc]}))
            contract.Gas -= 3

        case ADD:
            a, b := stack.Pop(), stack.Pop()
            stack.Push(new(big.Int).Add(a, b))
            contract.Gas -= 3

        case MUL:
            a, b := stack.Pop(), stack.Pop()
            stack.Push(new(big.Int).Mul(a, b))
            contract.Gas -= 5

        case SSTORE:
            loc, val := stack.Pop(), stack.Pop()
            evm.StateDB.SetState(contract.Address, common.BytesToHash(loc.Bytes()), common.BytesToHash(val.Bytes()))
            contract.Gas -= 20000

        case SLOAD:
            loc := stack.Pop()
            val := evm.StateDB.GetState(contract.Address, common.BytesToHash(loc.Bytes()))
            stack.Push(new(big.Int).SetBytes(val.Bytes()))
            contract.Gas -= 800

        case RETURN:
            offset, size := stack.Pop(), stack.Pop()
            return mem.GetCopy(int64(offset.Uint64()), int64(size.Uint64())), contract.Gas, nil

        case STOP:
            return nil, contract.Gas, nil
        }

        pc++

        if contract.Gas <= 0 {
            return nil, 0, fmt.Errorf("out of gas")
        }
    }

    return nil, contract.Gas, nil
}

// core/processor.go
package core

type Processor struct {
    stateDB *state.StateDB
    evm     *vm.EVM
}

func NewProcessor(stateDB *state.StateDB) *Processor {
    return &Processor{
        stateDB: stateDB,
        evm:     vm.NewEVM(stateDB),
    }
}

func (p *Processor) ProcessTransaction(tx *Transaction, gasLimit uint64) (*Receipt, error) {
    // 1. 验证nonce
    senderNonce := p.stateDB.GetNonce(tx.From)
    if tx.Nonce != senderNonce {
        return nil, fmt.Errorf("invalid nonce")
    }

    // 2. 扣除Gas费用
    gasCost := new(big.Int).Mul(new(big.Int).SetUint64(tx.GasLimit), tx.GasPrice)
    totalCost := new(big.Int).Add(gasCost, tx.Value)

    senderBalance := p.stateDB.GetBalance(tx.From)
    if senderBalance.Cmp(totalCost) < 0 {
        return nil, fmt.Errorf("insufficient balance")
    }

    p.stateDB.SetBalance(tx.From, new(big.Int).Sub(senderBalance, gasCost))

    // 3. 执行交易
    var (
        ret     []byte
        gasUsed uint64
        err     error
    )

    if tx.To == nil {
        // 合约创建
        ret, gasUsed, err = p.evm.Create(tx.From, tx.Data, tx.GasLimit, tx.Value)
    } else {
        // 合约调用
        ret, gasUsed, err = p.evm.Call(tx.From, *tx.To, tx.Data, tx.GasLimit, tx.Value)
    }

    // 4. 退还剩余Gas
    remainingGas := tx.GasLimit - gasUsed
    refund := new(big.Int).Mul(new(big.Int).SetUint64(remainingGas), tx.GasPrice)
    p.stateDB.SetBalance(tx.From, new(big.Int).Add(p.stateDB.GetBalance(tx.From), refund))

    // 5. 增加nonce
    p.stateDB.SetNonce(tx.From, senderNonce+1)

    // 6. 创建Receipt
    receipt := &Receipt{
        TxHash:          tx.Hash(),
        GasUsed:         gasUsed,
        ContractAddress: nil,
        Status:          1,
    }

    if err != nil {
        receipt.Status = 0
    }

    return receipt, nil
}

// main.go
func main() {
    // 1. 创建StateDB
    stateDB := state.NewStateDB()

    // 2. 创建账户
    alice := common.HexToAddress("0xAlice")
    bob := common.HexToAddress("0xBob")
    stateDB.SetBalance(alice, big.NewInt(10000000000))
    stateDB.SetNonce(alice, 0)

    // 3. 创建交易
    tx := &Transaction{
        From:     alice,
        To:       &bob,
        Value:    big.NewInt(1000000),
        GasLimit: 21000,
        GasPrice: big.NewInt(1000000000),
        Nonce:    0,
        Data:     nil,
    }

    // 4. 处理交易
    processor := NewProcessor(stateDB)
    receipt, err := processor.ProcessTransaction(tx, 100000)

    if err != nil {
        fmt.Printf("Transaction failed: %v\n", err)
    } else {
        fmt.Printf("Transaction succeeded!\n")
        fmt.Printf("Gas Used: %d\n", receipt.GasUsed)
        fmt.Printf("Alice Balance: %s\n", stateDB.GetBalance(alice))
        fmt.Printf("Bob Balance: %s\n", stateDB.GetBalance(bob))
    }

    // 5. 提交状态
    stateRoot := stateDB.Commit()
    fmt.Printf("State Root: %s\n", stateRoot.Hex())
}
```

**测试要求**：
- [ ] 实现账户余额转账
- [ ] 实现简单的合约部署（PUSH+SSTORE）
- [ ] 实现合约调用
- [ ] 验证状态根正确性
- [ ] 测试Gas计算准确性

**扩展挑战**：
- [ ] 添加更多EVM指令（CALL、DELEGATECALL等）
- [ ] 实现区块链数据结构（链接区块）
- [ ] 添加区块验证逻辑
- [ ] 实现简单的共识机制（PoW）

**提交要求**：
完成后提交：
1. 完整代码（带注释）
2. 测试用例和结果截图
3. 性能分析报告（每笔交易Gas消耗）
4. 学习总结（本周最大收获）

---

## **阶段3：P2P网络与共识机制（第5-6周）**

### **Day 22-25：P2P网络基础**

### **Day 22：Kademlia DHT原理**

**学习目标**：
- 理解分布式哈希表（DHT）的概念
- 掌握Kademlia算法的核心思想
- 分析以太坊的节点发现机制

**学习内容**：

1. **Kademlia核心概念**：
   ```go
   // p2p/discover/v4_udp.go

   const (
       alpha           = 3   // 并发查询数
       bucketSize      = 16  // 每个K桶大小
       maxReplacements = 10  // 替换列表大小
       hashBits        = len(common.Hash{}) * 8
       nBuckets        = hashBits + 1  // 256个K桶
   )

   // 节点表
   type Table struct {
       mutex   sync.Mutex
       buckets [nBuckets]*bucket  // K桶数组
       rand    *rand.Rand
       db      *enode.DB
       net     transport
       self    *enode.Node  // 本地节点
   }

   // K桶结构
   type bucket struct {
       entries      []*node       // 活跃节点列表
       replacements []*node       // 替换候选列表
       ips          netutil.DistinctNetSet
   }

   // 计算节点距离（XOR）
   func logdist(a, b common.Hash) int {
       lz := 0
       for i := range a {
           x := a[i] ^ b[i]
           if x == 0 {
               lz += 8
           } else {
               lz += bits.LeadingZeros8(x)
               break
           }
       }
       return len(a)*8 - lz
   }
   ```

2. **节点查找算法**：
   ```go
   // p2p/discover/v4_lookup.go

   func (t *Table) lookup(targetID enode.ID, refreshIfEmpty bool) []*enode.Node {
       var (
           target         = enode.ID(targetID)
           asked          = make(map[enode.ID]bool)
           seen           = make(map[enode.ID]bool)
           reply          = make(chan []*node, alpha)
           pendingQueries = 0
           result         *nodesByDistance
       )

       // 从本地表中找最近的节点
       result = t.closest(target, bucketSize, false)

       for {
           // 选择alpha个最近且未询问的节点
           for i := 0; i < len(result.entries) && pendingQueries < alpha; i++ {
               n := result.entries[i]
               if !asked[n.ID()] {
                   asked[n.ID()] = true
                   pendingQueries++

                   // 并发查询
                   go func(n *node) {
                       // 发送FIND_NODE请求
                       r := t.net.findnode(n.ID(), n.addr(), target)
                       reply <- r
                   }(n)
               }
           }

           if pendingQueries == 0 {
               break  // 查询完成
           }

           // 等待回复
           select {
           case nodes := <-reply:
               pendingQueries--
               for _, n := range nodes {
                   if !seen[n.ID()] {
                       seen[n.ID()] = true
                       result.push(n, bucketSize)
                   }
               }
           }
       }

       return unwrapNodes(result.entries)
   }
   ```

3. **节点距离排序**：
   ```go
   type nodesByDistance struct {
       entries []*node
       target  enode.ID
   }

   func (h *nodesByDistance) push(n *node, maxElems int) {
       // 插入排序，保持最近的maxElems个节点
       ix := sort.Search(len(h.entries), func(i int) bool {
           return enode.DistCmp(h.target, h.entries[i].ID(), n.ID()) > 0
       })

       if len(h.entries) < maxElems {
           h.entries = append(h.entries, n)
       }
       if ix < len(h.entries) {
           copy(h.entries[ix+1:], h.entries[ix:])
           h.entries[ix] = n
       }
   }
   ```

**作业**：
- [ ] 实现XOR距离计算函数
- [ ] 实现K桶的插入逻辑
- [ ] 模拟节点查找过程
- [ ] 画出Kademlia路由表结构图

**检查点**：
- [ ] 理解XOR距离的性质
- [ ] 知道为什么需要256个K桶
- [ ] 理解并发查询的优势
- [ ] 能解释节点替换策略

---

### **Day 23：DevP2P协议**

**学习目标**：
- 理解以太坊P2P协议栈
- 掌握RLPx握手流程
- 分析协议消息的编解码

**学习内容**：

1. **RLPx握手过程**：
   ```go
   // p2p/rlpx.go

   // 1. 发起方发送auth消息
   func (h *handshakeState) makeAuthMsg(prv *ecdsa.PrivateKey) ([]byte, error) {
       // 生成随机私钥
       h.initiatorNonce = make([]byte, shaLen)
       rand.Read(h.initiatorNonce)

       // ECDH密钥交换
       h.remotePub = h.remote.Pubkey()
       token, err := h.staticSharedSecret(prv)

       // 构造auth消息
       msg := &authMsgV4{
           Signature:       sig,
           InitiatorPubkey: crypto.FromECDSAPub(&prv.PublicKey)[1:],
           Nonce:           h.initiatorNonce,
           Version:         4,
       }

       return sealEIP8(msg, h)
   }

   // 2. 接收方回复ack消息
   func (h *handshakeState) handleAuthMsg(msg []byte, prv *ecdsa.PrivateKey) error {
       // 解密auth消息
       auth := new(authMsgV4)
       if err := readHandshakeMsg(auth, msg, prv); err != nil {
           return err
       }

       // 生成responder nonce
       h.responderNonce = make([]byte, shaLen)
       rand.Read(h.responderNonce)

       // 发送ack
       ack := &authRespV4{
           RandomPubkey: crypto.FromECDSAPub(&prv.PublicKey)[1:],
           Nonce:        h.responderNonce,
           Version:      4,
       }

       return h.write(sealEIP8(ack, h))
   }

   // 3. 派生会话密钥
   func (h *handshakeState) secrets(auth, authResp []byte) (secrets, error) {
       // 使用ECDH共享密钥
       sharedSecret := h.staticSharedSecret(prv)

       // 派生密钥材料
       kdf := crypto.Keccak256(sharedSecret, h.responderNonce, h.initiatorNonce, auth, authResp)

       // 生成AES和MAC密钥
       aesSecret := kdf[:16]
       macSecret := crypto.Keccak256(kdf[16:32])

       return secrets{
           AES:        aesSecret,
           MAC:        macSecret,
           EgressMAC:  egressMAC,
           IngressMAC: ingressMAC,
       }, nil
   }
   ```

2. **消息帧格式**：
   ```go
   // p2p/rlpx_frame.go

   type frameWriter struct {
       aes cipher.Stream  // AES-CTR加密
       mac hash.Hash      // MAC认证
   }

   func (w *frameWriter) WriteMsg(msg Msg) error {
       // 1. 消息头（16字节）
       head := make([]byte, 16)
       binary.BigEndian.PutUint32(head[:4], uint32(msg.Size))
       head[4] = 0xC2  // RLP list
       head[5] = byte(msg.Code)

       // 2. 加密消息头
       w.aes.XORKeyStream(head, head)

       // 3. 计算header MAC
       w.mac.Write(head)
       headerMAC := w.mac.Sum(nil)[:16]

       // 4. 写入：header + header-mac + frame + frame-mac
       w.conn.Write(head)
       w.conn.Write(headerMAC)

       // 5. 分帧传输消息体
       for {
           frame := make([]byte, 16*((n+15)/16))  // 填充到16字节倍数
           w.aes.XORKeyStream(frame, plaintext)
           w.mac.Write(frame)
           frameMAC := w.mac.Sum(nil)[:16]

           w.conn.Write(frame)
           w.conn.Write(frameMAC)
       }

       return nil
   }
   ```

3. **协议多路复用**：
   ```go
   // p2p/peer.go

   type Peer struct {
       rw       *conn
       running  map[string]*protoRW  // 协议名 -> 读写器
       protoErr chan error
       closed   chan struct{}
   }

   func (p *Peer) run() error {
       // 并发运行多个协议
       for _, proto := range p.running {
           go func(proto Protocol) {
               err := proto.Run(p, protoRW)
               p.protoErr <- err
           }(proto)
       }

       // 等待任一协议出错
       return <-p.protoErr
   }

   // 协议示例：eth protocol
   func (p *Peer) runEthProtocol() error {
       // 1. 握手
       if err := p.Handshake(p.NetworkId, p.Head, p.Genesis); err != nil {
           return err
       }

       // 2. 消息循环
       for {
           msg, err := p.rw.ReadMsg()
           if err != nil {
               return err
           }

           switch msg.Code {
           case StatusMsg:
               // 处理状态消息
           case NewBlockHashesMsg:
               // 处理新区块通知
           case GetBlockHeadersMsg:
               // 处理区块头请求
           }
       }
   }
   ```

**作业**：
- [ ] 抓包分析RLPx握手过程
- [ ] 实现简单的AES-CTR加密/解密
- [ ] 理解MAC计算过程
- [ ] 画出DevP2P协议栈层次图

---

### **Day 24：Gossip协议**

**学习目标**：
- 理解Gossip消息传播机制
- 掌握交易和区块的广播策略
- 分析网络拓扑对传播的影响

**学习内容**：

1. **交易广播**：
   ```go
   // eth/handler.go

   func (h *handler) BroadcastTransactions(txs types.Transactions) {
       var (
           annoCount   = 0
           annos       = make(map[*ethPeer][]common.Hash)       // 哈希通知
           broadcasts  = make(map[*ethPeer]types.Transactions)  // 完整交易
       )

       // 1. 分类peers
       for _, tx := range txs {
           peers := h.peers.peersWithoutTransaction(tx.Hash())

           // 对于大型交易，只发送哈希给部分peers
           numDirect := int(math.Sqrt(float64(len(peers))))
           for _, peer := range peers[:numDirect] {
               broadcasts[peer] = append(broadcasts[peer], tx)
           }

           // 其余peers只收到哈希通知
           for _, peer := range peers[numDirect:] {
               annos[peer] = append(annos[peer], tx.Hash())
               annoCount++
           }
       }

       // 2. 发送完整交易
       for peer, txs := range broadcasts {
           peer.AsyncSendTransactions(txs)
       }

       // 3. 发送哈希通知
       for peer, hashes := range annos {
           peer.AsyncSendPooledTransactionHashes(hashes)
       }
   }
   ```

2. **区块传播**：
   ```go
   // eth/handler.go

   func (h *handler) BroadcastBlock(block *types.Block, propagate bool) {
       hash := block.Hash()
       peers := h.peers.peersWithoutBlock(hash)

       if propagate {
           // 完整传播：发送给sqrt(peers)个节点
           var transfer []*ethPeer
           if len(peers) > int(math.Sqrt(float64(len(peers)))) {
               transfer = peers[:int(math.Sqrt(float64(len(peers))))]
           } else {
               transfer = peers
           }

           for _, peer := range transfer {
               peer.AsyncSendNewBlock(block, td)
           }

           log.Trace("Propagated block", "hash", hash, "recipients", len(transfer), "duration", common.PrettyDuration(time.Since(block.ReceivedAt)))
       }

       // 其余节点只发送哈希+高度
       if len(peers) > len(transfer) {
           for _, peer := range peers[len(transfer):] {
               peer.AsyncSendNewBlockHash(block)
           }
       }
   }
   ```

3. **Peer选择策略**：
   ```go
   // eth/peerset.go

   type peerSet struct {
       peers  map[string]*ethPeer
       lock   sync.RWMutex
   }

   func (ps *peerSet) peersWithoutBlock(hash common.Hash) []*ethPeer {
       ps.lock.RLock()
       defer ps.lock.RUnlock()

       list := make([]*ethPeer, 0, len(ps.peers))
       for _, p := range ps.peers {
           if !p.KnownBlock(hash) {
               list = append(list, p)
           }
       }
       return list
   }

   // 每个peer跟踪已知的哈希
   type ethPeer struct {
       knownBlocks     mapset.Set[common.Hash]  // 已知区块
       knownTxs        mapset.Set[common.Hash]  // 已知交易
       queuedBlocks    chan *blockPropagateMsg  // 待发送区块队列
       queuedBlockAnns chan *types.Block        // 待发送区块通知队列
   }

   func (p *ethPeer) MarkBlock(hash common.Hash) {
       // 使用LRU缓存，避免无限增长
       for p.knownBlocks.Cardinality() >= maxKnownBlocks {
           p.knownBlocks.Pop()
       }
       p.knownBlocks.Add(hash)
   }
   ```

4. **消息去重**：
   ```go
   // p2p/peer.go

   type Peer struct {
       // ...
       knownMsgs *lru.Cache  // 最近seen的消息
   }

   func (p *Peer) Handle(msg *Message) error {
       // 检查是否已处理过
       msgID := msg.Hash()
       if p.knownMsgs.Contains(msgID) {
           return nil  // 忽略重复消息
       }

       // 标记为已见
       p.knownMsgs.Add(msgID, true)

       // 处理消息
       return p.handleMessage(msg)
   }
   ```

**实践任务**：
模拟Gossip传播：
```go
package main

import (
    "fmt"
    "math/rand"
    "time"
)

type Node struct {
    ID       int
    Peers    []*Node
    Received map[string]bool
}

func (n *Node) Broadcast(msgID string) {
    if n.Received[msgID] {
        return  // 已收到，不再广播
    }

    n.Received[msgID] = true
    fmt.Printf("Node %d received message %s\n", n.ID, msgID)

    // 随机选择sqrt(peers)个节点转发
    numForward := int(math.Sqrt(float64(len(n.Peers))))
    selected := rand.Perm(len(n.Peers))[:numForward]

    for _, idx := range selected {
        go func(peer *Node) {
            time.Sleep(10 * time.Millisecond)  // 模拟网络延迟
            peer.Broadcast(msgID)
        }(n.Peers[idx])
    }
}

func main() {
    // 创建100个节点的网络
    nodes := make([]*Node, 100)
    for i := range nodes {
        nodes[i] = &Node{
            ID:       i,
            Peers:    make([]*Node, 0),
            Received: make(map[string]bool),
        }
    }

    // 随机连接（每个节点10个邻居）
    for _, node := range nodes {
        for len(node.Peers) < 10 {
            peer := nodes[rand.Intn(100)]
            if peer != node {
                node.Peers = append(node.Peers, peer)
            }
        }
    }

    // 节点0广播消息
    start := time.Now()
    nodes[0].Broadcast("MSG-123")
    time.Sleep(1 * time.Second)

    // 统计覆盖率
    received := 0
    for _, node := range nodes {
        if node.Received["MSG-123"] {
            received++
        }
    }

    fmt.Printf("Coverage: %d/%d (%.2f%%) in %v\n",
        received, len(nodes), float64(received)/float64(len(nodes))*100, time.Since(start))
}
```

**作业**：
- [ ] 分析sqrt传播策略的优势
- [ ] 计算不同拓扑下的传播延迟
- [ ] 实现消息去重机制
- [ ] 模拟网络分区场景

**检查点**：
- [ ] 理解为什么用sqrt而不是广播给所有peers
- [ ] 知道knownBlocks的作用
- [ ] 理解Gossip的最终一致性
- [ ] 能分析传播延迟的影响因素

---

### **Day 25：共识机制基础**

**学习目标**：
- 理解PoW、PoS、DPoS等共识机制
- 掌握Ethash算法原理
- 分析共识算法的安全性

**学习内容**：

1. **Ethash（PoW）实现**：
   ```go
   // consensus/ethash/ethash.go

   func (ethash *Ethash) VerifySeal(chain consensus.ChainHeaderReader, header *types.Header) error {
       // 1. 检查难度
       if header.Difficulty.Sign() <= 0 {
           return errInvalidDifficulty
       }

       // 2. 获取epoch对应的DAG
       number := header.Number.Uint64()
       epoch := number / epochLength

       // 3. 计算哈希
       digest, result := hashimotoLight(
           ethash.dataset(epoch, false),
           header.HashNoNonce().Bytes(),
           header.Nonce.Uint64(),
       )

       // 4. 验证MixDigest
       if !bytes.Equal(header.MixDigest[:], digest) {
           return errInvalidMixDigest
       }

       // 5. 验证是否满足难度目标
       target := new(big.Int).Div(two256, header.Difficulty)
       if new(big.Int).SetBytes(result).Cmp(target) > 0 {
           return errInvalidPoW
       }

       return nil
   }

   // Hashimoto算法
   func hashimotoLight(dataset []uint32, hash []byte, nonce uint64) ([]byte, []byte) {
       // 1. 初始化混合哈希
       seed := make([]byte, 40)
       copy(seed, hash)
       binary.LittleEndian.PutUint64(seed[32:], nonce)
       seed = crypto.Keccak512(seed)

       mix := make([]uint32, mixBytes/4)
       for i := 0; i < mixBytes/4; i++ {
           mix[i] = binary.LittleEndian.Uint32(seed[i%16*4:])
       }

       // 2. 从DAG中随机读取并混合
       rows := uint32(len(dataset)) / mixBytes
       for i := 0; i < loopAccesses; i++ {
           parent := fnv(uint32(i)^seed[0], mix[i%len(mix)]) % rows
           for j := uint32(0); j < mixBytes/hashBytes; j++ {
               index := 2*parent + j
               // FNV混合
               for k := range mix {
                   mix[k] = fnv(mix[k], dataset[index*hashWords+k])
               }
           }
       }

       // 3. 压缩混合哈希
       digest := make([]byte, common.HashLength)
       for i := 0; i < len(mix); i += 4 {
           val := fnv(fnv(fnv(mix[i], mix[i+1]), mix[i+2]), mix[i+3])
           binary.LittleEndian.PutUint32(digest[i:], val)
       }

       // 4. 最终哈希
       result := crypto.Keccak256(append(seed, digest...))
       return digest, result
   }
   ```

2. **难度调整**：
   ```go
   // consensus/ethash/consensus.go

   func calcDifficultyEip3554(time uint64, parent *types.Header) *big.Int {
       // 1. 基础难度调整
       bigTime := new(big.Int).SetUint64(time)
       bigParentTime := new(big.Int).SetUint64(parent.Time)

       x := new(big.Int)
       y := new(big.Int)

       // 如果出块太快，增加难度
       x.Sub(bigTime, bigParentTime)
       x.Div(x, big.NewInt(9))
       x.Sub(big.NewInt(1), x)
       if x.Cmp(big.NewInt(-99)) < 0 {
           x.Set(big.NewInt(-99))
       }

       y.Div(parent.Difficulty, params.DifficultyBoundDivisor)
       x.Mul(y, x)
       x.Add(parent.Difficulty, x)

       // 2. 难度炸弹（每100,000区块翻倍）
       periodCount := new(big.Int)
       periodCount.SetUint64(parent.Number.Uint64())
       periodCount.Div(periodCount, big.NewInt(100000))
       if periodCount.Cmp(big.NewInt(2)) > 0 {
           y.Sub(periodCount, big.NewInt(2))
           y.Exp(big.NewInt(2), y, nil)  // 2^(period-2)
           x.Add(x, y)
       }

       // 3. 最小难度
       if x.Cmp(params.MinimumDifficulty) < 0 {
           x.Set(params.MinimumDifficulty)
       }

       return x
   }
   ```

3. **PoS原理（Casper FFG概念）**：
   ```go
   // 概念代码（非实际实现）

   type Validator struct {
       PubKey  []byte
       Deposit *big.Int
       Active  bool
   }

   type Checkpoint struct {
       Epoch       uint64
       BlockHash   common.Hash
       Justified   bool
       Finalized   bool
   }

   type Vote struct {
       Source      *Checkpoint  // 源检查点
       Target      *Checkpoint  // 目标检查点
       ValidatorID uint64
       Signature   []byte
   }

   func ProcessVote(vote *Vote, validators map[uint64]*Validator) error {
       // 1. 验证签名
       if !VerifySignature(vote) {
           return errInvalidSignature
       }

       // 2. 检查Slashing条件
       // 条件1：双重投票（同一epoch投两次）
       // 条件2：环绕投票
       if IsSlashable(vote, previousVotes) {
           SlashValidator(vote.ValidatorID)
           return errSlashableVote
       }

       // 3. 记录投票
       votesByTarget[vote.Target] = append(votesByTarget[vote.Target], vote)

       // 4. 检查是否达到2/3多数
       totalDeposit := getTotalDeposit(validators)
       voteWeight := getVoteWeight(votesByTarget[vote.Target], validators)

       if voteWeight*3 >= totalDeposit*2 {
           // Justify检查点
           vote.Target.Justified = true

           // 如果source已经justified，finalize它
           if vote.Source.Justified && vote.Source.Epoch+1 == vote.Target.Epoch {
               vote.Source.Finalized = true
           }
       }

       return nil
   }
   ```

4. **DPoS概念**：
   ```go
   type DPoS struct {
       Witnesses   []common.Address  // 见证人列表
       VotesByNode map[common.Address]*big.Int  // 投票权重
       BlockTime   time.Duration
   }

   func (d *DPoS) ElectWitnesses(votes map[common.Address]*big.Int) {
       // 按票数排序
       type kv struct {
           Address common.Address
           Votes   *big.Int
       }

       var sorted []kv
       for addr, v := range votes {
           sorted = append(sorted, kv{addr, v})
       }

       sort.Slice(sorted, func(i, j int) bool {
           return sorted[i].Votes.Cmp(sorted[j].Votes) > 0
       })

       // 选出前21个
       d.Witnesses = make([]common.Address, 0, 21)
       for i := 0; i < 21 && i < len(sorted); i++ {
           d.Witnesses = append(d.Witnesses, sorted[i].Address)
       }
   }

   func (d *DPoS) GetBlockProducer(blockNum uint64) common.Address {
       // 轮流出块
       index := blockNum % uint64(len(d.Witnesses))
       return d.Witnesses[index]
   }
   ```

**作业**：
- [ ] 理解Ethash算法的ASIC抗性原理
- [ ] 计算不同难度下的预期出块时间
- [ ] 对比PoW、PoS、DPoS的优缺点
- [ ] 分析51%攻击的成本

**检查点**：
- [ ] 理解DAG的生成和作用
- [ ] 知道难度调整的公式
- [ ] 理解PoS的Nothing at Stake问题
- [ ] 能解释Slashing机制的作用

**面试问题准备**：
- Q: PoW和PoS的主要区别是什么？
- A: PoW基于算力竞争，能源消耗高；PoS基于质押代币，节能但需解决Nothing at Stake问题。PoS使用Slashing惩罚作恶者。

---

### **Day 26：Tendermint BFT基础**

**学习目标**：
- 理解拜占庭容错（BFT）原理
- 掌握Tendermint共识的三阶段协议
- 分析Tendermint的活性和安全性保证

**学习内容**：

1. **Tendermint核心概念**：
   ```go
   // tendermint/types/validator_set.go

   type Validator struct {
       Address     Address
       PubKey      crypto.PubKey
       VotingPower int64
       ProposerPriority int64
   }

   type ValidatorSet struct {
       Validators []*Validator
       Proposer   *Validator  // 当前提案者
       totalVotingPower int64
   }

   // 计算2/3多数
   func (vals *ValidatorSet) TotalVotingPower() int64 {
       if vals.totalVotingPower == 0 {
           sum := int64(0)
           for _, val := range vals.Validators {
               sum += val.VotingPower
           }
           vals.totalVotingPower = sum
       }
       return vals.totalVotingPower
   }

   func (vals *ValidatorSet) HasTwoThirdsMajority(votes []*Vote) bool {
       votingPower := int64(0)
       for _, vote := range votes {
           val := vals.GetByAddress(vote.ValidatorAddress)
           if val != nil {
               votingPower += val.VotingPower
           }
       }
       return votingPower*3 > vals.TotalVotingPower()*2
   }
   ```

2. **共识状态机**：
   ```go
   // tendermint/consensus/state.go

   type RoundStepType uint8

   const (
       RoundStepNewHeight     RoundStepType = 0x01
       RoundStepNewRound      RoundStepType = 0x02
       RoundStepPropose       RoundStepType = 0x03
       RoundStepPrevote       RoundStepType = 0x04
       RoundStepPrevoteWait   RoundStepType = 0x05
       RoundStepPrecommit     RoundStepType = 0x06
       RoundStepPrecommitWait RoundStepType = 0x07
       RoundStepCommit        RoundStepType = 0x08
   )

   type ConsensusState struct {
       Height int64
       Round  int32
       Step   RoundStepType

       Validators    *types.ValidatorSet
       Proposal      *types.Proposal
       ProposalBlock *types.Block

       Votes      *HeightVoteSet  // Prevote和Precommit投票
       CommitTime time.Time
   }

   // 共识主循环
   func (cs *ConsensusState) receiveRoutine() {
       for {
           select {
           case <-cs.timeoutTicker.Chan():
               cs.handleTimeout()

           case msg := <-cs.peerMsgQueue:
               cs.handleMsg(msg)

           case mi := <-cs.internalMsgQueue:
               cs.handleInternalMsg(mi)
           }
       }
   }
   ```

3. **三阶段协议实现**：
   ```go
   // 阶段1：Propose（提案）
   func (cs *ConsensusState) enterPropose(height int64, round int32) {
       // 检查是否是提案者
       if cs.isProposer() {
           // 从mempool获取交易
           block := cs.createProposalBlock()

           // 广播提案
           proposal := types.NewProposal(height, round, block)
           cs.signProposal(proposal)
           cs.sendInternalMessage(msgInfo{&ProposalMessage{proposal}})
       }

       // 设置超时
       cs.scheduleTimeout(cs.config.Propose(round), height, round, RoundStepPropose)
   }

   // 阶段2：Prevote（预投票）
   func (cs *ConsensusState) enterPrevote(height int64, round int32) {
       // 决定投票给哪个区块
       var blockID types.BlockID

       if cs.Proposal != nil && cs.ProposalBlock != nil {
           // 验证提案区块
           err := cs.blockExec.ValidateBlock(cs.state, cs.ProposalBlock)
           if err == nil {
               blockID = types.BlockID{Hash: cs.ProposalBlock.Hash()}
           }
       }

       // 广播Prevote
       vote := &types.Vote{
           Type:             tmproto.PrevoteType,
           Height:           height,
           Round:            round,
           BlockID:          blockID,
           ValidatorAddress: cs.privValidator.GetAddress(),
       }
       cs.signVote(vote)
       cs.sendInternalMessage(msgInfo{&VoteMessage{vote}})

       cs.Step = RoundStepPrevote
   }

   // 阶段3：Precommit（预提交）
   func (cs *ConsensusState) enterPrecommit(height int64, round int32) {
       // 检查Prevote是否达到2/3多数
       blockID, ok := cs.Votes.Prevotes(round).TwoThirdsMajority()

       var precommitBlockID types.BlockID
       if ok && !blockID.IsZero() {
           // 有2/3节点prevote了该区块
           precommitBlockID = blockID
       }

       // 广播Precommit
       vote := &types.Vote{
           Type:             tmproto.PrecommitType,
           Height:           height,
           Round:            round,
           BlockID:          precommitBlockID,
           ValidatorAddress: cs.privValidator.GetAddress(),
       }
       cs.signVote(vote)
       cs.sendInternalMessage(msgInfo{&VoteMessage{vote}})

       cs.Step = RoundStepPrecommit
   }

   // 提交区块
   func (cs *ConsensusState) enterCommit(height int64, commitRound int32) {
       // 必须有2/3的Precommit
       blockID, ok := cs.Votes.Precommits(commitRound).TwoThirdsMajority()
       if !ok {
           panic("enterCommit without 2/3 precommits")
       }

       // 执行区块
       stateCopy := cs.state.Copy()
       stateCopy, err := cs.blockExec.ApplyBlock(stateCopy, types.BlockID{Hash: blockID.Hash}, cs.ProposalBlock)

       // 保存区块
       cs.blockStore.SaveBlock(cs.ProposalBlock, cs.makeCommit(commitRound))

       // 更新状态
       cs.updateToState(stateCopy)

       // 进入下一高度
       cs.StartTime = cs.config.Commit(time.Now())
       cs.newStep()
   }
   ```

4. **锁机制（Locking）**：
   ```go
   type ConsensusState struct {
       // ...
       LockedRound  int32
       LockedBlock  *types.Block
       LockedBlockID types.BlockID
   }

   func (cs *ConsensusState) handlePrevotes(round int32) {
       blockID, ok := cs.Votes.Prevotes(round).TwoThirdsMajority()

       if ok && !blockID.IsZero() {
           // 锁定该区块
           if round > cs.LockedRound {
               cs.LockedRound = round
               cs.LockedBlock = cs.ProposalBlock
               cs.LockedBlockID = blockID
           }
       }
   }

   func (cs *ConsensusState) shouldPrevoteBlock(block *types.Block) bool {
       // 如果已锁定，只能prevote锁定的块或nil
       if cs.LockedBlock != nil {
           if bytes.Equal(block.Hash(), cs.LockedBlock.Hash()) {
               return true
           }
           return false
       }
       return true
   }
   ```

**实践任务**：
实现简化版Tendermint共识模拟器：
```go
package main

type SimpleTendermint struct {
    height     int64
    round      int32
    validators []string
    prevotes   map[string]string  // validator -> blockHash
    precommits map[string]string
}

func (t *SimpleTendermint) Propose(proposer string, blockHash string) {
    fmt.Printf("Round %d: %s proposes block %s\n", t.round, proposer, blockHash)
    t.currentProposal = blockHash
}

func (t *SimpleTendermint) Prevote(validator string, blockHash string) {
    t.prevotes[validator] = blockHash
    fmt.Printf("%s prevotes %s\n", validator, blockHash)

    // 检查是否达到2/3
    if t.hasTwoThirds(t.prevotes, blockHash) {
        fmt.Printf("✓ 2/3 prevotes for %s\n", blockHash)
        t.enterPrecommit(blockHash)
    }
}

func (t *SimpleTendermint) Precommit(validator string, blockHash string) {
    t.precommits[validator] = blockHash
    fmt.Printf("%s precommits %s\n", validator, blockHash)

    if t.hasTwoThirds(t.precommits, blockHash) {
        fmt.Printf("✓ 2/3 precommits for %s - COMMITTED!\n", blockHash)
        t.commit(blockHash)
    }
}

func (t *SimpleTendermint) hasTwoThirds(votes map[string]string, blockHash string) bool {
    count := 0
    for _, vote := range votes {
        if vote == blockHash {
            count++
        }
    }
    return count*3 > len(t.validators)*2
}
```

**作业**：
- [ ] 理解Tendermint的三阶段流程
- [ ] 分析锁机制如何保证安全性
- [ ] 模拟拜占庭节点的行为
- [ ] 对比Tendermint与PBFT的区别

**检查点**：
- [ ] 理解为什么需要Prevote和Precommit两轮投票
- [ ] 知道锁机制的作用
- [ ] 理解2/3多数的数学原理
- [ ] 能解释Tendermint如何处理网络分区

---

### **Day 27-28：周末项目 - P2P聊天应用**

**项目目标**：
实现一个基于Kademlia DHT的P2P聊天应用，整合本周所学：
- Kademlia节点发现
- RLPx加密通信
- Gossip消息广播
- 简单共识（消息排序）

**项目结构**：
```
p2p-chat/
├── dht/
│   ├── node.go          # 节点定义
│   ├── table.go         # 路由表
│   └── lookup.go        # 节点查找
├── network/
│   ├── transport.go     # RLPx传输
│   ├── peer.go          # 对等节点
│   └── protocol.go      # 消息协议
├── consensus/
│   └── ordering.go      # 消息排序共识
└── main.go
```

*(详细代码请参考前面的Day 22-25实现，整合成完整应用)*

**功能要求**：
- [ ] 节点启动后自动发现其他节点
- [ ] 支持一对一私聊
- [ ] 支持群聊（Gossip广播）
- [ ] 消息加密传输
- [ ] 消息去重
- [ ] 简单的消息排序共识

**测试要求**：
- [ ] 启动10个节点测试网络
- [ ] 测试消息传播延迟
- [ ] 测试节点离线/恢复场景
- [ ] 分析网络拓扑的演化

---

## **阶段4：存储优化与性能调优（第7-8周）**

### **Day 29：LevelDB存储引擎**

**学习目标**：
- 理解LevelDB的LSM-Tree架构
- 掌握数据读写流程
- 分析Compaction机制

**学习内容**：

1. **LevelDB架构**：
   ```go
   // ethdb/leveldb/leveldb.go

   type Database struct {
       fn string      // 数据库文件路径
       db *leveldb.DB // LevelDB实例

       compTimeMeter    metrics.Meter  // Compaction时间统计
       compReadMeter    metrics.Meter
       compWriteMeter   metrics.Meter
       writeDelayMeter  metrics.Meter
       diskReadMeter    metrics.Meter
       diskWriteMeter   metrics.Meter
   }

   // 打开数据库
   func New(file string, cache int, handles int, namespace string, readonly bool) (*Database, error) {
       options := &opt.Options{
           OpenFilesCacheCapacity: handles,
           BlockCacheCapacity:     cache / 2 * opt.MiB,
           WriteBuffer:            cache / 4 * opt.MiB,
           Filter:                 filter.NewBloomFilter(10),
           DisableSeeksCompaction: true,
       }

       db, err := leveldb.OpenFile(file, options)
       return &Database{
           fn: file,
           db: db,
       }, nil
   }
   ```

2. **读写操作**：
   ```go
   // 写入单个键值对
   func (db *Database) Put(key []byte, value []byte) error {
       return db.db.Put(key, value, nil)
   }

   // 批量写入
   func (db *Database) NewBatch() ethdb.Batch {
       return &batch{
           db: db.db,
           b:  new(leveldb.Batch),
       }
   }

   type batch struct {
       db   *leveldb.DB
       b    *leveldb.Batch
       size int
   }

   func (b *batch) Put(key, value []byte) error {
       b.b.Put(key, value)
       b.size += len(key) + len(value)
       return nil
   }

   func (b *batch) Write() error {
       return b.db.Write(b.b, nil)
   }

   // 读取
   func (db *Database) Get(key []byte) ([]byte, error) {
       dat, err := db.db.Get(key, nil)
       if err != nil {
           return nil, err
       }
       return dat, nil
   }

   // 范围查询
   func (db *Database) NewIterator(prefix []byte, start []byte) ethdb.Iterator {
       return db.db.NewIterator(bytesPrefixRange(prefix, start), nil)
   }
   ```

3. **Compaction流程**：
   ```
   MemTable (内存)
       |
       | 满了（默认4MB）
       v
   Immutable MemTable
       |
       | 刷盘
       v
   Level 0 (SSTable文件，可能有重叠)
       |
       | Compaction
       v
   Level 1 (10MB，无重叠)
       |
       | Compaction
       v
   Level 2 (100MB)
       |
       v
   Level 3... (每层10倍)
   ```

4. **性能优化技巧**：
   ```go
   // 1. 使用Batch减少写放大
   batch := db.NewBatch()
   for i := 0; i < 10000; i++ {
       batch.Put([]byte(fmt.Sprintf("key%d", i)), []byte("value"))
   }
   batch.Write()  // 一次性写入

   // 2. 使用Snapshot保证一致性读
   snapshot, _ := db.db.GetSnapshot()
   defer snapshot.Release()

   val1, _ := snapshot.Get(key1, nil)
   val2, _ := snapshot.Get(key2, nil)

   // 3. 使用Iterator高效遍历
   iter := db.NewIterator([]byte("prefix"), nil)
   defer iter.Release()

   for iter.Next() {
       key := iter.Key()
       value := iter.Value()
       // 处理...
   }

   // 4. 预分配Bloom Filter减少读放大
   options := &opt.Options{
       Filter: filter.NewBloomFilter(10),  // 10 bits per key
   }
   ```

**作业**：
- [ ] 阅读LevelDB的写入流程源码
- [ ] 理解Compaction的触发条件
- [ ] 测试不同写入模式的性能
- [ ] 分析Bloom Filter的作用

---

### **Day 30：Ancient Database（冷数据存储）**

**学习目标**：
- 理解Ancient Database的设计目的
- 掌握冷热数据分离策略
- 分析数据冻结（Freezing）流程

**学习内容**：

1. **Ancient Database架构**：
   ```go
   // core/rawdb/freezer.go

   type freezer struct {
       frozen    atomic.Uint64  // 已冻结的区块数
       tail      atomic.Uint64  // 最早的区块号

       datadir   string
       readonly  bool
       tables    map[string]*freezerTable  // 不同类型数据的表

       instanceLock fileutil.Releaser  // 文件锁
   }

   // 表定义
   const (
       freezerHeaderTable    = "headers"
       freezerHashTable      = "hashes"
       freezerBodiesTable    = "bodies"
       freezerReceiptTable   = "receipts"
       freezerDifficultyTable= "diffs"
   )

   type freezerTable struct {
       items      atomic.Uint64  // 已存储的条目数
       itemOffset atomic.Uint64  // 文件内偏移

       index      *os.File  // 索引文件
       head       *os.File  // 当前写入文件
       files      []*os.File // 历史数据文件

       headId     uint32
       maxFileSize uint32  // 单文件最大大小（默认2GB）
   }
   ```

2. **数据冻结流程**：
   ```go
   // core/rawdb/freezer.go

   func (f *freezer) freeze(db ethdb.KeyValueStore) {
       nfdb := &nofreezedb{KeyValueStore: db}

       for {
           // 1. 检查是否有足够多的区块可以冻结
           // 保留最近的90,000个区块在LevelDB中
           frozen := f.frozen.Load()
           head := ReadHeaderNumber(db, ReadHeadHeaderHash(db))
           if head-frozen <= params.FullImmutabilityThreshold {
               time.Sleep(freezerRecheckInterval)
               continue
           }

           // 2. 冻结一批区块（默认1000个）
           first := frozen + 1
           ancients := make([]common.Hash, 0, freezerBatchLimit)

           for i := first; i <= first+freezerBatchLimit && i <= head-params.FullImmutabilityThreshold; i++ {
               hash := ReadCanonicalHash(nfdb, i)
               ancients = append(ancients, hash)
           }

           // 3. 写入Ancient Database
           batch := f.tables[freezerHeaderTable].NewBatch()
           for i, hash := range ancients {
               number := first + uint64(i)

               // 写入header
               header := ReadHeader(nfdb, hash, number)
               batch.Append(number, header)

               // 写入body
               body := ReadBody(nfdb, hash, number)
               f.tables[freezerBodiesTable].Append(number, body)

               // 写入receipts
               receipts := ReadReceipts(nfdb, hash, number)
               f.tables[freezerReceiptTable].Append(number, receipts)
           }

           // 4. 提交
           if err := batch.commit(); err != nil {
               log.Error("Failed to freeze", "err", err)
               return
           }

           // 5. 从LevelDB中删除
           for i, hash := range ancients {
               DeleteBlock(db, hash, first+uint64(i))
           }

           // 6. 更新frozen计数
           f.frozen.Store(first + uint64(len(ancients)) - 1)
       }
   }
   ```

3. **数据读取**：
   ```go
   // core/rawdb/accessors_chain.go

   func ReadHeader(db ethdb.Reader, hash common.Hash, number uint64) *types.Header {
       // 1. 尝试从LevelDB读取（热数据）
       data := ReadHeaderRLP(db, hash, number)
       if len(data) > 0 {
           header := new(types.Header)
           rlp.DecodeBytes(data, header)
           return header
       }

       // 2. 从Ancient Database读取（冷数据）
       if ancient, ok := db.(ethdb.AncientReader); ok {
           data, _ := ancient.Ancient(freezerHeaderTable, number)
           if len(data) > 0 {
               header := new(types.Header)
               rlp.DecodeBytes(data, header)
               return header
           }
       }

       return nil
   }
   ```

4. **文件格式**：
   ```
   Ancient Database目录结构：
   chaindata/
   └── ancient/
       ├── headers.0000.cdat     # 数据文件
       ├── headers.cidx          # 索引文件
       ├── headers.0001.cdat
       ├── bodies.0000.cdat
       ├── bodies.cidx
       ├── receipts.0000.cdat
       └── receipts.cidx

   索引文件格式（每条8字节）：
   [offset(6字节) | size(2字节)]

   数据文件格式：
   [RLP编码的数据1][RLP编码的数据2]...
   ```

**实践任务**：
分析自己节点的存储分布：
```bash
# 查看LevelDB大小
du -sh ~/.ethereum/geth/chaindata/

# 查看Ancient Database大小
du -sh ~/.ethereum/geth/chaindata/ancient/

# 查看各表大小
du -sh ~/.ethereum/geth/chaindata/ancient/*

# 检查冻结进度
geth attach --exec "eth.syncing"
```

**作业**：
- [ ] 理解为什么需要冷热分离
- [ ] 分析Ancient Database的空间节省效果
- [ ] 计算不同保留策略的存储需求
- [ ] 实现一个简化版的Freezer

---

### **Day 31：Snap Sync同步优化**

**学习目标**：
- 理解不同同步模式的区别
- 掌握Snap Sync的原理和优势
- 分析状态快照的生成和使用

**学习内容**：

1. **同步模式对比**：
   ```go
   // eth/downloader/modes.go

   type SyncMode uint32

   const (
       FullSync  SyncMode = iota // 同步所有区块和状态
       FastSync                  // 同步区块头，最新状态
       SnapSync                  // 快照同步（最快）
       LightSync                 // 轻节点同步
   )

   // 性能对比（主网数据）：
   // FullSync:  ~1周（需要执行所有交易）
   // FastSync:  ~10小时（只验证PoW）
   // SnapSync:  ~2小时（并行下载状态）
   ```

2. **Snap Sync流程**：
   ```go
   // eth/downloader/downloader.go

   func (d *Downloader) snapSync() error {
       // 阶段1：下载区块头
       pivot := d.selectPivotBlock()
       if err := d.fetchHeaders(pivot); err != nil {
           return err
       }

       // 阶段2：下载Pivot区块的状态快照
       root := d.blockchain.GetHeaderByNumber(pivot).Root
       if err := d.downloadState(root); err != nil {
           return err
       }

       // 阶段3：下载区块体和收据
       if err := d.fetchBodies(); err != nil {
           return err
       }

       // 阶段4：追赶链头
       if err := d.processFastSyncContent(); err != nil {
           return err
       }

       return nil
   }
   ```

3. **状态快照下载**：
   ```go
   // eth/protocols/snap/sync.go

   type Syncer struct {
       root     common.Hash
       tasks    []*accountTask  // 账户下载任务
       healer   *healTask       // 修复任务
       update   chan struct{}
   }

   func (s *Syncer) downloadState(root common.Hash) error {
       // 1. 将状态树分割成多个区间
       s.tasks = s.splitState(root, 1024)  // 1024个并发任务

       // 2. 并行下载各区间
       var wg sync.WaitGroup
       for i := 0; i < 256; i++ {  // 256个goroutine
           wg.Add(1)
           go func() {
               defer wg.Done()
               for task := range s.taskQueue {
                   s.downloadAccountRange(task)
               }
           }()
       }

       // 3. 等待完成
       wg.Wait()

       // 4. 修复缺失部分
       return s.heal(root)
   }

   func (s *Syncer) downloadAccountRange(task *accountTask) error {
       // 构造GetAccountRange请求
       req := &GetAccountRangePacket{
           ID:     rand.Uint64(),
           Root:   task.root,
           Origin: task.origin,
           Limit:  task.limit,
           Bytes:  task.bytes,
       }

       // 发送到peer
       peer := s.selectPeer()
       accounts, proofs, err := peer.RequestAccountRange(req)

       // 验证Merkle证明
       if !s.verifyProof(accounts, proofs, task.root) {
           return errInvalidProof
       }

       // 写入数据库
       batch := s.db.NewBatch()
       for _, account := range accounts {
           batch.Put(account.Hash[:], account.Account)
       }
       batch.Write()

       return nil
   }
   ```

4. **状态快照结构**：
   ```go
   // core/state/snapshot/snapshot.go

   type Tree struct {
       layers map[common.Hash]snapshot  // 各层快照

       diskdb ethdb.KeyValueStore  // 持久化存储
       triedb *trie.Database       // Trie数据库

       lock sync.RWMutex
   }

   // 磁盘快照层
   type diskLayer struct {
       root  common.Hash
       cache *fastcache.Cache  // 账户缓存
       stale atomic.Bool
   }

   // 差异层（内存）
   type diffLayer struct {
       parent    snapshot
       root      common.Hash
       accounts  map[common.Hash][]byte  // 修改的账户
       storage   map[common.Hash]map[common.Hash][]byte  // 修改的存储
       stale     atomic.Bool
   }

   // 快照读取（O(1)复杂度）
   func (dl *diskLayer) Account(hash common.Hash) ([]byte, error) {
       // 1. 查缓存
       if blob := dl.cache.Get(nil, hash[:]); len(blob) > 0 {
           return blob, nil
       }

       // 2. 查数据库
       blob := rawdb.ReadAccountSnapshot(dl.diskdb, hash)
       if len(blob) > 0 {
           dl.cache.Set(hash[:], blob)
           return blob, nil
       }

       return nil, ErrNotFound
   }
   ```

**实践任务**：
对比不同同步模式：
```bash
# 1. 启动SnapSync
geth --syncmode "snap" --datadir /tmp/snap-test

# 2. 监控同步进度
geth attach /tmp/snap-test/geth.ipc --exec "eth.syncing"

# 输出示例：
{
  currentBlock: 15000000,
  healedAccounts: 25000000,
  healedBytecodes: 5000,
  healingTrienodes: 1234567,
  highestBlock: 16000000,
  startingBlock: 0,
  syncedAccounts: 75000000,
  syncedAccountBytes: 12345678900,
  syncedStorage: 150000000
}
```

**作业**：
- [ ] 对比FullSync和SnapSync的同步时间
- [ ] 理解为什么SnapSync需要healing阶段
- [ ] 分析状态快照的存储开销
- [ ] 计算并行度对同步速度的影响

---

### **Day 32-35：性能优化综合**

*(本节整合前面所学，进行综合性能调优实践)*

**Day 32：TPS优化**
- 并行交易执行
- 状态访问优化
- Gas计算优化

**Day 33：内存优化**
- 缓存策略
- 对象池复用
- GC调优

**Day 34：网络优化**
- 批量消息处理
- 压缩算法
- 连接池管理

**Day 35：周末项目 - 性能测试框架**
实现一个区块链性能测试工具，能够：
- 模拟大量交易负载
- 测量TPS、延迟、吞吐量
- 分析瓶颈并提出优化建议

---

## **阶段5：区块链安全（第9-10周）**

### **Day 36-42：智能合约安全**

### **Day 36：重入攻击（Reentrancy）**

**学习目标**：
- 理解重入攻击的原理
- 掌握防御技巧
- 分析著名的DAO攻击

**学习内容**：

1. **漏洞代码示例**：
   ```solidity
   // ❌ 存在重入漏洞的合约
   contract VulnerableBank {
       mapping(address => uint256) public balances;

       function deposit() public payable {
           balances[msg.sender] += msg.value;
       }

       function withdraw(uint256 amount) public {
           require(balances[msg.sender] >= amount, "Insufficient balance");

           // 漏洞：先转账，后更新状态
           (bool success, ) = msg.sender.call{value: amount}("");
           require(success, "Transfer failed");

           balances[msg.sender] -= amount;  // ⚠️ 在转账之后！
       }
   }

   // 攻击合约
   contract Attacker {
       VulnerableBank public bank;
       uint256 public attackAmount;

       constructor(address _bank) {
           bank = VulnerableBank(_bank);
       }

       function attack() public payable {
           attackAmount = msg.value;
           bank.deposit{value: msg.value}();
           bank.withdraw(msg.value);
       }

       // 重入点：receive函数
       receive() external payable {
           if (address(bank).balance >= attackAmount) {
               bank.withdraw(attackAmount);  // 再次调用withdraw！
           }
       }
   }
   ```

2. **攻击流程分析**：
   ```
   1. Attacker.attack() 调用 Bank.withdraw(1 ETH)
   2. Bank.withdraw() 检查余额: balances[attacker] = 1 ETH ✓
   3. Bank.withdraw() 转账: attacker.call{value: 1 ETH}()
   4. 触发 Attacker.receive()
   5. Attacker.receive() 再次调用 Bank.withdraw(1 ETH)
   6. Bank.withdraw() 再次检查余额: balances[attacker] = 1 ETH ✓ (还没更新！)
   7. 重复步骤3-6，直到Bank余额为0
   8. 最外层Bank.withdraw()才更新余额: balances[attacker] -= 1 ETH
   ```

3. **防御方案**：
   ```solidity
   // ✅ 方案1：检查-生效-交互模式（CEI Pattern）
   contract SecureBank {
       mapping(address => uint256) public balances;

       function withdraw(uint256 amount) public {
           require(balances[msg.sender] >= amount);

           // 先更新状态
           balances[msg.sender] -= amount;

           // 后转账
           (bool success, ) = msg.sender.call{value: amount}("");
           require(success);
       }
   }

   // ✅ 方案2：互斥锁
   contract BankWithLock {
       mapping(address => uint256) public balances;
       bool private locked;

       modifier noReentrant() {
           require(!locked, "Reentrant call");
           locked = true;
           _;
           locked = false;
       }

       function withdraw(uint256 amount) public noReentrant {
           require(balances[msg.sender] >= amount);
           (bool success, ) = msg.sender.call{value: amount}("");
           require(success);
           balances[msg.sender] -= amount;
       }
   }

   // ✅ 方案3：使用OpenZeppelin的ReentrancyGuard
   import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

   contract BankWithOZ is ReentrancyGuard {
       mapping(address => uint256) public balances;

       function withdraw(uint256 amount) public nonReentrant {
           require(balances[msg.sender] >= amount);
           (bool success, ) = msg.sender.call{value: amount}("");
           require(success);
           balances[msg.sender] -= amount;
       }
   }
   ```

4. **EVM层面分析**：
   ```go
   // 重入攻击在EVM中的调用栈

   // 初始状态：Bank.balance = 10 ETH, balances[attacker] = 1 ETH

   // 调用栈：
   1. Attacker.attack()
   2.   └─> Bank.withdraw(1 ETH)
   3.         └─> attacker.call{value: 1 ETH}()  // EVM: CALL指令
   4.               └─> Attacker.receive()
   5.                     └─> Bank.withdraw(1 ETH)  // 重入！
   6.                           └─> attacker.call{value: 1 ETH}()
   7.                                 └─> Attacker.receive()
   8.                                       └─> Bank.withdraw(1 ETH)
   ...  (直到Bank.balance < 1 ETH)

   // 关键：每次CALL时，调用者的storage状态不会改变
   // 所以每次withdraw()都能通过余额检查
   ```

**实践任务**：
1. 部署漏洞合约和攻击合约到测试网
2. 执行攻击并观察余额变化
3. 修复漏洞并重新测试
4. 使用Remix调试器查看调用栈

**作业**：
- [ ] 复现DAO攻击（Remix测试网）
- [ ] 实现三种防御方案并对比
- [ ] 分析跨函数重入攻击
- [ ] 学习read-only reentrancy

**检查点**：
- [ ] 理解为什么先转账后更新状态会导致重入
- [ ] 知道CEI模式的含义
- [ ] 理解ReentrancyGuard的实现原理
- [ ] 能识别代码中的重入风险

---

### **Day 37：整数溢出与下溢**

**学习目标**：
- 理解整数溢出的危害
- 掌握SafeMath的使用
- 分析Solidity 0.8的自动检查

**学习内容**：

1. **溢出漏洞示例**：
   ```solidity
   // ❌ Solidity < 0.8.0 不检查溢出
   pragma solidity ^0.7.0;

   contract VulnerableToken {
       mapping(address => uint256) public balances;

       function transfer(address to, uint256 amount) public {
           require(balances[msg.sender] - amount >= 0);  // ⚠️ 下溢检查无效！
           balances[msg.sender] -= amount;  // 下溢：1 - 2 = 2^256-1
           balances[to] += amount;
       }
   }

   // 攻击示例：
   // balances[attacker] = 1
   // transfer(victim, 2)
   // 结果：balances[attacker] = 2^256-1 (巨额代币！)
   ```

2. **SafeMath解决方案**：
   ```solidity
   // ✅ 使用SafeMath（Solidity < 0.8）
   pragma solidity ^0.7.0;

   library SafeMath {
       function add(uint256 a, uint256 b) internal pure returns (uint256) {
           uint256 c = a + b;
           require(c >= a, "SafeMath: addition overflow");
           return c;
       }

       function sub(uint256 a, uint256 b) internal pure returns (uint256) {
           require(b <= a, "SafeMath: subtraction overflow");
           uint256 c = a - b;
           return c;
       }

       function mul(uint256 a, uint256 b) internal pure returns (uint256) {
           if (a == 0) return 0;
           uint256 c = a * b;
           require(c / a == b, "SafeMath: multiplication overflow");
           return c;
       }
   }

   contract SecureToken {
       using SafeMath for uint256;
       mapping(address => uint256) public balances;

       function transfer(address to, uint256 amount) public {
           balances[msg.sender] = balances[msg.sender].sub(amount);  // 会revert
           balances[to] = balances[to].add(amount);
       }
   }
   ```

3. **Solidity 0.8+自动检查**：
   ```solidity
   // ✅ Solidity >= 0.8.0 默认检查溢出
   pragma solidity ^0.8.0;

   contract ModernToken {
       mapping(address => uint256) public balances;

       function transfer(address to, uint256 amount) public {
           balances[msg.sender] -= amount;  // 自动检查，溢出会revert
           balances[to] += amount;
       }

       // 如果确定不会溢出且需要节省Gas，可以使用unchecked
       function unsafeIncrement(uint256 x) public pure returns (uint256) {
           unchecked {
               return x + 1;  // 不检查溢出
           }
       }
   }
   ```

4. **真实案例：Beauty Chain (BEC)**：
   ```solidity
   // 2018年BEC溢出漏洞（价值数亿美元）
   function batchTransfer(address[] _receivers, uint256 _value) public {
       uint cnt = _receivers.length;
       uint256 amount = uint256(cnt) * _value;  // ⚠️ 溢出！
       require(cnt > 0 && cnt <= 20);
       require(_value > 0 && balances[msg.sender] >= amount);

       balances[msg.sender] = balances[msg.sender].sub(amount);
       for (uint i = 0; i < cnt; i++) {
           balances[_receivers[i]] = balances[_receivers[i]].add(_value);
       }
   }

   // 攻击：
   // _receivers = [addr1, addr2]  (cnt = 2)
   // _value = 2^255
   // amount = 2 * 2^255 = 2^256 = 0 (溢出！)
   // 绕过了 balances[msg.sender] >= amount 检查
   ```

**作业**：
- [ ] 复现整数溢出攻击
- [ ] 对比SafeMath和原生检查的Gas消耗
- [ ] 分析BEC漏洞的修复方案
- [ ] 学习其他算术漏洞（如类型转换）

---

### **Day 38-42：更多安全主题**

**Day 38：前端运行攻击（Front-Running）**
- MEV（Miner Extractable Value）
- Flashbots解决方案
- commit-reveal模式

**Day 39：闪电贷攻击（Flash Loan）**
- 价格预言机操纵
- 套利攻击案例分析
- 防御策略

**Day 40：访问控制漏洞**
- 未初始化的代理合约
- delegatecall安全性
- 函数可见性错误

**Day 41：拒绝服务攻击（DoS）**
- Gas limit DoS
- Revert DoS
- 区块Gas limit攻击

**Day 42：周末项目 - 安全审计工具**
开发一个简单的Solidity静态分析工具，检测：
- 重入漏洞
- 整数溢出
- 未检查的外部调用
- 访问控制问题

---

### **Day 43-49：协议层安全**

**Day 43-44：共识攻击**
- 51%攻击
- Long-range攻击
- Selfish mining

**Day 45-46：网络层攻击**
- Eclipse攻击
- DDoS防御
- Sybil攻击

**Day 47-48：密码学安全**
- 签名重放攻击
- 随机数安全
- 零知识证明基础

**Day 49：周末总结**
- 整理安全检查清单
- 学习审计报告编写
- 练习CTF题目

---

## **阶段6：Cosmos SDK开发（第11周）**

### **Day 50-56：Cosmos生态系统**

**Day 50：Cosmos架构概览**
- Cosmos Hub和Zone
- IBC跨链通信
- Tendermint BFT

**Day 51-52：SDK模块开发**
- 创建自定义模块
- Keeper模式
- Msg和Query处理

**Day 53-54：IBC协议**
- 跨链数据包
- 轻客户端验证
- 中继器实现

**Day 55-56：周末项目 - Cosmos链**
使用Cosmos SDK开发一个自定义区块链应用

---

## **阶段7：综合项目与面试准备（第12周）**

### **Day 57-59：大型项目实战**

选择一个综合项目：
1. **DeFi协议**：实现一个简化版的Uniswap或Aave
2. **NFT市场**：包含铸造、交易、拍卖功能
3. **跨链桥**：实现两条链之间的资产跨链

**Day 60-62：面试准备**

整理所有面试问题答案，准备：
- 技术深度问题（EVM、共识、P2P）
- 系统设计问题（如何设计一个Layer2）
- 代码题（实现Merkle Tree、Gas计算等）
- 项目经验讲述

**Day 63：模拟面试**

找同行或导师进行模拟面试，覆盖：
- 简历讲解（15分钟）
- 技术问答（30分钟）
- 代码实现（30分钟）
- 系统设计（30分钟）
- 反向提问（15分钟）

---

## **学习资源汇总**

### **必读书籍**：
1. 《精通以太坊》- Andreas Antonopoulos
2. 《区块链技术指南》- 邹均等
3. 《Mastering Bitcoin》- Andreas Antonopoulos

### **在线资源**：
1. Go-Ethereum源码：https://github.com/ethereum/go-ethereum
2. Solidity文档：https://docs.soliditylang.org
3. Ethereum.org：https://ethereum.org/en/developers
4. Cosmos SDK文档：https://docs.cosmos.network

### **练习平台**：
1. Ethernaut（智能合约CTF）：https://ethernaut.openzeppelin.com
2. Damn Vulnerable DeFi：https://www.damnvulnerabledefi.xyz
3. CryptoZombies：https://cryptozombies.io

---

## **总结与建议**

恭喜你完成了这份详尽的12周学习计划！以下是一些最后的建议：

### **持续学习**：
1. 订阅区块链技术周报（如Week in Ethereum News）
2. 参加线上/线下的区块链技术meetup
3. 贡献开源项目（go-ethereum, OpenZeppelin等）
4. 关注最新的EIP提案

### **面试技巧**：
1. **准备项目故事**：详细讲述你在项目中遇到的技术挑战和解决方案
2. **深度而非广度**：对简历上的每个技术点都要能深入讲解
3. **代码能力**：确保能在白板上写出常见算法（Merkle Tree, ECDSA验证等）
4. **系统思维**：能够从全局视角分析区块链系统的设计权衡

### **薪资谈判**：
对于30-50K的岗位：
- 展示你对go-ethereum源码的深入理解
- 强调你在性能优化、安全审计方面的经验
- 准备一个impressive的个人项目（如自己实现的Layer2或DeFi协议）

### **职业发展**：
1. **初级（1-2年）**：精通一条公链（以太坊/Cosmos），能独立开发DApp
2. **中级（3-5年）**：理解多条公链架构，能优化协议层性能
3. **高级（5-10年）**：主导区块链系统设计，参与核心协议研发

祝你面试成功，早日拿到心仪的offer！🚀

---

**学习进度跟踪表**：

| 周次 | 主题 | 状态 | 完成日期 |
|------|------|------|----------|
| Week 1 | 区块链基础 | ⬜ |  |
| Week 2 | 以太坊架构 | ⬜ |  |
| Week 3-4 | Go-Ethereum源码 | ⬜ |  |
| Week 5-6 | P2P与共识 | ⬜ |  |
| Week 7-8 | 存储与性能 | ⬜ |  |
| Week 9-10 | 安全 | ⬜ |  |
| Week 11 | Cosmos SDK | ⬜ |  |
| Week 12 | 项目与面试 | ⬜ |  |

**面试通过率目标**：≥95% ✅

如果你完成了这份计划，你将具备：
- ✅ 深入的区块链底层知识
- ✅ go-ethereum源码级别的理解
- ✅ 智能合约安全专家级能力
- ✅ 大型项目实战经验
- ✅ 出色的面试表现能力

开始你的学习之旅吧！💪
