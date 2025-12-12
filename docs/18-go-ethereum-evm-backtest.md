# Go-Ethereum EVM 深度解析：回测框架

## 第一百一十三节：历史数据采集

### 回测架构概览

```
回测系统架构：
┌─────────────────────────────────────────────────────────────────────┐
│                         回测系统                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │  数据采集    │────▶│  数据存储    │────▶│  回测引擎    │        │
│  │              │     │              │     │              │        │
│  │ • 区块数据   │     │ • TimescaleDB│     │ • 事件驱动   │        │
│  │ • 交易数据   │     │ • ClickHouse │     │ • 策略执行   │        │
│  │ • 事件日志   │     │ • Parquet    │     │ • 滑点模拟   │        │
│  │ • 池子状态   │     │              │     │ • Gas 模拟   │        │
│  └──────────────┘     └──────────────┘     └──────────────┘        │
│         │                    │                    │                 │
│         └────────────────────┼────────────────────┘                 │
│                              │                                      │
│                              ▼                                      │
│                    ┌──────────────────┐                            │
│                    │    分析报告      │                            │
│                    │ • PnL 曲线       │                            │
│                    │ • 风险指标       │                            │
│                    │ • 性能分析       │                            │
│                    └──────────────────┘                            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 历史数据采集器

```go
package backtest

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "math/big"
    "sync"
    "time"

    "github.com/ethereum/go-ethereum"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
)

// HistoricalDataCollector 历史数据采集器
type HistoricalDataCollector struct {
    client    *ethclient.Client
    storage   DataStorage
    config    *CollectorConfig
}

// CollectorConfig 采集器配置
type CollectorConfig struct {
    StartBlock      uint64
    EndBlock        uint64
    BatchSize       uint64
    Concurrency     int
    PoolAddresses   []common.Address
    TokenAddresses  []common.Address
}

// DataStorage 数据存储接口
type DataStorage interface {
    SaveBlock(block *BlockData) error
    SaveSwapEvent(event *SwapEvent) error
    SaveSyncEvent(event *SyncEvent) error
    SavePoolState(state *PoolState) error
    GetBlock(number uint64) (*BlockData, error)
    GetSwapEvents(pool common.Address, startBlock, endBlock uint64) ([]*SwapEvent, error)
}

// BlockData 区块数据
type BlockData struct {
    Number       uint64
    Hash         common.Hash
    ParentHash   common.Hash
    Timestamp    uint64
    GasUsed      uint64
    GasLimit     uint64
    BaseFee      *big.Int
    Transactions int
}

// SwapEvent Swap 事件
type SwapEvent struct {
    BlockNumber  uint64
    TxHash       common.Hash
    TxIndex      uint
    LogIndex     uint
    Pool         common.Address
    Sender       common.Address
    Recipient    common.Address
    Amount0In    *big.Int
    Amount1In    *big.Int
    Amount0Out   *big.Int
    Amount1Out   *big.Int
    Timestamp    uint64
}

// SyncEvent Sync 事件 (储备更新)
type SyncEvent struct {
    BlockNumber uint64
    TxHash      common.Hash
    LogIndex    uint
    Pool        common.Address
    Reserve0    *big.Int
    Reserve1    *big.Int
    Timestamp   uint64
}

// PoolState 池子状态快照
type PoolState struct {
    BlockNumber uint64
    Pool        common.Address
    Token0      common.Address
    Token1      common.Address
    Reserve0    *big.Int
    Reserve1    *big.Int
    Price0      *big.Float
    Price1      *big.Float
    Liquidity   *big.Int
    Fee         uint64
    Timestamp   uint64
}

// NewHistoricalDataCollector 创建采集器
func NewHistoricalDataCollector(
    client *ethclient.Client,
    storage DataStorage,
    config *CollectorConfig,
) *HistoricalDataCollector {
    return &HistoricalDataCollector{
        client:  client,
        storage: storage,
        config:  config,
    }
}

// Collect 开始采集
func (c *HistoricalDataCollector) Collect(ctx context.Context) error {
    fmt.Printf("开始采集区块 %d 到 %d\n", c.config.StartBlock, c.config.EndBlock)

    // 创建工作通道
    blockChan := make(chan uint64, c.config.Concurrency*2)
    errChan := make(chan error, 1)

    // 启动工作协程
    var wg sync.WaitGroup
    for i := 0; i < c.config.Concurrency; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for blockNum := range blockChan {
                if err := c.collectBlock(ctx, blockNum); err != nil {
                    select {
                    case errChan <- err:
                    default:
                    }
                    return
                }
            }
        }()
    }

    // 发送任务
    go func() {
        for blockNum := c.config.StartBlock; blockNum <= c.config.EndBlock; blockNum++ {
            select {
            case <-ctx.Done():
                close(blockChan)
                return
            case blockChan <- blockNum:
            }
        }
        close(blockChan)
    }()

    // 等待完成
    wg.Wait()

    select {
    case err := <-errChan:
        return err
    default:
        return nil
    }
}

