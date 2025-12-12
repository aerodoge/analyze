# Go-Ethereum EVM 深度解析：预言机机制（第十一部分）

## 第81节：预言机基础概念

### 81.1 预言机问题

```
┌─────────────────────────────────────────────────────────────────┐
│                    预言机问题（Oracle Problem）                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  区块链世界                          现实世界                    │
│  ┌─────────────────────┐            ┌─────────────────────┐    │
│  │                     │            │                     │    │
│  │  智能合约           │     ?      │  ETH/USD 价格       │    │
│  │  - 确定性执行       │◄──────────►│  天气数据           │    │
│  │  - 无法访问外部数据  │            │  体育比赛结果       │    │
│  │  - 需要可验证输入   │            │  随机数             │    │
│  │                     │            │  股票价格           │    │
│  └─────────────────────┘            └─────────────────────┘    │
│                                                                 │
│  核心挑战:                                                       │
│  1. 智能合约无法主动获取链外数据                                  │
│  2. 如何保证外部数据的真实性和可靠性                              │
│  3. 如何激励节点诚实提供数据                                     │
│  4. 如何防止单点故障和数据操纵                                    │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  预言机解决方案架构:                                              │
│                                                                 │
│  ┌─────────┐    ┌─────────────────┐    ┌─────────────────┐    │
│  │ 数据源   │───►│  预言机网络      │───►│  智能合约        │    │
│  │ (CEX等)  │    │  (Chainlink等)  │    │  (DeFi协议)     │    │
│  └─────────┘    └─────────────────┘    └─────────────────┘    │
│       │                 │                       │              │
│       │                 │                       │              │
│       ▼                 ▼                       ▼              │
│  多数据源聚合      去中心化节点网络         使用聚合后的价格    │
│  减少单点风险      共识机制验证数据         进行借贷/清算等    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 81.2 预言机类型分类

```go
package oracle

// 预言机类型分类

// 1. 按数据来源分类
type OracleBySource int
const (
    // 中心化预言机 - 单一数据提供者
    CentralizedOracle OracleBySource = iota
    // 去中心化预言机 - 多节点共识
    DecentralizedOracle
    // 链上预言机 - 从其他智能合约获取数据
    OnChainOracle
)

// 2. 按数据推送方式分类
type OracleByPush int
const (
    // 推送型 - 预言机主动推送数据
    PushOracle OracleByPush = iota
    // 拉取型 - 合约请求数据
    PullOracle
    // 混合型
    HybridOracle
)

// 3. 按数据类型分类
type OracleByData int
const (
    // 价格预言机
    PriceOracle OracleByData = iota
    // 随机数预言机 (VRF)
    RandomnessOracle
    // 储备证明预言机
    ProofOfReserveOracle
    // 跨链预言机
    CrossChainOracle
    // 计算预言机
    ComputationOracle
)

// 主流预言机项目对比
type OracleProject struct {
    Name             string
    Type             OracleBySource
    DataSources      int      // 数据源数量
    Nodes            int      // 节点数量
    UpdateFrequency  string   // 更新频率
    SecurityModel    string   // 安全模型
    Latency          string   // 延迟
    Cost             string   // 成本
}

var MainOracles = []OracleProject{
    {
        Name:            "Chainlink",
        Type:            DecentralizedOracle,
        DataSources:     7,
        Nodes:           31,
        UpdateFrequency: "心跳+偏差阈值",
        SecurityModel:   "质押+信誉",
        Latency:         "~1分钟",
        Cost:            "高(每次更新需gas)",
    },
    {
        Name:            "Uniswap TWAP",
        Type:            OnChainOracle,
        DataSources:     1,
        Nodes:           0,
        UpdateFrequency: "每笔交易",
        SecurityModel:   "操纵成本",
        Latency:         "即时(但需时间窗口)",
        Cost:            "低(读取免费)",
    },
    {
        Name:            "Pyth",
        Type:            DecentralizedOracle,
        DataSources:     70,
        Nodes:           70,
        UpdateFrequency: "400ms",
        SecurityModel:   "质押+拉取模型",
        Latency:         "亚秒级",
        Cost:            "低(按需拉取)",
    },
    {
        Name:            "Band Protocol",
        Type:            DecentralizedOracle,
        DataSources:     16,
        Nodes:           100,
        UpdateFrequency: "按需",
        SecurityModel:   "质押+DPoS",
        Latency:         "~6秒",
        Cost:            "中等",
    },
}
```

### 81.3 预言机数据结构

```go
package oracle

import (
    "math/big"
    "time"
)

// PriceFeed 价格馈送数据
type PriceFeed struct {
    RoundID         *big.Int  // 轮次ID
    Answer          *big.Int  // 价格答案 (8位小数)
    StartedAt       time.Time // 轮次开始时间
    UpdatedAt       time.Time // 最后更新时间
    AnsweredInRound *big.Int  // 回答所在轮次
}

// AggregatorV3Interface Chainlink聚合器接口
type AggregatorV3Interface interface {
    Decimals() (uint8, error)
    Description() (string, error)
    Version() (*big.Int, error)
    GetRoundData(roundId *big.Int) (*PriceFeed, error)
    LatestRoundData() (*PriceFeed, error)
}

// OracleData 通用预言机数据
type OracleData struct {
    Pair        string    // 交易对 (如 "ETH/USD")
    Price       *big.Int  // 价格
    Decimals    uint8     // 小数位数
    Timestamp   time.Time // 时间戳
    Source      string    // 数据来源
    Confidence  float64   // 置信度 (0-1)

    // 元数据
    BlockNumber uint64    // 区块号
    TxHash      string    // 交易哈希
    GasUsed     uint64    // gas消耗
}

// OracleConfig 预言机配置
type OracleConfig struct {
    // 更新策略
    HeartbeatInterval time.Duration // 心跳间隔
    DeviationThreshold float64      // 偏差阈值 (如 0.5%)

    // 安全参数
    MinResponses      int           // 最小响应数
    StalenessThreshold time.Duration // 数据过期阈值

    // 聚合策略
    AggregationType   string        // "median", "mean", "twap"
}
```

---

## 第82节：Chainlink 预言机详解

### 82.1 Chainlink 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Chainlink 架构详解                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  数据源层                                                        │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐              │
│  │Coinbase │ │ Kraken  │ │Binance  │ │ 其他API │              │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘              │
│       │          │          │          │                       │
│       └──────────┴──────────┴──────────┘                       │
│                      │                                          │
│  预言机节点层         ▼                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │           Chainlink 节点网络 (31个独立节点)               │   │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐       │   │
│  │  │Node1│ │Node2│ │Node3│ │Node4│ │Node5│ │ ... │       │   │
│  │  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘       │   │
│  │     │       │       │       │       │       │           │   │
│  │     └───────┴───────┴───────┴───────┴───────┘           │   │
│  │                         │                                │   │
│  │                    提交价格                               │   │
│  └─────────────────────────┼───────────────────────────────┘   │
│                            │                                    │
│  链上聚合层                 ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              FluxAggregator / OCR Aggregator             │   │
│  │                                                          │   │
│  │  1. 收集节点提交的价格                                     │   │
│  │  2. 计算中位数                                            │   │
│  │  3. 检查偏差阈值                                          │   │
│  │  4. 更新链上价格                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            │                                    │
│  消费者层                   ▼                                    │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐              │
│  │  Aave   │ │Compound │ │  MakerDAO│ │  其他   │              │
│  └─────────┘ └─────────┘ └─────────┘ └─────────┘              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 82.2 读取 Chainlink 价格

```go
package chainlink

import (
    "context"
    "fmt"
    "math/big"
    "strings"

    "github.com/ethereum/go-ethereum"
    "github.com/ethereum/go-ethereum/accounts/abi"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/ethclient"
)

// Chainlink AggregatorV3 ABI (精简版)
const AggregatorV3ABI = `[
    {
        "inputs": [],
        "name": "decimals",
        "outputs": [{"type": "uint8"}],
        "stateMutability": "view",
        "type": "function"
    },
    {
        "inputs": [],
        "name": "description",
        "outputs": [{"type": "string"}],
        "stateMutability": "view",
        "type": "function"
    },
    {
        "inputs": [],
        "name": "latestRoundData",
        "outputs": [
            {"name": "roundId", "type": "uint80"},
            {"name": "answer", "type": "int256"},
            {"name": "startedAt", "type": "uint256"},
            {"name": "updatedAt", "type": "uint256"},
            {"name": "answeredInRound", "type": "uint80"}
        ],
        "stateMutability": "view",
        "type": "function"
    },
    {
        "inputs": [{"name": "_roundId", "type": "uint80"}],
        "name": "getRoundData",
        "outputs": [
            {"name": "roundId", "type": "uint80"},
            {"name": "answer", "type": "int256"},
            {"name": "startedAt", "type": "uint256"},
            {"name": "updatedAt", "type": "uint256"},
            {"name": "answeredInRound", "type": "uint80"}
        ],
        "stateMutability": "view",
        "type": "function"
    }
]`

// ChainlinkReader Chainlink价格读取器
type ChainlinkReader struct {
    client *ethclient.Client
    abi    abi.ABI
}

