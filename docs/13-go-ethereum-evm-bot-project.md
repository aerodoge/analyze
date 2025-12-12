# Go-Ethereum EVM 深度解析：完整套利机器人实战项目

## 第九十三节：项目架构设计

### 整体架构

```
完整套利机器人架构：
┌─────────────────────────────────────────────────────────────────────┐
│                        Arbitrage Bot System                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     Data Layer                               │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │   │
│  │  │ Mempool  │  │  Block   │  │  Price   │  │   Pool   │    │   │
│  │  │ Monitor  │  │ Monitor  │  │  Oracle  │  │  State   │    │   │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘    │   │
│  │       │              │              │              │         │   │
│  │       └──────────────┴──────────────┴──────────────┘         │   │
│  │                           │                                   │   │
│  └───────────────────────────┼───────────────────────────────────┘   │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   Strategy Layer                             │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │   │
│  │  │  Triangular  │  │   Cross-DEX  │  │  Flashloan   │       │   │
│  │  │   Arbitrage  │  │   Arbitrage  │  │   Arbitrage  │       │   │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │   │
│  │         │                  │                  │               │   │
│  │         └──────────────────┼──────────────────┘               │   │
│  │                            │                                   │   │
│  └────────────────────────────┼───────────────────────────────────┘   │
│                               ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                  Execution Layer                             │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │   │
│  │  │   Bundle     │  │    Gas      │  │   MEV        │       │   │
│  │  │   Builder    │  │   Manager   │  │   Protect    │       │   │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │   │
│  │         │                  │                  │               │   │
│  │         └──────────────────┼──────────────────┘               │   │
│  │                            │                                   │   │
│  └────────────────────────────┼───────────────────────────────────┘   │
│                               ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                Infrastructure Layer                          │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │   │
│  │  │  Geth    │  │ Flashbots│  │  Logger  │  │  Metrics │    │   │
│  │  │  Client  │  │  Relay   │  │          │  │          │    │   │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 项目目录结构

```
arbitrage-bot/
├── cmd/
│   └── bot/
│       └── main.go           # 入口
├── internal/
│   ├── config/
│   │   └── config.go         # 配置管理
│   ├── chain/
│   │   ├── client.go         # 链客户端
│   │   ├── mempool.go        # 内存池监控
│   │   └── block.go          # 区块监控
│   ├── dex/
│   │   ├── factory.go        # DEX 工厂
│   │   ├── uniswapv2.go      # Uniswap V2
│   │   ├── uniswapv3.go      # Uniswap V3
│   │   └── sushiswap.go      # SushiSwap
│   ├── pool/
│   │   ├── manager.go        # 池管理器
│   │   ├── state.go          # 池状态
│   │   └── sync.go           # 状态同步
│   ├── strategy/
│   │   ├── interface.go      # 策略接口
│   │   ├── triangular.go     # 三角套利
│   │   ├── crossdex.go       # 跨 DEX 套利
│   │   └── flashloan.go      # 闪电贷套利
│   ├── execution/
│   │   ├── builder.go        # Bundle 构建
│   │   ├── gas.go            # Gas 管理
│   │   ├── flashbots.go      # Flashbots 提交
│   │   └── simulator.go      # 交易模拟
│   └── utils/
│       ├── math.go           # 数学工具
│       ├── encode.go         # ABI 编码
│       └── logger.go         # 日志
├── contracts/
│   └── Arbitrage.sol         # 套利合约
├── configs/
│   └── config.yaml           # 配置文件
├── scripts/
│   └── deploy.go             # 部署脚本
├── go.mod
└── go.sum
```

### 核心配置

```go
package config

import (
    "math/big"
    "os"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "gopkg.in/yaml.v3"
)

// Config 主配置
type Config struct {
    Chain      ChainConfig      `yaml:"chain"`
    DEXs       []DEXConfig      `yaml:"dexs"`
    Tokens     []TokenConfig    `yaml:"tokens"`
    Strategy   StrategyConfig   `yaml:"strategy"`
    Execution  ExecutionConfig  `yaml:"execution"`
    Flashbots  FlashbotsConfig  `yaml:"flashbots"`
    Monitoring MonitoringConfig `yaml:"monitoring"`
}

// ChainConfig 链配置
type ChainConfig struct {
    RPCURL        string `yaml:"rpc_url"`
    WSURL         string `yaml:"ws_url"`
    ChainID       int64  `yaml:"chain_id"`
    BlockTime     int    `yaml:"block_time"`
    Confirmations int    `yaml:"confirmations"`
}

// DEXConfig DEX 配置
type DEXConfig struct {
    Name          string `yaml:"name"`
    Type          string `yaml:"type"` // "uniswapv2", "uniswapv3", "curve"
    FactoryAddr   string `yaml:"factory_addr"`
    RouterAddr    string `yaml:"router_addr"`
    InitCodeHash  string `yaml:"init_code_hash"`
    Fee           int    `yaml:"fee"` // 基点
    Enabled       bool   `yaml:"enabled"`
}

// TokenConfig 代币配置
type TokenConfig struct {
    Symbol   string `yaml:"symbol"`
    Address  string `yaml:"address"`
    Decimals int    `yaml:"decimals"`
    IsBase   bool   `yaml:"is_base"` // 是否是基础代币（如 WETH）
}

// StrategyConfig 策略配置
type StrategyConfig struct {
    MinProfitUSD     float64       `yaml:"min_profit_usd"`
    MaxTradeUSD      float64       `yaml:"max_trade_usd"`
    MaxSlippage      float64       `yaml:"max_slippage"`       // 百分比
    MaxGasPrice      int64         `yaml:"max_gas_price"`      // Gwei
    EnableTriangular bool          `yaml:"enable_triangular"`
    EnableCrossDEX   bool          `yaml:"enable_cross_dex"`
    EnableFlashloan  bool          `yaml:"enable_flashloan"`
    ScanInterval     time.Duration `yaml:"scan_interval"`
}

// ExecutionConfig 执行配置
type ExecutionConfig struct {
    PrivateKey       string `yaml:"private_key"`
    ArbitrageContract string `yaml:"arbitrage_contract"`
    UseFlashbots     bool   `yaml:"use_flashbots"`
    SimulateFirst    bool   `yaml:"simulate_first"`
    MaxRetries       int    `yaml:"max_retries"`
    RetryDelay       time.Duration `yaml:"retry_delay"`
}

// FlashbotsConfig Flashbots 配置
type FlashbotsConfig struct {
    RelayURL     string `yaml:"relay_url"`
    SigningKey   string `yaml:"signing_key"`
    UseProtect   bool   `yaml:"use_protect"`
    RefundPercent int   `yaml:"refund_percent"`
}

// MonitoringConfig 监控配置
type MonitoringConfig struct {
    MetricsPort    int    `yaml:"metrics_port"`
    LogLevel       string `yaml:"log_level"`
    AlertWebhook   string `yaml:"alert_webhook"`
    ProfitReportInterval time.Duration `yaml:"profit_report_interval"`
}

// LoadConfig 加载配置
func LoadConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, err
    }

    var config Config
    if err := yaml.Unmarshal(data, &config); err != nil {
        return nil, err
    }

    return &config, nil
}

// GetDEXByName 根据名称获取 DEX 配置
func (c *Config) GetDEXByName(name string) *DEXConfig {
    for _, dex := range c.DEXs {
        if dex.Name == name && dex.Enabled {
            return &dex
        }
    }
    return nil
}

// GetTokenBySymbol 根据符号获取代币配置
func (c *Config) GetTokenBySymbol(symbol string) *TokenConfig {
    for _, token := range c.Tokens {
        if token.Symbol == symbol {
            return &token
        }
    }
    return nil
}

// GetBaseTokens 获取基础代币列表
func (c *Config) GetBaseTokens() []TokenConfig {
    var bases []TokenConfig
    for _, token := range c.Tokens {
        if token.IsBase {
            bases = append(bases, token)
        }
    }
    return bases
}
```

### 配置文件示例

```yaml
# configs/config.yaml
chain:
  rpc_url: "http://localhost:8545"
  ws_url: "ws://localhost:8546"
  chain_id: 1
  block_time: 12
  confirmations: 1

dexs:
  - name: "uniswap_v2"
    type: "uniswapv2"
    factory_addr: "0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f"
    router_addr: "0x7a250d5630B4cF539739dF2C5dAcb4c659F2488D"
    init_code_hash: "0x96e8ac4277198ff8b6f785478aa9a39f403cb768dd02cbee326c3e7da348845f"
    fee: 30
    enabled: true

  - name: "sushiswap"
    type: "uniswapv2"
    factory_addr: "0xC0AEe478e3658e2610c5F7A4A2E1777cE9e4f2Ac"
    router_addr: "0xd9e1cE17f2641f24aE83637ab66a2cca9C378B9F"
    init_code_hash: "0xe18a34eb0e04b04f7a0ac29a6e80748dca96319b42c54d679cb821dca90c6303"
    fee: 30
    enabled: true

  - name: "uniswap_v3"
    type: "uniswapv3"
    factory_addr: "0x1F98431c8aD98523631AE4a59f267346ea31F984"
    router_addr: "0xE592427A0AEce92De3Edee1F18E0157C05861564"
    init_code_hash: ""
    fee: 0  # V3 每个池子费率不同
    enabled: true

tokens:
  - symbol: "WETH"
    address: "0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"
    decimals: 18
    is_base: true

  - symbol: "USDC"
    address: "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
    decimals: 6
    is_base: true

  - symbol: "USDT"
    address: "0xdAC17F958D2ee523a2206206994597C13D831ec7"
    decimals: 6
    is_base: false

  - symbol: "DAI"
    address: "0x6B175474E89094C44Da98b954EescdeCB5cCC4"
    decimals: 18
    is_base: false

  - symbol: "WBTC"
    address: "0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599"
    decimals: 8
    is_base: false

strategy:
  min_profit_usd: 50.0
  max_trade_usd: 100000.0
  max_slippage: 1.0
  max_gas_price: 100
  enable_triangular: true
  enable_cross_dex: true
  enable_flashloan: true
  scan_interval: 100ms

execution:
  private_key: "${PRIVATE_KEY}"
  arbitrage_contract: "0x..."
  use_flashbots: true
  simulate_first: true
  max_retries: 3
  retry_delay: 500ms

flashbots:
  relay_url: "https://relay.flashbots.net"
  signing_key: "${FLASHBOTS_KEY}"
  use_protect: true
  refund_percent: 90

monitoring:
  metrics_port: 9090
  log_level: "info"
  alert_webhook: "${ALERT_WEBHOOK}"
  profit_report_interval: 1h
```

---

## 第九十四节：链上数据层实现

### 链客户端

```go
package chain

import (
    "context"
    "crypto/ecdsa"
    "fmt"
    "math/big"
    "sync"
    "time"

    "github.com/ethereum/go-ethereum"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/crypto"
    "github.com/ethereum/go-ethereum/ethclient"
)

// ChainClient 链客户端
type ChainClient struct {
    httpClient   *ethclient.Client
    wsClient     *ethclient.Client
    chainID      *big.Int
    privateKey   *ecdsa.PrivateKey
    address      common.Address

    mu           sync.RWMutex
    blockNumber  uint64
    baseFee      *big.Int
    nonce        uint64
}

// NewChainClient 创建链客户端
func NewChainClient(ctx context.Context, httpURL, wsURL, privateKeyHex string) (*ChainClient, error) {
    // 连接 HTTP 客户端
    httpClient, err := ethclient.DialContext(ctx, httpURL)
    if err != nil {
        return nil, fmt.Errorf("HTTP 连接失败: %w", err)
    }

    // 连接 WebSocket 客户端
    wsClient, err := ethclient.DialContext(ctx, wsURL)
    if err != nil {
        return nil, fmt.Errorf("WebSocket 连接失败: %w", err)
    }

    // 获取链 ID
    chainID, err := httpClient.ChainID(ctx)
    if err != nil {
        return nil, fmt.Errorf("获取链 ID 失败: %w", err)
    }

    // 解析私钥
    privateKey, err := crypto.HexToECDSA(privateKeyHex)
    if err != nil {
        return nil, fmt.Errorf("解析私钥失败: %w", err)
    }

    address := crypto.PubkeyToAddress(privateKey.PublicKey)

    client := &ChainClient{
        httpClient: httpClient,
        wsClient:   wsClient,
        chainID:    chainID,
        privateKey: privateKey,
        address:    address,
    }

    // 初始化状态
    if err := client.refreshState(ctx); err != nil {
        return nil, err
    }

    return client, nil
}

// refreshState 刷新状态
func (c *ChainClient) refreshState(ctx context.Context) error {
    // 获取当前区块
    block, err := c.httpClient.BlockByNumber(ctx, nil)
    if err != nil {
        return fmt.Errorf("获取区块失败: %w", err)
    }

    // 获取 nonce
    nonce, err := c.httpClient.PendingNonceAt(ctx, c.address)
    if err != nil {
        return fmt.Errorf("获取 nonce 失败: %w", err)
    }

    c.mu.Lock()
    c.blockNumber = block.NumberU64()
    c.baseFee = block.BaseFee()
    c.nonce = nonce
    c.mu.Unlock()

    return nil
}

// GetBlockNumber 获取当前区块号
func (c *ChainClient) GetBlockNumber() uint64 {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.blockNumber
}

// GetBaseFee 获取当前基础费
func (c *ChainClient) GetBaseFee() *big.Int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return new(big.Int).Set(c.baseFee)
}

// GetNonce 获取 nonce
func (c *ChainClient) GetNonce() uint64 {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.nonce
}

// IncrementNonce 增加 nonce
func (c *ChainClient) IncrementNonce() {
    c.mu.Lock()
    c.nonce++
    c.mu.Unlock()
}

// SubscribeNewBlocks 订阅新区块
func (c *ChainClient) SubscribeNewBlocks(ctx context.Context) (<-chan *types.Header, error) {
    headers := make(chan *types.Header, 10)

    sub, err := c.wsClient.SubscribeNewHead(ctx, headers)
    if err != nil {
        return nil, fmt.Errorf("订阅区块头失败: %w", err)
    }

    // 处理订阅错误
    go func() {
        for {
            select {
            case <-ctx.Done():
                sub.Unsubscribe()
                return
            case err := <-sub.Err():
                fmt.Printf("区块订阅错误: %v\n", err)
                return
            }
        }
    }()

    // 更新状态
    go func() {
        for header := range headers {
            c.mu.Lock()
            c.blockNumber = header.Number.Uint64()
            c.baseFee = header.BaseFee
            c.mu.Unlock()
        }
    }()

    return headers, nil
}

// Call 执行只读调用
func (c *ChainClient) Call(ctx context.Context, to common.Address, data []byte) ([]byte, error) {
    msg := ethereum.CallMsg{
        To:   &to,
        Data: data,
    }

    return c.httpClient.CallContract(ctx, msg, nil)
}

// EstimateGas 估算 gas
func (c *ChainClient) EstimateGas(ctx context.Context, to common.Address, value *big.Int, data []byte) (uint64, error) {
    msg := ethereum.CallMsg{
        From:  c.address,
        To:    &to,
        Value: value,
        Data:  data,
    }

    return c.httpClient.EstimateGas(ctx, msg)
}

// SimulateTransaction 模拟交易
func (c *ChainClient) SimulateTransaction(ctx context.Context, tx *types.Transaction) (*SimulationResult, error) {
    msg := ethereum.CallMsg{
        From:      c.address,
        To:        tx.To(),
        Gas:       tx.Gas(),
        GasPrice:  tx.GasPrice(),
        GasFeeCap: tx.GasFeeCap(),
        GasTipCap: tx.GasTipCap(),
        Value:     tx.Value(),
        Data:      tx.Data(),
    }

    // 执行模拟
    result, err := c.httpClient.CallContract(ctx, msg, nil)
    if err != nil {
        return &SimulationResult{
            Success: false,
            Error:   err.Error(),
        }, nil
    }

    return &SimulationResult{
        Success: true,
        Return:  result,
    }, nil
}

// SimulationResult 模拟结果
type SimulationResult struct {
    Success    bool
    Return     []byte
    GasUsed    uint64
    Error      string
    Logs       []*types.Log
}

// BuildTransaction 构建交易
func (c *ChainClient) BuildTransaction(ctx context.Context, to common.Address, value *big.Int, data []byte, gasLimit uint64) (*types.Transaction, error) {
    c.mu.RLock()
    nonce := c.nonce
    baseFee := c.baseFee
    c.mu.RUnlock()

    // 计算 gas 价格
    priorityFee := big.NewInt(2e9) // 2 Gwei
    maxFee := new(big.Int).Add(new(big.Int).Mul(baseFee, big.NewInt(2)), priorityFee)

    // 构建交易
    tx := types.NewTx(&types.DynamicFeeTx{
        ChainID:   c.chainID,
        Nonce:     nonce,
        GasTipCap: priorityFee,
        GasFeeCap: maxFee,
        Gas:       gasLimit,
        To:        &to,
        Value:     value,
        Data:      data,
    })

    // 签名
    return types.SignTx(tx, types.NewLondonSigner(c.chainID), c.privateKey)
}

