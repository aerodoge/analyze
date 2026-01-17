# 操作系统调度机制深度解析

## 1. 调度的本质：task_struct → CPU物理寄存器

### 1.1 核心答案

**调度的本质**：把`task_struct`中保存的寄存器值，**恢复到CPU的物理寄存器**中。

```
调度前：
CPU物理寄存器            task_struct（内存中）
┌─────────────┐         ┌────────────────────┐
│ PC = 0x1000 │  ←─┐    │ thread.ip = 0x5000 │  线程A（等待中）
│ SP = 0x2000 │    │    │ thread.sp = 0x7000 │
│ rax = 123   │    │    └────────────────────┘
└─────────────┘    │    ┌────────────────────┐
   当前执行线程B     └────│ thread.ip = 0x1000 │  线程B（正在运行）
                        │ thread.sp = 0x2000 │
                        │ thread.rax = 123   │
                        └────────────────────┘

调度后（B → A）：
CPU物理寄存器            task_struct（内存中）
┌─────────────┐         ┌────────────────────┐
│ PC = 0x5000 │  ←─────→│ thread.ip = 0x5000 │  线程A（正在运行）
│ SP = 0x7000 │         │ thread.sp = 0x7000 │
│ rax = 456   │         │ thread.rax = 456   │
└─────────────┘         └────────────────────┘
                        ┌────────────────────┐
                        │ thread.ip = 0x1000 │  线程B（已保存）
                        │ thread.sp = 0x2000 │
                        │ thread.rax = 123   │
                        └────────────────────┘
```

**关键理解**：
- **每个物理核心的寄存器数量取决于是否支持超线程**：
  - 无超线程：1个物理核 = 1套物理寄存器
  - 有超线程（如Intel HT）：1个物理核 = 2套架构寄存器 + 共享执行单元
- **task_struct 是内存中的数据结构**，保存线程的"快照"
- **调度就是交换：保存当前寄存器到旧task → 加载新task的寄存器到CPU**

**超线程（Hyper-Threading）技术说明**：
```
无超线程的CPU核心（传统）：
┌────────────────────────────┐
│ 物理核心0                   │
│  ┌──────────────────────┐  │
│  │ 寄存器（1套）          │  │  ← 只有1套物理寄存器
│  │ rax, rbx, rcx, ...   │  │
│  └──────────────────────┘  │
│  ┌──────────────────────┐  │
│  │ 执行单元（ALU/FPU）    │  │
│  └──────────────────────┘  │
└────────────────────────────┘
    ↓
操作系统看到：1个逻辑CPU

有超线程的CPU核心（现代Intel）：
┌──────────────────────────────────────────┐
│ 物理核心0                                 │
│  ┌──────────────────────┐                │
│  │ 寄存器组0（1套）       │  ← 线程0        │
│  │ rax, rbx, rcx, ...   │                │
│  └──────────────────────┘                │
│  ┌──────────────────────┐                │
│  │ 寄存器组1（1套）       │  ← 线程1        │
│  │ rax, rbx, rcx, ...   │                │
│  └──────────────────────┘                │
│  ┌──────────────────────┐                │
│  │ 执行单元（共享）        │ ← 两个线程共享！ │
│  │ ALU/FPU/加载存储单元   │                │
│  └──────────────────────┘                │
└──────────────────────────────────────────┘
    ↓
操作系统看到：2个逻辑CPU（但只有1个物理核）

关键点：
1. 超线程的核心有2套架构寄存器（rax, rbx等）
2. 但执行单元（ALU、FPU）只有1套，两个线程共享
3. 当一个线程等待内存时，另一个线程可以使用执行单元
4. 性能提升：~20-30%（不是2倍，因为共享执行单元）
```

---

## 2. Linux调度器的完整流程

### 2.1 数据结构：运行队列（Run Queue）

每个CPU核心都有自己的运行队列（Per-CPU Run Queue）：

