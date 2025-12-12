# Go-Ethereum EVM 深度解析：清算机器人

## 第一百零七节：借贷协议清算机制

### 清算原理概览

```
借贷协议清算机制：
┌─────────────────────────────────────────────────────────────────────┐
│                        借贷协议清算流程                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  用户借贷状态                                                        │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  抵押品: 10 ETH ($20,000)                                   │    │
│  │  借款:   15,000 USDC                                        │    │
│  │  健康因子: 抵押品价值 × LTV / 借款 = 1.33                    │    │
│  └────────────────────────────────────────────────────────────┘    │
│                               │                                     │
│                               ▼                                     │
│                    ETH 价格下跌至 $1,500                            │
│                               │                                     │
│                               ▼                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  抵押品: 10 ETH ($15,000)                                   │    │
│  │  借款:   15,000 USDC                                        │    │
│  │  健康因子: 15,000 × 0.8 / 15,000 = 0.8 < 1.0               │    │
│  │  状态: 可清算!                                               │    │
│  └────────────────────────────────────────────────────────────┘    │
│                               │                                     │
│                               ▼                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  清算人偿还: 7,500 USDC (50% 债务)                          │    │
│  │  获得抵押品: 5 ETH + 5% 清算奖励 = 5.25 ETH                 │    │
│  │  清算利润:   5.25 × $1,500 - 7,500 = $375                   │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 借贷协议接口

```go
package liquidation

import (
    "context"
    "fmt"
    "math/big"
    "sync"
    "time"

    "github.com/ethereum/go-ethereum"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/ethclient"
)

// Protocol 协议类型
type Protocol int

const (
    ProtocolAaveV2 Protocol = iota
    ProtocolAaveV3
    ProtocolCompoundV2
    ProtocolCompoundV3
)

// LendingProtocol 借贷协议接口
type LendingProtocol interface {
    // 获取用户账户数据
    GetAccountData(ctx context.Context, user common.Address) (*AccountData, error)
    // 获取可清算的账户
    GetLiquidatableAccounts(ctx context.Context) ([]*LiquidatableAccount, error)
    // 计算清算参数
    CalculateLiquidation(ctx context.Context, account *LiquidatableAccount) (*LiquidationParams, error)
    // 执行清算
    ExecuteLiquidation(ctx context.Context, params *LiquidationParams) (common.Hash, error)
    // 获取协议类型
    GetProtocol() Protocol
}

// AccountData 账户数据
type AccountData struct {
    User                    common.Address
    TotalCollateralETH      *big.Int
    TotalDebtETH            *big.Int
    AvailableBorrowsETH     *big.Int
    CurrentLiquidationThreshold *big.Int
    LTV                     *big.Int
    HealthFactor            *big.Int
}

// LiquidatableAccount 可清算账户
type LiquidatableAccount struct {
    User            common.Address
    Protocol        Protocol
    HealthFactor    *big.Int
    CollateralAssets []CollateralPosition
    DebtAssets      []DebtPosition
    TotalCollateral *big.Int
    TotalDebt       *big.Int
}

// CollateralPosition 抵押品头寸
type CollateralPosition struct {
    Asset       common.Address
    Amount      *big.Int
    ValueETH    *big.Int
    LiqThreshold *big.Int
}

// DebtPosition 债务头寸
type DebtPosition struct {
    Asset       common.Address
    Amount      *big.Int
    ValueETH    *big.Int
}

// LiquidationParams 清算参数
type LiquidationParams struct {
    User            common.Address
    CollateralAsset common.Address
    DebtAsset       common.Address
    DebtToCover     *big.Int
    CollateralToReceive *big.Int
    LiquidationBonus *big.Int
    EstimatedProfit *big.Int
    UseFlashLoan    bool
}

// AaveV3Protocol Aave V3 协议实现
type AaveV3Protocol struct {
    client          *ethclient.Client
    poolAddress     common.Address
    oracleAddress   common.Address
    dataProvider    common.Address
}

// NewAaveV3Protocol 创建 Aave V3 协议
func NewAaveV3Protocol(client *ethclient.Client) *AaveV3Protocol {
    return &AaveV3Protocol{
        client:        client,
        poolAddress:   common.HexToAddress("0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2"),
        oracleAddress: common.HexToAddress("0x54586bE62E3c3580375aE3723C145253060Ca0C2"),
        dataProvider:  common.HexToAddress("0x7B4EB56E7CD4b454BA8ff71E4518426369a138a3"),
    }
}

// GetProtocol 获取协议类型
func (a *AaveV3Protocol) GetProtocol() Protocol {
    return ProtocolAaveV3
}

// GetAccountData 获取账户数据
func (a *AaveV3Protocol) GetAccountData(ctx context.Context, user common.Address) (*AccountData, error) {
    // getUserAccountData(address user) returns (
    //   uint256 totalCollateralBase,
    //   uint256 totalDebtBase,
    //   uint256 availableBorrowsBase,
    //   uint256 currentLiquidationThreshold,
    //   uint256 ltv,
    //   uint256 healthFactor
    // )
    selector := common.Hex2Bytes("bf92857c")
    data := make([]byte, 36)
    copy(data[0:4], selector)
    copy(data[16:36], user.Bytes())

    result, err := a.client.CallContract(ctx, ethereum.CallMsg{
        To:   &a.poolAddress,
        Data: data,
    }, nil)
    if err != nil {
        return nil, err
    }

    if len(result) < 192 {
        return nil, fmt.Errorf("无效的返回数据")
    }

    return &AccountData{
        User:                    user,
        TotalCollateralETH:      new(big.Int).SetBytes(result[0:32]),
        TotalDebtETH:            new(big.Int).SetBytes(result[32:64]),
        AvailableBorrowsETH:     new(big.Int).SetBytes(result[64:96]),
        CurrentLiquidationThreshold: new(big.Int).SetBytes(result[96:128]),
        LTV:                     new(big.Int).SetBytes(result[128:160]),
        HealthFactor:            new(big.Int).SetBytes(result[160:192]),
    }, nil
}

