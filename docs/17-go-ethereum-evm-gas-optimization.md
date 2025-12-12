# Go-Ethereum EVM 深度解析：高级 Gas 优化

## 第一百一十节：EVM Gas 成本模型

### Gas 成本详解

```
EVM Gas 成本模型：
┌─────────────────────────────────────────────────────────────────────┐
│                       EVM Gas 成本结构                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  基础操作 Gas 成本                                                   │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  算术运算:     ADD/SUB: 3    MUL/DIV: 5    EXP: 10+        │    │
│  │  比较运算:     LT/GT/EQ: 3   ISZERO: 3                     │    │
│  │  位运算:       AND/OR/XOR: 3  SHL/SHR: 3                   │    │
│  │  栈操作:       PUSH: 3       POP: 2       DUP: 3           │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  存储操作 Gas 成本 (EIP-2929 后)                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  SLOAD:        冷访问: 2100   热访问: 100                   │    │
│  │  SSTORE:       0→非0: 22100  非0→非0: 2900  非0→0: 退款    │    │
│  │  BALANCE:      冷访问: 2600   热访问: 100                   │    │
│  │  EXTCODESIZE:  冷访问: 2600   热访问: 100                   │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  内存操作 Gas 成本                                                   │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  MLOAD/MSTORE:   3 + 内存扩展成本                          │    │
│  │  内存扩展:       memory_cost = words * 3 + words² / 512    │    │
│  │  MSTORE8:        3                                         │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  调用操作 Gas 成本                                                   │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  CALL:           冷: 2600 + 基础  热: 100 + 基础            │    │
│  │  STATICCALL:     同上                                       │    │
│  │  DELEGATECALL:   同上                                       │    │
│  │  CREATE:         32000                                      │    │
│  │  CREATE2:        32000 + 6 * 代码长度                       │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Gas 成本计算器

```go
package gasopt

import (
    "math/big"
)

// GasCosts EVM Gas 成本常量
var GasCosts = struct {
    // 基础操作
    Zero            uint64
    Base            uint64
    VeryLow         uint64
    Low             uint64
    Mid             uint64
    High            uint64

    // 存储操作 (EIP-2929)
    ColdSloadCost   uint64
    WarmStorageRead uint64
    SstoreSet       uint64
    SstoreReset     uint64
    SstoreClears    uint64

    // 内存操作
    Memory          uint64
    MemoryWord      uint64

    // 调用操作
    Call            uint64
    CallValue       uint64
    CallNewAccount  uint64
    ColdAccountAccess uint64
    WarmAccountAccess uint64

    // 日志操作
    Log             uint64
    LogData         uint64
    LogTopic        uint64

    // 创建操作
    Create          uint64
    Create2         uint64
    CodeDeposit     uint64

    // 特殊操作
    Keccak256       uint64
    Keccak256Word   uint64
    Copy            uint64
    Exp             uint64
    ExpByte         uint64
}{
    Zero:            0,
    Base:            2,
    VeryLow:         3,
    Low:             5,
    Mid:             8,
    High:            10,

    ColdSloadCost:   2100,
    WarmStorageRead: 100,
    SstoreSet:       22100,
    SstoreReset:     2900,
    SstoreClears:    4800,

    Memory:          3,
    MemoryWord:      3,

    Call:            0,
    CallValue:       9000,
    CallNewAccount:  25000,
    ColdAccountAccess: 2600,
    WarmAccountAccess: 100,

    Log:             375,
    LogData:         8,
    LogTopic:        375,

    Create:          32000,
    Create2:         32000,
    CodeDeposit:     200,

    Keccak256:       30,
    Keccak256Word:   6,
    Copy:            3,
    Exp:             10,
    ExpByte:         50,
}

// GasEstimator Gas 估算器
type GasEstimator struct {
    warmAccounts map[string]bool
    warmSlots    map[string]bool
}

// NewGasEstimator 创建估算器
func NewGasEstimator() *GasEstimator {
    return &GasEstimator{
        warmAccounts: make(map[string]bool),
        warmSlots:    make(map[string]bool),
    }
}

// EstimateSload 估算 SLOAD Gas
func (g *GasEstimator) EstimateSload(contract, slot string) uint64 {
    key := contract + ":" + slot
    if g.warmSlots[key] {
        return GasCosts.WarmStorageRead
    }
    g.warmSlots[key] = true
    return GasCosts.ColdSloadCost
}