// collectBlock 采集单个区块
func (c *HistoricalDataCollector) collectBlock(ctx context.Context, blockNum uint64) error {
    // 获取区块
    block, err := c.client.BlockByNumber(ctx, big.NewInt(int64(blockNum)))
    if err != nil {
        return fmt.Errorf("获取区块 %d 失败: %w", blockNum, err)
    }

    // 保存区块数据
    blockData := &BlockData{
        Number:       block.NumberU64(),
        Hash:         block.Hash(),
        ParentHash:   block.ParentHash(),
        Timestamp:    block.Time(),
        GasUsed:      block.GasUsed(),
        GasLimit:     block.GasLimit(),
        BaseFee:      block.BaseFee(),
        Transactions: len(block.Transactions()),
    }
    if err := c.storage.SaveBlock(blockData); err != nil {
        return err
    }

    // 获取事件日志
    if err := c.collectEvents(ctx, blockNum, block.Time()); err != nil {
        return err
    }

    if blockNum%1000 == 0 {
        fmt.Printf("已采集区块 %d\n", blockNum)
    }

    return nil
}

// collectEvents 采集事件
func (c *HistoricalDataCollector) collectEvents(ctx context.Context, blockNum, timestamp uint64) error {
    // Swap 事件签名
    swapV2Topic := common.HexToHash("0xd78ad95fa46c994b6551d0da85fc275fe613ce37657fb8d5e3d130840159d822")
    // Sync 事件签名
    syncTopic := common.HexToHash("0x1c411e9a96e071241c2f21f7726b17ae89e3cab4c78be50e062b03a9fffbbad1")

    query := ethereum.FilterQuery{
        FromBlock: big.NewInt(int64(blockNum)),
        ToBlock:   big.NewInt(int64(blockNum)),
        Addresses: c.config.PoolAddresses,
        Topics:    [][]common.Hash{{swapV2Topic, syncTopic}},
    }

    logs, err := c.client.FilterLogs(ctx, query)
    if err != nil {
        return err
    }

    for _, log := range logs {
        switch log.Topics[0] {
        case swapV2Topic:
            event := c.parseSwapEvent(log, timestamp)
            if err := c.storage.SaveSwapEvent(event); err != nil {
                return err
            }
        case syncTopic:
            event := c.parseSyncEvent(log, timestamp)
            if err := c.storage.SaveSyncEvent(event); err != nil {
                return err
            }
        }
    }

    return nil
}

// parseSwapEvent 解析 Swap 事件
func (c *HistoricalDataCollector) parseSwapEvent(log types.Log, timestamp uint64) *SwapEvent {
    event := &SwapEvent{
        BlockNumber: log.BlockNumber,
        TxHash:      log.TxHash,
        TxIndex:     log.TxIndex,
        LogIndex:    log.Index,
        Pool:        log.Address,
        Timestamp:   timestamp,
    }

    if len(log.Topics) >= 3 {
        event.Sender = common.BytesToAddress(log.Topics[1].Bytes())
        event.Recipient = common.BytesToAddress(log.Topics[2].Bytes())
    }

    if len(log.Data) >= 128 {
        event.Amount0In = new(big.Int).SetBytes(log.Data[0:32])
        event.Amount1In = new(big.Int).SetBytes(log.Data[32:64])
        event.Amount0Out = new(big.Int).SetBytes(log.Data[64:96])
        event.Amount1Out = new(big.Int).SetBytes(log.Data[96:128])
    }

    return event
}

// parseSyncEvent 解析 Sync 事件
func (c *HistoricalDataCollector) parseSyncEvent(log types.Log, timestamp uint64) *SyncEvent {
    event := &SyncEvent{
        BlockNumber: log.BlockNumber,
        TxHash:      log.TxHash,
        LogIndex:    log.Index,
        Pool:        log.Address,
        Timestamp:   timestamp,
    }

    if len(log.Data) >= 64 {
        event.Reserve0 = new(big.Int).SetBytes(log.Data[0:32])
        event.Reserve1 = new(big.Int).SetBytes(log.Data[32:64])
    }

    return event
}

// PostgresStorage PostgreSQL 存储实现
type PostgresStorage struct {
    db *sql.DB
}

// NewPostgresStorage 创建 PostgreSQL 存储
func NewPostgresStorage(connStr string) (*PostgresStorage, error) {
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, err
    }

    // 创建表
    if err := createTables(db); err != nil {
        return nil, err
    }

    return &PostgresStorage{db: db}, nil
}

// createTables 创建表
func createTables(db *sql.DB) error {
    queries := []string{
        `CREATE TABLE IF NOT EXISTS blocks (
            number BIGINT PRIMARY KEY,
            hash BYTEA,
            parent_hash BYTEA,
            timestamp BIGINT,
            gas_used BIGINT,
            gas_limit BIGINT,
            base_fee NUMERIC,
            transactions INT
        )`,
        `CREATE TABLE IF NOT EXISTS swap_events (
            id SERIAL PRIMARY KEY,
            block_number BIGINT,
            tx_hash BYTEA,
            tx_index INT,
            log_index INT,
            pool BYTEA,
            sender BYTEA,
            recipient BYTEA,
            amount0_in NUMERIC,
            amount1_in NUMERIC,
            amount0_out NUMERIC,
            amount1_out NUMERIC,
            timestamp BIGINT
        )`,
        `CREATE TABLE IF NOT EXISTS sync_events (
            id SERIAL PRIMARY KEY,
            block_number BIGINT,
            tx_hash BYTEA,
            log_index INT,
            pool BYTEA,
            reserve0 NUMERIC,
            reserve1 NUMERIC,
            timestamp BIGINT
        )`,
        `CREATE INDEX IF NOT EXISTS idx_swap_pool_block ON swap_events(pool, block_number)`,
        `CREATE INDEX IF NOT EXISTS idx_sync_pool_block ON sync_events(pool, block_number)`,
    }

    for _, query := range queries {
        if _, err := db.Exec(query); err != nil {
            return err
        }
    }

    return nil
}