// RoundData 轮次数据
type RoundData struct {
    RoundID         *big.Int
    Answer          *big.Int
    StartedAt       *big.Int
    UpdatedAt       *big.Int
    AnsweredInRound *big.Int
}

// PriceInfo 价格信息
type PriceInfo struct {
    Price       float64
    Decimals    uint8
    Description string
    UpdatedAt   int64
    RoundID     *big.Int
    IsStale     bool
}

// NewChainlinkReader 创建Chainlink读取器
func NewChainlinkReader(client *ethclient.Client) (*ChainlinkReader, error) {
    parsedABI, err := abi.JSON(strings.NewReader(AggregatorV3ABI))
    if err != nil {
        return nil, fmt.Errorf("failed to parse ABI: %w", err)
    }

    return &ChainlinkReader{
        client: client,
        abi:    parsedABI,
    }, nil
}

// GetLatestPrice 获取最新价格
func (cr *ChainlinkReader) GetLatestPrice(ctx context.Context, feedAddress common.Address) (*PriceInfo, error) {
    // 获取小数位数
    decimals, err := cr.getDecimals(ctx, feedAddress)
    if err != nil {
        return nil, fmt.Errorf("failed to get decimals: %w", err)
    }

    // 获取描述
    description, err := cr.getDescription(ctx, feedAddress)
    if err != nil {
        return nil, fmt.Errorf("failed to get description: %w", err)
    }

    // 获取最新轮次数据
    roundData, err := cr.getLatestRoundData(ctx, feedAddress)
    if err != nil {
        return nil, fmt.Errorf("failed to get latest round data: %w", err)
    }

    // 转换价格
    price := new(big.Float).SetInt(roundData.Answer)
    divisor := new(big.Float).SetInt(new(big.Int).Exp(big.NewInt(10), big.NewInt(int64(decimals)), nil))
    priceFloat, _ := new(big.Float).Quo(price, divisor).Float64()

    // 检查数据新鲜度
    currentTime := big.NewInt(time.Now().Unix())
    staleDuration := big.NewInt(3600) // 1小时
    timeDiff := new(big.Int).Sub(currentTime, roundData.UpdatedAt)
    isStale := timeDiff.Cmp(staleDuration) > 0

    return &PriceInfo{
        Price:       priceFloat,
        Decimals:    decimals,
        Description: description,
        UpdatedAt:   roundData.UpdatedAt.Int64(),
        RoundID:     roundData.RoundID,
        IsStale:     isStale,
    }, nil
}

func (cr *ChainlinkReader) getDecimals(ctx context.Context, feedAddress common.Address) (uint8, error) {
    data, err := cr.abi.Pack("decimals")
    if err != nil {
        return 0, err
    }

    result, err := cr.client.CallContract(ctx, ethereum.CallMsg{
        To:   &feedAddress,
        Data: data,
    }, nil)
    if err != nil {
        return 0, err
    }

    var decimals uint8
    err = cr.abi.UnpackIntoInterface(&decimals, "decimals", result)
    return decimals, err
}

func (cr *ChainlinkReader) getDescription(ctx context.Context, feedAddress common.Address) (string, error) {
    data, err := cr.abi.Pack("description")
    if err != nil {
        return "", err
    }

    result, err := cr.client.CallContract(ctx, ethereum.CallMsg{
        To:   &feedAddress,
        Data: data,
    }, nil)
    if err != nil {
        return "", err
    }

    var description string
    err = cr.abi.UnpackIntoInterface(&description, "description", result)
    return description, err
}

func (cr *ChainlinkReader) getLatestRoundData(ctx context.Context, feedAddress common.Address) (*RoundData, error) {
    data, err := cr.abi.Pack("latestRoundData")
    if err != nil {
        return nil, err
    }

    result, err := cr.client.CallContract(ctx, ethereum.CallMsg{
        To:   &feedAddress,
        Data: data,
    }, nil)
    if err != nil {
        return nil, err
    }

    // 解析结果
    var output struct {
        RoundId         *big.Int
        Answer          *big.Int
        StartedAt       *big.Int
        UpdatedAt       *big.Int
        AnsweredInRound *big.Int
    }

    err = cr.abi.UnpackIntoInterface(&output, "latestRoundData", result)
    if err != nil {
        return nil, err
    }

    return &RoundData{
        RoundID:         output.RoundId,
        Answer:          output.Answer,
        StartedAt:       output.StartedAt,
        UpdatedAt:       output.UpdatedAt,
        AnsweredInRound: output.AnsweredInRound,
    }, nil
}

// GetHistoricalPrice 获取历史价格
func (cr *ChainlinkReader) GetHistoricalPrice(ctx context.Context, feedAddress common.Address, roundID *big.Int) (*RoundData, error) {
    data, err := cr.abi.Pack("getRoundData", roundID)
    if err != nil {
        return nil, err
    }

    result, err := cr.client.CallContract(ctx, ethereum.CallMsg{
        To:   &feedAddress,
        Data: data,
    }, nil)
    if err != nil {
        return nil, err
    }

    var output struct {
        RoundId         *big.Int
        Answer          *big.Int
        StartedAt       *big.Int
        UpdatedAt       *big.Int
        AnsweredInRound *big.Int
    }

    err = cr.abi.UnpackIntoInterface(&output, "getRoundData", result)
    if err != nil {
        return nil, err
    }

    return &RoundData{
        RoundID:         output.RoundId,
        Answer:          output.Answer,
        StartedAt:       output.StartedAt,
        UpdatedAt:       output.UpdatedAt,
        AnsweredInRound: output.AnsweredInRound,
    }, nil
}

import "time"
```

### 82.3 Chainlink Price Feed 地址

```go
// 主网 Chainlink Price Feed 地址
var MainnetPriceFeeds = map[string]common.Address{
    // 主要交易对
    "ETH/USD":  common.HexToAddress("0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419"),
    "BTC/USD":  common.HexToAddress("0xF4030086522a5bEEa4988F8cA5B36dbC97BeE88c"),
    "LINK/USD": common.HexToAddress("0x2c1d072e956AFFC0D435Cb7AC38EF18d24d9127c"),
    "USDC/USD": common.HexToAddress("0x8fFfFfd4AfB6115b954Bd326cbe7B4BA576818f6"),
    "DAI/USD":  common.HexToAddress("0xAed0c38402a5d19df6E4c03F4E2DceD6e29c1ee9"),
    "USDT/USD": common.HexToAddress("0x3E7d1eAB13ad0104d2750B8863b489D65364e32D"),

    // DeFi 代币
    "AAVE/USD": common.HexToAddress("0x547a514d5e3769680Ce22B2361c10Ea13619e8a9"),
    "UNI/USD":  common.HexToAddress("0x553303d460EE0afB37EdFf9bE42922D8FF63220e"),
    "CRV/USD":  common.HexToAddress("0xCd627aA160A6fA45Eb793D19286F3f3A2042539A"),
    "MKR/USD":  common.HexToAddress("0xec1D1B3b0443256cc3860e24a46F108e699cF2b8"),

    // 稳定币
    "FRAX/USD": common.HexToAddress("0xB9E1E3A9feFf48998E45Fa90847ed4D467E8BcfD"),
    "LUSD/USD": common.HexToAddress("0x3D7aE7E594f2f2091Ad8798313450130d0Aba3a0"),

    // ETH 相关
    "stETH/USD":  common.HexToAddress("0xCfE54B5cD566aB89272946F602D76Ea879CAb4a8"),
    "cbETH/USD": common.HexToAddress("0xF017fcB346A1885194689bA23Eff2fE6fA5C483b"),
}

// PriceFeedRegistry 价格馈送注册表
type PriceFeedRegistry struct {
    feeds map[string]common.Address
}

func NewPriceFeedRegistry() *PriceFeedRegistry {
    return &PriceFeedRegistry{
        feeds: MainnetPriceFeeds,
    }
}

func (r *PriceFeedRegistry) GetFeedAddress(pair string) (common.Address, bool) {
    addr, ok := r.feeds[pair]
    return addr, ok
}

func (r *PriceFeedRegistry) RegisterFeed(pair string, address common.Address) {
    r.feeds[pair] = address
}
```

### 82.4 安全的价格获取

```go
// SafePriceReader 安全的价格读取器
type SafePriceReader struct {
    reader              *ChainlinkReader
    stalenessThreshold  time.Duration
    maxPriceDeviation   float64 // 允许的最大价格偏差
    previousPrices      map[common.Address]float64
}

// SafePriceConfig 安全配置
type SafePriceConfig struct {
    StalenessThreshold time.Duration
    MaxPriceDeviation  float64 // 如 0.1 表示 10%
}

func NewSafePriceReader(reader *ChainlinkReader, config SafePriceConfig) *SafePriceReader {
    return &SafePriceReader{
        reader:             reader,
        stalenessThreshold: config.StalenessThreshold,
        maxPriceDeviation:  config.MaxPriceDeviation,
        previousPrices:     make(map[common.Address]float64),
    }
}

