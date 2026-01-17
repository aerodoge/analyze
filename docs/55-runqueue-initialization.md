# 运行队列的创建和绑定过程

## 核心答案

**运行队列在Linux内核启动时创建，每个CPU一个，通过Per-CPU变量机制静态分配。**

```
时间线：

系统启动
    ↓
内核初始化（start_kernel）
    ↓
调度器初始化（sched_init）
    ↓
为每个可能的CPU创建运行队列（静态分配）
    ↓
CPU上线（cpu_online）
    ↓
运行队列与CPU核心绑定（激活）
    ↓
调度器开始工作
```

---

## 1. 内核启动过程中的调度器初始化

### 1.1 启动流程概览

```c
// init/main.c - Linux内核启动入口

asmlinkage __visible void __init start_kernel(void)
{
    char *command_line;

    // 1. 早期初始化
    setup_arch(&command_line);      // 架构相关初始化
    setup_per_cpu_areas();          // Per-CPU区域初始化（关键！）

    // 2. 调度器初始化（关键！）
    sched_init();                   // 创建运行队列

    // 3. 其他子系统初始化
    trap_init();                    // 中断/异常处理
    mm_init();                      // 内存管理
    time_init();                    // 时间管理

    // 4. 启动第一个用户进程
    rest_init();                    // 创建init进程
}

关键步骤：
1. setup_per_cpu_areas() - 为每个CPU分配Per-CPU数据区域
2. sched_init() - 初始化调度器，创建运行队列
```

---

## 2. Per-CPU变量机制

### 2.1 Per-CPU变量的原理

```c
// 运行队列的定义（kernel/sched/core.c）

// 声明Per-CPU变量（每个CPU一份）
DEFINE_PER_CPU_SHARED_ALIGNED(struct rq, runqueues);

// 展开后等价于：
struct rq runqueues __percpu __aligned(64);

// 实际内存布局：
┌────────────────────────────────────────────────┐
│ Per-CPU数据区域（在内核内存中）                    │
│                                                │
│ CPU 0的Per-CPU区域：                            │
│ ┌──────────────────────────────────────────┐   │
│ │ struct rq runqueues (CPU 0的运行队列)      │   │
│ │ - curr: NULL                             │   │
│ │ - cfs: {...}                             │   │
│ │ - nr_running: 0                          │   │
│ └──────────────────────────────────────────┘   │
│                                                │
│ CPU 1的Per-CPU区域：                            │
│ ┌──────────────────────────────────────────┐   │
│ │ struct rq runqueues (CPU 1的运行队列)      │   │
│ │ - curr: NULL                             │   │
│ │ - cfs: {...}                             │   │
│ │ - nr_running: 0                          │   │
│ └──────────────────────────────────────────┘   │
│                                                │
│ CPU 2的Per-CPU区域：                            │
│ ...                                            │
└────────────────────────────────────────────────┘

关键理解：
1. 每个CPU有独立的内存区域存放Per-CPU变量
2. 运行队列是Per-CPU变量，所以每个CPU有独立的一份
3. 这些内存在内核启动时就分配好了（静态分配）
```

### 2.2 访问Per-CPU变量

```c
// 访问当前CPU的运行队列
struct rq *this_rq(void)
{
    // 方式1：通过per_cpu宏
    return this_cpu_ptr(&runqueues);
}

// 访问指定CPU的运行队列
struct rq *cpu_rq(int cpu)
{
    // 方式2：通过cpu编号
    return per_cpu_ptr(&runqueues, cpu);
}

// 底层实现（x86-64）：
// 1. 每个CPU的GS段寄存器指向自己的Per-CPU区域基地址
// 2. 访问Per-CPU变量：GS:[offset]

static inline struct rq *this_rq(void)
{
    // 内联汇编（简化）
    unsigned long offset = __per_cpu_offset[smp_processor_id()];
    return (struct rq *)(percpu_base + offset + offsetof_runqueues);
}

// 示例：
CPU 0: GS = 0xFFFF888000000000 → runqueues在 GS:[0x1000]
CPU 1: GS = 0xFFFF888001000000 → runqueues在 GS:[0x1000]
CPU 2: GS = 0xFFFF888002000000 → runqueues在 GS:[0x1000]

每个CPU通过相同的偏移量访问，但GS不同，所以访问到不同的内存！
```

---

## 3. 调度器初始化：sched_init()

### 3.1 完整的初始化代码

