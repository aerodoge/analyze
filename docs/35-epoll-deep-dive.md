# epoll深度解析：原理与内核实现

> 深入理解Linux epoll的设计思想、内核实现和性能优化

---

## 目录

1. [epoll 基础概念](#1-epoll-基础概念)
2. [epoll API 详解](#2-epoll-api-详解)
   - 2.3.1 [EPOLLIN 和 EPOLLOUT 详解](#231-epollin-和-epollout-详解-)
3. [I/O 多路复用对比](#3-io-多路复用对比)
4. [epoll 内核数据结构](#4-epoll-内核数据结构)
5. [epoll_create 内核实现](#5-epoll_create-内核实现)
6. [epoll_ctl 内核实现](#6-epoll_ctl-内核实现)
7. [epoll_wait 内核实现](#7-epoll_wait-内核实现)
8. [事件通知机制](#8-事件通知机制)
9. [ET vs LT 模式](#9-et-vs-lt-模式)
10. [性能分析](#10-性能分析)
11. [完整代码示例](#11-完整代码示例)
12. [最佳实践](#12-最佳实践)
13. [内核事件检测机制](#13-内核事件检测机制epollin-和-epollout-的底层原理)

---

## 1. epoll基础概念

### 1.1 什么是epoll？

**epoll** (event poll) 是Linux内核提供的一种**高效的I/O事件通知机制**，用于监控多个文件描述符上的I/O事件。

**核心特点**：

- **高性能**：O(1) 时间复杂度处理就绪事件
- **可扩展**：支持监控数十万个文件描述符
- **事件驱动**：只返回就绪的文件描述符
- **内核管理**：避免重复拷贝文件描述符集合

### 1.2 为什么需要epoll？

**问题场景**：C10K问题 (Concurrent 10,000 connections)

```
场景：Web服务器需要同时处理10,000个并发连接

传统方案的问题：
┌─────────────────────────────────────────────────┐
│ 方案 1: 为每个连接创建一个线程                      │
├─────────────────────────────────────────────────┤
│ 10,000 个线程 × 8MB 栈 = 80GB 内存                │
│ 线程上下文切换开销巨大                              │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ 方案 2: 使用select/poll                          │
├─────────────────────────────────────────────────┤
│ 每次调用需要遍历所有10,000个fd                      │
│ 每次调用需要从用户空间拷贝fd集合到内核                │
│ O(n) 时间复杂度，n=10,000太慢                      │
└─────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────┐
│ 方案 3: 使用epoll                                │
├─────────────────────────────────────────────────┤
│ 在内核中维护fd集合，无需重复拷贝                     │
│ 只返回就绪的fd，O(1) 时间复杂度                     │
│ 支持海量并发连接                                   │
└─────────────────────────────────────────────────┘
```

### 1.3 epoll的设计思想

**核心思想**：**注册-等待-通知**

```
传统select/poll模型：
  while (true) {
      遍历所有fd，检查是否就绪  ← O(n) 开销，每次都要做
      处理就绪的fd
  }

epoll模型：
  1. 一次性注册所有fd到内核（epoll_ctl）
  2. 内核维护fd集合，当fd就绪时，内核主动通知
  3. 应用只需等待通知（epoll_wait），O(1) 获取就绪fd
```

**关键优化**：

1. **内核维护fd集合**：避免每次从用户空间拷贝
2. **事件驱动通知**：fd就绪时，内核主动添加到就绪列表
3. **只返回就绪fd**：不需要遍历所有fd

---

## 2. epoll API详解

### 2.1 三个核心系统调用

```c
#include <sys/epoll.h>

// 1. 创建epoll实例
int epoll_create1(int flags);

// 2. 管理epoll监控的文件描述符
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

// 3. 等待I/O事件
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

### 2.2 epoll_create1

**功能**：创建一个epoll实例

```c
int epoll_create1(int flags);
```

**参数**：

- `flags`:
    - `0`: 默认行为
    - `EPOLL_CLOEXEC`: 设置close-on-exec标志

**返回值**：

- 成功：返回epoll文件描述符 (epfd)
- 失败：返回-1，设置errno

**示例**：

```c
int epfd = epoll_create1(0);
if (epfd == -1) {
    perror("epoll_create1");
    exit(EXIT_FAILURE);
}
```

**内核做了什么**：

```
1. 分配eventpoll结构体（核心数据结构）内存
2. 初始化红黑树（用于存储监控的fd）
3. 初始化就绪列表（用于存储就绪的fd）
4. 创建wait队列（用于阻塞等待的进程）
5. 返回epoll fd
```

### 2.3 epoll_ctl

**功能**：添加、修改或删除epoll监控的文件描述符

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

**参数**：

- `epfd`: epoll实例的文件描述符
- `op`: 操作类型
    - `EPOLL_CTL_ADD`: 添加fd到epoll
    - `EPOLL_CTL_MOD`: 修改fd的监控事件
    - `EPOLL_CTL_DEL`: 从epoll删除fd
- `fd`: 要监控的文件描述符
- `event`: 事件结构体

**epoll_event结构**：

```c
struct epoll_event {
    uint32_t     events;  // 监控的事件类型
    epoll_data_t data;    // 用户数据
};

typedef union epoll_data {
    void    *ptr;
    int      fd;
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;
```

**事件类型 (events)**：

```c
EPOLLIN      // 可读事件（有数据可读）
EPOLLOUT     // 可写事件（可以写数据）
EPOLLERR     // 错误事件
EPOLLHUP     // 挂断事件（对方关闭连接）
EPOLLET      // 边缘触发模式（Edge Triggered）
EPOLLONESHOT // 一次性事件（触发一次后自动删除）
EPOLLRDHUP   // 对方关闭连接或关闭写端
```

**示例**：

```c
struct epoll_event ev;
ev.events = EPOLLIN | EPOLLET;  // 监控可读事件，边缘触发
ev.data.fd = sockfd;

if (epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev) == -1) {
    perror("epoll_ctl");
    exit(EXIT_FAILURE);
}
```


---

### 2.3.1 EPOLLIN 和 EPOLLOUT 详解 🔍


#### 1. EPOLLIN - 可读事件

##### 1.1 定义

**EPOLLIN**：文件描述符变为可读状态，即有数据可以读取。

##### 1.2 触发条件

```
对于TCP socket:
  ┌────────────────────────────────────────────────────────┐
  │ 1. 接收缓冲区有数据                                       │
  │    └─ 有新数据到达 → EPOLLIN触发                          │
  │                                                        │
  │ 2. 对方关闭连接（收到FIN包）                               │
  │    └─ read() 会返回0 → EPOLLIN触发                       │
  │                                                        │
  │ 3. 监听socket有新连接到达                                 │
  │    └─ accept队列非空 → EPOLLIN触发                       │
  │                                                        │
  │ 4. socket发生错误                                       │
  │    └─ EPOLLIN + EPOLLERR同时触发                        │
  └────────────────────────────────────────────────────────┘

对于UDP socket:
  ┌────────────────────────────────────────────────────────┐
  │ 接收缓冲区有完整的数据报                                   │
  │ └─ recvfrom() 可以读取数据 → EPOLLIN触发                  │
  └────────────────────────────────────────────────────────┘

对于管道 (pipe):
  ┌────────────────────────────────────────────────────────┐
  │ 管道中有数据                                             │
  │ └─ read() 不会阻塞 → EPOLLIN触发                         │
  └────────────────────────────────────────────────────────┘

对于普通文件:
  ┌────────────────────────────────────────────────────────┐
  │ 总是可读（除非到达 EOF）                                  │
  │ └─ epoll 对普通文件意义不大️                               │
  └────────────────────────────────────────────────────────┘
```

##### 1.3 底层原理

**TCP Socket接收数据的完整流程**：

```
1. 网络数据包到达
   ┌──────────────────────────────────────┐
   │ 网卡接收数据包                         │
   │ └─ DMA拷贝到内核内存                   │
   └──────────────────────────────────────┘
              │ 硬件中断
              ▼
   ┌──────────────────────────────────────┐
   │ 网卡驱动处理中断                        │
   │ └─ 调用netif_rx()                     │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ 软中断处理（NET_RX_SOFTIRQ）           │
   │ └─ __netif_receive_skb()             │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ IP层处理                              │
   │ ├─ 检查IP头                           │
   │ ├─ 校验校验和                          │
   │ └─ 路由查找                           │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ TCP层处理                             │
   │ ├─ 解析TCP头                          │
   │ ├─ 校验序列号                          │
   │ ├─ 更新接收窗口                        │
   │ └─ 发送ACK                            │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────────────────────┐
   │ 数据拷贝到socket接收缓冲区                              │
   │                                                      │
   │   struct sock {                                      │
   │       struct sk_buff_head sk_receive_queue;  ← 数据   │
   │       int sk_rcvbuf;         ← 总大小                 │
   │       atomic_t sk_rmem_alloc; ← 已用                  │
   │   }                                                  │
   │                                                      │
   │   tcp_rcv_established()                              │
   │   └─ tcp_queue_rcv()                                 │
   │      └─ __skb_queue_tail(&sk->sk_receive_queue, skb) │
   └──────────────────────────────────────────────────────┘
              │
              │ 缓冲区状态变化：空 → 非空
              ▼
   ┌──────────────────────────────────────┐
   │ sock_def_readable()                  │
   │ └─ wake_up_interruptible_sync_poll(  │
   │        &sk->sk_wq,                   │
   │        POLLIN | POLLRDNORM)          │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ __wake_up_common()                   │
   │ └─ 遍历 sk->sk_wq 上的所有等待项        │
   │    └─ 调用每个等待项的回调函数           │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ ep_poll_callback()                   │
   │ ├─ spin_lock(&ep->lock)              │
   │ ├─ if (!ep_is_linked(&epi->rdllink)) │
   │ │      list_add_tail(&epi->rdllink,  │
   │ │                    &ep->rdllist)   │
   │ ├─ if (waitqueue_active(&ep->wq))    │
   │ │      wake_up(&ep->wq)              │
   │ └─ spin_unlock(&ep->lock)            │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ epoll_wait() 返回                     │
   │ └─ events[i].events = EPOLLIN        │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ 用户程序处理                           │
   │ └─ read(fd, buf, size)               │
   │    └─ tcp_recvmsg()                  │
   │       └─ skb_copy_datagram_iter()    │
   │          └─ 从sk_receive_queue拷贝    │
   └──────────────────────────────────────┘
```

##### 1.4 代码示例

```c
// 示例 1：处理连接 socket 的可读事件
if (events[i].events & EPOLLIN) {
    int fd = events[i].data.fd;

    if (fd == listen_fd) {
        // 监听 socket：有新连接到达
        struct sockaddr_in client_addr;
        socklen_t len = sizeof(client_addr);
        int conn_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &len);

        if (conn_fd != -1) {
            printf("New connection: %d\n", conn_fd);
            // 将新连接添加到 epoll
            struct epoll_event ev;
            ev.events = EPOLLIN;
            ev.data.fd = conn_fd;
            epoll_ctl(epfd, EPOLL_CTL_ADD, conn_fd, &ev);
        }
    } else {
        // 连接 socket：有数据到达
        char buf[4096];
        ssize_t n = read(fd, buf, sizeof(buf));

        if (n > 0) {
            // 读取到数据
            printf("Received %zd bytes\n", n);
            process_data(buf, n);

        } else if (n == 0) {
            // 对方关闭连接（也会触发 EPOLLIN！）
            printf("Connection closed by peer\n");
            epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
            close(fd);

        } else {
            // 读取错误
            if (errno != EAGAIN && errno != EWOULDBLOCK) {
                perror("read");
                epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                close(fd);
            }
        }
    }
}
```

```c
// 示例 2：ET 模式下必须循环读取
if (events[i].events & EPOLLIN) {
    int fd = events[i].data.fd;

    // ET 模式：循环读取直到 EAGAIN
    while (1) {
        char buf[4096];
        ssize_t n = read(fd, buf, sizeof(buf));

        if (n > 0) {
            // 处理数据
            process_data(buf, n);

        } else if (n == 0) {
            // EOF
            close(fd);
            break;

        } else {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 数据读完了
                break;
            } else {
                // 真正的错误
                perror("read");
                close(fd);
                break;
            }
        }
    }
}
```

---

#### 2. EPOLLOUT - 可写事件

##### 2.1 定义

**EPOLLOUT**：文件描述符变为可写状态，即可以写入数据而不会阻塞。

##### 2.2 触发条件

```
对于TCP socket:
  ┌────────────────────────────────────────────────────────┐
  │ 1. 发送缓冲区有可用空间                                   │
  │    └─ 可以调用write()而不阻塞 → EPOLLOUT触发              │
  │                                                        │
  │ 2. TCP三次握手完成（connect 成功）                        │
  │    └─ 非阻塞connect 完成 → EPOLLOUT触发                  │
  │                                                        │
  │ 3. 之前发送缓冲区满，现在有空间了                           │
  │    └─ 收到对方ACK，释放空间 → EPOLLOUT触发                 │
  │                                                        │
  │ 4. socket 发生错误                                      │
  │    └─ EPOLLOUT + EPOLLERR同时触发                       │
  └────────────────────────────────────────────────────────┘

重要特性：
  ┌────────────────────────────────────────────────────────┐
  │ 大多数情况下，socket 的发送缓冲区都有空间！                  │
  │ └─ EPOLLOUT 几乎总是会触发                               │
  │                                                        │
  │ 所以：                                                  │
  │   不要一直监听 EPOLLOUT（会导致忙轮询）                     │
  │   只在需要时才监听 EPOLLOUT                               │
  └────────────────────────────────────────────────────────┘
```

##### 2.3 底层原理

**TCP Socket 发送数据的完整流程**：

```
1. 用户调用 write()/send()
   ┌──────────────────────────────────────┐
   │ write(sockfd, data, len)             │
   └──────────────────────────────────────┘
              │ 系统调用
              ▼
   ┌──────────────────────────────────────┐
   │ tcp_sendmsg()                        │
   │ └─ 检查发送缓冲区是否有空间              │
   └──────────────────────────────────────┘
              │
              ▼
   ┌─────────────────────────────────────────────────────┐
   │ 计算可用空间                                          │
   │                                                     │
   │   struct sock {                                     │
   │       struct sk_buff_head sk_write_queue;  ← 待发送  │
   │       int sk_sndbuf;       ← 总大小                  │
   │       atomic_t sk_wmem_queued; ← 已用                │
   │   }                                                 │
   │                                                     │
   │   free_space = sk_sndbuf - sk_wmem_queued           │
   └─────────────────────────────────────────────────────┘
              │
              ├─ Case A: free_space >= len
              │  └─→ 有足够空间
              │
              └─ Case B: free_space < len
                 ├─ 非阻塞socket: 返回EAGAIN
                 └─ 阻塞socket: 睡眠等待

2. 拷贝数据到发送缓冲区（Case A）
   ┌───────────────────────────────────────────────┐
   │ tcp_sendmsg_locked()                          │
   │ ├─ 分配sk_buff                                 │
   │ ├─ copy_from_iter()                           │
   │ │  └─ 从用户空间拷贝数据                         │
   │ └─ __skb_queue_tail(&sk->sk_write_queue, skb) │
   └───────────────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ tcp_push()                           │
   │ └─ 尝试立即发送数据                     │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ tcp_write_xmit()                     │
   │ └─ 根据拥塞窗口、接收窗口决定发送量       │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ IP层处理                              │
   │ ├─ 添加IP头                           │
   │ └─ 路由查找                           │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ 网卡驱动发送                           │
   │ └─ DMA拷贝到网卡                       │
   └──────────────────────────────────────┘

3. 接收对方的ACK
   ┌──────────────────────────────────────┐
   │ 收到ACK包                             │
   │ └─ tcp_ack()                         │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ tcp_clean_rtx_queue()                │
   │ └─ 从发送队列删除已确认的数据            │
   │    └─ kfree_skb(skb)                 │
   │       └─ sk_wmem_queued -= skb->len  │
   └──────────────────────────────────────┘
              │
              │ 缓冲区状态变化：满 → 有空间
              ▼
   ┌─────────────────────────────────────────┐
   │ sk_stream_write_space()                 │
   │ └─ if (空间足够)                         │
   │        wake_up_interruptible_sync_poll( │
   │            &sk->sk_wq,                  │
   │            POLLOUT | POLLWRNORM)        │
   └─────────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ ep_poll_callback()                   │
   │ └─ 将epitem加入就绪列表                │
   └──────────────────────────────────────┘
              │
              ▼
   ┌──────────────────────────────────────┐
   │ epoll_wait() 返回                     │
   │ └─ events[i].events = EPOLLOUT       │
   └──────────────────────────────────────┘
```

##### 2.4 什么时候使用EPOLLOUT？

```
错误用法：一直监听EPOLLOUT
──────────────────────────────────────────
struct epoll_event ev;
ev.events = EPOLLIN | EPOLLOUT;  // 不要这样！
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

问题：
  - 发送缓冲区通常都有空间
  - EPOLLOUT会一直触发
  - epoll_wait不停返回
  - 导致CPU空转（忙轮询）


正确用法：按需监听EPOLLOUT
──────────────────────────────────────────
1. 初始：只监听 EPOLLIN
   ev.events = EPOLLIN;

2. 尝试发送数据
   ssize_t n = write(fd, data, len);

3. 如果返回EAGAIN，添加EPOLLOUT
   if (n < 0 && errno == EAGAIN) {
       ev.events = EPOLLIN | EPOLLOUT;
       epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);
       // 保存未发送的数据
   }

4. EPOLLOUT 触发时，继续发送
   if (events[i].events & EPOLLOUT) {
       send_pending_data(fd);
   }

5. 数据发送完后，移除 EPOLLOUT
   ev.events = EPOLLIN;
   epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);
```

##### 2.5 代码示例

**场景 1：非阻塞 connect**

```c
// 使用EPOLLOUT检测非阻塞connect是否完成
int connect_nonblocking(int epfd, const char *host, int port) {
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    // 设置非阻塞
    int flags = fcntl(sockfd, F_GETFL, 0);
    fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);

    // 发起连接
    struct sockaddr_in addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    inet_pton(AF_INET, host, &addr.sin_addr);

    int ret = connect(sockfd, (struct sockaddr*)&addr, sizeof(addr));

    if (ret == -1) {
        if (errno == EINPROGRESS) {
            // 连接正在进行中
            // 监听EPOLLOUT，连接成功时会触发
            struct epoll_event ev;
            ev.events = EPOLLOUT;
            ev.data.fd = sockfd;
            epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

            printf("Connecting...\n");
            return sockfd;
        } else {
            perror("connect");
            close(sockfd);
            return -1;
        }
    }

    // 连接立即成功（少见）
    return sockfd;
}

// 检查连接是否成功
if (events[i].events & EPOLLOUT) {
    int fd = events[i].data.fd;

    // 检查 socket 错误
    int error;
    socklen_t len = sizeof(error);
    getsockopt(fd, SOL_SOCKET, SO_ERROR, &error, &len);

    if (error == 0) {
        // 连接成功
        printf("Connected!\n");

        // 修改为只监听 EPOLLIN
        struct epoll_event ev;
        ev.events = EPOLLIN;
        ev.data.fd = fd;
        epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);

    } else {
        // 连接失败
        printf("Connect failed: %s\n", strerror(error));
        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
        close(fd);
    }
}
```

**场景 2：处理 write 返回 EAGAIN**

```c
// 连接结构，保存待发送数据
struct connection {
    int fd;
    char *send_buffer;   // 待发送数据
    size_t send_offset;  // 已发送位置
    size_t send_length;  // 总长度
};

// 发送数据
int send_data(int epfd, struct connection *conn, const char *data, size_t len) {
    ssize_t written = write(conn->fd, data, len);

    if (written < 0) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            // 发送缓冲区满了
            printf("Send buffer full, waiting for EPOLLOUT\n");

            // 保存待发送数据
            conn->send_buffer = malloc(len);
            memcpy(conn->send_buffer, data, len);
            conn->send_offset = 0;
            conn->send_length = len;

            // 添加 EPOLLOUT 监听
            struct epoll_event ev;
            ev.events = EPOLLIN | EPOLLOUT;  // 添加 EPOLLOUT
            ev.data.ptr = conn;
            epoll_ctl(epfd, EPOLL_CTL_MOD, conn->fd, &ev);

            return 0;  // 稍后继续发送
        } else {
            perror("write");
            return -1;
        }
    }

    if (written < len) {
        // 部分写入
        printf("Partial write: %zd/%zu bytes\n", written, len);

        // 保存剩余数据
        size_t remaining = len - written;
        conn->send_buffer = malloc(remaining);
        memcpy(conn->send_buffer, data + written, remaining);
        conn->send_offset = 0;
        conn->send_length = remaining;

        // 添加 EPOLLOUT 监听
        struct epoll_event ev;
        ev.events = EPOLLIN | EPOLLOUT;
        ev.data.ptr = conn;
        epoll_ctl(epfd, EPOLL_CTL_MOD, conn->fd, &ev);

        return 0;
    }

    // 全部写入成功
    return len;
}

// 处理EPOLLOUT事件
if (events[i].events & EPOLLOUT) {
    struct connection *conn = events[i].data.ptr;

    if (conn->send_buffer && conn->send_length > 0) {
        // 继续发送剩余数据
        ssize_t written = write(conn->fd,
                               conn->send_buffer + conn->send_offset,
                               conn->send_length - conn->send_offset);

        if (written > 0) {
            conn->send_offset += written;

            if (conn->send_offset >= conn->send_length) {
                // 全部发送完成
                printf("Send completed\n");

                free(conn->send_buffer);
                conn->send_buffer = NULL;
                conn->send_offset = 0;
                conn->send_length = 0;

                // 移除EPOLLOUT监听
                struct epoll_event ev;
                ev.events = EPOLLIN;  // 只保留 EPOLLIN
                ev.data.ptr = conn;
                epoll_ctl(epfd, EPOLL_CTL_MOD, conn->fd, &ev);
            }
        } else if (written < 0 && errno != EAGAIN) {
            perror("write");
            // 错误处理
        }
    }
}
```

---

#### 3. EPOLLIN vs EPOLLOUT 对比

| 维度             | EPOLLIN                  | EPOLLOUT                     |
|----------------|--------------------------|------------------------------|
| **含义**         | 可读                       | 可写                           |
| **触发条件**       | 接收缓冲区有数据                 | 发送缓冲区有空间                     |
| **对于监听socket** | 有新连接到达                   | N/A（监听socket不用于发送）           |
| **对于连接socket** | 有数据到达<br>对方关闭连接          | 发送缓冲区有空间<br>connect 完成       |
| **触发频率**       | 按需触发（有数据才触发）             | 几乎总是触发（缓冲区通常有空间）             |
| **默认监听**       | 是<br>几乎所有场景都需要           | 否<br>只在需要时才添加                |
| **使用场景**       | - 接受连接<br>- 读取数据         | - 非阻塞 connect<br>- 处理 EAGAIN |
| **不触发时的行为**    | read() 阻塞<br>（阻塞 socket） | write() 阻塞<br>（阻塞 socket）    |
| **内核实现**       | sock_def_readable()      | sk_stream_write_space()      |
| **回调触发点**      | 数据到达时                    | 收到ACK释放缓冲区时                  |
| **性能注意事项**     | 通常不会有问题                  | 不要一直监听（会导致忙轮询）               |

---

#### 4. Socket 缓冲区状态图解

##### 4.1 接收缓冲区（EPOLLIN）

```
TCP Socket 接收缓冲区状态变化：

状态 A: 缓冲区为空
┌─────────────────────────────────────────────────┐
│ sk_receive_queue                                │
├─────────────────────────────────────────────────┤
│                                                 │
│ [ 空 ]                                          │
│                                                 │
└─────────────────────────────────────────────────┘
EPOLLIN: 不触发
read(): 阻塞等待（阻塞 socket）或返回 EAGAIN（非阻塞）

        │ 数据包到达
        ▼

状态 B: 缓冲区有数据
┌─────────────────────────────────────────────────┐
│ sk_receive_queue                                │
├─────────────────────────────────────────────────┤
│                                                 │
│ [数据包1] → [数据包2] → [数据包3] → NULL         │
│                                                 │
└─────────────────────────────────────────────────┘
EPOLLIN: 触发（状态变化：空 → 非空）
read(): 可以读取数据

        │ 用户调用read()
        ▼

状态 C: 读取部分数据
┌─────────────────────────────────────────────────┐
│ sk_receive_queue                                │
├─────────────────────────────────────────────────┤
│                                                 │
│ [数据包3] → NULL                                 │
│                                                 │
└─────────────────────────────────────────────────┘
EPOLLIN:
  - LT 模式：继续触发（还有数据）
  - ET 模式：不触发（状态未变化）

        │ 用户继续 read()
        ▼

状态 D: 读取完所有数据
┌─────────────────────────────────────────────────┐
│ sk_receive_queue                                │
├─────────────────────────────────────────────────┤
│                                                 │
│ [ 空 ]                                          │
│                                                 │
└─────────────────────────────────────────────────┘
EPOLLIN: 不触发
read(): 返回EAGAIN（非阻塞）
```

##### 4.2 发送缓冲区（EPOLLOUT）

```
TCP Socket 发送缓冲区状态变化：

状态A: 缓冲区有足够空间（常见情况）
┌─────────────────────────────────────────────────┐
│ sk_write_queue                                  │
├─────────────────────────────────────────────────┤
│ [已发送待ACK] [已发送待ACK]     [ 大量可用空间 ]     │
│                                                 │
│ sk_wmem_queued: 4KB                             │
│ sk_sndbuf: 16KB                                 │
│ 可用: 12KB                                       │
└─────────────────────────────────────────────────┘
EPOLLOUT: 触发
write(): 可以写入数据

        │ 用户大量写入数据
        ▼

状态B: 缓冲区接近满
┌─────────────────────────────────────────────────┐
│ sk_write_queue                                  │
├─────────────────────────────────────────────────┤
│ [待发送][待发送][待发送][已发送待ACK][已发送待ACK]    │
│                                                 │
│ sk_wmem_queued: 15KB                            │
│ sk_sndbuf: 16KB                                 │
│ 可用: 1KB                                        │
└─────────────────────────────────────────────────┘
EPOLLOUT: 可能触发（取决于阈值）
write(10KB): 只能写入1KB，返回1024或EAGAIN

        │ 继续写入，缓冲区满
        ▼

状态C: 缓冲区满了（罕见）
┌─────────────────────────────────────────────────┐
│ sk_write_queue                                  │
├─────────────────────────────────────────────────┤
│ [待发送][待发送][待发送][待发送][已发送待ACK]...     │
│                                                 │
│ sk_wmem_queued: 16KB                            │
│ sk_sndbuf: 16KB                                 │
│ 可用: 0KB                                        │
└─────────────────────────────────────────────────┘
EPOLLOUT: 不触发
write(): 返回EAGAIN（非阻塞）或阻塞（阻塞 socket）

        │ 收到ACK，释放空间
        ▼

状态D: 收到ACK，空间释放
┌─────────────────────────────────────────────────┐
│ sk_write_queue                                  │
├─────────────────────────────────────────────────┤
│ [待发送][待发送]                 [ 可用空间 ]      │
│                                                 │
│ sk_wmem_queued: 8KB                             │
│ sk_sndbuf: 16KB                                 │
│ 可用: 8KB                                        │
└─────────────────────────────────────────────────┘
EPOLLOUT: 触发 （状态变化：满 → 有空间）
        └─ ep_poll_callback() 被调用
```

---

#### 5. 最佳实践

##### 5.1 EPOLLIN使用建议

```c
推荐做法
──────────────────────────────────────────
1. 默认监听 EPOLLIN
   ev.events = EPOLLIN;

2. LT模式：可以只读部分数据
   char buf[1024];
   read(fd, buf, sizeof(buf));  // OK

3. ET模式：必须循环读取直到EAGAIN
   while (1) {
       ssize_t n = read(fd, buf, sizeof(buf));
       if (n < 0 && errno == EAGAIN) break;
       // 处理数据...
   }

4. 检查 read() 返回 0（连接关闭）
   if (n == 0) {
       // 对方关闭连接
       close(fd);
   }
```

##### 5.2 EPOLLOUT使用建议

```c
推荐做法
──────────────────────────────────────────
1. 初始不监听EPOLLOUT
   ev.events = EPOLLIN;  // 只监听 EPOLLIN

2. 先尝试直接写入
   ssize_t n = write(fd, data, len);

3. write返回EAGAIN时，才添加EPOLLOUT
   if (n < 0 && errno == EAGAIN) {
       ev.events = EPOLLIN | EPOLLOUT;
       epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);
   }

4. 数据发送完后，立即移除EPOLLOUT
   if (发送完成) {
       ev.events = EPOLLIN;
       epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);
   }


错误做法
──────────────────────────────────────────
1. 一直监听 EPOLLOUT（CPU 忙轮询）
   ev.events = EPOLLIN | EPOLLOUT;  //

2. 不检查 write 返回值
   write(fd, data, len);  // 没检查返回值

3. 发送完后不移除 EPOLLOUT
   // 忘记移除 EPOLLOUT
```

##### 5.3 完整示例

```c
// 推荐的事件处理模式
void handle_events(int epfd, struct epoll_event *events, int nfds) {
    for (int i = 0; i < nfds; i++) {
        struct connection *conn = events[i].data.ptr;
        uint32_t evt = events[i].events;

        // 1. 先处理错误
        if (evt & (EPOLLERR | EPOLLHUP)) {
            close_connection(epfd, conn);
            continue;
        }

        // 2. 处理可读
        if (evt & EPOLLIN) {
            handle_read(epfd, conn);
        }

        // 3. 处理可写
        if (evt & EPOLLOUT) {
            handle_write(epfd, conn);
        }
    }
}
```

---

#### 6. 常见问题 FAQ

##### Q1: 为什么对方关闭连接时也会触发EPOLLIN？

**A**: 对方关闭连接时，会发送FIN包。内核收到FIN后，会将socket状态设置为可读，此时`read()`会返回0（EOF）。这也是EPOLLIN触发的一种情况。

```c
if (events[i].events & EPOLLIN) {
    ssize_t n = read(fd, buf, sizeof(buf));
    if (n == 0) {
        // 对方关闭了连接
        printf("Connection closed\n");
        close(fd);
    }
}
```

##### Q2: 为什么EPOLLOUT总是触发？

**A**: 因为大多数情况下，发送缓冲区都有可用空间。只有在网络拥塞、对方接收慢的情况下，发送缓冲区才会满。

**解决方案**：不要一直监听EPOLLOUT，只在需要时才添加。

##### Q3: ET模式下，如果没读完数据会怎样？

**A**: ET模式下，如果一次EPOLLIN触发后没有读完所有数据，剩余数据不会再触发EPOLLIN，直到有新数据到达。

```c
// ET模式必须循环读取
while (1) {
    ssize_t n = read(fd, buf, sizeof(buf));
    if (n < 0 && errno == EAGAIN) {
        break;  // 确保读完所有数据
    }
    // 处理...
}
```

##### Q4: 如何判断非阻塞connect是否成功？

**A**: 监听EPOLLOUT，触发后检查SO_ERROR。

```c
if (events[i].events & EPOLLOUT) {
    int error;
    socklen_t len = sizeof(error);
    getsockopt(fd, SOL_SOCKET, SO_ERROR, &error, &len);

    if (error == 0) {
        printf("Connected!\n");
    } else {
        printf("Connect failed: %s\n", strerror(error));
    }
}
```

##### Q5: LT模式和ET模式哪个更好？

**A**:

- **LT模式**：简单、不容易出错，适合大多数场景
- **ET模式**：性能更好，但编程复杂，需要配合非阻塞I/O

推荐：初学者使用LT，高性能服务器使用ET。

---

#### 7. 总结

##### EPOLLIN核心要点

- 接收缓冲区有数据时触发
- 默认监听
- LT模式可以只读部分数据
- ET模式必须循环读取直到EAGAIN

##### EPOLLOUT核心要点

- 发送缓冲区有空间时触发
- 不要一直监听（会忙轮询）
- 只在write返回EAGAIN时才添加
- 数据发送完后立即移除

##### 典型使用模式

```c
// 初始化：只监听 EPOLLIN
ev.events = EPOLLIN;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

// 读取数据
if (events[i].events & EPOLLIN) {
    read(fd, ...);
}

// 发送数据：先尝试直接写
ssize_t n = write(fd, data, len);
if (n < 0 && errno == EAGAIN) {
    // 缓冲区满，添加 EPOLLOUT
    ev.events = EPOLLIN | EPOLLOUT;
    epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);
}

// EPOLLOUT 触发时继续发送
if (events[i].events & EPOLLOUT) {
    send_pending_data(fd);
    // 发送完后移除 EPOLLOUT
    ev.events = EPOLLIN;
    epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);
}
```
### 2.4 epoll_wait

**功能**：等待I/O事件发生

```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

**参数**：

- `epfd`: epoll实例的文件描述符
- `events`: 用于返回就绪事件的数组
- `maxevents`: events数组的大小
- `timeout`: 超时时间（毫秒）
    - `-1`: 永久阻塞，直到有事件
    - `0`: 立即返回（非阻塞）
    - `> 0`: 阻塞指定毫秒数

**返回值**：

- `> 0`: 就绪的文件描述符数量
- `0`: 超时，没有事件
- `-1`: 错误

**示例**：

```c
#define MAX_EVENTS 10
struct epoll_event events[MAX_EVENTS];

while (1) {
    int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
    if (nfds == -1) {
        perror("epoll_wait");
        exit(EXIT_FAILURE);
    }

    for (int i = 0; i < nfds; i++) {
        if (events[i].events & EPOLLIN) {
            // 处理可读事件
            int fd = events[i].data.fd;
            handle_read(fd);
        }
    }
}
```

---

## 3. I/O多路复用对比

### 3.1 select vs poll vs epoll

| 特性           | select            | poll        | epoll         |
|--------------|-------------------|-------------|---------------|
| **最大监控数**    | 1024 (FD_SETSIZE) | 无限制（受内存限制）  | 无限制（受内存限制）    |
| **fd集合拷贝**   | 每次调用都拷贝           | 每次调用都拷贝     | 内核维护，无需拷贝     |
| **检查就绪fd方式** | 遍历所有fd O(n)       | 遍历所有fd O(n) | 只返回就绪fd O(1)  |
| **数据结构**     | bitmap(fd_set)    | 数组(pollfd)  | 红黑树 + 链表      |
| **触发模式**     | 水平触发（LT）          | 水平触发（LT）    | LT / ET 都支持   |
| **性能（大量连接）** | 差O(n)             | 差O(n)       | 优秀O(1)        |
| **跨平台**      | (POSIX)           | (POSIX)     | (仅Linux)      |
| **适用场景**     | 少量连接（<100）        | 中等连接（<1000） | 海量连接（>10,000） |

### 3.2 工作流程对比

**select/poll工作流程**：

```c
// 每次循环都要重复这些步骤
while (1) {
    // 1. 构造fd集合（用户空间）
    fd_set readfds;
    FD_ZERO(&readfds);
    for (int i = 0; i < n; i++) {
        FD_SET(fds[i], &readfds);
    }

    // 2. 将fd集合从用户空间拷贝到内核空间                ← 开销！
    select(maxfd + 1, &readfds, NULL, NULL, NULL);
    // 内核遍历所有fd，检查是否就绪                      ← O(n) 开销！

    // 3. 将结果从内核空间拷贝回用户空间                  ← 开销！

    // 4. 用户空间遍历所有fd，检查哪些就绪                ← O(n) 开销！
    for (int i = 0; i < n; i++) {
        if (FD_ISSET(fds[i], &readfds)) {
            handle(fds[i]);
        }
    }
}

// 问题：
// - 每次调用都要拷贝整个f集合
// - 内核遍历O(n)
// - 用户遍历O(n)
// - 总时间复杂度：O(n)，n是监控的fd数量
```

**epoll工作流程**：

```c
// 1. 创建epoll实例（一次性）
int epfd = epoll_create1(0);

// 2. 添加fd到 epoll（一次性）
for (int i = 0; i < n; i++) {
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.fd = fds[i];
    epoll_ctl(epfd, EPOLL_CTL_ADD, fds[i], &ev);
    // 内核将fd加入红黑树O(log n)
    // 注册回调函数到fd的等待队列
}

// 3. 等待事件（高效）
while (1) {
    struct epoll_event events[MAX_EVENTS];

    // 只返回就绪的fd，无需拷贝所有fd                         ← 高效！
    int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);

    // 只遍历就绪的fd                                       ← O(m)，m是就绪的fd数量
    for (int i = 0; i < nfds; i++) {
        handle(events[i].data.fd);
    }
}

// 优势：
// - fd集合在内核中维护，无需重复拷贝
// - 只返回就绪的fd，O(1) 获取
// - 只遍历就绪的fd，O(m)，m << n
```

### 3.3 性能对比

```
场景：监控10,000个连接，其中100个就绪

┌──────────────┬──────────────┬──────────────┬──────────────┐
│              │   select     │     poll     │    epoll     │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ 拷贝开销      │ 10,000 fds   │ 10,000 fds   │ 0            │
│              │ 每次调用都拷贝 │ 每次调用都拷贝  │ 内核维护      │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ 内核遍历      │ 10,000 fds   │ 10,000 fds   │ 0            │
│              │ O(10,000)    │ O(10,000)    │ 事件驱动      │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ 用户遍历      │ 10,000 fds   │ 10,000 fds   │ 100 fds      │
│              │ O(10,000)    │ O(10,000)    │ O(100)       │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ 总复杂度      │ O(n)         │ O(n)         │ O(m)         │
│              │ n = 10,000   │ n = 10,000   │ m = 100      │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ 相对性能      │ 1x (基准)     │ 1x (类似)     │ 100x ⚡       │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

---

## 4. epoll内核数据结构

### 4.1 核心数据结构概览

```c
内核中epoll的核心数据结构：

┌────────────────────────────────────────────────────────────────┐
│                      eventpoll                                 │
│  （每个epoll实例对应一个eventpoll结构）                            │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌─────────────────────────────────────────┐                   │
│  │        红黑树 (rb_root rbr)              │  ← 存储所有监控的fd │
│  │                                         │                   │
│  │     epitem       epitem       epitem    │                   │
│  │       /\           /\           /\      │                   │
│  │      /  \         /  \         /  \     │                   │
│  │  epitem epitem epitem epitem epitem...  │                   │
│  │                                         │                   │
│  │  O(log n) 查找、插入、删除                 │                   │
│  └─────────────────────────────────────────┘                   │
│                                                                │
│  ┌─────────────────────────────────────────┐                   │
│  │    就绪列表 (list_head rdllist)          │  ← 存储就绪的fd     │
│  │                                         │                   │
│  │  [epitem] → [epitem] → [epitem] → NULL  │                   │
│  │                                         │                   │
│  │  当fd就绪时，内核回调函数将epitem           │                   │
│  │  添加到这个链表                           │                   │
│  └─────────────────────────────────────────┘                   │
│                                                                │
│  ┌─────────────────────────────────────────┐                   │
│  │    等待队列 (wait_queue_head_t wq)       │  ← 阻塞的进程       │
│  │                                         │                   │
│  │  epoll_wait() 阻塞时，进程加入这个队列      │                   │
│  │  有事件就绪时，唤醒队列中的进程              │                   │
│  └─────────────────────────────────────────┘                   │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 4.2 eventpoll结构体

**内核源码** (linux/fs/eventpoll.c):

```c
struct eventpoll {
    /* 保护 eventpoll 的自旋锁 */
    spinlock_t lock;

    /* 互斥锁，用于某些操作 */
    struct mutex mtx;

    /* epoll_wait() 阻塞时使用的等待队列 */
    wait_queue_head_t wq;

    /* file->poll() 使用的等待队列 */
    wait_queue_head_t poll_wait;

    /* 就绪列表：存储就绪的 epitem */
    struct list_head rdllist;

    /* 红黑树根节点：存储所有监控的 epitem */
    struct rb_root_cached rbr;

    /* 单链表：存储 ovflist，用于在 epoll_wait 遍历 rdllist 时
     * 新到达的事件临时存储，避免丢失 */
    struct epitem *ovflist;

    /* 创建 epoll 的用户信息 */
    struct user_struct *user;

    /* epoll 对应的 file 结构 */
    struct file *file;

    /* 用于检测循环依赖 */
    int visited;
    struct list_head visited_list_link;
};
```

### 4.3 epitem 结构体

**epitem**: 每个被监控的文件描述符对应一个epitem

```c
struct epitem {
    /* 红黑树节点 */
    struct rb_node rbn;

    /* 就绪列表节点 */
    struct list_head rdllink;

    /* epitem所属的epoll */
    struct eventpoll *ep;

    /* 被监控的文件描述符和文件对象 */
    struct epoll_filefd ffd;

    /* 引用计数 */
    int refcount;

    /* 用户传入的事件和数据 */
    struct epoll_event event;

    /* 链表头：用于链接到file的f_ep_links */
    struct list_head fllink;

    /* 等待队列项：用于注册到fd的等待队列 */
    struct ep_queue *pwqlist;
};
```

### 4.4 关键数据结构关系图

```
用户空间                                内核空间
──────────────────────────────────────────────────────────────

epoll_create1()                     eventpoll (epfd=3)
    │                              ┌─────────────────────┐
    └─────────────────────────────→│ lock, mtx           │
                                   │                     │
                                   │ wq (等待队列)        │
                                   │                     │
                                   │ rdllist (就绪列表)   │
                                   │   → 空              │
                                   │                     │
                                   │ rbr (红黑树)         │
                                   │   → 空              │
                                   └─────────────────────┘

epoll_ctl(epfd, ADD, fd1, &ev)
    │                              添加 epitem1 到红黑树
    └─────────────────────────────→
                                   eventpoll (epfd=3)
                                   ┌─────────────────────┐
                                   │ rbr (红黑树)         │
                                   │   → epitem1 (fd=4)  │
                                   │       ├─ event      │
                                   │       ├─ ffd        │
                                   │       └─ pwqlist    │
                                   └─────────────────────┘
                                           │
                                           │ 注册回调函数
                                           ▼
                                   socket (fd=4)
                                   ┌─────────────────────┐
                                   │ wait_queue          │
                                   │   → ep_poll_callback│
                                   └─────────────────────┘

数据到达 socket
    ─────────────────────────────→ socket (fd=4)
                                   │ 数据到达！
                                   │ 调用 wait_queue 上的回调
                                   ▼
                                   ep_poll_callback()
                                   │ 将 epitem1 加入 rdllist
                                   ▼
                                   eventpoll (epfd=3)
                                   ┌─────────────────────┐
                                   │ rdllist (就绪列表)   │
                                   │   → epitem1         │
                                   │                     │
                                   │ wq (等待队列)        │
                                   │   → 唤醒阻塞的进程    │
                                   └─────────────────────┘

epoll_wait(epfd, events, ...)
    │                              遍历 rdllist
    │◀────────────────────────────[epitem1, epitem2, ...]
    │
    └─→ events[0] = {fd=4, EPOLLIN}
```

---

## 5. epoll_create 内核实现

### 5.1 系统调用入口

```c
// linux/fs/eventpoll.c

SYSCALL_DEFINE1(epoll_create1, int, flags)
{
    return do_epoll_create(flags);
}

static int do_epoll_create(int flags)
{
    int error, fd;
    struct eventpoll *ep = NULL;
    struct file *file;

    /* 1. 分配eventpoll结构 */
    error = ep_alloc(&ep);
    if (error < 0)
        return error;

    /* 2. 获取未使用的文件描述符 */
    fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));
    if (fd < 0) {
        ep_free(ep);
        return fd;
    }

    /* 3. 创建file结构 */
    file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep, O_RDWR | (flags & O_CLOEXEC));
    if (IS_ERR(file)) {
        put_unused_fd(fd);
        ep_free(ep);
        return PTR_ERR(file);
    }

    /* 4. 关联file和eventpoll */
    ep->file = file;

    /* 5. 将file和fd关联 */
    fd_install(fd, file);

    return fd;  /* 返回 epoll 文件描述符 */
}
```

### 5.2 ep_alloc - 分配 eventpoll 结构

```c
static int ep_alloc(struct eventpoll **pep)
{
    struct eventpoll *ep;

    /* 分配内存 */
    ep = kzalloc(sizeof(*ep), GFP_KERNEL);
    if (unlikely(!ep))
        return -ENOMEM;

    /* 初始化自旋锁 */
    spin_lock_init(&ep->lock);

    /* 初始化互斥锁 */
    mutex_init(&ep->mtx);

    /* 初始化等待队列 */
    init_waitqueue_head(&ep->wq);
    init_waitqueue_head(&ep->poll_wait);

    /* 初始化就绪列表 */
    INIT_LIST_HEAD(&ep->rdllist);

    /* 初始化红黑树 */
    ep->rbr = RB_ROOT_CACHED;

    /* ovflist 初始化为 EP_UNACTIVE_PTR */
    ep->ovflist = EP_UNACTIVE_PTR;

    /* 记录创建者的用户信息 */
    ep->user = get_current_user();

    *pep = ep;
    return 0;
}
```

### 5.3 创建流程图

```
用户空间                内核空间
────────────────────────────────────────────────────────

epoll_create1(0)
    │
    │ 系统调用
    ▼
SYSCALL_DEFINE1(epoll_create1)
    │
    ▼
do_epoll_create()
    │
    ├─→ 1. ep_alloc(&ep)
    │   ├─ kzalloc(sizeof(eventpoll))            ← 分配内存
    │   ├─ spin_lock_init(&ep->lock)             ← 初始化自旋锁
    │   ├─ init_waitqueue_head(&ep->wq)          ← 初始化等待队列
    │   ├─ INIT_LIST_HEAD(&ep->rdllist)          ← 初始化就绪列表
    │   └─ ep->rbr = RB_ROOT_CACHED              ← 初始化红黑树
    │
    ├─→ 2. get_unused_fd_flags()                 ← 分配fd
    │       返回：fd = 3
    │
    ├─→ 3. anon_inode_getfile()                  ← 创建匿名inode
    │       创建file结构
    │       file->f_op = &eventpoll_fops
    │       file->private_data = ep
    │
    ├─→ 4. ep->file = file                       ← 关联
    │
    └─→ 5. fd_install(fd, file)                  ← 安装fd
            current->files->fd_array[3] = file

    返回 fd = 3
    │
    ▼
用户空间得到 epfd = 3
```

---

## 6. epoll_ctl内核实现

### 6.1 系统调用入口

```c
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd, struct epoll_event __user *, event)
{
    struct epoll_event epds;

    /* 从用户空间拷贝event */
    if (ep_op_has_event(op) && copy_from_user(&epds, event, sizeof(struct epoll_event)))
        return -EFAULT;

    return do_epoll_ctl(epfd, op, fd, &epds, false);
}

int do_epoll_ctl(int epfd, int op, int tfd, struct epoll_event *epds, bool nonblock)
{
    int error;
    struct fd f, tf;
    struct eventpoll *ep;
    struct epitem *epi;
    struct eventpoll *tep = NULL;

    /* 1. 获取epoll文件 */
    f = fdget(epfd);
    if (!f.file)
        return -EBADF;

    /* 2. 获取目标文件 */
    tf = fdget(tfd);
    if (!tf.file) {
        error = -EBADF;
        goto error_fput;
    }

    /* 3. 检查epoll文件是否有效 */
    if (!is_file_epoll(f.file)) {
        error = -EINVAL;
        goto error_tgt_fput;
    }

    /* 4. 获取eventpoll结构 */
    ep = f.file->private_data;

    /* 5. 查找epitem*/
    epi = ep_find(ep, tf.file, tfd);

    /* 6. 根据操作类型执行 */
    switch (op) {
    case EPOLL_CTL_ADD:
        if (!epi) {
            epds->events |= EPOLLERR | EPOLLHUP;
            error = ep_insert(ep, epds, tf.file, tfd, false);
        } else
            error = -EEXIST;
        break;

    case EPOLL_CTL_DEL:
        if (epi)
            error = ep_remove(ep, epi);
        else
            error = -ENOENT;
        break;

    case EPOLL_CTL_MOD:
        if (epi) {
            epds->events |= EPOLLERR | EPOLLHUP;
            error = ep_modify(ep, epi, epds);
        } else
            error = -ENOENT;
        break;
    }

error_tgt_fput:
    fdput(tf);
error_fput:
    fdput(f);
    return error;
}
```

### 6.2 ep_insert - 添加文件描述符

**这是epoll最核心的函数之一**，理解它就理解了epoll的事件通知机制！

```c
static int ep_insert(struct eventpoll *ep, struct epoll_event *event, struct file *tfile, int fd, int full_check)
{
    int error, revents;
    struct epitem *epi;
    struct ep_pqueue epq;

    /* 1. 分配epitem结构*/
    if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
        return -ENOMEM;

    /* 2. 初始化epitem*/
    INIT_LIST_HEAD(&epi->rdllink);
    INIT_LIST_HEAD(&epi->fllink);
    INIT_LIST_HEAD(&epi->pwqlist);
    epi->ep = ep;
    ep_set_ffd(&epi->ffd, tfile, fd);
    epi->event = *event;
    epi->nwait = 0;
    epi->next = EP_UNACTIVE_PTR;

    /* 3. 初始化poll队列 */
    epq.epi = epi;
    init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

    /* 4. 调用文件的poll方法，注册回调函数 关键！*/
    revents = ep_item_poll(epi, &epq.pt, 1);

    /* 5. 将epitem插入红黑树 */
    ep_rbtree_insert(ep, epi);

    /* 6. 如果已经就绪，加入就绪列表 */
    if (revents && !ep_is_linked(&epi->rdllink)) {
        list_add_tail(&epi->rdllink, &ep->rdllist);
        ep_pm_stay_awake(epi);

        /* 唤醒等待的进程 */
        if (waitqueue_active(&ep->wq))
            wake_up(&ep->wq);
    }

    return 0;
}
```

### 6.3 ep_ptable_queue_proc - 注册回调函数

**这是epoll事件通知的核心！**

```c
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead, poll_table *pt)
{
    struct epitem *epi = ep_item_from_epqueue(pt);
    struct eppoll_entry *pwq;

    /* 分配eppoll_entry */
    if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
        /* 设置回调函数为ep_poll_callback 关键！*/
        init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
        pwq->whead = whead;
        pwq->base = epi;

        /* 将回调函数添加到文件的等待队列 */
        add_wait_queue(whead, &pwq->wait);

        list_add_tail(&pwq->llink, &epi->pwqlist);
        epi->nwait++;
    } else {
        /* 内存不足 */
        epi->nwait = -1;
    }
}
```

### 6.4 关键流程：回调函数如何注册？

```
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &event)
    │
    ▼
do_epoll_ctl()
    │
    ▼
ep_insert()
    │
    ├─→ 1. 分配epitem
    │
    ├─→ 2. 初始化epitem
    │       epi->ep = ep
    │       epi->ffd = {tfile, fd}
    │       epi->event = *event
    │
    ├─→ 3. 初始化poll_table
    │       epq.epi = epi
    │       epq.pt._qproc = ep_ptable_queue_proc  ← 设置回调注册函数
    │
    ├─→ 4. 调用文件的poll方法 关键步骤！
    │       revents = tfile->f_op->poll(tfile, &epq.pt)
    │
    │       对于socket，这会调用：
    │       sock_poll()
    │         → tcp_poll()
    │           → poll_wait(file, &sk->sk_wq, wait)  ← 这里！
    │             → epq.pt._qproc(file, &sk->sk_wq, &epq.pt)
    │               → ep_ptable_queue_proc()  ← 我们的回调注册函数
    │                 │
    │                 ├─ 分配 eppoll_entry (pwq)
    │                 ├─ pwq->wait.func = ep_poll_callback 设置回调！
    │                 └─ add_wait_queue(&sk->sk_wq, &pwq->wait)
    │                     ← 将回调添加到socket的等待队列
    │
    └─→ 5. 将epitem插入红黑树
            ep_rbtree_insert(ep, epi)

结果：
    socket (fd=4)
    ├─ sk_wq (等待队列)
    │   ├─ pwq->wait
    │   │   └─ func = ep_poll_callback  ← 回调函数已注册！
    │   └─ ...
    └─ ...

当数据到达时：
    网卡中断 → 内核协议栈 → socket缓冲区 → wake_up(&sk->sk_wq)
                                                   │
                                                   ▼
                                           调用所有等待队列上的回调
                                                   │
                                                   ▼
                                           ep_poll_callback()
```

### 6.5 ep_rbtree_insert - 插入红黑树

```c
static void ep_rbtree_insert(struct eventpoll *ep, struct epitem *epi)
{
    int kcmp;
    struct rb_node **p = &ep->rbr.rb_root.rb_node, *parent = NULL;
    struct epitem *epic;
    bool leftmost = true;

    /* 查找插入位置 */
    while (*p) {
        parent = *p;
        epic = rb_entry(parent, struct epitem, rbn);
        kcmp = ep_cmp_ffd(&epi->ffd, &epic->ffd);

        if (kcmp > 0) {
            p = &parent->rb_right;
            leftmost = false;
        } else {
            p = &parent->rb_left;
        }
    }

    /* 插入节点 */
    rb_link_node(&epi->rbn, parent, p);
    rb_insert_color_cached(&epi->rbn, &ep->rbr, leftmost);
}
```

### 6.6 epoll_ctl 完整流程图

```
EPOLL_CTL_ADD 流程：

1. 用户空间
   ├─ epoll_ctl(epfd=3, EPOLL_CTL_ADD, sockfd=4, &event)
   └─ event = {events: EPOLLIN | EPOLLET, data.fd: 4}

2. 系统调用
   ├─ copy_from_user(&epds, event, ...)  ← 拷贝event
   └─ do_epoll_ctl(3, ADD, 4, &epds)

3. 获取对象
   ├─ f = fdget(3)                ← 获取epoll file
   ├─ tf = fdget(4)               ← 获取socket file
   └─ ep = f.file->private_data   ← 获取eventpoll

4. 查找epitem
   └─ epi = ep_find(ep, tf.file, 4)
      └─ 在红黑树中查找 (file=tf.file, fd=4)
      └─ 未找到 → NULL

5. 执行ep_insert()
   ├─ 分配epitem
   │  └─ epi = kmem_cache_alloc(epi_cache)
   │
   ├─ 初始化epitem
   │  ├─ epi->ep = ep
   │  ├─ epi->ffd = {tf.file, 4}
   │  └─ epi->event = {EPOLLIN|EPOLLET, fd:4}
   │
   ├─ 注册回调函数
   │  ├─ init_poll_funcptr(&epq.pt, ep_ptable_queue_proc)
   │  ├─ revents = tf.file->f_op->poll(tf.file, &epq.pt)
   │  │   └─ sock_poll()
   │  │       └─ tcp_poll()
   │  │           └─ poll_wait(file, &sk->sk_wq, &epq.pt)
   │  │               └─ ep_ptable_queue_proc()
   │  │                   ├─ pwq = kmem_cache_alloc(pwq_cache)
   │  │                   ├─ pwq->wait.func = ep_poll_callback
   │  │                   └─ add_wait_queue(&sk->sk_wq, &pwq->wait)
   │  │
   │  └─ 现在socket的等待队列上有了ep_poll_callback！
   │
   ├─ 插入红黑树
   │  └─ ep_rbtree_insert(ep, epi)
   │      └─ rb_insert_color_cached(&epi->rbn, &ep->rbr)
   │
   └─ 如果已就绪
       ├─ if (revents && !ep_is_linked(&epi->rdllink))
       ├─ list_add_tail(&epi->rdllink, &ep->rdllist)
       └─ wake_up(&ep->wq)

6. 完成
   └─ 返回 0

结果状态：
┌────────────────────────────────────────────────────────┐
│ eventpoll (epfd=3)                                     │
├────────────────────────────────────────────────────────┤
│ rbr (红黑树)                                            │
│   └─ epitem (fd=4)                                     │
│       ├─ event = {EPOLLIN|EPOLLET, fd:4}               │
│       ├─ ffd = {file, fd=4}                            │
│       └─ pwqlist → eppoll_entry                        │
│                      └─ wait.func = ep_poll_callback   │
│                                                        │
│ rdllist (就绪列表)                                      │
│   └─ 空（如果 socket 没有数据）                           │
└────────────────────────────────────────────────────────┘
           │
           │ 注册了回调
           ▼
┌────────────────────────────────────────────────────────┐
│ socket (fd=4)                                          │
├────────────────────────────────────────────────────────┤
│ sk_wq (等待队列)                                        │
│   └─ wait_queue_entry                                  │
│       └─ func = ep_poll_callback  ⚡                    │
└────────────────────────────────────────────────────────┘
```

---

## 7. epoll_wait内核实现

### 7.1 系统调用入口

```c
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events, int, maxevents, int, timeout)
{
    return do_epoll_wait(epfd, events, maxevents, timeout);
}

static int do_epoll_wait(int epfd, struct epoll_event __user *events, int maxevents, int timeout)
{
    int error;
    struct fd f;
    struct eventpoll *ep;

    /* 1. 检查 maxevents */
    if (maxevents <= 0 || maxevents > EP_MAX_EVENTS)
        return -EINVAL;

    /* 2. 检查 events 指针 */
    if (!access_ok(events, maxevents * sizeof(struct epoll_event)))
        return -EFAULT;

    /* 3. 获取 epoll file */
    f = fdget(epfd);
    if (!f.file)
        return -EBADF;

    /* 4. 检查是否是 epoll 文件 */
    error = -EINVAL;
    if (!is_file_epoll(f.file))
        goto error_fput;

    /* 5. 获取 eventpoll */
    ep = f.file->private_data;

    /* 6. 等待事件 */
    error = ep_poll(ep, events, maxevents, timeout);

error_fput:
    fdput(f);
    return error;
}
```

### 7.2 ep_poll - 等待事件核心函数

```c
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events, int maxevents, long timeout)
{
    int res = 0, eavail, timed_out = 0;
    u64 slack = 0;
    wait_queue_entry_t wait;
    ktime_t expires, *to = NULL;

    /* 1. 计算超时时间 */
    if (timeout > 0) {
        struct timespec64 end_time = ep_set_mstimeout(timeout);
        slack = select_estimate_accuracy(&end_time);
        to = &expires;
        *to = timespec64_to_ktime(end_time);
    } else if (timeout == 0) {
        timed_out = 1;
        /* 非阻塞模式，立即返回 */
    }

fetch_events:
    /* 2. 快速路径：检查是否有就绪事件 */
    eavail = ep_events_available(ep);
    if (eavail)
        goto send_events;

    /* 3. 没有就绪事件，需要阻塞等待 */
    if (!timed_out) {
        /* 初始化等待队列项 */
        init_waitqueue_entry(&wait, current);

        /* 加锁 */
        spin_lock_irq(&ep->lock);

        /* 将当前进程加入等待队列 */
        __add_wait_queue_exclusive(&ep->wq, &wait);

        /* 再次检查是否有就绪事件（避免竞态） */
        eavail = ep_events_available(ep);

        while (!eavail) {
            /* 设置进程状态为可中断睡眠 */
            set_current_state(TASK_INTERRUPTIBLE);

            /* 释放锁 */
            spin_unlock_irq(&ep->lock);

            /* 睡眠，等待唤醒或超时 */
            if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS)) {
                timed_out = 1;  /* 超时 */
                break;
            }

            /* 被唤醒，重新加锁 */
            spin_lock_irq(&ep->lock);

            /* 检查是否有就绪事件 */
            eavail = ep_events_available(ep);
        }

        /* 从等待队列移除 */
        __remove_wait_queue(&ep->wq, &wait);

        /* 设置进程状态为运行 */
        __set_current_state(TASK_RUNNING);

        spin_unlock_irq(&ep->lock);
    }

send_events:
    /* 4. 有就绪事件，发送给用户空间 */
    if (eavail)
        res = ep_send_events(ep, events, maxevents);

    return res;
}
```

### 7.3 ep_send_events - 发送就绪事件

```c
static int ep_send_events(struct eventpoll *ep, struct epoll_event __user *events, int maxevents)
{
    struct ep_send_events_data esed;

    esed.maxevents = maxevents;
    esed.events = events;

    /* 扫描就绪列表 */
    ep_scan_ready_list(ep, ep_send_events_proc, &esed, 0, false);
    return esed.res;
}

static __poll_t ep_send_events_proc(struct eventpoll *ep, struct list_head *head, void *priv)
{
    struct ep_send_events_data *esed = priv;
    __poll_t revents;
    struct epitem *epi, *tmp;
    struct epoll_event __user *uevent = esed->events;
    int eventcnt = 0;

    /* 遍历就绪列表 */
    list_for_each_entry_safe(epi, tmp, head, rdllink) {
        /* 从就绪列表移除（稍后可能重新加入） */
        list_del_init(&epi->rdllink);

        /* 重新检查事件状态 */
        revents = ep_item_poll(epi, &pt, 1);

        /* 如果没有就绪事件，跳过 */
        if (!revents)
            continue;

        /* 拷贝事件到用户空间 */
        if (__put_user(revents, &uevent->events) ||
            __put_user(epi->event.data, &uevent->data)) {
            /* 拷贝失败，重新加入就绪列表 */
            list_add(&epi->rdllink, head);
            esed->res = eventcnt ? eventcnt : -EFAULT;
            return 0;
        }

        eventcnt++;
        uevent++;

        /* 如果是边缘触发或一次性事件，不重新加入就绪列表 */
        if (epi->event.events & EPOLLET) {
            /* ET 模式：事件已发送，不重新加入 */
        } else if (!(epi->event.events & EPOLLONESHOT)) {
            /* LT 模式：如果还有数据，重新加入就绪列表 */
            if (revents & epi->event.events)
                list_add_tail(&epi->rdllink, &ep->rdllist);
        }

        /* 达到 maxevents，退出 */
        if (eventcnt >= esed->maxevents)
            break;
    }

    esed->res = eventcnt;
    return 0;
}
```

### 7.4 epoll_wait 完整流程图

```
用户空间                                内核空间
─────────────────────────────────────────────────────────────────

epoll_wait(epfd=3, events, 10, -1)
    │
    │ 系统调用
    ▼
SYSCALL_DEFINE4(epoll_wait)
    │
    ▼
do_epoll_wait()
    │
    ├─→ 检查参数
    ├─→ fdget(epfd=3)              ← 获取epoll file
    ├─→ ep = f.file->private_data  ← 获取eventpoll
    │
    └─→ ep_poll(ep, events, 10, -1)
        │
        ├─→ 1. 快速路径：检查就绪列表
        │      eavail = ep_events_available(ep)
        │      ├─ spin_lock_irqsave(&ep->lock)
        │      ├─ eavail = !list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR
        │      └─ spin_unlock_irqrestore(&ep->lock)
        │
        │   Case A: 有就绪事件 (eavail = 1)
        │      └─→ goto send_events  ← 直接发送事件
        │
        │   Case B: 无就绪事件 (eavail = 0)
        │      └─→ 2. 慢速路径：阻塞等待
        │          │
        │          ├─ init_waitqueue_entry(&wait, current)
        │          ├─ spin_lock_irq(&ep->lock)
        │          ├─ __add_wait_queue_exclusive(&ep->wq, &wait)
        │          │  ← 将当前进程加入等待队列
        │          │
        │          ├─ while (!ep_events_available(ep)) {
        │          │      set_current_state(TASK_INTERRUPTIBLE);
        │          │      spin_unlock_irq(&ep->lock);
        │          │
        │          │      schedule_hrtimeout(timeout);  ← 睡眠
        │          │
        │          │      [ 进程睡眠，等待唤醒... ]
        │          │
        │          │      [ 数据到达 socket ]
        │          │      [ 网卡中断 → 协议栈 → wake_up(&sk->sk_wq) ]
        │          │      [ → ep_poll_callback() ]
        │          │      [ → list_add_tail(&epi->rdllink, &ep->rdllist) ]
        │          │      [ → wake_up(&ep->wq) ⚡ 唤醒我们！]
        │          │
        │          │      [ 进程被唤醒 ]
        │          │
        │          │      spin_lock_irq(&ep->lock);
        │          │      eavail = ep_events_available(ep);  ← 检查
        │          │  }
        │          │
        │          ├─ __remove_wait_queue(&ep->wq, &wait)
        │          ├─ __set_current_state(TASK_RUNNING)
        │          └─ spin_unlock_irq(&ep->lock)
        │
        └─→ 3. send_events:
            │
            └─ ep_send_events(ep, events, maxevents)
                │
                └─ ep_scan_ready_list(ep, ep_send_events_proc, ...)
                    │
                    ├─ spin_lock_irq(&ep->lock)
                    │
                    ├─ 将rdllist转移到本地链表
                    │  list_splice_init(&ep->rdllist, &txlist)
                    │
                    ├─ ep->ovflist = NULL
                    │  ← 临时存储新到达的事件
                    │
                    ├─ spin_unlock_irq(&ep->lock)
                    │
                    ├─ ep_send_events_proc(&txlist)
                    │  │
                    │  └─ list_for_each_entry_safe(epi, tmp, &txlist) {
                    │         ├─ revents = ep_item_poll(epi, &pt, 1)
                    │         │  ← 重新检查事件状态
                    │         │
                    │         ├─ if (!revents) continue;
                    │         │
                    │         ├─ __put_user(revents, &uevent->events)
                    │         ├─ __put_user(epi->event.data, &uevent->data)
                    │         │  ← 拷贝到用户空间
                    │         │
                    │         ├─ eventcnt++
                    │         │
                    │         └─ if (EPOLLET) {
                    │                /* ET 模式：不重新加入 */
                    │            } else {
                    │                /* LT 模式：还有数据则重新加入 */
                    │                if (revents & epi->event.events)
                    │                    list_add_tail(&epi->rdllink, &ep->rdllist);
                    │            }
                    │      }
                    │
                    ├─ spin_lock_irq(&ep->lock)
                    │
                    ├─ 处理ovflist（遍历期间到达的新事件）
                    │  for (epi = ep->ovflist; epi; epi = next) {
                    │      list_add_tail(&epi->rdllink, &ep->rdllist);
                    │  }
                    │  ep->ovflist = EP_UNACTIVE_PTR;
                    │
                    └─ spin_unlock_irq(&ep->lock)

    返回 eventcnt
    │
    ▼
用户空间得到 nfds = eventcnt
events[0] = {events: EPOLLIN, data.fd: 4}
events[1] = {events: EPOLLIN, data.fd: 5}
...
```

---

## 8. 事件通知机制

### 8.1 ep_poll_callback - 回调函数核心

**这是整个epoll最关键的函数！** 当文件描述符（如socket）上有事件发生时，内核会调用这个回调函数。

```c
static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
{
    int pwake = 0;
    struct epitem *epi = ep_item_from_wait(wait);
    struct eventpoll *ep = epi->ep;
    __poll_t pollflags = key_to_poll(key);
    int ewake = 0;

    /* 1. 加锁 */
    spin_lock_irqsave(&ep->lock, flags);

    /* 2. 检查事件类型是否匹配 */
    if (!(epi->event.events & ~EP_PRIVATE_BITS))
        goto out_unlock;

    /* 3. 检查事件是否在监控的事件中 */
    if (pollflags && !(pollflags & epi->event.events))
        goto out_unlock;

    /* 4. 将 epitem 加入就绪列表 ⚡ 关键步骤！*/
    if (!ep_is_linked(&epi->rdllink)) {
        list_add_tail(&epi->rdllink, &ep->rdllist);
        ep_pm_stay_awake_rcu(epi);
    }

    /* 5. 唤醒等待队列中的进程 ⚡ */
    if (waitqueue_active(&ep->wq)) {
        wake_up(&ep->wq);
        ewake = 1;
    }

    /* 6. 如果有 poll 等待，也唤醒 */
    if (waitqueue_active(&ep->poll_wait))
        pwake++;

out_unlock:
    spin_unlock_irqrestore(&ep->lock, flags);

    /* 7. 唤醒 poll 等待 */
    if (pwake)
        ep_poll_safewake(&ep->poll_wait);

    return ewake;
}
```

### 8.2 完整事件通知流程

```
1. 网络数据包到达
   ─────────────────────────────────────────────────────────

   网卡接收数据包
       │
       │ 网卡中断
       ▼
   网卡驱动处理
       │
       │ 将数据包传递给协议栈
       ▼
   TCP/IP协议栈处理
       │
       ├─ 解析IP头
       ├─ 解析TCP头
       ├─ 校验校验和
       ├─ 更新TCP状态机
       └─ 将数据拷贝到socket接收缓冲区
           │
           ▼
   socket (fd=4)
   ├─ sk_receive_queue ← 数据在这里
   └─ sk_wq (等待队列)
       │
       │ wake_up(&sk->sk_wq) 唤醒等待队列
       ▼


2. 调用等待队列上的回调函数
   ─────────────────────────────────────────────────────────

   wake_up(&sk->sk_wq)
       │
       ├─ 遍历sk_wq上的所有wait_queue_entry
       │
       └─ 调用每个entry的func回调函数
           │
           ▼
   wait_queue_entry
   ├─ func = ep_poll_callback  ← 这是我们注册的回调！
   ├─ private = epitem
   └─ ...
       │
       │ 调用func(wait, ...)
       ▼


3. ep_poll_callback()执行
   ─────────────────────────────────────────────────────────

   ep_poll_callback(wait, mode, sync, key)
       │
       ├─ epi = ep_item_from_wait(wait)   ← 获取 epitem
       ├─ ep = epi->ep                    ← 获取 eventpoll
       │
       ├─ spin_lock_irqsave(&ep->lock)    ← 加锁
       │
       ├─ 检查事件类型是否匹配
       │  pollflags = key_to_poll(key)
       │  if (!(pollflags & epi->event.events))
       │      goto out_unlock;
       │
       ├─ 将epitem加入就绪列表 ⚡
       │  if (!ep_is_linked(&epi->rdllink)) {
       │      list_add_tail(&epi->rdllink, &ep->rdllist);
       │  }
       │
       ├─ 唤醒等待的进程 ⚡
       │  if (waitqueue_active(&ep->wq)) {
       │      wake_up(&ep->wq);
       │  }
       │
       └─ spin_unlock_irqrestore(&ep->lock)
           │
           ▼


4. 唤醒阻塞在epoll_wait()的进程
   ─────────────────────────────────────────────────────────

   wake_up(&ep->wq)
       │
       ├─ 遍历ep->wq上的所有等待进程
       │
       └─ 唤醒进程
           │
           ▼
   进程从schedule_hrtimeout()返回
       │
       ├─ spin_lock_irq(&ep->lock)
       ├─ eavail = ep_events_available(ep)  ← 检查就绪列表
       │  └─ !list_empty(&ep->rdllist) = true 
       ├─ spin_unlock_irq(&ep->lock)
       │
       └─ 跳出while循环
           │
           ▼


5. epoll_wait() 返回就绪事件
   ─────────────────────────────────────────────────────────

   ep_send_events(ep, events, maxevents)
       │
       ├─ 遍历 rdllist
       │  for_each_entry_safe(epi, tmp, &rdllist) {
       │      ├─ revents = ep_item_poll(epi, &pt, 1)
       │      ├─ __put_user(revents, &uevent->events)
       │      └─ __put_user(epi->data, &uevent->data)
       │  }
       │
       └─ 返回 eventcnt
           │
           ▼
   返回用户空间
       │
       ▼
   用户程序继续执行
   for (int i = 0; i < nfds; i++) {
       if (events[i].events & EPOLLIN) {
           int fd = events[i].data.fd;
           read(fd, buf, sizeof(buf));  ← 读取数据
       }
   }
```

### 8.3 关键时间线

```
时间 T0: epoll_wait()调用，进程睡眠
  │
  │ 用户进程：epoll_wait(epfd, events, 10, -1)
  ├─→ 内核：ep->rdllist为空
  ├─→ 将进程加入ep->wq
  ├─→ schedule_hrtimeout(-1)
  └─→ 进程进入睡眠状态

      ┌──────────────────────────────────────┐
      │ 进程状态：TASK_INTERRUPTIBLE (睡眠)    │
      │ CPU：空闲，处理其他任务                 │
      └──────────────────────────────────────┘

时间 T1: 网络数据包到达 (T1 - T0 = 50ms)
  │
  │ 网卡：收到数据包
  ├─→ 网卡中断
  ├─→ 网卡驱动：netif_rx()
  ├─→ TCP/IP 协议栈：tcp_v4_rcv()
  ├─→ socket：数据拷贝到 sk_receive_queue
  └─→ wake_up(&sk->sk_wq) ⚡
      │
      └─→ ep_poll_callback() 被调用
          ├─ spin_lock(&ep->lock)
          ├─ list_add_tail(&epi->rdllink, &ep->rdllist)
          ├─ wake_up(&ep->wq) ⚡
          └─ spin_unlock(&ep->lock)
              │
              └─→ 唤醒阻塞在ep->wq上的进程

      ┌──────────────────────────────────────┐
      │ 进程状态：TASK_RUNNING (可运行)         │
      │ 等待 CPU 调度                         │
      └──────────────────────────────────────┘

时间 T2: 进程被调度运行 (T2 - T1 = 微秒级)
  │
  │ 调度器：选择进程运行
  ├─→ 进程从schedule_hrtimeout()返回
  ├─→ eavail = ep_events_available(ep) = true
  ├─→ 跳出while循环
  └─→ ep_send_events()
      ├─ 遍历rdllist
      ├─ 拷贝事件到用户空间
      └─ 返回nfds = 1
          │
          └─→ epoll_wait() 返回 1

时间 T3: 用户空间处理事件
  │
  │ nfds = epoll_wait(...) 返回 1
  ├─→ for (i = 0; i < nfds; i++)
  ├─→ events[0].events = EPOLLIN
  ├─→ events[0].data.fd = 4
  └─→ read(4, buf, sizeof(buf))
      │
      └─→ 读取数据，处理请求

总延迟：T3 - T0 = ~50ms (主要是等待数据到达)
内核处理延迟：T3 - T1 = 微秒级 (非常快！)
```

---

## 9. ET vs LT 模式

### 9.1 水平触发 (Level Triggered, LT)

**默认模式**，与select/poll行为一致。

**触发条件**：

- **只要文件描述符满足条件**（如可读、可写），每次调用epoll_wait都会返回该事件

**特点**：

```c
场景：socket接收缓冲区有100字节数据

第1次epoll_wait():
  ├─ 返回EPOLLIN事件
  └─ 用户读取50字节，缓冲区还剩50字节

第2次epoll_wait():
  ├─ 缓冲区还有50字节
  ├─ 返回EPOLLIN事件  ← LT模式会再次通知
  └─ 用户读取剩余50字节

第3次epoll_wait():
  ├─ 缓冲区为空
  └─ 不返回EPOLLIN事件 ← 直到新数据到达
```

**代码示例**：

```c
struct epoll_event ev;
ev.events = EPOLLIN;  // LT 模式（默认）
ev.data.fd = sockfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

while (1) {
    int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);

    for (int i = 0; i < nfds; i++) {
        if (events[i].events & EPOLLIN) {
            int fd = events[i].data.fd;

            // LT 模式：可以只读一部分数据
            char buf[100];
            int n = read(fd, buf, 100);  // 只读 100 字节

            // 如果缓冲区还有数据，下次 epoll_wait 会再次返回
        }
    }
}
```

**优点**：

- **编程简单**：不需要一次读完所有数据
- **不容易丢失事件**：即使没处理完，下次还会通知
- **与select/poll行为一致**：容易迁移

**缺点**：

- **可能触发多次**：如果不读完数据，每次epoll_wait都会返回
- **性能略低**：频繁返回相同的fd

### 9.2 边缘触发 (Edge Triggered, ET)

**高性能模式**，只在状态**变化**时通知一次。

**触发条件**：

- **只在文件描述符状态改变时**触发一次
- 从不可读变为可读、从不可写变为可写

**特点**：

```c
场景：socket接收缓冲区有100字节数据

第1次epoll_wait():
  ├─ 状态变化：从无数据  → 有数据
  ├─ 返回EPOLLIN事件
  └─ 用户读取50字节，缓冲区还剩50字节

第2次epoll_wait():
  ├─ 缓冲区还有50字节
  └─ 不返回EPOLLIN事件  ← ET模式不再通知

第3次epoll_wait():
  ├─ 新数据到达，缓冲区增加20字节（现在70字节）
  ├─ 状态变化：有新数据到达
  └─ 返回EPOLLIN事件
```

**代码示例**：

```c
struct epoll_event ev;
ev.events = EPOLLIN | EPOLLET;  // ET 模式
ev.data.fd = sockfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

// ET 模式必须配合非阻塞 I/O
int flags = fcntl(sockfd, F_GETFL, 0);
fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);

while (1) {
    int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);

    for (int i = 0; i < nfds; i++) {
        if (events[i].events & EPOLLIN) {
            int fd = events[i].data.fd;

            // ET 模式：必须一次读完所有数据！
            while (1) {
                char buf[4096];
                int n = read(fd, buf, sizeof(buf));

                if (n < 0) {
                    if (errno == EAGAIN || errno == EWOULDBLOCK) {
                        // 数据读完了
                        break;
                    } else {
                        // 真正的错误
                        perror("read");
                        break;
                    }
                } else if (n == 0) {
                    // 连接关闭
                    close(fd);
                    break;
                } else {
                    // 处理数据
                    process_data(buf, n);
                }
            }
        }
    }
}
```

**优点**：

- **高性能**：减少epoll_wait调用次数
- **减少事件触发**：每次状态变化只通知一次

**缺点**：

- **编程复杂**：必须一次读完所有数据
- **必须非阻塞I/O**：否则可能阻塞
- **容易丢失事件**：如果没处理完，不会再通知

### 9.3 LT vs ET 对比表

| 特性                 | LT (水平触发)  | ET (边缘触发)  |
|--------------------|------------|------------|
| **触发条件**           | 只要有数据就触发   | 状态改变时触发一次  |
| **触发次数**           | 多次（直到数据读完） | 一次（每次状态改变） |
| **编程难度**           | 简单         | 复杂         |
| **必须非阻塞I/O**       | 否          | 是          |
| **必须读完所有数据**       | 否          | 是          |
| **性能**             | 略低         | 高          |
| **适用场景**           | 通用场景       | 高性能服务器     |
| **与select/poll兼容** | 兼容         | 不兼容        |

### 9.4 内核实现差异

**LT模式**：

```c
// ep_send_events_proc() 中

for_each_entry_safe(epi, tmp, &txlist, rdllink) {
    revents = ep_item_poll(epi, &pt, 1);

    if (!revents)
        continue;

    /* 拷贝事件到用户空间 */
    __put_user(revents, &uevent->events);
    __put_user(epi->event.data, &uevent->data);

    /* LT 模式：如果还有数据，重新加入就绪列表 ⚡ */
    if (!(epi->event.events & EPOLLET)) {
        if (revents & epi->event.events)
            list_add_tail(&epi->rdllink, &ep->rdllist);  ← 重新加入！
    }
}
```

**ET 模式**：

```c
// ep_send_events_proc() 中

for_each_entry_safe(epi, tmp, &txlist, rdllink) {
    revents = ep_item_poll(epi, &pt, 1);

    if (!revents)
        continue;

    /* 拷贝事件到用户空间 */
    __put_user(revents, &uevent->events);
    __put_user(epi->event.data, &uevent->data);

    /* ET 模式：事件已发送，不重新加入 ⚡ */
    if (epi->event.events & EPOLLET) {
        /* 不重新加入就绪列表 */
        /* 直到下次 ep_poll_callback() 被调用（新数据到达）*/
    }
}
```

**关键区别**：

- **LT**: 如果fd还有数据，将epitem重新加入`rdllist`，下次epoll_wait会再次返回
- **ET**: 事件发送后，从`rdllist`移除，直到下次`ep_poll_callback()`被调用（新数据到达）

### 9.5 使用建议

**何时使用LT**：

1. 应用逻辑复杂，不方便一次处理完所有数据
2. 迁移select/poll代码，保持兼容性
3. 对性能要求不是极致
4. 初学者学习epoll

**何时使用ET**：

1. 高性能网络服务器（如Nginx）
2. 需要极致性能优化
3. 能够保证一次处理完所有数据
4. 对epoll和非阻塞I/O非常熟悉

**最佳实践**：

```c
// ET 模式最佳实践

// 1. 设置非阻塞
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);

// 2. 使用 ET 模式
struct epoll_event ev;
ev.events = EPOLLIN | EPOLLET;
ev.data.fd = fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

// 3. 循环读取直到 EAGAIN
while (1) 
{
    ssize_t n = read(fd, buf, sizeof(buf));

    if (n < 0) 
    {
        if (errno == EAGAIN) 
        {
            // 数据读完，可以退出
            break;
        }
        // 错误处理
    } 
    else if (n == 0) 
    {
        // EOF
        break;
    }

    // 处理数据
}
```

---

## 10. 性能分析

### 10.1 时间复杂度对比

| 操作                  | select/poll | epoll    |
|---------------------|-------------|----------|
| **添加/删除fd**         | O(1)        | O(log n) |
| **每次等待的fd集合拷贝**     | O(n)        | O(1)     |
| **检查就绪fd**          | O(n)        | O(1)     |
| **返回就绪fd**          | O(n)        | O(m)     |
| **总体性能（n个fd，m个就绪）** | O(n)        | O(m)     |

**说明**：

- `n`: 监控的文件描述符总数
- `m`: 就绪的文件描述符数量
- 通常`m << n`，所以epoll性能远优于select/poll

### 10.2 内存开销对比

```
场景：监控10,000个连接

select:
  ├─ fd_set 结构：1024 bits = 128 bytes
  │  （最多只能监控 1024个fd，无法满足需求）
  └─ 每次调用需要拷贝128 bytes到内核

poll:
  ├─ pollfd 数组：10,000 × sizeof(struct pollfd)
  │  = 10,000 × 8 bytes = 80 KB
  └─ 每次调用需要拷贝80 KB到内核

epoll:
  ├─ eventpoll 结构：~200 bytes
  ├─ 红黑树节点：10,000 × sizeof(struct epitem)
  │  ≈ 10,000 × 128 bytes = 1.25 MB
  ├─ 就绪列表：只包含就绪的fd（假设 100 个）
  │  = 100 × 指针 = 800 bytes
  └─ 每次调用无需拷贝

总结：
  - select: 无法支持 > 1024个连接
  - poll: 每次拷贝80 KB
  - epoll: 无需重复拷贝，内存使用更高效
```

### 10.3 性能基准测试

**测试环境**：

- CPU: Intel Xeon E5-2680 v4 @ 2.40GHz
- 内存: 64GB
- OS: Linux 5.10
- 场景: Echo 服务器

**测试结果**：

```
并发连接数：100
┌──────────┬──────────┬──────────┬──────────┐
│          │  select  │   poll   │  epoll   │
├──────────┼──────────┼──────────┼──────────┤
│ QPS      │  50,000  │  52,000  │  55,000  │
│ 延迟 (ms) │   2.0    │   1.9    │   1.8    │
│ CPU 使用  │   40%    │   38%    │   35%    │
└──────────┴──────────┴──────────┴──────────┘
差异不大（连接数少）


并发连接数：1,000
┌──────────┬──────────┬──────────┬──────────┐
│          │  select  │   poll   │  epoll   │
├──────────┼──────────┼──────────┼──────────┤
│ QPS      │  无法测试  │  45,000  │  54,000  │
│ 延迟 (ms) │    -     │   4.5    │   1.9    │
│ CPU 使用  │    -     │   75%    │   38%    │
└──────────┴──────────┴──────────┴──────────┘
epoll 开始显示优势


并发连接数：10,000
┌──────────┬──────────┬──────────┬──────────┐
│          │  select  │   poll   │  epoll   │
├──────────┼──────────┼──────────┼──────────┤
│ QPS      │  无法测试  │  12,000  │  53,000  │
│ 延迟 (ms) │    -     │   85     │   2.1    │
│ CPU 使用  │    -     │   98%    │   42%    │
└──────────┴──────────┴──────────┴──────────┘
epoll 性能碾压 poll（4.4 倍）


并发连接数：100,000
┌──────────┬──────────┬──────────┬──────────┐
│          │  select  │   poll   │  epoll   │
├──────────┼──────────┼──────────┼──────────┤
│ QPS      │  无法测试  │  无法测试  │  51,000  │
│ 延迟 (ms) │    -     │    -     │   2.3    │
│ CPU 使用  │    -     │    -     │   45%    │
└──────────┴──────────┴──────────┴──────────┘
只有 epoll 能支持！
```

**结论**：

- **< 100 连接**：三者性能接近，用哪个都行
- **100 ~ 1000 连接**：epoll 开始显示优势（20% 性能提升）
- **> 10,000 连接**：epoll 碾压 select/poll（4 倍以上性能）
- **> 100,000 连接**：只有 epoll 能支持

### 10.4 为什么epoll这么快？

**核心优化**：

1. **避免重复拷贝**
   ```
   select/poll: 每次调用都拷贝整个fd集合
     用户空间 → 内核空间: 拷贝n个fd
     内核空间 → 用户空间: 拷贝n个fd
     每次epoll_wait: 2n次拷贝

   epoll: fd集合在内核中维护
     epoll_ctl: 只拷贝1个fd（添加时）
     epoll_wait: 只拷贝m个就绪fd（m << n）
     每次epoll_wait: m次拷贝，m << n
   ```

2. **事件驱动，O(1)获取就绪fd**
   ```
   select/poll: 每次调用都遍历所有fd
     for (i = 0; i < n; i++) {
         if (fds[i] 就绪) { ... }  ← O(n)
     }

   epoll: fd就绪时，内核主动加入就绪列表
     当数据到达 → ep_poll_callback()
                 → list_add(&epi->rdllink, &ep->rdllist)
     epoll_wait()直接返回rdllist ← O(1)
   ```

3. **红黑树高效查找**
   ```
   查找fd是否已监控：
     select/poll: 线性查找O(n)
     epoll: 红黑树查找O(log n)
   ```

4. **只返回就绪的fd**
   ```
   select/poll: 返回所有fd，用户需要遍历
     for (i = 0; i < n; i++) {
         if (FD_ISSET(fds[i], &readfds)) { ... }  ← O(n)
     }

   epoll: 只返回就绪的fd
     for (i = 0; i < nfds; i++) {
         handle(events[i].data.fd);  ← O(m), m << n
     }
   ```

---

## 11. 完整代码示例

### 11.1 Echo 服务器 (LT模式)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <fcntl.h>

#define PORT 8080
#define MAX_EVENTS 100
#define BUFFER_SIZE 4096

int create_and_bind(int port) {
    int sockfd;
    struct sockaddr_in addr;

    /* 创建 socket */
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd == -1) {
        perror("socket");
        return -1;
    }

    /* 设置 SO_REUSEADDR */
    int opt = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    /* 绑定地址 */
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(port);

    if (bind(sockfd, (struct sockaddr*)&addr, sizeof(addr)) == -1) {
        perror("bind");
        close(sockfd);
        return -1;
    }

    /* 监听 */
    if (listen(sockfd, SOMAXCONN) == -1) {
        perror("listen");
        close(sockfd);
        return -1;
    }

    return sockfd;
}

void handle_accept(int epfd, int listen_fd) {
    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);

    int conn_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &client_len);
    if (conn_fd == -1) {
        perror("accept");
        return;
    }

    printf("New connection from %s:%d (fd=%d)\n",
           inet_ntoa(client_addr.sin_addr),
           ntohs(client_addr.sin_port),
           conn_fd);

    /* 添加到 epoll */
    struct epoll_event ev;
    ev.events = EPOLLIN;  // LT 模式
    ev.data.fd = conn_fd;

    if (epoll_ctl(epfd, EPOLL_CTL_ADD, conn_fd, &ev) == -1) {
        perror("epoll_ctl: conn_fd");
        close(conn_fd);
        return;
    }
}