// GetSafePrice 安全地获取价格
func (spr *SafePriceReader) GetSafePrice(ctx context.Context, feedAddress common.Address) (float64, error) {
    priceInfo, err := spr.reader.GetLatestPrice(ctx, feedAddress)
    if err != nil {
        return 0, fmt.Errorf("failed to get price: %w", err)
    }

    // 检查1: 数据新鲜度
    if priceInfo.IsStale {
        return 0, fmt.Errorf("price data is stale: last update %d", priceInfo.UpdatedAt)
    }

    // 检查2: 价格有效性
    if priceInfo.Price <= 0 {
        return 0, fmt.Errorf("invalid price: %f", priceInfo.Price)
    }

    // 检查3: 价格偏差 (与上次价格比较)
    if prevPrice, ok := spr.previousPrices[feedAddress]; ok {
        deviation := (priceInfo.Price - prevPrice) / prevPrice
        if deviation < 0 {
            deviation = -deviation
        }

        if deviation > spr.maxPriceDeviation {
            return 0, fmt.Errorf("price deviation too large: %.2f%% (previous: %f, current: %f)",
                deviation*100, prevPrice, priceInfo.Price)
        }
    }

    // 更新缓存的价格
    spr.previousPrices[feedAddress] = priceInfo.Price

    return priceInfo.Price, nil
}

// GetMultiplePrices 批量获取多个价格
func (spr *SafePriceReader) GetMultiplePrices(ctx context.Context, feeds map[string]common.Address) (map[string]float64, error) {
    prices := make(map[string]float64)
    errors := make([]string, 0)

    for pair, addr := range feeds {
        price, err := spr.GetSafePrice(ctx, addr)
        if err != nil {
            errors = append(errors, fmt.Sprintf("%s: %v", pair, err))
            continue
        }
        prices[pair] = price
    }

    if len(errors) > 0 {
        return prices, fmt.Errorf("some prices failed: %v", errors)
    }

    return prices, nil
}

// ValidateOracleResponse 验证预言机响应
func ValidateOracleResponse(roundData *RoundData, maxAge time.Duration) error {
    // 检查答案有效性
    if roundData.Answer == nil || roundData.Answer.Sign() <= 0 {
        return fmt.Errorf("invalid answer: %v", roundData.Answer)
    }

    // 检查 roundId 有效性
    if roundData.RoundID == nil || roundData.RoundID.Sign() <= 0 {
        return fmt.Errorf("invalid round ID")
    }

    // 检查回答是否在正确的轮次
    if roundData.AnsweredInRound.Cmp(roundData.RoundID) < 0 {
        return fmt.Errorf("stale price: answered in round %v, current round %v",
            roundData.AnsweredInRound, roundData.RoundID)
    }

    // 检查时间戳
    if roundData.UpdatedAt == nil || roundData.UpdatedAt.Sign() <= 0 {
        return fmt.Errorf("invalid timestamp")
    }

    currentTime := time.Now().Unix()
    updatedAt := roundData.UpdatedAt.Int64()
    age := time.Duration(currentTime-updatedAt) * time.Second

    if age > maxAge {
        return fmt.Errorf("price too old: %v (max: %v)", age, maxAge)
    }

    return nil
}
```

---

## 第83节：Uniswap TWAP 预言机

### 83.1 TWAP 原理

```
┌─────────────────────────────────────────────────────────────────┐
│                    TWAP (时间加权平均价格)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  TWAP 定义:                                                      │
│  TWAP = ∫P(t)dt / T                                             │
│                                                                 │
│  其中:                                                           │
│  - P(t) 是时刻 t 的价格                                          │
│  - T 是时间窗口长度                                              │
│                                                                 │
│  价格                                                            │
│    ▲                                                            │
│    │                     ┌───────┐                              │
│    │           ┌─────────┤       │                              │
│    │     ┌─────┤         │       └───┐                          │
│    │ ────┤     │         │           │                          │
│    │     │     │         │           └──────                    │
│    │     │     │         │                                      │
│    └─────┴─────┴─────────┴─────────────────────► 时间           │
│          t0    t1        t2         t3                          │
│                                                                 │
│  TWAP = (P0*(t1-t0) + P1*(t2-t1) + P2*(t3-t2) + ...) / T       │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Uniswap V2 累加器:                                              │
│                                                                 │
│  price0CumulativeLast += price0 * timeElapsed                   │
│  price1CumulativeLast += price1 * timeElapsed                   │
│                                                                 │
│  TWAP 计算:                                                      │
│  twap = (priceCumulativeEnd - priceCumulativeStart) / T         │
│                                                                 │
│  优点:                                                           │
│  1. 操纵成本高 (需要持续改变价格整个时间窗口)                       │
│  2. 无需额外 gas 消耗读取                                        │
│  3. 完全去中心化                                                 │
│                                                                 │
│  缺点:                                                           │
│  1. 价格滞后 (反映的是历史价格)                                   │
│  2. 在剧烈波动时可能不准确                                        │
│  3. 需要两次读取计算                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 83.2 Uniswap V2 TWAP 实现

