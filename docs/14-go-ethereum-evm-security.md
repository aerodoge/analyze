# Go-Ethereum EVM 深度解析：DeFi 安全与漏洞防护

## 第一百节：常见智能合约漏洞

### 漏洞分类概览

```
DeFi 安全漏洞分类：
┌─────────────────────────────────────────────────────────────────┐
│                    智能合约安全漏洞                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   重入攻击       │  │   整数溢出       │  │   访问控制       │ │
│  │  Reentrancy     │  │  Integer Over   │  │  Access Control │ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘ │
│           │                    │                    │           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   价格操纵       │  │   闪电贷攻击     │  │   预言机操纵     │ │
│  │ Price Manipulat │  │  Flash Loan     │  │ Oracle Manipul  │ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘ │
│           │                    │                    │           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   前端攻击       │  │   委托调用       │  │   签名重放       │ │
│  │  Front-running  │  │  Delegatecall   │  │ Signature Replay│ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 重入攻击检测

```go
package security

import (
    "context"
    "fmt"
    "math/big"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/vm"
)

// ReentrancyDetector 重入攻击检测器
type ReentrancyDetector struct {
    // 调用栈追踪
    callStack    []CallFrame
    // 外部调用记录
    externalCalls map[common.Address]int
    // 状态修改记录
    stateChanges  []StateChange
    // 检测结果
    vulnerabilities []Vulnerability
}

// CallFrame 调用帧
type CallFrame struct {
    Caller    common.Address
    Callee    common.Address
    Value     *big.Int
    Input     []byte
    Depth     int
    IsStatic  bool
}

// StateChange 状态变更
type StateChange struct {
    Contract common.Address
    Slot     common.Hash
    OldValue common.Hash
    NewValue common.Hash
    Depth    int
}

// Vulnerability 漏洞信息
type Vulnerability struct {
    Type        VulnerabilityType
    Severity    Severity
    Contract    common.Address
    Description string
    Location    string
    Recommendation string
}

// VulnerabilityType 漏洞类型
type VulnerabilityType int

const (
    VulnReentrancy VulnerabilityType = iota
    VulnIntegerOverflow
    VulnAccessControl
    VulnPriceManipulation
    VulnOracleManipulation
    VulnDelegateCall
    VulnSignatureReplay
    VulnFrontRunning
)

// Severity 严重程度
type Severity int

const (
    SeverityInfo Severity = iota
    SeverityLow
    SeverityMedium
    SeverityHigh
    SeverityCritical
)

// NewReentrancyDetector 创建检测器
func NewReentrancyDetector() *ReentrancyDetector {
    return &ReentrancyDetector{
        callStack:       make([]CallFrame, 0),
        externalCalls:   make(map[common.Address]int),
        stateChanges:    make([]StateChange, 0),
        vulnerabilities: make([]Vulnerability, 0),
    }
}

// OnCallEnter 进入调用
func (d *ReentrancyDetector) OnCallEnter(
    caller common.Address,
    callee common.Address,
    value *big.Int,
    input []byte,
    depth int,
    isStatic bool,
) {
    frame := CallFrame{
        Caller:   caller,
        Callee:   callee,
        Value:    value,
        Input:    input,
        Depth:    depth,
        IsStatic: isStatic,
    }

    d.callStack = append(d.callStack, frame)

    // 检测重入
    if d.detectReentrancy(callee, depth) {
        d.vulnerabilities = append(d.vulnerabilities, Vulnerability{
            Type:        VulnReentrancy,
            Severity:    SeverityCritical,
            Contract:    callee,
            Description: fmt.Sprintf("检测到重入调用: %s 被多次调用", callee.Hex()),
            Recommendation: "使用 ReentrancyGuard 或 checks-effects-interactions 模式",
        })
    }

    // 记录外部调用
    d.externalCalls[callee]++
}

// OnCallExit 退出调用
func (d *ReentrancyDetector) OnCallExit(depth int) {
    if len(d.callStack) > 0 {
        d.callStack = d.callStack[:len(d.callStack)-1]
    }
}

// OnStorageWrite 存储写入
func (d *ReentrancyDetector) OnStorageWrite(
    contract common.Address,
    slot common.Hash,
    oldValue common.Hash,
    newValue common.Hash,
    depth int,
) {
    d.stateChanges = append(d.stateChanges, StateChange{
        Contract: contract,
        Slot:     slot,
        OldValue: oldValue,
        NewValue: newValue,
        Depth:    depth,
    })

    // 检测 "读后写" 模式在外部调用之后
    if d.detectWriteAfterExternalCall(contract, depth) {
        d.vulnerabilities = append(d.vulnerabilities, Vulnerability{
            Type:        VulnReentrancy,
            Severity:    SeverityHigh,
            Contract:    contract,
            Description: "在外部调用后写入状态，可能存在重入风险",
            Recommendation: "将状态更新移到外部调用之前",
        })
    }
}

