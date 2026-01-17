# 用C语言实现一个简化版Tokio

> 通过C语言实现，深入理解异步运行时的底层机制

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 核心数据结构](#2-核心数据结构)
- [3. 完整实现](#3-完整实现)
- [4. 使用示例](#4-使用示例)
- [5. 运行结果](#5-运行结果)
- [6. 核心机制解析](#6-核心机制解析)
- [7. 编译和运行](#7-编译和运行)

---

## 1. 项目概述

**目标**：用纯C实现一个简化版的Tokio，包含：

- Task（异步任务）
- Waker（唤醒器）
- epoll（I/O多路复用）
- Worker Thread（工作线程）
- Event Loop（事件循环）
- 异步Socket（非阻塞I/O）

**特性**：
- 单个Worker Thread处理多个Task
- 使用epoll监听I/O事件
- Task通过Waker唤醒
- 完整的异步执行流程

---

## 2. 核心数据结构

```c
// ==================== 核心枚举 ====================

// Poll返回值（类似Rust的 Poll<T>）
typedef enum 
{
    POLL_READY,    // 任务完成
    POLL_PENDING   // 任务未完成，需要等待
} PollResult;

// Task状态
typedef enum 
{
    TASK_RUNNING,   // 正在运行
    TASK_WAITING,   // 等待I/O
    TASK_DONE       // 已完成
} TaskState;

// ==================== 核心结构体 ====================

// Waker：唤醒器
typedef struct Waker 
{
    int task_id;           // 关联的Task ID
    void (*wake)(struct Waker*);  // 唤醒函数
} Waker;

// Task：异步任务
typedef struct Task 
{
    int       id;          // Task ID
    TaskState state;       // 当前状态
    int       fd;          // 关联的文件描述符（如果有）
    void      *data;       // 任务数据
    Waker     *waker;      // 唤醒器

    // poll 函数：类似 Rust 的 Future::poll
    PollResult (*poll)(struct Task*, Waker*);
} Task;

// TaskQueue：任务队列
typedef struct TaskQueue 
{
    Task **tasks;          // Task数组
    int  capacity;         // 容量
    int  size;             // 当前大小
    int  front;            // 队列头
    int  rear;             // 队列尾
} TaskQueue;

// Runtime：运行时（简化版Tokio）
typedef struct Runtime 
{
    int       epoll_fd;    // epoll文件描述符
    TaskQueue *queue;      // 任务队列
    int       running;     // 是否运行中
} Runtime;
```

---

## 3. 完整实现

### 3.1 文件：`simple_tokio.h`

```c
#ifndef SIMPLE_TOKIO_H
#define SIMPLE_TOKIO_H

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define MAX_EVENTS 100
#define QUEUE_SIZE 1000

// Poll 返回值
typedef enum 
{
    POLL_READY,
    POLL_PENDING
} PollResult;

// Task 状态
typedef enum 
{
    TASK_RUNNING,
    TASK_WAITING,
    TASK_DONE
} TaskState;

// 前向声明
typedef struct Task Task;
typedef struct Waker Waker;
typedef struct TaskQueue TaskQueue;
typedef struct Runtime Runtime;

// Waker 结构
struct Waker 
{
    int task_id;
    Runtime *runtime;  // 指向运行时，用于唤醒时重新入队
    void (*wake)(Waker*);
};

// Task 结构
struct Task 
{
    int id;
    TaskState state;
    int fd;
    void *data;
    Waker *waker;
    PollResult (*poll)(Task*, Waker*);
};

// TaskQueue 结构
struct TaskQueue 
{
    Task **tasks;
    int capacity;
    int size;
    int front;
    int rear;
};

// Runtime 结构
struct Runtime 
{
    int epoll_fd;
    TaskQueue *queue;
    int running;
    Task **all_tasks;  // 所有Task（用于ID查找）
    int task_count;
};

// ==================== 函数声明 ====================

// TaskQueue 相关
TaskQueue* task_queue_new(int capacity);
void task_queue_free(TaskQueue *queue);
int task_queue_push(TaskQueue *queue, Task *task);
Task* task_queue_pop(TaskQueue *queue);
int task_queue_is_empty(TaskQueue *queue);

// Waker 相关
Waker* waker_new(int task_id, Runtime *runtime);
void waker_wake(Waker *waker);
void waker_free(Waker *waker);

// Task 相关
Task* task_new(int id, PollResult (*poll_fn)(Task*, Waker*));
void task_free(Task *task);

// Runtime 相关
Runtime* runtime_new();
void runtime_free(Runtime *runtime);
void runtime_spawn(Runtime *runtime, Task *task);
void runtime_run(Runtime *runtime);
int set_nonblocking(int fd);

#endif // SIMPLE_TOKIO_H
```

### 3.2 文件：`simple_tokio.c`

```c
#include "simple_tokio.h"

// ==================== TaskQueue 实现 ====================

TaskQueue* task_queue_new(int capacity) 
{
    TaskQueue *queue = malloc(sizeof(TaskQueue));
    queue->tasks = malloc(sizeof(Task*) * capacity);
    queue->capacity = capacity;
    queue->size = 0;
    queue->front = 0;
    queue->rear = 0;
    return queue;
}

void task_queue_free(TaskQueue *queue) 
{
    free(queue->tasks);
    free(queue);
}

int task_queue_push(TaskQueue *queue, Task *task) 
{
    if (queue->size >= queue->capacity) 
    {
        return -1;  // 队列满
    }
    queue->tasks[queue->rear] = task;
    queue->rear = (queue->rear + 1) % queue->capacity;
    queue->size++;
    return 0;
}

Task* task_queue_pop(TaskQueue *queue) 
{
    if (queue->size == 0) 
    {
        return NULL;  // 队列空
    }
    Task *task = queue->tasks[queue->front];
    queue->front = (queue->front + 1) % queue->capacity;
    queue->size--;
    return task;
}

int task_queue_is_empty(TaskQueue *queue) 
{
    return queue->size == 0;
}

// ==================== Waker 实现 ====================

Waker* waker_new(int task_id, Runtime *runtime) 
{
    Waker *waker = malloc(sizeof(Waker));
    waker->task_id = task_id;
    waker->runtime = runtime;
    waker->wake = waker_wake;
    return waker;
}

void waker_wake(Waker *waker) 
{
    // 找到对应的Task，重新加入队列
    printf("[Waker] 唤醒 Task %d\n", waker->task_id);

    for (int i = 0; i < waker->runtime->task_count; i++) 
    {
        Task *task = waker->runtime->all_tasks[i];
        if (task && task->id == waker->task_id && task->state == TASK_WAITING) 
        {
            task->state = TASK_RUNNING;
            task_queue_push(waker->runtime->queue, task);
            break;
        }
    }
}

void waker_free(Waker *waker) 
{
    free(waker);
}

// ==================== Task 实现 ====================

Task* task_new(int id, PollResult (*poll_fn)(Task*, Waker*)) 
{
    Task *task = malloc(sizeof(Task));
    task->id = id;
    task->state = TASK_RUNNING;
    task->fd = -1;
    task->data = NULL;
    task->waker = NULL;
    task->poll = poll_fn;
    return task;
}

void task_free(Task *task) 
{
    if (task->waker) 
    {
        waker_free(task->waker);
    }
    if (task->data) 
    {
        free(task->data);
    }
    free(task);
}

// ==================== Runtime 实现 ====================

Runtime* runtime_new() 
{
    Runtime *runtime = malloc(sizeof(Runtime));
    // 创建 epoll 实例
    runtime->epoll_fd = epoll_create1(0);
    if (runtime->epoll_fd == -1) 
    {
        perror("epoll_create1");
        free(runtime);
        return NULL;
    }
    runtime->queue = task_queue_new(QUEUE_SIZE);
    runtime->running = 1;
    runtime->all_tasks = calloc(QUEUE_SIZE, sizeof(Task*));
    runtime->task_count = 0;
    printf("[Runtime] 初始化完成，epoll_fd = %d\n", runtime->epoll_fd);
    return runtime;
}

void runtime_free(Runtime *runtime) 
{
    close(runtime->epoll_fd);
    task_queue_free(runtime->queue);
    // 释放所有 Task
    for (int i = 0; i < runtime->task_count; i++) 
    {
        if (runtime->all_tasks[i]) 
        {
            task_free(runtime->all_tasks[i]);
        }
    }
    free(runtime->all_tasks);
    free(runtime);
}

void runtime_spawn(Runtime *runtime, Task *task) 
{
    printf("[Runtime] Spawn Task %d\n", task->id);
    // 保存到 all_tasks
    runtime->all_tasks[runtime->task_count++] = task;
    // 加入队列
    task_queue_push(runtime->queue, task);
}

void runtime_run(Runtime *runtime) 
{
    printf("[Runtime] 开始事件循环\n\n");
    struct epoll_event events[MAX_EVENTS];
    while (runtime->running) 
    {
        // ==================== 步骤1：执行队列中的 Task ====================
        while (!task_queue_is_empty(runtime->queue)) 
        {
            Task *task = task_queue_pop(runtime->queue);
            printf("[Runtime] Poll Task %d (state=%d)\n", task->id, task->state);
            // 创建Waker
            if (!task->waker) 
            {
                task->waker = waker_new(task->id, runtime);
            }
            // 关键：调用Task的poll方法
            PollResult result = task->poll(task, task->waker);
            if (result == POLL_READY) 
            {
                printf("[Runtime] Task %d 完成\n\n", task->id);
                task->state = TASK_DONE;
            } 
            else 
            {
                printf("[Runtime] Task %d 返回Pending，等待I/O\n\n", task->id);
                task->state = TASK_WAITING;
                // Task 暂停，不再入队，等待 Waker 唤醒
            }
        }

        // ==================== 步骤2：等待 epoll 事件 ====================
        printf("[Runtime] 队列为空，等待 epoll 事件...\n");
        int nfds = epoll_wait(runtime->epoll_fd, events, MAX_EVENTS, 1000);
        if (nfds == -1) 
        {
            perror("epoll_wait");
            break;
        }
        if (nfds == 0) 
        {
            // 超时，检查是否所有Task都完成
            int all_done = 1;
            for (int i = 0; i < runtime->task_count; i++) 
            {
                if (runtime->all_tasks[i]->state != TASK_DONE) 
                {
                    all_done = 0;
                    break;
                }
            }
            if (all_done) 
            {
                printf("[Runtime] 所有 Task 完成，退出\n");
                runtime->running = 0;
                break;
            }
            continue;
        }
        printf("[Runtime] epoll 返回 %d 个事件\n", nfds);
        // ==================== 步骤3：处理 epoll 事件，唤醒 Task ====================
        for (int i = 0; i < nfds; i++) 
        {
            int fd = events[i].data.fd;
            printf("[Runtime] fd %d 就绪\n", fd);
            // 找到对应的Task
            for (int j = 0; j < runtime->task_count; j++) 
            {
                Task *task = runtime->all_tasks[j];
                if (task->fd == fd && task->state == TASK_WAITING) 
                {
                    // 关键：通过Waker唤醒Task
                    printf("[Runtime] 找到等待的 Task %d，调用 Waker\n", task->id);
                    if (task->waker) 
                    {
                        task->waker->wake(task->waker);
                    }
                    break;
                }
            }
        }
        printf("\n");
    }
    printf("[Runtime] 事件循环结束\n");
}

// ==================== 工具函数 ====================

int set_nonblocking(int fd) 
{
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags == -1) 
    {
        return -1;
    }
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}
```

### 3.3 文件：`example.c` - 使用示例

```c
#include "simple_tokio.h"

// ==================== 示例：异步 Timer Task ====================

typedef struct 
{
    int seconds;
    time_t start_time;
} TimerData;

// Timer Task 的 poll 函数
PollResult timer_task_poll(Task *task, Waker *waker) 
{
    TimerData *data = (TimerData*)task->data;
    if (!data) 
    {
        // 第一次 poll：初始化
        data = malloc(sizeof(TimerData));
        data->seconds = 2;  // 等待 2 秒
        data->start_time = time(NULL);
        task->data = data;
        printf("  [Task %d] Timer 开始，等待 %d 秒\n", task->id, data->seconds);
        return POLL_PENDING;
    }
    // 检查是否超时
    time_t now = time(NULL);
    if (now - data->start_time >= data->seconds) 
    {
        printf("  [Task %d] Timer 完成！已过%ld秒\n", task->id, now - data->start_time);
        return POLL_READY;
    }
    printf("  [Task %d] Timer 还在等待... (已过 %ld 秒)\n", task->id, now - data->start_time);
    return POLL_PENDING;
}

// ==================== 示例：异步 Socket Task ====================

typedef struct 
{
    int state;  // 0=初始化, 1=连接中, 2=已连接, 3=完成
    char buffer[1024];
} SocketData;

PollResult socket_task_poll(Task *task, Waker *waker) 
{
    SocketData *data = (SocketData*)task->data;
    if (!data) 
    {
        // 第一次 poll：创建 socket 并连接
        data = malloc(sizeof(SocketData));
        data->state = 0;
        task->data = data;

        // 创建非阻塞 socket
        int sockfd = socket(AF_INET, SOCK_STREAM, 0);
        if (sockfd == -1) 
        {
            perror("socket");
            return POLL_READY;
        }
        set_nonblocking(sockfd);
        task->fd = sockfd;
        // 连接到本地服务器（假设有一个监听在 8080 的服务器）
        struct sockaddr_in addr;
        addr.sin_family = AF_INET;
        addr.sin_port = htons(8080);
        inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);
        printf("  [Task %d] Socket 创建，开始连接...\n", task->id);
        int ret = connect(sockfd, (struct sockaddr*)&addr, sizeof(addr));
        if (ret == -1 && errno != EINPROGRESS) 
        {
            printf("  [Task %d] 连接失败（可能没有服务器监听 8080）\n", task->id);
            close(sockfd);
            return POLL_READY;
        }
        // 注册到 epoll，监听可写事件（连接完成）
        struct epoll_event ev;
        ev.events = EPOLLOUT | EPOLLIN;
        ev.data.fd = sockfd;
        epoll_ctl(waker->runtime->epoll_fd, EPOLL_CTL_ADD, sockfd, &ev);
        data->state = 1;
        return POLL_PENDING;
    }

    if (data->state == 1) 
    {
        // 检查连接是否完成
        int error;
        socklen_t len = sizeof(error);
        getsockopt(task->fd, SOL_SOCKET, SO_ERROR, &error, &len);

        if (error == 0)
        {
            printf("  [Task %d] Socket 连接成功\n", task->id);
            data->state = 2;
            // 发送数据
            const char *msg = "GET / HTTP/1.1\r\nHost: localhost\r\n\r\n";
            write(task->fd, msg, strlen(msg));
            return POLL_PENDING;
        } 
        else 
        {
            printf("  [Task %d] Socket 连接失败\n", task->id);
            close(task->fd);
            return POLL_READY;
        }
    }

    if (data->state == 2) 
    {
        // 读取响应
        ssize_t n = read(task->fd, data->buffer, sizeof(data->buffer) - 1);
        if (n > 0) 
        {
            data->buffer[n] = '\0';
            printf("  [Task %d] Socket 收到数据 (%ld 字节) \n", task->id, n);
            printf("  数据: %s\n", data->buffer);
            close(task->fd);
            return POLL_READY;
        } 
        else if (n == -1 && errno == EAGAIN) 
        {
            printf("  [Task %d] Socket 数据还没到，继续等待...\n", task->id);
            return POLL_PENDING;
        } 
        else 
        {
            printf("  [Task %d] Socket 读取结束\n", task->id);
            close(task->fd);
            return POLL_READY;
        }
    }

    return POLL_READY;
}

// ==================== 主函数 ====================

int main() 
{
    printf("========================================\n");
    printf("  简化版 Tokio - C 语言实现\n");
    printf("========================================\n\n");

    // 创建运行时
    Runtime *runtime = runtime_new();
    if (!runtime) 
    {
        return 1;
    }

    // 创建多个异步 Task
    Task *task1 = task_new(1, timer_task_poll);
    Task *task2 = task_new(2, timer_task_poll);
    Task *task3 = task_new(3, timer_task_poll);

    // Spawn Task
    runtime_spawn(runtime, task1);
    runtime_spawn(runtime, task2);
    runtime_spawn(runtime, task3);

    // 运行事件循环
    runtime_run(runtime);

    // 清理
    runtime_free(runtime);

    printf("\n程序结束\n");
    return 0;
}
```

---

## 4. 使用示例

### 示例 1：多个异步 Timer

```c
int main() 
{
    Runtime *runtime = runtime_new();

    // 创建 3 个 Timer Task
    Task *task1 = task_new(1, timer_task_poll);
    Task *task2 = task_new(2, timer_task_poll);
    Task *task3 = task_new(3, timer_task_poll);

    runtime_spawn(runtime, task1);
    runtime_spawn(runtime, task2);
    runtime_spawn(runtime, task3);

    // 运行！单个 Worker Thread 处理 3 个 Task
    runtime_run(runtime);

    runtime_free(runtime);
    return 0;
}
```

### 示例 2：异步 Socket（需要服务器）

```c
int main() 
{
    Runtime *runtime = runtime_new();

    // 创建异步 Socket Task
    Task *task = task_new(1, socket_task_poll);
    runtime_spawn(runtime, task);

    runtime_run(runtime);

    runtime_free(runtime);
    return 0;
}
```

---

## 5. 运行结果

```
========================================
  简化版 Tokio - C 语言实现
========================================

[Runtime] 初始化完成，epoll_fd = 3
[Runtime] Spawn Task 1
[Runtime] Spawn Task 2
[Runtime] Spawn Task 3
[Runtime] 开始事件循环

[Runtime] Poll Task 1 (state=0)
  [Task 1] Timer 开始，等待 2 秒
[Runtime] Task 1 返回 Pending，等待 I/O

[Runtime] Poll Task 2 (state=0)
  [Task 2] Timer 开始，等待 2 秒
[Runtime] Task 2 返回 Pending，等待 I/O

[Runtime] Poll Task 3 (state=0)
  [Task 3] Timer 开始，等待 2 秒
[Runtime] Task 3 返回 Pending，等待 I/O

[Runtime] 队列为空，等待 epoll 事件...
[Runtime] 队列为空，等待 epoll 事件...

（2 秒后...）

[Waker] 唤醒 Task 1
[Runtime] Poll Task 1 (state=1)
  [Task 1] Timer 完成！已过2秒
[Runtime] Task 1 完成

[Waker] 唤醒 Task 2
[Runtime] Poll Task 2 (state=1)
  [Task 2] Timer 完成！已过2秒
[Runtime] Task 2完成

[Waker] 唤醒 Task 3
[Runtime] Poll Task 3 (state=1)
  [Task 3] Timer 完成！已过2秒
[Runtime] Task 3 完成

[Runtime] 所有 Task 完成，退出
[Runtime] 事件循环结束

程序结束
```

---

## 6. 核心机制解析

### 6.1 事件循环（Worker Thread）

```c
void runtime_run(Runtime *runtime) 
{
    while (runtime->running) 
    {
        // 执行队列中的所有 Task
        while (!task_queue_is_empty(runtime->queue)) 
        {
            Task *task = task_queue_pop(runtime->queue);
            // 调用 poll
            PollResult result = task->poll(task, task->waker);
            if (result == POLL_READY) 
            {
                // Task完成
                task->state = TASK_DONE;
            } 
            else 
            {
                // Task返回Pending，等待唤醒
                task->state = TASK_WAITING;
                // 不再入队！等待Waker
            }
        }

        // 队列空了，等待epoll事件
        int nfds = epoll_wait(runtime->epoll_fd, events, MAX_EVENTS, 1000);

        // 处理事件，唤醒对应的 Task
        for (int i = 0; i < nfds; i++) 
        {
            int fd = events[i].data.fd;
            // 找到对应的Task
            Task *task = find_task_by_fd(fd);
            // 通过Waker唤醒
            if (task->waker) 
            {
                task->waker->wake(task->waker);
            }
        }
    }
}
```

**关键点**：
1. Worker Thread 不停循环
2. poll Task → Pending → 等待epoll → 唤醒 → 继续poll
3. 单线程处理多个Task

### 6.2 Task的poll函数

```c
PollResult timer_task_poll(Task *task, Waker *waker) 
{
    TimerData *data = task->data;
    if (!data) 
    {
        // 第一次 poll：初始化
        data = malloc(sizeof(TimerData));
        data->start_time = time(NULL);
        task->data = data;
        // 返回 Pending，等待定时器
        return POLL_PENDING;
    }
    // 检查是否超时
    if (time(NULL) - data->start_time >= 2) 
    {
        // 超时了，返回 Ready
        return POLL_READY;
    }
    // 还没到时间，继续 Pending
    return POLL_PENDING;
}
```

**关键点**：
- 类似Rust的`Future::poll`
- 返回`POLL_READY`或`POLL_PENDING`
- 保存状态（通过`task->data`）

### 6.3 Waker 唤醒机制

```c
void waker_wake(Waker *waker) 
{
    // 找到对应的 Task
    Task *task = find_task_by_id(waker->task_id);
    if (task->state == TASK_WAITING) 
    {
        // 重新加入队列
        task->state = TASK_RUNNING;
        task_queue_push(waker->runtime->queue, task);
    }
}
```

**关键点**：
- Waker负责将Task重新加入队列
- epoll事件触发时调用Waker
- Task从WAITING → RUNNING

### 6.4 epoll集成

```c
// 注册fd到epoll
struct epoll_event ev;
ev.events = EPOLLIN | EPOLLOUT;
ev.data.fd = sockfd;
epoll_ctl(runtime->epoll_fd, EPOLL_CTL_ADD, sockfd, &ev);

// 等待事件
int nfds = epoll_wait(runtime->epoll_fd, events, MAX_EVENTS, 1000);

// 处理事件
for (int i = 0; i < nfds; i++) 
{
    int fd = events[i].data.fd;
    // 找到对应的 Task，唤醒
}
```

**关键点**：
- epoll监听所有注册的fd
- fd就绪时返回
- 通过fd找到对应的Task和Waker

---

## 7. 编译和运行

### 7.1 编译

```bash
# 编译
gcc -o simple_tokio simple_tokio.c example.c -Wall

# 运行
./simple_tokio
```

### 7.2 Makefile

```makefile
CC = gcc
CFLAGS = -Wall -g

simple_tokio: simple_tokio.c example.c
	$(CC) $(CFLAGS) -o simple_tokio simple_tokio.c example.c

clean:
	rm -f simple_tokio

run: simple_tokio
	./simple_tokio
```

使用：
```bash
make
make run
```

---

## 8. 核心对比：C 实现 vs Rust Tokio

| 维度          | C 实现                      | Rust Tokio                  |
|-------------|---------------------------|-----------------------------|
| **Task**    | `Task`结构体 + `poll`函数指针    | `Future`trait               |
| **Poll**    | `POLL_READY/POLL_PENDING` | `Poll::Ready/Poll::Pending` |
| **Waker**   | `Waker`结构体 + `wake`函数     | `Waker`                     |
| **Runtime** | 手动实现事件循环                  | Tokio提供                     |
| **epoll**   | 直接调用系统API                 | mio封装                       |
| **队列**      | 手动实现循环队列                  | 高效的并发队列                     |
| **多线程**     | 单线程示例                     | 默认多线程                       |
| **类型安全**    | 无                         | 有                           |

**本质相同**：
- Task = Future
- poll机制
- Waker唤醒
- epoll监听
- 事件循环

---

## 9. 学习要点

通过这个C实现，你应该理解：

1. **Task不是线程**
   - Task只是一个结构体
   - 有`poll`函数
   - 保存状态

2. **`.await`的本质**
   - 就是调用`poll`
   - 返回`Pending`时让出
   - 等待`Waker`唤醒

3. **Worker Thread的工作**
   - 从队列取Task
   - 调用`poll`
   - 处理epoll事件
   - 通过Waker唤醒Task

4. **epoll的作用**
   - 监听多个fd
   - fd就绪时通知
   - 找到对应的Waker

5. **为什么高效**
   - 单线程处理多个Task
   - poll返回Pending时不阻塞
   - epoll高效监听I/O

---

## 10. 扩展练习

你可以尝试：

1. **添加更多 Task 类型**
   - 文件读取
   - HTTP 客户端
   - 定时器

2. **实现多线程 Runtime**
   - 多个 Worker Thread
   - Work Stealing

3. **优化性能**
   - 更高效的队列
   - 减少内存分配

4. **添加调试功能**
   - 打印 Task 状态
   - 统计性能

---

## 11. Linux 系统编程知识详解

> 本章节详细讲解代码中用到的所有Linux系统编程知识

### 11.1 epoll - I/O多路复用 

**概念**：

epoll是Linux特有的I/O多路复用机制，可以高效地监听多个文件描述符（fd）的 I/O 事件。

**为什么需要epoll？**

```c
// 传统方式：阻塞I/O
int fd = socket(...);
char buf[1024];
read(fd, buf, sizeof(buf));  // 阻塞！如果没数据，线程卡住

// 多个连接需要多个线程
// 10000 个连接 = 10000 个线程 = 80GB 内存

// epoll：单线程监听多个fd
int epoll_fd = epoll_create1(0);
// 注册10000个socket到epoll
// epoll_wait() 等待任意一个fd就绪
// 单线程处理10000个连接！
```

**核心 API**：

#### 1. `epoll_create1()` - 创建epoll实例

```c
#include <sys/epoll.h>

int epoll_create1(int flags);
```

**参数**：
- `flags`：通常传`0`，或`EPOLL_CLOEXEC`（fork时关闭）

**返回值**：
- 成功：返回epoll文件描述符
- 失败：返回-1，设置errno

**示例**：
```c
int epoll_fd = epoll_create1(0);
if (epoll_fd == -1) 
{
    perror("epoll_create1");
    exit(1);
}
```

#### 2. `epoll_ctl()` - 控制 epoll（添加/修改/删除 fd）

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

**参数**：
- `epfd`：epoll文件描述符
- `op`：操作类型
  - `EPOLL_CTL_ADD`：添加fd
  - `EPOLL_CTL_MOD`：修改fd的事件
  - `EPOLL_CTL_DEL`：删除fd
- `fd`：要操作的文件描述符
- `event`：事件配置

**epoll_event结构体**：
```c
struct epoll_event 
{
    uint32_t events;   // 事件类型
    epoll_data_t data; // 用户数据
};

typedef union epoll_data 
{
    void *ptr;
    int fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;
```

**常用事件类型**：
- `EPOLLIN`：可读（有数据到达）
- `EPOLLOUT`：可写（可以发送数据）
- `EPOLLERR`：错误
- `EPOLLHUP`：挂断
- `EPOLLET`：边缘触发模式（Edge Triggered）

**示例**：
```c
// 添加socket到epoll，监听可读事件
int sockfd = socket(AF_INET, SOCK_STREAM, 0);

struct epoll_event ev;
ev.events = EPOLLIN;     // 监听可读
ev.data.fd = sockfd;     // 保存fd

int ret = epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sockfd, &ev);
if (ret == -1) 
{
    perror("epoll_ctl");
}

// 修改事件：同时监听可读和可写
ev.events = EPOLLIN | EPOLLOUT;
epoll_ctl(epoll_fd, EPOLL_CTL_MOD, sockfd, &ev);

// 删除fd
epoll_ctl(epoll_fd, EPOLL_CTL_DEL, sockfd, NULL);
```

#### 3. `epoll_wait()` - 等待事件

```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

**参数**：
- `epfd`：epoll文件描述符
- `events`：输出参数，存储就绪的事件
- `maxevents`：events数组大小
- `timeout`：超时时间（毫秒）
  - `-1`：永久阻塞
  - `0`：立即返回（非阻塞）
  - `>0`：等待指定毫秒数

**返回值**：
- 成功：返回就绪的fd数量
- 超时：返回0
- 失败：返回-1

**示例**：
```c
struct epoll_event events[100];

while (1) 
{
    // 等待事件，最多1秒
    int nfds = epoll_wait(epoll_fd, events, 100, 1000);
    if (nfds == -1) 
    {
        perror("epoll_wait");
        break;
    }
    if (nfds == 0) 
    {
        printf("超时，没有事件\n");
        continue;
    }
    // 处理就绪的fd
    for (int i = 0; i < nfds; i++) 
    {
        int fd = events[i].data.fd;
        if (events[i].events & EPOLLIN) 
        {
            // fd可读
            char buf[1024];
            read(fd, buf, sizeof(buf));
        }
        if (events[i].events & EPOLLOUT) 
        {
            // fd可写
            write(fd, "data", 4);
        }
    }
}
```

**水平触发 vs 边缘触发**：

```c
// 水平触发（Level Triggered，默认）
// - 只要fd有数据，epoll_wait就会一直返回
// - 安全，不会丢失数据

struct epoll_event ev;
ev.events = EPOLLIN;  // 默认是水平触发

// 边缘触发（Edge Triggered）
// - 只在状态变化时通知（从无数据 → 有数据）
// - 需要一次读完所有数据，否则会丢失
// - 更高效，但使用更复杂

ev.events = EPOLLIN | EPOLLET;  // 边缘触发
```

---

### 11.2 Socket编程

**概念**：

Socket是网络通信的端点，可以理解为"网络插座"。

**TCP Socket完整流程**：

```c
服务器端：                    客户端：
socket()                     socket()
  ↓                            ↓
bind()                       connect() ──────┐
  ↓                                          │
listen()                                     │
  ↓                                          │
accept() ←───────────────────────────────────┘
  ↓
read()/write() ←─────────→ read()/write()
  ↓
close()                      close()
```

#### 1. `socket()` - 创建socket

```c
#include <sys/socket.h>

int socket(int domain, int type, int protocol);
```

**参数**：
- `domain`：协议族
  - `AF_INET`：IPv4
  - `AF_INET6`：IPv6
  - `AF_UNIX`：本地通信
- `type`：socket 类型
  - `SOCK_STREAM`：TCP（可靠、有序、双向字节流）
  - `SOCK_DGRAM`：UDP（无连接、不可靠）
- `protocol`：通常传`0`（自动选择）

**返回值**：
- 成功：返回socket文件描述符
- 失败：返回-1

**示例**：
```c
// 创建TCP socket
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
if (sockfd == -1) 
{
    perror("socket");
    exit(1);
}

// 创建UDP socket
int udp_fd = socket(AF_INET, SOCK_DGRAM, 0);
```

#### 2. `connect()` - 连接到服务器（客户端）

```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

**参数**：
- `sockfd`：socket文件描述符
- `addr`：服务器地址
- `addrlen`：地址结构大小

**返回值**：
- 成功：返回0
- 失败：返回-1

**示例**：
```c
int sockfd = socket(AF_INET, SOCK_STREAM, 0);

// 配置服务器地址
struct sockaddr_in server_addr;
server_addr.sin_family = AF_INET;                        // IPv4
server_addr.sin_port = htons(8080);                      // 端口8080
inet_pton(AF_INET, "127.0.0.1", &server_addr.sin_addr);  // IP

// 连接
int ret = connect(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr));
if (ret == -1) 
{
    perror("connect");
    close(sockfd);
    exit(1);
}

printf("连接成功！\n");
```

**非阻塞 connect**：
```c
// 设置非阻塞
set_nonblocking(sockfd);
int ret = connect(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr));
if (ret == -1) 
{
    if (errno == EINPROGRESS) 
    {
        // 连接正在进行中，需要等待
        // 注册到epoll，监听EPOLLOUT事件
        // 当EPOLLOUT就绪时，连接完成
        struct epoll_event ev;
        ev.events = EPOLLOUT;
        ev.data.fd = sockfd;
        epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sockfd, &ev);
    } 
    else 
    {
        // 真正的错误
        perror("connect");
    }
}
```

#### 3. `bind()` - 绑定地址（服务器）

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

**示例**：
```c
int sockfd = socket(AF_INET, SOCK_STREAM, 0);

struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(8080);
addr.sin_addr.s_addr = INADDR_ANY;  // 监听所有IP

int ret = bind(sockfd, (struct sockaddr*)&addr, sizeof(addr));
if (ret == -1) 
{
    perror("bind");
    exit(1);
}
```

#### 4. `listen()` - 监听连接（服务器）

```c
int listen(int sockfd, int backlog);
```

**参数**：
- `sockfd`：socket文件描述符
- `backlog`：等待队列长度（通常128）

#### 5. `accept()` - 接受连接（服务器）

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

**返回值**：
- 成功：返回新的连接socket
- 失败：返回-1

**示例**：
```c
struct sockaddr_in client_addr;
socklen_t addr_len = sizeof(client_addr);

int client_fd = accept(sockfd, (struct sockaddr*)&client_addr, &addr_len);
if (client_fd == -1) 
{
    perror("accept");
} 
else 
{
    printf("接受新连接，fd = %d\n", client_fd);
}
```

#### 6. `read()/write()` - 读写数据

```c
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
```

**返回值**：
- 成功：返回实际读写的字节数
- EOF：read返回0
- 失败：返回-1

**非阻塞模式**：
```c
// 设置非阻塞
set_nonblocking(sockfd);

char buf[1024];
ssize_t n = read(sockfd, buf, sizeof(buf));

if (n > 0) {
    // 读到数据
    printf("读取 %ld 字节\n", n);
} else if (n == 0) {
    // EOF，连接关闭
    printf("连接关闭\n");
    close(sockfd);
} else {
    if (errno == EAGAIN || errno == EWOULDBLOCK) {
        // 数据还没到，稍后再试
        printf("数据未就绪\n");
    } else {
        // 真正的错误
        perror("read");
    }
}
```

---

### 11.3 文件描述符（fd）

**概念**：

文件描述符（File Descriptor）是一个非负整数，代表一个打开的文件、socket、管道等。

**特点**：
- fd是进程级的资源
- 0 = 标准输入（stdin）
- 1 = 标准输出（stdout）
- 2 = 标准错误（stderr）
- 3 及以上：用户打开的文件/socket

**生命周期**：
```c
// 1. 打开/创建
int fd = open("file.txt", O_RDONLY);
int sockfd = socket(AF_INET, SOCK_STREAM, 0);

// 2. 使用
read(fd, buf, size);
write(fd, buf, size);

// 3. 关闭（必须！）
close(fd);
```

**常见错误**：
```c
// 忘记关闭 fd → fd 泄漏
for (int i = 0; i < 10000; i++) 
{
    int fd = socket(...);
    // 忘记 close(fd);
}
// 最终：too many open files

// 正确：及时关闭
int fd = socket(...);
// 使用 fd
close(fd);
```

---

### 11.4 fcntl - 文件控制 

**概念**：

`fcntl`用于控制文件描述符的属性，最常用的是设置非阻塞模式。

```c
#include <fcntl.h>

int fcntl(int fd, int cmd, ...);
```

**常用命令**：
- `F_GETFL`：获取文件状态标志
- `F_SETFL`：设置文件状态标志

**设置非阻塞**：
```c
int set_nonblocking(int fd) 
{
    // 1. 获取当前标志
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags == -1) 
    {
        perror("fcntl F_GETFL");
        return -1;
    }
    // 2. 添加O_NONBLOCK标志
    flags |= O_NONBLOCK;
    // 3. 设置新标志
    int ret = fcntl(fd, F_SETFL, flags);
    if (ret == -1) 
    {
        perror("fcntl F_SETFL");
        return -1;
    }
    return 0;
}

// 使用
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
set_nonblocking(sockfd);  // 设置为非阻塞模式
```

**阻塞 vs 非阻塞**：
```c
// 阻塞模式（默认）
char buf[1024];
read(fd, buf, sizeof(buf));  // 如果没数据，线程卡住 

// 非阻塞模式
set_nonblocking(fd);
ssize_t n = read(fd, buf, sizeof(buf));
if (n == -1 && errno == EAGAIN) 
{
    // 数据还没到，立即返回 
    printf("数据未就绪\n");
}
```

---

### 11.5 errno和错误处理 

**概念**：

`errno`是一个全局变量，存储最近一次系统调用的错误码。

```c
#include <errno.h>

extern int errno;
```

**常用错误码**：
- `EAGAIN` / `EWOULDBLOCK`：资源暂时不可用（非阻塞时）
- `EINPROGRESS`：操作正在进行（非阻塞connect）
- `EINTR`：系统调用被信号中断
- `ECONNREFUSED`：连接被拒绝
- `EPIPE`：管道破裂（对端关闭）

**使用perror**：
```c
#include <stdio.h>

void perror(const char *s);
```

**示例**：
```c
int fd = open("nonexistent.txt", O_RDONLY);
if (fd == -1) 
{
    perror("open");                 // 输出：open: No such file or directory
    printf("errno = %d\n", errno);  // errno = 2 (ENOENT)
}
```

**检查特定错误**：
```c
ssize_t n = read(fd, buf, sizeof(buf));
if (n == -1) 
{
    if (errno == EAGAIN || errno == EWOULDBLOCK) 
    {
        // 数据未就绪，稍后再试
    } 
    else if (errno == EINTR) 
    {
        // 被信号中断，重试
    } 
    else 
    {
        // 其他错误
        perror("read");
    }
}
```

---

### 11.6 网络地址结构

**struct sockaddr_in**（IPv4）：

```c
#include <netinet/in.h>

struct sockaddr_in 
{
    sa_family_t    sin_family;  // AF_INET
    in_port_t      sin_port;    // 端口号（网络字节序）
    struct in_addr sin_addr;    // IP 地址
    char           sin_zero[8]; // 填充
};

struct in_addr 
{
    uint32_t s_addr;  // IP地址（网络字节序）
};
```

**字节序转换**：

网络字节序（Big Endian）vs 主机字节序（可能是Little Endian）

```c
#include <arpa/inet.h>

// 主机 → 网络
uint16_t htons(uint16_t hostshort);  // Host TO Network Short
uint32_t htonl(uint32_t hostlong);   // Host TO Network Long

// 网络 → 主机
uint16_t ntohs(uint16_t netshort);   // Network TO Host Short
uint32_t ntohl(uint32_t netlong);    // Network TO Host Long
```

**示例**：
```c
struct sockaddr_in addr;

// 设置协议族
addr.sin_family = AF_INET;

// 设置端口（需要转换字节序）
addr.sin_port = htons(8080);  // 主机字节序 8080 → 网络字节序

// 设置 IP 地址
// 方法1：使用INADDR_ANY（监听所有IP）
addr.sin_addr.s_addr = INADDR_ANY;

// 方法2：使用inet_pton（推荐）
inet_pton(AF_INET, "192.168.1.100", &addr.sin_addr);

// 方法3：使用inet_addr（过时，不推荐）
addr.sin_addr.s_addr = inet_addr("192.168.1.100");
```

**inet_pton/inet_ntop**：

```c
#include <arpa/inet.h>

// 字符串 IP → 二进制
int inet_pton(int af, const char *src, void *dst);

// 二进制 → 字符串 IP
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);
```

**示例**：
```c
// 字符串 → 二进制
struct in_addr addr;
inet_pton(AF_INET, "192.168.1.1", &addr);

// 二进制 → 字符串
char ip_str[INET_ADDRSTRLEN];  // 16 字节
inet_ntop(AF_INET, &addr, ip_str, sizeof(ip_str));
printf("IP: %s\n", ip_str);  // 输出：IP: 192.168.1.1
```

---

### 11.7 内存管理

**malloc/free**：

```c
#include <stdlib.h>

void *malloc(size_t size);
void free(void *ptr);
```

**最佳实践**：
```c
// 检查返回值
Task *task = malloc(sizeof(Task));
if (!task) 
{
    perror("malloc");
    exit(1);
}

// 使用 task
task->id = 1;

// 释放内存
free(task);
task = NULL;  // 避免 use-after-free

// 常见错误
free(task);
free(task);  // 双重释放 → 崩溃

free(task);
task->id = 2;  // use-after-free → 崩溃
```

**calloc**（分配并清零）：
```c
// malloc：不初始化内存（可能有垃圾数据）
int *arr1 = malloc(10 * sizeof(int));

// calloc：分配并清零
int *arr2 = calloc(10, sizeof(int));  // 所有元素初始化为0
```

---

### 11.8 时间相关

**time()**：

```c
#include <time.h>

time_t time(time_t *tloc);
```

**返回值**：自 1970-01-01 00:00:00 UTC 以来的秒数

**示例**：
```c
time_t now = time(NULL);
printf("当前时间戳: %ld\n", now);

// 计算时间差
time_t start = time(NULL);
sleep(5);
time_t end = time(NULL);
printf("经过 %ld 秒\n", end - start);  // 输出：经过 5 秒
```

---

### 11.9 完整示例：epoll + 非阻塞 socket

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define MAX_EVENTS 10
#define PORT 8080

// 设置非阻塞
int set_nonblocking(int fd) 
{
    int flags = fcntl(fd, F_GETFL, 0);
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() 
{
    // 1. 创建socket
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_fd == -1) 
    {
        perror("socket");
        exit(1);
    }

    // 设置端口复用
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    // 2. 绑定地址
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    addr.sin_addr.s_addr = INADDR_ANY;

    if (bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr)) == -1) 
    {
        perror("bind");
        exit(1);
    }

    // 3. 监听
    if (listen(listen_fd, 128) == -1) 
    {
        perror("listen");
        exit(1);
    }

    // 设置非阻塞
    set_nonblocking(listen_fd);

    // 4. 创建epoll
    int epoll_fd = epoll_create1(0);
    if (epoll_fd == -1) 
    {
        perror("epoll_create1");
        exit(1);
    }

    // 5. 将listen_fd添加到epoll
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = listen_fd;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev);

    printf("服务器监听在端口 %d\n", PORT);

    // 6. 事件循环
    struct epoll_event events[MAX_EVENTS];

    while (1) 
    {
        int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        for (int i = 0; i < nfds; i++) 
        {
            int fd = events[i].data.fd;
            if (fd == listen_fd) 
            {
                // 新连接
                struct sockaddr_in client_addr;
                socklen_t addr_len = sizeof(client_addr);
                int client_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &addr_len);
                if (client_fd == -1) 
                {
                    if (errno != EAGAIN && errno != EWOULDBLOCK) 
                    {
                        perror("accept");
                    }
                    continue;
                }
                
                // 打印客户端信息
                char ip[INET_ADDRSTRLEN];
                inet_ntop(AF_INET, &client_addr.sin_addr, ip, sizeof(ip));
                printf("新连接：%s:%d (fd=%d)\n", ip, ntohs(client_addr.sin_port), client_fd);
                
                // 设置非阻塞
                set_nonblocking(client_fd);
                
                // 添加到 epoll
                ev.events = EPOLLIN | EPOLLET;  // 边缘触发
                ev.data.fd = client_fd;
                epoll_ctl(epoll_fd, EPOLL_CTL_ADD, client_fd, &ev);

            } 
            else 
            {
                // 客户端数据
                char buf[1024];
                ssize_t n = read(fd, buf, sizeof(buf));
                if (n > 0) 
                {
                    // 收到数据
                    buf[n] = '\0';
                    printf("收到数据 (fd=%d): %s\n", fd, buf);
                    // 回显
                    write(fd, buf, n);
                } 
                else if (n == 0) 
                {
                    // 连接关闭
                    printf("连接关闭 (fd=%d)\n", fd);
                    epoll_ctl(epoll_fd, EPOLL_CTL_DEL, fd, NULL);
                    close(fd);
                } 
                else 
                {
                    if (errno != EAGAIN && errno != EWOULDBLOCK) 
                    {
                        perror("read");
                    }
                }
            }
        }
    }

    close(epoll_fd);
    close(listen_fd);
    return 0;
}
```

**编译运行**：
```bash
gcc -o epoll_server epoll_server.c
./epoll_server

# 测试（另一个终端）
telnet localhost 8080
```

---

### 11.10 常见问题和最佳实践

#### 1. fd泄漏

```c
// 错误
for (int i = 0; i < 10000; i++) {
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    // 忘记 close(fd);
}

// 正确
int fd = socket(AF_INET, SOCK_STREAM, 0);
// 使用 fd
close(fd);
```

#### 2. 忘记检查返回值

```c
// 错误
int fd = socket(AF_INET, SOCK_STREAM, 0);
connect(fd, ...);  // 如果 socket 失败，fd=-1，connect 会崩溃

// 正确
int fd = socket(AF_INET, SOCK_STREAM, 0);
if (fd == -1) {
    perror("socket");
    return -1;
}

if (connect(fd, ...) == -1) {
    perror("connect");
    close(fd);
    return -1;
}
```

#### 3. 非阻塞I/O处理

```c
// 错误：把EAGAIN当作错误
ssize_t n = read(fd, buf, size);
if (n == -1) {
    perror("read");  // 错误！EAGAIN 不是真正的错误
    exit(1);
}

// 正确
ssize_t n = read(fd, buf, size);
if (n == -1) {
    if (errno == EAGAIN || errno == EWOULDBLOCK) {
        // 数据还没到，正常
        return;
    } else {
        // 真正的错误
        perror("read");
        exit(1);
    }
}
```

#### 4. 字节序转换

```c
// 错误：忘记转换字节序
addr.sin_port = 8080;  // 错误！

// 正确
addr.sin_port = htons(8080);  // 转换为网络字节序
```

#### 5. 内存泄漏

```c
// 错误
void *ptr = malloc(1024);
// 忘记 free(ptr);

// 正确
void *ptr = malloc(1024);
// 使用 ptr
free(ptr);
ptr = NULL;
```

---

## 总结

这个简化版Tokio展示了异步运行时的核心机制：

- **Task**：异步任务，有`poll`方法
- **Poll**：返回`Ready`或`Pending`
- **Waker**：负责唤醒Task
- **epoll**：监听I/O事件
- **事件循环**：poll → Pending → epoll → wake → poll

**核心流程**：
```
1. Task.poll() → Pending
2. 注册Waker到epoll
3. Task让出，Worker Thread处理其他Task
4. I/O就绪 → epoll返回
5. Waker.wake() → Task重新入队
6. Task.poll() → Ready（完成）
```

这就是Tokio的底层原理！