// EstimateSstore 估算 SSTORE Gas
func (g *GasEstimator) EstimateSstore(contract, slot string, oldValue, newValue *big.Int) uint64 {
    key := contract + ":" + slot
    isWarm := g.warmSlots[key]
    g.warmSlots[key] = true

    var gas uint64 = 0

    // 冷访问成本
    if !isWarm {
        gas += GasCosts.ColdSloadCost
    }

    // 根据值变化计算成本
    oldIsZero := oldValue == nil || oldValue.Sign() == 0
    newIsZero := newValue == nil || newValue.Sign() == 0

    if oldIsZero && !newIsZero {
        // 0 -> 非0
        gas += GasCosts.SstoreSet
    } else if !oldIsZero && newIsZero {
        // 非0 -> 0 (有退款)
        gas += GasCosts.SstoreReset
    } else {
        // 非0 -> 非0
        gas += GasCosts.SstoreReset
    }

    return gas
}

// EstimateCall 估算 CALL Gas
func (g *GasEstimator) EstimateCall(to string, value *big.Int, isNew bool) uint64 {
    var gas uint64 = 0

    // 账户访问成本
    if g.warmAccounts[to] {
        gas += GasCosts.WarmAccountAccess
    } else {
        gas += GasCosts.ColdAccountAccess
        g.warmAccounts[to] = true
    }

    // 转账成本
    if value != nil && value.Sign() > 0 {
        gas += GasCosts.CallValue
    }

    // 新账户成本
    if isNew && value != nil && value.Sign() > 0 {
        gas += GasCosts.CallNewAccount
    }

    return gas
}

// EstimateMemory 估算内存扩展 Gas
func (g *GasEstimator) EstimateMemory(currentSize, newSize uint64) uint64 {
    if newSize <= currentSize {
        return 0
    }

    oldWords := (currentSize + 31) / 32
    newWords := (newSize + 31) / 32

    if newWords <= oldWords {
        return 0
    }

    oldCost := g.memoryGas(oldWords)
    newCost := g.memoryGas(newWords)

    return newCost - oldCost
}

// memoryGas 计算内存 Gas
func (g *GasEstimator) memoryGas(words uint64) uint64 {
    return words*GasCosts.MemoryWord + (words*words)/512
}

// EstimateKeccak256 估算 Keccak256 Gas
func (g *GasEstimator) EstimateKeccak256(dataSize uint64) uint64 {
    words := (dataSize + 31) / 32
    return GasCosts.Keccak256 + words*GasCosts.Keccak256Word
}

// EstimateLog 估算 LOG Gas
func (g *GasEstimator) EstimateLog(topics int, dataSize uint64) uint64 {
    return GasCosts.Log +
           uint64(topics)*GasCosts.LogTopic +
           dataSize*GasCosts.LogData
}

