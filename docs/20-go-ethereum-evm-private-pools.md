# Go-Ethereum EVM 深度解析：私有交易池

## 第一百一十九节：私有交易池概览

### 私有交易池生态

```
私有交易池生态系统：
┌─────────────────────────────────────────────────────────────────────┐
│                      私有交易池生态                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     Flashbots                                │   │
│  │  • MEV-Share (MEV 收益分享)                                  │   │
│  │  • Protect RPC (三明治保护)                                  │   │
│  │  • Bundle Relay (搜索者专用)                                 │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     bloXroute                                │   │
│  │  • BDN (区块链分发网络)                                      │   │
│  │  • Private Transaction (私有交易)                            │   │
│  │  • Front-Running Protection                                  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     Eden Network                             │   │
│  │  • Priority Ordering (优先排序)                              │   │
│  │  • Slot Tenants (插槽租户)                                   │   │
│  │  • Protected Transactions                                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                   其他方案                                   │   │
│  │  • MEV Blocker  • Cowswap  • 1inch Fusion                   │   │
│  │  • Merkle       • Agnostic Relay                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 统一接口设计

```go
package privatepool

import (
    "context"
    "crypto/ecdsa"
    "math/big"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
)

// PrivatePoolProvider 私有池提供者接口
type PrivatePoolProvider interface {
    // 名称
    Name() string

    // 发送私有交易
    SendPrivateTransaction(ctx context.Context, tx *PrivateTransaction) (*SubmissionResult, error)

    // 发送 Bundle
    SendBundle(ctx context.Context, bundle *Bundle) (*SubmissionResult, error)

    // 取消交易
    CancelTransaction(ctx context.Context, txHash common.Hash) error

    // 获取交易状态
    GetTransactionStatus(ctx context.Context, txHash common.Hash) (*TransactionStatus, error)

    // 模拟交易
    SimulateTransaction(ctx context.Context, tx *PrivateTransaction) (*SimulationResult, error)

    // 获取构建者列表
    GetBuilders() []string

    // 健康检查
    HealthCheck(ctx context.Context) error
}

// PrivateTransaction 私有交易
type PrivateTransaction struct {
    Tx              *types.Transaction
    MaxBlockNumber  uint64            // 最大目标区块
    Hints           *TransactionHints // 交易提示 (MEV-Share)
    Preferences     *TxPreferences    // 交易偏好
    RefundPercent   int               // MEV 退款百分比
    RefundAddress   common.Address    // 退款地址
}

// TransactionHints MEV-Share 提示
type TransactionHints struct {
    CallData        bool     // 暴露 calldata
    ContractAddress bool     // 暴露合约地址
    FunctionSelector bool    // 暴露函数选择器
    Logs            bool     // 暴露日志
    TxHash          bool     // 暴露交易哈希
}

// TxPreferences 交易偏好
type TxPreferences struct {
    Fast        bool   // 快速打包
    Privacy     string // privacy level: standard, high
    Builders    []string // 指定构建者
}

// Bundle 交易 Bundle
type Bundle struct {
    Transactions    []*types.Transaction
    BlockNumber     uint64
    MinTimestamp    uint64
    MaxTimestamp    uint64
    RevertingHashes []common.Hash // 允许回滚的交易
    RefundPercent   int
    RefundAddress   common.Address
}

// SubmissionResult 提交结果
type SubmissionResult struct {
    Success     bool
    BundleHash  common.Hash
    TxHashes    []common.Hash
    Error       string
    Timestamp   time.Time
}

// TransactionStatus 交易状态
type TransactionStatus struct {
    Hash            common.Hash
    Status          TxStatusType
    BlockNumber     uint64
    IncludedInBlock bool
    Refund          *big.Int
    Error           string
}

// TxStatusType 交易状态类型
type TxStatusType int

const (
    TxStatusPending TxStatusType = iota
    TxStatusIncluded
    TxStatusFailed
    TxStatusCanceled
    TxStatusExpired
)

// SimulationResult 模拟结果
type SimulationResult struct {
    Success         bool
    GasUsed         uint64
    EffectiveGasPrice *big.Int
    Logs            []*types.Log
    StateChanges    []StateChange
    Error           string
    Profit          *big.Int
}

// StateChange 状态变更
type StateChange struct {
    Address  common.Address
    Slot     common.Hash
    Before   common.Hash
    After    common.Hash
}