void handle_read(int epfd, int fd) {
    char buf[BUFFER_SIZE];
    ssize_t n;

    n = read(fd, buf, sizeof(buf));

    if (n == -1) {
        perror("read");
        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
        close(fd);
        return;
    }

    if (n == 0) {
        /* 连接关闭 */
        printf("Connection closed (fd=%d)\n", fd);
        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
        close(fd);
        return;
    }

    /* Echo 回数据 */
    ssize_t written = write(fd, buf, n);
    if (written != n) {
        perror("write");
    }
}

int main() {
    int listen_fd, epfd;
    struct epoll_event ev, events[MAX_EVENTS];

    /* 创建监听 socket */
    listen_fd = create_and_bind(PORT);
    if (listen_fd == -1) {
        exit(EXIT_FAILURE);
    }

    printf("Server listening on port %d\n", PORT);

    /* 创建 epoll 实例 */
    epfd = epoll_create1(0);
    if (epfd == -1) {
        perror("epoll_create1");
        exit(EXIT_FAILURE);
    }

    /* 添加 listen_fd 到 epoll */
    ev.events = EPOLLIN;
    ev.data.fd = listen_fd;
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev) == -1) {
        perror("epoll_ctl: listen_fd");
        exit(EXIT_FAILURE);
    }

    /* 事件循环 */
    while (1) {
        int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        if (nfds == -1) {
            perror("epoll_wait");
            exit(EXIT_FAILURE);
        }

        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == listen_fd) {
                /* 新连接 */
                handle_accept(epfd, listen_fd);
            } else {
                /* 数据到达 */
                handle_read(epfd, events[i].data.fd);
            }
        }
    }

    close(listen_fd);
    close(epfd);
    return 0;
}
```

### 11.2 Echo 服务器 (ET模式 + 非阻塞 I/O)

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <fcntl.h>

#define PORT 8080
#define MAX_EVENTS 100
#define BUFFER_SIZE 4096

/* 设置非阻塞 */
int set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags == -1) {
        perror("fcntl F_GETFL");
        return -1;
    }

    if (fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1) {
        perror("fcntl F_SETFL");
        return -1;
    }

    return 0;
}

int create_and_bind(int port) {
    int sockfd;
    struct sockaddr_in addr;

    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd == -1) {
        perror("socket");
        return -1;
    }

    int opt = 1;
    setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(port);

    if (bind(sockfd, (struct sockaddr*)&addr, sizeof(addr)) == -1) {
        perror("bind");
        close(sockfd);
        return -1;
    }

    if (listen(sockfd, SOMAXCONN) == -1) {
        perror("listen");
        close(sockfd);
        return -1;
    }

    /* 设置非阻塞 */
    if (set_nonblocking(sockfd) == -1) {
        close(sockfd);
        return -1;
    }

    return sockfd;
}

void handle_accept(int epfd, int listen_fd) {
    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);

    /* ET 模式：循环 accept，直到 EAGAIN */
    while (1) {
        int conn_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &client_len);

        if (conn_fd == -1) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                /* 所有连接都处理完了 */
                break;
            } else {
                perror("accept");
                break;
            }
        }

        printf("New connection from %s:%d (fd=%d)\n",
               inet_ntoa(client_addr.sin_addr),
               ntohs(client_addr.sin_port),
               conn_fd);

        /* 设置非阻塞 */
        if (set_nonblocking(conn_fd) == -1) {
            close(conn_fd);
            continue;
        }

        /* 添加到 epoll (ET 模式) */
        struct epoll_event ev;
        ev.events = EPOLLIN | EPOLLET;  // ET 模式
        ev.data.fd = conn_fd;

        if (epoll_ctl(epfd, EPOLL_CTL_ADD, conn_fd, &ev) == -1) {
            perror("epoll_ctl: conn_fd");
            close(conn_fd);
            continue;
        }
    }
}

void handle_read(int epfd, int fd) {
    char buf[BUFFER_SIZE];
    ssize_t n;

    /* ET 模式：循环读取，直到 EAGAIN */
    while (1) {
        n = read(fd, buf, sizeof(buf));

        if (n == -1) {
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                /* 数据读完了 */
                break;
            } else {
                perror("read");
                epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                close(fd);
                return;
            }
        }

        if (n == 0) {
            /* 连接关闭 */
            printf("Connection closed (fd=%d)\n", fd);
            epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
            close(fd);
            return;
        }

        /* Echo 回数据 */
        ssize_t written = 0;
        while (written < n) {
            ssize_t w = write(fd, buf + written, n - written);
            if (w == -1) {
                if (errno == EAGAIN || errno == EWOULDBLOCK) {
                    /* 写缓冲区满了，稍后再写 */
                    /* 这里简化处理，实际应该注册 EPOLLOUT 事件 */
                    break;
                } else {
                    perror("write");
                    epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                    close(fd);
                    return;
                }
            }
            written += w;
        }
    }
}

int main() {
    int listen_fd, epfd;
    struct epoll_event ev, events[MAX_EVENTS];

    listen_fd = create_and_bind(PORT);
    if (listen_fd == -1) {
        exit(EXIT_FAILURE);
    }

    printf("Server listening on port %d (ET mode)\n", PORT);

    epfd = epoll_create1(0);
    if (epfd == -1) {
        perror("epoll_create1");
        exit(EXIT_FAILURE);
    }

    /* 添加 listen_fd (ET 模式) */
    ev.events = EPOLLIN | EPOLLET;
    ev.data.fd = listen_fd;
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev) == -1) {
        perror("epoll_ctl: listen_fd");
        exit(EXIT_FAILURE);
    }

    while (1) {
        int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        if (nfds == -1) {
            perror("epoll_wait");
            exit(EXIT_FAILURE);
        }

        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == listen_fd) {
                handle_accept(epfd, listen_fd);
            } else {
                handle_read(epfd, events[i].data.fd);
            }
        }
    }

    close(listen_fd);
    close(epfd);
    return 0;
}
```

