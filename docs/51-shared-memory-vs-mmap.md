# 共享内存 vs 内存映射：深度对比

## 1. 概念澄清

### 1.1 术语定义

```
进程间通信（IPC）的三种"共享内存"技术：

1. System V 共享内存
   - API: shmget() / shmat() / shmdt()
   - 最传统的方式

2. POSIX 共享内存
   - API: shm_open() + mmap()
   - 现代推荐方式

3. 文件内存映射
   - API: open() + mmap()
   - 映射普通文件或特殊文件（如/dev/shm）

关键理解：
- "共享内存"通常指前两种（System V 和 POSIX）
- "内存映射"通常指 mmap（但POSIX共享内存底层也用mmap）
```

---

## 2. 底层实现原理

### 2.1 System V 共享内存（shmget）

```
原理：
┌──────────────────────────────────────────────────┐
│ 内核维护全局共享内存段表                             │
│ ┌────────────┬──────────┬──────────────────────┐ │
│ │ shmid=100  │ key=0x12 │ 物理内存：0xABCD0000   │ │
│ └────────────┴──────────┴──────────────────────┘ │
└──────────────────────────────────────────────────┘
       ↓ shmat()                    ↓ shmat()
┌─────────────────┐          ┌─────────────────┐
│  进程A           │          │  进程B          │
│  虚拟地址空间     │          │  虚拟地址空间     │
│  ┌────────────┐ │          │  ┌────────────┐ │
│  │ 0x70000000 │ │          │  │ 0x80000000 │ │
│  │     ↓      │ │          │  │     ↓      │ │
│  │  共享内存   │ │          │  │  共享内存    │ │
│  └────────────┘ │          │  └────────────┘ │
│       ↓         │          │       ↓         │
│    映射到物理内存 0xABCD0000 (同一块！)          │
└─────────────────┘          └─────────────────┘

特点：
1. 内核维护全局段表（所有进程可见）
2. 通过key或shmid标识唯一的共享内存段
3. shmat()将段映射到进程的虚拟地址空间
4. 虚拟地址可能不同，但物理地址相同
5. 生命周期独立于进程（需要显式删除）
```

### 2.2 POSIX 共享内存（shm_open + mmap）

```
原理：
┌──────────────────────────────────────────────────┐
│ /dev/shm/ 文件系统（tmpfs，在内存中）                │
│ ┌────────────────────────────────────────────┐   │
│ │ /dev/shm/orderbook → inode → 物理内存块      │   │
│ └────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
       ↓ mmap(MAP_SHARED)         ↓ mmap(MAP_SHARED)
┌─────────────────┐          ┌─────────────────┐
│  进程A           │          │  进程B          │
│  ┌────────────┐ │          │  ┌────────────┐ │
│  │ 0x70000000 │ │          │  │ 0x80000000 │ │
│  │     ↓      │ │          │  │     ↓      │ │
│  │  文件映射   │ │          │  │  文件映射    │ │
│  └────────────┘ │          │  └────────────┘ │
│       ↓         │          │       ↓         │
│    映射到同一个物理内存块（通过inode关联）         │
└─────────────────┘          └─────────────────┘

特点：
1. 基于文件系统（但文件在内存中，不写磁盘）
2. 通过文件名（如"/orderbook"）标识
3. shm_open()创建文件，mmap()映射
4. 本质上是特殊的文件映射
5. 遵循POSIX标准，可移植性好
```

### 2.3 文件内存映射（mmap普通文件）