// EstimateCreate 估算 CREATE/CREATE2 Gas
func (g *GasEstimator) EstimateCreate(codeSize uint64, isCreate2 bool) uint64 {
    gas := GasCosts.Create
    if isCreate2 {
        words := (codeSize + 31) / 32
        gas += words * GasCosts.Keccak256Word
    }
    gas += codeSize * GasCosts.CodeDeposit
    return gas
}
```

---

## 第一百一十一节：Solidity Gas 优化技巧

### 合约优化模式

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

/// @title Gas 优化示例合约
/// @notice 展示各种 Gas 优化技巧
contract GasOptimizedContract {

    // ============ 存储优化 ============

    // 差: 使用多个 storage slot
    // uint256 public value1;  // slot 0
    // uint256 public value2;  // slot 1
    // uint256 public value3;  // slot 2

    // 好: 打包到同一个 slot
    struct PackedData {
        uint128 value1;  // 16 bytes
        uint64 value2;   // 8 bytes
        uint64 value3;   // 8 bytes
    }  // 总共 32 bytes = 1 slot

    PackedData public packedData;

    // 更进一步: 使用更小的类型
    struct TightlyPacked {
        uint32 timestamp;   // 4 bytes
        uint32 amount;      // 4 bytes
        uint16 fee;         // 2 bytes
        uint8 status;       // 1 byte
        bool active;        // 1 byte
        address user;       // 20 bytes
    }  // 总共 32 bytes = 1 slot

    // ============ 变量可见性优化 ============

    // 差: public 会自动生成 getter
    // uint256 public expensiveValue;

    // 好: internal/private 更便宜
    uint256 internal _value;

    // 如果需要读取，手动实现更优化的 getter
    function getValue() external view returns (uint256) {
        return _value;
    }

    // ============ 常量和不可变量 ============

    // 差: 存储变量
    // address public owner;

    // 好: immutable (部署时设置，存储在字节码中)
    address public immutable owner;

    // 最好: constant (编译时常量)
    uint256 public constant MAX_SUPPLY = 1000000;

    constructor() {
        owner = msg.sender;
    }

    // ============ 循环优化 ============

    uint256[] public values;

    // 差: 每次迭代都读取 storage
    function sumBad() external view returns (uint256 total) {
        for (uint256 i = 0; i < values.length; i++) {
            total += values[i];
        }
    }

    // 好: 缓存数组长度
    function sumGood() external view returns (uint256 total) {
        uint256 len = values.length;
        for (uint256 i = 0; i < len; i++) {
            total += values[i];
        }
    }

    // 更好: 使用 unchecked (如果确定不会溢出)
    function sumBest() external view returns (uint256 total) {
        uint256 len = values.length;
        for (uint256 i = 0; i < len; ) {
            total += values[i];
            unchecked { ++i; }
        }
    }

    // ============ 函数优化 ============

    // 差: 多次 storage 读取
    function processBad(uint256 amount) external {
        require(_value >= amount, "Insufficient");
        _value = _value - amount;
        // 更多使用 _value 的操作...
    }

    // 好: 缓存到 memory
    function processGood(uint256 amount) external {
        uint256 cached = _value;
        require(cached >= amount, "Insufficient");
        _value = cached - amount;
    }

    // ============ 错误处理优化 ============

    // 差: 长字符串错误消息
    // require(condition, "This is a very long error message that costs gas");

    // 好: 短字符串或自定义错误
    error InsufficientBalance(uint256 available, uint256 required);

    function withdrawOptimized(uint256 amount) external {
        uint256 balance = _value;
        if (balance < amount) {
            revert InsufficientBalance(balance, amount);
        }
        _value = balance - amount;
    }

    // ============ Calldata vs Memory ============

    // 差: memory 会复制数据
    function processBadArray(uint256[] memory data) external pure returns (uint256) {
        uint256 sum;
        for (uint256 i = 0; i < data.length; i++) {
            sum += data[i];
        }
        return sum;
    }

    // 好: calldata 直接引用输入
    function processGoodArray(uint256[] calldata data) external pure returns (uint256) {
        uint256 sum;
        uint256 len = data.length;
        for (uint256 i = 0; i < len; ) {
            sum += data[i];
            unchecked { ++i; }
        }
        return sum;
    }

    // ============ 位操作优化 ============

    // 差: 使用除法和取模
    // uint256 result = value / 2;
    // uint256 remainder = value % 2;

    // 好: 使用位移
    function divideByPowerOf2(uint256 value, uint8 power) external pure returns (uint256) {
        return value >> power;
    }

    function multiplyByPowerOf2(uint256 value, uint8 power) external pure returns (uint256) {
        return value << power;
    }

    function isOdd(uint256 value) external pure returns (bool) {
        return value & 1 == 1;
    }

    // ============ 批量操作 ============

    mapping(address => uint256) public balances;

    // 差: 多次单独转账
    function transfersBad(address[] calldata recipients, uint256[] calldata amounts) external {
        for (uint256 i = 0; i < recipients.length; i++) {
            balances[msg.sender] -= amounts[i];
            balances[recipients[i]] += amounts[i];
        }
    }

    // 好: 批量处理，减少 storage 操作
    function transfersGood(address[] calldata recipients, uint256[] calldata amounts) external {
        uint256 totalAmount;
        uint256 len = recipients.length;

        // 先计算总量
        for (uint256 i = 0; i < len; ) {
            totalAmount += amounts[i];
            unchecked { ++i; }
        }

        // 一次性扣除发送者余额
        uint256 senderBalance = balances[msg.sender];
        require(senderBalance >= totalAmount, "Insufficient");
        balances[msg.sender] = senderBalance - totalAmount;

        // 分别增加接收者余额
        for (uint256 i = 0; i < len; ) {
            balances[recipients[i]] += amounts[i];
            unchecked { ++i; }
        }
    }
}
```

### 汇编优化

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