// SendTransaction 发送交易
func (c *ChainClient) SendTransaction(ctx context.Context, tx *types.Transaction) error {
    if err := c.httpClient.SendTransaction(ctx, tx); err != nil {
        return err
    }

    c.IncrementNonce()
    return nil
}

// GetBalance 获取余额
func (c *ChainClient) GetBalance(ctx context.Context, address common.Address) (*big.Int, error) {
    return c.httpClient.BalanceAt(ctx, address, nil)
}

// GetTokenBalance 获取代币余额
func (c *ChainClient) GetTokenBalance(ctx context.Context, token, holder common.Address) (*big.Int, error) {
    // balanceOf(address)
    data := append(common.Hex2Bytes("70a08231"), common.LeftPadBytes(holder.Bytes(), 32)...)

    result, err := c.Call(ctx, token, data)
    if err != nil {
        return nil, err
    }

    return new(big.Int).SetBytes(result), nil
}

// Close 关闭客户端
func (c *ChainClient) Close() {
    c.httpClient.Close()
    c.wsClient.Close()
}
```

### 内存池监控

```go
package chain

import (
    "context"
    "fmt"
    "sync"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient/gethclient"
    "github.com/ethereum/go-ethereum/rpc"
)

// MempoolMonitor 内存池监控
type MempoolMonitor struct {
    rpcClient   *rpc.Client
    gethClient  *gethclient.Client

    mu          sync.RWMutex
    handlers    []PendingTxHandler
    filters     []TxFilter

    running     bool
}

// PendingTxHandler 待处理交易处理器
type PendingTxHandler func(*types.Transaction)

// TxFilter 交易过滤器
type TxFilter func(*types.Transaction) bool

// NewMempoolMonitor 创建内存池监控
func NewMempoolMonitor(wsURL string) (*MempoolMonitor, error) {
    rpcClient, err := rpc.Dial(wsURL)
    if err != nil {
        return nil, fmt.Errorf("RPC 连接失败: %w", err)
    }

    gethClient := gethclient.New(rpcClient)

    return &MempoolMonitor{
        rpcClient:  rpcClient,
        gethClient: gethClient,
        handlers:   make([]PendingTxHandler, 0),
        filters:    make([]TxFilter, 0),
    }, nil
}

// AddHandler 添加处理器
func (m *MempoolMonitor) AddHandler(handler PendingTxHandler) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.handlers = append(m.handlers, handler)
}

// AddFilter 添加过滤器
func (m *MempoolMonitor) AddFilter(filter TxFilter) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.filters = append(m.filters, filter)
}

// Start 启动监控
func (m *MempoolMonitor) Start(ctx context.Context) error {
    m.mu.Lock()
    if m.running {
        m.mu.Unlock()
        return nil
    }
    m.running = true
    m.mu.Unlock()

    // 订阅待处理交易
    txCh := make(chan common.Hash, 1000)
    sub, err := m.gethClient.SubscribePendingTransactions(ctx, txCh)
    if err != nil {
        return fmt.Errorf("订阅待处理交易失败: %w", err)
    }

    go m.processLoop(ctx, txCh, sub)

    return nil
}

// processLoop 处理循环
func (m *MempoolMonitor) processLoop(ctx context.Context, txCh <-chan common.Hash, sub *rpc.ClientSubscription) {
    defer sub.Unsubscribe()

    for {
        select {
        case <-ctx.Done():
            return
        case err := <-sub.Err():
            fmt.Printf("订阅错误: %v\n", err)
            return
        case txHash := <-txCh:
            go m.processTxHash(ctx, txHash)
        }
    }
}

// processTxHash 处理交易哈希
func (m *MempoolMonitor) processTxHash(ctx context.Context, txHash common.Hash) {
    // 获取完整交易
    var tx *types.Transaction
    err := m.rpcClient.CallContext(ctx, &tx, "eth_getTransactionByHash", txHash)
    if err != nil || tx == nil {
        return
    }

    // 应用过滤器
    m.mu.RLock()
    filters := m.filters
    handlers := m.handlers
    m.mu.RUnlock()

    for _, filter := range filters {
        if !filter(tx) {
            return
        }
    }

    // 调用处理器
    for _, handler := range handlers {
        handler(tx)
    }
}

// Stop 停止监控
func (m *MempoolMonitor) Stop() {
    m.mu.Lock()
    m.running = false
    m.mu.Unlock()
}

// DEXSwapFilter 创建 DEX Swap 过滤器
func DEXSwapFilter(routerAddresses []common.Address) TxFilter {
    routerSet := make(map[common.Address]bool)
    for _, addr := range routerAddresses {
        routerSet[addr] = true
    }

    // Swap 函数选择器
    swapSelectors := map[string]bool{
        "38ed1739": true, // swapExactTokensForTokens
        "8803dbee": true, // swapTokensForExactTokens
        "7ff36ab5": true, // swapExactETHForTokens
        "4a25d94a": true, // swapTokensForExactETH
        "18cbafe5": true, // swapExactTokensForETH
        "fb3bdb41": true, // swapETHForExactTokens
        "5c11d795": true, // swapExactTokensForTokensSupportingFeeOnTransferTokens
        "b6f9de95": true, // swapExactETHForTokensSupportingFeeOnTransferTokens
        "791ac947": true, // swapExactTokensForETHSupportingFeeOnTransferTokens
        "414bf389": true, // exactInputSingle (V3)
        "c04b8d59": true, // exactInput (V3)
        "db3e2198": true, // exactOutputSingle (V3)
        "f28c0498": true, // exactOutput (V3)
    }

    return func(tx *types.Transaction) bool {
        if tx.To() == nil {
            return false
        }

        // 检查目标地址
        if !routerSet[*tx.To()] {
            return false
        }

        // 检查函数选择器
        if len(tx.Data()) < 4 {
            return false
        }

        selector := common.Bytes2Hex(tx.Data()[:4])
        return swapSelectors[selector]
    }
}

// LargeSwapFilter 创建大额交易过滤器
func LargeSwapFilter(minValue *big.Int) TxFilter {
    return func(tx *types.Transaction) bool {
        // 检查 ETH value
        if tx.Value().Cmp(minValue) >= 0 {
            return true
        }

        // 对于 ERC20 swap，需要解析 calldata 中的金额
        // 简化实现
        return false
    }
}
```

---

## 第九十五节：池状态管理

### 池管理器

```go
package pool

import (
    "context"
    "fmt"
    "math/big"
    "sync"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
)

// PoolManager 池管理器
type PoolManager struct {
    chainClient ChainCaller

    mu          sync.RWMutex
    pools       map[common.Address]*Pool
    pairIndex   map[string][]common.Address // token0-token1 -> pools

    // 更新通道
    updateCh    chan *PoolUpdate
}

// ChainCaller 链调用接口
type ChainCaller interface {
    Call(ctx context.Context, to common.Address, data []byte) ([]byte, error)
}

// Pool 池信息
type Pool struct {
    Address     common.Address
    DEX         string
    Type        PoolType
    Token0      common.Address
    Token1      common.Address

    // V2 池状态
    Reserve0    *big.Int
    Reserve1    *big.Int

    // V3 池状态
    SqrtPriceX96 *big.Int
    Liquidity    *big.Int
    Tick         int32
    Fee          uint32

    // 元数据
    LastUpdate  time.Time
    BlockNumber uint64
}

// PoolType 池类型
type PoolType int

const (
    PoolTypeUniswapV2 PoolType = iota
    PoolTypeUniswapV3
    PoolTypeCurve
)

// PoolUpdate 池更新
type PoolUpdate struct {
    Address     common.Address
    Reserve0    *big.Int
    Reserve1    *big.Int
    BlockNumber uint64
}

// NewPoolManager 创建池管理器
func NewPoolManager(chainClient ChainCaller) *PoolManager {
    return &PoolManager{
        chainClient: chainClient,
        pools:       make(map[common.Address]*Pool),
        pairIndex:   make(map[string][]common.Address),
        updateCh:    make(chan *PoolUpdate, 1000),
    }
}

// AddPool 添加池
func (pm *PoolManager) AddPool(pool *Pool) {
    pm.mu.Lock()
    defer pm.mu.Unlock()

    pm.pools[pool.Address] = pool

    // 建立索引
    key := pm.makePairKey(pool.Token0, pool.Token1)
    pm.pairIndex[key] = append(pm.pairIndex[key], pool.Address)
}

// makePairKey 生成交易对键
func (pm *PoolManager) makePairKey(token0, token1 common.Address) string {
    if token0.Hex() < token1.Hex() {
        return token0.Hex() + "-" + token1.Hex()
    }
    return token1.Hex() + "-" + token0.Hex()
}

// GetPool 获取池
func (pm *PoolManager) GetPool(address common.Address) *Pool {
    pm.mu.RLock()
    defer pm.mu.RUnlock()
    return pm.pools[address]
}

// GetPoolsByPair 根据交易对获取池
func (pm *PoolManager) GetPoolsByPair(token0, token1 common.Address) []*Pool {
    pm.mu.RLock()
    defer pm.mu.RUnlock()

    key := pm.makePairKey(token0, token1)
    addresses := pm.pairIndex[key]

    pools := make([]*Pool, 0, len(addresses))
    for _, addr := range addresses {
        if pool, exists := pm.pools[addr]; exists {
            pools = append(pools, pool)
        }
    }

    return pools
}

// GetAllPools 获取所有池
func (pm *PoolManager) GetAllPools() []*Pool {
    pm.mu.RLock()
    defer pm.mu.RUnlock()

    pools := make([]*Pool, 0, len(pm.pools))
    for _, pool := range pm.pools {
        pools = append(pools, pool)
    }
    return pools
}

// RefreshPool 刷新池状态
func (pm *PoolManager) RefreshPool(ctx context.Context, address common.Address) error {
    pool := pm.GetPool(address)
    if pool == nil {
        return fmt.Errorf("池不存在: %s", address.Hex())
    }

    switch pool.Type {
    case PoolTypeUniswapV2:
        return pm.refreshV2Pool(ctx, pool)
    case PoolTypeUniswapV3:
        return pm.refreshV3Pool(ctx, pool)
    default:
        return fmt.Errorf("不支持的池类型")
    }
}

// refreshV2Pool 刷新 V2 池
func (pm *PoolManager) refreshV2Pool(ctx context.Context, pool *Pool) error {
    // getReserves()
    data := common.Hex2Bytes("0902f1ac")
    result, err := pm.chainClient.Call(ctx, pool.Address, data)
    if err != nil {
        return fmt.Errorf("获取储备失败: %w", err)
    }

    if len(result) < 64 {
        return fmt.Errorf("无效的返回数据")
    }

    pm.mu.Lock()
    pool.Reserve0 = new(big.Int).SetBytes(result[:32])
    pool.Reserve1 = new(big.Int).SetBytes(result[32:64])
    pool.LastUpdate = time.Now()
    pm.mu.Unlock()

    return nil
}

// refreshV3Pool 刷新 V3 池
func (pm *PoolManager) refreshV3Pool(ctx context.Context, pool *Pool) error {
    // slot0()
    data := common.Hex2Bytes("3850c7bd")
    result, err := pm.chainClient.Call(ctx, pool.Address, data)
    if err != nil {
        return fmt.Errorf("获取 slot0 失败: %w", err)
    }

    if len(result) < 128 {
        return fmt.Errorf("无效的返回数据")
    }

    pm.mu.Lock()
    pool.SqrtPriceX96 = new(big.Int).SetBytes(result[:32])
    pool.Tick = int32(new(big.Int).SetBytes(result[32:64]).Int64())
    pool.LastUpdate = time.Now()
    pm.mu.Unlock()

    // liquidity()
    data = common.Hex2Bytes("1a686502")
    result, err = pm.chainClient.Call(ctx, pool.Address, data)
    if err != nil {
        return fmt.Errorf("获取流动性失败: %w", err)
    }

    pm.mu.Lock()
    pool.Liquidity = new(big.Int).SetBytes(result)
    pm.mu.Unlock()

    return nil
}

// RefreshAll 刷新所有池
func (pm *PoolManager) RefreshAll(ctx context.Context) error {
    pools := pm.GetAllPools()

    var wg sync.WaitGroup
    errCh := make(chan error, len(pools))

    for _, pool := range pools {
        wg.Add(1)
        go func(p *Pool) {
            defer wg.Done()
            if err := pm.RefreshPool(ctx, p.Address); err != nil {
                errCh <- err
            }
        }(pool)
    }

    wg.Wait()
    close(errCh)

    var errs []error
    for err := range errCh {
        errs = append(errs, err)
    }

    if len(errs) > 0 {
        return fmt.Errorf("刷新失败: %d 个池出错", len(errs))
    }

    return nil
}

// ProcessSwapLog 处理 Swap 日志更新池状态
func (pm *PoolManager) ProcessSwapLog(log *types.Log) {
    pool := pm.GetPool(log.Address)
    if pool == nil {
        return
    }

    switch pool.Type {
    case PoolTypeUniswapV2:
        pm.processV2SwapLog(pool, log)
    case PoolTypeUniswapV3:
        pm.processV3SwapLog(pool, log)
    }
}

// processV2SwapLog 处理 V2 Swap 日志
func (pm *PoolManager) processV2SwapLog(pool *Pool, log *types.Log) {
    // Sync 事件: Sync(uint112 reserve0, uint112 reserve1)
    syncTopic := common.HexToHash("0x1c411e9a96e071241c2f21f7726b17ae89e3cab4c78be50e062b03a9fffbbad1")

    if len(log.Topics) > 0 && log.Topics[0] == syncTopic {
        if len(log.Data) >= 64 {
            pm.mu.Lock()
            pool.Reserve0 = new(big.Int).SetBytes(log.Data[:32])
            pool.Reserve1 = new(big.Int).SetBytes(log.Data[32:64])
            pool.LastUpdate = time.Now()
            pm.mu.Unlock()
        }
    }
}

// processV3SwapLog 处理 V3 Swap 日志
func (pm *PoolManager) processV3SwapLog(pool *Pool, log *types.Log) {
    // Swap 事件
    swapTopic := common.HexToHash("0xc42079f94a6350d7e6235f29174924f928cc2ac818eb64fed8004e115fbcca67")

    if len(log.Topics) > 0 && log.Topics[0] == swapTopic {
        if len(log.Data) >= 160 {
            pm.mu.Lock()
            pool.SqrtPriceX96 = new(big.Int).SetBytes(log.Data[64:96])
            pool.Liquidity = new(big.Int).SetBytes(log.Data[96:128])
            pool.Tick = int32(new(big.Int).SetBytes(log.Data[128:160]).Int64())
            pool.LastUpdate = time.Now()
            pm.mu.Unlock()
        }
    }
}

// CalculateV2Output 计算 V2 输出金额
func (pm *PoolManager) CalculateV2Output(poolAddr common.Address, amountIn *big.Int, zeroForOne bool) (*big.Int, error) {
    pool := pm.GetPool(poolAddr)
    if pool == nil {
        return nil, fmt.Errorf("池不存在")
    }

    if pool.Type != PoolTypeUniswapV2 {
        return nil, fmt.Errorf("非 V2 池")
    }

    pm.mu.RLock()
    reserve0 := new(big.Int).Set(pool.Reserve0)
    reserve1 := new(big.Int).Set(pool.Reserve1)
    pm.mu.RUnlock()

    var reserveIn, reserveOut *big.Int
    if zeroForOne {
        reserveIn = reserve0
        reserveOut = reserve1
    } else {
        reserveIn = reserve1
        reserveOut = reserve0
    }

    // amountOut = (amountIn * 997 * reserveOut) / (reserveIn * 1000 + amountIn * 997)
    amountInWithFee := new(big.Int).Mul(amountIn, big.NewInt(997))
    numerator := new(big.Int).Mul(amountInWithFee, reserveOut)
    denominator := new(big.Int).Add(
        new(big.Int).Mul(reserveIn, big.NewInt(1000)),
        amountInWithFee,
    )

    return new(big.Int).Div(numerator, denominator), nil
}

