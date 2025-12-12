# Go-Ethereum EVM 深度解析：生产部署

## 第一百一十六节：高可用架构设计

### 生产架构概览

```
生产环境架构：
┌─────────────────────────────────────────────────────────────────────┐
│                        生产环境架构                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      负载均衡层                              │   │
│  │              (Nginx / HAProxy / Cloud LB)                    │   │
│  └────────────────────────────┬────────────────────────────────┘   │
│                               │                                     │
│         ┌─────────────────────┼─────────────────────┐               │
│         ▼                     ▼                     ▼               │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐      │
│  │  Bot 实例 1  │      │  Bot 实例 2  │      │  Bot 实例 N  │      │
│  │  (Active)    │      │  (Standby)   │      │  (Standby)   │      │
│  └──────┬───────┘      └──────┬───────┘      └──────┬───────┘      │
│         │                     │                     │               │
│         └─────────────────────┼─────────────────────┘               │
│                               │                                     │
│                               ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      共享状态层                              │   │
│  │           (Redis Cluster / etcd / Consul)                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                               │                                     │
│         ┌─────────────────────┼─────────────────────┐               │
│         ▼                     ▼                     ▼               │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐      │
│  │   RPC 节点   │      │  Flashbots   │      │   数据库     │      │
│  │   Pool       │      │   Relay      │      │   Cluster    │      │
│  └──────────────┘      └──────────────┘      └──────────────┘      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 配置管理

```go
package production

import (
    "context"
    "crypto/ecdsa"
    "fmt"
    "os"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/crypto"
    "gopkg.in/yaml.v3"
)

// Config 生产配置
type Config struct {
    // 基础配置
    Environment     string `yaml:"environment"` // dev, staging, prod
    InstanceID      string `yaml:"instance_id"`
    LogLevel        string `yaml:"log_level"`

    // 节点配置
    Nodes           NodeConfig `yaml:"nodes"`

    // 策略配置
    Strategy        StrategyConfig `yaml:"strategy"`

    // 执行配置
    Execution       ExecutionConfig `yaml:"execution"`

    // 监控配置
    Monitoring      MonitoringConfig `yaml:"monitoring"`

    // 告警配置
    Alerting        AlertingConfig `yaml:"alerting"`

    // 安全配置
    Security        SecurityConfig `yaml:"security"`
}

// NodeConfig 节点配置
type NodeConfig struct {
    Primary         []string `yaml:"primary"`
    Fallback        []string `yaml:"fallback"`
    WebSocket       []string `yaml:"websocket"`
    MaxConnections  int      `yaml:"max_connections"`
    Timeout         Duration `yaml:"timeout"`
    RetryAttempts   int      `yaml:"retry_attempts"`
    RetryDelay      Duration `yaml:"retry_delay"`
}

// StrategyConfig 策略配置
type StrategyConfig struct {
    Enabled         []string           `yaml:"enabled"`
    MinProfitETH    float64            `yaml:"min_profit_eth"`
    MaxGasPrice     uint64             `yaml:"max_gas_price_gwei"`
    MaxSlippage     float64            `yaml:"max_slippage_bps"`
    Pools           []common.Address   `yaml:"pools"`
    Tokens          []common.Address   `yaml:"tokens"`
    Custom          map[string]any     `yaml:"custom"`
}

// ExecutionConfig 执行配置
type ExecutionConfig struct {
    UseFlashbots    bool     `yaml:"use_flashbots"`
    FlashbotsRelay  string   `yaml:"flashbots_relay"`
    MaxPendingTx    int      `yaml:"max_pending_tx"`
    NonceManager    string   `yaml:"nonce_manager"` // local, redis
    GasMultiplier   float64  `yaml:"gas_multiplier"`
    SimulateFirst   bool     `yaml:"simulate_first"`
}

// MonitoringConfig 监控配置
type MonitoringConfig struct {
    Enabled         bool     `yaml:"enabled"`
    MetricsPort     int      `yaml:"metrics_port"`
    PrometheusPath  string   `yaml:"prometheus_path"`
    HealthCheckPort int      `yaml:"health_check_port"`
    PProfEnabled    bool     `yaml:"pprof_enabled"`
}