// detectReentrancy 检测重入
func (d *ReentrancyDetector) detectReentrancy(callee common.Address, depth int) bool {
    // 检查调用栈中是否已存在该合约
    for _, frame := range d.callStack[:len(d.callStack)-1] {
        if frame.Callee == callee {
            return true
        }
    }
    return false
}

// detectWriteAfterExternalCall 检测外部调用后写入
func (d *ReentrancyDetector) detectWriteAfterExternalCall(contract common.Address, depth int) bool {
    // 检查当前调用深度之前是否有外部调用
    for _, frame := range d.callStack {
        if frame.Depth < depth && frame.Callee != contract {
            // 存在外部调用在状态写入之前
            return true
        }
    }
    return false
}

// GetVulnerabilities 获取检测到的漏洞
func (d *ReentrancyDetector) GetVulnerabilities() []Vulnerability {
    return d.vulnerabilities
}

// AnalyzeTransaction 分析交易
func (d *ReentrancyDetector) AnalyzeTransaction(
    ctx context.Context,
    tracer *TransactionTracer,
) ([]Vulnerability, error) {
    traces := tracer.GetTraces()

    for _, trace := range traces {
        switch trace.Type {
        case TraceCall:
            d.OnCallEnter(
                trace.From,
                trace.To,
                trace.Value,
                trace.Input,
                trace.Depth,
                trace.IsStatic,
            )
        case TraceCallReturn:
            d.OnCallExit(trace.Depth)
        case TraceStorage:
            d.OnStorageWrite(
                trace.Contract,
                trace.Slot,
                trace.OldValue,
                trace.NewValue,
                trace.Depth,
            )
        }
    }

    return d.GetVulnerabilities(), nil
}

// TraceType 追踪类型
type TraceType int

const (
    TraceCall TraceType = iota
    TraceCallReturn
    TraceStorage
)

// Trace 追踪记录
type Trace struct {
    Type     TraceType
    From     common.Address
    To       common.Address
    Value    *big.Int
    Input    []byte
    Depth    int
    IsStatic bool
    Contract common.Address
    Slot     common.Hash
    OldValue common.Hash
    NewValue common.Hash
}

// TransactionTracer 交易追踪器
type TransactionTracer struct {
    traces []Trace
}

// GetTraces 获取追踪记录
func (t *TransactionTracer) GetTraces() []Trace {
    return t.traces
}
```

### 整数溢出检测

```go
package security

import (
    "math/big"
)

// IntegerOverflowDetector 整数溢出检测器
type IntegerOverflowDetector struct {
    maxUint256 *big.Int
    maxInt256  *big.Int
    minInt256  *big.Int
}

// NewIntegerOverflowDetector 创建检测器
func NewIntegerOverflowDetector() *IntegerOverflowDetector {
    maxUint256 := new(big.Int).Sub(
        new(big.Int).Exp(big.NewInt(2), big.NewInt(256), nil),
        big.NewInt(1),
    )

    maxInt256 := new(big.Int).Sub(
        new(big.Int).Exp(big.NewInt(2), big.NewInt(255), nil),
        big.NewInt(1),
    )

    minInt256 := new(big.Int).Neg(
        new(big.Int).Exp(big.NewInt(2), big.NewInt(255), nil),
    )

    return &IntegerOverflowDetector{
        maxUint256: maxUint256,
        maxInt256:  maxInt256,
        minInt256:  minInt256,
    }
}

// CheckAddOverflow 检查加法溢出 (unsigned)
func (d *IntegerOverflowDetector) CheckAddOverflow(a, b *big.Int) bool {
    result := new(big.Int).Add(a, b)
    return result.Cmp(d.maxUint256) > 0
}

// CheckSubUnderflow 检查减法下溢 (unsigned)
func (d *IntegerOverflowDetector) CheckSubUnderflow(a, b *big.Int) bool {
    return a.Cmp(b) < 0
}

// CheckMulOverflow 检查乘法溢出 (unsigned)
func (d *IntegerOverflowDetector) CheckMulOverflow(a, b *big.Int) bool {
    if a.Sign() == 0 || b.Sign() == 0 {
        return false
    }
    result := new(big.Int).Mul(a, b)
    return result.Cmp(d.maxUint256) > 0
}

// CheckSignedAddOverflow 检查有符号加法溢出
func (d *IntegerOverflowDetector) CheckSignedAddOverflow(a, b *big.Int) bool {
    result := new(big.Int).Add(a, b)

    // 正数 + 正数 = 负数 -> 溢出
    if a.Sign() > 0 && b.Sign() > 0 && result.Sign() < 0 {
        return true
    }

    // 负数 + 负数 = 正数 -> 下溢
    if a.Sign() < 0 && b.Sign() < 0 && result.Sign() > 0 {
        return true
    }

    return result.Cmp(d.maxInt256) > 0 || result.Cmp(d.minInt256) < 0
}