```go
package twap

import (
    "context"
    "fmt"
    "math/big"
    "strings"
    "sync"
    "time"

    "github.com/ethereum/go-ethereum"
    "github.com/ethereum/go-ethereum/accounts/abi"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/ethclient"
)

// Uniswap V2 Pair ABI
const UniswapV2PairABI = `[
    {
        "name": "getReserves",
        "outputs": [
            {"type": "uint112", "name": "reserve0"},
            {"type": "uint112", "name": "reserve1"},
            {"type": "uint32", "name": "blockTimestampLast"}
        ],
        "inputs": [],
        "stateMutability": "view",
        "type": "function"
    },
    {
        "name": "price0CumulativeLast",
        "outputs": [{"type": "uint256"}],
        "inputs": [],
        "stateMutability": "view",
        "type": "function"
    },
    {
        "name": "price1CumulativeLast",
        "outputs": [{"type": "uint256"}],
        "inputs": [],
        "stateMutability": "view",
        "type": "function"
    },
    {
        "name": "token0",
        "outputs": [{"type": "address"}],
        "inputs": [],
        "stateMutability": "view",
        "type": "function"
    },
    {
        "name": "token1",
        "outputs": [{"type": "address"}],
        "inputs": [],
        "stateMutability": "view",
        "type": "function"
    }
]`

// UniswapV2TWAPOracle Uniswap V2 TWAP预言机
type UniswapV2TWAPOracle struct {
    client    *ethclient.Client
    abi       abi.ABI

    // 累加器快照
    snapshots map[common.Address]*CumulativeSnapshot
    mu        sync.RWMutex
}

// CumulativeSnapshot 累加器快照
type CumulativeSnapshot struct {
    Price0Cumulative *big.Int
    Price1Cumulative *big.Int
    Timestamp        uint32
    BlockNumber      uint64
}

// TWAPResult TWAP结果
type TWAPResult struct {
    Price0TWAP float64 // Token1/Token0
    Price1TWAP float64 // Token0/Token1
    TimeWindow time.Duration
    StartTime  time.Time
    EndTime    time.Time
}

// NewUniswapV2TWAPOracle 创建TWAP预言机
func NewUniswapV2TWAPOracle(client *ethclient.Client) (*UniswapV2TWAPOracle, error) {
    parsedABI, err := abi.JSON(strings.NewReader(UniswapV2PairABI))
    if err != nil {
        return nil, fmt.Errorf("failed to parse ABI: %w", err)
    }

    return &UniswapV2TWAPOracle{
        client:    client,
        abi:       parsedABI,
        snapshots: make(map[common.Address]*CumulativeSnapshot),
    }, nil
}

// TakeSnapshot 记录当前累加器快照
func (oracle *UniswapV2TWAPOracle) TakeSnapshot(ctx context.Context, pairAddress common.Address) error {
    // 获取当前累加器值
    price0Cum, err := oracle.getPrice0Cumulative(ctx, pairAddress)
    if err != nil {
        return err
    }

    price1Cum, err := oracle.getPrice1Cumulative(ctx, pairAddress)
    if err != nil {
        return err
    }

    // 获取储备和时间戳
    _, _, timestamp, err := oracle.getReserves(ctx, pairAddress)
    if err != nil {
        return err
    }

    // 获取当前区块号
    blockNumber, err := oracle.client.BlockNumber(ctx)
    if err != nil {
        return err
    }

    oracle.mu.Lock()
    oracle.snapshots[pairAddress] = &CumulativeSnapshot{
        Price0Cumulative: price0Cum,
        Price1Cumulative: price1Cum,
        Timestamp:        timestamp,
        BlockNumber:      blockNumber,
    }
    oracle.mu.Unlock()

    return nil
}

// CalculateTWAP 计算TWAP
func (oracle *UniswapV2TWAPOracle) CalculateTWAP(ctx context.Context, pairAddress common.Address) (*TWAPResult, error) {
    oracle.mu.RLock()
    startSnapshot, ok := oracle.snapshots[pairAddress]
    oracle.mu.RUnlock()

    if !ok {
        return nil, fmt.Errorf("no snapshot found for pair %s", pairAddress.Hex())
    }

    // 获取当前累加器值
    endPrice0Cum, err := oracle.getPrice0Cumulative(ctx, pairAddress)
    if err != nil {
        return nil, err
    }

    endPrice1Cum, err := oracle.getPrice1Cumulative(ctx, pairAddress)
    if err != nil {
        return nil, err
    }

    // 获取当前储备和时间戳
    reserve0, reserve1, endTimestamp, err := oracle.getReserves(ctx, pairAddress)
    if err != nil {
        return nil, err
    }

    // 计算时间差
    timeElapsed := uint32(0)
    if endTimestamp > startSnapshot.Timestamp {
        timeElapsed = endTimestamp - startSnapshot.Timestamp
    }

    if timeElapsed == 0 {
        return nil, fmt.Errorf("no time elapsed between snapshots")
    }

    // 更新累加器到当前时间 (如果自上次同步后有时间流逝)
    currentPrice0Cum := new(big.Int).Set(endPrice0Cum)
    currentPrice1Cum := new(big.Int).Set(endPrice1Cum)

    // 如果储备没有在最新区块更新，需要手动计算
    // currentPriceCumulative = lastCumulative + currentPrice * timeElapsed
    // 这里简化处理，假设已经同步

    // 计算价格差
    price0CumDiff := new(big.Int).Sub(currentPrice0Cum, startSnapshot.Price0Cumulative)
    price1CumDiff := new(big.Int).Sub(currentPrice1Cum, startSnapshot.Price1Cumulative)

    // UQ112x112 格式转换
    // 价格 = priceCumDiff / timeElapsed / 2^112
    q112 := new(big.Int).Exp(big.NewInt(2), big.NewInt(112), nil)

    // TWAP = priceCumDiff / timeElapsed
    price0TWAPRaw := new(big.Int).Div(price0CumDiff, big.NewInt(int64(timeElapsed)))
    price1TWAPRaw := new(big.Int).Div(price1CumDiff, big.NewInt(int64(timeElapsed)))

    // 转换为浮点数
    price0TWAPFloat := new(big.Float).SetInt(price0TWAPRaw)
    price1TWAPFloat := new(big.Float).SetInt(price1TWAPRaw)
    q112Float := new(big.Float).SetInt(q112)

    price0TWAP, _ := new(big.Float).Quo(price0TWAPFloat, q112Float).Float64()
    price1TWAP, _ := new(big.Float).Quo(price1TWAPFloat, q112Float).Float64()

    // 考虑代币精度差异 (这里假设都是18位)
    // 实际应用中需要根据代币精度调整

    _ = reserve0 // 用于精度调整
    _ = reserve1

    return &TWAPResult{
        Price0TWAP: price0TWAP,
        Price1TWAP: price1TWAP,
        TimeWindow: time.Duration(timeElapsed) * time.Second,
        StartTime:  time.Unix(int64(startSnapshot.Timestamp), 0),
        EndTime:    time.Unix(int64(endTimestamp), 0),
    }, nil
}

// 辅助函数
func (oracle *UniswapV2TWAPOracle) getPrice0Cumulative(ctx context.Context, pairAddress common.Address) (*big.Int, error) {
    data, err := oracle.abi.Pack("price0CumulativeLast")
    if err != nil {
        return nil, err
    }

    result, err := oracle.client.CallContract(ctx, ethereum.CallMsg{
        To:   &pairAddress,
        Data: data,
    }, nil)
    if err != nil {
        return nil, err
    }

    return new(big.Int).SetBytes(result), nil
}

func (oracle *UniswapV2TWAPOracle) getPrice1Cumulative(ctx context.Context, pairAddress common.Address) (*big.Int, error) {
    data, err := oracle.abi.Pack("price1CumulativeLast")
    if err != nil {
        return nil, err
    }

    result, err := oracle.client.CallContract(ctx, ethereum.CallMsg{
        To:   &pairAddress,
        Data: data,
    }, nil)
    if err != nil {
        return nil, err
    }

    return new(big.Int).SetBytes(result), nil
}

func (oracle *UniswapV2TWAPOracle) getReserves(ctx context.Context, pairAddress common.Address) (*big.Int, *big.Int, uint32, error) {
    data, err := oracle.abi.Pack("getReserves")
    if err != nil {
        return nil, nil, 0, err
    }

    result, err := oracle.client.CallContract(ctx, ethereum.CallMsg{
        To:   &pairAddress,
        Data: data,
    }, nil)
    if err != nil {
        return nil, nil, 0, err
    }

    var output struct {
        Reserve0           *big.Int
        Reserve1           *big.Int
        BlockTimestampLast uint32
    }

    err = oracle.abi.UnpackIntoInterface(&output, "getReserves", result)
    if err != nil {
        return nil, nil, 0, err
    }

    return output.Reserve0, output.Reserve1, output.BlockTimestampLast, nil
}
```

### 83.3 Uniswap V3 TWAP (基于 Tick 的实现)

```go
package twap

import (
    "math"
    "math/big"
)

// Uniswap V3 使用 tick 累加器代替价格累加器

// V3Observation V3观测数据
type V3Observation struct {
    BlockTimestamp                    uint32
    TickCumulative                    int64
    SecondsPerLiquidityCumulativeX128 *big.Int
    Initialized                       bool
}

// UniswapV3TWAPOracle Uniswap V3 TWAP预言机
type UniswapV3TWAPOracle struct {
    // ... 类似V2的结构
}

// CalculateTickTWAP 计算基于tick的TWAP
func CalculateTickTWAP(observations []V3Observation, secondsAgo uint32) (int, error) {
    if len(observations) < 2 {
        return 0, fmt.Errorf("need at least 2 observations")
    }

    // 找到 secondsAgo 前的观测点
    oldestObservation := observations[0]
    newestObservation := observations[len(observations)-1]

    // 计算时间差
    timeElapsed := newestObservation.BlockTimestamp - oldestObservation.BlockTimestamp
    if timeElapsed == 0 {
        return 0, fmt.Errorf("no time elapsed")
    }

    // 计算tick累加差
    tickCumulativeDiff := newestObservation.TickCumulative - oldestObservation.TickCumulative

    // 平均tick
    avgTick := int(tickCumulativeDiff / int64(timeElapsed))

    return avgTick, nil
}

// TickToPrice 将tick转换为价格
func TickToPrice(tick int, token0Decimals, token1Decimals int) float64 {
    // price = 1.0001^tick
    price := math.Pow(1.0001, float64(tick))

    // 调整精度
    decimalsDiff := token1Decimals - token0Decimals
    price *= math.Pow(10, float64(decimalsDiff))

    return price
}

// PriceToTick 将价格转换为tick
func PriceToTick(price float64) int {
    // tick = log(price) / log(1.0001)
    return int(math.Floor(math.Log(price) / math.Log(1.0001)))
}

// V3 Pool Slot0 结构
type Slot0 struct {
    SqrtPriceX96               *big.Int
    Tick                       int
    ObservationIndex           uint16
    ObservationCardinality     uint16
    ObservationCardinalityNext uint16
    FeeProtocol                uint8
    Unlocked                   bool
}

// ConsultV3Oracle 查询V3预言机
// secondsAgo: 查询多少秒前的价格
func (oracle *UniswapV3TWAPOracle) ConsultV3Oracle(
    ctx context.Context,
    poolAddress common.Address,
    secondsAgo uint32,
) (int, error) {
    // V3 提供了内置的 observe 函数
    // function observe(uint32[] calldata secondsAgos)
    //     returns (int56[] memory tickCumulatives, uint160[] memory liquidityCumulatives)

    // 调用 observe([secondsAgo, 0]) 获取两个时间点的累加器
    // TWAP tick = (tickCumulative[1] - tickCumulative[0]) / secondsAgo

    // 这里简化实现，实际需要调用合约
    return 0, nil
}
```

---

## 第84节：预言机攻击与防护

### 84.1 常见预言机攻击类型

```
┌─────────────────────────────────────────────────────────────────┐
│                    预言机攻击类型分类                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 即时价格操纵 (Flash Loan Attack)                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  攻击流程:                                               │   │
│  │  1. 闪电贷借入大量资金                                    │   │
│  │  2. 在 DEX 中大量买入/卖出,操纵即时价格                   │   │
│  │  3. 利用被操纵的价格进行恶意操作 (清算/借贷)              │   │
│  │  4. 恢复价格,归还闪电贷                                  │   │
│  │                                                          │   │
│  │  案例: bZx ($8M), Harvest Finance ($34M)                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  2. TWAP 操纵攻击                                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  攻击流程:                                               │   │
│  │  1. 在时间窗口内持续操纵价格                              │   │
│  │  2. 需要更大资本但影响更持久                              │   │
│  │  3. 通常需要跨多个区块                                    │   │
│  │                                                          │   │
│  │  防御: 更长的时间窗口、多源验证                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  3. 预言机节点攻击                                              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  - 贿赂预言机节点                                        │   │
│  │  - 攻击数据源 API                                        │   │
│  │  - 利用节点同步延迟                                      │   │
│  │                                                          │   │
│  │  防御: 多节点共识、信誉系统、质押惩罚                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  4. 过期数据攻击 (Stale Price Attack)                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  - 利用预言机更新延迟                                    │   │
│  │  - 在剧烈波动时利用旧价格                                │   │
│  │                                                          │   │
│  │  防御: 时间戳检查、心跳监控                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 84.2 价格操纵检测

```go
package oraclesecurity