```c
// 每个CPU核心的运行队列（简化版）
struct rq 
{
    // 1. 当前正在运行的task
    struct task_struct *curr;       // 指向正在CPU上执行的task

    // 2. 就绪队列（Ready Queue）- 等待运行的线程
    struct cfs_rq cfs;              // CFS调度器的红黑树
    struct rt_rq  rt;               // 实时调度器的优先级队列

    // 3. 负载信息
    unsigned long nr_running;       // 队列中的线程数
    u64 clock;                      // 调度时钟

    // 4. 锁（保护运行队列）
    raw_spinlock_t lock;
};

// CFS调度器的红黑树（按vruntime排序）
struct cfs_rq 
{
    struct rb_root_cached tasks_timeline;  // 红黑树根节点
    struct rb_node *rb_leftmost;           // 最左节点（vruntime最小）
    unsigned int nr_running;               // 运行的任务数
    u64 min_vruntime;                      // 最小虚拟运行时间
};

// 每个task在红黑树中的节点
struct sched_entity 
{
    struct rb_node run_node;       // 红黑树节点
    u64 vruntime;                  // 虚拟运行时间（越小越优先）
    u64 sum_exec_runtime;          // 累计运行时间
};
```

**关键数据结构关系**：

```
CPU核心
  └── struct rq (运行队列)
       ├── curr → task_struct* (当前运行的线程)
       └── cfs_rq (CFS调度器)
            └── tasks_timeline (红黑树)
                 ├── task_struct A (vruntime=100)
                 ├── task_struct B (vruntime=120)
                 └── task_struct C (vruntime=150)
```

### 2.2 调度器核心函数：schedule()

```c
// 调度器入口函数（kernel/sched/core.c）
asmlinkage __visible void __sched schedule(void)
{
    struct task_struct *prev, *next;
    struct rq *rq;

    // 1. 获取当前CPU的运行队列
    rq = cpu_rq(smp_processor_id());
    prev = rq->curr;  // 当前正在运行的task

    // 2. 关闭抢占和中断（保护临界区）
    preempt_disable();
    local_irq_disable();
    rq_lock(rq);

    // 3. 选择下一个要运行的task
    next = pick_next_task(rq, prev);

    // 4. 如果需要切换（next != prev）
    if (likely(prev != next)) 
    {
        // 4.1 更新统计信息
        rq->nr_switches++;
        rq->curr = next;  // 更新curr指针！

        // 4.2 上下文切换（关键！）
        prev = context_switch(rq, prev, next);
    }

    // 5. 恢复抢占和中断
    rq_unlock(rq);
    local_irq_enable();
    preempt_enable();
}
```

### 2.3 选择下一个任务：pick_next_task()

```c
// 选择下一个要运行的task（CFS调度器）
static struct task_struct * pick_next_task_fair(struct rq *rq, struct task_struct *prev)
{
    struct cfs_rq *cfs_rq = &rq->cfs;
    struct sched_entity *se;

    // 从红黑树中选择vruntime最小的节点（最左节点）
    se = __pick_first_entity(cfs_rq);
    if (!se)
        return NULL;

    // 从sched_entity获取对应的task_struct
    struct task_struct *next = task_of(se);

    return next;
}

// 获取红黑树最左节点（vruntime最小 = 最优先）
static struct sched_entity *__pick_first_entity(struct cfs_rq *cfs_rq)
{
    struct rb_node *left = rb_first_cached(&cfs_rq->tasks_timeline);
    if (!left)
        return NULL;

    return rb_entry(left, struct sched_entity, run_node);
}
```

### 2.4 上下文切换：context_switch()

**这是调度的核心！真正把task"放到CPU上执行"：**

```c
// 上下文切换（kernel/sched/core.c）
static __always_inline struct task_struct * context_switch(struct rq *rq, struct task_struct *prev, struct task_struct *next)
{
    // 1. 切换内存上下文（页表）- 仅进程切换时需要
    if (!next->mm) // 内核线程
    {  
        next->active_mm = prev->active_mm;
    } 
    else 
    {          // 用户线程
        switch_mm(prev->mm, next->mm, next);  // 切换页表
    }

    // 2. 切换寄存器上下文（关键！）
    switch_to(prev, next, prev);

    return prev;
}
```