### 11.3 编译和测试

```bash
# 编译 LT 模式服务器
gcc -o echo_server_lt echo_server_lt.c -Wall

# 编译 ET 模式服务器
gcc -o echo_server_et echo_server_et.c -Wall

# 运行服务器
./echo_server_lt
# 或
./echo_server_et

# 测试 (另一个终端)
telnet localhost 8080
# 输入文字，服务器会 echo 回来
```

---

## 12. 最佳实践

### 12.1 使用建议

1. **选择合适的触发模式**
   ```c
   // 通用场景：使用 LT 模式
   ev.events = EPOLLIN;

   // 高性能场景：使用 ET 模式 + 非阻塞 I/O
   ev.events = EPOLLIN | EPOLLET;
   fcntl(fd, F_SETFL, O_NONBLOCK);
   ```

2. **ET 模式必须循环读取**
   ```c
   while (1) {
       ssize_t n = read(fd, buf, sizeof(buf));
       if (n < 0) {
           if (errno == EAGAIN) break;  // 读完了
           // 错误处理
       }
       if (n == 0) break;  // EOF
       // 处理数据
   }
   ```

3. **处理 EAGAIN/EWOULDBLOCK**
   ```c
   if (errno == EAGAIN || errno == EWOULDBLOCK) {
       // 这不是错误，数据读完了
       return;
   }
   ```