// GetLiquidatableAccounts 获取可清算账户
func (a *AaveV3Protocol) GetLiquidatableAccounts(ctx context.Context) ([]*LiquidatableAccount, error) {
    // 实际实现需要索引历史事件或使用 The Graph
    // 这里展示基于事件的方法
    return a.scanForLiquidatableAccounts(ctx)
}

// scanForLiquidatableAccounts 扫描可清算账户
func (a *AaveV3Protocol) scanForLiquidatableAccounts(ctx context.Context) ([]*LiquidatableAccount, error) {
    // 获取最近的 Borrow 事件
    borrowTopic := common.HexToHash("0xb3d084820fb1a9decffb176436bd02558d15fac9b0ddfed8c465bc7359d7dce0")

    currentBlock, err := a.client.BlockNumber(ctx)
    if err != nil {
        return nil, err
    }

    // 扫描最近 1000 个区块
    fromBlock := currentBlock - 1000
    if fromBlock < 0 {
        fromBlock = 0
    }

    query := ethereum.FilterQuery{
        FromBlock: big.NewInt(int64(fromBlock)),
        ToBlock:   big.NewInt(int64(currentBlock)),
        Addresses: []common.Address{a.poolAddress},
        Topics:    [][]common.Hash{{borrowTopic}},
    }

    logs, err := a.client.FilterLogs(ctx, query)
    if err != nil {
        return nil, err
    }

    // 收集所有借款用户
    users := make(map[common.Address]bool)
    for _, log := range logs {
        if len(log.Topics) >= 3 {
            user := common.BytesToAddress(log.Topics[2].Bytes())
            users[user] = true
        }
    }

    // 检查每个用户的健康因子
    var liquidatable []*LiquidatableAccount
    for user := range users {
        accountData, err := a.GetAccountData(ctx, user)
        if err != nil {
            continue
        }

        // 健康因子 < 1e18 表示可清算
        oneEther := new(big.Int).Exp(big.NewInt(10), big.NewInt(18), nil)
        if accountData.HealthFactor.Cmp(oneEther) < 0 {
            account := &LiquidatableAccount{
                User:          user,
                Protocol:      ProtocolAaveV3,
                HealthFactor:  accountData.HealthFactor,
                TotalCollateral: accountData.TotalCollateralETH,
                TotalDebt:     accountData.TotalDebtETH,
            }

            // 获取详细头寸
            positions, err := a.getUserPositions(ctx, user)
            if err == nil {
                account.CollateralAssets = positions.Collaterals
                account.DebtAssets = positions.Debts
            }

            liquidatable = append(liquidatable, account)
        }
    }

    return liquidatable, nil
}

// UserPositions 用户头寸
type UserPositions struct {
    Collaterals []CollateralPosition
    Debts       []DebtPosition
}

// getUserPositions 获取用户头寸详情
func (a *AaveV3Protocol) getUserPositions(ctx context.Context, user common.Address) (*UserPositions, error) {
    // getUserReserveData(address asset, address user)
    // 需要遍历所有资产
    reserves := a.getReservesList(ctx)

    positions := &UserPositions{}

    for _, reserve := range reserves {
        // 获取用户在该资产的头寸
        collateral, debt, err := a.getUserReserveData(ctx, reserve, user)
        if err != nil {
            continue
        }

        if collateral.Sign() > 0 {
            positions.Collaterals = append(positions.Collaterals, CollateralPosition{
                Asset:  reserve,
                Amount: collateral,
            })
        }

        if debt.Sign() > 0 {
            positions.Debts = append(positions.Debts, DebtPosition{
                Asset:  reserve,
                Amount: debt,
            })
        }
    }

    return positions, nil
}

// getReservesList 获取储备列表
func (a *AaveV3Protocol) getReservesList(ctx context.Context) []common.Address {
    // getReservesList() returns (address[] memory)
    selector := common.Hex2Bytes("d1946dbc")

    result, err := a.client.CallContract(ctx, ethereum.CallMsg{
        To:   &a.poolAddress,
        Data: selector,
    }, nil)
    if err != nil {
        return nil
    }

    // 解析动态数组
    if len(result) < 64 {
        return nil
    }

    // offset := new(big.Int).SetBytes(result[0:32])
    length := new(big.Int).SetBytes(result[32:64])

    var reserves []common.Address
    for i := int64(0); i < length.Int64(); i++ {
        start := 64 + i*32
        if start+32 <= int64(len(result)) {
            addr := common.BytesToAddress(result[start+12 : start+32])
            reserves = append(reserves, addr)
        }
    }

    return reserves
}

// getUserReserveData 获取用户储备数据
func (a *AaveV3Protocol) getUserReserveData(
    ctx context.Context,
    asset, user common.Address,
) (*big.Int, *big.Int, error) {
    // 从 DataProvider 获取
    // getUserReserveData(address asset, address user) returns (
    //   uint256 currentATokenBalance,
    //   uint256 currentStableDebt,
    //   uint256 currentVariableDebt,
    //   ...
    // )
    selector := common.Hex2Bytes("28dd2d01")
    data := make([]byte, 68)
    copy(data[0:4], selector)
    copy(data[16:36], asset.Bytes())
    copy(data[48:68], user.Bytes())

    result, err := a.client.CallContract(ctx, ethereum.CallMsg{
        To:   &a.dataProvider,
        Data: data,
    }, nil)
    if err != nil {
        return nil, nil, err
    }

    if len(result) < 96 {
        return nil, nil, fmt.Errorf("无效的返回数据")
    }

    collateral := new(big.Int).SetBytes(result[0:32])
    stableDebt := new(big.Int).SetBytes(result[32:64])
    variableDebt := new(big.Int).SetBytes(result[64:96])

    totalDebt := new(big.Int).Add(stableDebt, variableDebt)

    return collateral, totalDebt, nil
}