### 2.5 寄存器切换：switch_to()（汇编实现）

**这是真正的"魔法时刻"——把task的寄存器恢复到CPU！**

```asm
// arch/x86/entry/entry_64.S（x86-64架构）
ENTRY(__switch_to_asm)
    /*
     * 保存prev的寄存器到 prev->thread
     */
    movq    %rbx,   TASK_threadsp(%rdi)      /* 保存rbx */
    movq    %rbp,   TASK_threadbp(%rdi)      /* 保存rbp */
    movq    %r12,   TASK_threadr12(%rdi)     /* 保存r12 */
    movq    %r13,   TASK_threadr13(%rdi)     /* 保存r13 */
    movq    %r14,   TASK_threadr14(%rdi)     /* 保存r14 */
    movq    %r15,   TASK_threadr15(%rdi)     /* 保存r15 */
    movq    (%rsp), %r11                     /* 返回地址 */
    movq    %r11,   TASK_threadip(%rdi)      /* 保存PC */
    movq    %rsp,   TASK_threadsp(%rdi)      /* 保存SP */

    /*
     * 恢复next的寄存器从 next->thread
     */
    movq    TASK_threadsp(%rsi), %rsp        /* 恢复SP */
    movq    TASK_threadbp(%rsi), %rbp        /* 恢复rbp */
    movq    TASK_threadbx(%rsi), %rbx        /* 恢复rbx */
    movq    TASK_threadr12(%rsi), %r12       /* 恢复r12 */
    movq    TASK_threadr13(%rsi), %r13       /* 恢复r13 */
    movq    TASK_threadr14(%rsi), %r14       /* 恢复r14 */
    movq    TASK_threadr15(%rsi), %r15       /* 恢复r15 */

    /* 跳转到next线程的指令地址 */
    pushq   TASK_threadip(%rsi)              /* 压入next的PC */
    ret                                       /* 跳转执行！*/
END(__switch_to_asm)
```

**关键理解**：
- `%rdi` 指向 `prev->thread`（旧task）
- `%rsi` 指向 `next->thread`（新task）
- `movq %rbx, TASK_threadsp(%rdi)` = 把CPU的rbx寄存器保存到 `prev->thread.rbx`
- `movq TASK_threadbp(%rsi), %rbp` = 把 `next->thread.rbp` 加载到CPU的rbp寄存器
- 最后的 `ret` 指令跳转到 `next->thread.ip`（程序计数器），开始执行新线程！

---

## 3. 完整的调度流程演示

### 3.1 场景：线程A和线程B的调度

```
初始状态：CPU正在执行线程B

┌─────────────────────────────────────────────────────────┐
│ CPU核心0                                                 │
│ ┌─────────────┐                                         │
│ │ PC = 0x1234 │  ← 正在执行线程B的代码                     │
│ │ SP = 0x8000 │                                         │
│ │ rax = 100   │                                         │
│ └─────────────┘                                         │
└─────────────────────────────────────────────────────────┘

内存中的数据结构：
┌──────────────────────────────────────┐
│ rq (CPU0的运行队列)                    │
│ ┌──────────────────────────────────┐ │
│ │ curr ──────────────┐             │ │
│ │                    ▼             │ │
│ │ cfs_rq:                          │ │
│ │   红黑树:                         │ │
│ │   ┌─────────────────────────┐    │ │
│ │   │ task A (vruntime=100)   │    │ │ ← 最左节点（即将被选中）
│ │   └─────────────────────────┘    │ │
│ │   ┌─────────────────────────┐    │ │
│ │   │ task C (vruntime=200)   │    │ │
│ │   └─────────────────────────┘    │ │
│ └──────────────────────────────────┘ │
└──────────────────────────────────────┘
                │
                ▼
┌───────────────────────────────────────┐
│ task_struct (线程B - 正在运行)          │
│ ┌───────────────────────────────────┐ │
│ │ thread.ip = 0x1234  (PC)          │ │
│ │ thread.sp = 0x8000  (SP)          │ │
│ │ thread.rax = 100                  │ │
│ │ ...                               │ │
│ └───────────────────────────────────┘ │
│ se.vruntime = 250                     │
└───────────────────────────────────────┘

┌───────────────────────────────────────┐
│ task_struct (线程A - 等待运行)          │
│ ┌───────────────────────────────────┐ │
│ │ thread.ip = 0x5000  (保存的PC)     │ │
│ │ thread.sp = 0x9000  (保存的SP)     │ │
│ │ thread.rax = 200                  │ │
│ │ ...                               │ │
│ └───────────────────────────────────┘ │
│ se.vruntime = 100  ← 最小！应该运行     │
└───────────────────────────────────────┘
```