// CalculateV3Output 计算 V3 输出金额（简化版）
func (pm *PoolManager) CalculateV3Output(poolAddr common.Address, amountIn *big.Int, zeroForOne bool) (*big.Int, error) {
    pool := pm.GetPool(poolAddr)
    if pool == nil {
        return nil, fmt.Errorf("池不存在")
    }

    if pool.Type != PoolTypeUniswapV3 {
        return nil, fmt.Errorf("非 V3 池")
    }

    pm.mu.RLock()
    sqrtPriceX96 := new(big.Int).Set(pool.SqrtPriceX96)
    liquidity := new(big.Int).Set(pool.Liquidity)
    fee := pool.Fee
    pm.mu.RUnlock()

    // 简化计算（实际应该按 tick 逐步计算）
    // 这里假设流动性足够，价格变化不大

    q96 := new(big.Int).Exp(big.NewInt(2), big.NewInt(96), nil)

    if zeroForOne {
        // token0 -> token1
        // 计算新的 sqrtPrice
        // deltaY = L * deltaP
        feeAmount := new(big.Int).Div(
            new(big.Int).Mul(amountIn, big.NewInt(int64(fee))),
            big.NewInt(1000000),
        )
        amountInAfterFee := new(big.Int).Sub(amountIn, feeAmount)

        // deltaP = amountIn / L
        deltaP := new(big.Int).Div(
            new(big.Int).Mul(amountInAfterFee, q96),
            liquidity,
        )

        newSqrtPrice := new(big.Int).Add(sqrtPriceX96, deltaP)

        // amountOut = L * (1/sqrtP_old - 1/sqrtP_new)
        // 简化：amountOut ≈ amountIn * price * (1 - fee)
        price := new(big.Int).Div(
            new(big.Int).Mul(sqrtPriceX96, sqrtPriceX96),
            new(big.Int).Mul(q96, q96),
        )

        return new(big.Int).Div(
            new(big.Int).Mul(amountInAfterFee, price),
            big.NewInt(1e18),
        ), nil
    }

    // 反向类似
    return nil, fmt.Errorf("待实现")
}
```

---

## 第九十六节：套利策略实现

### 策略接口

```go
package strategy

import (
    "context"
    "math/big"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
)

// Strategy 策略接口
type Strategy interface {
    Name() string
    FindOpportunities(ctx context.Context) ([]*Opportunity, error)
    Execute(ctx context.Context, opp *Opportunity) (*ExecutionResult, error)
}

// Opportunity 套利机会
type Opportunity struct {
    ID            string
    Type          OpportunityType
    Path          []*SwapStep
    InputToken    common.Address
    InputAmount   *big.Int
    OutputAmount  *big.Int
    Profit        *big.Int
    ProfitUSD     float64
    GasEstimate   uint64
    GasCost       *big.Int
    NetProfit     *big.Int
    NetProfitUSD  float64
    BlockNumber   uint64
    Timestamp     int64
    Confidence    float64 // 0-1
}

// OpportunityType 机会类型
type OpportunityType int

const (
    OpportunityTriangular OpportunityType = iota
    OpportunityCrossDEX
    OpportunityFlashloan
)

// SwapStep 交换步骤
type SwapStep struct {
    DEX        string
    Pool       common.Address
    TokenIn    common.Address
    TokenOut   common.Address
    AmountIn   *big.Int
    AmountOut  *big.Int
    Fee        uint32
}

// ExecutionResult 执行结果
type ExecutionResult struct {
    Success       bool
    TxHash        common.Hash
    BlockNumber   uint64
    GasUsed       uint64
    ActualProfit  *big.Int
    Error         string
}
```

### 三角套利策略

```go
package strategy

import (
    "context"
    "fmt"
    "math/big"
    "sort"
    "sync"
    "time"

    "github.com/ethereum/go-ethereum/common"
)

// TriangularArbitrage 三角套利策略
type TriangularArbitrage struct {
    poolManager  PoolManager
    priceOracle  PriceOracle
    config       *TriangularConfig

    mu           sync.RWMutex
    opportunities []*Opportunity
}

// PoolManager 池管理器接口
type PoolManager interface {
    GetPool(address common.Address) *Pool
    GetPoolsByPair(token0, token1 common.Address) []*Pool
    GetAllPools() []*Pool
    CalculateV2Output(pool common.Address, amountIn *big.Int, zeroForOne bool) (*big.Int, error)
}

// PriceOracle 价格预言机接口
type PriceOracle interface {
    GetPriceUSD(token common.Address) (*big.Float, error)
}

// TriangularConfig 三角套利配置
type TriangularConfig struct {
    BaseTokens    []common.Address
    MinProfitUSD  float64
    MaxInputUSD   float64
    GasPrice      *big.Int
}

// Pool 池结构（简化）
type Pool struct {
    Address  common.Address
    DEX      string
    Token0   common.Address
    Token1   common.Address
    Reserve0 *big.Int
    Reserve1 *big.Int
}

// NewTriangularArbitrage 创建三角套利策略
func NewTriangularArbitrage(pm PoolManager, po PriceOracle, config *TriangularConfig) *TriangularArbitrage {
    return &TriangularArbitrage{
        poolManager: pm,
        priceOracle: po,
        config:      config,
    }
}

// Name 策略名称
func (t *TriangularArbitrage) Name() string {
    return "triangular_arbitrage"
}

// FindOpportunities 寻找套利机会
func (t *TriangularArbitrage) FindOpportunities(ctx context.Context) ([]*Opportunity, error) {
    var opportunities []*Opportunity

    // 对每个基础代币寻找三角套利
    for _, baseToken := range t.config.BaseTokens {
        opps, err := t.findTriangularPaths(ctx, baseToken)
        if err != nil {
            continue
        }
        opportunities = append(opportunities, opps...)
    }

    // 按利润排序
    sort.Slice(opportunities, func(i, j int) bool {
        return opportunities[i].NetProfitUSD > opportunities[j].NetProfitUSD
    })

    return opportunities, nil
}

// findTriangularPaths 寻找三角路径
func (t *TriangularArbitrage) findTriangularPaths(ctx context.Context, baseToken common.Address) ([]*Opportunity, error) {
    var opportunities []*Opportunity

    // 获取所有包含基础代币的池
    pools := t.poolManager.GetAllPools()

    // 建立图结构
    graph := t.buildGraph(pools)

    // 寻找三角路径: baseToken -> tokenA -> tokenB -> baseToken
    basePools := graph[baseToken]

    for _, poolA := range basePools {
        tokenA := t.getOtherToken(poolA, baseToken)

        // 从 tokenA 出发的池
        poolsFromA := graph[tokenA]
        for _, poolB := range poolsFromA {
            if poolB.Address == poolA.Address {
                continue
            }

            tokenB := t.getOtherToken(poolB, tokenA)
            if tokenB == baseToken {
                continue
            }

            // 寻找 tokenB -> baseToken 的池
            poolsFromB := graph[tokenB]
            for _, poolC := range poolsFromB {
                if poolC.Address == poolA.Address || poolC.Address == poolB.Address {
                    continue
                }

                tokenC := t.getOtherToken(poolC, tokenB)
                if tokenC != baseToken {
                    continue
                }

                // 找到三角路径
                opp := t.evaluateTriangular(ctx, baseToken, poolA, poolB, poolC, tokenA, tokenB)
                if opp != nil && opp.NetProfitUSD >= t.config.MinProfitUSD {
                    opportunities = append(opportunities, opp)
                }
            }
        }
    }

    return opportunities, nil
}

// buildGraph 构建图结构
func (t *TriangularArbitrage) buildGraph(pools []*Pool) map[common.Address][]*Pool {
    graph := make(map[common.Address][]*Pool)

    for _, pool := range pools {
        graph[pool.Token0] = append(graph[pool.Token0], pool)
        graph[pool.Token1] = append(graph[pool.Token1], pool)
    }

    return graph
}

// getOtherToken 获取另一个代币
func (t *TriangularArbitrage) getOtherToken(pool *Pool, token common.Address) common.Address {
    if pool.Token0 == token {
        return pool.Token1
    }
    return pool.Token0
}

// evaluateTriangular 评估三角套利
func (t *TriangularArbitrage) evaluateTriangular(
    ctx context.Context,
    baseToken common.Address,
    poolA, poolB, poolC *Pool,
    tokenA, tokenB common.Address,
) *Opportunity {
    // 获取基础代币价格
    basePrice, err := t.priceOracle.GetPriceUSD(baseToken)
    if err != nil {
        return nil
    }

    // 计算最优输入金额
    optimalInput := t.calculateOptimalInput(poolA, poolB, poolC, baseToken, tokenA, tokenB)
    if optimalInput == nil || optimalInput.Sign() <= 0 {
        return nil
    }

    // 限制最大输入
    maxInputWei := t.usdToWei(basePrice, t.config.MaxInputUSD)
    if optimalInput.Cmp(maxInputWei) > 0 {
        optimalInput = maxInputWei
    }

    // 计算输出
    output, steps := t.simulateSwaps(optimalInput, baseToken, poolA, poolB, poolC, tokenA, tokenB)
    if output == nil {
        return nil
    }

    // 计算利润
    profit := new(big.Int).Sub(output, optimalInput)
    if profit.Sign() <= 0 {
        return nil
    }

    profitUSD := t.weiToUSD(basePrice, profit)

    // 估算 gas
    gasEstimate := uint64(300000) // 三角套利大约 300k gas
    gasCost := new(big.Int).Mul(big.NewInt(int64(gasEstimate)), t.config.GasPrice)

    // 计算净利润
    netProfit := new(big.Int).Sub(profit, gasCost)
    netProfitUSD := t.weiToUSD(basePrice, netProfit)

    if netProfitUSD < t.config.MinProfitUSD {
        return nil
    }

    return &Opportunity{
        ID:           fmt.Sprintf("tri-%s-%s-%s", poolA.Address.Hex()[:8], poolB.Address.Hex()[:8], poolC.Address.Hex()[:8]),
        Type:         OpportunityTriangular,
        Path:         steps,
        InputToken:   baseToken,
        InputAmount:  optimalInput,
        OutputAmount: output,
        Profit:       profit,
        ProfitUSD:    profitUSD,
        GasEstimate:  gasEstimate,
        GasCost:      gasCost,
        NetProfit:    netProfit,
        NetProfitUSD: netProfitUSD,
        Timestamp:    time.Now().Unix(),
        Confidence:   0.8,
    }
}

// calculateOptimalInput 计算最优输入金额
func (t *TriangularArbitrage) calculateOptimalInput(
    poolA, poolB, poolC *Pool,
    baseToken, tokenA, tokenB common.Address,
) *big.Int {
    // 使用二分法寻找最优输入
    // 利润函数通常是先增后减的

    low := big.NewInt(1e15)   // 0.001 ETH
    high := big.NewInt(1e20)  // 100 ETH

    precision := big.NewInt(1e14) // 0.0001 ETH

    var bestInput *big.Int
    var bestProfit *big.Int = big.NewInt(0)

    for new(big.Int).Sub(high, low).Cmp(precision) > 0 {
        mid := new(big.Int).Add(low, new(big.Int).Div(new(big.Int).Sub(high, low), big.NewInt(2)))

        output, _ := t.simulateSwaps(mid, baseToken, poolA, poolB, poolC, tokenA, tokenB)
        if output == nil {
            high = mid
            continue
        }

        profit := new(big.Int).Sub(output, mid)

        if profit.Cmp(bestProfit) > 0 {
            bestProfit = profit
            bestInput = new(big.Int).Set(mid)
        }

        // 检查导数符号来决定搜索方向
        delta := big.NewInt(1e15)
        midPlus := new(big.Int).Add(mid, delta)
        outputPlus, _ := t.simulateSwaps(midPlus, baseToken, poolA, poolB, poolC, tokenA, tokenB)
        if outputPlus == nil {
            high = mid
            continue
        }

        profitPlus := new(big.Int).Sub(outputPlus, midPlus)

        if profitPlus.Cmp(profit) > 0 {
            low = mid
        } else {
            high = mid
        }
    }

    return bestInput
}

// simulateSwaps 模拟交换
func (t *TriangularArbitrage) simulateSwaps(
    inputAmount *big.Int,
    baseToken common.Address,
    poolA, poolB, poolC *Pool,
    tokenA, tokenB common.Address,
) (*big.Int, []*SwapStep) {
    steps := make([]*SwapStep, 0, 3)

    // Step 1: baseToken -> tokenA
    zeroForOne := poolA.Token0 == baseToken
    amount1, err := t.poolManager.CalculateV2Output(poolA.Address, inputAmount, zeroForOne)
    if err != nil || amount1.Sign() <= 0 {
        return nil, nil
    }
    steps = append(steps, &SwapStep{
        DEX:       poolA.DEX,
        Pool:      poolA.Address,
        TokenIn:   baseToken,
        TokenOut:  tokenA,
        AmountIn:  inputAmount,
        AmountOut: amount1,
    })

    // Step 2: tokenA -> tokenB
    zeroForOne = poolB.Token0 == tokenA
    amount2, err := t.poolManager.CalculateV2Output(poolB.Address, amount1, zeroForOne)
    if err != nil || amount2.Sign() <= 0 {
        return nil, nil
    }
    steps = append(steps, &SwapStep{
        DEX:       poolB.DEX,
        Pool:      poolB.Address,
        TokenIn:   tokenA,
        TokenOut:  tokenB,
        AmountIn:  amount1,
        AmountOut: amount2,
    })

    // Step 3: tokenB -> baseToken
    zeroForOne = poolC.Token0 == tokenB
    amount3, err := t.poolManager.CalculateV2Output(poolC.Address, amount2, zeroForOne)
    if err != nil || amount3.Sign() <= 0 {
        return nil, nil
    }
    steps = append(steps, &SwapStep{
        DEX:       poolC.DEX,
        Pool:      poolC.Address,
        TokenIn:   tokenB,
        TokenOut:  baseToken,
        AmountIn:  amount2,
        AmountOut: amount3,
    })

    return amount3, steps
}

// usdToWei 美元转 Wei
func (t *TriangularArbitrage) usdToWei(price *big.Float, usd float64) *big.Int {
    priceFloat, _ := price.Float64()
    wei := usd / priceFloat * 1e18
    return big.NewInt(int64(wei))
}

// weiToUSD Wei 转美元
func (t *TriangularArbitrage) weiToUSD(price *big.Float, wei *big.Int) float64 {
    priceFloat, _ := price.Float64()
    weiFloat := new(big.Float).SetInt(wei)
    weiFloat.Quo(weiFloat, big.NewFloat(1e18))
    weiValue, _ := weiFloat.Float64()
    return weiValue * priceFloat
}

// Execute 执行套利
func (t *TriangularArbitrage) Execute(ctx context.Context, opp *Opportunity) (*ExecutionResult, error) {
    // 执行逻辑将在执行层实现
    return nil, fmt.Errorf("通过执行层执行")
}
```

### 跨 DEX 套利策略

```go
package strategy

import (
    "context"
    "fmt"
    "math/big"
    "sort"
    "time"
)

// CrossDEXArbitrage 跨 DEX 套利策略
type CrossDEXArbitrage struct {
    poolManager  PoolManager
    priceOracle  PriceOracle
    config       *CrossDEXConfig
}

// CrossDEXConfig 跨 DEX 配置
type CrossDEXConfig struct {
    TokenPairs    [][2]common.Address // 监控的交易对
    DEXs          []string            // 参与的 DEX
    MinProfitUSD  float64
    MaxInputUSD   float64
    GasPrice      *big.Int
}

// NewCrossDEXArbitrage 创建跨 DEX 套利策略
func NewCrossDEXArbitrage(pm PoolManager, po PriceOracle, config *CrossDEXConfig) *CrossDEXArbitrage {
    return &CrossDEXArbitrage{
        poolManager: pm,
        priceOracle: po,
        config:      config,
    }
}

// Name 策略名称
func (c *CrossDEXArbitrage) Name() string {
    return "cross_dex_arbitrage"
}

// FindOpportunities 寻找机会
func (c *CrossDEXArbitrage) FindOpportunities(ctx context.Context) ([]*Opportunity, error) {
    var opportunities []*Opportunity

    for _, pair := range c.config.TokenPairs {
        token0, token1 := pair[0], pair[1]

        // 获取该交易对在不同 DEX 的池
        pools := c.poolManager.GetPoolsByPair(token0, token1)
        if len(pools) < 2 {
            continue
        }

        // 比较价格，寻找套利机会
        opps := c.findCrossDEXOpportunities(ctx, token0, token1, pools)
        opportunities = append(opportunities, opps...)
    }

    // 按利润排序
    sort.Slice(opportunities, func(i, j int) bool {
        return opportunities[i].NetProfitUSD > opportunities[j].NetProfitUSD
    })

    return opportunities, nil
}

// findCrossDEXOpportunities 寻找跨 DEX 机会
func (c *CrossDEXArbitrage) findCrossDEXOpportunities(
    ctx context.Context,
    token0, token1 common.Address,
    pools []*Pool,
) []*Opportunity {
    var opportunities []*Opportunity

    // 计算每个池的价格
    type poolPrice struct {
        pool  *Pool
        price *big.Float // token1/token0
    }

    var prices []poolPrice
    for _, pool := range pools {
        price := c.calculatePrice(pool, token0)
        if price != nil {
            prices = append(prices, poolPrice{pool, price})
        }
    }

    if len(prices) < 2 {
        return nil
    }

    // 比较价格差异
    for i := 0; i < len(prices); i++ {
        for j := i + 1; j < len(prices); j++ {
            // 计算价差
            priceDiff := new(big.Float).Sub(prices[i].price, prices[j].price)
            priceDiffAbs := new(big.Float).Abs(priceDiff)

            // 计算价差百分比
            avgPrice := new(big.Float).Add(prices[i].price, prices[j].price)
            avgPrice.Quo(avgPrice, big.NewFloat(2))

            diffPercent := new(big.Float).Quo(priceDiffAbs, avgPrice)
            diffPercentFloat, _ := diffPercent.Float64()

            // 如果价差 > 0.3%（扣除两次 0.3% 手续费后仍有利润）
            if diffPercentFloat > 0.006 {
                var buyPool, sellPool *Pool
                if prices[i].price.Cmp(prices[j].price) < 0 {
                    buyPool = prices[i].pool   // 价格低，买入
                    sellPool = prices[j].pool  // 价格高，卖出
                } else {
                    buyPool = prices[j].pool
                    sellPool = prices[i].pool
                }

                opp := c.evaluateCrossDEX(ctx, token0, token1, buyPool, sellPool)
                if opp != nil && opp.NetProfitUSD >= c.config.MinProfitUSD {
                    opportunities = append(opportunities, opp)
                }
            }
        }
    }

    return opportunities
}