// CalculateLiquidation 计算清算参数
func (a *AaveV3Protocol) CalculateLiquidation(
    ctx context.Context,
    account *LiquidatableAccount,
) (*LiquidationParams, error) {
    if len(account.CollateralAssets) == 0 || len(account.DebtAssets) == 0 {
        return nil, fmt.Errorf("无有效头寸")
    }

    // 选择最佳清算组合
    bestCollateral := account.CollateralAssets[0]
    bestDebt := account.DebtAssets[0]

    // 获取清算奖励 (通常 5-10%)
    liquidationBonus := big.NewInt(10500) // 105%

    // 计算可清算债务 (最多 50%)
    maxDebtToCover := new(big.Int).Div(bestDebt.Amount, big.NewInt(2))

    // 计算获得的抵押品
    // collateralToReceive = debtToCover * debtPrice / collateralPrice * liquidationBonus
    collateralToReceive := a.calculateCollateralAmount(
        ctx,
        bestDebt.Asset,
        bestCollateral.Asset,
        maxDebtToCover,
        liquidationBonus,
    )

    // 估算利润
    estimatedProfit := a.estimateLiquidationProfit(
        ctx,
        bestDebt.Asset,
        bestCollateral.Asset,
        maxDebtToCover,
        collateralToReceive,
    )

    return &LiquidationParams{
        User:               account.User,
        CollateralAsset:    bestCollateral.Asset,
        DebtAsset:          bestDebt.Asset,
        DebtToCover:        maxDebtToCover,
        CollateralToReceive: collateralToReceive,
        LiquidationBonus:   liquidationBonus,
        EstimatedProfit:    estimatedProfit,
        UseFlashLoan:       true, // 通常使用闪电贷
    }, nil
}

// calculateCollateralAmount 计算抵押品数量
func (a *AaveV3Protocol) calculateCollateralAmount(
    ctx context.Context,
    debtAsset, collateralAsset common.Address,
    debtAmount, liquidationBonus *big.Int,
) *big.Int {
    debtPrice := a.getAssetPrice(ctx, debtAsset)
    collateralPrice := a.getAssetPrice(ctx, collateralAsset)

    if collateralPrice.Sign() == 0 {
        return big.NewInt(0)
    }

    // collateral = debt * debtPrice * bonus / collateralPrice
    collateral := new(big.Int).Mul(debtAmount, debtPrice)
    collateral.Mul(collateral, liquidationBonus)
    collateral.Div(collateral, collateralPrice)
    collateral.Div(collateral, big.NewInt(10000))

    return collateral
}

// getAssetPrice 获取资产价格
func (a *AaveV3Protocol) getAssetPrice(ctx context.Context, asset common.Address) *big.Int {
    // getAssetPrice(address asset) returns (uint256)
    selector := common.Hex2Bytes("b3596f07")
    data := make([]byte, 36)
    copy(data[0:4], selector)
    copy(data[16:36], asset.Bytes())

    result, err := a.client.CallContract(ctx, ethereum.CallMsg{
        To:   &a.oracleAddress,
        Data: data,
    }, nil)
    if err != nil {
        return big.NewInt(0)
    }

    return new(big.Int).SetBytes(result)
}

// estimateLiquidationProfit 估算清算利润
func (a *AaveV3Protocol) estimateLiquidationProfit(
    ctx context.Context,
    debtAsset, collateralAsset common.Address,
    debtAmount, collateralAmount *big.Int,
) *big.Int {
    debtPrice := a.getAssetPrice(ctx, debtAsset)
    collateralPrice := a.getAssetPrice(ctx, collateralAsset)

    // 债务价值
    debtValue := new(big.Int).Mul(debtAmount, debtPrice)

    // 抵押品价值
    collateralValue := new(big.Int).Mul(collateralAmount, collateralPrice)

    // 利润 = 抵押品价值 - 债务价值 - Gas 成本
    profit := new(big.Int).Sub(collateralValue, debtValue)

    // 扣除估算的 Gas 成本 (约 0.01 ETH)
    gasCost := new(big.Int).Mul(big.NewInt(1e16), big.NewInt(1))
    profit.Sub(profit, gasCost)

    return profit
}

// ExecuteLiquidation 执行清算
func (a *AaveV3Protocol) ExecuteLiquidation(
    ctx context.Context,
    params *LiquidationParams,
) (common.Hash, error) {
    // liquidationCall(
    //   address collateralAsset,
    //   address debtAsset,
    //   address user,
    //   uint256 debtToCover,
    //   bool receiveAToken
    // )
    // selector: 0x00a718a9

    // 实际执行需要签名交易
    // 这里返回示例
    return common.Hash{}, fmt.Errorf("需要实现交易签名和发送")
}
```

---

## 第一百零八节：Compound 清算

### Compound V3 清算实现

```go
package liquidation

import (
    "context"
    "fmt"
    "math/big"

    "github.com/ethereum/go-ethereum"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/ethclient"
)

// CompoundV3Protocol Compound V3 协议
type CompoundV3Protocol struct {
    client      *ethclient.Client
    cometAddress common.Address // Comet (cUSDC) 地址
}