### 3.2 调度过程（分步详解）

```
步骤1：触发调度
─────────────────
线程B运行时间片到期 → 触发时钟中断 → 中断处理程序调用 schedule()


步骤2：schedule() 执行
─────────────────────
rq = cpu_rq(0);              // 获取CPU0的运行队列
prev = rq->curr;             // prev = 线程B


步骤3：pick_next_task() 选择下一个任务
──────────────────────────────────────
从红黑树选择vruntime最小的节点
→ 选中线程A（vruntime=100）
next = task A


步骤4：更新运行队列
─────────────────
rq->curr = next;             // rq->curr从线程B改为线程A！

┌──────────────────────────────────────┐
│ rq (CPU0的运行队列)                    │
│ curr ──────────────┐                 │
│                    ▼                 │
│              ┌─────────────────┐     │
│              │ task A          │     │  ← 现在指向线程A
│              └─────────────────┘     │
└──────────────────────────────────────┘


步骤5：context_switch(rq, prev=B, next=A)
────────────────────────────────────────
5.1 switch_mm() - 切换页表（如果是不同进程）
5.2 switch_to() - 切换寄存器（汇编代码）


步骤6：switch_to() 的汇编魔法
─────────────────────────────
# 保存线程B的寄存器到 B->thread
movq %rbx, [B->thread.rbx]     # CPU.rbx(100) → B->thread.rbx
movq %rsp, [B->thread.sp]      # CPU.SP(0x8000) → B->thread.sp
movq %r11, [B->thread.ip]      # 返回地址 → B->thread.ip

# 恢复线程A的寄存器从 A->thread
movq [A->thread.sp], %rsp      # A->thread.sp(0x9000) → CPU.SP
movq [A->thread.rbx], %rbx     # A->thread.rax(200) → CPU.rbx
...

# 跳转到线程A的代码地址
push [A->thread.ip]            # 压入0x5000
ret                            # 跳转到0x5000执行！


步骤7：线程A开始执行
───────────────────
CPU物理寄存器
┌─────────────┐
│ PC = 0x5000 │  ← 从A->thread.ip恢复
│ SP = 0x9000 │  ← 从A->thread.sp恢复
│ rax = 200   │  ← 从A->thread.rax恢复
└─────────────┘
现在CPU正在执行线程A的代码！


步骤8：线程B的状态
─────────────────
task_struct (线程B)
┌─────────────────────┐
│ thread.ip = 0x1238  │  ← 保存了被打断时的位置
│ thread.sp = 0x8000  │
│ thread.rbx = 100    │
└─────────────────────┘
se.vruntime = 250      ← 插入红黑树，等待下次调度
```

---

## 4. 关键概念总结

### 4.1 调度就是"指针切换 + 寄存器拷贝"

```
伪代码表示：

void schedule() 
{
    task_struct *prev = rq->curr;        // 当前运行的task
    task_struct *next = pick_next();     // 选择下一个task

    // 核心操作1：更新指针
    rq->curr = next;                     // ← 调度完成的标志！

    // 核心操作2：切换寄存器
    save_registers(&prev->thread);       // CPU寄存器 → prev->thread
    load_registers(&next->thread);       // next->thread → CPU寄存器

    // 核心操作3：跳转执行
    jump_to(next->thread.ip);            // 跳转到next的代码地址
}
```