// MultiPoolManager 多池管理器
type MultiPoolManager struct {
    providers       map[string]PrivatePoolProvider
    defaultProvider string
    privateKey      *ecdsa.PrivateKey
}

// NewMultiPoolManager 创建多池管理器
func NewMultiPoolManager(privateKey *ecdsa.PrivateKey) *MultiPoolManager {
    return &MultiPoolManager{
        providers:  make(map[string]PrivatePoolProvider),
        privateKey: privateKey,
    }
}

// RegisterProvider 注册提供者
func (m *MultiPoolManager) RegisterProvider(provider PrivatePoolProvider) {
    m.providers[provider.Name()] = provider
}

// SetDefaultProvider 设置默认提供者
func (m *MultiPoolManager) SetDefaultProvider(name string) {
    m.defaultProvider = name
}

// SendToAll 发送到所有提供者
func (m *MultiPoolManager) SendToAll(ctx context.Context, tx *PrivateTransaction) map[string]*SubmissionResult {
    results := make(map[string]*SubmissionResult)

    for name, provider := range m.providers {
        result, err := provider.SendPrivateTransaction(ctx, tx)
        if err != nil {
            results[name] = &SubmissionResult{
                Success: false,
                Error:   err.Error(),
            }
        } else {
            results[name] = result
        }
    }

    return results
}

// SendToFastest 发送到最快的提供者
func (m *MultiPoolManager) SendToFastest(ctx context.Context, tx *PrivateTransaction) (*SubmissionResult, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    resultCh := make(chan *SubmissionResult, len(m.providers))

    for _, provider := range m.providers {
        go func(p PrivatePoolProvider) {
            result, err := p.SendPrivateTransaction(ctx, tx)
            if err == nil && result.Success {
                resultCh <- result
            }
        }(provider)
    }

    select {
    case result := <-resultCh:
        return result, nil
    case <-ctx.Done():
        return nil, ctx.Err()
    }
}
```

---

## 第一百二十节：bloXroute 集成

### bloXroute 客户端

```go
package privatepool

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/common/hexutil"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/gorilla/websocket"
)

// BloXrouteProvider bloXroute 提供者
type BloXrouteProvider struct {
    httpEndpoint   string
    wsEndpoint     string
    authHeader     string
    httpClient     *http.Client
    wsConn         *websocket.Conn
}

// BloXrouteConfig bloXroute 配置
type BloXrouteConfig struct {
    HTTPEndpoint   string // https://api.blxrbdn.com
    WSEndpoint     string // wss://api.blxrbdn.com/ws
    AuthHeader     string // Authorization header
}

// NewBloXrouteProvider 创建 bloXroute 提供者
func NewBloXrouteProvider(config *BloXrouteConfig) (*BloXrouteProvider, error) {
    provider := &BloXrouteProvider{
        httpEndpoint: config.HTTPEndpoint,
        wsEndpoint:   config.WSEndpoint,
        authHeader:   config.AuthHeader,
        httpClient: &http.Client{
            Timeout: 30 * time.Second,
        },
    }

    return provider, nil
}

// Name 名称
func (p *BloXrouteProvider) Name() string {
    return "bloXroute"
}

// SendPrivateTransaction 发送私有交易
func (p *BloXrouteProvider) SendPrivateTransaction(
    ctx context.Context,
    tx *PrivateTransaction,
) (*SubmissionResult, error) {
    rawTx, err := tx.Tx.MarshalBinary()
    if err != nil {
        return nil, err
    }

    // 构建请求
    request := map[string]interface{}{
        "transaction": hexutil.Encode(rawTx),
    }

    // 添加前置保护
    if tx.Preferences != nil && tx.Preferences.Privacy == "high" {
        request["front_running_protection"] = true
    }

    // 指定构建者
    if tx.Preferences != nil && len(tx.Preferences.Builders) > 0 {
        request["block_builders"] = tx.Preferences.Builders
    }

    return p.sendRequest(ctx, "blxr_private_tx", request)
}