// calculatePrice 计算价格
func (c *CrossDEXArbitrage) calculatePrice(pool *Pool, token0 common.Address) *big.Float {
    if pool.Reserve0 == nil || pool.Reserve1 == nil {
        return nil
    }

    if pool.Reserve0.Sign() == 0 || pool.Reserve1.Sign() == 0 {
        return nil
    }

    reserve0 := new(big.Float).SetInt(pool.Reserve0)
    reserve1 := new(big.Float).SetInt(pool.Reserve1)

    if pool.Token0 == token0 {
        return new(big.Float).Quo(reserve1, reserve0)
    }
    return new(big.Float).Quo(reserve0, reserve1)
}

// evaluateCrossDEX 评估跨 DEX 套利
func (c *CrossDEXArbitrage) evaluateCrossDEX(
    ctx context.Context,
    token0, token1 common.Address,
    buyPool, sellPool *Pool,
) *Opportunity {
    // 获取 token0 价格
    price, err := c.priceOracle.GetPriceUSD(token0)
    if err != nil {
        return nil
    }

    // 计算最优输入
    optimalInput := c.calculateOptimalCrossDEXInput(buyPool, sellPool, token0, token1)
    if optimalInput == nil || optimalInput.Sign() <= 0 {
        return nil
    }

    // 限制最大输入
    maxInputWei := c.usdToWei(price, c.config.MaxInputUSD)
    if optimalInput.Cmp(maxInputWei) > 0 {
        optimalInput = maxInputWei
    }

    // 模拟交易
    // Step 1: 在 buyPool 买入 token1（用 token0）
    zeroForOne := buyPool.Token0 == token0
    amount1, err := c.poolManager.CalculateV2Output(buyPool.Address, optimalInput, zeroForOne)
    if err != nil || amount1.Sign() <= 0 {
        return nil
    }

    // Step 2: 在 sellPool 卖出 token1（得到 token0）
    zeroForOne = sellPool.Token0 == token1
    output, err := c.poolManager.CalculateV2Output(sellPool.Address, amount1, zeroForOne)
    if err != nil || output.Sign() <= 0 {
        return nil
    }

    // 计算利润
    profit := new(big.Int).Sub(output, optimalInput)
    if profit.Sign() <= 0 {
        return nil
    }

    profitUSD := c.weiToUSD(price, profit)

    // Gas 估算
    gasEstimate := uint64(200000) // 两次 swap 约 200k gas
    gasCost := new(big.Int).Mul(big.NewInt(int64(gasEstimate)), c.config.GasPrice)

    netProfit := new(big.Int).Sub(profit, gasCost)
    netProfitUSD := c.weiToUSD(price, netProfit)

    if netProfitUSD < c.config.MinProfitUSD {
        return nil
    }

    steps := []*SwapStep{
        {
            DEX:       buyPool.DEX,
            Pool:      buyPool.Address,
            TokenIn:   token0,
            TokenOut:  token1,
            AmountIn:  optimalInput,
            AmountOut: amount1,
        },
        {
            DEX:       sellPool.DEX,
            Pool:      sellPool.Address,
            TokenIn:   token1,
            TokenOut:  token0,
            AmountIn:  amount1,
            AmountOut: output,
        },
    }

    return &Opportunity{
        ID:           fmt.Sprintf("cross-%s-%s", buyPool.Address.Hex()[:8], sellPool.Address.Hex()[:8]),
        Type:         OpportunityCrossDEX,
        Path:         steps,
        InputToken:   token0,
        InputAmount:  optimalInput,
        OutputAmount: output,
        Profit:       profit,
        ProfitUSD:    profitUSD,
        GasEstimate:  gasEstimate,
        GasCost:      gasCost,
        NetProfit:    netProfit,
        NetProfitUSD: netProfitUSD,
        Timestamp:    time.Now().Unix(),
        Confidence:   0.9,
    }
}

// calculateOptimalCrossDEXInput 计算最优输入
func (c *CrossDEXArbitrage) calculateOptimalCrossDEXInput(
    buyPool, sellPool *Pool,
    token0, token1 common.Address,
) *big.Int {
    // 简化实现：使用公式计算
    // 对于恒定乘积池，最优输入有解析解

    // 假设 buyPool: R0_a, R1_a  sellPool: R0_b, R1_b
    // 我们要最大化: f(x) = sellPool.getAmountOut(buyPool.getAmountOut(x)) - x

    // 使用二分法
    low := big.NewInt(1e15)   // 0.001
    high := big.NewInt(1e20)  // 100
    precision := big.NewInt(1e14)

    var bestInput *big.Int
    var bestProfit *big.Int = big.NewInt(0)

    for new(big.Int).Sub(high, low).Cmp(precision) > 0 {
        mid := new(big.Int).Add(low, new(big.Int).Div(new(big.Int).Sub(high, low), big.NewInt(2)))

        // 计算利润
        profit := c.calculateCrossDEXProfit(mid, buyPool, sellPool, token0, token1)
        if profit == nil {
            high = mid
            continue
        }

        if profit.Cmp(bestProfit) > 0 {
            bestProfit = new(big.Int).Set(profit)
            bestInput = new(big.Int).Set(mid)
        }

        // 检查梯度
        delta := big.NewInt(1e15)
        midPlus := new(big.Int).Add(mid, delta)
        profitPlus := c.calculateCrossDEXProfit(midPlus, buyPool, sellPool, token0, token1)
        if profitPlus == nil {
            high = mid
            continue
        }

        if profitPlus.Cmp(profit) > 0 {
            low = mid
        } else {
            high = mid
        }
    }

    return bestInput
}

// calculateCrossDEXProfit 计算跨 DEX 利润
func (c *CrossDEXArbitrage) calculateCrossDEXProfit(
    input *big.Int,
    buyPool, sellPool *Pool,
    token0, token1 common.Address,
) *big.Int {
    // Step 1
    zeroForOne := buyPool.Token0 == token0
    amount1, err := c.poolManager.CalculateV2Output(buyPool.Address, input, zeroForOne)
    if err != nil || amount1.Sign() <= 0 {
        return nil
    }

    // Step 2
    zeroForOne = sellPool.Token0 == token1
    output, err := c.poolManager.CalculateV2Output(sellPool.Address, amount1, zeroForOne)
    if err != nil || output.Sign() <= 0 {
        return nil
    }

    return new(big.Int).Sub(output, input)
}

// 辅助方法
func (c *CrossDEXArbitrage) usdToWei(price *big.Float, usd float64) *big.Int {
    priceFloat, _ := price.Float64()
    wei := usd / priceFloat * 1e18
    return big.NewInt(int64(wei))
}

func (c *CrossDEXArbitrage) weiToUSD(price *big.Float, wei *big.Int) float64 {
    priceFloat, _ := price.Float64()
    weiFloat := new(big.Float).SetInt(wei)
    weiFloat.Quo(weiFloat, big.NewFloat(1e18))
    weiValue, _ := weiFloat.Float64()
    return weiValue * priceFloat
}

// Execute 执行套利
func (c *CrossDEXArbitrage) Execute(ctx context.Context, opp *Opportunity) (*ExecutionResult, error) {
    return nil, fmt.Errorf("通过执行层执行")
}
```

---

## 第九十七节：执行层实现

### Bundle 构建器

```go
package execution

import (
    "context"
    "crypto/ecdsa"
    "fmt"
    "math/big"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/crypto"
)

// BundleBuilder Bundle 构建器
type BundleBuilder struct {
    chainID        *big.Int
    privateKey     *ecdsa.PrivateKey
    arbitrageAddr  common.Address
    signer         types.Signer
}

// NewBundleBuilder 创建构建器
func NewBundleBuilder(chainID *big.Int, privateKey *ecdsa.PrivateKey, arbitrageAddr common.Address) *BundleBuilder {
    return &BundleBuilder{
        chainID:       chainID,
        privateKey:    privateKey,
        arbitrageAddr: arbitrageAddr,
        signer:        types.NewLondonSigner(chainID),
    }
}

// BuildArbitrageBundle 构建套利 Bundle
func (b *BundleBuilder) BuildArbitrageBundle(
    ctx context.Context,
    opp *Opportunity,
    nonce uint64,
    baseFee *big.Int,
    priorityFee *big.Int,
) (*Bundle, error) {
    // 构建套利交易数据
    calldata, err := b.buildArbitrageCalldata(opp)
    if err != nil {
        return nil, fmt.Errorf("构建 calldata 失败: %w", err)
    }

    // 计算 maxFeePerGas
    maxFee := new(big.Int).Add(new(big.Int).Mul(baseFee, big.NewInt(2)), priorityFee)

    // 构建交易
    tx := types.NewTx(&types.DynamicFeeTx{
        ChainID:   b.chainID,
        Nonce:     nonce,
        GasTipCap: priorityFee,
        GasFeeCap: maxFee,
        Gas:       opp.GasEstimate + 50000, // 添加缓冲
        To:        &b.arbitrageAddr,
        Value:     big.NewInt(0),
        Data:      calldata,
    })

    // 签名
    signedTx, err := types.SignTx(tx, b.signer, b.privateKey)
    if err != nil {
        return nil, fmt.Errorf("签名失败: %w", err)
    }

    return &Bundle{
        Transactions: []*types.Transaction{signedTx},
        BlockNumber:  0, // 将在提交时设置
    }, nil
}

// buildArbitrageCalldata 构建套利 calldata
func (b *BundleBuilder) buildArbitrageCalldata(opp *Opportunity) ([]byte, error) {
    // 根据机会类型构建不同的 calldata

    switch opp.Type {
    case OpportunityTriangular:
        return b.buildTriangularCalldata(opp)
    case OpportunityCrossDEX:
        return b.buildCrossDEXCalldata(opp)
    case OpportunityFlashloan:
        return b.buildFlashloanCalldata(opp)
    default:
        return nil, fmt.Errorf("未知机会类型")
    }
}

// buildTriangularCalldata 构建三角套利 calldata
func (b *BundleBuilder) buildTriangularCalldata(opp *Opportunity) ([]byte, error) {
    // executeTriangular(
    //     address[] pools,
    //     address[] tokens,
    //     uint256 amountIn,
    //     uint256 minAmountOut
    // )
    methodID := common.Hex2Bytes("12345678") // 替换为实际方法 ID

    // 编码参数
    pools := make([]common.Address, len(opp.Path))
    tokens := make([]common.Address, len(opp.Path)+1)

    tokens[0] = opp.InputToken
    for i, step := range opp.Path {
        pools[i] = step.Pool
        tokens[i+1] = step.TokenOut
    }

    // 计算最小输出（设置 1% 滑点保护）
    minOutput := new(big.Int).Mul(opp.OutputAmount, big.NewInt(99))
    minOutput.Div(minOutput, big.NewInt(100))

    // ABI 编码
    data := methodID
    data = append(data, b.encodeAddressArray(pools)...)
    data = append(data, b.encodeAddressArray(tokens)...)
    data = append(data, common.LeftPadBytes(opp.InputAmount.Bytes(), 32)...)
    data = append(data, common.LeftPadBytes(minOutput.Bytes(), 32)...)

    return data, nil
}

// buildCrossDEXCalldata 构建跨 DEX calldata
func (b *BundleBuilder) buildCrossDEXCalldata(opp *Opportunity) ([]byte, error) {
    // executeCrossDEX(
    //     address pool0,
    //     address pool1,
    //     address tokenIn,
    //     address tokenMid,
    //     uint256 amountIn,
    //     uint256 minAmountOut
    // )
    methodID := common.Hex2Bytes("87654321") // 替换为实际方法 ID

    pool0 := opp.Path[0].Pool
    pool1 := opp.Path[1].Pool
    tokenIn := opp.Path[0].TokenIn
    tokenMid := opp.Path[0].TokenOut

    minOutput := new(big.Int).Mul(opp.OutputAmount, big.NewInt(99))
    minOutput.Div(minOutput, big.NewInt(100))

    data := methodID
    data = append(data, common.LeftPadBytes(pool0.Bytes(), 32)...)
    data = append(data, common.LeftPadBytes(pool1.Bytes(), 32)...)
    data = append(data, common.LeftPadBytes(tokenIn.Bytes(), 32)...)
    data = append(data, common.LeftPadBytes(tokenMid.Bytes(), 32)...)
    data = append(data, common.LeftPadBytes(opp.InputAmount.Bytes(), 32)...)
    data = append(data, common.LeftPadBytes(minOutput.Bytes(), 32)...)

    return data, nil
}

// buildFlashloanCalldata 构建闪电贷 calldata
func (b *BundleBuilder) buildFlashloanCalldata(opp *Opportunity) ([]byte, error) {
    // executeFlashloan(
    //     address lender,
    //     address token,
    //     uint256 amount,
    //     bytes swapData
    // )
    methodID := common.Hex2Bytes("abcdef12") // 替换为实际方法 ID

    // 编码 swap 数据
    swapData, err := b.encodeSwapPath(opp.Path)
    if err != nil {
        return nil, err
    }

    data := methodID
    // lender 地址 (如 Aave)
    data = append(data, common.LeftPadBytes(common.HexToAddress("0x...").Bytes(), 32)...)
    data = append(data, common.LeftPadBytes(opp.InputToken.Bytes(), 32)...)
    data = append(data, common.LeftPadBytes(opp.InputAmount.Bytes(), 32)...)
    // swapData 偏移量和数据
    data = append(data, common.LeftPadBytes(big.NewInt(128).Bytes(), 32)...) // offset
    data = append(data, swapData...)

    return data, nil
}

// encodeAddressArray 编码地址数组
func (b *BundleBuilder) encodeAddressArray(addresses []common.Address) []byte {
    // 动态数组编码
    data := common.LeftPadBytes(big.NewInt(int64(len(addresses))).Bytes(), 32)
    for _, addr := range addresses {
        data = append(data, common.LeftPadBytes(addr.Bytes(), 32)...)
    }
    return data
}

// encodeSwapPath 编码 swap 路径
func (b *BundleBuilder) encodeSwapPath(path []*SwapStep) ([]byte, error) {
    // 自定义编码格式
    var data []byte

    // 步骤数量
    data = append(data, common.LeftPadBytes(big.NewInt(int64(len(path))).Bytes(), 32)...)

    for _, step := range path {
        data = append(data, common.LeftPadBytes(step.Pool.Bytes(), 32)...)
        data = append(data, common.LeftPadBytes(step.TokenIn.Bytes(), 32)...)
        data = append(data, common.LeftPadBytes(step.TokenOut.Bytes(), 32)...)
    }

    return data, nil
}

// Bundle 定义
type Bundle struct {
    Transactions []*types.Transaction
    BlockNumber  uint64
    MinTimestamp uint64
    MaxTimestamp uint64
}

// Opportunity 引用
type Opportunity struct {
    Type         OpportunityType
    Path         []*SwapStep
    InputToken   common.Address
    InputAmount  *big.Int
    OutputAmount *big.Int
    GasEstimate  uint64
}

type OpportunityType int

const (
    OpportunityTriangular OpportunityType = iota
    OpportunityCrossDEX
    OpportunityFlashloan
)

type SwapStep struct {
    Pool     common.Address
    TokenIn  common.Address
    TokenOut common.Address
}
```

### Flashbots 提交

```go
package execution

import (
    "bytes"
    "context"
    "crypto/ecdsa"
    "encoding/json"
    "fmt"
    "io"
    "math/big"
    "net/http"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/common/hexutil"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/crypto"
)

// FlashbotsSubmitter Flashbots 提交器
type FlashbotsSubmitter struct {
    relayURL   string
    signingKey *ecdsa.PrivateKey
    httpClient *http.Client
}

// NewFlashbotsSubmitter 创建提交器
func NewFlashbotsSubmitter(relayURL string, signingKey *ecdsa.PrivateKey) *FlashbotsSubmitter {
    return &FlashbotsSubmitter{
        relayURL:   relayURL,
        signingKey: signingKey,
        httpClient: &http.Client{Timeout: 10 * time.Second},
    }
}