// AlertingConfig 告警配置
type AlertingConfig struct {
    Enabled         bool           `yaml:"enabled"`
    Channels        []AlertChannel `yaml:"channels"`
    Rules           []AlertRule    `yaml:"rules"`
}

// AlertChannel 告警渠道
type AlertChannel struct {
    Type        string `yaml:"type"` // slack, telegram, pagerduty, email
    Webhook     string `yaml:"webhook"`
    APIKey      string `yaml:"api_key"`
    ChatID      string `yaml:"chat_id"`
}

// AlertRule 告警规则
type AlertRule struct {
    Name        string   `yaml:"name"`
    Condition   string   `yaml:"condition"`
    Severity    string   `yaml:"severity"` // info, warning, critical
    Channels    []string `yaml:"channels"`
}

// SecurityConfig 安全配置
type SecurityConfig struct {
    KeyFile         string   `yaml:"key_file"`
    KeyEncrypted    bool     `yaml:"key_encrypted"`
    MaxTradeETH     float64  `yaml:"max_trade_eth"`
    DailyLimitETH   float64  `yaml:"daily_limit_eth"`
    WhitelistTokens []string `yaml:"whitelist_tokens"`
    BlacklistTokens []string `yaml:"blacklist_tokens"`
}

// Duration 自定义时间类型
type Duration time.Duration

// UnmarshalYAML 解析时间
func (d *Duration) UnmarshalYAML(value *yaml.Node) error {
    var s string
    if err := value.Decode(&s); err != nil {
        return err
    }
    duration, err := time.ParseDuration(s)
    if err != nil {
        return err
    }
    *d = Duration(duration)
    return nil
}

// ConfigManager 配置管理器
type ConfigManager struct {
    config      *Config
    privateKey  *ecdsa.PrivateKey
    address     common.Address
}

// NewConfigManager 创建配置管理器
func NewConfigManager(configPath string) (*ConfigManager, error) {
    data, err := os.ReadFile(configPath)
    if err != nil {
        return nil, fmt.Errorf("读取配置文件失败: %w", err)
    }

    // 环境变量替换
    data = []byte(os.ExpandEnv(string(data)))

    var config Config
    if err := yaml.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("解析配置文件失败: %w", err)
    }

    // 验证配置
    if err := validateConfig(&config); err != nil {
        return nil, fmt.Errorf("配置验证失败: %w", err)
    }

    cm := &ConfigManager{config: &config}

    // 加载私钥
    if err := cm.loadPrivateKey(); err != nil {
        return nil, err
    }

    return cm, nil
}

// validateConfig 验证配置
func validateConfig(config *Config) error {
    if config.Environment == "" {
        return fmt.Errorf("环境未指定")
    }

    if len(config.Nodes.Primary) == 0 {
        return fmt.Errorf("未配置主节点")
    }

    if config.Security.KeyFile == "" {
        return fmt.Errorf("未配置密钥文件")
    }

    return nil
}

// loadPrivateKey 加载私钥
func (cm *ConfigManager) loadPrivateKey() error {
    keyData, err := os.ReadFile(cm.config.Security.KeyFile)
    if err != nil {
        return fmt.Errorf("读取密钥文件失败: %w", err)
    }

    if cm.config.Security.KeyEncrypted {
        // 从环境变量获取密码解密
        password := os.Getenv("KEY_PASSWORD")
        if password == "" {
            return fmt.Errorf("未设置 KEY_PASSWORD 环境变量")
        }
        // 解密逻辑...
        _ = password
    }

    privateKey, err := crypto.HexToECDSA(string(keyData))
    if err != nil {
        return fmt.Errorf("解析私钥失败: %w", err)
    }

    cm.privateKey = privateKey
    cm.address = crypto.PubkeyToAddress(privateKey.PublicKey)

    return nil
}

// GetConfig 获取配置
func (cm *ConfigManager) GetConfig() *Config {
    return cm.config
}

// GetPrivateKey 获取私钥
func (cm *ConfigManager) GetPrivateKey() *ecdsa.PrivateKey {
    return cm.privateKey
}

// GetAddress 获取地址
func (cm *ConfigManager) GetAddress() common.Address {
    return cm.address
}
```

### RPC 节点池

```go
package production

import (
    "context"
    "fmt"
    "math/big"
    "sync"
    "sync/atomic"
    "time"

    "github.com/ethereum/go-ethereum"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
)