// NewCompoundV3Protocol 创建 Compound V3 协议
func NewCompoundV3Protocol(client *ethclient.Client) *CompoundV3Protocol {
    return &CompoundV3Protocol{
        client:       client,
        cometAddress: common.HexToAddress("0xc3d688B66703497DAA19211EEdff47f25384cdc3"), // USDC Comet
    }
}

// GetProtocol 获取协议类型
func (c *CompoundV3Protocol) GetProtocol() Protocol {
    return ProtocolCompoundV3
}

// GetAccountData 获取账户数据
func (c *CompoundV3Protocol) GetAccountData(ctx context.Context, user common.Address) (*AccountData, error) {
    // 获取借款余额
    borrowBalance, err := c.getBorrowBalance(ctx, user)
    if err != nil {
        return nil, err
    }

    // 获取抵押品价值
    collateralValue, err := c.getCollateralValue(ctx, user)
    if err != nil {
        return nil, err
    }

    // 计算健康因子
    healthFactor := c.calculateHealthFactor(collateralValue, borrowBalance)

    return &AccountData{
        User:               user,
        TotalCollateralETH: collateralValue,
        TotalDebtETH:       borrowBalance,
        HealthFactor:       healthFactor,
    }, nil
}

// getBorrowBalance 获取借款余额
func (c *CompoundV3Protocol) getBorrowBalance(ctx context.Context, user common.Address) (*big.Int, error) {
    // borrowBalanceOf(address account) returns (uint256)
    selector := common.Hex2Bytes("374c49b4")
    data := make([]byte, 36)
    copy(data[0:4], selector)
    copy(data[16:36], user.Bytes())

    result, err := c.client.CallContract(ctx, ethereum.CallMsg{
        To:   &c.cometAddress,
        Data: data,
    }, nil)
    if err != nil {
        return nil, err
    }

    return new(big.Int).SetBytes(result), nil
}

// getCollateralValue 获取抵押品价值
func (c *CompoundV3Protocol) getCollateralValue(ctx context.Context, user common.Address) (*big.Int, error) {
    // 获取所有抵押资产
    assets := c.getCollateralAssets(ctx)

    totalValue := big.NewInt(0)

    for _, asset := range assets {
        balance, err := c.getCollateralBalance(ctx, user, asset)
        if err != nil {
            continue
        }

        if balance.Sign() > 0 {
            price := c.getAssetPrice(ctx, asset)
            value := new(big.Int).Mul(balance, price)
            totalValue.Add(totalValue, value)
        }
    }

    return totalValue, nil
}

// getCollateralAssets 获取抵押资产列表
func (c *CompoundV3Protocol) getCollateralAssets(ctx context.Context) []common.Address {
    // numAssets() returns (uint8)
    selector := common.Hex2Bytes("a46fe83b")

    result, err := c.client.CallContract(ctx, ethereum.CallMsg{
        To:   &c.cometAddress,
        Data: selector,
    }, nil)
    if err != nil {
        return nil
    }

    numAssets := new(big.Int).SetBytes(result).Int64()

    var assets []common.Address
    for i := int64(0); i < numAssets; i++ {
        asset := c.getAssetInfo(ctx, i)
        if asset != (common.Address{}) {
            assets = append(assets, asset)
        }
    }

    return assets
}

// getAssetInfo 获取资产信息
func (c *CompoundV3Protocol) getAssetInfo(ctx context.Context, index int64) common.Address {
    // getAssetInfo(uint8 i) returns (AssetInfo memory)
    selector := common.Hex2Bytes("c8c7fe6b")
    data := make([]byte, 36)
    copy(data[0:4], selector)
    data[35] = byte(index)

    result, err := c.client.CallContract(ctx, ethereum.CallMsg{
        To:   &c.cometAddress,
        Data: data,
    }, nil)
    if err != nil {
        return common.Address{}
    }

    // AssetInfo 结构: offset, asset, ...
    if len(result) >= 64 {
        return common.BytesToAddress(result[44:64])
    }

    return common.Address{}
}

// getCollateralBalance 获取抵押品余额
func (c *CompoundV3Protocol) getCollateralBalance(
    ctx context.Context,
    user, asset common.Address,
) (*big.Int, error) {
    // collateralBalanceOf(address account, address asset) returns (uint128)
    selector := common.Hex2Bytes("44c1e5eb")
    data := make([]byte, 68)
    copy(data[0:4], selector)
    copy(data[16:36], user.Bytes())
    copy(data[48:68], asset.Bytes())

    result, err := c.client.CallContract(ctx, ethereum.CallMsg{
        To:   &c.cometAddress,
        Data: data,
    }, nil)
    if err != nil {
        return nil, err
    }

    return new(big.Int).SetBytes(result), nil
}

// getAssetPrice 获取资产价格
func (c *CompoundV3Protocol) getAssetPrice(ctx context.Context, asset common.Address) *big.Int {
    // getPrice(address priceFeed) returns (uint256)
    // 需要先获取 priceFeed 地址
    return big.NewInt(1e8) // 简化，实际需要调用预言机
}

// calculateHealthFactor 计算健康因子
func (c *CompoundV3Protocol) calculateHealthFactor(collateral, debt *big.Int) *big.Int {
    if debt.Sign() == 0 {
        return new(big.Int).Mul(big.NewInt(1e18), big.NewInt(100)) // 无债务，返回很大的值
    }

    // healthFactor = collateral * liquidationFactor / debt
    // Compound V3 使用不同的清算阈值机制
    liquidationFactor := big.NewInt(9000) // 90%

    hf := new(big.Int).Mul(collateral, liquidationFactor)
    hf.Mul(hf, big.NewInt(1e18))
    hf.Div(hf, debt)
    hf.Div(hf, big.NewInt(10000))

    return hf
}