```
原理：
┌──────────────────────────────────────────────────┐
│ 磁盘文件：/data/market.bin                         │
│       ↓ 按需加载                                  │
│ 页缓存（Page Cache）                              │
│ ┌────────────────────────────────────────────┐   │
│ │ 物理内存：缓存文件的部分内容                    │   │
│ └────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
       ↓ mmap(MAP_SHARED)         ↓ mmap(MAP_SHARED)
┌─────────────────┐          ┌─────────────────┐
│  进程A           │          │  进程B          │
│  ┌────────────┐ │          │  ┌────────────┐ │
│  │ 虚拟地址    │ │          │  │ 虚拟地址     │ │
│  │     ↓      │ │          │  │     ↓      │ │
│  │  文件映射   │ │          │  │  文件映射    │ │
│  └────────────┘ │          │  └────────────┘ │
│       ↓         │          │       ↓         │
│    映射到页缓存（通过inode关联）                 │
└─────────────────┘          └─────────────────┘
       ↓ 写入时                     ↓
   脏页回写到磁盘（异步）

特点：
1. 映射真实的磁盘文件
2. 按需加载（页面错误时从磁盘读取）
3. 修改会写回磁盘（延迟写入）
4. 可以实现文件的零拷贝访问
5. 持久化存储
```

---

## 3. 完整代码对比

### 3.1 System V 共享内存

```c
#include <sys/shm.h>
#include <sys/ipc.h>
#include <stdio.h>
#include <string.h>

struct Data 
{
    int counter;
    char message[256];
};

// ========== 进程A：创建和写入 ==========
void process_a() 
{
    // 1. 生成唯一key
    key_t key = ftok("/tmp", 'R');

    // 2. 创建共享内存段（如果不存在）
    int shmid = shmget(key, sizeof(struct Data), IPC_CREAT | 0666);
    printf("shmid = %d\n", shmid);

    // 3. 附加到进程地址空间
    struct Data* data = (struct Data*)shmat(shmid, NULL, 0);
    printf("附加到地址: %p\n", data);

    // 4. 写入数据
    data->counter = 42;
    strcpy(data->message, "Hello from Process A");

    // 5. 分离（但不删除）
    shmdt(data);

    // 注意：共享内存段仍然存在！
    // 需要显式删除：shmctl(shmid, IPC_RMID, NULL);
}

// ========== 进程B：读取 ==========
void process_b() 
{
    // 1. 使用相同的key
    key_t key = ftok("/tmp", 'R');

    // 2. 获取已存在的共享内存段
    int shmid = shmget(key, sizeof(struct Data), 0666);

    // 3. 附加
    struct Data* data = (struct Data*)shmat(shmid, NULL, 0);
    printf("附加到地址: %p\n", data);

    // 4. 读取数据
    printf("counter = %d\n", data->counter);
    printf("message = %s\n", data->message);

    // 5. 分离
    shmdt(data);
}

// ========== 清理 ==========
void cleanup() 
{
    key_t key = ftok("/tmp", 'R');
    int shmid = shmget(key, 0, 0666);

    // 删除共享内存段
    shmctl(shmid, IPC_RMID, NULL);
}

// 查看系统中的共享内存段
// $ ipcs -m
// $ ipcrm -m <shmid>  # 手动删除
```

### 3.2 POSIX 共享内存

```c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

struct Data 
{
    int counter;
    char message[256];
};

// ========== 进程A：创建和写入 ==========
void process_a() 
{
    const char* name = "/my_shm";  // 名字以'/'开头

    // 1. 创建共享内存对象（类似文件）
    int fd = shm_open(name, O_CREAT | O_RDWR, 0666);
    printf("fd = %d\n", fd);

    // 2. 设置大小（必须！新创建的对象大小为0）
    ftruncate(fd, sizeof(struct Data));

    // 3. 映射到内存
    struct Data* data = (struct Data*)mmap(NULL, sizeof(struct Data),
                                           PROT_READ | PROT_WRITE,
                                           MAP_SHARED, fd, 0);
    printf("映射到地址: %p\n", data);

    // 4. 可以立即关闭fd（映射仍有效）
    close(fd);

    // 5. 写入数据
    data->counter = 42;
    strcpy(data->message, "Hello from Process A");

    // 6. 解除映射
    munmap(data, sizeof(struct Data));

    // 注意：共享内存对象仍然存在！
    // 需要显式删除：shm_unlink(name);
}

// ========== 进程B：读取 ==========
void process_b() 
{
    const char* name = "/my_shm";

    // 1. 打开已存在的共享内存对象
    int fd = shm_open(name, O_RDWR, 0666);

    // 2. 映射到内存
    struct Data* data = (struct Data*)mmap(NULL, sizeof(struct Data),
                                           PROT_READ | PROT_WRITE,
                                           MAP_SHARED, fd, 0);
    close(fd);

    // 3. 读取数据
    printf("counter = %d\n", data->counter);
    printf("message = %s\n", data->message);

    // 4. 解除映射
    munmap(data, sizeof(struct Data));
}

// ========== 清理 ==========
void cleanup() {
    shm_unlink("/my_shm");  // 删除共享内存对象
}

// 查看系统中的POSIX共享内存
// $ ls -lh /dev/shm/
```

