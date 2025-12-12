# Go-Ethereum EVM 详解（四）：MEV 深度剖析

## 目录

- [27. MEV 基础概念](#27-mev-基础概念)
- [28. MEV 的类型与来源](#28-mev-的类型与来源)
- [29. Flashbots 架构](#29-flashbots-架构)
- [30. Bundle 机制详解](#30-bundle-机制详解)
- [31. Searcher 策略开发](#31-searcher-策略开发)
- [32. MEV-Boost 与 PBS](#32-mev-boost-与-pbs)
- [33. 实战：构建套利机器人](#33-实战构建套利机器人)

---

## 27. MEV 基础概念

### 27.1 什么是 MEV？

MEV（Maximal Extractable Value，最大可提取价值）是指区块生产者（矿工或验证者）通过在区块中**包含、排除或重新排序交易**
所能获取的最大利润。

```
┌─────────────────────────────────────────────────────────────────┐
│                        MEV 概念演进                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  2019年之前        2019-2022          2022年之后（合并后）        │
│  ┌─────────┐      ┌─────────┐        ┌─────────┐               │
│  │  MEV    │      │  MEV    │        │  MEV    │               │
│  │ (Miner  │ ──▶  │ (Miner  │  ──▶   │(Maximal │               │
│  │Extracta-│      │Extracta-│        │Extracta-│               │
│  │ble Value│      │ble Value│        │ble Value│               │
│  │   )     │      │   )     │        │   )     │               │
│  └─────────┘      └─────────┘        └─────────┘               │
│       │                │                   │                    │
│       ▼                ▼                   ▼                    │
│  矿工手动排序     Flashbots出现      验证者+Builder分离          │
│  零散MEV提取      系统化MEV提取      MEV民主化                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 27.2 MEV 产生的根本原因

MEV 的存在源于以太坊的几个核心特性：

```
┌────────────────────────────────────────────────────────────────┐
│                    MEV 产生的根本原因                           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. 交易可见性（Mempool透明）                                   │
│     ┌─────────────────────────────────────────────────────┐   │
│     │ 用户广播交易 ──▶ Mempool（公开） ──▶ 所有节点可见    │   │
│     │                        │                            │   │
│     │                        ▼                            │   │
│     │               攻击者可以看到待处理交易               │   │
│     └─────────────────────────────────────────────────────┘   │
│                                                                │
│  2. 交易排序权（区块生产者决定）                                │
│     ┌─────────────────────────────────────────────────────┐   │
│     │ 区块生产者可以：                                     │   │
│     │   • 决定交易包含顺序                                 │   │
│     │   • 插入自己的交易                                   │   │
│     │   • 排除某些交易                                     │   │
│     │   • 延迟某些交易                                     │   │
│     └─────────────────────────────────────────────────────┘   │
│                                                                │
│  3. 状态依赖性（交易结果取决于执行顺序）                        │
│     ┌─────────────────────────────────────────────────────┐   │
│     │ 交易A: 在DEX买入Token (价格: 100)                    │   │
│     │ 交易B: 在DEX买入Token (价格: ?)                      │   │
│     │                                                      │   │
│     │ 如果 A 先执行: B的价格 = 105（滑点）                 │   │
│     │ 如果 B 先执行: A的价格 = 105（滑点）                 │   │
│     └─────────────────────────────────────────────────────┘   │
│                                                                │
│  4. 原子性执行（同一区块内交易的原子性）                        │
│     ┌─────────────────────────────────────────────────────┐   │
│     │ 在同一区块内可以：                                   │   │
│     │   • 借款 ──▶ 套利 ──▶ 还款（闪电贷）                │   │
│     │   • 无本金风险套利                                   │   │
│     └─────────────────────────────────────────────────────┘   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 27.3 MEV 生态系统参与者

```
┌─────────────────────────────────────────────────────────────────┐
│                     MEV 生态系统                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                                               │
│  │   Searcher   │  搜索者：寻找MEV机会，构建交易                 │
│  │  （搜索者）   │  • 套利机器人                                 │
│  └──────┬───────┘  • 清算机器人                                 │
│         │          • 三明治攻击者                                │
│         │ 提交Bundle                                            │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │   Builder    │  区块构建者：组装最优区块                      │
│  │  （构建者）   │  • 从多个Searcher收集Bundle                   │
│  └──────┬───────┘  • 优化区块价值                               │
│         │                                                       │
│         │ 提交区块                                              │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │   Proposer   │  提议者：选择并提议区块                        │
│  │  （验证者）   │  • 运行MEV-Boost                              │
│  └──────┬───────┘  • 选择出价最高的区块                         │
│         │                                                       │
│         │ 区块上链                                              │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │  Blockchain  │  区块链：最终状态                              │
│  │   （链上）    │                                               │
│  └──────────────┘                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 27.4 MEV 的历史数据

```go
// MEV 规模统计（截至2024年）
type MEVStatistics struct {
// 累计提取的MEV总量
TotalExtractedMEV    *big.Int // > $600M+ (从2020年开始统计)

// 日均MEV
DailyAverageMEV      *big.Int // $1-5M（波动较大）

// MEV类型分布
ArbitragePct         float64 // ~60% 套利
LiquidationPct       float64 // ~25% 清算
SandwichPct          float64   // ~15% 三明治攻击
}

// 主要MEV来源协议
var TopMEVProtocols = []string{
"Uniswap",  // DEX套利的主要来源
"Curve",    // 稳定币套利
"Aave",     // 清算
"Compound", // 清算
"Balancer",     // 多池套利
"SushiSwap", // 跨DEX套利
}
```

---

## 28. MEV 的类型与来源

### 28.1 套利（Arbitrage）

套利是最常见的MEV类型，利用不同市场间的价格差异获利。

```
┌─────────────────────────────────────────────────────────────────┐
│                      DEX 套利示例                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   场景：ETH/USDC 在不同DEX的价格不同                            │
│                                                                 │
│   ┌─────────────┐              ┌─────────────┐                  │
│   │  Uniswap V3 │              │  SushiSwap  │                  │
│   │             │              │             │                  │
│   │ 1 ETH =     │              │ 1 ETH =     │                  │
│   │ 2000 USDC   │              │ 2010 USDC   │                  │
│   └─────────────┘              └─────────────┘                  │
│          │                            ▲                         │
│          │  1. 用 2000 USDC          │                         │
│          │     买入 1 ETH            │  2. 卖出 1 ETH          │
│          └───────────────────────────┘     得到 2010 USDC      │
│                                                                 │
│   利润 = 2010 - 2000 = 10 USDC（减去Gas费用）                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 28.1.1 两池套利（Two-Pool Arbitrage）

```go
// 两池套利：最简单的套利形式
package arbitrage

import (
	"math/big"
)

// TwoPoolArbitrage 两池套利计算
type TwoPoolArbitrage struct {
	Pool1 *UniswapV2Pool
	Pool2 *UniswapV2Pool
}

// UniswapV2Pool Uniswap V2 池子
type UniswapV2Pool struct {
	Address  common.Address
	Token0   common.Address
	Token1   common.Address
	Reserve0 *big.Int
	Reserve1 *big.Int
	Fee      *big.Int // 通常是 997/1000 (0.3% fee)
}

// CalculateOptimalInput 计算最优输入量
// 使用数学推导：对于两个池子，存在一个最优输入量使利润最大化
func (t *TwoPoolArbitrage) CalculateOptimalInput() (*big.Int, *big.Int) {
	// Pool1: 用 Token0 换 Token1
	// Pool2: 用 Token1 换 Token0

	// 设输入量为 x，计算输出
	// Pool1 输出: y1 = (reserve1_1 * x * 997) / (reserve0_1 * 1000 + x * 997)
	// Pool2 输出: y2 = (reserve0_2 * y1 * 997) / (reserve1_2 * 1000 + y1 * 997)

	// 利润 = y2 - x
	// 对 x 求导并令其为 0，得到最优输入量

	r0_1 := t.Pool1.Reserve0
	r1_1 := t.Pool1.Reserve1
	r0_2 := t.Pool2.Reserve0
	r1_2 := t.Pool2.Reserve1

	// 简化计算（忽略手续费的精确公式）
	// 最优输入量的近似公式
	// optimalInput = sqrt(r0_1 * r1_1 * r0_2 * r1_2 * fee^2) - r0_1

	// 实际实现需要更精确的数学计算
	// 这里展示简化版本

	fee := big.NewInt(997)
	feeDenom := big.NewInt(1000)

	// 计算价格比率
	// price1 = r1_1 / r0_1 (Pool1 中 Token1 相对于 Token0 的价格)
	// price2 = r0_2 / r1_2 (Pool2 中 Token0 相对于 Token1 的价格)

	// 如果 price1 * price2 > 1，存在套利机会

	// 计算是否有套利机会
	// (r1_1 / r0_1) * (r0_2 / r1_2) > 1
	// 即 r1_1 * r0_2 > r0_1 * r1_2

	left := new(big.Int).Mul(r1_1, r0_2)
	right := new(big.Int).Mul(r0_1, r1_2)

	if left.Cmp(right) <= 0 {
		return nil, nil // 无套利机会
	}

	// 计算最优输入量（简化公式）
	// 实际项目中需要使用更精确的二分搜索或牛顿法
	optimalInput := t.binarySearchOptimalInput(fee, feeDenom)
	profit := t.calculateProfit(optimalInput)

	return optimalInput, profit
}

// binarySearchOptimalInput 二分搜索最优输入量
func (t *TwoPoolArbitrage) binarySearchOptimalInput(fee, feeDenom *big.Int) *big.Int {
	low := big.NewInt(1)
	high := new(big.Int).Set(t.Pool1.Reserve0) // 最大不超过池子储备

	bestInput := big.NewInt(0)
	bestProfit := big.NewInt(0)

	for low.Cmp(high) < 0 {
		mid := new(big.Int).Add(low, high)
		mid.Div(mid, big.NewInt(2))

		profit := t.calculateProfit(mid)

		// 检查是否是更好的输入量
		if profit.Cmp(bestProfit) > 0 {
			bestInput.Set(mid)
			bestProfit.Set(profit)
		}

		// 计算 mid+1 的利润来判断方向
		midPlusOne := new(big.Int).Add(mid, big.NewInt(1))
		profitPlusOne := t.calculateProfit(midPlusOne)

		if profitPlusOne.Cmp(profit) > 0 {
			low = midPlusOne
		} else {
			high = mid
		}
	}

	return bestInput
}

// calculateProfit 计算给定输入量的利润
func (t *TwoPoolArbitrage) calculateProfit(input *big.Int) *big.Int {
	// 第一步：在 Pool1 中用 input 数量的 Token0 换 Token1
	output1 := t.getAmountOut(input, t.Pool1.Reserve0, t.Pool1.Reserve1)

	// 第二步：在 Pool2 中用 output1 数量的 Token1 换 Token0
	output2 := t.getAmountOut(output1, t.Pool2.Reserve1, t.Pool2.Reserve0)

	// 利润 = output2 - input
	profit := new(big.Int).Sub(output2, input)

	return profit
}

// getAmountOut Uniswap V2 标准公式
func (t *TwoPoolArbitrage) getAmountOut(amountIn, reserveIn, reserveOut *big.Int) *big.Int {
	// amountOut = (amountIn * 997 * reserveOut) / (reserveIn * 1000 + amountIn * 997)

	amountInWithFee := new(big.Int).Mul(amountIn, big.NewInt(997))
	numerator := new(big.Int).Mul(amountInWithFee, reserveOut)
	denominator := new(big.Int).Mul(reserveIn, big.NewInt(1000))
	denominator.Add(denominator, amountInWithFee)

	return new(big.Int).Div(numerator, denominator)
}
```

#### 28.1.2 三角套利（Triangular Arbitrage）

```
┌─────────────────────────────────────────────────────────────────┐
│                     三角套利示例                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                      ETH                                        │
│                     /    \                                      │
│                    /      \                                     │
│              Pool1/        \Pool3                               │
│            (ETH/USDC)    (ETH/DAI)                              │
│                  /          \                                   │
│                 /            \                                  │
│              USDC ─────────── DAI                               │
│                    Pool2                                        │
│                  (USDC/DAI)                                     │
│                                                                 │
│   套利路径：                                                    │
│   1. USDC ──Pool1──▶ ETH                                       │
│   2. ETH  ──Pool3──▶ DAI                                       │
│   3. DAI  ──Pool2──▶ USDC                                      │
│                                                                 │
│   如果最终 USDC > 初始 USDC，则存在套利机会                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```go
// TriangularArbitrage 三角套利
type TriangularArbitrage struct {
Pools  []*Pool
Path   []common.Address // Token 路径
}

// Pool 通用池子接口
type Pool struct {
Address   common.Address
TokenA    common.Address
TokenB    common.Address
ReserveA  *big.Int
ReserveB  *big.Int
PoolType  string // "uniswap_v2", "uniswap_v3", "curve", etc.
}

// FindTriangularOpportunities 寻找三角套利机会
func FindTriangularOpportunities(pools []*Pool, baseToken common.Address) []*TriangularArbitrage {
opportunities := make([]*TriangularArbitrage, 0)

// 构建图：Token -> []Pool
graph := buildPoolGraph(pools)

// DFS 寻找所有长度为3的环
// 从 baseToken 出发，经过3个池子回到 baseToken
visited := make(map[common.Address]bool)
path := []common.Address{baseToken}
poolPath := make([]*Pool, 0)

var dfs func (current common.Address, depth int)
dfs = func (current common.Address, depth int) {
if depth == 3 {
// 检查是否能回到 baseToken
for _, pool := range graph[current] {
if canSwap(pool, current, baseToken) {
// 找到一个三角路径
arb := &TriangularArbitrage{
Pools: append(poolPath, pool),
Path:  append(path, baseToken),
}
if profit := arb.CalculateProfit(big.NewInt(1e18)); profit.Sign() > 0 {
opportunities = append(opportunities, arb)
}
}
}
return
}

for _, pool := range graph[current] {
nextToken := getOtherToken(pool, current)
if !visited[nextToken] && nextToken != baseToken {
visited[nextToken] = true
path = append(path, nextToken)
poolPath = append(poolPath, pool)

dfs(nextToken, depth+1)

path = path[:len(path)-1]
poolPath = poolPath[:len(poolPath)-1]
visited[nextToken] = false
}
}
}

dfs(baseToken, 0)
return opportunities
}

// CalculateProfit 计算三角套利利润
func (t *TriangularArbitrage) CalculateProfit(inputAmount *big.Int) *big.Int {
amount := new(big.Int).Set(inputAmount)

for i, pool := range t.Pools {
tokenIn := t.Path[i]
tokenOut := t.Path[i+1]

amount = getAmountOutForPool(pool, amount, tokenIn, tokenOut)
}

// 利润 = 最终金额 - 初始金额
return new(big.Int).Sub(amount, inputAmount)
}
```

#### 28.1.3 跨协议套利（Cross-Protocol Arbitrage）

```go
// CrossProtocolArbitrage 跨协议套利
// 例如：Uniswap V2 vs Uniswap V3 vs Curve vs Balancer
type CrossProtocolArbitrage struct {
SourceProtocol string // "uniswap_v2"
TargetProtocol string // "curve"
TokenIn        common.Address
TokenOut       common.Address
}

// Uniswap V3 特殊处理：需要考虑 tick 和流动性分布
type UniswapV3Pool struct {
Address       common.Address
Token0        common.Address
Token1        common.Address
Fee           uint32 // 500, 3000, 10000
TickSpacing   int32
SqrtPriceX96  *big.Int // 当前价格
Liquidity     *big.Int // 当前流动性
Tick          int32    // 当前 tick
}

// GetAmountOutV3 Uniswap V3 输出计算（简化版）
func (p *UniswapV3Pool) GetAmountOutV3(amountIn *big.Int, zeroForOne bool) *big.Int {
// Uniswap V3 使用集中流动性，计算更复杂
// 需要遍历 tick 范围内的流动性

// 简化计算：假设在当前 tick 范围内有足够流动性
// 实际需要模拟 tick 跨越

// sqrtPriceX96 表示 sqrt(price) * 2^96
// 对于 token0 -> token1 (zeroForOne = true)
// amountOut = amountIn * price * (1 - fee)

fee := big.NewInt(int64(p.Fee))
feeDenom := big.NewInt(1000000)

// 计算有效输入（扣除手续费）
effectiveInput := new(big.Int).Mul(amountIn, new(big.Int).Sub(feeDenom, fee))
effectiveInput.Div(effectiveInput, feeDenom)

// 价格计算（简化）
// price = (sqrtPriceX96 / 2^96)^2
// 实际实现需要更精确的定点数运算

var amountOut *big.Int
if zeroForOne {
// token0 -> token1
amountOut = calculateV3Output(effectiveInput, p.SqrtPriceX96, p.Liquidity, true)
} else {
// token1 -> token0
amountOut = calculateV3Output(effectiveInput, p.SqrtPriceX96, p.Liquidity, false)
}

return amountOut
}

// Curve 池子特殊处理：StableSwap 算法
type CurvePool struct {
Address    common.Address
Tokens     []common.Address
Balances   []*big.Int
A          *big.Int // 放大系数
Fee        *big.Int // 手续费
}

// GetAmountOutCurve Curve StableSwap 输出计算
func (p *CurvePool) GetAmountOutCurve(amountIn *big.Int, i, j int) *big.Int {
// Curve 使用 StableSwap 不变量
// A * sum(x_i) * n^n + D = A * D * n^n + D^(n+1) / (n^n * prod(x_i))

// 简化计算（实际需要牛顿迭代求解）
n := len(p.Tokens)

// 计算新的 D 值
D := p.calculateD()

// 计算交换后的余额
newBalanceI := new(big.Int).Add(p.Balances[i], amountIn)

// 求解 y（新的 balance[j]）
y := p.getY(i, j, newBalanceI, D)

// 输出 = 原余额 - 新余额 - 手续费
amountOut := new(big.Int).Sub(p.Balances[j], y)
fee := new(big.Int).Mul(amountOut, p.Fee)
fee.Div(fee, big.NewInt(10000000000)) // fee precision
amountOut.Sub(amountOut, fee)

return amountOut
}

// calculateD 计算 StableSwap 的 D 值
func (p *CurvePool) calculateD() *big.Int {
// 牛顿迭代法求解 D
// D^(n+1) / (n^n * prod(x_i)) + D * (A*n^n - 1) = A * n^n * sum(x_i)

n := big.NewInt(int64(len(p.Tokens)))

// S = sum(balances)
S := big.NewInt(0)
for _, b := range p.Balances {
S.Add(S, b)
}

if S.Sign() == 0 {
return big.NewInt(0)
}

D := new(big.Int).Set(S)
Ann := new(big.Int).Mul(p.A, n)
for i := 0; i < len(p.Tokens); i++ {
Ann.Mul(Ann, n)
}

// 迭代求解
for i := 0; i < 255; i++ {
D_P := new(big.Int).Set(D)
for _, b := range p.Balances {
D_P.Mul(D_P, D)
D_P.Div(D_P, new(big.Int).Mul(b, n))
}

prevD := new(big.Int).Set(D)

// D = (Ann * S + D_P * n) * D / ((Ann - 1) * D + (n + 1) * D_P)
numerator := new(big.Int).Mul(Ann, S)
numerator.Add(numerator, new(big.Int).Mul(D_P, n))
numerator.Mul(numerator, D)

denominator := new(big.Int).Sub(Ann, big.NewInt(1))
denominator.Mul(denominator, D)
denominator.Add(denominator, new(big.Int).Mul(new(big.Int).Add(n, big.NewInt(1)), D_P))

D.Div(numerator, denominator)

// 检查收敛
diff := new(big.Int).Sub(D, prevD)
if diff.CmpAbs(big.NewInt(1)) <= 0 {
break
}
}

return D
}
```

### 28.2 清算（Liquidation）

清算是借贷协议中的MEV机会，当抵押品价值下降导致健康因子不足时触发。

```
┌─────────────────────────────────────────────────────────────────┐
│                      清算机制示例                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   借款人状态：                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  抵押品：10 ETH (价值 $20,000，按 $2000/ETH)            │  │
│   │  借款：12,000 USDC                                      │  │
│   │  清算阈值：80%                                          │  │
│   │  当前 LTV = 12000/20000 = 60% ✓                        │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   价格下跌后：                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  ETH 价格跌至 $1400                                     │  │
│   │  抵押品价值：10 * 1400 = $14,000                        │  │
│   │  当前 LTV = 12000/14000 = 85.7% ✗ 超过清算阈值！        │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   清算者操作：                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  1. 还款 6000 USDC（最多清算50%债务）                   │  │
│   │  2. 获得抵押品：6000/1400 * 1.05 = 4.5 ETH             │  │
│   │     (1.05 是 5% 的清算奖励)                             │  │
│   │  3. 利润：4.5 * 1400 - 6000 = $300                     │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```go
// LiquidationBot 清算机器人
type LiquidationBot struct {
aaveV3    *AaveV3Client
compound  *CompoundClient
flashloan *FlashloanProvider
}

// Position 借贷仓位
type Position struct {
User            common.Address
Protocol        string // "aave", "compound"
Collaterals     []CollateralInfo
Debts           []DebtInfo
HealthFactor    *big.Int // 健康因子 (1e18 = 1.0)
TotalCollateral *big.Int // 总抵押品价值 (USD)
TotalDebt       *big.Int // 总债务价值 (USD)
}

type CollateralInfo struct {
Token          common.Address
Amount         *big.Int
ValueUSD       *big.Int
LiqThreshold   *big.Int // 清算阈值
}

type DebtInfo struct {
Token     common.Address
Amount    *big.Int
ValueUSD  *big.Int
}

// ScanLiquidatablePositions 扫描可清算仓位
func (b *LiquidationBot) ScanLiquidatablePositions(ctx context.Context) ([]*Position, error) {
liquidatable := make([]*Position, 0)

// 监听 Aave 的借款事件，追踪所有活跃仓位
// 这里简化为从数据库获取
positions, err := b.getAllActivePositions(ctx)
if err != nil {
return nil, err
}

for _, pos := range positions {
// 更新价格和健康因子
err := b.updatePositionHealth(ctx, pos)
if err != nil {
continue
}

// 健康因子 < 1 表示可清算
if pos.HealthFactor.Cmp(big.NewInt(1e18)) < 0 {
liquidatable = append(liquidatable, pos)
}
}

// 按利润排序
sort.Slice(liquidatable, func (i, j int) bool {
profitI := b.estimateLiquidationProfit(liquidatable[i])
profitJ := b.estimateLiquidationProfit(liquidatable[j])
return profitI.Cmp(profitJ) > 0
})

return liquidatable, nil
}

// ExecuteLiquidation 执行清算（使用闪电贷）
func (b *LiquidationBot) ExecuteLiquidation(ctx context.Context, pos *Position) (*types.Transaction, error) {
// 选择要清算的债务和抵押品
debtToLiquidate := b.selectDebtToLiquidate(pos)
collateralToReceive := b.selectCollateralToReceive(pos)

// 计算清算金额（通常最多清算50%）
maxLiquidation := new(big.Int).Div(debtToLiquidate.Amount, big.NewInt(2))

// 构建闪电贷 + 清算交易
// 1. 从 Aave 借出债务代币
// 2. 调用清算函数
// 3. 将获得的抵押品换成债务代币
// 4. 还款 + 利润

flashloanData := b.buildFlashloanLiquidation(
pos.User,
debtToLiquidate.Token,
maxLiquidation,
collateralToReceive.Token,
)

return b.flashloan.ExecuteFlashloan(ctx, flashloanData)
}

// Aave V3 清算函数签名
// function liquidationCall(
//     address collateralAsset,
//     address debtAsset,
//     address user,
//     uint256 debtToCover,
//     bool receiveAToken
// ) external;
```

### 28.3 三明治攻击（Sandwich Attack）

三明治攻击是一种有争议的MEV提取方式，通过在受害者交易前后插入交易获利。

```
┌─────────────────────────────────────────────────────────────────┐
│                    三明治攻击示意图                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   正常情况（无攻击）：                                          │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  用户交易：用 1000 USDC 买 ETH                          │  │
│   │  预期价格：$2000/ETH                                    │  │
│   │  滑点设置：1%                                           │  │
│   │  预期获得：≈0.5 ETH（最少 0.495 ETH）                   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   三明治攻击：                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  交易1（Front-run）：攻击者先买入 ETH                   │  │
│   │    • 买入大量 ETH，推高价格到 $2015                     │  │
│   │                                                         │  │
│   │  交易2（受害者）：用户的原始交易                        │  │
│   │    • 用户以更高价格买入，只得到 0.496 ETH               │  │
│   │    • 进一步推高价格到 $2020                             │  │
│   │                                                         │  │
│   │  交易3（Back-run）：攻击者卖出 ETH                      │  │
│   │    • 以 $2020 卖出，赚取差价                            │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
│   攻击者利润 ≈ (2020 - 2015) * 买入数量 - Gas费               │
│   受害者损失 ≈ 用户设置的滑点范围内的价值                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```go
// SandwichDetector 三明治攻击检测器
type SandwichDetector struct {
client *ethclient.Client
}

// PendingTx 待处理交易
type PendingTx struct {
Hash        common.Hash
From        common.Address
To          common.Address
Data        []byte
GasPrice    *big.Int
Value       *big.Int
}

// SandwichOpportunity 三明治攻击机会
type SandwichOpportunity struct {
VictimTx       *PendingTx
Pool           common.Address
TokenIn        common.Address
TokenOut       common.Address
AmountIn       *big.Int
MinAmountOut   *big.Int      // 受害者的最小输出（滑点保护）
FrontrunAmount *big.Int      // 前置交易数量
ExpectedProfit *big.Int
}

// DetectSandwichOpportunity 检测三明治攻击机会
func (d *SandwichDetector) DetectSandwichOpportunity(tx *PendingTx) *SandwichOpportunity {
// 解析交易数据，判断是否是 DEX 交易
decoded, err := d.decodeSwapTx(tx)
if err != nil {
return nil
}

// 检查滑点设置
// 滑点越大，攻击空间越大
slippage := d.calculateSlippage(decoded)
if slippage.Cmp(big.NewInt(50)) < 0 { // 0.5% 以下不值得攻击
return nil
}

// 计算最优前置交易数量
optimalFrontrun := d.calculateOptimalFrontrun(decoded)

// 估算利润
profit := d.estimateProfit(decoded, optimalFrontrun)

// 利润需要覆盖 Gas 成本（前置+后置两笔交易）
gasCost := d.estimateGasCost(2)
if profit.Cmp(gasCost) <= 0 {
return nil
}

return &SandwichOpportunity{
VictimTx:       tx,
Pool:           decoded.Pool,
TokenIn:        decoded.TokenIn,
TokenOut:       decoded.TokenOut,
AmountIn:       decoded.AmountIn,
MinAmountOut:   decoded.MinAmountOut,
FrontrunAmount: optimalFrontrun,
ExpectedProfit: new(big.Int).Sub(profit, gasCost),
}
}

// 注意：实现三明治攻击是不道德的行为
// 这里仅作为教育目的展示其原理
// 以下是检测和防护代码

// ProtectFromSandwich 保护用户交易免受三明治攻击
func ProtectFromSandwich(tx *types.Transaction) (*types.Transaction, error) {
// 方法1：使用私有交易池（Flashbots Protect）
// 方法2：设置合理的滑点（不要太大）
// 方法3：使用 DEX 聚合器的私有订单流
// 方法4：拆分大额交易

return tx, nil
}
```

### 28.4 JIT 流动性（Just-In-Time Liquidity）

JIT 流动性是一种在 Uniswap V3 上的 MEV 策略。

```
┌─────────────────────────────────────────────────────────────────┐
│                   JIT 流动性攻击                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Uniswap V3 特性：集中流动性                                    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │   价格范围          流动性分布                          │   │
│  │      │                                                  │   │
│  │  $2100│                    ┌──┐                        │   │
│  │  $2050│               ┌────┤  │                        │   │
│  │  $2000│          ┌────┤    │  │◄─── 当前价格           │   │
│  │  $1950│     ┌────┤    │    │  │                        │   │
│  │  $1900│─────┴────┴────┴────┴──┴───                     │   │
│  │       └──────────────────────────▶ 流动性              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  JIT 攻击流程：                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. 检测到大额 swap 交易（例如买入 ETH）                │   │
│  │  2. 在交易执行前，在精确价格范围添加大量流动性          │   │
│  │  3. 用户的交易执行，支付手续费给 LP（主要是攻击者）     │   │
│  │  4. 交易完成后立即移除流动性                            │   │
│  │  5. 获得用户支付的大部分手续费                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  与三明治区别：                                                 │
│  • 三明治：通过价格操纵损害用户                                │
│  • JIT：通过提供流动性赚取手续费（用户仍得到相同价格）         │
│  • JIT 可能被认为是"灰色地带"而非纯粹的攻击                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 28.5 NFT MEV

```go
// NFT 领域的 MEV 机会
type NFTMEVOpportunity struct {
Type        string // "mint_snipe", "listing_snipe", "trait_arbitrage"
Collection  common.Address
TokenID     *big.Int
Price       *big.Int
EstProfit   *big.Int
}

// 1. Mint 抢购：新NFT发售时的Gas竞争
// 2. Listing 抢购：低价挂单的快速购买
// 3. 特征套利：基于稀有度的套利
```

---

## 29. Flashbots 架构

### 29.1 Flashbots 概述

Flashbots 是一套旨在减轻 MEV 负面影响的基础设施。

```
┌─────────────────────────────────────────────────────────────────┐
│                    Flashbots 生态系统                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────────────────────────────────────────────────────┐ │
│   │                    MEV 的问题                             │ │
│   │  • 网络拥堵：Gas 竞价战（Priority Gas Auction）          │ │
│   │  • 区块链不稳定：叔块增加、重组风险                      │ │
│   │  • 用户损失：三明治攻击、抢跑                            │ │
│   │  • 中心化风险：专业化 MEV 提取导致验证者中心化           │ │
│   └──────────────────────────────────────────────────────────┘ │
│                           │                                     │
│                           ▼                                     │
│   ┌──────────────────────────────────────────────────────────┐ │
│   │                  Flashbots 解决方案                       │ │
│   │                                                          │ │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │ │
│   │  │  Flashbots  │  │  Flashbots  │  │  Flashbots  │      │ │
│   │  │   Auction   │  │   Protect   │  │   Data      │      │ │
│   │  │  (MEV拍卖)  │  │ (交易保护)  │  │  (数据透明) │      │ │
│   │  └─────────────┘  └─────────────┘  └─────────────┘      │ │
│   │                                                          │ │
│   │  合并后新增：                                            │ │
│   │  ┌─────────────┐  ┌─────────────┐                       │ │
│   │  │  MEV-Boost  │  │   SUAVE     │                       │ │
│   │  │ (PBS实现)   │  │ (去中心化)  │                       │ │
│   │  └─────────────┘  └─────────────┘                       │ │
│   └──────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 29.2 Flashbots Auction（合并前）

```
┌─────────────────────────────────────────────────────────────────┐
│              Flashbots Auction 架构（PoW时代）                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Searcher（搜索者）                                             │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  1. 监控 Mempool 和链上状态                               │  │
│   │  2. 发现 MEV 机会                                        │  │
│   │  3. 构建 Bundle（交易包）                                 │  │
│   │  4. 通过 eth_sendBundle 提交到 Flashbots Relay           │   │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           │ eth_sendBundle                      │
│                           ▼                                     │
│   Flashbots Relay（中继）                                       │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  1. 接收来自多个 Searcher 的 Bundle                        │  │
│   │  2. 模拟执行，验证有效性                                    │  │
│   │  3. 按出价排序                                            │  │
│   │  4. 转发给运行 mev-geth 的矿工                             │  │
│   └─────────────────────────────────────────────────────────┘  │
│                           │                                     │
│                           ▼                                     │
│   Miner（矿工，运行 mev-geth）                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  1. 接收 Bundle                                          │  │
│   │  2. 将 Bundle 合并到区块中                                 │  │
│   │  3. 如果 Bundle 使区块更有价值，就包含它                     │  │
│   │  4. 挖出区块，获得 Bundle 中的出价                          │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 29.3 Flashbots Bundle 格式

```go
// Bundle 结构
type FlashbotsBundle struct {
Txs               [][]byte // 签名后的原始交易
BlockNumber       uint64   // 目标区块号
MinTimestamp      uint64 // 最早包含时间（可选）
MaxTimestamp      uint64 // 最晚包含时间（可选）
RevertingTxHashes []common.Hash // 允许回滚的交易（可选）
}

// eth_sendBundle 请求参数
type SendBundleArgs struct {
Txs               []string `json:"txs"`         // 十六进制编码的签名交易
BlockNumber       string   `json:"blockNumber"` // 十六进制区块号
MinTimestamp      uint64   `json:"minTimestamp,omitempty"`
MaxTimestamp      uint64   `json:"maxTimestamp,omitempty"`
RevertingTxHashes []string `json:"revertingTxHashes,omitempty"`
}

// 发送 Bundle 到 Flashbots
func SendBundle(bundle *FlashbotsBundle, privateKey *ecdsa.PrivateKey) error {
// Flashbots RPC 端点
relayURL := "https://relay.flashbots.net"

// 准备请求
args := SendBundleArgs{
Txs:         make([]string, len(bundle.Txs)),
BlockNumber: fmt.Sprintf("0x%x", bundle.BlockNumber),
}
for i, tx := range bundle.Txs {
args.Txs[i] = hexutil.Encode(tx)
}

// 构建 JSON-RPC 请求
request := map[string]interface{}{
"jsonrpc": "2.0",
"id":      1,
"method":  "eth_sendBundle",
"params":  []interface{}{args},
}

body, _ := json.Marshal(request)

// Flashbots 需要签名认证
signature := signFlashbotsPayload(body, privateKey)

req, _ := http.NewRequest("POST", relayURL, bytes.NewReader(body))
req.Header.Set("Content-Type", "application/json")
req.Header.Set("X-Flashbots-Signature", signature)

resp, err := http.DefaultClient.Do(req)
if err != nil {
return err
}
defer resp.Body.Close()

// 处理响应
var result struct {
Result struct {
BundleHash string `json:"bundleHash"`
} `json:"result"`
Error *struct {
Message string `json:"message"`
} `json:"error"`
}
json.NewDecoder(resp.Body).Decode(&result)

if result.Error != nil {
return fmt.Errorf("flashbots error: %s", result.Error.Message)
}

fmt.Printf("Bundle submitted: %s\n", result.Result.BundleHash)
return nil
}

// 签名 Flashbots 请求
func signFlashbotsPayload(body []byte, privateKey *ecdsa.PrivateKey) string {
// Flashbots 使用 EIP-712 风格的签名
hashedBody := crypto.Keccak256Hash(body)
signature, _ := crypto.Sign(hashedBody.Bytes(), privateKey)

address := crypto.PubkeyToAddress(privateKey.PublicKey)
return fmt.Sprintf("%s:%s", address.Hex(), hexutil.Encode(signature))
}
```

### 29.4 Bundle 模拟

```go
// eth_callBundle - 模拟 Bundle 执行
type CallBundleArgs struct {
Txs              []string `json:"txs"`
BlockNumber      string   `json:"blockNumber"`
StateBlockNumber string   `json:"stateBlockNumber"`
Timestamp        uint64   `json:"timestamp,omitempty"`
}

type CallBundleResult struct {
BundleGasPrice    *big.Int            `json:"bundleGasPrice"`
BundleHash        common.Hash         `json:"bundleHash"`
CoinbaseDiff      *big.Int            `json:"coinbaseDiff"` // 给矿工的支付
EthSentToCoinbase *big.Int            `json:"ethSentToCoinbase"` // 直接转账
GasFees           *big.Int            `json:"gasFees"` // Gas 费用
Results           []TxSimulationResult `json:"results"`
StateBlockNumber  uint64              `json:"stateBlockNumber"`
TotalGasUsed      uint64              `json:"totalGasUsed"`
}

type TxSimulationResult struct {
CoinbaseDiff      *big.Int       `json:"coinbaseDiff"`
EthSentToCoinbase *big.Int       `json:"ethSentToCoinbase"`
FromAddress       common.Address `json:"fromAddress"`
GasFees           *big.Int       `json:"gasFees"`
GasPrice          *big.Int       `json:"gasPrice"`
GasUsed           uint64         `json:"gasUsed"`
ToAddress         common.Address `json:"toAddress"`
TxHash            common.Hash    `json:"txHash"`
Value             *big.Int       `json:"value"`
Error             string         `json:"error,omitempty"`
Revert            string         `json:"revert,omitempty"`
}

// SimulateBundle 模拟 Bundle 执行结果
func SimulateBundle(bundle *FlashbotsBundle) (*CallBundleResult, error) {
args := CallBundleArgs{
Txs:              make([]string, len(bundle.Txs)),
BlockNumber:      fmt.Sprintf("0x%x", bundle.BlockNumber),
StateBlockNumber: "latest",
}
for i, tx := range bundle.Txs {
args.Txs[i] = hexutil.Encode(tx)
}

// 调用 eth_callBundle
request := map[string]interface{}{
"jsonrpc": "2.0",
"id":      1,
"method":  "eth_callBundle",
"params":  []interface{}{args},
}

// ... 发送请求并解析结果
return nil, nil
}
```

---

## 30. Bundle 机制详解

### 30.1 Bundle 的核心概念

```
┌─────────────────────────────────────────────────────────────────┐
│                     Bundle 核心概念                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Bundle = 一组按特定顺序执行的交易                                  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    Bundle 特性                           │   │
│  │                                                          │   │
│  │  1. 原子性：Bundle 中的所有交易要么全部成功，                 │   │
│  │            要么全部不包含（除非指定 revertingTxHashes）      │   │
│  │                                                          │   │
│  │  2. 顺序性：交易按提交顺序执行，不会被打乱                     │   │
│  │                                                          │   │
│  │  3. 私密性：Bundle 不进入公开 mempool，                     │   │
│  │            其他人看不到                                    │   │
│  │                                                          │   │
│  │  4. 无 Gas 竞价：失败的 Bundle 不上链，不消耗 Gas            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Bundle 组成示例（套利）：                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Bundle [                                               │   │
│  │    Tx1: 在 Uniswap 买入 ETH (用 USDC)                     │   │
│  │    Tx2: 在 Sushiswap 卖出 ETH (获得 USDC)                 │   │
│  │    Tx3: 向 Coinbase 支付小费 (激励矿工包含)                 │   │
│  │  ]                                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 30.2 构建有效的 Bundle

```go
// BundleBuilder Bundle 构建器
type BundleBuilder struct {
client       *ethclient.Client
signer       types.Signer
privateKey   *ecdsa.PrivateKey
flashbotsKey *ecdsa.PrivateKey // Flashbots 认证用的密钥
}

// BuildArbitrageBundle 构建套利 Bundle
func (b *BundleBuilder) BuildArbitrageBundle(
ctx context.Context,
opportunity *ArbitrageOpportunity,
targetBlock uint64,
) (*FlashbotsBundle, error) {

transactions := make([][]byte, 0)
currentNonce, err := b.client.PendingNonceAt(ctx, b.getAddress())
if err != nil {
return nil, err
}

// 获取当前 Gas 价格作为参考
gasPrice, _ := b.client.SuggestGasPrice(ctx)

// 交易1：在低价池买入
buyTx, err := b.buildSwapTx(
opportunity.BuyPool,
opportunity.TokenIn,
opportunity.TokenOut,
opportunity.OptimalAmount,
currentNonce,
gasPrice,
)
if err != nil {
return nil, err
}
signedBuyTx, _ := types.SignTx(buyTx, b.signer, b.privateKey)
buyTxBytes, _ := signedBuyTx.MarshalBinary()
transactions = append(transactions, buyTxBytes)

// 交易2：在高价池卖出
sellTx, err := b.buildSwapTx(
opportunity.SellPool,
opportunity.TokenOut,
opportunity.TokenIn,
nil, // 卖出全部
currentNonce+1,
gasPrice,
)
if err != nil {
return nil, err
}
signedSellTx, _ := types.SignTx(sellTx, b.signer, b.privateKey)
sellTxBytes, _ := signedSellTx.MarshalBinary()
transactions = append(transactions, sellTxBytes)

// 交易3：支付矿工小费（可选但推荐）
tipAmount := b.calculateTip(opportunity.ExpectedProfit)
if tipAmount.Sign() > 0 {
tipTx := b.buildCoinbaseTipTx(tipAmount, currentNonce+2, gasPrice)
signedTipTx, _ := types.SignTx(tipTx, b.signer, b.privateKey)
tipTxBytes, _ := signedTipTx.MarshalBinary()
transactions = append(transactions, tipTxBytes)
}

return &FlashbotsBundle{
Txs:         transactions,
BlockNumber: targetBlock,
}, nil
}

// buildSwapTx 构建 swap 交易
func (b *BundleBuilder) buildSwapTx(
pool common.Address,
tokenIn, tokenOut common.Address,
amountIn *big.Int,
nonce uint64,
gasPrice *big.Int,
) (*types.Transaction, error) {

routerABI, _ := abi.JSON(strings.NewReader(UniswapV2RouterABI))
path := []common.Address{tokenIn, tokenOut}
deadline := big.NewInt(time.Now().Add(time.Minute * 5).Unix())
amountOutMin := big.NewInt(1)

data, err := routerABI.Pack(
"swapExactTokensForTokens",
amountIn,
amountOutMin,
path,
b.getAddress(),
deadline,
)
if err != nil {
return nil, err
}

return types.NewTransaction(
nonce,
common.HexToAddress(UniswapV2RouterAddress),
big.NewInt(0),
300000,
gasPrice,
data,
), nil
}

// calculateTip 计算小费金额（支付利润的90%给矿工）
func (b *BundleBuilder) calculateTip(expectedProfit *big.Int) *big.Int {
tip := new(big.Int).Mul(expectedProfit, big.NewInt(90))
tip.Div(tip, big.NewInt(100))
return tip
}
```

---

## 31. Searcher 策略开发

### 31.1 Searcher 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                  Searcher 系统架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    数据源层                               │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐     │   │
│  │  │Mempool  │  │ Block   │  │ Event   │  │External │     │   │
│  │  │Monitor  │  │ Monitor │  │ Monitor │  │  APIs   │     │   │
│  │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘     │   │
│  └───────┼────────────┼───────────┼────────────┼───────────┘   │
│          │            │           │            │               │
│          ▼            ▼           ▼            ▼               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    策略引擎层                              │  │
│  │  • Arbitrage Scanner（套利扫描）                           │  │
│  │  • Liquidation Scanner（清算扫描）                         │  │
│  │  • Sandwich Detector（三明治检测）                         │  │
│  │  • Simulation Engine（模拟引擎）                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            │                                   │
│                            ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    执行层                                 │  │
│  │  Bundle Builder ──▶ Flashbots Relay ──▶ Block Inclusion  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 31.2 完整 Searcher 实现

```go
// Searcher MEV 搜索者核心结构
type Searcher struct {
client        *ethclient.Client
pools         map[common.Address]*Pool
strategies    []Strategy
bundleBuilder *BundleBuilder
config        *SearcherConfig
}

type SearcherConfig struct {
MinProfitWei      *big.Int
MaxGasPrice       *big.Int
TargetBlocksAhead int
}

// Start 启动 Searcher
func (s *Searcher) Start(ctx context.Context) error {
// 启动区块监控
go s.monitorBlocks(ctx)
// 启动 mempool 监控
go s.monitorMempool(ctx)
// 启动策略引擎
go s.runStrategyEngine(ctx)
return nil
}

// monitorBlocks 监控新区块
func (s *Searcher) monitorBlocks(ctx context.Context) {
headers := make(chan *types.Header)
sub, _ := s.client.SubscribeNewHead(ctx, headers)
defer sub.Unsubscribe()

for {
select {
case <-ctx.Done():
return
case header := <-headers:
s.handleNewBlock(ctx, header)
}
}
}

// runStrategyEngine 运行策略引擎
func (s *Searcher) runStrategyEngine(ctx context.Context) {
ticker := time.NewTicker(100 * time.Millisecond)
defer ticker.Stop()

for {
select {
case <-ctx.Done():
return
case <-ticker.C:
s.scanForOpportunities(ctx)
}
}
}

// scanForOpportunities 扫描 MEV 机会
func (s *Searcher) scanForOpportunities(ctx context.Context) {
for _, strategy := range s.strategies {
opportunities, _ := strategy.Scan(ctx)
for _, opp := range opportunities {
if opp.ExpectedProfit.Cmp(s.config.MinProfitWei) > 0 {
s.executeOpportunity(ctx, opp)
}
}
}
}
```

### 31.3 套利策略实现

```go
// ArbitrageStrategy 套利策略
type ArbitrageStrategy struct {
pools     []*Pool
minProfit *big.Int
}

// Scan 扫描套利机会
func (a *ArbitrageStrategy) Scan(ctx context.Context) ([]*Opportunity, error) {
opportunities := make([]*Opportunity, 0)

// 更新所有池子储备量
for _, pool := range a.pools {
pool.UpdateReserves(ctx)
}

// 检查每对池子的套利机会
for i := 0; i < len(a.pools); i++ {
for j := i + 1; j < len(a.pools); j++ {
if opp := a.checkArbitrage(a.pools[i], a.pools[j]); opp != nil {
opportunities = append(opportunities, opp)
}
}
}
return opportunities, nil
}

// checkArbitrage 检查两池套利
func (a *ArbitrageStrategy) checkArbitrage(pool1, pool2 *Pool) *Opportunity {
// 使用二分搜索找最优输入量
profit, amount := a.calculateOptimalArbitrage(pool1, pool2)

if profit.Cmp(a.minProfit) <= 0 {
return nil
}

return &Opportunity{
Type:           "arbitrage",
ExpectedProfit: profit,
Data: &ArbitrageData{
BuyPool:       pool1,
SellPool:      pool2,
OptimalAmount: amount,
},
}
}

// Pool 池子结构和方法
type Pool struct {
Address  common.Address
Reserve0 *big.Int
Reserve1 *big.Int
Fee      uint64 // 基点
}

// GetAmountOut Uniswap V2 公式
func (p *Pool) GetAmountOut(amountIn *big.Int) *big.Int {
feeMultiplier := big.NewInt(10000 - int64(p.Fee))
amountInWithFee := new(big.Int).Mul(amountIn, feeMultiplier)
numerator := new(big.Int).Mul(amountInWithFee, p.Reserve1)
denominator := new(big.Int).Mul(p.Reserve0, big.NewInt(10000))
denominator.Add(denominator, amountInWithFee)
return new(big.Int).Div(numerator, denominator)
}
```

---

## 32. MEV-Boost 与 PBS

### 32.1 PBS 概念

```
┌─────────────────────────────────────────────────────────────────┐
│            PBS（Proposer-Builder Separation）                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  合并前：矿工 = 构建者 + 提议者                                     │
│                                                                 │
│  合并后（PBS）：                                                  │
│  ┌──────────────┐         ┌──────────────┐                      │
│  │   Builder    │ ──────▶ │   Proposer   │                      │
│  │  （构建者）   │  出价   │  （验证者）   │                        │
│  └──────────────┘         └──────────────┘                      │
│                                                                 │
│  优势：                                                          │
│  • 验证者无需专业 MEV 知识                                         │
│  • 降低中心化风险                                                 │
│  • 更公平的 MEV 分配                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 32.2 MEV-Boost 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    MEV-Boost 完整流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Searcher ──Bundle──▶ Builder ──Block+Bid──▶ Relay              │
│                                                 │               │
│                                                 ▼               │
│                                            MEV-Boost            │
│                                                 │               │
│                                                 ▼               │
│                                            Validator            │
│                                         (选择最高出价)            │
│                                                 │               │
│                                                 ▼               │
│                                           Block上链              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 32.3 MEV-Boost API

```go
// Builder API 主要端点

// 1. 注册验证者
// POST /eth/v1/builder/validators
type ValidatorRegistration struct {
FeeRecipient common.Address `json:"fee_recipient"`
GasLimit     uint64         `json:"gas_limit"`
Timestamp    uint64         `json:"timestamp"`
Pubkey       string         `json:"pubkey"`
}

// 2. 获取最高出价区块头
// GET /eth/v1/builder/header/{slot}/{parent_hash}/{pubkey}
type BidTrace struct {
Slot                 uint64         `json:"slot"`
ParentHash           common.Hash    `json:"parent_hash"`
BlockHash            common.Hash    `json:"block_hash"`
BuilderPubkey        string         `json:"builder_pubkey"`
ProposerFeeRecipient common.Address `json:"proposer_fee_recipient"`
Value                *big.Int       `json:"value"`
}

// GetBestHeader 获取最高出价
func (m *MEVBoostClient) GetBestHeader(slot uint64, parentHash common.Hash) (*BidTrace, error) {
var bestBid *BidTrace

// 并发查询所有 Relay，选择最高出价
for _, relay := range m.relays {
bid := m.queryRelay(relay, slot, parentHash)
if bestBid == nil || bid.Value.Cmp(bestBid.Value) > 0 {
bestBid = bid
}
}
return bestBid, nil
}
```

---

## 33. 实战：构建套利机器人

### 33.1 完整套利机器人架构

```go
package main

import (
	"context"
	"math/big"
	"time"

	"github.com/ethereum/go-ethereum/ethclient"
)

// ArbitrageBot 套利机器人
type ArbitrageBot struct {
	client    *ethclient.Client
	flashbots *FlashbotsClient
	pools     []*UniswapPool
	minProfit *big.Int
}

func main() {
	ctx := context.Background()

	// 初始化
	client, _ := ethclient.Dial("http://localhost:8545")
	bot := &ArbitrageBot{
		client:    client,
		minProfit: big.NewInt(1e16), // 0.01 ETH
	}

	// 加载池子
	bot.loadPools(ctx)

	// 主循环
	bot.run(ctx)
}

func (b *ArbitrageBot) run(ctx context.Context) {
	ticker := time.NewTicker(200 * time.Millisecond)

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			// 1. 更新池子状态
			b.updatePools(ctx)

			// 2. 扫描套利机会
			opps := b.scanArbitrage()

			// 3. 执行有利可图的机会
			for _, opp := range opps {
				if opp.Profit.Cmp(b.minProfit) > 0 {
					b.execute(ctx, opp)
				}
			}
		}
	}
}

func (b *ArbitrageBot) scanArbitrage() []*ArbitrageOpp {
	var opps []*ArbitrageOpp

	// 检查所有池子对
	for i, pool1 := range b.pools {
		for j, pool2 := range b.pools {
			if i >= j || pool1.Token0 != pool2.Token0 {
				continue
			}

			// 计算套利利润
			profit := b.calculateProfit(pool1, pool2)
			if profit.Sign() > 0 {
				opps = append(opps, &ArbitrageOpp{
					Pool1:  pool1,
					Pool2:  pool2,
					Profit: profit,
				})
			}
		}
	}
	return opps
}

func (b *ArbitrageBot) execute(ctx context.Context, opp *ArbitrageOpp) error {
	// 1. 构建交易
	txs := b.buildArbitrageTxs(opp)

	// 2. 模拟验证
	if !b.simulate(ctx, txs) {
		return errors.New("simulation failed")
	}

	// 3. 构建 Bundle
	bundle := &FlashbotsBundle{
		Txs:         txs,
		BlockNumber: b.getTargetBlock(),
	}

	// 4. 提交到 Flashbots
	return b.flashbots.SendBundle(ctx, bundle)
}
```

### 33.2 关键优化技巧

```go
// 1. 使用 Multicall 批量获取池子状态
func (b *ArbitrageBot) batchUpdatePools(ctx context.Context) {
multicall := NewMulticall(b.client)

calls := make([]Call, len(b.pools))
for i, pool := range b.pools {
calls[i] = Call{
Target: pool.Address,
Data:   getReservesCalldata,
}
}

results := multicall.Aggregate(ctx, calls)
// 解析结果更新池子状态...
}

// 2. 使用本地 EVM 模拟
func (b *ArbitrageBot) localSimulate(txs []*types.Transaction) bool {
// 使用 go-ethereum 的 vm 包进行本地模拟
// 比 eth_call 更快
return true
}

// 3. 延迟敏感优化
// - 使用 WebSocket 而非 HTTP
// - 直接连接 Builder（跳过公共 Relay）
// - 部署在低延迟节点附近
```

### 33.3 风险管理

```go
// RiskManager 风险管理
type RiskManager struct {
maxPositionSize  *big.Int
dailyLossLimit   *big.Int
currentDailyLoss *big.Int
}

func (r *RiskManager) CanExecute(opp *ArbitrageOpp) bool {
// 检查仓位大小
if opp.InputAmount.Cmp(r.maxPositionSize) > 0 {
return false
}

// 检查日亏损限制
if r.currentDailyLoss.Cmp(r.dailyLossLimit) > 0 {
return false
}

return true
}
```

---

## 总结

本文档详细介绍了 MEV 的核心概念和实战技术：

1. **MEV 基础**：理解 MEV 的来源、类型和生态系统参与者
2. **套利/清算/三明治**：三种主要 MEV 策略的原理和实现
3. **Flashbots**：Bundle 机制、提交流程和模拟验证
4. **MEV-Boost/PBS**：合并后的 MEV 基础设施
5. **Searcher 开发**：完整的套利机器人架构和优化技巧

下一篇将介绍交易池（TxPool）与交易排序机制。