// SaveBlock 保存区块
func (s *PostgresStorage) SaveBlock(block *BlockData) error {
    _, err := s.db.Exec(
        `INSERT INTO blocks (number, hash, parent_hash, timestamp, gas_used, gas_limit, base_fee, transactions)
         VALUES ($1, $2, $3, $4, $5, $6, $7, $8)
         ON CONFLICT (number) DO NOTHING`,
        block.Number,
        block.Hash.Bytes(),
        block.ParentHash.Bytes(),
        block.Timestamp,
        block.GasUsed,
        block.GasLimit,
        block.BaseFee.String(),
        block.Transactions,
    )
    return err
}

// SaveSwapEvent 保存 Swap 事件
func (s *PostgresStorage) SaveSwapEvent(event *SwapEvent) error {
    _, err := s.db.Exec(
        `INSERT INTO swap_events
         (block_number, tx_hash, tx_index, log_index, pool, sender, recipient,
          amount0_in, amount1_in, amount0_out, amount1_out, timestamp)
         VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12)`,
        event.BlockNumber,
        event.TxHash.Bytes(),
        event.TxIndex,
        event.LogIndex,
        event.Pool.Bytes(),
        event.Sender.Bytes(),
        event.Recipient.Bytes(),
        event.Amount0In.String(),
        event.Amount1In.String(),
        event.Amount0Out.String(),
        event.Amount1Out.String(),
        event.Timestamp,
    )
    return err
}

// SaveSyncEvent 保存 Sync 事件
func (s *PostgresStorage) SaveSyncEvent(event *SyncEvent) error {
    _, err := s.db.Exec(
        `INSERT INTO sync_events (block_number, tx_hash, log_index, pool, reserve0, reserve1, timestamp)
         VALUES ($1, $2, $3, $4, $5, $6, $7)`,
        event.BlockNumber,
        event.TxHash.Bytes(),
        event.LogIndex,
        event.Pool.Bytes(),
        event.Reserve0.String(),
        event.Reserve1.String(),
        event.Timestamp,
    )
    return err
}

// SavePoolState 保存池子状态
func (s *PostgresStorage) SavePoolState(state *PoolState) error {
    // 实现省略
    return nil
}

// GetBlock 获取区块
func (s *PostgresStorage) GetBlock(number uint64) (*BlockData, error) {
    // 实现省略
    return nil, nil
}

// GetSwapEvents 获取 Swap 事件
func (s *PostgresStorage) GetSwapEvents(pool common.Address, startBlock, endBlock uint64) ([]*SwapEvent, error) {
    rows, err := s.db.Query(
        `SELECT block_number, tx_hash, tx_index, log_index, pool, sender, recipient,
                amount0_in, amount1_in, amount0_out, amount1_out, timestamp
         FROM swap_events
         WHERE pool = $1 AND block_number >= $2 AND block_number <= $3
         ORDER BY block_number, log_index`,
        pool.Bytes(),
        startBlock,
        endBlock,
    )
    if err != nil {
        return nil, err
    }
    defer rows.Close()

    var events []*SwapEvent
    for rows.Next() {
        var e SwapEvent
        var txHash, poolBytes, sender, recipient []byte
        var amount0In, amount1In, amount0Out, amount1Out string

        err := rows.Scan(
            &e.BlockNumber, &txHash, &e.TxIndex, &e.LogIndex,
            &poolBytes, &sender, &recipient,
            &amount0In, &amount1In, &amount0Out, &amount1Out, &e.Timestamp,
        )
        if err != nil {
            return nil, err
        }

        e.TxHash = common.BytesToHash(txHash)
        e.Pool = common.BytesToAddress(poolBytes)
        e.Sender = common.BytesToAddress(sender)
        e.Recipient = common.BytesToAddress(recipient)
        e.Amount0In, _ = new(big.Int).SetString(amount0In, 10)
        e.Amount1In, _ = new(big.Int).SetString(amount1In, 10)
        e.Amount0Out, _ = new(big.Int).SetString(amount0Out, 10)
        e.Amount1Out, _ = new(big.Int).SetString(amount1Out, 10)

        events = append(events, &e)
    }

    return events, nil
}
```

---

## 第一百一十四节：回测引擎

### 事件驱动回测引擎

```go
package backtest

import (
    "context"
    "fmt"
    "math/big"
    "sort"
    "time"

    "github.com/ethereum/go-ethereum/common"
)

// BacktestEngine 回测引擎
type BacktestEngine struct {
    storage     DataStorage
    strategy    Strategy
    simulator   *TradeSimulator
    config      *EngineConfig
    state       *EngineState
    results     *BacktestResults
}

// EngineConfig 引擎配置
type EngineConfig struct {
    StartBlock      uint64
    EndBlock        uint64
    InitialCapital  *big.Int
    GasPrice        *big.Int
    SlippageModel   SlippageModel
    FeeModel        FeeModel
}

