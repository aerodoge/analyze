# Go-Ethereum EVM 详解（七）：L2 EVM 变体

## 目录

- [49. L2 概述](#49-l2-概述)
- [50. Optimistic Rollups](#50-optimistic-rollups)
- [51. ZK Rollups](#51-zk-rollups)
- [52. Arbitrum 详解](#52-arbitrum-详解)
- [53. Optimism 详解](#53-optimism-详解)
- [54. Base 与 OP Stack](#54-base-与-op-stack)
- [55. L2 MEV 特性](#55-l2-mev-特性)
- [56. 跨 L2 套利](#56-跨-l2-套利)

---

## 49. L2 概述

### 49.1 为什么需要 L2？

```
┌─────────────────────────────────────────────────────────────────┐
│                     L1 扩容困境                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  以太坊 L1 限制：                                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 区块 Gas 限制：~30M Gas/区块                         │   │
│  │  • 出块时间：12 秒                                       │   │
│  │  • TPS：~15-30 笔交易/秒                                │   │
│  │  • Gas 费用：高峰期 $50-100+                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  扩容三角困境：                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    安全性                                │   │
│  │                      /\                                  │   │
│  │                     /  \                                 │   │
│  │                    /    \                                │   │
│  │                   /  L1  \                               │   │
│  │                  /________\                              │   │
│  │            去中心化        可扩展性                       │   │
│  │                                                          │   │
│  │  L1 选择了安全性和去中心化，牺牲了可扩展性               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  L2 解决方案：                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 在 L2 执行交易（高 TPS）                             │   │
│  │  • 将数据/证明发布到 L1（继承安全性）                   │   │
│  │  • Gas 费用降低 10-100 倍                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 49.2 L2 类型对比

```
┌─────────────────────────────────────────────────────────────────┐
│                      L2 类型对比                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  类型              代表项目          特点                       │
│  ─────────────────────────────────────────────────────────────  │
│  Optimistic       Arbitrum          欺诈证明                    │
│  Rollup           Optimism          7天挑战期                   │
│                   Base              EVM 等价                    │
│                                                                 │
│  ZK Rollup        zkSync Era        有效性证明                  │
│                   Starknet          即时终局                    │
│                   Polygon zkEVM     数学保证                    │
│                   Linea                                         │
│                   Scroll                                        │
│                                                                 │
│  Validium         Immutable X       数据在链下                  │
│                   zkPorter          更低成本                    │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  EVM 兼容性：                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Type 1: EVM 等价（完全兼容）- Scroll, Taiko            │   │
│  │  Type 2: EVM 等价（细微差异）- Polygon zkEVM            │   │
│  │  Type 3: EVM 兼容（某些操作码不同）- zkSync Era         │   │
│  │  Type 4: 类 EVM（高级语言兼容）- Starknet               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 49.3 L2 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                    通用 L2 架构                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户                                                           │
│    │                                                            │
│    │ 发送交易                                                   │
│    ▼                                                            │
│  ┌─────────────┐                                                │
│  │  Sequencer  │  排序器：收集并排序交易                        │
│  │   （排序）   │  通常是中心化的（去中心化是发展方向）          │
│  └──────┬──────┘                                                │
│         │                                                       │
│         ▼                                                       │
│  ┌─────────────┐                                                │
│  │   L2 EVM    │  执行层：执行交易，更新状态                    │
│  │   （执行）   │                                                │
│  └──────┬──────┘                                                │
│         │                                                       │
│         ├────────────────────────────┐                          │
│         │                            │                          │
│         ▼                            ▼                          │
│  ┌─────────────┐              ┌─────────────┐                   │
│  │    Batch    │              │   Proof     │                   │
│  │  （批次）    │              │  （证明）    │                   │
│  │ 打包多个交易 │              │ 欺诈/有效性  │                   │
│  └──────┬──────┘              └──────┬──────┘                   │
│         │                            │                          │
│         └────────────┬───────────────┘                          │
│                      │                                          │
│                      ▼                                          │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    Ethereum L1                            │ │
│  │  • 数据可用性（Calldata / Blob）                          │ │
│  │  • 状态根                                                 │ │
│  │  • 证明验证                                               │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 50. Optimistic Rollups

### 50.1 核心原理

```
┌─────────────────────────────────────────────────────────────────┐
│                 Optimistic Rollup 原理                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "乐观" = 假设交易有效，除非被证明无效                          │
│                                                                 │
│  流程：                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. Sequencer 执行交易并提交状态根到 L1                  │   │
│  │  2. 开始挑战期（通常 7 天）                              │   │
│  │  3. 任何人可以提交欺诈证明                               │   │
│  │  4. 如无挑战，状态最终确定                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  欺诈证明：                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 发现错误状态转换                                      │   │
│  │  • 在 L1 上重放单步执行                                  │   │
│  │  • 如果证明成功，Sequencer 被罚没                        │   │
│  │  • 状态回滚到正确状态                                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  时间线：                                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  提交 ────▶ 挑战期（7天）────▶ 最终确定                 │   │
│  │    │                              │                      │   │
│  │    │ ◄──── 可能被挑战 ────►      │                      │   │
│  │    │                              │                      │   │
│  │    └──────────────────────────────┘                      │   │
│  │          在此期间提款需要等待                             │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 50.2 数据可用性

```go
// Optimistic Rollup 数据发布
type BatchData struct {
// 交易数据（压缩后）
Transactions []byte

// 之前的状态根
PrevStateRoot common.Hash

// 之后的状态根
PostStateRoot common.Hash

// 批次元数据
BatchIndex   uint64
L1BlockNum   uint64
Timestamp    uint64
}

// 数据发布方式（EIP-4844 后）
type DataAvailability struct {
// 方式1: Calldata（传统）
// - 永久存储在 L1
// - 成本较高
// - ~16 gas/byte

// 方式2: Blob（EIP-4844）
// - 临时存储（~18天）
// - 成本更低
// - ~1 gas/byte equivalent
}

// 计算发布成本
func CalculateDACost(data []byte, useBlob bool) *big.Int {
if useBlob {
// Blob gas 市场独立定价
blobGasPrice := big.NewInt(1) // 动态
blobGasUsed := uint64(len(data) / 131072 + 1) * 131072 // 每 blob 131KB
return new(big.Int).Mul(blobGasPrice, big.NewInt(int64(blobGasUsed)))
}

// Calldata: 非零字节 16 gas，零字节 4 gas
cost := uint64(0)
for _, b := range data {
if b == 0 {
cost += 4
} else {
cost += 16
}
}
gasPrice := big.NewInt(30e9) // 30 Gwei
return new(big.Int).Mul(gasPrice, big.NewInt(int64(cost)))
}
```

---

## 51. ZK Rollups

### 51.1 核心原理

```
┌─────────────────────────────────────────────────────────────────┐
│                    ZK Rollup 原理                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  "零知识" = 用数学证明状态转换正确                              │
│                                                                 │
│  流程：                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. Sequencer 执行交易                                   │   │
│  │  2. 生成 ZK 证明（证明执行正确）                         │   │
│  │  3. 提交状态根 + 证明到 L1                               │   │
│  │  4. L1 验证证明（毫秒级）                                │   │
│  │  5. 立即最终确定（无需等待期）                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ZK 证明类型：                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  SNARK (Succinct Non-interactive ARgument of Knowledge)  │   │
│  │  • 证明小（~300 bytes）                                  │   │
│  │  • 验证快（~毫秒）                                       │   │
│  │  • 需要可信设置                                          │   │
│  │                                                          │   │
│  │  STARK (Scalable Transparent ARgument of Knowledge)      │   │
│  │  • 证明大（~100 KB）                                     │   │
│  │  • 无需可信设置                                          │   │
│  │  • 抗量子攻击                                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  时间线：                                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  提交 ────▶ 证明验证 ────▶ 立即最终确定                 │   │
│  │    │          │              │                          │   │
│  │    │    几分钟到几小时       │                          │   │
│  │    │    （生成证明）         │                          │   │
│  │    │                        │                          │   │
│  │    └────────────────────────┘                          │   │
│  │       提款可以快速完成（分钟级）                         │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 51.2 zkEVM 类型

```
┌─────────────────────────────────────────────────────────────────┐
│                    zkEVM 类型分类                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Type 1: 完全等价于以太坊                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 100% 字节码兼容                                       │   │
│  │  • 相同的状态树结构                                      │   │
│  │  • 相同的 Gas 计算                                       │   │
│  │  • 证明生成最慢（小时级）                                │   │
│  │  • 代表：Taiko, Scroll                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Type 2: EVM 等价（细微差异）                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 字节码兼容                                            │   │
│  │  • 不同的状态树（更适合 ZK）                             │   │
│  │  • 可能有不同的 Gas 成本                                 │   │
│  │  • 代表：Polygon zkEVM                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Type 3: EVM 兼容                                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 大部分字节码兼容                                      │   │
│  │  • 部分操作码行为不同                                    │   │
│  │  • 更快的证明生成                                        │   │
│  │  • 代表：zkSync Era, Linea                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Type 4: 高级语言兼容                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • Solidity/Vyper 源码兼容                               │   │
│  │  • 编译到自定义 VM                                       │   │
│  │  • 最快的证明生成                                        │   │
│  │  • 代表：Starknet (Cairo)                                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 52. Arbitrum 详解

### 52.1 Arbitrum 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Arbitrum 架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  Arbitrum One                            │   │
│  │                                                          │   │
│  │  ┌─────────────┐    ┌─────────────┐                     │   │
│  │  │  Sequencer  │    │   Nitro     │                     │   │
│  │  │  (排序器)    │───▶│   (执行)    │                     │   │
│  │  └─────────────┘    └──────┬──────┘                     │   │
│  │                            │                             │   │
│  │  ┌────────────────────────────────────────────────────┐ │   │
│  │  │  ArbOS（Arbitrum 操作系统）                        │ │   │
│  │  │  • Gas 计量                                        │ │   │
│  │  │  • 跨链消息                                        │ │   │
│  │  │  • 预编译合约                                      │ │   │
│  │  └────────────────────────────────────────────────────┘ │   │
│  │                            │                             │   │
│  │                            ▼                             │   │
│  │  ┌────────────────────────────────────────────────────┐ │   │
│  │  │  Geth (修改版)                                     │ │   │
│  │  │  • 标准 EVM 执行                                   │ │   │
│  │  │  • 状态管理                                        │ │   │
│  │  └────────────────────────────────────────────────────┘ │   │
│  │                                                          │   │
│  └──────────────────────────┬───────────────────────────────┘   │
│                             │                                    │
│                             ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Ethereum L1                          │    │
│  │  • Rollup 合约                                          │    │
│  │  • 收件箱 (Inbox)                                       │    │
│  │  • 发件箱 (Outbox)                                      │    │
│  │  • 挑战管理器                                           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 52.2 Arbitrum 特有功能

```go
// ArbOS 预编译合约地址
var (
ArbSysAddress = common.HexToAddress("0x0000000000000000000000000000000000000064")
ArbInfoAddress = common.HexToAddress("0x0000000000000000000000000000000000000065")
ArbRetryableTxAddress   = common.HexToAddress("0x000000000000000000000000000000000000006E")
ArbGasInfoAddress = common.HexToAddress("0x000000000000000000000000000000000000006C")
ArbAggregatorAddress = common.HexToAddress("0x000000000000000000000000000000000000006D")
)

// ArbSys 接口
type ArbSys interface {
// 获取 Arbitrum 区块号
ArbBlockNumber() *big.Int

// 获取 L1 区块号
ArbBlockHash(blockNum uint64) common.Hash

// 发送 L2 到 L1 消息
SendTxToL1(destination common.Address, data []byte) (*big.Int, error)

// 获取当前 L1 区块号估计
ArbOSVersion() uint64
}

// 使用示例：获取 L1 区块号
func GetL1BlockNumber(client *ethclient.Client) (uint64, error) {
arbSysABI, _ := abi.JSON(strings.NewReader(`[{
        "inputs": [],
        "name": "arbBlockNumber",
        "outputs": [{"type": "uint256"}],
        "stateMutability": "view",
        "type": "function"
    }]`))

data, _ := arbSysABI.Pack("arbBlockNumber")

result, err := client.CallContract(context.Background(), ethereum.CallMsg{
To:   &ArbSysAddress,
Data: data,
}, nil)
if err != nil {
return 0, err
}

outputs, _ := arbSysABI.Unpack("arbBlockNumber", result)
return outputs[0].(*big.Int).Uint64(), nil
}
```

### 52.3 Arbitrum Gas 模型

```go
// Arbitrum Gas 计算
type ArbitrumGas struct {
// L2 Gas（执行成本）
L2Gas uint64

// L1 Gas（数据发布成本）
L1Gas uint64
}

// 计算 Arbitrum 交易成本
func CalculateArbitrumTxCost(tx *types.Transaction) (*big.Int, error) {
// L2 Gas 成本
l2GasPrice := big.NewInt(100000000) // 0.1 Gwei (示例)
l2Cost := new(big.Int).Mul(big.NewInt(int64(tx.Gas())), l2GasPrice)

// L1 数据成本
// Arbitrum 对发布到 L1 的数据收取额外费用
dataSize := len(tx.Data())
l1GasPerByte := uint64(16) // 大约值
l1GasUsed := uint64(dataSize) * l1GasPerByte
l1GasPrice := big.NewInt(30e9) // L1 Gas 价格
l1Cost := new(big.Int).Mul(big.NewInt(int64(l1GasUsed)), l1GasPrice)

// 总成本
totalCost := new(big.Int).Add(l2Cost, l1Cost)
return totalCost, nil
}

// 获取实时 Gas 价格
func GetArbitrumGasPrices(client *ethclient.Client) (*ArbitrumGasPrices, error) {
// 调用 ArbGasInfo 预编译
arbGasInfoABI, _ := abi.JSON(strings.NewReader(`[{
        "inputs": [],
        "name": "getPricesInWei",
        "outputs": [
            {"name": "perL2Tx", "type": "uint256"},
            {"name": "perL1CalldataUnit", "type": "uint256"},
            {"name": "perStorageCell", "type": "uint256"},
            {"name": "perArbGasBase", "type": "uint256"},
            {"name": "perArbGasCongestion", "type": "uint256"},
            {"name": "perArbGasTotal", "type": "uint256"}
        ],
        "stateMutability": "view",
        "type": "function"
    }]`))

data, _ := arbGasInfoABI.Pack("getPricesInWei")

result, err := client.CallContract(context.Background(), ethereum.CallMsg{
To:   &ArbGasInfoAddress,
Data: data,
}, nil)
if err != nil {
return nil, err
}

outputs, _ := arbGasInfoABI.Unpack("getPricesInWei", result)

return &ArbitrumGasPrices{
PerL2Tx:            outputs[0].(*big.Int),
PerL1CalldataUnit:  outputs[1].(*big.Int),
PerStorageCell:     outputs[2].(*big.Int),
PerArbGasBase:      outputs[3].(*big.Int),
PerArbGasCongestion: outputs[4].(*big.Int),
PerArbGasTotal:     outputs[5].(*big.Int),
}, nil
}

type ArbitrumGasPrices struct {
PerL2Tx            *big.Int
PerL1CalldataUnit  *big.Int
PerStorageCell     *big.Int
PerArbGasBase      *big.Int
PerArbGasCongestion *big.Int
PerArbGasTotal     *big.Int
}
```

---

## 53. Optimism 详解

### 53.1 Optimism 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Optimism 架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  OP Mainnet                              │   │
│  │                                                          │   │
│  │  ┌─────────────┐    ┌─────────────┐                     │   │
│  │  │  Sequencer  │    │   op-geth   │                     │   │
│  │  │  (op-node)  │───▶│   (执行)    │                     │   │
│  │  └─────────────┘    └──────┬──────┘                     │   │
│  │                            │                             │   │
│  │  ┌────────────────────────────────────────────────────┐ │   │
│  │  │  Bedrock 架构                                      │ │   │
│  │  │  • 模块化设计                                      │ │   │
│  │  │  • EVM 等价                                        │ │   │
│  │  │  • 派生（Derivation）系统                          │ │   │
│  │  └────────────────────────────────────────────────────┘ │   │
│  │                                                          │   │
│  └──────────────────────────┬───────────────────────────────┘   │
│                             │                                    │
│                             ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Ethereum L1                          │    │
│  │  • OptimismPortal                                       │    │
│  │  • L2OutputOracle                                       │    │
│  │  • SystemConfig                                         │    │
│  │  • Batch 数据（Blob/Calldata）                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 53.2 Optimism 特有功能

```go
// Optimism L1 Block 信息预编译
var L1BlockAddress = common.HexToAddress("0x4200000000000000000000000000000000000015")

// L1Block 合约接口
type L1Block interface {
// 获取 L1 区块号
Number() uint64

// 获取 L1 时间戳
Timestamp() uint64

// 获取 L1 base fee
BaseFee() *big.Int

// 获取 L1 区块哈希
Hash() common.Hash

// 获取序列号
SequenceNumber() uint64

// 获取 batcher 哈希
BatcherHash() common.Hash

// 获取 L1 fee overhead
L1FeeOverhead() *big.Int

// 获取 L1 fee scalar
L1FeeScalar() *big.Int
}

// 获取 L1 区块信息
func GetL1BlockInfo(client *ethclient.Client) (*L1BlockInfo, error) {
l1BlockABI, _ := abi.JSON(strings.NewReader(`[
        {"inputs":[],"name":"number","outputs":[{"type":"uint64"}],"stateMutability":"view","type":"function"},
        {"inputs":[],"name":"timestamp","outputs":[{"type":"uint64"}],"stateMutability":"view","type":"function"},
        {"inputs":[],"name":"basefee","outputs":[{"type":"uint256"}],"stateMutability":"view","type":"function"},
        {"inputs":[],"name":"hash","outputs":[{"type":"bytes32"}],"stateMutability":"view","type":"function"}
    ]`))

info := &L1BlockInfo{}

// 获取 number
data, _ := l1BlockABI.Pack("number")
result, _ := client.CallContract(context.Background(), ethereum.CallMsg{
To: &L1BlockAddress, Data: data,
}, nil)
outputs, _ := l1BlockABI.Unpack("number", result)
info.Number = outputs[0].(uint64)

// 获取其他字段...
return info, nil
}

type L1BlockInfo struct {
Number     uint64
Timestamp  uint64
BaseFee    *big.Int
Hash       common.Hash
}
```

### 53.3 Optimism Gas 模型

```go
// Optimism Gas 计算（EIP-1559 + L1 数据费）
func CalculateOptimismTxCost(tx *types.Transaction, l1BaseFee *big.Int) (*big.Int, error) {
// L2 执行成本（标准 EIP-1559）
l2GasCost := calculateL2Gas(tx)

// L1 数据成本
// l1Cost = l1BaseFee * l1GasUsed * l1FeeScalar / 1e6
txData := tx.Data()
l1GasUsed := calculateL1GasUsed(txData)

l1FeeScalar := big.NewInt(684000)  // 从 SystemConfig 获取
l1FeeOverhead := big.NewInt(2100)  // 固定开销

l1GasWithOverhead := new(big.Int).Add(big.NewInt(int64(l1GasUsed)), l1FeeOverhead)
l1Cost := new(big.Int).Mul(l1BaseFee, l1GasWithOverhead)
l1Cost.Mul(l1Cost, l1FeeScalar)
l1Cost.Div(l1Cost, big.NewInt(1e6))

// 总成本
return new(big.Int).Add(l2GasCost, l1Cost), nil
}

// 计算 L1 Gas 使用量
func calculateL1GasUsed(data []byte) uint64 {
// 零字节: 4 gas, 非零字节: 16 gas
var l1Gas uint64
for _, b := range data {
if b == 0 {
l1Gas += 4
} else {
l1Gas += 16
}
}
return l1Gas
}

// GasPriceOracle 合约
var GasPriceOracleAddress = common.HexToAddress("0x420000000000000000000000000000000000000F")

// 获取实时 L1 数据费用
func GetL1DataFee(client *ethclient.Client, txData []byte) (*big.Int, error) {
oracleABI, _ := abi.JSON(strings.NewReader(`[{
        "inputs": [{"name": "_data", "type": "bytes"}],
        "name": "getL1Fee",
        "outputs": [{"type": "uint256"}],
        "stateMutability": "view",
        "type": "function"
    }]`))

data, _ := oracleABI.Pack("getL1Fee", txData)

result, err := client.CallContract(context.Background(), ethereum.CallMsg{
To:   &GasPriceOracleAddress,
Data: data,
}, nil)
if err != nil {
return nil, err
}

outputs, _ := oracleABI.Unpack("getL1Fee", result)
return outputs[0].(*big.Int), nil
}
```

---

## 54. Base 与 OP Stack

### 54.1 OP Stack 概述

```
┌─────────────────────────────────────────────────────────────────┐
│                      OP Stack                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OP Stack = 模块化的 Rollup 构建框架                            │
│                                                                 │
│  使用 OP Stack 的链：                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • Optimism Mainnet (OP)                                │   │
│  │  • Base (Coinbase)                                      │   │
│  │  • Zora                                                 │   │
│  │  • Mode Network                                         │   │
│  │  • Mantle                                               │   │
│  │  • 更多...                                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  模块化组件：                                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │   │
│  │  │   op-node    │  │   op-geth    │  │  op-batcher  │  │   │
│  │  │  (共识层)    │  │   (执行层)   │  │  (批处理)    │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │   │
│  │                                                         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │   │
│  │  │ op-proposer  │  │ op-challenger│  │   Contracts  │  │   │
│  │  │  (提议者)    │  │   (挑战者)   │  │  (L1 合约)   │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Superchain 愿景：                                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 多个 OP Stack 链互联互通                             │   │
│  │  • 共享安全性                                           │   │
│  │  • 跨链消息传递                                         │   │
│  │  • 统一的用户体验                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 54.2 Base 特性

```go
// Base 特有配置
type BaseConfig struct {
// 与 Optimism 共享相同的合约和工具
// 但有独特的参数配置

ChainID      uint64 // 8453
L1ChainID    uint64 // 1 (Ethereum)
Sequencer    common.Address
BaseFeeVault common.Address
}

// Base 与 Optimism 的差异
type BaseDifferences struct {
// 1. 不同的 Sequencer 运营者
SequencerOperator string // Coinbase

// 2. 不同的费用接收地址
FeeRecipient common.Address

// 3. 可能有不同的 Gas 参数
L1FeeScalar *big.Int
}

// 跨 OP Stack 链互操作
func CrossChainCall(
sourceChain, targetChain *ethclient.Client,
message []byte,
) error {
// OP Stack 链间可以通过 L1 进行消息传递
// 1. 在源链发送消息到 L1
// 2. 在目标链从 L1 接收消息

// 使用 CrossDomainMessenger
return nil
}
```

---

## 55. L2 MEV 特性

### 55.1 L2 MEV 概述

```
┌─────────────────────────────────────────────────────────────────┐
│                    L2 MEV 特性                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  L2 与 L1 MEV 的差异：                                          │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  特性          L1 (Ethereum)      L2 (Arbitrum/Optimism)  │ │
│  │  ─────────────────────────────────────────────────────────│ │
│  │  区块生产者    验证者 (分散)       Sequencer (中心化)      │ │
│  │  出块时间      12 秒              ~250ms (Arb) / 2s (OP)  │ │
│  │  Mempool      公开               私有 (Sequencer 控制)    │ │
│  │  MEV 提取     Flashbots/Builder  Sequencer 决定          │ │
│  │  Gas 费用     高                  低                      │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  L2 MEV 机会：                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. DEX 套利 - 仍然存在，但竞争激烈                     │   │
│  │  2. 清算 - 与 L1 类似                                    │   │
│  │  3. 三明治攻击 - 部分 Sequencer 禁止                    │   │
│  │  4. 跨 L2 套利 - 新机会                                 │   │
│  │  5. L1 ↔ L2 套利 - 桥接延迟创造机会                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 55.2 L2 Sequencer 与 MEV

```go
// Sequencer MEV 策略
type SequencerMEVPolicy struct {
// Arbitrum
// - 公平排序（FCFS - First Come First Serve）
// - 不进行三明治攻击
// - 有时间戳容忍度

// Optimism
// - 类似 FCFS
// - 私有 Sequencer 无公开 Mempool

// 未来：
// - 去中心化 Sequencer
// - MEV 拍卖 (MEV-Share 类似机制)
}

// 检测 L2 MEV 机会
type L2MEVDetector struct {
client    *ethclient.Client
chainType string // "arbitrum", "optimism", "base"
}

// DetectArbitrage 检测 L2 套利机会
func (d *L2MEVDetector) DetectArbitrage(pools []common.Address) ([]*ArbitrageOpportunity, error) {
opportunities := make([]*ArbitrageOpportunity, 0)

// 获取池子储备
reserves := make(map[common.Address][2]*big.Int)
for _, pool := range pools {
r0, r1, _ := getReserves(d.client, pool)
reserves[pool] = [2]*big.Int{r0, r1}
}

// 检查所有池子对
for i := 0; i < len(pools); i++ {
for j := i + 1; j < len(pools); j++ {
profit := calculateArbitrageProfit(reserves[pools[i]], reserves[pools[j]])
if profit.Sign() > 0 {
// 检查利润是否覆盖 L2 Gas 成本
gasCost := d.estimateGasCost()
if profit.Cmp(gasCost) > 0 {
opportunities = append(opportunities, &ArbitrageOpportunity{
Pool1:  pools[i],
Pool2:  pools[j],
Profit: new(big.Int).Sub(profit, gasCost),
})
}
}
}
}

return opportunities, nil
}

// estimateGasCost 估算 L2 Gas 成本
func (d *L2MEVDetector) estimateGasCost() *big.Int {
switch d.chainType {
case "arbitrum":
// Arbitrum: L2 Gas + L1 数据费
return big.NewInt(0.0001e18)  // ~0.0001 ETH 示例
case "optimism", "base":
// OP Stack: L2 Gas + L1 数据费
return big.NewInt(0.0001e18)
default:
return big.NewInt(0.001e18)
}
}
```

### 55.3 L2 特有的 MEV 策略

```go
// L2 特有策略

// 1. 快速套利（利用快速出块）
type FastBlockArbitrage struct {
// L2 出块快，可以更频繁地检查和执行套利
CheckInterval time.Duration // 100ms on Arbitrum
}

func (f *FastBlockArbitrage) Run(ctx context.Context) {
ticker := time.NewTicker(f.CheckInterval)
for {
select {
case <-ctx.Done():
return
case <-ticker.C:
// 快速检查和执行
f.checkAndExecute()
}
}
}

// 2. Sequencer 时间戳利用
type TimestampStrategy struct {
// Sequencer 有一定的时间戳灵活性
// 可能影响某些 DeFi 操作（如 TWAP）
}

// 3. L1 → L2 消息领先
type L1ToL2FrontrunStrategy struct {
// 监控 L1 上发送到 L2 的消息
// 在消息执行前进行相关操作
}

func (s *L1ToL2FrontrunStrategy) MonitorL1Deposits(ctx context.Context) {
// 监控 L1 存款事件
// 这些存款会在 L2 上创建交易
// 可以预测并提前行动
}
```

---

## 56. 跨 L2 套利

### 56.1 跨 L2 套利概述

```
┌─────────────────────────────────────────────────────────────────┐
│                   跨 L2 套利                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  场景：同一代币在不同 L2 上价格不同                             │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │    Arbitrum                    Optimism                  │   │
│  │  ┌──────────────┐            ┌──────────────┐           │   │
│  │  │ ETH/USDC     │            │ ETH/USDC     │           │   │
│  │  │ 1 ETH =      │            │ 1 ETH =      │           │   │
│  │  │ $2000 USDC   │            │ $2010 USDC   │           │   │
│  │  └──────────────┘            └──────────────┘           │   │
│  │        │                            │                    │   │
│  │        │      存在 $10 价差        │                    │   │
│  │        └────────────────────────────┘                    │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  挑战：                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. 跨链桥接延迟（分钟到小时）                          │   │
│  │  2. 桥接费用                                             │   │
│  │  3. 价格变动风险                                         │   │
│  │  4. 资金效率（需要在两边都有资金）                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  策略：                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. 多边预留资金：在每个 L2 上都准备资金                │   │
│  │  2. 同步交易：在两边同时执行交易                        │   │
│  │  3. 定期再平衡：当某边资金不足时通过桥接补充            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 56.2 跨 L2 套利实现

```go
// CrossL2Arbitrage 跨 L2 套利器
type CrossL2Arbitrage struct {
// 客户端
arbitrumClient *ethclient.Client
optimismClient *ethclient.Client
baseClient     *ethclient.Client

// 钱包
wallet *Wallet

// 配置
minProfit      *big.Int
maxTradeSize   *big.Int
}

// Monitor 监控跨 L2 价格差异
func (c *CrossL2Arbitrage) Monitor(ctx context.Context) {
ticker := time.NewTicker(1 * time.Second)

for {
select {
case <-ctx.Done():
return
case <-ticker.C:
c.checkPriceDifferences()
}
}
}

// checkPriceDifferences 检查价格差异
func (c *CrossL2Arbitrage) checkPriceDifferences() {
// 获取各 L2 上的 ETH/USDC 价格
arbPrice, _ := c.getPrice(c.arbitrumClient, WETH, USDC)
opPrice, _ := c.getPrice(c.optimismClient, WETH, USDC)
basePrice, _ := c.getPrice(c.baseClient, WETH, USDC)

prices := map[string]*big.Int{
"arbitrum": arbPrice,
"optimism": opPrice,
"base":     basePrice,
}

// 找出最高价和最低价
var maxChain, minChain string
var maxPrice, minPrice *big.Int

for chain, price := range prices {
if maxPrice == nil || price.Cmp(maxPrice) > 0 {
maxPrice = price
maxChain = chain
}
if minPrice == nil || price.Cmp(minPrice) < 0 {
minPrice = price
minChain = chain
}
}

// 计算价差百分比
priceDiff := new(big.Int).Sub(maxPrice, minPrice)
priceDiffPct := new(big.Float).Quo(
new(big.Float).SetInt(priceDiff),
new(big.Float).SetInt(minPrice),
)

pctFloat, _ := priceDiffPct.Float64()

// 如果价差超过阈值，考虑套利
if pctFloat > 0.001 {  // 0.1%
fmt.Printf("Price diff: %.2f%% between %s and %s\n",
pctFloat*100, minChain, maxChain)

c.evaluateArbitrage(minChain, maxChain, priceDiff)
}
}

// executeArbitrage 执行跨 L2 套利
func (c *CrossL2Arbitrage) executeArbitrage(buyChain, sellChain string, amount *big.Int) error {
// 1. 在低价链买入
buyClient := c.getClient(buyChain)
buyTx, err := c.executeBuy(buyClient, amount)
if err != nil {
return err
}

// 2. 在高价链卖出
sellClient := c.getClient(sellChain)
sellTx, err := c.executeSell(sellClient, amount)
if err != nil {
// 需要处理部分执行的情况
return err
}

fmt.Printf("Arbitrage executed: buy on %s (tx: %s), sell on %s (tx: %s)\n",
buyChain, buyTx.Hash().Hex(), sellChain, sellTx.Hash().Hex())

return nil
}

// RebalanceFunds 再平衡资金
func (c *CrossL2Arbitrage) RebalanceFunds(fromChain, toChain string, amount *big.Int) error {
// 选择合适的桥
bridge := c.selectBridge(fromChain, toChain)

// 执行跨链转账
return bridge.Transfer(amount)
}

// Bridge 桥接口
type Bridge interface {
Transfer(amount *big.Int) error
EstimateFee(amount *big.Int) *big.Int
EstimateTime() time.Duration
}

// 官方桥实现
type OfficialBridge struct {
fromClient *ethclient.Client
toClient   *ethclient.Client
bridgeAddr common.Address
}

func (b *OfficialBridge) Transfer(amount *big.Int) error {
// 调用官方桥合约
return nil
}

func (b *OfficialBridge) EstimateTime() time.Duration {
// Optimistic Rollup: 7 天挑战期
// 但从 L1 到 L2 通常是 10-20 分钟
return 15 * time.Minute
}

// 第三方桥（更快但有费用）
type ThirdPartyBridge struct {
// Hop, Across, Stargate 等
provider string
}

func (b *ThirdPartyBridge) EstimateTime() time.Duration {
// 第三方桥通常 1-15 分钟
return 5 * time.Minute
}
```

### 56.3 完整跨 L2 套利系统

```go
// CrossL2ArbitrageSystem 完整的跨 L2 套利系统
type CrossL2ArbitrageSystem struct {
// L2 客户端
clients map[string]*ethclient.Client

// 资金追踪
balances map[string]map[common.Address]*big.Int // chain -> token -> balance

// 价格源
priceFeeds map[string]*PriceFeed

// 配置
config *CrossL2Config
}

type CrossL2Config struct {
MinProfitBps     uint64   // 最小利润 (基点)
MaxTradeSize     *big.Int // 最大交易规模
RebalanceThresh  *big.Int // 再平衡阈值
SupportedChains  []string // 支持的链
SupportedTokens  []common.Address
}

func NewCrossL2ArbitrageSystem(config *CrossL2Config) *CrossL2ArbitrageSystem {
system := &CrossL2ArbitrageSystem{
clients:    make(map[string]*ethclient.Client),
balances:   make(map[string]map[common.Address]*big.Int),
priceFeeds: make(map[string]*PriceFeed),
config:     config,
}

// 初始化各链客户端
for _, chain := range config.SupportedChains {
client, _ := ethclient.Dial(getRPCURL(chain))
system.clients[chain] = client
system.balances[chain] = make(map[common.Address]*big.Int)
}

return system
}

func (s *CrossL2ArbitrageSystem) Run(ctx context.Context) {
// 启动价格监控
go s.monitorPrices(ctx)

// 启动余额监控
go s.monitorBalances(ctx)

// 启动套利执行
go s.runArbitrage(ctx)

// 启动再平衡
go s.runRebalancer(ctx)

<-ctx.Done()
}

func (s *CrossL2ArbitrageSystem) monitorPrices(ctx context.Context) {
ticker := time.NewTicker(500 * time.Millisecond)

for {
select {
case <-ctx.Done():
return
case <-ticker.C:
for chain, client := range s.clients {
for _, token := range s.config.SupportedTokens {
price, _ := s.getTokenPrice(client, token)
s.priceFeeds[chain].Update(token, price)
}
}
}
}
}

func (s *CrossL2ArbitrageSystem) runArbitrage(ctx context.Context) {
for {
select {
case <-ctx.Done():
return
default:
// 检查所有代币对的跨链套利机会
for _, token := range s.config.SupportedTokens {
opportunity := s.findBestOpportunity(token)
if opportunity != nil && opportunity.ProfitBps > s.config.MinProfitBps {
s.executeOpportunity(opportunity)
}
}
time.Sleep(100 * time.Millisecond)
}
}
}

type CrossL2Opportunity struct {
Token     common.Address
BuyChain  string
SellChain string
BuyPrice  *big.Int
SellPrice *big.Int
Amount    *big.Int
ProfitBps uint64
}

func (s *CrossL2ArbitrageSystem) findBestOpportunity(token common.Address) *CrossL2Opportunity {
var best *CrossL2Opportunity

chains := s.config.SupportedChains
for i := 0; i < len(chains); i++ {
for j := i + 1; j < len(chains); j++ {
chain1, chain2 := chains[i], chains[j]
price1 := s.priceFeeds[chain1].Get(token)
price2 := s.priceFeeds[chain2].Get(token)

if price1 == nil || price2 == nil {
continue
}

var buyChain, sellChain string
var buyPrice, sellPrice *big.Int

if price1.Cmp(price2) < 0 {
buyChain, sellChain = chain1, chain2
buyPrice, sellPrice = price1, price2
} else {
buyChain, sellChain = chain2, chain1
buyPrice, sellPrice = price2, price1
}

// 计算利润
priceDiff := new(big.Int).Sub(sellPrice, buyPrice)
profitBps := new(big.Int).Mul(priceDiff, big.NewInt(10000))
profitBps.Div(profitBps, buyPrice)

if best == nil || profitBps.Uint64() > best.ProfitBps {
// 检查是否有足够余额
amount := s.calculateOptimalAmount(buyChain, sellChain, token)
if amount.Sign() > 0 {
best = &CrossL2Opportunity{
Token:     token,
BuyChain:  buyChain,
SellChain: sellChain,
BuyPrice:  buyPrice,
SellPrice: sellPrice,
Amount:    amount,
ProfitBps: profitBps.Uint64(),
}
}
}
}
}

return best
}
```

---

## 总结

本文档详细介绍了 L2 EVM 变体：

1. **L2 概述**：扩容原因、类型对比、架构
2. **Optimistic Rollups**：欺诈证明、数据可用性
3. **ZK Rollups**：有效性证明、zkEVM 类型
4. **Arbitrum**：架构、ArbOS、Gas 模型
5. **Optimism**：Bedrock、OP Stack
6. **Base**：OP Stack 应用、Superchain
7. **L2 MEV**：与 L1 差异、Sequencer 策略
8. **跨 L2 套利**：机会、挑战、实现

下一篇将介绍 EVM 未来演进（EOF、Verkle Trees 等）。