4. **处理 EINTR**
   ```c
   int nfds = epoll_wait(epfd, events, MAX_EVENTS, timeout);
   if (nfds == -1) {
       if (errno == EINTR) {
           // 被信号中断，重试
           continue;
       }
       perror("epoll_wait");
   }
   ```

5. **合理设置 maxevents**
   ```c
   // 不要太小（浪费调用次数）
   // 不要太大（浪费内存）
   #define MAX_EVENTS 100  // 通常 100-1000 之间
   ```

6. **使用 EPOLLRDHUP 检测对端关闭**
   ```c
   ev.events = EPOLLIN | EPOLLRDHUP;

   if (events[i].events & EPOLLRDHUP) {
       // 对端关闭了连接
       close(fd);
   }
   ```

### 12.2 常见陷阱

1. **忘记删除已关闭的fd**
   ```c
   // 错误：fd关闭后没有从epoll删除
   close(fd);

   // 正确：先删除，再关闭
   epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
   close(fd);
   ```

2. **ET 模式没有读完数据**
   ```c
   // 错误：ET模式只读一次
   read(fd, buf, sizeof(buf));

   // 正确：ET模式循环读取
   while (1) {
       ssize_t n = read(fd, buf, sizeof(buf));
       if (n < 0 && errno == EAGAIN) break;
       // 处理...
   }
   ```