// EngineState 引擎状态
type EngineState struct {
    CurrentBlock    uint64
    Timestamp       uint64
    PoolStates      map[common.Address]*PoolState
    Balances        map[common.Address]*big.Int
    Positions       []*Position
    PendingOrders   []*Order
}

// Position 头寸
type Position struct {
    Token       common.Address
    Amount      *big.Int
    EntryPrice  *big.Float
    EntryBlock  uint64
    EntryTime   uint64
}

// Order 订单
type Order struct {
    ID          string
    Type        OrderType
    Pool        common.Address
    TokenIn     common.Address
    TokenOut    common.Address
    AmountIn    *big.Int
    MinAmountOut *big.Int
    Deadline    uint64
    Status      OrderStatus
}

// OrderType 订单类型
type OrderType int

const (
    OrderTypeMarket OrderType = iota
    OrderTypeLimit
)

// OrderStatus 订单状态
type OrderStatus int

const (
    OrderStatusPending OrderStatus = iota
    OrderStatusFilled
    OrderStatusCanceled
    OrderStatusExpired
)

// Strategy 策略接口
type Strategy interface {
    // 初始化
    Initialize(state *EngineState) error
    // 处理新区块
    OnBlock(block *BlockData, state *EngineState) ([]*Order, error)
    // 处理 Swap 事件
    OnSwap(event *SwapEvent, state *EngineState) ([]*Order, error)
    // 处理 Sync 事件
    OnSync(event *SyncEvent, state *EngineState) ([]*Order, error)
    // 获取策略名称
    Name() string
}

// SlippageModel 滑点模型
type SlippageModel interface {
    CalculateSlippage(pool *PoolState, amountIn *big.Int, isBuy bool) float64
}

// FeeModel 费用模型
type FeeModel interface {
    CalculateFee(pool *PoolState, amount *big.Int) *big.Int
}

// BacktestResults 回测结果
type BacktestResults struct {
    StartBlock      uint64
    EndBlock        uint64
    StartTime       time.Time
    EndTime         time.Time
    InitialCapital  *big.Int
    FinalCapital    *big.Int
    TotalPnL        *big.Int
    TotalTrades     int
    WinningTrades   int
    LosingTrades    int
    TotalGasSpent   *big.Int
    MaxDrawdown     float64
    SharpeRatio     float64
    PnLHistory      []*PnLPoint
    TradeHistory    []*TradeRecord
}

// PnLPoint PnL 数据点
type PnLPoint struct {
    Block       uint64
    Timestamp   uint64
    PnL         *big.Int
    Capital     *big.Int
}

// TradeRecord 交易记录
type TradeRecord struct {
    Block       uint64
    Timestamp   uint64
    Pool        common.Address
    TokenIn     common.Address
    TokenOut    common.Address
    AmountIn    *big.Int
    AmountOut   *big.Int
    GasUsed     uint64
    GasCost     *big.Int
    Slippage    float64
    PnL         *big.Int
}

// NewBacktestEngine 创建回测引擎
func NewBacktestEngine(
    storage DataStorage,
    strategy Strategy,
    config *EngineConfig,
) *BacktestEngine {
    return &BacktestEngine{
        storage:  storage,
        strategy: strategy,
        config:   config,
        simulator: NewTradeSimulator(config.SlippageModel, config.FeeModel),
        state: &EngineState{
            PoolStates: make(map[common.Address]*PoolState),
            Balances:   make(map[common.Address]*big.Int),
        },
        results: &BacktestResults{
            InitialCapital: config.InitialCapital,
            TotalPnL:       big.NewInt(0),
            TotalGasSpent:  big.NewInt(0),
            PnLHistory:     make([]*PnLPoint, 0),
            TradeHistory:   make([]*TradeRecord, 0),
        },
    }
}

// Run 运行回测
func (e *BacktestEngine) Run(ctx context.Context) (*BacktestResults, error) {
    e.results.StartBlock = e.config.StartBlock
    e.results.EndBlock = e.config.EndBlock
    e.results.StartTime = time.Now()

    // 初始化策略
    if err := e.strategy.Initialize(e.state); err != nil {
        return nil, fmt.Errorf("策略初始化失败: %w", err)
    }

    // 设置初始资金
    e.state.Balances[common.Address{}] = new(big.Int).Set(e.config.InitialCapital) // ETH

    // 加载所有事件
    events, err := e.loadEvents(ctx)
    if err != nil {
        return nil, err
    }

    // 按区块和日志索引排序
    sort.Slice(events, func(i, j int) bool {
        if events[i].GetBlock() != events[j].GetBlock() {
            return events[i].GetBlock() < events[j].GetBlock()
        }
        return events[i].GetLogIndex() < events[j].GetLogIndex()
    })

    // 处理事件
    currentBlock := e.config.StartBlock
    for _, event := range events {
        // 检查是否新区块
        if event.GetBlock() > currentBlock {
            // 处理新区块
            blockData, _ := e.storage.GetBlock(event.GetBlock())
            if blockData != nil {
                e.processBlock(blockData)
            }
            currentBlock = event.GetBlock()
        }

        // 处理事件
        e.processEvent(event)
    }

    // 计算最终结果
    e.calculateFinalResults()

    e.results.EndTime = time.Now()
    return e.results, nil
}

// Event 事件接口
type Event interface {
    GetBlock() uint64
    GetLogIndex() uint
    GetTimestamp() uint64
}

