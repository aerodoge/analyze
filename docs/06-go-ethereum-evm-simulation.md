# Go-Ethereum EVM 详解（六）：EVM 模拟与本地执行

## 目录

- [42. EVM 模拟基础](#42-evm-模拟基础)
- [43. eth_call 详解](#43-eth_call-详解)
- [44. 本地 EVM 执行](#44-本地-evm-执行)
- [45. 状态 Fork](#45-状态-fork)
- [46. 交易模拟](#46-交易模拟)
- [47. 批量模拟优化](#47-批量模拟优化)
- [48. 实战：构建模拟引擎](#48-实战构建模拟引擎)

---

## 42. EVM 模拟基础

### 42.1 什么是 EVM 模拟？

```
┌─────────────────────────────────────────────────────────────────┐
│                     EVM 模拟概念                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EVM 模拟 = 在不实际上链的情况下执行交易/调用                    │
│                                                                 │
│  用途：                                                         │
│  1. 查询合约状态（view/pure 函数）                              │
│  2. 预估 Gas 消耗                                               │
│  3. 验证交易是否会成功                                          │
│  4. 模拟复杂交易序列（MEV）                                     │
│  5. 调试和测试                                                  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    模拟 vs 实际执行                       │   │
│  │                                                          │   │
│  │  实际执行：                                              │   │
│  │  交易 ──▶ TxPool ──▶ 打包 ──▶ 状态永久改变              │   │
│  │                                                          │   │
│  │  模拟执行：                                              │   │
│  │  调用 ──▶ 临时状态 ──▶ 返回结果 ──▶ 丢弃状态            │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 42.2 模拟的重要性（MEV 视角）

```
┌─────────────────────────────────────────────────────────────────┐
│                 模拟对 MEV 的重要性                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  为什么 Searcher 需要模拟？                                     │
│                                                                 │
│  1. 验证机会有效性                                              │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  发现套利机会 ──▶ 模拟执行 ──▶ 确认利润 ──▶ 提交    │    │
│     │                      │                               │    │
│     │                      └──▶ 失败/亏损 ──▶ 放弃        │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  2. 计算精确利润                                                │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  模拟前：预估利润 = $100                             │    │
│     │  模拟后：实际利润 = $85（扣除 Gas、滑点）            │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  3. 优化参数                                                    │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  二分搜索最优输入量：                                │    │
│     │  模拟(100 ETH) → $50 利润                           │    │
│     │  模拟(200 ETH) → $80 利润                           │    │
│     │  模拟(150 ETH) → $90 利润 ← 最优                    │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
│  4. 避免链上失败                                                │
│     ┌─────────────────────────────────────────────────────┐    │
│     │  链上失败 = 损失 Gas 费 + 暴露策略                   │    │
│     │  模拟失败 = 零成本                                   │    │
│     └─────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 42.3 模拟方法对比

```
┌─────────────────────────────────────────────────────────────────┐
│                    模拟方法对比                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  方法           优点                  缺点                      │
│  ─────────────────────────────────────────────────────────────  │
│  eth_call       简单、标准           依赖节点、较慢             │
│                                                                 │
│  本地 EVM       快速、可控           需要同步状态               │
│                                                                 │
│  Tenderly       功能丰富             收费、依赖第三方           │
│                                                                 │
│  Anvil/Hardhat  开发友好             非生产级性能               │
│                                                                 │
│  自建模拟器     最快、最灵活         开发成本高                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 43. eth_call 详解

### 43.1 eth_call 基础用法

```go
// eth_call 用于执行消息调用（不创建交易）
type CallMsg struct {
From       common.Address // 发送者（可选）
To         *common.Address  // 目标地址
Gas        uint64           // Gas 限制（可选）
GasPrice   *big.Int         // Gas 价格（可选）
GasFeeCap  *big.Int // EIP-1559 最大费用（可选）
GasTipCap  *big.Int // EIP-1559 小费（可选）
Value      *big.Int // 发送的 ETH（可选）
Data       []byte           // 调用数据
AccessList types.AccessList // 访问列表（可选）
}

// 基本用法：调用合约 view 函数
func CallContract(client *ethclient.Client, contractAddr common.Address) ([]byte, error) {
// 编码调用数据
abi, _ := abi.JSON(strings.NewReader(ContractABI))
data, _ := abi.Pack("balanceOf", common.HexToAddress("0x123..."))

// 构造调用消息
msg := ethereum.CallMsg{
To:   &contractAddr,
Data: data,
}

// 执行调用
result, err := client.CallContract(context.Background(), msg, nil) // nil = latest block
if err != nil {
return nil, err
}

return result, nil
}
```

### 43.2 指定区块状态

```go
// 在特定区块状态下调用
func CallAtBlock(client *ethclient.Client, msg ethereum.CallMsg, blockNumber *big.Int) ([]byte, error) {
return client.CallContract(context.Background(), msg, blockNumber)
}

// 在特定区块哈希状态下调用（更精确）
func CallAtBlockHash(client *rpc.Client, msg ethereum.CallMsg, blockHash common.Hash) ([]byte, error) {
var result hexutil.Bytes
err := client.Call(&result, "eth_call", toCallArg(msg), map[string]interface{}{
"blockHash": blockHash,
})
return result, err
}

// 使用 pending 状态（包含 TxPool 中的交易效果）
func CallAtPending(client *ethclient.Client, msg ethereum.CallMsg) ([]byte, error) {
return client.PendingCallContract(context.Background(), msg)
}
```

### 43.3 状态覆盖（State Override）

```go
// StateOverride 允许在调用前修改账户状态
type StateOverride map[common.Address]OverrideAccount

type OverrideAccount struct {
Nonce     *hexutil.Uint64              `json:"nonce,omitempty"`
Code      *hexutil.Bytes               `json:"code,omitempty"`
Balance   *hexutil.Big                 `json:"balance,omitempty"`
State     map[common.Hash]common.Hash  `json:"state,omitempty"` // 完全替换
StateDiff map[common.Hash]common.Hash  `json:"stateDiff,omitempty"` // 部分修改
}

// 示例：模拟拥有大量 ETH
func CallWithOverride(client *rpc.Client, msg ethereum.CallMsg) ([]byte, error) {
override := StateOverride{
msg.From: {
Balance: (*hexutil.Big)(big.NewInt(1e18 * 1000)), // 1000 ETH
},
}

var result hexutil.Bytes
err := client.Call(&result, "eth_call", toCallArg(msg), "latest", override)
return result, err
}

// 示例：注入自定义代码
func CallWithCodeOverride(client *rpc.Client, targetAddr common.Address, customCode []byte) ([]byte, error) {
override := StateOverride{
targetAddr: {
Code: (*hexutil.Bytes)(&customCode),
},
}

msg := ethereum.CallMsg{
To:   &targetAddr,
Data: []byte{}, // 调用 fallback
}

var result hexutil.Bytes
err := client.Call(&result, "eth_call", toCallArg(msg), "latest", override)
return result, err
}
```

### 43.4 eth_estimateGas

```go
// EstimateGas 估算 Gas 消耗
func EstimateGas(client *ethclient.Client, msg ethereum.CallMsg) (uint64, error) {
return client.EstimateGas(context.Background(), msg)
}

// 带状态覆盖的 Gas 估算
func EstimateGasWithOverride(client *rpc.Client, msg ethereum.CallMsg, override StateOverride) (uint64, error) {
var result hexutil.Uint64
err := client.Call(&result, "eth_estimateGas", toCallArg(msg), "latest", override)
return uint64(result), err
}
```

---

## 44. 本地 EVM 执行

### 44.1 构建本地 EVM

```go
package localvm

import (
	"math/big"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core"
	"github.com/ethereum/go-ethereum/core/state"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/core/vm"
	"github.com/ethereum/go-ethereum/params"
)

// LocalEVM 本地 EVM 执行器
type LocalEVM struct {
	chainConfig *params.ChainConfig
	vmConfig    vm.Config
}

// NewLocalEVM 创建本地 EVM
func NewLocalEVM() *LocalEVM {
	return &LocalEVM{
		chainConfig: params.MainnetChainConfig,
		vmConfig: vm.Config{
			// 可选配置
			// Tracer: tracer,  // 调试追踪
			// NoBaseFee: true, // 忽略基础费检查
		},
	}
}

// Execute 执行消息调用
func (l *LocalEVM) Execute(
	stateDB *state.StateDB,
	header *types.Header,
	msg *core.Message,
) (*core.ExecutionResult, error) {

	// 创建区块上下文
	blockContext := core.NewEVMBlockContext(header, nil, &header.Coinbase)

	// 创建交易上下文
	txContext := core.NewEVMTxContext(msg)

	// 创建 EVM 实例
	evm := vm.NewEVM(blockContext, txContext, stateDB, l.chainConfig, l.vmConfig)

	// 设置 Gas 池
	gp := new(core.GasPool).AddGas(header.GasLimit)

	// 执行消息
	result, err := core.ApplyMessage(evm, msg, gp)
	if err != nil {
		return nil, err
	}

	return result, nil
}

// Call 执行只读调用（不修改状态）
func (l *LocalEVM) Call(
	stateDB *state.StateDB,
	header *types.Header,
	from common.Address,
	to common.Address,
	data []byte,
	gas uint64,
	value *big.Int,
) ([]byte, uint64, error) {

	// 创建快照
	snapshot := stateDB.Snapshot()
	defer stateDB.RevertToSnapshot(snapshot)

	msg := &core.Message{
		From:      from,
		To:        &to,
		GasLimit:  gas,
		GasPrice:  big.NewInt(0),
		GasFeeCap: big.NewInt(0),
		GasTipCap: big.NewInt(0),
		Value:     value,
		Data:      data,
	}

	result, err := l.Execute(stateDB, header, msg)
	if err != nil {
		return nil, 0, err
	}

	if result.Err != nil {
		return nil, result.UsedGas, result.Err
	}

	return result.ReturnData, result.UsedGas, nil
}
```

### 44.2 内存状态数据库

```go
// MemoryStateDB 内存中的状态数据库
type MemoryStateDB struct {
accounts map[common.Address]*Account
codes    map[common.Hash][]byte
}

type Account struct {
Nonce    uint64
Balance  *big.Int
Storage  map[common.Hash]common.Hash
CodeHash common.Hash
}

// NewMemoryStateDB 创建内存状态数据库
func NewMemoryStateDB() *MemoryStateDB {
return &MemoryStateDB{
accounts: make(map[common.Address]*Account),
codes:    make(map[common.Hash][]byte),
}
}

// SetAccount 设置账户
func (m *MemoryStateDB) SetAccount(addr common.Address, balance *big.Int, code []byte) {
codeHash := crypto.Keccak256Hash(code)
m.accounts[addr] = &Account{
Nonce:    0,
Balance:  balance,
Storage:  make(map[common.Hash]common.Hash),
CodeHash: codeHash,
}
if len(code) > 0 {
m.codes[codeHash] = code
}
}

// 实现 vm.StateDB 接口的关键方法
func (m *MemoryStateDB) GetBalance(addr common.Address) *uint256.Int {
if acc, ok := m.accounts[addr]; ok {
return uint256.MustFromBig(acc.Balance)
}
return uint256.NewInt(0)
}

func (m *MemoryStateDB) GetCode(addr common.Address) []byte {
if acc, ok := m.accounts[addr]; ok {
return m.codes[acc.CodeHash]
}
return nil
}

func (m *MemoryStateDB) GetState(addr common.Address, key common.Hash) common.Hash {
if acc, ok := m.accounts[addr]; ok {
return acc.Storage[key]
}
return common.Hash{}
}

func (m *MemoryStateDB) SetState(addr common.Address, key, value common.Hash) {
if acc, ok := m.accounts[addr]; ok {
acc.Storage[key] = value
}
}
```

### 44.3 实用工具函数

```go
// SimulateSwap 模拟 DEX swap
func SimulateSwap(
stateDB *state.StateDB,
header *types.Header,
router common.Address,
tokenIn, tokenOut common.Address,
amountIn *big.Int,
from common.Address,
) (*big.Int, error) {

localEVM := NewLocalEVM()

// 编码 swap 调用
routerABI, _ := abi.JSON(strings.NewReader(UniswapV2RouterABI))
path := []common.Address{tokenIn, tokenOut}
deadline := big.NewInt(time.Now().Add(time.Hour).Unix())

data, _ := routerABI.Pack(
"swapExactTokensForTokens",
amountIn,
big.NewInt(0), // amountOutMin
path,
from,
deadline,
)

// 执行调用
result, _, err := localEVM.Call(
stateDB,
header,
from,
router,
data,
1000000, // gas
big.NewInt(0),
)
if err != nil {
return nil, err
}

// 解码返回值
outputs, _ := routerABI.Unpack("swapExactTokensForTokens", result)
amounts := outputs[0].([]*big.Int)

return amounts[len(amounts)-1], nil // 最后一个是输出量
}

// GetPoolReserves 获取池子储备量
func GetPoolReserves(
stateDB *state.StateDB,
header *types.Header,
pool common.Address,
) (*big.Int, *big.Int, error) {

localEVM := NewLocalEVM()

pairABI, _ := abi.JSON(strings.NewReader(UniswapV2PairABI))
data, _ := pairABI.Pack("getReserves")

result, _, err := localEVM.Call(
stateDB,
header,
common.Address{}, // from 不重要
pool,
data,
100000,
big.NewInt(0),
)
if err != nil {
return nil, nil, err
}

outputs, _ := pairABI.Unpack("getReserves", result)
reserve0 := outputs[0].(*big.Int)
reserve1 := outputs[1].(*big.Int)

return reserve0, reserve1, nil
}
```

---

## 45. 状态 Fork

### 45.1 Fork 概念

```
┌─────────────────────────────────────────────────────────────────┐
│                      状态 Fork                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Fork = 复制主网状态到本地，在本地进行模拟                       │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    主网状态                              │   │
│  │  Block #18000000                                        │   │
│  │  ┌─────────────────────────────────────────┐           │   │
│  │  │  Account A: 100 ETH                     │           │   │
│  │  │  Account B: 50 ETH                      │           │   │
│  │  │  Contract C: [code...]                  │           │   │
│  │  │  Pool D: reserve0=1000, reserve1=2000   │           │   │
│  │  └─────────────────────────────────────────┘           │   │
│  └──────────────────────┬──────────────────────────────────┘   │
│                         │                                       │
│                         │ Fork                                  │
│                         ▼                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    本地 Fork                             │   │
│  │  ┌─────────────────────────────────────────┐           │   │
│  │  │  与主网相同的初始状态                    │           │   │
│  │  │  可以在本地任意修改和模拟                │           │   │
│  │  │  不影响主网                              │           │   │
│  │  └─────────────────────────────────────────┘           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 45.2 实现状态 Fork

```go
// ForkState 从远程节点 fork 状态
type ForkState struct {
client      *ethclient.Client
blockNumber *big.Int

// 缓存
accounts    map[common.Address]*Account
codes       map[common.Hash][]byte
storage     map[common.Address]map[common.Hash]common.Hash

// 修改追踪
modified    map[common.Address]bool
mu          sync.RWMutex
}

// NewForkState 创建 Fork 状态
func NewForkState(rpcURL string, blockNumber *big.Int) (*ForkState, error) {
client, err := ethclient.Dial(rpcURL)
if err != nil {
return nil, err
}

return &ForkState{
client:      client,
blockNumber: blockNumber,
accounts:    make(map[common.Address]*Account),
codes:       make(map[common.Hash][]byte),
storage:     make(map[common.Address]map[common.Hash]common.Hash),
modified:    make(map[common.Address]bool),
}, nil
}

// GetBalance 获取余额（先查缓存，再查远程）
func (f *ForkState) GetBalance(addr common.Address) *uint256.Int {
f.mu.RLock()
if acc, ok := f.accounts[addr]; ok {
f.mu.RUnlock()
return uint256.MustFromBig(acc.Balance)
}
f.mu.RUnlock()

// 从远程获取
balance, err := f.client.BalanceAt(context.Background(), addr, f.blockNumber)
if err != nil {
return uint256.NewInt(0)
}

// 缓存
f.mu.Lock()
if f.accounts[addr] == nil {
f.accounts[addr] = &Account{Storage: make(map[common.Hash]common.Hash)}
}
f.accounts[addr].Balance = balance
f.mu.Unlock()

return uint256.MustFromBig(balance)
}

// GetCode 获取代码
func (f *ForkState) GetCode(addr common.Address) []byte {
f.mu.RLock()
if acc, ok := f.accounts[addr]; ok && acc.CodeHash != (common.Hash{}) {
code := f.codes[acc.CodeHash]
f.mu.RUnlock()
return code
}
f.mu.RUnlock()

// 从远程获取
code, err := f.client.CodeAt(context.Background(), addr, f.blockNumber)
if err != nil {
return nil
}

// 缓存
f.mu.Lock()
codeHash := crypto.Keccak256Hash(code)
if f.accounts[addr] == nil {
f.accounts[addr] = &Account{Storage: make(map[common.Hash]common.Hash)}
}
f.accounts[addr].CodeHash = codeHash
f.codes[codeHash] = code
f.mu.Unlock()

return code
}

// GetState 获取存储值
func (f *ForkState) GetState(addr common.Address, key common.Hash) common.Hash {
f.mu.RLock()
if storage, ok := f.storage[addr]; ok {
if val, ok := storage[key]; ok {
f.mu.RUnlock()
return val
}
}
f.mu.RUnlock()

// 从远程获取
val, err := f.client.StorageAt(context.Background(), addr, key, f.blockNumber)
if err != nil {
return common.Hash{}
}

// 缓存
f.mu.Lock()
if f.storage[addr] == nil {
f.storage[addr] = make(map[common.Hash]common.Hash)
}
f.storage[addr][key] = common.BytesToHash(val)
f.mu.Unlock()

return common.BytesToHash(val)
}

// SetState 设置存储值（只在本地生效）
func (f *ForkState) SetState(addr common.Address, key, value common.Hash) {
f.mu.Lock()
defer f.mu.Unlock()

if f.storage[addr] == nil {
f.storage[addr] = make(map[common.Hash]common.Hash)
}
f.storage[addr][key] = value
f.modified[addr] = true
}
```

### 45.3 批量预加载

```go
// PreloadAccounts 批量预加载账户状态
func (f *ForkState) PreloadAccounts(addresses []common.Address) error {
// 使用 Multicall 批量获取
multicall := common.HexToAddress("0xcA11bde05977b3631167028862bE2a173976CA11")

// 构建批量调用
calls := make([]MulticallCall, 0, len(addresses)*2)

for _, addr := range addresses {
// 余额调用
calls = append(calls, MulticallCall{
Target: multicall,
Data:   encodeGetEthBalance(addr),
})
// 代码哈希调用
calls = append(calls, MulticallCall{
Target: addr,
Data:   []byte{}, // 获取 codehash 通过 EXTCODEHASH
})
}

// 执行批量调用
results, err := f.executeMulticall(calls)
if err != nil {
return err
}

// 解析并缓存结果
f.mu.Lock()
defer f.mu.Unlock()

for i, addr := range addresses {
balance := new(big.Int).SetBytes(results[i*2])
if f.accounts[addr] == nil {
f.accounts[addr] = &Account{Storage: make(map[common.Hash]common.Hash)}
}
f.accounts[addr].Balance = balance
}

return nil
}

// PreloadPoolReserves 批量预加载池子储备量
func (f *ForkState) PreloadPoolReserves(pools []common.Address) (map[common.Address][2]*big.Int, error) {
multicall := common.HexToAddress("0xcA11bde05977b3631167028862bE2a173976CA11")

// 构建 getReserves 调用
pairABI, _ := abi.JSON(strings.NewReader(UniswapV2PairABI))
getReservesData, _ := pairABI.Pack("getReserves")

calls := make([]MulticallCall, len(pools))
for i, pool := range pools {
calls[i] = MulticallCall{
Target: pool,
Data:   getReservesData,
}
}

// 执行
results, err := f.executeMulticall(calls)
if err != nil {
return nil, err
}

// 解析
reserves := make(map[common.Address][2]*big.Int)
for i, pool := range pools {
outputs, _ := pairABI.Unpack("getReserves", results[i])
reserves[pool] = [2]*big.Int{
outputs[0].(*big.Int),
outputs[1].(*big.Int),
}
}

return reserves, nil
}
```

---

## 46. 交易模拟

### 46.1 单交易模拟

```go
// TransactionSimulator 交易模拟器
type TransactionSimulator struct {
forkState *ForkState
localEVM  *LocalEVM
header    *types.Header
}

// SimulateTransaction 模拟单个交易
func (s *TransactionSimulator) SimulateTransaction(tx *types.Transaction) (*SimulationResult, error) {
// 创建状态快照
snapshot := s.forkState.Snapshot()
defer s.forkState.RevertToSnapshot(snapshot)

// 获取发送者
from, err := types.Sender(types.LatestSignerForChainID(tx.ChainId()), tx)
if err != nil {
return nil, err
}

// 构造消息
msg := &core.Message{
From:      from,
To:        tx.To(),
GasLimit:  tx.Gas(),
GasPrice:  tx.GasPrice(),
GasFeeCap: tx.GasFeeCap(),
GasTipCap: tx.GasTipCap(),
Value:     tx.Value(),
Data:      tx.Data(),
Nonce:     tx.Nonce(),
}

// 执行
result, err := s.localEVM.Execute(s.forkState, s.header, msg)
if err != nil {
return nil, err
}

return &SimulationResult{
Success:    result.Err == nil,
GasUsed:    result.UsedGas,
ReturnData: result.ReturnData,
Error:      result.Err,
Logs:       s.forkState.GetLogs(tx.Hash()),
}, nil
}

type SimulationResult struct {
Success    bool
GasUsed    uint64
ReturnData []byte
Error      error
Logs       []*types.Log
}
```

### 46.2 交易序列模拟

```go
// SimulateTransactionSequence 模拟交易序列
func (s *TransactionSimulator) SimulateTransactionSequence(txs []*types.Transaction) ([]*SimulationResult, error) {
results := make([]*SimulationResult, len(txs))

// 创建快照
snapshot := s.forkState.Snapshot()
defer s.forkState.RevertToSnapshot(snapshot)

for i, tx := range txs {
from, _ := types.Sender(types.LatestSignerForChainID(tx.ChainId()), tx)

msg := &core.Message{
From:      from,
To:        tx.To(),
GasLimit:  tx.Gas(),
GasPrice:  tx.GasPrice(),
GasFeeCap: tx.GasFeeCap(),
GasTipCap: tx.GasTipCap(),
Value:     tx.Value(),
Data:      tx.Data(),
Nonce:     tx.Nonce(),
}

result, err := s.localEVM.Execute(s.forkState, s.header, msg)
if err != nil {
results[i] = &SimulationResult{Success: false, Error: err}
// 可以选择继续或停止
continue
}

results[i] = &SimulationResult{
Success:    result.Err == nil,
GasUsed:    result.UsedGas,
ReturnData: result.ReturnData,
Error:      result.Err,
}

// 状态已经被修改，下一个交易会看到这个效果
}

return results, nil
}

// SimulateBundle 模拟 Flashbots Bundle
func (s *TransactionSimulator) SimulateBundle(bundle *FlashbotsBundle) (*BundleSimulationResult, error) {
snapshot := s.forkState.Snapshot()
defer s.forkState.RevertToSnapshot(snapshot)

totalGasUsed := uint64(0)
txResults := make([]*SimulationResult, len(bundle.Txs))

for i, txBytes := range bundle.Txs {
var tx types.Transaction
if err := tx.UnmarshalBinary(txBytes); err != nil {
return nil, err
}

result, err := s.SimulateTransaction(&tx)
if err != nil {
return nil, err
}

txResults[i] = result
totalGasUsed += result.GasUsed

// Bundle 中如果有交易失败（且不在 revertingTxHashes 中），整个 Bundle 失败
if !result.Success && !contains(bundle.RevertingTxHashes, tx.Hash()) {
return &BundleSimulationResult{
Success:     false,
TxResults:   txResults[:i+1],
FailedIndex: i,
}, nil
}
}

return &BundleSimulationResult{
Success:      true,
TxResults:    txResults,
TotalGasUsed: totalGasUsed,
}, nil
}

type BundleSimulationResult struct {
Success      bool
TxResults    []*SimulationResult
TotalGasUsed uint64
FailedIndex  int
}
```

### 46.3 套利模拟

```go
// ArbitrageSimulator 套利模拟器
type ArbitrageSimulator struct {
simulator *TransactionSimulator
router    common.Address
}

// SimulateArbitrage 模拟套利交易
func (a *ArbitrageSimulator) SimulateArbitrage(
tokenIn, tokenOut common.Address,
amountIn *big.Int,
pools [2]common.Address,  // [buyPool, sellPool]
) (*ArbitrageResult, error) {

snapshot := a.simulator.forkState.Snapshot()
defer a.simulator.forkState.RevertToSnapshot(snapshot)

// 1. 在 buyPool 买入
buyOutput, err := a.simulateSwap(pools[0], tokenIn, tokenOut, amountIn)
if err != nil {
return nil, fmt.Errorf("buy failed: %w", err)
}

// 2. 在 sellPool 卖出
sellOutput, err := a.simulateSwap(pools[1], tokenOut, tokenIn, buyOutput)
if err != nil {
return nil, fmt.Errorf("sell failed: %w", err)
}

// 3. 计算利润
profit := new(big.Int).Sub(sellOutput, amountIn)

return &ArbitrageResult{
AmountIn:   amountIn,
BuyOutput:  buyOutput,
SellOutput: sellOutput,
Profit:     profit,
Profitable: profit.Sign() > 0,
}, nil
}

type ArbitrageResult struct {
AmountIn   *big.Int
BuyOutput  *big.Int
SellOutput *big.Int
Profit     *big.Int
Profitable bool
}

// FindOptimalAmount 二分搜索最优输入量
func (a *ArbitrageSimulator) FindOptimalAmount(
tokenIn, tokenOut common.Address,
pools [2]common.Address,
minAmount, maxAmount *big.Int,
) (*big.Int, *big.Int, error) {

bestAmount := big.NewInt(0)
bestProfit := big.NewInt(0)

// 二分搜索
low := new(big.Int).Set(minAmount)
high := new(big.Int).Set(maxAmount)

for i := 0; i < 100; i++ {
if low.Cmp(high) >= 0 {
break
}

mid := new(big.Int).Add(low, high)
mid.Div(mid, big.NewInt(2))

result, err := a.SimulateArbitrage(tokenIn, tokenOut, mid, pools)
if err != nil {
high = mid
continue
}

if result.Profit.Cmp(bestProfit) > 0 {
bestAmount.Set(mid)
bestProfit.Set(result.Profit)
}

// 检查 mid + delta 的利润来确定搜索方向
delta := new(big.Int).Div(new(big.Int).Sub(high, low), big.NewInt(10))
if delta.Cmp(big.NewInt(1e15)) < 0 {
delta = big.NewInt(1e15)
}

midPlus := new(big.Int).Add(mid, delta)
resultPlus, _ := a.SimulateArbitrage(tokenIn, tokenOut, midPlus, pools)

if resultPlus != nil && resultPlus.Profit.Cmp(result.Profit) > 0 {
low = mid
} else {
high = mid
}
}

return bestAmount, bestProfit, nil
}
```

---

## 47. 批量模拟优化

### 47.1 并行模拟

```go
// ParallelSimulator 并行模拟器
type ParallelSimulator struct {
baseState *ForkState
workers   int
}

// SimulateParallel 并行模拟多个独立场景
func (p *ParallelSimulator) SimulateParallel(scenarios []SimulationScenario) []SimulationResult {
results := make([]SimulationResult, len(scenarios))
sem := make(chan struct{}, p.workers)
var wg sync.WaitGroup

for i, scenario := range scenarios {
wg.Add(1)
go func (idx int, s SimulationScenario) {
defer wg.Done()
sem <- struct{}{}  // 获取信号量
defer func () { <-sem }() // 释放信号量

// 每个 goroutine 使用独立的状态副本
stateCopy := p.baseState.Copy()
simulator := NewTransactionSimulator(stateCopy)

result, _ := simulator.SimulateTransaction(s.Tx)
results[idx] = *result
}(i, scenario)
}

wg.Wait()
return results
}

type SimulationScenario struct {
Tx          *types.Transaction
Description string
}
```

### 47.2 增量状态

```go
// IncrementalState 增量状态（减少状态复制开销）
type IncrementalState struct {
parent    *IncrementalState // 父状态
changes   map[common.Address]*AccountChange // 本次修改
}

type AccountChange struct {
Balance *big.Int
Nonce   uint64
Code    []byte
Storage map[common.Hash]common.Hash
}

// Get 获取值（先查本地修改，再查父状态）
func (s *IncrementalState) GetBalance(addr common.Address) *big.Int {
if change, ok := s.changes[addr]; ok && change.Balance != nil {
return change.Balance
}
if s.parent != nil {
return s.parent.GetBalance(addr)
}
return big.NewInt(0)
}

// Fork 创建子状态
func (s *IncrementalState) Fork() *IncrementalState {
return &IncrementalState{
parent:  s,
changes: make(map[common.Address]*AccountChange),
}
}

// Commit 提交修改到父状态
func (s *IncrementalState) Commit() {
if s.parent == nil {
return
}
for addr, change := range s.changes {
if s.parent.changes[addr] == nil {
s.parent.changes[addr] = &AccountChange{
Storage: make(map[common.Hash]common.Hash),
}
}
if change.Balance != nil {
s.parent.changes[addr].Balance = change.Balance
}
if change.Code != nil {
s.parent.changes[addr].Code = change.Code
}
for k, v := range change.Storage {
s.parent.changes[addr].Storage[k] = v
}
}
}
```

### 47.3 缓存优化

```go
// CachedSimulator 带缓存的模拟器
type CachedSimulator struct {
simulator *TransactionSimulator

// 结果缓存
resultCache *lru.Cache // hash -> result

// 状态缓存
balanceCache map[common.Address]*big.Int
codeCache    map[common.Address][]byte
storageCache map[common.Address]map[common.Hash]common.Hash

mu sync.RWMutex
}

// SimulateWithCache 带缓存的模拟
func (c *CachedSimulator) SimulateWithCache(tx *types.Transaction) (*SimulationResult, error) {
// 检查缓存
cacheKey := c.computeCacheKey(tx)
if cached, ok := c.resultCache.Get(cacheKey); ok {
return cached.(*SimulationResult), nil
}

// 执行模拟
result, err := c.simulator.SimulateTransaction(tx)
if err != nil {
return nil, err
}

// 缓存结果
c.resultCache.Add(cacheKey, result)

return result, nil
}

// computeCacheKey 计算缓存键
func (c *CachedSimulator) computeCacheKey(tx *types.Transaction) string {
// 缓存键 = 交易哈希 + 区块号
// 注意：状态变化会使缓存失效
return fmt.Sprintf("%s-%d", tx.Hash().Hex(), c.simulator.header.Number.Uint64())
}

// InvalidateCache 使缓存失效
func (c *CachedSimulator) InvalidateCache() {
c.resultCache.Purge()
c.mu.Lock()
c.balanceCache = make(map[common.Address]*big.Int)
c.codeCache = make(map[common.Address][]byte)
c.storageCache = make(map[common.Address]map[common.Hash]common.Hash)
c.mu.Unlock()
}
```

---

## 48. 实战：构建模拟引擎

### 48.1 完整模拟引擎

```go
package simulation

import (
	"context"
	"math/big"
	"sync"

	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/ethclient"
)

// SimulationEngine 模拟引擎
type SimulationEngine struct {
	client    *ethclient.Client
	forkState *ForkState
	localEVM  *LocalEVM
	header    *types.Header

	// 配置
	maxWorkers int
	cacheSize  int

	// 缓存
	poolCache  map[common.Address]*PoolInfo
	priceCache map[string]*big.Int

	mu sync.RWMutex
}

type PoolInfo struct {
	Address  common.Address
	Token0   common.Address
	Token1   common.Address
	Reserve0 *big.Int
	Reserve1 *big.Int
	Fee      uint64
}

// NewSimulationEngine 创建模拟引擎
func NewSimulationEngine(rpcURL string) (*SimulationEngine, error) {
	client, err := ethclient.Dial(rpcURL)
	if err != nil {
		return nil, err
	}

	// 获取最新区块
	header, err := client.HeaderByNumber(context.Background(), nil)
	if err != nil {
		return nil, err
	}

	// 创建 fork 状态
	forkState, err := NewForkState(rpcURL, header.Number)
	if err != nil {
		return nil, err
	}

	return &SimulationEngine{
		client:     client,
		forkState:  forkState,
		localEVM:   NewLocalEVM(),
		header:     header,
		maxWorkers: 10,
		cacheSize:  10000,
		poolCache:  make(map[common.Address]*PoolInfo),
		priceCache: make(map[string]*big.Int),
	}, nil
}

// UpdateState 更新到最新状态
func (e *SimulationEngine) UpdateState(ctx context.Context) error {
	header, err := e.client.HeaderByNumber(ctx, nil)
	if err != nil {
		return err
	}

	e.mu.Lock()
	defer e.mu.Unlock()

	e.header = header
	e.forkState, _ = NewForkState(e.client.Client().URL(), header.Number)

	// 清空缓存
	e.poolCache = make(map[common.Address]*PoolInfo)
	e.priceCache = make(map[string]*big.Int)

	return nil
}

// SimulateArbitrage 模拟套利
func (e *SimulationEngine) SimulateArbitrage(
	tokenA, tokenB common.Address,
	pools []common.Address,
	amount *big.Int,
) (*ArbitrageSimulationResult, error) {

	e.mu.RLock()
	defer e.mu.RUnlock()

	// 创建状态快照
	snapshot := e.forkState.Snapshot()
	defer e.forkState.RevertToSnapshot(snapshot)

	// 更新池子储备
	poolInfos := make([]*PoolInfo, len(pools))
	for i, pool := range pools {
		info, err := e.getPoolInfo(pool)
		if err != nil {
			return nil, err
		}
		poolInfos[i] = info
	}

	// 模拟交换路径
	currentAmount := new(big.Int).Set(amount)
	currentToken := tokenA

	for i, poolInfo := range poolInfos {
		var outputAmount *big.Int
		var outputToken common.Address

		if currentToken == poolInfo.Token0 {
			outputAmount = e.calculateOutput(currentAmount, poolInfo.Reserve0, poolInfo.Reserve1, poolInfo.Fee)
			outputToken = poolInfo.Token1
		} else {
			outputAmount = e.calculateOutput(currentAmount, poolInfo.Reserve1, poolInfo.Reserve0, poolInfo.Fee)
			outputToken = poolInfo.Token0
		}

		// 更新储备量（模拟状态变化）
		e.updatePoolReserves(pools[i], currentToken, currentAmount, outputToken, outputAmount)

		currentAmount = outputAmount
		currentToken = outputToken
	}

	profit := new(big.Int).Sub(currentAmount, amount)

	return &ArbitrageSimulationResult{
		InputAmount:  amount,
		OutputAmount: currentAmount,
		Profit:       profit,
		Profitable:   profit.Sign() > 0,
		Path:         pools,
	}, nil
}

type ArbitrageSimulationResult struct {
	InputAmount  *big.Int
	OutputAmount *big.Int
	Profit       *big.Int
	Profitable   bool
	Path         []common.Address
}

// getPoolInfo 获取池子信息（带缓存）
func (e *SimulationEngine) getPoolInfo(pool common.Address) (*PoolInfo, error) {
	if info, ok := e.poolCache[pool]; ok {
		return info, nil
	}

	// 从链上获取
	reserve0, reserve1, err := GetPoolReserves(e.forkState, e.header, pool)
	if err != nil {
		return nil, err
	}

	token0, token1, err := e.getPoolTokens(pool)
	if err != nil {
		return nil, err
	}

	info := &PoolInfo{
		Address:  pool,
		Token0:   token0,
		Token1:   token1,
		Reserve0: reserve0,
		Reserve1: reserve1,
		Fee:      30, // 0.3%
	}

	e.poolCache[pool] = info
	return info, nil
}

// calculateOutput 计算输出量
func (e *SimulationEngine) calculateOutput(amountIn, reserveIn, reserveOut *big.Int, feeBps uint64) *big.Int {
	// amountOut = (amountIn * (10000 - fee) * reserveOut) / (reserveIn * 10000 + amountIn * (10000 - fee))
	feeMultiplier := big.NewInt(int64(10000 - feeBps))

	amountInWithFee := new(big.Int).Mul(amountIn, feeMultiplier)
	numerator := new(big.Int).Mul(amountInWithFee, reserveOut)
	denominator := new(big.Int).Mul(reserveIn, big.NewInt(10000))
	denominator.Add(denominator, amountInWithFee)

	return new(big.Int).Div(numerator, denominator)
}

// FindOptimalArbitrage 寻找最优套利参数
func (e *SimulationEngine) FindOptimalArbitrage(
	tokenA, tokenB common.Address,
	pools []common.Address,
) (*OptimalArbitrageResult, error) {

	// 二分搜索最优输入量
	minAmount := big.NewInt(1e15)   // 0.001 ETH
	maxAmount := big.NewInt(100e18) // 100 ETH

	bestAmount := big.NewInt(0)
	bestProfit := big.NewInt(0)

	for i := 0; i < 50; i++ {
		if minAmount.Cmp(maxAmount) >= 0 {
			break
		}

		mid := new(big.Int).Add(minAmount, maxAmount)
		mid.Div(mid, big.NewInt(2))

		result, err := e.SimulateArbitrage(tokenA, tokenB, pools, mid)
		if err != nil || !result.Profitable {
			maxAmount = mid
			continue
		}

		if result.Profit.Cmp(bestProfit) > 0 {
			bestAmount.Set(mid)
			bestProfit.Set(result.Profit)
		}

		// 检查方向
		midPlus := new(big.Int).Add(mid, big.NewInt(1e16))
		resultPlus, _ := e.SimulateArbitrage(tokenA, tokenB, pools, midPlus)

		if resultPlus != nil && resultPlus.Profit.Cmp(result.Profit) > 0 {
			minAmount = mid
		} else {
			maxAmount = mid
		}
	}

	if bestProfit.Sign() <= 0 {
		return nil, fmt.Errorf("no profitable arbitrage found")
	}

	return &OptimalArbitrageResult{
		OptimalAmount:  bestAmount,
		ExpectedProfit: bestProfit,
		Path:           pools,
	}, nil
}

type OptimalArbitrageResult struct {
	OptimalAmount  *big.Int
	ExpectedProfit *big.Int
	Path           []common.Address
}
```

### 48.2 使用示例

```go
package main

import (
	"fmt"
	"math/big"

	"github.com/ethereum/go-ethereum/common"
)

func main() {
	// 创建模拟引擎
	engine, err := NewSimulationEngine("http://localhost:8545")
	if err != nil {
		panic(err)
	}

	// 定义代币和池子
	WETH := common.HexToAddress("0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2")
	USDC := common.HexToAddress("0xA0b86a6E379B72e1c4E28e6B4E4C0B8fE8e78e8F")

	uniswapPool := common.HexToAddress("0x...")
	sushiPool := common.HexToAddress("0x...")

	// 模拟套利
	result, err := engine.SimulateArbitrage(
		WETH, USDC,
		[]common.Address{uniswapPool, sushiPool},
		big.NewInt(1e18), // 1 ETH
	)
	if err != nil {
		fmt.Printf("Simulation failed: %v\n", err)
		return
	}

	fmt.Printf("Input: %s ETH\n", weiToEth(result.InputAmount))
	fmt.Printf("Output: %s ETH\n", weiToEth(result.OutputAmount))
	fmt.Printf("Profit: %s ETH\n", weiToEth(result.Profit))
	fmt.Printf("Profitable: %v\n", result.Profitable)

	// 寻找最优套利
	optimal, err := engine.FindOptimalArbitrage(WETH, USDC, []common.Address{uniswapPool, sushiPool})
	if err != nil {
		fmt.Printf("No profitable arbitrage: %v\n", err)
		return
	}

	fmt.Printf("\nOptimal Arbitrage:\n")
	fmt.Printf("Amount: %s ETH\n", weiToEth(optimal.OptimalAmount))
	fmt.Printf("Expected Profit: %s ETH\n", weiToEth(optimal.ExpectedProfit))
}

func weiToEth(wei *big.Int) string {
	eth := new(big.Float).SetInt(wei)
	eth.Quo(eth, new(big.Float).SetInt64(1e18))
	return eth.Text('f', 6)
}
```

---

## 总结

本文档详细介绍了 EVM 模拟与本地执行：

1. **模拟基础**：概念、重要性、方法对比
2. **eth_call**：基础用法、状态覆盖、Gas 估算
3. **本地 EVM**：构建执行器、内存状态数据库
4. **状态 Fork**：从主网复制状态、批量预加载
5. **交易模拟**：单交易、序列、Bundle 模拟
6. **批量优化**：并行模拟、增量状态、缓存
7. **实战引擎**：完整的模拟引擎实现

下一篇将介绍 L2 EVM 变体（Arbitrum、Optimism 等）。