/// @title 汇编优化示例
contract AssemblyOptimized {

    // ============ 直接操作 Storage ============

    function getSlot(uint256 slot) external view returns (bytes32 value) {
        assembly {
            value := sload(slot)
        }
    }

    function setSlot(uint256 slot, bytes32 value) external {
        assembly {
            sstore(slot, value)
        }
    }

    // ============ 高效的地址检查 ============

    function isContract(address account) external view returns (bool result) {
        assembly {
            result := gt(extcodesize(account), 0)
        }
    }

    // ============ 高效的哈希计算 ============

    function efficientHash(bytes calldata data) external pure returns (bytes32 result) {
        assembly {
            // 直接从 calldata 计算 hash
            let ptr := mload(0x40)
            calldatacopy(ptr, data.offset, data.length)
            result := keccak256(ptr, data.length)
        }
    }

    // ============ 批量读取 ============

    function batchBalanceOf(
        address token,
        address[] calldata accounts
    ) external view returns (uint256[] memory results) {
        results = new uint256[](accounts.length);

        // balanceOf(address) selector
        bytes4 selector = 0x70a08231;

        assembly {
            let resultsPtr := add(results, 0x20)
            let accountsLen := accounts.length

            for { let i := 0 } lt(i, accountsLen) { i := add(i, 1) } {
                // 构造 calldata
                let ptr := mload(0x40)
                mstore(ptr, selector)
                mstore(add(ptr, 0x04), calldataload(add(accounts.offset, mul(i, 0x20))))

                // 调用
                let success := staticcall(gas(), token, ptr, 0x24, ptr, 0x20)

                if success {
                    mstore(add(resultsPtr, mul(i, 0x20)), mload(ptr))
                }
            }
        }
    }

    // ============ 高效的多调用 ============

    struct Call {
        address target;
        bytes callData;
    }

    function multicall(Call[] calldata calls) external returns (bytes[] memory results) {
        results = new bytes[](calls.length);

        for (uint256 i = 0; i < calls.length; ) {
            (bool success, bytes memory result) = calls[i].target.call(calls[i].callData);
            require(success, "Call failed");
            results[i] = result;
            unchecked { ++i; }
        }
    }

    // 更优化的 multicall (固定返回大小)
    function multicallFixed(
        address[] calldata targets,
        bytes[] calldata datas
    ) external returns (uint256[] memory results) {
        uint256 len = targets.length;
        results = new uint256[](len);

        assembly {
            let resultsPtr := add(results, 0x20)

            for { let i := 0 } lt(i, len) { i := add(i, 1) } {
                // 获取 target
                let target := calldataload(add(targets.offset, mul(i, 0x20)))

                // 获取 data offset 和 length
                let dataOffset := calldataload(add(datas.offset, mul(i, 0x20)))
                let dataPtr := add(datas.offset, dataOffset)
                let dataLen := calldataload(dataPtr)
                dataPtr := add(dataPtr, 0x20)

                // 复制 calldata
                let ptr := mload(0x40)
                calldatacopy(ptr, dataPtr, dataLen)

                // 调用
                let success := call(gas(), target, 0, ptr, dataLen, ptr, 0x20)

                if success {
                    mstore(add(resultsPtr, mul(i, 0x20)), mload(ptr))
                }
            }
        }
    }

    // ============ 零拷贝返回 ============

    function returnCalldata(bytes calldata data) external pure returns (bytes calldata) {
        return data;
    }

    // ============ 高效的条件检查 ============

    function efficientRequire(bool condition) external pure {
        assembly {
            if iszero(condition) {
                revert(0, 0)
            }
        }
    }

    // 带错误信息的高效 require
    function efficientRequireWithMessage(bool condition) external pure {
        assembly {
            if iszero(condition) {
                // 存储错误选择器: Error(string)
                mstore(0x00, 0x08c379a000000000000000000000000000000000000000000000000000000000)
                mstore(0x04, 0x20)
                mstore(0x24, 5)
                mstore(0x44, "Error")
                revert(0x00, 0x64)
            }
        }
    }
}
```

---

## 第一百一十二节：Go 层面的 Gas 优化

### 交易构建优化

```go
package gasopt

import (
    "context"
    "encoding/hex"
    "fmt"
    "math/big"

    "github.com/ethereum/go-ethereum"
    "github.com/ethereum/go-ethereum/common"
    "github.com/ethereum/go-ethereum/core/types"
    "github.com/ethereum/go-ethereum/ethclient"
)

// TransactionOptimizer 交易优化器
type TransactionOptimizer struct {
    client *ethclient.Client
}

