# ET模式下没读完的数据会丢失吗？

> 深入理解ET模式的数据处理机制

---

## 核心答案

**数据不会丢失！**

数据始终在内核的socket接收缓冲区（`sk_receive_queue`）中，直到被用户程序读取。

**但是有一个严重的问题**：如果你没有读完数据，ET模式**不会再次通知你**，导致数据"饿死"在缓冲区。

---

## 1. 数据在哪里？

### 1.1 数据流向

```
网络数据包的完整路径：

1. 网卡接收数据包
   ├─ 物理层：网线传输的电信号
   └─ 网卡芯片解析比特流

2. DMA拷贝到内核内存
   └─ 不需要CPU参与

3. 协议栈处理
   ├─ IP层：解析IP头
   └─ TCP层：解析TCP头

4. 拷贝到socket接收缓冲区 ⚡
   ┌────────────────────────────────────────┐
   │ struct sock {                          │
   │     struct sk_buff_head                │
   │         sk_receive_queue;  ← 数据在这里  │
   │ }                                      │
   └────────────────────────────────────────┘

5. 用户调用read()/recv()读取
   └─ 从sk_receive_queue拷贝到用户空间
```

**关键点**：
- 数据在步骤4就已经在**内核的 socket缓冲区**中了
- 只要用户不调用`read()`，数据就**一直在那里**
- ET模式不会影响数据的存储

### 1.2 socket 接收缓冲区结构

```c
// 内核中的 socket 结构（简化）
struct sock 
{
    // 接收队列：链表结构，存储数据包
    struct sk_buff_head sk_receive_queue;

    // 接收缓冲区大小限制（默认 ~87KB）
    int sk_rcvbuf;

    // 当前已用的缓冲区大小
    atomic_t sk_rmem_alloc;

    // 等待队列（epoll 回调注册在这里）
    struct socket_wq *sk_wq;
};

// sk_buff：socket 缓冲区中的数据包
struct sk_buff 
{
    struct sk_buff *next;     // 下一个数据包
    struct sk_buff *prev;     // 上一个数据包
    unsigned char *data;      // 实际数据
    unsigned int len;         // 数据长度
    // ...
};
```

**数据存储示意图**：

```
sk_receive_queue (链表)
┌──────────────────────────────────────────────────────┐
│ head                                                 │
│   │                                                  │
│   ▼                                                  │
│ ┌─────────┐    ┌─────────┐    ┌─────────┐            │
│ │ sk_buff │───→│ sk_buff │───→│ sk_buff │───→ NULL   │
│ │ len=100 │    │ len=200 │    │ len=150 │            │
│ │ data... │    │ data... │    │ data... │            │
│ └─────────┘    └─────────┘    └─────────┘            │
│                                                      │
│ 总共 450 字节在缓冲区中                                 │
└──────────────────────────────────────────────────────┘
```

---

## 2. ET 模式的真正问题

### 2.1 ET 模式 vs LT 模式

```
场景：socket接收缓冲区有1000字节数据

═══════════════════════════════════════════════════════
LT模式（水平触发）- 安全但可能低效
═══════════════════════════════════════════════════════

第1次epoll_wait():
  ├─ 返回EPOLLIN
  └─ 用户 read(fd, buf, 100)  // 只读了100字节

  缓冲区状态：还剩900字节

第2次epoll_wait():
  ├─ 返回EPOLLIN   ← LT 模式会再次通知！
  └─ 用户read(fd, buf, 100)  // 再读100字节

  缓冲区状态：还剩 800 字节

第3次epoll_wait():
  ├─ 返回 EPOLLIN  ← 继续通知
  └─ ...

结果：
  数据最终会被读完
  epoll_wait() 会频繁返回（低效）


═══════════════════════════════════════════════════════
ET模式（边缘触发）- 高效但需要小心
═══════════════════════════════════════════════════════

第1次 epoll_wait():
  ├─ 缓冲区状态变化：空 → 1000字节
  ├─ 返回 EPOLLIN  ← 状态改变，触发
  └─ 用户 read(fd, buf, 100)  // 只读了100字节

  缓冲区状态：还剩 900 字节

第2次 epoll_wait():
  ├─ 缓冲区状态：900字节（没变化）
  └─ 不返回 EPOLLIN   ← ET 模式不会再次通知！

第3次 epoll_wait():
  └─ 不返回 EPOLLIN   ← 一直不会通知

... （永远不会再通知）

结果：
  900 字节数据"饿死"在缓冲区
  应用以为没数据了，实际上有900字节
  但数据没有丢失，还在缓冲区
```