// IsLiquidatable 检查是否可清算
func (c *CompoundV3Protocol) IsLiquidatable(ctx context.Context, user common.Address) (bool, error) {
    // isLiquidatable(address account) returns (bool)
    selector := common.Hex2Bytes("042e02cf")
    data := make([]byte, 36)
    copy(data[0:4], selector)
    copy(data[16:36], user.Bytes())

    result, err := c.client.CallContract(ctx, ethereum.CallMsg{
        To:   &c.cometAddress,
        Data: data,
    }, nil)
    if err != nil {
        return false, err
    }

    return new(big.Int).SetBytes(result).Sign() > 0, nil
}

// GetLiquidatableAccounts 获取可清算账户
func (c *CompoundV3Protocol) GetLiquidatableAccounts(ctx context.Context) ([]*LiquidatableAccount, error) {
    // 通过事件扫描或 The Graph 获取借款人列表
    return c.scanBorrowers(ctx)
}

// scanBorrowers 扫描借款人
func (c *CompoundV3Protocol) scanBorrowers(ctx context.Context) ([]*LiquidatableAccount, error) {
    // Withdraw 事件表示借款
    // event Withdraw(address indexed src, address indexed to, uint256 amount)
    withdrawTopic := common.HexToHash("0x9b1bfa7fa9ee420a16e124f794c35ac9f90472acc99140eb2f6447c714cad8eb")

    currentBlock, err := c.client.BlockNumber(ctx)
    if err != nil {
        return nil, err
    }

    fromBlock := currentBlock - 5000
    if fromBlock < 0 {
        fromBlock = 0
    }

    query := ethereum.FilterQuery{
        FromBlock: big.NewInt(int64(fromBlock)),
        ToBlock:   big.NewInt(int64(currentBlock)),
        Addresses: []common.Address{c.cometAddress},
        Topics:    [][]common.Hash{{withdrawTopic}},
    }

    logs, err := c.client.FilterLogs(ctx, query)
    if err != nil {
        return nil, err
    }

    users := make(map[common.Address]bool)
    for _, log := range logs {
        if len(log.Topics) >= 2 {
            user := common.BytesToAddress(log.Topics[1].Bytes())
            users[user] = true
        }
    }

    var liquidatable []*LiquidatableAccount
    for user := range users {
        isLiq, err := c.IsLiquidatable(ctx, user)
        if err != nil {
            continue
        }

        if isLiq {
            accountData, _ := c.GetAccountData(ctx, user)
            liquidatable = append(liquidatable, &LiquidatableAccount{
                User:          user,
                Protocol:      ProtocolCompoundV3,
                HealthFactor:  accountData.HealthFactor,
                TotalCollateral: accountData.TotalCollateralETH,
                TotalDebt:     accountData.TotalDebtETH,
            })
        }
    }

    return liquidatable, nil
}

// CalculateLiquidation 计算清算参数
func (c *CompoundV3Protocol) CalculateLiquidation(
    ctx context.Context,
    account *LiquidatableAccount,
) (*LiquidationParams, error) {
    // Compound V3 使用 absorb 机制
    // absorb(address absorber, address[] memory accounts)

    return &LiquidationParams{
        User:            account.User,
        DebtToCover:     account.TotalDebt,
        EstimatedProfit: c.estimateAbsorbProfit(ctx, account),
        UseFlashLoan:    false, // Compound V3 absorb 不需要资金
    }, nil
}

// estimateAbsorbProfit 估算 absorb 利润
func (c *CompoundV3Protocol) estimateAbsorbProfit(
    ctx context.Context,
    account *LiquidatableAccount,
) *big.Int {
    // absorb 后获得的抵押品会进入协议储备
    // 清算人通过 buyCollateral 购买折扣抵押品获利

    // 简化计算
    discount := big.NewInt(500) // 5% 折扣
    profit := new(big.Int).Mul(account.TotalCollateral, discount)
    profit.Div(profit, big.NewInt(10000))

    return profit
}

// ExecuteLiquidation 执行清算
func (c *CompoundV3Protocol) ExecuteLiquidation(
    ctx context.Context,
    params *LiquidationParams,
) (common.Hash, error) {
    // absorb(address absorber, address[] memory accounts)
    // selector: 0x7b3f6bf2

    return common.Hash{}, fmt.Errorf("需要实现交易签名和发送")
}

// BuyCollateral 购买折扣抵押品
func (c *CompoundV3Protocol) BuyCollateral(
    ctx context.Context,
    asset common.Address,
    minAmount *big.Int,
    baseAmount *big.Int,
    recipient common.Address,
) (common.Hash, error) {
    // buyCollateral(address asset, uint256 minAmount, uint256 baseAmount, address recipient)
    // selector: 0xac753aab

    return common.Hash{}, fmt.Errorf("需要实现交易签名和发送")
}
```

---

## 第一百零九节：清算机器人架构

### 完整清算机器人

```go
package liquidation

import (
    "context"
    "crypto/ecdsa"
    "fmt"
    "math/big"
    "sort"
    "sync"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/crypto"
    "github.com/ethereum/go-ethereum/ethclient"
)

// LiquidationBot 清算机器人
type LiquidationBot struct {
    client      *ethclient.Client
    privateKey  *ecdsa.PrivateKey
    address     common.Address

    // 协议适配器
    protocols   []LendingProtocol

    // 配置
    config      *BotConfig

    // 状态
    pendingLiquidations map[common.Address]*PendingLiquidation
    mu                  sync.RWMutex

    // 统计
    stats       *LiquidationStats
}