### 4.2 "放到CPU上执行"的三重含义

| 操作        | 含义              | 实现                             |
|-----------|-----------------|--------------------------------|
| **逻辑上**   | 把task从队列中选出     | `next = pick_next_task(rq)`    |
| **数据结构上** | 更新运行队列的curr指针   | `rq->curr = next`              |
| **物理上**   | 把task的寄存器加载到CPU | `switch_to(prev, next)` - 汇编代码 |

### 4.3 为什么需要运行队列？

```
问题：为什么不直接把task_struct加载到CPU？

答案：因为有很多task在等待运行！

运行队列的作用：
1. 组织所有就绪的task（红黑树/优先级队列）
2. 快速选择下一个应该运行的task（O(1) 或 O(logN)）
3. 每个CPU独立的队列，减少锁竞争

┌─────────────────────────────────┐
│ CPU0运行队列                     │
│  curr → task B (正在运行)        │
│  队列: [task A, task C, task D]  │
└─────────────────────────────────┘
       ↓ 时间片到期
       ↓ 调度器从队列选择task A
       ▼
┌─────────────────────────────────┐
│ CPU0运行队列                     │
│  curr → task A (正在运行)        │
│  队列: [task B, task C, task D]  │
└─────────────────────────────────┘
```

---

## 5. 实验：观察调度过程

### 5.1 使用ftrace跟踪调度

```bash
# 启用调度事件跟踪
sudo su
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
echo 1 > /sys/kernel/debug/tracing/tracing_on

# 运行一个程序
sleep 1 &

# 查看调度事件
cat /sys/kernel/debug/tracing/trace

# 输出示例：
# prev_comm=sleep prev_pid=1234 prev_state=S ==> next_comm=bash next_pid=5678
# 解释：从sleep(PID=1234)切换到bash(PID=5678)，sleep进入睡眠状态(S)
```

### 5.2 使用perf查看上下文切换

```bash
# 跟踪context_switch函数的调用
sudo perf probe --add='context_switch prev->pid next->pid'
sudo perf record -e probe:context_switch -a sleep 5
sudo perf script

# 输出：
# context_switch: prev_pid=1234 next_pid=5678
# context_switch: prev_pid=5678 next_pid=9012
```

### 5.3 查看运行队列信息

```bash
# 查看每个CPU的运行队列长度
cat /proc/sched_debug

# 输出示例（节选）：
# cpu#0
#   .nr_running                    : 3        ← CPU0上有3个就绪任务
#   .load                          : 3072
#   .nr_switches                   : 123456   ← 已进行12万次切换
#
#   task   PID         tree-key  switches  prio     wait-time
#   -----------------------------------------------------------
#   bash   1234        1000.123       45    120         0.12
#   vim    5678        1200.456       89    120         0.08
#   gcc    9012        1150.789       34    120         0.05
```

### 5.4 C代码：模拟调度器核心逻辑

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// 简化的task结构
typedef struct task_struct 
{
    int pid;
    char name[16];
    unsigned long vruntime;  // 虚拟运行时间

    // 模拟寄存器（简化）
    struct 
    {
        unsigned long ip;  // 程序计数器
        unsigned long sp;  // 栈指针
        int rax;           // 通用寄存器
    } thread;

    struct task_struct *next;  // 链表
} task_t;

// 简化的运行队列
typedef struct 
{
    task_t *curr;       // 当前运行的task
    task_t *head;       // 就绪队列头
    int nr_running;     // 队列长度
} runqueue_t;

// 模拟CPU寄存器
typedef struct 
{
    unsigned long pc;
    unsigned long sp;
    int rax;
} cpu_regs_t;

cpu_regs_t cpu;  // 全局CPU寄存器

// 选择vruntime最小的task
task_t* pick_next_task(runqueue_t *rq) 
{
    task_t *min = rq->head;
    task_t *p = rq->head;

    while (p) 
    {
        if (p->vruntime < min->vruntime)
            min = p;
        p = p->next;
    }

    return min;
}