### 3.3 文件内存映射（mmap磁盘文件）

```c
#include <sys/mman.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

struct Data 
{
    int counter;
    char message[256];
};

// ========== 进程A：创建和写入 ==========
void process_a() 
{
    const char* filename = "/tmp/shared_file.bin";

    // 1. 创建/打开普通文件
    int fd = open(filename, O_CREAT | O_RDWR, 0666);

    // 2. 设置文件大小
    ftruncate(fd, sizeof(struct Data));

    // 3. 映射到内存
    struct Data* data = (struct Data*)mmap(NULL, sizeof(struct Data),
                                           PROT_READ | PROT_WRITE,
                                           MAP_SHARED, fd, 0);
    close(fd);

    // 4. 写入数据（会自动同步到文件）
    data->counter = 42;
    strcpy(data->message, "Hello from Process A");

    // 5. 强制同步到磁盘
    msync(data, sizeof(struct Data), MS_SYNC);

    // 6. 解除映射
    munmap(data, sizeof(struct Data));
}

// ========== 进程B：读取 ==========
void process_b() 
{
    const char* filename = "/tmp/shared_file.bin";

    // 1. 打开文件
    int fd = open(filename, O_RDWR, 0666);

    // 2. 映射到内存
    struct Data* data = (struct Data*)mmap(NULL, sizeof(struct Data),
                                           PROT_READ | PROT_WRITE,
                                           MAP_SHARED, fd, 0);
    close(fd);

    // 3. 读取数据（从内存，如果页面在缓存中）
    printf("counter = %d\n", data->counter);
    printf("message = %s\n", data->message);

    // 4. 解除映射
    munmap(data, sizeof(struct Data));
}
```

---

## 4. 详细对比表

### 4.1 功能对比

| 特性       | System V 共享内存 | POSIX 共享内存      | mmap磁盘文件    |
|----------|---------------|-----------------|-------------|
| **API**  | shmget/shmat  | shm_open + mmap | open + mmap |
| **标识符**  | key_t（数字）     | 文件名（字符串）        | 文件路径        |
| **位置**   | 内核段表          | /dev/shm/       | 任意路径        |
| **持久化**  | 内存（重启丢失）      | 内存（重启丢失）        | 磁盘（持久化）     |
| **可移植性** | Unix特定        | POSIX标准         | 跨平台         |
| **生命周期** | 独立于进程         | 独立于进程           | 跟随文件        |
| **大小调整** | 不支持           | ftruncate       | ftruncate   |
| **权限控制** | 权限位（0666）     | 文件权限            | 文件权限        |

### 4.2 性能对比

| 维度        | System V  | POSIX     | mmap文件         |
|-----------|-----------|-----------|----------------|
| **创建开销**  | 低（~1μs）   | 低（~1μs）   | 中（需创建文件）       |
| **访问延迟**  | 极低（~50ns） | 极低（~50ns） | 低（页缓存命中）/高（缺页） |
| **内存效率**  | 高（纯内存）    | 高（纯内存）    | 中（受页缓存限制）      |
| **I/O开销** | 无         | 无         | 有（写回磁盘）        |

### 4.3 使用场景对比