// BotConfig 机器人配置
type BotConfig struct {
    MinProfitETH        *big.Int      // 最小利润阈值
    MaxGasPrice         *big.Int      // 最大 Gas 价格
    ScanInterval        time.Duration // 扫描间隔
    MaxConcurrent       int           // 最大并发清算数
    UseFlashbots        bool          // 是否使用 Flashbots
    FlashbotsRelayURL   string
    EnabledProtocols    []Protocol
}

// PendingLiquidation 待处理清算
type PendingLiquidation struct {
    Account     *LiquidatableAccount
    Params      *LiquidationParams
    TxHash      common.Hash
    Submitted   time.Time
    Status      LiquidationStatus
}

// LiquidationStatus 清算状态
type LiquidationStatus int

const (
    StatusPending LiquidationStatus = iota
    StatusSubmitted
    StatusConfirmed
    StatusFailed
)

// LiquidationStats 清算统计
type LiquidationStats struct {
    TotalScanned      uint64
    TotalLiquidatable uint64
    TotalAttempted    uint64
    TotalSuccessful   uint64
    TotalProfit       *big.Int
    TotalGasSpent     *big.Int
    mu                sync.Mutex
}

// NewLiquidationBot 创建清算机器人
func NewLiquidationBot(
    client *ethclient.Client,
    privateKey *ecdsa.PrivateKey,
    config *BotConfig,
) *LiquidationBot {
    bot := &LiquidationBot{
        client:              client,
        privateKey:          privateKey,
        address:             crypto.PubkeyToAddress(privateKey.PublicKey),
        protocols:           make([]LendingProtocol, 0),
        config:              config,
        pendingLiquidations: make(map[common.Address]*PendingLiquidation),
        stats: &LiquidationStats{
            TotalProfit:   big.NewInt(0),
            TotalGasSpent: big.NewInt(0),
        },
    }

    // 初始化协议
    bot.initProtocols()

    return bot
}

// initProtocols 初始化协议
func (b *LiquidationBot) initProtocols() {
    for _, p := range b.config.EnabledProtocols {
        switch p {
        case ProtocolAaveV3:
            b.protocols = append(b.protocols, NewAaveV3Protocol(b.client))
        case ProtocolCompoundV3:
            b.protocols = append(b.protocols, NewCompoundV3Protocol(b.client))
        }
    }
}

// Start 启动机器人
func (b *LiquidationBot) Start(ctx context.Context) error {
    fmt.Println("启动清算机器人...")

    // 启动扫描循环
    go b.scanLoop(ctx)

    // 启动确认监控
    go b.monitorConfirmations(ctx)

    <-ctx.Done()
    return nil
}

// scanLoop 扫描循环
func (b *LiquidationBot) scanLoop(ctx context.Context) {
    ticker := time.NewTicker(b.config.ScanInterval)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            b.scan(ctx)
        }
    }
}

// scan 执行扫描
func (b *LiquidationBot) scan(ctx context.Context) {
    var allLiquidatable []*LiquidatableAccount

    // 并行扫描所有协议
    var wg sync.WaitGroup
    var mu sync.Mutex

    for _, protocol := range b.protocols {
        wg.Add(1)
        go func(p LendingProtocol) {
            defer wg.Done()

            accounts, err := p.GetLiquidatableAccounts(ctx)
            if err != nil {
                fmt.Printf("扫描协议 %d 失败: %v\n", p.GetProtocol(), err)
                return
            }

            mu.Lock()
            allLiquidatable = append(allLiquidatable, accounts...)
            mu.Unlock()
        }(protocol)
    }
    wg.Wait()

    b.stats.mu.Lock()
    b.stats.TotalLiquidatable += uint64(len(allLiquidatable))
    b.stats.mu.Unlock()

    if len(allLiquidatable) == 0 {
        return
    }

    fmt.Printf("发现 %d 个可清算账户\n", len(allLiquidatable))

    // 按利润排序
    opportunities := b.evaluateOpportunities(ctx, allLiquidatable)

    // 执行最优机会
    for _, opp := range opportunities {
        if b.shouldExecute(opp) {
            go b.executeLiquidation(ctx, opp)
        }
    }
}

// LiquidationOpportunity 清算机会
type LiquidationOpportunity struct {
    Account     *LiquidatableAccount
    Params      *LiquidationParams
    Protocol    LendingProtocol
    Score       float64
}

// evaluateOpportunities 评估机会
func (b *LiquidationBot) evaluateOpportunities(
    ctx context.Context,
    accounts []*LiquidatableAccount,
) []*LiquidationOpportunity {
    var opportunities []*LiquidationOpportunity

    for _, account := range accounts {
        // 找到对应的协议
        var protocol LendingProtocol
        for _, p := range b.protocols {
            if p.GetProtocol() == account.Protocol {
                protocol = p
                break
            }
        }

        if protocol == nil {
            continue
        }

        // 计算清算参数
        params, err := protocol.CalculateLiquidation(ctx, account)
        if err != nil {
            continue
        }

        // 检查利润是否满足阈值
        if params.EstimatedProfit.Cmp(b.config.MinProfitETH) < 0 {
            continue
        }

        // 计算得分
        score := b.calculateScore(account, params)

        opportunities = append(opportunities, &LiquidationOpportunity{
            Account:  account,
            Params:   params,
            Protocol: protocol,
            Score:    score,
        })
    }

    // 按得分排序
    sort.Slice(opportunities, func(i, j int) bool {
        return opportunities[i].Score > opportunities[j].Score
    })

    return opportunities
}

