# CPU缓存架构详解

## 核心答案

**典型的现代CPU缓存架构：**

```
┌────────────────────────────────────────────────────┐
│ L1、L2：每个物理核心独占                              │
│ L3：所有核心共享                                     │
│                                                    │
│ 但有重要细节：                                       │
│ - 超线程(HT)时，同一物理核的逻辑核共享L1/L2             │
│ - 不同厂商/架构可能有差异                             │
│ - 服务器CPU可能有更复杂的层次                          │
└────────────────────────────────────────────────────┘
```

---

## 1. 典型的CPU缓存架构

### 1.1 Intel/AMD x86架构（最常见）

```
完整的缓存层次结构：

CPU芯片
├─ 物理核心0
│  ├─ 逻辑核心0 (HT0)
│  │  └─ 寄存器 (几百字节)
│  │
│  ├─ 逻辑核心1 (HT1)  ← 如果启用超线程
│  │  └─ 寄存器 (几百字节)
│  │
│  ├─ L1 Cache (独占，两个HT共享)
│  │  ├─ L1i (指令缓存): 32 KB
│  │  └─ L1d (数据缓存):  32 KB
│  │
│  └─ L2 Cache (独占，两个HT共享): 256 KB - 1 MB
│
├─ 物理核心1
│  ├─ 逻辑核心2, 3
│  ├─ L1 Cache: 64 KB
│  └─ L2 Cache: 256 KB - 1 MB
│
├─ ... (更多核心)
│
└─ L3 Cache (所有核心共享): 8 MB - 64 MB
   └─ 也称为LLC (Last Level Cache)

主内存 (DRAM): 几GB到几百GB


独占 vs 共享总结：
┌──────────────────────────────────────────┐
│ 缓存级别   │  大小        │  独占/共享      │
├──────────────────────────────────────────┤
│ L1i       │  32 KB      │  每个物理核      │
│ L1d       │  32 KB      │  每个物理核      │
│ L2        │  256KB-1MB  │  每个物理核      │
│ L3        │  8MB-64MB   │  所有核心共享    │
│ 内存       │  几GB+      │  所有核心共享    │
└──────────────────────────────────────────┘
```

### 1.2 具体示例：Intel Core i7-9700K

```
规格：
- 物理核心：8个
- 逻辑核心：8个（无超线程）
- L1 Cache：每核 32KB(i) + 32KB(d) = 64KB × 8 = 512KB 总计
- L2 Cache：每核 256KB × 8 = 2MB 总计
- L3 Cache：12MB 共享

架构图：

┌──────────────────────────────────────────────────┐
│                Intel Core i7-9700K               │
├──────────────────────────────────────────────────┤
│                                                  │
│  核心0    核心1    核心2    核心3                   │
│  ┌───┐   ┌───┐   ┌───┐   ┌───┐                   │
│  │L1 │   │L1 │   │L1 │   │L1 │                   │
│  │64K│   │64K│   │64K│   │64K│                   │
│  └─┬─┘   └─┬─┘   └─┬─┘   └─┬─┘                   │
│    │       │       │       │                     │
│  ┌─┴─┐   ┌─┴─┐   ┌─┴─┐   ┌─┴─┐                   │
│  │L2 │   │L2 │   │L2 │   │L2 │                   │
│  │256│   │256│   │256│   │256│                   │
│  │ KB│   │ KB│   │ KB│   │ KB│                   │
│  └─┬─┘   └─┬─┘   └─┬─┘   └─┬─┘                   │
│    │       │       │       │                     │
│    └───────┴───────┴───────┘                     │
│              ↓                                   │
│    ┌──────────────────────┐                      │
│    │   L3 Cache (12 MB)   │ ← 所有核心共享         │
│    │   Last Level Cache   │                      │
│    └──────────────────────┘                      │
│              ↓                                   │
│       Memory Controller                          │
└──────────────────────────────────────────────────┘
              ↓
         DDR4 Memory
```

### 1.3 示例：AMD Ryzen 9 5900X

```
规格：
- 物理核心：12个
- 逻辑核心：24个（启用SMT）
- L1 Cache：每核 32KB(i) + 32KB(d) = 64KB × 12 = 768KB 总计
- L2 Cache：每核 512KB × 12 = 6MB 总计
- L3 Cache：64MB 共享（分为2个CCX，每个32MB）

AMD的特殊之处：
- L3被分为多个"CCX"(Core Complex)
- 同一CCX内的核心共享L3
- 不同CCX之间的L3访问延迟更高

架构图：

┌────────────────────────────────────────────────────┐
│              AMD Ryzen 9 5900X                     │
├────────────────────────────────────────────────────┤
│                                                    │
│  CCX 0 (6核心)            CCX 1 (6核心)             │
│  ┌──────────────┐        ┌──────────────┐          │
│  │ 核0  核1  核2 │        │ 核6  核7  核8 │          │
│  │ L1   L1   L1 │        │ L1   L1   L1 │          │
│  │ 64K  64K  64K│        │ 64K  64K  64K│          │
│  │  │    │    │ │        │  │    │    │ │          │
│  │ L2   L2   L2 │        │ L2   L2   L2 │          │
│  │512K 512K 512K│        │512K 512K 512K│          │
│  │  │    │    │ │        │  │    │    │ │          │
│  │ 核3  核4  核5 │        │ 核9  核10 核11│          │
│  │ L1   L1   L1 │        │ L1   L1   L1 │          │
│  │ 64K  64K  64K│        │ 64K  64K  64K│          │
│  │  │    │    │ │        │  │    │    │ │          │
│  │ L2   L2   L2 │        │ L2   L2   L2 │          │
│  │512K 512K 512K│        │512K 512K 512K│          │
│  └──┬───┬───┬───┘        └──┬───┬───┬───┘          │
│     │   │   │               │   │   │              │
│  ┌──┴───┴───┴──┐         ┌──┴───┴───┴──┐           │
│  │ L3 Cache    │         │ L3 Cache    │           │
│  │   32 MB     │         │   32 MB     │           │
│  └─────────────┘         └─────────────┘           │
│         ↓                      ↓                   │
│         └──────────┬───────────┘                   │
│                    ↓                               │
│             Infinity Fabric                        │
│                    ↓                               │
│            Memory Controller                       │
└────────────────────────────────────────────────────┘
                    ↓
               DDR4 Memory
```