// SubmitBundle 提交 Bundle
func (f *FlashbotsSubmitter) SubmitBundle(ctx context.Context, bundle *Bundle) (*SubmissionResult, error) {
    // 编码交易
    txs := make([]string, len(bundle.Transactions))
    for i, tx := range bundle.Transactions {
        txBytes, err := tx.MarshalBinary()
        if err != nil {
            return nil, fmt.Errorf("编码交易失败: %w", err)
        }
        txs[i] = hexutil.Encode(txBytes)
    }

    // 构建请求
    params := map[string]interface{}{
        "txs":         txs,
        "blockNumber": hexutil.EncodeUint64(bundle.BlockNumber),
    }

    if bundle.MinTimestamp > 0 {
        params["minTimestamp"] = bundle.MinTimestamp
    }
    if bundle.MaxTimestamp > 0 {
        params["maxTimestamp"] = bundle.MaxTimestamp
    }

    request := map[string]interface{}{
        "jsonrpc": "2.0",
        "id":      1,
        "method":  "eth_sendBundle",
        "params":  []interface{}{params},
    }

    // 发送请求
    result, err := f.sendRequest(ctx, request)
    if err != nil {
        return nil, err
    }

    return &SubmissionResult{
        BundleHash: common.HexToHash(result.BundleHash),
        Success:    true,
    }, nil
}

// SimulateBundle 模拟 Bundle
func (f *FlashbotsSubmitter) SimulateBundle(ctx context.Context, bundle *Bundle) (*SimulationResult, error) {
    // 编码交易
    txs := make([]string, len(bundle.Transactions))
    for i, tx := range bundle.Transactions {
        txBytes, err := tx.MarshalBinary()
        if err != nil {
            return nil, fmt.Errorf("编码交易失败: %w", err)
        }
        txs[i] = hexutil.Encode(txBytes)
    }

    // 构建请求
    params := map[string]interface{}{
        "txs":              txs,
        "blockNumber":      hexutil.EncodeUint64(bundle.BlockNumber),
        "stateBlockNumber": "latest",
    }

    request := map[string]interface{}{
        "jsonrpc": "2.0",
        "id":      1,
        "method":  "eth_callBundle",
        "params":  []interface{}{params},
    }

    // 发送请求
    result, err := f.sendSimRequest(ctx, request)
    if err != nil {
        return nil, err
    }

    return result, nil
}

// sendRequest 发送请求
func (f *FlashbotsSubmitter) sendRequest(ctx context.Context, request map[string]interface{}) (*BundleResponse, error) {
    body, err := json.Marshal(request)
    if err != nil {
        return nil, fmt.Errorf("序列化请求失败: %w", err)
    }

    // 签名
    signature := f.signBody(body)

    req, err := http.NewRequestWithContext(ctx, "POST", f.relayURL, bytes.NewReader(body))
    if err != nil {
        return nil, fmt.Errorf("创建请求失败: %w", err)
    }

    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-Flashbots-Signature", signature)

    resp, err := f.httpClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("发送请求失败: %w", err)
    }
    defer resp.Body.Close()

    respBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("读取响应失败: %w", err)
    }

    var rpcResp struct {
        Result *BundleResponse `json:"result"`
        Error  *struct {
            Code    int    `json:"code"`
            Message string `json:"message"`
        } `json:"error"`
    }

    if err := json.Unmarshal(respBody, &rpcResp); err != nil {
        return nil, fmt.Errorf("解析响应失败: %w", err)
    }

    if rpcResp.Error != nil {
        return nil, fmt.Errorf("RPC 错误: %s", rpcResp.Error.Message)
    }

    return rpcResp.Result, nil
}

// sendSimRequest 发送模拟请求
func (f *FlashbotsSubmitter) sendSimRequest(ctx context.Context, request map[string]interface{}) (*SimulationResult, error) {
    body, err := json.Marshal(request)
    if err != nil {
        return nil, err
    }

    signature := f.signBody(body)

    req, err := http.NewRequestWithContext(ctx, "POST", f.relayURL, bytes.NewReader(body))
    if err != nil {
        return nil, err
    }

    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-Flashbots-Signature", signature)

    resp, err := f.httpClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    respBody, _ := io.ReadAll(resp.Body)

    var rpcResp struct {
        Result *SimulationResult `json:"result"`
        Error  *struct {
            Message string `json:"message"`
        } `json:"error"`
    }

    if err := json.Unmarshal(respBody, &rpcResp); err != nil {
        return nil, err
    }

    if rpcResp.Error != nil {
        return nil, fmt.Errorf("模拟失败: %s", rpcResp.Error.Message)
    }

    return rpcResp.Result, nil
}

// signBody 签名请求体
func (f *FlashbotsSubmitter) signBody(body []byte) string {
    hash := crypto.Keccak256Hash(body)
    sig, _ := crypto.Sign(hash.Bytes(), f.signingKey)
    pubKey := crypto.PubkeyToAddress(f.signingKey.PublicKey)
    return fmt.Sprintf("%s:%s", pubKey.Hex(), hexutil.Encode(sig))
}

// BundleResponse Bundle 响应
type BundleResponse struct {
    BundleHash string `json:"bundleHash"`
}

// SubmissionResult 提交结果
type SubmissionResult struct {
    BundleHash common.Hash
    Success    bool
    Error      string
}

// SimulationResult 模拟结果
type SimulationResult struct {
    Success         bool         `json:"success"`
    TxResults       []TxResult   `json:"results"`
    CoinbaseDiff    *big.Int     `json:"coinbaseDiff"`
    GasFees         *big.Int     `json:"gasFees"`
    StateBlockNumber uint64      `json:"stateBlockNumber"`
    TotalGasUsed    uint64       `json:"totalGasUsed"`
}

// TxResult 交易结果
type TxResult struct {
    Success   bool   `json:"success"`
    TxHash    string `json:"txHash"`
    GasUsed   uint64 `json:"gasUsed"`
    GasPrice  string `json:"gasPrice"`
    GasFees   string `json:"gasFees"`
    Error     string `json:"error,omitempty"`
    Revert    string `json:"revert,omitempty"`
}
```

---

## 第九十八节：主入口与协调器

### Bot 主程序

```go
package main

import (
    "context"
    "flag"
    "fmt"
    "math/big"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/ethereum/go-ethereum/crypto"
)

func main() {
    // 解析命令行参数
    configPath := flag.String("config", "configs/config.yaml", "配置文件路径")
    flag.Parse()

    // 加载配置
    config, err := LoadConfig(*configPath)
    if err != nil {
        fmt.Printf("加载配置失败: %v\n", err)
        os.Exit(1)
    }

    // 创建上下文
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // 初始化机器人
    bot, err := NewArbitrageBot(ctx, config)
    if err != nil {
        fmt.Printf("初始化机器人失败: %v\n", err)
        os.Exit(1)
    }

    // 启动
    if err := bot.Start(ctx); err != nil {
        fmt.Printf("启动失败: %v\n", err)
        os.Exit(1)
    }

    fmt.Println("套利机器人已启动...")

    // 等待退出信号
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    <-sigCh

    fmt.Println("正在关闭...")
    cancel()
    bot.Stop()
    fmt.Println("已关闭")
}

// ArbitrageBot 套利机器人
type ArbitrageBot struct {
    config          *Config
    chainClient     *ChainClient
    poolManager     *PoolManager
    strategies      []Strategy
    bundleBuilder   *BundleBuilder
    flashbots       *FlashbotsSubmitter
    executor        *Executor

    // 状态
    running         bool
    lastProfit      *big.Int
    totalProfit     *big.Int
    executionCount  int
}

// NewArbitrageBot 创建机器人
func NewArbitrageBot(ctx context.Context, config *Config) (*ArbitrageBot, error) {
    // 初始化链客户端
    chainClient, err := NewChainClient(
        ctx,
        config.Chain.RPCURL,
        config.Chain.WSURL,
        config.Execution.PrivateKey,
    )
    if err != nil {
        return nil, fmt.Errorf("初始化链客户端失败: %w", err)
    }

    // 初始化池管理器
    poolManager := NewPoolManager(chainClient)

    // 加载池
    if err := loadPools(ctx, config, poolManager); err != nil {
        return nil, fmt.Errorf("加载池失败: %w", err)
    }

    // 初始化价格预言机
    priceOracle := NewChainlinkOracle(chainClient)

    // 初始化策略
    var strategies []Strategy

    if config.Strategy.EnableTriangular {
        triangular := NewTriangularArbitrage(
            poolManager,
            priceOracle,
            &TriangularConfig{
                BaseTokens:   getBaseTokenAddresses(config),
                MinProfitUSD: config.Strategy.MinProfitUSD,
                MaxInputUSD:  config.Strategy.MaxTradeUSD,
                GasPrice:     big.NewInt(config.Strategy.MaxGasPrice * 1e9),
            },
        )
        strategies = append(strategies, triangular)
    }

    if config.Strategy.EnableCrossDEX {
        crossDEX := NewCrossDEXArbitrage(
            poolManager,
            priceOracle,
            &CrossDEXConfig{
                TokenPairs:   getTokenPairs(config),
                MinProfitUSD: config.Strategy.MinProfitUSD,
                MaxInputUSD:  config.Strategy.MaxTradeUSD,
                GasPrice:     big.NewInt(config.Strategy.MaxGasPrice * 1e9),
            },
        )
        strategies = append(strategies, crossDEX)
    }

    // 初始化私钥
    privateKey, err := crypto.HexToECDSA(config.Execution.PrivateKey)
    if err != nil {
        return nil, fmt.Errorf("解析私钥失败: %w", err)
    }

    flashbotsKey, err := crypto.HexToECDSA(config.Flashbots.SigningKey)
    if err != nil {
        return nil, fmt.Errorf("解析 Flashbots 密钥失败: %w", err)
    }

    // 初始化 Bundle 构建器
    bundleBuilder := NewBundleBuilder(
        big.NewInt(config.Chain.ChainID),
        privateKey,
        common.HexToAddress(config.Execution.ArbitrageContract),
    )

    // 初始化 Flashbots 提交器
    flashbots := NewFlashbotsSubmitter(config.Flashbots.RelayURL, flashbotsKey)

    // 初始化执行器
    executor := NewExecutor(chainClient, bundleBuilder, flashbots, config)

    return &ArbitrageBot{
        config:        config,
        chainClient:   chainClient,
        poolManager:   poolManager,
        strategies:    strategies,
        bundleBuilder: bundleBuilder,
        flashbots:     flashbots,
        executor:      executor,
        totalProfit:   big.NewInt(0),
    }, nil
}

// Start 启动机器人
func (bot *ArbitrageBot) Start(ctx context.Context) error {
    bot.running = true

    // 订阅新区块
    headers, err := bot.chainClient.SubscribeNewBlocks(ctx)
    if err != nil {
        return fmt.Errorf("订阅区块失败: %w", err)
    }

    // 启动主循环
    go bot.mainLoop(ctx, headers)

    // 启动监控
    go bot.monitorLoop(ctx)

    return nil
}

// mainLoop 主循环
func (bot *ArbitrageBot) mainLoop(ctx context.Context, headers <-chan *types.Header) {
    ticker := time.NewTicker(bot.config.Strategy.ScanInterval)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case header := <-headers:
            // 新区块到达，刷新池状态
            bot.onNewBlock(ctx, header)
        case <-ticker.C:
            // 定时扫描机会
            bot.scanAndExecute(ctx)
        }
    }
}

// onNewBlock 处理新区块
func (bot *ArbitrageBot) onNewBlock(ctx context.Context, header *types.Header) {
    // 刷新池状态
    if err := bot.poolManager.RefreshAll(ctx); err != nil {
        fmt.Printf("刷新池状态失败: %v\n", err)
    }
}

// scanAndExecute 扫描并执行
func (bot *ArbitrageBot) scanAndExecute(ctx context.Context) {
    // 寻找机会
    var allOpportunities []*Opportunity

    for _, strategy := range bot.strategies {
        opps, err := strategy.FindOpportunities(ctx)
        if err != nil {
            continue
        }
        allOpportunities = append(allOpportunities, opps...)
    }

    if len(allOpportunities) == 0 {
        return
    }

    // 选择最佳机会
    bestOpp := selectBestOpportunity(allOpportunities)
    if bestOpp == nil {
        return
    }

    // 执行
    result, err := bot.executor.Execute(ctx, bestOpp)
    if err != nil {
        fmt.Printf("执行失败: %v\n", err)
        return
    }

    if result.Success {
        bot.totalProfit.Add(bot.totalProfit, result.ActualProfit)
        bot.executionCount++
        fmt.Printf("执行成功! 利润: %s, 交易: %s\n",
            result.ActualProfit.String(),
            result.TxHash.Hex())
    }
}

// selectBestOpportunity 选择最佳机会
func selectBestOpportunity(opps []*Opportunity) *Opportunity {
    if len(opps) == 0 {
        return nil
    }

    // 按净利润排序，选择最高的
    best := opps[0]
    for _, opp := range opps[1:] {
        if opp.NetProfit.Cmp(best.NetProfit) > 0 {
            best = opp
        }
    }

    return best
}

// monitorLoop 监控循环
func (bot *ArbitrageBot) monitorLoop(ctx context.Context) {
    ticker := time.NewTicker(bot.config.Monitoring.ProfitReportInterval)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            bot.printStats()
        }
    }
}

// printStats 打印统计
func (bot *ArbitrageBot) printStats() {
    fmt.Printf("\n=== 统计报告 ===\n")
    fmt.Printf("总利润: %s wei\n", bot.totalProfit.String())
    fmt.Printf("执行次数: %d\n", bot.executionCount)
    fmt.Printf("================\n\n")
}

// Stop 停止机器人
func (bot *ArbitrageBot) Stop() {
    bot.running = false
    bot.chainClient.Close()
}

// 辅助函数
func loadPools(ctx context.Context, config *Config, pm *PoolManager) error {
    // 从链上加载池信息
    // 实现省略
    return nil
}

func getBaseTokenAddresses(config *Config) []common.Address {
    var addrs []common.Address
    for _, token := range config.Tokens {
        if token.IsBase {
            addrs = append(addrs, common.HexToAddress(token.Address))
        }
    }
    return addrs
}

func getTokenPairs(config *Config) [][2]common.Address {
    // 生成所有代币对
    var pairs [][2]common.Address
    for i := 0; i < len(config.Tokens); i++ {
        for j := i + 1; j < len(config.Tokens); j++ {
            pairs = append(pairs, [2]common.Address{
                common.HexToAddress(config.Tokens[i].Address),
                common.HexToAddress(config.Tokens[j].Address),
            })
        }
    }
    return pairs
}
```

### 执行器

```go
package main

import (
    "context"
    "fmt"
    "math/big"
)

// Executor 执行器
type Executor struct {
    chainClient   *ChainClient
    bundleBuilder *BundleBuilder
    flashbots     *FlashbotsSubmitter
    config        *Config
}

// NewExecutor 创建执行器
func NewExecutor(
    chainClient *ChainClient,
    bundleBuilder *BundleBuilder,
    flashbots *FlashbotsSubmitter,
    config *Config,
) *Executor {
    return &Executor{
        chainClient:   chainClient,
        bundleBuilder: bundleBuilder,
        flashbots:     flashbots,
        config:        config,
    }
}

// Execute 执行套利
func (e *Executor) Execute(ctx context.Context, opp *Opportunity) (*ExecutionResult, error) {
    // 1. 构建 Bundle
    nonce := e.chainClient.GetNonce()
    baseFee := e.chainClient.GetBaseFee()
    priorityFee := big.NewInt(2e9) // 2 Gwei

    bundle, err := e.bundleBuilder.BuildArbitrageBundle(
        ctx,
        opp,
        nonce,
        baseFee,
        priorityFee,
    )
    if err != nil {
        return nil, fmt.Errorf("构建 Bundle 失败: %w", err)
    }

    // 设置目标区块
    bundle.BlockNumber = e.chainClient.GetBlockNumber() + 1

    // 2. 模拟（如果启用）
    if e.config.Execution.SimulateFirst {
        simResult, err := e.flashbots.SimulateBundle(ctx, bundle)
        if err != nil {
            return nil, fmt.Errorf("模拟失败: %w", err)
        }

        if !simResult.Success {
            return &ExecutionResult{
                Success: false,
                Error:   "模拟失败",
            }, nil
        }

        // 检查利润是否符合预期
        if simResult.CoinbaseDiff.Cmp(opp.NetProfit) < 0 {
            return &ExecutionResult{
                Success: false,
                Error:   "模拟利润低于预期",
            }, nil
        }
    }

    // 3. 提交
    if e.config.Execution.UseFlashbots {
        return e.submitViaFlashbots(ctx, bundle)
    }

    return e.submitDirect(ctx, bundle)
}