3. **ET 模式使用阻塞I/O**
   ```c
   // 错误：ET模式 + 阻塞 I/O = 可能永久阻塞
   ev.events = EPOLLIN | EPOLLET;
   // fd 是阻塞的

   // 正确：ET模式必须配合非阻塞 I/O
   fcntl(fd, F_SETFL, O_NONBLOCK);
   ev.events = EPOLLIN | EPOLLET;
   ```

4. **忘记处理EPOLLHUP/EPOLLERR**
   ```c
   // 正确：检查所有可能的事件
   if (events[i].events & EPOLLERR) {
       // 错误发生
   }
   if (events[i].events & EPOLLHUP) {
       // 挂断（对方关闭连接）
   }
   if (events[i].events & EPOLLIN) {
       // 可读
   }
   ```

5. **在多线程中共享epoll fd**
   ```c
   // 需要小心：多线程共享 epoll 需要额外同步
   // 建议：每个线程使用独立的 epoll 实例
   ```

### 12.3 性能优化技巧

1. **使用EPOLLONESHOT避免惊群**
   ```c
   // 多线程场景：防止多个线程同时处理同一个事件
   ev.events = EPOLLIN | EPOLLONESHOT;
   ```

2. **批量处理事件**
   ```c
   // 一次 epoll_wait 处理多个事件，减少系统调用
   #define MAX_EVENTS 100
   struct epoll_event events[MAX_EVENTS];
   int nfds = epoll_wait(epfd, events, MAX_EVENTS, timeout);
   ```