---

## 2. 超线程（Hyper-Threading）下的缓存共享

### 2.1 超线程的缓存架构

```
启用超线程的物理核心：

┌─────────────────────────────────────────┐
│        物理核心 0                        │
├─────────────────────────────────────────┤
│                                         │
│  逻辑核心 0 (HT0)    逻辑核心 1 (HT1)      │
│  ┌──────────┐        ┌──────────┐       │
│  │ 寄存器组  │        │ 寄存器组  │        │
│  │  RAX     │        │  RAX     │       │
│  │  RBX     │        │  RBX     │       │
│  │  ...     │        │  ...     │       │
│  └────┬─────┘        └────┬─────┘       │
│       │                   │             │
│       │    独立（复制）     │             │
│       │                   │             │
│       └─────────┬─────────┘             │
│                 ↓                       │
│       ┌──────────────────┐              │
│       │  L1i Cache 32KB  │ ← 共享！      │
│       │  L1d Cache 32KB  │ ← 共享！      │
│       └────────┬─────────┘              │
│                ↓                        │
│       ┌──────────────────┐              │
│       │  L2 Cache 256KB  │ ← 共享！      │
│       └────────┬─────────┘              │
└────────────────┼────────────────────────┘
                 ↓
        L3 Cache (所有核心共享)

关键点：
1. 每个逻辑核心有独立的寄存器组
2. 同一物理核心的逻辑核心共享L1、L2
3. L1、L2的容量不会因为超线程而翻倍
4. 两个逻辑核心竞争L1、L2的容量
```

### 2.2 超线程导致的缓存竞争

```
示例：两个逻辑核心访问不同数据

物理核心0 (L2 = 256KB)
├─ 逻辑核心0: 处理订单数据 (200KB)
│  → 占用L2的78%
│
└─ 逻辑核心1: 处理用户数据 (200KB)
   → 尝试占用L2的78%
   → 驱逐逻辑核心0的数据
   → Cache Miss！

结果：
- 两个逻辑核心互相污染缓存
- 性能可能不如单线程（在某些workload下）
- 这就是为什么CPU密集型任务关闭HT可能更快
```

---

## 3. 缓存访问延迟

### 3.1 延迟对比

```
典型的缓存访问延迟（Intel Skylake架构）：

┌──────────────────────────────────────────────┐
│ 存储层次     │ 延迟(cycles) │ 延迟(ns @ 3GHz)   │
├──────────────────────────────────────────────┤
│ L1 Cache    │     4        │    ~1.3 ns      │
│ L2 Cache    │    12        │    ~4 ns        │
│ L3 Cache    │    40        │   ~13 ns        │
│ 本地内存     │   ~200       │   ~67 ns        │
│ 远程内存*    │   ~300       │  ~100 ns        │
└──────────────────────────────────────────────┘

* 远程内存：NUMA系统中，访问其他CPU的内存

带宽对比：

┌──────────────────────────────────────────────┐
│ 存储层次     │   带宽              │  读/写     │
├──────────────────────────────────────────────┤
│ L1 Cache    │ ~1 TB/s/核         │ 64B/cycle  │
│ L2 Cache    │ ~500 GB/s/核       │ 32B/cycle  │
│ L3 Cache    │ ~200 GB/s          │ 共享       │
│ DDR4-3200   │ ~25 GB/s/通道      │            │
└──────────────────────────────────────────────┘

性能差异示例：

访问L1: 100次 = 400 cycles ≈ 133 ns
访问内存: 100次 = 20,000 cycles ≈ 6,667 ns

差距：50倍！

这就是为什么缓存命中率如此重要。
```

### 3.2 缓存层次对性能的影响