// submitViaFlashbots 通过 Flashbots 提交
func (e *Executor) submitViaFlashbots(ctx context.Context, bundle *Bundle) (*ExecutionResult, error) {
    // 向多个区块提交
    for i := 0; i < e.config.Execution.MaxRetries; i++ {
        bundle.BlockNumber = e.chainClient.GetBlockNumber() + uint64(i) + 1

        result, err := e.flashbots.SubmitBundle(ctx, bundle)
        if err != nil {
            continue
        }

        if result.Success {
            e.chainClient.IncrementNonce()
            return &ExecutionResult{
                Success:    true,
                TxHash:     result.BundleHash,
                BlockNumber: bundle.BlockNumber,
            }, nil
        }
    }

    return &ExecutionResult{
        Success: false,
        Error:   "所有重试都失败",
    }, nil
}

// submitDirect 直接提交
func (e *Executor) submitDirect(ctx context.Context, bundle *Bundle) (*ExecutionResult, error) {
    if len(bundle.Transactions) == 0 {
        return nil, fmt.Errorf("Bundle 为空")
    }

    tx := bundle.Transactions[0]

    if err := e.chainClient.SendTransaction(ctx, tx); err != nil {
        return &ExecutionResult{
            Success: false,
            Error:   err.Error(),
        }, nil
    }

    return &ExecutionResult{
        Success: true,
        TxHash:  tx.Hash(),
    }, nil
}
```

---

## 第九十九节：套利合约

### Solidity 合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

interface IUniswapV2Pair {
    function swap(uint amount0Out, uint amount1Out, address to, bytes calldata data) external;
    function getReserves() external view returns (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast);
    function token0() external view returns (address);
    function token1() external view returns (address);
}

interface IUniswapV3Pool {
    function swap(
        address recipient,
        bool zeroForOne,
        int256 amountSpecified,
        uint160 sqrtPriceLimitX96,
        bytes calldata data
    ) external returns (int256 amount0, int256 amount1);
}

interface IFlashLoanReceiver {
    function executeOperation(
        address[] calldata assets,
        uint256[] calldata amounts,
        uint256[] calldata premiums,
        address initiator,
        bytes calldata params
    ) external returns (bool);
}

interface IPool {
    function flashLoan(
        address receiverAddress,
        address[] calldata assets,
        uint256[] calldata amounts,
        uint256[] calldata modes,
        address onBehalfOf,
        bytes calldata params,
        uint16 referralCode
    ) external;
}

/// @title ArbitrageBot
/// @notice 套利机器人合约
contract ArbitrageBot is Ownable, ReentrancyGuard, IFlashLoanReceiver {
    using SafeERC20 for IERC20;

    // Aave V3 Pool
    IPool public immutable AAVE_POOL;

    // 授权的执行者
    mapping(address => bool) public authorizedExecutors;

    // 最小利润要求（防止无意义交易）
    uint256 public minProfit;

    // V3 回调验证
    bytes32 public constant CALLBACK_SUCCESS = keccak256("ERC3156FlashBorrower.onFlashLoan");

    // 事件
    event ArbitrageExecuted(
        address indexed token,
        uint256 inputAmount,
        uint256 outputAmount,
        uint256 profit
    );
    event ExecutorUpdated(address indexed executor, bool authorized);
    event MinProfitUpdated(uint256 newMinProfit);

    // 修饰器
    modifier onlyAuthorized() {
        require(authorizedExecutors[msg.sender] || msg.sender == owner(), "Not authorized");
        _;
    }

    constructor(address _aavePool, uint256 _minProfit) {
        AAVE_POOL = IPool(_aavePool);
        minProfit = _minProfit;
        authorizedExecutors[msg.sender] = true;
    }

    /// @notice 执行三角套利
    /// @param pools 池地址数组
    /// @param tokens 代币地址数组
    /// @param amountIn 输入金额
    /// @param minAmountOut 最小输出金额
    function executeTriangular(
        address[] calldata pools,
        address[] calldata tokens,
        uint256 amountIn,
        uint256 minAmountOut
    ) external onlyAuthorized nonReentrant {
        require(pools.length >= 2 && pools.length == tokens.length - 1, "Invalid path");

        address tokenIn = tokens[0];
        address tokenOut = tokens[tokens.length - 1];
        require(tokenIn == tokenOut, "Must be circular path");

        uint256 balanceBefore = IERC20(tokenIn).balanceOf(address(this));

        // 执行交换路径
        uint256 currentAmount = amountIn;
        for (uint256 i = 0; i < pools.length; i++) {
            currentAmount = _swapV2(
                pools[i],
                tokens[i],
                tokens[i + 1],
                currentAmount
            );
        }

        uint256 balanceAfter = IERC20(tokenIn).balanceOf(address(this));
        uint256 profit = balanceAfter - balanceBefore;

        require(currentAmount >= minAmountOut, "Insufficient output");
        require(profit >= minProfit, "Profit too low");

        emit ArbitrageExecuted(tokenIn, amountIn, currentAmount, profit);
    }

    /// @notice 执行跨 DEX 套利
    /// @param pool0 买入池
    /// @param pool1 卖出池
    /// @param tokenIn 输入代币
    /// @param tokenMid 中间代币
    /// @param amountIn 输入金额
    /// @param minAmountOut 最小输出金额
    function executeCrossDEX(
        address pool0,
        address pool1,
        address tokenIn,
        address tokenMid,
        uint256 amountIn,
        uint256 minAmountOut
    ) external onlyAuthorized nonReentrant {
        uint256 balanceBefore = IERC20(tokenIn).balanceOf(address(this));

        // Step 1: 在 pool0 买入 tokenMid
        uint256 midAmount = _swapV2(pool0, tokenIn, tokenMid, amountIn);

        // Step 2: 在 pool1 卖出 tokenMid
        uint256 outputAmount = _swapV2(pool1, tokenMid, tokenIn, midAmount);

        uint256 balanceAfter = IERC20(tokenIn).balanceOf(address(this));
        uint256 profit = balanceAfter - balanceBefore;

        require(outputAmount >= minAmountOut, "Insufficient output");
        require(profit >= minProfit, "Profit too low");

        emit ArbitrageExecuted(tokenIn, amountIn, outputAmount, profit);
    }

    /// @notice 执行闪电贷套利
    /// @param token 借贷代币
    /// @param amount 借贷金额
    /// @param swapData 交换数据
    function executeFlashloan(
        address token,
        uint256 amount,
        bytes calldata swapData
    ) external onlyAuthorized nonReentrant {
        address[] memory assets = new address[](1);
        assets[0] = token;

        uint256[] memory amounts = new uint256[](1);
        amounts[0] = amount;

        uint256[] memory modes = new uint256[](1);
        modes[0] = 0; // 不产生债务

        AAVE_POOL.flashLoan(
            address(this),
            assets,
            amounts,
            modes,
            address(this),
            swapData,
            0
        );
    }

    /// @notice Aave 闪电贷回调
    function executeOperation(
        address[] calldata assets,
        uint256[] calldata amounts,
        uint256[] calldata premiums,
        address initiator,
        bytes calldata params
    ) external override returns (bool) {
        require(msg.sender == address(AAVE_POOL), "Invalid caller");
        require(initiator == address(this), "Invalid initiator");

        // 解码并执行交换
        (
            address[] memory pools,
            address[] memory tokens,
            bool[] memory isV3
        ) = abi.decode(params, (address[], address[], bool[]));

        uint256 currentAmount = amounts[0];
        for (uint256 i = 0; i < pools.length; i++) {
            if (isV3[i]) {
                currentAmount = _swapV3(
                    pools[i],
                    tokens[i],
                    tokens[i + 1],
                    currentAmount
                );
            } else {
                currentAmount = _swapV2(
                    pools[i],
                    tokens[i],
                    tokens[i + 1],
                    currentAmount
                );
            }
        }

        // 计算需要偿还的金额
        uint256 amountOwed = amounts[0] + premiums[0];
        require(currentAmount >= amountOwed, "Insufficient profit");

        // 授权还款
        IERC20(assets[0]).safeApprove(address(AAVE_POOL), amountOwed);

        return true;
    }

    /// @notice V2 交换
    function _swapV2(
        address pool,
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) internal returns (uint256 amountOut) {
        IUniswapV2Pair pair = IUniswapV2Pair(pool);

        (uint112 reserve0, uint112 reserve1,) = pair.getReserves();
        address token0 = pair.token0();

        bool isToken0 = tokenIn == token0;
        (uint112 reserveIn, uint112 reserveOut) = isToken0
            ? (reserve0, reserve1)
            : (reserve1, reserve0);

        // 计算输出
        uint256 amountInWithFee = amountIn * 997;
        amountOut = (amountInWithFee * reserveOut) / (reserveIn * 1000 + amountInWithFee);

        // 转移代币到池
        IERC20(tokenIn).safeTransfer(pool, amountIn);

        // 执行交换
        (uint256 amount0Out, uint256 amount1Out) = isToken0
            ? (uint256(0), amountOut)
            : (amountOut, uint256(0));

        pair.swap(amount0Out, amount1Out, address(this), "");
    }

    /// @notice V3 交换
    function _swapV3(
        address pool,
        address tokenIn,
        address tokenOut,
        uint256 amountIn
    ) internal returns (uint256 amountOut) {
        IUniswapV3Pool v3Pool = IUniswapV3Pool(pool);

        bool zeroForOne = tokenIn < tokenOut;

        // 转移代币
        IERC20(tokenIn).safeTransfer(pool, amountIn);

        // 执行交换
        (int256 amount0, int256 amount1) = v3Pool.swap(
            address(this),
            zeroForOne,
            int256(amountIn),
            zeroForOne ? 4295128740 : 1461446703485210103287273052203988822378723970341,
            ""
        );

        amountOut = uint256(zeroForOne ? -amount1 : -amount0);
    }

    /// @notice 设置授权执行者
    function setExecutor(address executor, bool authorized) external onlyOwner {
        authorizedExecutors[executor] = authorized;
        emit ExecutorUpdated(executor, authorized);
    }

    /// @notice 设置最小利润
    function setMinProfit(uint256 _minProfit) external onlyOwner {
        minProfit = _minProfit;
        emit MinProfitUpdated(_minProfit);
    }

    /// @notice 提取代币
    function withdraw(address token, uint256 amount) external onlyOwner {
        IERC20(token).safeTransfer(msg.sender, amount);
    }

    /// @notice 提取 ETH
    function withdrawETH() external onlyOwner {
        payable(msg.sender).transfer(address(this).balance);
    }

    receive() external payable {}
}
```

---

## 本文总结

本文详细介绍了一个完整的套利机器人项目实现，主要涵盖：

### 架构层次

```
┌─────────────────────────────────────────────────────────────┐
│                       完整套利机器人                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   配置层    │───▶│   数据层    │───▶│   策略层    │     │
│  │  Config     │    │  Chain/Pool │    │  Strategy   │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                              │               │
│                                              ▼               │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   监控层    │◀───│   执行层    │◀───│   决策层    │     │
│  │  Monitor    │    │  Executor   │    │  Selector   │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 核心组件

| 组件 | 功能 | 关键实现 |
|-----|------|---------|
| ChainClient | 链交互 | RPC/WS 连接，交易构建签名 |
| PoolManager | 池状态管理 | 储备同步，价格计算 |
| Strategy | 套利策略 | 三角套利，跨 DEX 套利 |
| BundleBuilder | Bundle 构建 | calldata 编码，交易打包 |
| FlashbotsSubmitter | Bundle 提交 | 签名认证，模拟验证 |
| ArbitrageContract | 链上执行 | swap 执行，闪电贷回调 |

### 执行流程

```
1. 数据采集
   └─ 订阅区块 → 刷新池状态 → 更新价格

2. 机会发现
   └─ 遍历策略 → 计算利润 → 筛选机会

3. 交易构建
   └─ 构建 calldata → 估算 gas → 签名交易

4. 模拟验证
   └─ eth_callBundle → 验证利润 → 检查回滚

5. 提交执行
   └─ eth_sendBundle → 等待确认 → 记录结果
```

### 最佳实践

1. **性能优化**
   - 使用 WebSocket 订阅减少延迟
   - 并发刷新池状态
   - 缓存计算结果

2. **安全措施**
   - 模拟后再提交
   - 设置滑点保护
   - 限制最大损失

3. **运维建议**
   - 监控利润和成功率
   - 自动告警机制
   - 日志分级记录

### 下一步学习

- **安全与漏洞**：深入了解 DeFi 安全和常见攻击模式

---

## 附录 A：测试框架

完善的测试是套利机器人稳定运行的基础。本附录提供单元测试、集成测试和模拟测试的完整实现。

### A.1 测试工具和 Mock

```go
package testing

import (
    "context"
    "math/big"
    "sync"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
)

// MockPool 模拟流动性池
type MockPool struct {
    Address  common.Address
    Token0   common.Address
    Token1   common.Address
    Reserve0 *big.Int
    Reserve1 *big.Int
    Fee      uint64 // basis points (30 = 0.3%)
}

// NewMockPool 创建模拟池
func NewMockPool(token0, token1 common.Address, reserve0, reserve1 *big.Int, fee uint64) *MockPool {
    return &MockPool{
        Address:  common.HexToAddress("0x" + token0.Hex()[2:10] + token1.Hex()[2:10]),
        Token0:   token0,
        Token1:   token1,
        Reserve0: new(big.Int).Set(reserve0),
        Reserve1: new(big.Int).Set(reserve1),
        Fee:      fee,
    }
}

// GetAmountOut 计算输出金额
func (p *MockPool) GetAmountOut(amountIn *big.Int, zeroForOne bool) *big.Int {
    var reserveIn, reserveOut *big.Int
    if zeroForOne {
        reserveIn = p.Reserve0
        reserveOut = p.Reserve1
    } else {
        reserveIn = p.Reserve1
        reserveOut = p.Reserve0
    }

    // amountOut = (amountIn * (10000 - fee) * reserveOut) / (reserveIn * 10000 + amountIn * (10000 - fee))
    amountInWithFee := new(big.Int).Mul(amountIn, big.NewInt(int64(10000-p.Fee)))
    numerator := new(big.Int).Mul(amountInWithFee, reserveOut)
    denominator := new(big.Int).Add(
        new(big.Int).Mul(reserveIn, big.NewInt(10000)),
        amountInWithFee,
    )

    return new(big.Int).Div(numerator, denominator)
}

// MockPoolManager 模拟池管理器
type MockPoolManager struct {
    pools map[common.Address]*MockPool
    mu    sync.RWMutex
}

func NewMockPoolManager() *MockPoolManager {
    return &MockPoolManager{
        pools: make(map[common.Address]*MockPool),
    }
}

func (m *MockPoolManager) AddPool(pool *MockPool) {
    m.mu.Lock()
    defer m.mu.Unlock()
    m.pools[pool.Address] = pool
}

func (m *MockPoolManager) GetPool(address common.Address) *MockPool {
    m.mu.RLock()
    defer m.mu.RUnlock()
    return m.pools[address]
}

func (m *MockPoolManager) GetReserves(address common.Address) (*big.Int, *big.Int, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()

    pool := m.pools[address]
    if pool == nil {
        return nil, nil, ErrPoolNotFound
    }
    return new(big.Int).Set(pool.Reserve0), new(big.Int).Set(pool.Reserve1), nil
}

func (m *MockPoolManager) CalculateV2Output(address common.Address, amountIn *big.Int, zeroForOne bool) (*big.Int, error) {
    m.mu.RLock()
    defer m.mu.RUnlock()

    pool := m.pools[address]
    if pool == nil {
        return nil, ErrPoolNotFound
    }
    return pool.GetAmountOut(amountIn, zeroForOne), nil
}

// MockChainClient 模拟链客户端
type MockChainClient struct {
    blockNumber  uint64
    baseFee      *big.Int
    pendingNonce uint64
    balance      *big.Int
    callResults  map[string][]byte
    sendTxResult *types.Transaction
    mu           sync.RWMutex
}

func NewMockChainClient() *MockChainClient {
    return &MockChainClient{
        blockNumber:  18000000,
        baseFee:      big.NewInt(30e9), // 30 gwei
        pendingNonce: 0,
        balance:      new(big.Int).Mul(big.NewInt(100), big.NewInt(1e18)), // 100 ETH
        callResults:  make(map[string][]byte),
    }
}

func (c *MockChainClient) BlockNumber(ctx context.Context) (uint64, error) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.blockNumber, nil
}

func (c *MockChainClient) SetBlockNumber(n uint64) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.blockNumber = n
}

func (c *MockChainClient) SuggestGasPrice(ctx context.Context) (*big.Int, error) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return new(big.Int).Set(c.baseFee), nil
}

func (c *MockChainClient) PendingNonceAt(ctx context.Context, account common.Address) (uint64, error) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.pendingNonce, nil
}

func (c *MockChainClient) BalanceAt(ctx context.Context, account common.Address, blockNumber *big.Int) (*big.Int, error) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return new(big.Int).Set(c.balance), nil
}

func (c *MockChainClient) SetCallResult(callData string, result []byte) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.callResults[callData] = result
}

// MockFlashbotsClient 模拟 Flashbots 客户端
type MockFlashbotsClient struct {
    simulateResult *SimulateResult
    sendBundleHash common.Hash
    bundleIncluded bool
}