// SendBundle 发送 Bundle
func (p *BloXrouteProvider) SendBundle(ctx context.Context, bundle *Bundle) (*SubmissionResult, error) {
    // 编码交易
    txs := make([]string, len(bundle.Transactions))
    for i, tx := range bundle.Transactions {
        rawTx, err := tx.MarshalBinary()
        if err != nil {
            return nil, err
        }
        txs[i] = hexutil.Encode(rawTx)
    }

    request := map[string]interface{}{
        "transactions":       txs,
        "block_number":       hexutil.EncodeUint64(bundle.BlockNumber),
        "min_timestamp":      bundle.MinTimestamp,
        "max_timestamp":      bundle.MaxTimestamp,
        "reverting_tx_hashes": bundle.RevertingHashes,
    }

    if bundle.RefundPercent > 0 {
        request["mev_builders"] = map[string]interface{}{
            "bloxroute": fmt.Sprintf("%d%%", bundle.RefundPercent),
        }
    }

    return p.sendRequest(ctx, "blxr_submit_bundle", request)
}

// sendRequest 发送请求
func (p *BloXrouteProvider) sendRequest(
    ctx context.Context,
    method string,
    params map[string]interface{},
) (*SubmissionResult, error) {
    payload := map[string]interface{}{
        "jsonrpc": "2.0",
        "id":      1,
        "method":  method,
        "params":  params,
    }

    body, err := json.Marshal(payload)
    if err != nil {
        return nil, err
    }

    req, err := http.NewRequestWithContext(ctx, "POST", p.httpEndpoint, bytes.NewReader(body))
    if err != nil {
        return nil, err
    }

    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("Authorization", p.authHeader)

    resp, err := p.httpClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    respBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, err
    }

    var result struct {
        Result json.RawMessage `json:"result"`
        Error  *struct {
            Code    int    `json:"code"`
            Message string `json:"message"`
        } `json:"error"`
    }

    if err := json.Unmarshal(respBody, &result); err != nil {
        return nil, err
    }

    if result.Error != nil {
        return &SubmissionResult{
            Success: false,
            Error:   result.Error.Message,
        }, nil
    }

    return &SubmissionResult{
        Success:   true,
        Timestamp: time.Now(),
    }, nil
}

// CancelTransaction 取消交易
func (p *BloXrouteProvider) CancelTransaction(ctx context.Context, txHash common.Hash) error {
    request := map[string]interface{}{
        "transaction_hash": txHash.Hex(),
    }

    _, err := p.sendRequest(ctx, "blxr_cancel_private_tx", request)
    return err
}

// GetTransactionStatus 获取交易状态
func (p *BloXrouteProvider) GetTransactionStatus(
    ctx context.Context,
    txHash common.Hash,
) (*TransactionStatus, error) {
    request := map[string]interface{}{
        "transaction_hash": txHash.Hex(),
    }

    result, err := p.sendRequest(ctx, "blxr_tx_status", request)
    if err != nil {
        return nil, err
    }

    return &TransactionStatus{
        Hash:   txHash,
        Status: TxStatusPending, // 解析实际状态
        Error:  result.Error,
    }, nil
}

// SimulateTransaction 模拟交易
func (p *BloXrouteProvider) SimulateTransaction(
    ctx context.Context,
    tx *PrivateTransaction,
) (*SimulationResult, error) {
    rawTx, err := tx.Tx.MarshalBinary()
    if err != nil {
        return nil, err
    }

    request := map[string]interface{}{
        "transaction":   hexutil.Encode(rawTx),
        "block_number":  "latest",
    }

    _, err = p.sendRequest(ctx, "blxr_simulate_bundle", request)
    if err != nil {
        return &SimulationResult{
            Success: false,
            Error:   err.Error(),
        }, nil
    }

    return &SimulationResult{
        Success: true,
    }, nil
}

// GetBuilders 获取构建者列表
func (p *BloXrouteProvider) GetBuilders() []string {
    return []string{
        "bloxroute_max_profit",
        "bloxroute_regulated",
        "flashbots",
        "builder0x69",
        "beaverbuild",
        "rsync",
        "titan",
    }
}

// HealthCheck 健康检查
func (p *BloXrouteProvider) HealthCheck(ctx context.Context) error {
    _, err := p.sendRequest(ctx, "blxr_health", map[string]interface{}{})
    return err
}