// calculateScore 计算机会得分
func (b *LiquidationBot) calculateScore(
    account *LiquidatableAccount,
    params *LiquidationParams,
) float64 {
    score := 0.0

    // 利润因素 (权重 50%)
    profitETH, _ := new(big.Float).SetInt(params.EstimatedProfit).Float64()
    profitETH /= 1e18
    score += profitETH * 0.5

    // 健康因子因素 (越低越紧急，权重 30%)
    hf, _ := new(big.Float).SetInt(account.HealthFactor).Float64()
    hf /= 1e18
    if hf < 1.0 && hf > 0 {
        urgency := (1.0 - hf) * 10 // 0-10 分
        score += urgency * 0.3
    }

    // 债务规模因素 (权重 20%)
    debtETH, _ := new(big.Float).SetInt(account.TotalDebt).Float64()
    debtETH /= 1e18
    if debtETH > 100 {
        score += 2.0 * 0.2 // 大额清算
    } else if debtETH > 10 {
        score += 1.0 * 0.2
    }

    return score
}

// shouldExecute 判断是否应该执行
func (b *LiquidationBot) shouldExecute(opp *LiquidationOpportunity) bool {
    b.mu.RLock()
    defer b.mu.RUnlock()

    // 检查是否已有待处理的清算
    if _, exists := b.pendingLiquidations[opp.Account.User]; exists {
        return false
    }

    // 检查并发数
    if len(b.pendingLiquidations) >= b.config.MaxConcurrent {
        return false
    }

    // 检查 Gas 价格
    gasPrice, _ := b.client.SuggestGasPrice(context.Background())
    if gasPrice.Cmp(b.config.MaxGasPrice) > 0 {
        fmt.Printf("Gas 价格过高: %s > %s\n", gasPrice.String(), b.config.MaxGasPrice.String())
        return false
    }

    return true
}

// executeLiquidation 执行清算
func (b *LiquidationBot) executeLiquidation(ctx context.Context, opp *LiquidationOpportunity) {
    fmt.Printf("执行清算: 用户=%s, 协议=%d, 预估利润=%s\n",
        opp.Account.User.Hex()[:10],
        opp.Account.Protocol,
        opp.Params.EstimatedProfit.String())

    // 记录待处理
    b.mu.Lock()
    b.pendingLiquidations[opp.Account.User] = &PendingLiquidation{
        Account:   opp.Account,
        Params:    opp.Params,
        Submitted: time.Now(),
        Status:    StatusPending,
    }
    b.mu.Unlock()

    defer func() {
        b.mu.Lock()
        delete(b.pendingLiquidations, opp.Account.User)
        b.mu.Unlock()
    }()

    var txHash common.Hash
    var err error

    if opp.Params.UseFlashLoan {
        txHash, err = b.executeWithFlashLoan(ctx, opp)
    } else {
        txHash, err = opp.Protocol.ExecuteLiquidation(ctx, opp.Params)
    }

    if err != nil {
        fmt.Printf("清算失败: %v\n", err)
        return
    }

    b.mu.Lock()
    if pending, exists := b.pendingLiquidations[opp.Account.User]; exists {
        pending.TxHash = txHash
        pending.Status = StatusSubmitted
    }
    b.mu.Unlock()

    b.stats.mu.Lock()
    b.stats.TotalAttempted++
    b.stats.mu.Unlock()

    fmt.Printf("清算交易已提交: %s\n", txHash.Hex())
}

// executeWithFlashLoan 使用闪电贷执行清算
func (b *LiquidationBot) executeWithFlashLoan(
    ctx context.Context,
    opp *LiquidationOpportunity,
) (common.Hash, error) {
    // 构建闪电贷清算交易
    // 1. 从 Aave/Balancer 借入 debtAsset
    // 2. 调用 liquidationCall
    // 3. 在 DEX 交换 collateral -> debtAsset
    // 4. 偿还闪电贷

    if b.config.UseFlashbots {
        return b.submitViaFlashbots(ctx, opp)
    }

    return b.submitDirect(ctx, opp)
}

// submitViaFlashbots 通过 Flashbots 提交
func (b *LiquidationBot) submitViaFlashbots(
    ctx context.Context,
    opp *LiquidationOpportunity,
) (common.Hash, error) {
    // 构建 bundle
    // 使用 Flashbots API 提交

    return common.Hash{}, fmt.Errorf("Flashbots 提交未实现")
}

// submitDirect 直接提交
func (b *LiquidationBot) submitDirect(
    ctx context.Context,
    opp *LiquidationOpportunity,
) (common.Hash, error) {
    // 构建交易
    tx, err := b.buildLiquidationTx(ctx, opp)
    if err != nil {
        return common.Hash{}, err
    }

    // 发送交易
    err = b.client.SendTransaction(ctx, tx)
    if err != nil {
        return common.Hash{}, err
    }

    return tx.Hash(), nil
}

// buildLiquidationTx 构建清算交易
func (b *LiquidationBot) buildLiquidationTx(
    ctx context.Context,
    opp *LiquidationOpportunity,
) (*types.Transaction, error) {
    // 获取 nonce
    nonce, err := b.client.PendingNonceAt(ctx, b.address)
    if err != nil {
        return nil, err
    }

    // 获取 gas 价格
    gasPrice, err := b.client.SuggestGasPrice(ctx)
    if err != nil {
        return nil, err
    }

    // 构建调用数据
    calldata := b.buildLiquidationCalldata(opp)

    // 目标合约 (清算执行合约)
    to := b.getLiquidationContract(opp.Account.Protocol)

    tx := types.NewTransaction(
        nonce,
        to,
        big.NewInt(0),
        uint64(500000),
        gasPrice,
        calldata,
    )

    // 签名
    chainID, _ := b.client.ChainID(ctx)
    signedTx, err := types.SignTx(tx, types.NewEIP155Signer(chainID), b.privateKey)
    if err != nil {
        return nil, err
    }

    return signedTx, nil
}

