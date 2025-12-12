# Go-Ethereum EVM 详解（八）：EVM 未来演进

## 目录

- [57. EVM 演进路线](#57-evm-演进路线)
- [58. EOF（EVM Object Format）](#58-eofeum-object-format)
- [59. Verkle Trees](#59-verkle-trees)
- [60. Account Abstraction（AA）](#60-account-abstractionaa)
- [61. EIP-4844 与 Danksharding](#61-eip-4844-与-danksharding)
- [62. 其他重要 EIP](#62-其他重要-eip)
- [63. 对 MEV 的影响](#63-对-mev-的影响)
- [64. 学习资源与总结](#64-学习资源与总结)

---

## 57. EVM 演进路线

### 57.1 以太坊路线图

```
┌─────────────────────────────────────────────────────────────────┐
│                   以太坊发展路线图                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  The Merge (2022.09) ✓                                  │   │
│  │  • PoW → PoS                                            │   │
│  │  • 能源消耗降低 99.95%                                  │   │
│  │  • 质押验证                                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  The Surge (进行中)                                     │   │
│  │  • EIP-4844 (Proto-danksharding) ✓                     │   │
│  │  • Full Danksharding (未来)                            │   │
│  │  • 目标：100k+ TPS (with L2s)                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  The Scourge (规划中)                                   │   │
│  │  • MEV 缓解                                             │   │
│  │  • PBS (Proposer-Builder Separation) 协议内置          │   │
│  │  • Inclusion Lists                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  The Verge (规划中)                                     │   │
│  │  • Verkle Trees                                         │   │
│  │  • 无状态客户端                                         │   │
│  │  • 状态过期                                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  The Purge (规划中)                                     │   │
│  │  • 历史数据过期                                         │   │
│  │  • 状态过期                                             │   │
│  │  • 简化协议                                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           │                                     │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  The Splurge                                            │   │
│  │  • EVM 改进 (EOF)                                       │   │
│  │  • Account Abstraction                                  │   │
│  │  • 其他优化                                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 57.2 近期重要升级

```
┌─────────────────────────────────────────────────────────────────┐
│                    近期/即将到来的升级                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Shanghai (2023.04) ✓                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • EIP-4895: Beacon chain withdrawals                   │   │
│  │  • EIP-3651: Warm COINBASE                              │   │
│  │  • EIP-3855: PUSH0 instruction                          │   │
│  │  • EIP-3860: Limit initcode size                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Cancun/Deneb (2024.03) ✓                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • EIP-4844: Blob transactions (Proto-danksharding)     │   │
│  │  • EIP-1153: Transient storage (TLOAD/TSTORE)          │   │
│  │  • EIP-4788: Beacon block root in EVM                   │   │
│  │  • EIP-5656: MCOPY instruction                          │   │
│  │  • EIP-6780: SELFDESTRUCT only in same tx               │   │
│  │  • EIP-7516: BLOBBASEFEE opcode                         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Prague/Electra (预计 2025)                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • EIP-7702: Set EOA account code                       │   │
│  │  • EIP-2537: BLS12-381 precompiles                      │   │
│  │  • EIP-2935: Historical block hashes in state          │   │
│  │  • PeerDAS (数据可用性采样)                             │   │
│  │  • EOF (可能)                                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 58. EOF（EVM Object Format）

### 58.1 EOF 概述

```
┌─────────────────────────────────────────────────────────────────┐
│                        EOF 概述                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EOF = EVM Object Format（EVM 对象格式）                        │
│                                                                 │
│  目标：                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. 代码与数据分离                                       │   │
│  │  2. 部署时验证（而非运行时）                             │   │
│  │  3. 禁用动态跳转（JUMP/JUMPI → 静态跳转）               │   │
│  │  4. 更好的版本控制                                       │   │
│  │  5. 为未来优化铺路（JIT 编译等）                        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  当前问题：                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Legacy EVM:                                             │   │
│  │  • 代码和数据混合                                        │   │
│  │  • 动态跳转难以分析                                      │   │
│  │  • 运行时才验证指令                                      │   │
│  │  • 无法确定函数边界                                      │   │
│  │                                                          │   │
│  │  例如：代码末尾的 metadata 可能被误解析为指令           │   │
│  │  PUSH1 0x00 PUSH1 0x20 ... <metadata bytes>             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 58.2 EOF 格式结构

```
┌─────────────────────────────────────────────────────────────────┐
│                     EOF 容器格式                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  EOF 容器结构：                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  +----------------+                                      │   │
│  │  | Magic (0xEF00) |  2 bytes - EOF 标识                 │   │
│  │  +----------------+                                      │   │
│  │  | Version (0x01) |  1 byte - 版本号                    │   │
│  │  +----------------+                                      │   │
│  │  | Type Section   |  类型信息                           │   │
│  │  +----------------+                                      │   │
│  │  | Code Section 1 |  代码段 1                           │   │
│  │  +----------------+                                      │   │
│  │  | Code Section 2 |  代码段 2 (可选)                    │   │
│  │  +----------------+                                      │   │
│  │  | ...            |                                      │   │
│  │  +----------------+                                      │   │
│  │  | Data Section   |  数据段                             │   │
│  │  +----------------+                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Header 格式：                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  0xEF00           - Magic                               │   │
│  │  0x01             - Version                             │   │
│  │  0x01             - Kind (types)                        │   │
│  │  <size>           - Types section size                  │   │
│  │  0x02             - Kind (code)                         │   │
│  │  <num_sections>   - Number of code sections             │   │
│  │  <size1>...<sizeN>- Code section sizes                  │   │
│  │  0x04             - Kind (data)                         │   │
│  │  <size>           - Data section size                   │   │
│  │  0x00             - Terminator                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 58.3 EOF 新操作码

```go
// EOF 新增操作码
const (
// 相对跳转（替代 JUMP/JUMPI）
RJUMP = 0xe0  // 无条件相对跳转
RJUMPI = 0xe1 // 条件相对跳转
RJUMPV     = 0xe2 // 跳转表

// 函数调用
CALLF = 0xe3 // 调用函数
RETF = 0xe4  // 从函数返回
JUMPF = 0xe5 // 尾调用

// 数据操作
DATALOAD = 0xd0 // 从数据段加载
DATALOADN  = 0xd1 // 从数据段加载（立即数偏移）
DATASIZE = 0xd2   // 数据段大小
DATACOPY = 0xd3  // 复制数据段

// 创建操作
EOFCREATE = 0xec      // 创建 EOF 合约
RETURNCONTRACT = 0xee // 返回新合约代码

// 其他
EXTCALL    = 0xf8 // 外部调用（EOF 版本）
EXTDELEGATECALL = 0xf9
EXTSTATICCALL = 0xfb
RETURNDATALOAD = 0xf7
)

// EOF 代码示例
type EOFContainer struct {
Magic      [2]byte // 0xEF00
Version    byte    // 0x01
TypesSize  uint16
CodeSizes  []uint16
DataSize   uint16

Types      []FunctionType
CodeSections [][]byte
Data       []byte
}

type FunctionType struct {
Inputs     uint8  // 输入参数数量
Outputs    uint8  // 输出参数数量
MaxStack   uint16 // 最大栈深度
}

// 验证 EOF 容器
func ValidateEOF(container []byte) error {
// 1. 检查 magic
if len(container) < 2 || container[0] != 0xEF || container[1] != 0x00 {
return errors.New("invalid EOF magic")
}

// 2. 检查版本
if container[2] != 0x01 {
return errors.New("unsupported EOF version")
}

// 3. 解析并验证 header
// 4. 验证代码段（静态分析）
// 5. 验证类型信息与代码匹配

return nil
}
```

### 58.4 EOF 对开发的影响

```go
// EOF 对智能合约开发的影响

// 1. 禁用动态跳转
// 之前：
// PUSH1 destination
// JUMP

// EOF：
// RJUMP offset  (相对跳转)

// 2. 函数调用更清晰
// 之前：复杂的 JUMP/JUMPDEST 模式
// EOF：CALLF/RETF 明确的函数边界

// 3. 数据访问
// 之前：CODECOPY 混合代码和数据
// EOF：DATALOAD 专门访问数据段

// 示例：EOF 合约结构
/*
contract Example {
    uint256 public value;

    function setValue(uint256 _value) public {
        value = _value;
    }
}

EOF 编译后：
- Code Section 0: constructor
- Code Section 1: setValue 函数
- Code Section 2: value getter
- Data Section: metadata
*/

// EOF 验证器实现
type EOFValidator struct {
container *EOFContainer
}

func (v *EOFValidator) Validate() error {
// 1. 验证所有操作码有效
for i, code := range v.container.CodeSections {
if err := v.validateCodeSection(i, code); err != nil {
return err
}
}

// 2. 验证栈操作
for i, code := range v.container.CodeSections {
if err := v.validateStack(i, code); err != nil {
return err
}
}

// 3. 验证跳转目标
for i, code := range v.container.CodeSections {
if err := v.validateJumps(i, code); err != nil {
return err
}
}

return nil
}

func (v *EOFValidator) validateCodeSection(idx int, code []byte) error {
pc := 0
for pc < len(code) {
op := code[pc]

// 检查操作码是否在 EOF 中有效
if !isValidEOFOpcode(op) {
return fmt.Errorf("invalid opcode 0x%02x at pc %d in section %d", op, pc, idx)
}

// 禁止旧版跳转指令
if op == JUMP || op == JUMPI || op == PC || op == JUMPDEST {
return fmt.Errorf("legacy jump opcode not allowed in EOF")
}

// 移动到下一条指令
pc += opcodeSize(op)
}
return nil
}
```

---

## 59. Verkle Trees

### 59.1 Verkle Trees 概述

```
┌─────────────────────────────────────────────────────────────────┐
│                   Verkle Trees 概述                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  当前：Merkle Patricia Trie                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  特点：                                                  │   │
│  │  • 证明大小：~3KB（平均）                               │   │
│  │  • 分支因子：16                                         │   │
│  │  • 需要所有兄弟节点哈希                                 │   │
│  │                                                          │   │
│  │           Root                                          │   │
│  │          /    \                                         │   │
│  │         A      B    ◄── 需要整个路径的兄弟节点          │   │
│  │        / \    / \                                       │   │
│  │       C   D  E   F                                      │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  未来：Verkle Trees                                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  特点：                                                  │   │
│  │  • 证明大小：~150 bytes（固定）                         │   │
│  │  • 分支因子：256                                        │   │
│  │  • 使用 KZG 承诺（多项式承诺）                          │   │
│  │                                                          │   │
│  │           Root                                          │   │
│  │        /  |  |  \                                       │   │
│  │       ... ... ...   ◄── 只需要一个证明，不需要兄弟      │   │
│  │            |                                            │   │
│  │          Leaf                                           │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  优势：                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. 证明更小（20倍压缩）                                │   │
│  │  2. 启用无状态客户端                                    │   │
│  │  3. 更快的同步                                          │   │
│  │  4. 降低节点存储要求                                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 59.2 Verkle Trees 技术细节

```go
// Verkle Tree 节点结构
type VerkleNode interface {
Commit() *Commitment // KZG 承诺
Get(key []byte) ([]byte, error)
Insert(key, value []byte) error
}

// 内部节点
type VerkleInternalNode struct {
children   [256]VerkleNode // 256 个子节点
commitment *Commitment
depth      int
}

// 叶子节点
type VerkleLeafNode struct {
stem       []byte // 31 bytes stem
values     [256][]byte // 256 个值槽
commitment *Commitment
}

// KZG 承诺（基于 BLS12-381 曲线）
type Commitment struct {
// 多项式承诺
point bls12381.G1Affine
}

// 证明结构
type VerkleProof struct {
// 与 Merkle 不同，不需要兄弟节点
// 只需要：
Commitments []*Commitment // 路径上的承诺
Proof       *IPAProof     // Inner Product Argument 证明
}

// 状态访问与证明
type StatelessExecution struct {
// 无状态执行需要：
// 1. 区块头（包含状态根）
// 2. 交易
// 3. 访问状态的证明

BlockHeader *types.Header
Transactions []*types.Transaction
Witness      *VerkleWitness // 状态访问证明
}

type VerkleWitness struct {
// 包含所有被访问状态项的证明
Proofs map[common.Hash]*VerkleProof
Values map[common.Hash][]byte
}

// 无状态验证
func (s *StatelessExecution) Verify() error {
// 1. 验证证明有效性
for key, proof := range s.Witness.Proofs {
value := s.Witness.Values[key]
if !VerifyVerkleProof(s.BlockHeader.Root, key[:], value, proof) {
return errors.New("invalid verkle proof")
}
}

// 2. 执行交易（使用 witness 中的状态）
// 3. 验证状态根匹配

return nil
}
```

### 59.3 状态树迁移

```
┌─────────────────────────────────────────────────────────────────┐
│                   状态树迁移计划                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  迁移挑战：                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 当前状态 > 100GB                                     │   │
│  │  • 需要重新组织所有状态                                 │   │
│  │  • 地址空间变化（stem + suffix）                        │   │
│  │  • 需要保持向后兼容                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  迁移方案（overlay）：                                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  转换前：                                                │   │
│  │  ┌─────────────────────────────────────┐               │   │
│  │  │        Merkle Patricia Trie          │               │   │
│  │  │        (所有状态)                    │               │   │
│  │  └─────────────────────────────────────┘               │   │
│  │                                                          │   │
│  │  转换期：                                                │   │
│  │  ┌─────────────────────────────────────┐               │   │
│  │  │        Verkle Tree (新写入)          │               │   │
│  │  └─────────────────────────────────────┘               │   │
│  │  ┌─────────────────────────────────────┐               │   │
│  │  │        MPT (旧状态, 只读)            │               │   │
│  │  └─────────────────────────────────────┘               │   │
│  │                                                          │   │
│  │  转换后：                                                │   │
│  │  ┌─────────────────────────────────────┐               │   │
│  │  │        Verkle Tree (所有状态)        │               │   │
│  │  └─────────────────────────────────────┘               │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 60. Account Abstraction（AA）

### 60.1 AA 概述

```
┌─────────────────────────────────────────────────────────────────┐
│                   账户抽象 (AA)                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  当前账户模型：                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  EOA (Externally Owned Account)                         │   │
│  │  • 由私钥控制                                            │   │
│  │  • 只能使用 ECDSA 签名                                   │   │
│  │  • 用户必须持有 ETH 支付 Gas                             │   │
│  │                                                          │   │
│  │  Contract Account                                        │   │
│  │  • 由代码控制                                            │   │
│  │  • 不能发起交易                                          │   │
│  │  • 没有私钥                                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  AA 目标：                                                      │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. 自定义签名验证（多签、社交恢复、硬件等）            │   │
│  │  2. Gas 代付（Paymaster）                               │   │
│  │  3. 批量交易                                             │   │
│  │  4. Session Keys                                         │   │
│  │  5. 更好的用户体验                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  实现方式：                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  ERC-4337 (当前)：协议外实现                            │   │
│  │  • EntryPoint 合约                                       │   │
│  │  • Bundler 打包 UserOps                                  │   │
│  │  • 不需要协议更改                                        │   │
│  │                                                          │   │
│  │  EIP-7702 (即将)：EOA 临时设置代码                      │   │
│  │  • EOA 可以委托给智能合约                                │   │
│  │  • 保持 EOA 灵活性 + 智能合约功能                        │   │
│  │                                                          │   │
│  │  原生 AA (未来)：协议内置                               │   │
│  │  • 所有账户都是智能合约                                  │   │
│  │  • 统一的交易处理流程                                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 60.2 ERC-4337 架构

```go
// ERC-4337 核心组件

// UserOperation 用户操作
type UserOperation struct {
Sender               common.Address // 智能合约钱包地址
Nonce                *big.Int
InitCode             []byte // 创建钱包的代码（如果需要）
CallData             []byte          // 要执行的调用
CallGasLimit         *big.Int
VerificationGasLimit *big.Int
PreVerificationGas   *big.Int
MaxFeePerGas         *big.Int
MaxPriorityFeePerGas *big.Int
PaymasterAndData     []byte // Paymaster 地址 + 数据
Signature            []byte          // 签名（可自定义验证）
}

// EntryPoint 合约接口
type EntryPoint interface {
// 处理一批 UserOperations
HandleOps(ops []UserOperation, beneficiary common.Address) error

// 模拟验证（用于 Bundler）
SimulateValidation(op UserOperation) (ValidationResult, error)
}

// IAccount 智能合约钱包接口
type IAccount interface {
// 验证 UserOperation
ValidateUserOp(
userOp UserOperation,
userOpHash common.Hash,
missingAccountFunds *big.Int,
) (validationData *big.Int, err error)
}

// IPaymaster Gas 代付接口
type IPaymaster interface {
// 验证 Paymaster 是否愿意支付
ValidatePaymasterUserOp(
userOp UserOperation,
userOpHash common.Hash,
maxCost *big.Int,
) (context []byte, validationData *big.Int, err error)

// 操作后处理
PostOp(
mode PostOpMode,
context []byte,
actualGasCost *big.Int,
) error
}

// Bundler 实现
type Bundler struct {
entryPoint  common.Address
client      *ethclient.Client
mempool     []*UserOperation
}

// Bundle UserOperations
func (b *Bundler) Bundle(ops []*UserOperation) (*types.Transaction, error) {
// 1. 验证每个 UserOp
for _, op := range ops {
if err := b.validateOp(op); err != nil {
continue
}
}

// 2. 模拟执行
if err := b.simulateBundle(ops); err != nil {
return nil, err
}

// 3. 创建交易调用 EntryPoint.handleOps
entryPointABI, _ := abi.JSON(strings.NewReader(EntryPointABI))
data, _ := entryPointABI.Pack("handleOps", ops, b.beneficiary)

return types.NewTransaction(
b.nonce,
b.entryPoint,
big.NewInt(0),
b.estimateGas(ops),
b.gasPrice,
data,
), nil
}
```

### 60.3 EIP-7702：EOA 代码设置

```go
// EIP-7702：允许 EOA 临时设置代码

// 新交易类型
type SetCodeTransaction struct {
ChainID    uint64
Nonce      uint64
GasTipCap  *big.Int
GasFeeCap  *big.Int
Gas        uint64
To         common.Address
Value      *big.Int
Data       []byte

// 新字段：授权列表
AuthorizationList []Authorization
}

type Authorization struct {
ChainID uint64         // 链 ID
Address common.Address // 要委托的合约地址
Nonce   uint64           // EOA nonce
V, R, S *big.Int         // 签名
}

// 效果：
// 1. EOA 签署授权，允许其代码被设置为特定合约
// 2. 交易执行时，EOA 暂时拥有该合约的代码
// 3. 可以执行批量操作、自定义验证等
// 4. 交易结束后代码被移除（除非持久化）

// 使用示例
func DemonstrateBatchTransfer() {
// 用户 EOA 想要批量转账
// 传统方式：需要多个交易
// EIP-7702：一个交易完成

batchContract := common.HexToAddress("0x...") // 批量转账合约

auth := Authorization{
ChainID: 1,
Address: batchContract,
Nonce:   userNonce,
}
// 用户签名授权

tx := &SetCodeTransaction{
// ... 基本字段
AuthorizationList: []Authorization{auth},
Data: encodeBatchTransfer(recipients, amounts),
}
// 执行时，EOA 临时拥有 batchContract 的代码
// 可以执行批量转账
}
```

---

## 61. EIP-4844 与 Danksharding

### 61.1 EIP-4844 (Proto-Danksharding)

```
┌─────────────────────────────────────────────────────────────────┐
│                 EIP-4844 Proto-Danksharding                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  核心：Blob 交易                                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 新交易类型：Type 3                                    │   │
│  │  • 携带大量数据（Blob），但不被 EVM 执行                │   │
│  │  • 数据临时存储（~18天后删除）                          │   │
│  │  • 独立的 Blob Gas 市场                                 │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  用途：L2 数据可用性                                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  之前：L2 数据发布到 Calldata                           │   │
│  │  • ~16 gas/byte                                         │   │
│  │  • 永久存储                                              │   │
│  │  • 占用宝贵的区块空间                                   │   │
│  │                                                          │   │
│  │  之后：L2 数据发布到 Blob                               │   │
│  │  • ~1 gas/byte equivalent                               │   │
│  │  • 临时存储（足够长以完成挑战期）                       │   │
│  │  • 独立的数据空间                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  每区块 Blob 限制：                                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 目标：3 blobs/区块                                   │   │
│  │  • 最大：6 blobs/区块                                   │   │
│  │  • 每 blob：~128 KB                                     │   │
│  │  • 总计：~768 KB/区块                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 61.2 Blob 交易结构

```go
// Type 3 Blob 交易
type BlobTx struct {
ChainID    *uint256.Int
Nonce      uint64
GasTipCap  *uint256.Int // maxPriorityFeePerGas
GasFeeCap  *uint256.Int  // maxFeePerGas
Gas        uint64
To         common.Address
Value      *uint256.Int
Data       []byte
AccessList AccessList

// Blob 特有字段
BlobFeeCap *uint256.Int // maxFeePerBlobGas
BlobHashes []common.Hash  // blob 版本化哈希
Sidecar    *BlobTxSidecar // blob 数据（不上链）
}

type BlobTxSidecar struct {
Blobs       []kzg4844.Blob // 实际 blob 数据
Commitments []kzg4844.Commitment // KZG 承诺
Proofs      []kzg4844.Proof      // KZG 证明
}

// Blob Gas 定价
func CalcBlobFee(excessBlobGas uint64) *big.Int {
// 类似 EIP-1559 的指数定价
// blobBaseFee = MIN_BLOB_GASPRICE * e^(excessBlobGas / BLOB_GASPRICE_UPDATE_FRACTION)
return fakeExponential(
big.NewInt(params.MinBlobGasPrice),
new(big.Int).SetUint64(excessBlobGas),
big.NewInt(params.BlobGasPriceUpdateFraction),
)
}

// 新操作码
const (
BLOBHASH = 0x49 // 获取 blob 版本化哈希
BLOBBASEFEE = 0x4a  // 获取当前 blob base fee
)

// 使用示例：L2 发布数据
func PublishL2Data(client *ethclient.Client, data []byte) (*types.Transaction, error) {
// 1. 将数据编码为 blob
blob := encodeToBlob(data)

// 2. 计算 KZG 承诺和证明
commitment, _ := kzg4844.BlobToCommitment(blob)
proof, _ := kzg4844.ComputeBlobProof(blob, commitment)

// 3. 计算版本化哈希
blobHash := kzg4844.CalcBlobHashV1(sha256.Sum256(commitment[:]))

// 4. 构建 Blob 交易
tx := &types.BlobTx{
ChainID:    uint256.NewInt(1),
Nonce:      nonce,
GasTipCap:  uint256.NewInt(1e9),
GasFeeCap:  uint256.NewInt(50e9),
Gas:        100000,
To:         l2InboxAddress,
Value:      uint256.NewInt(0),
Data:       []byte{}, // 调用数据
BlobFeeCap: uint256.NewInt(1e9),
BlobHashes: []common.Hash{blobHash},
Sidecar: &types.BlobTxSidecar{
Blobs:       []kzg4844.Blob{blob},
Commitments: []kzg4844.Commitment{commitment},
Proofs:      []kzg4844.Proof{proof},
},
}

return types.SignTx(tx, signer, privateKey)
}
```

### 61.3 Full Danksharding（未来）

```
┌─────────────────────────────────────────────────────────────────┐
│                   Full Danksharding                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Proto-Danksharding (EIP-4844)：                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 6 blobs/区块                                         │   │
│  │  • 所有验证者存储所有 blob                              │   │
│  │  • ~0.375 MB/区块                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Full Danksharding：                                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 64-128 blobs/区块                                    │   │
│  │  • 数据可用性采样 (DAS)                                 │   │
│  │  • 验证者只采样部分数据                                 │   │
│  │  • ~16 MB/区块                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  DAS（数据可用性采样）：                                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  Blob 数据被编码并扩展                                  │   │
│  │  ┌─────────────────────────────────────────┐           │   │
│  │  │  原始数据 ──▶ Reed-Solomon 编码 ──▶ 2x 扩展        │   │
│  │  └─────────────────────────────────────────┘           │   │
│  │                                                          │   │
│  │  验证者随机采样几个点                                   │   │
│  │  ┌─────────────────────────────────────────┐           │   │
│  │  │  [■][□][■][□][□][■][□][□]...            │           │   │
│  │  │   ↑     ↑        ↑                       │           │   │
│  │  │   采样   采样     采样                   │           │   │
│  │  └─────────────────────────────────────────┘           │   │
│  │                                                          │   │
│  │  如果足够多的验证者成功采样，数据被认为可用            │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 62. 其他重要 EIP

### 62.1 EIP-2537: BLS12-381 预编译

```go
// BLS12-381 曲线预编译合约
// 用于高效的 BLS 签名验证

var (
BLS12_G1ADD = common.HexToAddress("0x0b") // G1 点加法
BLS12_G1MUL = common.HexToAddress("0x0c") // G1 标量乘法
BLS12_G1MULTIEXP = common.HexToAddress("0x0d") // G1 多标量乘法
BLS12_G2ADD = common.HexToAddress("0x0e") // G2 点加法
BLS12_G2MUL = common.HexToAddress("0x0f") // G2 标量乘法
BLS12_G2MULTIEXP = common.HexToAddress("0x10") // G2 多标量乘法
BLS12_PAIRING = common.HexToAddress("0x11") // 配对检查
BLS12_MAP_FP_TO_G1 = common.HexToAddress("0x12")
BLS12_MAP_FP2_TO_G2 = common.HexToAddress("0x13")
)

// 用途：
// 1. 验证 BLS 签名（信标链使用）
// 2. zkSNARK 验证
// 3. 阈值签名
// 4. 随机性生成
```

### 62.2 EIP-2935: 历史区块哈希

```go
// EIP-2935: 在状态中存储历史区块哈希
// 扩展 BLOCKHASH 可访问范围

// 当前限制：只能访问最近 256 个区块
// EIP-2935：可以访问 ~8192 个区块

const HISTORY_BUFFER_LENGTH = 8192

// 系统合约存储历史哈希
var HISTORY_STORAGE_ADDRESS = common.HexToAddress("0x0aae40965e6800cd9b1f4b05ff21581047e3f91e")

func GetHistoricalBlockHash(blockNumber uint64) common.Hash {
// 计算存储槽
slot := blockNumber % HISTORY_BUFFER_LENGTH

// 从系统合约读取
return state.GetState(HISTORY_STORAGE_ADDRESS, common.BigToHash(big.NewInt(int64(slot))))
}

// 用途：
// 1. 更长的 RANDAO reveal 期
// 2. 跨链桥验证
// 3. 历史数据访问
```

### 62.3 其他值得关注的 EIP

```go
// EIP-6110: 信标链存款在 EL 处理
// EIP-7002: EL 触发的验证者退出
// EIP-7251: 提高最大有效余额
// EIP-7549: 移动委员会索引到证明外

// EIP-7691: Blob 基础费用调整
// 调整 blob gas 定价参数

// EIP-7623: 增加 Calldata 成本
// 鼓励使用 blob 而非 calldata

// EIP-7742: 共识层设置 blob 参数
// 更灵活的 blob 限制调整
```

---

## 63. 对 MEV 的影响

### 63.1 协议级 MEV 缓解

```
┌─────────────────────────────────────────────────────────────────┐
│                  协议级 MEV 缓解                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Inclusion Lists (IL)：                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 提议者必须包含指定交易                               │   │
│  │  • 防止审查                                              │   │
│  │  • 保证交易及时上链                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ePBS (Enshrined PBS)：                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • PBS 内置到协议中                                      │   │
│  │  • 取代 MEV-Boost                                       │   │
│  │  • 更公平的 MEV 分配                                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  MEV Burn：                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 部分 MEV 收益被销毁                                  │   │
│  │  • 减少中心化激励                                       │   │
│  │  • 惠及所有 ETH 持有者                                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  加密 Mempool：                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • 交易内容加密直到包含                                 │   │
│  │  • 防止抢跑和三明治                                     │   │
│  │  • 使用阈值加密/TEE                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 63.2 升级对 MEV 策略的影响

```go
// 各升级对 MEV 的影响分析

type MEVImpact struct {
Upgrade     string
Impact      string
Adaptation  string
}

var UpgradeImpacts = []MEVImpact{
{
Upgrade: "EIP-4844 (Blobs)",
Impact:  "L2 成本降低，更多活动迁移到 L2",
Adaptation: "增加 L2 MEV 策略，跨 L2 套利",
},
{
Upgrade: "EOF",
Impact:  "代码更容易分析，静态跳转",
Adaptation: "更可靠的模拟，更精确的 Gas 估算",
},
{
Upgrade: "Verkle Trees",
Impact:  "无状态执行可能，更快的状态访问",
Adaptation: "利用无状态客户端进行更快的模拟",
},
{
Upgrade: "Account Abstraction",
Impact:  "新的交易类型，Paymaster 机制",
Adaptation: "新的 MEV 向量（UserOp 排序），Bundler 策略",
},
{
Upgrade: "ePBS",
Impact:  "构建者市场变化",
Adaptation: "适应新的 PBS 拍卖机制",
},
{
Upgrade: "Inclusion Lists",
Impact:  "不能随意排除交易",
Adaptation: "调整三明治策略，更依赖速度",
},
}

// AA 带来的新 MEV 向量
type AAMEVOpportunity struct {
// UserOp 排序
// Bundler 可以控制 UserOps 的顺序

// Paymaster 套利
// 如果 Paymaster 使用的价格不准确

// 批量操作前置
// 预测用户的批量操作意图
}
```

---

## 64. 学习资源与总结

### 64.1 推荐学习资源

```
┌─────────────────────────────────────────────────────────────────┐
│                    学习资源推荐                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  官方文档：                                                     │
│  • ethereum.org                                                 │
│  • github.com/ethereum/go-ethereum                             │
│  • eips.ethereum.org                                           │
│                                                                 │
│  EVM 深入：                                                     │
│  • evm.codes (操作码参考)                                       │
│  • www.ethervm.io                                              │
│  • github.com/ethereum/execution-specs                         │
│                                                                 │
│  MEV 相关：                                                     │
│  • flashbots.net                                               │
│  • docs.flashbots.net                                          │
│  • explore.flashbots.net                                       │
│  • mev.io                                                      │
│                                                                 │
│  L2 相关：                                                      │
│  • l2beat.com                                                  │
│  • arbitrum.io/docs                                            │
│  • docs.optimism.io                                            │
│  • docs.base.org                                               │
│                                                                 │
│  研究论文：                                                     │
│  • "Flash Boys 2.0" (MEV 开创性论文)                           │
│  • Flashbots 研究论文                                          │
│  • ethresear.ch (以太坊研究论坛)                               │
│                                                                 │
│  工具：                                                         │
│  • Foundry (forge, cast, anvil)                                │
│  • Hardhat                                                     │
│  • Tenderly                                                    │
│  • Dune Analytics                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 64.2 系列文档总结

```
┌─────────────────────────────────────────────────────────────────┐
│                    文档系列总结                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  01-go-ethereum-evm-design.md                                   │
│  ├── EVM 基础概念和设计哲学                                    │
│  ├── 核心数据结构                                               │
│  ├── 指令集系统                                                │
│  ├── 执行引擎                                                  │
│  └── Gas 机制、内存/存储模型                                   │
│                                                                 │
│  02-go-ethereum-evm-advanced.md                                 │
│  ├── 状态管理（Snapshot/Journal）                              │
│  ├── 日志和事件系统                                            │
│  ├── 调试与追踪                                                │
│  └── 硬分叉演进历史                                            │
│                                                                 │
│  03-go-ethereum-evm-bytecode.md                                 │
│  ├── 字节码结构                                                │
│  ├── 合约部署流程                                              │
│  ├── 函数选择器和 ABI                                          │
│  └── 字节码反编译                                              │
│                                                                 │
│  04-go-ethereum-evm-mev.md                                      │
│  ├── MEV 基础概念                                              │
│  ├── 套利/清算/三明治策略                                      │
│  ├── Flashbots 架构                                            │
│  └── Bundle 机制和 Searcher 开发                               │
│                                                                 │
│  05-go-ethereum-evm-txpool.md                                   │
│  ├── 交易池架构                                                │
│  ├── 交易验证和排序                                            │
│  ├── Nonce 管理                                                │
│  └── Gas 价格机制                                              │
│                                                                 │
│  06-go-ethereum-evm-simulation.md                               │
│  ├── EVM 模拟基础                                              │
│  ├── eth_call 和状态覆盖                                       │
│  ├── 本地 EVM 执行                                             │
│  └── 套利模拟引擎                                              │
│                                                                 │
│  07-go-ethereum-evm-l2.md                                       │
│  ├── L2 类型对比                                               │
│  ├── Arbitrum/Optimism 详解                                    │
│  ├── L2 MEV 特性                                               │
│  └── 跨 L2 套利                                                │
│                                                                 │
│  08-go-ethereum-evm-future.md                                   │
│  ├── EVM 演进路线                                              │
│  ├── EOF、Verkle Trees                                         │
│  ├── Account Abstraction                                       │
│  └── Danksharding                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 64.3 下一步学习建议

```
┌─────────────────────────────────────────────────────────────────┐
│                    下一步建议                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  对于套利开发者：                                               │
│  1. 搭建本地节点（Geth/Reth）                                  │
│  2. 实现简单的两池套利机器人                                   │
│  3. 学习 Flashbots Bundle 提交                                 │
│  4. 优化模拟速度和准确性                                       │
│  5. 扩展到 L2 和跨链                                           │
│                                                                 │
│  实践项目：                                                     │
│  1. EVM 字节码反编译器                                         │
│  2. 本地 EVM 模拟器                                            │
│  3. Mempool 监控系统                                           │
│  4. DEX 套利扫描器                                             │
│  5. 跨 L2 价格监控                                             │
│                                                                 │
│  持续关注：                                                     │
│  • Ethereum Magicians (核心开发讨论)                           │
│  • All Core Devs 会议纪要                                      │
│  • EIP 更新                                                    │
│  • Flashbots/MEV 研究进展                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 结语

本系列文档从 EVM 基础到未来演进，系统性地介绍了 go-ethereum EVM 的设计与实现。

核心要点回顾：

1. **EVM 是栈式虚拟机**，256位字长，确定性执行
2. **Gas 机制**防止资源滥用，驱动经济模型
3. **状态管理**通过 Snapshot/Journal 实现原子性
4. **MEV**是 DeFi 生态的重要组成部分
5. **L2**是扩容的主要方向
6. **未来升级**（EOF、Verkle、AA）将带来重大变化

希望这些文档能帮助你深入理解 EVM，并在实际项目中应用这些知识。

祝学习愉快，套利顺利！