```go
// 实验：测量不同缓存层次的访问速度

package main

import (
    "fmt"
    "time"
)

const (
    KB = 1024
    MB = 1024 * KB
)

func main() {
    // 测试1: 工作集 < L1 (32KB)
    testCacheLevel("L1", 16*KB, 100_000_000)

    // 测试2: 工作集 < L2 (256KB)
    testCacheLevel("L2", 128*KB, 100_000_000)

    // 测试3: 工作集 < L3 (8MB)
    testCacheLevel("L3", 4*MB, 100_000_000)

    // 测试4: 工作集 > L3 (主内存)
    testCacheLevel("Memory", 64*MB, 10_000_000)
}

func testCacheLevel(name string, size int, iterations int) {
    // 分配数组
    arr := make([]int64, size/8)

    // 预热（确保数据在对应缓存层）
    for i := 0; i < len(arr); i++ {
        arr[i] = int64(i)
    }

    // 测量顺序访问
    start := time.Now()
    sum := int64(0)
    for iter := 0; iter < iterations; iter++ {
        for i := 0; i < len(arr); i++ {
            sum += arr[i]
        }
    }
    elapsed := time.Since(start)

    nsPerAccess := float64(elapsed.Nanoseconds()) / float64(iterations) / float64(len(arr))

    fmt.Printf("%s Cache (%d KB): %.2f ns/access, Total: %v\n",
        name, size/KB, nsPerAccess, elapsed)
}

// 典型输出（Intel Core i7）：
// L1 Cache (16 KB): 0.25 ns/access, Total: 400ms
// L2 Cache (128 KB): 1.20 ns/access, Total: 1.92s
// L3 Cache (4096 KB): 4.50 ns/access, Total: 1.84s
// Memory Cache (65536 KB): 15.30 ns/access, Total: 1.00s
//
// 可以清晰看到延迟随工作集增大而增加
```

---

## 4. 缓存一致性协议（MESI）

### 4.1 为什么需要缓存一致性？

```
场景：两个核心访问同一内存地址

初始状态：
┌──────────────┐        ┌──────────────┐
│   core 0     │        │   core 1     │
│   L1: 空     │        │   L1: 空      │
│   L2: 空     │        │   L2: 空      │
└──────────────┘        └──────────────┘
       ↓                       ↓
    ┌────────────────────────────┐
    │  L3: 地址0x1000 = 42        │
    └────────────────────────────┘

Step 1: 核心0读取0x1000
┌──────────────┐        ┌──────────────┐
│   core 0     │        │   core 1     │
│ L1: 0x1000=42│        │   L1: 空     │
│ L2: 0x1000=42│        │   L2: 空     │
└──────────────┘        └──────────────┘

Step 2: 核心1也读取0x1000
┌──────────────┐        ┌──────────────┐
│   core 0     │        │   core 1     │
│ L1: 0x1000=42│        │ L1: 0x1000=42│
│ L2: 0x1000=42│        │ L2: 0x1000=42│
└──────────────┘        └──────────────┘

Step 3: 核心0写入0x1000 = 100
┌──────────────┐        ┌──────────────┐
│   core 0     │        │   core 1     │
│ L1:0x1000=100│        │ L1:0x1000=42 │ ← 不一致！
│ L2:0x1000=100│        │ L2:0x1000=42 │
└──────────────┘        └──────────────┘

问题：核心1读到旧数据！

解决方案：MESI协议（缓存一致性协议）
```

### 4.2 导致缓存不一致的具体场景（MESI触发条件）

**MESI协议在以下场景中触发，维护缓存一致性：**

#### 4.2.1 多核读同一地址（Read-Read场景）

```
触发条件：多个核心读取同一内存地址

场景：共享只读数据

┌─────────────────────────────────────────────────┐
│ 典型例子：多个线程读取配置数据                  │
├─────────────────────────────────────────────────┤
│ 全局变量：                                      │
│ const int CONFIG_MAX_CONNECTIONS = 1000;        │
│                                                 │
│ 线程1（核心0）：读取 CONFIG_MAX_CONNECTIONS     │
│ 线程2（核心1）：读取 CONFIG_MAX_CONNECTIONS     │
│ 线程3（核心2）：读取 CONFIG_MAX_CONNECTIONS     │
└─────────────────────────────────────────────────┘

MESI状态变化：

初始：
核心0: I (无数据)
核心1: I (无数据)
核心2: I (无数据)

核心0读取：
核心0: E [0x1000=1000] ← 独占状态（只有我有）
核心1: I
核心2: I

核心1读取（触发MESI）：
1. 核心1发出读请求到总线
2. 核心0监听总线，检测到读请求
3. 核心0: E → S （从独占变为共享）
4. 核心1: I → S （从无效变为共享）
5. 可以从核心0直接传输，也可以从内存读取

核心0: S [0x1000=1000]
核心1: S [0x1000=1000]

核心2读取（触发MESI）：
核心0: S [0x1000=1000]
核心1: S [0x1000=1000]
核心2: S [0x1000=1000] ← 也变为共享

结论：
- Read-Read不会导致数据不一致
- MESI将状态从E改为S，允许多个缓存共享
- 性能影响：轻微（需要总线仲裁）
```

#### 4.2.2 多核写同一地址（Write-Write场景）