### 2.2 什么时候 ET 模式会再次触发？

**只有当缓冲区状态发生变化时**：

```
场景：ET模式下，第一次没读完

时间 T0:
  ├─ 缓冲区：0 字节（空）
  └─ EPOLLIN: 不触发

时间 T1: 收到 1000 字节数据
  ├─ 缓冲区：0 → 1000 字节（状态改变！）
  └─ EPOLLIN: 触发

时间 T2: 用户只读了 100 字节
  ├─ 缓冲区：1000 → 900 字节（还有数据）
  └─ EPOLLIN: 不触发（状态没变，还是"有数据"）

时间 T3-T10: 用户继续等待
  ├─ 缓冲区：900 字节（不变）
  └─ EPOLLIN: 不触发

时间 T11: 又收到 500 字节新数据 ⚡
  ├─ 缓冲区：900 → 1400 字节（状态改变！）
  └─ EPOLLIN: 触发
      └─ 这时才会再次通知！
```

**总结**：
- ET模式只在**状态改变**时触发
- "有数据" → "有更多数据" **不算状态改变**
- 只有 "空" → "有数据" 或 "有数据" → "空" 才算

---

## 3. 数据会丢失吗？

### 3.1 答案：不会丢失

```
内核视角：

┌─────────────────────────────────────────────────────┐
│ socket 接收缓冲区                                     │
├─────────────────────────────────────────────────────┤
│                                                     │
│ [数据包1] → [数据包2] → [数据包3] → NULL               │
│   100字节     200字节     600字节                     │
│                                                     │
│ 总共 900 字节                                        │
│                                                     │
│ 这些数据会一直保存，直到：                              │
│ 1. 用户调用 read() 读取                               │
│ 2. socket 关闭                                      │
│ 3. 缓冲区满了，后续数据被丢弃                           │
│                                                     │
└─────────────────────────────────────────────────────┘

用户视角（ET 模式，没读完）：

┌─────────────────────────────────────────────────────┐
│ 应用程序状态                                          │
├─────────────────────────────────────────────────────┤
│                                                     │
│ 1. epoll_wait()返回EPOLLIN                          │
│ 2. read()读了100字节                                 │
│ 3. 没有继续读，直接return                             │
│ 4. 再次epoll_wait()                                 │
│ 5. 不返回EPOLLIN（因为没有新数据）                      │
│ 6. 阻塞等待...                                       │
│                                                     │
│ 结果：                                               │
│ - 应用以为数据读完了                                   │
│ - 实际上还有 900 字节在内核缓冲区                       │
│ - 数据"饿死"，但没有丢失                               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 3.2 数据何时真正丢失？

**真正会导致数据丢失的情况**：

```
1. 接收缓冲区满了
   ┌──────────────────────────────────────────────┐
   │ sk_receive_queue: 已满（87KB）                │
   ├──────────────────────────────────────────────┤
   │ 新数据包到达                                   │
   │ └─ 内核无法加入队列                             │
   │    └─ 丢弃数据包                               │
   │       └─ TCP层不发送ACK                       │
   │          └─ 发送端会重传                       │
   └──────────────────────────────────────────────┘

2. socket关闭
   ┌──────────────────────────────────────────────┐
   │ close(sockfd)                                │
   │ └─ 清空接收缓冲区                              │
   │    └─ 未读的数据全部丢失                        │
   └──────────────────────────────────────────────┘

3. 进程崩溃/被杀
   └─ 所有socket关闭，数据丢失
```

**不会丢失的情况**：
- ET 模式下没读完（数据还在缓冲区）
- LT 模式下没读完（数据还在缓冲区）
- epoll_wait() 超时返回（数据还在缓冲区）

---

## 4. ET 模式的正确用法

### 4.1 必须循环读取直到 EAGAIN

```c
// 正确的ET模式读取方式