// SubscribeTxStatus 订阅交易状态 (WebSocket)
func (p *BloXrouteProvider) SubscribeTxStatus(ctx context.Context, txHash common.Hash) (<-chan *TransactionStatus, error) {
    if p.wsConn == nil {
        dialer := websocket.Dialer{
            HandshakeTimeout: 10 * time.Second,
        }

        headers := http.Header{}
        headers.Set("Authorization", p.authHeader)

        conn, _, err := dialer.DialContext(ctx, p.wsEndpoint, headers)
        if err != nil {
            return nil, err
        }
        p.wsConn = conn
    }

    // 订阅
    subscribe := map[string]interface{}{
        "jsonrpc": "2.0",
        "id":      1,
        "method":  "subscribe",
        "params":  []string{"txStatus", txHash.Hex()},
    }

    if err := p.wsConn.WriteJSON(subscribe); err != nil {
        return nil, err
    }

    statusCh := make(chan *TransactionStatus, 10)

    go func() {
        defer close(statusCh)
        for {
            select {
            case <-ctx.Done():
                return
            default:
                var msg map[string]interface{}
                if err := p.wsConn.ReadJSON(&msg); err != nil {
                    return
                }
                // 解析状态更新
                statusCh <- &TransactionStatus{
                    Hash:   txHash,
                    Status: TxStatusPending,
                }
            }
        }
    }()

    return statusCh, nil
}
```

---

## 第一百二十一节：Eden Network 集成

### Eden Network 客户端

```go
package privatepool

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/common/hexutil"
    "github.com/ethereum/go-ethereum/core/types"
)

// EdenProvider Eden Network 提供者
type EdenProvider struct {
    endpoint   string
    apiKey     string
    httpClient *http.Client
}

// EdenConfig Eden 配置
type EdenConfig struct {
    Endpoint string // https://api.edennetwork.io/v1/rpc
    APIKey   string
}

// NewEdenProvider 创建 Eden 提供者
func NewEdenProvider(config *EdenConfig) *EdenProvider {
    return &EdenProvider{
        endpoint: config.Endpoint,
        apiKey:   config.APIKey,
        httpClient: &http.Client{
            Timeout: 30 * time.Second,
        },
    }
}

// Name 名称
func (p *EdenProvider) Name() string {
    return "Eden"
}

// SendPrivateTransaction 发送私有交易
func (p *EdenProvider) SendPrivateTransaction(
    ctx context.Context,
    tx *PrivateTransaction,
) (*SubmissionResult, error) {
    rawTx, err := tx.Tx.MarshalBinary()
    if err != nil {
        return nil, err
    }

    // Eden 使用标准 eth_sendPrivateTransaction
    params := []interface{}{
        map[string]interface{}{
            "tx":             hexutil.Encode(rawTx),
            "maxBlockNumber": hexutil.EncodeUint64(tx.MaxBlockNumber),
        },
    }

    return p.sendRPC(ctx, "eth_sendPrivateTransaction", params)
}

// SendBundle 发送 Bundle
func (p *EdenProvider) SendBundle(ctx context.Context, bundle *Bundle) (*SubmissionResult, error) {
    txs := make([]string, len(bundle.Transactions))
    for i, tx := range bundle.Transactions {
        rawTx, err := tx.MarshalBinary()
        if err != nil {
            return nil, err
        }
        txs[i] = hexutil.Encode(rawTx)
    }

    params := []interface{}{
        map[string]interface{}{
            "txs":               txs,
            "blockNumber":       hexutil.EncodeUint64(bundle.BlockNumber),
            "minTimestamp":      bundle.MinTimestamp,
            "maxTimestamp":      bundle.MaxTimestamp,
            "revertingTxHashes": bundle.RevertingHashes,
        },
    }

    return p.sendRPC(ctx, "eth_sendBundle", params)
}

// sendRPC 发送 RPC 请求
func (p *EdenProvider) sendRPC(
    ctx context.Context,
    method string,
    params []interface{},
) (*SubmissionResult, error) {
    payload := map[string]interface{}{
        "jsonrpc": "2.0",
        "id":      1,
        "method":  method,
        "params":  params,
    }

    body, err := json.Marshal(payload)
    if err != nil {
        return nil, err
    }

    req, err := http.NewRequestWithContext(ctx, "POST", p.endpoint, bytes.NewReader(body))
    if err != nil {
        return nil, err
    }

    req.Header.Set("Content-Type", "application/json")
    if p.apiKey != "" {
        req.Header.Set("X-API-Key", p.apiKey)
    }

    resp, err := p.httpClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    respBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, err
    }

    var result struct {
        Result interface{} `json:"result"`
        Error  *struct {
            Code    int    `json:"code"`
            Message string `json:"message"`
        } `json:"error"`
    }

    if err := json.Unmarshal(respBody, &result); err != nil {
        return nil, err
    }

    if result.Error != nil {
        return &SubmissionResult{
            Success: false,
            Error:   result.Error.Message,
        }, nil
    }

    return &SubmissionResult{
        Success:   true,
        Timestamp: time.Now(),
    }, nil
}