// AnalyzeArithmetic 分析算术操作
func (d *IntegerOverflowDetector) AnalyzeArithmetic(
    opcode byte,
    operands []*big.Int,
) *Vulnerability {
    switch opcode {
    case 0x01: // ADD
        if len(operands) >= 2 && d.CheckAddOverflow(operands[0], operands[1]) {
            return &Vulnerability{
                Type:           VulnIntegerOverflow,
                Severity:       SeverityHigh,
                Description:    "检测到无符号整数加法溢出",
                Recommendation: "使用 SafeMath 或 Solidity 0.8+ 的内置检查",
            }
        }

    case 0x03: // SUB
        if len(operands) >= 2 && d.CheckSubUnderflow(operands[0], operands[1]) {
            return &Vulnerability{
                Type:           VulnIntegerOverflow,
                Severity:       SeverityHigh,
                Description:    "检测到无符号整数减法下溢",
                Recommendation: "使用 SafeMath 或 Solidity 0.8+ 的内置检查",
            }
        }

    case 0x02: // MUL
        if len(operands) >= 2 && d.CheckMulOverflow(operands[0], operands[1]) {
            return &Vulnerability{
                Type:           VulnIntegerOverflow,
                Severity:       SeverityHigh,
                Description:    "检测到无符号整数乘法溢出",
                Recommendation: "使用 SafeMath 或 Solidity 0.8+ 的内置检查",
            }
        }
    }

    return nil
}
```

---

## 第一百零一节：价格操纵与闪电贷攻击

### 价格操纵检测

```go
package security

import (
    "context"
    "fmt"
    "math/big"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
)

// PriceManipulationDetector 价格操纵检测器
type PriceManipulationDetector struct {
    // 价格历史
    priceHistory map[common.Address][]PricePoint
    // 配置
    config *PriceDetectorConfig
}

// PriceDetectorConfig 检测器配置
type PriceDetectorConfig struct {
    MaxPriceDeviation  float64       // 最大价格偏差百分比
    MinLookbackPeriod  time.Duration // 最小回溯期
    MaxPriceImpact     float64       // 单笔交易最大价格影响
    FlashloanThreshold *big.Int      // 闪电贷阈值
}

// PricePoint 价格点
type PricePoint struct {
    Price       *big.Float
    BlockNumber uint64
    Timestamp   time.Time
    Volume      *big.Int
}

// PriceManipulationAlert 价格操纵警报
type PriceManipulationAlert struct {
    Token         common.Address
    Type          ManipulationType
    Severity      Severity
    PriceBefore   *big.Float
    PriceAfter    *big.Float
    PriceChange   float64
    BlockNumber   uint64
    TxHash        common.Hash
    Description   string
}

// ManipulationType 操纵类型
type ManipulationType int

const (
    ManipulationFlashLoan ManipulationType = iota
    ManipulationSandwich
    ManipulationPumpDump
    ManipulationOracleAttack
)

// NewPriceManipulationDetector 创建检测器
func NewPriceManipulationDetector(config *PriceDetectorConfig) *PriceManipulationDetector {
    return &PriceManipulationDetector{
        priceHistory: make(map[common.Address][]PricePoint),
        config:       config,
    }
}

// RecordPrice 记录价格
func (d *PriceManipulationDetector) RecordPrice(
    token common.Address,
    price *big.Float,
    blockNumber uint64,
    volume *big.Int,
) {
    point := PricePoint{
        Price:       price,
        BlockNumber: blockNumber,
        Timestamp:   time.Now(),
        Volume:      volume,
    }

    d.priceHistory[token] = append(d.priceHistory[token], point)

    // 保留最近的价格历史
    if len(d.priceHistory[token]) > 1000 {
        d.priceHistory[token] = d.priceHistory[token][1:]
    }
}

// DetectManipulation 检测价格操纵
func (d *PriceManipulationDetector) DetectManipulation(
    ctx context.Context,
    tx *types.Transaction,
    receipt *types.Receipt,
) (*PriceManipulationAlert, error) {
    // 分析交易中的价格变化
    priceChanges := d.extractPriceChanges(receipt)

    for token, change := range priceChanges {
        // 检查价格变化是否异常
        if d.isAbnormalPriceChange(token, change) {
            manipType := d.classifyManipulation(tx, change)

            return &PriceManipulationAlert{
                Token:        token,
                Type:         manipType,
                Severity:     d.calculateSeverity(change),
                PriceBefore:  change.Before,
                PriceAfter:   change.After,
                PriceChange:  change.Percentage,
                BlockNumber:  receipt.BlockNumber.Uint64(),
                TxHash:       tx.Hash(),
                Description:  d.generateDescription(manipType, change),
            }, nil
        }
    }

    return nil, nil
}