```c
// kernel/sched/core.c

void __init sched_init(void)
{
    unsigned long ptr = 0;
    int i;

    // 1. 遍历所有可能的CPU（possible CPUs）
    for_each_possible_cpu(i) 
    {
        struct rq *rq;

        // 2. 获取CPU i的运行队列
        rq = cpu_rq(i);

        // 3. 初始化运行队列的锁
        raw_spin_lock_init(&rq->lock);

        // 4. 初始化运行队列的字段
        rq->nr_running = 0;
        rq->nr_switches = 0;
        rq->clock = 0;
        rq->cpu = i;  // 绑定到CPU编号i

        // 5. 初始化CFS调度器的红黑树
        init_cfs_rq(&rq->cfs);
        rq->cfs.tasks_timeline = RB_ROOT_CACHED;

        // 6. 初始化实时调度器
        init_rt_rq(&rq->rt);

        // 7. 初始化deadline调度器
        init_dl_rq(&rq->dl);

        // 8. 设置当前任务（启动时是init_task）
        rq->curr = &init_task;
        rq->idle = &init_task;
    }

    // 9. 其他初始化
    scheduler_running = 1;
}

关键点：
1. for_each_possible_cpu(i) - 遍历所有可能的CPU（包括未上线的）
2. cpu_rq(i) - 获取CPU i的运行队列（通过Per-CPU变量）
3. 初始化时，运行队列已经在内存中（Per-CPU区域），只是填充初始值
4. rq->cpu = i - 记录这个运行队列属于哪个CPU
```

### 3.2 for_each_possible_cpu 的含义

```c
// 可能的CPU（possible CPUs）：系统配置的最大CPU数
// 在线的CPU（online CPUs）：当前激活的CPU数

示例：服务器有20个物理核心，但可以热插拔到40个
- possible_cpus = 40 (最大可能)
- online_cpus = 20 (当前在线)

// 宏定义
#define for_each_possible_cpu(cpu) \
    for ((cpu) = 0; (cpu) < NR_CPUS; (cpu)++) \
        if (cpu_possible(cpu))

// 所以sched_init()会为所有可能的CPU创建运行队列
// 即使这些CPU还没有上线
```

---

## 4. CPU上线时的绑定过程

### 4.1 CPU热插拔流程

```c
// kernel/cpu.c - CPU上线流程

static int _cpu_up(unsigned int cpu)
{
    struct task_struct *idle;

    // 1. 创建该CPU的idle线程（每个CPU一个）
    idle = idle_thread_get(cpu);

    // 2. 激活CPU（硬件相关）
    __cpu_up(cpu, idle);

    // 3. 通知调度器（关键！）
    sched_cpu_starting(cpu);

    // 4. 设置CPU为online状态
    set_cpu_online(cpu, true);

    return 0;
}

// 调度器的CPU启动回调
void sched_cpu_starting(unsigned int cpu)
{
    struct rq *rq = cpu_rq(cpu);

    // 1. 启动运行队列的时钟
    rq->clock = sched_clock_cpu(cpu);

    // 2. 设置运行队列为激活状态
    rq->online = 1;

    // 3. 启动负载均衡
    set_cpu_active(cpu, true);

    // 4. 设置该CPU的idle任务
    rq->idle = idle_thread_get(cpu);
    rq->curr = rq->idle;

    // 5. 允许调度器使用这个CPU
    update_rq_clock(rq);
}

关键理解：
1. 运行队列在sched_init()时就创建了（静态分配）
2. CPU上线时，只是激活这个运行队列（设置online标志）
3. 真正的"绑定"是在CPU上线的那一刻发生的
```

---

## 5. 完整的时间线和内存布局

### 5.1 系统启动时间线

```
T0: 系统启动，BIOS/UEFI加载内核
    ↓
T1: 内核入口点 start_kernel()
    内存：只有引导CPU（BSP，CPU 0）在运行

T2: setup_per_cpu_areas()
    内存：为所有可能的CPU分配Per-CPU区域
    ┌────────────────────────────────────────┐
    │ CPU 0的Per-CPU区域（0xFFFF888000000000） │
    │ CPU 1的Per-CPU区域（0xFFFF888001000000） │
    │ CPU 2的Per-CPU区域（0xFFFF888002000000） │
    │ ...                                    │
    │ CPU 19的Per-CPU区域                     │
    └────────────────────────────────────────┘

T3: sched_init()
    操作：初始化每个CPU的运行队列
    for (i = 0; i < 20; i++) 
    {
        rq = cpu_rq(i);  // 获取CPU i的运行队列
        初始化rq...
    }

    内存：
    CPU 0: runqueues = {curr: &init_task, nr_running: 0, ...}
    CPU 1: runqueues = {curr: NULL, nr_running: 0, ...}  ← 初始化但未激活
    CPU 2: runqueues = {curr: NULL, nr_running: 0, ...}
    ...

T4: rest_init() → kernel_init() → SMP启动
    操作：启动其他CPU（Application Processors）

T5: CPU 1上线
    操作：sched_cpu_starting(1)
    内存：
    CPU 1: runqueues = {
        curr: &idle_task_cpu1,  // 设置为idle任务
        online: 1,              // 标记为在线
        clock: 当前时间,
        ...
    }

T6: CPU 2上线
    ...

T7: 所有CPU上线完成
    内存：
    CPU 0: runqueues = {online: 1, curr: running_task, ...}
    CPU 1: runqueues = {online: 1, curr: running_task, ...}
    ...
    CPU 19: runqueues = {online: 1, curr: running_task, ...}

T8: 系统正常运行
    每个CPU的调度器独立工作，从自己的运行队列中选择任务
```