// RPCPool RPC 节点池
type RPCPool struct {
    nodes       []*RPCNode
    primary     []*RPCNode
    fallback    []*RPCNode
    current     int64
    healthCheck *time.Ticker
    mu          sync.RWMutex
}

// RPCNode RPC 节点
type RPCNode struct {
    URL         string
    Client      *ethclient.Client
    IsHealthy   atomic.Bool
    Latency     atomic.Int64 // 纳秒
    ErrorCount  atomic.Int64
    RequestCount atomic.Int64
    LastCheck   atomic.Int64
    Priority    int // 0: primary, 1: fallback
}

// NewRPCPool 创建节点池
func NewRPCPool(config *NodeConfig) (*RPCPool, error) {
    pool := &RPCPool{
        nodes:    make([]*RPCNode, 0),
        primary:  make([]*RPCNode, 0),
        fallback: make([]*RPCNode, 0),
    }

    // 连接主节点
    for _, url := range config.Primary {
        node, err := pool.connectNode(url, 0)
        if err != nil {
            fmt.Printf("警告: 主节点 %s 连接失败: %v\n", url, err)
            continue
        }
        pool.primary = append(pool.primary, node)
        pool.nodes = append(pool.nodes, node)
    }

    // 连接备用节点
    for _, url := range config.Fallback {
        node, err := pool.connectNode(url, 1)
        if err != nil {
            fmt.Printf("警告: 备用节点 %s 连接失败: %v\n", url, err)
            continue
        }
        pool.fallback = append(pool.fallback, node)
        pool.nodes = append(pool.nodes, node)
    }

    if len(pool.nodes) == 0 {
        return nil, fmt.Errorf("没有可用节点")
    }

    // 启动健康检查
    pool.healthCheck = time.NewTicker(10 * time.Second)
    go pool.healthCheckLoop()

    return pool, nil
}

// connectNode 连接节点
func (p *RPCPool) connectNode(url string, priority int) (*RPCNode, error) {
    client, err := ethclient.Dial(url)
    if err != nil {
        return nil, err
    }

    node := &RPCNode{
        URL:      url,
        Client:   client,
        Priority: priority,
    }
    node.IsHealthy.Store(true)

    return node, nil
}

// healthCheckLoop 健康检查循环
func (p *RPCPool) healthCheckLoop() {
    for range p.healthCheck.C {
        p.checkAllNodes()
    }
}

// checkAllNodes 检查所有节点
func (p *RPCPool) checkAllNodes() {
    var wg sync.WaitGroup

    for _, node := range p.nodes {
        wg.Add(1)
        go func(n *RPCNode) {
            defer wg.Done()
            p.checkNode(n)
        }(node)
    }

    wg.Wait()
}

// checkNode 检查单个节点
func (p *RPCPool) checkNode(node *RPCNode) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    start := time.Now()
    _, err := node.Client.BlockNumber(ctx)
    latency := time.Since(start)

    node.Latency.Store(latency.Nanoseconds())
    node.LastCheck.Store(time.Now().Unix())

    if err != nil {
        node.ErrorCount.Add(1)
        // 连续 3 次失败标记为不健康
        if node.ErrorCount.Load() >= 3 {
            node.IsHealthy.Store(false)
        }
    } else {
        node.ErrorCount.Store(0)
        node.IsHealthy.Store(true)
    }
}

// GetClient 获取客户端 (轮询负载均衡)
func (p *RPCPool) GetClient() *ethclient.Client {
    p.mu.RLock()
    defer p.mu.RUnlock()

    // 优先使用健康的主节点
    for _, node := range p.primary {
        if node.IsHealthy.Load() {
            node.RequestCount.Add(1)
            return node.Client
        }
    }

    // 回退到备用节点
    for _, node := range p.fallback {
        if node.IsHealthy.Load() {
            node.RequestCount.Add(1)
            return node.Client
        }
    }

    // 无健康节点，返回任意主节点
    if len(p.primary) > 0 {
        return p.primary[0].Client
    }

    return nil
}