// PriceChange 价格变化
type PriceChange struct {
    Before     *big.Float
    After      *big.Float
    Percentage float64
}

// extractPriceChanges 从收据中提取价格变化
func (d *PriceManipulationDetector) extractPriceChanges(receipt *types.Receipt) map[common.Address]*PriceChange {
    changes := make(map[common.Address]*PriceChange)

    // 分析 Swap 事件
    swapTopic := common.HexToHash("0xd78ad95fa46c994b6551d0da85fc275fe613ce37657fb8d5e3d130840159d822")
    syncTopic := common.HexToHash("0x1c411e9a96e071241c2f21f7726b17ae89e3cab4c78be50e062b03a9fffbbad1")

    for _, log := range receipt.Logs {
        if len(log.Topics) > 0 {
            if log.Topics[0] == syncTopic {
                // 从 Sync 事件提取新的储备
                if len(log.Data) >= 64 {
                    reserve0 := new(big.Int).SetBytes(log.Data[:32])
                    reserve1 := new(big.Int).SetBytes(log.Data[32:64])

                    // 计算新价格
                    price := new(big.Float).Quo(
                        new(big.Float).SetInt(reserve1),
                        new(big.Float).SetInt(reserve0),
                    )

                    // 获取历史价格比较
                    if history, exists := d.priceHistory[log.Address]; exists && len(history) > 0 {
                        lastPrice := history[len(history)-1].Price
                        change := d.calculatePriceChange(lastPrice, price)
                        changes[log.Address] = change
                    }
                }
            }
        }
    }

    return changes
}

// calculatePriceChange 计算价格变化
func (d *PriceManipulationDetector) calculatePriceChange(before, after *big.Float) *PriceChange {
    diff := new(big.Float).Sub(after, before)
    percentChange := new(big.Float).Quo(diff, before)
    percentChangeFloat, _ := percentChange.Float64()

    return &PriceChange{
        Before:     before,
        After:      after,
        Percentage: percentChangeFloat * 100,
    }
}

// isAbnormalPriceChange 判断是否异常价格变化
func (d *PriceManipulationDetector) isAbnormalPriceChange(
    token common.Address,
    change *PriceChange,
) bool {
    // 价格变化超过阈值
    if change.Percentage > d.config.MaxPriceDeviation ||
       change.Percentage < -d.config.MaxPriceDeviation {
        return true
    }

    // 检查历史波动率
    history := d.priceHistory[token]
    if len(history) > 10 {
        volatility := d.calculateVolatility(history)
        // 如果变化超过 3 倍标准差
        if change.Percentage > volatility*3 || change.Percentage < -volatility*3 {
            return true
        }
    }

    return false
}

// calculateVolatility 计算波动率
func (d *PriceManipulationDetector) calculateVolatility(history []PricePoint) float64 {
    if len(history) < 2 {
        return 0
    }

    var returns []float64
    for i := 1; i < len(history); i++ {
        prev, _ := history[i-1].Price.Float64()
        curr, _ := history[i].Price.Float64()
        if prev > 0 {
            returns = append(returns, (curr-prev)/prev*100)
        }
    }

    // 计算标准差
    var sum, sumSq float64
    for _, r := range returns {
        sum += r
        sumSq += r * r
    }
    n := float64(len(returns))
    mean := sum / n
    variance := sumSq/n - mean*mean

    if variance < 0 {
        return 0
    }

    return variance // 简化，实际应返回 sqrt(variance)
}

// classifyManipulation 分类操纵类型
func (d *PriceManipulationDetector) classifyManipulation(
    tx *types.Transaction,
    change *PriceChange,
) ManipulationType {
    // 检测闪电贷特征
    if d.detectFlashloanPattern(tx) {
        return ManipulationFlashLoan
    }

    // 检测三明治特征
    // 需要分析前后区块

    // 默认归类为拉高出货
    if change.Percentage > 0 {
        return ManipulationPumpDump
    }

    return ManipulationOracleAttack
}

// detectFlashloanPattern 检测闪电贷模式
func (d *PriceManipulationDetector) detectFlashloanPattern(tx *types.Transaction) bool {
    // 检查交易是否调用了闪电贷接口
    if len(tx.Data()) < 4 {
        return false
    }

    selector := common.Bytes2Hex(tx.Data()[:4])

    // 常见闪电贷函数选择器
    flashloanSelectors := map[string]bool{
        "ab9c4b5d": true, // Aave flashLoan
        "5cffe9de": true, // dYdX operate
        "c45a0155": true, // Uniswap V3 flash
    }

    return flashloanSelectors[selector]
}