// 实现 Event 接口
func (e *SwapEvent) GetBlock() uint64     { return e.BlockNumber }
func (e *SwapEvent) GetLogIndex() uint    { return e.LogIndex }
func (e *SwapEvent) GetTimestamp() uint64 { return e.Timestamp }

func (e *SyncEvent) GetBlock() uint64     { return e.BlockNumber }
func (e *SyncEvent) GetLogIndex() uint    { return e.LogIndex }
func (e *SyncEvent) GetTimestamp() uint64 { return e.Timestamp }

// loadEvents 加载事件
func (e *BacktestEngine) loadEvents(ctx context.Context) ([]Event, error) {
    var allEvents []Event

    // 获取所有池子的 Swap 事件
    for pool := range e.state.PoolStates {
        swapEvents, err := e.storage.GetSwapEvents(pool, e.config.StartBlock, e.config.EndBlock)
        if err != nil {
            return nil, err
        }
        for _, ev := range swapEvents {
            allEvents = append(allEvents, ev)
        }
    }

    return allEvents, nil
}

// processBlock 处理区块
func (e *BacktestEngine) processBlock(block *BlockData) {
    e.state.CurrentBlock = block.Number
    e.state.Timestamp = block.Timestamp

    // 调用策略
    orders, err := e.strategy.OnBlock(block, e.state)
    if err != nil {
        return
    }

    // 执行订单
    for _, order := range orders {
        e.executeOrder(order)
    }

    // 记录 PnL
    capital := e.calculateTotalCapital()
    pnl := new(big.Int).Sub(capital, e.config.InitialCapital)
    e.results.PnLHistory = append(e.results.PnLHistory, &PnLPoint{
        Block:     block.Number,
        Timestamp: block.Timestamp,
        PnL:       pnl,
        Capital:   capital,
    })
}

// processEvent 处理事件
func (e *BacktestEngine) processEvent(event Event) {
    var orders []*Order
    var err error

    switch ev := event.(type) {
    case *SwapEvent:
        // 更新池子状态
        if state, exists := e.state.PoolStates[ev.Pool]; exists {
            // 简化：从 swap 推断新储备
            e.updatePoolFromSwap(state, ev)
        }
        orders, err = e.strategy.OnSwap(ev, e.state)

    case *SyncEvent:
        // 更新池子状态
        if state, exists := e.state.PoolStates[ev.Pool]; exists {
            state.Reserve0 = ev.Reserve0
            state.Reserve1 = ev.Reserve1
        } else {
            e.state.PoolStates[ev.Pool] = &PoolState{
                Pool:     ev.Pool,
                Reserve0: ev.Reserve0,
                Reserve1: ev.Reserve1,
            }
        }
        orders, err = e.strategy.OnSync(ev, e.state)
    }

    if err != nil {
        return
    }

    for _, order := range orders {
        e.executeOrder(order)
    }
}

// updatePoolFromSwap 从 swap 更新池子
func (e *BacktestEngine) updatePoolFromSwap(state *PoolState, swap *SwapEvent) {
    // 简化计算
    state.Reserve0.Add(state.Reserve0, swap.Amount0In)
    state.Reserve0.Sub(state.Reserve0, swap.Amount0Out)
    state.Reserve1.Add(state.Reserve1, swap.Amount1In)
    state.Reserve1.Sub(state.Reserve1, swap.Amount1Out)
}

// executeOrder 执行订单
func (e *BacktestEngine) executeOrder(order *Order) {
    // 获取池子状态
    pool := e.state.PoolStates[order.Pool]
    if pool == nil {
        order.Status = OrderStatusCanceled
        return
    }

    // 模拟交易
    result := e.simulator.SimulateTrade(pool, order.AmountIn, order.TokenIn == pool.Token0)

    // 检查滑点
    if result.AmountOut.Cmp(order.MinAmountOut) < 0 {
        order.Status = OrderStatusCanceled
        return
    }

    // 检查余额
    balance := e.state.Balances[order.TokenIn]
    if balance == nil || balance.Cmp(order.AmountIn) < 0 {
        order.Status = OrderStatusCanceled
        return
    }

    // 执行交易
    e.state.Balances[order.TokenIn].Sub(e.state.Balances[order.TokenIn], order.AmountIn)
    if e.state.Balances[order.TokenOut] == nil {
        e.state.Balances[order.TokenOut] = big.NewInt(0)
    }
    e.state.Balances[order.TokenOut].Add(e.state.Balances[order.TokenOut], result.AmountOut)

    // 扣除 Gas
    gasCost := new(big.Int).Mul(e.config.GasPrice, big.NewInt(150000))
    e.state.Balances[common.Address{}].Sub(e.state.Balances[common.Address{}], gasCost)
    e.results.TotalGasSpent.Add(e.results.TotalGasSpent, gasCost)

    // 记录交易
    e.results.TradeHistory = append(e.results.TradeHistory, &TradeRecord{
        Block:     e.state.CurrentBlock,
        Timestamp: e.state.Timestamp,
        Pool:      order.Pool,
        TokenIn:   order.TokenIn,
        TokenOut:  order.TokenOut,
        AmountIn:  order.AmountIn,
        AmountOut: result.AmountOut,
        GasUsed:   150000,
        GasCost:   gasCost,
        Slippage:  result.Slippage,
    })

    e.results.TotalTrades++
    order.Status = OrderStatusFilled
}