// 上下文切换
void context_switch(runqueue_t *rq, task_t *prev, task_t *next) 
{
    printf("\n[Context Switch] %s(PID=%d) → %s(PID=%d)\n", prev->name, prev->pid, next->name, next->pid);

    // 1. 保存prev的寄存器
    prev->thread.ip = cpu.pc;
    prev->thread.sp = cpu.sp;
    prev->thread.rax = cpu.rax;
    printf("  保存 %s: PC=0x%lx SP=0x%lx rax=%d\n", prev->name, prev->thread.ip, prev->thread.sp, prev->thread.rax);

    // 2. 恢复next的寄存器
    cpu.pc = next->thread.ip;
    cpu.sp = next->thread.sp;
    cpu.rax = next->thread.rax;
    printf("  恢复 %s: PC=0x%lx SP=0x%lx rax=%d\n", next->name, next->thread.ip, next->thread.sp, next->thread.rax);

    // 3. 更新curr指针
    rq->curr = next;
    printf("  rq->curr 现在指向 %s\n", next->name);
}

// 调度函数
void schedule(runqueue_t *rq) 
{
    task_t *prev = rq->curr;
    task_t *next = pick_next_task(rq);

    if (prev != next) 
    {
        context_switch(rq, prev, next);
    }
}

int main() 
{
    // 初始化运行队列
    runqueue_t rq = {0};

    // 创建3个任务
    task_t tasks[3] = 
    {
        {.pid = 100, .name = "bash", .vruntime = 150,
         .thread = {.ip = 0x1000, .sp = 0x8000, .rax = 10}},
        {.pid = 200, .name = "vim", .vruntime = 100,   // vruntime最小
         .thread = {.ip = 0x2000, .sp = 0x9000, .rax = 20}},
        {.pid = 300, .name = "gcc", .vruntime = 200,
         .thread = {.ip = 0x3000, .sp = 0xA000, .rax = 30}}
    };

    // 构建链表
    tasks[0].next = &tasks[1];
    tasks[1].next = &tasks[2];
    tasks[2].next = NULL;
    rq.head = &tasks[0];
    rq.nr_running = 3;

    // 初始：bash正在运行
    rq.curr = &tasks[0];
    cpu.pc = tasks[0].thread.ip;
    cpu.sp = tasks[0].thread.sp;
    cpu.rax = tasks[0].thread.rax;

    printf("初始状态：%s 正在运行\n", rq.curr->name);
    printf("CPU寄存器：PC=0x%lx SP=0x%lx rax=%d\n\n", cpu.pc, cpu.sp, cpu.rax);

    // 模拟调度
    printf("========== 第1次调度 ==========\n");
    schedule(&rq);

    // 运行一段时间后再次调度
    printf("\n执行 %s 一段时间...\n", rq.curr->name);
    rq.curr->vruntime += 50;  // 增加运行时间
    cpu.pc += 0x10;           // 程序计数器前进
    cpu.rax = 999;            // 寄存器被修改

    printf("\n========== 第2次调度 ==========\n");
    schedule(&rq);

    return 0;
}
```

**编译运行**：
```bash
gcc -o scheduler scheduler.c
./scheduler
```

**输出**：
```
初始状态：bash 正在运行
CPU寄存器：PC=0x1000 SP=0x8000 rax=10

========== 第1次调度 ==========

[Context Switch] bash(PID=100) → vim(PID=200)
  保存 bash: PC=0x1000 SP=0x8000 rax=10
  恢复 vim: PC=0x2000 SP=0x9000 rax=20
  rq->curr 现在指向 vim

执行 vim 一段时间...

========== 第2次调度 ==========

[Context Switch] vim(PID=200) → bash(PID=100)
  保存 vim: PC=0x2010 SP=0x9000 rax=999     ← 保存了修改后的值
  恢复 bash: PC=0x1000 SP=0x8000 rax=10     ← 恢复之前保存的值
  rq->curr 现在指向 bash