| 场景          | 推荐方案          | 原因                 |
|-------------|---------------|--------------------|
| **高频交易订单簿** | POSIX共享内存     | 纯内存访问，延迟最低，POSIX标准 |
| **跨进程缓存**   | POSIX共享内存     | 简单、高效、现代           |
| **大文件处理**   | mmap磁盘文件      | 零拷贝读写，按需加载         |
| **日志分析**    | mmap磁盘文件      | 避免read()系统调用       |
| **进程间消息队列** | POSIX共享内存 + 锁 | 配合Seqlock实现无锁队列    |
| **持久化数据**   | mmap磁盘文件      | 自动同步到磁盘            |
| **老系统兼容**   | System V共享内存  | 兼容老代码              |

---

## 5. 原理深度解析

### 5.1 为什么POSIX共享内存比System V更推荐？

```
1. 语义更清晰
   System V:  shmget(key, ...) → 需要ftok()生成key，容易冲突
   POSIX:     shm_open("/my_shm", ...) → 直观的文件名

2. 可移植性
   System V:  Unix特定，Windows不支持
   POSIX:     标准接口，跨平台

3. 权限管理
   System V:  权限位难以管理
   POSIX:     文件系统权限，易于理解

4. 生命周期
   System V:  需要ipcrm手动删除，容易泄漏
   POSIX:     shm_unlink()显式删除，语义清晰

5. 调试
   System V:  ipcs查看，不直观
   POSIX:     ls /dev/shm/ 直接查看
```

### 5.2 mmap的页面错误处理（Page Fault）

```
场景：进程访问mmap映射的文件区域

时间线：
T1: 进程执行 data[1000] = 42;
    ↓
    CPU检查页表：地址0x70001000对应的页面不在物理内存
    ↓
T2: 触发页面错误（Page Fault）中断
    ↓
    内核页面错误处理程序：
    1. 检查虚拟地址是否合法（在mmap的范围内？）
    2. 如果是文件映射：
       a. 从磁盘读取对应的文件页（阻塞I/O！）
       b. 分配物理页帧
       c. 拷贝文件数据到物理页
       d. 更新页表：虚拟地址 → 物理地址
    3. 如果是匿名映射（如POSIX共享内存）：
       a. 分配物理页帧
       b. 清零
       c. 更新页表
    ↓
T3: 返回用户态，重新执行指令
    ↓
    data[1000] = 42;  // 这次成功！

性能影响：
- 匿名映射（POSIX共享内存）：页面错误延迟 ~1-5μs
- 文件映射（mmap磁盘文件）：页面错误延迟 ~5-10ms（磁盘I/O！）
```

### 5.3 写时复制（Copy-on-Write, COW）

```
MAP_PRIVATE vs MAP_SHARED：

MAP_SHARED（共享映射）：
┌─────────┐     ┌─────────┐
│ 进程A    │     │ 进程B   │
│ 写入42   │ ←→  │ 读取42  │  ← 看到相同的数据
└─────────┘     └─────────┘
       ↓             ↓
    同一块物理内存（0xABCD0000）

MAP_PRIVATE（私有映射，写时复制）：
┌─────────┐     ┌─────────┐
│ 进程A    │     │ 进程B   │
│ 写入42   │  ✗  │ 读取0   │  ← 看到不同的数据
└─────────┘     └─────────┘
       ↓             ↓
初始：共享同一块物理内存
       ↓
进程A写入时：
  1. 触发页面错误
  2. 内核分配新物理页（拷贝）
  3. 进程A的页表指向新页
  4. 进程B仍指向原页

用途：
- MAP_SHARED：进程间通信（IPC）
- MAP_PRIVATE：fork()后父子进程的内存优化
```

---

## 6. 优缺点总结

### 6.1 System V 共享内存

**优点**：
- 性能极高（纯内存访问，~50ns延迟）
- 成熟稳定（使用数十年）
- 生命周期独立于进程（崩溃不影响）

**缺点**：
- API复杂（ftok、key管理）
- 不支持动态调整大小
- 容易资源泄漏（需要手动ipcrm删除）
- 可移植性差（Unix特定）
- 调试困难（ipcs不直观）

### 6.2 POSIX 共享内存

**优点**：
- 性能极高（纯内存访问，~50ns延迟）
- API简洁（基于文件名）
- POSIX标准，可移植性好
- 支持动态调整大小（ftruncate）
- 调试友好（ls /dev/shm/）
- 权限管理清晰（文件权限）