// calculateTotalCapital 计算总资本
func (e *BacktestEngine) calculateTotalCapital() *big.Int {
    total := new(big.Int).Set(e.state.Balances[common.Address{}])

    // 加上其他代币价值 (简化：假设都有 ETH 交易对)
    for token, balance := range e.state.Balances {
        if token == (common.Address{}) {
            continue
        }
        // 简化：使用池子价格
        // 实际需要查询价格
        total.Add(total, balance)
    }

    return total
}

// calculateFinalResults 计算最终结果
func (e *BacktestEngine) calculateFinalResults() {
    e.results.FinalCapital = e.calculateTotalCapital()
    e.results.TotalPnL = new(big.Int).Sub(e.results.FinalCapital, e.config.InitialCapital)

    // 计算胜率
    for _, trade := range e.results.TradeHistory {
        if trade.PnL != nil && trade.PnL.Sign() > 0 {
            e.results.WinningTrades++
        } else {
            e.results.LosingTrades++
        }
    }

    // 计算最大回撤
    e.results.MaxDrawdown = e.calculateMaxDrawdown()

    // 计算夏普比率
    e.results.SharpeRatio = e.calculateSharpeRatio()
}

// calculateMaxDrawdown 计算最大回撤
func (e *BacktestEngine) calculateMaxDrawdown() float64 {
    if len(e.results.PnLHistory) == 0 {
        return 0
    }

    peak := new(big.Float).SetInt(e.results.PnLHistory[0].Capital)
    maxDrawdown := 0.0

    for _, point := range e.results.PnLHistory {
        capital := new(big.Float).SetInt(point.Capital)

        if capital.Cmp(peak) > 0 {
            peak = capital
        }

        drawdown := new(big.Float).Sub(peak, capital)
        drawdown.Quo(drawdown, peak)
        dd, _ := drawdown.Float64()

        if dd > maxDrawdown {
            maxDrawdown = dd
        }
    }

    return maxDrawdown
}

// calculateSharpeRatio 计算夏普比率
func (e *BacktestEngine) calculateSharpeRatio() float64 {
    if len(e.results.PnLHistory) < 2 {
        return 0
    }

    // 计算收益率
    var returns []float64
    for i := 1; i < len(e.results.PnLHistory); i++ {
        prev := new(big.Float).SetInt(e.results.PnLHistory[i-1].Capital)
        curr := new(big.Float).SetInt(e.results.PnLHistory[i].Capital)

        ret := new(big.Float).Sub(curr, prev)
        ret.Quo(ret, prev)
        r, _ := ret.Float64()
        returns = append(returns, r)
    }

    // 计算均值和标准差
    var sum, sumSq float64
    for _, r := range returns {
        sum += r
        sumSq += r * r
    }

    n := float64(len(returns))
    mean := sum / n
    variance := sumSq/n - mean*mean

    if variance <= 0 {
        return 0
    }

    stdDev := variance // sqrt 省略

    // 假设无风险利率为 0
    return mean / stdDev
}

// TradeSimulator 交易模拟器
type TradeSimulator struct {
    slippageModel SlippageModel
    feeModel      FeeModel
}

// TradeResult 交易结果
type TradeResult struct {
    AmountOut   *big.Int
    Fee         *big.Int
    Slippage    float64
    PriceImpact float64
}

// NewTradeSimulator 创建模拟器
func NewTradeSimulator(slippage SlippageModel, fee FeeModel) *TradeSimulator {
    return &TradeSimulator{
        slippageModel: slippage,
        feeModel:      fee,
    }
}

// SimulateTrade 模拟交易
func (s *TradeSimulator) SimulateTrade(pool *PoolState, amountIn *big.Int, zeroForOne bool) *TradeResult {
    var reserveIn, reserveOut *big.Int
    if zeroForOne {
        reserveIn = pool.Reserve0
        reserveOut = pool.Reserve1
    } else {
        reserveIn = pool.Reserve1
        reserveOut = pool.Reserve0
    }

    // 计算费用
    fee := s.feeModel.CalculateFee(pool, amountIn)
    amountInAfterFee := new(big.Int).Sub(amountIn, fee)

    // 计算输出 (恒定乘积)
    numerator := new(big.Int).Mul(amountInAfterFee, reserveOut)
    denominator := new(big.Int).Add(reserveIn, amountInAfterFee)
    amountOut := new(big.Int).Div(numerator, denominator)

    // 计算滑点
    slippage := s.slippageModel.CalculateSlippage(pool, amountIn, zeroForOne)

    // 应用滑点
    slippageAmount := new(big.Float).Mul(
        new(big.Float).SetInt(amountOut),
        big.NewFloat(slippage),
    )
    slippageInt, _ := slippageAmount.Int(nil)
    amountOut.Sub(amountOut, slippageInt)

    return &TradeResult{
        AmountOut: amountOut,
        Fee:       fee,
        Slippage:  slippage,
    }
}

// DefaultSlippageModel 默认滑点模型
type DefaultSlippageModel struct{}