type SimulateResult struct {
    Success     bool
    TotalProfit *big.Int
    GasUsed     uint64
    Error       string
}

func NewMockFlashbotsClient() *MockFlashbotsClient {
    return &MockFlashbotsClient{
        simulateResult: &SimulateResult{
            Success:     true,
            TotalProfit: big.NewInt(1e16), // 0.01 ETH profit
            GasUsed:     300000,
        },
        sendBundleHash: common.HexToHash("0x1234567890abcdef"),
        bundleIncluded: true,
    }
}

func (c *MockFlashbotsClient) SetSimulateResult(result *SimulateResult) {
    c.simulateResult = result
}

func (c *MockFlashbotsClient) SimulateBundle(ctx context.Context, bundle *Bundle, blockNumber uint64) (*SimulateResult, error) {
    return c.simulateResult, nil
}

func (c *MockFlashbotsClient) SendBundle(ctx context.Context, bundle *Bundle, blockNumber uint64) (common.Hash, error) {
    return c.sendBundleHash, nil
}

func (c *MockFlashbotsClient) GetBundleStats(ctx context.Context, bundleHash common.Hash, blockNumber uint64) (*BundleStats, error) {
    return &BundleStats{
        IsIncluded: c.bundleIncluded,
    }, nil
}

// TestHelper 测试辅助函数
type TestHelper struct{}

func (h *TestHelper) CreateTokenAddresses(count int) []common.Address {
    addresses := make([]common.Address, count)
    for i := 0; i < count; i++ {
        addresses[i] = common.HexToAddress("0x" + string(rune('A'+i)) + "000000000000000000000000000000000000000")
    }
    return addresses
}

func (h *TestHelper) CreateTriangularPools(tokens []common.Address) []*MockPool {
    // 创建三角套利所需的三个池
    // A-B, B-C, C-A
    reserve := new(big.Int).Mul(big.NewInt(1000000), big.NewInt(1e18))
    return []*MockPool{
        NewMockPool(tokens[0], tokens[1], reserve, reserve, 30),
        NewMockPool(tokens[1], tokens[2], reserve, reserve, 30),
        NewMockPool(tokens[2], tokens[0], reserve, reserve, 30),
    }
}

func (h *TestHelper) CreateArbitrageOpportunity(pools []*MockPool) {
    // 调整池储备以创建套利机会
    // 使 A-B 池的 A 价格便宜
    pools[0].Reserve0 = new(big.Int).Mul(big.NewInt(1100000), big.NewInt(1e18)) // 多 A
    pools[0].Reserve1 = new(big.Int).Mul(big.NewInt(900000), big.NewInt(1e18))  // 少 B
}
```

### A.2 单元测试

```go
package arbitrage_test

import (
    "context"
    "math/big"
    "testing"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

// TestGetAmountOut 测试 AMM 输出计算
func TestGetAmountOut(t *testing.T) {
    tests := []struct {
        name       string
        reserveIn  *big.Int
        reserveOut *big.Int
        amountIn   *big.Int
        fee        uint64 // basis points
        expected   *big.Int
    }{
        {
            name:       "standard swap with 0.3% fee",
            reserveIn:  big.NewInt(1000e18),
            reserveOut: big.NewInt(1000e18),
            amountIn:   big.NewInt(1e18),
            fee:        30,
            expected:   big.NewInt(996006981039903216), // ~0.996 tokens
        },
        {
            name:       "large swap with high impact",
            reserveIn:  big.NewInt(1000e18),
            reserveOut: big.NewInt(1000e18),
            amountIn:   big.NewInt(100e18),
            fee:        30,
            expected:   big.NewInt(90661089388014913), // ~90.66 tokens (9% slippage)
        },
        {
            name:       "swap with 1% fee",
            reserveIn:  big.NewInt(1000e18),
            reserveOut: big.NewInt(1000e18),
            amountIn:   big.NewInt(1e18),
            fee:        100,
            expected:   big.NewInt(989020869339354039),
        },
        {
            name:       "tiny swap",
            reserveIn:  big.NewInt(1000e18),
            reserveOut: big.NewInt(1000e18),
            amountIn:   big.NewInt(1e15), // 0.001 token
            fee:        30,
            expected:   big.NewInt(996999006980026), // ~0.000997 tokens
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            token0 := common.HexToAddress("0xA")
            token1 := common.HexToAddress("0xB")

            pool := NewMockPool(token0, token1, tt.reserveIn, tt.reserveOut, tt.fee)
            result := pool.GetAmountOut(tt.amountIn, true)

            // 允许 0.1% 误差
            diff := new(big.Int).Sub(result, tt.expected)
            diff.Abs(diff)
            maxDiff := new(big.Int).Div(tt.expected, big.NewInt(1000))

            assert.True(t, diff.Cmp(maxDiff) <= 0,
                "expected %s, got %s (diff: %s)",
                tt.expected.String(), result.String(), diff.String())
        })
    }
}

// TestTriangularArbitrageDetection 测试三角套利检测
func TestTriangularArbitrageDetection(t *testing.T) {
    helper := &TestHelper{}

    tests := []struct {
        name              string
        setupPools        func([]*MockPool)
        expectOpportunity bool
        minProfit         *big.Int
    }{
        {
            name: "no arbitrage - balanced pools",
            setupPools: func(pools []*MockPool) {
                // 所有池平衡，无套利机会
            },
            expectOpportunity: false,
        },
        {
            name: "profitable triangular arbitrage",
            setupPools: func(pools []*MockPool) {
                // A-B: A 便宜
                pools[0].Reserve0 = big.NewInt(1100e18) // 更多 A
                pools[0].Reserve1 = big.NewInt(900e18)  // 更少 B
                // B-C: 平衡
                // C-A: 平衡
            },
            expectOpportunity: true,
            minProfit:         big.NewInt(1e15), // 至少 0.001 ETH 利润
        },
        {
            name: "unprofitable after fees",
            setupPools: func(pools []*MockPool) {
                // 微小价差，不足以覆盖手续费
                pools[0].Reserve0 = big.NewInt(1001e18)
                pools[0].Reserve1 = big.NewInt(999e18)
            },
            expectOpportunity: false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            tokens := helper.CreateTokenAddresses(3)
            pools := helper.CreateTriangularPools(tokens)

            if tt.setupPools != nil {
                tt.setupPools(pools)
            }

            // 创建池管理器
            poolManager := NewMockPoolManager()
            for _, pool := range pools {
                poolManager.AddPool(pool)
            }

            // 创建三角套利检测器
            strategy := NewTriangularArbitrage(&TriangularConfig{
                BaseTokens:   []common.Address{tokens[0]},
                MinProfitUSD: 1.0,
                MaxInputUSD:  10000,
                GasPrice:     big.NewInt(30e9),
            }, poolManager, nil)

            // 检测机会
            ctx := context.Background()
            opportunities, err := strategy.FindOpportunities(ctx, pools)
            require.NoError(t, err)

            if tt.expectOpportunity {
                require.NotEmpty(t, opportunities, "expected to find opportunities")
                assert.True(t, opportunities[0].NetProfit.Cmp(tt.minProfit) >= 0,
                    "profit %s below minimum %s",
                    opportunities[0].NetProfit.String(), tt.minProfit.String())
            } else {
                assert.Empty(t, opportunities, "expected no opportunities")
            }
        })
    }
}

// TestOptimalInputCalculation 测试最优输入计算
func TestOptimalInputCalculation(t *testing.T) {
    helper := &TestHelper{}
    tokens := helper.CreateTokenAddresses(3)
    pools := helper.CreateTriangularPools(tokens)

    // 创建明显的套利机会
    pools[0].Reserve0 = big.NewInt(1200e18)
    pools[0].Reserve1 = big.NewInt(800e18)

    poolManager := NewMockPoolManager()
    for _, pool := range pools {
        poolManager.AddPool(pool)
    }

    strategy := NewTriangularArbitrage(&TriangularConfig{
        BaseTokens:   []common.Address{tokens[0]},
        MinProfitUSD: 0,
        MaxInputUSD:  100000,
        GasPrice:     big.NewInt(30e9),
    }, poolManager, nil)

    // 计算最优输入
    optimalInput := strategy.calculateOptimalInput(
        pools[0], pools[1], pools[2],
        tokens[0], tokens[1], tokens[2],
    )

    require.NotNil(t, optimalInput)
    assert.True(t, optimalInput.Sign() > 0, "optimal input should be positive")

    // 验证最优性：在最优点附近，利润应该更低
    testProfit := func(input *big.Int) *big.Int {
        output, _ := strategy.simulateSwaps(input, tokens[0], pools[0], pools[1], pools[2], tokens[1], tokens[2])
        if output == nil {
            return big.NewInt(0)
        }
        return new(big.Int).Sub(output, input)
    }

    optimalProfit := testProfit(optimalInput)
    delta := big.NewInt(1e17) // 0.1 ETH

    // 少一点
    lessInput := new(big.Int).Sub(optimalInput, delta)
    lessProfit := testProfit(lessInput)

    // 多一点
    moreInput := new(big.Int).Add(optimalInput, delta)
    moreProfit := testProfit(moreInput)

    assert.True(t, optimalProfit.Cmp(lessProfit) >= 0,
        "profit at optimal (%s) should be >= profit at less (%s)",
        optimalProfit.String(), lessProfit.String())
    assert.True(t, optimalProfit.Cmp(moreProfit) >= 0,
        "profit at optimal (%s) should be >= profit at more (%s)",
        optimalProfit.String(), moreProfit.String())
}

// TestGasEstimation 测试 Gas 估算
func TestGasEstimation(t *testing.T) {
    tests := []struct {
        name        string
        swapCount   int
        isFlashLoan bool
        minGas      uint64
        maxGas      uint64
    }{
        {
            name:        "simple two-hop swap",
            swapCount:   2,
            isFlashLoan: false,
            minGas:      150000,
            maxGas:      250000,
        },
        {
            name:        "triangular arbitrage",
            swapCount:   3,
            isFlashLoan: false,
            minGas:      250000,
            maxGas:      400000,
        },
        {
            name:        "triangular with flash loan",
            swapCount:   3,
            isFlashLoan: true,
            minGas:      300000,
            maxGas:      500000,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            estimator := NewGasEstimator()
            gas := estimator.EstimateArbitrage(tt.swapCount, tt.isFlashLoan)

            assert.True(t, gas >= tt.minGas && gas <= tt.maxGas,
                "gas %d should be between %d and %d",
                gas, tt.minGas, tt.maxGas)
        })
    }
}

// TestBundleBuilder 测试 Bundle 构建
func TestBundleBuilder(t *testing.T) {
    ctx := context.Background()
    mockClient := NewMockChainClient()

    builder := NewBundleBuilder(
        mockClient,
        common.HexToAddress("0x1234567890123456789012345678901234567890"), // contract
        nil, // private key (not needed for test)
    )

    // 创建测试机会
    opportunity := &Opportunity{
        ID:   "test-opp-1",
        Type: OpportunityTriangular,
        Path: []*SwapStep{
            {
                DEX:       "uniswap_v2",
                Pool:      common.HexToAddress("0xPool1"),
                TokenIn:   common.HexToAddress("0xTokenA"),
                TokenOut:  common.HexToAddress("0xTokenB"),
                AmountIn:  big.NewInt(1e18),
                AmountOut: big.NewInt(1e18),
            },
            {
                DEX:       "sushiswap",
                Pool:      common.HexToAddress("0xPool2"),
                TokenIn:   common.HexToAddress("0xTokenB"),
                TokenOut:  common.HexToAddress("0xTokenC"),
                AmountIn:  big.NewInt(1e18),
                AmountOut: big.NewInt(1e18),
            },
            {
                DEX:       "uniswap_v2",
                Pool:      common.HexToAddress("0xPool3"),
                TokenIn:   common.HexToAddress("0xTokenC"),
                TokenOut:  common.HexToAddress("0xTokenA"),
                AmountIn:  big.NewInt(1e18),
                AmountOut: big.NewInt(101e16), // 1.01 ETH output
            },
        },
        InputToken:   common.HexToAddress("0xTokenA"),
        InputAmount:  big.NewInt(1e18),
        OutputAmount: big.NewInt(101e16),
        GasEstimate:  300000,
    }

    bundle, err := builder.BuildBundle(ctx, opportunity, 18000001)
    require.NoError(t, err)
    require.NotNil(t, bundle)

    assert.Len(t, bundle.Transactions, 1, "bundle should have 1 transaction")
    assert.Equal(t, uint64(18000001), bundle.BlockNumber)
}

// TestProfitCalculation 测试利润计算
func TestProfitCalculation(t *testing.T) {
    tests := []struct {
        name           string
        inputAmount    *big.Int
        outputAmount   *big.Int
        gasUsed        uint64
        gasPrice       *big.Int
        expectedProfit *big.Int
    }{
        {
            name:           "profitable trade",
            inputAmount:    big.NewInt(1e18),
            outputAmount:   big.NewInt(102e16), // 1.02 ETH
            gasUsed:        300000,
            gasPrice:       big.NewInt(30e9),
            expectedProfit: big.NewInt(11e15), // 0.02 - 0.009 = 0.011 ETH
        },
        {
            name:           "break even after gas",
            inputAmount:    big.NewInt(1e18),
            outputAmount:   big.NewInt(1009e15), // 1.009 ETH
            gasUsed:        300000,
            gasPrice:       big.NewInt(30e9),
            expectedProfit: big.NewInt(0), // 0.009 - 0.009 = 0
        },
        {
            name:           "loss after gas",
            inputAmount:    big.NewInt(1e18),
            outputAmount:   big.NewInt(1005e15), // 1.005 ETH
            gasUsed:        300000,
            gasPrice:       big.NewInt(30e9),
            expectedProfit: big.NewInt(-4e15), // 0.005 - 0.009 = -0.004 ETH
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            calculator := NewProfitCalculator()

            result := calculator.Calculate(
                tt.inputAmount,
                tt.outputAmount,
                tt.gasUsed,
                tt.gasPrice,
            )

            // 允许小误差
            diff := new(big.Int).Sub(result.NetProfit, tt.expectedProfit)
            diff.Abs(diff)
            maxDiff := big.NewInt(1e12) // 0.000001 ETH

            assert.True(t, diff.Cmp(maxDiff) <= 0,
                "expected %s, got %s",
                tt.expectedProfit.String(), result.NetProfit.String())
        })
    }
}
```

### A.3 集成测试

```go
package integration_test

import (
    "context"
    "math/big"
    "sync"
    "testing"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

// TestFullArbitrageFlow 测试完整套利流程
func TestFullArbitrageFlow(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test in short mode")
    }

    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    // 设置测试环境
    helper := &TestHelper{}
    tokens := helper.CreateTokenAddresses(3)
    pools := helper.CreateTriangularPools(tokens)

    // 创建套利机会
    helper.CreateArbitrageOpportunity(pools)

    // 创建模拟组件
    poolManager := NewMockPoolManager()
    for _, pool := range pools {
        poolManager.AddPool(pool)
    }

    chainClient := NewMockChainClient()
    flashbotsClient := NewMockFlashbotsClient()

    // 创建机器人
    bot := NewArbitrageBot(&BotConfig{
        MinProfitUSD:   1.0,
        MaxInputUSD:    10000,
        GasMultiplier:  1.1,
        SimulateFirst:  true,
        SubmitToPublic: false,
    })

    bot.SetPoolManager(poolManager)
    bot.SetChainClient(chainClient)
    bot.SetFlashbotsClient(flashbotsClient)

    // 注册策略
    triangularStrategy := NewTriangularArbitrage(&TriangularConfig{
        BaseTokens:   []common.Address{tokens[0]},
        MinProfitUSD: 1.0,
        MaxInputUSD:  10000,
        GasPrice:     big.NewInt(30e9),
    }, poolManager, nil)

    bot.RegisterStrategy(triangularStrategy)

    // 执行一个周期
    result, err := bot.ExecuteCycle(ctx, pools)
    require.NoError(t, err)

    // 验证结果
    assert.True(t, result.OpportunitiesFound > 0, "should find opportunities")
    assert.True(t, result.SimulationsRun > 0, "should run simulations")

    // 验证模拟成功
    if result.BundlesSent > 0 {
        assert.True(t, result.BundlesIncluded >= 0, "bundles should be tracked")
    }
}