if (events[i].events & EPOLLIN) {
    int fd = events[i].data.fd;

    // 关键：循环读取！
    while (1) {
        char buf[4096];
        ssize_t n = read(fd, buf, sizeof(buf));

        if (n > 0) {
            // 读取到数据，处理
            process_data(buf, n);
            // 继续读取 ← 不要 break！

        } else if (n == 0) {
            // EOF，对方关闭连接
            close(fd);
            break;

        } else {  // n < 0
            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                // 缓冲区空了，所有数据读完
                break;  // 现在才能退出
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

### 4.2 为什么必须读到EAGAIN？

```
完整的读取流程：

初始状态：
┌──────────────────────────────────────────┐
│ sk_receive_queue: [1000 字节]             │
└──────────────────────────────────────────┘

第1次read(fd, buf, 4096):
  ├─ 读取 1000 字节
  ├─ 缓冲区清空
  └─ 返回 1000

第2次read(fd, buf, 4096):
  ├─ 缓冲区为空
  ├─ 非阻塞socket
  └─ 返回-1，errno = EAGAIN
      └─ 这是正常的，表示数据读完了！

如果没有第2次read：
┌──────────────────────────────────────────┐
│ 问题：你怎么知道数据读完了？                  │
├──────────────────────────────────────────┤
│ - 第1次读了1000字节                        │
│ - 可能还有数据在缓冲区                      │
│ - 必须继续读，直到EAGAIN                   │
│ - 才能确认数据真的读完了                     │
└──────────────────────────────────────────┘
```

### 4.3 错误示例和后果

```c
// 错误示例 1：只读一次

if (events[i].events & EPOLLIN) {
    char buf[4096];
    ssize_t n = read(fd, buf, sizeof(buf));

    if (n > 0) {
        process_data(buf, n);
    }
    // 没有循环读取！
}

// 后果：
// - 如果缓冲区有 10KB 数据
// - 只读了 4KB
// - 剩余 6KB "饿死"
// - 直到下次有新数据到达


// 错误示例 2：读固定次数

if (events[i].events & EPOLLIN) {
    for (int i = 0; i < 10; i++) {  // 最多读 10 次
        char buf[4096];
        ssize_t n = read(fd, buf, sizeof(buf));
        if (n <= 0) break;
        process_data(buf, n);
    }
}

// 后果：
// - 如果有 100KB 数据（需要 25 次读取）
// - 只读了 10 次（40KB）
// - 剩余 60KB "饿死"


// ✅ 正确示例：读到 EAGAIN

if (events[i].events & EPOLLIN) {
    while (1) {
        char buf[4096];
        ssize_t n = read(fd, buf, sizeof(buf));

        if (n > 0) {
            process_data(buf, n);
        } else if (n == 0) {
            close(fd);
            break;
        } else {
            if (errno == EAGAIN) {
                break;  // 数据读完 ✅
            }
            // 错误处理...
        }
    }
}
```

---

## 5. 实际场景演示

### 5.1 场景：HTTP 请求

```
客户端发送一个 HTTP 请求：

POST /api/data HTTP/1.1
Host: example.com
Content-Length: 10240
Content-Type: application/json

{
  "data": "...",   ← 10KB JSON 数据
  ...
}

总大小：约 10.5KB
```

**ET 模式处理（错误方式）**：

```c
// ❌ 只读一次
char buf[4096];
ssize_t n = read(fd, buf, sizeof(buf));

结果：
  ├─ 读取了 4KB（HTTP 头 + 部分 JSON）
  ├─ 剩余 6.5KB 在缓冲区
  ├─ 应用解析 JSON 失败（数据不完整）
  └─ 请求处理失败 ❌
```

**ET 模式处理（正确方式）**：

```c
// ✅ 循环读取
char buffer[10240] = {0};
size_t total = 0;

while (1) {
    ssize_t n = read(fd, buffer + total, sizeof(buffer) - total);

    if (n > 0) {
        total += n;
    } else if (n < 0 && errno == EAGAIN) {
        break;  // 数据读完
    } else {
        // 错误处理
    }
}

结果：
  ├─ 读取了完整的 10.5KB
  ├─ 应用解析 JSON 成功 ✅
  └─ 请求处理成功 ✅
```

### 5.2 场景：大文件传输

```
客户端上传一个 1MB 文件
```

**ET 模式处理**：

```c
// ✅ 正确处理大量数据

FILE *fp = fopen("upload.dat", "wb");
char buf[65536];  // 64KB 缓冲区

while (1) {
    ssize_t n = read(sockfd, buf, sizeof(buf));

    if (n > 0) {
        fwrite(buf, 1, n, fp);
        // 继续读取，不要 break！

    } else if (n == 0) {
        // 传输完成
        fclose(fp);
        break;

    } else {
        if (errno == EAGAIN) {
            // 当前所有数据读完，等待下一批
            break;
        }
        // 错误处理...
    }
}

说明：
  - 1MB 文件需要多次 EPOLLIN 触发
  - 每次触发都要读到 EAGAIN
  - 确保每一批数据都完整读取
```

---

## 6. 总结

### 6.1 核心要点

| 问题                | 答案                          |
|-------------------|------------------------------|
| **数据会丢失吗？**     | ❌ 不会，数据在内核缓冲区         |
| **数据在哪里？**      | ✅ `sk_receive_queue`（内核） |
| **ET 模式的问题？**   | ⚠️ 不会再次通知，数据"饿死"      |
| **如何避免饿死？**     | ✅ 循环读取直到 EAGAIN          |
| **为什么要读到 EAGAIN？** | ✅ 确保所有数据读完            |

### 6.2 ET 模式最佳实践

```c
✅ DO（必须做）:
  1. 使用非阻塞 socket
  2. 循环读取直到 EAGAIN
  3. 检查 errno

❌ DON'T（不要做）:
  1. 只读一次就停止
  2. 读固定次数
  3. 使用阻塞 socket
```

### 6.3 记忆口诀

```
ET 模式三原则：

1. 非阻塞 socket，必须设
2. 循环读取，读到 EAGAIN
3. 数据不丢失，就在内核中

LT 模式更宽容：
  - 没读完？没关系，下次继续通知
  - 适合初学者

ET 模式更高效：
  - 没读完？不再通知，数据饿死
  - 适合高性能场景
```

---

## 7. 验证实验

### 7.1 实验代码

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <netinet/in.h>

void test_et_partial_read() {
    int epfd = epoll_create1(0);
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    // 设置非阻塞
    fcntl(sockfd, F_SETFL, O_NONBLOCK);

    // 监听端口
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8888);
    addr.sin_addr.s_addr = INADDR_ANY;

    bind(sockfd, (struct sockaddr*)&addr, sizeof(addr));
    listen(sockfd, 10);

    // 添加到 epoll (ET 模式)
    struct epoll_event ev;
    ev.events = EPOLLIN | EPOLLET;
    ev.data.fd = sockfd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd, &ev);

    printf("服务器启动，端口 8888\n");
    printf("使用命令测试：echo '12345678901234567890' | nc localhost 8888\n\n");

    struct epoll_event events[10];

    while (1) {
        int nfds = epoll_wait(epfd, events, 10, -1);

        for (int i = 0; i < nfds; i++) {
            if (events[i].data.fd == sockfd) {
                // 接受连接
                int conn = accept(sockfd, NULL, NULL);
                fcntl(conn, F_SETFL, O_NONBLOCK);

                ev.events = EPOLLIN | EPOLLET;
                ev.data.fd = conn;
                epoll_ctl(epfd, EPOLL_CTL_ADD, conn, &ev);

                printf("新连接 fd=%d\n", conn);

            } else {
                int fd = events[i].data.fd;
                printf("\n=== EPOLLIN 触发，fd=%d ===\n", fd);

                // ❌ 错误方式：只读一次
                char buf[10];
                ssize_t n = read(fd, buf, sizeof(buf));

                if (n > 0) {
                    printf("第 1 次读取：%zd 字节\n", n);
                    write(1, buf, n);
                    printf("\n");

                    // 检查缓冲区是否还有数据
                    printf("尝试第 2 次读取...\n");
                    n = read(fd, buf, sizeof(buf));

                    if (n > 0) {
                        printf("第 2 次读取：%zd 字节（数据还在！）\n", n);
                        write(1, buf, n);
                        printf("\n");
                    } else if (n < 0 && errno == EAGAIN) {
                        printf("第 2 次读取：EAGAIN（缓冲区空了）\n");
                    }
                } else if (n == 0) {
                    printf("连接关闭\n");
                    close(fd);
                }

                printf("等待下一次 EPOLLIN...\n");
            }
        }
    }
}

int main() {
    test_et_partial_read();
    return 0;
}
```

### 7.2 实验结果

```bash
# 终端 1：启动服务器
$ gcc test_et.c -o test_et
$ ./test_et
服务器启动，端口 8888

# 终端 2：发送 20 字节数据
$ echo '12345678901234567890' | nc localhost 8888

# 终端 1 输出：
新连接 fd=4

=== EPOLLIN 触发，fd=4 ===
第 1 次读取：10 字节
1234567890
尝试第 2 次读取...
第 2 次读取：10 字节（数据还在！）  ← 数据没丢失 ✅
1234567890
等待下一次 EPOLLIN...

结论：
  ✅ 数据没有丢失
  ✅ 还在内核缓冲区
  ⚠️ 如果只读一次，剩余数据"饿死"
```

---

**最终结论**：

在 ET 模式下，如果没有读完所有数据：
- ✅ **数据不会丢失**，依然在内核缓冲区
- ❌ **但不会再次通知**，导致数据"饿死"
- ⚠️ **必须循环读取直到 EAGAIN**，确保读完所有数据

这就是为什么 ET 模式比 LT 模式难用，但性能更好的原因！