// GetFastestClient 获取最快的客户端
func (p *RPCPool) GetFastestClient() *ethclient.Client {
    p.mu.RLock()
    defer p.mu.RUnlock()

    var fastest *RPCNode
    var minLatency int64 = 1<<63 - 1

    for _, node := range p.nodes {
        if node.IsHealthy.Load() && node.Latency.Load() < minLatency {
            minLatency = node.Latency.Load()
            fastest = node
        }
    }

    if fastest != nil {
        fastest.RequestCount.Add(1)
        return fastest.Client
    }

    return p.GetClient()
}

// ExecuteWithRetry 带重试的执行
func (p *RPCPool) ExecuteWithRetry(
    ctx context.Context,
    fn func(*ethclient.Client) error,
    maxRetries int,
) error {
    var lastErr error

    for i := 0; i < maxRetries; i++ {
        client := p.GetClient()
        if client == nil {
            return fmt.Errorf("没有可用客户端")
        }

        if err := fn(client); err != nil {
            lastErr = err
            time.Sleep(time.Duration(i+1) * 100 * time.Millisecond)
            continue
        }

        return nil
    }

    return fmt.Errorf("重试 %d 次后失败: %w", maxRetries, lastErr)
}

// Call 调用合约
func (p *RPCPool) Call(ctx context.Context, msg ethereum.CallMsg) ([]byte, error) {
    var result []byte

    err := p.ExecuteWithRetry(ctx, func(client *ethclient.Client) error {
        var err error
        result, err = client.CallContract(ctx, msg, nil)
        return err
    }, 3)

    return result, err
}

// SendTransaction 发送交易
func (p *RPCPool) SendTransaction(ctx context.Context, tx *types.Transaction) error {
    return p.ExecuteWithRetry(ctx, func(client *ethclient.Client) error {
        return client.SendTransaction(ctx, tx)
    }, 3)
}

// BroadcastTransaction 广播交易到所有节点
func (p *RPCPool) BroadcastTransaction(ctx context.Context, tx *types.Transaction) []error {
    var errors []error
    var mu sync.Mutex
    var wg sync.WaitGroup

    for _, node := range p.nodes {
        if !node.IsHealthy.Load() {
            continue
        }

        wg.Add(1)
        go func(n *RPCNode) {
            defer wg.Done()
            if err := n.Client.SendTransaction(ctx, tx); err != nil {
                mu.Lock()
                errors = append(errors, err)
                mu.Unlock()
            }
        }(node)
    }

    wg.Wait()
    return errors
}

// GetStatus 获取池状态
func (p *RPCPool) GetStatus() *PoolStatus {
    p.mu.RLock()
    defer p.mu.RUnlock()

    status := &PoolStatus{
        TotalNodes:   len(p.nodes),
        HealthyNodes: 0,
        Nodes:        make([]NodeStatus, len(p.nodes)),
    }

    for i, node := range p.nodes {
        isHealthy := node.IsHealthy.Load()
        if isHealthy {
            status.HealthyNodes++
        }

        status.Nodes[i] = NodeStatus{
            URL:          node.URL,
            IsHealthy:    isHealthy,
            Latency:      time.Duration(node.Latency.Load()),
            ErrorCount:   node.ErrorCount.Load(),
            RequestCount: node.RequestCount.Load(),
            LastCheck:    time.Unix(node.LastCheck.Load(), 0),
        }
    }

    return status
}

// PoolStatus 池状态
type PoolStatus struct {
    TotalNodes   int
    HealthyNodes int
    Nodes        []NodeStatus
}

// NodeStatus 节点状态
type NodeStatus struct {
    URL          string
    IsHealthy    bool
    Latency      time.Duration
    ErrorCount   int64
    RequestCount int64
    LastCheck    time.Time
}

// Close 关闭池
func (p *RPCPool) Close() {
    p.healthCheck.Stop()
    for _, node := range p.nodes {
        node.Client.Close()
    }
}
```

---

## 第一百一十七节：监控与告警

### Prometheus 指标

```go
package production