// NewTransactionOptimizer 创建优化器
func NewTransactionOptimizer(client *ethclient.Client) *TransactionOptimizer {
    return &TransactionOptimizer{client: client}
}

// OptimizeCalldata 优化 calldata
func (o *TransactionOptimizer) OptimizeCalldata(data []byte) []byte {
    // 统计零字节和非零字节
    zeroBytes := 0
    nonZeroBytes := 0

    for _, b := range data {
        if b == 0 {
            zeroBytes++
        } else {
            nonZeroBytes++
        }
    }

    // calldata 成本: 零字节 4 gas, 非零字节 16 gas
    currentCost := zeroBytes*4 + nonZeroBytes*16

    fmt.Printf("Calldata 成本: %d gas (零字节: %d, 非零字节: %d)\n",
        currentCost, zeroBytes, nonZeroBytes)

    return data
}

// EstimateOptimalGas 估算最优 Gas
func (o *TransactionOptimizer) EstimateOptimalGas(
    ctx context.Context,
    from, to common.Address,
    data []byte,
    value *big.Int,
) (*GasEstimate, error) {
    // 估算 Gas
    msg := ethereum.CallMsg{
        From:  from,
        To:    &to,
        Data:  data,
        Value: value,
    }

    gasEstimate, err := o.client.EstimateGas(ctx, msg)
    if err != nil {
        return nil, err
    }

    // 获取当前 Gas 价格
    gasPrice, err := o.client.SuggestGasPrice(ctx)
    if err != nil {
        return nil, err
    }

    // 获取 baseFee 和 优先费
    header, err := o.client.HeaderByNumber(ctx, nil)
    if err != nil {
        return nil, err
    }

    baseFee := header.BaseFee
    suggestedTip, err := o.client.SuggestGasTipCap(ctx)
    if err != nil {
        return nil, err
    }

    // 计算最优参数
    return &GasEstimate{
        GasLimit:         gasEstimate,
        BaseFee:          baseFee,
        MaxPriorityFee:   suggestedTip,
        MaxFee:           new(big.Int).Add(new(big.Int).Mul(baseFee, big.NewInt(2)), suggestedTip),
        LegacyGasPrice:   gasPrice,
        EstimatedCostWei: new(big.Int).Mul(gasPrice, big.NewInt(int64(gasEstimate))),
    }, nil
}

// GasEstimate Gas 估算结果
type GasEstimate struct {
    GasLimit         uint64
    BaseFee          *big.Int
    MaxPriorityFee   *big.Int
    MaxFee           *big.Int
    LegacyGasPrice   *big.Int
    EstimatedCostWei *big.Int
}

// BuildOptimizedTx 构建优化交易
func (o *TransactionOptimizer) BuildOptimizedTx(
    ctx context.Context,
    from, to common.Address,
    data []byte,
    value *big.Int,
    nonce uint64,
) (*types.Transaction, error) {
    estimate, err := o.EstimateOptimalGas(ctx, from, to, data, value)
    if err != nil {
        return nil, err
    }

    chainID, err := o.client.ChainID(ctx)
    if err != nil {
        return nil, err
    }

    // 使用 EIP-1559 交易
    tx := types.NewTx(&types.DynamicFeeTx{
        ChainID:   chainID,
        Nonce:     nonce,
        GasTipCap: estimate.MaxPriorityFee,
        GasFeeCap: estimate.MaxFee,
        Gas:       estimate.GasLimit + estimate.GasLimit/10, // 10% 余量
        To:        &to,
        Value:     value,
        Data:      data,
    })

    return tx, nil
}

// CalldataBuilder 高效的 Calldata 构建器
type CalldataBuilder struct {
    data []byte
}

// NewCalldataBuilder 创建构建器
func NewCalldataBuilder(selector string) *CalldataBuilder {
    selectorBytes, _ := hex.DecodeString(selector)
    return &CalldataBuilder{
        data: selectorBytes,
    }
}

// AddAddress 添加地址参数
func (b *CalldataBuilder) AddAddress(addr common.Address) *CalldataBuilder {
    param := make([]byte, 32)
    copy(param[12:], addr.Bytes())
    b.data = append(b.data, param...)
    return b
}

// AddUint256 添加 uint256 参数
func (b *CalldataBuilder) AddUint256(value *big.Int) *CalldataBuilder {
    param := make([]byte, 32)
    if value != nil {
        value.FillBytes(param)
    }
    b.data = append(b.data, param...)
    return b
}