// calculateSeverity 计算严重程度
func (d *PriceManipulationDetector) calculateSeverity(change *PriceChange) Severity {
    absChange := change.Percentage
    if absChange < 0 {
        absChange = -absChange
    }

    switch {
    case absChange > 50:
        return SeverityCritical
    case absChange > 20:
        return SeverityHigh
    case absChange > 10:
        return SeverityMedium
    case absChange > 5:
        return SeverityLow
    default:
        return SeverityInfo
    }
}

// generateDescription 生成描述
func (d *PriceManipulationDetector) generateDescription(
    manipType ManipulationType,
    change *PriceChange,
) string {
    typeStr := ""
    switch manipType {
    case ManipulationFlashLoan:
        typeStr = "闪电贷攻击"
    case ManipulationSandwich:
        typeStr = "三明治攻击"
    case ManipulationPumpDump:
        typeStr = "拉高出货"
    case ManipulationOracleAttack:
        typeStr = "预言机攻击"
    }

    return fmt.Sprintf("%s: 价格变化 %.2f%% (%.6f -> %.6f)",
        typeStr,
        change.Percentage,
        change.Before,
        change.After,
    )
}
```

### 闪电贷攻击模拟

```go
package security

import (
    "context"
    "fmt"
    "math/big"

    "github.com/ethereum/go-ethereum/common"
)

// FlashloanAttackSimulator 闪电贷攻击模拟器
type FlashloanAttackSimulator struct {
    chainClient ChainClient
    poolManager PoolManager
}

// ChainClient 链客户端接口
type ChainClient interface {
    SimulateTransaction(ctx context.Context, data []byte) (*SimulationResult, error)
    Call(ctx context.Context, to common.Address, data []byte) ([]byte, error)
}

// PoolManager 池管理器接口
type PoolManager interface {
    GetPoolsByToken(token common.Address) []*Pool
    CalculateOutput(pool common.Address, amountIn *big.Int, zeroForOne bool) (*big.Int, error)
}

// Pool 池信息
type Pool struct {
    Address  common.Address
    Token0   common.Address
    Token1   common.Address
    Reserve0 *big.Int
    Reserve1 *big.Int
    DEX      string
}

// SimulationResult 模拟结果
type SimulationResult struct {
    Success bool
    Return  []byte
    GasUsed uint64
    Error   string
}

// AttackVector 攻击向量
type AttackVector struct {
    Type           AttackType
    FlashloanToken common.Address
    FlashloanAmount *big.Int
    TargetPool     common.Address
    Steps          []AttackStep
    EstimatedProfit *big.Int
    GasEstimate    uint64
    Feasible       bool
}

// AttackType 攻击类型
type AttackType int

const (
    AttackPriceManipulation AttackType = iota
    AttackLiquidation
    AttackGovernance
    AttackReentrancy
)

// AttackStep 攻击步骤
type AttackStep struct {
    Action      string
    Target      common.Address
    TokenIn     common.Address
    TokenOut    common.Address
    AmountIn    *big.Int
    AmountOut   *big.Int
    Description string
}

// NewFlashloanAttackSimulator 创建模拟器
func NewFlashloanAttackSimulator(
    chainClient ChainClient,
    poolManager PoolManager,
) *FlashloanAttackSimulator {
    return &FlashloanAttackSimulator{
        chainClient: chainClient,
        poolManager: poolManager,
    }
}

// SimulatePriceManipulation 模拟价格操纵攻击
func (s *FlashloanAttackSimulator) SimulatePriceManipulation(
    ctx context.Context,
    targetPool common.Address,
    targetToken common.Address,
) (*AttackVector, error) {
    // 获取目标池信息
    pools := s.poolManager.GetPoolsByToken(targetToken)
    if len(pools) < 2 {
        return nil, fmt.Errorf("需要至少两个池来进行套利")
    }

    // 计算最优闪电贷金额
    optimalAmount := s.calculateOptimalFlashloanAmount(pools, targetToken)

    // 构建攻击步骤
    steps := []AttackStep{
        {
            Action:      "flashloan",
            Target:      common.HexToAddress("0x7d2768de32b0b80b7a3454c06bdac94a69ddc7a9"), // Aave
            TokenIn:     targetToken,
            AmountIn:    optimalAmount,
            Description: fmt.Sprintf("借入 %s %s", optimalAmount.String(), targetToken.Hex()[:10]),
        },
        {
            Action:      "swap",
            Target:      pools[0].Address,
            TokenIn:     targetToken,
            AmountIn:    optimalAmount,
            Description: "在第一个池子中交换，操纵价格",
        },
        {
            Action:      "profit",
            Target:      pools[1].Address,
            Description: "在价格变化后获利",
        },
        {
            Action:      "repay",
            Target:      common.HexToAddress("0x7d2768de32b0b80b7a3454c06bdac94a69ddc7a9"),
            TokenOut:    targetToken,
            Description: "偿还闪电贷",
        },
    }

    // 计算预期利润
    profit := s.calculateExpectedProfit(pools, targetToken, optimalAmount)

    // 估算 gas
    gasEstimate := uint64(500000)

    return &AttackVector{
        Type:            AttackPriceManipulation,
        FlashloanToken:  targetToken,
        FlashloanAmount: optimalAmount,
        TargetPool:      targetPool,
        Steps:           steps,
        EstimatedProfit: profit,
        GasEstimate:     gasEstimate,
        Feasible:        profit.Sign() > 0,
    }, nil
}