import (
    "net/http"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

// Metrics 监控指标
type Metrics struct {
    // 交易指标
    TxSubmitted     prometheus.Counter
    TxConfirmed     prometheus.Counter
    TxFailed        prometheus.Counter
    TxPending       prometheus.Gauge

    // 利润指标
    ProfitTotal     prometheus.Counter
    ProfitPerTx     prometheus.Histogram

    // Gas 指标
    GasUsed         prometheus.Counter
    GasPrice        prometheus.Gauge

    // 延迟指标
    BlockLatency    prometheus.Histogram
    TxLatency       prometheus.Histogram
    RPCLatency      prometheus.HistogramVec

    // 机会指标
    OpportunitiesFound    prometheus.Counter
    OpportunitiesExecuted prometheus.Counter
    OpportunitiesMissed   prometheus.Counter

    // 系统指标
    Uptime          prometheus.Counter
    HeartbeatAge    prometheus.Gauge
    MemoryUsage     prometheus.Gauge
    GoroutineCount  prometheus.Gauge

    // 节点指标
    RPCNodeHealth   *prometheus.GaugeVec
    RPCNodeLatency  *prometheus.GaugeVec
}

// NewMetrics 创建指标
func NewMetrics(namespace string) *Metrics {
    m := &Metrics{
        TxSubmitted: promauto.NewCounter(prometheus.CounterOpts{
            Namespace: namespace,
            Name:      "tx_submitted_total",
            Help:      "提交的交易总数",
        }),
        TxConfirmed: promauto.NewCounter(prometheus.CounterOpts{
            Namespace: namespace,
            Name:      "tx_confirmed_total",
            Help:      "确认的交易总数",
        }),
        TxFailed: promauto.NewCounter(prometheus.CounterOpts{
            Namespace: namespace,
            Name:      "tx_failed_total",
            Help:      "失败的交易总数",
        }),
        TxPending: promauto.NewGauge(prometheus.GaugeOpts{
            Namespace: namespace,
            Name:      "tx_pending",
            Help:      "待处理交易数",
        }),
        ProfitTotal: promauto.NewCounter(prometheus.CounterOpts{
            Namespace: namespace,
            Name:      "profit_total_wei",
            Help:      "总利润 (wei)",
        }),
        ProfitPerTx: promauto.NewHistogram(prometheus.HistogramOpts{
            Namespace: namespace,
            Name:      "profit_per_tx_eth",
            Help:      "每笔交易利润 (ETH)",
            Buckets:   []float64{0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0},
        }),
        GasUsed: promauto.NewCounter(prometheus.CounterOpts{
            Namespace: namespace,
            Name:      "gas_used_total",
            Help:      "使用的 Gas 总量",
        }),
        GasPrice: promauto.NewGauge(prometheus.GaugeOpts{
            Namespace: namespace,
            Name:      "gas_price_gwei",
            Help:      "当前 Gas 价格 (Gwei)",
        }),
        BlockLatency: promauto.NewHistogram(prometheus.HistogramOpts{
            Namespace: namespace,
            Name:      "block_latency_seconds",
            Help:      "区块接收延迟",
            Buckets:   []float64{0.1, 0.5, 1, 2, 5, 10},
        }),
        TxLatency: promauto.NewHistogram(prometheus.HistogramOpts{
            Namespace: namespace,
            Name:      "tx_latency_seconds",
            Help:      "交易确认延迟",
            Buckets:   []float64{1, 5, 10, 30, 60, 120},
        }),
        RPCLatency: *promauto.NewHistogramVec(prometheus.HistogramOpts{
            Namespace: namespace,
            Name:      "rpc_latency_seconds",
            Help:      "RPC 调用延迟",
            Buckets:   []float64{0.01, 0.05, 0.1, 0.5, 1, 5},
        }, []string{"node", "method"}),
        OpportunitiesFound: promauto.NewCounter(prometheus.CounterOpts{
            Namespace: namespace,
            Name:      "opportunities_found_total",
            Help:      "发现的机会总数",
        }),
        OpportunitiesExecuted: promauto.NewCounter(prometheus.CounterOpts{
            Namespace: namespace,
            Name:      "opportunities_executed_total",
            Help:      "执行的机会总数",
        }),
        OpportunitiesMissed: promauto.NewCounter(prometheus.CounterOpts{
            Namespace: namespace,
            Name:      "opportunities_missed_total",
            Help:      "错过的机会总数",
        }),
        Uptime: promauto.NewCounter(prometheus.CounterOpts{
            Namespace: namespace,
            Name:      "uptime_seconds_total",
            Help:      "运行时间 (秒)",
        }),
        HeartbeatAge: promauto.NewGauge(prometheus.GaugeOpts{
            Namespace: namespace,
            Name:      "heartbeat_age_seconds",
            Help:      "上次心跳距今时间",
        }),
        MemoryUsage: promauto.NewGauge(prometheus.GaugeOpts{
            Namespace: namespace,
            Name:      "memory_usage_bytes",
            Help:      "内存使用量",
        }),
        GoroutineCount: promauto.NewGauge(prometheus.GaugeOpts{
            Namespace: namespace,
            Name:      "goroutine_count",
            Help:      "Goroutine 数量",
        }),
        RPCNodeHealth: promauto.NewGaugeVec(prometheus.GaugeOpts{
            Namespace: namespace,
            Name:      "rpc_node_health",
            Help:      "RPC 节点健康状态 (1=健康, 0=不健康)",
        }, []string{"node"}),
        RPCNodeLatency: promauto.NewGaugeVec(prometheus.GaugeOpts{
            Namespace: namespace,
            Name:      "rpc_node_latency_ms",
            Help:      "RPC 节点延迟 (毫秒)",
        }, []string{"node"}),
    }

    return m
}

// StartMetricsServer 启动指标服务
func StartMetricsServer(port int, path string) *http.Server {
    mux := http.NewServeMux()
    mux.Handle(path, promhttp.Handler())

    server := &http.Server{
        Addr:    fmt.Sprintf(":%d", port),
        Handler: mux,
    }

    go server.ListenAndServe()

    return server
}

// AlertManager 告警管理器
type AlertManager struct {
    config   *AlertingConfig
    channels map[string]AlertSender
    rules    []*AlertRule
}

// AlertSender 告警发送接口
type AlertSender interface {
    Send(alert *Alert) error
}

// Alert 告警
type Alert struct {
    Name      string
    Severity  string
    Message   string
    Timestamp time.Time
    Labels    map[string]string
}

// NewAlertManager 创建告警管理器
func NewAlertManager(config *AlertingConfig) *AlertManager {
    am := &AlertManager{
        config:   config,
        channels: make(map[string]AlertSender),
        rules:    config.Rules,
    }

    // 初始化渠道
    for _, ch := range config.Channels {
        switch ch.Type {
        case "slack":
            am.channels[ch.Type] = NewSlackSender(ch.Webhook)
        case "telegram":
            am.channels[ch.Type] = NewTelegramSender(ch.APIKey, ch.ChatID)
        case "pagerduty":
            am.channels[ch.Type] = NewPagerDutySender(ch.APIKey)
        }
    }

    return am
}

// SendAlert 发送告警
func (am *AlertManager) SendAlert(alert *Alert) {
    for _, rule := range am.rules {
        if rule.Name == alert.Name {
            for _, chType := range rule.Channels {
                if sender, exists := am.channels[chType]; exists {
                    go sender.Send(alert)
                }
            }
            break
        }
    }
}

// SlackSender Slack 发送器
type SlackSender struct {
    webhookURL string
}

// NewSlackSender 创建 Slack 发送器
func NewSlackSender(webhookURL string) *SlackSender {
    return &SlackSender{webhookURL: webhookURL}
}

// Send 发送告警
func (s *SlackSender) Send(alert *Alert) error {
    // 构建 Slack 消息
    payload := map[string]interface{}{
        "text": fmt.Sprintf("[%s] %s: %s", alert.Severity, alert.Name, alert.Message),
        "attachments": []map[string]interface{}{
            {
                "color": s.getSeverityColor(alert.Severity),
                "fields": []map[string]interface{}{
                    {"title": "时间", "value": alert.Timestamp.Format(time.RFC3339), "short": true},
                    {"title": "严重程度", "value": alert.Severity, "short": true},
                },
            },
        },
    }

    // 发送 HTTP POST
    _ = payload // 实际实现需要 HTTP 请求
    return nil
}

func (s *SlackSender) getSeverityColor(severity string) string {
    switch severity {
    case "critical":
        return "#FF0000"
    case "warning":
        return "#FFA500"
    default:
        return "#00FF00"
    }
}

// TelegramSender Telegram 发送器
type TelegramSender struct {
    apiKey string
    chatID string
}

// NewTelegramSender 创建 Telegram 发送器
func NewTelegramSender(apiKey, chatID string) *TelegramSender {
    return &TelegramSender{apiKey: apiKey, chatID: chatID}
}

// Send 发送告警
func (t *TelegramSender) Send(alert *Alert) error {
    // 实现 Telegram Bot API 调用
    return nil
}

// PagerDutySender PagerDuty 发送器
type PagerDutySender struct {
    apiKey string
}

// NewPagerDutySender 创建 PagerDuty 发送器
func NewPagerDutySender(apiKey string) *PagerDutySender {
    return &PagerDutySender{apiKey: apiKey}
}

// Send 发送告警
func (p *PagerDutySender) Send(alert *Alert) error {
    // 实现 PagerDuty API 调用
    return nil
}
```

---

## 第一百一十八节：分布式协调

### 分布式锁与领导选举

```go
package production

import (
    "context"
    "fmt"
    "time"

    "github.com/go-redis/redis/v8"
)

// DistributedCoordinator 分布式协调器
type DistributedCoordinator struct {
    redis      *redis.Client
    instanceID string
    namespace  string
}

// NewDistributedCoordinator 创建协调器
func NewDistributedCoordinator(redisURL, instanceID, namespace string) (*DistributedCoordinator, error) {
    opt, err := redis.ParseURL(redisURL)
    if err != nil {
        return nil, err
    }

    client := redis.NewClient(opt)

    // 测试连接
    if err := client.Ping(context.Background()).Err(); err != nil {
        return nil, fmt.Errorf("Redis 连接失败: %w", err)
    }

    return &DistributedCoordinator{
        redis:      client,
        instanceID: instanceID,
        namespace:  namespace,
    }, nil
}

// TryLock 尝试获取锁
func (dc *DistributedCoordinator) TryLock(ctx context.Context, key string, ttl time.Duration) (bool, error) {
    lockKey := dc.namespace + ":lock:" + key

    // SET NX EX
    result, err := dc.redis.SetNX(ctx, lockKey, dc.instanceID, ttl).Result()
    if err != nil {
        return false, err
    }

    return result, nil
}

// Unlock 释放锁
func (dc *DistributedCoordinator) Unlock(ctx context.Context, key string) error {
    lockKey := dc.namespace + ":lock:" + key

    // 使用 Lua 脚本确保只释放自己的锁
    script := `
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
    `

    _, err := dc.redis.Eval(ctx, script, []string{lockKey}, dc.instanceID).Result()
    return err
}

// ExtendLock 延长锁
func (dc *DistributedCoordinator) ExtendLock(ctx context.Context, key string, ttl time.Duration) (bool, error) {
    lockKey := dc.namespace + ":lock:" + key

    script := `
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("pexpire", KEYS[1], ARGV[2])
        else
            return 0
        end
    `

    result, err := dc.redis.Eval(ctx, script, []string{lockKey}, dc.instanceID, ttl.Milliseconds()).Result()
    if err != nil {
        return false, err
    }

    return result.(int64) == 1, nil
}

// LeaderElection 领导选举
type LeaderElection struct {
    coord       *DistributedCoordinator
    electionKey string
    ttl         time.Duration
    isLeader    bool
    onBecomeLeader func()
    onLoseLeader   func()
    stopCh      chan struct{}
}

// NewLeaderElection 创建领导选举
func NewLeaderElection(
    coord *DistributedCoordinator,
    electionKey string,
    ttl time.Duration,
) *LeaderElection {
    return &LeaderElection{
        coord:       coord,
        electionKey: electionKey,
        ttl:         ttl,
        stopCh:      make(chan struct{}),
    }
}

// Start 启动选举
func (le *LeaderElection) Start(ctx context.Context) {
    ticker := time.NewTicker(le.ttl / 3)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-le.stopCh:
            return
        case <-ticker.C:
            le.tryBecomeLeader(ctx)
        }
    }
}