// AddBytes 添加 bytes 参数
func (b *CalldataBuilder) AddBytes(data []byte) *CalldataBuilder {
    // 计算偏移量
    currentLen := len(b.data) - 4 // 减去 selector
    offset := make([]byte, 32)
    big.NewInt(int64(currentLen + 32)).FillBytes(offset)
    b.data = append(b.data, offset...)

    // 长度
    length := make([]byte, 32)
    big.NewInt(int64(len(data))).FillBytes(length)
    b.data = append(b.data, length...)

    // 数据 (填充到 32 字节边界)
    paddedLen := ((len(data) + 31) / 32) * 32
    padded := make([]byte, paddedLen)
    copy(padded, data)
    b.data = append(b.data, padded...)

    return b
}

// Build 构建最终 calldata
func (b *CalldataBuilder) Build() []byte {
    return b.data
}

// PackedEncoder 紧凑编码器 (非标准 ABI)
type PackedEncoder struct {
    data []byte
}

// NewPackedEncoder 创建紧凑编码器
func NewPackedEncoder() *PackedEncoder {
    return &PackedEncoder{
        data: make([]byte, 0),
    }
}

// AddUint8 添加 uint8
func (e *PackedEncoder) AddUint8(v uint8) *PackedEncoder {
    e.data = append(e.data, v)
    return e
}

// AddUint16 添加 uint16
func (e *PackedEncoder) AddUint16(v uint16) *PackedEncoder {
    e.data = append(e.data, byte(v>>8), byte(v))
    return e
}

// AddUint24 添加 uint24
func (e *PackedEncoder) AddUint24(v uint32) *PackedEncoder {
    e.data = append(e.data, byte(v>>16), byte(v>>8), byte(v))
    return e
}

// AddAddress 添加地址 (20 bytes)
func (e *PackedEncoder) AddAddress(addr common.Address) *PackedEncoder {
    e.data = append(e.data, addr.Bytes()...)
    return e
}

// Build 构建
func (e *PackedEncoder) Build() []byte {
    return e.data
}

// 示例: 紧凑编码的 swap 路径
// 标准 ABI: 每个地址 32 bytes, 每个 fee 32 bytes
// 紧凑编码: 每个地址 20 bytes, 每个 fee 3 bytes
func BuildPackedSwapPath(tokens []common.Address, fees []uint32) []byte {
    encoder := NewPackedEncoder()

    for i, token := range tokens {
        encoder.AddAddress(token)
        if i < len(fees) {
            encoder.AddUint24(fees[i])
        }
    }

    return encoder.Build()
}
```

### Gas 监控和策略

```go
package gasopt

import (
    "context"
    "math/big"
    "sort"
    "sync"
    "time"

    "github.com/ethereum/go-ethereum/ethclient"
)

// GasMonitor Gas 监控器
type GasMonitor struct {
    client *ethclient.Client

    // 历史数据
    history     []GasDataPoint
    historyMu   sync.RWMutex
    maxHistory  int

    // 当前状态
    currentGas  *GasState
    stateMu     sync.RWMutex
}

// GasDataPoint Gas 数据点
type GasDataPoint struct {
    Timestamp     time.Time
    BlockNumber   uint64
    BaseFee       *big.Int
    GasUsedRatio  float64
    PriorityFees  []*big.Int
}

// GasState 当前 Gas 状态
type GasState struct {
    BaseFee         *big.Int
    SafeLow         *big.Int // 慢速
    Standard        *big.Int // 标准
    Fast            *big.Int // 快速
    Instant         *big.Int // 即时
    PendingCount    uint64
    NextBaseFee     *big.Int
}

// NewGasMonitor 创建监控器
func NewGasMonitor(client *ethclient.Client) *GasMonitor {
    return &GasMonitor{
        client:     client,
        history:    make([]GasDataPoint, 0),
        maxHistory: 1000,
    }
}

// Start 启动监控
func (m *GasMonitor) Start(ctx context.Context) {
    go m.monitorLoop(ctx)
}

// monitorLoop 监控循环
func (m *GasMonitor) monitorLoop(ctx context.Context) {
    ticker := time.NewTicker(12 * time.Second) // 每个区块
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            m.updateGasState(ctx)
        }
    }
}