// calculateOptimalFlashloanAmount 计算最优闪电贷金额
func (s *FlashloanAttackSimulator) calculateOptimalFlashloanAmount(
    pools []*Pool,
    token common.Address,
) *big.Int {
    // 简化计算：使用最小池子储备的 10%
    minReserve := new(big.Int).Set(pools[0].Reserve0)
    if pools[0].Token1 == token {
        minReserve = new(big.Int).Set(pools[0].Reserve1)
    }

    for _, pool := range pools[1:] {
        reserve := pool.Reserve0
        if pool.Token1 == token {
            reserve = pool.Reserve1
        }
        if reserve.Cmp(minReserve) < 0 {
            minReserve = reserve
        }
    }

    // 借入储备的 10%
    return new(big.Int).Div(minReserve, big.NewInt(10))
}

// calculateExpectedProfit 计算预期利润
func (s *FlashloanAttackSimulator) calculateExpectedProfit(
    pools []*Pool,
    token common.Address,
    amount *big.Int,
) *big.Int {
    // 模拟交易路径并计算利润
    // 简化实现

    // 假设价格影响为 1%，套利利润为影响的一半
    profitRate := big.NewInt(50) // 0.5%
    profit := new(big.Int).Mul(amount, profitRate)
    profit.Div(profit, big.NewInt(10000))

    // 减去闪电贷费用 (0.09%)
    flashloanFee := new(big.Int).Mul(amount, big.NewInt(9))
    flashloanFee.Div(flashloanFee, big.NewInt(10000))

    return new(big.Int).Sub(profit, flashloanFee)
}

// SimulateAndValidate 模拟并验证攻击
func (s *FlashloanAttackSimulator) SimulateAndValidate(
    ctx context.Context,
    vector *AttackVector,
) (*AttackValidationResult, error) {
    // 构建模拟交易
    calldata := s.buildAttackCalldata(vector)

    // 执行模拟
    result, err := s.chainClient.SimulateTransaction(ctx, calldata)
    if err != nil {
        return nil, fmt.Errorf("模拟失败: %w", err)
    }

    validation := &AttackValidationResult{
        Vector:       vector,
        Simulated:    true,
        Success:      result.Success,
        ActualProfit: big.NewInt(0),
        GasUsed:      result.GasUsed,
    }

    if result.Success {
        // 从返回值解析实际利润
        if len(result.Return) >= 32 {
            validation.ActualProfit = new(big.Int).SetBytes(result.Return[:32])
        }
    } else {
        validation.Error = result.Error
    }

    return validation, nil
}

// AttackValidationResult 攻击验证结果
type AttackValidationResult struct {
    Vector       *AttackVector
    Simulated    bool
    Success      bool
    ActualProfit *big.Int
    GasUsed      uint64
    Error        string
}

// buildAttackCalldata 构建攻击 calldata
func (s *FlashloanAttackSimulator) buildAttackCalldata(vector *AttackVector) []byte {
    // 构建闪电贷攻击的 calldata
    // 实现省略
    return nil
}
```

---

## 第一百零二节：安全审计工具

### 静态分析器

```go
package security

import (
    "fmt"
    "regexp"
    "strings"
)

// StaticAnalyzer 静态分析器
type StaticAnalyzer struct {
    patterns []VulnerabilityPattern
}

// VulnerabilityPattern 漏洞模式
type VulnerabilityPattern struct {
    Name        string
    Type        VulnerabilityType
    Severity    Severity
    Pattern     *regexp.Regexp
    Description string
    Remediation string
}

// NewStaticAnalyzer 创建静态分析器
func NewStaticAnalyzer() *StaticAnalyzer {
    analyzer := &StaticAnalyzer{
        patterns: make([]VulnerabilityPattern, 0),
    }

    // 注册漏洞模式
    analyzer.registerPatterns()

    return analyzer
}