// tryBecomeLeader 尝试成为领导
func (le *LeaderElection) tryBecomeLeader(ctx context.Context) {
    if le.isLeader {
        // 已是领导，尝试续期
        ok, err := le.coord.ExtendLock(ctx, le.electionKey, le.ttl)
        if err != nil || !ok {
            // 失去领导权
            le.isLeader = false
            if le.onLoseLeader != nil {
                le.onLoseLeader()
            }
        }
    } else {
        // 尝试竞选
        ok, err := le.coord.TryLock(ctx, le.electionKey, le.ttl)
        if err == nil && ok {
            le.isLeader = true
            if le.onBecomeLeader != nil {
                le.onBecomeLeader()
            }
        }
    }
}

// IsLeader 是否是领导
func (le *LeaderElection) IsLeader() bool {
    return le.isLeader
}

// Stop 停止选举
func (le *LeaderElection) Stop() {
    close(le.stopCh)
    if le.isLeader {
        le.coord.Unlock(context.Background(), le.electionKey)
    }
}

// OnBecomeLeader 设置成为领导回调
func (le *LeaderElection) OnBecomeLeader(fn func()) {
    le.onBecomeLeader = fn
}

// OnLoseLeader 设置失去领导回调
func (le *LeaderElection) OnLoseLeader(fn func()) {
    le.onLoseLeader = fn
}