// updateGasState 更新 Gas 状态
func (m *GasMonitor) updateGasState(ctx context.Context) {
    // 获取最新区块
    block, err := m.client.BlockByNumber(ctx, nil)
    if err != nil {
        return
    }

    // 获取待处理交易
    pendingCount, _ := m.client.PendingTransactionCount(ctx)

    // 收集区块内的优先费
    var priorityFees []*big.Int
    for _, tx := range block.Transactions() {
        if tx.Type() == 2 { // EIP-1559
            priorityFees = append(priorityFees, tx.GasTipCap())
        }
    }

    // 排序
    sort.Slice(priorityFees, func(i, j int) bool {
        return priorityFees[i].Cmp(priorityFees[j]) < 0
    })

    // 计算百分位
    baseFee := block.BaseFee()
    state := &GasState{
        BaseFee:      baseFee,
        PendingCount: pendingCount,
    }

    if len(priorityFees) > 0 {
        state.SafeLow = m.percentile(priorityFees, 10)
        state.Standard = m.percentile(priorityFees, 50)
        state.Fast = m.percentile(priorityFees, 75)
        state.Instant = m.percentile(priorityFees, 95)
    } else {
        defaultTip := big.NewInt(1e9) // 1 Gwei
        state.SafeLow = defaultTip
        state.Standard = new(big.Int).Mul(defaultTip, big.NewInt(2))
        state.Fast = new(big.Int).Mul(defaultTip, big.NewInt(3))
        state.Instant = new(big.Int).Mul(defaultTip, big.NewInt(5))
    }

    // 预测下一个 baseFee
    gasUsed := block.GasUsed()
    gasLimit := block.GasLimit()
    state.NextBaseFee = m.predictNextBaseFee(baseFee, gasUsed, gasLimit)

    // 更新状态
    m.stateMu.Lock()
    m.currentGas = state
    m.stateMu.Unlock()

    // 记录历史
    m.recordHistory(block.NumberU64(), baseFee, float64(gasUsed)/float64(gasLimit), priorityFees)
}

// percentile 计算百分位
func (m *GasMonitor) percentile(values []*big.Int, p int) *big.Int {
    if len(values) == 0 {
        return big.NewInt(0)
    }

    index := len(values) * p / 100
    if index >= len(values) {
        index = len(values) - 1
    }

    return new(big.Int).Set(values[index])
}

// predictNextBaseFee 预测下一个 baseFee
func (m *GasMonitor) predictNextBaseFee(currentBaseFee *big.Int, gasUsed, gasLimit uint64) *big.Int {
    // EIP-1559 公式
    // nextBaseFee = currentBaseFee * (1 + 0.125 * (gasUsed - target) / target)
    // target = gasLimit / 2

    target := gasLimit / 2

    var delta *big.Int
    if gasUsed > target {
        // 增加
        diff := new(big.Int).SetUint64(gasUsed - target)
        delta = new(big.Int).Mul(currentBaseFee, diff)
        delta.Div(delta, new(big.Int).SetUint64(target))
        delta.Div(delta, big.NewInt(8)) // / 8 = * 0.125
    } else if gasUsed < target {
        // 减少
        diff := new(big.Int).SetUint64(target - gasUsed)
        delta = new(big.Int).Mul(currentBaseFee, diff)
        delta.Div(delta, new(big.Int).SetUint64(target))
        delta.Div(delta, big.NewInt(8))
        delta.Neg(delta)
    } else {
        delta = big.NewInt(0)
    }

    nextBaseFee := new(big.Int).Add(currentBaseFee, delta)

    // 最小值
    if nextBaseFee.Sign() <= 0 {
        nextBaseFee = big.NewInt(1)
    }

    return nextBaseFee
}

// recordHistory 记录历史
func (m *GasMonitor) recordHistory(
    blockNumber uint64,
    baseFee *big.Int,
    gasUsedRatio float64,
    priorityFees []*big.Int,
) {
    m.historyMu.Lock()
    defer m.historyMu.Unlock()

    m.history = append(m.history, GasDataPoint{
        Timestamp:    time.Now(),
        BlockNumber:  blockNumber,
        BaseFee:      new(big.Int).Set(baseFee),
        GasUsedRatio: gasUsedRatio,
        PriorityFees: priorityFees,
    })

    // 保持最大长度
    if len(m.history) > m.maxHistory {
        m.history = m.history[1:]
    }
}