### 5.2 内存布局详解

```
内核内存空间：
┌──────────────────────────────────────────────────────┐
│ 0xFFFFFFFF00000000 - 0xFFFFFFFFFFFFFFFF (内核空间)    │
│                                                      │
│ Per-CPU区域（每个CPU独立）：                            │
│ ┌──────────────────────────────────────────────────┐ │
│ │ CPU 0的Per-CPU区域（基地址：__per_cpu_offset[0]）   │ │
│ │ ┌────────────────────────────────────────────┐   │ │
│ │ │ struct rq runqueues                        │   │ │
│ │ │ ┌────────────────────────────────────────┐ │   │ │
│ │ │ │ curr: task_struct*                     │ │   │ │
│ │ │ │ cfs: cfs_rq {                          │ │   │ │
│ │ │ │   tasks_timeline: rb_root_cached       │ │   │ │
│ │ │ │   nr_running: 3                        │ │   │ │
│ │ │ │ }                                      │ │   │ │
│ │ │ │ rt: rt_rq {...}                        │ │   │ │
│ │ │ │ nr_running: 3                          │ │   │ │
│ │ │ │ clock: 123456789                       │ │   │ │
│ │ │ │ lock: raw_spinlock_t                   │ │   │ │
│ │ │ └────────────────────────────────────────┘ │   │ │
│ │ │ 其他Per-CPU变量...                          │   │ │
│ │ └────────────────────────────────────────────┘   │ │
│ └──────────────────────────────────────────────────┘ │
│                                                      │
│ ┌──────────────────────────────────────────────────┐ │
│ │ CPU 1的Per-CPU区域（基地址：__per_cpu_offset[1]）   │ │
│ │ ┌────────────────────────────────────────────┐   │ │
│ │ │ struct rq runqueues                        │   │ │
│ │ │ ...（与CPU 0类似，但独立的内存）               │   │ │
│ │ └────────────────────────────────────────────┘   │ │
│ └──────────────────────────────────────────────────┘ │
│                                                      │
│ ... (CPU 2-19的Per-CPU区域)                           │
└──────────────────────────────────────────────────────┘

访问方式：
CPU 0执行：rq = this_rq() → 通过GS:[offset] → 访问CPU 0的runqueues
CPU 1执行：rq = this_rq() → 通过GS:[offset] → 访问CPU 1的runqueues
          (相同的指令，不同的GS值，访问不同的内存！)
```

---

## 6. 为什么要Per-CPU运行队列？

### 6.1 性能优势

```
单一全局运行队列（旧方案）：
┌────────────────────────────────┐
│ 全局运行队列（锁保护）             │
│ [task1, task2, ..., task1000]  │
└────────────────────────────────┘
     ↓      ↓      ↓     ↓
   CPU0   CPU1   CPU2  CPU3 (竞争锁！)

问题：
1. 所有CPU竞争同一个锁（spinlock）
2. 缓存抖动（cache bouncing）
3. 扩展性差（CPU数量增加，性能下降）

Per-CPU运行队列（现代方案）：
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ CPU0队列  │  │ CPU1队列 │  │ CPU2队列  │  │ CPU3队列  │
│ [task1]  │  │ [task2]  │  │ [task3]  │  │ [task4]  │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
     ↓            ↓            ↓            ↓
   CPU0         CPU1         CPU2         CPU3 (无竞争！)

优势：
1. 每个CPU独立的运行队列，无锁竞争
2. 缓存亲和性好（队列在CPU的L1/L2缓存中）
3. 扩展性好（CPU数量增加，性能线性提升）
4. 负载均衡通过work stealing实现
```

### 6.2 缓存亲和性

```
Per-CPU运行队列的内存布局：
__aligned(64) - 对齐到Cache Line

CPU 0访问自己的runqueues：
1. CPU 0读取runqueues → Cache Miss（首次）
2. 加载整个Cache Line到L1 Cache
3. 后续访问runqueues → Cache Hit（极快！）
4. CPU 1访问自己的runqueues → 不影响CPU 0的缓存

如果是全局队列：
1. CPU 0读取global_rq → Cache Miss
2. 加载到L1 Cache
3. CPU 1修改global_rq → 通过MESI协议使CPU 0的Cache Line失效
4. CPU 0再次读取global_rq → Cache Miss（频繁！）
5. 性能下降严重（False Sharing）
```