// SharedState 共享状态
type SharedState struct {
    coord *DistributedCoordinator
}

// NewSharedState 创建共享状态
func NewSharedState(coord *DistributedCoordinator) *SharedState {
    return &SharedState{coord: coord}
}

// SetPendingTx 设置待处理交易
func (ss *SharedState) SetPendingTx(ctx context.Context, txHash string, data []byte, ttl time.Duration) error {
    key := ss.coord.namespace + ":pending_tx:" + txHash
    return ss.coord.redis.Set(ctx, key, data, ttl).Err()
}

// GetPendingTx 获取待处理交易
func (ss *SharedState) GetPendingTx(ctx context.Context, txHash string) ([]byte, error) {
    key := ss.coord.namespace + ":pending_tx:" + txHash
    return ss.coord.redis.Get(ctx, key).Bytes()
}

// DeletePendingTx 删除待处理交易
func (ss *SharedState) DeletePendingTx(ctx context.Context, txHash string) error {
    key := ss.coord.namespace + ":pending_tx:" + txHash
    return ss.coord.redis.Del(ctx, key).Err()
}

// IncrementNonce 增加 nonce
func (ss *SharedState) IncrementNonce(ctx context.Context, address string) (int64, error) {
    key := ss.coord.namespace + ":nonce:" + address
    return ss.coord.redis.Incr(ctx, key).Result()
}