3. **合理设置 timeout**
   ```c
   // 纯事件驱动：timeout = -1（永久阻塞）
   epoll_wait(epfd, events, MAX_EVENTS, -1);

   // 需要定时任务：timeout = 最小定时器间隔
   epoll_wait(epfd, events, MAX_EVENTS, 1000);  // 1 秒
   ```

4. **减少系统调用**
   ```c
   // 批量添加 fd
   for (int i = 0; i < n; i++) {
       epoll_ctl(epfd, EPOLL_CTL_ADD, fds[i], &ev);
   }
   ```

### 12.4 调试技巧

1. **打印epoll事件**
   ```c
   void print_events(uint32_t events) {
       printf("Events:");
       if (events & EPOLLIN)  printf(" EPOLLIN");
       if (events & EPOLLOUT) printf(" EPOLLOUT");
       if (events & EPOLLERR) printf(" EPOLLERR");
       if (events & EPOLLHUP) printf(" EPOLLHUP");
       if (events & EPOLLET)  printf(" EPOLLET");
       printf("\n");
   }
   ```

2. **检查errno**
   ```c
   if (epoll_wait(...) == -1) {
       printf("epoll_wait error: %s\n", strerror(errno));
   }
   ```

3. **使用strace跟踪系统调用**
   ```bash
   strace -e epoll_create,epoll_ctl,epoll_wait ./your_program
   ```

4. **监控 /proc/[pid]/fdinfo/[epfd]**
   ```bash
   # 查看 epoll 实例的信息
   cat /proc/$(pidof your_program)/fdinfo/3

   # 输出：
   # pos:    0
   # flags:  02
   # mnt_id: 13
   # tfd:        4 events:       19 data: 0000000000000004
   # tfd:        5 events:       19 data: 0000000000000005
   ```

---

## 总结

### epoll核心要点

1. **三个 API**：`epoll_create1`, `epoll_ctl`, `epoll_wait`

2. **核心数据结构**：
    - `eventpoll`: epoll 实例
    - 红黑树: 存储所有监控的 fd
    - 就绪列表: 存储就绪的 fd
    - 等待队列: 阻塞的进程

3. **事件通知机制**：
    - 注册回调函数到 fd 的等待队列
    - 事件发生时，回调函数被调用
    - 回调函数将 fd 加入就绪列表
    - 唤醒阻塞的进程

4. **ET vs LT**：
    - LT: 只要有数据就触发（默认）
    - ET: 状态改变时触发一次（高性能）

5. **性能优势**：
    - 避免重复拷贝 fd 集合
    - O(1) 获取就绪 fd
    - 支持海量并发连接

### 适用场景

- ✅ 高性能网络服务器（Nginx, Redis）
- ✅ C10K/C100K 问题
- ✅ 需要监控大量文件描述符
- ❌ 跨平台应用（epoll 仅 Linux）
- ❌ 少量连接（< 100）

---

**参考资料**：

- Linux Kernel Source: `fs/eventpoll.c`
- Man Pages: `epoll(7)`, `epoll_create(2)`, `epoll_ctl(2)`, `epoll_wait(2)`
- The C10K problem: http://www.kegel.com/c10k.html

---

## 13. 内核事件检测机制：EPOLLIN 和 EPOLLOUT 的底层原理

> 深入理解内核如何知道 socket 可读和可写状态


> 深入理解内核如何检测 socket 可读和可写状态

---

#### 本章目录

