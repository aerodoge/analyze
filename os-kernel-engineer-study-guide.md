# 操作系统内核工程师学习指南

> 基于微内核软件开发岗位要求整理

---

## 目录

- [1. 进程与线程](#1-进程与线程)
- [2. 内存管理](#2-内存管理)
- [3. 编译与链接](#3-编译与链接)
- [4. 操作系统架构](#4-操作系统架构)
- [5. I/O系统](#5-io系统)
- [6. 数据结构与算法](#6-数据结构与算法)
- [7. 微内核设计](#7-微内核设计)
- [8. 实战项目](#8-实战项目)

---

## 1. 进程与线程

### 1.1 进程与线程的概念和区别

**进程（Process）**：
- **定义**：进程是资源分配的基本单位，是程序的一次执行实例
- **特点**：
  - 拥有独立的地址空间（代码段、数据段、堆、栈）
  - 拥有独立的文件描述符表、信号处理器等资源
  - 进程间相互隔离，互不影响

**线程（Thread）**：
- **定义**：线程是CPU调度的基本单位，是进程内的执行流
- **特点**：
  - 共享进程的地址空间和资源
  - 拥有独立的栈、寄存器、程序计数器（PC）
  - 线程间可以直接访问共享内存

**对比表格**：

| 维度       | 进程                   | 线程               |
|----------|----------------------|------------------|
| **资源分配** | 是资源分配的基本单位           | 不拥有资源，共享进程资源     |
| **调度**   | 可被调度                 | 是调度的基本单位         |
| **地址空间** | 独立的地址空间              | 共享进程地址空间         |
| **通信**   | 需要IPC（管道、消息队列、共享内存等） | 可以直接读写共享变量       |
| **创建开销** | 大（需要分配地址空间、复制页表等）    | 小（只需分配栈和TCB）     |
| **切换开销** | 大（需要切换页表、刷新TLB等）     | 小（只需切换寄存器）       |
| **安全性**  | 隔离性好，一个进程崩溃不影响其他进程   | 一个线程崩溃可能导致整个进程崩溃 |

**Linux中的实现**：
```c
// Linux使用 task_struct 统一表示进程和线程
struct task_struct {
    // 进程标识
    pid_t pid;                    // 进程ID
    pid_t tgid;                   // 线程组ID（主线程的PID）

    // 调度信息
    int prio;                     // 优先级
    unsigned int policy;          // 调度策略
    struct sched_entity se;       // 调度实体

    // 内存信息
    struct mm_struct *mm;         // 内存描述符（进程共享，线程为NULL或指向同一个）

    // 文件系统信息
    struct fs_struct *fs;         // 文件系统信息
    struct files_struct *files;   // 打开的文件描述符

    // 栈信息
    void *stack;                  // 内核栈

    // 父子关系
    struct task_struct *parent;   // 父进程
    struct list_head children;    // 子进程链表
    struct list_head sibling;     // 兄弟进程链表
};
```

**创建进程 vs 创建线程**：

```c
// 创建进程 - fork()
pid_t fork(void) {
    // 1. 分配新的 task_struct
    // 2. 复制父进程的 mm_struct（写时复制，Copy-on-Write）
    // 3. 复制文件描述符表
    // 4. 分配新的PID
    // 5. 子进程加入就绪队列
    return child_pid;
}

// 创建线程 - clone() 带 CLONE_VM 等标志
pid_t clone(int (*fn)(void *), void *stack, int flags, void *arg) {
    // flags = CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND
    // 1. 分配新的 task_struct
    // 2. 共享父进程的 mm_struct（mm指向同一个）
    // 3. 共享文件描述符表
    // 4. 分配新的TID（线程ID）
    // 5. 线程加入就绪队列
    return thread_id;
}
```

### 1.2 进程/线程管理

**进程状态转换**：

```
┌─────────────────────────────────────────────────────────────────┐
│                      进程状态转换图                               │
└─────────────────────────────────────────────────────────────────┘

              fork()
          ┌──────────────┐
          │     新建      │ (New)
          │   (创建中)    │
          └──────┬───────┘
                 │ 创建完成
                 ▼
          ┌──────────────┐
    ┌────▶│     就绪      │ (Ready)
    │     │  (等待CPU)    │
    │     └──────┬───────┘
    │            │ 调度器选中
    │   时间片用完 │
    │   或被抢占  ▼
    │     ┌──────────────┐
    └─────│     运行      │ (Running)
          │  (占用CPU)    │
          └──────┬───────┘
                 │
        ┌────────┼────────┐
        │                 │
    等待I/O          exit()
    或事件                 │
        │                 ▼
        ▼        ┌──────────────┐
  ┌──────────┐   │     终止      │ (Terminated)
  │   阻塞    │   │   (已结束)    │
  │ (Blocked)│   └──────────────┘
  └────┬─────┘
       │ I/O完成
       │ 或事件发生
       └──────────┐
                  │
                  ▼
           (回到就绪队列)
```

**进程控制块（PCB / task_struct）关键字段**：

```c
struct task_struct {
    // === 进程状态 ===
    volatile long state;        // 进程状态: TASK_RUNNING, TASK_INTERRUPTIBLE等

    // === 进程标识 ===
    pid_t pid;                  // 进程ID
    pid_t tgid;                 // 线程组ID

    // === 调度信息 ===
    int prio;                   // 动态优先级
    int static_prio;            // 静态优先级
    int normal_prio;            // 归一化优先级
    unsigned int policy;        // 调度策略: SCHED_NORMAL, SCHED_FIFO, SCHED_RR

    struct sched_entity se;     // CFS调度实体
    struct sched_rt_entity rt;  // 实时调度实体

    // === 内存管理 ===
    struct mm_struct *mm;       // 进程地址空间
    struct mm_struct *active_mm;// 活动地址空间（内核线程用）

    // === 文件系统 ===
    struct fs_struct *fs;       // 文件系统信息（根目录、当前目录）
    struct files_struct *files; // 打开的文件描述符

    // === 信号处理 ===
    struct signal_struct *signal;  // 信号处理器
    sigset_t blocked;              // 阻塞的信号集
    struct sigpending pending;     // 待处理信号

    // === CPU上下文 ===
    struct thread_struct thread;   // CPU寄存器状态
    void *stack;                   // 内核栈指针

    // === 进程关系 ===
    struct task_struct *parent;    // 父进程
    struct list_head children;     // 子进程链表
    struct list_head sibling;      // 兄弟进程链表

    // === 时间统计 ===
    u64 utime;                     // 用户态时间
    u64 stime;                     // 内核态时间
    unsigned long nvcsw;           // 自愿上下文切换次数
    unsigned long nivcsw;          // 非自愿上下文切换次数
};
```

### 1.3 进程/线程间通信（IPC）

**1. 管道（Pipe）**

**原理**：
- 半双工通信（单向数据流）
- 数据以FIFO方式读写
- 内核中实现为环形缓冲区（默认64KB）

**实现**：
```c
// 管道创建
int pipe(int pipefd[2]);  // pipefd[0]读端, pipefd[1]写端

// 示例
int pipefd[2];
pipe(pipefd);

if (fork() == 0) {
    // 子进程：写数据
    close(pipefd[0]);  // 关闭读端
    write(pipefd[1], "Hello", 5);
    close(pipefd[1]);
} else {
    // 父进程：读数据
    char buf[100];
    close(pipefd[1]);  // 关闭写端
    read(pipefd[0], buf, 100);
    close(pipefd[0]);
}
```

**内核实现**：
```c
// 管道结构（简化）
struct pipe_inode_info {
    struct pipe_buffer *bufs;  // 环形缓冲区数组
    unsigned int head;         // 写位置
    unsigned int tail;         // 读位置
    unsigned int max_usage;    // 最大缓冲区数

    wait_queue_head_t wait;    // 等待队列（读者和写者）
    struct mutex mutex;        // 互斥锁
};
```

**2. 消息队列（Message Queue）**

**原理**：
- 消息以链表形式存储在内核
- 支持消息类型，接收时可指定类型
- 消息有最大长度限制

**实现**：
```c
// System V 消息队列
#include <sys/msg.h>

// 创建/获取消息队列
int msgid = msgget(key, IPC_CREAT | 0666);

// 发送消息
struct msgbuf {
    long mtype;      // 消息类型
    char mtext[100]; // 消息内容
};

struct msgbuf msg;
msg.mtype = 1;
strcpy(msg.mtext, "Hello");
msgsnd(msgid, &msg, strlen(msg.mtext), 0);

// 接收消息
msgrcv(msgid, &msg, sizeof(msg.mtext), 1, 0);
```

**3. 共享内存（Shared Memory）**

**原理**：
- 多个进程映射同一块物理内存
- 最快的IPC方式（无需数据拷贝）
- 需要额外的同步机制（信号量、互斥锁）

**实现**：
```c
// System V 共享内存
#include <sys/shm.h>

// 创建共享内存
int shmid = shmget(key, size, IPC_CREAT | 0666);

// 映射到进程地址空间
void *addr = shmat(shmid, NULL, 0);

// 使用共享内存
strcpy((char *)addr, "Shared Data");

// 解除映射
shmdt(addr);

// 删除共享内存
shmctl(shmid, IPC_RMID, NULL);
```

**内核实现**：
```c
// 共享内存段结构
struct shmid_kernel {
    struct kern_ipc_perm shm_perm;  // 权限
    size_t shm_segsz;               // 段大小
    unsigned long shm_nattch;       // 当前附加的进程数
    struct file *shm_file;          // 对应的文件（tmpfs）
    // ...
};

// 映射到进程地址空间
// 实际是将 shm_file 映射到进程的VMA（虚拟内存区域）
```

**4. 信号量（Semaphore）**

**原理**：
- 用于进程/线程同步
- 本质是一个计数器 + 等待队列
- PV操作（P: wait, V: signal）

**实现**：
```c
// POSIX 信号量
#include <semaphore.h>

sem_t sem;
sem_init(&sem, 0, 1);  // 初始值为1（二值信号量，类似互斥锁）

// P操作（等待）
sem_wait(&sem);
// 临界区
sem_post(&sem);  // V操作（通知）

sem_destroy(&sem);
```

**内核实现**：
```c
struct semaphore {
    raw_spinlock_t lock;       // 自旋锁保护
    unsigned int count;        // 计数器
    struct list_head wait_list;// 等待队列
};

// P操作（down）
void down(struct semaphore *sem) {
    spin_lock(&sem->lock);
    if (sem->count > 0) {
        sem->count--;
        spin_unlock(&sem->lock);
    } else {
        // 加入等待队列，睡眠
        add_wait_queue(&sem->wait_list, &wait);
        spin_unlock(&sem->lock);
        schedule();  // 让出CPU
    }
}

// V操作（up）
void up(struct semaphore *sem) {
    spin_lock(&sem->lock);
    if (list_empty(&sem->wait_list)) {
        sem->count++;
    } else {
        // 唤醒等待队列中的一个进程
        wake_up(&sem->wait_list);
    }
    spin_unlock(&sem->lock);
}
```

**5. 信号（Signal）**

**原理**：
- 异步通知机制
- 用于通知进程发生了某个事件
- 常见信号：SIGINT（Ctrl+C）、SIGKILL（强制杀死）、SIGSEGV（段错误）

**实现**：
```c
#include <signal.h>

// 注册信号处理函数
void handler(int sig) {
    printf("Received signal %d\n", sig);
}

signal(SIGINT, handler);

// 发送信号
kill(pid, SIGINT);

// 阻塞信号
sigset_t set;
sigemptyset(&set);
sigaddset(&set, SIGINT);
sigprocmask(SIG_BLOCK, &set, NULL);
```

**IPC方式对比**：

| IPC方式  | 速度 | 数据拷贝         | 同步     | 使用场景      |
|--------|----|--------------|--------|-----------|
| 管道     | 中等 | 2次（用户→内核→用户） | 无      | 简单的父子进程通信 |
| 消息队列   | 中等 | 2次           | 无      | 需要消息类型区分  |
| 共享内存   | 最快 | 0次           | 需要额外同步 | 大数据量、高频通信 |
| 信号量    | -  | -            | 是      | 进程/线程同步   |
| 信号     | 快  | 无数据          | 异步     | 异步事件通知    |
| Socket | 慢  | 2次           | 无      | 网络通信、本地通信 |

### 1.4 进程/线程同步

**1. 互斥锁（Mutex）**

**原理**：
- 保护临界区，同一时刻只有一个线程可以持有锁
- 获取锁失败时，线程会睡眠（阻塞）

**实现**：
```c
#include <pthread.h>

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;

// 加锁
pthread_mutex_lock(&mutex);
// 临界区
pthread_mutex_unlock(&mutex);
```

**内核实现（Linux futex）**：
```c
// 快速用户态互斥锁（Fast Userspace Mutex）
struct futex_q {
    struct plist_node list;    // 优先级链表节点
    struct task_struct *task;  // 等待的任务
    // ...
};

// futex系统调用
long sys_futex(u32 *uaddr, int op, u32 val, ...) {
    switch (op) {
    case FUTEX_WAIT:
        // 如果 *uaddr == val，则睡眠等待
        if (atomic_read(uaddr) == val) {
            add_to_futex_wait_queue(uaddr);
            schedule();  // 让出CPU
        }
        break;
    case FUTEX_WAKE:
        // 唤醒等待在 uaddr 上的线程
        wake_up_futex_waiters(uaddr, val);
        break;
    }
}
```

**2. 自旋锁（Spinlock）**

**原理**：
- 获取锁失败时，线程会忙等待（自旋）而不是睡眠
- 适用于临界区很小、持有时间短的场景
- 避免了上下文切换的开销

**实现**：
```c
// 用户态自旋锁（原子操作）
typedef struct {
    volatile int lock;
} spinlock_t;

void spin_lock(spinlock_t *lock) {
    while (__sync_lock_test_and_set(&lock->lock, 1)) {
        // 自旋等待
        while (lock->lock) {
            __asm__ __volatile__("pause");  // x86 PAUSE指令，降低功耗
        }
    }
}

void spin_unlock(spinlock_t *lock) {
    __sync_lock_release(&lock->lock);
}
```

**内核自旋锁**：
```c
// Linux内核自旋锁
typedef struct {
    arch_spinlock_t raw_lock;
} raw_spinlock_t;

// x86实现（使用LOCK前缀的原子操作）
static inline void arch_spin_lock(arch_spinlock_t *lock) {
    asm volatile(
        "1: lock; decb %0\n"      // 原子递减
        "   jns 3f\n"              // 如果>=0，跳到3（获取成功）
        "2: pause\n"               // PAUSE指令
        "   cmpb $0, %0\n"         // 检查锁是否可用
        "   jle 2b\n"              // 如果<=0，继续自旋
        "   jmp 1b\n"              // 重新尝试获取
        "3:\n"
        : "+m" (lock->slock)
        :: "memory"
    );
}
```

**Mutex vs Spinlock**：

| 特性    | Mutex       | Spinlock     |
|-------|-------------|--------------|
| 等待方式  | 睡眠（阻塞）      | 自旋（忙等待）      |
| 上下文切换 | 有           | 无            |
| CPU占用 | 不占用         | 占用（100%）     |
| 适用场景  | 临界区较大、持有时间长 | 临界区很小、持有时间极短 |
| 使用环境  | 用户态、内核态均可   | 主要用于内核态      |
| 中断上下文 | 不可用         | 可用           |

**3. 读写锁（RWLock）**

**原理**：
- 允许多个读者同时读，但写者独占
- 读者优先 vs 写者优先

**实现**：
```c
#include <pthread.h>

pthread_rwlock_t rwlock = PTHREAD_RWLOCK_INITIALIZER;

// 读锁（共享锁）
pthread_rwlock_rdlock(&rwlock);
// 读操作
pthread_rwlock_unlock(&rwlock);

// 写锁（独占锁）
pthread_rwlock_wrlock(&rwlock);
// 写操作
pthread_rwlock_unlock(&rwlock);
```

**4. 条件变量（Condition Variable）**

**原理**：
- 用于线程间的条件等待和通知
- 必须与互斥锁配合使用

**实现**：
```c
#include <pthread.h>

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
int ready = 0;

// 等待线程
pthread_mutex_lock(&mutex);
while (!ready) {
    pthread_cond_wait(&cond, &mutex);  // 原子：释放锁 + 睡眠
}
pthread_mutex_unlock(&mutex);

// 通知线程
pthread_mutex_lock(&mutex);
ready = 1;
pthread_cond_signal(&cond);  // 唤醒一个等待线程
pthread_mutex_unlock(&mutex);
```

**经典同步问题**：

**生产者-消费者问题**：
```c
#define BUFFER_SIZE 10

int buffer[BUFFER_SIZE];
int count = 0;  // 缓冲区中的数据数量
int in = 0;     // 生产者放入位置
int out = 0;    // 消费者取出位置

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
pthread_cond_t not_full = PTHREAD_COND_INITIALIZER;
pthread_cond_t not_empty = PTHREAD_COND_INITIALIZER;

// 生产者
void *producer(void *arg) 
{
    int item;
    while (1) 
    {
        item = produce_item();

        pthread_mutex_lock(&mutex);
        while (count == BUFFER_SIZE) 
        {
            pthread_cond_wait(&not_full, &mutex);  // 缓冲区满，等待
        }

        buffer[in] = item;
        in = (in + 1) % BUFFER_SIZE;
        count++;

        pthread_cond_signal(&not_empty);  // 通知消费者
        pthread_mutex_unlock(&mutex);
    }
}

// 消费者
void *consumer(void *arg) 
{
    int item;
    while (1) 
    {
        pthread_mutex_lock(&mutex);
        while (count == 0) 
        {
            pthread_cond_wait(&not_empty, &mutex);  // 缓冲区空，等待
        }

        item = buffer[out];
        out = (out + 1) % BUFFER_SIZE;
        count--;

        pthread_cond_signal(&not_full);  // 通知生产者
        pthread_mutex_unlock(&mutex);

        consume_item(item);
    }
}
```

### 1.5 进程/线程调度

**调度算法分类**：

**1. 批处理系统调度算法**：
- **先来先服务（FCFS）**：按到达顺序调度，简单但平均等待时间长
- **最短作业优先（SJF）**：选择执行时间最短的作业，最优但难以预测
- **最短剩余时间优先（SRTN）**：SJF的抢占版本

**2. 交互式系统调度算法**：
- **时间片轮转（Round Robin）**：每个进程轮流执行固定时间片
- **优先级调度**：高优先级进程先执行
- **多级反馈队列**：多个优先级队列，动态调整优先级

**3. 实时系统调度算法**：
- **速率单调调度（RMS）**：周期越短优先级越高
- **最早截止期限优先（EDF）**：截止期限越早优先级越高

**Linux CFS调度器（完全公平调度器）**：

**原理**：
- 基于虚拟运行时间（vruntime）调度
- 保证每个进程获得公平的CPU时间
- 使用红黑树组织可运行进程

**核心思想**：
```
虚拟运行时间 = 实际运行时间 × (NICE_0_LOAD / 进程权重)

- 权重由进程的nice值决定（-20 ~ 19）
- nice值越小，权重越大，vruntime增长越慢
- 调度器总是选择vruntime最小的进程运行
```

**数据结构**：
```c
// 调度实体（每个进程/线程）
struct sched_entity 
{
    u64 vruntime;              // 虚拟运行时间
    u64 exec_start;            // 开始执行时间
    u64 sum_exec_runtime;      // 累计运行时间
    struct load_weight load;   // 权重
    struct rb_node run_node;   // 红黑树节点
};

// 运行队列（每个CPU）
struct cfs_rq 
{
    struct rb_root_cached tasks_timeline;  // 红黑树根（按vruntime排序）
    struct sched_entity *curr;             // 当前运行的实体
    u64 min_vruntime;                      // 最小虚拟时间（单调递增）
};
```

**调度过程**：
```c
// 1. 选择下一个进程（pick_next_task）
struct task_struct *pick_next_task_fair(struct rq *rq) 
{
    struct cfs_rq *cfs_rq = &rq->cfs;
    struct sched_entity *se;

    // 从红黑树中选择最左边的节点（vruntime最小）
    se = __pick_first_entity(cfs_rq);

    return task_of(se);
}

// 2. 时钟中断更新vruntime
void entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr) 
{
    // 更新当前进程的vruntime
    update_curr(cfs_rq);

    // 检查是否需要抢占
    if (cfs_rq->nr_running > 1) 
    {
        check_preempt_tick(cfs_rq, curr);
    }
}

// 3. 更新vruntime
static void update_curr(struct cfs_rq *cfs_rq) 
{
    struct sched_entity *curr = cfs_rq->curr;
    u64 now = rq_clock_task(rq_of(cfs_rq));
    u64 delta_exec;

    delta_exec = now - curr->exec_start;
    curr->exec_start = now;

    curr->sum_exec_runtime += delta_exec;

    // 计算vruntime增量
    curr->vruntime += calc_delta_fair(delta_exec, curr);

    // 更新最小vruntime
    update_min_vruntime(cfs_rq);
}

// 4. 计算vruntime增量
static u64 calc_delta_fair(u64 delta, struct sched_entity *se) 
{
    // delta_vruntime = delta_exec * (NICE_0_LOAD / weight)
    if (se->load.weight != NICE_0_LOAD)
        delta = __calc_delta(delta, NICE_0_LOAD, &se->load);

    return delta;
}
```

**实时调度（SCHED_FIFO / SCHED_RR）**：

**SCHED_FIFO（先进先出）**：
- 没有时间片，运行直到主动让出或被更高优先级抢占
- 优先级范围：1-99

**SCHED_RR（时间片轮转）**：
- 类似FIFO，但有时间片限制
- 时间片用完后，加入同优先级队列尾部

**实现**：
```c
struct rt_rq {
    struct rt_prio_array active;  // 按优先级组织的队列数组
    unsigned int rt_nr_running;   // 实时进程数量
    u64 rt_time;                  // 实时进程运行时间
};

struct rt_prio_array {
    DECLARE_BITMAP(bitmap, MAX_RT_PRIO+1);  // 优先级位图
    struct list_head queue[MAX_RT_PRIO];    // 每个优先级一个队列
};

// 选择下一个实时进程
static struct task_struct *pick_next_task_rt(struct rq *rq) {
    struct rt_prio_array *array = &rq->rt.active;
    int idx;

    // 找到最高优先级非空队列
    idx = sched_find_first_bit(array->bitmap);

    // 从队列头部取出进程
    struct sched_rt_entity *rt_se = list_first_entry(&array->queue[idx], ...);

    return task_of(rt_se);
}
```

### 1.6 上下文切换

**上下文切换过程**：

```
┌────────────────────────────────────────────────────────────────┐
│                      上下文切换流程                              │
└────────────────────────────────────────────────────────────────┘

进程A运行中
    │
    ├─ 1. 时钟中断 / 系统调用
    │
    ▼
保存进程A的上下文
    ├─ 保存CPU寄存器（PC、SP、通用寄存器）
    ├─ 保存FPU状态（浮点寄存器）
    └─ 保存其他硬件状态
    │
    ▼
调度器选择下一个进程B
    ├─ 从运行队列选择
    └─ 更新调度信息
    │
    ▼
切换地址空间（如果需要）
    ├─ 切换页表（CR3寄存器）
    └─ 刷新TLB
    │
    ▼
恢复进程B的上下文
    ├─ 恢复CPU寄存器
    ├─ 恢复FPU状态
    └─ 恢复其他硬件状态
    │
    ▼
进程B开始运行
```

**代码实现（简化）**：

```c
// 上下文切换函数（switch_to）
#define switch_to(prev, next, last) do {                    \
    /* 保存prev的寄存器状态 */                                \
    asm volatile(                                            \
        "pushq %%rbp\n\t"           /* 保存栈帧指针 */        \
        "pushq %%rbx\n\t"           /* 保存被调用者保存寄存器 */\
        "pushq %%r12\n\t"                                    \
        "pushq %%r13\n\t"                                    \
        "pushq %%r14\n\t"                                    \
        "pushq %%r15\n\t"                                    \
        "movq %%rsp, %0\n\t"        /* 保存栈指针 */          \
        /* 恢复next的寄存器状态 */                            \
        "movq %1, %%rsp\n\t"        /* 恢复栈指针 */          \
        "popq %%r15\n\t"                                     \
        "popq %%r14\n\t"                                     \
        "popq %%r13\n\t"                                     \
        "popq %%r12\n\t"                                     \
        "popq %%rbx\n\t"                                     \
        "popq %%rbp\n\t"                                     \
        : "=m" (prev->thread.sp)    /* 输出 */               \
        : "m" (next->thread.sp)     /* 输入 */               \
        : "memory"                                           \
    );                                                       \
} while (0)

// 完整的上下文切换
struct task_struct *__switch_to(struct task_struct *prev, struct task_struct *next) {
    // 1. 保存FPU状态
    __fpu_switch_to(prev, next);

    // 2. 切换内核栈
    load_sp0(next);

    // 3. 切换线程局部存储（TLS）
    load_TLS(next);

    // 4. 切换调试寄存器
    if (unlikely(task_thread_info(prev)->flags & _TIF_WORK_CTXSW))
        __switch_to_extra(prev, next);

    // 5. 切换硬件上下文（寄存器）
    switch_to(prev, next, prev);

    return prev;
}
```

**进程切换 vs 线程切换**：

| 操作        | 进程切换      | 线程切换       |
|-----------|-----------|------------|
| 保存/恢复寄存器  | ✓         | ✓          |
| 切换内核栈     | ✓         | ✓          |
| 切换页表（CR3） | ✓         | ✗（共享地址空间）  |
| 刷新TLB     | ✓         | ✗          |
| 切换文件描述符表  | ✓         | ✗（共享）      |
| 开销        | 大（1-10微秒） | 小（0.1-1微秒） |

**上下文切换开销分析**：

```c
// 测量上下文切换时间（使用lmbench）
// 方法：使用管道在两个进程间互相唤醒

int main() 
{
    int pipe1[2], pipe2[2];
    char buf;

    pipe(pipe1);
    pipe(pipe2);

    if (fork() == 0) 
    {
        // 子进程
        while (1) 
        {
            read(pipe1[0], &buf, 1);   // 等待父进程
            write(pipe2[1], "x", 1);   // 唤醒父进程
        }
    } 
    else 
    {
        // 父进程
        struct timespec start, end;
        int iterations = 100000;

        clock_gettime(CLOCK_MONOTONIC, &start);

        for (int i = 0; i < iterations; i++) 
        {
            write(pipe1[1], "x", 1);   // 唤醒子进程
            read(pipe2[0], &buf, 1);   // 等待子进程
        }

        clock_gettime(CLOCK_MONOTONIC, &end);
        double elapsed = (end.tv_sec - start.tv_sec) + (end.tv_nsec - start.tv_nsec) / 1e9;
        printf("Context switch time: %.2f μs\n", elapsed / iterations / 2 * 1e6);
    }
}

// 典型结果：
// - 进程切换：2-5μs
// - 线程切换：0.5-1μs
// - 函数调用：<0.01μs
```

---

## 2. 内存管理

### 2.1 内存分配算法

**用户态内存分配器（glibc malloc）**

**核心数据结构**：

```c
// chunk结构（分配的内存块）
struct malloc_chunk 
{
    size_t prev_size;   // 前一个chunk的大小（如果前一个chunk空闲）
    size_t size;        // 当前chunk的大小（含元数据），最低3位是标志位

    // 当chunk空闲时，以下字段有效
    struct malloc_chunk *fd;  // forward指针（指向下一个空闲chunk）
    struct malloc_chunk *bk;  // backward指针（指向上一个空闲chunk）
};

// size字段的标志位
#define PREV_INUSE  0x1  // 前一个chunk正在使用
#define IS_MMAPPED  0x2  // 通过mmap分配
#define NON_MAIN_ARENA 0x4  // 不属于主分配区
```

**分配策略（多级分箱）**：

```
┌─────────────────────────────────────────────────────────────┐
│                    malloc分配策略                            │
└─────────────────────────────────────────────────────────────┘

请求大小                    分配策略
   │
   ├─ < 64B ──────────▶ fast bins（10个，LIFO）
   │                    8B, 16B, 24B, ..., 80B
   │
   ├─ 64B ~ 512B ─────▶ small bins（62个，FIFO）
   │                    每个bin大小差8B
   │
   ├─ 512B ~ 128KB ───▶ large bins（63个）
   │                    大小范围递增
   │                    每个bin内按大小排序
   │
   ├─ > 128KB ────────▶ mmap直接分配
   │                    独立映射区域
   │
   └─ 特殊情况 ────────▶ unsorted bin（临时存放）
                        回收的chunk先放这里
                        下次分配时整理到对应bin
```

**malloc实现流程**：

```c
void *malloc(size_t size) 
{
    size_t nb;  // 实际分配大小（对齐后）

    // 1. 大小对齐到chunk_size（8/16字节对齐）
    nb = request2size(size);

    // 2. Fast bins快速路径（< 64B）
    if (nb <= get_max_fast()) 
    {
        mchunkptr victim = fastbin[fastbin_index(nb)];
        if (victim != NULL) 
        {
            fastbin[fastbin_index(nb)] = victim->fd;
            return chunk2mem(victim);
        }
    }

    // 3. Small bins（64B ~ 512B）
    if (in_smallbin_range(nb)) 
    {
        int idx = smallbin_index(nb);
        mchunkptr victim = bin_at(av, idx)->bk;  // 取链表尾部

        if (victim != bin_at(av, idx)) 
        {
            unlink(victim);
            return chunk2mem(victim);
        }
    }

    // 4. Large bins（512B ~ 128KB）
    if (!in_smallbin_range(nb)) {
        int idx = largebin_index(nb);

        // 从对应large bin中找最小的满足要求的chunk
        for (victim = bin_at(av, idx)->bk; victim != bin_at(av, idx); victim = victim->bk) 
        {

            if (chunksize(victim) >= nb) 
            {
                unlink(victim);

                // 如果chunk过大，分割
                if (chunksize(victim) > nb + MIN_CHUNK_SIZE) 
                {
                    split_chunk(victim, nb);
                }

                return chunk2mem(victim);
            }
        }
    }

    // 5. 从top chunk分配
    victim = av->top;
    if (chunksize(victim) > nb + MIN_CHUNK_SIZE) 
    {
        av->top = chunk_at_offset(victim, nb);
        set_head(av->top, (chunksize(victim) - nb) | PREV_INUSE);
        return chunk2mem(victim);
    }

    // 6. 通过sbrk/mmap扩展堆
    if (nb >= mmap_threshold) 
    {
        return mmap_alloc(nb);
    } 
    else 
    {
        return sbrk_alloc(nb);
    }
}
```

**free实现流程**：

```c
void free(void *mem) 
{
    mchunkptr p = mem2chunk(mem);
    size_t size = chunksize(p);

    // 1. Fast bins快速回收（< 64B）
    if (size <= get_max_fast()) 
    {
        int idx = fastbin_index(size);
        p->fd = fastbin[idx];
        fastbin[idx] = p;
        return;
    }

    // 2. 合并相邻空闲chunk
    mchunkptr next = chunk_at_offset(p, size);

    // 向前合并
    if (!prev_inuse(p)) 
    {
        mchunkptr prev = prev_chunk(p);
        size += chunksize(prev);
        unlink(prev);
        p = prev;
    }

    // 向后合并
    if (!inuse(next)) 
    {
        size += chunksize(next);
        unlink(next);
    }

    set_head(p, size | PREV_INUSE);
    set_foot(p, size);

    // 3. 如果chunk太大，返回给系统
    if (size > mmap_threshold && p == chunk_at_offset(av->top, -size)) 
    {
        // 通过sbrk收缩堆
        sbrk(-size);
        return;
    }

    // 4. 放入unsorted bin
    p->fd = unsorted_bin->fd;
    p->bk = unsorted_bin;
    unsorted_bin->fd->bk = p;
    unsorted_bin->fd = p;
}
```

**其他内存分配算法**：

**1. Buddy System（伙伴系统）**

**原理**：
- 内存按2的幂次划分（1KB, 2KB, 4KB, 8KB, ...）
- 分配时找最小的满足要求的块
- 释放时与相邻伙伴合并

**示例**：
```
初始状态：1个64KB块

分配7KB:
64KB split → 32KB + 32KB
32KB split → 16KB + 16KB
16KB split → 8KB + 8KB
分配8KB块

分配3KB:
8KB split → 4KB + 4KB
分配4KB块

释放7KB:
8KB块释放，与相邻8KB合并 → 16KB
16KB与相邻16KB合并 → 32KB
```

**Linux内核中的实现**：
```c
// Linux使用伙伴系统管理物理页
struct free_area 
{
    struct list_head free_list[MIGRATE_TYPES];
    unsigned long nr_free;
};

struct zone 
{
    struct free_area free_area[MAX_ORDER];  // MAX_ORDER=11，管理1,2,4,8,...,1024页
};

// 分配2^order个连续页
struct page *__alloc_pages(unsigned int order) 
{
    // 从order开始找，如果没有就找更大的块并分裂
    for (int current_order = order; current_order < MAX_ORDER; current_order++) 
    {
        if (!list_empty(&zone->free_area[current_order].free_list)) 
        {
            page = list_first_entry(&zone->free_area[current_order].free_list);

            // 分裂大块
            while (current_order > order) {
                current_order--;
                split_page(page, current_order);
            }

            return page;
        }
    }

    return NULL;
}
```

**2. Slab Allocator（板分配器）**

**原理**：
- 专门用于分配内核对象（如task_struct、inode等）
- 预先分配一组对象，缓存起来
- 避免频繁分配/释放小对象的开销

**实现**：
```c
// slab缓存
struct kmem_cache 
{
    const char *name;           // 对象名称
    size_t object_size;         // 对象大小
    size_t align;               // 对齐要求

    struct kmem_cache_cpu __percpu *cpu_slab;  // 每CPU的slab
    struct kmem_cache_node *node[MAX_NUMNODES];  // 每节点的slab
};

// slab（一组对象）
struct slab 
{
    struct list_head list;      // 链表节点
    void *freelist;             // 空闲对象链表
    unsigned int inuse;         // 已使用对象数
    unsigned int objects;       // 总对象数
};

// 创建slab缓存
struct kmem_cache *kmem_cache_create(const char *name, size_t size, size_t align) 
{
    struct kmem_cache *s = kzalloc(sizeof(*s), GFP_KERNEL);

    s->name = name;
    s->object_size = size;
    s->align = align;

    // 初始化每CPU和每节点的slab
    init_kmem_cache_cpus(s);
    init_kmem_cache_nodes(s);

    return s;
}

// 分配对象
void *kmem_cache_alloc(struct kmem_cache *s, gfp_t flags) 
{
    void *object;
    struct kmem_cache_cpu *c = this_cpu_ptr(s->cpu_slab);

    // 快速路径：从CPU本地slab分配
    object = c->freelist;
    if (likely(object)) 
    {
        c->freelist = get_freepointer(s, object);
        return object;
    }

    // 慢速路径：从slab节点分配或创建新slab
    return __slab_alloc(s, flags);
}

// 释放对象
void kmem_cache_free(struct kmem_cache *s, void *x) 
{
    struct kmem_cache_cpu *c = this_cpu_ptr(s->cpu_slab);

    // 快速路径：放回CPU本地slab
    set_freepointer(s, x, c->freelist);
    c->freelist = x;
}
```

**3. tcmalloc（Thread-Caching Malloc）**

**原理**：
- Google开发，优化多线程性能
- 每个线程有本地缓存，无需加锁
- 大对象由中央堆管理

**结构**：
```
┌──────────────────────────────────────────────────────────┐
│                   tcmalloc架构                            │
└──────────────────────────────────────────────────────────┘

线程1               线程2              线程3
  │                  │                  │
  ▼                  ▼                  ▼
┌─────────┐      ┌─────────┐      ┌─────────┐
│Thread   │      │Thread   │      │Thread   │
│Cache 1  │      │Cache 2  │      │Cache 3  │
│(无锁)    │      │(无锁)   │      │(无锁)    │
└────┬────┘      └────┬────┘      └────┬────┘
     │                │                │
     │   缓存不足/过多  │                │
     └────────┬───────┴────────────────┘
              │
              ▼
       ┌──────────────┐
       │Central Cache │  (有锁，但锁竞争少)
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │  Page Heap   │  (管理大块内存)
       └──────────────┘
```

### 2.2 虚拟内存与分页机制

**虚拟内存的概念**：

**为什么需要虚拟内存？**
1. **内存隔离**：每个进程有独立的地址空间，互不干扰
2. **内存保护**：进程无法访问其他进程的内存
3. **简化编程**：程序可以使用连续的虚拟地址空间
4. **支持超额分配**：虚拟内存总量可以超过物理内存

**地址转换**：

```
虚拟地址 ──[MMU]──▶ 物理地址
   │                   │
   │                   │
   ▼                   ▼
 进程看到的地址      实际物理RAM地址
```

**分页机制**：

**页表结构（x86-64，4级页表）**：

```
┌────────────────────────────────────────────────────────────┐
│              虚拟地址转换过程（4级页表）                       │
└────────────────────────────────────────────────────────────┘

虚拟地址（48位）:
┌──────┬─────┬─────┬─────┬─────┬──────────┐
│未使用 │ PML4│ PDP │ PD  │ PT  │  Offset  │
│16位  │ 9位  │ 9位 │ 9位 │ 9位  │  12位    │
└──────┴──┬──┴──┬──┴──┬──┴──┬──┴────┬─────┘
          │     │     │     │       │
    CR3寄存器    │     │     │       │
       │  │     │     │     │       │
       ▼  │     │     │     │       │
    ┌─────────┐ │     │     │       │
    │PML4 Table││     │     │       │
    │512 entry│ │     │     │       │
    └────┬────┘ │     │     │       │
         │◀─────┘     │     │       │
         ▼            │     │       │
    ┌─────────┐       │     │       │
    │PDP Table│       │     │       │
    │512 entry│       │     │       │
    └────┬────┘       │     │       │
         │◀───────────┘     │       │
         ▼                  │       │
    ┌─────────┐             │       │
    │PD Table │             │       │
    │512 entry│             │       │
    └────┬────┘             │       │
         │◀─────────────────┘       │
         ▼                          │
    ┌─────────┐                     │
    │PT Table │                     │
    │512 entry│                     │
    └────┬────┘                     │
         │                          │
         ▼                          │
    ┌────────────┐                  │
    │物理页框地址│                    │
    │(4KB)       │                  │
    └─────┬──────┘                  │
          │◀────────────────────────┘
          ▼
       物理地址
```

**页表项（PTE）结构**：

```c
// x86-64 页表项（64位）
typedef struct 
{
    unsigned long present    : 1;  // 存在位（1=在内存，0=在磁盘）
    unsigned long rw         : 1;  // 读写位（1=可写，0=只读）
    unsigned long user       : 1;  // 用户位（1=用户态可访问，0=仅内核）
    unsigned long pwt        : 1;  // 页级写穿
    unsigned long pcd        : 1;  // 页级缓存禁用
    unsigned long accessed   : 1;  // 访问位（硬件设置）
    unsigned long dirty      : 1;  // 脏位（页被写过）
    unsigned long pat        : 1;  // 页属性表
    unsigned long global     : 1;  // 全局页（TLB不刷新）
    unsigned long ignored    : 3;  // 忽略
    unsigned long pfn        : 40; // 物理页框号（Page Frame Number）
    unsigned long ignored2   : 11; // 忽略
    unsigned long nx         : 1;  // 不可执行位
} pte_t;
```

**TLB（Translation Lookaside Buffer）**：

**作用**：缓存虚拟地址到物理地址的映射，加速地址转换

**工作流程**：
```c
物理地址 = translate_address(虚拟地址) 
{
    // 1. 查TLB缓存
    if (tlb_entry = tlb_lookup(virtual_addr)) 
    {
        return tlb_entry.physical_addr;  // TLB命中，直接返回
    }

    // 2. TLB未命中，查页表（页表遍历，Page Walk）
    pml4_entry = cr3[PML4_INDEX(virtual_addr)];
    pdp_entry  = pml4_entry.base[PDP_INDEX(virtual_addr)];
    pd_entry   = pdp_entry.base[PD_INDEX(virtual_addr)];
    pt_entry   = pd_entry.base[PT_INDEX(virtual_addr)];

    if (!pt_entry.present) 
    {
        // 页不在内存，触发缺页异常
        page_fault();
    }

    physical_addr = pt_entry.pfn << 12 | OFFSET(virtual_addr);

    // 3. 更新TLB
    tlb_insert(virtual_addr, physical_addr);

    return physical_addr;
}
```

**TLB刷新**：
```c
// 刷新整个TLB
void flush_tlb_all(void) 
{
    // 重新加载CR3寄存器
    write_cr3(read_cr3());
}

// 刷新单个页的TLB
void flush_tlb_single(unsigned long addr) 
{
    asm volatile("invlpg (%0)" :: "r" (addr) : "memory");
}
```

**缺页异常处理**：

```c
// 缺页异常处理流程
void do_page_fault(struct pt_regs *regs, unsigned long error_code) 
{
    unsigned long address = read_cr2();  // CR2存储导致缺页的地址
    struct mm_struct *mm = current->mm;
    struct vm_area_struct *vma;

    // 1. 查找VMA（虚拟内存区域）
    vma = find_vma(mm, address);
    if (!vma || vma->vm_start > address) 
    {
        // 非法地址，发送SIGSEGV信号
        send_signal(SIGSEGV);
        return;
    }

    // 2. 检查权限
    if (error_code & PF_WRITE) 
    {
        if (!(vma->vm_flags & VM_WRITE)) 
        {
            send_signal(SIGSEGV);
            return;
        }
    }

    // 3. 处理不同类型的缺页
    if (error_code & PF_PROT) 
    {
        // 写时复制（Copy-on-Write）
        handle_cow_fault(vma, address);
    } 
    else 
    {
        // 需求分页（Demand Paging）
        handle_pte_fault(vma, address);
    }
}

// 需求分页处理
void handle_pte_fault(struct vm_area_struct *vma, unsigned long address) 
{
    pte_t *pte = get_pte(vma->vm_mm, address);

    if (pte_none(*pte)) 
    {
        // 页表项为空，首次访问
        if (vma->vm_ops && vma->vm_ops->fault) 
        {
            // 文件映射，从文件读取
            vma->vm_ops->fault(vma, address);
        } 
        else 
        {
            // 匿名映射，分配新页
            struct page *page = alloc_page(GFP_KERNEL);
            pte_t entry = mk_pte(page, vma->vm_page_prot);
            set_pte_at(vma->vm_mm, address, pte, entry);
        }
    } 
    else if (!pte_present(*pte)) 
    {
        // 页在磁盘（swap），换入内存
        do_swap_page(vma, address, pte);
    }
}

// 写时复制处理
void handle_cow_fault(struct vm_area_struct *vma, unsigned long address) 
{
    pte_t *pte = get_pte(vma->vm_mm, address);
    struct page *old_page = pte_page(*pte);

    // 1. 检查页的引用计数
    if (page_count(old_page) == 1) 
    {
        // 只有一个进程引用，直接标记为可写
        pte_t entry = pte_mkwrite(pte_mkdirty(*pte));
        set_pte_at(vma->vm_mm, address, pte, entry);
    } 
    else 
    {
        // 多个进程共享，复制页
        struct page *new_page = alloc_page(GFP_KERNEL);
        copy_page(new_page, old_page);

        pte_t entry = mk_pte(new_page, vma->vm_page_prot);
        entry = pte_mkwrite(pte_mkdirty(entry));
        set_pte_at(vma->vm_mm, address, pte, entry);

        page_put(old_page);  // 减少旧页引用计数
    }
}
```

### 2.3 内存映射

**mmap系统调用**：

**功能**：
- 将文件或设备映射到进程地址空间
- 可以用于共享内存、匿名内存分配

**原型**：
```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);

// addr: 建议映射地址（NULL表示由内核选择）
// length: 映射长度
// prot: 内存保护标志（PROT_READ | PROT_WRITE | PROT_EXEC）
// flags: 映射类型（MAP_SHARED | MAP_PRIVATE | MAP_ANONYMOUS）
// fd: 文件描述符（匿名映射时为-1）
// offset: 文件偏移
```

**示例**：

```c
// 1. 文件映射（用于读取大文件）
int fd = open("large_file.dat", O_RDONLY);
void *addr = mmap(NULL, file_size, PROT_READ, MAP_PRIVATE, fd, 0);

// 直接访问文件内容（按需加载，Demand Paging）
char data = ((char *)addr)[1000000];

munmap(addr, file_size);
close(fd);

// 2. 匿名内存映射（替代malloc）
void *addr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                  MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

// 3. 共享内存
void *addr = mmap(NULL, size, PROT_READ | PROT_WRITE,
                  MAP_SHARED | MAP_ANONYMOUS, -1, 0);

if (fork() == 0) 
{
    // 子进程可以访问共享内存
    ((int *)addr)[0] = 123;
} 
else 
{
    wait(NULL);
    printf("%d\n", ((int *)addr)[0]);  // 输出123
}
```

**内核实现**：

```c
// VMA（虚拟内存区域）
struct vm_area_struct 
{
    unsigned long vm_start;     // 起始地址
    unsigned long vm_end;       // 结束地址
    struct mm_struct *vm_mm;    // 所属地址空间
    pgprot_t vm_page_prot;      // 页保护标志
    unsigned long vm_flags;     // 区域标志（VM_READ, VM_WRITE, VM_EXEC）

    struct file *vm_file;       // 映射的文件（如果有）
    unsigned long vm_pgoff;     // 文件偏移

    const struct vm_operations_struct *vm_ops;  // 操作函数集

    struct vm_area_struct *vm_next, *vm_prev;  // 链表节点
    struct rb_node vm_rb;       // 红黑树节点（用于快速查找）
};

// mmap系统调用实现
unsigned long do_mmap(struct file *file, unsigned long addr,
                      unsigned long len, unsigned long prot,
                      unsigned long flags, unsigned long pgoff) 
{
    struct mm_struct *mm = current->mm;
    struct vm_area_struct *vma;

    // 1. 参数检查
    if (len == 0 || len > TASK_SIZE)
        return -EINVAL;

    // 2. 查找可用的地址空间
    if (addr == 0) 
    {
        addr = get_unmapped_area(file, addr, len, pgoff, flags);
    }

    // 3. 创建VMA
    vma = kmem_cache_alloc(vm_area_cachep, GFP_KERNEL);
    vma->vm_start = addr;
    vma->vm_end = addr + len;
    vma->vm_flags = calc_vm_flags(prot, flags);
    vma->vm_page_prot = vm_get_page_prot(vma->vm_flags);
    vma->vm_file = file;
    vma->vm_pgoff = pgoff;

    // 4. 插入VMA到进程地址空间
    insert_vm_struct(mm, vma);

    // 5. 如果是文件映射，设置操作函数
    if (file && file->f_op->mmap) 
    {
        file->f_op->mmap(file, vma);
    }

    return addr;
}
```

### 2.4 分段机制

**分段vs分页**：

| 特性   | 分段      | 分页            |
|------|---------|---------------|
| 单位   | 段（大小可变） | 页（大小固定，通常4KB） |
| 地址   | 段号:段内偏移 | 虚拟页号:页内偏移     |
| 碎片   | 外部碎片    | 内部碎片          |
| 保护   | 按段保护    | 按页保护          |
| 共享   | 容易      | 较难            |
| 程序视角 | 符合逻辑结构  | 对程序透明         |

**x86分段机制**：

```c
// 段描述符（8字节）
struct segment_descriptor 
{
    unsigned short limit_low;      // 段界限（低16位）
    unsigned short base_low;       // 基地址（低16位）
    unsigned char base_mid;        // 基地址（中8位）
    unsigned char type       : 4;  // 段类型
    unsigned char s          : 1;  // 0=系统段 1=代码/数据段
    unsigned char dpl        : 2;  // 特权级（0-3）
    unsigned char present    : 1;  // 存在位
    unsigned char limit_high : 4;  // 段界限（高4位）
    unsigned char avl        : 1;  // 可用位
    unsigned char l          : 1;  // 64位模式标志
    unsigned char db         : 1;  // 32/16位模式
    unsigned char g          : 1;  // 粒度（0=字节，1=4KB）
    unsigned char base_high;       // 基地址（高8位）
};

// 段选择子（16位）
struct segment_selector 
{
    unsigned short rpl : 2;  // 请求特权级
    unsigned short ti  : 1;  // 表指示器（0=GDT, 1=LDT）
    unsigned short index : 13;  // 段描述符索引
};
```

**地址转换**：

```
逻辑地址（段选择子:偏移）
    │
    ├─ 1. 段选择子 → 段描述符表（GDT/LDT）→ 段描述符
    │
    ├─ 2. 段基址 + 偏移 → 线性地址
    │
    └─ 3. 线性地址 → [分页机制] → 物理地址
```

**Linux中的分段使用**：

**问题**：Linux需要同时支持分段和分页，但分段在x86-64中已基本弃用

**解决方案**：Linux使用"平坦内存模型"
- 所有段的基址设为0，界限设为最大（4GB/256TB）
- 实际上绕过了分段机制，直接使用分页

```c
// Linux GDT设置（简化）
struct desc_struct gdt[] = {
    [GDT_ENTRY_KERNEL_CS] = {  // 内核代码段
        .base = 0,
        .limit = 0xFFFFF,  // 4GB
        .type = 0xA,       // 可执行、可读
        .dpl = 0,          // 特权级0
        .g = 1,            // 4KB粒度
    },
    [GDT_ENTRY_KERNEL_DS] = {  // 内核数据段
        .base = 0,
        .limit = 0xFFFFF,
        .type = 0x2,       // 可读写
        .dpl = 0,
        .g = 1,
    },
    [GDT_ENTRY_USER_CS] = {  // 用户代码段
        .base = 0,
        .limit = 0xFFFFF,
        .type = 0xA,
        .dpl = 3,          // 特权级3
        .g = 1,
    },
    [GDT_ENTRY_USER_DS] = {  // 用户数据段
        .base = 0,
        .limit = 0xFFFFF,
        .type = 0x2,
        .dpl = 3,
        .g = 1,
    },
};
```

### 2.5 I/O系统架构

**I/O层次结构**：

```
┌─────────────────────────────────────────────────────────────┐
│                    I/O软件层次                                │
└─────────────────────────────────────────────────────────────┘

用户程序
   │  系统调用（read/write）
   ▼
┌──────────────┐
│  设备无关层    │  VFS（虚拟文件系统）
└──────┬───────┘
       │  通用块层
       ▼
┌──────────────┐
│  设备驱动层    │  块设备驱动、字符设备驱动、网络设备驱动
└──────┬───────┘
       │  I/O请求队列、中断处理
       ▼
┌──────────────┐
│    硬件层     │  DMA控制器、I/O端口、中断控制器
└──────────────┘
```

**I/O方式**：

**1. 程序控制I/O（Programmed I/O）**
```c
// CPU轮询设备状态
while (!(inb(device_status_port) & READY_BIT)) {
    // 忙等待
}
outb(device_data_port, data);  // 发送数据
```
**缺点**：CPU忙等待，效率低

**2. 中断驱动I/O**
```c
// 启动I/O操作
outb(device_command_port, START_COMMAND);

// 中断处理函数
void device_interrupt_handler(void) {
    char data = inb(device_data_port);
    buffer[index++] = data;

    if (index < buffer_size) {
        outb(device_command_port, START_COMMAND);  // 继续读取
    } else {
        wake_up_process(waiting_process);  // 唤醒等待的进程
    }
}
```
**优点**：CPU不需要忙等待
**缺点**：每个字节都要中断一次，中断开销大

**3. DMA（Direct Memory Access）**
```c
// 设置DMA
dma_setup(device_id, buffer_addr, buffer_size, DMA_READ);

// DMA控制器直接在内存和设备间传输数据
// CPU可以做其他事情

// 传输完成后触发中断
void dma_interrupt_handler(void) {
    // 数据已经在buffer中
    wake_up_process(waiting_process);
}
```
**优点**：CPU完全不参与数据传输，效率最高

**Linux I/O调度器**：

**作用**：优化磁盘I/O请求的顺序，减少磁盘寻道时间

**1. CFQ（Completely Fair Queuing）**
- 为每个进程维护一个请求队列
- 轮流服务每个队列，保证公平性

**2. Deadline**
- 为每个请求设置截止期限
- 优先服务即将超时的请求

**3. Noop（No Operation）**
- 简单的FIFO队列
- 适用于SSD（无寻道时间）

---

## 3. 编译与链接

### 3.1 编译流程

**C程序编译的完整流程**：

```
┌────────────────────────────────────────────────────────────────┐
│                      编译流程                                    │
└────────────────────────────────────────────────────────────────┘

hello.c (源文件)
   │
   │  1. 预处理（Preprocessing）
   │     gcc -E hello.c -o hello.i
   ▼
hello.i (预处理后的文件)
   │  - 展开宏定义（#define）
   │  - 处理条件编译（#if, #ifdef）
   │  - 包含头文件（#include）
   │  - 删除注释
   │
   │  2. 编译（Compilation）
   │     gcc -S hello.i -o hello.s
   ▼
hello.s (汇编文件)
   │  - 词法分析
   │  - 语法分析
   │  - 语义分析
   │  - 中间代码生成
   │  - 代码优化
   │  - 目标代码生成
   │
   │  3. 汇编（Assembly）
   │     gcc -c hello.s -o hello.o
   ▼
hello.o (目标文件)
   │  - 将汇编指令转换为机器码
   │  - 生成符号表
   │  - 生成重定位表
   │
   │  4. 链接（Linking）
   │     gcc hello.o -o hello
   ▼
hello (可执行文件)
   │  - 符号解析（Symbol Resolution）
   │  - 重定位（Relocation）
   │  - 合并段（Section Merging）
```

**示例代码**：

```c
// hello.c
#include <stdio.h>

#define MESSAGE "Hello, World!"

int main() {
    printf("%s\n", MESSAGE);
    return 0;
}
```

**1. 预处理后（hello.i）**：
```c
// 展开stdio.h的内容（数千行）
...
extern int printf (const char *__restrict __format, ...);
...

int main() {
    printf("%s\n", "Hello, World!");  // 宏已展开
    return 0;
}
```

**2. 编译后（hello.s）**：
```asm
    .file   "hello.c"
    .section    .rodata
.LC0:
    .string "%s\n"
.LC1:
    .string "Hello, World!"
    .text
    .globl  main
    .type   main, @function
main:
    pushq   %rbp
    movq    %rsp, %rbp
    movl    $.LC1, %esi
    movl    $.LC0, %edi
    movl    $0, %eax
    call    printf
    movl    $0, %eax
    popq    %rbp
    ret
```

**3. 汇编后（hello.o，二进制）**：
```
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00...
  Class:   ELF64
  Type:    REL (Relocatable file)
  Machine: Advanced Micro Devices X86-64

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .text             PROGBITS         0000000000000000  00000040
       0000000000000015  0000000000000000  AX       0     0     1
  [ 2] .rela.text        RELA             0000000000000000  00000298
       0000000000000030  0000000000000018   I       8     1     8
  [ 3] .data             PROGBITS         0000000000000000  00000055
       0000000000000000  0000000000000000  WA       0     0     1
  [ 4] .bss              NOBITS           0000000000000000  00000055
       0000000000000000  0000000000000000  WA       0     0     1
  [ 5] .rodata           PROGBITS         0000000000000000  00000055
       0000000000000012  0000000000000000   A       0     0     1
  ...

Symbol table '.symtab' contains 10 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS hello.c
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    5
     4: 0000000000000000     0 SECTION LOCAL  DEFAULT    6
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    7
     6: 0000000000000000     0 SECTION LOCAL  DEFAULT    8
     7: 0000000000000000     0 SECTION LOCAL  DEFAULT    9
     8: 0000000000000000    21 FUNC    GLOBAL DEFAULT    1 main
     9: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND printf

Relocation section '.rela.text' at offset 0x298 contains 2 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000007  000300000002 R_X86_64_PC32     0000000000000000 .rodata + 10
00000000000c  000900000002 R_X86_64_PC32     0000000000000000 printf - 4
```

### 3.2 静态链接

**链接过程**：

**1. 符号解析（Symbol Resolution）**

**目的**：将符号引用与符号定义关联起来

**示例**：
```c
// main.c
extern int add(int a, int b);

int main() {
    int result = add(3, 4);
    return result;
}

// add.c
int add(int a, int b) {
    return a + b;
}
```

编译后：
```
main.o:
  符号定义: main
  符号引用: add (UND, 未定义)

add.o:
  符号定义: add
  符号引用: 无
```

链接后：
```
hello:
  符号定义: main, add
  符号引用: 无（全部解析完成）
```

**2. 重定位（Relocation）**

**目的**：修正目标文件中的地址引用

**示例**：
```asm
// main.o中的代码（链接前）
main:
    call add    ; add的地址未知（0x00000000）

// 重定位表
Relocation section '.rela.text':
  Offset: 0x000c
  Type: R_X86_64_PC32
  Symbol: add
  Addend: -4
```

链接后：
```asm
// hello中的代码（链接后）
main:
    call add    ; add的地址已确定（0x00401020）
```

**重定位类型**：

| 类型 | 含义 | 计算公式 |
|------|------|----------|
| R_X86_64_PC32 | PC相对寻址 | S + A - P |
| R_X86_64_32 | 绝对寻址（32位） | S + A |
| R_X86_64_64 | 绝对寻址（64位） | S + A |

- S: 符号地址
- A: 加数（Addend）
- P: 重定位位置

**3. 段合并（Section Merging）**

```
┌────────────────────────────────────────────────────────────┐
│                    段合并过程                                │
└────────────────────────────────────────────────────────────┘

main.o           add.o            hello (链接后)
┌────────┐      ┌────────┐        ┌────────────┐
│ .text  │      │ .text  │        │   .text    │
│  main  │      │  add   │  ───▶  │  main      │
└────────┘      └────────┘        │  add       │
┌────────┐      ┌────────┐        └────────────┘
│ .data  │      │ .data  │        ┌────────────┐
│  ...   │      │  ...   │  ───▶  │   .data    │
└────────┘      └────────┘        │  ...       │
┌────────┐      ┌────────┐        └────────────┘
│ .rodata│      │ .rodata│        ┌────────────┐
│  ...   │      │  ...   │  ───▶  │  .rodata   │
└────────┘      └────────┘        │  ...       │
                                  └────────────┘
```

**链接器实现（简化）**：

```c
// 符号表
struct symbol {
    char *name;
    void *addr;
    int type;  // GLOBAL, LOCAL, WEAK
};

// 重定位条目
struct relocation {
    void *offset;      // 需要重定位的位置
    int type;          // 重定位类型
    struct symbol *sym;  // 引用的符号
    int addend;        // 加数
};

// 链接过程
void link(object_file *files[], int n) {
    // 1. 符号解析
    struct symbol *symtab = merge_symbols(files, n);

    // 2. 分配地址
    assign_addresses(files, n);

    // 3. 重定位
    for (int i = 0; i < n; i++) {
        for (each relocation in files[i]) {
            relocate(relocation, symtab);
        }
    }

    // 4. 生成可执行文件
    write_executable(files, n);
}

// 重定位函数
void relocate(struct relocation *rel, struct symbol *symtab) {
    struct symbol *sym = lookup_symbol(symtab, rel->sym->name);

    void *ref_addr = rel->offset;
    void *def_addr = sym->addr;

    switch (rel->type) {
    case R_X86_64_PC32:
        // PC相对寻址: S + A - P
        *(int *)ref_addr = (int)(def_addr + rel->addend - ref_addr);
        break;
    case R_X86_64_32:
        // 绝对寻址（32位）: S + A
        *(int *)ref_addr = (int)(def_addr + rel->addend);
        break;
    case R_X86_64_64:
        // 绝对寻址（64位）: S + A
        *(long *)ref_addr = (long)(def_addr + rel->addend);
        break;
    }
}
```

### 3.3 动态链接

**静态链接的问题**：
1. **浪费磁盘空间**：每个程序都包含完整的库代码
2. **浪费内存**：多个程序加载相同的库代码到内存
3. **更新困难**：库更新后需要重新编译所有程序

**动态链接的优势**：
1. **节省磁盘和内存**：多个程序共享同一份库代码
2. **易于更新**：库更新后，程序自动使用新版本
3. **插件机制**：运行时加载/卸载库

**动态链接方式**：

**1. 加载时动态链接（Load-time Dynamic Linking）**

程序启动时，动态链接器（ld.so）加载所有依赖的共享库

```c
// 编译动态链接程序
gcc -o hello main.c -lm  // 链接libm.so

// 查看依赖
$ ldd hello
    linux-vdso.so.1 (0x00007ffe7b3f1000)
    libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f8e2e400000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f8e2e200000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f8e2e600000)
```

**2. 运行时动态链接（Run-time Dynamic Linking）**

程序运行过程中，使用dlopen/dlsym显式加载库

```c
#include <dlfcn.h>

int main() {
    // 加载共享库
    void *handle = dlopen("libexample.so", RTLD_LAZY);
    if (!handle) {
        fprintf(stderr, "dlopen failed: %s\n", dlerror());
        return 1;
    }

    // 查找符号
    typedef int (*func_t)(int, int);
    func_t add = (func_t)dlsym(handle, "add");
    if (!add) {
        fprintf(stderr, "dlsym failed: %s\n", dlerror());
        return 1;
    }

    // 调用函数
    int result = add(3, 4);
    printf("Result: %d\n", result);

    // 卸载库
    dlclose(handle);

    return 0;
}
```

**位置无关代码（PIC, Position Independent Code）**：

**问题**：共享库被加载到不同进程的不同地址，如何保证代码正确执行？

**解决方案**：使用位置无关代码

**实现**：

```c
// 共享库代码
int global_var = 100;

int add(int a, int b) {
    return a + b + global_var;
}
```

**编译为PIC**：
```bash
gcc -fPIC -shared -o libexample.so example.c
```

**PIC代码（使用GOT和PLT）**：

```asm
// 非PIC代码（绝对寻址）
add:
    movl global_var, %eax  ; 直接访问全局变量（地址固定）
    addl %edi, %eax
    addl %esi, %eax
    ret

// PIC代码（相对寻址）
add:
    call __x86.get_pc_thunk.ax  ; 获取当前PC
    addl $_GLOBAL_OFFSET_TABLE_, %eax  ; 计算GOT地址
    movl global_var@GOT(%eax), %edx  ; 从GOT获取global_var地址
    movl (%edx), %edx  ; 读取global_var的值
    addl %edi, %edx
    addl %esi, %edx
    movl %edx, %eax
    ret

__x86.get_pc_thunk.ax:
    movl (%esp), %eax  ; 返回地址即为PC
    ret
```

**GOT（Global Offset Table）**：

**作用**：存储全局变量和函数的地址

```
┌────────────────────────────────────────────────────────┐
│                    GOT结构                               │
└────────────────────────────────────────────────────────┘

.got (全局变量)
  global_var@GOT  ─────▶  [0x00601000] ─────▶ global_var的实际地址

.got.plt (函数)
  printf@GOT      ─────▶  [0x00601020] ─────▶ printf的实际地址（由动态链接器填充）
```

**PLT（Procedure Linkage Table）**：

**作用**：延迟绑定（Lazy Binding），第一次调用时才解析函数地址

```asm
// PLT条目
printf@plt:
    jmp *printf@GOT(%rip)  ; 跳转到GOT中的地址
    push $index            ; 如果是第一次调用，GOT指向下一条指令
    jmp .plt0              ; 跳转到动态链接器

// 第一次调用流程
1. 调用 printf@plt
2. 跳转到 printf@GOT（初始值指向下一条指令）
3. push $index
4. jmp .plt0（动态链接器入口）
5. 动态链接器解析printf地址，更新printf@GOT
6. 跳转到printf

// 后续调用
1. 调用 printf@plt
2. 直接跳转到 printf@GOT（已经是printf的真实地址）
```

**动态链接器（ld.so）工作流程**：

```c
// 简化的动态链接器实现
void _start() {
    // 1. 解析ELF头
    Elf64_Ehdr *ehdr = (Elf64_Ehdr *)program_base;

    // 2. 加载依赖的共享库
    for (each DT_NEEDED in dynamic_section) {
        load_library(needed_name);
    }

    // 3. 符号解析
    for (each relocation in .rela.plt) {
        void *sym_addr = resolve_symbol(relocation->symbol);
        *relocation->offset = sym_addr;
    }

    // 4. 跳转到程序入口点
    void (*entry)(void) = ehdr->e_entry;
    entry();
}

// 延迟绑定（Lazy Binding）
void *_dl_runtime_resolve(int index) {
    // 1. 获取符号名
    char *sym_name = get_symbol_name(index);

    // 2. 在所有加载的库中查找符号
    void *sym_addr = lookup_symbol(sym_name);

    // 3. 更新GOT
    update_got(index, sym_addr);

    // 4. 返回符号地址
    return sym_addr;
}
```

---

## 4. 操作系统架构

### 4.1 宏内核 vs 微内核

**宏内核（Monolithic Kernel）**：

**特点**：
- 所有系统服务运行在内核态
- 模块间可以直接调用
- 性能高但可靠性较差

**架构**：
```
┌──────────────────────────────────────────────────────────┐
│                  用户态（User Space）                      │
│   应用程序、库、Shell等                                     │
└─────────────────────┬────────────────────────────────────┘
                      │ 系统调用
                      ▼
┌──────────────────────────────────────────────────────────┐
│                  内核态（Kernel Space）                    │
│                                                          │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐             │
│  │进程管理  │  │ 内存管理  │  │文件系统  │             │
│  └──────────┘  └───────────┘  └──────────┘             │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐             │
│  │ 设备驱动 │  │  网络协议栈│  │  IPC     │             │
│  └──────────┘  └───────────┘  └──────────┘             │
│                                                          │
│  所有模块在同一地址空间，可以直接调用                       │
└──────────────────────────────────────────────────────────┘
```

**优点**：
- 性能高（无需上下文切换）
- 实现简单

**缺点**：
- 可靠性差（一个模块崩溃导致整个内核崩溃）
- 难以维护（模块间耦合严重）

**代表**：Linux、Unix、Windows早期版本

**微内核（Microkernel）**：

**特点**：
- 内核只保留最基本的功能（进程调度、IPC、内存管理）
- 其他服务作为用户态进程运行
- 高可靠性但性能较差

**架构**：
```
┌──────────────────────────────────────────────────────────┐
│                  用户态（User Space）                      │
│                                                          │
│  ┌──────────┐  ┌───────────┐  ┌──────────┐             │
│  │应用程序  │  │文件服务器 │  │ 设备驱动 │             │
│  └────┬─────┘  └─────┬─────┘  └────┬─────┘             │
│       │              │              │                   │
│       └──────────────┼──────────────┘                   │
│                      │ IPC通信                           │
└──────────────────────┼──────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────┐
│              微内核（Microkernel）                         │
│                                                          │
│  ┌──────────────┐  ┌─────────────┐  ┌──────────────┐   │
│  │  进程调度    │  │     IPC     │  │  内存管理    │   │
│  └──────────────┘  └─────────────┘  └──────────────┘   │
│                                                          │
│  最小化的内核，只提供核心功能                               │
└──────────────────────────────────────────────────────────┘
```

**优点**：
- 可靠性高（服务进程崩溃不影响内核）
- 易于扩展和维护
- 安全性好（服务进程隔离）

**缺点**：
- 性能较差（频繁的IPC和上下文切换）
- 实现复杂

**代表**：QNX、Minix、seL4

**混合内核（Hybrid Kernel）**：

**特点**：结合宏内核和微内核的优点

**架构**：
- 核心服务（调度、内存管理）在内核态
- 部分服务（文件系统、驱动）可以在用户态运行

**代表**：Windows NT、macOS（XNU）

**对比表格**：

| 特性 | 宏内核 | 微内核 | 混合内核 |
|------|--------|--------|----------|
| 性能 | 高 | 低 | 中 |
| 可靠性 | 低 | 高 | 中 |
| 安全性 | 低 | 高 | 中 |
| 扩展性 | 差 | 好 | 中 |
| 复杂度 | 低 | 高 | 中 |
| 代表系统 | Linux, Unix | QNX, Minix | Windows NT, macOS |

### 4.2 微内核设计

**微内核的核心原则**：

1. **最小化内核**：只保留必不可少的功能
2. **机制与策略分离**：内核提供机制，策略由用户空间决定
3. **服务器模型**：系统服务作为用户态进程运行

**微内核必须提供的功能**：

1. **地址空间管理**
2. **线程管理**
3. **进程间通信（IPC）**
4. **中断处理**

**微内核IPC机制**：

**同步IPC（L4内核）**：

```c
// 发送消息并等待回复
int ipc_call(thread_id_t dest, message_t *msg) {
    // 1. 发送消息
    send(dest, msg);

    // 2. 阻塞等待回复
    receive(dest, msg);

    return 0;
}

// 接收消息并回复
int ipc_reply_and_receive(thread_id_t *sender, message_t *msg) {
    // 1. 回复上一个消息
    reply(*sender, msg);

    // 2. 等待下一个消息
    receive(ANY, msg, sender);

    return 0;
}
```

**内核实现**：
```c
// IPC系统调用
int sys_ipc(thread_id_t dest, message_t *send_msg,
            thread_id_t *from, message_t *recv_msg) {

    // 1. 发送阶段
    if (dest != INVALID_THREAD) {
        thread_t *dest_thread = get_thread(dest);

        if (dest_thread->state == IPC_WAITING &&
            dest_thread->ipc_from == current->id) {
            // 对方正在等待，直接传递消息
            copy_message(&dest_thread->ipc_msg, send_msg);
            dest_thread->state = READY;
            schedule(dest_thread);
        } else {
            // 对方未准备好，阻塞
            current->ipc_msg = *send_msg;
            current->ipc_to = dest;
            current->state = IPC_SENDING;
            schedule();
        }
    }

    // 2. 接收阶段
    if (from != NULL) {
        // 检查是否有等待的发送者
        thread_t *sender = find_sender(*from);

        if (sender) {
            // 有发送者，直接接收
            copy_message(recv_msg, &sender->ipc_msg);
            *from = sender->id;
            sender->state = READY;
            schedule(sender);
        } else {
            // 无发送者，阻塞等待
            current->ipc_from = *from;
            current->state = IPC_WAITING;
            schedule();
        }
    }

    return 0;
}
```

**seL4微内核**：

**特点**：
- 经过形式化验证的微内核
- 保证内存安全和功能正确性
- 提供细粒度的权限控制

**核心抽象**：

1. **CNode（Capability Node）**：权能表
2. **Thread**：线程
3. **Endpoint**：IPC端点
4. **Address Space**：地址空间

```c
// seL4权能系统
typedef struct {
    uint32_t type;       // 权能类型
    uint32_t rights;     // 权限（读、写、授权）
    void *object;        // 指向的对象
} capability_t;

// 使用权能访问对象
int seL4_Send(capability_t ep_cap, message_t *msg) {
    // 检查权能类型
    if (ep_cap.type != ENDPOINT_TYPE) {
        return ERROR_INVALID_CAPABILITY;
    }

    // 检查权限
    if (!(ep_cap.rights & WRITE)) {
        return ERROR_NO_PERMISSION;
    }

    // 执行操作
    endpoint_t *ep = (endpoint_t *)ep_cap.object;
    send_message(ep, msg);

    return 0;
}
```

---

继续编写其他章节...（由于内容太长，我会分批完成）

这个文档我会继续完善，包括：
- 5. I/O系统（详细讲解I/O子系统）
- 6. 数据结构与算法（红黑树、哈希表等）
- 7. 微内核设计（更深入的微内核知识）
- 8. 实战项目（推荐的学习项目）

需要我继续吗？

## 5. 数据结构与算法（内核常用）

### 5.1 链表

**内核双向循环链表（Linux list.h）**：

**特点**：
- 侵入式设计（链表节点嵌入到数据结构中）
- 通用性强（可用于任何数据结构）
- 无需额外内存分配

**实现**：

```c
// 链表节点
struct list_head {
    struct list_head *next, *prev;
};

// 初始化链表头
#define LIST_HEAD_INIT(name) { &(name), &(name) }
#define LIST_HEAD(name) \
    struct list_head name = LIST_HEAD_INIT(name)

// 初始化节点
static inline void INIT_LIST_HEAD(struct list_head *list) {
    list->next = list;
    list->prev = list;
}

// 插入节点（内部函数）
static inline void __list_add(struct list_head *new,
                               struct list_head *prev,
                               struct list_head *next) {
    next->prev = new;
    new->next = next;
    new->prev = prev;
    prev->next = new;
}

// 在头部插入
static inline void list_add(struct list_head *new, struct list_head *head) {
    __list_add(new, head, head->next);
}

// 在尾部插入
static inline void list_add_tail(struct list_head *new, struct list_head *head) {
    __list_add(new, head->prev, head);
}

// 删除节点
static inline void list_del(struct list_head *entry) {
    entry->next->prev = entry->prev;
    entry->prev->next = entry->next;
    entry->next = NULL;
    entry->prev = NULL;
}

// 遍历链表
#define list_for_each(pos, head) \
    for (pos = (head)->next; pos != (head); pos = pos->next)

// 获取包含链表节点的结构体
#define list_entry(ptr, type, member) \
    container_of(ptr, type, member)

#define container_of(ptr, type, member) ({                  \
    const typeof(((type *)0)->member) *__mptr = (ptr);      \
    (type *)((char *)__mptr - offsetof(type, member)); })
```

**使用示例**：

```c
// 定义数据结构
struct task {
    int pid;
    char name[32];
    struct list_head list;  // 链表节点嵌入
};

// 初始化链表头
LIST_HEAD(task_list);

// 添加任务
struct task *t1 = kmalloc(sizeof(*t1), GFP_KERNEL);
t1->pid = 1;
strcpy(t1->name, "init");
INIT_LIST_HEAD(&t1->list);
list_add_tail(&t1->list, &task_list);

// 遍历任务
struct list_head *pos;
list_for_each(pos, &task_list) {
    struct task *t = list_entry(pos, struct task, list);
    printk("PID: %d, Name: %s\n", t->pid, t->name);
}
```

### 5.2 红黑树

**特性**：
- 自平衡二叉搜索树
- 保证查找、插入、删除的时间复杂度为O(log n)
- Linux内核广泛使用（进程调度、虚拟内存管理）

**红黑树性质**：
1. 每个节点是红色或黑色
2. 根节点是黑色
3. 所有叶子节点（NIL）是黑色
4. 红色节点的两个子节点都是黑色（不能有连续的红色节点）
5. 从任一节点到其叶子节点的所有路径包含相同数目的黑色节点

**数据结构**：

```c
// 红黑树节点
struct rb_node {
    unsigned long rb_parent_color;  // 父节点指针 + 颜色（最低位）
    struct rb_node *rb_right;
    struct rb_node *rb_left;
};

// 红黑树根
struct rb_root {
    struct rb_node *rb_node;
};

// 颜色定义
#define RB_RED   0
#define RB_BLACK 1

// 获取颜色
#define rb_color(n) ((n)->rb_parent_color & 1)

// 设置颜色
static inline void rb_set_red(struct rb_node *node) {
    node->rb_parent_color &= ~1;
}

static inline void rb_set_black(struct rb_node *node) {
    node->rb_parent_color |= 1;
}
```

**插入操作**：

```c
// 插入节点（简化版）
void rb_insert_color(struct rb_node *node, struct rb_root *root) {
    struct rb_node *parent, *gparent;

    // 新节点初始为红色
    rb_set_red(node);

    while ((parent = rb_parent(node)) && rb_is_red(parent)) {
        gparent = rb_parent(parent);

        if (parent == gparent->rb_left) {
            // 父节点是祖父节点的左子节点
            struct rb_node *uncle = gparent->rb_right;

            if (uncle && rb_is_red(uncle)) {
                // Case 1: 叔叔节点是红色
                rb_set_black(uncle);
                rb_set_black(parent);
                rb_set_red(gparent);
                node = gparent;
                continue;
            }

            if (parent->rb_right == node) {
                // Case 2: 当前节点是父节点的右子节点
                __rb_rotate_left(parent, root);
                struct rb_node *tmp = parent;
                parent = node;
                node = tmp;
            }

            // Case 3: 当前节点是父节点的左子节点
            rb_set_black(parent);
            rb_set_red(gparent);
            __rb_rotate_right(gparent, root);
        } else {
            // 对称情况（父节点是祖父节点的右子节点）
            // ...
        }
    }

    rb_set_black(root->rb_node);  // 根节点始终为黑色
}

// 左旋
static void __rb_rotate_left(struct rb_node *node, struct rb_root *root) {
    struct rb_node *right = node->rb_right;
    struct rb_node *parent = rb_parent(node);

    if ((node->rb_right = right->rb_left))
        rb_set_parent(right->rb_left, node);

    right->rb_left = node;
    rb_set_parent(right, parent);

    if (parent) {
        if (node == parent->rb_left)
            parent->rb_left = right;
        else
            parent->rb_right = right;
    } else
        root->rb_node = right;

    rb_set_parent(node, right);
}
```

**使用场景**：

**1. CFS调度器（按vruntime排序进程）**
```c
struct cfs_rq {
    struct rb_root_cached tasks_timeline;  // 红黑树根
};

// 插入进程到红黑树
void enqueue_entity(struct cfs_rq *cfs_rq, struct sched_entity *se) {
    struct rb_node **link = &cfs_rq->tasks_timeline.rb_root.rb_node;
    struct rb_node *parent = NULL;

    // 查找插入位置
    while (*link) {
        parent = *link;
        struct sched_entity *entry = rb_entry(parent, struct sched_entity, run_node);

        if (se->vruntime < entry->vruntime)
            link = &parent->rb_left;
        else
            link = &parent->rb_right;
    }

    // 插入节点
    rb_link_node(&se->run_node, parent, link);
    rb_insert_color(&se->run_node, &cfs_rq->tasks_timeline.rb_root);
}

// 选择vruntime最小的进程
struct sched_entity *__pick_first_entity(struct cfs_rq *cfs_rq) {
    struct rb_node *left = rb_first_cached(&cfs_rq->tasks_timeline);

    if (!left)
        return NULL;

    return rb_entry(left, struct sched_entity, run_node);
}
```

**2. VMA管理（按地址排序虚拟内存区域）**
```c
struct mm_struct {
    struct rb_root mm_rb;  // VMA红黑树
};

// 查找包含地址的VMA
struct vm_area_struct *find_vma(struct mm_struct *mm, unsigned long addr) {
    struct rb_node *node = mm->mm_rb.rb_node;
    struct vm_area_struct *vma = NULL;

    while (node) {
        struct vm_area_struct *tmp = rb_entry(node, struct vm_area_struct, vm_rb);

        if (tmp->vm_end > addr) {
            vma = tmp;
            if (tmp->vm_start <= addr)
                break;
            node = node->rb_left;
        } else
            node = node->rb_right;
    }

    return vma;
}
```

### 5.3 哈希表

**Linux内核哈希表（hlist）**：

**特点**：
- 单向链表（节省内存）
- 支持链地址法解决冲突

**数据结构**：

```c
// 哈希表头
struct hlist_head {
    struct hlist_node *first;
};

// 哈希节点
struct hlist_node {
    struct hlist_node *next, **pprev;  // pprev指向前一个节点的next指针
};

// 初始化
#define HLIST_HEAD_INIT { .first = NULL }
#define HLIST_HEAD(name) struct hlist_head name = HLIST_HEAD_INIT

// 添加节点到头部
static inline void hlist_add_head(struct hlist_node *n, struct hlist_head *h) {
    struct hlist_node *first = h->first;
    n->next = first;
    if (first)
        first->pprev = &n->next;
    h->first = n;
    n->pprev = &h->first;
}

// 删除节点
static inline void hlist_del(struct hlist_node *n) {
    struct hlist_node *next = n->next;
    struct hlist_node **pprev = n->pprev;

    *pprev = next;
    if (next)
        next->pprev = pprev;
}

// 遍历
#define hlist_for_each_entry(pos, head, member)  \
    for (pos = hlist_entry_safe((head)->first, typeof(*(pos)), member); \
         pos; \
         pos = hlist_entry_safe((pos)->member.next, typeof(*(pos)), member))
```

**使用示例（进程哈希表）**：

```c
#define PID_HASH_SIZE 4096
struct hlist_head pid_hash[PID_HASH_SIZE];

// 进程结构
struct task_struct {
    pid_t pid;
    char name[32];
    struct hlist_node pid_hash_node;
};

// 哈希函数
static inline unsigned int pid_hashfn(pid_t pid) {
    return pid % PID_HASH_SIZE;
}

// 添加进程到哈希表
void add_task_to_hash(struct task_struct *task) {
    unsigned int hash = pid_hashfn(task->pid);
    hlist_add_head(&task->pid_hash_node, &pid_hash[hash]);
}

// 查找进程
struct task_struct *find_task_by_pid(pid_t pid) {
    unsigned int hash = pid_hashfn(pid);
    struct task_struct *task;

    hlist_for_each_entry(task, &pid_hash[hash], pid_hash_node) {
        if (task->pid == pid)
            return task;
    }

    return NULL;
}
```

### 5.4 队列和栈

**内核队列（kfifo）**：

```c
// 环形缓冲区（FIFO）
struct kfifo {
    unsigned char *buffer;  // 缓冲区
    unsigned int size;      // 大小（必须是2的幂）
    unsigned int in;        // 写位置
    unsigned int out;       // 读位置
    spinlock_t *lock;       // 自旋锁
};

// 初始化
int kfifo_alloc(struct kfifo *fifo, unsigned int size, gfp_t gfp_mask) {
    // size必须是2的幂
    if (!is_power_of_2(size))
        size = roundup_pow_of_two(size);

    fifo->buffer = kmalloc(size, gfp_mask);
    if (!fifo->buffer)
        return -ENOMEM;

    fifo->size = size;
    fifo->in = fifo->out = 0;

    return 0;
}

// 入队
unsigned int kfifo_in(struct kfifo *fifo, const void *from, unsigned int len) {
    unsigned long flags;

    spin_lock_irqsave(fifo->lock, flags);

    len = min(len, fifo->size - fifo->in + fifo->out);

    // 拷贝数据到缓冲区
    unsigned int l = min(len, fifo->size - (fifo->in & (fifo->size - 1)));
    memcpy(fifo->buffer + (fifo->in & (fifo->size - 1)), from, l);
    memcpy(fifo->buffer, from + l, len - l);

    // 内存屏障（保证写操作完成）
    smp_wmb();

    fifo->in += len;

    spin_unlock_irqrestore(fifo->lock, flags);

    return len;
}

// 出队
unsigned int kfifo_out(struct kfifo *fifo, void *to, unsigned int len) {
    unsigned long flags;

    spin_lock_irqsave(fifo->lock, flags);

    len = min(len, fifo->in - fifo->out);

    // 拷贝数据到用户缓冲区
    unsigned int l = min(len, fifo->size - (fifo->out & (fifo->size - 1)));
    memcpy(to, fifo->buffer + (fifo->out & (fifo->size - 1)), l);
    memcpy(to + l, fifo->buffer, len - l);

    // 内存屏障
    smp_rmb();

    fifo->out += len;

    spin_unlock_irqrestore(fifo->lock, flags);

    return len;
}
```

### 5.5 排序算法

**Linux内核排序（lib/sort.c）**：

**堆排序实现**：

```c
// 通用排序函数
void sort(void *base, size_t num, size_t size,
          int (*cmp)(const void *, const void *),
          void (*swap)(void *, void *, int)) {

    // 使用堆排序
    int i = (num / 2 - 1) * size, n = num * size, c, r;

    // 建堆
    for (; i >= 0; i -= size) {
        for (r = i; r * 2 + size < n; r = c) {
            c = r * 2 + size;
            if (c < n - size &&
                cmp(base + c, base + c + size) < 0)
                c += size;
            if (cmp(base + r, base + c) >= 0)
                break;
            swap(base + r, base + c, size);
        }
    }

    // 排序
    for (i = n - size; i > 0; i -= size) {
        swap(base, base + i, size);
        for (r = 0; r * 2 + size < i; r = c) {
            c = r * 2 + size;
            if (c < i - size &&
                cmp(base + c, base + c + size) < 0)
                c += size;
            if (cmp(base + r, base + c) >= 0)
                break;
            swap(base + r, base + c, size);
        }
    }
}

// 使用示例
int cmp_int(const void *a, const void *b) {
    return *(int *)a - *(int *)b;
}

void swap_int(void *a, void *b, int size) {
    int t = *(int *)a;
    *(int *)a = *(int *)b;
    *(int *)b = t;
}

int arr[] = {5, 2, 8, 1, 9};
sort(arr, 5, sizeof(int), cmp_int, swap_int);
```

---

## 6. 实战学习路径

### 6.1 推荐学习项目

**1. 从零实现简单OS（推荐顺序）**

**Level 1: Bare Metal编程**
```c
// 1. 编写引导加载程序（Bootloader）
// boot.asm - 16位实模式
[BITS 16]
[ORG 0x7C00]

start:
    mov ax, 0x07C0
    mov ds, ax
    mov es, ax

    ; 打印字符串
    mov si, msg
    call print_string

    ; 无限循环
    jmp $

print_string:
    lodsb
    or al, al
    jz done
    mov ah, 0x0E
    int 0x10
    jmp print_string
done:
    ret

msg db 'Hello, Kernel!', 0

times 510-($-$$) db 0
dw 0xAA55  ; 引导扇区签名
```

**Level 2: 进入保护模式**
```asm
; 切换到32位保护模式
cli                    ; 禁用中断
lgdt [gdt_descriptor]  ; 加载GDT
mov eax, cr0
or eax, 0x1            ; 设置PE位
mov cr0, eax
jmp CODE_SEG:init_pm   ; 远跳转到保护模式

[BITS 32]
init_pm:
    mov ax, DATA_SEG
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    mov ss, ax
    mov ebp, 0x90000
    mov esp, ebp

    call main          ; 调用C代码
```

**Level 3: 实现基本内核**
```c
// kernel.c - C语言内核入口
void main() {
    clear_screen();
    print("Welcome to MyOS!\n");

    // 初始化中断描述符表
    init_idt();

    // 初始化定时器
    init_timer(50);  // 50Hz

    // 初始化键盘
    init_keyboard();

    // 进入主循环
    while (1) {
        asm("hlt");  // 等待中断
    }
}

// 中断处理
void timer_handler() {
    static int ticks = 0;
    ticks++;
    if (ticks % 50 == 0) {
        print("1 second elapsed\n");
    }
}
```

**2. 实现核心功能模块**

**内存管理器**：
```c
// 简单的物理内存管理（位图法）
#define BITMAP_SIZE 4096

uint32_t mem_bitmap[BITMAP_SIZE];  // 管理128MB内存（每位表示4KB）

void *pmm_alloc_page() {
    for (int i = 0; i < BITMAP_SIZE; i++) {
        if (mem_bitmap[i] != 0xFFFFFFFF) {
            // 找到空闲页
            int bit = __builtin_ffs(~mem_bitmap[i]) - 1;
            mem_bitmap[i] |= (1 << bit);
            return (void *)((i * 32 + bit) * 4096);
        }
    }
    return NULL;  // 内存不足
}

void pmm_free_page(void *addr) {
    uint32_t page = (uint32_t)addr / 4096;
    mem_bitmap[page / 32] &= ~(1 << (page % 32));
}
```

**进程调度器**：
```c
// 简单的轮转调度
#define MAX_TASKS 64

struct task {
    uint32_t esp;      // 栈指针
    uint32_t ebp;      // 基址指针
    uint32_t eip;      // 指令指针
    uint32_t eflags;   // 标志寄存器
    int state;         // 状态
    char name[32];
};

struct task tasks[MAX_TASKS];
int current_task = 0;
int num_tasks = 0;

void schedule() {
    // 保存当前任务状态
    save_context(&tasks[current_task]);

    // 选择下一个任务
    do {
        current_task = (current_task + 1) % num_tasks;
    } while (tasks[current_task].state != TASK_RUNNING);

    // 切换到新任务
    load_context(&tasks[current_task]);
}
```

**文件系统**：
```c
// 简单的FAT12文件系统实现
struct fat12_boot_sector {
    uint8_t jump[3];
    char oem[8];
    uint16_t bytes_per_sector;
    uint8_t sectors_per_cluster;
    uint16_t reserved_sectors;
    uint8_t num_fats;
    uint16_t root_entries;
    uint16_t total_sectors;
    // ...
};

struct directory_entry {
    char name[11];
    uint8_t attributes;
    uint8_t reserved;
    uint8_t creation_time_ms;
    uint16_t creation_time;
    uint16_t creation_date;
    uint16_t last_access_date;
    uint16_t cluster_high;
    uint16_t modified_time;
    uint16_t modified_date;
    uint16_t cluster_low;
    uint32_t file_size;
};

// 读取文件
int read_file(const char *filename, void *buffer) {
    // 1. 在根目录中查找文件
    struct directory_entry *entry = find_file(filename);
    if (!entry)
        return -1;

    // 2. 遍历FAT链表读取簇
    uint16_t cluster = entry->cluster_low;
    uint32_t offset = 0;

    while (cluster < 0xFF8) {
        read_cluster(cluster, buffer + offset);
        offset += CLUSTER_SIZE;
        cluster = get_next_cluster(cluster);
    }

    return entry->file_size;
}
```

### 6.2 推荐学习资源

**经典书籍**：

1. **《操作系统概念》（恐龙书）**
   - 作者：Abraham Silberschatz
   - 难度：⭐⭐⭐
   - 适合：理论基础

2. **《深入理解计算机系统》（CSAPP）**
   - 作者：Randal E. Bryant
   - 难度：⭐⭐⭐⭐
   - 适合：系统级编程

3. **《Linux内核设计与实现》**
   - 作者：Robert Love
   - 难度：⭐⭐⭐⭐
   - 适合：Linux内核入门

4. **《深入Linux内核架构》**
   - 作者：Wolfgang Mauerer
   - 难度：⭐⭐⭐⭐⭐
   - 适合：深入理解Linux

5. **《Orange'S：一个操作系统的实现》**
   - 作者：于渊
   - 难度：⭐⭐⭐
   - 适合：实践项目

**开源项目**：

1. **xv6（MIT教学操作系统）**
   - 地址：https://github.com/mit-pdos/xv6-public
   - 特点：简单、易懂、适合学习
   - 代码量：~8000行

2. **Linux 0.11**
   - 特点：早期Linux版本，代码简洁
   - 代码量：~10000行

3. **Minix 3**
   - 地址：https://www.minix3.org/
   - 特点：微内核设计典范

4. **seL4**
   - 地址：https://sel4.systems/
   - 特点：形式化验证的微内核

**在线课程**：

1. **MIT 6.828: Operating System Engineering**
   - 使用xv6教学
   - 配套实验完善

2. **Stanford CS140: Operating Systems**
   - Pintos项目

3. **Berkeley CS162: Operating Systems**
   - 配套丰富的讲义

### 6.3 实战练习建议

**阶段1: 基础准备（1-2个月）**
- 熟练掌握C语言（指针、结构体、内存管理）
- 学习汇编语言（x86/x86-64）
- 理解计算机组成原理

**阶段2: 理论学习（2-3个月）**
- 系统学习操作系统概念
- 深入理解进程、内存、文件系统
- 学习编译原理基础

**阶段3: 实践项目（3-6个月）**
- 完成一个简单的OS（参考xv6）
- 实现基本的进程调度
- 实现简单的文件系统

**阶段4: 深入Linux内核（持续）**
- 阅读Linux内核源码
- 编写内核模块
- 参与开源项目

### 6.4 面试准备要点

**必须掌握的知识点**：

1. **进程与线程**
   - ✓ 进程和线程的区别
   - ✓ 进程状态转换
   - ✓ 上下文切换过程
   - ✓ 各种IPC机制
   - ✓ 进程调度算法

2. **内存管理**
   - ✓ 虚拟内存原理
   - ✓ 分页和分段机制
   - ✓ 页表结构（多级页表）
   - ✓ TLB工作原理
   - ✓ 缺页异常处理
   - ✓ 内存分配算法（buddy、slab）

3. **编译与链接**
   - ✓ 编译流程（预处理、编译、汇编、链接）
   - ✓ ELF文件格式
   - ✓ 静态链接和动态链接
   - ✓ PIC（位置无关代码）
   - ✓ GOT和PLT

4. **数据结构**
   - ✓ 链表（Linux list.h）
   - ✓ 红黑树
   - ✓ 哈希表
   - ✓ 队列和栈

5. **微内核设计**
   - ✓ 宏内核vs微内核
   - ✓ IPC机制设计
   - ✓ 权能系统（Capability）

**常见面试题**：

**Q1: 解释进程和线程的区别**
A: （参考1.1节详细内容）

**Q2: 什么是虚拟内存？如何实现地址转换？**
A: （参考2.2节详细内容）

**Q3: Linux如何实现零拷贝？**
A: 
- sendfile系统调用
- mmap + write
- splice系统调用
- DMA直接传输

**Q4: 如何实现一个内存池？**
A:
```c
struct memory_pool {
    void *memory;           // 内存块
    size_t block_size;      // 块大小
    size_t num_blocks;      // 块数量
    void *free_list;        // 空闲链表
};

void *pool_alloc(struct memory_pool *pool) {
    if (!pool->free_list)
        return NULL;

    void *block = pool->free_list;
    pool->free_list = *(void **)block;
    return block;
}

void pool_free(struct memory_pool *pool, void *block) {
    *(void **)block = pool->free_list;
    pool->free_list = block;
}
```

**Q5: 什么是写时复制（Copy-on-Write）？**
A: （参考2.2节缺页异常处理）

---

## 7. 总结与学习建议

### 7.1 知识体系总结

```
┌─────────────────────────────────────────────────────────────┐
│               操作系统内核工程师知识体系                      │
└─────────────────────────────────────────────────────────────┘

1. 进程与线程
   ├─ 进程/线程概念和区别
   ├─ 进程管理（PCB、状态转换）
   ├─ IPC机制（管道、消息队列、共享内存、信号量）
   ├─ 进程同步（互斥锁、自旋锁、读写锁、条件变量）
   ├─ 进程调度（CFS、实时调度）
   └─ 上下文切换

2. 内存管理
   ├─ 内存分配算法（malloc、buddy、slab）
   ├─ 虚拟内存与分页机制
   ├─ 页表结构（4级页表）
   ├─ TLB
   ├─ 缺页异常处理
   ├─ 内存映射（mmap）
   ├─ 分段机制
   └─ 写时复制（COW）

3. 编译与链接
   ├─ 编译流程（预处理、编译、汇编、链接）
   ├─ ELF文件格式
   ├─ 静态链接（符号解析、重定位）
   ├─ 动态链接（GOT、PLT）
   ├─ 位置无关代码（PIC）
   └─ 动态链接器（ld.so）

4. 操作系统架构
   ├─ 宏内核 vs 微内核
   ├─ 微内核设计原则
   ├─ IPC机制设计
   ├─ 权能系统（Capability）
   └─ 实例（Linux、QNX、seL4）

5. I/O系统
   ├─ I/O层次结构
   ├─ I/O方式（程序控制、中断、DMA）
   ├─ I/O调度器
   └─ 设备驱动模型

6. 数据结构与算法
   ├─ 链表（Linux list.h）
   ├─ 红黑树
   ├─ 哈希表（hlist）
   ├─ 队列（kfifo）
   └─ 排序算法

7. 实战能力
   ├─ 从零实现OS
   ├─ 阅读Linux源码
   ├─ 编写内核模块
   └─ 调试内核（gdb、qemu）
```

### 7.2 学习建议

**1. 循序渐进**
- 不要急于求成，打好基础
- 先理论后实践
- 从简单的OS开始（xv6）

**2. 动手实践**
- 纸上得来终觉浅，绝知此事要躬行
- 编写代码、调试内核
- 参与开源项目

**3. 深入源码**
- 阅读Linux内核源码
- 理解设计思想
- 学习优秀的代码风格

**4. 持续学习**
- 操作系统是不断演进的
- 关注最新技术（容器、虚拟化）
- 参加技术社区

**5. 记录总结**
- 写博客记录学习过程
- 整理笔记
- 分享给他人

### 7.3 推荐工具

**开发工具**：
- **编译器**：GCC、Clang
- **调试器**：GDB、LLDB
- **虚拟机**：QEMU、VirtualBox
- **版本控制**：Git
- **文本编辑器**：Vim、Emacs、VS Code

**分析工具**：
- **性能分析**：perf、ftrace、eBPF
- **内存分析**：Valgrind、AddressSanitizer
- **代码分析**：Cppcheck、Clang Static Analyzer
- **调用图生成**：Doxygen、Graphviz

---

**祝你学习顺利，成为优秀的内核工程师！** 🚀

**最后的建议**：
- 保持好奇心
- 永不停止学习
- 享受探索的过程

---

## 附录：参考资料

**官方文档**：
- Linux Kernel Documentation: https://www.kernel.org/doc/html/latest/
- Intel Software Developer Manuals: https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html

**在线资源**：
- OSDev Wiki: https://wiki.osdev.org/
- Linux Cross Reference: https://elixir.bootlin.com/linux/latest/source

**开源项目**：
- Linux Kernel: https://github.com/torvalds/linux
- xv6: https://github.com/mit-pdos/xv6-public
- seL4: https://github.com/seL4/seL4

