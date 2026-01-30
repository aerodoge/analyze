# Go后端开发工程师面试准备文档

基于简历项目经验整理的技术知识点和面试问答

---

## 第一部分：Golang核心基础知识

### 1.1 Goroutine和并发编程

#### 问题1：Goroutine和线程的区别是什么？

**答案：**
1. **资源占用**：
   - Goroutine初始栈大小只有2KB，可动态扩容
   - 线程栈大小通常为1-2MB，固定分配
   - 因此可以轻松创建成千上万个Goroutine

2. **调度方式**：
   - Goroutine由Go运行时的调度器（GMP模型）在用户态调度
   - 线程由操作系统内核调度，涉及上下文切换开销大

3. **创建销毁成本**：
   - Goroutine创建和销毁成本极低
   - 线程创建需要系统调用，成本较高

4. **通信方式**：
   - Goroutine通过channel通信（CSP模型）
   - 线程通过共享内存+锁通信

**项目应用**：在游戏交易平台中，使用Goroutine并发处理订单请求，在跨境电商中使用Goroutine并发生成报表。

---

#### 问题2：GMP模型是什么？请详细解释

**答案：**

GMP模型是Go语言的调度模型，包含三个核心组件：

1. **G (Goroutine)**：
   - 代表一个Goroutine
   - 包含栈、指令指针、调度信息等
   - 状态：运行中、就绪、阻塞

2. **M (Machine/OS Thread)**：
   - 代表操作系统线程
   - 执行G的实体
   - 数量可动态增减，但有上限（默认10000）

3. **P (Processor)**：
   - 逻辑处理器，代表执行G所需的资源
   - 包含本地运行队列（Local Run Queue）
   - 数量默认等于CPU核心数（GOMAXPROCS）

**调度流程**：
```
1. 创建Goroutine后，G被放入P的本地队列
2. M从P的本地队列获取G执行
3. 如果本地队列为空，M会从全局队列或其他P"偷取"G（Work Stealing）
4. G阻塞时（如系统调用），M会释放P，P可绑定其他M继续执行其他G
```

**关键优势**：
- P的本地队列减少锁竞争
- Work Stealing实现负载均衡
- M和P解耦，避免线程频繁创建销毁

**项目应用**：在套利交易系统中，利用GMP模型的高并发特性处理多交易所行情，实现毫秒级信号生成。

---

#### 问题3：Channel的底层原理是什么？有哪些使用注意事项？

**答案：**

**底层数据结构**：
```go
type hchan struct {
    qcount   uint           // 当前队列中元素个数
    dataqsiz uint           // 循环队列大小
    buf      unsafe.Pointer // 指向循环队列
    elemsize uint16         // 元素大小
    closed   uint32         // 是否关闭
    sendx    uint           // 发送索引
    recvx    uint           // 接收索引
    recvq    waitq          // 等待接收的Goroutine队列
    sendq    waitq          // 等待发送的Goroutine队列
    lock     mutex          // 互斥锁
}
```

**工作原理**：
1. **发送数据**：
   - 如果有等待接收的G，直接发送给它
   - 否则，如果buf未满，放入buf
   - 否则，当前G阻塞并加入sendq

2. **接收数据**：
   - 如果有等待发送的G，直接接收
   - 否则，如果buf非空，从buf取出
   - 否则，当前G阻塞并加入recvq

**使用注意事项**：
1. **避免死锁**：
   - 无缓冲channel必须有接收者才能发送
   - 不要在同一个Goroutine中对无缓冲channel发送和接收

2. **关闭channel的规则**：
   - 不要关闭已关闭的channel（会panic）
   - 不要向已关闭的channel发送数据（会panic）
   - 可以从已关闭的channel接收数据（返回零值+false）
   - 只由发送方关闭channel

3. **性能考虑**：
   - 有缓冲channel减少阻塞，提高性能
   - 避免频繁创建销毁channel

**项目应用示例**：
```go
// 游戏交易平台中，使用channel控制并发订单处理
orderChan := make(chan Order, 1000) // 缓冲channel

// 生产者：接收订单
go func() {
    for order := range getOrders() {
        orderChan <- order
    }
    close(orderChan)
}()

// 消费者：处理订单
for i := 0; i < 10; i++ {
    go func() {
        for order := range orderChan {
            processOrder(order)
        }
    }()
}
```

---

### 1.2 内存管理和垃圾回收

#### 问题4：Go的内存分配机制是怎样的？

**答案：**

**内存分配策略（按对象大小分类）**：

1. **微对象（Tiny）：< 16B**
   - 多个微对象可共享一个16B的内存块
   - 减少内存碎片

2. **小对象（Small）：16B ~ 32KB**
   - 从P的mcache中分配
   - mcache包含各种规格的mspan（67种）
   - 无锁分配，速度快

3. **大对象（Large）：> 32KB**
   - 直接从mheap分配
   - 需要加锁，速度较慢

**分配层次**：
```
Goroutine请求内存
    ↓
P的mcache（无锁，快速）
    ↓ (miss)
中心缓存mcentral（有锁）
    ↓ (miss)
堆mheap（有锁）
    ↓ (miss)
向OS申请
```

**关键优化**：
- 每个P有独立的mcache，减少锁竞争
- 内存复用，减少系统调用
- 多级缓存，平衡速度和内存使用

**项目应用**：
在高频交易系统中，大量小对象分配（订单、行情tick）会触发频繁的小对象分配，利用对象池（sync.Pool）复用对象减少GC压力。

---

#### 问题5：Go的GC是如何工作的？如何优化GC性能？

**答案：**

**GC算法**：三色标记法 + 并发清除

**GC流程**：
1. **STW（Stop The World）阶段1**：
   - 暂停所有Goroutine
   - 启动标记工作协程
   - 开启写屏障（Write Barrier）
   - 耗时极短（~10-30微秒）

2. **并发标记阶段**：
   - 从GC Root开始标记
   - 白色：未访问
   - 灰色：已访问，但子对象未访问
   - 黑色：已访问，且子对象已访问
   - 用户程序和GC并发执行

3. **STW阶段2**：
   - 重新扫描（处理标记期间的变化）
   - 关闭写屏障
   - 耗时极短

4. **并发清除阶段**：
   - 清除白色对象
   - 与用户程序并发执行

**GC触发条件**：
1. 内存增长达到阈值（GOGC环境变量，默认100，即翻倍）
2. 定时触发（2分钟）
3. 手动调用runtime.GC()

**优化GC性能的方法**：

1. **减少内存分配**：
```go
// 不好：频繁分配
for i := 0; i < n; i++ {
    buf := make([]byte, 1024)
    // 使用buf
}

// 好：复用buffer
buf := make([]byte, 1024)
for i := 0; i < n; i++ {
    // 使用buf
}
```

2. **使用sync.Pool复用对象**：
```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 1024)
    },
}

buf := bufferPool.Get().([]byte)
// 使用buf
bufferPool.Put(buf)
```

3. **使用指针切片代替值切片**：
```go
// 不好：值切片，GC需要扫描每个元素
type Order struct {
    ID   int64
    Data [1024]byte
}
orders := make([]Order, 10000)

// 好：指针切片，GC只需扫描指针
orders := make([]*Order, 10000)
```

4. **调整GOGC参数**：
```bash
# 增大GOGC，减少GC频率，但增加内存占用
export GOGC=200
```

5. **使用堆外内存**：
```go
// 使用mmap等系统调用分配堆外内存，不受GC管理
// 适合大块、长期存活的数据
```

**项目应用**：
在游戏交易平台中，使用sync.Pool复用订单对象和buffer，使用pprof监控GC，优化后GC暂停时间从5ms降至1ms以内。

---

### 1.3 Sync包和并发控制

#### 问题6：Mutex和RWMutex的区别？什么时候使用哪个？

**答案：**

**Mutex（互斥锁）**：
```go
var mu sync.Mutex
mu.Lock()
// 临界区
mu.Unlock()
```
- 同时只允许一个Goroutine访问
- 读写都需要加锁
- 适合写操作频繁的场景

**RWMutex（读写锁）**：
```go
var mu sync.RWMutex

// 读锁（共享锁）
mu.RLock()
// 读操作
mu.RUnlock()

// 写锁（排他锁）
mu.Lock()
// 写操作
mu.Unlock()
```
- 允许多个Goroutine同时读
- 写操作独占
- 适合读多写少的场景

**性能对比**：

| 场景         | Mutex | RWMutex |
|------------|-------|---------|
| 纯读         | 慢     | 快       |
| 读多写少(90%读) | 慢     | 快       |
| 读写均衡(50%读) | 相近    | 相近      |
| 写多读少(10%读) | 快     | 慢       |

**选择原则**：
- 读写比 > 5:1 → 使用RWMutex
- 读写比 < 5:1 → 使用Mutex
- 临界区极短 → 使用Mutex（RWMutex有额外开销）

**项目应用示例**：
```go
// 游戏交易平台中的商品缓存
type ProductCache struct {
    mu    sync.RWMutex
    cache map[int64]*Product
}

// 读操作（频繁）
func (c *ProductCache) Get(id int64) *Product {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.cache[id]
}

// 写操作（不频繁）
func (c *ProductCache) Set(id int64, p *Product) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.cache[id] = p
}
```

---

#### 问题7：sync.Map和普通map+锁的区别？什么时候用？

**答案：**

**普通map+锁**：
```go
type SafeMap struct {
    mu   sync.RWMutex
    data map[string]interface{}
}
```
- 需要手动加锁
- 锁粒度粗，性能受限
- 适合通用场景

**sync.Map**：
```go
var m sync.Map
m.Store("key", "value")
value, ok := m.Load("key")
m.Delete("key")
m.Range(func(k, v interface{}) bool {
    // 遍历
    return true
})
```

**sync.Map优化场景**：
1. **读多写少**：
   - 内部使用read和dirty两个map
   - read map无锁读取
   - 热点数据存在read中

2. **键值稳定**：
   - 键值对固定，很少增删
   - 频繁读取

**性能对比**：

| 场景         | map+RWMutex | sync.Map |
|------------|-------------|----------|
| 读多写少(95%读) | 慢           | 快        |
| 频繁写入       | 快           | 慢        |
| 频繁增删       | 快           | 慢        |

**选择建议**：
- 读多写少 + 键稳定 → sync.Map
- 其他场景 → map + RWMutex
- 需要复杂操作（遍历+修改） → map + RWMutex

**项目应用**：
```go
// 游戏交易平台中的用户会话缓存
var sessionCache sync.Map

// 高频读取用户session
func GetSession(token string) *Session {
    if v, ok := sessionCache.Load(token); ok {
        return v.(*Session)
    }
    return nil
}

// 低频更新session
func SetSession(token string, s *Session) {
    sessionCache.Store(token, s)
}
```

---

#### 问题8：如何实现一个并发安全的连接池？

**答案：**

**实现思路**：
```go
type ConnPool struct {
    conns   chan *Conn      // 使用channel作为连接池
    factory func() *Conn    // 连接工厂函数
    maxSize int             // 最大连接数
    size    int32           // 当前连接数（原子操作）
    mu      sync.Mutex
}

func NewConnPool(maxSize int, factory func() *Conn) *ConnPool {
    return &ConnPool{
        conns:   make(chan *Conn, maxSize),
        factory: factory,
        maxSize: maxSize,
    }
}

// 获取连接
func (p *ConnPool) Get() (*Conn, error) {
    select {
    case conn := <-p.conns:
        // 从池中获取
        return conn, nil
    default:
        // 池为空，尝试创建新连接
        if atomic.LoadInt32(&p.size) < int32(p.maxSize) {
            p.mu.Lock()
            if p.size < p.maxSize {
                conn := p.factory()
                atomic.AddInt32(&p.size, 1)
                p.mu.Unlock()
                return conn, nil
            }
            p.mu.Unlock()
        }
        // 达到最大连接数，阻塞等待
        return <-p.conns, nil
    }
}

// 归还连接
func (p *ConnPool) Put(conn *Conn) {
    if conn == nil {
        return
    }
    select {
    case p.conns <- conn:
        // 归还成功
    default:
        // 池已满，关闭连接
        conn.Close()
        atomic.AddInt32(&p.size, -1)
    }
}

// 关闭连接池
func (p *ConnPool) Close() {
    close(p.conns)
    for conn := range p.conns {
        conn.Close()
    }
}
```

**关键点**：
1. 使用buffered channel作为连接容器
2. 使用atomic保证size的并发安全
3. Get时先尝试从池获取，失败则创建新连接
4. Put时尝试归还，池满则关闭连接

**项目应用**：
在游戏交易平台中，使用连接池管理数据库连接、Redis连接，避免频繁建立关闭连接，性能提升300%。

---

### 1.4 Context使用

#### 问题9：Context的作用是什么？如何使用？

**答案：**

**Context的作用**：
1. **传递取消信号**：
   - 通知Goroutine停止工作
   - 避免Goroutine泄漏

2. **传递截止时间**：
   - 超时控制
   - 防止请求无限等待

3. **传递请求范围的值**：
   - 如trace id、用户信息
   - 跨函数传递

**Context类型**：
```go
// 1. 根Context（永不取消）
ctx := context.Background()
ctx := context.TODO()

// 2. WithCancel：手动取消
ctx, cancel := context.WithCancel(parentCtx)
defer cancel()

// 3. WithTimeout：超时自动取消
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
defer cancel()

// 4. WithDeadline：到期自动取消
deadline := time.Now().Add(10 * time.Second)
ctx, cancel := context.WithDeadline(parentCtx, deadline)
defer cancel()

// 5. WithValue：传递值
ctx := context.WithValue(parentCtx, key, value)
```

**最佳实践**：
```go
// 1. Context作为第一个参数
func ProcessOrder(ctx context.Context, order *Order) error {
    // 检查是否取消
    select {
    case <-ctx.Done():
        return ctx.Err() // 返回context.Canceled或context.DeadlineExceeded
    default:
    }

    // 业务逻辑
    return nil
}

// 2. 使用WithTimeout控制超时
func CallExternalAPI(ctx context.Context, url string) error {
    ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()

    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    return nil
}

// 3. 传递trace id
func HandleRequest(w http.ResponseWriter, r *http.Request) {
    traceID := generateTraceID()
    ctx := context.WithValue(r.Context(), "traceID", traceID)

    processRequest(ctx)
}

func processRequest(ctx context.Context) {
    traceID := ctx.Value("traceID").(string)
    log.Printf("[%s] processing...", traceID)
}
```

**注意事项**：
1. 不要把Context放在结构体中，作为参数传递
2. 不要传递nil Context，使用context.TODO()
3. WithValue只用于请求范围的数据，不要滥用
4. 父Context取消，所有子Context自动取消

**项目应用示例**：
```go
// 游戏交易平台订单处理
func ProcessOrder(ctx context.Context, orderID int64) error {
    // 设置5秒超时
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    // 并发执行多个步骤
    g, ctx := errgroup.WithContext(ctx)

    g.Go(func() error {
        return checkInventory(ctx, orderID)
    })

    g.Go(func() error {
        return validatePayment(ctx, orderID)
    })

    g.Go(func() error {
        return updateUserBalance(ctx, orderID)
    })

    // 任意一个失败或超时，其他自动取消
    return g.Wait()
}
```

---

### 1.5 错误处理

#### 问题10：Go的错误处理机制？如何优雅处理错误？

**答案：**

**基本错误处理**：
```go
// 1. 返回error
func Divide(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// 2. 自定义错误类型
type ValidationError struct {
    Field string
    Msg   string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed: %s - %s", e.Field, e.Msg)
}

// 3. 错误包装（Go 1.13+）
if err != nil {
    return fmt.Errorf("failed to process order: %w", err)
}

// 4. 错误判断
if errors.Is(err, ErrNotFound) {
    // 处理未找到错误
}

if var validErr *ValidationError; errors.As(err, &validErr) {
    // 处理验证错误
}
```

**优雅的错误处理模式**：

1. **Sentinel Error（哨兵错误）**：
```go
var (
    ErrNotFound    = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrInvalidInput = errors.New("invalid input")
)

func GetUser(id int64) (*User, error) {
    if id <= 0 {
        return nil, ErrInvalidInput
    }
    // ...
    if user == nil {
        return nil, ErrNotFound
    }
    return user, nil
}

// 使用
user, err := GetUser(id)
if errors.Is(err, ErrNotFound) {
    return nil, status.Error(codes.NotFound, "user not found")
}
```

2. **错误包装保留上下文**：
```go
func ProcessOrder(orderID int64) error {
    order, err := getOrder(orderID)
    if err != nil {
        return fmt.Errorf("get order %d: %w", orderID, err)
    }

    if err := validateOrder(order); err != nil {
        return fmt.Errorf("validate order %d: %w", orderID, err)
    }

    return nil
}

// 错误链：
// validate order 123: insufficient balance: user 456 balance is 10, need 100
```

3. **统一错误码**：
```go
type ErrorCode int

const (
    CodeSuccess         ErrorCode = 0
    CodeInvalidParam    ErrorCode = 1000
    CodeNotFound        ErrorCode = 1001
    CodeUnauthorized    ErrorCode = 1002
    CodeInternalError   ErrorCode = 5000
)

type BizError struct {
    Code ErrorCode
    Msg  string
    Err  error
}

func (e *BizError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("[%d] %s: %v", e.Code, e.Msg, e.Err)
    }
    return fmt.Sprintf("[%d] %s", e.Code, e.Msg)
}

func NewBizError(code ErrorCode, msg string) *BizError {
    return &BizError{Code: code, Msg: msg}
}

// API统一返回格式
type Response struct {
    Code ErrorCode   `json:"code"`
    Msg  string      `json:"msg"`
    Data interface{} `json:"data,omitempty"`
}
```

**项目应用示例**：
```go
// 游戏交易平台错误处理
var (
    ErrInsufficientBalance = NewBizError(2001, "insufficient balance")
    ErrProductSoldOut      = NewBizError(2002, "product sold out")
    ErrOrderExpired        = NewBizError(2003, "order expired")
)

func CreateOrder(ctx context.Context, req *CreateOrderRequest) (*Order, error) {
    // 验证参数
    if err := validateRequest(req); err != nil {
        return nil, fmt.Errorf("validate request: %w", err)
    }

    // 检查库存
    if err := checkStock(ctx, req.ProductID, req.Quantity); err != nil {
        if errors.Is(err, ErrProductSoldOut) {
            return nil, err // 直接返回业务错误
        }
        return nil, fmt.Errorf("check stock: %w", err) // 包装系统错误
    }

    // 扣款
    if err := deductBalance(ctx, req.UserID, req.Amount); err != nil {
        if errors.Is(err, ErrInsufficientBalance) {
            return nil, err
        }
        return nil, fmt.Errorf("deduct balance: %w", err)
    }

    // 创建订单
    order, err := createOrderInDB(ctx, req)
    if err != nil {
        return nil, fmt.Errorf("create order in db: %w", err)
    }

    return order, nil
}
```

---

## 第二部分：HTTP协议与网络基础

### 2.1 HTTP基础

#### 问题1：HTTP/1.0、HTTP/1.1、HTTP/2、HTTP/3 的区别？

**答案：**

**HTTP/1.0（1996年）**：
- **特点**：
  - 每个请求都需要建立新的TCP连接
  - 请求完成后立即关闭连接
  - 无状态协议

- **问题**：
  - 连接无法复用，每次请求都要三次握手
  - 性能差，延迟高

**HTTP/1.1（1999年）**：
- **改进**：
  1. **持久连接（Keep-Alive）**：
     - 默认开启长连接，一个TCP连接可以发送多个HTTP请求
     - `Connection: keep-alive`

  2. **管道化（Pipelining）**：
     - 可以在同一个连接上同时发送多个请求
     - 但响应必须按顺序返回（队头阻塞问题）

  3. **Host头字段**：
     - 支持虚拟主机，一个IP可以托管多个域名

  4. **分块传输编码**：
     - `Transfer-Encoding: chunked`
     - 支持流式传输，不需要提前知道内容长度

  5. **新增方法**：
     - PUT, DELETE, OPTIONS, TRACE 等

- **问题**：
  - **队头阻塞（Head-of-Line Blocking）**：前一个请求阻塞会影响后续请求
  - 只能客户端发起请求，服务端无法主动推送

**HTTP/2（2015年）**：
- **核心改进**：
  1. **二进制分帧（Binary Framing）**：
     - HTTP/1.x 是文本协议，HTTP/2 是二进制协议
     - 更高效的解析和传输

  2. **多路复用（Multiplexing）**：
     - 一个TCP连接可以并发处理多个请求和响应
     - 每个请求/响应被拆分成帧（Frame），帧可以乱序发送
     - 解决了HTTP/1.1的队头阻塞问题
     ```
     HTTP/1.1（队头阻塞）：
     请求1 ████████████ 响应1
     请求2          等待... ████████ 响应2
     请求3                    等待... ██ 响应3

     HTTP/2（多路复用）：
     请求1 ██ ██ ██ 响应1
     请求2  ██ ██ 响应2
     请求3   ██ 响应3
     同时传输，互不阻塞！
     ```

  3. **头部压缩（HPACK）**：
     - HTTP头部通常很大且重复
     - HPACK算法压缩头部，减少传输数据量

  4. **服务端推送（Server Push）**：
     - 服务器可以主动推送资源给客户端
     - 例如：浏览器请求 index.html，服务器主动推送 style.css 和 script.js

  5. **流优先级（Stream Priority）**：
     - 可以设置请求的优先级
     - 重要资源优先传输

**HTTP/3（2020年）**：
- **核心改进**：
  1. **基于 QUIC 协议**：
     - HTTP/1.1 和 HTTP/2 基于 TCP
     - HTTP/3 基于 UDP + QUIC

  2. **解决TCP队头阻塞**：
     - HTTP/2虽然解决了应用层队头阻塞
     - 但TCP层丢包仍会阻塞整个连接
     - QUIC在传输层实现多路复用，丢包只影响单个流

  3. **更快的连接建立**：
     - TCP + TLS 需要 3次握手 + 2-3次TLS握手 = 5-6次RTT
     - QUIC只需 0-1次RTT（首次连接1-RTT，后续0-RTT）

  4. **连接迁移**：
     - TCP连接由四元组标识（源IP、源端口、目的IP、目的端口）
     - 网络切换（WiFi → 4G）会断开连接
     - QUIC使用连接ID，支持网络切换时保持连接

**对比总结表**：

| 特性    | HTTP/1.0 | HTTP/1.1 | HTTP/2    | HTTP/3     |
|-------|----------|----------|-----------|------------|
| 传输协议  | TCP      | TCP      | TCP       | QUIC (UDP) |
| 连接复用  | ❌ 短连接    | ✅ 长连接    | ✅ 长连接     | ✅ 长连接      |
| 多路复用  | ❌        | ❌        | ✅         | ✅          |
| 队头阻塞  | ✅ 严重     | ✅ 存在     | ⚠️ TCP层存在 | ✅ 完全解决     |
| 头部压缩  | ❌        | ❌        | ✅ HPACK   | ✅ QPACK    |
| 服务端推送 | ❌        | ❌        | ✅         | ✅          |
| 传输格式  | 文本       | 文本       | 二进制       | 二进制        |
| 连接建立  | 3-RTT    | 3-RTT    | 3-RTT     | 0-1 RTT    |

**项目应用**：
在游戏交易平台中，使用HTTP/2提升API性能，通过多路复用减少延迟，头部压缩减少带宽消耗。

---

#### 问题2：HTTP请求的完整流程是什么？（从输入URL到页面显示）

**答案：**

**完整流程（7个步骤）**：

**1. DNS解析（域名 → IP地址）**
```
用户输入：https://www.example.com/api/orders

步骤1.1：浏览器检查DNS缓存
- 浏览器缓存
- 操作系统缓存
- 路由器缓存

步骤1.2：查询DNS服务器
浏览器 → 本地DNS服务器 → 根DNS服务器 → .com顶级域服务器 → example.com权威服务器

结果：www.example.com → 192.168.1.100
```

**2. 建立TCP连接（三次握手）**
```
客户端                        服务器(192.168.1.100:443)
  │                                    │
  │─────── SYN (seq=100) ────────────>│  第1次握手：客户端发起连接
  │                                    │
  │<──── SYN+ACK (seq=200,ack=101) ───│  第2次握手：服务器确认
  │                                    │
  │─────── ACK (ack=201) ────────────>│  第3次握手：客户端确认
  │                                    │
  │          TCP连接建立                │
```

**为什么需要三次握手？**
- 第1次：服务器知道客户端能发送
- 第2次：客户端知道服务器能接收和发送
- 第3次：服务器知道客户端能接收
- **防止旧连接请求**：防止已失效的连接请求突然传到服务器

**3. TLS握手（HTTPS）**
```
客户端                        服务器
  │                                    │
  │────── ClientHello ──────────────>│  1. 客户端支持的加密套件
  │                                    │
  │<───── ServerHello ───────────────│  2. 服务器选择的加密套件
  │<───── Certificate ───────────────│     + 服务器证书
  │<───── ServerHelloDone ───────────│
  │                                    │
  │────── ClientKeyExchange ────────>│  3. 客户端验证证书，发送密钥
  │────── ChangeCipherSpec ─────────>│
  │────── Finished ─────────────────>│
  │                                    │
  │<───── ChangeCipherSpec ──────────│  4. 服务器确认
  │<───── Finished ──────────────────│
  │                                    │
  │       加密通道建立                  │
```

**4. 发送HTTP请求**
```http
GET /api/orders?user_id=123 HTTP/1.1
Host: www.example.com
User-Agent: Mozilla/5.0
Accept: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Connection: keep-alive
```

**5. 服务器处理请求**
```
Web服务器(Nginx/Apache)
  ↓
应用服务器(Go/Java/Python)
  ↓
业务逻辑处理
  ↓
数据库查询(MySQL/MongoDB)
  ↓
缓存查询(Redis)
  ↓
生成响应
```

**6. 服务器返回HTTP响应**
```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 1234
Date: Fri, 24 Jan 2025 10:00:00 GMT
Connection: keep-alive

{
  "code": 0,
  "data": {
    "orders": [...]
  }
}
```

**7. 浏览器渲染页面**
- 解析HTML
- 加载CSS、JS、图片等资源
- 渲染页面

**时间消耗分析**：
```
DNS解析：        20-100ms
TCP三次握手：    20-100ms (取决于网络延迟)
TLS握手：        40-200ms (需要2-3个RTT)
HTTP请求响应：   50-500ms (取决于服务器处理时间)
──────────────────────────────
总计：          130-900ms
```

**Go语言实现（服务端）**：
```go
package main

import (
    "encoding/json"
    "log"
    "net/http"
)

func main() {
    // 注册路由
    http.HandleFunc("/api/orders", getOrders)

    // 启动HTTPS服务器
    log.Println("Server starting on https://localhost:443")
    err := http.ListenAndServeTLS(":443", "cert.pem", "key.pem", nil)
    if err != nil {
        log.Fatal(err)
    }
}

func getOrders(w http.ResponseWriter, r *http.Request) {
    // 1. 解析请求
    userID := r.URL.Query().Get("user_id")

    // 2. 业务逻辑处理
    orders := getOrdersFromDB(userID)

    // 3. 构造响应
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)

    response := map[string]interface{}{
        "code": 0,
        "data": map[string]interface{}{
            "orders": orders,
        },
    }

    json.NewEncoder(w).Encode(response)
}
```

**项目应用**：
在跨境电商平台中，通过优化DNS解析（使用CDN）、启用HTTP/2、减少TLS握手时间（Session复用），将API响应时间从300ms降至50ms。

---

#### 问题3：HTTP常见状态码及含义？

**答案：**

**1xx - 信息性状态码**：
- **100 Continue**：客户端应继续请求
- **101 Switching Protocols**：协议切换（如WebSocket升级）

**2xx - 成功状态码**：
- **200 OK**：请求成功
- **201 Created**：资源创建成功（常用于POST）
- **204 No Content**：成功但无返回内容（常用于DELETE）
- **206 Partial Content**：部分内容（用于断点续传）

**3xx - 重定向状态码**：
- **301 Moved Permanently**：永久重定向，浏览器会缓存
- **302 Found**：临时重定向
- **304 Not Modified**：资源未修改，使用缓存
- **307 Temporary Redirect**：临时重定向，保持请求方法不变

**4xx - 客户端错误**：
- **400 Bad Request**：请求参数错误
- **401 Unauthorized**：未认证（需要登录）
- **403 Forbidden**：已认证但无权限
- **404 Not Found**：资源不存在
- **405 Method Not Allowed**：请求方法不允许
- **429 Too Many Requests**：请求过于频繁（限流）

**5xx - 服务器错误**：
- **500 Internal Server Error**：服务器内部错误
- **502 Bad Gateway**：网关错误（上游服务器响应无效）
- **503 Service Unavailable**：服务不可用（维护或过载）
- **504 Gateway Timeout**：网关超时

**Go语言实践**：
```go
func handleAPI(w http.ResponseWriter, r *http.Request) {
    // 400 - 参数错误
    if r.URL.Query().Get("id") == "" {
        http.Error(w, "缺少id参数", http.StatusBadRequest)
        return
    }

    // 401 - 未认证
    token := r.Header.Get("Authorization")
    if token == "" {
        http.Error(w, "未登录", http.StatusUnauthorized)
        return
    }

    // 403 - 无权限
    if !checkPermission(token) {
        http.Error(w, "无权限访问", http.StatusForbidden)
        return
    }

    // 404 - 资源不存在
    data, err := getFromDB(id)
    if err == sql.ErrNoRows {
        http.Error(w, "资源不存在", http.StatusNotFound)
        return
    }

    // 500 - 服务器错误
    if err != nil {
        http.Error(w, "服务器内部错误", http.StatusInternalServerError)
        return
    }

    // 200 - 成功
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(data)
}
```

**项目应用**：
在游戏交易平台API设计中，严格遵循HTTP状态码规范，使用400表示参数错误，401表示未登录，403表示权限不足，提升API的可读性。

---

#### 问题4：GET和POST的区别？

**答案：**

**本质区别**：语义不同
- **GET**：获取资源（幂等操作，不改变服务器状态）
- **POST**：提交数据（非幂等操作，可能改变服务器状态）

**具体区别**：

| 特性 | GET | POST |
|------|-----|------|
| **参数位置** | URL查询字符串 | 请求体（Body） |
| **参数长度** | 受URL长度限制（约2KB） | 无限制 |
| **缓存** | 可被浏览器缓存 | 默认不缓存 |
| **书签** | 可收藏为书签 | 不可以 |
| **历史记录** | 保留在浏览器历史 | 不保留 |
| **安全性** | 参数暴露在URL | 参数在Body中 |
| **幂等性** | 幂等（多次调用结果相同） | 非幂等 |
| **编码** | URL编码 | 多种（form、json、xml） |

**示例对比**：

```http
GET请求：
GET /api/users?id=123&name=张三 HTTP/1.1
Host: api.example.com

特点：
- 参数在URL中可见
- 刷新页面会重复发送
- 可以直接在浏览器地址栏输入


POST请求：
POST /api/users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "id": 123,
  "name": "张三"
}

特点：
- 参数在请求体中
- 刷新页面浏览器会提示"是否重新提交"
- 不能直接在地址栏输入
```

**常见误解**：

❌ **误解1**：GET比POST更快
- **真相**：性能差异微乎其微，主要区别在语义

❌ **误解2**：GET不安全，POST安全
- **真相**：都不安全，HTTPS才安全。GET参数在URL中可见，但POST的Body也能被抓包看到

❌ **误解3**：GET只能传少量数据
- **真相**：HTTP协议没有限制，是浏览器和服务器限制了URL长度

**Go语言实践**：
```go
// GET - 查询用户信息（幂等）
func getUser(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodGet {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }

    // 从URL查询参数获取
    userID := r.URL.Query().Get("id")

    user, err := db.QueryUser(userID)
    // ...
}

// POST - 创建用户（非幂等）
func createUser(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodPost {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }

    // 从请求体获取
    var req struct {
        Name  string `json:"name"`
        Email string `json:"email"`
    }
    json.NewDecoder(r.Body).Decode(&req)

    userID, err := db.CreateUser(req.Name, req.Email)
    // ...
}
```

**RESTful API设计规范**：
```
GET    /api/users          - 获取用户列表
GET    /api/users/123      - 获取ID=123的用户
POST   /api/users          - 创建新用户
PUT    /api/users/123      - 更新ID=123的用户（完整更新）
PATCH  /api/users/123      - 更新ID=123的用户（部分更新）
DELETE /api/users/123      - 删除ID=123的用户
```

---

#### 问题5：什么是Keep-Alive长连接？如何实现？

**答案：**

**定义**：
HTTP Keep-Alive（持久连接）允许在一个TCP连接上发送多个HTTP请求/响应，避免每次请求都建立新连接。

**没有Keep-Alive的情况**（HTTP/1.0默认）：
```
请求1：建立连接 → 发送请求 → 接收响应 → 关闭连接
请求2：建立连接 → 发送请求 → 接收响应 → 关闭连接
请求3：建立连接 → 发送请求 → 接收响应 → 关闭连接

每次都要三次握手 + 四次挥手，开销巨大！
```

**使用Keep-Alive的情况**（HTTP/1.1默认）：
```
建立连接 → 请求1 → 响应1 → 请求2 → 响应2 → 请求3 → 响应3 → 空闲超时 → 关闭连接

只需要一次三次握手和四次挥手！
```

**性能提升**：
```
假设RTT（往返时延）= 50ms

无Keep-Alive（发送3个请求）：
3 × (三次握手50ms + 请求50ms + 响应50ms + 四次挥手50ms) = 600ms

有Keep-Alive（发送3个请求）：
三次握手50ms + (请求50ms + 响应50ms) × 3 + 四次挥手50ms = 400ms

提升33%性能！
```

**HTTP头部设置**：

**HTTP/1.0**（需要显式声明）：
```http
请求：
GET /api/users HTTP/1.0
Connection: keep-alive

响应：
HTTP/1.0 200 OK
Connection: keep-alive
Keep-Alive: timeout=5, max=100
```

**HTTP/1.1**（默认开启）：
```http
请求：
GET /api/users HTTP/1.1
Connection: keep-alive  # 默认就是，可以不写

响应：
HTTP/1.1 200 OK
Connection: keep-alive
Keep-Alive: timeout=5, max=100

如果要关闭：
Connection: close
```

**Keep-Alive参数**：
- **timeout**：连接空闲多久后关闭（秒）
- **max**：这个连接最多处理多少个请求

**Go语言实现**：

**服务端配置**：
```go
package main

import (
    "net/http"
    "time"
)

func main() {
    server := &http.Server{
        Addr:           ":8080",
        Handler:        http.DefaultServeMux,
        ReadTimeout:    10 * time.Second,
        WriteTimeout:   10 * time.Second,
        IdleTimeout:    30 * time.Second,  // Keep-Alive空闲超时
        MaxHeaderBytes: 1 << 20,
    }

    http.HandleFunc("/api/users", handleUsers)

    server.ListenAndServe()
}

func handleUsers(w http.ResponseWriter, r *http.Request) {
    // Go的http包默认支持Keep-Alive
    // 响应头会自动添加 Connection: keep-alive

    w.WriteHeader(http.StatusOK)
    w.Write([]byte("User data"))
}
```

**客户端使用**：
```go
package main

import (
    "net/http"
    "time"
)

func main() {
    // 创建支持连接池的HTTP客户端
    client := &http.Client{
        Transport: &http.Transport{
            MaxIdleConns:        100,              // 最大空闲连接数
            MaxIdleConnsPerHost: 10,               // 每个host的最大空闲连接
            IdleConnTimeout:     90 * time.Second, // 空闲连接超时
        },
        Timeout: 10 * time.Second,
    }

    // 发送多个请求，复用同一个TCP连接
    for i := 0; i < 10; i++ {
        resp, err := client.Get("http://api.example.com/users")
        if err != nil {
            panic(err)
        }
        resp.Body.Close()
    }
    // 这10个请求会复用同一个TCP连接！
}
```

**连接池原理**：
```
客户端维护连接池：

Host: api.example.com
  ├─ 连接1：空闲
  ├─ 连接2：使用中
  └─ 连接3：空闲

发送请求时：
1. 检查连接池是否有空闲连接
2. 有 → 复用
3. 无 → 建立新连接
4. 请求完成 → 归还到连接池
```

**Nginx配置**：
```nginx
http {
    # 客户端Keep-Alive配置
    keepalive_timeout  65s;      # 空闲65秒后关闭
    keepalive_requests 100;      # 一个连接最多处理100个请求

    upstream backend {
        server 192.168.1.10:8080;
        server 192.168.1.11:8080;

        # 上游服务器Keep-Alive配置
        keepalive 32;  # 保持32个空闲连接
    }
}
```

**注意事项**：

1. **慢速客户端攻击**：
   - 客户端故意保持连接不发送请求
   - 占用服务器连接资源
   - 解决：设置合理的 timeout

2. **连接泄漏**：
   - 客户端忘记关闭 Response.Body
   - 连接无法归还连接池
   ```go
   resp, err := client.Get(url)
   if err != nil {
       return err
   }
   defer resp.Body.Close()  // 必须关闭！
   ```

**项目应用**：
在游戏交易平台中，使用HTTP连接池复用连接，减少三次握手开销，QPS从3000提升到5000。

---

#### 问题6：HTTPS的加密原理？对称加密和非对称加密？

**答案：**

**HTTPS = HTTP + SSL/TLS**

**加密方式对比**：

**1. 对称加密**：
- **定义**：加密和解密使用同一个密钥
- **算法**：AES、DES、3DES
- **优点**：速度快
- **缺点**：密钥传输不安全

```
发送方                        接收方
原文 → [密钥K加密] → 密文 → [密钥K解密] → 原文

问题：密钥K如何安全传输？
如果密钥在网络中被截获，数据就不安全了！
```

**2. 非对称加密**：
- **定义**：公钥加密，私钥解密（或私钥签名，公钥验证）
- **算法**：RSA、ECC
- **优点**：安全，公钥可以公开
- **缺点**：速度慢（比对称加密慢100-1000倍）

```
发送方                            接收方
原文 → [接收方公钥加密] → 密文 → [接收方私钥解密] → 原文

特点：
- 公钥可以公开，任何人都能用公钥加密
- 只有持有私钥的人才能解密
```

**HTTPS的混合加密方案**：

**结合两者优点**：
1. 用**非对称加密**传输**对称加密的密钥**（安全）
2. 用**对称加密**传输实际数据（快速）

**TLS握手详细流程**：

```
客户端                                    服务器
  │                                          │
  │────── ① ClientHello ──────────────────>│
  │   (支持的加密套件、随机数Client Random)   │
  │                                          │
  │<────── ② ServerHello ──────────────────│
  │   (选择的加密套件、随机数Server Random)   │
  │                                          │
  │<────── ③ Certificate ──────────────────│
  │   (服务器证书，包含服务器公钥)             │
  │                                          │
  │<────── ④ ServerHelloDone ──────────────│
  │                                          │
  │────── ⑤ ClientKeyExchange ────────────>│
  │   (用服务器公钥加密的Pre-Master Secret)  │
  │                                          │
  │  双方根据三个随机数生成对称密钥：           │
  │  Client Random + Server Random +        │
  │  Pre-Master Secret → Master Secret      │
  │                                          │
  │────── ⑥ ChangeCipherSpec ─────────────>│
  │────── ⑦ Finished ─────────────────────>│
  │   (用对称密钥加密的握手信息摘要)           │
  │                                          │
  │<────── ⑧ ChangeCipherSpec ─────────────│
  │<────── ⑨ Finished ─────────────────────│
  │   (用对称密钥加密的握手信息摘要)           │
  │                                          │
  │          ✅ 加密通道建立                  │
  │                                          │
  │<══════ 后续数据用对称密钥加密 ═══════════>│
```

**证书验证（防止中间人攻击）**：

**问题**：如何确保服务器公钥是真实的，不是被篡改的？

**解决**：数字证书 + CA认证

```
证书内容：
- 域名：www.example.com
- 公钥：MIIBIjANBgkqhkiG9w0...
- 有效期：2024-01-01 至 2025-01-01
- 颁发者：Let's Encrypt
- 数字签名：CA用私钥对以上内容的哈希值加密

客户端验证：
1. 浏览器内置CA公钥
2. 用CA公钥解密数字签名，得到哈希值A
3. 对证书内容计算哈希值，得到哈希值B
4. 比较A和B，一致则证书有效
```

**中间人攻击示例**：

```
无证书验证（不安全）：
客户端 → 黑客（假装是服务器）→ 真实服务器
黑客可以截获和篡改所有数据！

有证书验证（安全）：
客户端验证证书 → 发现证书是黑客自签名的 → 浏览器警告 → 用户不信任
```

**Go语言实现HTTPS服务器**：

```go
package main

import (
    "log"
    "net/http"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        w.Write([]byte("Hello HTTPS!"))
    })

    // 启动HTTPS服务器
    // cert.pem: 证书文件（包含公钥）
    // key.pem: 私钥文件
    log.Println("Starting HTTPS server on :443")
    err := http.ListenAndServeTLS(":443", "cert.pem", "key.pem", nil)
    if err != nil {
        log.Fatal(err)
    }
}
```

**生成自签名证书（测试用）**：
```bash
# 生成私钥
openssl genrsa -out key.pem 2048

# 生成证书（自签名）
openssl req -new -x509 -key key.pem -out cert.pem -days 365
```

**HTTPS性能优化**：

1. **Session复用**：
   - 缓存对称密钥，下次连接直接使用
   - 避免重复TLS握手

2. **HTTP/2**：
   - 一个TCP连接复用多个请求
   - 减少TLS握手次数

3. **证书链优化**：
   - 减少证书链长度
   - 启用OCSP Stapling

**项目应用**：
在跨境电商平台中，全站启用HTTPS保护用户隐私，使用Let's Encrypt免费证书，配置TLS 1.3提升性能。

---

### 2.2 HTTP实战问题

#### 问题7：如何设计RESTful API？

**答案：**

**REST核心原则**：
1. **资源（Resource）**：URL表示资源
2. **动词（Verb）**：HTTP方法表示操作
3. **无状态（Stateless）**：每次请求独立
4. **统一接口（Uniform Interface）**

**RESTful API设计规范**：

**1. URL设计**：
```
✅ 好的设计（资源导向）：
GET    /api/v1/users              获取用户列表
GET    /api/v1/users/123          获取ID=123的用户
POST   /api/v1/users              创建用户
PUT    /api/v1/users/123          更新用户123（完整更新）
PATCH  /api/v1/users/123          更新用户123（部分更新）
DELETE /api/v1/users/123          删除用户123

GET    /api/v1/users/123/orders   获取用户123的订单列表
POST   /api/v1/users/123/orders   为用户123创建订单

❌ 不好的设计（动词导向）：
GET    /api/getUsers
POST   /api/createUser
POST   /api/updateUser
POST   /api/deleteUser
```

**2. HTTP方法使用**：

| 方法 | 作用 | 幂等性 | 安全性 |
|------|------|--------|--------|
| GET | 查询 | ✅ | ✅ |
| POST | 创建 | ❌ | ❌ |
| PUT | 完整更新 | ✅ | ❌ |
| PATCH | 部分更新 | ❌ | ❌ |
| DELETE | 删除 | ✅ | ❌ |

**幂等性**：多次调用结果相同
**安全性**：不改变资源状态

**3. 状态码使用**：
```go
// 成功
200 OK              - GET、PUT、PATCH成功
201 Created         - POST创建成功
204 No Content      - DELETE成功

// 客户端错误
400 Bad Request     - 参数错误
401 Unauthorized    - 未认证
403 Forbidden       - 无权限
404 Not Found       - 资源不存在
409 Conflict        - 资源冲突（如用户名已存在）
422 Unprocessable Entity - 参数验证失败

// 服务器错误
500 Internal Server Error - 服务器错误
503 Service Unavailable   - 服务不可用
```

**4. 响应格式统一**：
```json
成功响应：
{
  "code": 0,
  "message": "success",
  "data": {
    "id": 123,
    "name": "张三"
  }
}

错误响应：
{
  "code": 400,
  "message": "参数错误",
  "errors": [
    {
      "field": "email",
      "message": "邮箱格式不正确"
    }
  ]
}
```

**Go实现完整示例**：

```go
package main

import (
    "encoding/json"
    "net/http"
    "strconv"
    "strings"
)

// 统一响应结构
type Response struct {
    Code    int         `json:"code"`
    Message string      `json:"message"`
    Data    interface{} `json:"data,omitempty"`
}

// 用户结构
type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

// 路由注册
func main() {
    http.HandleFunc("/api/v1/users", usersHandler)
    http.HandleFunc("/api/v1/users/", userHandler) // 注意末尾的/

    http.ListenAndServe(":8080", nil)
}

// 用户列表处理
func usersHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        // GET /api/v1/users - 获取用户列表
        getUsers(w, r)
    case http.MethodPost:
        // POST /api/v1/users - 创建用户
        createUser(w, r)
    default:
        sendError(w, http.StatusMethodNotAllowed, "方法不允许")
    }
}

// 单个用户处理
func userHandler(w http.ResponseWriter, r *http.Request) {
    // 提取用户ID
    path := strings.TrimPrefix(r.Path, "/api/v1/users/")
    userID, err := strconv.Atoi(path)
    if err != nil {
        sendError(w, http.StatusBadRequest, "无效的用户ID")
        return
    }

    switch r.Method {
    case http.MethodGet:
        // GET /api/v1/users/123 - 获取用户
        getUser(w, r, userID)
    case http.MethodPut:
        // PUT /api/v1/users/123 - 更新用户
        updateUser(w, r, userID)
    case http.MethodDelete:
        // DELETE /api/v1/users/123 - 删除用户
        deleteUser(w, r, userID)
    default:
        sendError(w, http.StatusMethodNotAllowed, "方法不允许")
    }
}

// 获取用户列表
func getUsers(w http.ResponseWriter, r *http.Request) {
    // 解析查询参数
    page := r.URL.Query().Get("page")
    pageSize := r.URL.Query().Get("page_size")

    // 从数据库查询（省略）
    users := []User{
        {ID: 1, Name: "张三", Email: "zhangsan@example.com"},
        {ID: 2, Name: "李四", Email: "lisi@example.com"},
    }

    sendSuccess(w, http.StatusOK, "success", users)
}

// 创建用户
func createUser(w http.ResponseWriter, r *http.Request) {
    var req User
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        sendError(w, http.StatusBadRequest, "请求体解析失败")
        return
    }

    // 参数验证
    if req.Name == "" || req.Email == "" {
        sendError(w, http.StatusBadRequest, "姓名和邮箱不能为空")
        return
    }

    // 创建用户（省略数据库操作）
    newUser := User{
        ID:    123,
        Name:  req.Name,
        Email: req.Email,
    }

    sendSuccess(w, http.StatusCreated, "创建成功", newUser)
}

// 获取单个用户
func getUser(w http.ResponseWriter, r *http.Request, userID int) {
    // 从数据库查询（省略）
    user := User{ID: userID, Name: "张三", Email: "zhangsan@example.com"}

    sendSuccess(w, http.StatusOK, "success", user)
}

// 更新用户
func updateUser(w http.ResponseWriter, r *http.Request, userID int) {
    var req User
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        sendError(w, http.StatusBadRequest, "请求体解析失败")
        return
    }

    // 更新数据库（省略）
    updatedUser := User{
        ID:    userID,
        Name:  req.Name,
        Email: req.Email,
    }

    sendSuccess(w, http.StatusOK, "更新成功", updatedUser)
}

// 删除用户
func deleteUser(w http.ResponseWriter, r *http.Request, userID int) {
    // 删除数据库记录（省略）

    sendSuccess(w, http.StatusNoContent, "删除成功", nil)
}

// 发送成功响应
func sendSuccess(w http.ResponseWriter, statusCode int, message string, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)

    resp := Response{
        Code:    0,
        Message: message,
        Data:    data,
    }
    json.NewEncoder(w).Encode(resp)
}

// 发送错误响应
func sendError(w http.ResponseWriter, statusCode int, message string) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(statusCode)

    resp := Response{
        Code:    statusCode,
        Message: message,
    }
    json.NewEncoder(w).Encode(resp)
}
```

**使用Gin框架（推荐）**：
```go
package main

import (
    "github.com/gin-gonic/gin"
    "net/http"
)

func main() {
    r := gin.Default()

    // API版本分组
    v1 := r.Group("/api/v1")
    {
        users := v1.Group("/users")
        {
            users.GET("", getUsers)           // GET /api/v1/users
            users.POST("", createUser)        // POST /api/v1/users
            users.GET("/:id", getUser)        // GET /api/v1/users/123
            users.PUT("/:id", updateUser)     // PUT /api/v1/users/123
            users.DELETE("/:id", deleteUser)  // DELETE /api/v1/users/123
        }
    }

    r.Run(":8080")
}

func getUsers(c *gin.Context) {
    page := c.DefaultQuery("page", "1")
    pageSize := c.DefaultQuery("page_size", "10")

    // 查询数据库...
    users := []User{
        {ID: 1, Name: "张三", Email: "zhangsan@example.com"},
    }

    c.JSON(http.StatusOK, gin.H{
        "code":    0,
        "message": "success",
        "data":    users,
    })
}

func createUser(c *gin.Context) {
    var req User
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{
            "code":    400,
            "message": "参数错误",
        })
        return
    }

    // 创建用户...
    c.JSON(http.StatusCreated, gin.H{
        "code":    0,
        "message": "创建成功",
        "data":    req,
    })
}

func getUser(c *gin.Context) {
    id := c.Param("id")

    // 查询数据库...
    user := User{ID: 1, Name: "张三", Email: "zhangsan@example.com"}

    c.JSON(http.StatusOK, gin.H{
        "code":    0,
        "message": "success",
        "data":    user,
    })
}

func updateUser(c *gin.Context) {
    id := c.Param("id")
    var req User
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{
            "code":    400,
            "message": "参数错误",
        })
        return
    }

    // 更新数据库...
    c.JSON(http.StatusOK, gin.H{
        "code":    0,
        "message": "更新成功",
        "data":    req,
    })
}

func deleteUser(c *gin.Context) {
    id := c.Param("id")

    // 删除数据库记录...
    c.JSON(http.StatusNoContent, gin.H{
        "code":    0,
        "message": "删除成功",
    })
}
```

**项目应用**：
在游戏交易平台中，严格遵循RESTful API设计规范，使用Gin框架开发API，接口清晰易用，前端对接效率提升50%。

---

## 第三部分：数据库技术（MySQL、MongoDB、Redis）

### 2.1 MySQL相关

#### 问题11：MySQL索引的底层实现？B+树和B树的区别？

**答案：**

**B+树特点**：
1. **所有数据存储在叶子节点**：
   - 非叶子节点只存储索引键
   - 叶子节点存储完整数据或主键
   - 叶子节点之间通过双向链表连接

2. **与B树的区别**：
   - B树：每个节点都存储数据
   - B+树：只有叶子节点存储数据，非叶子节点只存索引

3. **优势**：
   - 范围查询效率高（叶子节点链表遍历）
   - 单次IO可以读取更多索引（非叶子节点不存数据，单页可存更多键）
   - 树的高度更低（3-4层支撑千万级数据）

**索引类型**：

**重要概念：一张表有多棵B+树**

在InnoDB中，**每个索引都是一棵独立的B+树**！

**示例**：假设有这样一张表
```sql
CREATE TABLE users (
    id INT PRIMARY KEY,           -- 聚簇索引
    name VARCHAR(50),
    age INT,
    email VARCHAR(100),
    INDEX idx_name (name),        -- 非聚簇索引1（你只指定了name）
    INDEX idx_age (age),          -- 非聚簇索引2（你只指定了age）
    INDEX idx_email (email)       -- 非聚簇索引3（你只指定了email）
);
```

**重要说明**：
虽然创建索引时你只指定了单列（如 `name`），但 **InnoDB会自动在非聚簇索引的叶子节点中添加主键**。

**为什么要自动添加主键？**
- 非聚簇索引必须通过主键才能回表查找完整数据
- 所以InnoDB自动在每个非聚簇索引的叶子节点中存储主键值
- 这是InnoDB的内部实现，不需要你手动指定

**这张表在磁盘上有4棵B+树**：

```
┌─────────────────────────────────────────────────────────────┐
│                         users 表                            │
└─────────────────────────────────────────────────────────────┘
                           │
     ┌─────────────────────┼────────────────────┬─────────────┐
     │                     │                    │             │
     ▼                     ▼                    ▼             ▼
┌──────────┐          ┌──────────┐        ┌──────────┐  ┌──────────┐
│ 聚簇索引  │          │ name索引  │        │ age索引   │  │email索引 │
│   树1    │          │   树2     │        │   树3    │  │   树4    │
│ (主键id) │           │(非聚簇)   │        │(非聚簇)   │  │(非聚簇)  │
└──────────┘          └──────────┘        └──────────┘  └──────────┘

树1（聚簇索引）           树2（name索引）       树3（age索引）
叶子节点存：              叶子节点存：          叶子节点存：
[id=1, name='Alice',     [name='Alice', id=1]  [age=25, id=1]
 age=25, email='...']    [name='Bob', id=2]    [age=28, id=2]
[id=2, name='Bob',       [name='Tom', id=3]    [age=30, id=3]
 age=28, email='...']
[id=3, ...]

← 存完整行数据            ← 只存(索引值+主键)    ← 只存(索引值+主键)
```

**为什么这样设计？**

1. **聚簇索引树（1棵）**：
   - 存储实际数据
   - 数据只能按一种顺序物理存储，所以只有1棵

2. **非聚簇索引树（N棵）**：
   - 每个索引一棵树
   - 叶子节点只存(索引列值 + 主键值)
   - **主键值是InnoDB自动添加的**，即使你创建索引时没有指定
   - 通过主键可以回到聚簇索引树查完整数据

   **示例对比**：
   ```
   -- 你创建索引时写的：
   INDEX idx_name (name)

   -- InnoDB 实际存储的：
   叶子节点：[name='Alice', id=1]
             ^^^^^^^^      ^^^^
             你指定的      InnoDB自动加的主键
   ```

**回表的本质：在两棵树之间跳转**

```
-- 查询：SELECT * FROM users WHERE name = 'Alice';

步骤1：在name索引树中查找，找到[name='Alice', id=1]
步骤2：拿着id=1去聚簇索引树中查找，找到 [id=1, name='Alice', age=25, email='alice@example.com']
步骤3：返回完整数据

这就是"回表"：从非聚簇索引树 → 聚簇索引树
```

**空间和性能影响**：

```
-- 假设users表有100万行数据

磁盘空间占用：
- 聚簇索引树（id）：约 100MB（存完整数据）
- name索引树：约20MB（只存name + id）
- age索引树：约15MB（只存age + id）
- email索引树：约25MB（只存email + id）
总计：160MB

如果有10个索引，空间占用会更大！

写入性能影响：
INSERT一条新数据：
- 需要在4棵树中都插入新节点
- 可能触发4次树的平衡调整
索引越多，写入越慢！
```

**索引设计原则**：
- 只在高频查询的列上建索引
- 使用联合索引代替多个单列索引
- 不要盲目建立过多索引（每个索引都是一棵树！）

---

1. **聚簇索引（Clustered Index / 主键索引）**：
   **定义**：数据行和索引存储在一起，叶子节点直接包含完整的行数据。
   **结构示意图**：
   ```
   聚簇索引（主键id）

                [10]                ← 非叶子节点：只存索引
            /          \
         [5]            [15]        ← 非叶子节点：只存索引
        /   \         /     \
   [1,行]  [6,行]  [11,行]  [16,行]  ← 叶子节点：存储完整行数据
           
    ↓        ↓       ↓        ↓
   id=1     id=6    id=11    id=16
   name     name    name     name
   age      age     age      age
   email    email   email    email
   ...      ...     ...      ...
   ```

   **重要说明：B+ 树非叶子节点只是"路标"**（高频面试点）

   **问题**：如果非叶子节点有值 5，查询 id=5 时怎么办？

   **答案**：非叶子节点的值只是用于导航的"路标"，不存储实际数据！

   **查找 id=5 的完整过程**：
   ```
   SELECT * FROM users WHERE id = 5;

   步骤1：从根节点[10]开始
          判断：5 < 10 → 走左子树

   步骤2：到达非叶子节点[5]
          判断：5 >= 5 → 走右子树
          关键：这里的[5]只是路标，不包含数据！

   步骤3：继续向下到达叶子节点
          找到 id=5 的叶子节点
          返回完整行数据：{id:5, name:'Alice', age:25, email:'...'}
   ```

   **B+树的核心特性**：
   - **所有实际数据都在叶子节点**（这是B+树与B树的最大区别）
   - 非叶子节点只存"分隔值"，用于导航查找路径
   - 即使非叶子节点有值5，叶子节点中也必然有id=5的完整数据
   - 所有叶子节点通过链表连接，便于范围查询

   **对比B树**：
   ```
   B树：非叶子节点也存数据，找到就返回
   B+树：非叶子节点只是路标，必须到叶子节点才有数据
   ```

   **特点**：
   - 叶子节点存储**完整行数据**（所有字段）
   - 一张表只有**一个**聚簇索引（因为数据只能按一种顺序物理存储）
   - 查询时直接通过索引找到数据，**无需回表**
   - 数据物理存储顺序与索引顺序一致，范围查询效率高

   **聚簇索引和主键的关系**：

   聚簇索引 ≠ 主键，但在InnoDB中通常是同一个东西。

   **InnoDB 选择聚簇索引的优先级**：
   ```
   如果定义了主键（PRIMARY KEY）
      → 主键就是聚簇索引

   如果没有主键，但有唯一非空索引（UNIQUE NOT NULL）
      → 第一个唯一非空索引作为聚簇索引

   如果既没有主键，也没有唯一非空索引
      → InnoDB 自动生成隐藏的 6 字节 ROWID 列作为聚簇索引
   ```

   **无论哪种情况，叶子节点都存储完整行数据！**

   **具体示例**：

   ```sql
   -- 情况1：有主键（最常见）
   CREATE TABLE users (
       id INT PRIMARY KEY,      -- id 是聚簇索引
       name VARCHAR(50),
       email VARCHAR(100)
   );
   -- 数据按 id 顺序物理存储

   -- 情况2：没有主键，有唯一非空索引
   CREATE TABLE logs (
       log_id VARCHAR(36) UNIQUE NOT NULL,  -- log_id 是聚簇索引
       message TEXT,
       created_at DATETIME
   );
   -- 数据按 log_id 顺序物理存储

   -- 情况3：没有主键，也没有唯一非空索引
   CREATE TABLE temp_data (
       name VARCHAR(50),
       value INT
   );
   -- InnoDB 自动创建隐藏的 _rowid 列（用户看不到）
   -- 数据按隐藏的 _rowid 顺序物理存储
   -- 叶子节点仍然存储完整行：(_rowid, name, value)
   ```

   **为什么InnoDB必须有聚簇索引？**
   - InnoDB的数据必须按某种顺序组织在B+树中
   - 没有聚簇索引就无法存储数据
   - 所以即使你不定义主键，InnoDB也会自动创建隐藏的

   **其他存储引擎的差异**：
   ```
   MyISAM：
   - 没有聚簇索引概念
   - 索引和数据完全分离
   - 所有索引（包括主键）都是非聚簇索引
   - 索引叶子节点存储的是数据文件的物理地址指针
   ```

   **如果一行数据太大怎么办？行溢出机制（Row Overflow）**：

   InnoDB的页（Page）大小默认是 **16KB**，一个页必须至少能存放2行数据。如果单行数据过大：

   **行溢出处理方式**：
   ```
   普通情况（行数据 < 8KB）：
   ┌─────────────────────┐
   │  InnoDB 数据页       │
   │  (16KB)             │
   │ ┌─────────────────┐ │
   │ │ 行1: id, name,  │ │ ← 完整数据都在页内
   │ │      age, email │ │
   │ ├─────────────────┤ │
   │ │ 行2: ...        │ │
   │ └─────────────────┘ │
   └─────────────────────┘

   行溢出情况（行数据 > 8KB）：
   ┌─────────────────────┐         ┌──────────────────┐
   │  InnoDB 数据页       │         │  溢出页           │
   │  (16KB)             │         │  (Overflow Page) │
   │ ┌─────────────────┐ │         │ ┌──────────────┐ │
   │ │ 行1:            │ │  指针    │ │ article 剩余  │ │
   │ │  id: 1          │ │ ──────> │ │ 的大量文本...  │ │
   │ │  name: "..."    │ │         │ │ 数据存在这里   │ │
   │ │  article: 前768 │ │         │ └──────────────┘ │
   │ │  字节 + 指针     │ │         └──────────────────┘
   │ └─────────────────┘ │
   └─────────────────────┘
   ```

   **两种行格式的差异**：

   1. **COMPACT / REDUNDANT 格式**（旧格式）：
      - 大字段（VARCHAR、TEXT、BLOB）的**前 768 字节**存储在原始页
      - 剩余数据存储到溢出页
      - 原始页中保留 20 字节指针指向溢出页

   2. **DYNAMIC / COMPRESSED 格式**（MySQL 5.7+ 默认）：
      - 如果行数据过大，大字段**完全存储在溢出页**
      - 原始页中**只保留 20 字节指针**
      - 更节省主页空间，适合现代应用

   **实际案例**：
   ```sql
   -- 创建包含大字段的表
   CREATE TABLE articles (
       id INT PRIMARY KEY,
       title VARCHAR(200),
       content TEXT,           -- 可能很大
       author VARCHAR(50)
   ) ENGINE=InnoDB ROW_FORMAT=DYNAMIC;

   -- 插入一篇长文章（content 字段 100KB）
   INSERT INTO articles VALUES (1, '标题', '很长很长的文章内容...', '作者');

   -- 存储方式：
   -- 数据页：id=1, title='标题', content指针(20字节), author='作者'
   -- 溢出页：content 的完整 100KB 数据
   ```

   **性能影响**：
   - 查询`SELECT id, title FROM articles`不需要读取溢出页，性能好
   - 查询`SELECT * FROM articles`需要读取溢出页，性能下降
   - **优化建议**：避免`SELECT *`，只查询需要的字段

   **最大行大小限制**：
   - InnoDB最大行大小约为 **8KB**（页的一半，确保一页至少存2行）
   - 但通过溢出页，实际可以存储更大的行（TEXT/BLOB 最大 4GB）
   - 如果所有非溢出列加起来超过8KB，会报错：`Row size too large`

2. **非聚簇索引（Non-Clustered Index / 辅助索引 / 二级索引）**：

   **定义**：索引和数据分开存储，叶子节点只存储索引列值和主键值（主键值是 InnoDB 自动添加的）。

   **结构示意图**：
   ```
   非聚簇索引（普通索引name）

         ["Mike"]       ← 非叶子节点：只存索引
        /        \
    ["Alice"]  ["Tom"]  ← 非叶子节点：只存索引
     /    \     /    \
   [Alice, [Bob,  [Mike, [Tom,  ← 叶子节点：只存(索引值 + 主键)
    id=3]  id=7]  id=1]  id=9]
     ↓      ↓      ↓      ↓
   找到主键后，需要再去聚簇索引查找完整数据
   ```

   **什么是回表（Table Lookup）**：

   回表是指通过**非聚簇索引查询数据时的二次查找过程**：

   步骤示例（查询：`SELECT * FROM users WHERE name = 'Alice'`）：
   ```
   第1步：查找非聚簇索引（name索引）
   找到 name='Alice' 对应的主键 id=3

   第2步：拿着 id=3 去聚簇索引（主键索引）查找
   找到 id=3 的完整行数据：{id:3, name:'Alice', age:25, email:'alice@example.com'}

   第3步：返回完整数据
   ```

   **回表的性能影响**：
   - 回表需要**两次B+树查找**（先查非聚簇索引，再查聚簇索引）
   - 大量回表会导致**随机IO**，性能下降

   **如何避免回表**：使用**覆盖索引（Covering Index）**
   ```sql
   -- 场景：只查询索引中包含的列
   CREATE INDEX idx_user_name_age ON users(name, age);

   -- 不需要回表（索引已包含name和age）
   SELECT name, age FROM users WHERE name = 'Alice';

   -- 需要回表（索引没有email字段）
   SELECT name, age, email FROM users WHERE name = 'Alice';
   ```

3. **联合索引（Composite Index / 复合索引）**：
   **定义**：多个列组合成一个索引。
   **最左前缀原则（Leftmost Prefix Principle）**：
   联合索引遵循**最左匹配原则**，索引的使用必须从最左边的列开始，且不能跳过中间的列。

   **示例：假设有联合索引 `INDEX(a, b, c)`**

   **索引结构示意**：
   ```
   联合索引 (a, b, c)

   叶子节点按 a → b → c 的顺序排序：
   (1, 2, 3) → (1, 2, 5) → (1, 3, 1) → (2, 1, 4) → (2, 2, 2)
    ↑   ↑   ↑
    a   b   c

   排序规则：
   1. 先按a排序
   2. a相同时按b排序
   3. a、b都相同时按c排序
   ```

   **哪些查询可以使用该索引**：

   **可以使用索引的情况**（从左开始连续匹配）：
   ```sql
   WHERE a = 1                     -- 使用索引(a)
   WHERE a = 1 AND b = 2           -- 使用索引(a, b)
   WHERE a = 1 AND b = 2 AND c = 3 -- 使用索引(a, b, c) 完整
   WHERE a = 1 AND c = 3           -- 使用索引(a)，c无法使用
   WHERE a > 1 AND b = 2           -- 使用索引(a)，b无法使用（a是范围查询后中断）
   ```

   **无法使用索引的情况**（不从最左开始）：
   ```sql
   WHERE b = 2                    -- 不使用索引（缺少a）
   WHERE c = 3                    -- 不使用索引（缺少a）
   WHERE b = 2 AND c = 3          -- 不使用索引（缺少a）
   ```

   **记忆口诀**：
   - "联合索引像字典，必须从首字母查"
   - 就像查字典，你不能跳过拼音首字母直接查第二个字母

   **实际案例**：
   ```sql
   -- 订单表联合索引
   CREATE INDEX idx_order ON orders(user_id, status, created_at);

   -- 高效查询（命中索引）
   SELECT * FROM orders WHERE user_id = 123;
   SELECT * FROM orders WHERE user_id = 123 AND status = 'paid';
   SELECT * FROM orders WHERE user_id = 123 AND status = 'paid' AND created_at > '2024-01-01';

   -- 无法使用索引（跳过了user_id）
   SELECT * FROM orders WHERE status = 'paid';
   SELECT * FROM orders WHERE created_at > '2024-01-01';

   -- 部分使用索引（只用到user_id）
   SELECT * FROM orders WHERE user_id = 123 AND created_at > '2024-01-01';
   -- 原因：跳过了中间的status
   ```

   **范围查询中断原则**：
   ```sql
   -- 索引：INDEX(a, b, c)

   WHERE a = 1 AND b > 2 AND c = 3
   -- 使用索引(a, b)，c无法使用
   -- 原因：b是范围查询，导致后续的c无法使用索引

   WHERE a = 1 AND b = 2 AND c > 3
   -- 使用索引(a, b, c) 完整
   -- 原因：范围查询在最后，不影响前面的列
   ```

**索引优化原则**：
```sql
-- 1. 选择性高的列建索引
CREATE INDEX idx_user_email ON users(email);

-- 2. 联合索引遵循最左前缀
CREATE INDEX idx_order ON orders(user_id, status, created_at);
-- 可以使用：user_id | user_id+status | user_id+status+created_at
-- 不能使用：status | created_at

-- 3. 覆盖索引避免回表
CREATE INDEX idx_user_info ON users(id, name, email);
SELECT id, name, email FROM users WHERE id = 1; -- 不需要回表

-- 4. 避免索引失效
-- 不好：函数操作
SELECT * FROM users WHERE YEAR(created_at) = 2024;
-- 好：
SELECT * FROM users WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';

-- 不好：隐式类型转换
SELECT * FROM users WHERE phone = 12345; -- phone是varchar
-- 好：
SELECT * FROM users WHERE phone = '12345';
```

**项目应用**：
在游戏交易平台的订单表中，建立联合索引 `(user_id, status, created_at)`，查询用户订单时命中索引，查询性能从300ms降至5ms。

---

#### 问题12：MySQL事务隔离级别？如何解决幻读？

**答案：**

**四种隔离级别**：

| 隔离级别                | 脏读  | 不可重复读 | 幻读  |
|---------------------|-----|-------|-----|
| READ UNCOMMITTED    | 可能  | 可能    | 可能  |
| READ COMMITTED      | 不可能 | 可能    | 可能  |
| REPEATABLE READ（默认） | 不可能 | 不可能   | 可能* |
| SERIALIZABLE        | 不可能 | 不可能   | 不可能 |

**问题解释**：
1. **脏读**：读到其他事务未提交的数据
2. **不可重复读**：同一事务内多次读取同一行，结果不同（其他事务修改了）
3. **幻读**：同一事务内多次查询，结果集行数不同（其他事务插入或删除了）

**MySQL如何解决幻读**：

1. **MVCC（多版本并发控制）**：
```
每行记录有隐藏列：
- DB_TRX_ID：最后修改事务ID
- DB_ROLL_PTR：回滚指针，指向undo log
- DB_ROW_ID：隐藏主键

Read View（快照）：
- 事务开始时创建快照
- 读取时根据事务ID判断可见性
- 只能看到快照创建前已提交的数据
```

2. **Next-Key Lock（间隙锁+记录锁）**：
```sql
-- 假设表中有id: 1, 5, 10
BEGIN;
SELECT * FROM users WHERE id > 3 AND id < 8 FOR UPDATE;
-- 锁定：
-- 记录锁：id=5
-- 间隙锁：(3, 5), (5, 8)
-- 其他事务无法在(3, 8)范围内插入新记录
```

**隔离级别选择**：
- **READ COMMITTED**（Oracle默认）：
  - 避免脏读
  - 性能好
  - 适合大多数业务场景

- **REPEATABLE READ**（MySQL默认）：
  - 避免不可重复读
  - 通过MVCC+Next-Key Lock基本避免幻读
  - 适合对一致性要求高的场景

**项目应用示例**：
```go
// 游戏交易平台防止超卖
func CreateOrder(ctx context.Context, productID, quantity int) error {
    tx, _ := db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelRepeatableRead,
    })
    defer tx.Rollback()

    // 使用FOR UPDATE锁定库存行
    var stock int
    err := tx.QueryRow("SELECT stock FROM products WHERE id = ? FOR UPDATE", productID).Scan(&stock)
    if err != nil {
        return err
    }

    if stock < quantity {
        return errors.New("insufficient stock")
    }

    // 扣减库存
    _, err = tx.Exec("UPDATE products SET stock = stock - ? WHERE id = ?", quantity, productID)
    if err != nil {
        return err
    }

    // 创建订单
    _, err = tx.Exec("INSERT INTO orders (product_id, quantity) VALUES (?, ?)", productID, quantity)
    if err != nil {
        return err
    }

    return tx.Commit()
}
```

---

#### 问题13：MySQL如何进行分库分表？什么时候需要分库分表？

**答案：**

**何时需要分库分表**：
1. **单表数据量 > 1000万**：查询性能下降
2. **QPS过高**：单库连接数、IO成为瓶颈
3. **存储空间不足**：单库存储达到上限

**分库分表策略**：

1. **垂直拆分**：
```
订单库：
- orders（订单表）
- order_items（订单明细表）

用户库：
- users（用户表）
- user_profiles（用户资料表）

商品库：
- products（商品表）
- product_details（商品详情表）
```

2. **水平拆分**：
```
按用户ID取模：
- orders_0（user_id % 4 = 0）
- orders_1（user_id % 4 = 1）
- orders_2（user_id % 4 = 2）
- orders_3（user_id % 4 = 3）

按时间范围：
- orders_2024_q1
- orders_2024_q2
- orders_2024_q3
- orders_2024_q4
```

**分片键选择**：
1. **用户ID**：
   - 优点：数据均匀分布
   - 缺点：跨用户查询困难

2. **订单ID**：
   - 优点：订单查询快
   - 缺点：用户维度查询困难

3. **时间**：
   - 优点：归档方便
   - 缺点：热点数据集中

**分库分表带来的问题及解决方案**：

1. **分布式事务**：
```go
// 方案1：最终一致性（推荐）
func CreateOrder(ctx context.Context, order *Order) error {
    // 1. 创建订单
    if err := createOrderInDB(ctx, order); err != nil {
        return err
    }

    // 2. 发送消息到Kafka
    msg := &OrderCreatedEvent{OrderID: order.ID}
    if err := kafkaProducer.Send(ctx, msg); err != nil {
        log.Error("send kafka failed, will retry", err)
        // 异步重试或定时任务补偿
    }

    return nil
}

// 消费者更新库存
func ConsumeOrderCreated(msg *OrderCreatedEvent) error {
    return updateStock(msg.OrderID)
}

// 方案2：分布式事务框架（Seata等）
// 性能较差，不推荐
```

2. **全局唯一ID**：
```go
// 雪花算法（Snowflake）
type Snowflake struct {
    mu          sync.Mutex
    timestamp   int64 // 41位时间戳
    workerID    int64 // 10位机器ID
    sequence    int64 // 12位序列号
}

func (s *Snowflake) NextID() int64 {
    s.mu.Lock()
    defer s.mu.Unlock()

    now := time.Now().UnixNano() / 1e6
    if now == s.timestamp {
        s.sequence = (s.sequence + 1) & 4095
        if s.sequence == 0 {
            // 等待下一毫秒
            for now <= s.timestamp {
                now = time.Now().UnixNano() / 1e6
            }
        }
    } else {
        s.sequence = 0
    }

    s.timestamp = now
    id := ((now - epoch) << 22) | (s.workerID << 12) | s.sequence
    return id
}
```

3. **跨分片查询**：
```go
// 方案1：并行查询多个分片
func GetUserOrders(userID int64) ([]*Order, error) {
    // 查询所有分片
    var wg sync.WaitGroup
    results := make([][]*Order, 4)
    for i := 0; i < 4; i++ {
        wg.Add(1)
        go func(shard int) {
            defer wg.Done()
            db := getShardDB(shard)
            orders, _ := queryOrders(db, userID)
            results[shard] = orders
        }(i)
    }
    wg.Wait()

    // 合并结果
    var allOrders []*Order
    for _, orders := range results {
        allOrders = append(allOrders, orders...)
    }
    return allOrders, nil
}

// 方案2：ES等搜索引擎（推荐）
// 将数据同步到ES，复杂查询走ES
```

**项目应用**：
在游戏交易平台中，订单表按user_id分4个库，每个库8张表（共32分片），支撑千万级订单，单次查询<10ms。

---

### 2.2 MongoDB相关

#### 问题14：MongoDB的优势？什么场景下使用？

**答案：**

**MongoDB特点**：
1. **文档型数据库**：
   - 存储BSON格式（类JSON）
   - Schema灵活，无需预定义表结构
   - 支持嵌套文档和数组

2. **高性能**：
   - 内存映射文件
   - 索引支持（B树）
   - 水平扩展（分片）

3. **高可用**：
   - 副本集（Replica Set）
   - 自动故障转移

**适用场景**：

1. **非结构化数据**：
```
// 游戏道具元数据（属性不固定）
{
  "_id": ObjectId("..."),
  "itemID": 12345,
  "name": "神剑",
  "type": "weapon",
  "attributes": {
    "attack": 100,
    "defense": 20,
    "critRate": 0.15,
    "skill": {
      "name": "破天斩",
      "damage": 500,
      "cooldown": 10
    }
  },
  "rarity": "legendary",
  "customFields": {
    "color": "red",
    "effect": "fire"
  }
}
```

2. **日志和事件存储**：
```
// 用户行为日志
{
  "userID": 10001,
  "action": "purchase",
  "timestamp": ISODate("2024-01-15T10:30:00Z"),
  "details": {
    "productID": 88888,
    "amount": 99.99,
    "paymentMethod": "credit_card"
  },
  "metadata": {
    "ip": "1.2.3.4",
    "userAgent": "...",
    "referrer": "..."
  }
}
```

3. **实时分析**：
```
// 聚合查询
db.orders.aggregate([
  {
    $match: {
      createdAt: { $gte: ISODate("2024-01-01"), $lt: ISODate("2024-02-01") }
    }
  },
  {
    $group: {
      _id: "$productID",
      totalSales: { $sum: "$amount" },
      count: { $sum: 1 }
    }
  },
  {
    $sort: { totalSales: -1 }
  },
  {
    $limit: 10
  }
])
```

**MongoDB vs MySQL**：

| 维度   | MongoDB     | MySQL        |
|------|-------------|--------------|
| 数据模型 | 文档型，灵活      | 关系型，固定Schema |
| 扩展性  | 水平扩展容易      | 垂直扩展为主       |
| 事务   | 4.0+支持多文档事务 | 完整ACID支持     |
| 查询   | 聚合管道强大      | SQL标准，生态好    |
| 适用场景 | 非结构化数据、日志   | 结构化数据、强一致性   |

**项目应用示例**：
```go
// 游戏交易平台存储游戏道具
type GameItem struct {
    ID         primitive.ObjectID     `bson:"_id,omitempty"`
    ItemID     int64                  `bson:"itemID"`
    Name       string                 `bson:"name"`
    Attributes map[string]interface{} `bson:"attributes"` // 灵活属性
    CreatedAt  time.Time             `bson:"createdAt"`
}

func SaveGameItem(ctx context.Context, item *GameItem) error {
    collection := mongoClient.Database("game").Collection("items")
    _, err := collection.InsertOne(ctx, item)
    return err
}

// 查询稀有度为legendary的道具
func GetLegendaryItems(ctx context.Context) ([]*GameItem, error) {
    collection := mongoClient.Database("game").Collection("items")
    filter := bson.M{"attributes.rarity": "legendary"}
    cursor, err := collection.Find(ctx, filter)
    if err != nil {
        return nil, err
    }
    defer cursor.Close(ctx)

    var items []*GameItem
    if err := cursor.All(ctx, &items); err != nil {
        return nil, err
    }
    return items, nil
}
```

---

### 2.3 Redis相关

#### 问题15：Redis的数据结构及底层实现？

**答案：**

**5种基本数据结构**：

1. **String（字符串）**：
```
底层：SDS（Simple Dynamic String）
应用：缓存、计数器、分布式锁

示例：
SET user:1000:name "张三"
INCR user:1000:login_count
SETEX session:abc123 3600 "user_data"  // 带过期时间
```

2. **Hash（哈希表）**：
```
底层：ziplist（小数据）或 hashtable（大数据）
应用：存储对象

示例：
HSET user:1000 name "张三" age 25 city "成都"
HGET user:1000 name
HGETALL user:1000
```

3. **List（列表）**：
```
底层：quicklist（ziplist的双向链表）
应用：消息队列、最新列表

示例：
LPUSH latest:orders order:123 order:124
LRANGE latest:orders 0 9  // 获取最新10个订单
RPOP queue:tasks  // 队列
```

4. **Set（集合）**：
```
底层：intset（整数）或 hashtable
应用：标签、好友关系、去重

示例：
SADD user:1000:tags "游戏" "电影"
SISMEMBER user:1000:tags "游戏"
SINTER user:1000:tags user:2000:tags  // 交集
```

5. **Sorted Set（有序集合）**：
```
底层：ziplist（小数据）或 skiplist+hashtable（大数据）
应用：排行榜、延迟队列

示例：
ZADD leaderboard 1000 "player1" 950 "player2"
ZREVRANGE leaderboard 0 9 WITHSCORES  // Top 10
ZRANK leaderboard "player1"
```

**高级数据结构**：

1. **HyperLogLog**：
```
应用：基数统计（UV统计）
误差率：0.81%
优势：占用内存极小（12KB）

PFADD uv:2024-01-15 user1 user2 user3
PFCOUNT uv:2024-01-15
```

2. **Bitmap**：
```
应用：签到、用户在线状态
优势：节省内存

SETBIT user:1000:signin:2024-01 15 1  // 1月15日签到
BITCOUNT user:1000:signin:2024-01  // 统计签到天数
```

3. **Geo**：
```
应用：LBS（附近的人）

GEOADD locations 116.40 39.90 "beijing" 121.47 31.23 "shanghai"
GEORADIUS locations 116.40 39.90 100 km
```

**性能优化**：
```
1. 使用合适的数据结构
   - 小数据用ziplist（压缩列表），省内存
   - 大数据自动转换为hashtable

2. 设置过期时间
   - 避免内存泄漏
   - 自动清理无用数据

3. 避免大key
   - 单个key过大影响性能
   - 使用SCAN代替KEYS

4. 使用pipeline批量操作
   - 减少网络往返
   - 提升性能10倍+
```

**项目应用示例**：
```go
// 游戏交易平台排行榜
func UpdateLeaderboard(userID int64, score float64) error {
    key := "leaderboard:2024"
    _, err := redisClient.ZAdd(ctx, key, &redis.Z{
        Score:  score,
        Member: userID,
    }).Result()
    return err
}

func GetTopPlayers(limit int) ([]string, error) {
    key := "leaderboard:2024"
    return redisClient.ZRevRange(ctx, key, 0, int64(limit-1)).Result()
}

// 商品库存缓存（防止缓存击穿）
func GetProductStock(productID int64) (int, error) {
    key := fmt.Sprintf("product:%d:stock", productID)

    // 先从缓存获取
    stock, err := redisClient.Get(ctx, key).Int()
    if err == nil {
        return stock, nil
    }

    // 使用互斥锁防止缓存击穿
    lockKey := fmt.Sprintf("lock:product:%d", productID)
    lock, err := redisClient.SetNX(ctx, lockKey, 1, 5*time.Second).Result()
    if err != nil || !lock {
        time.Sleep(100 * time.Millisecond)
        return GetProductStock(productID) // 重试
    }
    defer redisClient.Del(ctx, lockKey)

    // 从数据库加载
    stock, err = loadStockFromDB(productID)
    if err != nil {
        return 0, err
    }

    // 写入缓存
    redisClient.Set(ctx, key, stock, 10*time.Minute)
    return stock, nil
}
```

---

#### 问题16：Redis分布式锁如何实现？有什么问题？

**答案：**

**基本实现（SETNX + 过期时间）**：
```go
// 错误实现（不具备原子性）
func Lock(key string) bool {
    success := redisClient.SetNX(ctx, key, 1, 0).Val()
    if success {
        redisClient.Expire(ctx, key, 10*time.Second) // 不原子！
    }
    return success
}

// 正确实现（原子操作）
func Lock(key string, value string, expiration time.Duration) bool {
    success, err := redisClient.SetNX(ctx, key, value, expiration).Result()
    return err == nil && success
}

func Unlock(key string, value string) bool {
    // 使用Lua脚本保证原子性
    script := `
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
    `
    result, err := redisClient.Eval(ctx, script, []string{key}, value).Result()
    return err == nil && result.(int64) == 1
}
```

**完整实现**：
```go
type RedisLock struct {
    client     *redis.Client
    key        string
    value      string
    expiration time.Duration
}

func NewRedisLock(client *redis.Client, key string, expiration time.Duration) *RedisLock {
    return &RedisLock{
        client:     client,
        key:        key,
        value:      generateUniqueID(), // UUID
        expiration: expiration,
    }
}

func (l *RedisLock) Lock() bool {
    success, err := l.client.SetNX(ctx, l.key, l.value, l.expiration).Result()
    return err == nil && success
}

func (l *RedisLock) Unlock() bool {
    script := `
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
    `
    result, err := l.client.Eval(ctx, script, []string{l.key}, l.value).Result()
    return err == nil && result.(int64) == 1
}

// 自动续期（看门狗）
func (l *RedisLock) StartWatchdog(stopCh <-chan struct{}) {
    ticker := time.NewTicker(l.expiration / 3)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            l.client.Expire(ctx, l.key, l.expiration)
        case <-stopCh:
            return
        }
    }
}
```

**使用示例**：
```go
// 游戏交易平台防止重复下单
func CreateOrder(orderID int64) error {
    lockKey := fmt.Sprintf("lock:order:%d", orderID)
    lock := NewRedisLock(redisClient, lockKey, 10*time.Second)

    if !lock.Lock() {
        return errors.New("order is being processed")
    }
    defer lock.Unlock()

    // 业务逻辑
    return processOrder(orderID)
}
```

**分布式锁的问题**：

1. **锁超时问题**：
```
问题：业务执行时间 > 锁过期时间
后果：锁自动释放，其他线程获取锁，出现并发问题

解决方案：
1. 延长锁时间（不推荐，可能死锁）
2. 自动续期/看门狗（Redisson实现）
3. 使用Redlock算法（多Redis实例）
```

2. **Redis主从切换导致锁丢失**：
```
问题：
1. 客户端A在Master获取锁
2. Master宕机，锁未同步到Slave
3. Slave升级为Master
4. 客户端B在新Master获取同一把锁

解决方案：Redlock算法
- 在多个独立的Redis实例上获取锁
- 超过半数实例获取成功才算成功
- 保证不会因单点故障丢锁
```

3. **性能问题**：
```
问题：高并发下大量SETNX请求

解决方案：
1. 分段锁（如订单ID % 10）
2. 本地锁 + 分布式锁（双重检查）
3. 使用消息队列削峰
```

**Redlock算法实现**：
```go
func RedlockLock(clients []*redis.Client, key string, value string, expiration time.Duration) bool {
    successCount := 0
    startTime := time.Now()

    for _, client := range clients {
        success, err := client.SetNX(ctx, key, value, expiration).Result()
        if err == nil && success {
            successCount++
        }
    }

    // 超过半数成功，且耗时 < 锁有效期
    elapsed := time.Since(startTime)
    if successCount >= len(clients)/2+1 && elapsed < expiration {
        return true
    }

    // 失败，释放已获取的锁
    for _, client := range clients {
        client.Del(ctx, key)
    }
    return false
}
```

**项目应用**：
在游戏交易平台中，使用Redis分布式锁解决超卖问题，锁粒度精确到商品ID，性能和安全性平衡。

---

#### 问题16.1：分布式锁如何解决超卖问题？能解决重复下单吗？

**答案：**

**一、分布式锁如何解决超卖问题**

**超卖问题的本质**：
```
并发场景（库存只剩1件）：

时间线：
请求A：读取库存 stock = 1 有货
请求B：读取库存 stock = 1 有货  （同时读取）
请求A：扣减库存 stock = 0，创建订单成功
请求B：扣减库存 stock = -1，创建订单成功 超卖了！

结果：库存变成-1，但实际卖出了2件商品
```

**分布式锁的解决方案**：

通过加锁，保证**同一时刻只有一个请求能执行库存扣减操作**。

```go
func CreateOrder(productID int, userID int, quantity int) error {
    // 1. 获取分布式锁（锁的key是商品ID）
    lockKey := fmt.Sprintf("lock:product:%d", productID)
    lock := NewRedisLock(redisClient, lockKey, 10*time.Second)

    if !lock.Lock() {
        return errors.New("系统繁忙，请稍后重试")
    }
    defer lock.Unlock() // 确保释放锁

    // 2. 查询库存（此时只有一个请求能执行到这里）
    var stock int
    err := db.QueryRow("SELECT stock FROM products WHERE id = ?", productID).Scan(&stock)
    if err != nil {
        return err
    }

    // 3. 检查库存是否充足
    if stock < quantity {
        return errors.New("库存不足")
    }

    // 4. 扣减库存
    _, err = db.Exec("UPDATE products SET stock = stock - ? WHERE id = ?", quantity, productID)
    if err != nil {
        return err
    }

    // 5. 创建订单
    _, err = db.Exec(`
        INSERT INTO orders (user_id, product_id, quantity, created_at)
        VALUES (?, ?, ?, NOW())
    `, userID, productID, quantity)

    return err
}
```

**执行流程对比**：

```
无锁情况（会超卖）：
请求A：查库存=1 → 扣库存=0 → 创建订单
请求B：查库存=1 → 扣库存=-1 → 创建订单 超卖！

有锁情况（不会超卖）：
请求A：获取锁成功 → 查库存=1 → 扣库存=0 → 创建订单 → 释放锁
请求B：等待锁...
请求B：获取锁成功 → 查库存=0 → 返回"库存不足" → 释放锁 正确！
```

**关键点**：
- 锁的粒度：以商品ID为锁key，不同商品之间不会互相影响
- 锁的范围：查库存 + 扣库存 + 创建订单 必须在同一把锁内
- 锁的超时：设置合理的超时时间（10秒），防止死锁

---

**二、分布式锁能解决重复下单吗？**

**答案：能解决，但不是最优方案！**

**场景1：用户重复点击提交按钮**

**使用分布式锁（可行但不推荐）**：
```go
func CreateOrder(userID int, productID int) error {
    // 锁key = 用户ID + 商品ID
    lockKey := fmt.Sprintf("lock:order:%d:%d", userID, productID)
    lock := NewRedisLock(redisClient, lockKey, 5*time.Second)

    if !lock.Lock() {
        return errors.New("请勿重复下单，请稍后再试")
    }
    defer lock.Unlock()

    // 创建订单的业务逻辑...
    return createOrderLogic(userID, productID)
}
```

**问题**：
- 锁的过期时间不好设置（太短会误判，太长影响用户体验）
- 如果服务重启，Redis 锁丢失，仍可能重复
- 高并发下大量 SETNX 请求，性能开销大
- 用户如果真的想买2件，也会被阻止

**更好的方案1：幂等性设计 - 唯一订单号**

```go
func CreateOrder(userID int, productID int, orderNo string) error {
    // orderNo 由前端生成（UUID）或后端基于请求参数生成

    // 数据库表设计：
    // CREATE UNIQUE INDEX idx_order_no ON orders(order_no);

    _, err := db.Exec(`
        INSERT INTO orders (order_no, user_id, product_id, created_at)
        VALUES (?, ?, ?, NOW())
    `, orderNo, userID, productID)

    if err != nil {
        if isDuplicateKeyError(err) {
            // 订单号重复，说明已经创建过
            return errors.New("订单已存在，请勿重复提交")
        }
        return err
    }

    return nil
}

// 判断是否是唯一键冲突错误
func isDuplicateKeyError(err error) bool {
    if mysqlErr, ok := err.(*mysql.MySQLError); ok {
        return mysqlErr.Number == 1062 // Duplicate entry
    }
    return false
}
```

**优势**：
- 数据库层面保证唯一性，天然幂等
- 不依赖Redis，更可靠
- 即使服务重启也有效
- 无性能问题

**更好的方案2：防重表 + 唯一索引**

**requestID 的生成方式**：

**方案A：前端生成（推荐）** ✅

前端在用户点击提交按钮时生成唯一ID，重复点击使用同一个ID。

```javascript
// 前端代码（React/Vue示例）
function SubmitOrderButton() {
    const [requestID, setRequestID] = useState(null);
    const [loading, setLoading] = useState(false);

    const handleSubmit = async () => {
        // 1. 首次点击时生成 requestID
        if (!requestID) {
            const newRequestID = generateUUID(); // crypto.randomUUID() 或 uuid.v4()
            setRequestID(newRequestID);
        }

        setLoading(true);

        try {
            // 2. 发送请求（带上 requestID）
            await axios.post('/api/orders', {
                user_id: currentUser.id,
                product_id: selectedProduct.id,
                request_id: requestID  // ← 传给后端
            });

            alert('下单成功');
            setRequestID(null); // 成功后清空，允许下次下单
        } catch (error) {
            alert('下单失败: ' + error.message);
        } finally {
            setLoading(false);
        }
    };

    return <button onClick={handleSubmit} disabled={loading}>提交订单</button>;
}

// UUID 生成函数
function generateUUID() {
    // 方法1：现代浏览器支持
    return crypto.randomUUID(); // "550e8400-e29b-41d4-a716-446655440000"

    // 方法2：使用 uuid 库
    // import { v4 as uuidv4 } from 'uuid';
    // return uuidv4();

    // 方法3：时间戳 + 随机数（简单方案）
    // return Date.now() + '-' + Math.random().toString(36).substr(2, 9);
}
```

**优点**：
- ✅ 前端控制，重复点击自动使用同一ID
- ✅ 减少网络请求（后端直接判断重复）
- ✅ 用户体验好

**方案B：后端生成（备选）**

后端根据请求参数（用户ID + 商品ID + 时间窗口）生成固定ID。

```go
import (
    "crypto/md5"
    "fmt"
    "time"
)

func generateRequestID(userID, productID int) string {
    // 时间窗口：精确到分钟（1分钟内的请求视为重复）
    timeWindow := time.Now().Format("2006-01-02 15:04")

    // MD5(用户ID + 商品ID + 时间窗口)
    data := fmt.Sprintf("%d-%d-%s", userID, productID, timeWindow)
    hash := md5.Sum([]byte(data))
    return fmt.Sprintf("%x", hash)
}

// 使用示例
func CreateOrderHandler(w http.ResponseWriter, r *http.Request) {
    var req struct {
        UserID    int `json:"user_id"`
        ProductID int `json:"product_id"`
    }
    json.NewDecoder(r.Body).Decode(&req)

    // 后端生成 requestID
    requestID := generateRequestID(req.UserID, req.ProductID)

    err := CreateOrder(req.UserID, req.ProductID, requestID)
    // ...
}
```

**优点**：
- ✅ 前端无需改动，兼容老系统
- ✅ 同一用户对同一商品在时间窗口内只能下单一次

**缺点**：
- ❌ 时间窗口不好设置（太短防不了重复点击，太长影响用户体验）
- ❌ 用户真的想买2次会被阻止

**推荐方案：前端生成 + 后端兜底**

```go
func CreateOrder(userID int, productID int, requestID string) error {
    // 如果前端没传 requestID，后端生成一个
    if requestID == "" {
        requestID = generateRequestID(userID, productID)
    }

    // 后续逻辑...
}
```

---

**后端实现**：

```go
func CreateOrder(userID int, productID int, requestID string) error {
    // requestID 由前端生成（每次请求唯一）
    // 如：requestID = "550e8400-e29b-41d4-a716-446655440000"

    // 开启事务
    tx, _ := db.Begin()
    defer tx.Rollback()

    // 1. 插入防重表（requestID 唯一索引）
    _, err := tx.Exec(`
        INSERT INTO request_idempotent (request_id, created_at)
        VALUES (?, NOW())
    `, requestID)

    if err != nil {
        if isDuplicateKeyError(err) {
            return errors.New("请求已处理，请勿重复提交")
        }
        return err
    }

    // 2. 创建订单
    _, err = tx.Exec(`
        INSERT INTO orders (user_id, product_id, created_at)
        VALUES (?, ?, NOW())
    `, userID, productID)
    if err != nil {
        return err
    }

    // 3. 提交事务
    return tx.Commit()
}
```

**表结构**：
```sql
CREATE TABLE request_idempotent (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    request_id VARCHAR(64) NOT NULL UNIQUE,  -- 唯一请求ID
    created_at DATETIME NOT NULL,
    INDEX idx_created_at (created_at)  -- 方便清理过期数据
);
```

**更好的方案3：组合方案（分布式锁 + 幂等性）**

适用于高并发 + 防重的场景（如秒杀）：

```go
func CreateOrder(userID int, productID int, requestID string) error {
    // 1. 先检查幂等性（快速失败，避免无效竞争锁）
    cacheKey := fmt.Sprintf("order:request:%s", requestID)
    if exists := redisClient.Exists(ctx, cacheKey).Val(); exists > 0 {
        return errors.New("订单已创建")
    }

    // 2. 获取库存锁（防止超卖）
    lockKey := fmt.Sprintf("lock:product:%d", productID)
    lock := NewRedisLock(redisClient, lockKey, 10*time.Second)

    if !lock.Lock() {
        return errors.New("系统繁忙，请稍后重试")
    }
    defer lock.Unlock()

    // 3. 再次检查幂等性（双重检查）
    if exists := redisClient.Exists(ctx, cacheKey).Val(); exists > 0 {
        return errors.New("订单已创建")
    }

    // 4. 开启事务：扣减库存 + 创建订单
    tx, _ := db.Begin()
    defer tx.Rollback()

    // 检查库存（行锁）
    var stock int
    err := tx.QueryRow("SELECT stock FROM products WHERE id = ? FOR UPDATE", productID).Scan(&stock)
    if err != nil {
        return err
    }

    if stock <= 0 {
        return errors.New("库存不足")
    }

    // 扣减库存
    _, err = tx.Exec("UPDATE products SET stock = stock - 1 WHERE id = ?", productID)
    if err != nil {
        return err
    }

    // 创建订单
    _, err = tx.Exec(`
        INSERT INTO orders (user_id, product_id, created_at)
        VALUES (?, ?, NOW())
    `, userID, productID)
    if err != nil {
        return err
    }

    // 提交事务
    if err := tx.Commit(); err != nil {
        return err
    }

    // 5. 记录请求ID（防重）
    redisClient.SetEx(ctx, cacheKey, "1", 5*time.Minute)

    return nil
}
```

---

**三、总结对比**

| 场景       | 分布式锁       | 幂等性设计    | 推荐方案                  |
|----------|------------|----------|-----------------------|
| **超卖问题** | 适用（商品ID加锁） | 需配合数据库行锁 | **分布式锁 OR Redis+Lua** |
| **重复下单** | 能解决但不推荐    | 适用（唯一索引） | **幂等性设计（唯一索引）**       |
| **秒杀场景** | 适用         | 适用       | **组合方案：锁+幂等**         |

**核心原则**：
- **超卖问题**：用分布式锁保护库存扣减的原子性
- **重复下单**：用幂等性设计（唯一索引 + requestID）
- **高并发场景**：组合使用（先幂等快速失败，再加锁保护核心逻辑）

**记忆口诀**：
- 超卖用锁（保护库存）
- 重复用幂等（保护请求）
- 秒杀两者组合（性能 + 安全）

---

#### 问题17：Redis缓存穿透、缓存击穿、缓存雪崩如何解决？

**答案：**

**1. 缓存穿透（查询不存在的数据）**：

**问题**：
```
用户查询 product_id = -1（不存在）
→ Redis没有
→ 查询MySQL也没有
→ 不缓存结果
→ 每次都查数据库，压垮数据库
```

**解决方案**：

方案1：缓存空对象
```go
func GetProduct(productID int64) (*Product, error) {
    key := fmt.Sprintf("product:%d", productID)

    // 从缓存获取
    data, err := redisClient.Get(ctx, key).Bytes()
    if err == nil {
        if string(data) == "NULL" {
            return nil, nil // 缓存的空对象
        }
        var product Product
        json.Unmarshal(data, &product)
        return &product, nil
    }

    // 从数据库查询
    product, err := queryProductFromDB(productID)
    if err != nil {
        return nil, err
    }

    if product == nil {
        // 缓存空对象，设置较短过期时间
        redisClient.Set(ctx, key, "NULL", 5*time.Minute)
        return nil, nil
    }

    // 缓存正常数据
    data, _ = json.Marshal(product)
    redisClient.Set(ctx, key, data, 30*time.Minute)
    return product, nil
}
```

方案2：布隆过滤器（Bloom Filter）
```go
// 初始化布隆过滤器
var bloomFilter *bloom.BloomFilter

func InitBloomFilter() {
    bloomFilter = bloom.NewWithEstimates(1000000, 0.01) // 100万数据，1%误判率

    // 将所有商品ID加入布隆过滤器
    products := getAllProductIDs()
    for _, id := range products {
        bloomFilter.AddString(fmt.Sprintf("%d", id))
    }
}

func GetProduct(productID int64) (*Product, error) {
    key := fmt.Sprintf("product:%d", productID)

    // 先检查布隆过滤器
    if !bloomFilter.TestString(fmt.Sprintf("%d", productID)) {
        return nil, nil // 一定不存在
    }

    // 可能存在，继续查询缓存和数据库
    return getProductFromCacheOrDB(productID)
}
```

**2. 缓存击穿（热点key过期）**：

**问题**：
```
热点商品的缓存过期
→ 大量并发请求同时查询数据库
→ 数据库压力激增
```

**解决方案**：

方案1：互斥锁
```go
func GetProduct(productID int64) (*Product, error) {
    key := fmt.Sprintf("product:%d", productID)

    // 从缓存获取
    data, err := redisClient.Get(ctx, key).Bytes()
    if err == nil {
        var product Product
        json.Unmarshal(data, &product)
        return &product, nil
    }

    // 使用分布式锁
    lockKey := fmt.Sprintf("lock:product:%d", productID)
    lock, err := redisClient.SetNX(ctx, lockKey, 1, 5*time.Second).Result()

    if lock {
        // 获取锁成功，查询数据库
        defer redisClient.Del(ctx, lockKey)

        product, err := queryProductFromDB(productID)
        if err != nil {
            return nil, err
        }

        // 写入缓存
        data, _ := json.Marshal(product)
        redisClient.Set(ctx, key, data, 30*time.Minute)
        return product, nil
    } else {
        // 获取锁失败，等待后重试
        time.Sleep(100 * time.Millisecond)
        return GetProduct(productID)
    }
}
```

方案2：热点数据永不过期
```go
// 逻辑过期而非真实过期
type CacheValue struct {
    Data       *Product  `json:"data"`
    ExpireTime time.Time `json:"expire_time"`
}

func GetProduct(productID int64) (*Product, error) {
    key := fmt.Sprintf("product:%d", productID)

    data, err := redisClient.Get(ctx, key).Bytes()
    if err != nil {
        return rebuildCache(productID)
    }

    var cache CacheValue
    json.Unmarshal(data, &cache)

    // 检查逻辑过期时间
    if time.Now().After(cache.ExpireTime) {
        // 异步重建缓存
        go rebuildCache(productID)
    }

    // 返回旧数据（虽然可能过期，但保证可用性）
    return cache.Data, nil
}
```

**3. 缓存雪崩（大量key同时过期）**：

**问题**：
```
大量缓存在同一时间过期
→ 大量请求打到数据库
→ 数据库崩溃
```

**解决方案**：

方案1：随机过期时间
```go
func SetCache(key string, value interface{}) {
    baseExpiration := 30 * time.Minute
    randomExpiration := time.Duration(rand.Intn(300)) * time.Second // 0-5分钟随机
    expiration := baseExpiration + randomExpiration

    data, _ := json.Marshal(value)
    redisClient.Set(ctx, key, data, expiration)
}
```

方案2：多级缓存
```go
// L1: 本地缓存（进程内）
var localCache = cache.New(5*time.Minute, 10*time.Minute)

// L2: Redis缓存

// L3: 数据库

func GetProduct(productID int64) (*Product, error) {
    key := fmt.Sprintf("product:%d", productID)

    // L1: 本地缓存
    if data, found := localCache.Get(key); found {
        return data.(*Product), nil
    }

    // L2: Redis缓存
    data, err := redisClient.Get(ctx, key).Bytes()
    if err == nil {
        var product Product
        json.Unmarshal(data, &product)
        localCache.Set(key, &product, cache.DefaultExpiration)
        return &product, nil
    }

    // L3: 数据库
    product, err := queryProductFromDB(productID)
    if err != nil {
        return nil, err
    }

    // 写回缓存
    data, _ = json.Marshal(product)
    redisClient.Set(ctx, key, data, 30*time.Minute)
    localCache.Set(key, product, cache.DefaultExpiration)
    return product, nil
}
```

方案3：Redis高可用（哨兵/集群）
```go
// 使用Redis Sentinel或Cluster
// 即使部分节点宕机，服务仍可用
import "github.com/go-redis/redis/v8"

func InitRedisSentinel() *redis.Client {
    return redis.NewFailoverClient(&redis.FailoverOptions{
        MasterName:    "mymaster",
        SentinelAddrs: []string{"sentinel1:26379", "sentinel2:26379", "sentinel3:26379"},
    })
}
```

**项目应用总结**：
```go
// 游戏交易平台商品缓存综合方案
type ProductCache struct {
    redis       *redis.Client
    localCache  *cache.Cache
    bloomFilter *bloom.BloomFilter
}

func (c *ProductCache) GetProduct(productID int64) (*Product, error) {
    // 1. 布隆过滤器（防穿透）
    if !c.bloomFilter.TestString(fmt.Sprintf("%d", productID)) {
        return nil, nil
    }

    // 2. 本地缓存（防雪崩）
    key := fmt.Sprintf("product:%d", productID)
    if data, found := c.localCache.Get(key); found {
        return data.(*Product), nil
    }

    // 3. Redis缓存
    data, err := c.redis.Get(ctx, key).Bytes()
    if err == nil {
        if string(data) == "NULL" {
            return nil, nil // 空对象（防穿透）
        }
        var product Product
        json.Unmarshal(data, &product)
        c.localCache.Set(key, &product, cache.DefaultExpiration)
        return &product, nil
    }

    // 4. 分布式锁（防击穿）
    lockKey := fmt.Sprintf("lock:product:%d", productID)
    lock, _ := c.redis.SetNX(ctx, lockKey, 1, 5*time.Second).Result()
    if !lock {
        time.Sleep(100 * time.Millisecond)
        return c.GetProduct(productID)
    }
    defer c.redis.Del(ctx, lockKey)

    // 5. 查询数据库
    product, err := queryProductFromDB(productID)
    if err != nil {
        return nil, err
    }

    if product == nil {
        c.redis.Set(ctx, key, "NULL", 5*time.Minute)
        return nil, nil
    }

    // 6. 写入缓存（随机过期时间）
    expiration := 30*time.Minute + time.Duration(rand.Intn(300))*time.Second
    data, _ = json.Marshal(product)
    c.redis.Set(ctx, key, data, expiration)
    c.localCache.Set(key, product, cache.DefaultExpiration)

    return product, nil
}
```

---

## 第三部分：微服务架构与消息队列

### 3.1 微服务架构设计

#### 问题18：什么是微服务架构？与单体架构有什么区别？

**答案：**

**单体架构（Monolithic）**：
```
所有功能模块打包在一个应用中
- 优点：开发简单、部署简单、测试简单
- 缺点：扩展困难、维护困难、技术栈固定、单点故障

示例：
[用户模块 + 订单模块 + 支付模块 + 库存模块] → 一个WAR/JAR包
```

**微服务架构（Microservices）**：
```
将应用拆分为多个独立的小服务
- 优点：独立部署、技术栈灵活、易扩展、容错性好
- 缺点：系统复杂、分布式事务、网络延迟、运维成本高

示例：
用户服务 → 独立部署
订单服务 → 独立部署
支付服务 → 独立部署
库存服务 → 独立部署
```

**微服务拆分原则**：

1. **按业务能力拆分**：
```
- 用户服务：注册、登录、个人信息管理
- 订单服务：订单创建、查询、取消
- 支付服务：支付、退款
- 商品服务：商品管理、库存管理
```

2. **单一职责原则**：
```
每个服务只负责一个业务领域
- 服务边界清晰
- 高内聚、低耦合
```

3. **数据独立性**：
```
每个服务有自己的数据库
- 避免数据库成为瓶颈
- 服务间通过API通信
```

**微服务通信方式**：

1. **同步通信**：
```
HTTP/RESTful API：
- 简单、易理解
- 适合对外API

gRPC：
- 性能高（Protobuf序列化）
- 支持流式传输
- 适合内部服务通信
```

2. **异步通信**：
```
消息队列（Kafka、RabbitMQ）：
- 解耦服务
- 削峰填谷
- 最终一致性
```

**项目应用示例**：
```go
// 游戏交易平台微服务架构
服务拆分：
1. user-service：用户注册、登录、认证
2. product-service：商品管理、库存管理
3. order-service：订单创建、查询、状态流转
4. payment-service：支付、退款
5. notification-service：邮件、短信通知

// 订单服务调用其他服务
type OrderService struct {
    productClient  *ProductClient  // gRPC客户端
    paymentClient  *PaymentClient
    kafkaProducer  *kafka.Producer
}

func (s *OrderService) CreateOrder(ctx context.Context, req *CreateOrderRequest) error {
    // 1. 检查库存（同步调用商品服务）
    stock, err := s.productClient.GetStock(ctx, req.ProductID)
    if err != nil {
        return err
    }
    if stock < req.Quantity {
        return errors.New("insufficient stock")
    }

    // 2. 创建订单
    order := &Order{
        ID:        generateOrderID(),
        UserID:    req.UserID,
        ProductID: req.ProductID,
        Quantity:  req.Quantity,
        Amount:    req.Amount,
        Status:    StatusPending,
    }
    if err := s.saveOrder(order); err != nil {
        return err
    }

    // 3. 发送消息到Kafka（异步处理）
    msg := &OrderCreatedEvent{
        OrderID:   order.ID,
        ProductID: order.ProductID,
        Quantity:  order.Quantity,
    }
    s.kafkaProducer.Send(ctx, "order.created", msg)

    return nil
}
```

---

#### 问题19：微服务如何实现服务发现和负载均衡？

**答案：**

**服务发现（Service Discovery）**：

1. **客户端发现模式**：
```
流程：
1. 服务启动时，向注册中心注册（IP、端口、健康检查地址）
2. 客户端从注册中心获取服务列表
3. 客户端根据负载均衡策略选择一个实例
4. 客户端直接调用服务实例

代表：Consul、Eureka、Etcd
```

2. **服务端发现模式**：
```
流程：
1. 服务启动时，向注册中心注册
2. 客户端请求发送到负载均衡器（如Nginx、Envoy）
3. 负载均衡器查询注册中心
4. 负载均衡器转发请求到服务实例

代表：Kubernetes Service、AWS ELB
```

**实现示例（使用Consul）**：
```go
// 服务注册
import "github.com/hashicorp/consul/api"

func RegisterService(serviceName, serviceID, address string, port int) error {
    config := api.DefaultConfig()
    config.Address = "consul:8500"
    client, err := api.NewClient(config)
    if err != nil {
        return err
    }

    registration := &api.AgentServiceRegistration{
        ID:      serviceID,
        Name:    serviceName,
        Address: address,
        Port:    port,
        Check: &api.AgentServiceCheck{
            HTTP:     fmt.Sprintf("http://%s:%d/health", address, port),
            Interval: "10s",
            Timeout:  "3s",
        },
    }

    return client.Agent().ServiceRegister(registration)
}

// 服务发现
func DiscoverService(serviceName string) ([]string, error) {
    config := api.DefaultConfig()
    config.Address = "consul:8500"
    client, err := api.NewClient(config)
    if err != nil {
        return nil, err
    }

    services, _, err := client.Health().Service(serviceName, "", true, nil)
    if err != nil {
        return nil, err
    }

    var addresses []string
    for _, service := range services {
        addr := fmt.Sprintf("%s:%d", service.Service.Address, service.Service.Port)
        addresses = append(addresses, addr)
    }
    return addresses, nil
}

// 客户端负载均衡
type LoadBalancer struct {
    serviceName string
    instances   []string
    index       int
    mu          sync.Mutex
}

// 轮询算法
func (lb *LoadBalancer) Next() string {
    lb.mu.Lock()
    defer lb.mu.Unlock()

    if len(lb.instances) == 0 {
        // 重新发现服务
        lb.instances, _ = DiscoverService(lb.serviceName)
    }

    instance := lb.instances[lb.index]
    lb.index = (lb.index + 1) % len(lb.instances)
    return instance
}

// 随机算法
func (lb *LoadBalancer) Random() string {
    lb.mu.Lock()
    defer lb.mu.Unlock()

    if len(lb.instances) == 0 {
        lb.instances, _ = DiscoverService(lb.serviceName)
    }

    index := rand.Intn(len(lb.instances))
    return lb.instances[index]
}

// 一致性哈希（适合有状态服务）
type ConsistentHash struct {
    circle map[uint32]string
    sorted []uint32
}

func (ch *ConsistentHash) Add(node string) {
    for i := 0; i < 100; i++ { // 虚拟节点
        hash := crc32.ChecksumIEEE([]byte(fmt.Sprintf("%s:%d", node, i)))
        ch.circle[hash] = node
        ch.sorted = append(ch.sorted, hash)
    }
    sort.Slice(ch.sorted, func(i, j int) bool {
        return ch.sorted[i] < ch.sorted[j]
    })
}

func (ch *ConsistentHash) Get(key string) string {
    hash := crc32.ChecksumIEEE([]byte(key))
    idx := sort.Search(len(ch.sorted), func(i int) bool {
        return ch.sorted[i] >= hash
    })
    if idx == len(ch.sorted) {
        idx = 0
    }
    return ch.circle[ch.sorted[idx]]
}
```

**项目应用**：
在游戏交易平台中，使用Consul作为服务注册中心，订单服务、支付服务、商品服务都注册到Consul，客户端使用轮询负载均衡调用。

---

### 3.2 Kafka消息队列

#### 问题20：Kafka的架构是怎样的？核心概念有哪些？

**答案：**

**Kafka核心概念**：

1. **Producer（生产者）**：
   - 发送消息到Topic
   - 可指定分区或由Kafka自动分配

2. **Consumer（消费者）**：
   - 从Topic订阅并消费消息
   - 属于Consumer Group

3. **Topic（主题）**：
   - 消息的分类
   - 类似数据库的表

4. **Partition（分区）**：
   - Topic的物理分片
   - 每个分区是有序的消息队列
   - 支持并行消费

5. **Broker**：
   - Kafka服务器节点
   - 存储消息

6. **Consumer Group**：
   - 消费者组
   - 同一组内的消费者协同消费，每条消息只被组内一个消费者消费
   - 不同组之间独立消费

7. **Offset**：
   - 消息在分区中的位置
   - 消费者通过offset记录消费进度

**Kafka架构**：
```
Producer1 ─┐
Producer2 ─┼──→ Topic A (Partition 0, 1, 2) ──→ Consumer Group 1 (Consumer 1, 2)
Producer3 ─┘                                  └──→ Consumer Group 2 (Consumer 3, 4, 5)

存储：
- Partition 0 → Broker 1 (Leader), Broker 2 (Replica)
- Partition 1 → Broker 2 (Leader), Broker 3 (Replica)
- Partition 2 → Broker 3 (Leader), Broker 1 (Replica)
```

**Kafka特性**：

1. **高吞吐量**：
   - 顺序写磁盘
   - 零拷贝技术
   - 批量发送和压缩

2. **持久化**：
   - 消息持久化到磁盘
   - 支持副本机制

3. **分布式**：
   - 水平扩展
   - 分区并行处理

4. **消息顺序性**：
   - 单个分区内消息有序
   - 全局无序（多分区）

**使用示例**：
```go
import "github.com/segmentio/kafka-go"

// 生产者
type KafkaProducer struct {
    writer *kafka.Writer
}

func NewKafkaProducer(brokers []string, topic string) *KafkaProducer {
    writer := &kafka.Writer{
        Addr:     kafka.TCP(brokers...),
        Topic:    topic,
        Balancer: &kafka.LeastBytes{}, // 负载均衡策略
    }
    return &KafkaProducer{writer: writer}
}

func (p *KafkaProducer) Send(ctx context.Context, key, value []byte) error {
    msg := kafka.Message{
        Key:   key,
        Value: value,
    }
    return p.writer.WriteMessages(ctx, msg)
}

// 消费者
type KafkaConsumer struct {
    reader *kafka.Reader
}

func NewKafkaConsumer(brokers []string, topic, groupID string) *KafkaConsumer {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers:  brokers,
        Topic:    topic,
        GroupID:  groupID,
        MinBytes: 10e3, // 10KB
        MaxBytes: 10e6, // 10MB
    })
    return &KafkaConsumer{reader: reader}
}

func (c *KafkaConsumer) Consume(ctx context.Context, handler func([]byte) error) error {
    for {
        msg, err := c.reader.ReadMessage(ctx)
        if err != nil {
            return err
        }

        if err := handler(msg.Value); err != nil {
            log.Printf("handle message failed: %v", err)
            // 根据业务决定是否重试或跳过
        }
    }
}
```

**项目应用示例**：
```go
// 游戏交易平台订单事件处理
// 生产者：订单服务
func (s *OrderService) CreateOrder(ctx context.Context, order *Order) error {
    // 创建订单
    if err := s.db.Create(order).Error; err != nil {
        return err
    }

    // 发送订单创建事件
    event := &OrderCreatedEvent{
        OrderID:   order.ID,
        UserID:    order.UserID,
        ProductID: order.ProductID,
        Quantity:  order.Quantity,
        Amount:    order.Amount,
    }
    data, _ := json.Marshal(event)
    return s.kafkaProducer.Send(ctx, []byte(fmt.Sprintf("%d", order.ID)), data)
}

// 消费者：库存服务
func ConsumeOrderCreated(ctx context.Context) {
    consumer := NewKafkaConsumer(
        []string{"kafka1:9092", "kafka2:9092"},
        "order.created",
        "inventory-service",
    )

    consumer.Consume(ctx, func(data []byte) error {
        var event OrderCreatedEvent
        if err := json.Unmarshal(data, &event); err != nil {
            return err
        }

        // 扣减库存
        return updateStock(event.ProductID, -event.Quantity)
    })
}

// 消费者：通知服务
func ConsumeOrderCreatedForNotification(ctx context.Context) {
    consumer := NewKafkaConsumer(
        []string{"kafka1:9092", "kafka2:9092"},
        "order.created",
        "notification-service", // 不同的消费者组
    )

    consumer.Consume(ctx, func(data []byte) error {
        var event OrderCreatedEvent
        if err := json.Unmarshal(data, &event); err != nil {
            return err
        }

        // 发送通知
        return sendOrderNotification(event.UserID, event.OrderID)
    })
}
```

---

#### 问题21：Kafka如何保证消息不丢失？如何保证消息不重复？

**答案：**

**1. 保证消息不丢失**：

**生产者端**：
```go
// 配置ACK机制
writer := &kafka.Writer{
    Addr:  kafka.TCP("localhost:9092"),
    Topic: "orders",
    // RequiredAcks配置：
    // -1 或 all：等待所有副本确认（最安全，性能最差）
    // 1：等待Leader确认（默认，平衡）
    // 0：不等待确认（最快，可能丢失）
    RequiredAcks: kafka.RequireAll,
    // 异步发送失败后重试
    MaxAttempts: 3,
    // 批量发送超时
    BatchTimeout: 10 * time.Millisecond,
}

// 同步发送（等待确认）
err := writer.WriteMessages(ctx, kafka.Message{
    Key:   []byte("key"),
    Value: []byte("value"),
})
if err != nil {
    log.Printf("send failed: %v", err)
    // 重试或记录失败消息
}
```

**Broker端**：
```
配置副本：
- replication.factor >= 2：至少2个副本
- min.insync.replicas >= 2：至少2个副本同步成功才算写入成功

配置刷盘：
- log.flush.interval.messages=10000：每10000条消息刷盘
- log.flush.interval.ms=1000：每1秒刷盘
```

**消费者端**：
```go
// 手动提交offset
reader := kafka.NewReader(kafka.ReaderConfig{
    Brokers: []string{"localhost:9092"},
    Topic:   "orders",
    GroupID: "order-service",
    // 禁用自动提交
    CommitInterval: 0,
})

for {
    msg, err := reader.ReadMessage(ctx)
    if err != nil {
        break
    }

    // 处理消息
    if err := processMessage(msg.Value); err != nil {
        log.Printf("process failed: %v", err)
        continue // 不提交offset，下次重新消费
    }

    // 处理成功后手动提交offset
    if err := reader.CommitMessages(ctx, msg); err != nil {
        log.Printf("commit offset failed: %v", err)
    }
}
```

**2. 保证消息不重复（幂等性）**：

**问题场景**：
```
1. 生产者重试导致重复发送
2. 消费者处理成功但提交offset失败，重新消费
3. Rebalance导致重复消费
```

**解决方案**：

方案1：生产者幂等性（Kafka 0.11+）
```go
writer := &kafka.Writer{
    Addr:  kafka.TCP("localhost:9092"),
    Topic: "orders",
    // 开启幂等性（Kafka自动去重）
    Idempotent: true,
}
```

方案2：消费者业务幂等性
```go
// 使用唯一ID去重
type OrderService struct {
    db          *gorm.DB
    processedIDs *sync.Map // 或使用Redis
}

func (s *OrderService) ProcessOrder(msg *OrderMessage) error {
    // 检查是否已处理
    if _, exists := s.processedIDs.Load(msg.OrderID); exists {
        log.Printf("order %d already processed, skip", msg.OrderID)
        return nil
    }

    // 使用数据库唯一约束
    order := &Order{
        ID:     msg.OrderID, // 使用消息中的唯一ID
        UserID: msg.UserID,
        Amount: msg.Amount,
    }

    // 数据库设置唯一索引: CREATE UNIQUE INDEX idx_order_id ON orders(id)
    err := s.db.Create(order).Error
    if err != nil {
        if isDuplicateError(err) {
            log.Printf("order %d already exists", msg.OrderID)
            return nil // 幂等，返回成功
        }
        return err
    }

    // 记录已处理
    s.processedIDs.Store(msg.OrderID, true)
    return nil
}

// 方案3：事务消息（类似RocketMQ）
func (s *OrderService) CreateOrderWithTransaction() error {
    tx := s.db.Begin()

    // 1. 创建订单
    order := &Order{...}
    if err := tx.Create(order).Error; err != nil {
        tx.Rollback()
        return err
    }

    // 2. 创建消息发送记录（状态：待发送）
    msgRecord := &MessageRecord{
        ID:      generateID(),
        OrderID: order.ID,
        Status:  StatusPending,
        Payload: toJSON(order),
    }
    if err := tx.Create(msgRecord).Error; err != nil {
        tx.Rollback()
        return err
    }

    tx.Commit()

    // 3. 发送消息
    if err := s.kafkaProducer.Send(ctx, msgRecord.Payload); err != nil {
        // 发送失败，定时任务会重试
        log.Printf("send message failed: %v", err)
        return err
    }

    // 4. 更新消息状态（已发送）
    s.db.Model(&MessageRecord{}).Where("id = ?", msgRecord.ID).Update("status", StatusSent)
    return nil
}

// 定时任务：重试发送失败的消息
func RetryFailedMessages() {
    var messages []MessageRecord
    db.Where("status = ? AND created_at > ?", StatusPending, time.Now().Add(-24*time.Hour)).Find(&messages)

    for _, msg := range messages {
        if err := kafkaProducer.Send(ctx, msg.Payload); err != nil {
            log.Printf("retry send failed: %v", err)
        } else {
            db.Model(&msg).Update("status", StatusSent)
        }
    }
}
```

**项目应用总结**：
```go
// 游戏交易平台消息可靠性方案
type ReliableMessageService struct {
    db       *gorm.DB
    producer *KafkaProducer
}

// 发送可靠消息
func (s *ReliableMessageService) SendOrderEvent(order *Order) error {
    // 1. 开启事务
    tx := s.db.Begin()

    // 2. 创建订单
    if err := tx.Create(order).Error; err != nil {
        tx.Rollback()
        return err
    }

    // 3. 记录消息（幂等性）
    msgID := generateMessageID(order.ID)
    msg := &OutboxMessage{
        ID:      msgID,
        Topic:   "order.created",
        Key:     fmt.Sprintf("%d", order.ID),
        Payload: toJSON(order),
        Status:  StatusPending,
    }
    if err := tx.Create(msg).Error; err != nil {
        tx.Rollback()
        return err
    }

    // 4. 提交事务
    if err := tx.Commit().Error; err != nil {
        return err
    }

    // 5. 异步发送消息（失败不影响订单创建）
    go s.sendMessage(msg)

    return nil
}

func (s *ReliableMessageService) sendMessage(msg *OutboxMessage) {
    err := s.producer.Send(context.Background(), []byte(msg.Key), []byte(msg.Payload))
    if err != nil {
        log.Printf("send message failed: %v", err)
        // 定时任务会重试
        return
    }

    // 更新状态
    s.db.Model(&OutboxMessage{}).Where("id = ?", msg.ID).Update("status", StatusSent)
}

// 消费者幂等处理
func (s *OrderConsumer) HandleMessage(data []byte) error {
    var event OrderCreatedEvent
    json.Unmarshal(data, &event)

    // 使用分布式锁 + 数据库唯一索引保证幂等
    lockKey := fmt.Sprintf("lock:order:%d", event.OrderID)
    lock := NewRedisLock(redisClient, lockKey, 5*time.Second)

    if !lock.Lock() {
        return errors.New("lock failed")
    }
    defer lock.Unlock()

    // 检查是否已处理
    var count int64
    s.db.Model(&ProcessedMessage{}).Where("order_id = ?", event.OrderID).Count(&count)
    if count > 0 {
        return nil // 已处理，幂等
    }

    // 处理业务
    if err := s.processOrder(&event); err != nil {
        return err
    }

    // 记录已处理
    s.db.Create(&ProcessedMessage{OrderID: event.OrderID, ProcessedAt: time.Now()})
    return nil
}
```

---

#### 问题22：Kafka分区策略？如何保证消息顺序性？

**答案：**

**分区策略**：

1. **轮询（Round-Robin）**：
```go
writer := &kafka.Writer{
    Addr:     kafka.TCP("localhost:9092"),
    Topic:    "orders",
    Balancer: &kafka.RoundRobin{}, // 默认
}
// 消息均匀分布到各分区，无法保证顺序
```

2. **Key Hash**：
```go
writer := &kafka.Writer{
    Addr:     kafka.TCP("localhost:9092"),
    Topic:    "orders",
    Balancer: &kafka.Hash{}, // 根据Key的Hash分区
}

// 相同Key的消息发送到同一分区
writer.WriteMessages(ctx, kafka.Message{
    Key:   []byte(fmt.Sprintf("user:%d", userID)), // 同一用户的消息有序
    Value: []byte(data),
})
```

3. **自定义分区**：
```go
type CustomBalancer struct{}

func (b *CustomBalancer) Balance(msg kafka.Message, partitions ...int) int {
    // 根据业务规则选择分区
    // 例如：VIP用户发送到特定分区
    var userID int64
    json.Unmarshal(msg.Key, &userID)

    if isVIP(userID) {
        return 0 // VIP分区
    }
    return int(userID) % len(partitions) // 普通用户分区
}

writer := &kafka.Writer{
    Addr:     kafka.TCP("localhost:9092"),
    Topic:    "orders",
    Balancer: &CustomBalancer{},
}
```

**保证消息顺序性**：

**单分区顺序**：
```go
// 方案1：单分区（吞吐量受限）
writer := &kafka.Writer{
    Addr:     kafka.TCP("localhost:9092"),
    Topic:    "orders",
    Balancer: &kafka.Manual{Partition: 0}, // 固定分区0
}

// 方案2：按业务Key分区（推荐）
// 同一用户的订单保证顺序
writer.WriteMessages(ctx, kafka.Message{
    Key:   []byte(fmt.Sprintf("user:%d", userID)), // 相同userID发送到同一分区
    Value: orderData,
})

// 消费者：单线程消费每个分区
reader := kafka.NewReader(kafka.ReaderConfig{
    Brokers:  []string{"localhost:9092"},
    Topic:    "orders",
    GroupID:  "order-consumer",
    // 每个消费者处理多个分区，但单个分区只被一个消费者处理
})

for {
    msg, err := reader.ReadMessage(ctx)
    // 顺序处理
    processMessage(msg)
}
```

**全局顺序**：
```go
// 方案：单分区 + 单消费者（严重影响性能，不推荐）
topic配置：partitions=1
消费者配置：consumers=1

// 更好的方案：业务层保证
// 使用版本号或时间戳
type Order struct {
    ID        int64
    UserID    int64
    Status    string
    Version   int64     // 版本号
    UpdatedAt time.Time
}

func UpdateOrder(order *Order) error {
    // 乐观锁
    result := db.Model(&Order{}).
        Where("id = ? AND version = ?", order.ID, order.Version).
        Updates(map[string]interface{}{
            "status":     order.Status,
            "version":    order.Version + 1,
            "updated_at": time.Now(),
        })

    if result.RowsAffected == 0 {
        return errors.New("version conflict")
    }
    return nil
}
```

**项目应用**：
```go
// 游戏交易平台订单状态流转保证顺序
// 生产者：按订单ID分区
func (s *OrderService) UpdateOrderStatus(orderID int64, status string) error {
    event := &OrderStatusChangedEvent{
        OrderID:   orderID,
        Status:    status,
        Timestamp: time.Now().Unix(),
    }
    data, _ := json.Marshal(event)

    return s.kafkaProducer.writer.WriteMessages(ctx, kafka.Message{
        Key:   []byte(fmt.Sprintf("%d", orderID)), // 同一订单发送到同一分区
        Value: data,
    })
}

// 消费者：单线程顺序处理
func ConsumeOrderStatus(ctx context.Context) {
    reader := kafka.NewReader(kafka.ReaderConfig{
        Brokers:  []string{"kafka1:9092"},
        Topic:    "order.status.changed",
        GroupID:  "order-status-consumer",
        // 单分区单消费者保证顺序
    })

    for {
        msg, err := reader.ReadMessage(ctx)
        if err != nil {
            break
        }

        var event OrderStatusChangedEvent
        json.Unmarshal(msg.Value, &event)

        // 顺序处理状态变更
        if err := processStatusChange(&event); err != nil {
            log.Printf("process failed: %v", err)
            // 处理失败，不提交offset，重新消费
            continue
        }

        // 成功后提交offset
        reader.CommitMessages(ctx, msg)
    }
}
```

---

## 第四部分：云平台部署（AWS、阿里云）

### 4.1 AWS云服务

#### 问题23：介绍一下AWS的核心服务？在项目中如何使用？

**答案：**

**AWS核心服务**：

1. **EC2（Elastic Compute Cloud）**：
```
虚拟服务器
- 弹性伸缩：Auto Scaling
- 按需付费
- 多种实例类型（通用、计算优化、内存优化）

项目使用：
- 部署游戏交易平台后端服务
- 使用t3.medium实例（2核4G）
- 配置Auto Scaling：CPU > 70%时自动扩容
```

2. **RDS（Relational Database Service）**：
```
托管数据库服务
- 支持MySQL、PostgreSQL、Aurora等
- 自动备份、故障转移
- 主从复制

项目使用：
- MySQL 8.0，db.t3.large（2核8G）
- Multi-AZ部署（主从双机房）
- 自动备份保留7天
```

3. **S3（Simple Storage Service）**：
```
对象存储服务
- 无限存储空间
- 99.999999999%数据持久性
- 支持版本控制、生命周期管理

项目使用：
- 存储游戏道具图片、用户头像
- 配置CloudFront CDN加速
- 设置生命周期：30天后转储到Glacier
```

4. **ElastiCache（Redis/Memcached）**：
```
托管缓存服务
- 自动故障转移
- 集群模式

项目使用：
- Redis 7.0集群（3主3从）
- cache.t3.medium节点
- 用于会话缓存、商品缓存
```

5. **ELB（Elastic Load Balancer）**：
```
负载均衡器
- ALB：应用层（HTTP/HTTPS）
- NLB：网络层（TCP/UDP）
- CLB：经典负载均衡

项目使用：
- ALB分发HTTP请求到多个EC2实例
- 配置健康检查：/health endpoint
- SSL/TLS终止
```

6. **CloudFront（CDN）**：
```
内容分发网络
- 全球边缘节点
- 降低延迟

项目使用：
- 加速S3静态资源访问
- 缓存TTL设置为24小时
```

**项目架构示例**：
```
用户请求
    ↓
Route 53 (DNS)
    ↓
CloudFront (CDN)
    ↓
ALB (负载均衡)
    ↓
EC2 Auto Scaling Group (游戏交易平台后端)
    ├─→ RDS (MySQL主从)
    ├─→ ElastiCache (Redis集群)
    └─→ S3 (静态资源)
```

**Go代码使用AWS SDK**：
```go
import (
    "github.com/aws/aws-sdk-go/aws"
    "github.com/aws/aws-sdk-go/aws/session"
    "github.com/aws/aws-sdk-go/service/s3"
)

// 上传文件到S3
func UploadToS3(bucket, key string, data []byte) error {
    sess := session.Must(session.NewSession(&aws.Config{
        Region: aws.String("us-west-2"),
    }))

    svc := s3.New(sess)
    _, err := svc.PutObject(&s3.PutObjectInput{
        Bucket: aws.String(bucket),
        Key:    aws.String(key),
        Body:   bytes.NewReader(data),
        ACL:    aws.String("public-read"),
    })
    return err
}

// 从S3下载文件
func DownloadFromS3(bucket, key string) ([]byte, error) {
    sess := session.Must(session.NewSession(&aws.Config{
        Region: aws.String("us-west-2"),
    }))

    svc := s3.New(sess)
    result, err := svc.GetObject(&s3.GetObjectInput{
        Bucket: aws.String(bucket),
        Key:    aws.String(key),
    })
    if err != nil {
        return nil, err
    }
    defer result.Body.Close()

    return io.ReadAll(result.Body)
}

// 生成S3预签名URL（临时访问链接）
func GeneratePresignedURL(bucket, key string, expiration time.Duration) (string, error) {
    sess := session.Must(session.NewSession(&aws.Config{
        Region: aws.String("us-west-2"),
    }))

    svc := s3.New(sess)
    req, _ := svc.GetObjectRequest(&s3.GetObjectInput{
        Bucket: aws.String(bucket),
        Key:    aws.String(key),
    })

    return req.Presign(expiration)
}
```

**项目应用场景**：
```go
// 游戏交易平台：上传游戏道具图片
func (s *ProductService) UploadProductImage(productID int64, imageData []byte) (string, error) {
    // 1. 生成唯一文件名
    filename := fmt.Sprintf("products/%d/%s.jpg", productID, uuid.New().String())

    // 2. 上传到S3
    bucket := "game-platform-assets"
    if err := UploadToS3(bucket, filename, imageData); err != nil {
        return "", err
    }

    // 3. 返回CloudFront CDN URL
    cdnDomain := "https://d1234567890.cloudfront.net"
    imageURL := fmt.Sprintf("%s/%s", cdnDomain, filename)

    // 4. 保存URL到数据库
    s.db.Model(&Product{}).Where("id = ?", productID).Update("image_url", imageURL)

    return imageURL, nil
}
```

---

#### 问题24：如何在AWS上实现高可用架构？

**答案：**

**高可用架构设计原则**：

1. **多可用区（Multi-AZ）部署**：
```
Region: us-west-2
  ├─ AZ-1 (us-west-2a)
  │   ├─ EC2实例 x2
  │   ├─ RDS主库
  │   └─ ElastiCache节点
  ├─ AZ-2 (us-west-2b)
  │   ├─ EC2实例 x2
  │   ├─ RDS从库
  │   └─ ElastiCache节点
  └─ AZ-3 (us-west-2c)
      ├─ EC2实例 x2
      └─ ElastiCache节点

优势：
- 单个AZ故障不影响服务
- 自动故障转移
```

2. **Auto Scaling自动伸缩**：
```
配置：
- 最小实例数：2
- 最大实例数：10
- 目标CPU使用率：70%

扩容策略：
- CPU > 70%，持续5分钟 → 增加2个实例
- CPU < 30%，持续15分钟 → 减少1个实例

健康检查：
- ELB健康检查间隔：30秒
- 不健康阈值：2次失败
- 自动替换不健康实例
```

3. **数据库高可用**：
```
RDS Multi-AZ：
- 主库：us-west-2a
- 从库：us-west-2b（同步复制）
- 自动故障转移：DNS切换，无需应用改代码
- 故障转移时间：1-2分钟

读写分离：
- 写操作 → 主库
- 读操作 → 读副本（Read Replica）
- 读副本可跨Region部署
```

4. **缓存高可用**：
```
ElastiCache Redis集群：
- 3个分片（Shard）
- 每个分片：1主2从
- 自动故障转移
- 客户端自动重连
```

**Terraform配置示例**：
```hcl
# Auto Scaling组
resource "aws_autoscaling_group" "app" {
  name                = "game-platform-asg"
  min_size            = 2
  max_size            = 10
  desired_capacity    = 4
  health_check_type   = "ELB"
  health_check_grace_period = 300
  vpc_zone_identifier = [
    aws_subnet.private_a.id,
    aws_subnet.private_b.id,
    aws_subnet.private_c.id,
  ]

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "game-platform-instance"
    propagate_at_launch = true
  }
}

# Auto Scaling策略
resource "aws_autoscaling_policy" "scale_up" {
  name                   = "scale-up"
  scaling_adjustment     = 2
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300
  autoscaling_group_name = aws_autoscaling_group.app.name
}

resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = "300"
  statistic           = "Average"
  threshold           = "70"
  alarm_actions       = [aws_autoscaling_policy.scale_up.arn]
}

# RDS Multi-AZ
resource "aws_db_instance" "main" {
  identifier              = "game-platform-db"
  engine                  = "mysql"
  engine_version          = "8.0"
  instance_class          = "db.t3.large"
  allocated_storage       = 100
  multi_az                = true  # 开启Multi-AZ
  db_subnet_group_name    = aws_db_subnet_group.main.name
  vpc_security_group_ids  = [aws_security_group.rds.id]
  backup_retention_period = 7
  skip_final_snapshot     = false
}

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "game-platform-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = [
    aws_subnet.public_a.id,
    aws_subnet.public_b.id,
    aws_subnet.public_c.id,
  ]
}

resource "aws_lb_target_group" "app" {
  name     = "app-tg"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id

  health_check {
    path                = "/health"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}
```

**Go应用配置读写分离**：
```go
import "gorm.io/gorm"

type DBCluster struct {
    master  *gorm.DB
    replicas []*gorm.DB
    index    int
    mu       sync.Mutex
}

func NewDBCluster(masterDSN string, replicaDSNs []string) (*DBCluster, error) {
    // 主库连接
    master, err := gorm.Open(mysql.Open(masterDSN), &gorm.Config{})
    if err != nil {
        return nil, err
    }

    // 从库连接
    var replicas []*gorm.DB
    for _, dsn := range replicaDSNs {
        replica, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
        if err != nil {
            return nil, err
        }
        replicas = append(replicas, replica)
    }

    return &DBCluster{
        master:   master,
        replicas: replicas,
    }, nil
}

// 写操作用主库
func (c *DBCluster) Master() *gorm.DB {
    return c.master
}

// 读操作用从库（轮询）
func (c *DBCluster) Replica() *gorm.DB {
    c.mu.Lock()
    defer c.mu.Unlock()

    if len(c.replicas) == 0 {
        return c.master // 没有从库，fallback到主库
    }

    replica := c.replicas[c.index]
    c.index = (c.index + 1) % len(c.replicas)
    return replica
}

// 使用示例
var dbCluster *DBCluster

func init() {
    dbCluster, _ = NewDBCluster(
        "user:pass@tcp(master.rds.amazonaws.com:3306)/gamedb",
        []string{
            "user:pass@tcp(replica1.rds.amazonaws.com:3306)/gamedb",
            "user:pass@tcp(replica2.rds.amazonaws.com:3306)/gamedb",
        },
    )
}

// 写操作
func CreateOrder(order *Order) error {
    return dbCluster.Master().Create(order).Error
}

// 读操作
func GetOrder(orderID int64) (*Order, error) {
    var order Order
    err := dbCluster.Replica().Where("id = ?", orderID).First(&order).Error
    return &order, err
}
```

**项目应用**：
游戏交易平台部署在AWS us-west-2区域，3个可用区各部署2台EC2，RDS开启Multi-AZ，ElastiCache使用集群模式，实现99.95%可用性。

---

### 4.2 阿里云服务

#### 问题25：阿里云核心服务有哪些？与AWS的对比？

**答案：**

**阿里云核心服务**：

1. **ECS（Elastic Compute Service）**：
```
对标AWS EC2
- 弹性伸缩：Auto Scaling
- 实例规格：ecs.c6.large（2核4G）

项目使用：
- 跨境电商后台部署在ECS
- 使用按量付费模式
- 配置弹性伸缩规则
```

2. **RDS（云数据库）**：
```
对标AWS RDS
- 支持MySQL、PostgreSQL、SQL Server等
- 主备双机热备份
- 自动备份

项目使用：
- MySQL 8.0高可用版
- 主备实例（同城双机房）
- 每日自动备份
```

3. **OSS（Object Storage Service）**：
```
对标AWS S3
- 对象存储
- 支持CDN加速

项目使用：
- 存储商品图片
- 配置CDN加速域名
- 设置跨域访问规则
```

4. **Redis（云数据库Redis版）**：
```
对标AWS ElastiCache
- 集群版、读写分离版
- 自动故障切换

项目使用：
- Redis 6.0集群版
- 4分片，每分片1主1从
- 用于商品缓存、会话存储
```

5. **SLB（Server Load Balancer）**：
```
对标AWS ELB
- 7层（HTTP/HTTPS）
- 4层（TCP/UDP）

项目使用：
- 7层SLB，HTTPS卸载
- 后端服务器健康检查
- 会话保持
```

6. **CDN**：
```
对标AWS CloudFront
- 全球加速节点
- 静态资源加速

项目使用：
- 加速OSS图片访问
- 配置回源规则
```

**阿里云 vs AWS 对比**：

| 服务   | 阿里云   | AWS         | 备注           |
|------|-------|-------------|--------------|
| 计算   | ECS   | EC2         | 功能相似         |
| 数据库  | RDS   | RDS         | AWS选择更多      |
| 存储   | OSS   | S3          | 功能相似         |
| 缓存   | Redis | ElastiCache | 阿里云Redis功能更强 |
| 负载均衡 | SLB   | ELB/ALB     | 功能相似         |
| CDN  | CDN   | CloudFront  | 阿里云国内节点多     |
| 价格   | 相对便宜  | 相对贵         | 国内用阿里云更便宜    |
| 国内访问 | 快     | 慢           | 阿里云国内体验好     |

**Go代码使用阿里云SDK**：
```go
import (
    "github.com/aliyun/aliyun-oss-go-sdk/oss"
)

// 上传文件到OSS
func UploadToOSS(bucket, key string, data []byte) error {
    client, err := oss.New(
        "oss-cn-hangzhou.aliyuncs.com",
        "AccessKeyID",
        "AccessKeySecret",
    )
    if err != nil {
        return err
    }

    bucketObj, err := client.Bucket(bucket)
    if err != nil {
        return err
    }

    return bucketObj.PutObject(key, bytes.NewReader(data))
}

// 生成签名URL
func GenerateOSSSignedURL(bucket, key string, expiration int64) (string, error) {
    client, _ := oss.New(
        "oss-cn-hangzhou.aliyuncs.com",
        "AccessKeyID",
        "AccessKeySecret",
    )

    bucketObj, _ := client.Bucket(bucket)
    return bucketObj.SignURL(key, oss.HTTPGet, expiration)
}
```

**项目应用场景**：
```go
// 跨境电商平台：商品图片上传到OSS
func (s *ProductService) UploadProductImage(productID int64, image []byte) (string, error) {
    // 1. 生成文件名
    filename := fmt.Sprintf("products/%d/%s.jpg", productID, uuid.New().String())

    // 2. 上传到OSS
    bucket := "ecommerce-assets"
    if err := UploadToOSS(bucket, filename, image); err != nil {
        return "", err
    }

    // 3. 返回CDN URL
    cdnDomain := "https://cdn.example.com"
    imageURL := fmt.Sprintf("%s/%s", cdnDomain, filename)

    // 4. 保存到数据库
    s.db.Model(&Product{}).Where("id = ?", productID).Update("image_url", imageURL)

    return imageURL, nil
}

// 生成临时下载链接（防盗链）
func (s *ProductService) GenerateDownloadURL(productID int64) (string, error) {
    var product Product
    if err := s.db.First(&product, productID).Error; err != nil {
        return "", err
    }

    // 从URL提取OSS key
    key := extractKeyFromURL(product.ImageURL)

    // 生成1小时有效的签名URL
    return GenerateOSSSignedURL("ecommerce-assets", key, 3600)
}
```

---

## 第五部分：性能优化与监控

### 5.1 性能优化

#### 问题26：Go程序如何进行性能分析和优化？

**答案：**

**性能分析工具**：

1. **pprof（性能分析）**：
```go
import (
    "net/http"
    _ "net/http/pprof"
)

func main() {
    // 启动pprof HTTP服务
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()

    // 应用代码...
}

// 访问：
// http://localhost:6060/debug/pprof/
// - /debug/pprof/profile：CPU分析（30秒）
// - /debug/pprof/heap：内存分析
// - /debug/pprof/goroutine：Goroutine分析
// - /debug/pprof/block：阻塞分析
// - /debug/pprof/mutex：锁竞争分析
```

**命令行分析**：
```bash
# CPU分析
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# 进入交互模式后：
(pprof) top10        # 查看top 10热点函数
(pprof) list funcName # 查看函数源码
(pprof) web          # 生成调用图（需要graphviz）
(pprof) pdf > cpu.pdf # 导出PDF

# 内存分析
go tool pprof http://localhost:6060/debug/pprof/heap

# 对比两次heap快照（找内存泄漏）
curl http://localhost:6060/debug/pprof/heap > heap1.prof
# 等待一段时间
curl http://localhost:6060/debug/pprof/heap > heap2.prof
go tool pprof -base heap1.prof heap2.prof
```

**2. benchmark（基准测试）**：
```go
// product_test.go
func BenchmarkGetProduct(b *testing.B) {
    productID := int64(12345)

    b.ResetTimer() // 重置计时器
    for i := 0; i < b.N; i++ {
        GetProduct(productID)
    }
}

func BenchmarkGetProductParallel(b *testing.B) {
    productID := int64(12345)

    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            GetProduct(productID)
        }
    })
}

// 运行：
// go test -bench=. -benchmem -cpuprofile=cpu.prof -memprofile=mem.prof
//
// 输出：
// BenchmarkGetProduct-8         1000000    1234 ns/op    512 B/op    10 allocs/op
//                               次数       耗时          分配内存   分配次数
```

**常见性能优化技巧**：

**1. 减少内存分配**：
```go
// 不好：频繁分配
func ProcessOrders(orders []Order) []Result {
    var results []Result
    for _, order := range orders {
        result := processOrder(order) // 每次都分配
        results = append(results, result)
    }
    return results
}

// 好：预分配容量
func ProcessOrders(orders []Order) []Result {
    results := make([]Result, 0, len(orders)) // 预分配
    for _, order := range orders {
        result := processOrder(order)
        results = append(results, result)
    }
    return results
}

// 不好：字符串拼接
func BuildSQL(ids []int64) string {
    sql := "SELECT * FROM orders WHERE id IN ("
    for i, id := range ids {
        if i > 0 {
            sql += ","
        }
        sql += fmt.Sprintf("%d", id) // 每次拼接都分配新字符串
    }
    sql += ")"
    return sql
}

// 好：使用strings.Builder
func BuildSQL(ids []int64) string {
    var builder strings.Builder
    builder.WriteString("SELECT * FROM orders WHERE id IN (")
    for i, id := range ids {
        if i > 0 {
            builder.WriteString(",")
        }
        builder.WriteString(strconv.FormatInt(id, 10))
    }
    builder.WriteString(")")
    return builder.String()
}
```

**2. 使用对象池**：
```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func ProcessRequest(data []byte) []byte {
    buf := bufferPool.Get().(*bytes.Buffer)
    buf.Reset() // 重置
    defer bufferPool.Put(buf) // 归还

    // 使用buffer
    buf.Write(data)
    return buf.Bytes()
}
```

**3. 并发控制**：
```go
// 不好：无限制并发
func ProcessOrders(orders []Order) {
    for _, order := range orders {
        go processOrder(order) // 可能创建百万个Goroutine
    }
}

// 好：使用worker pool
func ProcessOrders(orders []Order) {
    workers := 100
    jobs := make(chan Order, len(orders))
    var wg sync.WaitGroup

    // 启动worker
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for order := range jobs {
                processOrder(order)
            }
        }()
    }

    // 发送任务
    for _, order := range orders {
        jobs <- order
    }
    close(jobs)

    wg.Wait()
}
```

**4. 批量操作**：
```go
// 不好：逐条插入
func SaveOrders(orders []Order) error {
    for _, order := range orders {
        if err := db.Create(&order).Error; err != nil {
            return err
        }
    }
    return nil
}

// 好：批量插入
func SaveOrders(orders []Order) error {
    batchSize := 1000
    for i := 0; i < len(orders); i += batchSize {
        end := i + batchSize
        if end > len(orders) {
            end = len(orders)
        }
        batch := orders[i:end]
        if err := db.CreateInBatches(batch, batchSize).Error; err != nil {
            return err
        }
    }
    return nil
}
```

**项目应用示例**：
```go
// 游戏交易平台性能优化实例
// 优化前：订单列表查询300ms
func GetUserOrdersSlow(userID int64) ([]Order, error) {
    var orders []Order
    err := db.Where("user_id = ?", userID).Find(&orders).Error
    if err != nil {
        return nil, err
    }

    // 逐个查询商品信息
    for i := range orders {
        var product Product
        db.First(&product, orders[i].ProductID)
        orders[i].Product = product
    }

    return orders, nil
}

// 优化后：50ms
func GetUserOrdersFast(userID int64) ([]Order, error) {
    // 1. 从Redis缓存获取
    cacheKey := fmt.Sprintf("user:orders:%d", userID)
    if cached, err := getFromRedis(cacheKey); err == nil {
        return cached, nil
    }

    // 2. 数据库查询（使用预加载）
    var orders []Order
    err := db.Preload("Product").Where("user_id = ?", userID).Find(&orders).Error
    if err != nil {
        return nil, err
    }

    // 3. 写入缓存
    setToRedis(cacheKey, orders, 5*time.Minute)

    return orders, nil
}
```

---

#### 问题27：如何使用Prometheus和Grafana监控Go应用？

**答案：**

**Prometheus监控架构**：
```
Go应用 → 暴露/metrics接口 → Prometheus定期拉取 → 存储时序数据 → Grafana可视化展示
```

**1. Go应用集成Prometheus**：
```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

// 定义指标
var (
    // Counter：只增不减的计数器
    httpRequestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )

    // Histogram：直方图，统计分布
    httpRequestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets, // [0.005, 0.01, 0.025, 0.05, ...]
        },
        []string{"method", "endpoint"},
    )

    // Gauge：可增可减的测量值
    activeConnections = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Number of active connections",
        },
    )

    // Summary：摘要，统计分位数
    orderProcessDuration = promauto.NewSummaryVec(
        prometheus.SummaryOpts{
            Name:       "order_process_duration_seconds",
            Help:       "Order processing duration",
            Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001}, // P50, P90, P99
        },
        []string{"status"},
    )

    // 业务指标
    orderCreatedTotal = promauto.NewCounter(
        prometheus.CounterOpts{
            Name: "order_created_total",
            Help: "Total number of orders created",
        },
    )
)

// HTTP中间件：记录请求指标
func PrometheusMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()

        // 包装ResponseWriter以捕获状态码
        wrapped := &responseWriter{ResponseWriter: w, statusCode: 200}

        next.ServeHTTP(wrapped, r)

        // 记录指标
        duration := time.Since(start).Seconds()
        httpRequestsTotal.WithLabelValues(
            r.Method,
            r.URL.Path,
            strconv.Itoa(wrapped.statusCode),
        ).Inc()

        httpRequestDuration.WithLabelValues(
            r.Method,
            r.URL.Path,
        ).Observe(duration)
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

func main() {
    // 暴露/metrics接口
    http.Handle("/metrics", promhttp.Handler())

    // 应用路由
    mux := http.NewServeMux()
    mux.HandleFunc("/orders", handleOrders)

    // 使用Prometheus中间件
    handler := PrometheusMiddleware(mux)

    http.ListenAndServe(":8080", handler)
}
```

**2. 业务指标采集**：
```go
// 订单服务指标采集
func CreateOrder(ctx context.Context, req *CreateOrderRequest) (*Order, error) {
    start := time.Now()

    // 业务逻辑
    order, err := createOrderInDB(req)

    // 记录处理时长
    duration := time.Since(start).Seconds()
    if err != nil {
        orderProcessDuration.WithLabelValues("failed").Observe(duration)
        return nil, err
    }

    orderProcessDuration.WithLabelValues("success").Observe(duration)
    orderCreatedTotal.Inc()

    return order, nil
}

// 监控数据库连接池
func MonitorDBPool(db *sql.DB) {
    go func() {
        ticker := time.NewTicker(10 * time.Second)
        for range ticker.C {
            stats := db.Stats()

            dbConnectionsOpen.Set(float64(stats.OpenConnections))
            dbConnectionsInUse.Set(float64(stats.InUse))
            dbConnectionsIdle.Set(float64(stats.Idle))
        }
    }()
}
```

**3. Prometheus配置**（prometheus.yml）：
```yaml
global:
  scrape_interval: 15s # 拉取间隔
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'game-platform'
    static_configs:
      - targets: ['localhost:8080', 'localhost:8081', 'localhost:8082']
        labels:
          env: 'production'
          service: 'game-platform'

  - job_name: 'order-service'
    static_configs:
      - targets: ['order-service:8080']

  - job_name: 'payment-service'
    static_configs:
      - targets: ['payment-service:8080']

# 告警规则
rule_files:
  - 'alert_rules.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

**4. 告警规则**（alert_rules.yml）：
```yaml
groups:
  - name: api_alerts
    rules:
      # API错误率超过5%
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          /
          sum(rate(http_requests_total[5m])) by (service)
          > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.service }}"
          description: "Error rate is {{ $value | humanizePercentage }}"

      # API响应时间P99 > 1秒
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint))
          > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency on {{ $labels.endpoint }}"
          description: "P99 latency is {{ $value }}s"

      # 订单创建失败率 > 1%
      - alert: OrderCreationFailure
        expr: |
          sum(rate(order_process_duration_seconds_count{status="failed"}[5m]))
          /
          sum(rate(order_process_duration_seconds_count[5m]))
          > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High order creation failure rate"

      # 数据库连接池耗尽
      - alert: DBConnectionPoolExhausted
        expr: db_connections_in_use / db_connections_open > 0.9
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Database connection pool nearly exhausted"
```

**5. Grafana Dashboard配置**：
```json
{
  "dashboard": {
    "title": "Game Platform Monitoring",
    "panels": [
      {
        "title": "QPS",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[1m])) by (service)",
            "legendFormat": "{{service}}"
          }
        ]
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status=~\"5..\"}[5m])) by (service) / sum(rate(http_requests_total[5m])) by (service)",
            "legendFormat": "{{service}}"
          }
        ]
      },
      {
        "title": "P99 Latency",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint))",
            "legendFormat": "{{endpoint}}"
          }
        ]
      },
      {
        "title": "Order Created",
        "targets": [
          {
            "expr": "rate(order_created_total[5m])",
            "legendFormat": "Orders/sec"
          }
        ]
      }
    ]
  }
}
```

**项目应用**：
```go
// 游戏交易平台监控实例
type MetricsCollector struct {
    // HTTP指标
    requestsTotal   *prometheus.CounterVec
    requestDuration *prometheus.HistogramVec

    // 业务指标
    orderCreated  prometheus.Counter
    orderFailed   prometheus.Counter
    paymentTotal  *prometheus.CounterVec

    // 系统指标
    goroutines prometheus.Gauge
    memAlloc   prometheus.Gauge
}

func NewMetricsCollector() *MetricsCollector {
    mc := &MetricsCollector{
        requestsTotal: promauto.NewCounterVec(
            prometheus.CounterOpts{
                Name: "game_platform_requests_total",
                Help: "Total HTTP requests",
            },
            []string{"method", "endpoint", "status"},
        ),
        // ... 其他指标定义
    }

    // 启动系统指标采集
    go mc.collectSystemMetrics()

    return mc
}

func (mc *MetricsCollector) collectSystemMetrics() {
    ticker := time.NewTicker(10 * time.Second)
    for range ticker.C {
        mc.goroutines.Set(float64(runtime.NumGoroutine()))

        var m runtime.MemStats
        runtime.ReadMemStats(&m)
        mc.memAlloc.Set(float64(m.Alloc))
    }
}

// 使用示例
func (s *OrderService) CreateOrder(ctx context.Context, req *CreateOrderRequest) error {
    start := time.Now()

    err := s.createOrder(req)

    duration := time.Since(start).Seconds()
    if err != nil {
        s.metrics.orderFailed.Inc()
        s.metrics.requestDuration.WithLabelValues("POST", "/orders").Observe(duration)
        return err
    }

    s.metrics.orderCreated.Inc()
    s.metrics.requestDuration.WithLabelValues("POST", "/orders").Observe(duration)
    return nil
}
```

**关键监控指标**：
```
1. 黄金指标（Google SRE）：
   - Latency：延迟（P50, P90, P99）
   - Traffic：流量（QPS）
   - Errors：错误率
   - Saturation：饱和度（CPU、内存、磁盘）

2. USE方法（系统资源）：
   - Utilization：使用率
   - Saturation：饱和度
   - Errors：错误数

3. RED方法（微服务）：
   - Rate：请求速率
   - Errors：错误数
   - Duration：持续时间
```

---

## 第六部分：项目具体问题

### 6.1 套利交易系统

#### 问题28：套利交易系统如何降低延迟到50ms以内？

**答案：**

**低延迟优化策略**：

1. **网络优化**：
```go
// 使用TCP长连接，避免三次握手开销
type ExchangeConnector struct {
    conn    net.Conn
    encoder *json.Encoder
    decoder *json.Decoder
}

func NewExchangeConnector(host string) (*ExchangeConnector, error) {
    // TCP连接优化
    conn, err := net.Dial("tcp", host)
    if err != nil {
        return nil, err
    }

    // 禁用Nagle算法（降低延迟）
    if tcpConn, ok := conn.(*net.TCPConn); ok {
        tcpConn.SetNoDelay(true)
    }

    // 设置读写缓冲区
    conn.(*net.TCPConn).SetReadBuffer(256 * 1024)
    conn.(*net.TCPConn).SetWriteBuffer(256 * 1024)

    return &ExchangeConnector{
        conn:    conn,
        encoder: json.NewEncoder(conn),
        decoder: json.NewDecoder(conn),
    }, nil
}

// WebSocket连接池（复用连接）
type WSConnectionPool struct {
    connections []*websocket.Conn
    mu          sync.Mutex
    index       int
}

func (p *WSConnectionPool) GetConnection() *websocket.Conn {
    p.mu.Lock()
    defer p.mu.Unlock()

    conn := p.connections[p.index]
    p.index = (p.index + 1) % len(p.connections)
    return conn
}
```

2. **数据结构优化**：
```go
// 使用无锁队列传递行情数据
type LockFreeQueue struct {
    buffer []*Tick
    head   uint64
    tail   uint64
    mask   uint64
}

func NewLockFreeQueue(size int) *LockFreeQueue {
    // size必须是2的幂次
    return &LockFreeQueue{
        buffer: make([]*Tick, size),
        mask:   uint64(size - 1),
    }
}

func (q *LockFreeQueue) Push(tick *Tick) bool {
    tail := atomic.LoadUint64(&q.tail)
    next := tail + 1
    if next-atomic.LoadUint64(&q.head) > q.mask {
        return false // 队列满
    }
    q.buffer[tail&q.mask] = tick
    atomic.StoreUint64(&q.tail, next)
    return true
}

func (q *LockFreeQueue) Pop() *Tick {
    head := atomic.LoadUint64(&q.head)
    if head >= atomic.LoadUint64(&q.tail) {
        return nil // 队列空
    }
    tick := q.buffer[head&q.mask]
    atomic.StoreUint64(&q.head, head+1)
    return tick
}
```

3. **避免内存分配**：
```go
// 使用对象池复用订单对象
var orderPool = sync.Pool{
    New: func() interface{} {
        return &Order{}
    },
}

func ProcessTick(tick *Tick) {
    // 从池获取订单对象
    order := orderPool.Get().(*Order)
    defer orderPool.Put(order)

    // 重置并填充数据
    order.Reset()
    order.Symbol = tick.Symbol
    order.Price = tick.Price
    order.Quantity = calculateQuantity(tick)

    // 发送订单
    sendOrder(order)
}

// 预分配切片避免扩容
type TickProcessor struct {
    tickBuffer []*Tick // 预分配1000个
}

func NewTickProcessor() *TickProcessor {
    return &TickProcessor{
        tickBuffer: make([]*Tick, 0, 1000),
    }
}
```

4. **算法优化**：
```go
// 使用查表法代替实时计算
type PriceCalculator struct {
    feeTable [10000]float64 // 预计算手续费表
}

func (c *PriceCalculator) Init() {
    for i := 0; i < 10000; i++ {
        price := float64(i) * 0.01
        c.feeTable[i] = price * 0.001 // 0.1%手续费
    }
}

func (c *PriceCalculator) GetFee(price float64) float64 {
    index := int(price * 100)
    if index < 10000 {
        return c.feeTable[index]
    }
    return price * 0.001 // fallback
}

// 双边套利信号检测（O(1)复杂度）
func DetectArbitrage(bidA, askA, bidB, askB float64) bool {
    // bidB > askA 且价差超过阈值
    spread := (bidB - askA) / askA
    return spread > 0.0005 // 0.05%
}
```

5. **并发优化**：
```go
// 每个交易所独立Goroutine处理，避免锁竞争
type ArbitrageEngine struct {
    exchangeA *ExchangeConnector
    exchangeB *ExchangeConnector
    tickQueue *LockFreeQueue
}

func (e *ArbitrageEngine) Start() {
    // 交易所A行情处理
    go func() {
        for tick := range e.exchangeA.TickChannel {
            e.tickQueue.Push(tick)
        }
    }()

    // 交易所B行情处理
    go func() {
        for tick := range e.exchangeB.TickChannel {
            e.tickQueue.Push(tick)
        }
    }()

    // 套利信号生成（单独Goroutine，无锁）
    go func() {
        for {
            tick := e.tickQueue.Pop()
            if tick == nil {
                time.Sleep(10 * time.Microsecond)
                continue
            }
            e.detectAndTrade(tick)
        }
    }()
}
```

**项目应用**：
```
优化前延迟：200-300ms
优化后延迟：30-50ms

优化措施：
1. 使用WebSocket代替HTTP轮询：减少50ms
2. TCP参数优化（NoDelay）：减少20ms
3. 无锁队列：减少30ms
4. 对象池+预分配：减少40ms
5. 查表法：减少10ms
6. 并发模型优化：减少30ms

最终：平均延迟35ms，P99延迟48ms
```

---

#### 问题29：套利系统如何监控风险和实时仓位？

**答案：**

**风险监控系统**：

```go
type RiskMonitor struct {
    positions    map[string]*Position   // 实时仓位
    orderBook    map[string]*OrderBook  // 委托单
    riskMetrics  *RiskMetrics
    alertChannel chan *RiskAlert
    mu           sync.RWMutex
}

type RiskMetrics struct {
    TotalExposure    float64 // 总敞口
    MaxDrawdown      float64 // 最大回撤
    PnL              float64 // 盈亏
    VaR              float64 // 风险价值
    MarginUsage      float64 // 保证金使用率
}

type Position struct {
    Symbol      string
    Quantity    float64
    AvgPrice    float64
    CurrentPrice float64
    UnrealizedPnL float64
    UpdatedAt   time.Time
}

// 实时仓位更新
func (rm *RiskMonitor) UpdatePosition(symbol string, quantity, price float64) {
    rm.mu.Lock()
    defer rm.mu.Unlock()

    pos, exists := rm.positions[symbol]
    if !exists {
        pos = &Position{Symbol: symbol}
        rm.positions[symbol] = pos
    }

    // 更新均价
    totalValue := pos.Quantity*pos.AvgPrice + quantity*price
    pos.Quantity += quantity
    if pos.Quantity != 0 {
        pos.AvgPrice = totalValue / pos.Quantity
    }
    pos.UpdatedAt = time.Now()

    // 检查风险
    rm.checkRisk(pos)
}

// 风险检查
func (rm *RiskMonitor) checkRisk(pos *Position) {
    // 1. 单个品种持仓限制
    if abs(pos.Quantity) > MAX_POSITION_LIMIT {
        rm.sendAlert(&RiskAlert{
            Level:   AlertCritical,
            Type:    "PositionLimit",
            Message: fmt.Sprintf("%s position exceeds limit: %.2f", pos.Symbol, pos.Quantity),
        })
        // 强制平仓
        rm.forceClosePosition(pos.Symbol)
    }

    // 2. 未实现亏损限制
    if pos.UnrealizedPnL < -MAX_LOSS_LIMIT {
        rm.sendAlert(&RiskAlert{
            Level:   AlertCritical,
            Type:    "LossLimit",
            Message: fmt.Sprintf("%s unrealized loss: %.2f", pos.Symbol, pos.UnrealizedPnL),
        })
        rm.forceClosePosition(pos.Symbol)
    }
}

// 计算风险指标
func (rm *RiskMonitor) CalculateRiskMetrics() *RiskMetrics {
    rm.mu.RLock()
    defer rm.mu.RUnlock()

    var totalExposure, totalPnL float64

    for _, pos := range rm.positions {
        // 更新市价
        currentPrice := rm.getLatestPrice(pos.Symbol)
        pos.CurrentPrice = currentPrice
        pos.UnrealizedPnL = (currentPrice - pos.AvgPrice) * pos.Quantity

        totalExposure += abs(pos.Quantity * pos.CurrentPrice)
        totalPnL += pos.UnrealizedPnL
    }

    rm.riskMetrics.TotalExposure = totalExposure
    rm.riskMetrics.PnL = totalPnL
    rm.riskMetrics.MarginUsage = totalExposure / TOTAL_CAPITAL

    return rm.riskMetrics
}

// Prometheus监控指标
var (
    positionGauge = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "arbitrage_position_quantity",
            Help: "Current position quantity",
        },
        []string{"symbol"},
    )

    pnlGauge = promauto.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "arbitrage_unrealized_pnl",
            Help: "Unrealized PnL",
        },
        []string{"symbol"},
    )

    exposureGauge = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "arbitrage_total_exposure",
            Help: "Total exposure amount",
        },
    )
)

// 定时上报监控指标
func (rm *RiskMonitor) ReportMetrics() {
    ticker := time.NewTicker(1 * time.Second)
    for range ticker.C {
        metrics := rm.CalculateRiskMetrics()

        // 上报到Prometheus
        exposureGauge.Set(metrics.TotalExposure)

        rm.mu.RLock()
        for symbol, pos := range rm.positions {
            positionGauge.WithLabelValues(symbol).Set(pos.Quantity)
            pnlGauge.WithLabelValues(symbol).Set(pos.UnrealizedPnL)
        }
        rm.mu.RUnlock()

        // 数据库持久化
        rm.saveMetricsToDB(metrics)
    }
}
```

**Grafana Dashboard**：
```
指标面板：
1. 实时仓位
   - 各品种持仓量（柱状图）
   - 持仓市值分布（饼图）

2. 盈亏监控
   - 累计盈亏曲线
   - 各品种未实现盈亏

3. 风险指标
   - 总敞口 / 资金上限
   - 保证金使用率
   - 最大回撤

4. 告警历史
   - 风险告警记录
   - 强制平仓记录
```

---

### 6.2 游戏交易平台

#### 问题30：游戏交易平台如何防止超卖问题？

**答案：**

**多重防护方案**：

**1. 数据库层面（悲观锁）**：
```go
func CreateOrder(ctx context.Context, productID, quantity int) error {
    tx, _ := db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelRepeatableRead,
    })
    defer tx.Rollback()

    // FOR UPDATE锁定库存行
    var stock int
    err := tx.QueryRow(`
        SELECT stock FROM products
        WHERE id = ? FOR UPDATE
    `, productID).Scan(&stock)

    if stock < quantity {
        return errors.New("insufficient stock")
    }

    // 扣减库存
    _, err = tx.Exec(`
        UPDATE products
        SET stock = stock - ?
        WHERE id = ?
    `, quantity, productID)

    // 创建订单
    _, err = tx.Exec(`
        INSERT INTO orders (product_id, quantity, user_id)
        VALUES (?, ?, ?)
    `, productID, quantity, userID)

    return tx.Commit()
}
```

**2. Redis分布式锁**：
```go
func CreateOrderWithRedisLock(productID, quantity int) error {
    lockKey := fmt.Sprintf("lock:product:%d", productID)
    lock := NewRedisLock(redisClient, lockKey, 5*time.Second)

    if !lock.Lock() {
        return errors.New("system busy")
    }
    defer lock.Unlock()

    // 检查Redis库存
    stockKey := fmt.Sprintf("product:stock:%d", productID)
    stock, err := redisClient.Get(ctx, stockKey).Int()
    if err != nil || stock < quantity {
        return errors.New("insufficient stock")
    }

    // Redis扣库存
    newStock, err := redisClient.DecrBy(ctx, stockKey, int64(quantity)).Result()
    if err != nil || newStock < 0 {
        // 回滚
        redisClient.IncrBy(ctx, stockKey, int64(quantity))
        return errors.New("insufficient stock")
    }

    // 异步创建订单（通过MQ）
    sendOrderCreateEvent(productID, quantity)

    return nil
}
```

**3. Lua脚本原子操作**：
```go
// Redis Lua脚本（原子性扣库存）
const luaDecrStock = `
    local stock = redis.call('GET', KEYS[1])
    if not stock then
        return -1  -- 库存不存在
    end
    stock = tonumber(stock)
    local quantity = tonumber(ARGV[1])
    if stock < quantity then
        return 0  -- 库存不足
    end
    redis.call('DECRBY', KEYS[1], quantity)
    return 1  -- 扣减成功
`

func DecrStockAtomic(productID, quantity int) error {
    stockKey := fmt.Sprintf("product:stock:%d", productID)

    result, err := redisClient.Eval(ctx, luaDecrStock, []string{stockKey}, quantity).Int()
    if err != nil {
        return err
    }

    switch result {
    case -1:
        return errors.New("product not found")
    case 0:
        return errors.New("insufficient stock")
    case 1:
        return nil
    default:
        return errors.New("unknown error")
    }
}
```

**4. 预扣库存+异步确认**：
```go
// 两阶段提交
func CreateOrderTwoPhase(productID, quantity int) error {
    orderID := generateOrderID()

    // 阶段1：预扣库存
    reserveKey := fmt.Sprintf("product:reserved:%d:%s", productID, orderID)
    if err := reserveStock(productID, quantity, reserveKey); err != nil {
        return err
    }

    // 创建订单（15分钟内支付）
    order := &Order{
        ID:        orderID,
        ProductID: productID,
        Quantity:  quantity,
        Status:    StatusPending,
        ExpireAt:  time.Now().Add(15 * time.Minute),
    }
    if err := db.Create(order).Error; err != nil {
        // 回滚预扣
        releaseStock(productID, quantity, reserveKey)
        return err
    }

    // 异步：15分钟后检查订单状态
    go func() {
        time.Sleep(15 * time.Minute)
        checkAndReleaseExpiredOrder(orderID)
    }()

    return nil
}

// 定时任务：释放过期预扣库存
func checkAndReleaseExpiredOrder(orderID string) {
    var order Order
    db.First(&order, orderID)

    if order.Status == StatusPending && time.Now().After(order.ExpireAt) {
        // 订单过期，释放库存
        reserveKey := fmt.Sprintf("product:reserved:%d:%s", order.ProductID, orderID)
        releaseStock(order.ProductID, order.Quantity, reserveKey)

        // 取消订单
        db.Model(&order).Update("status", StatusCanceled)
    }
}
```

**5. 限流+排队**：
```go
// 秒杀场景：令牌桶限流
type TokenBucket struct {
    capacity int
    tokens   int
    rate     int // 每秒补充tokens
    mu       sync.Mutex
    lastTime time.Time
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    now := time.Now()
    elapsed := now.Sub(tb.lastTime).Seconds()

    // 补充令牌
    tb.tokens += int(elapsed * float64(tb.rate))
    if tb.tokens > tb.capacity {
        tb.tokens = tb.capacity
    }
    tb.lastTime = now

    if tb.tokens > 0 {
        tb.tokens--
        return true
    }
    return false
}

// 使用MQ排队处理
func HandleFlashSale(productID int) {
    bucket := NewTokenBucket(1000, 100) // 容量1000，每秒100

    for req := range requestQueue {
        if !bucket.Allow() {
            respondError(req, "too many requests")
            continue
        }

        // 发送到Kafka异步处理
        kafkaProducer.Send(ctx, &OrderRequest{
            ProductID: productID,
            UserID:    req.UserID,
            Quantity:  req.Quantity,
        })
    }
}
```

**项目实施方案**：
```
游戏交易平台防超卖综合方案：

1. 常规商品：
   - Redis预扣库存（Lua脚本）
   - 异步同步到MySQL
   - 每小时校对库存

2. 限量商品（秒杀）：
   - Redis预加载库存
   - 分布式锁 + Lua脚本
   - MQ削峰
   - 预扣+15分钟超时释放

3. 监控告警：
   - 实时库存监控
   - 超卖告警（Redis < MySQL）
   - 库存校对任务

结果：
- QPS 5000+
- 0超卖事故
- 库存准确率99.99%
```

---

#### 问题31：如何实现游戏道具的实时推荐系统？

**答案：**

**推荐系统架构**：

```go
// 推荐引擎
type RecommendationEngine struct {
    userProfiles    map[int64]*UserProfile // 用户画像
    itemFeatures    map[int64]*ItemFeature // 道具特征
    similarityCache *cache.Cache           // 相似度缓存
}

// 用户画像
type UserProfile struct {
    UserID       int64
    Preferences  map[string]float64 // 偏好：{"weapon": 0.8, "armor": 0.6}
    RecentViews  []int64            // 最近浏览
    Purchases    []int64            // 购买历史
    Level        int                // 用户等级
}

// 道具特征
type ItemFeature struct {
    ItemID     int64
    Category   string           // 类别
    Rarity     string           // 稀有度
    Attributes map[string]float64 // 属性
    Tags       []string         // 标签
}

// 协同过滤推荐
func (re *RecommendationEngine) CollaborativeFilter(userID int64, limit int) []int64 {
    // 1. 找相似用户
    similarUsers := re.findSimilarUsers(userID, 50)

    // 2. 收集相似用户购买的道具
    itemScores := make(map[int64]float64)
    for _, similarUser := range similarUsers {
        profile := re.userProfiles[similarUser.UserID]
        for _, itemID := range profile.Purchases {
            // 加权：相似度 * 道具评分
            itemScores[itemID] += similarUser.Similarity * re.getItemScore(itemID)
        }
    }

    // 3. 过滤已购买道具
    userProfile := re.userProfiles[userID]
    purchased := make(map[int64]bool)
    for _, itemID := range userProfile.Purchases {
        purchased[itemID] = true
    }

    // 4. 排序
    var recommendations []ItemScore
    for itemID, score := range itemScores {
        if !purchased[itemID] {
            recommendations = append(recommendations, ItemScore{ItemID: itemID, Score: score})
        }
    }
    sort.Slice(recommendations, func(i, j int) bool {
        return recommendations[i].Score > recommendations[j].Score
    })

    // 5. 返回Top N
    result := make([]int64, 0, limit)
    for i := 0; i < len(recommendations) && i < limit; i++ {
        result = append(result, recommendations[i].ItemID)
    }
    return result
}

// 基于内容的推荐
func (re *RecommendationEngine) ContentBasedRecommend(userID int64, limit int) []int64 {
    userProfile := re.userProfiles[userID]

    // 计算每个道具与用户偏好的匹配度
    itemScores := make(map[int64]float64)
    for itemID, feature := range re.itemFeatures {
        score := re.calculateMatchScore(userProfile, feature)
        itemScores[itemID] = score
    }

    // 排序并返回
    var recommendations []ItemScore
    for itemID, score := range itemScores {
        recommendations = append(recommendations, ItemScore{ItemID: itemID, Score: score})
    }
    sort.Slice(recommendations, func(i, j int) bool {
        return recommendations[i].Score > recommendations[j].Score
    })

    result := make([]int64, 0, limit)
    for i := 0; i < len(recommendations) && i < limit; i++ {
        result = append(result, recommendations[i].ItemID)
    }
    return result
}

// 混合推荐（协同过滤 + 内容推荐）
func (re *RecommendationEngine) HybridRecommend(userID int64, limit int) []int64 {
    // 50%协同过滤 + 50%内容推荐
    cf := re.CollaborativeFilter(userID, limit*2)
    cb := re.ContentBasedRecommend(userID, limit*2)

    // 合并去重
    itemSet := make(map[int64]bool)
    var result []int64

    for i := 0; i < len(cf) && len(result) < limit; i++ {
        if !itemSet[cf[i]] {
            result = append(result, cf[i])
            itemSet[cf[i]] = true
        }
    }

    for i := 0; i < len(cb) && len(result) < limit; i++ {
        if !itemSet[cb[i]] {
            result = append(result, cb[i])
            itemSet[cb[i]] = true
        }
    }

    return result
}

// 实时推荐API
func GetRecommendations(w http.ResponseWriter, r *http.Request) {
    userID := getUserID(r)

    // 从缓存获取
    cacheKey := fmt.Sprintf("recommend:%d", userID)
    if cached, err := getFromCache(cacheKey); err == nil {
        respondJSON(w, cached)
        return
    }

    // 计算推荐
    recommendations := recommendEngine.HybridRecommend(userID, 20)

    // 查询道具详情
    var items []Item
    db.Where("id IN ?", recommendations).Find(&items)

    // 缓存5分钟
    setToCache(cacheKey, items, 5*time.Minute)

    respondJSON(w, items)
}

// 用户行为追踪（更新画像）
func TrackUserBehavior(userID int64, itemID int64, action string) {
    go func() {
        profile := recommendEngine.userProfiles[userID]

        switch action {
        case "view":
            profile.RecentViews = append(profile.RecentViews, itemID)
            if len(profile.RecentViews) > 50 {
                profile.RecentViews = profile.RecentViews[1:]
            }

        case "purchase":
            profile.Purchases = append(profile.Purchases, itemID)

            // 更新偏好
            item := recommendEngine.itemFeatures[itemID]
            if pref, exists := profile.Preferences[item.Category]; exists {
                profile.Preferences[item.Category] = pref*0.9 + 0.1 // 衰减旧偏好
            } else {
                profile.Preferences[item.Category] = 0.5
            }
        }

        // 异步保存到数据库
        saveUserProfile(profile)

        // 清除推荐缓存
        cacheKey := fmt.Sprintf("recommend:%d", userID)
        deleteFromCache(cacheKey)
    }()
}
```

**项目应用**：
```
游戏交易平台推荐系统：

1. 数据采集：
   - 浏览行为：Kafka采集
   - 购买行为：数据库Event
   - 用户画像：Redis存储

2. 推荐算法：
   - 协同过滤（UserCF）
   - 内容推荐（ItemCF）
   - 热门推荐（Fallback）

3. 性能优化：
   - Redis缓存推荐结果（5分钟）
   - 预计算相似度矩阵（每小时）
   - Goroutine并行计算

结果：
- 推荐响应时间：<20ms
- 点击率提升：45%
- 转化率提升：28%
```

---

## 面试准备总结

### 重点准备方向

1. **Golang基础（必考）**：
   - GMP模型、Channel、Context
   - GC机制、内存管理
   - 并发控制（锁、sync.Map、sync.Pool）

2. **数据库（高频）**：
   - MySQL索引、事务、分库分表
   - Redis缓存策略、分布式锁
   - MongoDB文档模型

3. **微服务架构（重点）**：
   - 服务拆分、服务发现
   - Kafka消息队列、幂等性
   - 分布式事务

4. **云平台部署**：
   - AWS/阿里云核心服务
   - 高可用架构设计
   - Auto Scaling

5. **性能优化（加分项）**：
   - pprof性能分析
   - Prometheus监控
   - 性能调优实战

6. **项目深挖**：
   - 准备每个项目的难点、亮点
   - 数据指标（QPS、延迟、可用性）
   - 问题排查案例

### 面试话术建议

**回答结构**：
```
1. 简要说明（What）
2. 为什么这样做（Why）
3. 具体实现（How）
4. 项目应用（Where）
5. 优化效果（Result）
```

**示例**：
```
面试官：如何解决缓存穿透问题？

回答：
1. 缓存穿透是指查询不存在的数据，导致请求打到数据库
2. 我们在游戏交易平台用了布隆过滤器+缓存空对象两种方案
3. 布隆过滤器预加载所有商品ID，快速判断不存在的请求
   对于存在但查不到的，缓存NULL值5分钟
4. 代码实现是... [展示代码]
5. 上线后，数据库QPS从1000降到200，CPU使用率下降60%
```

### 面试前检查清单

- [ ] 熟悉简历每个项目的技术栈
- [ ] 准备每个项目的难点和解决方案
- [ ] 准备项目的数据指标（QPS、延迟、用户量等）
- [ ] 复习Golang基础知识（GMP、Channel、GC）
- [ ] 复习MySQL索引、事务、分库分表
- [ ] 复习Redis缓存策略、分布式锁
- [ ] 复习Kafka消息可靠性、幂等性
- [ ] 准备AWS/阿里云使用经验
- [ ] 准备性能优化案例（pprof、监控）
- [ ] 准备1-2个失败的项目经验（吸取的教训）

---

## 第七部分：电商系统核心问题

### 7.1 订单系统

#### 问题32：电商系统的订单状态机如何设计？

**答案：**

**订单状态定义**：
```go
type OrderStatus int

const (
    StatusPending      OrderStatus = 1  // 待支付
    StatusPaying       OrderStatus = 2  // 支付中
    StatusPaid         OrderStatus = 3  // 已支付
    StatusProcessing   OrderStatus = 4  // 处理中
    StatusShipped      OrderStatus = 5  // 已发货
    StatusDelivered    OrderStatus = 6  // 已送达
    StatusCompleted    OrderStatus = 7  // 已完成
    StatusCanceling    OrderStatus = 8  // 取消中
    StatusCanceled     OrderStatus = 9  // 已取消
    StatusRefunding    OrderStatus = 10 // 退款中
    StatusRefunded     OrderStatus = 11 // 已退款
)

// 订单状态流转规则
var StatusTransitions = map[OrderStatus][]OrderStatus{
    StatusPending:    {StatusPaying, StatusCanceled},
    StatusPaying:     {StatusPaid, StatusCanceled},
    StatusPaid:       {StatusProcessing, StatusRefunding},
    StatusProcessing: {StatusShipped, StatusRefunding},
    StatusShipped:    {StatusDelivered, StatusRefunding},
    StatusDelivered:  {StatusCompleted, StatusRefunding},
    StatusCompleted:  {}, // 终态
    StatusRefunding:  {StatusRefunded, StatusPaid}, // 退款失败回到已支付
    StatusRefunded:   {}, // 终态
    StatusCanceled:   {}, // 终态
}
```

**订单状态机实现**：
```go
type Order struct {
    ID            int64
    UserID        int64
    Amount        float64
    Status        OrderStatus
    PaidAt        *time.Time
    ShippedAt     *time.Time
    CompletedAt   *time.Time
    CanceledAt    *time.Time
    Version       int64  // 乐观锁版本号
    CreatedAt     time.Time
    UpdatedAt     time.Time
}

type OrderStateMachine struct {
    db *gorm.DB
}

// 状态流转（带幂等性保证）
func (sm *OrderStateMachine) TransitionTo(orderID int64, targetStatus OrderStatus, reason string) error {
    return sm.db.Transaction(func(tx *gorm.DB) error {
        // 1. 查询订单（加锁）
        var order Order
        err := tx.Clauses(clause.Locking{Strength: "UPDATE"}).
            First(&order, orderID).Error
        if err != nil {
            return err
        }

        // 2. 检查状态流转是否合法
        if !sm.canTransition(order.Status, targetStatus) {
            return fmt.Errorf("invalid transition: %d -> %d", order.Status, targetStatus)
        }

        // 3. 更新状态（乐观锁）
        result := tx.Model(&Order{}).
            Where("id = ? AND version = ?", orderID, order.Version).
            Updates(map[string]interface{}{
                "status":     targetStatus,
                "version":    order.Version + 1,
                "updated_at": time.Now(),
            })

        if result.RowsAffected == 0 {
            return errors.New("order status changed, please retry")
        }

        // 4. 记录状态变更日志
        log := &OrderStatusLog{
            OrderID:    orderID,
            FromStatus: order.Status,
            ToStatus:   targetStatus,
            Reason:     reason,
            CreatedAt:  time.Now(),
        }
        if err := tx.Create(log).Error; err != nil {
            return err
        }

        // 5. 触发后续动作
        sm.afterTransition(tx, &order, targetStatus)

        return nil
    })
}

func (sm *OrderStateMachine) canTransition(from, to OrderStatus) bool {
    allowedStatuses, exists := StatusTransitions[from]
    if !exists {
        return false
    }
    for _, status := range allowedStatuses {
        if status == to {
            return true
        }
    }
    return false
}

// 状态流转后的动作
func (sm *OrderStateMachine) afterTransition(tx *gorm.DB, order *Order, newStatus OrderStatus) {
    switch newStatus {
    case StatusPaid:
        // 扣减库存
        deductStock(tx, order.ID)
        // 发送通知
        sendNotification(order.UserID, "订单支付成功")

    case StatusShipped:
        // 发送物流通知
        sendShippingNotification(order.UserID, order.ID)

    case StatusCanceled:
        // 释放库存
        releaseStock(tx, order.ID)
        // 退款（如果已支付）
        if order.PaidAt != nil {
            refundOrder(order.ID)
        }

    case StatusRefunded:
        // 释放库存
        releaseStock(tx, order.ID)
    }
}
```

**项目应用**：
```go
// 跨境电商订单状态流转示例
// 用户支付成功回调
func HandlePaymentCallback(req *PaymentCallback) error {
    // 幂等性检查
    if isDuplicate(req.OrderID, req.TransactionID) {
        return nil
    }

    // 状态流转：待支付 -> 已支付
    err := orderStateMachine.TransitionTo(
        req.OrderID,
        StatusPaid,
        fmt.Sprintf("payment success: %s", req.TransactionID),
    )

    if err != nil {
        log.Printf("transition failed: %v", err)
        return err
    }

    return nil
}

// 自动取消超时未支付订单
func CancelExpiredOrders() {
    expiredTime := time.Now().Add(-30 * time.Minute)

    var orders []Order
    db.Where("status = ? AND created_at < ?", StatusPending, expiredTime).Find(&orders)

    for _, order := range orders {
        orderStateMachine.TransitionTo(
            order.ID,
            StatusCanceled,
            "payment timeout",
        )
    }
}
```

---

#### 问题33：如何防止重复下单和重复支付？

**答案：**

**重复下单的场景**：
1. 用户快速点击多次下单按钮
2. 网络抖动导致请求重试
3. 恶意刷单

**防重复下单方案**：

**方案1：前端防抖+后端幂等性Token**
```go
// 生成下单Token
func GenerateOrderToken(userID int64) (string, error) {
    token := uuid.New().String()
    key := fmt.Sprintf("order:token:%d:%s", userID, token)

    // Redis存储，5分钟有效
    err := redisClient.Set(ctx, key, "1", 5*time.Minute).Err()
    return token, err
}

// 创建订单（验证Token）
func CreateOrder(req *CreateOrderRequest) (*Order, error) {
    // 1. 验证并删除Token（原子操作）
    key := fmt.Sprintf("order:token:%d:%s", req.UserID, req.Token)
    result, err := redisClient.Del(ctx, key).Result()
    if err != nil || result == 0 {
        return nil, errors.New("invalid or expired token")
    }

    // 2. 创建订单
    order := &Order{
        ID:     generateOrderID(),
        UserID: req.UserID,
        Amount: calculateAmount(req.Items),
        Status: StatusPending,
    }

    if err := db.Create(order).Error; err != nil {
        return nil, err
    }

    // 3. 创建订单明细
    for _, item := range req.Items {
        orderItem := &OrderItem{
            OrderID:   order.ID,
            ProductID: item.ProductID,
            Quantity:  item.Quantity,
            Price:     item.Price,
        }
        db.Create(orderItem)
    }

    return order, nil
}
```

**方案2：分布式锁**
```go
func CreateOrder(req *CreateOrderRequest) (*Order, error) {
    // 使用用户ID作为锁的key，防止同一用户并发下单
    lockKey := fmt.Sprintf("lock:create_order:%d", req.UserID)
    lock := NewRedisLock(redisClient, lockKey, 5*time.Second)

    if !lock.Lock() {
        return nil, errors.New("请勿重复提交")
    }
    defer lock.Unlock()

    // 检查是否存在待支付订单
    var count int64
    db.Model(&Order{}).
        Where("user_id = ? AND status = ?", req.UserID, StatusPending).
        Count(&count)
    if count > 0 {
        return nil, errors.New("您有待支付的订单，请先完成支付")
    }

    // 创建订单...
    return createOrderInternal(req)
}
```

**重复支付的场景**：
1. 支付回调重复通知
2. 用户重复点击支付按钮
3. 网络重试导致重复请求

**防重复支付方案**：

**方案1：订单状态+事务ID去重**
```go
type PaymentRecord struct {
    ID            int64
    OrderID       int64
    TransactionID string  // 支付平台交易号（唯一）
    Amount        float64
    Status        string
    CreatedAt     time.Time
}

// 处理支付回调（幂等性）
func HandlePaymentCallback(callback *PaymentCallback) error {
    return db.Transaction(func(tx *gorm.DB) error {
        // 1. 检查交易号是否已处理（唯一索引）
        var existRecord PaymentRecord
        err := tx.Where("transaction_id = ?", callback.TransactionID).
            First(&existRecord).Error

        if err == nil {
            // 已处理过，直接返回成功
            log.Printf("duplicate callback: %s", callback.TransactionID)
            return nil
        }

        // 2. 查询订单
        var order Order
        err = tx.Clauses(clause.Locking{Strength: "UPDATE"}).
            First(&order, callback.OrderID).Error
        if err != nil {
            return err
        }

        // 3. 检查订单状态
        if order.Status == StatusPaid {
            log.Printf("order already paid: %d", order.ID)
            return nil // 已支付，幂等返回
        }

        if order.Status != StatusPending && order.Status != StatusPaying {
            return fmt.Errorf("invalid order status: %d", order.Status)
        }

        // 4. 验证金额
        if math.Abs(order.Amount-callback.Amount) > 0.01 {
            return errors.New("amount mismatch")
        }

        // 5. 创建支付记录
        record := &PaymentRecord{
            OrderID:       callback.OrderID,
            TransactionID: callback.TransactionID,
            Amount:        callback.Amount,
            Status:        "success",
            CreatedAt:     time.Now(),
        }
        if err := tx.Create(record).Error; err != nil {
            return err
        }

        // 6. 更新订单状态
        now := time.Now()
        err = tx.Model(&Order{}).
            Where("id = ? AND status IN ?", order.ID, []OrderStatus{StatusPending, StatusPaying}).
            Updates(map[string]interface{}{
                "status":  StatusPaid,
                "paid_at": now,
            }).Error

        if err != nil {
            return err
        }

        // 7. 扣减库存
        if err := deductStock(tx, order.ID); err != nil {
            return err
        }

        return nil
    })
}

// 数据库设置唯一索引
// CREATE UNIQUE INDEX idx_transaction_id ON payment_records(transaction_id);
```

**方案2：支付流水号去重**
```go
// 用户发起支付
func InitiatePayment(orderID int64) (*PaymentResponse, error) {
    // 1. 查询订单
    var order Order
    if err := db.First(&order, orderID).Error; err != nil {
        return nil, err
    }

    if order.Status != StatusPending {
        return nil, errors.New("order status invalid")
    }

    // 2. 生成支付流水号（幂等性key）
    paymentNo := generatePaymentNo(orderID)

    // 3. 检查是否已发起过支付
    key := fmt.Sprintf("payment:init:%s", paymentNo)
    exists, _ := redisClient.Exists(ctx, key).Result()
    if exists > 0 {
        // 已发起，返回之前的支付信息
        return getExistingPayment(paymentNo)
    }

    // 4. 调用支付平台API
    paymentResp, err := callPaymentAPI(&PaymentRequest{
        OrderID:   orderID,
        PaymentNo: paymentNo,
        Amount:    order.Amount,
    })
    if err != nil {
        return nil, err
    }

    // 5. 缓存支付信息（1小时）
    redisClient.Set(ctx, key, toJSON(paymentResp), 1*time.Hour)

    // 6. 更新订单状态为支付中
    db.Model(&Order{}).Where("id = ?", orderID).Update("status", StatusPaying)

    return paymentResp, nil
}

// 生成支付流水号（订单ID+时间戳）
func generatePaymentNo(orderID int64) string {
    return fmt.Sprintf("PAY%d%d", orderID, time.Now().Unix())
}
```

**完整防重支付流程**：
```go
// 完整的支付防重方案
type PaymentService struct {
    db          *gorm.DB
    redisClient *redis.Client
}

// 发起支付
func (s *PaymentService) Pay(orderID int64) (*PaymentResult, error) {
    // 1. 分布式锁（防止并发支付同一订单）
    lockKey := fmt.Sprintf("lock:pay:%d", orderID)
    lock := NewRedisLock(s.redisClient, lockKey, 10*time.Second)
    if !lock.Lock() {
        return nil, errors.New("支付请求处理中，请稍后")
    }
    defer lock.Unlock()

    // 2. 查询订单
    var order Order
    if err := s.db.First(&order, orderID).Error; err != nil {
        return nil, err
    }

    // 3. 检查订单状态
    if order.Status == StatusPaid {
        return nil, errors.New("订单已支付")
    }
    if order.Status != StatusPending {
        return nil, errors.New("订单状态异常")
    }

    // 4. 生成支付请求（幂等性）
    paymentNo := generatePaymentNo(orderID)

    // 5. 调用第三方支付
    result, err := s.callThirdPartyPayment(order.ID, paymentNo, order.Amount)
    if err != nil {
        return nil, err
    }

    return result, nil
}

// 支付回调处理（核心：幂等性）
func (s *PaymentService) HandleCallback(cb *PaymentCallback) error {
    // 1. 验证签名
    if !s.verifySignature(cb) {
        return errors.New("invalid signature")
    }

    // 2. 使用事务ID作为幂等性key
    idempotentKey := fmt.Sprintf("payment:callback:%s", cb.TransactionID)

    // 使用Redis的SETNX实现幂等性
    success, err := s.redisClient.SetNX(ctx, idempotentKey, "1", 24*time.Hour).Result()
    if err != nil {
        return err
    }
    if !success {
        log.Printf("duplicate callback: %s", cb.TransactionID)
        return nil // 重复回调，直接返回成功
    }

    // 3. 处理支付结果
    return s.db.Transaction(func(tx *gorm.DB) error {
        var order Order
        if err := tx.Clauses(clause.Locking{Strength: "UPDATE"}).
            First(&order, cb.OrderID).Error; err != nil {
            return err
        }

        // 已支付，幂等返回
        if order.Status == StatusPaid {
            return nil
        }

        // 验证金额
        if order.Amount != cb.Amount {
            return errors.New("amount mismatch")
        }

        // 创建支付记录
        payment := &PaymentRecord{
            OrderID:       cb.OrderID,
            TransactionID: cb.TransactionID,
            Amount:        cb.Amount,
            Status:        cb.Status,
        }
        if err := tx.Create(payment).Error; err != nil {
            return err
        }

        // 更新订单状态
        now := time.Now()
        return tx.Model(&Order{}).Where("id = ?", order.ID).Updates(map[string]interface{}{
            "status":  StatusPaid,
            "paid_at": now,
        }).Error
    })
}
```

**项目应用总结**：
```
跨境电商防重复支付方案：

1. 防重复下单：
   - 前端按钮禁用（体验层）
   - 幂等性Token（应用层）
   - 分布式锁（服务层）
   - 数据库唯一约束（数据层）

2. 防重复支付：
   - 订单状态校验
   - 交易ID唯一约束（数据库）
   - Redis幂等性key（24小时）
   - 支付回调签名验证

结果：
- 0重复支付事故
- 支付成功率99.8%
- 回调处理<50ms
```

---

### 7.2 支付系统

#### 问题34：电商系统如何对接多个支付渠道？

**答案：**

**支付渠道统一抽象**：
```go
// 支付渠道接口
type PaymentChannel interface {
    // 发起支付
    Pay(req *PaymentRequest) (*PaymentResponse, error)

    // 查询支付结果
    Query(orderID string) (*PaymentResult, error)

    // 退款
    Refund(req *RefundRequest) (*RefundResponse, error)

    // 验证回调签名
    VerifyCallback(data map[string]string) bool

    // 解析回调数据
    ParseCallback(data map[string]string) (*PaymentCallback, error)
}

// 支付请求
type PaymentRequest struct {
    OrderID     string
    Amount      float64
    Currency    string
    Subject     string
    NotifyURL   string
    ReturnURL   string
    UserID      int64
    Extra       map[string]interface{}
}

// 支付响应
type PaymentResponse struct {
    PaymentNo   string // 支付流水号
    PaymentURL  string // 支付跳转URL
    QRCode      string // 二维码（扫码支付）
    Extra       map[string]interface{}
}
```

**各支付渠道实现**：

**1. 支付宝实现**
```go
type AlipayChannel struct {
    appID      string
    privateKey string
    publicKey  string
    gateway    string
}

func (c *AlipayChannel) Pay(req *PaymentRequest) (*PaymentResponse, error) {
    client, err := alipay.New(c.appID, c.privateKey, false)
    if err != nil {
        return nil, err
    }

    // 构造支付参数
    param := alipay.TradePagePay{
        Trade: alipay.Trade{
            Subject:     req.Subject,
            OutTradeNo:  req.OrderID,
            TotalAmount: fmt.Sprintf("%.2f", req.Amount),
            ProductCode: "FAST_INSTANT_TRADE_PAY",
        },
        ReturnURL: req.ReturnURL,
        NotifyURL: req.NotifyURL,
    }

    // 生成支付URL
    url, err := client.TradePagePay(param)
    if err != nil {
        return nil, err
    }

    return &PaymentResponse{
        PaymentNo:  req.OrderID,
        PaymentURL: url.String(),
    }, nil
}

func (c *AlipayChannel) VerifyCallback(data map[string]string) bool {
    // 验证支付宝签名
    return alipay.VerifySign(data, c.publicKey)
}

func (c *AlipayChannel) ParseCallback(data map[string]string) (*PaymentCallback, error) {
    amount, _ := strconv.ParseFloat(data["total_amount"], 64)

    return &PaymentCallback{
        OrderID:       data["out_trade_no"],
        TransactionID: data["trade_no"],
        Amount:        amount,
        Status:        data["trade_status"],
        PaidAt:        parseTime(data["gmt_payment"]),
    }, nil
}
```

**2. 微信支付实现**
```go
type WechatChannel struct {
    appID     string
    mchID     string
    apiKey    string
    certPath  string
}

func (c *WechatChannel) Pay(req *PaymentRequest) (*PaymentResponse, error) {
    client := wechat.NewClient(c.appID, c.mchID, c.apiKey)

    // 统一下单
    unifiedOrder := &wechat.UnifiedOrderRequest{
        Body:           req.Subject,
        OutTradeNo:     req.OrderID,
        TotalFee:       int(req.Amount * 100), // 分
        SpbillCreateIP: "127.0.0.1",
        NotifyURL:      req.NotifyURL,
        TradeType:      "NATIVE", // 扫码支付
    }

    resp, err := client.UnifiedOrder(unifiedOrder)
    if err != nil {
        return nil, err
    }

    return &PaymentResponse{
        PaymentNo: req.OrderID,
        QRCode:    resp.CodeURL, // 二维码URL
    }, nil
}
```

**3. Stripe支付实现（国际支付）**
```go
type StripeChannel struct {
    apiKey string
}

func (c *StripeChannel) Pay(req *PaymentRequest) (*PaymentResponse, error) {
    stripe.Key = c.apiKey

    params := &stripe.PaymentIntentParams{
        Amount:   stripe.Int64(int64(req.Amount * 100)), // cents
        Currency: stripe.String(req.Currency),
        PaymentMethodTypes: stripe.StringSlice([]string{
            "card",
        }),
        Metadata: map[string]string{
            "order_id": req.OrderID,
        },
    }

    pi, err := paymentintent.New(params)
    if err != nil {
        return nil, err
    }

    return &PaymentResponse{
        PaymentNo: pi.ID,
        Extra: map[string]interface{}{
            "client_secret": pi.ClientSecret,
        },
    }, nil
}
```

**支付渠道工厂**：
```go
type ChannelType string

const (
    ChannelAlipay ChannelType = "alipay"
    ChannelWechat ChannelType = "wechat"
    ChannelStripe ChannelType = "stripe"
    ChannelPayPal ChannelType = "paypal"
)

type PaymentChannelFactory struct {
    channels map[ChannelType]PaymentChannel
}

func NewPaymentChannelFactory() *PaymentChannelFactory {
    return &PaymentChannelFactory{
        channels: make(map[ChannelType]PaymentChannel),
    }
}

func (f *PaymentChannelFactory) Register(channelType ChannelType, channel PaymentChannel) {
    f.channels[channelType] = channel
}

func (f *PaymentChannelFactory) Get(channelType ChannelType) (PaymentChannel, error) {
    channel, exists := f.channels[channelType]
    if !exists {
        return nil, fmt.Errorf("unsupported channel: %s", channelType)
    }
    return channel, nil
}

// 初始化
func InitPaymentChannels() *PaymentChannelFactory {
    factory := NewPaymentChannelFactory()

    // 注册支付宝
    factory.Register(ChannelAlipay, &AlipayChannel{
        appID:      config.Alipay.AppID,
        privateKey: config.Alipay.PrivateKey,
        publicKey:  config.Alipay.PublicKey,
    })

    // 注册微信
    factory.Register(ChannelWechat, &WechatChannel{
        appID:  config.Wechat.AppID,
        mchID:  config.Wechat.MchID,
        apiKey: config.Wechat.APIKey,
    })

    // 注册Stripe
    factory.Register(ChannelStripe, &StripeChannel{
        apiKey: config.Stripe.APIKey,
    })

    return factory
}
```

**统一支付服务**：
```go
type PaymentService struct {
    channelFactory *PaymentChannelFactory
    db             *gorm.DB
}

// 发起支付
func (s *PaymentService) CreatePayment(orderID int64, channelType ChannelType) (*PaymentResponse, error) {
    // 1. 查询订单
    var order Order
    if err := s.db.First(&order, orderID).Error; err != nil {
        return nil, err
    }

    // 2. 获取支付渠道
    channel, err := s.channelFactory.Get(channelType)
    if err != nil {
        return nil, err
    }

    // 3. 构造支付请求
    req := &PaymentRequest{
        OrderID:   fmt.Sprintf("%d", order.ID),
        Amount:    order.Amount,
        Currency:  "USD",
        Subject:   fmt.Sprintf("Order #%d", order.ID),
        NotifyURL: fmt.Sprintf("https://api.example.com/payment/callback/%s", channelType),
        ReturnURL: "https://www.example.com/order/success",
        UserID:    order.UserID,
    }

    // 4. 调用支付渠道
    resp, err := channel.Pay(req)
    if err != nil {
        return nil, err
    }

    // 5. 记录支付流水
    payment := &PaymentRecord{
        OrderID:     order.ID,
        PaymentNo:   resp.PaymentNo,
        Channel:     string(channelType),
        Amount:      order.Amount,
        Status:      "pending",
        CreatedAt:   time.Now(),
    }
    s.db.Create(payment)

    return resp, nil
}

// 统一回调处理
func (s *PaymentService) HandleCallback(channelType ChannelType, data map[string]string) error {
    // 1. 获取支付渠道
    channel, err := s.channelFactory.Get(channelType)
    if err != nil {
        return err
    }

    // 2. 验证签名
    if !channel.VerifyCallback(data) {
        return errors.New("invalid signature")
    }

    // 3. 解析回调数据
    callback, err := channel.ParseCallback(data)
    if err != nil {
        return err
    }

    // 4. 处理支付结果（幂等性）
    return s.processPaymentResult(callback)
}

func (s *PaymentService) processPaymentResult(callback *PaymentCallback) error {
    // 前面已实现的幂等性处理逻辑
    return HandlePaymentCallback(callback)
}
```

**项目应用**：
```
跨境电商支付渠道：

国内：
- 支付宝（60%）
- 微信支付（30%）

国际：
- Stripe（5%）
- PayPal（5%）

优势：
1. 统一接口，易于扩展新渠道
2. 各渠道独立配置和降级
3. 自动路由最优支付渠道
4. 支持A/B测试不同渠道

结果：
- 支付成功率：98.5%
- 平均支付时长：3.2秒
- 回调处理成功率：99.9%
```

---

#### 问题35：如何处理退款流程？退款失败如何处理？

**答案：**

**退款状态定义**：
```go
type RefundStatus string

const (
    RefundStatusPending    RefundStatus = "pending"    // 退款中
    RefundStatusSuccess    RefundStatus = "success"    // 退款成功
    RefundStatusFailed     RefundStatus = "failed"     // 退款失败
    RefundStatusProcessing RefundStatus = "processing" // 处理中
)

type RefundRecord struct {
    ID            int64
    OrderID       int64
    RefundNo      string       // 退款单号
    TransactionID string       // 原支付交易号
    Amount        float64      // 退款金额
    Reason        string       // 退款原因
    Status        RefundStatus
    Channel       string       // 退款渠道
    FailedReason  string       // 失败原因
    RetryCount    int          // 重试次数
    SuccessAt     *time.Time
    CreatedAt     time.Time
    UpdatedAt     time.Time
}
```

**退款流程**：
```go
type RefundService struct {
    db             *gorm.DB
    channelFactory *PaymentChannelFactory
}

// 发起退款
func (s *RefundService) CreateRefund(orderID int64, amount float64, reason string) (*RefundRecord, error) {
    return s.db.Transaction(func(tx *gorm.DB) error {
        // 1. 查询订单
        var order Order
        err := tx.Clauses(clause.Locking{Strength: "UPDATE"}).
            First(&order, orderID).Error
        if err != nil {
            return err
        }

        // 2. 检查订单状态
        if order.Status != StatusPaid && order.Status != StatusShipped {
            return errors.New("order cannot be refunded")
        }

        // 3. 检查退款金额
        if amount > order.Amount {
            return errors.New("refund amount exceeds order amount")
        }

        // 4. 检查是否已有退款记录
        var existingRefund RefundRecord
        err = tx.Where("order_id = ? AND status IN ?", orderID,
            []RefundStatus{RefundStatusPending, RefundStatusProcessing, RefundStatusSuccess}).
            First(&existingRefund).Error
        if err == nil {
            return errors.New("refund already exists")
        }

        // 5. 查询支付记录
        var payment PaymentRecord
        err = tx.Where("order_id = ? AND status = ?", orderID, "success").
            First(&payment).Error
        if err != nil {
            return errors.New("payment record not found")
        }

        // 6. 创建退款记录
        refund := &RefundRecord{
            OrderID:       orderID,
            RefundNo:      generateRefundNo(),
            TransactionID: payment.TransactionID,
            Amount:        amount,
            Reason:        reason,
            Status:        RefundStatusPending,
            Channel:       payment.Channel,
            CreatedAt:     time.Now(),
        }
        if err := tx.Create(refund).Error; err != nil {
            return err
        }

        // 7. 更新订单状态
        err = tx.Model(&Order{}).Where("id = ?", orderID).
            Update("status", StatusRefunding).Error
        if err != nil {
            return err
        }

        // 8. 异步调用退款接口
        go s.processRefund(refund.ID)

        return nil
    })
}

// 处理退款
func (s *RefundService) processRefund(refundID int64) error {
    // 1. 查询退款记录
    var refund RefundRecord
    if err := s.db.First(&refund, refundID).Error; err != nil {
        return err
    }

    // 2. 获取支付渠道
    channel, err := s.channelFactory.Get(ChannelType(refund.Channel))
    if err != nil {
        s.updateRefundStatus(refundID, RefundStatusFailed, err.Error())
        return err
    }

    // 3. 调用退款接口
    req := &RefundRequest{
        RefundNo:      refund.RefundNo,
        TransactionID: refund.TransactionID,
        Amount:        refund.Amount,
        Reason:        refund.Reason,
    }

    resp, err := channel.Refund(req)
    if err != nil {
        s.updateRefundStatus(refundID, RefundStatusFailed, err.Error())

        // 加入重试队列
        s.addToRetryQueue(refundID)
        return err
    }

    // 4. 更新退款状态
    if resp.Success {
        s.updateRefundStatus(refundID, RefundStatusSuccess, "")

        // 更新订单状态为已退款
        s.db.Model(&Order{}).Where("id = ?", refund.OrderID).
            Update("status", StatusRefunded)
    } else {
        s.updateRefundStatus(refundID, RefundStatusFailed, resp.Message)
        s.addToRetryQueue(refundID)
    }

    return nil
}

// 更新退款状态
func (s *RefundService) updateRefundStatus(refundID int64, status RefundStatus, reason string) {
    updates := map[string]interface{}{
        "status":     status,
        "updated_at": time.Now(),
    }

    if status == RefundStatusFailed {
        updates["failed_reason"] = reason
        updates["retry_count"] = gorm.Expr("retry_count + 1")
    }

    if status == RefundStatusSuccess {
        now := time.Now()
        updates["success_at"] = &now
    }

    s.db.Model(&RefundRecord{}).Where("id = ?", refundID).Updates(updates)
}

// 退款重试队列
func (s *RefundService) addToRetryQueue(refundID int64) {
    // 发送到Kafka重试队列
    msg := &RefundRetryMessage{
        RefundID:  refundID,
        RetryAt:   time.Now().Add(5 * time.Minute),
        CreatedAt: time.Now(),
    }
    kafkaProducer.Send(ctx, "refund.retry", msg)
}

// 定时任务：重试失败的退款
func (s *RefundService) RetryFailedRefunds() {
    // 查询失败的退款（重试次数<3）
    var refunds []RefundRecord
    s.db.Where("status = ? AND retry_count < ?", RefundStatusFailed, 3).
        Where("updated_at < ?", time.Now().Add(-10*time.Minute)).
        Find(&refunds)

    for _, refund := range refunds {
        go s.processRefund(refund.ID)
    }
}

// 手动重试
func (s *RefundService) ManualRetry(refundID int64) error {
    var refund RefundRecord
    if err := s.db.First(&refund, refundID).Error; err != nil {
        return err
    }

    if refund.Status != RefundStatusFailed {
        return errors.New("only failed refunds can be retried")
    }

    // 重置状态
    s.db.Model(&RefundRecord{}).Where("id = ?", refundID).
        Updates(map[string]interface{}{
            "status":        RefundStatusPending,
            "failed_reason": "",
        })

    // 重新处理
    return s.processRefund(refundID)
}
```

**退款异常处理**：
```go
// 退款对账
func (s *RefundService) ReconcileRefunds() {
    // 1. 查询处理中的退款（超过1小时）
    var refunds []RefundRecord
    s.db.Where("status IN ?", []RefundStatus{RefundStatusPending, RefundStatusProcessing}).
        Where("created_at < ?", time.Now().Add(-1*time.Hour)).
        Find(&refunds)

    for _, refund := range refunds {
        // 2. 查询支付渠道退款状态
        channel, _ := s.channelFactory.Get(ChannelType(refund.Channel))
        result, err := channel.QueryRefund(refund.RefundNo)
        if err != nil {
            log.Printf("query refund failed: %v", err)
            continue
        }

        // 3. 更新退款状态
        if result.Status == "success" {
            s.updateRefundStatus(refund.ID, RefundStatusSuccess, "")
            s.db.Model(&Order{}).Where("id = ?", refund.OrderID).
                Update("status", StatusRefunded)
        } else if result.Status == "failed" {
            s.updateRefundStatus(refund.ID, RefundStatusFailed, result.Reason)
        }
    }
}

// 部分退款
func (s *RefundService) PartialRefund(orderID int64, items []RefundItem) error {
    // 计算退款金额
    var totalRefund float64
    for _, item := range items {
        totalRefund += item.Amount
    }

    // 创建退款
    refund, err := s.CreateRefund(orderID, totalRefund, "部分退款")
    if err != nil {
        return err
    }

    // 记录退款明细
    for _, item := range items {
        detail := &RefundDetail{
            RefundID:  refund.ID,
            ProductID: item.ProductID,
            Quantity:  item.Quantity,
            Amount:    item.Amount,
        }
        s.db.Create(detail)
    }

    return nil
}
```

**项目应用**：
```
跨境电商退款处理方案：

1. 自动退款：
   - 取消未发货订单：秒级退款
   - 拒收订单：1-3个工作日

2. 退款重试机制：
   - 失败立即重试1次
   - 5分钟后重试
   - 1小时后重试
   - 超过3次转人工处理

3. 退款对账：
   - 每小时对账处理中的退款
   - 每日全量对账
   - 差异自动告警

4. 监控告警：
   - 退款失败率 > 5% 告警
   - 退款处理时长 > 10分钟告警
   - 单日退款金额异常告警

结果：
- 自动退款成功率：97%
- 平均退款时长：2.5小时
- 退款对账准确率：99.99%
```

---

### 7.3 优惠券/促销系统

#### 问题36：优惠券系统如何设计？如何防止超发和重复使用？

**答案：**

**优惠券模型设计**：
```go
// 优惠券模板
type CouponTemplate struct {
    ID          int64
    Name        string
    Type        string  // 类型：满减、折扣、代金券
    Amount      float64 // 优惠金额
    Discount    float64 // 折扣率（0.8表示8折）
    MinAmount   float64 // 最低消费金额
    TotalCount  int     // 总发行量
    UsedCount   int     // 已使用数量
    PerUserLimit int    // 每人限领
    StartAt     time.Time
    EndAt       time.Time
    Status      string  // 状态：active, inactive
    CreatedAt   time.Time
}

// 用户优惠券
type UserCoupon struct {
    ID         int64
    UserID     int64
    TemplateID int64
    Code       string      // 优惠券码
    Status     string      // 状态：unused, used, expired
    OrderID    *int64      // 使用的订单ID
    UsedAt     *time.Time
    ExpireAt   time.Time
    CreatedAt  time.Time
}

// CREATE UNIQUE INDEX idx_user_template ON user_coupons(user_id, template_id);
// CREATE UNIQUE INDEX idx_code ON user_coupons(code);
```

**优惠券领取（防超发）**：
```go
type CouponService struct {
    db    *gorm.DB
    redis *redis.Client
}

// 领取优惠券
func (s *CouponService) ClaimCoupon(userID, templateID int64) (*UserCoupon, error) {
    // 1. 分布式锁（防止并发领取）
    lockKey := fmt.Sprintf("lock:claim_coupon:%d:%d", userID, templateID)
    lock := NewRedisLock(s.redis, lockKey, 5*time.Second)
    if !lock.Lock() {
        return nil, errors.New("请勿重复领取")
    }
    defer lock.Unlock()

    // 2. 检查Redis库存（快速失败）
    stockKey := fmt.Sprintf("coupon:stock:%d", templateID)
    stock, err := s.redis.Get(ctx, stockKey).Int()
    if err == nil && stock <= 0 {
        return nil, errors.New("优惠券已领完")
    }

    return s.db.Transaction(func(tx *gorm.DB) error {
        // 3. 查询优惠券模板（加锁）
        var template CouponTemplate
        err := tx.Clauses(clause.Locking{Strength: "UPDATE"}).
            First(&template, templateID).Error
        if err != nil {
            return err
        }

        // 4. 检查是否已过期
        now := time.Now()
        if now.Before(template.StartAt) || now.After(template.EndAt) {
            return errors.New("优惠券不在有效期内")
        }

        // 5. 检查总库存
        if template.UsedCount >= template.TotalCount {
            return errors.New("优惠券已领完")
        }

        // 6. 检查用户领取次数
        if template.PerUserLimit > 0 {
            var count int64
            tx.Model(&UserCoupon{}).
                Where("user_id = ? AND template_id = ?", userID, templateID).
                Count(&count)
            if count >= int64(template.PerUserLimit) {
                return errors.New(fmt.Sprintf("每人限领%d张", template.PerUserLimit))
            }
        }

        // 7. 增加已使用计数
        err = tx.Model(&CouponTemplate{}).
            Where("id = ? AND used_count < total_count", templateID).
            Update("used_count", gorm.Expr("used_count + 1")).Error
        if err != nil {
            return err
        }

        // 8. 创建用户优惠券
        coupon := &UserCoupon{
            UserID:     userID,
            TemplateID: templateID,
            Code:       generateCouponCode(),
            Status:     "unused",
            ExpireAt:   template.EndAt,
            CreatedAt:  now,
        }
        if err := tx.Create(coupon).Error; err != nil {
            return err
        }

        // 9. 扣减Redis库存
        s.redis.Decr(ctx, stockKey)

        return nil
    })
}

// 预加载库存到Redis
func (s *CouponService) PreloadStock(templateID int64) error {
    var template CouponTemplate
    if err := s.db.First(&template, templateID).Error; err != nil {
        return err
    }

    remaining := template.TotalCount - template.UsedCount
    stockKey := fmt.Sprintf("coupon:stock:%d", templateID)

    return s.redis.Set(ctx, stockKey, remaining, 24*time.Hour).Err()
}
```

**优惠券使用（防重复使用）**：
```go
// 使用优惠券
func (s *CouponService) UseCoupon(userID, couponID, orderID int64) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        // 1. 查询优惠券（加锁）
        var coupon UserCoupon
        err := tx.Clauses(clause.Locking{Strength: "UPDATE"}).
            Where("id = ? AND user_id = ?", couponID, userID).
            First(&coupon).Error
        if err != nil {
            return errors.New("优惠券不存在")
        }

        // 2. 检查优惠券状态
        if coupon.Status != "unused" {
            return errors.New("优惠券已使用或已过期")
        }

        // 3. 检查是否过期
        if time.Now().After(coupon.ExpireAt) {
            tx.Model(&coupon).Update("status", "expired")
            return errors.New("优惠券已过期")
        }

        // 4. 查询优惠券模板
        var template CouponTemplate
        if err := tx.First(&template, coupon.TemplateID).Error; err != nil {
            return err
        }

        // 5. 查询订单
        var order Order
        if err := tx.First(&order, orderID).Error; err != nil {
            return err
        }

        // 6. 验证订单金额是否满足最低消费
        if order.Amount < template.MinAmount {
            return fmt.Errorf("订单金额不满足最低消费%.2f元", template.MinAmount)
        }

        // 7. 更新优惠券状态
        now := time.Now()
        err = tx.Model(&UserCoupon{}).
            Where("id = ? AND status = ?", couponID, "unused").
            Updates(map[string]interface{}{
                "status":   "used",
                "order_id": orderID,
                "used_at":  &now,
            }).Error

        if err != nil {
            return err
        }

        // 8. 计算优惠金额
        var discountAmount float64
        if template.Type == "cash" {
            discountAmount = template.Amount
        } else if template.Type == "discount" {
            discountAmount = order.Amount * (1 - template.Discount)
        }

        // 9. 更新订单金额
        finalAmount := order.Amount - discountAmount
        if finalAmount < 0 {
            finalAmount = 0
        }

        err = tx.Model(&Order{}).Where("id = ?", orderID).
            Updates(map[string]interface{}{
                "coupon_id":       couponID,
                "discount_amount": discountAmount,
                "final_amount":    finalAmount,
            }).Error

        return err
    })
}

// 取消订单后释放优惠券
func (s *CouponService) ReleaseCoupon(orderID int64) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        // 查询订单
        var order Order
        if err := tx.First(&order, orderID).Error; err != nil {
            return err
        }

        if order.CouponID == nil {
            return nil // 没有使用优惠券
        }

        // 释放优惠券
        return tx.Model(&UserCoupon{}).
            Where("id = ? AND order_id = ?", *order.CouponID, orderID).
            Updates(map[string]interface{}{
                "status":   "unused",
                "order_id": nil,
                "used_at":  nil,
            }).Error
    })
}
```

**优惠券库存校对**：
```go
// 定时任务：校对优惠券库存
func (s *CouponService) ReconcileStock() {
    var templates []CouponTemplate
    s.db.Where("status = ?", "active").Find(&templates)

    for _, template := range templates {
        // 数据库实际使用数
        var dbUsedCount int64
        s.db.Model(&UserCoupon{}).
            Where("template_id = ?", template.ID).
            Count(&dbUsedCount)

        // Redis库存
        stockKey := fmt.Sprintf("coupon:stock:%d", template.ID)
        redisStock, err := s.redis.Get(ctx, stockKey).Int()

        // 计算差异
        expectedStock := template.TotalCount - int(dbUsedCount)
        if err != nil || redisStock != expectedStock {
            log.Printf("stock mismatch: template=%d, redis=%d, expected=%d",
                template.ID, redisStock, expectedStock)

            // 修正Redis库存
            s.redis.Set(ctx, stockKey, expectedStock, 24*time.Hour)

            // 修正数据库计数
            s.db.Model(&CouponTemplate{}).
                Where("id = ?", template.ID).
                Update("used_count", dbUsedCount)
        }
    }
}
```

**项目应用**：
```
跨境电商优惠券系统：

1. 优惠券类型：
   - 满减券：满100减20
   - 折扣券：8折券
   - 新人券：首单立减
   - 品类券：仅限特定品类

2. 防超发方案：
   - Redis预扣库存（秒杀场景）
   - 数据库行锁+计数器
   - 定时库存校对（每小时）

3. 防重复使用：
   - 优惠券状态机
   - 订单关联（一对一）
   - 取消订单自动释放

4. 性能优化：
   - Redis缓存可用优惠券
   - 异步发券（MQ）
   - 批量发券支持

结果：
- 优惠券发放QPS：1000+
- 0超发事故
- 使用成功率：99.5%
```

---

## 面试准备补充（电商系统详解）

---

## 1. 订单系统详解

### 1.1 订单系统核心概念

**订单系统是电商的核心**，它串联起商品、用户、库存、支付、物流等各个环节。一个好的订单系统需要保证：
- **数据一致性**：订单、库存、支付状态必须一致
- **高并发处理**：秒杀、大促场景下每秒可能有上万订单
- **幂等性**：用户重复提交、网络重试不会产生多个订单
- **可追溯性**：订单的每一步操作都要有日志记录

### 1.2 订单状态机设计

**为什么需要状态机？**
- 明确订单的生命周期
- 防止非法状态转换（如已支付的订单不能取消）
- 便于追踪订单当前状态

**完整的订单状态机**：

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          订单状态机流程图                                  │
└──────────────────────────────────────────────────────────────────────────┘

                           ┌──────────────┐
                           │   待支付     │ (Pending)
                           │ (创建订单)    │
                           └──────┬───────┘
                                  │
                     ┌────────────┼────────────┐
                     │            │            │
                超时15分钟      用户取消      去支付
                     │            │            │
                     ▼            ▼            ▼
              ┌──────────┐  ┌──────────┐  ┌──────────┐
              │ 已取消    │  │ 已取消    │  │ 支付中    │ (Paying)
              └──────────┘  └──────────┘  └────┬─────┘
                                                │
                                      ┌─────────┼─────────┐
                                      │                   │
                                   支付成功            支付失败
                                      │                   │
                                      ▼                   ▼
                                ┌──────────┐        ┌──────────┐
                                │ 待发货    │        │ 已取消    │
                                │ (Paid)   │        └──────────┘
                                └────┬─────┘
                                     │
                                  商家发货
                                     │
                                     ▼
                                ┌──────────┐
                                │ 待收货    │ (Shipped)
                                │ (运输中)  │
                                └────┬─────┘
                                     │
                        ┌────────────┼────────────┐
                        │                         │
                    用户确认收货              申请退款
                        │                         │
                        ▼                         ▼
                   ┌───────────┐             ┌──────────┐
                   │ 已完成     │             │ 退款中    │
                   │(Completed)│             └────┬─────┘
                   └────┬──────┘                  │
                        │                    ┌────┴────┐
                        │                    │         │
                    自动好评               退款成功  退款失败
                        │                    │         │
                        ▼                    ▼         ▼
                   ┌──────────┐        ┌──────────┐ 回到待收货
                   │ 已评价    │        │ 已退款    │
                   └──────────┘        └──────────┘
```

**状态定义（Go代码）**：

```go
type OrderStatus int

const (
    OrderStatusPending   OrderStatus = 1  // 待支付
    OrderStatusPaying    OrderStatus = 2  // 支付中
    OrderStatusPaid      OrderStatus = 3  // 待发货（已支付）
    OrderStatusShipped   OrderStatus = 4  // 待收货（已发货）
    OrderStatusCompleted OrderStatus = 5  // 已完成
    OrderStatusCancelled OrderStatus = 6  // 已取消
    OrderStatusRefunding OrderStatus = 7  // 退款中
    OrderStatusRefunded  OrderStatus = 8  // 已退款
    OrderStatusReviewed  OrderStatus = 9  // 已评价
)

// 状态转换规则
var orderStatusTransitions = map[OrderStatus][]OrderStatus{
    OrderStatusPending: {
        OrderStatusPaying,    // 去支付
        OrderStatusCancelled, // 用户取消/超时取消
    },
    OrderStatusPaying: {
        OrderStatusPaid,      // 支付成功
        OrderStatusCancelled, // 支付失败
    },
    OrderStatusPaid: {
        OrderStatusShipped,   // 商家发货
        OrderStatusRefunding, // 申请退款
    },
    OrderStatusShipped: {
        OrderStatusCompleted, // 用户确认收货
        OrderStatusRefunding, // 申请退款
    },
    OrderStatusCompleted: {
        OrderStatusReviewed,  // 用户评价
        OrderStatusRefunding, // 申请售后
    },
    OrderStatusRefunding: {
        OrderStatusRefunded,  // 退款成功
        OrderStatusShipped,   // 退款失败，回到待收货
    },
}

// 检查状态转换是否合法
func (o *Order) CanTransitionTo(targetStatus OrderStatus) bool {
    allowedStatuses, ok := orderStatusTransitions[o.Status]
    if !ok {
        return false
    }
    for _, status := range allowedStatuses {
        if status == targetStatus {
            return true
        }
    }
    return false
}
```

### 1.3 数据库表设计

**订单主表 (orders)**：

```sql
CREATE TABLE `orders` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '订单ID',
    `order_no` VARCHAR(32) NOT NULL COMMENT '订单号（唯一）',
    `user_id` BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
    `status` TINYINT NOT NULL DEFAULT 1 COMMENT '订单状态：1待支付 2支付中 3待发货 4待收货 5已完成 6已取消 7退款中 8已退款',

    -- 金额相关（单位：分）
    `total_amount` BIGINT NOT NULL COMMENT '订单总金额（分）',
    `discount_amount` BIGINT NOT NULL DEFAULT 0 COMMENT '优惠金额（分）',
    `freight_amount` BIGINT NOT NULL DEFAULT 0 COMMENT '运费（分）',
    `pay_amount` BIGINT NOT NULL COMMENT '实付金额（分）',

    -- 收货信息
    `receiver_name` VARCHAR(50) NOT NULL COMMENT '收货人姓名',
    `receiver_phone` VARCHAR(20) NOT NULL COMMENT '收货人电话',
    `receiver_address` VARCHAR(255) NOT NULL COMMENT '收货地址',

    -- 支付信息
    `pay_channel` VARCHAR(20) DEFAULT NULL COMMENT '支付渠道：alipay/wechat/balance',
    `pay_time` DATETIME DEFAULT NULL COMMENT '支付时间',
    `trade_no` VARCHAR(64) DEFAULT NULL COMMENT '第三方支付交易号',

    -- 时间信息
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    `expired_at` DATETIME DEFAULT NULL COMMENT '过期时间（15分钟后）',
    `shipped_at` DATETIME DEFAULT NULL COMMENT '发货时间',
    `completed_at` DATETIME DEFAULT NULL COMMENT '完成时间',

    -- 其他
    `remark` VARCHAR(255) DEFAULT NULL COMMENT '订单备注',
    `cancel_reason` VARCHAR(255) DEFAULT NULL COMMENT '取消原因',

    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_order_no` (`order_no`),
    KEY `idx_user_id_status` (`user_id`, `status`),
    KEY `idx_created_at` (`created_at`),
    KEY `idx_expired_at` (`expired_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单主表';
```

**订单明细表 (order_items)**：

```sql
CREATE TABLE `order_items` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `order_id` BIGINT UNSIGNED NOT NULL COMMENT '订单ID',
    `order_no` VARCHAR(32) NOT NULL COMMENT '订单号',

    -- 商品信息（冗余，防止商品信息变更）
    `product_id` BIGINT UNSIGNED NOT NULL COMMENT '商品ID',
    `product_name` VARCHAR(255) NOT NULL COMMENT '商品名称',
    `product_image` VARCHAR(512) DEFAULT NULL COMMENT '商品图片',
    `sku_id` BIGINT UNSIGNED NOT NULL COMMENT 'SKU ID',
    `sku_name` VARCHAR(255) DEFAULT NULL COMMENT 'SKU名称（如：红色-XL）',

    -- 价格数量
    `price` BIGINT NOT NULL COMMENT '商品单价（分）',
    `quantity` INT NOT NULL COMMENT '购买数量',
    `total_amount` BIGINT NOT NULL COMMENT '小计金额（分）',

    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (`id`),
    KEY `idx_order_id` (`order_id`),
    KEY `idx_order_no` (`order_no`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单明细表';
```

**订单日志表 (order_logs)**：

```sql
CREATE TABLE `order_logs` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `order_id` BIGINT UNSIGNED NOT NULL,
    `order_no` VARCHAR(32) NOT NULL,
    `old_status` TINYINT DEFAULT NULL COMMENT '变更前状态',
    `new_status` TINYINT NOT NULL COMMENT '变更后状态',
    `operator` VARCHAR(50) DEFAULT NULL COMMENT '操作人（user/system/admin）',
    `remark` VARCHAR(255) DEFAULT NULL COMMENT '备注',
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (`id`),
    KEY `idx_order_id` (`order_id`),
    KEY `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单操作日志';
```

### 1.4 核心功能实现

#### 1.4.1 创建订单（幂等性保证）

**核心问题**：用户可能因为网络问题重复点击"提交订单"，如何保证只创建一个订单？

**解决方案**：前端生成唯一Token，后端使用Token + 分布式锁保证幂等

```go
package service

import (
    "context"
    "errors"
    "fmt"
    "time"

    "github.com/go-redis/redis/v8"
    "gorm.io/gorm"
)

type OrderService struct {
    db    *gorm.DB
    redis *redis.Client
}

// CreateOrderRequest 创建订单请求
type CreateOrderRequest struct {
    UserID      int64              `json:"user_id"`
    Items       []OrderItemRequest `json:"items"`       // 商品列表
    ReceiverInfo ReceiverInfo      `json:"receiver_info"` // 收货信息
    CouponID    int64              `json:"coupon_id"`    // 优惠券ID（可选）
    Remark      string             `json:"remark"`       // 备注
    IdempotentToken string         `json:"idempotent_token"` // 幂等Token（前端生成UUID）
}

type OrderItemRequest struct {
    ProductID int64 `json:"product_id"`
    SkuID     int64 `json:"sku_id"`
    Quantity  int   `json:"quantity"`
}

type ReceiverInfo struct {
    Name    string `json:"name"`
    Phone   string `json:"phone"`
    Address string `json:"address"`
}

// CreateOrder 创建订单（核心方法）
func (s *OrderService) CreateOrder(ctx context.Context, req *CreateOrderRequest) (*Order, error) {
    // 1. 幂等性校验（使用Redis分布式锁）
    lockKey := fmt.Sprintf("order:create:lock:%d:%s", req.UserID, req.IdempotentToken)
    locked, err := s.redis.SetNX(ctx, lockKey, 1, 10*time.Second).Result()
    if err != nil {
        return nil, fmt.Errorf("获取分布式锁失败: %w", err)
    }
    if !locked {
        return nil, errors.New("订单创建中，请勿重复提交")
    }
    defer s.redis.Del(ctx, lockKey) // 释放锁

    // 2. 检查是否已存在订单（通过Token查询）
    cacheKey := fmt.Sprintf("order:token:%s", req.IdempotentToken)
    cachedOrderNo, _ := s.redis.Get(ctx, cacheKey).Result()
    if cachedOrderNo != "" {
        // 已创建过订单，直接返回
        var order Order
        if err := s.db.Where("order_no = ?", cachedOrderNo).First(&order).Error; err != nil {
            return nil, err
        }
        return &order, nil
    }

    // 3. 参数校验
    if err := s.validateCreateOrderRequest(req); err != nil {
        return nil, err
    }

    // 4. 预扣库存（调用库存服务，见后文）
    if err := s.stockService.PreDeduct(ctx, req.Items); err != nil {
        return nil, fmt.Errorf("库存不足: %w", err)
    }

    // 5. 计算订单金额
    orderAmount, err := s.calculateOrderAmount(ctx, req)
    if err != nil {
        // 回滚库存
        s.stockService.RollbackPreDeduct(ctx, req.Items)
        return nil, err
    }

    // 6. 创建订单（数据库事务）
    order, err := s.createOrderInDB(ctx, req, orderAmount)
    if err != nil {
        // 回滚库存
        s.stockService.RollbackPreDeduct(ctx, req.Items)
        return nil, err
    }

    // 7. 缓存订单号（防止重复创建），有效期30分钟
    s.redis.Set(ctx, cacheKey, order.OrderNo, 30*time.Minute)

    // 8. 发送延迟消息（15分钟后检查订单状态，未支付则自动取消）
    s.sendDelayedCancelMessage(ctx, order.OrderNo, 15*time.Minute)

    return order, nil
}

// createOrderInDB 数据库事务创建订单
func (s *OrderService) createOrderInDB(ctx context.Context, req *CreateOrderRequest, amount *OrderAmount) (*Order, error) {
    var order *Order

    err := s.db.Transaction(func(tx *gorm.DB) error {
        // 1. 创建订单主记录
        order = &Order{
            OrderNo:         generateOrderNo(), // 生成唯一订单号
            UserID:          req.UserID,
            Status:          OrderStatusPending,
            TotalAmount:     amount.TotalAmount,
            DiscountAmount:  amount.DiscountAmount,
            FreightAmount:   amount.FreightAmount,
            PayAmount:       amount.PayAmount,
            ReceiverName:    req.ReceiverInfo.Name,
            ReceiverPhone:   req.ReceiverInfo.Phone,
            ReceiverAddress: req.ReceiverInfo.Address,
            Remark:          req.Remark,
            ExpiredAt:       time.Now().Add(15 * time.Minute), // 15分钟支付有效期
        }

        if err := tx.Create(order).Error; err != nil {
            return err
        }

        // 2. 创建订单明细
        for _, item := range req.Items {
            product, _ := s.getProduct(ctx, item.ProductID)
            sku, _ := s.getSku(ctx, item.SkuID)

            orderItem := &OrderItem{
                OrderID:      order.ID,
                OrderNo:      order.OrderNo,
                ProductID:    item.ProductID,
                ProductName:  product.Name,
                ProductImage: product.Image,
                SkuID:        item.SkuID,
                SkuName:      sku.Name,
                Price:        sku.Price,
                Quantity:     item.Quantity,
                TotalAmount:  sku.Price * int64(item.Quantity),
            }

            if err := tx.Create(orderItem).Error; err != nil {
                return err
            }
        }

        // 3. 记录操作日志
        log := &OrderLog{
            OrderID:   order.ID,
            OrderNo:   order.OrderNo,
            OldStatus: nil,
            NewStatus: OrderStatusPending,
            Operator:  fmt.Sprintf("user:%d", req.UserID),
            Remark:    "创建订单",
        }

        if err := tx.Create(log).Error; err != nil {
            return err
        }

        // 4. 如果使用了优惠券，锁定优惠券
        if req.CouponID > 0 {
            if err := s.couponService.LockCoupon(tx, req.CouponID, order.OrderNo); err != nil {
                return err
            }
        }

        return nil
    })

    return order, err
}

// generateOrderNo 生成唯一订单号
func generateOrderNo() string {
    // 格式: 时间戳(14位) + 随机数(4位) + 机器ID(2位)
    // 示例: 20250121150523123456
    timestamp := time.Now().Format("20060102150405")
    random := rand.Intn(10000)
    machineID := getMachineID() // 分布式环境下的机器ID
    return fmt.Sprintf("%s%04d%02d", timestamp, random, machineID)
}
```

#### 1.4.2 订单超时自动取消

**方案对比**：

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 定时任务扫描 | 实现简单 | 延迟高、数据库压力大 | 低并发场景 |
| 延迟队列（RabbitMQ/Redis） | 准时、低开销 | 需要引入MQ | 推荐方案 |
| 时间轮算法 | 性能极高 | 实现复杂、内存占用 | 超高并发场景 |

**推荐方案：Redis延迟队列实现**

```go
// 发送延迟取消消息（使用Redis的ZSet实现延迟队列）
func (s *OrderService) sendDelayedCancelMessage(ctx context.Context, orderNo string, delay time.Duration) error {
    score := float64(time.Now().Add(delay).Unix())
    key := "order:delay:cancel"

    return s.redis.ZAdd(ctx, key, &redis.Z{
        Score:  score,
        Member: orderNo,
    }).Err()
}

// 后台协程：定时扫描并处理到期的订单
func (s *OrderService) StartDelayedCancelWorker(ctx context.Context) {
    ticker := time.NewTicker(1 * time.Second) // 每秒扫描一次
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            s.processPendingCancelOrders(ctx)
        }
    }
}

func (s *OrderService) processPendingCancelOrders(ctx context.Context) {
    key := "order:delay:cancel"
    now := float64(time.Now().Unix())

    // 获取所有到期的订单号（分数 <= 当前时间）
    orderNos, err := s.redis.ZRangeByScore(ctx, key, &redis.ZRangeBy{
        Min:   "0",
        Max:   fmt.Sprintf("%f", now),
        Count: 100, // 每次最多处理100个
    }).Result()

    if err != nil || len(orderNos) == 0 {
        return
    }

    // 批量取消订单
    for _, orderNo := range orderNos {
        // 删除延迟任务
        s.redis.ZRem(ctx, key, orderNo)

        // 异步取消订单（避免阻塞）
        go func(no string) {
            if err := s.CancelOrderIfUnpaid(context.Background(), no); err != nil {
                log.Errorf("自动取消订单失败: %s, error: %v", no, err)
            }
        }(orderNo)
    }
}

// CancelOrderIfUnpaid 取消未支付订单
func (s *OrderService) CancelOrderIfUnpaid(ctx context.Context, orderNo string) error {
    var order Order
    if err := s.db.Where("order_no = ?", orderNo).First(&order).Error; err != nil {
        return err
    }

    // 只有待支付状态才能自动取消
    if order.Status != OrderStatusPending {
        return nil
    }

    return s.db.Transaction(func(tx *gorm.DB) error {
        // 1. 更新订单状态
        if err := tx.Model(&order).Updates(map[string]interface{}{
            "status":        OrderStatusCancelled,
            "cancel_reason": "超时未支付，自动取消",
        }).Error; err != nil {
            return err
        }

        // 2. 回滚预扣库存
        var items []OrderItem
        tx.Where("order_id = ?", order.ID).Find(&items)
        if err := s.stockService.RollbackPreDeduct(ctx, convertToStockItems(items)); err != nil {
            return err
        }

        // 3. 释放优惠券（如果使用了）
        if order.CouponID > 0 {
            s.couponService.UnlockCoupon(tx, order.CouponID)
        }

        // 4. 记录日志
        log := &OrderLog{
            OrderID:   order.ID,
            OrderNo:   order.OrderNo,
            OldStatus: OrderStatusPending,
            NewStatus: OrderStatusCancelled,
            Operator:  "system",
            Remark:    "超时未支付，自动取消",
        }

        return tx.Create(log).Error
    })
}
```

#### 1.4.3 订单拆分逻辑

**什么时候需要拆单？**
1. 多商家场景：用户购买了来自不同商家的商品
2. 多仓库场景：商品分布在不同仓库，需要分别发货
3. 物流限制：部分商品需要特殊物流（如冷链）

**拆单实现**：

```go
// SplitOrder 订单拆分（按商家或仓库）
func (s *OrderService) SplitOrder(ctx context.Context, mainOrder *Order) ([]*Order, error) {
    // 1. 查询订单明细
    var items []OrderItem
    if err := s.db.Where("order_id = ?", mainOrder.ID).Find(&items).Error; err != nil {
        return nil, err
    }

    // 2. 按商家ID分组
    merchantGroups := make(map[int64][]OrderItem)
    for _, item := range items {
        product, _ := s.getProduct(ctx, item.ProductID)
        merchantGroups[product.MerchantID] = append(merchantGroups[product.MerchantID], item)
    }

    // 3. 如果只有一个商家，不需要拆单
    if len(merchantGroups) == 1 {
        return []*Order{mainOrder}, nil
    }

    // 4. 创建子订单
    var subOrders []*Order

    err := s.db.Transaction(func(tx *gorm.DB) error {
        for merchantID, merchantItems := range merchantGroups {
            subOrder := &Order{
                OrderNo:         fmt.Sprintf("%s-%d", mainOrder.OrderNo, merchantID),
                UserID:          mainOrder.UserID,
                ParentOrderNo:   mainOrder.OrderNo, // 关联主订单
                MerchantID:      merchantID,
                Status:          mainOrder.Status,
                ReceiverName:    mainOrder.ReceiverName,
                ReceiverPhone:   mainOrder.ReceiverPhone,
                ReceiverAddress: mainOrder.ReceiverAddress,
            }

            // 计算子订单金额
            var totalAmount int64
            for _, item := range merchantItems {
                totalAmount += item.TotalAmount
            }
            subOrder.TotalAmount = totalAmount
            subOrder.PayAmount = totalAmount // 简化处理，实际需按比例分摊优惠

            // 保存子订单
            if err := tx.Create(subOrder).Error; err != nil {
                return err
            }

            // 保存子订单明细
            for _, item := range merchantItems {
                item.OrderID = subOrder.ID
                item.OrderNo = subOrder.OrderNo
                if err := tx.Create(&item).Error; err != nil {
                    return err
                }
            }

            subOrders = append(subOrders, subOrder)
        }

        // 标记主订单已拆分
        return tx.Model(mainOrder).Update("is_split", true).Error
    })

    return subOrders, err
}
```

### 1.5 面试高频问题

**Q1: 如何保证订单号的唯一性？**

A: 组合方案：
```go
// 订单号生成策略
// 格式: 业务前缀(2位) + 日期(8位) + 时间(6位) + 用户ID(后4位) + 随机数(4位) + 机器ID(2位)
// 示例: OD20250121150523001212340101
func generateOrderNo(userID int64) string {
    prefix := "OD"
    date := time.Now().Format("20060102")
    timeStr := time.Now().Format("150405")
    userSuffix := fmt.Sprintf("%04d", userID%10000)
    random := fmt.Sprintf("%04d", rand.Intn(10000))
    machineID := fmt.Sprintf("%02d", getLocalMachineID())

    return prefix + date + timeStr + userSuffix + random + machineID
}
```

还可以配合数据库唯一索引兜底：
```sql
UNIQUE KEY `uk_order_no` (`order_no`)
```

**Q2: 订单金额如何保证精度？**

A: 使用整数存储（单位：分），避免浮点数精度问题
```go
type Money int64 // 单位：分

func Yuan(yuan float64) Money {
    return Money(yuan * 100) // 元转分
}

func (m Money) Yuan() float64 {
    return float64(m) / 100 // 分转元
}

// 示例
price := Yuan(19.99)  // 1999分
total := price * 3    // 5997分
fmt.Println(total.Yuan()) // 59.97元
```

**Q3: 高并发下如何防止订单重复创建？**

A: 多层防护：
1. **前端防重复点击**：按钮点击后置为loading状态
2. **幂等Token**：前端生成UUID，后端用Token判断重复
3. **分布式锁**：Redis SETNX，确保同一Token只处理一次
4. **数据库唯一索引**：兜底方案

```go
// 完整的幂等性方案
func (s *OrderService) CreateOrderIdempotent(ctx context.Context, req *CreateOrderRequest) (*Order, error) {
    // 1. Token校验
    if req.IdempotentToken == "" {
        return nil, errors.New("缺少幂等Token")
    }

    // 2. 检查缓存（快速返回）
    cacheKey := "order:token:" + req.IdempotentToken
    if orderNo, _ := s.redis.Get(ctx, cacheKey).Result(); orderNo != "" {
        return s.getOrderByNo(ctx, orderNo)
    }

    // 3. 分布式锁（防并发）
    lockKey := "order:lock:" + req.IdempotentToken
    lock := s.redis.SetNX(ctx, lockKey, 1, 10*time.Second)
    if !lock.Val() {
        return nil, errors.New("订单创建中，请勿重复提交")
    }
    defer s.redis.Del(ctx, lockKey)

    // 4. 双重检查（获取锁后再查一次）
    if orderNo, _ := s.redis.Get(ctx, cacheKey).Result(); orderNo != "" {
        return s.getOrderByNo(ctx, orderNo)
    }

    // 5. 创建订单
    order, err := s.createOrder(ctx, req)
    if err != nil {
        return nil, err
    }

    // 6. 缓存订单号（30分钟）
    s.redis.Set(ctx, cacheKey, order.OrderNo, 30*time.Minute)

    return order, nil
}
```

**Q4: 订单取消后如何处理库存和优惠券？**

A: 补偿事务机制：
```go
func (s *OrderService) CancelOrder(ctx context.Context, orderNo string) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        var order Order
        if err := tx.Where("order_no = ?", orderNo).First(&order).Error; err != nil {
            return err
        }

        // 1. 校验状态（只有待支付可以取消）
        if !order.CanTransitionTo(OrderStatusCancelled) {
            return errors.New("当前状态不允许取消")
        }

        // 2. 更新订单状态
        if err := tx.Model(&order).Update("status", OrderStatusCancelled).Error; err != nil {
            return err
        }

        // 3. 回滚库存（补偿）
        var items []OrderItem
        tx.Where("order_id = ?", order.ID).Find(&items)
        for _, item := range items {
            if err := s.stockService.Rollback(ctx, item.SkuID, item.Quantity); err != nil {
                // 库存回滚失败，记录日志，人工介入
                log.Errorf("订单取消回滚库存失败: order=%s, sku=%d, qty=%d",
                    orderNo, item.SkuID, item.Quantity)
                // 发送告警通知
                s.alertService.Send("库存回滚失败", fmt.Sprintf("订单:%s", orderNo))
            }
        }

        // 4. 释放优惠券
        if order.CouponID > 0 {
            s.couponService.Unlock(tx, order.CouponID)
        }

        return nil
    })
}
```

---

## 2. 支付系统详解

### 2.1 支付系统核心概念

**支付系统是电商的资金流转核心**，需要保证：
- **资金安全**：绝对不能出现少收、多收、重复扣款
- **幂等性**：支付回调可能重复，必须幂等处理
- **可扩展性**：支持多种支付渠道（支付宝、微信、银联等）
- **对账**：每日与第三方支付平台对账，发现差异及时处理

**支付业务流程**：

```
┌────────────────────────────────────────────────────────────────────────┐
│                          支付完整流程                                    │
└────────────────────────────────────────────────────────────────────────┘

用户端                业务系统              支付系统           第三方支付
  │                      │                     │                    │
  │  1. 点击支付         │                     │                    │
  ├──────────────────────▶                     │                    │
  │                      │ 2. 创建支付订单     │                    │
  │                      ├─────────────────────▶                    │
  │                      │                     │ 3. 调用支付渠道    │
  │                      │                     ├────────────────────▶
  │                      │                     │ 4. 返回支付凭证    │
  │  5. 展示支付页面     │                     │◀────────────────────
  │◀──────────────────────────────────────────┤                    │
  │                      │                     │                    │
  │  6. 输入密码/指纹    │                     │                    │
  ├────────────────────────────────────────────┼────────────────────▶
  │                      │                     │  7. 支付成功通知   │
  │                      │  8. 异步回调通知    │◀────────────────────
  │                      │◀─────────────────────                    │
  │                      │                     │                    │
  │                      │ 9. 更新订单状态     │                    │
  │                      │    扣减库存等       │                    │
  │  10. 跳转成功页面    │                     │                    │
  │◀──────────────────────                     │                    │
  │                      │                     │                    │
  │                      │ 11. 主动查询（兜底）│                    │
  │                      ├─────────────────────┼────────────────────▶
  │                      │                     │  12. 返回支付结果  │
  │                      │                     │◀────────────────────
```

### 2.2 支付状态机

```
┌──────────────────────────────────────────────────────────────────┐
│                        支付状态流转                                │
└──────────────────────────────────────────────────────────────────┘

                    ┌──────────────┐
                    │   待支付     │ (Pending)
                    │ (创建支付单) │
                    └──────┬───────┘
                           │
             ┌─────────────┼─────────────┐
             │             │             │
          超时取消      用户支付      用户取消
             │             │             │
             ▼             ▼             ▼
        ┌─────────┐  ┌─────────┐  ┌─────────┐
        │ 已关闭  │  │ 支付中  │  │ 已关闭  │
        └─────────┘  └────┬────┘  └─────────┘
                          │
                ┌─────────┼─────────┐
                │                   │
             支付成功            支付失败
                │                   │
                ▼                   ▼
           ┌─────────┐        ┌─────────┐
           │ 已支付  │        │ 已关闭  │
           │(Success)│        └─────────┘
           └────┬────┘
                │
                │ 发生退款
                ▼
           ┌─────────┐
           │ 已退款  │ (Refunded)
           └─────────┘
```

### 2.3 多支付渠道抽象设计

**问题**：如何优雅地支持多个支付渠道（支付宝、微信、银联等）？

**解决方案**：使用适配器模式 + 工厂模式

```go
// PaymentChannel 支付渠道接口（统一抽象）
type PaymentChannel interface {
    // GetName 获取渠道名称
    GetName() string

    // CreatePayment 创建支付
    CreatePayment(ctx context.Context, req *PaymentRequest) (*PaymentResponse, error)

    // QueryPayment 查询支付结果
    QueryPayment(ctx context.Context, tradeNo string) (*PaymentQueryResponse, error)

    // ParseNotify 解析支付回调
    ParseNotify(ctx context.Context, params map[string]string) (*PaymentNotify, error)

    // Refund 退款
    Refund(ctx context.Context, req *RefundRequest) (*RefundResponse, error)
}

// PaymentRequest 支付请求
type PaymentRequest struct {
    OrderNo     string // 订单号
    Amount      int64  // 金额（分）
    Subject     string // 商品标题
    ReturnURL   string // 同步回调地址
    NotifyURL   string // 异步回调地址
    UserID      int64  // 用户ID
}

// PaymentResponse 支付响应
type PaymentResponse struct {
    TradeNo    string // 支付流水号
    PayURL     string // 支付页面URL（跳转支付）
    PayForm    string // 支付表单HTML（自动提交）
    CodeURL    string // 二维码URL（扫码支付）
}

// PaymentNotify 支付回调通知
type PaymentNotify struct {
    TradeNo     string // 第三方交易号
    OrderNo     string // 订单号
    Amount      int64  // 支付金额
    Status      string // 支付状态：success/fail
    PaidAt      time.Time // 支付时间
}
```

**支付宝实现**：

```go
package channel

import (
    "context"
    "github.com/smartwalle/alipay/v3"
)

type AlipayChannel struct {
    client *alipay.Client
}

func NewAlipayChannel(appID, privateKey, alipayPublicKey string) *AlipayChannel {
    client, _ := alipay.New(appID, privateKey, false)
    client.LoadAliPayPublicKey(alipayPublicKey)

    return &AlipayChannel{client: client}
}

func (a *AlipayChannel) GetName() string {
    return "alipay"
}

func (a *AlipayChannel) CreatePayment(ctx context.Context, req *PaymentRequest) (*PaymentResponse, error) {
    // 1. 构造支付宝请求参数
    p := alipay.TradePagePay{}
    p.OutTradeNo = req.OrderNo
    p.TotalAmount = fmt.Sprintf("%.2f", float64(req.Amount)/100) // 分转元
    p.Subject = req.Subject
    p.ReturnURL = req.ReturnURL
    p.NotifyURL = req.NotifyURL

    // 2. 调用支付宝API
    payURL, err := a.client.TradePagePay(p)
    if err != nil {
        return nil, fmt.Errorf("支付宝下单失败: %w", err)
    }

    return &PaymentResponse{
        TradeNo: req.OrderNo, // 支付宝会在回调中返回真实trade_no
        PayURL:  payURL.String(),
    }, nil
}

func (a *AlipayChannel) QueryPayment(ctx context.Context, orderNo string) (*PaymentQueryResponse, error) {
    p := alipay.TradeQuery{}
    p.OutTradeNo = orderNo

    resp, err := a.client.TradeQuery(p)
    if err != nil {
        return nil, err
    }

    status := "pending"
    if resp.TradeStatus == "TRADE_SUCCESS" || resp.TradeStatus == "TRADE_FINISHED" {
        status = "success"
    } else if resp.TradeStatus == "TRADE_CLOSED" {
        status = "closed"
    }

    amount, _ := strconv.ParseFloat(resp.TotalAmount, 64)

    return &PaymentQueryResponse{
        OrderNo:    resp.OutTradeNo,
        TradeNo:    resp.TradeNo,
        Amount:     int64(amount * 100), // 元转分
        Status:     status,
        TradeStatus: resp.TradeStatus,
    }, nil
}

func (a *AlipayChannel) ParseNotify(ctx context.Context, params map[string]string) (*PaymentNotify, error) {
    // 1. 验签
    if err := a.client.VerifySign(params); err != nil {
        return nil, fmt.Errorf("支付宝回调验签失败: %w", err)
    }

    // 2. 解析参数
    tradeStatus := params["trade_status"]
    status := "fail"
    if tradeStatus == "TRADE_SUCCESS" || tradeStatus == "TRADE_FINISHED" {
        status = "success"
    }

    amount, _ := strconv.ParseFloat(params["total_amount"], 64)
    paidAt, _ := time.Parse("2006-01-02 15:04:05", params["gmt_payment"])

    return &PaymentNotify{
        TradeNo: params["trade_no"],
        OrderNo: params["out_trade_no"],
        Amount:  int64(amount * 100),
        Status:  status,
        PaidAt:  paidAt,
    }, nil
}

func (a *AlipayChannel) Refund(ctx context.Context, req *RefundRequest) (*RefundResponse, error) {
    p := alipay.TradeRefund{}
    p.OutTradeNo = req.OrderNo
    p.RefundAmount = fmt.Sprintf("%.2f", float64(req.Amount)/100)
    p.RefundReason = req.Reason
    p.OutRequestNo = req.RefundNo // 退款单号

    resp, err := a.client.TradeRefund(p)
    if err != nil {
        return nil, err
    }

    return &RefundResponse{
        RefundNo:  req.RefundNo,
        TradeNo:   resp.TradeNo,
        Success:   resp.Code == alipay.CodeSuccess,
        Message:   resp.Msg,
    }, nil
}
```

**微信支付实现**（类似结构，略）

**支付渠道工厂**：

```go
type PaymentChannelFactory struct {
    channels map[string]PaymentChannel
}

func NewPaymentChannelFactory() *PaymentChannelFactory {
    factory := &PaymentChannelFactory{
        channels: make(map[string]PaymentChannel),
    }

    // 注册所有支付渠道
    factory.Register(NewAlipayChannel(config.Alipay.AppID, config.Alipay.PrivateKey, config.Alipay.PublicKey))
    factory.Register(NewWechatChannel(config.Wechat.AppID, config.Wechat.MchID, config.Wechat.APIKey))
    factory.Register(NewUnionPayChannel(config.UnionPay.MerchantID, config.UnionPay.CertPath))

    return factory
}

func (f *PaymentChannelFactory) Register(channel PaymentChannel) {
    f.channels[channel.GetName()] = channel
}

func (f *PaymentChannelFactory) GetChannel(name string) (PaymentChannel, error) {
    channel, ok := f.channels[name]
    if !ok {
        return nil, fmt.Errorf("不支持的支付渠道: %s", name)
    }
    return channel, nil
}
```

### 2.4 数据库表设计

**支付订单表 (payments)**：

```sql
CREATE TABLE `payments` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `payment_no` VARCHAR(32) NOT NULL COMMENT '支付单号（唯一）',
    `order_no` VARCHAR(32) NOT NULL COMMENT '订单号',
    `user_id` BIGINT UNSIGNED NOT NULL COMMENT '用户ID',

    -- 支付信息
    `channel` VARCHAR(20) NOT NULL COMMENT '支付渠道：alipay/wechat/unionpay',
    `amount` BIGINT NOT NULL COMMENT '支付金额（分）',
    `status` TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1待支付 2支付中 3已支付 4已关闭 5已退款',

    -- 第三方信息
    `trade_no` VARCHAR(64) DEFAULT NULL COMMENT '第三方交易号',
    `notify_data` TEXT DEFAULT NULL COMMENT '回调原始数据（JSON）',

    -- 时间信息
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    `paid_at` DATETIME DEFAULT NULL COMMENT '支付完成时间',
    `expired_at` DATETIME DEFAULT NULL COMMENT '过期时间',

    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_payment_no` (`payment_no`),
    UNIQUE KEY `uk_trade_no` (`trade_no`),
    KEY `idx_order_no` (`order_no`),
    KEY `idx_user_id` (`user_id`),
    KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='支付订单表';
```

**支付回调日志表 (payment_notify_logs)**：

```sql
CREATE TABLE `payment_notify_logs` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `payment_no` VARCHAR(32) NOT NULL COMMENT '支付单号',
    `channel` VARCHAR(20) NOT NULL COMMENT '支付渠道',
    `notify_data` TEXT NOT NULL COMMENT '回调数据（JSON）',
    `verify_result` TINYINT NOT NULL COMMENT '验签结果：1成功 0失败',
    `handle_result` TINYINT NOT NULL COMMENT '处理结果：1成功 0失败',
    `error_msg` VARCHAR(255) DEFAULT NULL COMMENT '错误信息',
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (`id`),
    KEY `idx_payment_no` (`payment_no`),
    KEY `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='支付回调日志';
```

### 2.5 支付核心流程实现

#### 2.5.1 创建支付

```go
type PaymentService struct {
    db      *gorm.DB
    redis   *redis.Client
    factory *PaymentChannelFactory
    mq      MessageQueue
}

// CreatePayment 创建支付订单
func (s *PaymentService) CreatePayment(ctx context.Context, orderNo string, channel string) (*Payment, *PaymentResponse, error) {
    // 1. 查询订单信息
    var order Order
    if err := s.db.Where("order_no = ?", orderNo).First(&order).Error; err != nil {
        return nil, nil, fmt.Errorf("订单不存在: %w", err)
    }

    // 2. 校验订单状态（只有待支付可以支付）
    if order.Status != OrderStatusPending {
        return nil, nil, errors.New("订单状态不允许支付")
    }

    // 3. 检查是否已存在支付单
    var existPayment Payment
    if err := s.db.Where("order_no = ? AND status IN (1,2,3)", orderNo).First(&existPayment).Error; err == nil {
        // 已存在未完成的支付单，直接返回
        payChannel, _ := s.factory.GetChannel(existPayment.Channel)
        resp, _ := s.getPaymentResponse(ctx, &existPayment, payChannel)
        return &existPayment, resp, nil
    }

    // 4. 创建支付订单
    payment := &Payment{
        PaymentNo: generatePaymentNo(),
        OrderNo:   orderNo,
        UserID:    order.UserID,
        Channel:   channel,
        Amount:    order.PayAmount,
        Status:    PaymentStatusPending,
        ExpiredAt: time.Now().Add(15 * time.Minute),
    }

    if err := s.db.Create(payment).Error; err != nil {
        return nil, nil, err
    }

    // 5. 调用支付渠道
    payChannel, err := s.factory.GetChannel(channel)
    if err != nil {
        return nil, nil, err
    }

    payReq := &PaymentRequest{
        OrderNo:   payment.PaymentNo,
        Amount:    payment.Amount,
        Subject:   fmt.Sprintf("订单支付-%s", orderNo),
        ReturnURL: config.Payment.ReturnURL,
        NotifyURL: config.Payment.NotifyURL,
        UserID:    order.UserID,
    }

    payResp, err := payChannel.CreatePayment(ctx, payReq)
    if err != nil {
        // 更新支付单状态为失败
        s.db.Model(payment).Update("status", PaymentStatusClosed)
        return nil, nil, fmt.Errorf("调用支付渠道失败: %w", err)
    }

    // 6. 更新支付单状态为支付中
    s.db.Model(payment).Updates(map[string]interface{}{
        "status":   PaymentStatusPaying,
        "trade_no": payResp.TradeNo,
    })

    return payment, payResp, nil
}

// generatePaymentNo 生成支付单号
func generatePaymentNo() string {
    // 格式: PAY + 时间戳(14位) + 随机数(6位)
    timestamp := time.Now().Format("20060102150405")
    random := rand.Intn(1000000)
    return fmt.Sprintf("PAY%s%06d", timestamp, random)
}
```

#### 2.5.2 处理支付回调（核心难点：幂等性）

**关键问题**：
1. 支付平台会多次发送回调通知（直到收到success响应）
2. 网络抖动可能导致重复通知
3. 必须保证幂等：同一笔支付只能被处理一次

**解决方案**：分布式锁 + 状态机 + 日志记录

```go
// HandleNotify 处理支付回调（幂等）
func (s *PaymentService) HandleNotify(ctx context.Context, channel string, params map[string]string) error {
    // 1. 获取支付渠道
    payChannel, err := s.factory.GetChannel(channel)
    if err != nil {
        return err
    }

    // 2. 解析并验签
    notify, err := payChannel.ParseNotify(ctx, params)
    if err != nil {
        // 记录验签失败日志
        s.saveNotifyLog(&PaymentNotifyLog{
            Channel:      channel,
            NotifyData:   toJSON(params),
            VerifyResult: 0,
            HandleResult: 0,
            ErrorMsg:     err.Error(),
        })
        return fmt.Errorf("回调验签失败: %w", err)
    }

    // 3. 查询支付单
    var payment Payment
    if err := s.db.Where("payment_no = ?", notify.OrderNo).First(&payment).Error; err != nil {
        return fmt.Errorf("支付单不存在: %w", err)
    }

    // 4. 幂等性检查（快速返回）
    if payment.Status == PaymentStatusPaid {
        // 已经支付成功，直接返回成功（幂等）
        log.Infof("支付回调重复通知，幂等返回: payment_no=%s", payment.PaymentNo)
        return nil
    }

    // 5. 分布式锁（防止并发处理）
    lockKey := fmt.Sprintf("payment:notify:lock:%s", payment.PaymentNo)
    locked, err := s.redis.SetNX(ctx, lockKey, 1, 30*time.Second).Result()
    if err != nil {
        return err
    }
    if !locked {
        // 正在处理中，直接返回成功（幂等）
        return nil
    }
    defer s.redis.Del(ctx, lockKey)

    // 6. 双重检查（获取锁后再查一次）
    if err := s.db.Where("payment_no = ?", notify.OrderNo).First(&payment).Error; err != nil {
        return err
    }
    if payment.Status == PaymentStatusPaid {
        return nil // 幂等返回
    }

    // 7. 金额校验（防止篡改）
    if notify.Amount != payment.Amount {
        errMsg := fmt.Sprintf("回调金额不匹配: 期望=%d, 实际=%d", payment.Amount, notify.Amount)
        s.saveNotifyLog(&PaymentNotifyLog{
            PaymentNo:    payment.PaymentNo,
            Channel:      channel,
            NotifyData:   toJSON(params),
            VerifyResult: 1,
            HandleResult: 0,
            ErrorMsg:     errMsg,
        })
        // 发送告警
        s.alertService.Send("支付金额异常", errMsg)
        return errors.New(errMsg)
    }

    // 8. 处理支付成功逻辑（数据库事务）
    if notify.Status == "success" {
        if err := s.handlePaymentSuccess(ctx, &payment, notify); err != nil {
            // 记录失败日志
            s.saveNotifyLog(&PaymentNotifyLog{
                PaymentNo:    payment.PaymentNo,
                Channel:      channel,
                NotifyData:   toJSON(params),
                VerifyResult: 1,
                HandleResult: 0,
                ErrorMsg:     err.Error(),
            })
            return err
        }
    }

    // 9. 记录成功日志
    s.saveNotifyLog(&PaymentNotifyLog{
        PaymentNo:    payment.PaymentNo,
        Channel:      channel,
        NotifyData:   toJSON(params),
        VerifyResult: 1,
        HandleResult: 1,
    })

    return nil
}

// handlePaymentSuccess 处理支付成功（事务）
func (s *PaymentService) handlePaymentSuccess(ctx context.Context, payment *Payment, notify *PaymentNotify) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        // 1. 更新支付单状态
        if err := tx.Model(payment).Updates(map[string]interface{}{
            "status":      PaymentStatusPaid,
            "trade_no":    notify.TradeNo,
            "paid_at":     notify.PaidAt,
            "notify_data": toJSON(notify),
        }).Error; err != nil {
            return err
        }

        // 2. 更新订单状态（待支付 -> 待发货）
        if err := tx.Model(&Order{}).Where("order_no = ?", payment.OrderNo).Updates(map[string]interface{}{
            "status":       OrderStatusPaid,
            "pay_channel":  payment.Channel,
            "pay_time":     notify.PaidAt,
            "trade_no":     notify.TradeNo,
        }).Error; err != nil {
            return err
        }

        // 3. 记录订单日志
        orderLog := &OrderLog{
            OrderNo:   payment.OrderNo,
            OldStatus: OrderStatusPending,
            NewStatus: OrderStatusPaid,
            Operator:  "system",
            Remark:    fmt.Sprintf("支付成功，支付渠道:%s", payment.Channel),
        }
        if err := tx.Create(orderLog).Error; err != nil {
            return err
        }

        // 4. 扣减真实库存（之前是预扣）
        var order Order
        tx.Where("order_no = ?", payment.OrderNo).First(&order)
        var items []OrderItem
        tx.Where("order_id = ?", order.ID).Find(&items)

        for _, item := range items {
            if err := s.stockService.ConfirmDeduct(ctx, item.SkuID, item.Quantity); err != nil {
                // 库存确认失败，发送告警
                log.Errorf("支付成功但库存确认失败: order=%s, sku=%d", payment.OrderNo, item.SkuID)
                s.alertService.Send("库存确认失败", fmt.Sprintf("订单:%s", payment.OrderNo))
            }
        }

        // 5. 消费优惠券（标记已使用）
        if order.CouponID > 0 {
            s.couponService.Use(tx, order.CouponID)
        }

        // 6. 发送MQ消息（通知其他系统）
        s.mq.Publish("order.paid", map[string]interface{}{
            "order_no":   payment.OrderNo,
            "user_id":    payment.UserID,
            "amount":     payment.Amount,
            "paid_at":    notify.PaidAt,
        })

        return nil
    })
}
```

#### 2.5.3 主动查询支付结果（兜底方案）

**为什么需要主动查询？**
- 支付回调可能丢失（网络问题、服务宕机）
- 用户支付后长时间未收到回调
- 兜底保障，确保最终一致性

```go
// QueryPaymentStatus 主动查询支付结果
func (s *PaymentService) QueryPaymentStatus(ctx context.Context, paymentNo string) (*Payment, error) {
    // 1. 查询本地支付单
    var payment Payment
    if err := s.db.Where("payment_no = ?", paymentNo).First(&payment).Error; err != nil {
        return nil, err
    }

    // 2. 如果已经支付成功，直接返回
    if payment.Status == PaymentStatusPaid {
        return &payment, nil
    }

    // 3. 调用支付渠道查询
    payChannel, err := s.factory.GetChannel(payment.Channel)
    if err != nil {
        return nil, err
    }

    queryResp, err := payChannel.QueryPayment(ctx, payment.PaymentNo)
    if err != nil {
        return nil, err
    }

    // 4. 根据查询结果更新状态
    if queryResp.Status == "success" {
        // 模拟回调通知，复用回调处理逻辑
        notify := &PaymentNotify{
            TradeNo: queryResp.TradeNo,
            OrderNo: payment.PaymentNo,
            Amount:  queryResp.Amount,
            Status:  "success",
            PaidAt:  time.Now(),
        }

        if err := s.handlePaymentSuccess(ctx, &payment, notify); err != nil {
            return nil, err
        }

        // 重新查询
        s.db.Where("payment_no = ?", paymentNo).First(&payment)
    } else if queryResp.Status == "closed" {
        // 支付已关闭
        s.db.Model(&payment).Update("status", PaymentStatusClosed)
    }

    return &payment, nil
}

// StartQueryWorker 启动定时查询任务（兜底）
func (s *PaymentService) StartQueryWorker(ctx context.Context) {
    ticker := time.NewTicker(1 * time.Minute) // 每分钟执行一次
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            s.queryPendingPayments(ctx)
        }
    }
}

func (s *PaymentService) queryPendingPayments(ctx context.Context) {
    // 查询所有支付中且创建时间 > 5分钟的支付单
    var payments []Payment
    s.db.Where("status = ? AND created_at < ?", PaymentStatusPaying, time.Now().Add(-5*time.Minute)).
        Limit(100).
        Find(&payments)

    for _, payment := range payments {
        go func(p Payment) {
            if _, err := s.QueryPaymentStatus(context.Background(), p.PaymentNo); err != nil {
                log.Errorf("主动查询支付失败: payment_no=%s, error=%v", p.PaymentNo, err)
            }
        }(payment)
    }
}
```

### 2.6 对账机制

**为什么需要对账？**
- 防止数据不一致（本地已支付，第三方未支付或反之）
- 发现异常交易（金额篡改、重复支付等）
- 合规要求

**对账流程**：

```
┌────────────────────────────────────────────────────────────────┐
│                        每日对账流程                              │
└────────────────────────────────────────────────────────────────┘

凌晨2点（定时任务）
  │
  ├──▶ 1. 下载第三方对账单（支付宝/微信）
  │       - 下载昨日账单文件（CSV/Excel）
  │       - 解析账单数据
  │
  ├──▶ 2. 生成本地对账单
  │       - 查询本地昨日所有已支付订单
  │       - 生成对账数据
  │
  ├──▶ 3. 数据比对
  │       - 订单号匹配
  │       - 金额校验
  │       - 状态校验
  │
  ├──▶ 4. 差异处理
  │       - 本地有，第三方没有 → 主动查询确认
  │       - 第三方有，本地没有 → 补单处理
  │       - 金额不一致 → 人工介入
  │
  └──▶ 5. 生成对账报告
          - 对账总数
          - 成功数
          - 差异数
          - 差异明细
```

**对账实现**：

```go
type ReconciliationService struct {
    db      *gorm.DB
    factory *PaymentChannelFactory
}

// DailyReconciliation 每日对账
func (s *ReconciliationService) DailyReconciliation(ctx context.Context, date time.Time) (*ReconciliationReport, error) {
    report := &ReconciliationReport{
        Date:     date,
        StartAt:  time.Now(),
    }

    // 1. 下载第三方对账单
    channels := []string{"alipay", "wechat"}
    for _, channelName := range channels {
        channel, _ := s.factory.GetChannel(channelName)

        // 下载对账文件
        billData, err := s.downloadBill(ctx, channel, date)
        if err != nil {
            log.Errorf("下载对账单失败: channel=%s, error=%v", channelName, err)
            continue
        }

        // 解析对账数据
        thirdPartyRecords := s.parseBillData(channelName, billData)

        // 2. 查询本地对账数据
        localRecords := s.getLocalRecords(date, channelName)

        // 3. 数据比对
        diff := s.compareRecords(localRecords, thirdPartyRecords)

        // 4. 处理差异
        s.handleDifferences(ctx, diff)

        // 5. 汇总报告
        report.TotalCount += len(localRecords)
        report.SuccessCount += len(localRecords) - len(diff)
        report.DiffCount += len(diff)
        report.Differences = append(report.Differences, diff...)
    }

    report.FinishAt = time.Now()

    // 6. 保存对账报告
    s.saveReport(report)

    // 7. 发送告警（如果有差异）
    if report.DiffCount > 0 {
        s.alertService.Send("对账发现差异", fmt.Sprintf("日期:%s, 差异数:%d", date.Format("2006-01-02"), report.DiffCount))
    }

    return report, nil
}

// 比对记录
func (s *ReconciliationService) compareRecords(local, third []ReconciliationRecord) []ReconciliationDiff {
    var diffs []ReconciliationDiff

    // 构建第三方记录Map（key: 订单号）
    thirdMap := make(map[string]ReconciliationRecord)
    for _, record := range third {
        thirdMap[record.OrderNo] = record
    }

    // 遍历本地记录，查找差异
    for _, localRecord := range local {
        thirdRecord, exists := thirdMap[localRecord.OrderNo]

        if !exists {
            // 本地有，第三方没有
            diffs = append(diffs, ReconciliationDiff{
                Type:        "missing_in_third",
                OrderNo:     localRecord.OrderNo,
                LocalAmount: localRecord.Amount,
            })
            continue
        }

        // 金额不一致
        if localRecord.Amount != thirdRecord.Amount {
            diffs = append(diffs, ReconciliationDiff{
                Type:         "amount_mismatch",
                OrderNo:      localRecord.OrderNo,
                LocalAmount:  localRecord.Amount,
                ThirdAmount:  thirdRecord.Amount,
            })
        }

        // 从Map中删除已匹配的
        delete(thirdMap, localRecord.OrderNo)
    }

    // 第三方有，本地没有
    for orderNo, thirdRecord := range thirdMap {
        diffs = append(diffs, ReconciliationDiff{
            Type:        "missing_in_local",
            OrderNo:     orderNo,
            ThirdAmount: thirdRecord.Amount,
        })
    }

    return diffs
}
```

### 2.7 面试高频问题

**Q1: 如何保证支付回调的幂等性？**

A: 多层防护机制：
1. **状态机校验**：已支付状态不可重复处理
2. **分布式锁**：使用Redis SETNX锁住支付单号
3. **数据库唯一约束**：trade_no唯一索引
4. **日志记录**：记录每次回调，便于排查

**Q2: 支付回调丢失怎么办？**

A: 兜底方案：
1. **主动查询**：用户支付后前端轮询查询支付状态
2. **定时任务**：后台定时扫描支付中的订单，主动查询结果
3. **重试机制**：第三方平台会多次发送回调（最多10次）

**Q3: 如何设计支付系统的数据库？**

A: 核心表设计：
- **payments**: 支付订单主表
- **payment_notify_logs**: 回调日志（审计）
- **payment_refunds**: 退款记录
- **payment_reconciliation**: 对账记录

**Q4: 如何防止支付金额被篡改？**

A: 多重校验：
1. **后端计算金额**：前端传的金额只做展示，后端重新计算
2. **回调金额校验**：对比本地金额和回调金额
3. **签名验证**：验证第三方回调签名
4. **日志记录**：记录金额变更历史

**Q5: 多渠道支付如何设计？**

A: 适配器模式 + 工厂模式：
1. 定义统一的支付接口(PaymentChannel)
2. 每个渠道实现该接口
3. 工厂类负责创建和管理渠道实例
4. 业务层只依赖抽象接口，不依赖具体实现

---

## 3. 库存系统详解

### 3.1 库存系统核心概念

**库存系统是防止超卖的核心**，需要保证：
- **准确性**：库存数据必须准确，不能超卖也不能少卖
- **并发安全**：高并发下的原子性操作
- **性能**：扣减库存是高频操作，需要高性能
- **最终一致性**：预扣库存与真实库存的对账

**超卖问题示例**：

```
场景：商品库存10个，同时有20个用户下单

错误实现（会超卖）:
User1: 查询库存=10  ✓
User2: 查询库存=10  ✓
User1: 扣减库存 10-1=9  ✓
User2: 扣减库存 10-1=9  ✓ （应该是8，但并发导致数据错误）
...
最终可能卖出15个，超卖5个！
```

### 3.2 超卖问题解决方案对比

| 方案 | 原理 | 优点 | 缺点 | 适用场景 |
|------|------|------|------|----------|
| 悲观锁(FOR UPDATE) | 数据库行锁 | 绝对安全 | 性能差、易死锁 | 低并发 |
| 乐观锁(版本号) | CAS机制 | 无锁、高性能 | 高并发下失败率高 | 中等并发 |
| Redis单线程 | 串行化处理 | 高性能 | Redis故障影响大 | 高并发 |
| Redis+Lua | 原子脚本 | 高性能+原子性 | 实现复杂 | **推荐方案** |
| 消息队列削峰 | 异步处理 | 削峰填谷 | 实时性差 | 秒杀场景 |

### 3.3 数据库表设计

**商品库存表 (product_stocks)**：

```sql
CREATE TABLE `product_stocks` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `sku_id` BIGINT UNSIGNED NOT NULL COMMENT 'SKU ID',
    `warehouse_id` BIGINT UNSIGNED DEFAULT 1 COMMENT '仓库ID',
    
    -- 库存数量
    `total_stock` INT NOT NULL DEFAULT 0 COMMENT '总库存',
    `available_stock` INT NOT NULL DEFAULT 0 COMMENT '可用库存',
    `locked_stock` INT NOT NULL DEFAULT 0 COMMENT '锁定库存（预扣）',
    `sold_stock` INT NOT NULL DEFAULT 0 COMMENT '已售库存',
    
    -- 版本号（乐观锁）
    `version` INT NOT NULL DEFAULT 0 COMMENT '版本号',
    
    -- 时间
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_sku_warehouse` (`sku_id`, `warehouse_id`),
    KEY `idx_sku_id` (`sku_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品库存表';

-- 数据约束：available_stock + locked_stock + sold_stock = total_stock
```

**库存操作日志表 (stock_logs)**：

```sql
CREATE TABLE `stock_logs` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `sku_id` BIGINT UNSIGNED NOT NULL,
    `order_no` VARCHAR(32) DEFAULT NULL COMMENT '订单号',
    `operation_type` VARCHAR(20) NOT NULL COMMENT '操作类型：lock/unlock/deduct/add',
    `quantity` INT NOT NULL COMMENT '数量（正数或负数）',
    `before_stock` INT NOT NULL COMMENT '操作前库存',
    `after_stock` INT NOT NULL COMMENT '操作后库存',
    `operator` VARCHAR(50) DEFAULT NULL COMMENT '操作人',
    `remark` VARCHAR(255) DEFAULT NULL,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    PRIMARY KEY (`id`),
    KEY `idx_sku_id` (`sku_id`),
    KEY `idx_order_no` (`order_no`),
    KEY `idx_created_at` (`created_at`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='库存操作日志';
```

### 3.4 方案一：数据库悲观锁（FOR UPDATE）

**原理**：使用数据库行锁，锁住库存记录

```go
// DeductStockWithPessimisticLock 悲观锁扣减库存
func (s *StockService) DeductStockWithPessimisticLock(ctx context.Context, skuID int64, quantity int) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        // 1. 锁定库存记录（FOR UPDATE）
        var stock ProductStock
        if err := tx.Where("sku_id = ?", skuID).
            Clauses(clause.Locking{Strength: "UPDATE"}).
            First(&stock).Error; err != nil {
            return err
        }
        
        // 2. 检查库存是否充足
        if stock.AvailableStock < quantity {
            return errors.New("库存不足")
        }
        
        // 3. 扣减库存
        if err := tx.Model(&stock).Updates(map[string]interface{}{
            "available_stock": gorm.Expr("available_stock - ?", quantity),
            "sold_stock":      gorm.Expr("sold_stock + ?", quantity),
        }).Error; err != nil {
            return err
        }
        
        // 4. 记录日志
        log := &StockLog{
            SkuID:        skuID,
            OperationType: "deduct",
            Quantity:     -quantity,
            BeforeStock:  stock.AvailableStock,
            AfterStock:   stock.AvailableStock - quantity,
        }
        
        return tx.Create(log).Error
    })
}
```

**优点**：实现简单，绝对安全
**缺点**：性能差（每秒只能处理几百个请求），容易死锁

### 3.5 方案二：数据库乐观锁（版本号）

**原理**：使用版本号（version）实现CAS，更新时检查版本号

```go
// DeductStockWithOptimisticLock 乐观锁扣减库存
func (s *StockService) DeductStockWithOptimisticLock(ctx context.Context, skuID int64, quantity int) error {
    maxRetry := 3 // 最多重试3次
    
    for i := 0; i < maxRetry; i++ {
        // 1. 查询当前库存
        var stock ProductStock
        if err := s.db.Where("sku_id = ?", skuID).First(&stock).Error; err != nil {
            return err
        }
        
        // 2. 检查库存
        if stock.AvailableStock < quantity {
            return errors.New("库存不足")
        }
        
        // 3. 乐观锁更新（WHERE version = 旧版本号）
        oldVersion := stock.Version
        result := s.db.Model(&ProductStock{}).
            Where("sku_id = ? AND version = ?", skuID, oldVersion).
            Updates(map[string]interface{}{
                "available_stock": gorm.Expr("available_stock - ?", quantity),
                "sold_stock":      gorm.Expr("sold_stock + ?", quantity),
                "version":         gorm.Expr("version + 1"),
            })
        
        if result.Error != nil {
            return result.Error
        }
        
        // 4. 判断是否更新成功
        if result.RowsAffected > 0 {
            // 更新成功，记录日志
            s.saveStockLog(skuID, "deduct", -quantity, stock.AvailableStock)
            return nil
        }
        
        // 5. 更新失败（版本号不匹配），重试
        log.Warnf("乐观锁冲突，重试: sku=%d, attempt=%d", skuID, i+1)
        time.Sleep(time.Millisecond * time.Duration(rand.Intn(10))) // 随机退避
    }
    
    return errors.New("库存扣减失败，请重试")
}
```

**优点**：无锁，性能较好
**缺点**：高并发下冲突率高，需要重试

### 3.6 方案三：Redis + Lua原子扣减（推荐）

**原理**：利用Redis单线程特性 + Lua脚本原子性

**为什么选择Redis + Lua？**
1. Redis单线程，天然串行化，无并发问题
2. Lua脚本在Redis中原子执行，不会被打断
3. 性能极高（QPS可达10万+）
4. 简单可靠

**实现步骤**：
```
1. 将库存同步到Redis（SKU_ID -> 库存数量）
2. 使用Lua脚本原子扣减
3. 异步同步回MySQL（最终一致性）
```

**Lua扣减脚本**：

```lua
-- deduct_stock.lua
-- KEYS[1]: 库存key（如 stock:sku:12345）
-- ARGV[1]: 扣减数量

local stock = redis.call('GET', KEYS[1])

-- 库存不存在
if not stock then
    return -1
end

stock = tonumber(stock)
local quantity = tonumber(ARGV[1])

-- 库存不足
if stock < quantity then
    return -2
end

-- 扣减库存
redis.call('DECRBY', KEYS[1], quantity)

-- 返回剩余库存
return stock - quantity
```

**Go代码实现**：

```go
package service

import (
    "context"
    "fmt"
    _ "embed"
    
    "github.com/go-redis/redis/v8"
)

//go:embed scripts/deduct_stock.lua
var deductStockScript string

type StockService struct {
    db    *gorm.DB
    redis *redis.Client
}

// InitStockCache 初始化库存缓存（从MySQL同步到Redis）
func (s *StockService) InitStockCache(ctx context.Context) error {
    var stocks []ProductStock
    if err := s.db.Find(&stocks).Error; err != nil {
        return err
    }
    
    pipe := s.redis.Pipeline()
    for _, stock := range stocks {
        key := fmt.Sprintf("stock:sku:%d", stock.SkuID)
        pipe.Set(ctx, key, stock.AvailableStock, 0)
    }
    
    _, err := pipe.Exec(ctx)
    return err
}

// DeductStock 扣减库存（Redis + Lua）
func (s *StockService) DeductStock(ctx context.Context, skuID int64, quantity int) error {
    key := fmt.Sprintf("stock:sku:%d", skuID)
    
    // 执行Lua脚本
    result, err := s.redis.Eval(ctx, deductStockScript, []string{key}, quantity).Int()
    if err != nil {
        return fmt.Errorf("扣减库存失败: %w", err)
    }
    
    if result == -1 {
        return errors.New("商品不存在")
    }
    if result == -2 {
        return errors.New("库存不足")
    }
    
    // 异步同步到MySQL（MQ）
    s.syncStockToMySQL(skuID, -quantity)
    
    return nil
}

// syncStockToMySQL 异步同步到MySQL
func (s *StockService) syncStockToMySQL(skuID int64, delta int) {
    // 发送MQ消息
    s.mq.Publish("stock.change", map[string]interface{}{
        "sku_id":   skuID,
        "delta":    delta,
        "op_type":  "deduct",
    })
}

// SyncStockWorker 消费MQ，同步到MySQL
func (s *StockService) SyncStockWorker(ctx context.Context) {
    s.mq.Subscribe("stock.change", func(msg StockChangeMessage) {
        err := s.db.Model(&ProductStock{}).
            Where("sku_id = ?", msg.SkuID).
            Updates(map[string]interface{}{
                "available_stock": gorm.Expr("available_stock + ?", msg.Delta),
                "sold_stock":      gorm.Expr("sold_stock - ?", msg.Delta),
            }).Error
        
        if err != nil {
            log.Errorf("同步库存到MySQL失败: sku=%d, error=%v", msg.SkuID, err)
            // 重试或人工介入
        }
    })
}
```

### 3.7 预扣库存机制（订单场景）

**为什么需要预扣库存？**
- 用户下单后15分钟内可能不支付
- 不能立即扣减真实库存，否则影响其他用户
- 需要"占住"库存，支付后再确认扣减

**预扣流程**：

```
┌────────────────────────────────────────────────────────────────┐
│                      预扣库存流程                                │
└────────────────────────────────────────────────────────────────┘

1. 用户下单
   ├──▶ 预扣库存（available_stock - 1, locked_stock + 1）
   │
   ├──▶ 创建订单（15分钟有效期）
   │
   ├──▶ 等待支付...
   │
   └──▶ 两种结果：
        ├─▶ 支付成功：确认扣减（locked_stock - 1, sold_stock + 1）
        └─▶ 超时/取消：释放库存（locked_stock - 1, available_stock + 1）
```

**实现代码**：

```go
// PreDeductStock 预扣库存
func (s *StockService) PreDeductStock(ctx context.Context, skuID int64, quantity int, orderNo string) error {
    // 使用Lua脚本原子操作
    script := `
        local available = redis.call('GET', KEYS[1])
        if not available or tonumber(available) < tonumber(ARGV[1]) then
            return -1
        end
        
        -- 扣减可用库存
        redis.call('DECRBY', KEYS[1], ARGV[1])
        
        -- 增加锁定库存
        redis.call('INCRBY', KEYS[2], ARGV[1])
        
        -- 记录预扣信息（15分钟过期）
        redis.call('SETEX', KEYS[3], 900, ARGV[1])
        
        return 1
    `
    
    availableKey := fmt.Sprintf("stock:available:%d", skuID)
    lockedKey := fmt.Sprintf("stock:locked:%d", skuID)
    orderKey := fmt.Sprintf("stock:order:%s", orderNo)
    
    result, err := s.redis.Eval(ctx, script, 
        []string{availableKey, lockedKey, orderKey}, 
        quantity).Int()
    
    if err != nil {
        return err
    }
    
    if result == -1 {
        return errors.New("库存不足")
    }
    
    // 记录日志
    s.saveStockLog(skuID, "lock", -quantity, 0)
    
    return nil
}

// ConfirmDeduct 确认扣减（支付成功后调用）
func (s *StockService) ConfirmDeduct(ctx context.Context, skuID int64, orderNo string) error {
    orderKey := fmt.Sprintf("stock:order:%s", orderNo)
    
    // 获取预扣数量
    quantity, err := s.redis.Get(ctx, orderKey).Int()
    if err != nil {
        return errors.New("预扣记录不存在")
    }
    
    script := `
        -- 扣减锁定库存
        redis.call('DECRBY', KEYS[1], ARGV[1])
        
        -- 增加已售库存
        redis.call('INCRBY', KEYS[2], ARGV[1])
        
        -- 删除预扣记录
        redis.call('DEL', KEYS[3])
        
        return 1
    `
    
    lockedKey := fmt.Sprintf("stock:locked:%d", skuID)
    soldKey := fmt.Sprintf("stock:sold:%d", skuID)
    
    _, err = s.redis.Eval(ctx, script, 
        []string{lockedKey, soldKey, orderKey}, 
        quantity).Result()
    
    if err != nil {
        return err
    }
    
    // 异步同步到MySQL
    s.syncConfirmToMySQL(skuID, quantity, orderNo)
    
    return nil
}

// RollbackPreDeduct 回滚预扣（订单取消）
func (s *StockService) RollbackPreDeduct(ctx context.Context, skuID int64, orderNo string) error {
    orderKey := fmt.Sprintf("stock:order:%s", orderNo)
    
    // 获取预扣数量
    quantity, err := s.redis.Get(ctx, orderKey).Int()
    if err != nil {
        // 预扣记录已过期或不存在
        return nil
    }
    
    script := `
        -- 增加可用库存
        redis.call('INCRBY', KEYS[1], ARGV[1])
        
        -- 扣减锁定库存
        redis.call('DECRBY', KEYS[2], ARGV[1])
        
        -- 删除预扣记录
        redis.call('DEL', KEYS[3])
        
        return 1
    `
    
    availableKey := fmt.Sprintf("stock:available:%d", skuID)
    lockedKey := fmt.Sprintf("stock:locked:%d", skuID)
    
    _, err = s.redis.Eval(ctx, script, 
        []string{availableKey, lockedKey, orderKey}, 
        quantity).Result()
    
    return err
}
```

### 3.8 库存对账机制

**为什么需要对账？**
- Redis是缓存，可能丢失数据
- 异步同步可能失败
- 需要定期校对Redis和MySQL的库存数据

**对账实现**：

```go
// StockReconciliation 库存对账
func (s *StockService) StockReconciliation(ctx context.Context) error {
    // 1. 查询所有SKU
    var stocks []ProductStock
    if err := s.db.Find(&stocks).Error; err != nil {
        return err
    }
    
    var diffs []StockDiff
    
    for _, stock := range stocks {
        // 2. 获取Redis中的库存
        availableKey := fmt.Sprintf("stock:available:%d", stock.SkuID)
        lockedKey := fmt.Sprintf("stock:locked:%d", stock.SkuID)
        soldKey := fmt.Sprintf("stock:sold:%d", stock.SkuID)
        
        redisAvailable, _ := s.redis.Get(ctx, availableKey).Int()
        redisLocked, _ := s.redis.Get(ctx, lockedKey).Int()
        redisSold, _ := s.redis.Get(ctx, soldKey).Int()
        
        // 3. 对比差异
        if stock.AvailableStock != redisAvailable ||
           stock.LockedStock != redisLocked ||
           stock.SoldStock != redisSold {
            
            diff := StockDiff{
                SkuID:              stock.SkuID,
                MySQLAvailable:     stock.AvailableStock,
                RedisAvailable:     redisAvailable,
                MySQLLocked:        stock.LockedStock,
                RedisLocked:        redisLocked,
                MySQLSold:          stock.SoldStock,
                RedisSold:          redisSold,
            }
            
            diffs = append(diffs, diff)
            
            // 4. 以MySQL为准，修复Redis
            s.redis.Set(ctx, availableKey, stock.AvailableStock, 0)
            s.redis.Set(ctx, lockedKey, stock.LockedStock, 0)
            s.redis.Set(ctx, soldKey, stock.SoldStock, 0)
            
            log.Warnf("库存对账发现差异并已修复: sku=%d, diff=%+v", stock.SkuID, diff)
        }
    }
    
    // 5. 发送对账报告
    if len(diffs) > 0 {
        s.sendReconciliationReport(diffs)
    }
    
    return nil
}

// StartReconciliationWorker 启动定时对账任务（每小时）
func (s *StockService) StartReconciliationWorker(ctx context.Context) {
    ticker := time.NewTicker(1 * time.Hour)
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            if err := s.StockReconciliation(ctx); err != nil {
                log.Errorf("库存对账失败: %v", err)
            }
        }
    }
}
```

### 3.9 面试高频问题

**Q1: 如何解决库存超卖问题？**

A: 4种方案：
1. **数据库悲观锁**：SELECT FOR UPDATE，性能差，适合低并发
2. **数据库乐观锁**：WHERE version = ?，高并发下冲突率高
3. **Redis + Lua**：原子脚本，高性能，**推荐方案**
4. **消息队列**：削峰填谷，适合秒杀场景

**Q2: Redis库存和MySQL库存如何保持一致？**

A: 最终一致性方案：
1. 库存优先扣减Redis（高性能）
2. 异步通过MQ同步到MySQL
3. 定时对账（每小时），发现差异以MySQL为准修复Redis
4. MySQL作为数据源，Redis作为缓存

**Q3: 预扣库存的作用是什么？**

A: 解决订单未支付占用库存的问题：
- 下单时预扣（available_stock -> locked_stock）
- 支付成功后确认扣减（locked_stock -> sold_stock）
- 订单取消/超时回滚（locked_stock -> available_stock）
- 防止恶意占库存

**Q4: 高并发下如何保证库存扣减的原子性？**

A: 使用Redis + Lua脚本：
```lua
-- Lua脚本在Redis中原子执行
local stock = redis.call('GET', KEYS[1])
if tonumber(stock) < tonumber(ARGV[1]) then
    return -1
end
redis.call('DECRBY', KEYS[1], ARGV[1])
return 1
```
**特点**：
- Lua脚本原子性（不会被打断）
- Redis单线程（天然串行化）
- 性能极高（QPS 10万+）

**Q5: 库存数据库设计有什么要点？**

A: 关键字段设计：
```sql
available_stock  -- 可用库存
locked_stock     -- 锁定库存（预扣）
sold_stock       -- 已售库存
total_stock      -- 总库存

-- 约束：available + locked + sold = total
```
还需要：
- 版本号（version）用于乐观锁
- 操作日志表（审计）
- 仓库ID（多仓库场景）

---


## 4. 退款系统详解

### 4.1 退款系统核心概念

**退款系统是售后服务的核心**，需要保证：
- **资金安全**：退款金额准确，不能多退少退
- **幂等性**：重复请求不会导致重复退款
- **可追溯**：每笔退款都有完整记录
- **时效性**：退款及时到账，避免用户投诉

**退款业务流程**：

```
┌────────────────────────────────────────────────────────────────┐
│                      退款完整流程                                │
└────────────────────────────────────────────────────────────────┘

用户                   业务系统                第三方支付
 │                        │                        │
 │  1. 申请退款           │                        │
 ├───────────────────────▶                        │
 │                        │ 2. 创建退款单          │
 │                        │    校验订单状态        │
 │                        │                        │
 │                        │ 3. 调用支付渠道退款    │
 │                        ├───────────────────────▶
 │                        │ 4. 返回退款结果        │
 │                        │◀───────────────────────
 │                        │                        │
 │                        │ 5. 异步回调通知        │
 │                        │◀───────────────────────
 │                        │                        │
 │                        │ 6. 更新订单/库存      │
 │                        │    退款成功            │
 │  7. 退款成功通知       │                        │
 │◀───────────────────────                        │
 │                        │                        │
 │                        │ 8. 定时查询（兜底）    │
 │                        ├───────────────────────▶
 │                        │ 9. 返回退款状态        │
 │                        │◀───────────────────────
```

### 4.2 退款状态机

```
┌──────────────────────────────────────────────────────────┐
│                     退款状态流转                           │
└──────────────────────────────────────────────────────────┘

                ┌──────────────┐
                │   待审核     │ (Pending)
                │ (用户申请)    │
                └──────┬───────┘
                       │
         ┌─────────────┼─────────────┐
         │                           │
     审核通过                     审核拒绝
         │                           │
         ▼                           ▼
   ┌──────────┐                ┌──────────┐
   │ 退款中   │                │ 已拒绝   │
   └────┬─────┘                └──────────┘
        │
   ┌────┴────┐
   │         │
退款成功  退款失败
   │         │
   ▼         ▼
┌─────────┐ ┌─────────┐
│ 已退款  │ │重试/关闭 │
└─────────┘ └─────────┘
```

### 4.3 数据库表设计

**退款单表 (refunds)**：

```sql
CREATE TABLE `refunds` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `refund_no` VARCHAR(32) NOT NULL COMMENT '退款单号',
    `order_no` VARCHAR(32) NOT NULL COMMENT '订单号',
    `payment_no` VARCHAR(32) DEFAULT NULL COMMENT '支付单号',
    `user_id` BIGINT UNSIGNED NOT NULL,
    
    -- 退款信息
    `refund_type` TINYINT NOT NULL COMMENT '退款类型：1仅退款 2退货退款',
    `refund_amount` BIGINT NOT NULL COMMENT '退款金额（分）',
    `refund_reason` VARCHAR(255) NOT NULL COMMENT '退款原因',
    `status` TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1待审核 2退款中 3已退款 4已拒绝 5已关闭',
    
    -- 第三方信息
    `channel` VARCHAR(20) DEFAULT NULL COMMENT '支付渠道',
    `refund_trade_no` VARCHAR(64) DEFAULT NULL COMMENT '第三方退款流水号',
    `refund_resp` TEXT DEFAULT NULL COMMENT '退款响应数据',
    
    -- 审核信息
    `audit_status` TINYINT DEFAULT 1 COMMENT '审核状态：1待审核 2通过 3拒绝',
    `audit_remark` VARCHAR(255) DEFAULT NULL COMMENT '审核备注',
    `auditor` VARCHAR(50) DEFAULT NULL COMMENT '审核人',
    `audited_at` DATETIME DEFAULT NULL COMMENT '审核时间',
    
    -- 时间
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    `refunded_at` DATETIME DEFAULT NULL COMMENT '退款完成时间',
    
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_refund_no` (`refund_no`),
    KEY `idx_order_no` (`order_no`),
    KEY `idx_user_id` (`user_id`),
    KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='退款单表';
```

### 4.4 核心功能实现

#### 4.4.1 创建退款申请

```go
type RefundService struct {
    db              *gorm.DB
    redis           *redis.Client
    paymentFactory  *PaymentChannelFactory
}

// CreateRefund 创建退款申请
func (s *RefundService) CreateRefund(ctx context.Context, req *CreateRefundRequest) (*Refund, error) {
    // 1. 查询订单
    var order Order
    if err := s.db.Where("order_no = ?", req.OrderNo).First(&order).Error; err != nil {
        return nil, errors.New("订单不存在")
    }
    
    // 2. 校验订单状态（只有已支付、已发货、已完成可以退款）
    if order.Status != OrderStatusPaid && 
       order.Status != OrderStatusShipped && 
       order.Status != OrderStatusCompleted {
        return nil, errors.New("订单状态不允许退款")
    }
    
    // 3. 检查是否已存在退款单
    var existRefund Refund
    if err := s.db.Where("order_no = ? AND status IN (1,2,3)", req.OrderNo).
        First(&existRefund).Error; err == nil {
        return nil, errors.New("已存在退款申请")
    }
    
    // 4. 校验退款金额
    if req.RefundAmount > order.PayAmount {
        return nil, errors.New("退款金额不能大于实付金额")
    }
    
    // 5. 创建退款单
    refund := &Refund{
        RefundNo:     generateRefundNo(),
        OrderNo:      req.OrderNo,
        PaymentNo:    order.PaymentNo,
        UserID:       order.UserID,
        RefundType:   req.RefundType,
        RefundAmount: req.RefundAmount,
        RefundReason: req.RefundReason,
        Channel:      order.PayChannel,
        Status:       RefundStatusPending,
        AuditStatus:  AuditStatusPending,
    }
    
    if err := s.db.Create(refund).Error; err != nil {
        return nil, err
    }
    
    // 6. 更新订单状态
    s.db.Model(&order).Update("status", OrderStatusRefunding)
    
    // 7. 自动审核（或人工审核）
    if s.shouldAutoApprove(refund) {
        go s.ApproveRefund(context.Background(), refund.RefundNo, "system", "自动审核通过")
    }
    
    return refund, nil
}

// shouldAutoApprove 判断是否自动审核
func (s *RefundService) shouldAutoApprove(refund *Refund) bool {
    // 业务规则：
    // - 金额 < 100元自动通过
    // - 用户信誉良好自动通过
    // - 其他情况需要人工审核
    
    if refund.RefundAmount < 10000 { // 100元以下
        return true
    }
    
    // 查询用户退款历史
    var count int64
    s.db.Model(&Refund{}).Where("user_id = ? AND status = 3", refund.UserID).Count(&count)
    
    if count > 5 { // 退款次数过多，需要人工审核
        return false
    }
    
    return true
}

// ApproveRefund 审核通过并执行退款
func (s *RefundService) ApproveRefund(ctx context.Context, refundNo string, auditor string, remark string) error {
    var refund Refund
    if err := s.db.Where("refund_no = ?", refundNo).First(&refund).Error; err != nil {
        return err
    }
    
    // 1. 更新审核状态
    if err := s.db.Model(&refund).Updates(map[string]interface{}{
        "audit_status": AuditStatusApproved,
        "audit_remark": remark,
        "auditor":      auditor,
        "audited_at":   time.Now(),
        "status":       RefundStatusProcessing,
    }).Error; err != nil {
        return err
    }
    
    // 2. 调用支付渠道退款
    return s.processRefund(ctx, &refund)
}

// processRefund 执行退款
func (s *RefundService) processRefund(ctx context.Context, refund *Refund) error {
    // 1. 获取支付渠道
    channel, err := s.paymentFactory.GetChannel(refund.Channel)
    if err != nil {
        return err
    }
    
    // 2. 构造退款请求
    refundReq := &RefundRequest{
        RefundNo:  refund.RefundNo,
        OrderNo:   refund.OrderNo,
        Amount:    refund.RefundAmount,
        Reason:    refund.RefundReason,
    }
    
    // 3. 调用退款API
    refundResp, err := channel.Refund(ctx, refundReq)
    if err != nil {
        // 退款失败，更新状态
        s.db.Model(refund).Updates(map[string]interface{}{
            "status":      RefundStatusFailed,
            "refund_resp": err.Error(),
        })
        
        // 发送告警
        s.alertService.Send("退款失败", fmt.Sprintf("退款单:%s, 错误:%v", refund.RefundNo, err))
        
        return err
    }
    
    // 4. 退款成功
    if refundResp.Success {
        return s.handleRefundSuccess(ctx, refund, refundResp)
    }
    
    // 5. 退款处理中（异步）
    s.db.Model(refund).Updates(map[string]interface{}{
        "refund_trade_no": refundResp.TradeNo,
        "refund_resp":     toJSON(refundResp),
    })
    
    return nil
}

// handleRefundSuccess 处理退款成功
func (s *RefundService) handleRefundSuccess(ctx context.Context, refund *Refund, resp *RefundResponse) error {
    return s.db.Transaction(func(tx *gorm.DB) error {
        // 1. 更新退款单状态
        if err := tx.Model(refund).Updates(map[string]interface{}{
            "status":          RefundStatusSuccess,
            "refund_trade_no": resp.TradeNo,
            "refund_resp":     toJSON(resp),
            "refunded_at":     time.Now(),
        }).Error; err != nil {
            return err
        }
        
        // 2. 更新订单状态
        if err := tx.Model(&Order{}).Where("order_no = ?", refund.OrderNo).
            Update("status", OrderStatusRefunded).Error; err != nil {
            return err
        }
        
        // 3. 回退库存（如果是退货退款）
        if refund.RefundType == RefundTypeReturnGoods {
            var order Order
            tx.Where("order_no = ?", refund.OrderNo).First(&order)
            
            var items []OrderItem
            tx.Where("order_id = ?", order.ID).Find(&items)
            
            for _, item := range items {
                s.stockService.AddStock(ctx, item.SkuID, item.Quantity)
            }
        }
        
        // 4. 退还优惠券（如果使用了）
        var order Order
        tx.Where("order_no = ?", refund.OrderNo).First(&order)
        if order.CouponID > 0 {
            s.couponService.Refund(tx, order.CouponID)
        }
        
        // 5. 发送MQ通知
        s.mq.Publish("refund.success", map[string]interface{}{
            "refund_no": refund.RefundNo,
            "order_no":  refund.OrderNo,
            "amount":    refund.RefundAmount,
        })
        
        return nil
    })
}
```

#### 4.4.2 退款重试机制

**为什么需要重试？**
- 网络抖动导致退款失败
- 第三方支付平台偶尔故障
- 提高退款成功率

```go
// RetryRefund 退款重试
func (s *RefundService) RetryRefund(ctx context.Context, refundNo string) error {
    var refund Refund
    if err := s.db.Where("refund_no = ?", refundNo).First(&refund).Error; err != nil {
        return err
    }
    
    // 只有失败状态才能重试
    if refund.Status != RefundStatusFailed {
        return errors.New("退款状态不允许重试")
    }
    
    // 检查重试次数（最多3次）
    retryKey := fmt.Sprintf("refund:retry:%s", refundNo)
    retryCount, _ := s.redis.Incr(ctx, retryKey).Result()
    s.redis.Expire(ctx, retryKey, 24*time.Hour)
    
    if retryCount > 3 {
        // 重试次数超限，转人工处理
        s.alertService.Send("退款重试失败", fmt.Sprintf("退款单:%s 已重试3次仍失败，请人工处理", refundNo))
        return errors.New("重试次数超限")
    }
    
    // 更新状态为退款中
    s.db.Model(&refund).Update("status", RefundStatusProcessing)
    
    // 执行退款
    return s.processRefund(ctx, &refund)
}

// StartRetryWorker 启动退款重试定时任务
func (s *RefundService) StartRetryWorker(ctx context.Context) {
    ticker := time.NewTicker(10 * time.Minute) // 每10分钟执行一次
    defer ticker.Stop()
    
    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            s.retryFailedRefunds(ctx)
        }
    }
}

func (s *RefundService) retryFailedRefunds(ctx context.Context) {
    // 查询最近1小时内失败的退款单
    var refunds []Refund
    s.db.Where("status = ? AND created_at > ?", 
        RefundStatusFailed, 
        time.Now().Add(-1*time.Hour)).
        Limit(50).
        Find(&refunds)
    
    for _, refund := range refunds {
        go func(r Refund) {
            if err := s.RetryRefund(context.Background(), r.RefundNo); err != nil {
                log.Errorf("退款重试失败: refund_no=%s, error=%v", r.RefundNo, err)
            }
        }(refund)
    }
}
```

### 4.5 面试高频问题

**Q1: 退款如何保证幂等性？**

A: 多层保障：
1. **退款单号唯一性**：同一订单只能创建一次退款单
2. **状态机校验**：已退款状态不可重复退款
3. **第三方退款号**：支付平台退款接口本身幂等

**Q2: 退款失败怎么办？**

A: 重试 + 兜底方案：
1. **自动重试**：失败后定时重试3次
2. **主动查询**：定时查询退款状态（兜底）
3. **人工介入**：重试3次仍失败，转人工处理
4. **告警通知**：退款异常及时告警

**Q3: 部分退款如何实现？**

A: 支持多次退款：
1. 记录累计已退款金额
2. 校验：已退款金额 + 本次退款 <= 实付金额
3. 每次退款创建独立退款单
4. 全部退款完成后更新订单状态

---

## 5. 优惠系统详解

### 5.1 优惠系统核心概念

**优惠系统是营销活动的核心**，需要保证：
- **防超发**：优惠券数量有限，不能超发
- **防重复领取/使用**：同一用户/订单只能使用一次
- **高并发**：秒杀券场景下每秒上万请求
- **灵活性**：支持多种优惠类型和规则

**优惠券类型**：
1. **满减券**：满100减20
2. **折扣券**：8折券
3. **现金券**：10元无门槛券
4. **商品券**：特定商品可用
5. **品类券**：特定品类可用

### 5.2 数据库表设计

**优惠券模板表 (coupon_templates)**：

```sql
CREATE TABLE `coupon_templates` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(100) NOT NULL COMMENT '优惠券名称',
    `coupon_type` TINYINT NOT NULL COMMENT '类型：1满减 2折扣 3现金券',
    
    -- 优惠规则
    `discount_amount` BIGINT DEFAULT NULL COMMENT '优惠金额（分）',
    `discount_rate` DECIMAL(4,2) DEFAULT NULL COMMENT '折扣率（0.8=8折）',
    `min_amount` BIGINT DEFAULT 0 COMMENT '最低消费金额（分）',
    `max_discount` BIGINT DEFAULT NULL COMMENT '最高优惠金额（封顶）',
    
    -- 发放规则
    `total_count` INT NOT NULL COMMENT '总发行量',
    `issued_count` INT NOT NULL DEFAULT 0 COMMENT '已发放数量',
    `per_user_limit` INT DEFAULT 1 COMMENT '每人限领数量',
    
    -- 使用规则
    `valid_days` INT DEFAULT 7 COMMENT '有效天数',
    `valid_start` DATETIME DEFAULT NULL COMMENT '有效期开始',
    `valid_end` DATETIME DEFAULT NULL COMMENT '有效期结束',
    
    -- 适用范围
    `scope_type` TINYINT DEFAULT 1 COMMENT '适用范围：1全场 2指定商品 3指定品类',
    `scope_ids` VARCHAR(500) DEFAULT NULL COMMENT '适用ID列表（JSON）',
    
    `status` TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1启用 0禁用',
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    PRIMARY KEY (`id`),
    KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='优惠券模板';
```

**用户优惠券表 (user_coupons)**：

```sql
CREATE TABLE `user_coupons` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `coupon_no` VARCHAR(32) NOT NULL COMMENT '优惠券编号',
    `template_id` BIGINT UNSIGNED NOT NULL COMMENT '模板ID',
    `user_id` BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
    
    -- 状态
    `status` TINYINT NOT NULL DEFAULT 1 COMMENT '状态：1未使用 2已使用 3已过期 4已锁定',
    `order_no` VARCHAR(32) DEFAULT NULL COMMENT '使用的订单号',
    
    -- 时间
    `received_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '领取时间',
    `used_at` DATETIME DEFAULT NULL COMMENT '使用时间',
    `expired_at` DATETIME NOT NULL COMMENT '过期时间',
    
    PRIMARY KEY (`id`),
    UNIQUE KEY `uk_coupon_no` (`coupon_no`),
    KEY `idx_user_status` (`user_id`, `status`),
    KEY `idx_template_id` (`template_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户优惠券';
```

### 5.3 核心功能实现

#### 5.3.1 发放优惠券（防超发）

```go
type CouponService struct {
    db    *gorm.DB
    redis *redis.Client
}

// IssueCoupon 发放优惠券（防超发）
func (s *CouponService) IssueCoupon(ctx context.Context, userID int64, templateID int64) (*UserCoupon, error) {
    // 1. 查询优惠券模板
    var template CouponTemplate
    if err := s.db.Where("id = ? AND status = 1", templateID).First(&template).Error; err != nil {
        return nil, errors.New("优惠券不存在或已下架")
    }
    
    // 2. 检查用户领取次数
    var userCount int64
    s.db.Model(&UserCoupon{}).
        Where("user_id = ? AND template_id = ?", userID, templateID).
        Count(&userCount)
    
    if userCount >= int64(template.PerUserLimit) {
        return nil, errors.New("已达到领取上限")
    }
    
    // 3. Redis预扣库存（防超发）
    stockKey := fmt.Sprintf("coupon:stock:%d", templateID)
    
    // 使用Lua脚本原子扣减
    script := `
        local stock = redis.call('GET', KEYS[1])
        if not stock then
            return -1
        end
        
        stock = tonumber(stock)
        if stock <= 0 then
            return -2
        end
        
        redis.call('DECR', KEYS[1])
        return stock - 1
    `
    
    result, err := s.redis.Eval(ctx, script, []string{stockKey}, 1).Int()
    if err != nil {
        return nil, err
    }
    
    if result == -1 {
        // 库存未初始化，从数据库加载
        remaining := template.TotalCount - template.IssuedCount
        s.redis.Set(ctx, stockKey, remaining, 0)
        
        if remaining <= 0 {
            return nil, errors.New("优惠券已抢完")
        }
    } else if result == -2 {
        return nil, errors.New("优惠券已抢完")
    }
    
    // 4. 创建用户优惠券
    coupon := &UserCoupon{
        CouponNo:   generateCouponNo(),
        TemplateID: templateID,
        UserID:     userID,
        Status:     CouponStatusUnused,
        ReceivedAt: time.Now(),
        ExpiredAt:  time.Now().AddDate(0, 0, template.ValidDays),
    }
    
    if template.ValidStart != nil && template.ValidEnd != nil {
        coupon.ExpiredAt = *template.ValidEnd
    }
    
    if err := s.db.Create(coupon).Error; err != nil {
        // 创建失败，回滚Redis库存
        s.redis.Incr(ctx, stockKey)
        return nil, err
    }
    
    // 5. 更新模板已发放数量（异步，最终一致性）
    go s.updateIssuedCount(templateID)
    
    return coupon, nil
}

// updateIssuedCount 更新已发放数量
func (s *CouponService) updateIssuedCount(templateID int64) {
    s.db.Model(&CouponTemplate{}).
        Where("id = ?", templateID).
        UpdateColumn("issued_count", gorm.Expr("issued_count + 1"))
}
```

#### 5.3.2 使用优惠券（防重复使用）

```go
// LockCoupon 锁定优惠券（下单时调用）
func (s *CouponService) LockCoupon(tx *gorm.DB, couponID int64, orderNo string) error {
    result := tx.Model(&UserCoupon{}).
        Where("id = ? AND status = ?", couponID, CouponStatusUnused).
        Updates(map[string]interface{}{
            "status":   CouponStatusLocked,
            "order_no": orderNo,
        })
    
    if result.Error != nil {
        return result.Error
    }
    
    if result.RowsAffected == 0 {
        return errors.New("优惠券不可用")
    }
    
    return nil
}

// UseCoupon 使用优惠券（支付成功后调用）
func (s *CouponService) UseCoupon(tx *gorm.DB, couponID int64) error {
    return tx.Model(&UserCoupon{}).
        Where("id = ?", couponID).
        Updates(map[string]interface{}{
            "status":  CouponStatusUsed,
            "used_at": time.Now(),
        }).Error
}

// UnlockCoupon 解锁优惠券（订单取消时调用）
func (s *CouponService) UnlockCoupon(tx *gorm.DB, couponID int64) error {
    return tx.Model(&UserCoupon{}).
        Where("id = ?", couponID).
        Updates(map[string]interface{}{
            "status":   CouponStatusUnused,
            "order_no": nil,
        }).Error
}

// RefundCoupon 退还优惠券（退款成功后调用）
func (s *CouponService) RefundCoupon(tx *gorm.DB, couponID int64) error {
    // 退还后延长7天有效期
    return tx.Model(&UserCoupon{}).
        Where("id = ?", couponID).
        Updates(map[string]interface{}{
            "status":     CouponStatusUnused,
            "order_no":   nil,
            "used_at":    nil,
            "expired_at": gorm.Expr("DATE_ADD(expired_at, INTERVAL 7 DAY)"),
        }).Error
}
```

#### 5.3.3 优惠计算

```go
// CalculateDiscount 计算优惠金额
func (s *CouponService) CalculateDiscount(ctx context.Context, couponID int64, orderAmount int64) (int64, error) {
    // 1. 查询优惠券
    var coupon UserCoupon
    if err := s.db.Where("id = ?", couponID).First(&coupon).Error; err != nil {
        return 0, errors.New("优惠券不存在")
    }
    
    // 2. 查询模板
    var template CouponTemplate
    if err := s.db.Where("id = ?", coupon.TemplateID).First(&template).Error; err != nil {
        return 0, err
    }
    
    // 3. 校验优惠券状态
    if coupon.Status != CouponStatusUnused {
        return 0, errors.New("优惠券不可用")
    }
    
    if time.Now().After(coupon.ExpiredAt) {
        return 0, errors.New("优惠券已过期")
    }
    
    // 4. 校验最低消费
    if orderAmount < template.MinAmount {
        return 0, fmt.Errorf("订单金额不满足使用条件（需满%d元）", template.MinAmount/100)
    }
    
    // 5. 计算优惠金额
    var discountAmount int64
    
    switch template.CouponType {
    case CouponTypeFixAmount:
        // 现金券/满减券
        discountAmount = template.DiscountAmount
        
    case CouponTypeDiscount:
        // 折扣券
        discountAmount = int64(float64(orderAmount) * (1 - float64(template.DiscountRate)))
        
        // 封顶金额
        if template.MaxDiscount > 0 && discountAmount > template.MaxDiscount {
            discountAmount = template.MaxDiscount
        }
    }
    
    // 6. 优惠金额不能超过订单金额
    if discountAmount > orderAmount {
        discountAmount = orderAmount
    }
    
    return discountAmount, nil
}
```

### 5.4 面试高频问题

**Q1: 如何防止优惠券超发？**

A: Redis + Lua原子扣减：
```lua
-- 预扣优惠券库存
local stock = redis.call('GET', KEYS[1])
if tonumber(stock) <= 0 then
    return -1  -- 已抢完
end
redis.call('DECR', KEYS[1])
return 1
```
特点：
- Redis单线程，无并发问题
- Lua脚本原子性
- 性能极高

**Q2: 如何防止优惠券重复使用？**

A: 状态机 + 数据库行锁：
1. 下单时锁定优惠券（status = locked）
2. 支付成功标记已使用（status = used）
3. 订单取消解锁优惠券（status = unused）
4. WHERE status = unused保证只能使用一次

**Q3: 优惠券退款后如何处理？**

A: 退还优惠券并延长有效期：
```sql
UPDATE user_coupons 
SET status = 1,  -- 未使用
    order_no = NULL,
    used_at = NULL,
    expired_at = DATE_ADD(expired_at, INTERVAL 7 DAY)  -- 延长7天
WHERE id = ?
```

**Q4: 多张优惠券如何叠加使用？**

A: 业务规则：
1. 同类型优惠券互斥（如满减券只能用一张）
2. 不同类型可叠加（如满减券 + 折扣券）
3. 计算顺序：先折扣后满减
4. 设置叠加上限（最多优惠50%）

---

## 总结：电商系统关键技术点

| 系统   | 核心问题     | 解决方案                   | 关键技术      |
|------|----------|------------------------|-----------|
| 订单系统 | 幂等性、超时取消 | Token + 分布式锁、Redis延迟队列 | 状态机、事务    |
| 支付系统 | 回调幂等、对账  | 分布式锁 + 日志、定时对账         | 适配器模式、幂等性 |
| 库存系统 | 超卖问题     | Redis + Lua原子扣减        | 预扣库存、对账   |
| 退款系统 | 重试、幂等性   | 定时重试、状态机               | 补偿事务      |
| 优惠系统 | 超发、重复使用  | Redis预扣、状态机            | 原子性、规则引擎  |

---

**祝你面试成功！** 🎉