// GetCurrentState 获取当前状态
func (m *GasMonitor) GetCurrentState() *GasState {
    m.stateMu.RLock()
    defer m.stateMu.RUnlock()

    if m.currentGas == nil {
        return nil
    }

    return &GasState{
        BaseFee:      new(big.Int).Set(m.currentGas.BaseFee),
        SafeLow:      new(big.Int).Set(m.currentGas.SafeLow),
        Standard:     new(big.Int).Set(m.currentGas.Standard),
        Fast:         new(big.Int).Set(m.currentGas.Fast),
        Instant:      new(big.Int).Set(m.currentGas.Instant),
        PendingCount: m.currentGas.PendingCount,
        NextBaseFee:  new(big.Int).Set(m.currentGas.NextBaseFee),
    }
}

// GetOptimalGasPrice 获取最优 Gas 价格
func (m *GasMonitor) GetOptimalGasPrice(urgency GasUrgency) (*big.Int, *big.Int) {
    state := m.GetCurrentState()
    if state == nil {
        return big.NewInt(30e9), big.NewInt(2e9) // 默认值
    }

    var priorityFee *big.Int
    switch urgency {
    case UrgencySafeLow:
        priorityFee = state.SafeLow
    case UrgencyStandard:
        priorityFee = state.Standard
    case UrgencyFast:
        priorityFee = state.Fast
    case UrgencyInstant:
        priorityFee = state.Instant
    default:
        priorityFee = state.Standard
    }

    // maxFee = 2 * nextBaseFee + priorityFee
    maxFee := new(big.Int).Mul(state.NextBaseFee, big.NewInt(2))
    maxFee.Add(maxFee, priorityFee)

    return maxFee, priorityFee
}

// GasUrgency Gas 紧急程度
type GasUrgency int

const (
    UrgencySafeLow GasUrgency = iota
    UrgencyStandard
    UrgencyFast
    UrgencyInstant
)

// ShouldWaitForLowerGas 判断是否应该等待更低的 Gas
func (m *GasMonitor) ShouldWaitForLowerGas(maxAcceptableGwei float64) bool {
    state := m.GetCurrentState()
    if state == nil {
        return false
    }

    currentGwei := new(big.Float).SetInt(state.BaseFee)
    currentGwei.Quo(currentGwei, big.NewFloat(1e9))
    currentGweiFloat, _ := currentGwei.Float64()

    return currentGweiFloat > maxAcceptableGwei
}

// GetHistoricalAverage 获取历史平均值
func (m *GasMonitor) GetHistoricalAverage(hours int) *big.Int {
    m.historyMu.RLock()
    defer m.historyMu.RUnlock()

    cutoff := time.Now().Add(-time.Duration(hours) * time.Hour)

    var sum *big.Int = big.NewInt(0)
    var count int64 = 0

    for _, point := range m.history {
        if point.Timestamp.After(cutoff) {
            sum.Add(sum, point.BaseFee)
            count++
        }
    }

    if count == 0 {
        return big.NewInt(0)
    }

    return new(big.Int).Div(sum, big.NewInt(count))
}
```

---

## 本文总结

本文深入探讨了 EVM Gas 优化技术，主要涵盖：

### Gas 成本速查表

| 操作 | Gas 成本 | 优化建议 |
|------|---------|---------|
| SLOAD (冷) | 2100 | 缓存到 memory |
| SLOAD (热) | 100 | - |
| SSTORE (0→非0) | 22100 | 避免存储 |
| SSTORE (非0→非0) | 2900 | 批量更新 |
| CALL (冷) | 2600 | 预热账户 |
| 零字节 calldata | 4 | 使用更多零字节 |
| 非零字节 calldata | 16 | 紧凑编码 |

### 优化策略

```
Gas 优化层次：
┌─────────────────────────────────────────────────────────────────┐
│                       Gas 优化策略                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  合约层                                                          │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ • 存储打包    • 常量/不可变量   • 短字符串错误         │    │
│  │ • unchecked   • calldata       • 汇编优化             │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                 │
│  交易层                                                          │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ • 紧凑 calldata • EIP-1559     • Gas 价格监控         │    │
│  │ • 批量操作      • 时机选择     • 预热账户/存储        │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 最佳实践

1. **存储优化**: 使用紧凑存储布局，最小化 SLOAD/SSTORE
2. **计算优化**: 使用 unchecked，位操作代替乘除
3. **Calldata 优化**: 使用紧凑编码，最大化零字节
4. **时机优化**: 监控 Gas 价格，选择低峰期执行