// buildLiquidationCalldata 构建清算调用数据
func (b *LiquidationBot) buildLiquidationCalldata(opp *LiquidationOpportunity) []byte {
    switch opp.Account.Protocol {
    case ProtocolAaveV3:
        return b.buildAaveLiquidationCalldata(opp.Params)
    case ProtocolCompoundV3:
        return b.buildCompoundAbsorbCalldata(opp.Params)
    default:
        return nil
    }
}

// buildAaveLiquidationCalldata 构建 Aave 清算调用数据
func (b *LiquidationBot) buildAaveLiquidationCalldata(params *LiquidationParams) []byte {
    // liquidationCall(address,address,address,uint256,bool)
    // selector: 0x00a718a9
    selector := common.Hex2Bytes("00a718a9")

    data := make([]byte, 164)
    copy(data[0:4], selector)
    copy(data[16:36], params.CollateralAsset.Bytes())
    copy(data[48:68], params.DebtAsset.Bytes())
    copy(data[80:100], params.User.Bytes())
    params.DebtToCover.FillBytes(data[100:132])
    // receiveAToken = false
    data[163] = 0

    return data
}

// buildCompoundAbsorbCalldata 构建 Compound absorb 调用数据
func (b *LiquidationBot) buildCompoundAbsorbCalldata(params *LiquidationParams) []byte {
    // absorb(address,address[])
    // selector: 0x7b3f6bf2
    selector := common.Hex2Bytes("7b3f6bf2")

    data := make([]byte, 132)
    copy(data[0:4], selector)
    copy(data[16:36], b.address.Bytes()) // absorber
    // accounts array offset
    new(big.Int).SetInt64(64).FillBytes(data[36:68])
    // array length
    new(big.Int).SetInt64(1).FillBytes(data[68:100])
    // account[0]
    copy(data[112:132], params.User.Bytes())

    return data
}

// getLiquidationContract 获取清算合约地址
func (b *LiquidationBot) getLiquidationContract(protocol Protocol) common.Address {
    switch protocol {
    case ProtocolAaveV3:
        return common.HexToAddress("0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2")
    case ProtocolCompoundV3:
        return common.HexToAddress("0xc3d688B66703497DAA19211EEdff47f25384cdc3")
    default:
        return common.Address{}
    }
}

// monitorConfirmations 监控确认
func (b *LiquidationBot) monitorConfirmations(ctx context.Context) {
    ticker := time.NewTicker(3 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            b.checkConfirmations(ctx)
        }
    }
}

// checkConfirmations 检查确认状态
func (b *LiquidationBot) checkConfirmations(ctx context.Context) {
    b.mu.RLock()
    pending := make([]*PendingLiquidation, 0)
    for _, p := range b.pendingLiquidations {
        if p.Status == StatusSubmitted {
            pending = append(pending, p)
        }
    }
    b.mu.RUnlock()

    for _, p := range pending {
        receipt, err := b.client.TransactionReceipt(ctx, p.TxHash)
        if err != nil {
            continue
        }

        b.mu.Lock()
        if receipt.Status == types.ReceiptStatusSuccessful {
            p.Status = StatusConfirmed
            b.stats.mu.Lock()
            b.stats.TotalSuccessful++
            b.stats.TotalProfit.Add(b.stats.TotalProfit, p.Params.EstimatedProfit)
            gasUsed := new(big.Int).Mul(
                big.NewInt(int64(receipt.GasUsed)),
                receipt.EffectiveGasPrice,
            )
            b.stats.TotalGasSpent.Add(b.stats.TotalGasSpent, gasUsed)
            b.stats.mu.Unlock()

            fmt.Printf("清算成功: 用户=%s, Gas=%d\n",
                p.Account.User.Hex()[:10],
                receipt.GasUsed)
        } else {
            p.Status = StatusFailed
            fmt.Printf("清算失败: 用户=%s\n", p.Account.User.Hex()[:10])
        }
        b.mu.Unlock()
    }
}

// GetStats 获取统计信息
func (b *LiquidationBot) GetStats() *LiquidationStats {
    b.stats.mu.Lock()
    defer b.stats.mu.Unlock()

    return &LiquidationStats{
        TotalScanned:      b.stats.TotalScanned,
        TotalLiquidatable: b.stats.TotalLiquidatable,
        TotalAttempted:    b.stats.TotalAttempted,
        TotalSuccessful:   b.stats.TotalSuccessful,
        TotalProfit:       new(big.Int).Set(b.stats.TotalProfit),
        TotalGasSpent:     new(big.Int).Set(b.stats.TotalGasSpent),
    }
}
```

---

## 本文总结

本文深入探讨了 DeFi 借贷协议清算机制，主要涵盖：

### 清算流程对比

| 协议 | 清算触发条件 | 清算机制 | 清算奖励 |
|------|-------------|---------|---------|
| Aave V3 | 健康因子 < 1 | liquidationCall | 5-10% |
| Compound V3 | isLiquidatable | absorb + buyCollateral | 5% 折扣 |

### 核心组件

```
清算机器人架构：
┌─────────────────────────────────────────────────────────────────┐
│                       清算机器人                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │  扫描模块    │────▶│  评估模块    │────▶│  执行模块    │    │
│  │              │     │              │     │              │    │
│  │ • 多协议扫描 │     │ • 利润计算   │     │ • 闪电贷     │    │
│  │ • 事件监听   │     │ • 优先级排序 │     │ • Flashbots  │    │
│  │ • 健康因子   │     │ • 风险评估   │     │ • 直接提交   │    │
│  └──────────────┘     └──────────────┘     └──────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 关键挑战

1. **竞争激烈**: 清算机会抢夺
2. **Gas 成本**: 高 Gas 环境下利润压缩
3. **价格波动**: 执行期间价格变化
4. **流动性**: DEX 滑点影响利润