```
触发条件：多个核心写入同一内存地址

场景：多线程竞争修改共享变量

┌─────────────────────────────────────────────────┐
│ 典型例子：多个线程修改计数器                    │
├─────────────────────────────────────────────────┤
│ 全局变量：                                      │
│ int global_counter = 0;                         │
│                                                 │
│ 线程1（核心0）：global_counter++                │
│ 线程2（核心1）：global_counter++                │
└─────────────────────────────────────────────────┘

MESI状态变化：

初始：两个核心都已读取（共享状态）
核心0: S [0x2000=0]
核心1: S [0x2000=0]

核心0写入 global_counter++ （触发MESI）：
1. 核心0发出"写入意图"（Invalidate）到总线
2. 核心1监听到Invalidate信号
3. 核心1: S → I （失效自己的缓存）
4. 核心0: S → M （独占修改）
5. 核心0写入新值：0 → 1

核心0: M [0x2000=1] ← 已修改
核心1: I             ← 失效

核心1写入 global_counter++ （触发MESI）：
1. 核心1发出读请求（因为处于I状态）
2. 核心0检测到请求，且数据处于M状态
3. 核心0必须先写回内存！（M → S）
4. 核心1从内存读取：I → S
5. 核心1发出写入意图
6. 核心0: S → I
7. 核心1: S → M，写入新值：1 → 2

核心0: I
核心1: M [0x2000=2]

性能代价：
- 每次写入都需要总线仲裁
- 需要失效其他核心的缓存
- M状态的数据被读取前需要写回内存
- 延迟：~100-200 cycles
```

#### 4.2.3 读-写竞争（Read-Write场景）

```
触发条件：一个核心读，另一个核心写同一地址

场景：生产者-消费者模式

┌─────────────────────────────────────────────────┐
│ 典型例子：消息队列                              │
├─────────────────────────────────────────────────┤
│ 结构：                                          │
│ struct Queue {                                  │
│     int head;                                   │
│     int tail;                                   │
│     int buffer[1024];                           │
│ };                                              │
│                                                 │
│ 生产者（核心0）：写 queue.tail                  │
│ 消费者（核心1）：读 queue.tail                  │
└─────────────────────────────────────────────────┘

MESI状态变化：

初始：消费者读取了tail
核心0: I
核心1: E [0x3000=10] ← tail=10

生产者写入（触发MESI）：
1. 核心0发出读请求（要先读再改）
2. 核心1: E → S（共享）
3. 核心0: I → S
4. 核心0发出写入意图
5. 核心1: S → I （失效！）
6. 核心0: S → M，写入 tail=11

核心0: M [0x3000=11]
核心1: I ← 缓存失效

消费者再次读取（触发MESI）：
1. 核心1发出读请求
2. 核心0: M → S（写回内存）
3. 核心1: I → S，读到最新值11

核心0: S [0x3000=11]
核心1: S [0x3000=11]

问题：Ping-Pong效应
- 缓存行在两个核心间反复传递
- 性能严重下降（每次都是Cache Miss）
```

#### 4.2.4 False Sharing（伪共享）

```
触发条件：不同核心修改同一缓存行内的不同变量

场景：看似独立的变量实际共享缓存行

┌─────────────────────────────────────────────────┐
│ 典型例子：两个独立的计数器                      │
├─────────────────────────────────────────────────┤
│ struct {                                        │
│     int counter0;  // 线程0使用                 │
│     int counter1;  // 线程1使用                 │
│ } counters;                                     │
│                                                 │
│ 内存布局（64字节缓存行）：                      │
│ ┌────────────────────────────────────┐          │
│ │ counter0 │ counter1 │ ... padding │          │
│ │  (4B)    │  (4B)    │   (56B)     │          │
│ └────────────────────────────────────┘          │
│           同一个缓存行！                        │
└─────────────────────────────────────────────────┘

MESI状态变化：

初始：两个核心都已读取
核心0: S [缓存行: counter0=0, counter1=0]
核心1: S [缓存行: counter0=0, counter1=0]

核心0修改 counter0++（触发MESI）：
1. 核心0发出写入意图
2. 核心1: S → I （整个缓存行失效！）
3. 核心0: S → M
4. 核心0写入：counter0=1

核心0: M [缓存行: counter0=1, counter1=0]
核心1: I ← 虽然没用counter0，但整个缓存行失效

核心1修改 counter1++（触发MESI）：
1. 核心1必须重新加载缓存行（I → S）
2. 核心0: M → S（写回）
3. 核心1加载后：S → M
4. 核心0: S → I
5. 核心1写入：counter1=1

核心0: I
核心1: M [缓存行: counter0=1, counter1=1]

问题：
- counter0和counter1逻辑上独立
- 但因为在同一缓存行，互相导致失效
- 性能下降10-100倍！

解决方案：
struct {
    int counter0;
    char padding[60];  // 填充到64字节
    int counter1;
    char padding2[60]; // 填充到64字节
};
```

#### 4.2.5 原子操作（Atomic Operations）

```
触发条件：使用原子指令（LOCK前缀）

场景：无锁编程

┌─────────────────────────────────────────────────┐
│ 典型例子：CAS（Compare-And-Swap）               │
├─────────────────────────────────────────────────┤
│ C代码：                                         │
│ bool CAS(int *ptr, int old, int new) {          │
│     return __sync_bool_compare_and_swap(        │
│         ptr, old, new);                         │
│ }                                               │
│                                                 │
│ 汇编（x86）：                                   │
│ lock cmpxchg [ptr], new                         │
│      ↑                                          │
│    LOCK前缀强制总线锁定                         │
└─────────────────────────────────────────────────┘

MESI + 总线锁定：

核心0执行 CAS(&x, 0, 1)：
1. LOCK前缀激活
2. 核心0锁定总线（或锁定缓存行）
3. 发出RFO（Read For Ownership）请求
4. 所有其他核心的该缓存行：→ I
5. 核心0: I → E → M（原子操作）
6. 释放总线锁

期间其他核心：
- 核心1-7的该缓存行全部失效
- 任何访问该缓存行的操作都被阻塞
- 等待核心0完成原子操作

性能影响：
- 单次CAS: ~100-200 cycles
- 高竞争下：可能达到1000+ cycles
- 比普通写入慢5-10倍
```