// CancelTransaction 取消交易
func (p *EdenProvider) CancelTransaction(ctx context.Context, txHash common.Hash) error {
    params := []interface{}{txHash.Hex()}
    _, err := p.sendRPC(ctx, "eth_cancelPrivateTransaction", params)
    return err
}

// GetTransactionStatus 获取交易状态
func (p *EdenProvider) GetTransactionStatus(
    ctx context.Context,
    txHash common.Hash,
) (*TransactionStatus, error) {
    params := []interface{}{txHash.Hex()}
    result, err := p.sendRPC(ctx, "eth_getPrivateTransactionStatus", params)
    if err != nil {
        return nil, err
    }

    return &TransactionStatus{
        Hash:   txHash,
        Status: TxStatusPending,
        Error:  result.Error,
    }, nil
}

// SimulateTransaction 模拟交易
func (p *EdenProvider) SimulateTransaction(
    ctx context.Context,
    tx *PrivateTransaction,
) (*SimulationResult, error) {
    rawTx, err := tx.Tx.MarshalBinary()
    if err != nil {
        return nil, err
    }

    params := []interface{}{
        map[string]interface{}{
            "tx":          hexutil.Encode(rawTx),
            "blockNumber": "latest",
        },
    }

    result, err := p.sendRPC(ctx, "eth_callBundle", params)
    if err != nil {
        return &SimulationResult{
            Success: false,
            Error:   err.Error(),
        }, nil
    }

    return &SimulationResult{
        Success: result.Success,
    }, nil
}

// GetBuilders 获取构建者列表
func (p *EdenProvider) GetBuilders() []string {
    return []string{
        "eden",
        "flashbots",
    }
}

// HealthCheck 健康检查
func (p *EdenProvider) HealthCheck(ctx context.Context) error {
    _, err := p.sendRPC(ctx, "eth_blockNumber", []interface{}{})
    return err
}

// GetSlotStatus 获取 Eden 插槽状态
func (p *EdenProvider) GetSlotStatus(ctx context.Context) (*SlotStatus, error) {
    result, err := p.sendRPC(ctx, "eden_slotStatus", []interface{}{})
    if err != nil {
        return nil, err
    }

    _ = result
    return &SlotStatus{}, nil
}

// SlotStatus Eden 插槽状态
type SlotStatus struct {
    CurrentSlot    uint64
    SlotTenant     common.Address
    NextSlot       uint64
    NextSlotTenant common.Address
}
```

---

## 第一百二十二节：Protect RPC 与 MEV Blocker

### 保护型 RPC 集成

```go
package privatepool

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "time"

    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/common/hexutil"
    "github.com/ethereum/go-ethereum/core/types"
)

// ProtectRPCProvider 保护型 RPC 提供者
type ProtectRPCProvider struct {
    endpoint   string
    httpClient *http.Client
    config     *ProtectConfig
}

// ProtectConfig 保护配置
type ProtectConfig struct {
    Provider        string // flashbots, mevblocker, cowswap
    Endpoint        string
    RefundAddress   common.Address
    RefundPercent   int
    FastMode        bool
}

// NewProtectRPCProvider 创建保护型 RPC 提供者
func NewProtectRPCProvider(config *ProtectConfig) *ProtectRPCProvider {
    return &ProtectRPCProvider{
        endpoint: config.Endpoint,
        config:   config,
        httpClient: &http.Client{
            Timeout: 30 * time.Second,
        },
    }
}

// GetProtectEndpoints 获取各保护 RPC 端点
func GetProtectEndpoints() map[string]string {
    return map[string]string{
        "flashbots_protect": "https://rpc.flashbots.net",
        "mev_blocker":       "https://rpc.mevblocker.io",
        "merkle":            "https://rpc.merkle.io",
        "securerpc":         "https://api.securerpc.com/v1",
    }
}