// CalculateSlippage 计算滑点
func (m *DefaultSlippageModel) CalculateSlippage(pool *PoolState, amountIn *big.Int, isBuy bool) float64 {
    // 基于交易规模的滑点
    var reserve *big.Int
    if isBuy {
        reserve = pool.Reserve0
    } else {
        reserve = pool.Reserve1
    }

    ratio := new(big.Float).Quo(
        new(big.Float).SetInt(amountIn),
        new(big.Float).SetInt(reserve),
    )

    r, _ := ratio.Float64()

    // 滑点 = 交易规模 / 储备 * 系数
    return r * 0.5 // 50% 的价格影响作为滑点
}

// DefaultFeeModel 默认费用模型
type DefaultFeeModel struct {
    FeeBps uint64 // 费率 (基点)
}

// CalculateFee 计算费用
func (m *DefaultFeeModel) CalculateFee(pool *PoolState, amount *big.Int) *big.Int {
    fee := new(big.Int).Mul(amount, big.NewInt(int64(m.FeeBps)))
    return fee.Div(fee, big.NewInt(10000))
}
```

---

## 第一百一十五节：策略示例与分析

### 三角套利回测策略

```go
package backtest

import (
    "math/big"

    "github.com/ethereum/go-ethereum/common"
)

// TriangularArbitrageStrategy 三角套利策略
type TriangularArbitrageStrategy struct {
    pools           []common.Address
    minProfitBps    uint64
    maxTradeAmount  *big.Int
}

// NewTriangularArbitrageStrategy 创建策略
func NewTriangularArbitrageStrategy(
    pools []common.Address,
    minProfitBps uint64,
    maxTradeAmount *big.Int,
) *TriangularArbitrageStrategy {
    return &TriangularArbitrageStrategy{
        pools:          pools,
        minProfitBps:   minProfitBps,
        maxTradeAmount: maxTradeAmount,
    }
}

// Name 策略名称
func (s *TriangularArbitrageStrategy) Name() string {
    return "TriangularArbitrage"
}

// Initialize 初始化
func (s *TriangularArbitrageStrategy) Initialize(state *EngineState) error {
    // 初始化池子状态
    for _, pool := range s.pools {
        if _, exists := state.PoolStates[pool]; !exists {
            state.PoolStates[pool] = &PoolState{
                Pool:     pool,
                Reserve0: big.NewInt(0),
                Reserve1: big.NewInt(0),
            }
        }
    }
    return nil
}

// OnBlock 处理区块
func (s *TriangularArbitrageStrategy) OnBlock(block *BlockData, state *EngineState) ([]*Order, error) {
    // 每个区块检查套利机会
    return s.checkArbitrageOpportunities(state)
}

// OnSwap 处理 Swap 事件
func (s *TriangularArbitrageStrategy) OnSwap(event *SwapEvent, state *EngineState) ([]*Order, error) {
    // Swap 后立即检查机会
    return s.checkArbitrageOpportunities(state)
}

// OnSync 处理 Sync 事件
func (s *TriangularArbitrageStrategy) OnSync(event *SyncEvent, state *EngineState) ([]*Order, error) {
    // Sync 后检查机会
    return s.checkArbitrageOpportunities(state)
}

// checkArbitrageOpportunities 检查套利机会
func (s *TriangularArbitrageStrategy) checkArbitrageOpportunities(state *EngineState) ([]*Order, error) {
    if len(s.pools) < 3 {
        return nil, nil
    }

    // 获取三个池子
    pool1 := state.PoolStates[s.pools[0]]
    pool2 := state.PoolStates[s.pools[1]]
    pool3 := state.PoolStates[s.pools[2]]

    if pool1 == nil || pool2 == nil || pool3 == nil {
        return nil, nil
    }

    // 计算三角套利路径
    // A -> B -> C -> A
    // 检查是否有利润

    // 简化：假设 pool1: A/B, pool2: B/C, pool3: C/A
    amountIn := s.calculateOptimalAmount(pool1, pool2, pool3)
    if amountIn.Sign() <= 0 {
        return nil, nil
    }

    profit := s.calculateProfit(pool1, pool2, pool3, amountIn)

    // 检查利润是否满足阈值
    profitBps := new(big.Int).Mul(profit, big.NewInt(10000))
    profitBps.Div(profitBps, amountIn)

    if profitBps.Uint64() < s.minProfitBps {
        return nil, nil
    }

    // 创建套利订单
    orders := []*Order{
        {
            Type:        OrderTypeMarket,
            Pool:        s.pools[0],
            TokenIn:     pool1.Token0,
            TokenOut:    pool1.Token1,
            AmountIn:    amountIn,
            MinAmountOut: big.NewInt(0), // 简化
        },
        // 后续订单依赖前一个结果
    }

    return orders, nil
}

// calculateOptimalAmount 计算最优交易量
func (s *TriangularArbitrageStrategy) calculateOptimalAmount(
    pool1, pool2, pool3 *PoolState,
) *big.Int {
    // 简化计算
    // 使用最小储备的 1%
    minReserve := pool1.Reserve0
    if pool2.Reserve0.Cmp(minReserve) < 0 {
        minReserve = pool2.Reserve0
    }
    if pool3.Reserve0.Cmp(minReserve) < 0 {
        minReserve = pool3.Reserve0
    }

    optimal := new(big.Int).Div(minReserve, big.NewInt(100))

    if s.maxTradeAmount != nil && optimal.Cmp(s.maxTradeAmount) > 0 {
        return s.maxTradeAmount
    }

    return optimal
}