// registerPatterns 注册漏洞模式
func (a *StaticAnalyzer) registerPatterns() {
    // 重入漏洞模式
    a.patterns = append(a.patterns, VulnerabilityPattern{
        Name:        "Reentrancy - External Call Before State Update",
        Type:        VulnReentrancy,
        Severity:    SeverityCritical,
        Pattern:     regexp.MustCompile(`\.call\{.*\}\(.*\).*\n.*=`),
        Description: "在状态更新之前进行外部调用",
        Remediation: "使用 checks-effects-interactions 模式，或 ReentrancyGuard",
    })

    // tx.origin 使用
    a.patterns = append(a.patterns, VulnerabilityPattern{
        Name:        "Use of tx.origin",
        Type:        VulnAccessControl,
        Severity:    SeverityHigh,
        Pattern:     regexp.MustCompile(`tx\.origin`),
        Description: "使用 tx.origin 进行授权检查",
        Remediation: "使用 msg.sender 代替 tx.origin",
    })

    // 未检查的返回值
    a.patterns = append(a.patterns, VulnerabilityPattern{
        Name:        "Unchecked Return Value",
        Type:        VulnAccessControl,
        Severity:    SeverityMedium,
        Pattern:     regexp.MustCompile(`\.transfer\(|\.send\(`),
        Description: "未检查 transfer/send 的返回值",
        Remediation: "使用 SafeERC20 或检查返回值",
    })

    // Delegatecall 到不可信地址
    a.patterns = append(a.patterns, VulnerabilityPattern{
        Name:        "Delegatecall to Untrusted Contract",
        Type:        VulnDelegateCall,
        Severity:    SeverityCritical,
        Pattern:     regexp.MustCompile(`\.delegatecall\(`),
        Description: "使用 delegatecall 可能导致存储冲突或逻辑漏洞",
        Remediation: "仅对可信的、不可变的合约使用 delegatecall",
    })

    // 时间戳依赖
    a.patterns = append(a.patterns, VulnerabilityPattern{
        Name:        "Timestamp Dependence",
        Type:        VulnFrontRunning,
        Severity:    SeverityLow,
        Pattern:     regexp.MustCompile(`block\.timestamp|now`),
        Description: "依赖区块时间戳可能被矿工操纵",
        Remediation: "避免在关键逻辑中依赖精确时间戳",
    })

    // 弱随机数
    a.patterns = append(a.patterns, VulnerabilityPattern{
        Name:        "Weak Randomness",
        Type:        VulnFrontRunning,
        Severity:    SeverityHigh,
        Pattern:     regexp.MustCompile(`block\.difficulty|blockhash|block\.number`),
        Description: "使用可预测的值作为随机数来源",
        Remediation: "使用 Chainlink VRF 或其他安全随机数源",
    })

    // 自毁
    a.patterns = append(a.patterns, VulnerabilityPattern{
        Name:        "Selfdestruct Usage",
        Type:        VulnAccessControl,
        Severity:    SeverityMedium,
        Pattern:     regexp.MustCompile(`selfdestruct|suicide`),
        Description: "合约可以被销毁",
        Remediation: "确保只有授权用户可以调用 selfdestruct",
    })
}

// Analyze 分析合约源码
func (a *StaticAnalyzer) Analyze(sourceCode string) []Finding {
    var findings []Finding

    // 按行分析
    lines := strings.Split(sourceCode, "\n")

    for _, pattern := range a.patterns {
        matches := pattern.Pattern.FindAllStringIndex(sourceCode, -1)

        for _, match := range matches {
            // 找到匹配位置的行号
            lineNum := strings.Count(sourceCode[:match[0]], "\n") + 1
            lineContent := ""
            if lineNum <= len(lines) {
                lineContent = strings.TrimSpace(lines[lineNum-1])
            }

            findings = append(findings, Finding{
                Title:       pattern.Name,
                Type:        pattern.Type,
                Severity:    pattern.Severity,
                Line:        lineNum,
                Code:        lineContent,
                Description: pattern.Description,
                Remediation: pattern.Remediation,
            })
        }
    }

    return findings
}

// Finding 发现
type Finding struct {
    Title       string
    Type        VulnerabilityType
    Severity    Severity
    Line        int
    Code        string
    Description string
    Remediation string
}

// AnalyzeContract 分析合约（带上下文）
func (a *StaticAnalyzer) AnalyzeContract(sourceCode string) *AuditReport {
    findings := a.Analyze(sourceCode)

    // 统计
    stats := make(map[Severity]int)
    for _, f := range findings {
        stats[f.Severity]++
    }

    // 计算风险评分
    riskScore := a.calculateRiskScore(findings)

    return &AuditReport{
        Findings:     findings,
        Stats:        stats,
        RiskScore:    riskScore,
        TotalIssues:  len(findings),
        CriticalCount: stats[SeverityCritical],
        HighCount:    stats[SeverityHigh],
        MediumCount:  stats[SeverityMedium],
        LowCount:     stats[SeverityLow],
    }
}

// AuditReport 审计报告
type AuditReport struct {
    Findings      []Finding
    Stats         map[Severity]int
    RiskScore     float64
    TotalIssues   int
    CriticalCount int
    HighCount     int
    MediumCount   int
    LowCount      int
}