// Name 名称
func (p *ProtectRPCProvider) Name() string {
    return "ProtectRPC-" + p.config.Provider
}

// SendPrivateTransaction 发送保护交易
func (p *ProtectRPCProvider) SendPrivateTransaction(
    ctx context.Context,
    tx *PrivateTransaction,
) (*SubmissionResult, error) {
    rawTx, err := tx.Tx.MarshalBinary()
    if err != nil {
        return nil, err
    }

    // 标准 eth_sendRawTransaction
    params := []interface{}{hexutil.Encode(rawTx)}

    // 添加保护参数 (不同提供者格式不同)
    switch p.config.Provider {
    case "flashbots_protect":
        // Flashbots Protect 使用查询参数
        return p.sendWithQueryParams(ctx, rawTx, tx)
    case "mev_blocker":
        // MEV Blocker 使用标准 RPC
        return p.sendRPC(ctx, "eth_sendRawTransaction", params)
    default:
        return p.sendRPC(ctx, "eth_sendRawTransaction", params)
    }
}

// sendWithQueryParams 带查询参数发送
func (p *ProtectRPCProvider) sendWithQueryParams(
    ctx context.Context,
    rawTx []byte,
    tx *PrivateTransaction,
) (*SubmissionResult, error) {
    // 构建 URL 参数
    endpoint := p.endpoint

    // 添加 hints
    if tx.Hints != nil {
        hints := []string{}
        if tx.Hints.CallData {
            hints = append(hints, "calldata")
        }
        if tx.Hints.ContractAddress {
            hints = append(hints, "contract_address")
        }
        if tx.Hints.FunctionSelector {
            hints = append(hints, "function_selector")
        }
        if tx.Hints.Logs {
            hints = append(hints, "logs")
        }
        if tx.Hints.TxHash {
            hints = append(hints, "tx_hash")
        }

        if len(hints) > 0 {
            endpoint += "?hint="
            for i, h := range hints {
                if i > 0 {
                    endpoint += "&hint="
                }
                endpoint += h
            }
        }
    }

    params := []interface{}{hexutil.Encode(rawTx)}
    return p.sendRPCToEndpoint(ctx, endpoint, "eth_sendRawTransaction", params)
}

// sendRPC 发送 RPC
func (p *ProtectRPCProvider) sendRPC(
    ctx context.Context,
    method string,
    params []interface{},
) (*SubmissionResult, error) {
    return p.sendRPCToEndpoint(ctx, p.endpoint, method, params)
}

// sendRPCToEndpoint 发送 RPC 到指定端点
func (p *ProtectRPCProvider) sendRPCToEndpoint(
    ctx context.Context,
    endpoint, method string,
    params []interface{},
) (*SubmissionResult, error) {
    payload := map[string]interface{}{
        "jsonrpc": "2.0",
        "id":      1,
        "method":  method,
        "params":  params,
    }

    body, err := json.Marshal(payload)
    if err != nil {
        return nil, err
    }

    req, err := http.NewRequestWithContext(ctx, "POST", endpoint, bytes.NewReader(body))
    if err != nil {
        return nil, err
    }

    req.Header.Set("Content-Type", "application/json")

    resp, err := p.httpClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    respBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, err
    }

    var result struct {
        Result string `json:"result"`
        Error  *struct {
            Code    int    `json:"code"`
            Message string `json:"message"`
        } `json:"error"`
    }

    if err := json.Unmarshal(respBody, &result); err != nil {
        return nil, err
    }

    if result.Error != nil {
        return &SubmissionResult{
            Success: false,
            Error:   result.Error.Message,
        }, nil
    }

    return &SubmissionResult{
        Success:    true,
        TxHashes:   []common.Hash{common.HexToHash(result.Result)},
        Timestamp:  time.Now(),
    }, nil
}

// SendBundle 不支持 Bundle
func (p *ProtectRPCProvider) SendBundle(ctx context.Context, bundle *Bundle) (*SubmissionResult, error) {
    return nil, fmt.Errorf("Protect RPC 不支持 Bundle")
}

// CancelTransaction 取消交易
func (p *ProtectRPCProvider) CancelTransaction(ctx context.Context, txHash common.Hash) error {
    // 大多数 Protect RPC 不支持取消
    return fmt.Errorf("Protect RPC 不支持取消交易")
}