// GetNonce 获取 nonce
func (ss *SharedState) GetNonce(ctx context.Context, address string) (int64, error) {
    key := ss.coord.namespace + ":nonce:" + address
    val, err := ss.coord.redis.Get(ctx, key).Int64()
    if err == redis.Nil {
        return 0, nil
    }
    return val, err
}

// SetNonce 设置 nonce
func (ss *SharedState) SetNonce(ctx context.Context, address string, nonce int64) error {
    key := ss.coord.namespace + ":nonce:" + address
    return ss.coord.redis.Set(ctx, key, nonce, 0).Err()
}
```

---

## 本文总结

本文深入探讨了套利机器人生产部署，主要涵盖：

### 生产部署清单

| 组件 | 功能 | 关键配置 |
|------|------|---------|
| 配置管理 | 环境变量、密钥管理 | YAML + 环境变量 |
| RPC 节点池 | 高可用、负载均衡 | 健康检查、重试 |
| 监控系统 | Prometheus 指标 | 自定义指标 |
| 告警系统 | 多渠道通知 | Slack、Telegram |
| 分布式协调 | 锁、选举、状态 | Redis Cluster |

### 高可用架构

```
高可用部署模式：
┌─────────────────────────────────────────────────────────────────┐
│                       高可用架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  主从模式                          集群模式                      │
│  ┌────────────────┐               ┌────────────────┐           │
│  │ Active         │               │   Node 1       │           │
│  │ (处理交易)     │               │   (分片 1)     │           │
│  └───────┬────────┘               └───────┬────────┘           │
│          │ 心跳                            │                    │
│          ▼                                 ▼                    │
│  ┌────────────────┐               ┌────────────────┐           │
│  │ Standby        │               │   Node 2       │           │
│  │ (热备)         │               │   (分片 2)     │           │
│  └────────────────┘               └────────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 最佳实践

1. **密钥安全**: 使用 HSM 或加密存储
2. **多节点冗余**: 至少 3 个 RPC 节点
3. **监控告警**: 实时监控关键指标
4. **优雅降级**: 出错时安全停止