// calculateProfit 计算利润
func (s *TriangularArbitrageStrategy) calculateProfit(
    pool1, pool2, pool3 *PoolState,
    amountIn *big.Int,
) *big.Int {
    // 模拟三次交换
    // A -> B
    amount1 := calculateSwapOutput(pool1.Reserve0, pool1.Reserve1, amountIn, 30)
    // B -> C
    amount2 := calculateSwapOutput(pool2.Reserve0, pool2.Reserve1, amount1, 30)
    // C -> A
    amountOut := calculateSwapOutput(pool3.Reserve0, pool3.Reserve1, amount2, 30)

    return new(big.Int).Sub(amountOut, amountIn)
}

// calculateSwapOutput 计算交换输出
func calculateSwapOutput(reserveIn, reserveOut, amountIn *big.Int, feeBps uint64) *big.Int {
    amountInWithFee := new(big.Int).Mul(amountIn, big.NewInt(int64(10000-feeBps)))
    numerator := new(big.Int).Mul(amountInWithFee, reserveOut)
    denominator := new(big.Int).Mul(reserveIn, big.NewInt(10000))
    denominator.Add(denominator, amountInWithFee)
    return new(big.Int).Div(numerator, denominator)
}

// BacktestReporter 回测报告生成器
type BacktestReporter struct{}

// GenerateReport 生成报告
func (r *BacktestReporter) GenerateReport(results *BacktestResults) string {
    report := fmt.Sprintf(`
=== 回测报告 ===

时间范围: 区块 %d - %d
运行时间: %v

== 资金概况 ==
初始资金: %s ETH
最终资金: %s ETH
总盈亏:   %s ETH
收益率:   %.2f%%

== 交易统计 ==
总交易数: %d
盈利交易: %d
亏损交易: %d
胜率:     %.2f%%

== 风险指标 ==
最大回撤: %.2f%%
夏普比率: %.2f
总 Gas 消耗: %s ETH

== 交易明细 ==
`,
        results.StartBlock, results.EndBlock,
        results.EndTime.Sub(results.StartTime),
        formatETH(results.InitialCapital),
        formatETH(results.FinalCapital),
        formatETH(results.TotalPnL),
        calculateReturn(results.InitialCapital, results.FinalCapital),
        results.TotalTrades,
        results.WinningTrades,
        results.LosingTrades,
        float64(results.WinningTrades)/float64(results.TotalTrades)*100,
        results.MaxDrawdown*100,
        results.SharpeRatio,
        formatETH(results.TotalGasSpent),
    )

    // 添加交易明细
    for i, trade := range results.TradeHistory {
        if i >= 20 { // 只显示前 20 条
            report += fmt.Sprintf("... 共 %d 条交易\n", len(results.TradeHistory))
            break
        }
        report += fmt.Sprintf(
            "  #%d 区块:%d 入:%s 出:%s 滑点:%.2f%%\n",
            i+1, trade.Block,
            formatAmount(trade.AmountIn),
            formatAmount(trade.AmountOut),
            trade.Slippage*100,
        )
    }

    return report
}

func formatETH(wei *big.Int) string {
    eth := new(big.Float).Quo(
        new(big.Float).SetInt(wei),
        big.NewFloat(1e18),
    )
    return fmt.Sprintf("%.4f", eth)
}

func formatAmount(amount *big.Int) string {
    return amount.String()
}

func calculateReturn(initial, final *big.Int) float64 {
    diff := new(big.Float).Sub(
        new(big.Float).SetInt(final),
        new(big.Float).SetInt(initial),
    )
    ret := new(big.Float).Quo(diff, new(big.Float).SetInt(initial))
    r, _ := ret.Float64()
    return r * 100
}
```

---

## 本文总结

本文深入探讨了套利策略回测框架，主要涵盖：

### 回测流程

| 阶段 | 组件 | 功能 |
|------|------|------|
| 数据采集 | HistoricalDataCollector | 区块、事件、状态 |
| 数据存储 | PostgresStorage | 高效查询、索引 |
| 回测引擎 | BacktestEngine | 事件驱动、状态管理 |
| 交易模拟 | TradeSimulator | 滑点、费用、价格影响 |
| 结果分析 | BacktestReporter | PnL、风险指标 |

### 核心指标

```
回测分析指标：
┌─────────────────────────────────────────────────────────────────┐
│                       回测指标体系                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  收益指标                    风险指标                           │
│  ┌────────────────┐         ┌────────────────┐                 │
│  │ • 总收益率     │         │ • 最大回撤     │                 │
│  │ • 年化收益     │         │ • 波动率       │                 │
│  │ • 平均单笔收益 │         │ • 夏普比率     │                 │
│  └────────────────┘         └────────────────┘                 │
│                                                                 │
│  交易指标                    成本指标                           │
│  ┌────────────────┐         ┌────────────────┐                 │
│  │ • 胜率         │         │ • 总 Gas 成本  │                 │
│  │ • 盈亏比       │         │ • 平均滑点     │                 │
│  │ • 交易频率     │         │ • 手续费总额   │                 │
│  └────────────────┘         └────────────────┘                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 最佳实践

1. **数据质量**: 确保历史数据完整准确
2. **真实模拟**: 包含滑点、Gas、MEV 影响
3. **样本外测试**: 避免过拟合
4. **参数敏感性**: 测试不同参数组合