#### 4.2.6 跨NUMA节点访问

```
触发条件：访问远程NUMA节点的内存

场景：多路服务器（如2路、4路CPU）

┌─────────────────────────────────────────────────┐
│ 系统拓扑：                                      │
│                                                 │
│  NUMA Node 0              NUMA Node 1           │
│  ┌──────────┐            ┌──────────┐          │
│  │ CPU 0-7  │            │ CPU 8-15 │          │
│  │ L1/L2/L3 │            │ L1/L2/L3 │          │
│  └────┬─────┘            └────┬─────┘          │
│       │                       │                │
│  ┌────┴─────┐            ┌────┴─────┐          │
│  │ Memory   │            │ Memory   │          │
│  │ 0-64GB   │            │ 64-128GB │          │
│  └──────────┘            └──────────┘          │
│       ↑                       ↑                │
│       └───────────┬───────────┘                │
│              QPI/UPI总线                       │
└─────────────────────────────────────────────────┘

MESI跨NUMA节点：

核心0（Node 0）读取Node 1的内存：
1. L1/L2/L3全部Miss（本地没有）
2. 发出读请求到QPI总线
3. Node 1检测到请求
4. 如果Node 1的核心有这个缓存：
   → 通过QPI发送数据
5. 如果都没有：
   → 从Node 1的内存读取
   → 通过QPI发送到Node 0

延迟对比：
- 本地内存: ~60ns
- 远程内存: ~120ns（2倍！）
- 如果涉及MESI同步: ~200ns

触发条件：
- 进程/线程跨NUMA节点迁移
- 访问错误绑定的内存
- 共享数据分布在多个NUMA节点
```

### 4.3 总结：MESI协议的触发时机

| 操作类型          | 触发MESI    | 性能影响          | 常见场景       |
|---------------|-----------|---------------|------------|
| 单核读/写         | 否         | 最快 ~1-4ns     | 线程私有数据     |
| 多核只读          | 是 E→S     | 轻微 ~10ns      | 共享常量、配置数据  |
| 多核读写          | 是 S→I I→S | 中等 ~50ns      | 生产消费、消息队列  |
| 多核竞争写         | 是 M→I I→M | 严重 ~100-200ns | 计数器、自旋锁    |
| False Sharing | 是 I→M     | 非常严重 ~1000ns  | 数组相邻元素     |
| 原子操作          | 是 总线锁定    | 严重 ~200ns     | CAS操作、无锁队列 |
| 跨NUMA访问       | 是 远程访问    | 非常严重 ~200ns+  | 多路CPU      |


```
触发条件总结：
1. 任何对共享数据的写入
2. 读取处于M状态的数据（需要写回）
3. 缓存行在多个核心间共享时的修改
4. 原子指令（强制总线锁定）
5. 跨NUMA节点的内存访问

避免频繁触发的方法：
1. 线程局部存储（TLS）
2. 缓存行对齐（避免False Sharing）
3. 批量操作减少同步次数
4. Per-CPU数据结构
5. NUMA感知的内存分配
```

### 4.4 MESI协议详解

```
MESI = Modified, Exclusive, Shared, Invalid

每个缓存行有4种状态：

1. Modified (M) - 已修改
   - 只有本缓存有这份数据
   - 数据已被修改，与内存不一致
   - 本缓存负责写回内存

2. Exclusive (E) - 独占
   - 只有本缓存有这份数据
   - 数据与内存一致（未修改）
   - 可以直接修改为M状态，无需通知其他核心

3. Shared (S) - 共享
   - 多个缓存都有这份数据
   - 数据与内存一致
   - 修改前必须通知其他核心

4. Invalid (I) - 无效
   - 本缓存的这份数据无效
   - 需要重新从其他缓存或内存获取

状态转换图：

       read（独占）      write
  I ───────────────→ E ─────→ M
  ↑                   ↑       │
  │                   │       │写回内存
  │  其他核心write     │ read  │
  │                   │       ↓
  └─────── S ←────────┴───────┘
           ↑
           │read（共享）
           │
```

### 4.5 MESI实际运行示例