**缺点**：
- 生命周期独立于进程（需手动unlink）
- 重启后数据丢失

### 6.3 mmap磁盘文件

**优点**：
- 持久化存储（数据写回磁盘）
- 零拷贝访问（避免read/write）
- 按需加载（节省内存）
- 支持大文件（超过物理内存）
- 跨进程共享文件数据

**缺点**：
- 首次访问有页面错误开销（磁盘I/O，~5-10ms）
- 写入有磁盘I/O开销
- 不适合高频随机访问
- 需要管理磁盘空间

---

## 7. 高频交易场景的最佳实践

```c
// 推荐：POSIX共享内存 + Seqlock

// 1. 市场数据进程（写者）
struct OrderBook 
{
    _Atomic uint32_t seq;
    uint64_t timestamp;
    double bid[100];
    double ask[100];
};

void market_data_process() 
{
    // 创建POSIX共享内存
    int fd = shm_open("/orderbook", O_CREAT | O_RDWR, 0666);
    ftruncate(fd, sizeof(struct OrderBook));

    struct OrderBook* book = mmap(NULL, sizeof(struct OrderBook), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    close(fd);

    while (1) 
    {
        // Seqlock写入
        uint32_t seq = atomic_load(&book->seq);
        atomic_store(&book->seq, seq + 1);  // 奇数=正在写

        update_market_data(book);

        atomic_store(&book->seq, seq + 2);  // 偶数=写完成
    }
}

// 2. 策略进程（读者，无锁！）
void strategy_process() 
{
    int fd = shm_open("/orderbook", O_RDWR, 0666);
    struct OrderBook* book = mmap(NULL, sizeof(struct OrderBook), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    close(fd);

    while (1) 
    {
        uint32_t seq1, seq2;
        double bid;

        do 
        {
            seq1 = atomic_load(&book->seq);
            if (seq1 & 1) continue;  // 写者正在写

            bid = book->bid[0];  // 读取数据

            seq2 = atomic_load(&book->seq);
        } while (seq1 != seq2);  // 版本号变了，重试

        if (bid > threshold) send_order();
    }
}

// 性能：
// - 写者：~5ns（仅原子操作）
// - 读者：~3ns（完全无锁，无系统调用）
// - 延迟抖动：< 10ns
```

---

## 8. 面试回答模板

**问题**："共享内存和mmap有什么区别？"

**标准答案**：

"这个问题需要区分三种技术：

**1. System V共享内存（shmget）**：
- 内核维护全局段表，通过key标识
- 纯内存存储，不涉及文件系统
- 性能最高但API较复杂，属于旧标准

**2. POSIX共享内存（shm_open + mmap）**：
- 基于/dev/shm/内存文件系统
- 数据在内存中，不写磁盘
- 本质是特殊的mmap，推荐使用

**3. 文件mmap（普通文件）**：
- 映射磁盘文件到内存
- 支持持久化，但有磁盘I/O开销
- 适合大文件处理

**核心区别**：
- 前两者纯内存，适合IPC（延迟~50ns）
- 文件mmap涉及磁盘，适合文件I/O（首次访问~5ms）

**底层原理**：
所有方式都通过页表将虚拟地址映射到同一块物理内存，实现零拷贝共享。区别在于物理内存的来源（内核段、tmpfs、页缓存）和生命周期管理。

**高频交易场景**：
推荐POSIX共享内存 + Seqlock，读者完全无锁，延迟可达3纳秒。"

---

## 9. 实验验证

```bash
# 实验1：观察POSIX共享内存
$ ls -lh /dev/shm/
$ df -h /dev/shm/  # 查看tmpfs大小

# 实验2：对比性能
$ perf stat -e cache-misses,page-faults ./shm_test
$ perf stat -e cache-misses,page-faults ./mmap_test

# 实验3：观察页面错误
$ strace -e mmap,mmap2,munmap ./program

# 实验4：查看System V共享内存
$ ipcs -m  # 列出所有段
$ ipcs -m -i <shmid>  # 查看详情
```