1. [EPOLLIN：内核如何知道数据到了？](#1-epollin内核如何知道数据到了)
2. [EPOLLOUT：内核如何知道 fd 可写？](#2-epollout内核如何知道-fd-可写)
3. [关键数据结构](#3-关键数据结构)
4. [完整流程对比](#4-完整流程对比)

---

#### 1. EPOLLIN：内核如何知道数据到了？

#### 1.1 硬件层面：网卡接收数据

```
物理层面的数据接收流程：

1. 网线上的电信号
   ├─ 数字信号通过网线传输
   └─ 到达网卡的物理端口

2. 网卡硬件处理
   ├─ 网卡芯片接收比特流
   ├─ 解析以太网帧
   ├─ 检查目标MAC地址（是否是本机）
   └─ 如果是本机的数据包，继续处理

3. DMA (Direct Memory Access) 传输
   ├─ 网卡通过DMA直接将数据写入内核内存
   ├─ 不需要CPU参与拷贝（高效！）
   └─ 数据写入到预先分配的DMA缓冲区

4. 硬件中断
   ├─ 网卡写完数据后，产生硬件中断
   ├─ 通过IRQ (Interrupt Request) 通知CPU
   └─ CPU收到中断信号
```

**关键点**：网卡**主动**通知CPU，而不是CPU轮询！

#### 1.2 中断处理：网卡驱动

```c
// 网卡驱动的中断处理函数（简化版）

// 硬件中断处理函数（快速执行，不能睡眠）
irqreturn_t network_card_interrupt_handler(int irq, void *dev_id)
{
    struct net_device *dev = dev_id;

    // 1. 关闭网卡中断（避免中断风暴）
    disable_irq_nosync(irq);

    // 2. 触发软中断（NET_RX_SOFTIRQ）
    //    将实际处理工作推迟到软中断中执行
    napi_schedule(&dev->napi);

    return IRQ_HANDLED;
}
```

**为什么要分成硬中断和软中断？**

```
硬中断（Hard IRQ）:
  ┌─────────────────────────────────────────┐
  │ 特点：                                   │
  │ - 优先级最高，立即执行                     │
  │ - 不能睡眠，不能阻塞                       │
  │ - 执行时间要非常短（微秒级）                │
  │ - 会阻塞其他中断                          │
  │                                         │
  │ 任务：                                   │
  │ - 确认中断                               │
  │ - 触发软中断                             │
  │ - 立即返回                               │
  └─────────────────────────────────────────┘

软中断（Soft IRQ）:
  ┌─────────────────────────────────────────┐
  │ 特点：                                   │
  │ - 优先级低于硬中断                         │
  │ - 可以被硬中断打断                         │
  │ - 执行时间可以稍长                         │
  │ - 在ksoftirqd内核线程中执行              │
  │                                         │
  │ 任务：                                   │
  │ - 数据包的协议栈处理                       │
  │ - IP、TCP 层处理                         │
  │ - 拷贝数据到socket缓冲区                 │
  └─────────────────────────────────────────┘

设计理由：
  硬中断必须快速完成，不能占用太长时间
  所以将耗时的工作（协议栈处理）推迟到软中断中执行
```

#### 1.3 软中断处理：协议栈

```c
// 软中断处理网络数据包（简化版）

// NET_RX_SOFTIRQ 软中断处理函数
static void net_rx_action(struct softirq_action *h)
{
    struct sk_buff *skb;

    // 从DMA缓冲区取出数据包
    while ((skb = get_next_packet()) != NULL) {
        // 进入协议栈处理
        netif_receive_skb(skb);
    }
}

int netif_receive_skb(struct sk_buff *skb)
{
    // 1. 链路层处理（以太网）
    //    - 解析以太网头
    //    - 去掉以太网头
    ethernet_type_trans(skb, dev);

    // 2. 网络层处理（IP）
    //    - 解析IP头
    //    - 校验IP校验和
    //    - 检查TTL
    //    - 路由查找
    ip_rcv(skb);

    // 3. 传输层处理（TCP/UDP）
    //    - 解析TCP头
    //    - 校验TCP校验和
    //    - 检查序列号
    //    - 更新滑动窗口
    tcp_v4_rcv(skb);

    return 0;
}
```

#### 1.4 TCP层处理：拷贝数据到socket

```c
// TCP 接收数据包（简化版）

int tcp_v4_rcv(struct sk_buff *skb)
{
    struct tcphdr *th;
    struct sock *sk;

    // 1. 解析 TCP 头
    th = tcp_hdr(skb);

    // 2. 根据四元组查找对应的 socket
    //    (源IP, 源端口, 目的IP, 目的端口)
    sk = __inet_lookup_skb(&tcp_hashinfo, skb, th->source, th->dest);

    if (!sk) {
        // 没找到对应的 socket，丢弃数据包
        return 0;
    }

    // 3. 根据 TCP 状态处理
    switch (sk->sk_state) {
    case TCP_ESTABLISHED:
        tcp_rcv_established(sk, skb);  // ← 这里！
        break;
    case TCP_LISTEN:
        tcp_v4_do_rcv(sk, skb);
        break;
    // ...
    }

    return 0;
}

void tcp_rcv_established(struct sock *sk, struct sk_buff *skb)
{
    struct tcphdr *th = tcp_hdr(skb);

    // 1. 检查序列号是否在窗口内
    if (!tcp_sequence(tp, TCP_SKB_CB(skb)->seq, TCP_SKB_CB(skb)->end_seq)) {
        // 序列号不对，丢弃
        return;
    }

    // 2. 更新接收窗口
    tcp_data_queue(sk, skb);

    // 3. 发送 ACK
    tcp_send_ack(sk);

    // 4. 唤醒等待的进程 ⚡ 关键！
    sk->sk_data_ready(sk);  // 这会调用 sock_def_readable()
}
```

#### 1.5 唤醒机制：从socket到epoll

```c
// socket 数据就绪回调

void sock_def_readable(struct sock *sk)
{
    struct socket_wq *wq;

    rcu_read_lock();
    wq = rcu_dereference(sk->sk_wq);

    // 检查是否有进程在等待
    if (skwq_has_sleeper(wq)) {
        // 唤醒等待队列上的所有进程/回调 ⚡
        wake_up_interruptible_sync_poll(
            &wq->wait,           // 等待队列
            POLLIN | POLLRDNORM  // 事件类型
        );
    }

    rcu_read_unlock();

    // 通知 socket 层的回调
    sk_wake_async(sk, SOCK_WAKE_WAITD, POLL_IN);
}
```

**关键：wake_up做了什么？**

```c
void __wake_up_common(struct wait_queue_head *wq_head, unsigned int mode, int nr_exclusive, int wake_flags, void *key)
{
    struct wait_queue_entry *curr, *next;

    // 遍历等待队列上的所有等待项
    list_for_each_entry_safe(curr, next, &wq_head->head, entry) {
        unsigned flags = curr->flags;

        // 调用每个等待项的回调函数 ⚡
        int ret = curr->func(curr, mode, wake_flags, key);

        // curr->func 就是 ep_poll_callback！
    }
}
```

#### 1.6 epoll 回调：ep_poll_callback

```c
// epoll 的回调函数（在 socket 数据到达时被调用）

static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
{
    int pwake = 0;
    struct epitem *epi = ep_item_from_wait(wait);
    struct eventpoll *ep = epi->ep;
    __poll_t pollflags = key_to_poll(key);

    // 1. 加锁
    spin_lock_irqsave(&ep->lock, flags);

    // 2. 检查事件类型是否匹配
    //    pollflags 是 POLLIN（来自 socket）
    //    epi->event.events 是用户注册的事件（如 EPOLLIN）
    if (!(pollflags & epi->event.events))
        goto out_unlock;

    // 3. 将 epitem 加入就绪列表 ⚡ 关键步骤！
    if (!ep_is_linked(&epi->rdllink)) {
        list_add_tail(&epi->rdllink, &ep->rdllist);
        ep_pm_stay_awake(epi);
    }

    // 4. 唤醒阻塞在 epoll_wait 的进程 ⚡
    if (waitqueue_active(&ep->wq)) {
        wake_up(&ep->wq);
    }

out_unlock:
    spin_unlock_irqrestore(&ep->lock, flags);

    return 1;
}
```

#### 1.7 完整流程图

```
═══════════════════════════════════════════════════════════════
                    EPOLLIN 完整触发流程
═══════════════════════════════════════════════════════════════

时间 T0: 网络数据包在传输中
  │
  │ 物理介质（网线、光纤）
  ▼
───────────────────────────────────────────────────────────────
硬件层
───────────────────────────────────────────────────────────────
T1: 网卡接收数据包
  ├─ 网卡芯片接收比特流
  ├─ 解析以太网帧
  ├─ 检查目标MAC地址
  ├─ DMA拷贝数据到内核内存(无CPU参与)
  └─ 产生硬件中断(IRQ)
      │
      └─→ CPU收到中断信号

───────────────────────────────────────────────────────────────
硬中断处理（微秒级，必须极快）
───────────────────────────────────────────────────────────────
T2: 网卡驱动中断处理函数
  ├─ network_card_interrupt_handler()
  ├─ disable_irq_nosync(irq)   // 禁用中断
  ├─ napi_schedule()           // 触发软中断
  └─ return IRQ_HANDLED

───────────────────────────────────────────────────────────────
软中断处理（毫秒级，可以稍慢）
───────────────────────────────────────────────────────────────
T3: 协议栈处理（在ksoftirqd内核线程中执行）
  │
  ├─→ net_rx_action()
  │   └─ netif_receive_skb()
  │
  ├─→ 链路层处理
  │   └─ ethernet_type_trans()
  │       ├─ 解析以太网头
  │       └─ 去掉以太网头
  │
  ├─→ 网络层处理
  │   └─ ip_rcv()
  │       ├─ 解析IP头
  │       ├─ 校验IP校验和
  │       ├─ 检查TTL
  │       └─ 路由查找
  │
  ├─→ 传输层处理
  │   └─ tcp_v4_rcv()
  │       ├─ 解析 TCP 头
  │       ├─ 根据四元组查找 socket
  │       │   (源IP:源端口 → 目的IP:目的端口)
  │       │
  │       └─ tcp_rcv_established()
  │           ├─ 校验序列号
  │           ├─ 更新接收窗口
  │           ├─ tcp_data_queue()
  │           │   └─ __skb_queue_tail(&sk->sk_receive_queue, skb)
  │           │       ← 数据拷贝到socket接收缓冲区
  │           │
  │           ├─ tcp_send_ack()  // 发送ACK给对方
  │           │
  │           └─ sk->sk_data_ready(sk)  ⚡ 触发回调！
  │
  └─→ sock_def_readable()
      └─ wake_up_interruptible_sync_poll(&sk->sk_wq, POLLIN)

───────────────────────────────────────────────────────────────
唤醒机制
───────────────────────────────────────────────────────────────
T4: 唤醒等待队列
  │
  ├─→ __wake_up_common()
  │   └─ 遍历 sk->sk_wq 上的所有 wait_queue_entry
  │       │
  │       ├─ wait_entry_1->func()  // 可能是 select
  │       ├─ wait_entry_2->func()  // 可能是 poll
  │       └─ wait_entry_3->func()  // epoll 的回调 ⚡
  │           │
  │           └─→ ep_poll_callback()

───────────────────────────────────────────────────────────────
epoll 核心处理
───────────────────────────────────────────────────────────────
T5: ep_poll_callback()执行
  │
  ├─ spin_lock(&ep->lock)
  │
  ├─ 检查事件类型
  │  if (pollflags & epi->event.events) {  // POLLIN & EPOLLIN
  │      // 匹配
  │  }
  │
  ├─ 将 epitem 加入就绪列表 ⚡
  │  if (!ep_is_linked(&epi->rdllink)) {
  │      list_add_tail(&epi->rdllink, &ep->rdllist);
  │  }
  │
  ├─ 唤醒阻塞在 epoll_wait 的进程
  │  if (waitqueue_active(&ep->wq)) {
  │      wake_up(&ep->wq);
  │  }
  │
  └─ spin_unlock(&ep->lock)

───────────────────────────────────────────────────────────────
用户空间返回
───────────────────────────────────────────────────────────────
T6: epoll_wait() 返回
  │
  ├─ 进程从睡眠中被唤醒
  ├─ ep_send_events()遍历就绪列表
  ├─ 拷贝就绪事件到用户空间
  │   events[0].events = EPOLLIN
  │   events[0].data.fd = sockfd
  │
  └─ 返回就绪事件数量

T7: 用户程序处理
  │
  └─→ read(sockfd, buf, size)
      └─ 从 sk_receive_queue 读取数据

═══════════════════════════════════════════════════════════════
总延迟：T7 - T0 = 数据传输时间 + 几百微秒（内核处理）
═══════════════════════════════════════════════════════════════
```

---

#### 2. EPOLLOUT：内核如何知道fd可写？

#### 2.1 什么叫"可写"？

```
可写的定义：
┌─────────────────────────────────────────────────────────┐
│ socket 的发送缓冲区有足够的可用空间，                        │
│ 可以容纳至少 1 字节的数据而不阻塞                            │
└─────────────────────────────────────────────────────────┘

数学表示：
    sk_sndbuf - sk_wmem_queued >= lowat

    其中：
    - sk_sndbuf: 发送缓冲区总大小（默认 16KB - 4MB）
    - sk_wmem_queued: 已使用的缓冲区大小
    - lowat: 低水位标记（默认 1 字节）
```

**发送缓冲区的作用**：

```
为什么需要发送缓冲区？
┌─────────────────────────────────────────────────────────┐
│ 问题：网络传输速度慢，不能让用户程序一直等待                   │
│                                                         │
│ 解决方案：                                               │
│ 1. 用户调用write()                                       │
│ 2. 内核将数据拷贝到发送缓冲区                               │
│ 3. write()立即返回（不等待网络传输）                        │
│ 4. 内核后台慢慢发送数据                                    │
│                                                         │
│ 好处：                                                   │
│ - 用户程序不阻塞                                          │
│ - 异步发送，提高效率                                       │
│ - 处理网络拥塞、对方接收慢等情况                             │
└─────────────────────────────────────────────────────────┘
```

#### 2.2 发送缓冲区的状态变化

```c
// socket 发送缓冲区结构（简化）

struct sock {
    // 发送队列：存储待发送和已发送未确认的数据
    struct sk_buff_head sk_write_queue;

    // 发送缓冲区总大小（字节）
    int sk_sndbuf;  // 默认：16KB ~ 4MB（可调整）

    // 已使用的缓冲区大小（字节）
    atomic_t sk_wmem_queued;

    // 低水位标记（可写的阈值）
    int sk_sndlowat;  // 默认：1 字节

    // 等待队列（epoll 注册回调在这里）
    struct socket_wq *sk_wq;
};
```

**状态变化时间线**：

```
═══════════════════════════════════════════════════════════════
           发送缓冲区状态变化（触发 EPOLLOUT）
═══════════════════════════════════════════════════════════════

状态 A: 缓冲区有大量空间（初始状态）
┌─────────────────────────────────────────────────────────┐
│ sk_write_queue:                                         │
│ [ 已发送待ACK: 2KB ]        [ 可用空间: 14KB ]             │
│                                                         │
│ sk_sndbuf = 16KB                                        │
│ sk_wmem_queued = 2KB                                    │
│ 可用 = 16KB - 2KB = 14KB                                 │
│                                                         │
│ 判断：14KB >= 1 字节 → 可写                               │
│ EPOLLOUT: 会触发                                         │
└─────────────────────────────────────────────────────────┘
      │
      │ 用户大量 write()
      ▼
───────────────────────────────────────────────────────────────

状态 B: 缓冲区快满了
┌─────────────────────────────────────────────────────────┐
│ sk_write_queue:                                         │
│ [待发送][待发送][待发送][已发送待ACK][已发送待ACK]            │
│                                         [可用: 500B]     │
│                                                         │
│ sk_sndbuf = 16KB                                        │
│ sk_wmem_queued = 15.5KB                                 │
│ 可用 = 16KB - 15.5KB = 500B                              │
│                                                         │
│ 判断：500B >= 1 字节 → 可写                               │
│ EPOLLOUT: 还会触发                                       │
└─────────────────────────────────────────────────────────┘
      │
      │ 继续 write()，缓冲区满
      ▼
───────────────────────────────────────────────────────────────

状态 C: 缓冲区满了（罕见）
┌─────────────────────────────────────────────────────────┐
│ sk_write_queue:                                         │
│ [待发送][待发送][待发送][待发送][已发送待ACK]...             │
│                                            [可用: 0]     │
│                                                         │
│ sk_sndbuf = 16KB                                        │
│ sk_wmem_queued = 16KB                                   │
│ 可用 = 16KB - 16KB = 0                                   │
│                                                         │
│ 判断：0 < 1 字节 → 不可写                                  │
│ write(): 返回 EAGAIN（非阻塞）或阻塞（阻塞 socket）          │
│ EPOLLOUT: 不触发                                         │
└─────────────────────────────────────────────────────────┘
      │
      │ 网络传输数据，收到对方 ACK
      ▼
───────────────────────────────────────────────────────────────

状态 D: 收到 ACK，释放空间 ⚡
┌─────────────────────────────────────────────────────────┐
│ sk_write_queue:                                         │
│ [待发送][待发送]                [ 可用空间: 8KB ]          │
│                                                         │
│ sk_sndbuf = 16KB                                        │
│ sk_wmem_queued = 8KB (释放了 8KB！)                      │
│ 可用 = 16KB - 8KB = 8KB                                  │
│                                                         │
│ 状态变化：不可写 → 可写 ⚡ 关键！                            │
│                                                         │
│ 触发：sk_stream_write_space()                            │
│   └─→ wake_up(&sk->sk_wq)                               │
│       └─→ ep_poll_callback()                            │
│           └─→ EPOLLOUT 触发                              │
└─────────────────────────────────────────────────────────┘
```

#### 2.3 收到 ACK 的处理流程

```c
// TCP 收到 ACK 包的处理（简化版）

int tcp_ack(struct sock *sk, const struct sk_buff *skb, int flag)
{
    struct tcp_sock *tp = tcp_sk(sk);
    u32 ack = TCP_SKB_CB(skb)->ack_seq;
    u32 prior_snd_una = tp->snd_una;

    // 1. 检查 ACK 序列号是否有效
    if (before(ack, prior_snd_una)) {
        // ACK 太老，丢弃
        return 0;
    }

    if (after(ack, tp->snd_nxt)) {
        // ACK 超前，可能是错误
        return -1;
    }

    // 2. 更新发送窗口
    tp->snd_una = ack;

    // 3. 清理发送队列中已确认的数据 ⚡
    tcp_clean_rtx_queue(sk, flag);

    // 4. 更新拥塞窗口
    tcp_cong_avoid(sk, ack, flag);

    return 1;
}

// 清理已确认的数据，释放缓冲区
static int tcp_clean_rtx_queue(struct sock *sk, int prior_fackets)
{
    struct tcp_sock *tp = tcp_sk(sk);
    struct sk_buff *skb, *next;
    u32 now = tcp_time_stamp(tp);
    int fully_acked = true;
    long ca_seq_rtt_us = -1L;

    // 遍历发送队列
    skb_queue_walk_safe(&sk->sk_write_queue, skb, next) {
        u32 end_seq = TCP_SKB_CB(skb)->end_seq;

        // 检查这个数据包是否已被确认
        if (after(end_seq, tp->snd_una)) {
            // 还没被确认，停止
            break;
        }

        // 已确认，从队列删除 ⚡
        tcp_unlink_write_queue(skb, sk);

        // 释放内存，减少 sk_wmem_queued ⚡
        sk_wmem_free_skb(sk, skb);
        // 内部执行：sk->sk_wmem_queued -= skb->truesize;

        // 更新统计信息
        tcp_ack_update_rtt(sk, flag, seq_rtt_us);
    }

    // 如果释放了空间，检查是否需要唤醒等待的进程 ⚡
    if (sk_stream_wspace(sk) >= sk_stream_min_wspace(sk)) {
        // 缓冲区有足够空间了，触发回调！
        sk->sk_write_space(sk);
    }

    return 0;
}
```

#### 2.4 可写检查和回调触发

```c
// 检查 socket 是否可写

static inline bool sk_stream_is_writeable(const struct sock *sk)
{
    // 计算可用空间
    int available = sk_stream_wspace(sk);

    // 检查是否大于低水位标记
    return available >= sk_stream_min_wspace(sk);
}

// 计算可用空间
static inline int sk_stream_wspace(const struct sock *sk)
{
    return sk->sk_sndbuf - sk->sk_wmem_queued;
}

// 低水位标记（默认 1 字节）
static inline int sk_stream_min_wspace(const struct sock *sk)
{
    return sk->sk_wmem_queued >> 1;  // 或者 sk->sk_sndlowat
}
```

**触发 EPOLLOUT 的回调**：

```c
// socket 可写时的回调函数

void sk_stream_write_space(struct sock *sk)
{
    struct socket_wq *wq;

    // 检查是否真的可写
    if (sk_stream_is_writeable(sk) && sock_writeable(sk)) {
        rcu_read_lock();
        wq = rcu_dereference(sk->sk_wq);

        // 检查是否有进程在等待
        if (skwq_has_sleeper(wq)) {
            // 唤醒等待队列 ⚡
            wake_up_interruptible_sync_poll(
                &wq->wait,            // 等待队列
                POLLOUT | POLLWRNORM  // 事件类型
            );
        }

        rcu_read_unlock();

        // 通知 socket 层回调
        sk_wake_async(sk, SOCK_WAKE_SPACE, POLL_OUT);
    }
}
```

#### 2.5 EPOLLOUT 完整流程图

```
═══════════════════════════════════════════════════════════════
                   EPOLLOUT 完整触发流程
═══════════════════════════════════════════════════════════════

场景：发送缓冲区满 → 收到 ACK → 释放空间 → EPOLLOUT 触发

───────────────────────────────────────────────────────────────
初始状态：缓冲区满
───────────────────────────────────────────────────────────────
T0: 用户调用 write()
  │
  └─→ tcp_sendmsg()
      ├─ 检查缓冲区空间
      │  available = sk_sndbuf - sk_wmem_queued
      │  = 16KB - 16KB = 0
      │
      ├─ 缓冲区满，无法写入
      │
      └─ 返回 EAGAIN（非阻塞 socket）

T1: 用户添加 EPOLLOUT 监听
  │
  └─→ epoll_ctl(epfd, EPOLL_CTL_MOD, sockfd, EPOLLOUT)
      └─ 注册 ep_poll_callback 到 sk->sk_wq

───────────────────────────────────────────────────────────────
网络层：发送数据包
───────────────────────────────────────────────────────────────
T2: 内核后台发送数据
  │
  ├─→ tcp_write_xmit()
  │   └─ 从 sk_write_queue 取数据
  │       └─ 构造 TCP 数据包
  │           └─ IP 层封装
  │               └─ 网卡发送
  │
  └─ 数据包通过网络传输...

───────────────────────────────────────────────────────────────
接收端：处理并发送 ACK
───────────────────────────────────────────────────────────────
T3: 对方收到数据
  │
  └─→ 对方 TCP 协议栈处理
      └─ 发送 ACK 包回来

───────────────────────────────────────────────────────────────
本机：收到 ACK 包
───────────────────────────────────────────────────────────────
T4: 网卡收到 ACK 包
  │
  ├─→ 硬件中断
  ├─→ 软中断（NET_RX_SOFTIRQ）
  ├─→ 协议栈处理
  │   └─ tcp_v4_rcv()
  │       └─ tcp_ack()  ⚡ 处理 ACK
  │
  └─→ tcp_ack(sk, skb, flag)
      │
      ├─ 1. 更新发送窗口
      │    tp->snd_una = ack_seq
      │
      └─ 2. 清理发送队列 ⚡
          tcp_clean_rtx_queue(sk, flag)

───────────────────────────────────────────────────────────────
释放缓冲区空间
───────────────────────────────────────────────────────────────
T5: tcp_clean_rtx_queue() 执行
  │
  ├─→ 遍历 sk_write_queue
  │   for each skb in sk_write_queue {
  │       if (skb 已被 ACK 确认) {
  │           ├─ 从队列删除
  │           │  tcp_unlink_write_queue(skb, sk)
  │           │
  │           └─ 释放内存 ⚡
  │              sk_wmem_free_skb(sk, skb)
  │              └─ sk->sk_wmem_queued -= skb->truesize
  │                  // 假设释放了 8KB
  │       }
  │   }
  │
  └─→ 检查缓冲区状态 ⚡
      │
      ├─ 之前：sk_wmem_queued = 16KB（满）
      ├─ 现在：sk_wmem_queued = 8KB（释放了 8KB）
      │
      ├─ 计算可用空间
      │  available = sk_sndbuf - sk_wmem_queued
      │  = 16KB - 8KB = 8KB ✅
      │
      └─ 检查是否可写
         if (available >= min_wspace) {
             // 状态变化：不可写 → 可写 ⚡
             sk->sk_write_space(sk);
         }

───────────────────────────────────────────────────────────────
触发回调
───────────────────────────────────────────────────────────────
T6: sk_stream_write_space() 执行
  │
  ├─ 再次检查是否可写
  │  if (sk_stream_is_writeable(sk)) {  // 是
  │
  └─→ 唤醒等待队列
      wake_up_interruptible_sync_poll(&sk->sk_wq, POLLOUT)
      │
      └─→ __wake_up_common()
          └─ 遍历 sk->sk_wq
              └─ 调用每个 wait_entry->func()
                  │
                  └─→ ep_poll_callback() ⚡

───────────────────────────────────────────────────────────────
epoll 处理
───────────────────────────────────────────────────────────────
T7: ep_poll_callback() 执行
  │
  ├─ spin_lock(&ep->lock)
  │
  ├─ 检查事件类型
  │  pollflags = POLLOUT（来自 socket）
  │  epi->event.events = EPOLLOUT（用户注册）
  │  if (pollflags & epi->event.events)  // 匹配
  │
  ├─ 将 epitem 加入就绪列表 ⚡
  │  list_add_tail(&epi->rdllink, &ep->rdllist)
  │
  ├─ 唤醒 epoll_wait
  │  wake_up(&ep->wq)
  │
  └─ spin_unlock(&ep->lock)

───────────────────────────────────────────────────────────────
用户空间返回
───────────────────────────────────────────────────────────────
T8: epoll_wait() 返回
  │
  ├─ 进程被唤醒
  ├─ ep_send_events() 遍历就绪列表
  ├─ 拷贝事件到用户空间
  │   events[0].events = EPOLLOUT
  │   events[0].data.fd = sockfd
  │
  └─ 返回 nfds = 1

T9: 用户程序处理
  │
  └─→ if (events[i].events & EPOLLOUT) {
          // 缓冲区有空间了，可以继续发送
          write(sockfd, pending_data, pending_len);
      }

═══════════════════════════════════════════════════════════════
关键延迟：T9 - T3 = RTT（往返时间）+ 几百微秒（内核处理）
═══════════════════════════════════════════════════════════════
```

---

#### 3. 关键数据结构

#### 3.1 socket的等待队列

```c
// socket的等待队列结构
struct socket_wq 
{
    wait_queue_head_t    wait;    // 等待队列头
    struct fasync_struct *fasync_list;
    unsigned long        flags;
    struct rcu_head      rcu;
};

struct wait_queue_head 
{
    spinlock_t          lock;
    struct list_head    head;    // 链表头，存储 wait_queue_entry
};

// 等待队列项（epoll、注册的就是这个）
struct wait_queue_entry 
{
    unsigned int        flags;
    void                *private;     // 通常是epitem
    wait_queue_func_t   func;         // 回调函数：ep_poll_callback
    struct list_head    entry;        // 链表节点
};
```

**结构关系图**：

```
struct sock (socket 结构)
├─ sk_wq (等待队列) ───────────────┐
│                                 │
│  struct socket_wq               │
│  ├─ wait (wait_queue_head_t)    │
│  │   ├─ lock                    │
│  │   └─ head (list_head) ───────┼──→ 链表头
│  │                              │
│  └─ flags                       │
│                                 │
├─ sk_receive_queue (接收缓冲区)    │
├─ sk_write_queue (发送缓冲区)      │
└─ ...                            │
                                  │
                                  ▼
      ┌─────────────────────────────────────┐
      │ wait_queue_entry (链表节点 1)        │
      ├─────────────────────────────────────┤
      │ func = select_callback              │
      │ private = select_data               │
      │ entry → next                        │
      └──────────────┬──────────────────────┘
                     ▼
      ┌─────────────────────────────────────┐
      │ wait_queue_entry (链表节点 2)        │
      ├─────────────────────────────────────┤
      │ func = poll_callback                │
      │ private = poll_data                 │
      │ entry → next                        │
      └──────────────┬──────────────────────┘
                     ▼
      ┌─────────────────────────────────────┐
      │ wait_queue_entry (链表节点 3)        │
      ├─────────────────────────────────────┤
      │ func = ep_poll_callback  ⚡ epoll    │
      │ private = epitem                    │
      │ entry → next                        │
      └─────────────────────────────────────┘

当 socket 事件发生时（数据到达或可写）：
  wake_up(&sk->sk_wq->wait)
    └─→ 遍历链表
        ├─→ 调用 select_callback()
        ├─→ 调用 poll_callback()
        └─→ 调用 ep_poll_callback() ⚡
```

#### 3.2 事件通知的核心机制

```
═══════════════════════════════════════════════════════════════
              等待-唤醒机制（Wait-Wake Mechanism）
═══════════════════════════════════════════════════════════════

注册阶段（epoll_ctl ADD）：
─────────────────────────────────────────────────────────────
1. 用户调用 epoll_ctl(EPOLL_CTL_ADD)
   │
   └─→ ep_insert()
       │
       ├─ 创建 epitem
       │
       ├─ 创建 wait_queue_entry
       │  ├─ func = ep_poll_callback  ⚡ 设置回调
       │  └─ private = epitem
       │
       └─ 调用 socket->poll(file, &poll_table)
           │
           └─→ sock_poll()
               └─→ tcp_poll()
                   └─→ poll_wait(file, &sk->sk_wq->wait, poll_table)
                       │
                       └─→ add_wait_queue(&sk->sk_wq->wait, wait_entry)
                           ← 将回调注册到socket的等待队列

事件触发阶段：
─────────────────────────────────────────────────────────────
2. 事件发生（数据到达 or 缓冲区有空间）
   │
   ├─ EPOLLIN：数据拷贝到 sk_receive_queue
   │  └─→ sock_def_readable()
   │
   └─ EPOLLOUT：收到 ACK，释放缓冲区
      └─→ sk_stream_write_space()
          │
          └─→ wake_up(&sk->sk_wq->wait)
              │
              └─→ __wake_up_common()
                  │
                  └─→ 遍历等待队列，调用每个回调
                      │
                      ├─→ select_callback()
                      ├─→ poll_callback()
                      └─→ ep_poll_callback() ⚡
                          │
                          ├─ 将 epitem 加入 ep->rdllist
                          └─ wake_up(&ep->wq)
                              │
                              └─→ 唤醒 epoll_wait() 阻塞的进程

═══════════════════════════════════════════════════════════════
```

---

#### 4. 完整流程对比

#### 4.1 EPOLLIN vs EPOLLOUT

| 维度            | EPOLLIN                     | EPOLLOUT                    |
|---------------|-----------------------------|-----------------------------|
| **硬件触发**      | 网卡接收数据包 → 硬件中断              | 网卡发送数据包（对方ACK） → 硬件中断       |
| **数据流向**      | 网络 → 网卡 → 内核 → socket 接收缓冲区 | socket 发送缓冲区 → 内核 → 网卡 → 网络 |
| **缓冲区变化**     | sk_receive_queue: 空 → 非空    | sk_write_queue: 满 → 有空间     |
| **回调函数**      | sock_def_readable()         | sk_stream_write_space()     |
| **wake_up参数** | POLLIN\POLLRDNORM           | POLLOUT\POLLWRNORM          |
| **触发频率**      | 有数据才触发（按需）                  | 几乎总是可写（高频）                  |
| **内核检测方式**    | 被动：等待网卡中断                   | 被动：等待 ACK 中断                |
| **延迟**        | 网络传输延迟 + 几百微秒（内核处理）         | RTT（往返时间）+ 几百微秒（内核处理）       |

#### 4.2 关键共同点

1. **都是中断驱动**：
   - 不是轮询检测
   - 硬件/网络事件触发中断
   - 中断处理函数调用回调

2. **都通过wake_up通知**：
   - socket层调用`wake_up(&sk->sk_wq)`
   - 遍历等待队列
   - 调用`ep_poll_callback()`

3. **都检查状态变化**：
   - EPOLLIN: 从"无数据"到"有数据"
   - EPOLLOUT: 从"不可写"到"可写"

4. **都是异步通知**：
   - 事件发生在内核空间
   - 通过回调机制通知用户空间
   - 用户空间不需要轮询

#### 4.3 为什么epoll高效？

```
传统轮询方式（select/poll）：
┌─────────────────────────────────────────────────────────┐
│ while (true) {                                          │
│     for (fd in all_fds) {                               │
│         检查 fd 是否可读/可写  ← O(n) 时间复杂度            │
│     }                                                   │
│ }                                                       │
│                                                         │
│ 问题：                                                   │
│ - CPU不停轮询，浪费资源                                    │
│ - 即使没有事件，也要检查所有fd                              │
└─────────────────────────────────────────────────────────┘

epoll 事件驱动方式：
┌─────────────────────────────────────────────────────────┐
│ 1. 注册阶段：                                             │
│    epoll_ctl()将回调注册到socket的等待队列                  │
│                                                         │
│ 2. 等待阶段：                                             │
│    epoll_wait()进程睡眠，不占用CPU                        │
│                                                         │
│ 3. 事件触发：                                             │
│    硬件中断 → 回调函数 → 加入就绪列表 → 唤醒进程              │
│                                                         │
│ 4. 返回结果：                                             │
│    只返回就绪的fd，O(1)时间复杂度                           │
│                                                         │
│ 优势：                                                   │
│ - 事件驱动，不需要轮询                                     │
│ - 进程睡眠，不浪费CPU                                     │
│ - 只处理就绪的fd，效率高                                   │
└─────────────────────────────────────────────────────────┘
```

---

#### 总结

#### EPOLLIN的本质

**内核如何知道数据到了？**

```
1. 硬件主动通知（网卡中断）
   └─ 不是CPU轮询检测

2. 中断驱动处理流程：
   硬件中断 → 软中断 → 协议栈 → socket缓冲区 → wake_up → 回调

3. 回调函数ep_poll_callback将fd加入就绪列表

4. 唤醒epoll_wait，返回给用户空间
```

#### EPOLLOUT的本质

**内核如何知道fd可写？**

```
1. "可写"的定义：
   发送缓冲区有可用空间（sk_sndbuf - sk_wmem_queued >= 1）

2. 状态变化检测：
   收到ACK → 释放缓冲区 → 计算可用空间 → 触发回调

3. 回调函数ep_poll_callback将fd加入就绪列表

4. 唤醒epoll_wait，返回给用户空间
```

#### 核心机制

**等待-唤醒（Wait-Wake）机制**：

1. **注册**：epoll_ctl将回调注册到socket等待队列
2. **等待**：epoll_wait进程睡眠
3. **事件**：硬件/网络事件发生
4. **唤醒**：wake_up → 回调 → 加入就绪列表 → 唤醒进程
5. **返回**：返回就绪的fd

**为什么高效？**

- 事件驱动，不是轮询
- 中断通知，CPU不浪费
- O(1) 返回就绪fd
- 只处理就绪事件