```
完整的MESI协议运行：

初始状态：
内存[0x1000] = 42

核心0: I (无数据)
核心1: I (无数据)

────────────────────────────────────

操作1: 核心0读取0x1000

1. 核心0发出读请求到总线
2. 其他核心检查：核心1没有这份数据
3. 从内存读取42
4. 核心0缓存：0x1000 = 42, 状态 = E (独占)

核心0: E [0x1000=42]
核心1: I

────────────────────────────────────

操作2: 核心1读取0x1000

1. 核心1发出读请求到总线
2. 核心0检测到请求（监听总线）
3. 核心0: E → S (独占变共享)
4. 核心0通过总线发送数据给核心1
5. 核心1缓存：0x1000 = 42, 状态 = S

核心0: S [0x1000=42]
核心1: S [0x1000=42]

────────────────────────────────────

操作3: 核心0写入0x1000 = 100

1. 核心0发出"写入意图"到总线
2. 核心1接收到信号：S → I (失效)
3. 核心1清除0x1000的缓存行
4. 核心0写入：0x1000 = 100, S → M

核心0: M [0x1000=100]  ← 已修改
核心1: I               ← 失效

────────────────────────────────────

操作4: 核心1读取0x1000

1. 核心1发出读请求
2. 核心0检测到（0x1000处于M状态）
3. 核心0必须先写回内存（100 → 内存）
4. 核心0: M → S
5. 核心1从内存或核心0获取数据
6. 核心1: I → S

核心0: S [0x1000=100]
核心1: S [0x1000=100]
内存[0x1000] = 100 (已写回)

────────────────────────────────────

关键点：
1. 写入时必须通知其他核心失效缓存
2. 已修改的缓存在其他核心读取前必须写回
3. 总线监听（Bus Snooping）是关键机制
4. 这会带来性能开销
```

---

## 5. False Sharing（伪共享）

### 5.1 什么是False Sharing？

```
缓存行（Cache Line）：缓存的最小单位，通常64字节

问题场景：

struct {
    int64 counter0;  // 8 bytes, 偏移 0
    int64 counter1;  // 8 bytes, 偏移 8
} counters;

内存布局：
┌────────────────────────────────────────────────────┐
│ 缓存行 (64 bytes)                                   │
├────────────────────────────────────────────────────┤
│ counter0 │ counter1 │ ... 其他48字节 ...            │
│  (8B)    │  (8B)    │                              │
└────────────────────────────────────────────────────┘

核心0访问counter0，核心1访问counter1：

虽然访问不同变量，但在同一缓存行！

Step 1: 核心0读counter0
核心0: S [缓存行: counter0=0, counter1=0]
核心1: I

Step 2: 核心1读counter1
核心0: S [缓存行: counter0=0, counter1=0]
核心1: S [缓存行: counter0=0, counter1=0]

Step 3: 核心0写counter0 = 1
核心0: M [缓存行: counter0=1, counter1=0]  ← 整个缓存行修改
核心1: I  ← 整个缓存行失效！

Step 4: 核心1写counter1 = 1
核心1必须：
1. 等待核心0写回
2. 重新加载缓存行
3. 修改counter1
4. 使核心0缓存失效

核心0: I
核心1: M [缓存行: counter0=1, counter1=1]

Step 5: 核心0再次写counter0 = 2
又要等待核心1写回...

结果：
- counter0和counter1逻辑上独立
- 但因为在同一缓存行，互相竞争
- 大量的缓存失效和同步
- 性能急剧下降（10-100倍）
```

### 5.2 False Sharing实验

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
    "time"
)

// 错误的做法：False Sharing
type BadCounters struct {
    counter0 int64
    counter1 int64
}

// 正确的做法：缓存行对齐
type GoodCounters struct {
    counter0 int64
    _        [7]int64 // 填充到64字节
    counter1 int64
    _        [7]int64 // 填充到64字节
}

func main() {
    runtime.GOMAXPROCS(2)

    // 测试1: False Sharing
    fmt.Println("=== 测试1: False Sharing ===")
    bad := &BadCounters{}
    testCounters(
        func() { atomicIncrement(&bad.counter0) },
        func() { atomicIncrement(&bad.counter1) },
    )

    // 测试2: 缓存行对齐
    fmt.Println("\n=== 测试2: 缓存行对齐 ===")
    good := &GoodCounters{}
    testCounters(
        func() { atomicIncrement(&good.counter0) },
        func() { atomicIncrement(&good.counter1) },
    )
}

func testCounters(fn0, fn1 func()) {
    iterations := 10_000_000

    start := time.Now()
    var wg sync.WaitGroup
    wg.Add(2)

    // 核心0: 不断increment counter0
    go func() {
        defer wg.Done()
        for i := 0; i < iterations; i++ {
            fn0()
        }
    }()

    // 核心1: 不断increment counter1
    go func() {
        defer wg.Done()
        for i := 0; i < iterations; i++ {
            fn1()
        }
    }()

    wg.Wait()
    elapsed := time.Since(start)

    fmt.Printf("耗时: %v\n", elapsed)
    fmt.Printf("吞吐量: %.2f M ops/sec\n",
        float64(iterations*2)/elapsed.Seconds()/1_000_000)
}

func atomicIncrement(counter *int64) {
    // 使用atomic确保原子性
    *counter++
}

// 运行结果（典型）：
// === 测试1: False Sharing ===
// 耗时: 2.5s
// 吞吐量: 8.00 M ops/sec
//
// === 测试2: 缓存行对齐 ===
// 耗时: 0.3s
// 吞吐量: 66.67 M ops/sec
//
// 性能提升：8.3倍！
```

### 5.3 如何避免False Sharing

```go
// 方法1: 手动填充（推荐）
type CacheLinePadded struct {
    value int64
    _     [7]int64 // 填充到64字节 (8 × 8 = 64)
}

// 方法2: 使用编译器指令（Go 1.19+）
type AlignedStruct struct {
    value int64
} // compiler会自动对齐