import (
    "fmt"
    "math"
    "math/big"
    "time"
)

// PriceManipulationDetector 价格操纵检测器
type PriceManipulationDetector struct {
    // 历史价格窗口
    priceHistory    []PricePoint
    maxHistorySize  int

    // 检测参数
    volatilityThreshold   float64 // 波动率阈值
    deviationThreshold    float64 // 偏差阈值
    liquidityThreshold    *big.Int // 流动性阈值
}

// PricePoint 价格点
type PricePoint struct {
    Price     float64
    Timestamp time.Time
    Source    string
    Volume    *big.Int
}

// ManipulationAlert 操纵警报
type ManipulationAlert struct {
    Type        string
    Severity    string // "low", "medium", "high", "critical"
    Description string
    Price       float64
    ExpectedPrice float64
    Deviation   float64
    Timestamp   time.Time
}

// NewPriceManipulationDetector 创建检测器
func NewPriceManipulationDetector(config DetectorConfig) *PriceManipulationDetector {
    return &PriceManipulationDetector{
        priceHistory:         make([]PricePoint, 0),
        maxHistorySize:       config.MaxHistorySize,
        volatilityThreshold:  config.VolatilityThreshold,
        deviationThreshold:   config.DeviationThreshold,
        liquidityThreshold:   config.LiquidityThreshold,
    }
}

type DetectorConfig struct {
    MaxHistorySize       int
    VolatilityThreshold  float64
    DeviationThreshold   float64
    LiquidityThreshold   *big.Int
}

// AddPricePoint 添加价格点
func (d *PriceManipulationDetector) AddPricePoint(point PricePoint) {
    d.priceHistory = append(d.priceHistory, point)

    // 保持历史窗口大小
    if len(d.priceHistory) > d.maxHistorySize {
        d.priceHistory = d.priceHistory[1:]
    }
}

// DetectManipulation 检测价格操纵
func (d *PriceManipulationDetector) DetectManipulation(currentPrice float64) (*ManipulationAlert, error) {
    if len(d.priceHistory) < 10 {
        return nil, nil // 历史数据不足
    }

    // 1. 检查价格跳跃
    if alert := d.checkPriceJump(currentPrice); alert != nil {
        return alert, nil
    }

    // 2. 检查波动率异常
    if alert := d.checkVolatilityAnomaly(currentPrice); alert != nil {
        return alert, nil
    }

    // 3. 检查与历史均值的偏差
    if alert := d.checkMeanDeviation(currentPrice); alert != nil {
        return alert, nil
    }

    return nil, nil
}

// checkPriceJump 检查价格跳跃
func (d *PriceManipulationDetector) checkPriceJump(currentPrice float64) *ManipulationAlert {
    if len(d.priceHistory) == 0 {
        return nil
    }

    lastPrice := d.priceHistory[len(d.priceHistory)-1].Price
    change := (currentPrice - lastPrice) / lastPrice

    // 单次价格变化超过10%
    if math.Abs(change) > 0.10 {
        severity := "medium"
        if math.Abs(change) > 0.25 {
            severity = "high"
        }
        if math.Abs(change) > 0.50 {
            severity = "critical"
        }

        return &ManipulationAlert{
            Type:          "price_jump",
            Severity:      severity,
            Description:   fmt.Sprintf("Sudden price jump of %.2f%%", change*100),
            Price:         currentPrice,
            ExpectedPrice: lastPrice,
            Deviation:     math.Abs(change),
            Timestamp:     time.Now(),
        }
    }

    return nil
}

// checkVolatilityAnomaly 检查波动率异常
func (d *PriceManipulationDetector) checkVolatilityAnomaly(currentPrice float64) *ManipulationAlert {
    if len(d.priceHistory) < 20 {
        return nil
    }

    // 计算历史波动率
    prices := make([]float64, len(d.priceHistory))
    for i, p := range d.priceHistory {
        prices[i] = p.Price
    }

    historicalVol := calculateVolatility(prices[:len(prices)-10])

    // 计算最近波动率
    recentVol := calculateVolatility(prices[len(prices)-10:])

    // 波动率突然增加超过阈值
    if recentVol > historicalVol*d.volatilityThreshold {
        return &ManipulationAlert{
            Type:        "volatility_anomaly",
            Severity:    "medium",
            Description: fmt.Sprintf("Volatility spike: recent %.2f%% vs historical %.2f%%",
                recentVol*100, historicalVol*100),
            Price:     currentPrice,
            Deviation: recentVol / historicalVol,
            Timestamp: time.Now(),
        }
    }

    return nil
}

// checkMeanDeviation 检查均值偏差
func (d *PriceManipulationDetector) checkMeanDeviation(currentPrice float64) *ManipulationAlert {
    // 计算移动平均
    sum := 0.0
    for _, p := range d.priceHistory {
        sum += p.Price
    }
    mean := sum / float64(len(d.priceHistory))

    // 计算标准差
    sumSq := 0.0
    for _, p := range d.priceHistory {
        diff := p.Price - mean
        sumSq += diff * diff
    }
    stdDev := math.Sqrt(sumSq / float64(len(d.priceHistory)))

    // 计算 z-score
    zScore := (currentPrice - mean) / stdDev

    // z-score 超过 3 视为异常
    if math.Abs(zScore) > 3 {
        severity := "low"
        if math.Abs(zScore) > 4 {
            severity = "medium"
        }
        if math.Abs(zScore) > 5 {
            severity = "high"
        }

        return &ManipulationAlert{
            Type:          "mean_deviation",
            Severity:      severity,
            Description:   fmt.Sprintf("Price deviates %.2f std from mean", zScore),
            Price:         currentPrice,
            ExpectedPrice: mean,
            Deviation:     math.Abs(zScore),
            Timestamp:     time.Now(),
        }
    }

    return nil
}

// calculateVolatility 计算波动率 (使用对数收益率标准差)
func calculateVolatility(prices []float64) float64 {
    if len(prices) < 2 {
        return 0
    }

    // 计算对数收益率
    returns := make([]float64, len(prices)-1)
    for i := 1; i < len(prices); i++ {
        returns[i-1] = math.Log(prices[i] / prices[i-1])
    }

    // 计算标准差
    mean := 0.0
    for _, r := range returns {
        mean += r
    }
    mean /= float64(len(returns))

    variance := 0.0
    for _, r := range returns {
        diff := r - mean
        variance += diff * diff
    }
    variance /= float64(len(returns))

    return math.Sqrt(variance)
}
```

### 84.3 防护措施实现

```go
// OracleGuard 预言机防护
type OracleGuard struct {
    primaryOracle   OracleInterface
    backupOracles   []OracleInterface
    detector        *PriceManipulationDetector

    // 配置
    maxDeviation     float64       // 主副源最大偏差
    maxStaleness     time.Duration // 最大过期时间
    minConfirmations int           // 最小确认源数量
}

type OracleInterface interface {
    GetPrice(ctx context.Context, pair string) (float64, time.Time, error)
}

// GuardedPrice 受保护的价格
type GuardedPrice struct {
    Price         float64
    Timestamp     time.Time
    Source        string
    Confidence    float64
    IsManipulated bool
    Warnings      []string
}

// GetGuardedPrice 获取受保护的价格
func (og *OracleGuard) GetGuardedPrice(ctx context.Context, pair string) (*GuardedPrice, error) {
    warnings := make([]string, 0)

    // 1. 从主预言机获取价格
    primaryPrice, primaryTime, err := og.primaryOracle.GetPrice(ctx, pair)
    if err != nil {
        warnings = append(warnings, fmt.Sprintf("Primary oracle error: %v", err))
    }

    // 2. 检查价格新鲜度
    if time.Since(primaryTime) > og.maxStaleness {
        warnings = append(warnings, fmt.Sprintf("Primary price stale: %v old", time.Since(primaryTime)))
    }

    // 3. 从备用预言机获取价格
    backupPrices := make([]float64, 0)
    for i, backup := range og.backupOracles {
        price, timestamp, err := backup.GetPrice(ctx, pair)
        if err != nil {
            warnings = append(warnings, fmt.Sprintf("Backup oracle %d error: %v", i, err))
            continue
        }

        if time.Since(timestamp) <= og.maxStaleness {
            backupPrices = append(backupPrices, price)
        }
    }

    // 4. 验证价格一致性
    isManipulated := false
    confidence := 1.0

    if len(backupPrices) > 0 {
        medianBackup := calculateMedian(backupPrices)
        deviation := math.Abs(primaryPrice-medianBackup) / medianBackup

        if deviation > og.maxDeviation {
            isManipulated = true
            warnings = append(warnings, fmt.Sprintf("Price deviation %.2f%% exceeds threshold", deviation*100))
            confidence = 0.3
        }
    }

    // 5. 运行操纵检测
    og.detector.AddPricePoint(PricePoint{
        Price:     primaryPrice,
        Timestamp: primaryTime,
        Source:    "primary",
    })

    alert, _ := og.detector.DetectManipulation(primaryPrice)
    if alert != nil {
        warnings = append(warnings, alert.Description)
        if alert.Severity == "high" || alert.Severity == "critical" {
            isManipulated = true
            confidence *= 0.5
        }
    }

    // 6. 决定最终价格
    finalPrice := primaryPrice
    if isManipulated && len(backupPrices) >= og.minConfirmations {
        // 使用备用源的中位数
        finalPrice = calculateMedian(backupPrices)
        warnings = append(warnings, "Using backup oracle median due to manipulation detection")
    }

    return &GuardedPrice{
        Price:         finalPrice,
        Timestamp:     primaryTime,
        Source:        "guarded",
        Confidence:    confidence,
        IsManipulated: isManipulated,
        Warnings:      warnings,
    }, nil
}