// calculateRiskScore 计算风险评分
func (a *StaticAnalyzer) calculateRiskScore(findings []Finding) float64 {
    if len(findings) == 0 {
        return 0
    }

    var score float64
    for _, f := range findings {
        switch f.Severity {
        case SeverityCritical:
            score += 10
        case SeverityHigh:
            score += 5
        case SeverityMedium:
            score += 2
        case SeverityLow:
            score += 1
        }
    }

    // 归一化到 0-100
    maxScore := float64(len(findings)) * 10
    return (score / maxScore) * 100
}

// GenerateReport 生成报告
func (r *AuditReport) GenerateReport() string {
    var sb strings.Builder

    sb.WriteString("=== 安全审计报告 ===\n\n")
    sb.WriteString(fmt.Sprintf("风险评分: %.1f/100\n", r.RiskScore))
    sb.WriteString(fmt.Sprintf("总问题数: %d\n", r.TotalIssues))
    sb.WriteString(fmt.Sprintf("  - 严重: %d\n", r.CriticalCount))
    sb.WriteString(fmt.Sprintf("  - 高危: %d\n", r.HighCount))
    sb.WriteString(fmt.Sprintf("  - 中危: %d\n", r.MediumCount))
    sb.WriteString(fmt.Sprintf("  - 低危: %d\n", r.LowCount))
    sb.WriteString("\n--- 详细发现 ---\n\n")

    for i, f := range r.Findings {
        severityStr := ""
        switch f.Severity {
        case SeverityCritical:
            severityStr = "[严重]"
        case SeverityHigh:
            severityStr = "[高危]"
        case SeverityMedium:
            severityStr = "[中危]"
        case SeverityLow:
            severityStr = "[低危]"
        }

        sb.WriteString(fmt.Sprintf("%d. %s %s\n", i+1, severityStr, f.Title))
        sb.WriteString(fmt.Sprintf("   行号: %d\n", f.Line))
        sb.WriteString(fmt.Sprintf("   代码: %s\n", f.Code))
        sb.WriteString(fmt.Sprintf("   描述: %s\n", f.Description))
        sb.WriteString(fmt.Sprintf("   建议: %s\n\n", f.Remediation))
    }

    return sb.String()
}
```

---

## 本文总结

本文深入探讨了 DeFi 安全与漏洞防护，主要涵盖：

### 漏洞类型总结

| 漏洞类型 | 严重程度 | 检测方法 | 防护措施 |
|---------|---------|---------|---------|
| 重入攻击 | 严重 | 调用栈分析 | ReentrancyGuard |
| 整数溢出 | 高 | 算术检查 | SafeMath / Solidity 0.8+ |
| 价格操纵 | 严重 | 价格波动监控 | TWAP / 多源预言机 |
| 闪电贷攻击 | 严重 | 交易模拟 | 延迟敏感操作 |
| 访问控制 | 高 | 静态分析 | 正确的权限检查 |
| 签名重放 | 中 | nonce 检查 | EIP-712 |

### 安全检测流程

```
安全审计流程：
┌─────────────────────────────────────────────────────────────┐
│                      安全检测流水线                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  静态分析   │───▶│  动态分析   │───▶│  形式验证   │     │
│  │  Source     │    │  Runtime    │    │  Formal     │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│         │                  │                  │             │
│         ▼                  ▼                  ▼             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  漏洞模式   │    │  执行追踪   │    │  不变量    │     │
│  │  匹配       │    │  分析       │    │  验证       │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                             │
│  ────────────────────────────────────────────────────────  │
│                          │                                  │
│                          ▼                                  │
│                  ┌─────────────┐                           │
│                  │  审计报告   │                           │
│                  │  生成       │                           │
│                  └─────────────┘                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 最佳实践清单

1. **开发阶段**
   - 使用 Solidity 0.8+ 内置检查
   - 遵循 checks-effects-interactions 模式
   - 实施访问控制
   - 避免使用 tx.origin

2. **测试阶段**
   - 单元测试覆盖边界情况
   - 模糊测试
   - 符号执行
   - 主网分叉测试

3. **部署阶段**
   - 多签控制关键操作
   - 时间锁延迟敏感操作
   - 暂停机制
   - 升级能力

4. **运维阶段**
   - 实时监控异常交易
   - 设置价格偏离警报
   - 准备应急响应计划
   - 定期安全审计

### 系列总结

本系列文档共 14 篇，从 EVM 基础到完整套利机器人实现，涵盖了：

1. EVM 基础与操作码
2. 存储与内存模型
3. Gas 机制
4. 交易类型与签名
5. 智能合约交互
6. DEX 原理
7. MEV 机制
8. Flashbots 集成
9. 闪电贷
10. DEX 数学模型
11. 预言机机制
12. MEV-Share
13. 完整实战项目
14. 安全与漏洞

希望这些内容能帮助您深入理解以太坊 EVM 和 DeFi 生态系统！