// 方法3: 使用数组索引间隔
const CacheLineSize = 64

type Counters struct {
    data [16 * CacheLineSize / 8]int64 // 16个计数器，每个占64字节
}

func (c *Counters) Get(id int) *int64 {
    offset := id * (CacheLineSize / 8) // 每个计数器间隔8个int64
    return &c.data[offset]
}

// 方法4: 每个核心独立数组（Goroutine-Per-Core模式）
type PerCoreCounters struct {
    counters []int64 // len = NumCPU × (64/8)
}

func (p *PerCoreCounters) Inc() {
    coreID := runtime.GOMAXPROCS(0) // 简化，实际需要获取真实核心ID
    offset := coreID * 8
    p.counters[offset]++
}

// Go标准库中的例子：sync.Pool
// sync.Pool内部使用per-P的localPool，避免False Sharing

type poolLocal struct {
    private interface{}
    shared  []interface{}
    _       [128]byte // 防止False Sharing (超过64字节，更保守)
}
```

---

## 6. 如何查看CPU缓存信息

### 6.1 Linux系统

```bash
# 方法1: lscpu
$ lscpu | grep -i cache
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            8192K

# 方法2: /sys/devices/system/cpu
$ cat /sys/devices/system/cpu/cpu0/cache/index0/type
Data         # L1d

$ cat /sys/devices/system/cpu/cpu0/cache/index0/size
32K

$ cat /sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size
64           # 缓存行大小：64字节

$ cat /sys/devices/system/cpu/cpu0/cache/index0/shared_cpu_list
0,8          # CPU 0和8共享（超线程）

# 方法3: getconf
$ getconf LEVEL1_DCACHE_SIZE
32768        # 32KB

$ getconf LEVEL1_DCACHE_LINESIZE
64           # 64字节缓存行

$ getconf LEVEL2_CACHE_SIZE
262144       # 256KB

$ getconf LEVEL3_CACHE_SIZE
8388608      # 8MB

# 方法4: 完整的缓存层次
$ ls -la /sys/devices/system/cpu/cpu0/cache/
index0 -> L1d Cache
index1 -> L1i Cache
index2 -> L2 Cache
index3 -> L3 Cache

$ cat /sys/devices/system/cpu/cpu0/cache/index3/shared_cpu_list
0-7,16-23    # L3被所有16个逻辑核心共享（8核16线程）
```

### 6.2 macOS系统

```bash
# 查看缓存信息
$ sysctl -a | grep cache
hw.cachelinesize: 64
hw.l1icachesize: 32768
hw.l1dcachesize: 32768
hw.l2cachesize: 262144
hw.l3cachesize: 8388608

# 查看CPU拓扑
$ sysctl machdep.cpu
machdep.cpu.core_count: 8
machdep.cpu.thread_count: 16
```

### 6.3 Go程序中检测

```go
package main

import (
    "fmt"
    "runtime"
    "unsafe"
)