// TestConcurrentPoolUpdates 测试并发池更新
func TestConcurrentPoolUpdates(t *testing.T) {
    poolManager := NewMockPoolManager()

    // 创建多个池
    for i := 0; i < 100; i++ {
        token0 := common.BigToAddress(big.NewInt(int64(i * 2)))
        token1 := common.BigToAddress(big.NewInt(int64(i*2 + 1)))
        reserve := new(big.Int).Mul(big.NewInt(1000000), big.NewInt(1e18))
        pool := NewMockPool(token0, token1, reserve, reserve, 30)
        poolManager.AddPool(pool)
    }

    // 并发读写测试
    var wg sync.WaitGroup
    errChan := make(chan error, 200)

    // 并发写入
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(idx int) {
            defer wg.Done()
            token0 := common.BigToAddress(big.NewInt(int64(idx * 2)))
            token1 := common.BigToAddress(big.NewInt(int64(idx*2 + 1)))
            reserve := new(big.Int).Mul(big.NewInt(int64(1000000+idx)), big.NewInt(1e18))
            pool := NewMockPool(token0, token1, reserve, reserve, 30)
            poolManager.AddPool(pool)
        }(i)
    }

    // 并发读取
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(idx int) {
            defer wg.Done()
            addr := common.BigToAddress(big.NewInt(int64(idx)))
            _, _, err := poolManager.GetReserves(addr)
            if err != nil && err != ErrPoolNotFound {
                errChan <- err
            }
        }(i)
    }

    wg.Wait()
    close(errChan)

    // 检查错误
    for err := range errChan {
        t.Errorf("concurrent operation error: %v", err)
    }
}

// TestStrategyPipeline 测试策略管道
func TestStrategyPipeline(t *testing.T) {
    ctx := context.Background()

    helper := &TestHelper{}
    tokens := helper.CreateTokenAddresses(4)

    // 创建多种类型的池
    pools := []*MockPool{
        // 三角套利池
        NewMockPool(tokens[0], tokens[1], big.NewInt(1000e18), big.NewInt(1000e18), 30),
        NewMockPool(tokens[1], tokens[2], big.NewInt(1000e18), big.NewInt(1000e18), 30),
        NewMockPool(tokens[2], tokens[0], big.NewInt(1000e18), big.NewInt(1000e18), 30),
        // 跨 DEX 套利池（同一交易对，不同 DEX）
        NewMockPool(tokens[0], tokens[3], big.NewInt(1000e18), big.NewInt(950e18), 30),  // DEX A
        NewMockPool(tokens[0], tokens[3], big.NewInt(1000e18), big.NewInt(1050e18), 30), // DEX B
    }

    poolManager := NewMockPoolManager()
    for _, pool := range pools {
        poolManager.AddPool(pool)
    }

    // 创建策略管道
    pipeline := NewStrategyPipeline()

    // 添加三角套利策略
    triangular := NewTriangularArbitrage(&TriangularConfig{
        BaseTokens:   []common.Address{tokens[0]},
        MinProfitUSD: 1.0,
        MaxInputUSD:  10000,
        GasPrice:     big.NewInt(30e9),
    }, poolManager, nil)
    pipeline.AddStrategy(triangular)

    // 添加跨 DEX 策略
    crossDEX := NewCrossDEXArbitrage(&CrossDEXConfig{
        MinProfitUSD: 1.0,
        MaxInputUSD:  10000,
        GasPrice:     big.NewInt(30e9),
    }, poolManager, nil)
    pipeline.AddStrategy(crossDEX)

    // 执行所有策略
    allOpportunities, err := pipeline.FindAllOpportunities(ctx, pools)
    require.NoError(t, err)

    t.Logf("Found %d total opportunities", len(allOpportunities))

    // 验证机会被正确分类
    var triangularCount, crossDEXCount int
    for _, opp := range allOpportunities {
        switch opp.Type {
        case OpportunityTriangular:
            triangularCount++
        case OpportunityCrossDEX:
            crossDEXCount++
        }
    }

    t.Logf("Triangular: %d, CrossDEX: %d", triangularCount, crossDEXCount)
}

// TestFlashbotsIntegration 测试 Flashbots 集成
func TestFlashbotsIntegration(t *testing.T) {
    ctx := context.Background()

    // 创建模拟客户端
    flashbots := NewMockFlashbotsClient()
    chainClient := NewMockChainClient()

    // 设置预期的模拟结果
    flashbots.SetSimulateResult(&SimulateResult{
        Success:     true,
        TotalProfit: big.NewInt(5e16), // 0.05 ETH profit
        GasUsed:     280000,
    })

    // 创建 Bundle
    bundle := &Bundle{
        Transactions: []*types.Transaction{
            // 模拟交易...
        },
        BlockNumber: 18000001,
    }

    // 模拟 Bundle
    simResult, err := flashbots.SimulateBundle(ctx, bundle, bundle.BlockNumber)
    require.NoError(t, err)

    assert.True(t, simResult.Success)
    assert.True(t, simResult.TotalProfit.Cmp(big.NewInt(0)) > 0)

    // 发送 Bundle
    bundleHash, err := flashbots.SendBundle(ctx, bundle, bundle.BlockNumber)
    require.NoError(t, err)
    assert.NotEqual(t, common.Hash{}, bundleHash)

    // 检查 Bundle 状态
    stats, err := flashbots.GetBundleStats(ctx, bundleHash, bundle.BlockNumber)
    require.NoError(t, err)
    assert.True(t, stats.IsIncluded)
}

// TestErrorRecovery 测试错误恢复
func TestErrorRecovery(t *testing.T) {
    ctx := context.Background()

    helper := &TestHelper{}
    tokens := helper.CreateTokenAddresses(3)
    pools := helper.CreateTriangularPools(tokens)
    helper.CreateArbitrageOpportunity(pools)

    poolManager := NewMockPoolManager()
    for _, pool := range pools {
        poolManager.AddPool(pool)
    }

    // 创建会失败的 Flashbots 客户端
    flashbots := NewMockFlashbotsClient()
    flashbots.SetSimulateResult(&SimulateResult{
        Success: false,
        Error:   "execution reverted",
    })

    bot := NewArbitrageBot(&BotConfig{
        MinProfitUSD:    1.0,
        MaxInputUSD:     10000,
        SimulateFirst:   true,
        MaxRetries:      3,
        RetryDelay:      100 * time.Millisecond,
    })

    bot.SetPoolManager(poolManager)
    bot.SetFlashbotsClient(flashbots)

    triangularStrategy := NewTriangularArbitrage(&TriangularConfig{
        BaseTokens:   []common.Address{tokens[0]},
        MinProfitUSD: 1.0,
        MaxInputUSD:  10000,
        GasPrice:     big.NewInt(30e9),
    }, poolManager, nil)

    bot.RegisterStrategy(triangularStrategy)

    // 执行（应该优雅地处理失败）
    result, err := bot.ExecuteCycle(ctx, pools)
    require.NoError(t, err) // 不应该返回错误，而是记录失败

    assert.True(t, result.OpportunitiesFound > 0)
    assert.Equal(t, 0, result.BundlesIncluded) // 没有成功包含的 Bundle
    assert.True(t, result.SimulationsFailed > 0) // 模拟失败计数
}

// TestRateLimiting 测试速率限制
func TestRateLimiting(t *testing.T) {
    limiter := NewRateLimiter(&RateLimiterConfig{
        RequestsPerSecond: 10,
        BurstSize:         5,
    })

    start := time.Now()
    successCount := 0

    // 尝试发送 20 个请求
    for i := 0; i < 20; i++ {
        if limiter.Allow() {
            successCount++
        }
    }

    elapsed := time.Since(start)

    // 初始 burst 应该允许 5 个请求
    assert.True(t, successCount >= 5, "should allow at least burst size")

    // 如果测试执行很快，应该只允许 burst 数量
    if elapsed < 100*time.Millisecond {
        assert.True(t, successCount <= 10, "should not exceed burst + some refill")
    }
}

// TestMetricsCollection 测试指标收集
func TestMetricsCollection(t *testing.T) {
    metrics := NewMetricsCollector()

    // 记录一些操作
    metrics.RecordOpportunity("triangular", big.NewInt(5e16))
    metrics.RecordOpportunity("triangular", big.NewInt(3e16))
    metrics.RecordOpportunity("cross_dex", big.NewInt(8e16))

    metrics.RecordExecution(true, big.NewInt(4e16), 300000)
    metrics.RecordExecution(false, big.NewInt(0), 0)

    metrics.RecordLatency("pool_refresh", 50*time.Millisecond)
    metrics.RecordLatency("pool_refresh", 60*time.Millisecond)

    // 验证指标
    stats := metrics.GetStats()

    assert.Equal(t, int64(3), stats.TotalOpportunities)
    assert.Equal(t, int64(2), stats.TriangularOpportunities)
    assert.Equal(t, int64(1), stats.CrossDEXOpportunities)

    assert.Equal(t, int64(2), stats.TotalExecutions)
    assert.Equal(t, int64(1), stats.SuccessfulExecutions)
    assert.InDelta(t, 50.0, stats.SuccessRate, 1.0)

    assert.InDelta(t, 55.0, stats.AvgPoolRefreshLatency.Milliseconds(), 10.0)
}
```

### A.4 Benchmark 测试

```go
package benchmark_test

import (
    "context"
    "math/big"
    "testing"

    "github.com/ethereum/go-ethereum/common"
)

// BenchmarkGetAmountOut 测试 AMM 计算性能
func BenchmarkGetAmountOut(b *testing.B) {
    token0 := common.HexToAddress("0xA")
    token1 := common.HexToAddress("0xB")
    reserve := new(big.Int).Mul(big.NewInt(1000000), big.NewInt(1e18))
    pool := NewMockPool(token0, token1, reserve, reserve, 30)
    amountIn := big.NewInt(1e18)

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        pool.GetAmountOut(amountIn, true)
    }
}

// BenchmarkTriangularSearch 测试三角套利搜索性能
func BenchmarkTriangularSearch(b *testing.B) {
    helper := &TestHelper{}

    // 创建 100 个代币和 300 个池
    tokens := helper.CreateTokenAddresses(100)
    pools := make([]*MockPool, 0, 300)

    reserve := new(big.Int).Mul(big.NewInt(1000000), big.NewInt(1e18))
    for i := 0; i < 100; i++ {
        for j := i + 1; j < 100 && len(pools) < 300; j++ {
            pool := NewMockPool(tokens[i], tokens[j], reserve, reserve, 30)
            pools = append(pools, pool)
        }
    }

    poolManager := NewMockPoolManager()
    for _, pool := range pools {
        poolManager.AddPool(pool)
    }

    strategy := NewTriangularArbitrage(&TriangularConfig{
        BaseTokens:   tokens[:5], // 使用前 5 个作为基础代币
        MinProfitUSD: 1.0,
        MaxInputUSD:  10000,
        GasPrice:     big.NewInt(30e9),
    }, poolManager, nil)

    ctx := context.Background()

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        strategy.FindOpportunities(ctx, pools)
    }
}

// BenchmarkOptimalInputCalculation 测试最优输入计算性能
func BenchmarkOptimalInputCalculation(b *testing.B) {
    helper := &TestHelper{}
    tokens := helper.CreateTokenAddresses(3)
    pools := helper.CreateTriangularPools(tokens)
    helper.CreateArbitrageOpportunity(pools)

    poolManager := NewMockPoolManager()
    for _, pool := range pools {
        poolManager.AddPool(pool)
    }

    strategy := NewTriangularArbitrage(&TriangularConfig{
        BaseTokens:   []common.Address{tokens[0]},
        MinProfitUSD: 0,
        MaxInputUSD:  100000,
        GasPrice:     big.NewInt(30e9),
    }, poolManager, nil)

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        strategy.calculateOptimalInput(
            pools[0], pools[1], pools[2],
            tokens[0], tokens[1], tokens[2],
        )
    }
}

// BenchmarkConcurrentPoolLookup 测试并发池查找性能
func BenchmarkConcurrentPoolLookup(b *testing.B) {
    poolManager := NewMockPoolManager()

    // 创建 1000 个池
    reserve := new(big.Int).Mul(big.NewInt(1000000), big.NewInt(1e18))
    addresses := make([]common.Address, 1000)

    for i := 0; i < 1000; i++ {
        token0 := common.BigToAddress(big.NewInt(int64(i * 2)))
        token1 := common.BigToAddress(big.NewInt(int64(i*2 + 1)))
        pool := NewMockPool(token0, token1, reserve, reserve, 30)
        poolManager.AddPool(pool)
        addresses[i] = pool.Address
    }

    b.ResetTimer()
    b.RunParallel(func(pb *testing.PB) {
        i := 0
        for pb.Next() {
            poolManager.GetPool(addresses[i%1000])
            i++
        }
    })
}

// BenchmarkBundleBuilding 测试 Bundle 构建性能
func BenchmarkBundleBuilding(b *testing.B) {
    ctx := context.Background()
    mockClient := NewMockChainClient()

    builder := NewBundleBuilder(
        mockClient,
        common.HexToAddress("0x1234567890123456789012345678901234567890"),
        nil,
    )

    opportunity := &Opportunity{
        ID:   "bench-opp",
        Type: OpportunityTriangular,
        Path: []*SwapStep{
            {
                DEX:       "uniswap_v2",
                Pool:      common.HexToAddress("0xPool1"),
                TokenIn:   common.HexToAddress("0xTokenA"),
                TokenOut:  common.HexToAddress("0xTokenB"),
                AmountIn:  big.NewInt(1e18),
                AmountOut: big.NewInt(1e18),
            },
            {
                DEX:       "sushiswap",
                Pool:      common.HexToAddress("0xPool2"),
                TokenIn:   common.HexToAddress("0xTokenB"),
                TokenOut:  common.HexToAddress("0xTokenC"),
                AmountIn:  big.NewInt(1e18),
                AmountOut: big.NewInt(1e18),
            },
            {
                DEX:       "uniswap_v2",
                Pool:      common.HexToAddress("0xPool3"),
                TokenIn:   common.HexToAddress("0xTokenC"),
                TokenOut:  common.HexToAddress("0xTokenA"),
                AmountIn:  big.NewInt(1e18),
                AmountOut: big.NewInt(101e16),
            },
        },
        InputToken:   common.HexToAddress("0xTokenA"),
        InputAmount:  big.NewInt(1e18),
        OutputAmount: big.NewInt(101e16),
        GasEstimate:  300000,
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        builder.BuildBundle(ctx, opportunity, uint64(18000000+i))
    }
}
```

### A.5 测试覆盖率和 CI 配置

```yaml
# .github/workflows/test.yml
name: Test

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.21"

      - name: Cache Go modules
        uses: actions/cache@v4
        with:
          path: |
            ~/.cache/go-build
            ~/go/pkg/mod
          key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
          restore-keys: |
            ${{ runner.os }}-go-

      - name: Install dependencies
        run: go mod download

      - name: Run unit tests
        run: |
          go test -v -race -coverprofile=coverage.out -covermode=atomic ./...

      - name: Run integration tests
        run: |
          go test -v -race -tags=integration ./tests/integration/...

      - name: Run benchmarks
        run: |
          go test -v -bench=. -benchmem -run=^$ ./...

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.out
          flags: unittests
          name: codecov-umbrella

  lint:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.21"

      - name: golangci-lint
        uses: golangci/golangci-lint-action@v4
        with:
          version: latest
          args: --timeout=5m

  security:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Run Gosec Security Scanner
        uses: securego/gosec@master
        with:
          args: ./...

      - name: Run Staticcheck
        uses: dominikh/staticcheck-action@v1
        with:
          version: "latest"
```

```makefile
# Makefile
.PHONY: test test-unit test-integration test-bench test-coverage lint

# 运行所有测试
test:
	go test -v -race ./...

# 运行单元测试
test-unit:
	go test -v -race -short ./...

# 运行集成测试
test-integration:
	go test -v -race -tags=integration ./tests/integration/...

# 运行 benchmark
test-bench:
	go test -v -bench=. -benchmem -run=^$$ ./...

# 生成覆盖率报告
test-coverage:
	go test -v -race -coverprofile=coverage.out -covermode=atomic ./...
	go tool cover -html=coverage.out -o coverage.html
	@echo "Coverage report generated: coverage.html"

# 运行 linter
lint:
	golangci-lint run --timeout=5m

# 运行安全扫描
security:
	gosec ./...

# 清理测试缓存
clean:
	go clean -testcache
	rm -f coverage.out coverage.html
```

### A.6 测试最佳实践

| 实践 | 说明 | 示例 |
|-----|------|------|
| 表驱动测试 | 使用测试表覆盖多种场景 | `tests := []struct{...}` |
| Mock 依赖 | 隔离外部依赖 | `MockChainClient`, `MockFlashbotsClient` |
| 并发测试 | 验证并发安全性 | `t.Run("parallel", ...)` |
| Benchmark | 测量关键路径性能 | `BenchmarkGetAmountOut` |
| 集成测试 | 验证组件协作 | `TestFullArbitrageFlow` |
| 覆盖率目标 | 核心逻辑 80%+ 覆盖 | `go test -cover` |

```
┌─────────────────────────────────────────────────────────────┐
│                       测试金字塔                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                      ┌─────────┐                            │
│                      │  E2E    │  少量端到端测试              │
│                      │ Tests   │  (真实网络交互)              │
│                    ┌─┴─────────┴─┐                          │
│                    │ Integration │  中等集成测试              │
│                    │   Tests     │  (组件协作)               │
│                  ┌─┴─────────────┴─┐                        │
│                  │   Unit Tests    │  大量单元测试            │
│                  │                 │  (隔离函数)              │
│                  └─────────────────┘                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```