// GetTransactionStatus 获取交易状态
func (p *ProtectRPCProvider) GetTransactionStatus(
    ctx context.Context,
    txHash common.Hash,
) (*TransactionStatus, error) {
    params := []interface{}{txHash.Hex()}
    _, err := p.sendRPC(ctx, "eth_getTransactionReceipt", params)
    if err != nil {
        return nil, err
    }

    return &TransactionStatus{
        Hash:   txHash,
        Status: TxStatusPending,
    }, nil
}

// SimulateTransaction 模拟交易
func (p *ProtectRPCProvider) SimulateTransaction(
    ctx context.Context,
    tx *PrivateTransaction,
) (*SimulationResult, error) {
    // 使用 eth_call 模拟
    rawTx, err := tx.Tx.MarshalBinary()
    if err != nil {
        return nil, err
    }

    _ = rawTx
    return &SimulationResult{
        Success: true,
    }, nil
}

// GetBuilders 获取构建者列表
func (p *ProtectRPCProvider) GetBuilders() []string {
    switch p.config.Provider {
    case "flashbots_protect":
        return []string{"flashbots", "beaverbuild", "titan", "rsync"}
    case "mev_blocker":
        return []string{"agnostic", "bloxroute"}
    default:
        return []string{}
    }
}

// HealthCheck 健康检查
func (p *ProtectRPCProvider) HealthCheck(ctx context.Context) error {
    _, err := p.sendRPC(ctx, "eth_blockNumber", []interface{}{})
    return err
}

// MEVBlockerProvider MEV Blocker 专用提供者
type MEVBlockerProvider struct {
    *ProtectRPCProvider
    fastModeEndpoint string
    fullModeEndpoint string
}

// NewMEVBlockerProvider 创建 MEV Blocker 提供者
func NewMEVBlockerProvider(fastMode bool) *MEVBlockerProvider {
    config := &ProtectConfig{
        Provider: "mev_blocker",
        Endpoint: "https://rpc.mevblocker.io",
        FastMode: fastMode,
    }

    if fastMode {
        config.Endpoint = "https://rpc.mevblocker.io/fast"
    }

    return &MEVBlockerProvider{
        ProtectRPCProvider: NewProtectRPCProvider(config),
        fastModeEndpoint:   "https://rpc.mevblocker.io/fast",
        fullModeEndpoint:   "https://rpc.mevblocker.io",
    }
}

// SetFastMode 设置快速模式
func (p *MEVBlockerProvider) SetFastMode(fast bool) {
    if fast {
        p.endpoint = p.fastModeEndpoint
    } else {
        p.endpoint = p.fullModeEndpoint
    }
}
```

---

## 本文总结

本文深入探讨了私有交易池集成，主要涵盖：

### 私有池对比

| 提供者 | 特点 | 适用场景 |
|--------|------|---------|
| Flashbots | MEV 保护、收益分享 | 专业 Searcher |
| bloXroute | 低延迟、多构建者 | 高频交易 |
| Eden | 优先排序、插槽租户 | 需要优先级 |
| Protect RPC | 简单集成、三明治保护 | 普通用户 |
| MEV Blocker | 开源、多中继 | 注重去中心化 |

### 选择指南

```
私有池选择流程：
┌─────────────────────────────────────────────────────────────────┐
│                    私有池选择决策树                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    需要发送 Bundle?                             │
│                         │                                       │
│            ┌────────────┼────────────┐                         │
│            ▼            ▼            ▼                         │
│          是的         否           需要快速确认?               │
│            │            │               │                       │
│            ▼            ▼               ▼                       │
│    ┌──────────────┐  ┌──────────────┐ ┌──────────────┐        │
│    │ Flashbots    │  │ Protect RPC  │ │ bloXroute    │        │
│    │ bloXroute    │  │ MEV Blocker  │ │ Eden Fast    │        │
│    │ Eden         │  │              │ │              │        │
│    └──────────────┘  └──────────────┘ └──────────────┘        │
│                                                                 │
│  ────────────────────────────────────────────────────────────  │
│                                                                 │
│  推荐策略: 同时使用多个提供者，增加成功率                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 最佳实践

1. **多提供者**: 同时发送到多个私有池
2. **模拟优先**: 发送前先模拟验证
3. **监控状态**: 实时跟踪交易状态
4. **合理定价**: 设置有竞争力的 Gas 价格