func calculateMedian(values []float64) float64 {
    if len(values) == 0 {
        return 0
    }

    // 复制并排序
    sorted := make([]float64, len(values))
    copy(sorted, values)

    for i := 0; i < len(sorted); i++ {
        for j := i + 1; j < len(sorted); j++ {
            if sorted[i] > sorted[j] {
                sorted[i], sorted[j] = sorted[j], sorted[i]
            }
        }
    }

    mid := len(sorted) / 2
    if len(sorted)%2 == 0 {
        return (sorted[mid-1] + sorted[mid]) / 2
    }
    return sorted[mid]
}
```

### 84.4 电路断路器

```go
// CircuitBreaker 电路断路器
// 当检测到异常时暂停预言机使用
type CircuitBreaker struct {
    state           CircuitState
    failureCount    int
    lastFailure     time.Time
    lastSuccess     time.Time

    // 配置
    failureThreshold int           // 触发断开的失败次数
    resetTimeout     time.Duration // 断开后重置时间
    halfOpenMaxTries int           // 半开状态最大尝试次数

    mu sync.RWMutex
}

type CircuitState int

const (
    StateClosed CircuitState = iota // 正常状态
    StateOpen                       // 断开状态
    StateHalfOpen                   // 半开状态
)

func NewCircuitBreaker(failureThreshold int, resetTimeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        state:            StateClosed,
        failureThreshold: failureThreshold,
        resetTimeout:     resetTimeout,
        halfOpenMaxTries: 3,
    }
}

// AllowRequest 检查是否允许请求
func (cb *CircuitBreaker) AllowRequest() bool {
    cb.mu.RLock()
    defer cb.mu.RUnlock()

    switch cb.state {
    case StateClosed:
        return true
    case StateOpen:
        // 检查是否可以进入半开状态
        if time.Since(cb.lastFailure) > cb.resetTimeout {
            cb.mu.RUnlock()
            cb.mu.Lock()
            cb.state = StateHalfOpen
            cb.mu.Unlock()
            cb.mu.RLock()
            return true
        }
        return false
    case StateHalfOpen:
        return true
    }

    return false
}

// RecordSuccess 记录成功
func (cb *CircuitBreaker) RecordSuccess() {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    cb.lastSuccess = time.Now()
    cb.failureCount = 0

    if cb.state == StateHalfOpen {
        cb.state = StateClosed
    }
}

// RecordFailure 记录失败
func (cb *CircuitBreaker) RecordFailure() {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    cb.lastFailure = time.Now()
    cb.failureCount++

    if cb.failureCount >= cb.failureThreshold {
        cb.state = StateOpen
    }
}

// GetState 获取状态
func (cb *CircuitBreaker) GetState() CircuitState {
    cb.mu.RLock()
    defer cb.mu.RUnlock()
    return cb.state
}

import "sync"

// ProtectedOracleClient 受保护的预言机客户端
type ProtectedOracleClient struct {
    oracle         OracleInterface
    guard          *OracleGuard
    circuitBreaker *CircuitBreaker
}

// GetPrice 安全获取价格
func (poc *ProtectedOracleClient) GetPrice(ctx context.Context, pair string) (*GuardedPrice, error) {
    // 检查电路断路器
    if !poc.circuitBreaker.AllowRequest() {
        return nil, fmt.Errorf("circuit breaker open: oracle temporarily unavailable")
    }

    // 获取受保护的价格
    price, err := poc.guard.GetGuardedPrice(ctx, pair)
    if err != nil {
        poc.circuitBreaker.RecordFailure()
        return nil, err
    }

    // 检查是否检测到操纵
    if price.IsManipulated {
        poc.circuitBreaker.RecordFailure()
    } else {
        poc.circuitBreaker.RecordSuccess()
    }

    return price, nil
}
```

---

## 第85节：Pyth Network 预言机

### 85.1 Pyth 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Pyth Network 架构                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  特点:                                                           │
│  - 亚秒级更新 (400ms)                                           │
│  - 拉取模型 (Pull-based) - 按需获取价格                          │
│  - 置信区间 (Confidence Intervals)                              │
│  - 跨链支持                                                     │
│                                                                 │
│  数据发布者                                                       │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐              │
│  │ Jump    │ │ Virtu   │ │ GTS     │ │ 70+ 做市商│              │
│  │ Trading │ │ Financial│ │         │ │ & 交易所 │              │
│  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘              │
│       │          │          │          │                       │
│       └──────────┴──────────┴──────────┘                       │
│                      │                                          │
│                      ▼                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Pyth Network                           │   │
│  │                                                          │   │
│  │  1. 聚合价格数据                                          │   │
│  │  2. 计算置信区间                                          │   │
│  │  3. 生成签名证明                                          │   │
│  │                                                          │   │
│  │  ┌──────────────────────────────────────────────────┐   │   │
│  │  │  PriceUpdate {                                    │   │   │
│  │  │    price: int64,     // 聚合价格                  │   │   │
│  │  │    conf: uint64,     // 置信区间                  │   │   │
│  │  │    expo: int32,      // 指数                     │   │   │
│  │  │    publishTime: u64  // 发布时间                  │   │   │
│  │  │  }                                               │   │   │
│  │  └──────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                      │                                          │
│                      ▼                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  Wormhole 跨链桥                         │   │
│  │              传递签名价格更新到各链                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│       │          │          │          │                       │
│       ▼          ▼          ▼          ▼                       │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐                  │
│  │Ethereum│ │Arbitrum│ │ Solana │ │  BNB   │                  │
│  │        │ │        │ │        │ │ Chain  │                  │
│  └────────┘ └────────┘ └────────┘ └────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 85.2 Pyth 价格读取

```go
package pyth

import (
    "context"
    "encoding/hex"
    "fmt"
    "math"
    "math/big"
    "strings"

    "github.com/ethereum/go-ethereum"
    "github.com/ethereum/go-ethereum/accounts/abi"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/ethclient"
)

// Pyth Contract ABI (精简版)
const PythABI = `[
    {
        "name": "getPriceNoOlderThan",
        "inputs": [
            {"name": "id", "type": "bytes32"},
            {"name": "age", "type": "uint256"}
        ],
        "outputs": [
            {
                "components": [
                    {"name": "price", "type": "int64"},
                    {"name": "conf", "type": "uint64"},
                    {"name": "expo", "type": "int32"},
                    {"name": "publishTime", "type": "uint256"}
                ],
                "name": "",
                "type": "tuple"
            }
        ],
        "stateMutability": "view",
        "type": "function"
    },
    {
        "name": "getPrice",
        "inputs": [{"name": "id", "type": "bytes32"}],
        "outputs": [
            {
                "components": [
                    {"name": "price", "type": "int64"},
                    {"name": "conf", "type": "uint64"},
                    {"name": "expo", "type": "int32"},
                    {"name": "publishTime", "type": "uint256"}
                ],
                "name": "",
                "type": "tuple"
            }
        ],
        "stateMutability": "view",
        "type": "function"
    },
    {
        "name": "updatePriceFeeds",
        "inputs": [{"name": "updateData", "type": "bytes[]"}],
        "outputs": [],
        "stateMutability": "payable",
        "type": "function"
    },
    {
        "name": "getUpdateFee",
        "inputs": [{"name": "updateData", "type": "bytes[]"}],
        "outputs": [{"name": "feeAmount", "type": "uint256"}],
        "stateMutability": "view",
        "type": "function"
    }
]`

// PythPrice Pyth价格数据
type PythPrice struct {
    Price       int64
    Conf        uint64
    Expo        int32
    PublishTime uint64
}

// PythReader Pyth价格读取器
type PythReader struct {
    client       *ethclient.Client
    abi          abi.ABI
    pythContract common.Address
}

// Pyth 合约地址
var PythContracts = map[string]common.Address{
    "mainnet":  common.HexToAddress("0x4305FB66699C3B2702D4d05CF36551390A4c69C6"),
    "arbitrum": common.HexToAddress("0xff1a0f4744e8582DF1aE09D5611b887B6a12925C"),
    "optimism": common.HexToAddress("0xff1a0f4744e8582DF1aE09D5611b887B6a12925C"),
    "base":     common.HexToAddress("0x8250f4aF4B972684F7b336503E2D6dFeDeB1487a"),
}