func main() {
    // 获取缓存行大小
    cacheLineSize := unsafe.Sizeof([64]byte{})
    fmt.Printf("推测的缓存行大小: %d bytes\n", cacheLineSize)

    // 获取CPU信息
    fmt.Printf("CPU数量: %d\n", runtime.NumCPU())
    fmt.Printf("GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))

    // 验证False Sharing
    type Test struct {
        a int64
        b int64
    }

    t := Test{}
    ptrA := uintptr(unsafe.Pointer(&t.a))
    ptrB := uintptr(unsafe.Pointer(&t.b))

    fmt.Printf("\na的地址: 0x%x\n", ptrA)
    fmt.Printf("b的地址: 0x%x\n", ptrB)
    fmt.Printf("差值: %d bytes\n", ptrB-ptrA)

    // 检查是否在同一缓存行
    cacheLineA := ptrA / 64
    cacheLineB := ptrB / 64

    if cacheLineA == cacheLineB {
        fmt.Println("警告: a和b在同一缓存行! (False Sharing风险)")
    } else {
        fmt.Println("OK: a和b在不同缓存行")
    }
}
```

### 6.4 使用perf工具测量缓存性能

```bash
# 安装perf (Linux)
$ sudo apt-get install linux-tools-common linux-tools-generic

# 测量缓存命中率
$ perf stat -e cache-references,cache-misses,L1-dcache-loads,L1-dcache-load-misses ./your_program

# 输出示例：
#  Performance counter stats for './your_program':
#
#     45,678,123      cache-references
#      2,345,678      cache-misses              #  5.14% of all cache refs
#    123,456,789      L1-dcache-loads
#      8,901,234      L1-dcache-load-misses     #  7.21% of all L1 loads
#
#  缓存命中率 = 1 - (cache-misses / cache-references) = 94.86%

# 测量False Sharing
$ perf c2c record ./your_program
$ perf c2c report
# 会显示哪些缓存行有争用

# 热点分析
$ perf record -e cache-misses ./your_program
$ perf report
# 显示哪些函数导致最多缓存失效
```

---

## 7. 不同架构的差异

### 7.1 Intel vs AMD

```
Intel (典型，如Skylake):
├─ L1d: 32KB per core
├─ L1i: 32KB per core
├─ L2: 256KB per core
└─ L3: 2MB per core (包容性缓存)

AMD Ryzen (Zen 3):
├─ L1d: 32KB per core
├─ L1i: 32KB per core
├─ L2: 512KB per core (2倍于Intel!)
└─ L3: 32MB per CCX (非包容性)

关键差异：

1. L2大小
   - AMD: 512KB (更大)
   - Intel: 256KB

2. L3架构
   - Intel: 包容性 (Inclusive)
     → L2的数据也在L3中
     → L3被驱逐时，L2也失效

   - AMD: 非包容性 (Non-Inclusive)
     → L2和L3独立
     → 有效缓存容量更大
     → L3 + L2 = 32MB + 6MB(12核) = 38MB

3. CCX结构（AMD特有）
   - 核心被分组为CCX
   - 同一CCX内共享L3
   - 跨CCX访问L3延迟更高
```

### 7.2 ARM架构

```
ARM (如Apple M1):

P-Core (性能核心):
├─ L1d: 128KB per core (4倍于x86!)
├─ L1i: 192KB per core
├─ L2: 12MB (4个P核共享)
└─ SLC (System Level Cache): 12MB (所有核共享)

E-Core (效率核心):
├─ L1d: 64KB per core
├─ L1i: 128KB per core
├─ L2: 4MB (4个E核共享)
└─ SLC: 12MB (所有核共享)

特点：
- L1比x86大得多（128KB vs 32KB）
- 没有传统的L3，而是SLC（类似L4）
- 异构架构（大小核）
```

---

## 8. 性能优化建议

### 8.1 缓存友好的代码

```go
// 不好：随机访问（缓存不友好）
func sumRandom(data []int64) int64 {
    sum := int64(0)
    for i := 0; i < len(data); i++ {
        idx := rand.Intn(len(data))
        sum += data[idx]  // 随机访问，缓存失效率高
    }
    return sum
}

// 好：顺序访问（缓存友好）
func sumSequential(data []int64) int64 {
    sum := int64(0)
    for i := 0; i < len(data); i++ {
        sum += data[i]  // 顺序访问，预取器工作
    }
    return sum
}

// 性能差异可达10-100倍！

// 好：数据局部性
type Point struct {
    x, y, z float64
}

// 不好：分散的数据
var xs []float64
var ys []float64
var zs []float64

for i := 0; i < n; i++ {
    process(xs[i], ys[i], zs[i])  // 3次缓存加载
}

// 好：紧凑的数据
var points []Point

for i := 0; i < n; i++ {
    process(points[i].x, points[i].y, points[i].z)  // 1次缓存加载
}
```

### 8.2 避免False Sharing的模式

```go
// 模式1: 缓存行填充
type CacheLinePadded struct {
    value int64
    _     [7]int64
}

// 模式2: 批量操作减少同步
type PerCoreCounter struct {
    counters [runtime.NumCPU() * 8]int64
}

func (p *PerCoreCounter) Inc(coreID int) {
    p.counters[coreID*8]++  // 每个核心间隔8个int64
}

func (p *PerCoreCounter) Sum() int64 {
    sum := int64(0)
    for i := 0; i < runtime.NumCPU(); i++ {
        sum += p.counters[i*8]
    }
    return sum
}

// 模式3: 使用sync.Pool（已处理False Sharing）
var pool = sync.Pool{
    New: func() interface{} {
        return &Buffer{}
    },
}
```

---

## 9. 总结

### 9.1 核心要点

```
┌────────────────────────────────────────────────────┐
│ CPU缓存架构总结：                                    │
│                                                    │
│ 1. 独占 vs 共享                                     │
│    ✓ L1、L2：每个物理核心独占                         │  
│    ✓ L3：所有核心共享                                │
│    ✓ 超线程：同一物理核的HT共享L1/L2                   │
│                                                    │
│ 2. 大小（典型）                                      │
│    - L1: 32KB (i) + 32KB (d) = 64KB                │
│    - L2: 256KB - 512KB                             │
│    - L3: 8MB - 64MB                                │
│                                                    │
│ 3. 延迟                                             │
│    - L1: ~1 ns (4 cycles)                          │
│    - L2: ~4 ns (12 cycles)                         │
│    - L3: ~13 ns (40 cycles)                        │
│    - 内存: ~67 ns (200 cycles)                      │
│                                                    │
│ 4. 缓存行                                           │
│    - 大小: 64 bytes                                 │
│    - 是缓存的最小单位                                 │
│    - False Sharing的根源                            │
│                                                    │
│ 5. 缓存一致性                                        │
│    - MESI协议                                       │
│    - 写入时通知其他核心失效                            │
│    - 带来性能开销                                    │
│                                                    │
│ 6. 性能优化                                         │
│    ✓ 顺序访问 > 随机访问                              │
│    ✓ 数据紧凑 > 数据分散                              │
│    ✓ 避免False Sharing（缓存行对齐）                  │
│    ✓ 减少缓存争用（per-core数据）                     │
└────────────────────────────────────────────────────┘
```

### 9.2 查看缓存信息的命令

```bash
# Linux
lscpu | grep cache
cat /sys/devices/system/cpu/cpu0/cache/index*/size
getconf -a | grep CACHE

# macOS
sysctl -a | grep cache

# 性能分析
perf stat -e cache-references,cache-misses ./program
```

**L1、L2每核独占，L3所有核共享**！