---

## 7. 查看运行队列信息

### 7.1 查看Per-CPU数据

```bash
# 查看调度器统计信息
$ cat /proc/sched_debug

# 输出示例（节选）：
cpu#0
  .nr_running                    : 2        ← CPU 0的运行队列有2个任务
  .load                          : 2048
  .nr_switches                   : 123456   ← CPU 0已经进行了12万次切换
  .nr_load_updates              : 45678
  .curr->pid                     : 1234     ← 当前运行的任务PID

  cfs_rq[0]:/
    .exec_clock                  : 987654.123456
    .MIN_vruntime                : 0.000001
    .min_vruntime                : 123456.789012
    .max_vruntime                : 123456.789999
    .nr_running                  : 2

  task   PID         tree-key  switches  prio     wait-time
  -----------------------------------------------------------
  bash   1234        123.456       123    120       0.12
  vim    5678        124.789       456    120       0.08

cpu#1
  .nr_running                    : 3        ← CPU 1的运行队列有3个任务
  .nr_switches                   : 98765
  ...

# 解读：
# 每个"cpu#N"块对应一个CPU的运行队列
# 这些数据存储在 cpu_rq(N) 指向的内存中
```

### 7.2 查看Per-CPU变量地址

```bash
# 查看Per-CPU变量的地址
$ sudo cat /proc/kallsyms | grep runqueues
ffff888100000000 D runqueues  ← CPU 0的runqueues地址
ffff888100100000 D runqueues  ← CPU 1的runqueues地址（偏移0x100000）
...

# 每个CPU的Per-CPU区域间隔固定（通常64KB或更大）
```

---

## 8. 面试回答模板

**问题**："每个CPU的运行队列是什么时候创建并绑定到核心上的？

**标准答案**：

"运行队列在Linux内核启动时通过两个关键步骤创建和绑定：

**1. 创建时机（内核启动）**：
在`start_kernel()`函数中，`sched_init()`被调用时创建。这个函数遍历所有可能的CPU（`for_each_possible_cpu`），初始化每个CPU的运行队列。

**2. 存储机制（Per-CPU变量）**：
运行队列通过Per-CPU变量机制静态分配：
```c
DEFINE_PER_CPU_SHARED_ALIGNED(struct rq, runqueues);
```
这意味着每个CPU有独立的内存区域存放运行队列，通过CPU的GS段寄存器访问。

**3. 绑定时机（CPU上线）**：
物理上的绑定发生在CPU上线时：
- 引导CPU（BSP，CPU 0）在`sched_init()`时立即激活
- 其他CPU在SMP初始化时通过`sched_cpu_starting()`激活

**4. 为什么Per-CPU？**
- **无锁竞争**：每个CPU独立操作自己的队列
- **缓存亲和性**：运行队列常驻CPU的L1/L2缓存
- **扩展性好**：CPU数量增加不影响性能

**关键点**：
- 运行队列在内核启动时就全部创建好了（包括未上线的CPU）
- 每个CPU通过Per-CPU变量机制访问自己的独立运行队列
- 这是Linux调度器性能优化的核心设计

**访问方式**：
```c
struct rq *rq = this_rq();  // 当前CPU的运行队列
struct rq *rq = cpu_rq(2);  // CPU 2的运行队列
```


---

## 9. 总结

```
关键时间点：
┌────────────────────────────────────────┐
│ 内核启动（start_kernel）                 │
│   ↓                                    │
│ setup_per_cpu_areas()                  │
│   → 分配Per-CPU内存区域                  │
│   ↓                                    │
│ sched_init()                           │
│   → 初始化所有CPU的运行队列               │
│   → 此时运行队列已经在内存中！             │
│   ↓                                    │
│ rest_init()                            │
│   → SMP初始化                           │
│   ↓                                    │
│ CPU 1上线（sched_cpu_starting(1)）      │
│   → 激活CPU 1的运行队列                  │
│   ↓                                    │
│ CPU 2上线...                            │
│   ↓                                    │
│ 系统正常运行                             │
│   → 每个CPU独立调度，从自己的队列选任务     │
└────────────────────────────────────────┘

内存视图：
每个CPU的Per-CPU区域包含：
- struct rq runqueues（运行队列）
- 其他Per-CPU变量（统计信息等）

访问原理：
- 通过GS段寄存器 + 偏移量
- 相同的代码，不同的GS值，访问不同的内存
- 编译器和CPU协作实现（无需开发者关心）
```

**核心思想是：运行队列在内核启动时静态分配，每个CPU一个，通过Per-CPU变量机制确保每个CPU访问自己独立的队列，避免锁竞争和缓存抖动。**