// Pyth Price Feed IDs
var PythPriceFeeds = map[string]string{
    "ETH/USD":  "0xff61491a931112ddf1bd8147cd1b641375f79f5825126d665480874634fd0ace",
    "BTC/USD":  "0xe62df6c8b4a85fe1a67db44dc12de5db330f7ac66b72dc658afedf0f4a415b43",
    "USDC/USD": "0xeaa020c61cc479712813461ce153894a96a6c00b21ed0cfc2798d1f9a9e9c94a",
    "SOL/USD":  "0xef0d8b6fda2ceba41da15d4095d1da392a0d2f8ed0c6c7bc0f4cfac8c280b56d",
    "LINK/USD": "0x8ac0c70fff57e9aefdf5edf44b51d62c2d433653cbb2cf5cc06bb115af04d221",
}

// NewPythReader 创建Pyth读取器
func NewPythReader(client *ethclient.Client, network string) (*PythReader, error) {
    parsedABI, err := abi.JSON(strings.NewReader(PythABI))
    if err != nil {
        return nil, fmt.Errorf("failed to parse ABI: %w", err)
    }

    contractAddr, ok := PythContracts[network]
    if !ok {
        return nil, fmt.Errorf("unknown network: %s", network)
    }

    return &PythReader{
        client:       client,
        abi:          parsedABI,
        pythContract: contractAddr,
    }, nil
}

// GetPrice 获取价格
func (pr *PythReader) GetPrice(ctx context.Context, priceFeedID string) (*PythPrice, error) {
    // 解析 price feed ID
    id, err := hex.DecodeString(strings.TrimPrefix(priceFeedID, "0x"))
    if err != nil {
        return nil, fmt.Errorf("invalid price feed ID: %w", err)
    }

    var idBytes [32]byte
    copy(idBytes[:], id)

    data, err := pr.abi.Pack("getPrice", idBytes)
    if err != nil {
        return nil, fmt.Errorf("failed to pack data: %w", err)
    }

    result, err := pr.client.CallContract(ctx, ethereum.CallMsg{
        To:   &pr.pythContract,
        Data: data,
    }, nil)
    if err != nil {
        return nil, fmt.Errorf("contract call failed: %w", err)
    }

    var price struct {
        Price       int64
        Conf        uint64
        Expo        int32
        PublishTime *big.Int
    }

    err = pr.abi.UnpackIntoInterface(&price, "getPrice", result)
    if err != nil {
        return nil, fmt.Errorf("failed to unpack: %w", err)
    }

    return &PythPrice{
        Price:       price.Price,
        Conf:        price.Conf,
        Expo:        price.Expo,
        PublishTime: price.PublishTime.Uint64(),
    }, nil
}

// GetPriceNoOlderThan 获取不早于指定时间的价格
func (pr *PythReader) GetPriceNoOlderThan(ctx context.Context, priceFeedID string, maxAge uint64) (*PythPrice, error) {
    id, err := hex.DecodeString(strings.TrimPrefix(priceFeedID, "0x"))
    if err != nil {
        return nil, fmt.Errorf("invalid price feed ID: %w", err)
    }

    var idBytes [32]byte
    copy(idBytes[:], id)

    data, err := pr.abi.Pack("getPriceNoOlderThan", idBytes, big.NewInt(int64(maxAge)))
    if err != nil {
        return nil, fmt.Errorf("failed to pack data: %w", err)
    }

    result, err := pr.client.CallContract(ctx, ethereum.CallMsg{
        To:   &pr.pythContract,
        Data: data,
    }, nil)
    if err != nil {
        return nil, fmt.Errorf("contract call failed (price might be stale): %w", err)
    }

    var price struct {
        Price       int64
        Conf        uint64
        Expo        int32
        PublishTime *big.Int
    }

    err = pr.abi.UnpackIntoInterface(&price, "getPriceNoOlderThan", result)
    if err != nil {
        return nil, fmt.Errorf("failed to unpack: %w", err)
    }

    return &PythPrice{
        Price:       price.Price,
        Conf:        price.Conf,
        Expo:        price.Expo,
        PublishTime: price.PublishTime.Uint64(),
    }, nil
}

// ToFloat 将Pyth价格转换为浮点数
func (pp *PythPrice) ToFloat() float64 {
    return float64(pp.Price) * math.Pow(10, float64(pp.Expo))
}

// GetConfidenceRange 获取置信区间
func (pp *PythPrice) GetConfidenceRange() (float64, float64) {
    price := pp.ToFloat()
    conf := float64(pp.Conf) * math.Pow(10, float64(pp.Expo))

    return price - conf, price + conf
}

// PythPriceWithConfidence 带置信度的价格
type PythPriceWithConfidence struct {
    Price     float64
    LowPrice  float64
    HighPrice float64
    Confidence float64 // 相对置信度 conf/price
    Timestamp time.Time
}

func (pr *PythReader) GetPriceWithConfidence(ctx context.Context, priceFeedID string) (*PythPriceWithConfidence, error) {
    pythPrice, err := pr.GetPrice(ctx, priceFeedID)
    if err != nil {
        return nil, err
    }

    price := pythPrice.ToFloat()
    low, high := pythPrice.GetConfidenceRange()

    relativeConf := float64(pythPrice.Conf) / math.Abs(float64(pythPrice.Price))

    return &PythPriceWithConfidence{
        Price:      price,
        LowPrice:   low,
        HighPrice:  high,
        Confidence: relativeConf,
        Timestamp:  time.Unix(int64(pythPrice.PublishTime), 0),
    }, nil
}
```

---

## 第86节：预言机聚合策略

### 86.1 多源聚合器

```go
package aggregator

import (
    "context"
    "fmt"
    "math"
    "sort"
    "sync"
    "time"
)

// OracleSource 预言机源
type OracleSource struct {
    Name     string
    Weight   float64 // 权重
    Priority int     // 优先级 (越小越优先)
    GetPrice func(ctx context.Context, pair string) (float64, time.Time, error)
}

// MultiSourceAggregator 多源聚合器
type MultiSourceAggregator struct {
    sources         []*OracleSource
    aggregateMethod AggregateMethod
    minSources      int
    maxStaleness    time.Duration
}

type AggregateMethod int

const (
    AggregateMedian AggregateMethod = iota
    AggregateWeightedMean
    AggregateTrimmedMean
    AggregateMode
)

// AggregatedPrice 聚合后的价格
type AggregatedPrice struct {
    Price          float64
    Timestamp      time.Time
    SourcesUsed    int
    TotalSources   int
    Confidence     float64
    IndividualPrices map[string]float64
    Method         string
}

// NewMultiSourceAggregator 创建多源聚合器
func NewMultiSourceAggregator(sources []*OracleSource, method AggregateMethod, minSources int, maxStaleness time.Duration) *MultiSourceAggregator {
    return &MultiSourceAggregator{
        sources:         sources,
        aggregateMethod: method,
        minSources:      minSources,
        maxStaleness:    maxStaleness,
    }
}

// GetAggregatedPrice 获取聚合价格
func (msa *MultiSourceAggregator) GetAggregatedPrice(ctx context.Context, pair string) (*AggregatedPrice, error) {
    // 并发获取所有源的价格
    type sourceResult struct {
        source *OracleSource
        price  float64
        time   time.Time
        err    error
    }

    results := make(chan sourceResult, len(msa.sources))
    var wg sync.WaitGroup

    for _, source := range msa.sources {
        wg.Add(1)
        go func(src *OracleSource) {
            defer wg.Done()

            price, timestamp, err := src.GetPrice(ctx, pair)
            results <- sourceResult{
                source: src,
                price:  price,
                time:   timestamp,
                err:    err,
            }
        }(source)
    }

    wg.Wait()
    close(results)

    // 收集有效价格
    validPrices := make([]float64, 0)
    weights := make([]float64, 0)
    individualPrices := make(map[string]float64)
    var latestTimestamp time.Time

    for result := range results {
        if result.err != nil {
            continue
        }

        // 检查新鲜度
        if time.Since(result.time) > msa.maxStaleness {
            continue
        }

        validPrices = append(validPrices, result.price)
        weights = append(weights, result.source.Weight)
        individualPrices[result.source.Name] = result.price

        if result.time.After(latestTimestamp) {
            latestTimestamp = result.time
        }
    }

    if len(validPrices) < msa.minSources {
        return nil, fmt.Errorf("insufficient sources: got %d, need %d", len(validPrices), msa.minSources)
    }

    // 聚合价格
    var aggregatedPrice float64
    var method string

    switch msa.aggregateMethod {
    case AggregateMedian:
        aggregatedPrice = msa.calculateMedian(validPrices)
        method = "median"
    case AggregateWeightedMean:
        aggregatedPrice = msa.calculateWeightedMean(validPrices, weights)
        method = "weighted_mean"
    case AggregateTrimmedMean:
        aggregatedPrice = msa.calculateTrimmedMean(validPrices, 0.1) // 去掉10%的极端值
        method = "trimmed_mean"
    default:
        aggregatedPrice = msa.calculateMedian(validPrices)
        method = "median"
    }

    // 计算置信度 (基于价格离散程度)
    confidence := msa.calculateConfidence(validPrices, aggregatedPrice)

    return &AggregatedPrice{
        Price:            aggregatedPrice,
        Timestamp:        latestTimestamp,
        SourcesUsed:      len(validPrices),
        TotalSources:     len(msa.sources),
        Confidence:       confidence,
        IndividualPrices: individualPrices,
        Method:           method,
    }, nil
}