```

---

## 6. 高级话题：调度器的演进

### 6.1 Linux调度器历史

| 调度器       | 时间           | 特点         | 复杂度     |
|-----------|--------------|------------|---------|
| **O(n)**  | 2.4内核        | 遍历所有task选择 | O(n)    |
| **O(1)**  | 2.6.0-2.6.22 | 140个优先级队列  | O(1)    |
| **CFS**   | 2.6.23+      | 红黑树，公平调度   | O(logN) |
| **EEVDF** | 6.6+         | 更好的延迟保证    | O(logN) |

### 6.2 CFS调度器的核心思想

```
传统调度器：基于优先级（不公平）
高优先级 → 更多CPU时间
低优先级 → 饥饿

CFS：所有进程公平分享CPU
┌─────────────────────────────────┐
│ 理想的多任务处理器：               │
│ n个任务同时运行，每个获得1/n CPU    │
└─────────────────────────────────┘
        ↓ 现实实现
┌─────────────────────────────────┐
│ 追踪每个任务的vruntime            │
│ 总是选择vruntime最小的任务         │
│ → 落后的任务会被优先调度           │
└─────────────────────────────────┘

示例：
任务A: vruntime=100ms → 运行10ms → vruntime=110ms
任务B: vruntime=120ms → 现在是最小 → 被调度
任务B: vruntime=120ms → 运行10ms → vruntime=130ms
任务A: vruntime=110ms → 又是最小 → 被调度
```

---

## 7. 面试回答模板

**问题**："线程调度时，OS把task对象放到什么地方？"

**标准答案**：

"线程调度的本质是**把task_struct中保存的寄存器状态，恢复到CPU的物理寄存器**。

具体流程：
1. **数据结构层面**：每个CPU核心有一个运行队列（struct rq），包含一个curr指针指向当前运行的task，以及一个就绪队列（CFS用红黑树实现）存放等待运行的task。

2. **调度选择**：schedule()函数从红黑树中选择vruntime最小的task（表示最应该获得CPU的任务），更新rq->curr指针指向这个新task。

3. **物理切换**：context_switch()函数通过汇编代码执行：
   - 保存旧task的寄存器到prev->thread结构
   - 加载新task的寄存器从next->thread结构到CPU
   - 跳转到新task的PC（程序计数器）开始执行

所以'放到CPU上'的含义是：把task的thread.ip加载到CPU的PC寄存器，thread.sp加载到SP寄存器等，然后CPU就开始执行这个task的代码了。

这也解释了为什么上下文切换有成本：每次切换需要保存/恢复十几个寄存器（通用寄存器+浮点寄存器），加上缓存污染、TLB刷新等间接成本，总代价约10-100微秒。"

---

## 8. 延伸问题

### Q1: 为什么不直接让所有task共享CPU寄存器？
**A**: 因为CPU只有一套物理寄存器，多个task无法同时使用。必须通过时间分片（time slicing）轮流使用。

### Q2: 运行队列在内存哪里？
**A**: 在内核内存中，每个CPU核心有独立的运行队列（per-CPU变量），避免跨CPU访问锁竞争。

### Q3: 红黑树的查找不是O(logN)吗？为什么还说CFS高效？
**A**: CFS通过缓存最左节点（rb_leftmost）实现O(1)获取下一个task。插入/删除是O(logN)，但选择是O(1)。

### Q4: 为什么高频交易要用实时调度(SCHED_FIFO)？
**A**: SCHED_FIFO保证高优先级任务不会被抢占（除非自己主动让出或更高优先级任务），避免CFS的"公平"导致关键任务被打断。

---

## 9. 推荐资源

1. **Linux内核源码**：
   - `kernel/sched/core.c` - schedule()函数
   - `kernel/sched/fair.c` - CFS调度器
   - `arch/x86/entry/entry_64.S` - switch_to汇编

2. **书籍**：
   - 《深入理解Linux内核》第7章
   - 《Linux内核设计与实现》第4章

3. **工具**：
   ```bash
   # 查看调度统计
   cat /proc/sched_debug

   # 跟踪调度事件
   perf sched record
   perf sched latency

   # 查看任务调度类
   chrt -p <PID>
   ```