func (msa *MultiSourceAggregator) calculateMedian(prices []float64) float64 {
    sorted := make([]float64, len(prices))
    copy(sorted, prices)
    sort.Float64s(sorted)

    mid := len(sorted) / 2
    if len(sorted)%2 == 0 {
        return (sorted[mid-1] + sorted[mid]) / 2
    }
    return sorted[mid]
}

func (msa *MultiSourceAggregator) calculateWeightedMean(prices, weights []float64) float64 {
    totalWeight := 0.0
    weightedSum := 0.0

    for i, price := range prices {
        weightedSum += price * weights[i]
        totalWeight += weights[i]
    }

    if totalWeight == 0 {
        return msa.calculateMedian(prices)
    }

    return weightedSum / totalWeight
}

func (msa *MultiSourceAggregator) calculateTrimmedMean(prices []float64, trimPercent float64) float64 {
    sorted := make([]float64, len(prices))
    copy(sorted, prices)
    sort.Float64s(sorted)

    // 计算要去掉的数量
    trimCount := int(float64(len(sorted)) * trimPercent)
    if trimCount*2 >= len(sorted) {
        return msa.calculateMedian(prices)
    }

    // 去掉两端的极端值
    trimmed := sorted[trimCount : len(sorted)-trimCount]

    sum := 0.0
    for _, p := range trimmed {
        sum += p
    }

    return sum / float64(len(trimmed))
}

func (msa *MultiSourceAggregator) calculateConfidence(prices []float64, aggregated float64) float64 {
    if len(prices) == 0 {
        return 0
    }

    // 计算相对标准差
    sumSqDiff := 0.0
    for _, p := range prices {
        diff := (p - aggregated) / aggregated
        sumSqDiff += diff * diff
    }

    relStdDev := math.Sqrt(sumSqDiff / float64(len(prices)))

    // 置信度 = 1 / (1 + 相对标准差 * 放大系数)
    confidence := 1.0 / (1.0 + relStdDev*10)

    // 也考虑源的数量
    sourceRatio := float64(len(prices)) / float64(len(msa.sources))
    confidence *= (0.5 + 0.5*sourceRatio)

    return math.Min(confidence, 1.0)
}
```

### 86.2 优先级故障转移

```go
// FallbackOracle 故障转移预言机
type FallbackOracle struct {
    sources      []*OracleSource // 按优先级排序
    maxStaleness time.Duration
    cacheTime    time.Duration

    // 缓存
    cache     map[string]*CachedPrice
    cacheMu   sync.RWMutex
}

type CachedPrice struct {
    Price     float64
    Source    string
    Timestamp time.Time
    CachedAt  time.Time
}

func NewFallbackOracle(sources []*OracleSource, maxStaleness, cacheTime time.Duration) *FallbackOracle {
    // 按优先级排序
    sort.Slice(sources, func(i, j int) bool {
        return sources[i].Priority < sources[j].Priority
    })

    return &FallbackOracle{
        sources:      sources,
        maxStaleness: maxStaleness,
        cacheTime:    cacheTime,
        cache:        make(map[string]*CachedPrice),
    }
}

// GetPrice 获取价格 (按优先级尝试)
func (fo *FallbackOracle) GetPrice(ctx context.Context, pair string) (float64, string, error) {
    // 检查缓存
    fo.cacheMu.RLock()
    if cached, ok := fo.cache[pair]; ok {
        if time.Since(cached.CachedAt) < fo.cacheTime {
            fo.cacheMu.RUnlock()
            return cached.Price, cached.Source + " (cached)", nil
        }
    }
    fo.cacheMu.RUnlock()

    // 按优先级尝试每个源
    var lastErr error
    for _, source := range fo.sources {
        price, timestamp, err := source.GetPrice(ctx, pair)
        if err != nil {
            lastErr = err
            continue
        }

        // 检查新鲜度
        if time.Since(timestamp) > fo.maxStaleness {
            lastErr = fmt.Errorf("price from %s is stale: %v old", source.Name, time.Since(timestamp))
            continue
        }

        // 成功获取价格，更新缓存
        fo.cacheMu.Lock()
        fo.cache[pair] = &CachedPrice{
            Price:     price,
            Source:    source.Name,
            Timestamp: timestamp,
            CachedAt:  time.Now(),
        }
        fo.cacheMu.Unlock()

        return price, source.Name, nil
    }

    return 0, "", fmt.Errorf("all oracle sources failed: %v", lastErr)
}

// GetPriceWithFallback 获取价格并记录使用的源
type PriceResult struct {
    Price      float64
    Source     string
    Timestamp  time.Time
    Fallbacks  []string // 失败的源
    IsFallback bool     // 是否使用了降级源
}

func (fo *FallbackOracle) GetPriceWithFallback(ctx context.Context, pair string) (*PriceResult, error) {
    var fallbacks []string

    for i, source := range fo.sources {
        price, timestamp, err := source.GetPrice(ctx, pair)
        if err != nil {
            fallbacks = append(fallbacks, source.Name)
            continue
        }

        if time.Since(timestamp) > fo.maxStaleness {
            fallbacks = append(fallbacks, source.Name)
            continue
        }

        return &PriceResult{
            Price:      price,
            Source:     source.Name,
            Timestamp:  timestamp,
            Fallbacks:  fallbacks,
            IsFallback: i > 0,
        }, nil
    }

    return nil, fmt.Errorf("all sources failed: %v", fallbacks)
}
```

### 86.3 跨协议价格验证

```go
// CrossProtocolValidator 跨协议价格验证器
type CrossProtocolValidator struct {
    chainlinkReader *ChainlinkReader
    pythReader      *PythReader
    uniswapTWAP     *UniswapV2TWAPOracle

    maxDeviation float64 // 最大允许偏差
}

// ValidationResult 验证结果
type ValidationResult struct {
    IsValid        bool
    ChainlinkPrice float64
    PythPrice      float64
    TWAPPrice      float64
    MaxDeviation   float64
    Errors         []string
}

// ValidatePrice 验证价格
func (cpv *CrossProtocolValidator) ValidatePrice(
    ctx context.Context,
    chainlinkFeed common.Address,
    pythFeedID string,
    uniswapPair common.Address,
) (*ValidationResult, error) {
    result := &ValidationResult{
        IsValid: true,
        Errors:  make([]string, 0),
    }

    prices := make([]float64, 0)

    // 1. 获取 Chainlink 价格
    chainlinkPrice, err := cpv.chainlinkReader.GetLatestPrice(ctx, chainlinkFeed)
    if err != nil {
        result.Errors = append(result.Errors, fmt.Sprintf("Chainlink error: %v", err))
    } else {
        result.ChainlinkPrice = chainlinkPrice.Price
        prices = append(prices, chainlinkPrice.Price)
    }

    // 2. 获取 Pyth 价格
    pythPrice, err := cpv.pythReader.GetPriceWithConfidence(ctx, pythFeedID)
    if err != nil {
        result.Errors = append(result.Errors, fmt.Sprintf("Pyth error: %v", err))
    } else {
        result.PythPrice = pythPrice.Price
        prices = append(prices, pythPrice.Price)
    }

    // 3. 获取 TWAP 价格
    twapResult, err := cpv.uniswapTWAP.CalculateTWAP(ctx, uniswapPair)
    if err != nil {
        result.Errors = append(result.Errors, fmt.Sprintf("TWAP error: %v", err))
    } else {
        result.TWAPPrice = twapResult.Price0TWAP
        prices = append(prices, twapResult.Price0TWAP)
    }

    // 计算偏差
    if len(prices) >= 2 {
        minPrice := prices[0]
        maxPrice := prices[0]
        for _, p := range prices {
            if p < minPrice {
                minPrice = p
            }
            if p > maxPrice {
                maxPrice = p
            }
        }

        result.MaxDeviation = (maxPrice - minPrice) / minPrice

        if result.MaxDeviation > cpv.maxDeviation {
            result.IsValid = false
            result.Errors = append(result.Errors,
                fmt.Sprintf("Price deviation %.2f%% exceeds threshold %.2f%%",
                    result.MaxDeviation*100, cpv.maxDeviation*100))
        }
    }

    return result, nil
}
```

---

## 总结

本文档详细介绍了预言机机制的核心概念和实现：

1. **预言机基础** (第81节)
   - 预言机问题的本质
   - 预言机类型分类
   - 数据结构设计

2. **Chainlink 预言机** (第82节)
   - 架构详解
   - 价格读取实现
   - 安全的价格获取

3. **Uniswap TWAP** (第83节)
   - TWAP 原理
   - V2/V3 实现
   - 使用场景

4. **预言机安全** (第84节)
   - 攻击类型分析
   - 操纵检测
   - 防护措施
   - 电路断路器

5. **Pyth Network** (第85节)
   - 架构特点
   - 置信区间
   - 拉取模型

6. **聚合策略** (第86节)
   - 多源聚合
   - 故障转移
   - 跨协议验证

预言机是 DeFi 协议的关键基础设施，正确使用和保护预言机对于协议安全至关重要。