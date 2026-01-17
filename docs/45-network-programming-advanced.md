# 网络编程进阶主题

---

## 目录

1. [零拷贝技术（Zero-Copy）](#1-零拷贝技术zero-copy)
   - 1.1 传统IO的拷贝问题
   - 1.2 sendfile()系统调用
   - 1.3 splice()和tee()
   - 1.4 mmap() + write()
   - 1.5 DMA与Page Cache
   - 1.6 应用场景对比
   - 1.7 高频面试题

2. [高性能网络编程](#2-高性能网络编程)
   - 2.1 DPDK（绕过内核）
   - 2.2 io_uring（新异步IO）
   - 2.3 XDP（eBPF）
   - 2.4 性能对比与选型
   - 2.5 高频面试题

3. [协议实现](#3-协议实现)
   - 3.1 HTTP/1.1 实现
   - 3.2 HTTP/2 多路复用
   - 3.3 WebSocket 长连接
   - 3.4 QUIC（HTTP/3）
   - 3.5 高频面试题

4. [分布式系统网络编程](#4-分布式系统网络编程)
   - 4.1 RPC 框架原理
   - 4.2 服务发现机制
   - 4.3 负载均衡策略
   - 4.4 熔断、降级、限流
   - 4.5 高频面试题

5. [网络安全](#5-网络安全)
   - 5.1 TLS/SSL 原理
   - 5.2 DDoS 防御
   - 5.3 限流算法
   - 5.4 降级策略
   - 5.5 高频面试题

---

## 1. 零拷贝技术（Zero-Copy）

### 1.1 传统IO的拷贝问题

#### 问题：传统文件发送的4次拷贝

```c
// 传统方式发送文件
int fd = open("file.txt", O_RDONLY);
char buf[4096];
while ((n = read(fd, buf, sizeof(buf))) > 0) 
{
    send(sockfd, buf, n, 0);
}
```

**内存拷贝流程**：

```
┌──────────────────────────────────────────────────────────┐
│  1. 磁盘 → 内核缓冲区（DMA拷贝）                             │
│  2. 内核缓冲区 → 用户空间缓冲区（CPU拷贝）                    │
│  3. 用户空间缓冲区 → Socket缓冲区（CPU拷贝）                  │
│  4. Socket缓冲区 → 网卡（DMA拷贝）                          │
└──────────────────────────────────────────────────────────┘

总共：4次拷贝（2次DMA + 2次CPU）+ 4次上下文切换

上下文切换：
1. read()  ：用户态 → 内核态
2. read()返回：内核态 → 用户态
3. send()  ：用户态 → 内核态
4. send()返回：内核态 → 用户态
```

**详细流程图**：

```
磁盘文件                                          网络
   ↓                                              ↑
   │ ① DMA拷贝                                    │ ④ DMA拷贝
   ↓                                              │
┌─────────────────┐                    ┌─────────────────┐
│  内核缓冲区       │ ② CPU拷贝          │  Socket缓冲区    │
│ (Page Cache)    │ ────────────────→  │                 │
└─────────────────┘                    └─────────────────┘
        │                                       ↑
        │ ② CPU拷贝                             │ ③ CPU拷贝
        ↓                                       │
┌─────────────────────────────────────────────────────────┐
│              用户空间缓冲区 buf[4096]                     │
└─────────────────────────────────────────────────────────┘
```

**性能开销**：
- 2次CPU拷贝（浪费CPU资源）
- 4次上下文切换（用户态↔内核态）
- 用户空间缓冲区多余（buf[]没必要存在）

---

### 1.2 sendfile()系统调用

#### 原理：在内核空间直接传输

```c
#include <sys/sendfile.h>

// 函数签名
ssize_t sendfile(int out_fd,    // 输出文件描述符（socket）
                 int in_fd,     // 输入文件描述符（文件）
                 off_t *offset, // 文件偏移（NULL=从头开始）
                 size_t count); // 传输字节数

// 返回值：成功返回传输的字节数，失败返回-1
```

#### 完整示例：发送文件

```c
// sendfile_example.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/socket.h>
#include <sys/sendfile.h>
#include <sys/stat.h>
#include <arpa/inet.h>

// 使用sendfile发送文件
int send_file_with_sendfile(int sockfd, const char *filename) 
{
    int file_fd;
    struct stat file_stat;
    off_t offset = 0;
    ssize_t sent_bytes;

    // 1. 打开文件
    file_fd = open(filename, O_RDONLY);
    if (file_fd < 0) 
    {
        perror("open");
        return -1;
    }

    // 2. 获取文件大小
    if (fstat(file_fd, &file_stat) < 0) 
    {
        perror("fstat");
        close(file_fd);
        return -1;
    }

    // 3. 零拷贝发送（一次系统调用完成）
    sent_bytes = sendfile(sockfd, file_fd, &offset, file_stat.st_size);

    if (sent_bytes < 0) 
    {
        perror("sendfile");
        close(file_fd);
        return -1;
    }

    printf("✓ Sent %ld bytes using sendfile\n", sent_bytes);

    close(file_fd);
    return 0;
}

// 完整的HTTP文件服务器
void handle_http_request(int client_fd) 
{
    char buffer[1024];
    int n = recv(client_fd, buffer, sizeof(buffer) - 1, 0);
    buffer[n] = '\0';

    // 解析请求：GET /file.txt HTTP/1.1
    char method[16], path[256], version[16];
    sscanf(buffer, "%s %s %s", method, path, version);

    // 去掉开头的 '/'
    const char *filename = path + 1;

    // 获取文件信息
    struct stat file_stat;
    if (stat(filename, &file_stat) < 0 || !S_ISREG(file_stat.st_mode)) 
    {
        // 文件不存在或不是普通文件
        const char *response = "HTTP/1.1 404 Not Found\r\n\r\n";
        send(client_fd, response, strlen(response), 0);
        close(client_fd);
        return;
    }

    // 发送HTTP响应头
    char header[256];
    snprintf(header, sizeof(header),
             "HTTP/1.1 200 OK\r\n"
             "Content-Length: %ld\r\n"
             "Content-Type: application/octet-stream\r\n"
             "\r\n",
             file_stat.st_size);
    send(client_fd, header, strlen(header), 0);

    // 使用sendfile零拷贝发送文件内容
    send_file_with_sendfile(client_fd, filename);

    close(client_fd);
}

int main() 
{
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);

    int reuse = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8080);
    addr.sin_addr.s_addr = INADDR_ANY;

    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 128);

    printf("HTTP server listening on port 8080...\n");
    printf("Try: curl http://localhost:8080/file.txt\n");

    while (1) 
    {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);
        int client_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &client_len);

        if (client_fd < 0) continue;

        handle_http_request(client_fd);
    }

    return 0;
}
```

#### sendfile() 的优化效果

```
优化后的流程：只需2次拷贝 + 2次上下文切换

磁盘文件                                          网络
   ↓                                              ↑
   │ ① DMA拷贝                                    │ ② DMA拷贝
   ↓                                              │
┌─────────────────┐                     ┌─────────────────┐
│  内核缓冲区       │ ── 内核空间直接传输 ─→│  Socket缓冲区    │
│ (Page Cache)    │    （无CPU拷贝）      │                 │
└─────────────────┘                     └─────────────────┘

✓ 减少到2次拷贝（2次DMA，0次CPU拷贝）
✓ 减少到2次上下文切换
✓ 不需要用户空间缓冲区
✓ CPU资源释放出来处理其他任务
```

**性能提升**：
- 大文件传输：提升**2-3倍**性能
- CPU使用率：降低**50%**以上
- 适用场景：文件服务器、视频流、CDN

#### sendfile() 的局限性

```c
// 限制1：只能用于文件 → Socket
sendfile(sockfd, file_fd, NULL, size);  // ✓ 可以
sendfile(file_fd, sockfd, NULL, size);  // ✗ 不行

// 限制2：不能修改数据（例如加密、压缩）
// 如果需要修改数据，必须用read/write

// 限制3：in_fd必须支持mmap（普通文件可以，socket不行）
```

---

### 1.3 splice()和tee()

#### splice()：管道零拷贝

```c
#include <fcntl.h>

ssize_t splice(int fd_in,           // 输入fd
               loff_t *off_in,      // 输入偏移
               int fd_out,          // 输出fd
               loff_t *off_out,     // 输出偏移
               size_t len,          // 传输长度
               unsigned int flags); // 标志位

// flags:
// - SPLICE_F_MOVE    : 尝试移动页面而不是拷贝
// - SPLICE_F_NONBLOCK: 非阻塞模式
// - SPLICE_F_MORE    : 后续还有更多数据
```

**splice() vs sendfile()**：

```
sendfile():
  文件 → Socket（单向，内核直接传输）

splice():
  任意fd ↔ 管道 ↔ 任意fd（通过管道中转）
  - Socket → 管道 → Socket
  - 文件 → 管道 → Socket
  - Socket → 管道 → 文件
```

#### 完整示例：Socket 数据转发（零拷贝代理）

```c
// splice_proxy.c - 零拷贝TCP代理
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <errno.h>

#define BUFFER_SIZE 65536

// 使用splice实现零拷贝转发
int splice_forward(int from_fd, int to_fd) 
{
    int pipefd[2];
    ssize_t bytes;

    // 1. 创建管道
    if (pipe(pipefd) < 0) 
    {
        perror("pipe");
        return -1;
    }

    // 2. from_fd → pipe （零拷贝）
    bytes = splice(from_fd, NULL, pipefd[1], NULL, BUFFER_SIZE, SPLICE_F_MOVE | SPLICE_F_MORE);

    if (bytes < 0) 
    {
        perror("splice from");
        close(pipefd[0]);
        close(pipefd[1]);
        return -1;
    }

    if (bytes == 0) 
    {
        // 对端关闭
        close(pipefd[0]);
        close(pipefd[1]);
        return 0;
    }

    // 3. pipe → to_fd （零拷贝）
    ssize_t written = splice(pipefd[0], NULL, to_fd, NULL, bytes, SPLICE_F_MOVE | SPLICE_F_MORE);

    close(pipefd[0]);
    close(pipefd[1]);

    if (written < 0) 
    {
        perror("splice to");
        return -1;
    }

    return written;
}

// 完整的TCP代理服务器
void handle_proxy(int client_fd, const char *backend_host, int backend_port) 
{
    int backend_fd;
    struct sockaddr_in backend_addr;

    // 1. 连接后端服务器
    backend_fd = socket(AF_INET, SOCK_STREAM, 0);
    backend_addr.sin_family = AF_INET;
    backend_addr.sin_port = htons(backend_port);
    inet_pton(AF_INET, backend_host, &backend_addr.sin_addr);

    if (connect(backend_fd, (struct sockaddr*)&backend_addr, sizeof(backend_addr)) < 0) 
    {
        perror("connect");
        close(client_fd);
        return;
    }

    // 2. 设置为非阻塞
    fcntl(client_fd, F_SETFL, O_NONBLOCK);
    fcntl(backend_fd, F_SETFL, O_NONBLOCK);

    printf("✓ Proxy connection established\n");

    // 3. 双向转发（使用 splice 零拷贝）
    fd_set readfds;
    while (1) 
    {
        FD_ZERO(&readfds);
        FD_SET(client_fd, &readfds);
        FD_SET(backend_fd, &readfds);

        int maxfd = (client_fd > backend_fd) ? client_fd : backend_fd;
        int ret = select(maxfd + 1, &readfds, NULL, NULL, NULL);

        if (ret < 0) break;

        // 客户端 → 后端
        if (FD_ISSET(client_fd, &readfds)) 
        {
            int n = splice_forward(client_fd, backend_fd);
            if (n <= 0) break;
            printf("→ Client to backend: %d bytes\n", n);
        }

        // 后端 → 客户端
        if (FD_ISSET(backend_fd, &readfds)) 
        {
            int n = splice_forward(backend_fd, client_fd);
            if (n <= 0) break;
            printf("← Backend to client: %d bytes\n", n);
        }
    }

    close(client_fd);
    close(backend_fd);
}

int main(int argc, char *argv[]) 
{
    if (argc != 4) 
    {
        fprintf(stderr, "Usage: %s <listen_port> <backend_host> <backend_port>\n", argv[0]);
        exit(1);
    }

    int listen_port = atoi(argv[1]);
    const char *backend_host = argv[2];
    int backend_port = atoi(argv[3]);

    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    int reuse = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(listen_port);
    addr.sin_addr.s_addr = INADDR_ANY;

    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 128);

    printf("Splice proxy listening on port %d\n", listen_port);
    printf("Forwarding to %s:%d\n", backend_host, backend_port);

    while (1) 
    {
        int client_fd = accept(listen_fd, NULL, NULL);
        if (client_fd < 0) continue;

        // 实际应用中应该fork或用线程
        handle_proxy(client_fd, backend_host, backend_port);
    }

    return 0;
}
```

**编译运行**：

```bash
# 编译
gcc splice_proxy.c -o splice_proxy

# 运行：监听9000端口，转发到localhost:8080
./splice_proxy 9000 127.0.0.1 8080

# 测试
curl http://localhost:9000/
```

#### tee()：复制管道数据

```c
#include <fcntl.h>

ssize_t tee(int fd_in,      // 输入管道
            int fd_out,     // 输出管道
            size_t len,     // 复制长度
            unsigned int flags);

// 作用：复制管道数据（不消耗输入管道的数据）
```

**应用场景：流量复制**

```c
// tee_traffic.c - 流量复制（例如镜像到监控系统）
int mirror_traffic(int client_fd, int backend_fd, int monitor_fd) 
{
    int pipe_in[2], pipe_out[2];
    pipe(pipe_in);
    pipe(pipe_out);

    // 1. 客户端 → pipe_in
    splice(client_fd, NULL, pipe_in[1], NULL, 4096, SPLICE_F_MOVE);

    // 2. pipe_in 复制到 pipe_out （tee不消耗pipe_in的数据）
    tee(pipe_in[0], pipe_out[1], 4096, 0);

    // 3. pipe_in → 后端服务器
    splice(pipe_in[0], NULL, backend_fd, NULL, 4096, SPLICE_F_MOVE);

    // 4. pipe_out → 监控系统
    splice(pipe_out[0], NULL, monitor_fd, NULL, 4096, SPLICE_F_MOVE);

    // 结果：数据同时发送到后端和监控系统，且零拷贝
}
```

---

### 1.4 mmap() + write()

#### mmap()：内存映射文件

```c
#include <sys/mman.h>

void *mmap(void *addr,      // 映射地址（通常为NULL自动选择）
           size_t length,   // 映射长度
           int prot,        // 保护标志（PROT_READ/PROT_WRITE）
           int flags,       // 映射标志（MAP_SHARED/MAP_PRIVATE）
           int fd,          // 文件描述符
           off_t offset);   // 文件偏移

// 返回值：成功返回映射地址，失败返回MAP_FAILED

int munmap(void *addr, size_t length);  // 解除映射
```

#### 原理：减少一次拷贝

```
传统 read/write：
  磁盘 → 内核缓冲区 → 用户缓冲区 → Socket缓冲区 → 网卡
       (DMA)      (CPU)        (CPU)        (DMA)

mmap + write：
  磁盘 → 内核缓冲区（映射到用户空间）→ Socket缓冲区 → 网卡
       (DMA)                   (CPU)        (DMA)

✓ 减少1次CPU拷贝（内核→用户）
✓ 用户空间可以直接访问内核缓冲区
```

#### 完整示例：mmap发送文件

```c
// mmap_sendfile.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <sys/socket.h>
#include <arpa/inet.h>

int send_file_with_mmap(int sockfd, const char *filename) 
{
    int file_fd;
    struct stat file_stat;
    void *mapped;
    ssize_t sent_bytes;

    // 1. 打开文件
    file_fd = open(filename, O_RDONLY);
    if (file_fd < 0) 
    {
        perror("open");
        return -1;
    }

    // 2. 获取文件大小
    if (fstat(file_fd, &file_stat) < 0) 
    {
        perror("fstat");
        close(file_fd);
        return -1;
    }

    // 3. 将文件映射到内存
    mapped = mmap(NULL, file_stat.st_size, PROT_READ, MAP_PRIVATE, file_fd, 0);
    if (mapped == MAP_FAILED) 
    {
        perror("mmap");
        close(file_fd);
        return -1;
    }

    // 4. 直接发送映射的内存（减少一次拷贝）
    sent_bytes = send(sockfd, mapped, file_stat.st_size, 0);

    if (sent_bytes < 0) 
    {
        perror("send");
    } 
    else 
    {
        printf("✓ Sent %ld bytes using mmap\n", sent_bytes);
    }

    // 5. 清理
    munmap(mapped, file_stat.st_size);
    close(file_fd);

    return (sent_bytes < 0) ? -1 : 0;
}

// 性能对比测试
void performance_test(const char *filename) 
{
    struct timeval start, end;
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(9999);
    inet_pton(AF_INET, "127.0.0.1", &addr.sin_addr);
    connect(sockfd, (struct sockaddr*)&addr, sizeof(addr));

    // 测试1：传统方式
    gettimeofday(&start, NULL);
    // send_file_traditional(sockfd, filename);
    gettimeofday(&end, NULL);
    long traditional_us = (end.tv_sec - start.tv_sec) * 1000000 + (end.tv_usec - start.tv_usec);

    // 测试2：mmap方式
    gettimeofday(&start, NULL);
    send_file_with_mmap(sockfd, filename);
    gettimeofday(&end, NULL);
    long mmap_us = (end.tv_sec - start.tv_sec) * 1000000 + (end.tv_usec - start.tv_usec);

    printf("\nPerformance comparison:\n");
    printf("  Traditional: %ld μs\n", traditional_us);
    printf("  mmap:        %ld μs\n", mmap_us);
    printf("  Speedup:     %.2fx\n", (double)traditional_us / mmap_us);

    close(sockfd);
}

int main() 
{
    // 创建测试文件
    system("dd if=/dev/zero of=test.dat bs=1M count=100");
    performance_test("test.dat");
    return 0;
}
```

#### mmap的应用场景

**✓ 适合的场景**：

```c
// 1. 大文件随机访问
void *data = mmap(...);
int value = ((int*)data)[1000000];  // 随机访问第1M个元素

// 2. 进程间共享内存（IPC）
// 进程A
void *shm = mmap(NULL, 4096, PROT_READ | PROT_WRITE,
                 MAP_SHARED | MAP_ANONYMOUS, -1, 0);
// 进程B可以访问同一块内存

// 3. 数据库文件映射
// 数据库将数据文件mmap到内存，直接操作
```

**✗ 不适合的场景**：

```c
// 1. 小文件（<4KB）
//    → mmap的页面对齐开销大于收益

// 2. 顺序读取大文件
//    → sendfile()更快

// 3. 需要修改文件后立即发送
//    → mmap的写回是异步的，可能导致数据不一致
```

---

### 1.5 DMA与Page Cache

#### DMA（Direct Memory Access）原理

```
传统IO（Programmed I/O, PIO）：
  CPU逐字节搬运数据
  ┌─────┐      ┌──────┐      ┌──────┐
  │ 磁盘 │ ───→ │ CPU  │ ───→ │ 内存  │
  └─────┘      └──────┘      └──────┘
  缺点：CPU全程参与，浪费CPU资源

DMA（Direct Memory Access）：
  DMA控制器直接搬运数据，CPU只需发起命令
  ┌─────┐      ┌─────────────┐      ┌──────┐
  │ 磁盘 │ ───→ │ DMA控制器    │ ───→ │ 内存  │
  └─────┘      └─────────────┘      └──────┘
        ↑                               ↓
        └───────── CPU发起命令 ──────────┘

  ✓ CPU只需初始化DMA（设置源、目标、长度）
  ✓ DMA传输期间CPU可以做其他事情
  ✓ 传输完成后DMA发送中断通知CPU
```

#### Page Cache（页缓存）

```c
// Page Cache是内核用来缓存磁盘文件的内存区域

读取文件时：
1. 先查Page Cache有没有
2. 有 → 直接返回（避免磁盘IO）
3. 没有 → 从磁盘读取并缓存到Page Cache

写入文件时：
1. 写到Page Cache（异步）
2. 标记为"脏页"
3. 后台线程（pdflush）定期刷回磁盘
```

**Page Cache的作用**：

```
┌──────────────────────────────────────────────────────┐
│                   应用程序                            │
└──────────────────────────────────────────────────────┘
         │ read()/write()
         ↓
┌──────────────────────────────────────────────────────┐
│               VFS (虚拟文件系统)                       │
└──────────────────────────────────────────────────────┘
         │
         ↓
┌──────────────────────────────────────────────────────┐
│               Page Cache（页缓存）                     │
│  ┌───────┐  ┌───────┐  ┌───────┐  ┌──────┐           │
│  │ Page1 │  │ Page2 │  │ Page3 │  │ ...  │           │
│  └───────┘  └───────┘  └───────┘  └──────┘           │
│  ↑ 命中缓存：直接返回（快）                              │
│  ↓ 未命中：触发磁盘IO（慢）                              │
└──────────────────────────────────────────────────────┘
         │
         ↓ 缺页中断
┌──────────────────────────────────────────────────────┐
│               文件系统 (ext4/xfs/...)                 │
└──────────────────────────────────────────────────────┘
         │
         ↓ DMA传输
┌──────────────────────────────────────────────────────┐
│                   磁盘设备                            │
└──────────────────────────────────────────────────────┘
```

#### 零拷贝与DMA/Page Cache的关系

```
sendfile() 利用了DMA + Page Cache：

1. 文件数据被DMA读入Page Cache
2. sendfile()直接将Page Cache数据发送到Socket缓冲区
3. Socket缓冲区数据被DMA发送到网卡

整个过程：
  磁盘 ───────────→ Page Cache ───────────→ Socket缓冲区 ───────────→ 网卡
         [DMA]                [内核直接传输]                 [DMA]

✓ 只有2次DMA拷贝
✓ 数据不经过用户空间
✓ CPU参与度极低
```

#### 查看Page Cache使用情况

```bash
# 查看系统内存使用
free -h
#              total        used        free      shared  buff/cache   available
# Mem:           15Gi       2.0Gi       8.0Gi       100Mi       5.0Gi        13Gi
#                                                                 ↑
#                                                          Page Cache在这里

# 查看某个文件是否在Page Cache中
vmtouch -v file.txt

# 清空Page Cache（慎用！）
echo 3 > /proc/sys/vm/drop_caches
```

---

### 1.6 应用场景对比

#### 四种方案的对比

| 方案             | 拷贝次数                  | 上下文切换 | 使用场景       | 优点   | 缺点                          |
|----------------|-----------------------|-------|------------|------|-----------------------------|
| **read/write** | 4次<br>(2 DMA + 2 CPU) | 4次    | 小文件、需要修改数据 | 灵活   | 性能差                         |
| **sendfile()** | 2次<br>(2 DMA)         | 2次    | 大文件发送      | 最快   | 不能修改数据，<br>Linux限制文件→Socket |
| **splice()**   | 2次<br>(2 DMA)         | 2次    | 任意fd之间传输   | 灵活性高 | 需要管道中转                      |
| **mmap+write** | 3次<br>(2 DMA + 1 CPU) | 4次    | 随机访问、共享内存  | 支持修改 | 比sendfile慢                  |

#### 选型决策树

```
开始
  │
  ├─ 需要修改数据？
  │   ├─ 是 → mmap+write或read/write
  │   └─ 否 → 继续
  │
  ├─ 文件 → Socket？
  │   ├─ 是 → sendfile()  ← 最优选择
  │   └─ 否 → 继续
  │
  ├─ Socket → Socket（代理）？
  │   ├─ 是 → splice()
  │   └─ 否 → 继续
  │
  ├─ 需要随机访问？
  │   ├─ 是 → mmap()
  │   └─ 否 → sendfile()或splice()
  │
  └─ 文件大小？
      ├─ < 4KB → read/write（mmap开销大）
      └─ > 1MB → sendfile()
```

#### 实际项目中的应用

**1. Nginx文件服务器**

```c
// Nginx在Linux上发送文件时使用sendfile()
// nginx/src/os/unix/ngx_linux_sendfile_chain.c

ssize_t ngx_linux_sendfile(ngx_connection_t *c, ngx_buf_t *file, size_t size) 
{
    // ...
    sent = sendfile(c->fd, file->file->fd, &offset, size);
    // ...
}
```

**2. Kafka高性能日志传输**

```java
// Kafka使用Java NIO的transferTo()（底层是sendfile）
// kafka/core/src/main/scala/kafka/log/LogSegment.scala

fileChannel.transferTo(position, count, socketChannel);
// 底层调用：sendfile() on Linux
```

**3. Redis RDB持久化**

```c
// Redis保存RDB文件时使用write()而不是sendfile
// 因为需要压缩、加密等数据转换
// redis/src/rdb.c

rioWrite(&rdb, buf, len);  // 需要修改数据，不能用零拷贝
```

---

### 1.7 高频面试题

#### Q1: 什么是零拷贝？为什么它能提升性能？

**答案**：

零拷贝是指数据在内核空间和用户空间之间传输时，减少不必要的CPU拷贝次数。

**传统方式的问题**：
```
read() + write() 需要4次拷贝：
1. 磁盘 → 内核缓冲区（DMA）
2. 内核缓冲区 → 用户空间（CPU拷贝，浪费！）
3. 用户空间 → Socket缓冲区（CPU拷贝，浪费！）
4. Socket缓冲区 → 网卡（DMA）
```

**零拷贝优化**：
```
sendfile() 只需2次拷贝：
1. 磁盘 → 内核缓冲区（DMA）
2. 内核缓冲区 → 网卡（DMA，或通过DMA gather）
```

**性能提升原因**：
- ✓ 减少CPU拷贝（释放CPU资源）
- ✓ 减少上下文切换（用户态↔内核态）
- ✓ 减少内存占用（不需要用户空间缓冲区）

---

#### Q2: sendfile、splice、mmap有什么区别？

**答案**：

|            | sendfile()                   | splice()                | mmap()           |
|------------|------------------------------|-------------------------|------------------|
| **系统调用**   | sendfile(out_fd, in_fd, ...) | splice(in_fd, pipe_out) | mmap() + write() |
| **拷贝次数**   | 2次DMA                        | 2次DMA                   | 2次DMA + 1次CPU    |
| **适用场景**   | 文件→Socket                    | 任意fd↔管道↔任意fd            | 随机访问、共享内存        |
| **能否修改数据** | ✗                            | ✗                       | ✓                |
| **性能**     | 最快                           | 快（需管道中转）                | 中等               |

**选择建议**：
- 发送文件 → **sendfile()**
- Socket代理 → **splice()**
- 需要修改数据 → **mmap()**

---

#### Q3: 为什么sendfile比read/write快？请从内核角度解释。

**答案**：

**1. 减少CPU拷贝**

```c
// read/write 方式
char buf[4096];
read(file_fd, buf, 4096);   // 内核 → 用户空间（CPU拷贝）
write(sock_fd, buf, 4096);  // 用户空间 → 内核（CPU拷贝）

// sendfile 方式
sendfile(sock_fd, file_fd, NULL, 4096);  // 内核空间直接传输
```

**2. 减少上下文切换**

```
read/write:
  read()调用   → 用户态 → 内核态 → 用户态
  write()调用  → 用户态 → 内核态 → 用户态
  总共4次切换

sendfile():
  sendfile()调用 → 用户态 → 内核态 → 用户态
  总共2次切换
```

**3. 内核内部优化**

```c
// sendfile() 内核实现（简化版）
// fs/read_write.c

SYSCALL_DEFINE4(sendfile, ...) 
{
    // 1. 找到文件在Page Cache中的页面
    struct page *page = find_get_page(mapping, index);

    // 2. 如果不在缓存，触发DMA读取
    if (!page) 
    {
        page = page_cache_alloc();
        // DMA: 磁盘 → Page Cache
    }

    // 3. 直接将Page Cache的页面添加到Socket发送队列
    tcp_sendpage(sock, page, offset, size);

    // 4. 网卡通过DMA gather直接读取Page Cache
    //    （甚至不需要拷贝到Socket缓冲区）
}
```

**性能数据**（1GB文件传输）：
- read/write: 2.5秒
- sendfile: 0.8秒
- 提升: **3倍**

---

#### Q4: Page Cache是什么？它与零拷贝有什么关系？

**答案**：

**Page Cache定义**：
内核用来缓存磁盘文件内容的内存区域，以页面（4KB）为单位。

**作用**：
```
1. 缓存热点数据，避免重复磁盘IO
2. 异步写入（写Page Cache后立即返回，后台刷盘）
3. 预读（readahead）优化顺序读取
```

**与零拷贝的关系**：

```
sendfile() 的完整流程：

1. 检查文件是否在Page Cache
   ↓
2. 未命中 → DMA读取磁盘到Page Cache
   ↓
3. sendfile()直接引用Page Cache的页面
   ↓
4. 网卡通过DMA gather直接读取Page Cache
   ↓
5. 数据传输完成

关键点：数据始终在Page Cache，sendfile()只是传递页面引用
```

**示例**：

```c
// 第一次读取：触发磁盘IO
sendfile(sock, fd, NULL, 1GB);  // 慢（需要读磁盘）

// 第二次读取：Page Cache命中
sendfile(sock, fd, NULL, 1GB);  // 快（直接从内存）
```

**查看Page Cache命中率**：

```bash
# 安装pcstat工具
go get github.com/tobert/pcstat/pcstat

# 查看文件缓存情况
pcstat file.txt
# file.txt: 1024 pages (4MB) cached (100%)
```

---

#### Q5: 实际项目中如何使用零拷贝？有什么坑？

**答案**：

**1. 典型应用场景**

```c
// 场景1：HTTP文件服务器（类似Nginx）
void serve_static_file(int client_fd, const char *path) 
{
    int file_fd = open(path, O_RDONLY);
    struct stat st;
    fstat(file_fd, &st);

    // 发送HTTP头
    char header[256];
    sprintf(header, "HTTP/1.1 200 OK\r\nContent-Length: %ld\r\n\r\n", st.st_size);
    write(client_fd, header, strlen(header));

    // 零拷贝发送文件
    sendfile(client_fd, file_fd, NULL, st.st_size);  // ← 关键

    close(file_fd);
}

// 场景2：TCP代理（类似HAProxy）
void proxy_forward(int client_fd, int backend_fd) 
{
    int pipefd[2];
    pipe(pipefd);

    // client → pipe → backend (零拷贝)
    splice(client_fd, NULL, pipefd[1], NULL, 65536, SPLICE_F_MOVE);
    splice(pipefd[0], NULL, backend_fd, NULL, 65536, SPLICE_F_MOVE);

    close(pipefd[0]);
    close(pipefd[1]);
}

// 场景3：日志收集（需要修改数据则不能用零拷贝）
void collect_log(int client_fd) 
{
    char buf[4096];
    int n = read(client_fd, buf, sizeof(buf));

    // 需要解析、过滤、加时间戳等
    add_timestamp(buf, &n);
    filter_sensitive_data(buf, &n);

    // 不能用sendfile，因为数据被修改了
    write(log_fd, buf, n);
}
```

**2. 常见的坑**

**坑1：sendfile不支持所有文件系统**

```c
// 某些文件系统（如NFS）不支持sendfile
ssize_t n = sendfile(sock_fd, file_fd, NULL, size);
if (n < 0 && errno == EINVAL) 
{
    // 回退到传统方式
    char buf[4096];
    while ((n = read(file_fd, buf, sizeof(buf))) > 0) 
    {
        write(sock_fd, buf, n);
    }
}
```

**坑2：大文件发送被中断**

```c
// 错误：大文件一次sendfile可能被信号中断
sendfile(sock_fd, file_fd, NULL, 10GB);  // ✗ 可能失败

// 正确：循环发送
off_t offset = 0;
size_t remain = file_size;
while (remain > 0) 
{
    ssize_t sent = sendfile(sock_fd, file_fd, &offset, remain);
    if (sent < 0) 
    {
        if (errno == EINTR) continue;  // 被信号中断，重试
        if (errno == EAGAIN) // 非阻塞socket，稍后重试
        {         
            // 等待socket可写
            continue;
        }
        perror("sendfile");
        break;
    }
    remain -= sent;
}
```

**坑3：splice需要一端是管道**

```c
// 错误：splice直接连接两个socket
splice(client_fd, NULL, backend_fd, NULL, 4096, 0);  // ✗ 失败

// 正确：必须通过管道中转
int pipefd[2];
pipe(pipefd);
splice(client_fd, NULL, pipefd[1], NULL, 4096, SPLICE_F_MOVE);
splice(pipefd[0], NULL, backend_fd, NULL, 4096, SPLICE_F_MOVE);
```

**坑4：mmap大文件可能导致地址空间不足**

```c
// 32位系统地址空间只有4GB
void *mapped = mmap(NULL, 100GB, ...);  // ✗ 失败

// 解决：分段映射
size_t chunk_size = 1GB;
for (off_t offset = 0; offset < file_size; offset += chunk_size) 
{
    void *mapped = mmap(NULL, chunk_size, PROT_READ, MAP_PRIVATE, file_fd, offset);
    // 处理这1GB数据
    munmap(mapped, chunk_size);
}
```

**3. 性能优化建议**

```c
// 1. 开启TCP_CORK减少小包
int cork = 1;
setsockopt(sock_fd, IPPROTO_TCP, TCP_CORK, &cork, sizeof(cork));

// 2. 发送HTTP头
write(sock_fd, header, header_len);

// 3. 零拷贝发送文件
sendfile(sock_fd, file_fd, NULL, file_size);

// 4. 关闭TCP_CORK，立即发送
cork = 0;
setsockopt(sock_fd, IPPROTO_TCP, TCP_CORK, &cork, sizeof(cork));
```

---

#### Q6: 为什么Java NIO的transferTo()比传统IO快？底层原理是什么？

**答案**：

**Java代码对比**：

```java
// 传统IO
FileInputStream in = new FileInputStream("file.txt");
OutputStream out = socket.getOutputStream();
byte[] buf = new byte[4096];
int n;
while ((n = in.read(buf)) > 0) 
{
    out.write(buf, 0, n);  // ✗ 4次拷贝
}

// NIO零拷贝
FileChannel fileChannel = new FileInputStream("file.txt").getChannel();
SocketChannel socketChannel = socket.getChannel();
fileChannel.transferTo(0, fileChannel.size(), socketChannel);  // ✓ 2次拷贝
```

**底层原理**（Linux）：

```java
// OpenJDK: src/java.base/linux/native/libnio/ch/FileChannelImpl.c

JNIEXPORT jlong JNICALL
Java_sun_nio_ch_FileChannelImpl_transferTo0(JNIEnv *env, jobject this, jint srcFD, jlong position, jlong count, jint dstFD) 
{
    // 调用Linux的sendfile系统调用
    ssize_t n = sendfile(dstFD, srcFD, &offset, (size_t)count);
    return (jlong)n;
}
```

**Windows上的实现**：

```java
// Windows使用TransmitFile() API
// src/java.base/windows/native/libnio/ch/FileChannelImpl.c

BOOL success = TransmitFile(
    (SOCKET)dstFD,      // socket
    (HANDLE)srcFD,      // 文件句柄
    count,              // 字节数
    0, NULL, NULL, 0
);
```

**性能数据**：

```java
// 发送1GB文件
// 传统IO:      5.2秒
// transferTo:  1.8秒
// 提升:        2.9倍
```

---


## 2. 高性能网络编程

### 2.1 DPDK（Data Plane Development Kit）

#### 什么是DPDK？

DPDK是Intel开源的**用户态网络框架**，通过**绕过内核协议栈**直接操作网卡，实现极致性能。

```
传统网络栈：
应用程序 → Socket API → 内核协议栈 → 网卡驱动 → 网卡
         (系统调用)    (TCP/IP栈)     (中断处理)

DPDK：
应用程序 → DPDK库 → 网卡（PMD轮询驱动）
         (用户态)  (无中断，轮询)

✓ 绕过内核（避免系统调用、上下文切换）
✓ 零拷贝（用户态直接访问网卡DMA区域）
✓ 轮询模式（避免中断开销）
✓ 大页内存（减少TLB miss）
✓ CPU亲和性（绑定核心）
```

#### 核心技术

**1. PMD（Poll Mode Driver）- 轮询模式驱动**

```c
// 传统网卡驱动（中断模式）
void network_rx_interrupt_handler() 
{
    // 收到数据包 → 触发中断 → CPU处理
    // 问题：高PPS时中断风暴
}

// DPDK（轮询模式）
while (1) 
{
    // CPU不停地主动查询网卡是否有数据
    nb_rx = rte_eth_rx_burst(port, queue, pkts, BURST_SIZE);
    if (nb_rx > 0) 
    {
        process_packets(pkts, nb_rx);
    }
    // 问题：CPU使用率100%，但延迟极低
}
```

**2. Hugepage（大页内存）**

```bash
# 普通页面：4KB
# 问题：频繁访问内存导致TLB miss

# 大页内存：2MB或1GB
# 配置DPDK使用大页
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# DPDK分配内存时使用大页
rte_mempool_create(..., MEMPOOL_F_NO_CACHE_ALIGN);

# 结果：TLB miss减少90%，性能提升15-20%
```

**3. UIO/VFIO（用户态IO）**

```
传统：
  应用 → 内核 → 驱动 → 网卡
       (系统调用)

DPDK + UIO/VFIO：
  应用 → DPDK → 网卡（通过mmap直接访问）
       (用户态)

// 应用程序直接读写网卡寄存器
volatile uint32_t *nic_reg = mmap(...);
*nic_reg = command;  // 直接操作网卡
```

#### DPDK示例代码

```c
// dpdk_l2fwd.c - 简单的L2转发（学习用）
#include <rte_eal.h>
#include <rte_ethdev.h>
#include <rte_mbuf.h>

#define RX_RING_SIZE 1024
#define TX_RING_SIZE 1024
#define NUM_MBUFS 8191
#define MBUF_CACHE_SIZE 250
#define BURST_SIZE 32

// 初始化DPDK
int main(int argc, char *argv[]) 
{
    struct rte_mempool *mbuf_pool;
    uint16_t nb_ports;
    uint16_t portid;

    // 1. 初始化EAL（Environment Abstraction Layer）
    int ret = rte_eal_init(argc, argv);
    if (ret < 0) 
    {
        rte_exit(EXIT_FAILURE, "EAL initialization failed\n");
    }

    // 2. 创建mbuf内存池
    mbuf_pool = rte_pktmbuf_pool_create("MBUF_POOL", NUM_MBUFS, MBUF_CACHE_SIZE, 0, RTE_MBUF_DEFAULT_BUF_SIZE, rte_socket_id());

    // 3. 获取网卡数量
    nb_ports = rte_eth_dev_count_avail();
    printf("Found %u ports\n", nb_ports);

    // 4. 初始化每个网口
    RTE_ETH_FOREACH_DEV(portid) 
    {
        if (port_init(portid, mbuf_pool) != 0) 
        {
            rte_exit(EXIT_FAILURE, "Cannot init port %u\n", portid);
        }
    }

    // 5. 主循环：L2转发
    printf("Starting L2 forwarding...\n");
    lcore_main();

    return 0;
}

// 初始化网口
static int port_init(uint16_t port, struct rte_mempool *mbuf_pool) 
{
    struct rte_eth_conf port_conf = {0};
    uint16_t nb_rxd = RX_RING_SIZE;
    uint16_t nb_txd = TX_RING_SIZE;
    int retval;
    uint16_t q;

    // 1. 配置网口（1个RX队列，1个TX队列）
    retval = rte_eth_dev_configure(port, 1, 1, &port_conf);
    if (retval != 0) return retval;

    // 2. 分配RX队列
    retval = rte_eth_rx_queue_setup(port, 0, nb_rxd,
                rte_eth_dev_socket_id(port), NULL, mbuf_pool);
    if (retval < 0) return retval;

    // 3. 分配TX队列
    retval = rte_eth_tx_queue_setup(port, 0, nb_txd,
                rte_eth_dev_socket_id(port), NULL);
    if (retval < 0) return retval;

    // 4. 启动网口
    retval = rte_eth_dev_start(port);
    if (retval < 0) return retval;

    // 5. 开启混杂模式（接收所有数据包）
    rte_eth_promiscuous_enable(port);

    return 0;
}

// 主循环：L2转发
static void lcore_main(void) 
{
    uint16_t port;
    struct rte_mbuf *bufs[BURST_SIZE];

    // 检查当前逻辑核心是否是主核心
    printf("Core %u doing L2 forwarding\n", rte_lcore_id());

    while (1) 
    {
        // 遍历所有网口
        RTE_ETH_FOREACH_DEV(port) 
        {
            // 1. 批量接收数据包（关键：burst批量操作）
            const uint16_t nb_rx = rte_eth_rx_burst(port, 0, bufs, BURST_SIZE);

            if (nb_rx == 0) continue;

            // 2. L2转发：从哪个口收到，就从哪个口发回去
            //    （实际应用会查MAC表决定转发端口）
            const uint16_t nb_tx = rte_eth_tx_burst(port ^ 1, 0, bufs, nb_rx);

            // 3. 释放未发送成功的mbuf
            if (unlikely(nb_tx < nb_rx)) 
            {
                uint16_t buf;
                for (buf = nb_tx; buf < nb_rx; buf++) 
                {
                    rte_pktmbuf_free(bufs[buf]);
                }
            }
        }
    }
}
```

**编译运行**：

```bash
# 1. 安装DPDK
sudo apt install dpdk dpdk-dev

# 2. 配置大页内存
echo 1024 | sudo tee /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# 3. 绑定网卡到DPDK（警告：会断网！）
sudo dpdk-devbind.py --bind=vfio-pci 0000:01:00.0

# 4. 编译
gcc dpdk_l2fwd.c -o l2fwd \
    $(pkg-config --cflags --libs libdpdk)

# 5. 运行（需要root）
sudo ./l2fwd -l 0-1 -n 4
```

#### DPDK性能数据

```
普通Linux网络栈：
- PPS: ~1-2 Mpps (百万包每秒)
- 延迟: 50-100 μs
- CPU: 中断处理消耗大量CPU

DPDK：
- PPS: ~30-80 Mpps (10G网卡)
- 延迟: <10 μs
- CPU: 100%使用率（轮询），但吞吐量高

性能提升：30-50倍
```

#### 应用场景

```
✓ 高频交易系统（低延迟）
✓ 虚拟交换机（OVS-DPDK）
✓ 防火墙/IDS（高PPS处理）
✓ 5G基站（uRLLC低延迟）
✓ CDN加速（边缘节点）

✗ 普通Web服务器（不需要极致性能，浪费CPU）
✗ 桌面应用（没必要）
```

---

### 2.2 io_uring（新异步IO框架）

#### 什么是io_uring？

io_uring是Linux 5.1引入的**全新异步IO框架**，解决了传统异步IO（AIO）的所有问题。

```
传统异步IO的问题：
- Linux AIO只支持O_DIRECT（绕过Page Cache，不适合大多数场景）
- API复杂难用
- 性能不佳

io_uring的优势：
✓ 真正的异步IO（所有系统调用都支持）
✓ 零系统调用（通过共享内存环形队列）
✓ 支持批量操作（提交/完成队列）
✓ 性能接近DPDK（但更通用）
```

#### 核心原理：双环形队列

```
用户空间 ←→ 内核空间（通过mmap共享内存）

┌──────────────────────────────────────────────────┐
│                  用户空间                         │
│  ┌─────────────────────────────────────────┐     │
│  │ 提交队列（SQ, Submission Queue）          │     │
│  │  [SQE0] [SQE1] [SQE2] ...               │     │
│  │  ↓ 应用写入IO请求                         │     │ 
│  └─────────────────────────────────────────┘     │
│                                                  │
│  ┌─────────────────────────────────────────┐     │
│  │ 完成队列（CQ, Completion Queue）          │     │
│  │  [CQE0] [CQE1] [CQE2] ...               │     │
│  │  ↑ 应用读取IO结果                         │     │
│  └─────────────────────────────────────────┘     │
└──────────────────────────────────────────────────┘
         ↓ io_uring_enter()        ↑
         ↓ (可选，或直接轮询)         ↑
┌──────────────────────────────────────────────────┐
│                  内核空间                         │
│  内核从SQ读取请求 → 执行IO → 写入结果到CQ             │
└──────────────────────────────────────────────────┘

关键：SQ和CQ都是mmap共享内存，减少拷贝
```

#### io_uring基本使用

```c
// io_uring_example.c
#include <stdio.h>
#include <liburing.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

#define QUEUE_DEPTH 32
#define BLOCK_SIZE 4096

// 使用io_uring读取文件
int read_file_with_io_uring(const char *filename) 
{
    struct io_uring ring;
    struct io_uring_sqe *sqe;
    struct io_uring_cqe *cqe;
    int fd, ret;
    char *buffer;

    // 1. 初始化io_uring
    ret = io_uring_queue_init(QUEUE_DEPTH, &ring, 0);
    if (ret < 0) 
    {
        perror("io_uring_queue_init");
        return -1;
    }

    // 2. 打开文件
    fd = open(filename, O_RDONLY);
    if (fd < 0) 
    {
        perror("open");
        io_uring_queue_exit(&ring);
        return -1;
    }

    // 3. 分配缓冲区
    buffer = malloc(BLOCK_SIZE);

    // 4. 获取SQE（Submission Queue Entry）
    sqe = io_uring_get_sqe(&ring);
    if (!sqe) 
    {
        fprintf(stderr, "Failed to get SQE\n");
        goto cleanup;
    }

    // 5. 准备读取操作
    io_uring_prep_read(sqe, fd, buffer, BLOCK_SIZE, 0);

    // 设置用户数据（用于识别请求）
    io_uring_sqe_set_data(sqe, buffer);

    // 6. 提交请求
    ret = io_uring_submit(&ring);
    if (ret < 0) 
    {
        perror("io_uring_submit");
        goto cleanup;
    }

    printf("✓ Submitted %d requests\n", ret);

    // 7. 等待完成
    ret = io_uring_wait_cqe(&ring, &cqe);
    if (ret < 0) {
        perror("io_uring_wait_cqe");
        goto cleanup;
    }

    // 8. 处理结果
    if (cqe->res < 0) 
    {
        fprintf(stderr, "IO failed: %s\n", strerror(-cqe->res));
    } 
    else 
    {
        printf("✓ Read %d bytes\n", cqe->res);
        printf("Content: %.100s...\n", buffer);
    }

    // 9. 标记CQE已处理
    io_uring_cqe_seen(&ring, cqe);

cleanup:
    free(buffer);
    close(fd);
    io_uring_queue_exit(&ring);
    return 0;
}

int main() 
{
    read_file_with_io_uring("test.txt");
    return 0;
}
```

#### 网络IO示例：Echo服务器

```c
// io_uring_echo_server.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <liburing.h>

#define QUEUE_DEPTH 256
#define BUFFER_SIZE 4096

enum 
{
    EVENT_TYPE_ACCEPT,
    EVENT_TYPE_READ,
    EVENT_TYPE_WRITE,
};

typedef struct 
{
    int type;
    int fd;
    char *buffer;
    size_t size;
} request_t;

// 添加accept请求
void add_accept_request(struct io_uring *ring, int listen_fd, struct sockaddr_in *client_addr, socklen_t *client_len) 
{
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
    
    request_t *req = malloc(sizeof(request_t));
    req->type = EVENT_TYPE_ACCEPT;
    req->fd = listen_fd;
    
    io_uring_prep_accept(sqe, listen_fd, (struct sockaddr*)client_addr, client_len, 0);
    io_uring_sqe_set_data(sqe, req);
}

// 添加read请求
void add_read_request(struct io_uring *ring, int client_fd) 
{
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
    
    request_t *req = malloc(sizeof(request_t));
    req->type = EVENT_TYPE_READ;
    req->fd = client_fd;
    req->buffer = malloc(BUFFER_SIZE);
    
    io_uring_prep_recv(sqe, client_fd, req->buffer, BUFFER_SIZE, 0);
    io_uring_sqe_set_data(sqe, req);
}

// 添加write请求
void add_write_request(struct io_uring *ring, int client_fd, char *buffer, size_t size) 
{
    struct io_uring_sqe *sqe = io_uring_get_sqe(ring);
    
    request_t *req = malloc(sizeof(request_t));
    req->type = EVENT_TYPE_WRITE;
    req->fd = client_fd;
    req->buffer = buffer;
    req->size = size;
    
    io_uring_prep_send(sqe, client_fd, buffer, size, 0);
    io_uring_sqe_set_data(sqe, req);
}

int main() 
{
    struct io_uring ring;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_len = sizeof(client_addr);
    int listen_fd;

    // 1. 初始化io_uring
    if (io_uring_queue_init(QUEUE_DEPTH, &ring, 0) < 0) 
    {
        perror("io_uring_queue_init");
        return 1;
    }

    // 2. 创建监听socket
    listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    int reuse = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(8080);
    server_addr.sin_addr.s_addr = INADDR_ANY;

    bind(listen_fd, (struct sockaddr*)&server_addr, sizeof(server_addr));
    listen(listen_fd, 128);

    printf("io_uring echo server listening on port 8080...\n");

    // 3. 添加第一个accept请求
    add_accept_request(&ring, listen_fd, &client_addr, &client_len);
    io_uring_submit(&ring);

    // 4. 事件循环
    while (1) 
    {
        struct io_uring_cqe *cqe;
        int ret;

        // 等待完成事件
        ret = io_uring_wait_cqe(&ring, &cqe);
        if (ret < 0) 
        {
            perror("io_uring_wait_cqe");
            break;
        }

        request_t *req = (request_t*)io_uring_cqe_get_data(cqe);

        switch (req->type) 
        {
            case EVENT_TYPE_ACCEPT: 
            {
                // 新连接
                int client_fd = cqe->res;
                if (client_fd >= 0) 
                {
                    printf("✓ New connection: fd=%d\n", client_fd);
                    add_read_request(&ring, client_fd);
                }
                // 继续accept下一个连接
                add_accept_request(&ring, listen_fd, &client_addr, &client_len);
                break;
            }

            case EVENT_TYPE_READ: 
            {
                // 收到数据
                int bytes_read = cqe->res;
                if (bytes_read > 0) 
                {
                    printf("← Received %d bytes from fd=%d\n", bytes_read, req->fd);
                    // Echo back
                    add_write_request(&ring, req->fd, req->buffer, bytes_read);
                } 
                else 
                {
                    // 连接关闭
                    printf("✗ Connection closed: fd=%d\n", req->fd);
                    close(req->fd);
                    free(req->buffer);
                }
                break;
            }

            case EVENT_TYPE_WRITE: 
            {
                // 发送完成
                int bytes_written = cqe->res;
                if (bytes_written > 0) 
                {
                    printf("→ Sent %d bytes to fd=%d\n", bytes_written, req->fd);
                }
                // 继续读取
                add_read_request(&ring, req->fd);
                free(req->buffer);
                break;
            }
        }

        free(req);
        io_uring_cqe_seen(&ring, cqe);

        // 提交所有挂起的请求
        io_uring_submit(&ring);
    }

    io_uring_queue_exit(&ring);
    close(listen_fd);
    return 0;
}
```

**编译运行**：

```bash
# 安装liburing
sudo apt install liburing-dev

# 编译
gcc io_uring_echo_server.c -o io_uring_echo -luring

# 运行
./io_uring_echo

# 测试
echo "Hello io_uring" | nc localhost 8080
```

#### io_uring性能优势

```
传统epoll：
- 每次epoll_wait()都是系统调用
- accept/recv/send都是系统调用
- 处理10K连接：10K次系统调用/秒

io_uring：
- 批量提交请求：1次io_uring_submit()提交100个请求
- 批量收割结果：1次io_uring_wait_cqe()获取100个结果
- 处理10K连接：~100次系统调用/秒（减少100倍）

性能提升：
- 小包场景：提升30-50%
- 大文件IO：提升200-300%
- 延迟：降低20-30%
```

#### io_uring的高级特性

**1. SQPOLL模式（零系统调用）**

```c
// 内核线程轮询SQ队列，应用无需调用io_uring_submit()
struct io_uring_params params = {0};
params.flags = IORING_SETUP_SQPOLL;  // 开启SQPOLL
params.sq_thread_idle = 2000;        // 空闲2秒后睡眠

io_uring_queue_init_params(QUEUE_DEPTH, &ring, &params);

// 之后无需调用io_uring_submit()
// 直接写入SQE，内核线程自动处理
sqe = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe, fd, buf, size, 0);
// 内核线程会自动发现并执行
```

**2. 链式请求（IOSQE_IO_LINK）**

```c
// 读取文件后立即写入socket（原子操作）
struct io_uring_sqe *sqe1, *sqe2;

// 请求1：读取文件
sqe1 = io_uring_get_sqe(&ring);
io_uring_prep_read(sqe1, file_fd, buf, size, 0);
sqe1->flags |= IOSQE_IO_LINK;  // 标记为链式

// 请求2：写入socket（只有请求1成功才执行）
sqe2 = io_uring_get_sqe(&ring);
io_uring_prep_write(sqe2, sock_fd, buf, size, 0);

// 一次提交，内核保证顺序执行
io_uring_submit(&ring);
```

**3. 固定缓冲区（Registered Buffers）**

```c
// 预注册缓冲区，避免每次IO都pin内存
struct iovec iov[16];
for (int i = 0; i < 16; i++) 
{
    iov[i].iov_base = malloc(4096);
    iov[i].iov_len = 4096;
}

// 注册到内核
io_uring_register_buffers(&ring, iov, 16);

// 使用固定缓冲区（性能提升10-15%）
sqe = io_uring_get_sqe(&ring);
io_uring_prep_read_fixed(sqe, fd, iov[0].iov_base, 4096, 0, 0);
//                                                          ↑ 缓冲区索引
```

---

### 2.3 XDP（eXpress Data Path）+ eBPF

#### 什么是XDP？

XDP是Linux内核中**最早处理数据包的hook点**，可以在网卡驱动层直接处理/丢弃/转发数据包。

```
传统网络栈：
网卡 → 驱动 → 内核协议栈 → netfilter → Socket
              ↑ 这里才能处理（延迟高）

XDP：
网卡 → 驱动 → XDP程序 → 决定：丢弃/通过/转发
              ↑ 最早处理点（延迟低）

处理点对比：
- XDP:        数据包到达后立即处理（<50个CPU周期）
- iptables:   在netfilter中处理（~1000个CPU周期）
- 应用程序:    在Socket recv()中处理（~5000个CPU周期）
```

#### XDP程序的返回值

```c
enum xdp_action 
{
    XDP_ABORTED = 0,  // 错误，丢弃包
    XDP_DROP,         // 丢弃包（用于DDoS防御）
    XDP_PASS,         // 允许通过，继续内核协议栈
    XDP_TX,           // 从收包的网卡转发回去（反射攻击防御）
    XDP_REDIRECT,     // 转发到其他网卡或CPU（负载均衡）
};
```

#### XDP示例：简单的包过滤

```c
// xdp_drop_tcp.c - 丢弃所有TCP SYN包（用于防护SYN Flood）
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/tcp.h>
#include <bpf/bpf_helpers.h>

SEC("xdp")
int xdp_drop_syn(struct xdp_md *ctx) 
{
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;

    // 1. 解析以太网头
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end) 
    {
        return XDP_PASS;  // 包太小，放行
    }

    // 2. 检查是否是IP包
    if (eth->h_proto != __builtin_bswap16(ETH_P_IP)) 
    {
        return XDP_PASS;  // 不是IP包，放行
    }

    // 3. 解析IP头
    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end) 
    {
        return XDP_PASS;
    }

    // 4. 检查是否是TCP
    if (ip->protocol != IPPROTO_TCP) 
    {
        return XDP_PASS;  // 不是TCP，放行
    }

    // 5. 解析TCP头
    struct tcphdr *tcp = (void *)ip + (ip->ihl * 4);
    if ((void *)(tcp + 1) > data_end) 
    {
        return XDP_PASS;
    }

    // 6. 检查是否是SYN包
    if (tcp->syn && !tcp->ack) 
    {
        // 这是SYN包，丢弃！
        bpf_printk("Dropped SYN from %pI4:%d\n", &ip->saddr, __builtin_bswap16(tcp->source));
        return XDP_DROP;
    }

    return XDP_PASS;  // 其他包放行
}

char _license[] SEC("license") = "GPL";
```

#### 加载XDP程序

```bash
# 1. 安装工具
sudo apt install clang llvm libbpf-dev linux-headers-$(uname -r)

# 2. 编译XDP程序为BPF字节码
clang -O2 -g -target bpf -c xdp_drop_tcp.c -o xdp_drop_tcp.o

# 3. 加载到网卡（例如eth0）
sudo ip link set dev eth0 xdp obj xdp_drop_tcp.o sec xdp

# 4. 查看XDP程序
sudo ip link show dev eth0
# xdp/generic ← 表示XDP已加载

# 5. 查看统计（需要在XDP程序中实现）
sudo bpftool prog show
sudo bpftool map show

# 6. 卸载XDP程序
sudo ip link set dev eth0 xdp off
```

#### 完整示例：XDP负载均衡器

```c
// xdp_lb.c - 简单的L4负载均衡器
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <linux/tcp.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

// 后端服务器列表（BPF Map）
struct 
{
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __type(key, __u32);
    __type(value, __u32);  // 后端IP地址
    __uint(max_entries, 4);
} backend_servers SEC(".maps");

// 连接跟踪表（记录客户端→后端的映射）
struct 
{
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __type(key, __u32);    // 客户端IP
    __type(value, __u32);  // 后端IP
    __uint(max_entries, 65536);
} conn_track SEC(".maps");

// 统计计数器
struct 
{
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __type(key, __u32);
    __type(value, __u64);
    __uint(max_entries, 2);
} stats SEC(".maps");

#define STAT_PACKETS 0
#define STAT_DROPPED 1

static __always_inline __u32 hash_ip(__u32 ip) 
{
    return ip % 4;  // 简单哈希到4个后端
}

SEC("xdp")
int xdp_loadbalancer(struct xdp_md *ctx) 
{
    void *data_end = (void *)(long)ctx->data_end;
    void *data = (void *)(long)ctx->data;
    __u64 *counter;
    __u32 key;

    // 统计：总包数
    key = STAT_PACKETS;
    counter = bpf_map_lookup_elem(&stats, &key);
    if (counter) 
    {
        __sync_fetch_and_add(counter, 1);
    }

    // 解析以太网头
    struct ethhdr *eth = data;
    if ((void *)(eth + 1) > data_end) return XDP_DROP;

    // 只处理IP包
    if (eth->h_proto != bpf_htons(ETH_P_IP)) return XDP_PASS;

    // 解析IP头
    struct iphdr *ip = (void *)(eth + 1);
    if ((void *)(ip + 1) > data_end) return XDP_DROP;

    // 只处理TCP
    if (ip->protocol != IPPROTO_TCP) return XDP_PASS;

    __u32 client_ip = ip->saddr;

    // 1. 查找是否已有连接跟踪
    __u32 *backend_ip = bpf_map_lookup_elem(&conn_track, &client_ip);

    if (!backend_ip) 
    {
        // 2. 没有跟踪记录，分配新后端（一致性哈希）
        __u32 backend_idx = hash_ip(client_ip);
        backend_ip = bpf_map_lookup_elem(&backend_servers, &backend_idx);

        if (!backend_ip) 
        {
            // 没有可用后端
            key = STAT_DROPPED;
            counter = bpf_map_lookup_elem(&stats, &key);
            if (counter) __sync_fetch_and_add(counter, 1);
            return XDP_DROP;
        }

        // 3. 记录连接跟踪
        bpf_map_update_elem(&conn_track, &client_ip, backend_ip, BPF_ANY);
    }

    // 4. 修改目标IP为后端IP（DNAT）
    __u32 old_ip = ip->daddr;
    ip->daddr = *backend_ip;

    // 5. 重新计算IP校验和
    __u32 csum = ip->check;
    csum = bpf_csum_diff(&old_ip, sizeof(old_ip), 
                         backend_ip, sizeof(*backend_ip), ~csum);
    ip->check = csum_fold_helper(csum);

    // 6. 转发到原网卡（或XDP_REDIRECT到其他网卡）
    return XDP_TX;  // 从原网卡转发回去
}

char _license[] SEC("license") = "GPL";
```

**用户态程序：配置后端**

```c
// xdp_lb_user.c - 配置后端服务器
#include <stdio.h>
#include <arpa/inet.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>

int main(int argc, char *argv[]) 
{
    struct bpf_object *obj;
    int map_fd;

    // 1. 加载XDP程序
    obj = bpf_object__open_file("xdp_lb.o", NULL);
    bpf_object__load(obj);

    // 2. 获取map fd
    map_fd = bpf_object__find_map_fd_by_name(obj, "backend_servers");

    // 3. 配置后端服务器
    const char *backends[] = 
    {
        "192.168.1.10",
        "192.168.1.11",
        "192.168.1.12",
        "192.168.1.13",
    };

    for (int i = 0; i < 4; i++) 
    {
        __u32 key = i;
        __u32 value;
        inet_pton(AF_INET, backends[i], &value);
        bpf_map_update_elem(map_fd, &key, &value, BPF_ANY);
        printf("✓ Backend %d: %s\n", i, backends[i]);
    }

    // 4. 附加到网卡
    int ifindex = if_nametoindex("eth0");
    struct bpf_program *prog = bpf_object__find_program_by_name(obj, "xdp_loadbalancer");
    int prog_fd = bpf_program__fd(prog);

    bpf_set_link_xdp_fd(ifindex, prog_fd, 0);

    printf("✓ XDP load balancer attached to eth0\n");
    printf("Press Ctrl+C to detach...\n");

    // 5. 定期打印统计
    int stats_fd = bpf_object__find_map_fd_by_name(obj, "stats");
    while (1) 
    {
        sleep(5);

        __u32 key = 0;
        __u64 packets = 0;
        bpf_map_lookup_elem(stats_fd, &key, &packets);

        key = 1;
        __u64 dropped = 0;
        bpf_map_lookup_elem(stats_fd, &key, &dropped);

        printf("Stats: packets=%llu, dropped=%llu\n", packets, dropped);
    }

    return 0;
}
```

#### XDP性能数据

```
场景：DDoS防御（丢弃恶意流量）

传统iptables：
- 丢包率：~1-2 Mpps
- CPU：100%
- 延迟：~50 μs

XDP：
- 丢包率：~20-30 Mpps（10G网卡）
- CPU：50%
- 延迟：<1 μs

性能提升：20-30倍
```

---

### 2.4 性能对比与选型

#### 四种方案对比

| 方案           | 性能(PPS) | 延迟       | CPU使用率 | 适用场景          | 学习曲线 |
|--------------|---------|----------|--------|---------------|------|
| **传统epoll**  | 1-2M    | 50-100μs | 中等     | 通用网络服务        | 低    |
| **io_uring** | 5-10M   | 10-20μs  | 中等     | 高性能网络服务       | 中    |
| **XDP**      | 20-30M  | <1μs     | 低      | DDoS防御、L4负载均衡 | 高    |
| **DPDK**     | 30-80M  | <1μs     | 100%   | 极致性能场景        | 非常高  |

#### 选型决策树

```
开始
  │
  ├─ 需要极致性能（>10Mpps）？
  │   ├─ 是 → 继续
  │   └─ 否 → io_uring或epoll
  │
  ├─ 需要完全绕过内核？
  │   ├─ 是 → DPDK
  │   └─ 否 → 继续
  │
  ├─ 只需包过滤/转发（无需应用层处理）？
  │   ├─ 是 → XDP（最适合）
  │   └─ 否 → DPDK
  │
  └─ 能接受100% CPU使用率？
      ├─ 是 → DPDK
      └─ 否 → XDP或io_uring
```

#### 实际应用案例

**1. Cloudflare DDoS防御**
```
使用XDP在边缘服务器上：
- 检测并丢弃DDoS流量（SYN Flood、DNS Amplification）
- 性能：每台服务器抵御300Gbps攻击
- 成本：无需专用硬件
```

**2. Facebook Katran（L4负载均衡器）**
```
使用XDP实现高性能负载均衡：
- 性能：100Gbps网络，99.9%线速
- 延迟：<10μs
- GitHub: https://github.com/facebookincubator/katran
```

**3. Nginx + io_uring**
```
从epoll迁移到io_uring：
- 性能提升：30-40%
- 延迟降低：20%
- 版本：Nginx 1.25+实验性支持
```

**4. 高频交易系统**
```
使用DPDK构建：
- 延迟：端到端<1μs
- 吞吐：处理数百万订单/秒
- 成本：独占多核CPU
```

---

### 2.5 高频面试题

#### Q1: DPDK为什么比传统网络栈快？

**答案**：

DPDK快在以下几个方面：

**1. 绕过内核（最核心）**
```
传统：应用 → 系统调用 → 内核 → 驱动 → 网卡
      (上下文切换)  (协议栈处理)

DPDK：应用 → DPDK库 → 网卡（PMD）
      (用户态)   (无系统调用)

✓ 避免系统调用开销
✓ 避免用户态↔内核态切换
✓ 避免内核协议栈处理
```

**2. 轮询模式（PMD）**
```
传统：中断驱动
  - 收到包 → 触发中断 → CPU处理
  - 高PPS时中断风暴，性能崩溃

DPDK：轮询模式
  - CPU不停查询网卡是否有数据
  - 延迟极低（<10μs）
  - 代价：CPU 100%使用率
```

**3. 批量处理（Batch）**
```c
// 传统：一次处理一个包
recv(fd, buf, size, 0);  // 每个包一次系统调用

// DPDK：批量处理
rte_eth_rx_burst(port, queue, pkts, 32);  // 一次获取32个包
// 减少函数调用开销，提高缓存命中率
```

**4. 大页内存（Hugepage）**
```
传统：4KB页面 → TLB频繁miss
DPDK：2MB/1GB大页 → TLB miss减少90%

性能提升：15-20%
```

**5. CPU亲和性**
```c
// 绑定线程到特定CPU核心
rte_eal_remote_launch(worker, arg, lcore_id);

✓ 避免线程迁移
✓ 提高L1/L2缓存命中率
✓ 提高性能5-10%
```

---

#### Q2: io_uring相比epoll有什么优势？

**答案**：

**1. 减少系统调用次数**

```c
// epoll：每个操作都需要系统调用
while (1) 
{
    int n = epoll_wait(epfd, events, ...);  // 系统调用1
    for (int i = 0; i < n; i++) 
    {
        int fd = events[i].data.fd;
        recv(fd, buf, size, 0);            // 系统调用2
        send(fd, buf, size, 0);            // 系统调用3
    }
}
// 处理1000个连接：至少3000次系统调用

// io_uring：批量操作
while (1) 
{
    // 批量提交100个请求
    for (int i = 0; i < 100; i++) 
    {
        sqe = io_uring_get_sqe(&ring);
        io_uring_prep_recv(sqe, ...);
    }
    io_uring_submit(&ring);  // 1次系统调用提交100个

    // 批量收割100个结果
    for (int i = 0; i < 100; i++) 
    {
        io_uring_wait_cqe(&ring, &cqe);
        // 处理结果
        io_uring_cqe_seen(&ring, cqe);
    }
}
// 处理1000个连接：~20次系统调用（减少150倍）
```

**2. 真正的异步IO**

```c
// epoll：仍然是同步IO
epoll_wait(...);  // 等待可读
recv(...);        // 同步读取（如果数据未到，可能阻塞）

// io_uring：真正的异步
io_uring_prep_recv(sqe, ...);  // 提交请求
io_uring_submit(&ring);        // 立即返回
// ... 做其他事情 ...
io_uring_wait_cqe(&ring, ...); // 等待完成
```

**3. 内核优化**

```c
// epoll：每次都需要查找fd、加锁
epoll_wait() 
{
    // 1. 遍历就绪队列
    // 2. 拷贝事件到用户空间
    // 3. 加锁/解锁
}

// io_uring：无锁环形队列
// - 用户和内核通过mmap共享内存
// - 无需拷贝数据
// - 无需加锁（单生产者单消费者）
```

**性能数据**：
- 小包场景：提升 30-50%
- 大文件IO：提升 200-300%
- 延迟：降低 20-30%

---

#### Q3: XDP适合做什么？不适合做什么？

**答案**：

**✓ 适合的场景**

1. **DDoS防御**
```c
// 在网卡驱动层直接丢弃恶意流量
if (is_ddos_packet(ip, tcp)) 
{
    return XDP_DROP;  // 最快速度丢弃
}
```

2. **L4负载均衡**
```c
// 修改IP/端口，转发到后端
ip->daddr = backend_ip;
tcp->dest = backend_port;
return XDP_TX;  // 从原网卡发回
```

3. **包过滤/采样**
```c
// 只采样1%的流量到监控系统
if (bpf_get_prandom_u32() % 100 == 0) 
{
    return XDP_PASS;  // 送到用户态分析
} 
else 
{
    return XDP_DROP;
}
```

4. **网络统计**
```c
// 统计每个源IP的流量
bpf_map_update_elem(&ip_stats, &ip->saddr, &bytes, BPF_ADD);
return XDP_PASS;
```

**✗ 不适合的场景**

1. **需要应用层处理**
```c
// XDP只能访问包头，无法解析HTTP、TLS等应用层协议
// 例如：URL过滤、WAF → 用iptables或用户态程序
```

2. **复杂状态管理**
```c
// XDP的BPF程序有复杂度限制：
// - 指令数限制（1M条）
// - 栈大小限制（512字节）
// - 不能使用循环（除非可证明有界）

// 复杂的连接跟踪 → 用iptables或用户态
```

3. **需要修改包内容**
```c
// XDP可以修改包头（IP/TCP/UDP），但修改payload很困难
// 例如：NAT、加密、压缩 → 用户态程序
```

**总结**：
- XDP = 包过滤/转发的"快速通道"
- 不能替代完整的网络栈
- 与传统网络栈配合使用（XDP过滤，合法流量交给内核）

---

#### Q4: DPDK、io_uring、XDP如何选择？

**答案**：

**场景分析决策表**：

| 你的需求                  | 推荐方案           | 理由             |
|-----------------------|----------------|----------------|
| Web服务器（Nginx/Apache）  | **io_uring**   | 性能足够，通用性好      |
| API服务器（Spring/Django） | **epoll**      | 简单够用，无需优化      |
| L4负载均衡（HAProxy）       | **XDP**        | 只需转发，性能最佳      |
| L7负载均衡（Nginx）         | **io_uring**   | 需要解析HTTP，XDP不够 |
| DDoS防御                | **XDP**        | 在驱动层丢包，最快      |
| 高频交易系统                | **DPDK**       | 需要极致延迟（<1μs）   |
| 防火墙/IDS               | **XDP + DPDK** | XDP过滤，DPDK深度检测 |
| 文件服务器                 | **io_uring**   | 异步IO性能好        |
| 虚拟交换机                 | **DPDK**       | 需要极致PPS        |
| 5G基站                  | **DPDK**       | uRLLC要求低延迟     |

**快速决策流程**：

```
1. 需要 >10Mpps？
   否 → io_uring 或 epoll
   是 → 继续

2. 只做包过滤/转发？
   是 → XDP
   否 → 继续

3. 能接受CPU 100%？
   是 → DPDK
   否 → XDP（如果只做L4）或 io_uring

4. 需要应用层处理？
   是 → DPDK（如果需要极致性能）或 io_uring
   否 → XDP
```

**混合使用**：

```
现实中常见组合：

1. Cloudflare边缘节点：
   XDP（DDoS防御）→ Nginx + io_uring（HTTP处理）

2. Facebook Katran负载均衡：
   XDP（L4负载均衡）→ 后端服务器（正常网络栈）

3. 高频交易所：
   DPDK（市场数据接收）+ XDP（恶意流量过滤）

4. 大型Web服务：
   XDP（SYN Cookie）→ epoll（TCP连接）→ io_uring（文件IO）
```

---

#### Q5: 实际项目中使用DPDK/io_uring/XDP有什么坑？

**答案**：

**DPDK的坑**：

1. **独占网卡**
```bash
# DPDK绑定网卡后，内核无法使用该网卡
dpdk-devbind.py --bind=vfio-pci eth0
# eth0从此对ssh/ping不可见！！！

# 解决：保留一张管理网卡，或使用SR-IOV
```

2. **CPU 100%使用率**
```c
// DPDK轮询模式会让CPU一直满载
while (1) {
    rte_eth_rx_burst(...);  // 不停轮询
}

// 解决：使用中断模式（牺牲一些性能）
rte_eth_dev_rx_intr_enable(port_id, queue_id);
```

3. **不支持fork()**
```c
// DPDK初始化后不能fork
rte_eal_init(...);
fork();  // ✗ 崩溃

// 原因：大页内存、UIO映射无法继承
// 解决：改用多进程模式（rte_mp_*）
```

**io_uring的坑**：

1. **需要Linux 5.1+内核**
```bash
uname -r
# 如果 < 5.1，需要升级内核

# 特性支持情况：
# 5.1: 基础功能
# 5.5: SQPOLL模式
# 5.6: 固定缓冲区
# 5.11: 网络多播
```

2. **需要处理EAGAIN**
```c
// io_uring的非阻塞模式可能返回EAGAIN
cqe = io_uring_wait_cqe(&ring, &cqe);
if (cqe->res == -EAGAIN) 
{
    // 重新提交请求
}
```

3. **内存泄漏风险**
```c
// 每个请求分配的buffer需要手动释放
request_t *req = io_uring_cqe_get_data(cqe);
// ... 处理 ...
free(req->buffer);  // ← 别忘了！
free(req);
```

**XDP的坑**：

1. **eBPF验证器限制**
```c
// ✗ 不能有无界循环
for (int i = 0; ; i++) // 验证器拒绝
{  
    ...
}

// ✓ 必须有界
for (int i = 0; i < 100; i++) // 通过验证
{  
    ...
}
```

2. **栈大小限制**
```c
// ✗ 栈只有512字节
char buf[4096];  // 编译失败

// ✓ 使用BPF Map
struct 
{
    __uint(type, BPF_MAP_TYPE_PERCPU_ARRAY);
    __uint(max_entries, 1);
    __type(key, __u32);
    __type(value, char[4096]);
} temp_buf SEC(".maps");
```

3. **调试困难**
```c
// 不能用printf，只能用bpf_printk
bpf_printk("value = %d\n", x);

// 查看输出
sudo cat /sys/kernel/debug/tracing/trace_pipe
```

4. **兼容性问题**
```c
// XDP Generic模式：任何网卡都支持，但慢
ip link set dev eth0 xdpgeneric obj prog.o

// XDP Native模式：快，但只有部分驱动支持
// 支持的驱动：ixgbe, i40e, mlx5, 等
ip link set dev eth0 xdp obj prog.o

// XDP Offload模式：最快，但需要硬件支持
ip link set dev eth0 xdpoffload obj prog.o
```

**通用建议**：

1. **从简单开始**
```
第一步：用epoll写原型
第二步：性能测试，找瓶颈
第三步：如果瓶颈在IO → 考虑io_uring
第四步：如果需要包过滤 → 考虑XDP
第五步：如果需要极致性能 → 考虑DPDK
```

2. **充分测试**
```bash
# 压测工具
wrk -t 4 -c 1000 http://localhost:8080/  # HTTP压测
ab -n 100000 -c 100 http://localhost:8080/  # Apache Bench
iperf3 -c server -t 60  # 网络吞吐量

# 监控工具
perf record -a -g -- sleep 10  # CPU性能分析
perf report
```

3. **看开源项目学习**
```
io_uring: https://github.com/axboe/liburing/tree/master/examples
DPDK: https://github.com/DPDK/dpdk/tree/main/examples
XDP: https://github.com/xdp-project/xdp-tutorial
```

---


## 3. 协议实现

### 3.1 HTTP/1.1 实现

#### HTTP/1.1 协议基础

```
HTTP请求格式：
┌─────────────────────────────────────────┐
│ GET /index.html HTTP/1.1        ← 请求行 │
│ Host: www.example.com           ← 请求头 │
│ User-Agent: curl/7.68.0                 │
│ Accept: */*                             │
│                                 ← 空行   │
│ [request body]                  ← 请求体 │
└─────────────────────────────────────────┘

HTTP响应格式：
┌─────────────────────────────────────────┐
│ HTTP/1.1 200 OK                 ← 状态行 │
│ Content-Type: text/html         ← 响应头 │
│ Content-Length: 1234                    │
│ Connection: keep-alive                  │
│                                 ← 空行   │
│ <html>...</html>                ← 响应体 │
└─────────────────────────────────────────┘
```

#### 完整的HTTP/1.1服务器实现

```c
// http_server.c - 支持Keep-Alive的HTTP/1.1服务器
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <errno.h>

#define MAX_EVENTS 1024
#define BUFFER_SIZE 8192

typedef struct 
{
    int fd;
    char read_buf[BUFFER_SIZE];
    int read_pos;
    char write_buf[BUFFER_SIZE];
    int write_pos;
    int write_len;
    int keep_alive;
} http_conn_t;

// 设置非阻塞
void set_nonblocking(int fd) 
{
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

// 解析HTTP请求
int parse_http_request(const char *buf, int len, char *method, char *path, char *version, int *keep_alive) 
{
    // 简化版解析（实际应该更完善）
    if (sscanf(buf, "%s %s %s", method, path, version) != 3) 
    {
        return -1;
    }

    // 检查Connection头
    *keep_alive = 0;
    const char *conn_header = strstr(buf, "Connection:");
    if (conn_header) 
    {
        if (strstr(conn_header, "keep-alive")) 
        {
            *keep_alive = 1;
        }
    }

    return 0;
}

// 构造HTTP响应
int build_http_response(char *buf, int buf_size, int status_code, const char *status_text,const char *content_type, const char *body, int keep_alive) 
{
    int body_len = strlen(body);
    
    int n = snprintf(buf, buf_size,
        "HTTP/1.1 %d %s\r\n"
        "Server: SimpleHTTP/1.0\r\n"
        "Content-Type: %s\r\n"
        "Content-Length: %d\r\n"
        "Connection: %s\r\n"
        "\r\n"
        "%s",
        status_code, status_text,
        content_type,
        body_len,
        keep_alive ? "keep-alive" : "close",
        body
    );

    return n;
}

// 处理HTTP请求
void handle_http_request(http_conn_t *conn) 
{
    char method[16], path[256], version[16];
    
    // 解析请求
    if (parse_http_request(conn->read_buf, conn->read_pos, method, path, version, &conn->keep_alive) < 0) 
    {
        // 400 Bad Request
        conn->write_len = build_http_response(conn->write_buf, BUFFER_SIZE, 400, "Bad Request", "text/plain", "Bad Request\n", 0);
        conn->keep_alive = 0;
        return;
    }

    printf("← %s %s %s (keep-alive=%d)\n", method, path, version, conn->keep_alive);

    // 简单路由
    const char *body;
    if (strcmp(path, "/") == 0) 
    {
        body = "<html><body><h1>Welcome!</h1></body></html>";
    } 
    else if (strcmp(path, "/api/hello") == 0) 
    {
        body = "{\"message\": \"Hello, World!\"}";
    } 
    else 
    {
        // 404 Not Found
        conn->write_len = build_http_response(conn->write_buf, BUFFER_SIZE, 404, "Not Found", "text/plain", "404 Not Found\n", conn->keep_alive);
        return;
    }

    // 200 OK
    const char *content_type = (strstr(path, "/api/") == path)  ? "application/json"  : "text/html";
    
    conn->write_len = build_http_response(conn->write_buf, BUFFER_SIZE, 200, "OK", content_type, body, conn->keep_alive);
}

// 处理读事件
int handle_read(http_conn_t *conn) 
{
    while (1) 
    {
        int n = recv(conn->fd,  conn->read_buf + conn->read_pos, BUFFER_SIZE - conn->read_pos - 1, 0);

        if (n > 0) 
        {
            conn->read_pos += n;
            conn->read_buf[conn->read_pos] = '\0';

            // 检查是否收到完整HTTP请求（以\r\n\r\n结束）
            if (strstr(conn->read_buf, "\r\n\r\n")) 
            {
                handle_http_request(conn);
                return 1;  // 准备好写响应
            }

            if (conn->read_pos >= BUFFER_SIZE - 1) 
            {
                // 请求太大
                conn->write_len = build_http_response(conn->write_buf, BUFFER_SIZE, 413, "Payload Too Large", "text/plain", "Request Too Large\n", 0);
                conn->keep_alive = 0;
                return 1;
            }
        } 
        else if (n == 0) 
        {
            // 对端关闭
            return -1;
        } 
        else 
        {
            if (errno == EAGAIN || errno == EWOULDBLOCK) 
            {
                // 数据读完了，继续等待
                return 0;
            } 
            else if (errno == EINTR) 
            {
                continue;
            } 
            else 
            {
                perror("recv");
                return -1;
            }
        }
    }
}

// 处理写事件
int handle_write(http_conn_t *conn) 
{
    while (conn->write_pos < conn->write_len) 
    {
        int n = send(conn->fd, conn->write_buf + conn->write_pos, conn->write_len - conn->write_pos, 0);

        if (n > 0) 
        {
            conn->write_pos += n;
        } 
        else if (n == 0) 
        {
            return -1;
        } 
        else 
        {
            if (errno == EAGAIN || errno == EWOULDBLOCK) 
            {
                return 0;  // 发送缓冲区满，稍后再试
            } 
            else if (errno == EINTR) 
            {
                continue;
            } 
            else 
            {
                perror("send");
                return -1;
            }
        }
    }

    // 写完了
    if (conn->keep_alive) 
    {
        // Keep-Alive，复用连接
        conn->read_pos = 0;
        conn->write_pos = 0;
        conn->write_len = 0;
        return 0;  // 继续等待下一个请求
    } 
    else 
    {
        // 关闭连接
        return -1;
    }
}

int main() 
{
    int listen_fd, epfd;
    struct sockaddr_in addr;
    struct epoll_event ev, events[MAX_EVENTS];

    // 1. 创建监听socket
    listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    set_nonblocking(listen_fd);

    int reuse = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

    // 启用TCP_DEFER_ACCEPT（优化：只在收到数据时才accept）
    int defer = 5;
    setsockopt(listen_fd, IPPROTO_TCP, TCP_DEFER_ACCEPT, &defer, sizeof(defer));

    addr.sin_family = AF_INET;
    addr.sin_port = htons(8080);
    addr.sin_addr.s_addr = INADDR_ANY;

    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 128);

    // 2. 创建epoll
    epfd = epoll_create1(0);

    ev.events = EPOLLIN | EPOLLET;  // 边缘触发
    ev.data.fd = listen_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

    printf("HTTP/1.1 server listening on port 8080...\n");
    printf("Try: curl http://localhost:8080/\n");
    printf("     curl http://localhost:8080/api/hello\n");

    // 3. 事件循环
    while (1) 
    {
        int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);

        for (int i = 0; i < nfds; i++) 
        {
            int fd = events[i].data.fd;
            if (fd == listen_fd) 
            {
                // 新连接
                while (1) 
                {
                    struct sockaddr_in client_addr;
                    socklen_t client_len = sizeof(client_addr);
                    int client_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &client_len);

                    if (client_fd < 0) 
                    {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) 
                        {
                            break;  // 没有更多连接
                        } 
                        else 
                        {
                            perror("accept");
                            break;
                        }
                    }

                    printf("✓ New connection: fd=%d\n", client_fd);
                    
                    set_nonblocking(client_fd);

                    // 启用TCP_NODELAY（禁用Nagle算法，降低延迟）
                    int nodelay = 1;
                    setsockopt(client_fd, IPPROTO_TCP, TCP_NODELAY,  &nodelay, sizeof(nodelay));

                    // 分配连接结构
                    http_conn_t *conn = calloc(1, sizeof(http_conn_t));
                    conn->fd = client_fd;

                    ev.events = EPOLLIN | EPOLLET;
                    ev.data.ptr = conn;
                    epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);
                }
            } 
            else 
            {
                // 客户端事件
                http_conn_t *conn = (http_conn_t*)events[i].data.ptr;

                if (events[i].events & EPOLLIN) 
                {
                    // 可读
                    int ret = handle_read(conn);
                    if (ret > 0) 
                    {
                        // 准备好写响应
                        ev.events = EPOLLOUT | EPOLLET;
                        ev.data.ptr = conn;
                        epoll_ctl(epfd, EPOLL_CTL_MOD, conn->fd, &ev);
                    } 
                    else if (ret < 0) 
                    {
                        // 关闭连接
                        printf("✗ Connection closed: fd=%d\n", conn->fd);
                        epoll_ctl(epfd, EPOLL_CTL_DEL, conn->fd, NULL);
                        close(conn->fd);
                        free(conn);
                    }
                }

                if (events[i].events & EPOLLOUT) 
                {
                    // 可写
                    int ret = handle_write(conn);
                    if (ret == 0) 
                    {
                        if (conn->keep_alive) 
                        {
                            // Keep-Alive，切换回读模式
                            ev.events = EPOLLIN | EPOLLET;
                            ev.data.ptr = conn;
                            epoll_ctl(epfd, EPOLL_CTL_MOD, conn->fd, &ev);
                        } 
                        else 
                        {
                            // 关闭连接
                            printf("✗ Connection closed: fd=%d\n", conn->fd);
                            epoll_ctl(epfd, EPOLL_CTL_DEL, conn->fd, NULL);
                            close(conn->fd);
                            free(conn);
                        }
                    } 
                    else if (ret < 0) 
                    {
                        printf("✗ Connection error: fd=%d\n", conn->fd);
                        epoll_ctl(epfd, EPOLL_CTL_DEL, conn->fd, NULL);
                        close(conn->fd);
                        free(conn);
                    }
                }
            }
        }
    }

    return 0;
}
```

**HTTP/1.1关键特性**：

1. **Keep-Alive（持久连接）**
```
HTTP/1.0：每个请求新建一个连接
  客户端 → [3次握手] → 服务器 → 处理 → [4次挥手]
  (延迟高，资源浪费)

HTTP/1.1：默认Keep-Alive
  客户端 → [3次握手] → 服务器
    请求1 → 处理 → 响应1
    请求2 → 处理 → 响应2
    ...
    [4次挥手]
  (减少连接开销，提升性能)
```

2. **Pipeline（管道化）**
```
不使用Pipeline：
  请求1 → 响应1 → 请求2 → 响应2
  (串行，慢)

使用Pipeline：
  请求1 → 请求2 → 请求3 → 响应1 → 响应2 → 响应3
  (并行，快)

注意：响应顺序必须与请求顺序一致（队头阻塞问题）
```

---

### 3.2 HTTP/2 多路复用

#### HTTP/2 vs HTTP/1.1

```
HTTP/1.1的问题：
1. 队头阻塞：一个慢请求阻塞后续请求
2. 连接数限制：浏览器限制每个域6个连接
3. 头部冗余：每个请求重复发送相同的头部

HTTP/2的解决：
1. 多路复用：一个连接处理多个请求（并行）
2. 头部压缩：HPACK算法压缩头部
3. 服务器推送：主动推送资源
4. 二进制帧：更高效的解析
```

#### HTTP/2 核心概念

```
HTTP/2 概念层次：

连接（Connection）
  ↓ 包含多个
流（Stream）        ← 一个流 = 一次请求/响应
  ↓ 由多个
帧（Frame）组成     ← 数据单位
  - HEADERS帧（头部）
  - DATA帧（数据）
  - PRIORITY帧（优先级）
  - RST_STREAM帧（重置流）
  - SETTINGS帧（设置）
  - PING帧（心跳）
  - GOAWAY帧（关闭连接）

示例：在一个TCP连接上同时传输3个请求
┌────────────────────────────────────────────┐
│ TCP连接                                     │
│  Stream 1: GET /index.html                 │
│    Frame: HEADERS, DATA                    │
│  Stream 3: GET /style.css                  │
│    Frame: HEADERS, DATA                    │
│  Stream 5: GET /script.js                  │
│    Frame: HEADERS, DATA                    │
└────────────────────────────────────────────┘

特点：
- 流ID为奇数：客户端发起
- 流ID为偶数：服务器发起（Server Push）
- 帧可以交错传输（真正的多路复用）
```

#### HTTP/2 帧格式

```
HTTP/2 Frame格式（9字节固定头 + 变长payload）：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Length (24)                   |
+---------------+---------------+---------------+
|   Type (8)    |   Flags (8)   |
+-+-+-----------+---------------+-------------------------------+
|R|                 Stream Identifier (31)                      |
+=+=============================================================+
|                   Frame Payload (0...)                      ...
+---------------------------------------------------------------+

Length: 帧payload长度（24位，最大16MB）
Type: 帧类型（0x0=DATA, 0x1=HEADERS, ...）
Flags: 标志位（如END_STREAM, END_HEADERS）
Stream ID: 流标识符（31位）
```

#### 简化的HTTP/2示例（概念）

```c
// http2_concepts.c - HTTP/2核心概念演示（伪代码）

// 帧类型
typedef enum {
    FRAME_TYPE_DATA          = 0x0,
    FRAME_TYPE_HEADERS       = 0x1,
    FRAME_TYPE_PRIORITY      = 0x2,
    FRAME_TYPE_RST_STREAM    = 0x3,
    FRAME_TYPE_SETTINGS      = 0x4,
    FRAME_TYPE_PUSH_PROMISE  = 0x5,
    FRAME_TYPE_PING          = 0x6,
    FRAME_TYPE_GOAWAY        = 0x7,
    FRAME_TYPE_WINDOW_UPDATE = 0x8,
    FRAME_TYPE_CONTINUATION  = 0x9,
} frame_type_t;

// 帧标志
#define FLAG_END_STREAM  0x01
#define FLAG_END_HEADERS 0x04
#define FLAG_PADDED      0x08
#define FLAG_PRIORITY    0x20

// HTTP/2 帧头
typedef struct 
{
    uint32_t length : 24;    // 3字节长度
    uint8_t  type;           // 帧类型
    uint8_t  flags;          // 标志
    uint32_t stream_id : 31; // 流ID
    uint8_t  reserved : 1;   // 保留位
} __attribute__((packed)) http2_frame_header_t;

// HTTP/2 流状态机
typedef enum 
{
    STREAM_IDLE,           // 初始状态
    STREAM_OPEN,           // 打开（可以发送/接收）
    STREAM_HALF_CLOSED_LOCAL,   // 本地半关闭
    STREAM_HALF_CLOSED_REMOTE,  // 远端半关闭
    STREAM_CLOSED,         // 完全关闭
} stream_state_t;

// 流结构
typedef struct 
{
    uint32_t stream_id;
    stream_state_t state;
    // HPACK编码的头部
    char *headers;
    int headers_len;
    // 数据缓冲区
    char *data;
    int data_len;
    // 流量控制窗口
    int window_size;
} http2_stream_t;

// HTTP/2 连接
typedef struct 
{
    int sockfd;
    // 流表（哈希表）
    http2_stream_t *streams[MAX_STREAMS];
    // HPACK上下文（头部压缩）
    hpack_context_t *hpack;
    // 设置
    int max_concurrent_streams;
    int initial_window_size;
} http2_connection_t;

// 发送HEADERS帧
int send_headers_frame(http2_connection_t *conn, uint32_t stream_id, const char *method, const char *path) 
{
    http2_frame_header_t header;
    header.type = FRAME_TYPE_HEADERS;
    header.flags = FLAG_END_HEADERS;
    header.stream_id = stream_id;

    // 使用HPACK压缩头部
    char compressed[1024];
    int compressed_len = hpack_encode(conn->hpack,
        ":method", method,
        ":path", path,
        ":scheme", "https",
        ":authority", "www.example.com",
        compressed, sizeof(compressed)
    );

    header.length = compressed_len;

    // 发送帧头
    send(conn->sockfd, &header, 9, 0);
    // 发送payload
    send(conn->sockfd, compressed, compressed_len, 0);

    return 0;
}

// 接收帧
int recv_frame(http2_connection_t *conn, http2_frame_header_t *header, char *payload, int max_len) 
{
    // 1. 接收帧头（9字节）
    if (recv(conn->sockfd, header, 9, MSG_WAITALL) != 9) 
    {
        return -1;
    }

    // 2. 网络字节序转换
    header->length = ntohl(header->length) >> 8;  // 24位
    header->stream_id = ntohl(header->stream_id) & 0x7FFFFFFF;

    // 3. 接收payload
    if (header->length > max_len) 
    {
        return -1;  // payload太大
    }

    if (recv(conn->sockfd, payload, header->length, MSG_WAITALL) != header->length) 
    {
        return -1;
    }

    return header->length;
}

// 处理DATA帧
void handle_data_frame(http2_connection_t *conn,  http2_frame_header_t *header, const char *payload) 
{
    uint32_t stream_id = header->stream_id;
    http2_stream_t *stream = conn->streams[stream_id];

    if (!stream)
    {
        // 流不存在，发送RST_STREAM
        send_rst_stream(conn, stream_id, ERROR_STREAM_CLOSED);
        return;
    }

    // 追加数据到流缓冲区
    append_data(stream, payload, header->length);

    // 更新流量控制窗口
    stream->window_size -= header->length;

    // 如果窗口太小，发送WINDOW_UPDATE
    if (stream->window_size < INITIAL_WINDOW_SIZE / 2) 
    {
        send_window_update(conn, stream_id, INITIAL_WINDOW_SIZE);
        stream->window_size = INITIAL_WINDOW_SIZE;
    }

    // 检查END_STREAM标志
    if (header->flags & FLAG_END_STREAM) 
    {
        stream->state = STREAM_HALF_CLOSED_REMOTE;
        // 通知应用层：请求接收完毕
        on_request_complete(stream);
    }
}
```

#### HTTP/2 实际使用（基于nghttp2库）

```c
// http2_server_nghttp2.c - 使用nghttp2库实现HTTP/2服务器
#include <nghttp2/nghttp2.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <string.h>

// 发送回调
static ssize_t send_callback(nghttp2_session *session, const uint8_t *data, size_t length, int flags, void *user_data) 
{
    int fd = *(int*)user_data;
    ssize_t n = send(fd, data, length, 0);
    if (n < 0) 
    {
        if (errno == EAGAIN || errno == EWOULDBLOCK) 
        {
            return NGHTTP2_ERR_WOULDBLOCK;
        }
        return NGHTTP2_ERR_CALLBACK_FAILURE;
    }
    return n;
}

// 接收帧头回调
static int on_frame_recv_callback(nghttp2_session *session, const nghttp2_frame *frame, void *user_data) 
{
    if (frame->hd.type == NGHTTP2_HEADERS && frame->headers.cat == NGHTTP2_HCAT_REQUEST) 
    {
        printf("✓ Received request on stream %d\n", frame->hd.stream_id);

        // 构造响应
        const char *body = "<html><body><h1>Hello HTTP/2!</h1></body></html>";
        size_t body_len = strlen(body);

        // 响应头
        nghttp2_nv hdrs[] = 
        {
            NGHTTP2_MAKE_NV(":status", "200"),
            NGHTTP2_MAKE_NV("content-type", "text/html"),
        };

        nghttp2_data_provider data_prd;
        data_prd.source.ptr = (void*)body;
        data_prd.read_callback = read_callback;

        // 提交响应
        nghttp2_submit_response(session, frame->hd.stream_id, hdrs, 2, &data_prd);
    }

    return 0;
}

// 数据提供回调
static ssize_t read_callback(nghttp2_session *session, int32_t stream_id, uint8_t *buf, size_t length, 
   uint32_t *data_flags, nghttp2_data_source *source, void *user_data) 
{
    const char *body = (const char*)source->ptr;
    size_t body_len = strlen(body);

    if (length < body_len) 
    {
        memcpy(buf, body, length);
        return length;
    } 
    else 
    {
        memcpy(buf, body, body_len);
        *data_flags |= NGHTTP2_DATA_FLAG_EOF;
        return body_len;
    }
}

int main() 
{
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8443);
    addr.sin_addr.s_addr = INADDR_ANY;

    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 10);

    printf("HTTP/2 server listening on port 8443...\n");
    printf("Note: Requires HTTPS in production\n");

    while (1) 
    {
        int client_fd = accept(listen_fd, NULL, NULL);

        // 创建nghttp2会话
        nghttp2_session_callbacks *callbacks;
        nghttp2_session_callbacks_new(&callbacks);
        nghttp2_session_callbacks_set_send_callback(callbacks, send_callback);
        nghttp2_session_callbacks_set_on_frame_recv_callback(callbacks, on_frame_recv_callback);

        nghttp2_session *session;
        nghttp2_session_server_new(&session, callbacks, &client_fd);

        // 发送连接前言
        nghttp2_settings_entry iv[] = 
        {
            {NGHTTP2_SETTINGS_MAX_CONCURRENT_STREAMS, 100},
        };
        nghttp2_submit_settings(session, NGHTTP2_FLAG_NONE, iv, 1);

        // 事件循环
        while (1) 
        {
            // 发送待发送的数据
            if (nghttp2_session_send(session) != 0) break;

            // 接收数据
            uint8_t buf[4096];
            ssize_t n = recv(client_fd, buf, sizeof(buf), 0);
            if (n <= 0) break;

            // 处理接收的数据
            if (nghttp2_session_mem_recv(session, buf, n) < 0) break;
        }

        nghttp2_session_del(session);
        nghttp2_session_callbacks_del(callbacks);
        close(client_fd);
    }

    return 0;
}
```

**HTTP/2性能优势**：

```
测试：加载一个包含100个资源的网页

HTTP/1.1（6个并发连接）：
- 时间：5.2秒
- 连接数：6个
- 请求：串行排队

HTTP/2（1个连接）：
- 时间：2.1秒
- 连接数：1个
- 请求：并行处理

性能提升：2.5倍
```

---

### 3.3 WebSocket 长连接

#### WebSocket vs HTTP

```
HTTP：
  客户端 → 请求 → 服务器
  客户端 ← 响应 ← 服务器
  (单向，服务器不能主动推送)

WebSocket：
  客户端 ⇄ 双向通信 ⇄ 服务器
  (全双工，服务器可以主动推送)

应用场景：
- 聊天室
- 实时通知
- 在线游戏
- 股票行情推送
- 协同编辑
```

#### WebSocket握手

```
WebSocket基于HTTP升级：

客户端请求：
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket                     ← 关键
Connection: Upgrade                    ← 关键
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==  ← 随机key
Sec-WebSocket-Version: 13

服务器响应：
HTTP/1.1 101 Switching Protocols        ← 状态码101
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
                      ↑ 根据客户端key计算得出

之后就是WebSocket帧，不再是HTTP
```

#### WebSocket帧格式

```
WebSocket Frame格式：

  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
 |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
 |N|V|V|V|       |S|             |   (if payload len==126/127)   |
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
 |     Extended payload length continued, if payload len == 127  |
 + - - - - - - - - - - - - - - - +-------------------------------+
 |                               |Masking-key, if MASK set to 1  |
 +-------------------------------+-------------------------------+
 | Masking-key (continued)       |          Payload Data         |
 +-------------------------------- - - - - - - - - - - - - - - - +
 :                     Payload Data continued ...                :
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 |                     Payload Data continued ...                |
 +---------------------------------------------------------------+

FIN: 1=最后一帧，0=后续还有帧
opcode: 帧类型
  0x0: 续帧
  0x1: 文本帧
  0x2: 二进制帧
  0x8: 关闭连接
  0x9: Ping
  0xA: Pong
MASK: 1=payload被掩码（客户端必须掩码，服务器不能）
Payload len: 数据长度
Masking-key: 4字节掩码key（如果MASK=1）
Payload: 实际数据
```

#### WebSocket服务器实现

```c
// websocket_server.c - 完整的WebSocket服务器
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <openssl/sha.h>
#include <openssl/bio.h>
#include <openssl/evp.h>
#include <openssl/buffer.h>

#define WS_GUID "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"

// Base64编码
char *base64_encode(const unsigned char *input, int length) 
{
    BIO *bio, *b64;
    BUF_MEM *buffer_ptr;

    b64 = BIO_new(BIO_f_base64());
    bio = BIO_new(BIO_s_mem());
    bio = BIO_push(b64, bio);

    BIO_set_flags(bio, BIO_FLAGS_BASE64_NO_NL);
    BIO_write(bio, input, length);
    BIO_flush(bio);

    BIO_get_mem_ptr(bio, &buffer_ptr);
    char *output = malloc(buffer_ptr->length + 1);
    memcpy(output, buffer_ptr->data, buffer_ptr->length);
    output[buffer_ptr->length] = '\0';

    BIO_free_all(bio);
    return output;
}

// 计算Sec-WebSocket-Accept
char *compute_accept_key(const char *client_key) 
{
    // 1. client_key + GUID
    char combined[256];
    snprintf(combined, sizeof(combined), "%s%s", client_key, WS_GUID);

    // 2. SHA1哈希
    unsigned char hash[SHA_DIGEST_LENGTH];
    SHA1((unsigned char*)combined, strlen(combined), hash);

    // 3. Base64编码
    return base64_encode(hash, SHA_DIGEST_LENGTH);
}

// 处理WebSocket握手
int handle_websocket_handshake(int client_fd) 
{
    char buffer[2048];
    int n = recv(client_fd, buffer, sizeof(buffer) - 1, 0);
    if (n <= 0) return -1;

    buffer[n] = '\0';

    // 提取Sec-WebSocket-Key
    const char *key_header = "Sec-WebSocket-Key: ";
    char *key_start = strstr(buffer, key_header);
    if (!key_start) 
    {
        return -1;
    }

    key_start += strlen(key_header);
    char *key_end = strstr(key_start, "\r\n");
    if (!key_end) 
    {
        return -1;
    }

    char client_key[128];
    int key_len = key_end - key_start;
    strncpy(client_key, key_start, key_len);
    client_key[key_len] = '\0';

    printf("Client Key: %s\n", client_key);

    // 计算Accept key
    char *accept_key = compute_accept_key(client_key);
    printf("Accept Key: %s\n", accept_key);

    // 发送握手响应
    char response[512];
    snprintf(response, sizeof(response),
        "HTTP/1.1 101 Switching Protocols\r\n"
        "Upgrade: websocket\r\n"
        "Connection: Upgrade\r\n"
        "Sec-WebSocket-Accept: %s\r\n"
        "\r\n",
        accept_key
    );

    send(client_fd, response, strlen(response), 0);
    free(accept_key);

    printf("✓ WebSocket handshake complete\n");
    return 0;
}

// 解析WebSocket帧
typedef struct 
{
    int fin;
    int opcode;
    int masked;
    uint64_t payload_len;
    unsigned char mask[4];
    unsigned char *payload;
} ws_frame_t;

int parse_websocket_frame(const unsigned char *buf, int len, ws_frame_t *frame) 
{
    if (len < 2) return -1;

    // 第1字节：FIN + RSV + opcode
    frame->fin = (buf[0] & 0x80) >> 7;
    frame->opcode = buf[0] & 0x0F;

    // 第2字节：MASK + payload len
    frame->masked = (buf[1] & 0x80) >> 7;
    frame->payload_len = buf[1] & 0x7F;

    int pos = 2;

    // 扩展长度
    if (frame->payload_len == 126) 
    {
        if (len < 4) return -1;
        frame->payload_len = (buf[2] << 8) | buf[3];
        pos = 4;
    } 
    else if (frame->payload_len == 127) 
    {
        if (len < 10) return -1;
        frame->payload_len = 0;
        for (int i = 0; i < 8; i++) 
        {
            frame->payload_len = (frame->payload_len << 8) | buf[2 + i];
        }
        pos = 10;
    }

    // Masking key（客户端发送的帧必须有）
    if (frame->masked) 
    {
        if (len < pos + 4) return -1;
        memcpy(frame->mask, buf + pos, 4);
        pos += 4;
    }

    // Payload
    if (len < pos + frame->payload_len) return -1;

    frame->payload = malloc(frame->payload_len + 1);
    memcpy(frame->payload, buf + pos, frame->payload_len);

    // 解除掩码
    if (frame->masked) 
    {
        for (uint64_t i = 0; i < frame->payload_len; i++) 
        {
            frame->payload[i] ^= frame->mask[i % 4];
        }
    }

    frame->payload[frame->payload_len] = '\0';

    return pos + frame->payload_len;
}

// 构造WebSocket帧
int build_websocket_frame(unsigned char *buf, int buf_size, int opcode, const char *payload, uint64_t len) 
{
    int pos = 0;

    // 第1字节：FIN=1, RSV=0, opcode
    buf[pos++] = 0x80 | opcode;

    // 第2字节：MASK=0（服务器不掩码）, payload len
    if (len < 126) 
    {
        buf[pos++] = len;
    } 
    else if (len < 65536) 
    {
        buf[pos++] = 126;
        buf[pos++] = (len >> 8) & 0xFF;
        buf[pos++] = len & 0xFF;
    } 
    else 
    {
        buf[pos++] = 127;
        for (int i = 7; i >= 0; i--) 
        {
            buf[pos++] = (len >> (i * 8)) & 0xFF;
        }
    }

    // Payload
    memcpy(buf + pos, payload, len);
    pos += len;

    return pos;
}

// 处理WebSocket连接
void handle_websocket_connection(int client_fd) 
{
    unsigned char buffer[4096];
    while (1) 
    {
        int n = recv(client_fd, buffer, sizeof(buffer), 0);
        if (n <= 0) break;

        ws_frame_t frame;
        int frame_len = parse_websocket_frame(buffer, n, &frame);

        if (frame_len < 0) 
        {
            fprintf(stderr, "Invalid frame\n");
            break;
        }

        switch (frame.opcode) 
        {
            case 0x1:  // 文本帧
                printf("← Received text: %s\n", frame.payload);

                // Echo back
                unsigned char response[4096];
                int resp_len = build_websocket_frame(response, sizeof(response), 0x1, (char*)frame.payload, frame.payload_len);
                send(client_fd, response, resp_len, 0);
                printf("→ Sent echo: %s\n", frame.payload);
                break;

            case 0x2:  // 二进制帧
                printf("← Received binary: %llu bytes\n", frame.payload_len);
                break;

            case 0x8:  // 关闭帧
                printf("✗ Client closing connection\n");
                free(frame.payload);
                return;

            case 0x9:  // Ping
                printf("← Received Ping\n");
                // 回复Pong
                int pong_len = build_websocket_frame(response, sizeof(response), 0xA, "", 0);
                send(client_fd, response, pong_len, 0);
                printf("→ Sent Pong\n");
                break;

            case 0xA:  // Pong
                printf("← Received Pong\n");
                break;
        }

        free(frame.payload);
    }
}

int main() 
{
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    int reuse = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(9001);
    addr.sin_addr.s_addr = INADDR_ANY;

    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 10);

    printf("WebSocket server listening on port 9001...\n");
    printf("Try: wscat -c ws://localhost:9001\n");

    while (1) 
    {
        int client_fd = accept(listen_fd, NULL, NULL);
        printf("\n✓ New connection\n");

        // WebSocket握手
        if (handle_websocket_handshake(client_fd) < 0) 
        {
            fprintf(stderr, "Handshake failed\n");
            close(client_fd);
            continue;
        }

        // 处理WebSocket连接
        handle_websocket_connection(client_fd);

        close(client_fd);
        printf("✗ Connection closed\n");
    }

    return 0;
}
```

**测试WebSocket服务器**：

```bash
# 编译
gcc websocket_server.c -o ws_server -lssl -lcrypto

# 运行服务器
./ws_server

# 客户端测试（安装wscat）
npm install -g wscat
wscat -c ws://localhost:9001

# 或使用JavaScript客户端
# test.html:
<script>
const ws = new WebSocket('ws://localhost:9001');

ws.onopen = () => {
    console.log('Connected');
    ws.send('Hello WebSocket!');
};

ws.onmessage = (event) => {
    console.log('Received:', event.data);
};

ws.onclose = () => {
    console.log('Disconnected');
};
</script>
```

---

### 3.4 QUIC（HTTP/3）

#### QUIC vs TCP

```
TCP的问题：
1. 队头阻塞：一个丢包阻塞整个连接
2. 握手延迟：3次握手 + TLS握手 = 2-3 RTT
3. 连接迁移：IP变化需要重新建立连接

QUIC的解决：
✓ 基于UDP（避免内核TCP栈的限制）
✓ 0-RTT连接建立（缓存密钥）
✓ 多路复用无队头阻塞（流独立）
✓ 连接迁移（Connection ID）
✓ 内置TLS 1.3加密
```

#### QUIC特性

**1. 0-RTT连接建立**

```
TCP + TLS 1.2：需要3 RTT
客户端 ────SYN────> 服务器          (RTT 1)
客户端 <───SYN+ACK── 服务器
客户端 ────ACK────> 服务器          (RTT 2)
客户端 ──ClientHello→ 服务器
客户端 ←─ServerHello── 服务器       (RTT 3)
客户端 ───数据────> 服务器

QUIC：0-RTT（首次连接1-RTT，后续0-RTT）
客户端 ──Initial+Data→ 服务器       (RTT 0-1)
客户端 ←──Response──── 服务器

优势：减少2个RTT延迟（～100ms）
```

**2. 流级别的多路复用**

```
TCP + HTTP/2：
┌────────────────────────────────────┐
│ TCP连接                             │
│  Stream 1: ████████ (正常)          │
│  Stream 2: ██▓▓▓▓▓▓ (丢包阻塞)       │ ← Stream 2丢包阻塞了
│  Stream 3: ▓▓▓▓▓▓▓▓ (也被阻塞)       │    Stream 3
└────────────────────────────────────┘

QUIC + HTTP/3：
┌────────────────────────────────────┐
│ QUIC连接                            │
│  Stream 1: ████████ (正常)          │
│  Stream 2: ██░░░░░░ (丢包)          │ ← Stream 2丢包不影响
│  Stream 3: ████████ (正常继续)      │    Stream 3
└────────────────────────────────────┘
```

**3. 连接迁移**

```
TCP：IP/端口变化 → 连接断开
移动网络：WiFi → 4G
  旧连接：192.168.1.10:12345 → 服务器
  ✗ 连接断开
  新连接：10.0.0.5:54321 → 服务器
  需要重新建立连接

QUIC：Connection ID标识连接
移动网络：WiFi → 4G
  连接ID：0x1234567890abcdef
  ✓ 连接保持
  新IP：10.0.0.5:54321 → 服务器
  服务器识别Connection ID，连接无缝迁移
```

#### QUIC包格式（简化）

```
QUIC Packet（Long Header）：

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+
|1|1|T T|X X X X|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Version (32)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| DCID Len (8)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Destination Connection ID (0..160)            ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| SCID Len (8)  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                 Source Connection ID (0..160)               ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Payload (*)                       ...
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

关键字段：
- Version: QUIC版本
- Destination Connection ID (DCID): 目标连接ID
- Source Connection ID (SCID): 源连接ID
- Payload: 加密的帧数据（包含STREAM帧、ACK帧等）
```

#### QUIC实际使用（概念，生产用quiche/msquic库）

QUIC实现非常复杂，生产环境使用现成的库：

```c
// quiche_example.c - 使用Cloudflare的quiche库
#include <quiche.h>

// 1. 创建QUIC配置
quiche_config *config = quiche_config_new(QUICHE_PROTOCOL_VERSION);
quiche_config_set_application_protos(config, (const uint8_t *)"\x05hq-29", 6);  // HTTP/3

// 2. 创建QUIC连接
quiche_conn *conn = quiche_connect(server_name, &scid, &local_addr, &peer_addr, config);

// 3. 发送Initial包（包含TLS ClientHello）
uint8_t out[MAX_DATAGRAM_SIZE];
ssize_t written = quiche_conn_send(conn, out, sizeof(out), &send_info);
sendto(sockfd, out, written, 0, ...);

// 4. 接收响应
uint8_t buf[MAX_DATAGRAM_SIZE];
ssize_t n = recvfrom(sockfd, buf, sizeof(buf), 0, ...);
quiche_conn_recv(conn, buf, n, &recv_info);

// 5. 发送HTTP/3请求
uint8_t request[] = 
    ":method GET\r\n"
    ":scheme https\r\n"
    ":authority example.com\r\n"
    ":path /\r\n";

quiche_h3_send_request(h3_conn, conn, request, sizeof(request), true);

// 6. 接收响应
uint8_t response[4096];
bool fin;
ssize_t n = quiche_h3_recv_body(h3_conn, conn, stream_id, response, sizeof(response), &fin);
```

#### QUIC性能数据

```
测试：加载网页（100个资源）

TCP + HTTP/2（延迟50ms）：
- 首次连接：2.8秒（3-RTT握手）
- 后续连接：2.1秒

QUIC + HTTP/3（延迟50ms）：
- 首次连接：2.1秒（1-RTT握手）
- 后续连接：1.4秒（0-RTT）

弱网环境（1%丢包率）：
- HTTP/2：6.5秒（队头阻塞严重）
- HTTP/3：3.2秒（流独立，无阻塞）

性能提升：30-50%（尤其在弱网环境）
```

---

### 3.5 高频面试题

#### Q1: HTTP/1.1、HTTP/2、HTTP/3有什么区别？

**答案**：

| 特性        | HTTP/1.1       | HTTP/2      | HTTP/3        |
|-----------|----------------|-------------|---------------|
| **传输层**   | TCP            | TCP         | QUIC (UDP)    |
| **连接复用**  | Keep-Alive（串行） | 多路复用（并行）    | 多路复用（并行）      |
| **队头阻塞**  | 有（应用层+传输层）     | 有（传输层TCP丢包） | 无（流独立）        |
| **头部压缩**  | 无              | HPACK       | QPACK         |
| **服务器推送** | 无              | 有           | 有             |
| **加密**    | 可选（HTTPS）      | 可选但推荐       | 强制（内置TLS 1.3） |
| **连接建立**  | 3-RTT          | 3-RTT       | 0-1 RTT       |
| **连接迁移**  | 不支持            | 不支持         | 支持            |

**选择建议**：
- 兼容性：HTTP/1.1（所有设备都支持）
- 性能优化：HTTP/2（现代浏览器都支持）
- 极致性能：HTTP/3（需要服务器和CDN支持）

---

#### Q2: WebSocket和HTTP长轮询有什么区别？

**答案**：

**HTTP长轮询（Long Polling）**：
```javascript
// 客户端
function poll() {
    fetch('/poll', {timeout: 30000})
        .then(response => {
            // 处理消息
            poll();  // 立即发起下一个请求
        });
}
poll();

特点：
- 客户端不停发起请求
- 服务器hold住请求直到有数据或超时
- 单向（服务器 → 客户端）
- 开销大：每次都需要HTTP头
```

**WebSocket**：
```javascript
// 客户端
const ws = new WebSocket('ws://server');
ws.onmessage = (msg) => {
    // 处理消息
};
ws.send('Hello');  // 客户端也能发送

特点：
- 建立连接后保持
- 双向通信
- 低开销：帧头只有2-10字节
- 实时性好：延迟<1ms
```

**对比**：

|         | HTTP长轮询      | WebSocket   |
|---------|--------------|-------------|
| 连接      | 每次新建         | 持久连接        |
| 开销      | 高（HTTP头几百字节） | 低（帧头2-10字节） |
| 实时性     | 中等（秒级）       | 高（毫秒级）      |
| 服务器推送   | 支持           | 支持          |
| 客户端→服务器 | 需要额外请求       | 直接发送        |
| 兼容性     | 好（HTTP）      | 好（现代浏览器）    |
| 适用场景    | 通知、低频更新      | 聊天、游戏、实时数据  |

---

#### Q3: HTTP/2的多路复用是如何实现的？为什么还会有队头阻塞？

**答案**：

**多路复用实现**：

```
HTTP/2在一个TCP连接上传输多个流：

时间轴：
  帧1（Stream 1）→ 帧2（Stream 3）→ 帧3（Stream 1）→ 帧4（Stream 5）
                    ↑ 交错传输

接收端重组：
  Stream 1: 帧1 + 帧3 → 完整响应
  Stream 3: 帧2 → 完整响应
  Stream 5: 帧4 → 完整响应

优势：不再需要多个TCP连接，减少握手开销
```

**为什么还有队头阻塞？**

```
问题出在TCP层：

场景：Stream 1和Stream 3同时传输
  Stream 1: 帧1-1 ✓ 帧1-2 ✗ 帧1-3 等待
  Stream 3: 帧3-1 ✓ 帧3-2 ✓ 帧3-3 等待

TCP层：
  [帧1-1] [帧3-1] [帧3-2] [帧1-2丢失] [帧1-3] [帧3-3]
                            ↑ TCP必须重传
                            ↓ 阻塞后续所有数据

即使帧3-3已到达，TCP也不会交给应用层，必须等帧1-2重传

这就是HTTP/2的队头阻塞：
- 应用层无阻塞（流独立）
- 传输层有阻塞（TCP丢包重传）

HTTP/3解决：
- 使用QUIC（UDP），流完全独立
- Stream 1丢包不影响Stream 3
```

---

#### Q4: QUIC为什么能实现0-RTT连接？安全吗？

**答案**：

**0-RTT实现原理**：

```
首次连接（1-RTT）：
客户端 ──Initial+TLS ClientHello─→ 服务器
客户端 ←─TLS ServerHello+票据──── 服务器
  ↑ 服务器发送Session Ticket（包含密钥）

客户端缓存票据

后续连接（0-RTT）：
客户端 ──0-RTT Data（加密，用票据）─→ 服务器
       ↑ 直接发送应用数据
客户端 ←────Response──────────────── 服务器

服务器用票据解密，立即响应
```

**安全性考虑**：

**✓ 安全点**：
1. 数据加密：使用TLS 1.3
2. 前向保密：定期更新密钥
3. 防重放：Ticket有效期限制

**✗ 风险点**：
1. **重放攻击**（Replay Attack）：
```
场景：攻击者截获0-RTT数据包，重放给服务器

风险操作：
  POST /transfer?amount=1000  ← 如果被重放，会转账多次

缓解措施：
- 服务器记录已处理的0-RTT请求（需要状态存储）
- 0-RTT只用于幂等操作（GET、OPTIONS）
- 敏感操作（POST、DELETE）在1-RTT后发送
```

**最佳实践**：

```
✓ 0-RTT可用于：
  - GET请求
  - 幂等操作
  - 不敏感数据

✗ 0-RTT不应用于：
  - POST/PUT/DELETE
  - 支付转账
  - 状态修改操作

示例：
GET /api/news (0-RTT) ✓
POST /api/order (1-RTT) ✓
```

---

#### Q5: 如何选择HTTP/1.1、HTTP/2、HTTP/3？

**答案**：

**决策树**：

```
1. 你的用户群？
   ├─ 老旧浏览器（IE11等）
   │  → HTTP/1.1
   │
   └─ 现代浏览器
       ↓
2. 你的应用类型？
   ├─ 静态网站（少量请求）
   │  → HTTP/1.1足够
   │
   ├─ 复杂Web应用（大量资源）
   │  → HTTP/2
   │
   └─ 实时应用（视频、游戏）
       ↓
3. 网络环境？
   ├─ 稳定网络（有线、WiFi）
   │  → HTTP/2
   │
   └─ 弱网环境（移动网络、高丢包率）
       → HTTP/3

4. 服务器支持？
   ├─ 老旧服务器
   │  → HTTP/1.1
   │
   ├─ Nginx/Apache现代版本
   │  → HTTP/2
   │
   └─ Cloudflare/CDN
       → HTTP/3
```

**实际案例**：

```
1. 简单博客：
   → HTTP/1.1 + HTTPS
   原因：请求少，简单够用

2. 电商网站（100+图片）：
   → HTTP/2
   原因：多路复用，减少连接数

3. Google搜索：
   → HTTP/3
   原因：全球部署，优化弱网体验

4. 在线视频（YouTube）：
   → HTTP/3
   原因：低延迟，连接迁移，弱网优化

5. 企业内网应用：
   → HTTP/2
   原因：网络稳定，HTTP/3收益不大
```

**迁移建议**：

```
第一步：支持HTTP/2
  - Nginx: http2 on;
  - Apache: LoadModule http2_module
  - 兼容：自动降级到HTTP/1.1

第二步：启用HTTP/3（可选）
  - Cloudflare CDN（自动支持）
  - Nginx + quiche模块
  - Caddy（内置QUIC）

第三步：优化
  - 启用TLS 1.3
  - 配置0-RTT（谨慎）
  - 监控性能指标
```

---


## 4. 分布式系统网络编程

### 4.1 RPC框架原理

#### 什么是RPC？

RPC（Remote Procedure Call，远程过程调用）让调用远程服务像调用本地函数一样简单。

```
本地函数调用：
  result = add(1, 2)  // 直接调用

RPC调用：
  result = rpc_client.add(1, 2)  // 调用远程服务
  
  底层实际发生：
  客户端 → 序列化参数 → 网络传输 → 服务器
  客户端 ← 反序列化结果 ← 网络传输 ← 服务器
```

#### RPC核心组件

```
┌──────────────────────────────────────────────────────────┐
│                      客户端                               │
│  ┌────────────┐    ┌──────────┐    ┌────────────────┐    │
│  │ 业务代码    │ ──→│  Stub    │ ──→ │  序列化器       │    │
│  │ add(1,2)   │    │ (代理)    │    │ (Protobuf等)   │    │
│  └────────────┘    └──────────┘    └────────────────┘    │
│                          ↓                               │
│                    ┌──────────┐                          │
│                    │ 网络传输  │                          │
│                    └──────────┘                          │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│                      服务器                               │
│                    ┌──────────┐                          │
│                    │ 网络接收  │                          │
│                    └──────────┘                          │
│                          ↓                               │
│  ┌────────────┐   ┌───────────┐   ┌────────────────┐     │
│  │ 业务实现    │←──│  Skeleton │←──│  反序列化器      │     │
│  │ add(a,b)   │   │           │   │                │     │
│  └────────────┘   └───────────┘   └────────────────┘     │
└──────────────────────────────────────────────────────────┘

关键组件：
1. Stub（客户端桩）：代理对象，拦截本地调用
2. Skeleton（服务端骨架）：调度到实际实现
3. 序列化器：对象 ↔ 字节流
4. 网络传输：TCP/UDP/HTTP
5. 服务注册中心：服务发现
```

#### 简单RPC实现

**协议设计**：

```c
// 自定义RPC协议（简单版）
typedef struct 
{
    uint32_t magic;      // 魔数：0x12345678
    uint32_t version;    // 版本：1
    uint32_t msg_id;     // 消息ID（用于匹配请求响应）
    uint32_t func_id;    // 函数ID（0=add, 1=sub, ...）
    uint32_t param_len;  // 参数长度
    // 紧跟param_len字节的参数数据（JSON/Protobuf/...）
} __attribute__((packed)) rpc_header_t;

请求示例：
┌──────────────────────────────────────────┐
│ Header:                                  │
│   magic: 0x12345678                      │
│   version: 1                             │
│   msg_id: 100                            │
│   func_id: 0 (add)                       │
│   param_len: 11                          │
├──────────────────────────────────────────┤
│ Payload:                                 │
│   {"a":1,"b":2}  (JSON, 11字节)           │
└──────────────────────────────────────────┘

响应示例：
┌──────────────────────────────────────────┐
│ Header:                                  │
│   magic: 0x12345678                      │
│   version: 1                             │
│   msg_id: 100  (与请求相同)                │
│   func_id: 0                             │
│   param_len: 11                          │
├──────────────────────────────────────────┤
│ Payload:                                 │
│   {"result":3}  (JSON, 11字节)            │
└──────────────────────────────────────────┘
```

**服务器实现**：

```c
// rpc_server.c - 简单RPC服务器
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <jansson.h>  // JSON库

#define RPC_MAGIC 0x12345678
#define RPC_VERSION 1

typedef struct 
{
    uint32_t magic;
    uint32_t version;
    uint32_t msg_id;
    uint32_t func_id;
    uint32_t param_len;
} __attribute__((packed)) rpc_header_t;

// 业务函数：加法
int rpc_add(int a, int b) 
{
    return a + b;
}

// 业务函数：减法
int rpc_sub(int a, int b) 
{
    return a - b;
}

// 函数注册表
typedef int (*rpc_func_t)(int, int);
rpc_func_t func_table[] = 
{
    rpc_add,  // func_id = 0
    rpc_sub,  // func_id = 1
};

// 处理RPC请求
void handle_rpc_request(int client_fd) 
{
    rpc_header_t req_header, resp_header;
    char *payload = NULL;

    // 1. 接收请求头
    if (recv(client_fd, &req_header, sizeof(req_header), MSG_WAITALL) != sizeof(req_header)) 
    {
        perror("recv header");
        return;
    }

    // 2. 字节序转换
    req_header.magic = ntohl(req_header.magic);
    req_header.version = ntohl(req_header.version);
    req_header.msg_id = ntohl(req_header.msg_id);
    req_header.func_id = ntohl(req_header.func_id);
    req_header.param_len = ntohl(req_header.param_len);

    // 3. 验证magic
    if (req_header.magic != RPC_MAGIC) 
    {
        fprintf(stderr, "Invalid magic: 0x%x\n", req_header.magic);
        return;
    }

    printf("← Request: msg_id=%u, func_id=%u, param_len=%u\n", req_header.msg_id, req_header.func_id, req_header.param_len);

    // 4. 接收payload
    payload = malloc(req_header.param_len + 1);
    if (recv(client_fd, payload, req_header.param_len, MSG_WAITALL) != req_header.param_len) 
    {
        perror("recv payload");
        free(payload);
        return;
    }
    payload[req_header.param_len] = '\0';

    printf("   Payload: %s\n", payload);

    // 5. 解析JSON参数
    json_error_t error;
    json_t *root = json_loads(payload, 0, &error);
    free(payload);

    if (!root) 
    {
        fprintf(stderr, "JSON parse error: %s\n", error.text);
        return;
    }

    int a = json_integer_value(json_object_get(root, "a"));
    int b = json_integer_value(json_object_get(root, "b"));
    json_decref(root);

    // 6. 调用业务函数
    if (req_header.func_id >= sizeof(func_table) / sizeof(func_table[0])) 
    {
        fprintf(stderr, "Invalid func_id: %u\n", req_header.func_id);
        return;
    }

    int result = func_table[req_header.func_id](a, b);
    printf("   Result: %d\n", result);

    // 7. 构造响应
    json_t *resp_json = json_object();
    json_object_set_new(resp_json, "result", json_integer(result));
    char *resp_payload = json_dumps(resp_json, JSON_COMPACT);
    json_decref(resp_json);

    // 8. 发送响应头
    resp_header.magic = htonl(RPC_MAGIC);
    resp_header.version = htonl(RPC_VERSION);
    resp_header.msg_id = htonl(req_header.msg_id);
    resp_header.func_id = htonl(req_header.func_id);
    resp_header.param_len = htonl(strlen(resp_payload));

    send(client_fd, &resp_header, sizeof(resp_header), 0);
    send(client_fd, resp_payload, strlen(resp_payload), 0);

    free(resp_payload);

    printf("→ Response sent\n\n");
}

int main() 
{
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    int reuse = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(5000);
    addr.sin_addr.s_addr = INADDR_ANY;

    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 10);

    printf("RPC Server listening on port 5000...\n");

    while (1) 
    {
        int client_fd = accept(listen_fd, NULL, NULL);
        handle_rpc_request(client_fd);
        close(client_fd);
    }

    return 0;
}
```

**客户端实现**：

```c
// rpc_client.c - 简单RPC客户端
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <jansson.h>

#define RPC_MAGIC 0x12345678
#define RPC_VERSION 1

typedef struct 
{
    uint32_t magic;
    uint32_t version;
    uint32_t msg_id;
    uint32_t func_id;
    uint32_t param_len;
} __attribute__((packed)) rpc_header_t;

// RPC调用封装
int rpc_call(const char *server_ip, int port,  uint32_t func_id, int a, int b) 
{
    int sockfd;
    rpc_header_t req_header, resp_header;

    // 1. 连接服务器
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    inet_pton(AF_INET, server_ip, &addr.sin_addr);

    if (connect(sockfd, (struct sockaddr*)&addr, sizeof(addr)) < 0) 
    {
        perror("connect");
        return -1;
    }

    // 2. 构造请求payload
    json_t *req_json = json_object();
    json_object_set_new(req_json, "a", json_integer(a));
    json_object_set_new(req_json, "b", json_integer(b));
    char *req_payload = json_dumps(req_json, JSON_COMPACT);
    json_decref(req_json);

    // 3. 构造请求头
    req_header.magic = htonl(RPC_MAGIC);
    req_header.version = htonl(RPC_VERSION);
    req_header.msg_id = htonl(100);  // 简化：固定msg_id
    req_header.func_id = htonl(func_id);
    req_header.param_len = htonl(strlen(req_payload));

    // 4. 发送请求
    send(sockfd, &req_header, sizeof(req_header), 0);
    send(sockfd, req_payload, strlen(req_payload), 0);
    free(req_payload);

    printf("→ Request sent: func_id=%u, a=%d, b=%d\n", func_id, a, b);

    // 5. 接收响应头
    if (recv(sockfd, &resp_header, sizeof(resp_header), MSG_WAITALL) != sizeof(resp_header)) 
    {
        perror("recv header");
        close(sockfd);
        return -1;
    }

    // 6. 字节序转换
    resp_header.param_len = ntohl(resp_header.param_len);

    // 7. 接收响应payload
    char *resp_payload = malloc(resp_header.param_len + 1);
    if (recv(sockfd, resp_payload, resp_header.param_len, MSG_WAITALL) != resp_header.param_len) 
    {
        perror("recv payload");
        free(resp_payload);
        close(sockfd);
        return -1;
    }
    resp_payload[resp_header.param_len] = '\0';

    // 8. 解析响应
    json_error_t error;
    json_t *resp_json = json_loads(resp_payload, 0, &error);
    free(resp_payload);

    if (!resp_json) 
    {
        fprintf(stderr, "JSON parse error: %s\n", error.text);
        close(sockfd);
        return -1;
    }

    int result = json_integer_value(json_object_get(resp_json, "result"));
    json_decref(resp_json);

    printf("← Response received: result=%d\n\n", result);

    close(sockfd);
    return result;
}

int main() 
{
    // 调用RPC：add(10, 20)
    int result1 = rpc_call("127.0.0.1", 5000, 0, 10, 20);
    printf("10 + 20 = %d\n\n", result1);

    // 调用RPC：sub(30, 10)
    int result2 = rpc_call("127.0.0.1", 5000, 1, 30, 10);
    printf("30 - 10 = %d\n\n", result2);

    return 0;
}
```

**编译运行**：

```bash
# 安装jansson库
sudo apt install libjansson-dev

# 编译
gcc rpc_server.c -o rpc_server -ljansson
gcc rpc_client.c -o rpc_client -ljansson

# 运行服务器
./rpc_server

# 运行客户端
./rpc_client
```

#### 主流RPC框架对比

| 框架           | 语言     | 序列化      | 传输协议     | 服务发现            | 特点             |
|--------------|--------|----------|----------|-----------------|----------------|
| **gRPC**     | 多语言    | Protobuf | HTTP/2   | 需插件             | 高性能，流式RPC      |
| **Thrift**   | 多语言    | Thrift   | TCP/HTTP | 需自行实现           | 跨语言，Facebook开源 |
| **Dubbo**    | Java为主 | 多种       | TCP      | Zookeeper/Nacos | 阿里开源，生态完善      |
| **JSON-RPC** | 多语言    | JSON     | HTTP     | 无               | 简单，兼容性好        |

---

### 4.2 服务发现机制

#### 为什么需要服务发现？

```
硬编码问题：
  客户端 → 192.168.1.10:8080  (服务器IP固定)
  
  问题：
  - 服务器IP变化需要修改代码
  - 无法动态扩缩容
  - 无法故障转移

服务发现解决：
  客户端 → 服务注册中心 → 查询"user-service"
                       → 返回：[192.168.1.10:8080, 192.168.1.11:8080]
  客户端选择一个健康的实例调用
```

#### 服务注册与发现流程

```
┌────────────────────────────────────────────────────────┐
│                  服务注册中心                            │
│         (Consul / etcd / Zookeeper / Nacos)            │
│                                                        │
│  服务列表：                                              │
│    user-service:                                       │
│      - 192.168.1.10:8080 (healthy)                     │
│      - 192.168.1.11:8080 (healthy)                     │
│      - 192.168.1.12:8080 (unhealthy) ✗                 │
│    order-service:                                      │
│      - 192.168.1.20:9000 (healthy)                     │
└────────────────────────────────────────────────────────┘
        ↑ 注册/心跳         ↓ 查询服务
        │                   │
┌───────────────┐    ┌─────────────┐
│ 服务提供者      │    │ 服务消费者   │
│ (user-service)│    │ (web-app)   │
│               │    │             │
│ 启动时注册      │    │ 查询并调用   │
│ 定期发送心跳    │    │             │
└───────────────┘    └─────────────┘

流程：
1. 服务提供者启动 → 注册到服务中心
2. 服务中心记录：服务名、IP、端口、健康状态
3. 服务提供者定期发送心跳（如每10秒）
4. 服务消费者启动 → 查询服务列表
5. 服务消费者选择一个实例调用
6. 服务中心检测心跳超时 → 标记为不健康
```

#### 使用etcd实现服务注册与发现

```c
// service_register_etcd.c - 使用etcd注册服务
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <curl/curl.h>

// 服务注册（简化版，实际应用使用etcd客户端库）
int register_service(const char *service_name, const char *instance_addr, int ttl) 
{
    CURL *curl;
    CURLcode res;
    char url[256], data[256];

    // etcd v3 API端点
    snprintf(url, sizeof(url), "http://localhost:2379/v3/lease/grant");

    // 1. 创建租约（TTL秒后自动过期）
    snprintf(data, sizeof(data), "{\"TTL\": %d}", ttl);

    curl = curl_easy_init();
    curl_easy_setopt(curl, CURLOPT_URL, url);
    curl_easy_setopt(curl, CURLOPT_POSTFIELDS, data);
    
    res = curl_easy_perform(curl);
    if (res != CURLE_OK) 
    {
        fprintf(stderr, "curl failed: %s\n", curl_easy_strerror(res));
        curl_easy_cleanup(curl);
        return -1;
    }

    // 2. 注册服务（绑定到租约）
    snprintf(url, sizeof(url), "http://localhost:2379/v3/kv/put");
    snprintf(data, sizeof(data), "{\"key\": \"/services/%s/%s\", \"value\": \"%s\", \"lease\": \"lease_id\"}",
         service_name, instance_addr, instance_addr);

    curl_easy_setopt(curl, CURLOPT_URL, url);
    curl_easy_setopt(curl, CURLOPT_POSTFIELDS, data);
    curl_easy_perform(curl);

    curl_easy_cleanup(curl);

    printf("✓ Service registered: %s -> %s\n", service_name, instance_addr);
    return 0;
}

// 心跳保活
void heartbeat_loop(const char *service_name, const char *instance_addr) 
{
    while (1) 
    {
        // 每10秒续约
        // lease_keepalive() ...
        
        printf("❤ Heartbeat: %s\n", instance_addr);
        sleep(10);
    }
}

int main() 
{
    const char *service_name = "user-service";
    const char *instance_addr = "192.168.1.10:8080";

    // 注册服务
    register_service(service_name, instance_addr, 30);

    // 心跳保活
    heartbeat_loop(service_name, instance_addr);

    return 0;
}
```

```c
// service_discovery_etcd.c - 服务发现
#include <stdio.h>
#include <curl/curl.h>

// 查询服务实例
void discover_service(const char *service_name) 
{
    CURL *curl;
    char url[256];

    snprintf(url, sizeof(url), "http://localhost:2379/v3/kv/range");

    // 查询前缀为 /services/user-service/ 的所有key
    const char *data = "{\"key\": \"/services/user-service/\", \"range_end\": \"/services/user-service0\"}";

    curl = curl_easy_init();
    curl_easy_setopt(curl, CURLOPT_URL, url);
    curl_easy_setopt(curl, CURLOPT_POSTFIELDS, data);
    
    // 解析响应获取服务列表
    curl_easy_perform(curl);

    // 输出示例：
    // /services/user-service/192.168.1.10:8080 -> 192.168.1.10:8080
    // /services/user-service/192.168.1.11:8080 -> 192.168.1.11:8080

    curl_easy_cleanup(curl);
}
```

---

### 4.3 负载均衡策略

#### 常见负载均衡算法

**1. 轮询（Round Robin）**

```c
// 轮询负载均衡
typedef struct 
{
    char **instances;  // 实例列表
    int count;         // 实例数量
    int current;       // 当前索引
} round_robin_lb_t;

const char *round_robin_select(round_robin_lb_t *lb) 
{
    if (lb->count == 0) return NULL;
    
    const char *selected = lb->instances[lb->current];
    lb->current = (lb->current + 1) % lb->count;
    
    return selected;
}

// 示例：
// 实例：[A, B, C]
// 请求1 → A
// 请求2 → B
// 请求3 → C
// 请求4 → A (循环)

优点：简单，分布均匀
缺点：不考虑实例负载
```

**2. 加权轮询（Weighted Round Robin）**

```c
// 加权轮询
typedef struct 
{
    char *addr;
    int weight;        // 权重
    int current_weight; // 当前权重
} instance_t;

const char *weighted_round_robin_select(instance_t *instances, int count) 
{
    int total_weight = 0;
    instance_t *selected = NULL;
    for (int i = 0; i < count; i++) 
    {
        instances[i].current_weight += instances[i].weight;
        total_weight += instances[i].weight;
        if (!selected || instances[i].current_weight > selected->current_weight) 
        {
            selected = &instances[i];
        }
    }
    selected->current_weight -= total_weight;
    return selected->addr;
}

// 示例：
// 实例：A(权重5), B(权重1), C(权重1)
// 请求序列：A, A, A, B, A, C, A

优点：考虑实例性能差异
缺点：需要预先配置权重
```

**3. 随机（Random）**

```c
// 随机选择
const char *random_select(char **instances, int count) 
{
    if (count == 0) return NULL;
    int idx = rand() % count;
    return instances[idx];
}

优点：简单，无状态
缺点：短期内可能不均匀
```

**4. 最少连接（Least Connections）**

```c
// 最少连接
typedef struct 
{
    char *addr;
    int active_conns;  // 当前活跃连接数
} instance_conn_t;

const char *least_conn_select(instance_conn_t *instances, int count) 
{
    if (count == 0) return NULL;
    instance_conn_t *selected = &instances[0];
    for (int i = 1; i < count; i++) 
    {
        if (instances[i].active_conns < selected->active_conns) 
        {
            selected = &instances[i];
        }
    }

    selected->active_conns++;  // 选中后增加计数
    return selected->addr;
}

优点：考虑实际负载
缺点：需要维护连接状态
```

**5. 一致性哈希（Consistent Hashing）**

```c
// 一致性哈希
#include <stdint.h>

uint32_t hash(const char *key) 
{
    // 简单哈希（实际使用MurmurHash等）
    uint32_t h = 0;
    while (*key) 
    {
        h = h * 31 + *key++;
    }
    return h;
}

const char *consistent_hash_select(char **instances, int count, const char *key) 
{
    if (count == 0) return NULL;

    uint32_t key_hash = hash(key);
    
    // 找到哈希环上第一个大于key_hash的节点
    uint32_t min_dist = UINT32_MAX;
    int selected_idx = 0;

    for (int i = 0; i < count; i++) 
    {
        uint32_t instance_hash = hash(instances[i]);
        uint32_t dist;
        
        if (instance_hash >= key_hash) 
        {
            dist = instance_hash - key_hash;
        } 
        else 
        {
            dist = UINT32_MAX - key_hash + instance_hash;
        }

        if (dist < min_dist) {
            min_dist = dist;
            selected_idx = i;
        }
    }

    return instances[selected_idx];
}

// 示例：
// 用户ID作为key
// user_id="123" → hash → 选择实例A
// user_id="456" → hash → 选择实例B
// 同一用户总是路由到同一实例（会话亲和性）

优点：
- 会话亲和性（同一key总是路由到同一实例）
- 扩缩容时只影响少量key

缺点：
- 实现复杂
- 可能不均匀（需要虚拟节点）
```

---

### 4.4 熔断、降级、限流

#### 熔断器（Circuit Breaker）

```
熔断器状态机：

        失败率过高
  CLOSED ────────→ OPEN
    ↑                 │
    │                 │ 超时后进入半开
    │                 ↓
    └──── HALF_OPEN
         成功则关闭，失败则再次打开

CLOSED（关闭）：
  - 正常调用远程服务
  - 记录成功/失败次数
  - 失败率 > 阈值 → OPEN

OPEN（打开）：
  - 直接返回错误，不调用远程服务
  - 避免雪崩
  - 等待超时（如30秒）→ HALF_OPEN

HALF_OPEN（半开）：
  - 尝试一次调用
  - 成功 → CLOSED
  - 失败 → OPEN
```

**熔断器实现**：

```c
// circuit_breaker.c
#include <stdio.h>
#include <time.h>

typedef enum 
{
    STATE_CLOSED,
    STATE_OPEN,
    STATE_HALF_OPEN,
} circuit_state_t;

typedef struct 
{
    circuit_state_t state;
    int failure_count;
    int success_count;
    int failure_threshold;  // 失败阈值（如5次）
    time_t open_time;       // 打开时间
    int timeout;            // 超时秒数（如30秒）
} circuit_breaker_t;

void circuit_breaker_init(circuit_breaker_t *cb, int failure_threshold, int timeout) 
{
    cb->state = STATE_CLOSED;
    cb->failure_count = 0;
    cb->success_count = 0;
    cb->failure_threshold = failure_threshold;
    cb->timeout = timeout;
}

// 检查是否允许调用
int circuit_breaker_allow_request(circuit_breaker_t *cb) 
{
    if (cb->state == STATE_CLOSED) 
    {
        return 1;  // 允许
    }

    if (cb->state == STATE_OPEN) 
    {
        // 检查是否超时
        time_t now = time(NULL);
        if (now - cb->open_time >= cb->timeout) 
        {
            // 进入半开状态
            cb->state = STATE_HALF_OPEN;
            printf("Circuit breaker: OPEN → HALF_OPEN\n");
            return 1;  // 尝试一次
        }
        return 0;  // 拒绝
    }

    if (cb->state == STATE_HALF_OPEN) 
    {
        return 1;  // 尝试一次
    }

    return 0;
}

// 记录成功
void circuit_breaker_on_success(circuit_breaker_t *cb) 
{
    cb->failure_count = 0;
    cb->success_count++;

    if (cb->state == STATE_HALF_OPEN) 
    {
        // 半开状态成功 → 关闭
        cb->state = STATE_CLOSED;
        printf("Circuit breaker: HALF_OPEN → CLOSED\n");
    }
}

// 记录失败
void circuit_breaker_on_failure(circuit_breaker_t *cb) 
{
    cb->failure_count++;

    if (cb->state == STATE_HALF_OPEN)
    {
        // 半开状态失败 → 打开
        cb->state = STATE_OPEN;
        cb->open_time = time(NULL);
        printf("Circuit breaker: HALF_OPEN → OPEN\n");
    } 
    else if (cb->state == STATE_CLOSED) 
    {
        // 失败次数超过阈值 → 打开
        if (cb->failure_count >= cb->failure_threshold) 
        {
            cb->state = STATE_OPEN;
            cb->open_time = time(NULL);
            printf("Circuit breaker: CLOSED → OPEN (failures=%d)\n", cb->failure_count);
        }
    }
}

// 使用示例
int call_remote_service(circuit_breaker_t *cb) 
{
    if (!circuit_breaker_allow_request(cb)) 
    {
        printf("✗ Request rejected by circuit breaker\n");
        return -1;  // 快速失败
    }

    // 调用远程服务
    printf("→ Calling remote service...\n");
    int success = (rand() % 10) < 7;  // 70%成功率

    if (success) 
    {
        printf("✓ Remote call succeeded\n");
        circuit_breaker_on_success(cb);
        return 0;
    } 
    else 
    {
        printf("✗ Remote call failed\n");
        circuit_breaker_on_failure(cb);
        return -1;
    }
}

int main() 
{
    circuit_breaker_t cb;
    circuit_breaker_init(&cb, 3, 10);  // 3次失败后熔断，10秒后尝试恢复

    for (int i = 0; i < 20; i++) 
    {
        printf("\n--- Request %d ---\n", i + 1);
        call_remote_service(&cb);
        sleep(2);
    }

    return 0;
}
```

#### 限流（Rate Limiting）

**1. 固定窗口算法**

```c
// 固定窗口限流
typedef struct 
{
    int limit;         // 限制：每秒N个请求
    int counter;       // 当前计数
    time_t window_start; // 窗口开始时间
} fixed_window_limiter_t;

int fixed_window_allow(fixed_window_limiter_t *limiter) 
{
    time_t now = time(NULL);

    if (now != limiter->window_start) 
    {
        // 新窗口
        limiter->window_start = now;
        limiter->counter = 0;
    }

    if (limiter->counter < limiter->limit) 
    {
        limiter->counter++;
        return 1;  // 允许
    }

    return 0;  // 拒绝
}

// 问题：窗口边界突刺
// 例如：限制100 req/s
// 0.9秒：100个请求
// 1.1秒：100个请求
// 0.2秒内200个请求（突破限制）
```

**2. 滑动窗口算法**

```c
// 滑动窗口限流（简化版）
typedef struct 
{
    int limit;
    time_t *timestamps;  // 时间戳数组
    int size;
    int head;
} sliding_window_limiter_t;

int sliding_window_allow(sliding_window_limiter_t *limiter) 
{
    time_t now = time(NULL);

    // 移除1秒前的时间戳
    while (limiter->size > 0 &&  now - limiter->timestamps[limiter->head] > 1) 
    {
        limiter->head = (limiter->head + 1) % limiter->limit;
        limiter->size--;
    }

    if (limiter->size < limiter->limit) 
    {
        int tail = (limiter->head + limiter->size) % limiter->limit;
        limiter->timestamps[tail] = now;
        limiter->size++;
        return 1;
    }

    return 0;
}

优点：平滑，无突刺
缺点：内存开销大
```

**3. 令牌桶算法（Token Bucket）**

```c
// 令牌桶限流
typedef struct 
{
    int capacity;      // 桶容量
    int tokens;        // 当前令牌数
    int refill_rate;   // 每秒添加N个令牌
    time_t last_refill; // 上次添加时间
} token_bucket_limiter_t;

int token_bucket_allow(token_bucket_limiter_t *limiter) 
{
    time_t now = time(NULL);

    // 添加令牌
    int elapsed = now - limiter->last_refill;
    if (elapsed > 0) 
    {
        int new_tokens = elapsed * limiter->refill_rate;
        limiter->tokens = (limiter->tokens + new_tokens > limiter->capacity)
                          ? limiter->capacity
                          : limiter->tokens + new_tokens;
        limiter->last_refill = now;
    }

    // 消耗令牌
    if (limiter->tokens > 0) 
    {
        limiter->tokens--;
        return 1;
    }

    return 0;
}

优点：
- 允许突发流量（桶内有积累的令牌）
- 实现简单
- 性能好

应用：
- Nginx: limit_req（漏桶）
- 云服务API限流：大多使用令牌桶
```

**4. 漏桶算法（Leaky Bucket）**

```c
// 漏桶限流
typedef struct 
{
    int capacity;      // 桶容量
    int water;         // 当前水量
    int leak_rate;     // 每秒漏出N个请求
    time_t last_leak;  // 上次漏出时间
} leaky_bucket_limiter_t;

int leaky_bucket_allow(leaky_bucket_limiter_t *limiter) 
{
    time_t now = time(NULL);

    // 漏水
    int elapsed = now - limiter->last_leak;
    if (elapsed > 0) {
        int leaked = elapsed * limiter->leak_rate;
        limiter->water = (limiter->water > leaked) ? (limiter->water - leaked) : 0;
        limiter->last_leak = now;
    }

    // 加水
    if (limiter->water < limiter->capacity) 
    {
        limiter->water++;
        return 1;
    }

    return 0;
}

优点：流量平滑，不允许突发
缺点：限制了突发流量的灵活性
```

---

### 4.5 高频面试题

#### Q1: RPC和HTTP有什么区别？什么时候用RPC？

**答案**：

| 特性       | RPC             | HTTP/REST    |
|----------|-----------------|--------------|
| **调用方式** | 像调用本地函数         | 发送HTTP请求     |
| **性能**   | 高（二进制协议）        | 中等（文本协议）     |
| **序列化**  | Protobuf/Thrift | JSON/XML     |
| **传输层**  | TCP/HTTP/2      | HTTP/1.1     |
| **服务发现** | 通常内置            | 需自行实现        |
| **跨语言**  | 支持（需工具生成代码）     | 天然支持         |
| **调试**   | 困难（二进制）         | 容易（curl/浏览器） |

**使用场景**：

```
RPC适合：
✓ 微服务内部调用（性能要求高）
✓ 同公司不同服务（统一技术栈）
✓ 高频调用（如每秒上万次）

HTTP/REST适合：
✓ 对外开放API（跨公司、跨语言）
✓ 低频调用（如配置查询）
✓ 需要浏览器直接访问

实际案例：
- Google内部：gRPC（高性能）
- Google对外API：REST（兼容性）
- 微信后端服务：自研RPC（性能）
- 微信公众平台API：REST（开放性）
```

---

#### Q2: 什么是服务雪崩？如何防止？

**答案**：

**雪崩现象**：

```
场景：
  用户请求 → Web服务 → 订单服务 → 库存服务（慢/挂了）

雪崩过程：
1. 库存服务响应慢（比如数据库查询慢）
2. 订单服务等待超时，线程阻塞
3. 订单服务所有线程被占满
4. Web服务调用订单服务超时，线程阻塞
5. Web服务线程耗尽，无法响应用户请求
6. 用户请求失败，重试 → 更多请求 → 系统崩溃

结果：一个服务的问题导致整个系统瘫痪
```

**防止措施**：

**1. 熔断器（Circuit Breaker）**
```
订单服务调用库存服务失败超过阈值
→ 熔断器打开
→ 快速失败，不再调用库存服务
→ 避免线程阻塞
→ 定期尝试恢复
```

**2. 超时设置**
```c
// 设置每个RPC调用的超时
struct timeval timeout = {.tv_sec = 2, .tv_usec = 0};
setsockopt(sockfd, SOL_SOCKET, SO_RCVTIMEO, &timeout, sizeof(timeout));

// 而不是无限等待（默认）
```

**3. 限流**
```
订单服务设置：最多处理1000 req/s
超过的请求直接拒绝（返回429 Too Many Requests）
避免被打垮
```

**4. 降级**
```
库存服务不可用时：
  - 不调用库存服务
  - 返回默认库存（如"有货"）
  - 或显示"库存信息暂时无法获取"
保证主流程可用
```

**5. 隔离**
```
线程池隔离：
  - 订单服务有100个线程
  - 只用20个线程调用库存服务
  - 即使库存服务挂了，只阻塞20个线程
  - 其他80个线程可以处理其他请求
```

**6. 限制重试**
```
// ✗ 无限重试会加剧雪崩
for (int i = 0; i < INT_MAX; i++) 
{
    if (call_service() == 0) break;
}

// ✓ 限制重试次数和间隔
for (int i = 0; i < 3; i++) 
{
    if (call_service() == 0) break;
    sleep(1 << i);  // 指数退避：1s, 2s, 4s
}
```

---

#### Q3: 一致性哈希的原理是什么？为什么需要虚拟节点？

**答案**：

**一致性哈希原理**：

```
传统哈希：
  node_idx = hash(key) % node_count
  
  问题：增加/删除节点时，大部分key的映射都会改变
  例如：3个节点 → 4个节点
    key1: hash(key1) % 3 = 2 → hash(key1) % 4 = 1  (变化)
    key2: hash(key2) % 3 = 0 → hash(key2) % 4 = 0  (不变)
    key3: hash(key3) % 3 = 1 → hash(key3) % 4 = 3  (变化)
  结果：大量key需要迁移

一致性哈希：
  将hash值空间想象成一个环（0 ~ 2^32-1）
  
  哈希环：
    0 ─────→ 2^32-1 ─────┐
    ↑                    │
    └────────────────────┘

  1. 节点映射到环上：
     Node A: hash("Node A") = 1000
     Node B: hash("Node B") = 2000
     Node C: hash("Node C") = 3000

  2. Key映射到环上，顺时针找第一个节点：
     key1: hash(key1) = 500   → Node A (1000)
     key2: hash(key2) = 1500  → Node B (2000)
     key3: hash(key3) = 2500  → Node C (3000)

  3. 增加节点 Node D (hash = 1800)：
     只有 key2 需要迁移（1500 → Node D）
     其他key不变

  优点：增删节点只影响相邻节点，大部分key不变
```

**为什么需要虚拟节点？**

```
问题：节点数量少时，分布不均匀

示例：3个节点
  Node A: 1000
  Node B: 2000  
  Node C: 3000

  哈希环上分布：
  [0, 1000) → Node A  (25%)
  [1000, 2000) → Node B  (25%)
  [2000, 3000) → Node C  (25%)
  [3000, 2^32) → Node A  (25%)

  如果hash分布不均匀，某个节点可能负载很高

解决：虚拟节点

  每个物理节点映射到环上多个位置：
  Node A → "Node A#1", "Node A#2", "Node A#3", ...
           hash → 100, 1500, 2800, ...

  Node B → "Node B#1", "Node B#2", "Node B#3", ...
           hash → 500, 2200, 3500, ...

  Node C → "Node C#1", "Node C#2", "Node C#3", ...
           hash → 800, 1800, 3200, ...

  这样环上有3 * N个虚拟节点，分布更均匀

  key1 → 找到"Node A#2" → 实际调用 Node A
  key2 → 找到"Node B#3" → 实际调用 Node B

优点：负载更均衡
通常：每个物理节点对应150-200个虚拟节点
```

**代码示例**：

```c
// 一致性哈希with虚拟节点
#define VIRTUAL_NODE_COUNT 150

typedef struct 
{
    uint32_t hash;   // 虚拟节点的hash
    int physical_idx; // 对应的物理节点索引
} virtual_node_t;

virtual_node_t virtual_nodes[MAX_NODES * VIRTUAL_NODE_COUNT];
int virtual_node_count = 0;

void add_physical_node(const char *addr, int idx) 
{
    for (int i = 0; i < VIRTUAL_NODE_COUNT; i++) 
    {
        char vnode_name[256];
        snprintf(vnode_name, sizeof(vnode_name), "%s#%d", addr, i);
        
        virtual_nodes[virtual_node_count].hash = hash(vnode_name);
        virtual_nodes[virtual_node_count].physical_idx = idx;
        virtual_node_count++;
    }
    
    // 排序虚拟节点（按hash值）
    qsort(virtual_nodes, virtual_node_count, sizeof(virtual_node_t), cmp);
}

int consistent_hash_select(const char *key) 
{
    uint32_t key_hash = hash(key);
    
    // 二分查找第一个 >= key_hash 的虚拟节点
    int left = 0, right = virtual_node_count - 1;
    while (left < right) 
    {
        int mid = (left + right) / 2;
        if (virtual_nodes[mid].hash < key_hash) 
        {
            left = mid + 1;
        } 
        else 
        {
            right = mid;
        }
    }
    
    return virtual_nodes[left].physical_idx;
}
```

---

#### Q4: 熔断、降级、限流的区别是什么？

**答案**：

|          | 熔断 (Circuit Breaker) | 降级 (Fallback) | 限流 (Rate Limiting) |
|----------|----------------------|---------------|--------------------|
| **目的**   | 保护调用方，避免雪崩           | 保证核心功能可用      | 保护服务方，避免过载         |
| **触发条件** | 错误率过高                | 依赖服务不可用       | 请求速率过高             |
| **动作**   | 快速失败，不调用             | 返回默认值/兜底数据    | 拒绝部分请求             |
| **恢复**   | 自动尝试恢复               | 依赖服务恢复后自动     | 流量降低后自动            |

**示例场景**：

```
电商系统：查询商品详情

正常流程：
  用户 → Web → 商品服务 → 库存服务
                       → 评论服务
                       → 推荐服务

1. 限流：
   商品服务设置：最多10000 req/s
   超过限制：返回429错误
   目的：保护商品服务不被打垮

2. 熔断：
   评论服务连续失败5次
   → 熔断器打开
   → 不再调用评论服务，直接返回错误
   → 30秒后尝试恢复
   目的：避免商品服务被评论服务拖垮

3. 降级：
   推荐服务不可用
   → 不返回个性化推荐
   → 返回热门商品（兜底数据）
   目的：保证商品详情页可访问

综合应用：
  用户请求
    ↓ 限流（保护商品服务）
  商品服务
    ├→ 库存服务（核心，不降级）
    ├→ 评论服务（熔断：失败太多则快速失败）
    └→ 推荐服务（降级：失败则返回默认推荐）
```

**实践建议**：

```
核心服务（必须调用）：
  - 限流保护
  - 超时设置
  - 重试（有限次数）

非核心服务（可降级）：
  - 熔断保护
  - 降级策略（返回默认值）
  - 异步调用（不阻塞主流程）

对外API（保护自己）：
  - 限流（令牌桶/漏桶）
  - 黑名单
  - IP限制
```

---

## 5. 网络安全

### 5.1 TLS/SSL 原理

#### TLS握手过程

```
TLS 1.3 握手（1-RTT）：

客户端                                    服务器
   │                                         │
   │─── ClientHello ─────────────────────→   │
   │    - 支持的加密套件                       │
   │    - 随机数 (Client Random)              │
   │    - 支持的TLS版本                        │
   │                                         │
   │  ←─── ServerHello ───────────────────── │
   │  ←─── Certificate (服务器证书) ─────────  │
   │  ←─── ServerKeyExchange ─────────────── │
   │  ←─── ServerHelloDone ───────────────── │
   │      - 选择的加密套件                     │
   │      - 随机数 (Server Random)            │
   │      - 公钥                              │
   │                                         │
   │─── ClientKeyExchange ───────────────→   │
   │─── ChangeCipherSpec ────────────────→   │
   │─── Finished (加密) ──────────────────→   │
   │    - Pre-Master Secret (用服务器公钥加密)  │
   │                                         │
   │  ←─── ChangeCipherSpec ──────────────── │
   │  ←─── Finished (加密) ────────────────── │
   │                                         │
   │←───────── 加密通信 ─────────────────→     │

密钥派生：
1. Pre-Master Secret (48字节随机数)
2. Master Secret = PRF(Pre-Master, Client Random, Server Random)
3. 派生6个密钥：
   - Client Write MAC Key
   - Server Write MAC Key
   - Client Write Encryption Key
   - Server Write Encryption Key
   - Client Write IV
   - Server Write IV
```

#### TLS证书验证

```
证书链验证：

网站证书（叶子证书）
  ↓ 签发者
中间CA证书
  ↓ 签发者
根CA证书（系统信任）

验证过程：
1. 检查证书域名与访问域名是否匹配
2. 检查证书是否在有效期内
3. 验证证书签名：
   - 用中间CA的公钥验证网站证书的签名
   - 用根CA的公钥验证中间CA证书的签名
4. 检查证书是否被吊销（CRL/OCSP）
```

#### TLS编程示例（OpenSSL）

```c
// tls_server.c - 简单的TLS服务器
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <openssl/ssl.h>
#include <openssl/err.h>

void init_openssl() 
{
    SSL_load_error_strings();
    OpenSSL_add_ssl_algorithms();
}

void cleanup_openssl() 
{
    EVP_cleanup();
}

SSL_CTX *create_context() 
{
    const SSL_METHOD *method;
    SSL_CTX *ctx;

    method = TLS_server_method();  // TLS 1.3
    ctx = SSL_CTX_new(method);
    if (!ctx) 
    {
        perror("Unable to create SSL context");
        ERR_print_errors_fp(stderr);
        exit(1);
    }

    return ctx;
}

void configure_context(SSL_CTX *ctx) 
{
    // 加载证书
    if (SSL_CTX_use_certificate_file(ctx, "cert.pem", SSL_FILETYPE_PEM) <= 0) 
    {
        ERR_print_errors_fp(stderr);
        exit(1);
    }

    // 加载私钥
    if (SSL_CTX_use_PrivateKey_file(ctx, "key.pem", SSL_FILETYPE_PEM) <= 0) 
    {
        ERR_print_errors_fp(stderr);
        exit(1);
    }

    // 验证私钥与证书是否匹配
    if (!SSL_CTX_check_private_key(ctx)) 
    {
        fprintf(stderr, "Private key does not match the certificate\n");
        exit(1);
    }
}

int main() 
{
    int listen_fd, client_fd;
    struct sockaddr_in addr;
    SSL_CTX *ctx;

    init_openssl();
    ctx = create_context();
    configure_context(ctx);

    // 创建监听socket
    listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    int reuse = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

    addr.sin_family = AF_INET;
    addr.sin_port = htons(4433);
    addr.sin_addr.s_addr = INADDR_ANY;

    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 1);

    printf("TLS server listening on port 4433...\n");

    while (1) 
    {
        client_fd = accept(listen_fd, NULL, NULL);
        printf("✓ New connection\n");

        // 创建SSL对象
        SSL *ssl = SSL_new(ctx);
        SSL_set_fd(ssl, client_fd);

        // TLS握手
        if (SSL_accept(ssl) <= 0) 
        {
            ERR_print_errors_fp(stderr);
        } 
        else 
        {
            printf("✓ TLS handshake completed\n");
            printf("  Cipher: %s\n", SSL_get_cipher(ssl));

            // 接收加密数据
            char buf[1024];
            int bytes = SSL_read(ssl, buf, sizeof(buf));
            if (bytes > 0) 
            {
                buf[bytes] = '\0';
                printf("← Received: %s\n", buf);

                // 发送加密数据
                const char *reply = "Hello from TLS server!";
                SSL_write(ssl, reply, strlen(reply));
                printf("→ Sent: %s\n", reply);
            }
        }

        SSL_shutdown(ssl);
        SSL_free(ssl);
        close(client_fd);
    }

    close(listen_fd);
    SSL_CTX_free(ctx);
    cleanup_openssl();
    return 0;
}
```

**生成自签名证书**：

```bash
# 生成私钥
openssl genrsa -out key.pem 2048

# 生成证书签名请求（CSR）
openssl req -new -key key.pem -out csr.pem

# 生成自签名证书
openssl x509 -req -days 365 -in csr.pem -signkey key.pem -out cert.pem

# 编译
gcc tls_server.c -o tls_server -lssl -lcrypto

# 运行
./tls_server

# 测试
openssl s_client -connect localhost:4433
```

---

### 5.2 DDoS 防御

#### 常见DDoS攻击类型

**1. SYN Flood**

```
原理：利用TCP三次握手
攻击者 ──SYN──→ 服务器（分配资源，等待ACK）
攻击者不发送ACK，服务器维持大量半连接

防御：SYN Cookie

正常流程：
  SYN → 分配内存存储连接信息
  SYN+ACK ←
  ACK → 建立连接

SYN Cookie：
  SYN → 不分配内存，计算Cookie
        Cookie = Hash(客户端IP, 端口, 时间戳, 密钥)
  SYN+ACK(seq=Cookie) ←
  ACK(ack=Cookie+1) → 验证Cookie，建立连接

Linux开启SYN Cookie：
  sysctl -w net.ipv4.tcp_syncookies=1
```

**2. UDP Flood**

```
原理：发送大量UDP包

防御：
1. 限速（iptables）
   iptables -A INPUT -p udp -m limit --limit 100/s -j ACCEPT
   iptables -A INPUT -p udp -j DROP

2. 无连接验证（挑战-响应）
   服务器 → 客户端：Challenge
   客户端 → 服务器：Response = Hash(Challenge)
   验证通过后才处理请求
```

**3. HTTP Flood**

```
原理：大量HTTP请求（应用层攻击）

防御：
1. Rate Limiting（限流）
2. JavaScript挑战
   返回JavaScript代码，计算结果后才允许访问
   （机器人难以执行JavaScript）
3. CAPTCHA（验证码）
4. CDN（分散流量）
```

---

### 5.3 限流算法（详细）

#### 令牌桶算法（Token Bucket） - 推荐

```c
// token_bucket.c - 完整的令牌桶实现
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <sys/time.h>

typedef struct 
{
    int capacity;           // 桶容量
    int tokens;            // 当前令牌数
    double refill_rate;    // 每秒添加N个令牌
    struct timeval last_refill; // 上次添加时间（微秒精度）
} token_bucket_t;

void token_bucket_init(token_bucket_t *tb, int capacity, double refill_rate) 
{
    tb->capacity = capacity;
    tb->tokens = capacity;  // 初始满
    tb->refill_rate = refill_rate;
    gettimeofday(&tb->last_refill, NULL);
}

int token_bucket_consume(token_bucket_t *tb, int count) 
{
    struct timeval now;
    gettimeofday(&now, NULL);

    // 计算时间差（秒）
    double elapsed = (now.tv_sec - tb->last_refill.tv_sec) + (now.tv_usec - tb->last_refill.tv_usec) / 1000000.0;

    // 添加令牌
    if (elapsed > 0) 
    {
        int new_tokens = (int)(elapsed * tb->refill_rate);
        tb->tokens += new_tokens;
        if (tb->tokens > tb->capacity) 
        {
            tb->tokens = tb->capacity;
        }
        tb->last_refill = now;
    }

    // 消耗令牌
    if (tb->tokens >= count) 
    {
        tb->tokens -= count;
        return 1;  // 允许
    }

    return 0;  // 拒绝
}

int main() {
    token_bucket_t tb;
    token_bucket_init(&tb, 10, 5.0);  // 容量10，每秒添加5个

    printf("Token Bucket: capacity=10, refill_rate=5/s\n\n");

    for (int i = 0; i < 30; i++) {
        int allowed = token_bucket_consume(&tb, 1);
        printf("Request %2d: %s (tokens=%d)\n", 
               i + 1, allowed ? "✓ PASS" : "✗ REJECT", tb.tokens);

        usleep(100000);  // 0.1秒间隔（10 req/s）
    }

    return 0;
}
```

---

### 5.4 降级策略

#### 降级方案设计

```c
// service_degradation.c - 服务降级示例
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// 降级级别
typedef enum 
{
    LEVEL_NORMAL,      // 正常
    LEVEL_DEGRADED,    // 部分降级
    LEVEL_CRITICAL,    // 严重降级
} degradation_level_t;

degradation_level_t current_level = LEVEL_NORMAL;

// 根据系统负载调整降级级别
void adjust_degradation_level(int cpu_usage, int error_rate) 
{
    if (cpu_usage > 90 || error_rate > 50) 
    {
        current_level = LEVEL_CRITICAL;
        printf("⚠ Degradation: CRITICAL\n");
    } 
    else if (cpu_usage > 70 || error_rate > 20) 
    {
        current_level = LEVEL_DEGRADED;
        printf("⚠ Degradation: DEGRADED\n");
    } 
    else 
    {
        current_level = LEVEL_NORMAL;
        printf("✓ Degradation: NORMAL\n");
    }
}

// 商品详情页查询
void get_product_detail(int product_id) 
{
    // 1. 核心数据（不降级）
    printf("  [Core] Product info: id=%d, name=..., price=...\n", product_id);

    // 2. 评论（降级处理）
    if (current_level == LEVEL_NORMAL) 
    {
        printf("  [Optional] Reviews: 5 stars, 100 reviews\n");
    } 
    else 
    {
        printf("  [Degraded] Reviews: Loading...\n");  // 兜底数据
    }

    // 3. 推荐（严重时完全关闭）
    if (current_level == LEVEL_NORMAL) 
    {
        printf("  [Optional] Recommendations: Product A, B, C\n");
    } 
    else if (current_level == LEVEL_DEGRADED) 
    {
        printf("  [Degraded] Recommendations: Hot products\n");  // 缓存数据
    } 
    else 
    {
        // CRITICAL: 不调用推荐服务
    }
}

int main() 
{
    // 模拟不同负载场景
    printf("--- Scenario 1: Normal ---\n");
    adjust_degradation_level(50, 5);
    get_product_detail(123);

    printf("\n--- Scenario 2: High Load ---\n");
    adjust_degradation_level(75, 25);
    get_product_detail(123);

    printf("\n--- Scenario 3: Critical ---\n");
    adjust_degradation_level(95, 60);
    get_product_detail(123);

    return 0;
}
```

---

### 5.5 高频面试题

#### Q1: HTTPS是如何保证安全的？

**答案**：

HTTPS = HTTP + TLS，提供三大安全保障：

**1. 加密（Encryption）**
```
问题：HTTP明文传输，可被窃听
解决：TLS加密

对称加密：
  - 通信双方用相同密钥加密/解密
  - 算法：AES-256
  - 问题：如何安全地交换密钥？

非对称加密：
  - 公钥加密，私钥解密
  - 算法：RSA/ECDSA
  - 解决：密钥交换问题

HTTPS结合：
  1. 非对称加密交换对称密钥（握手阶段）
  2. 对称加密加密数据（通信阶段，快）
```

**2. 身份认证（Authentication）**
```
问题：如何确认服务器身份？
解决：数字证书

证书包含：
  - 服务器公钥
  - 服务器域名
  - 证书有效期
  - CA签名

验证过程：
  1. 浏览器获取服务器证书
  2. 验证证书签名（用CA公钥）
  3. 验证域名匹配
  4. 验证有效期
  5. 验证通过 → 信任服务器
```

**3. 完整性（Integrity）**
```
问题：数据传输中被篡改
解决：消息认证码（MAC）

HMAC-SHA256：
  MAC = HMAC(密钥, 数据)

发送方：
  发送：数据 + MAC

接收方：
  1. 重新计算MAC
  2. 对比接收的MAC
  3. 一致 → 数据完整
  4. 不一致 → 数据被篡改
```

**示例攻击与防御**：
```
中间人攻击（MITM）：
  攻击者截获通信，伪造证书

  防御：
  - 浏览器验证证书（CA签名）
  - HSTS（强制HTTPS）
  - Certificate Pinning（固定证书）

重放攻击（Replay）：
  攻击者截获请求，重放

  防御：
  - TLS记录序号（防止重放）
  - Nonce（一次性随机数）
  - 时间戳验证
```

---

#### Q2: 如何防御DDoS攻击？

**答案**：

**分层防御策略**：

**1. 网络层（Layer 3/4）**
```
SYN Flood防御：
  - SYN Cookie（内核级别）
  - SYN Proxy（代理验证）
  - 限制半连接数

UDP Flood防御：
  - 限速（iptables/ipset）
  - 无状态过滤
  - 丢弃非法UDP包

实施：
  sysctl -w net.ipv4.tcp_syncookies=1
  sysctl -w net.ipv4.tcp_max_syn_backlog=8192
  
  iptables -A INPUT -p tcp --syn -m limit --limit 10/s -j ACCEPT
  iptables -A INPUT -p tcp --syn -j DROP
```

**2. 应用层（Layer 7）**
```
HTTP Flood防御：
  - Rate Limiting（限流）
  - JavaScript Challenge
  - CAPTCHA验证
  - 行为分析（机器人检测）

实施：
  # Nginx限流
  limit_req_zone $binary_remote_addr zone=one:10m rate=10r/s;
  limit_req zone=one burst=20;
```

**3. CDN分散流量**
```
原理：
  攻击者 ──大量请求──→ CDN边缘节点（全球分布）
                      ↓ 过滤恶意流量
                    源服务器（只接收少量合法流量）

优势：
  - CDN节点多，分散攻击流量
  - 边缘节点过滤，保护源站
  - 带宽大，吸收攻击

推荐：Cloudflare, Akamai, AWS CloudFront
```

**4. IP黑名单/白名单**
```
实时检测：
  - 识别攻击IP
  - 自动加入黑名单
  - 拒绝后续请求

实施：
  # ipset高效黑名单
  ipset create blacklist hash:ip
  ipset add blacklist 1.2.3.4
  iptables -A INPUT -m set --match-set blacklist src -j DROP
```

**5. 流量清洗**
```
原理：
  攻击流量 → 清洗中心（分析、过滤）→ 合法流量 → 源服务器

服务商：
  - Cloudflare（免费基础DDoS防护）
  - AWS Shield
  - 阿里云DDoS高防
```

**实战案例**：

```
场景：遭受100Gbps HTTP Flood攻击

应对：
1. 立即启用CDN DDoS防护
2. 开启JavaScript Challenge（过滤机器人）
3. 限流：单IP 10 req/s
4. 分析攻击特征：
   - User-Agent: 全是相同的
   - 请求路径: 集中在 /api/expensive
5. 针对性防御：
   - 封禁特定User-Agent
   - /api/expensive 单独限流 1 req/s
6. 通知上游ISP协助过滤

结果：
- 攻击流量从100Gbps降到10Gbps
- 源服务器负载正常
- 合法用户不受影响
```

---

好的！所有内容已经完成，让我加上文档的结尾部分。

```bash
# 添加文档结尾
cat >> /Users/chensiqi/go/src/evm-analyze/docs/45-network-programming-advanced.md << 'ENDOF_DOC'

---

## 总结

本文档涵盖了网络编程的5大进阶主题：

### 已学内容回顾

**1. 零拷贝技术**
- sendfile()：最快，适合文件→Socket
- splice()：灵活，任意fd通过管道传输
- mmap()：支持修改数据，随机访问
- 性能提升：2-3倍（大文件传输）

**2. 高性能网络编程**
- DPDK：绕过内核，30-80 Mpps，极致性能
- io_uring：真正异步IO，减少系统调用100倍
- XDP：驱动层处理，20-30 Mpps，DDoS防御利器
- 选型：epoll → io_uring → XDP → DPDK

**3. 协议实现**
- HTTP/1.1：Keep-Alive持久连接
- HTTP/2：多路复用，头部压缩，性能提升2.5倍
- WebSocket：全双工，实时通信，延迟<1ms
- HTTP/3：QUIC，0-RTT连接，无队头阻塞，弱网优化

**4. 分布式系统**
- RPC：高性能远程调用，gRPC/Thrift
- 服务发现：etcd/Consul，动态扩缩容
- 负载均衡：轮询、加权、一致性哈希
- 高可用：熔断、降级、限流

**5. 网络安全**
- TLS/SSL：加密、身份认证、完整性
- DDoS防御：SYN Cookie、CDN、限流
- 限流算法：令牌桶（推荐）、漏桶、滑动窗口
- 降级策略：保证核心功能，牺牲非核心功能

---

## 面试准备建议

### 必背知识点

**零拷贝**：
- ✓ 传统IO 4次拷贝 vs sendfile 2次拷贝
- ✓ DMA作用、Page Cache原理
- ✓ sendfile/splice/mmap区别和选型

**高性能IO**：
- ✓ DPDK绕过内核原理（PMD轮询）
- ✓ io_uring双环形队列、0系统调用
- ✓ XDP处理点、eBPF限制

**HTTP协议**：
- ✓ HTTP/1.1 vs HTTP/2 vs HTTP/3
- ✓ HTTP/2多路复用、队头阻塞
- ✓ QUIC 0-RTT连接、连接迁移

**分布式**：
- ✓ RPC vs HTTP/REST
- ✓ 服务发现流程（注册、心跳、查询）
- ✓ 一致性哈希原理、虚拟节点
- ✓ 熔断、降级、限流区别

**安全**：
- ✓ TLS握手过程、密钥协商
- ✓ DDoS攻击类型和防御
- ✓ 限流算法对比（令牌桶 vs 漏桶）

### 高频面试题 Top 10

1. **零拷贝为什么快？sendfile、splice、mmap有什么区别？**
   - 答案见 1.7节 Q1-Q2

2. **DPDK为什么比传统网络栈快？适合什么场景？**
   - 答案见 2.5节 Q1

3. **io_uring相比epoll有什么优势？**
   - 答案见 2.5节 Q2

4. **HTTP/2的多路复用是如何实现的？为什么还有队头阻塞？**
   - 答案见 3.5节 Q3

5. **QUIC为什么能实现0-RTT？安全吗？**
   - 答案见 3.5节 Q4

6. **RPC和HTTP有什么区别？什么时候用RPC？**
   - 答案见 4.5节 Q1

7. **什么是服务雪崩？如何防止？**
   - 答案见 4.5节 Q2

8. **一致性哈希的原理？为什么需要虚拟节点？**
   - 答案见 4.5节 Q3

9. **熔断、降级、限流的区别？**
   - 答案见 4.5节 Q4

10. **HTTPS是如何保证安全的？如何防御DDoS？**
    - 答案见 5.5节 Q1-Q2

### 实战项目经验准备

**建议准备1-2个项目经验**，能讲清楚以下内容：

```
项目背景：
  - 业务场景（如高并发API网关、实时推送系统）
  - 遇到的问题（如延迟高、吞吐量低、连接数限制）

技术选型：
  - 为什么选择XX技术（如io_uring、QUIC）
  - 对比了哪些方案
  - 最终选择的原因

实现细节：
  - 架构设计（画图）
  - 核心代码（能手写伪代码）
  - 踩过的坑（限流算法选择、熔断阈值调优等）

效果数据：
  - 优化前后对比（延迟降低X%，吞吐提升X倍）
  - 支持的并发数
  - 资源使用（CPU、内存）
```

**示例**：

```
项目：实时消息推送系统

背景：
  - 100万在线用户
  - 要求延迟<100ms
  - 原有HTTP轮询方案延迟高、服务器负载大

技术选型：
  - 对比：HTTP Long Polling vs WebSocket vs Server-Sent Events
  - 选择：WebSocket（双向通信、延迟低）
  - 框架：基于epoll自研（避免第三方库限制）

实现：
  - 架构：Nginx → WebSocket网关 → Redis Pub/Sub → 业务服务器
  - 连接管理：epoll ET模式，单进程10万连接
  - 心跳：60秒Ping/Pong，检测死连接
  - 限流：令牌桶算法，单用户10 msg/s

优化：
  - 问题1：大量CLOSE_WAIT → 检查代码未close() → 修复
  - 问题2：内存泄漏 → Valgrind检测 → 修复缓冲区未释放
  - 问题3：CPU 100% → perf分析 → 优化JSON序列化

效果：
  - 延迟：从5秒降到50ms（100倍）
  - 吞吐：单机支持10万并发连接
  - 成本：服务器减少70%
```

---

## 推荐学习资源

### 书籍

1. **《Unix网络编程》（UNP）**
   - 作者：W. Richard Stevens
   - 必读经典，socket编程圣经

2. **《TCP/IP详解 卷1：协议》**
   - 深入理解TCP/IP协议栈

3. **《Linux高性能服务器编程》**
   - 作者：游双
   - 涵盖epoll、零拷贝、线程池等

4. **《高性能网络编程》（HTTP/2, QUIC）**
   - 了解现代协议

### 开源项目

1. **Redis**（网络模型）
   - https://github.com/redis/redis
   - 学习：单线程 + epoll + 事件驱动

2. **Nginx**（高性能Web服务器）
   - https://github.com/nginx/nginx
   - 学习：事件驱动、模块化设计

3. **muduo**（C++网络库）
   - https://github.com/chenshuo/muduo
   - 学习：Reactor模式、one loop per thread

4. **liburing**（io_uring库）
   - https://github.com/axboe/liburing
   - 学习：异步IO编程

5. **DPDK**（用户态网络）
   - https://github.com/DPDK/dpdk
   - 学习：极致性能优化

### 在线资源

1. **The C10K Problem**
   - http://www.kegel.com/c10k.html
   - 理解高并发挑战

2. **Cloudflare Blog**
   - https://blog.cloudflare.com
   - 学习：CDN、DDoS防御、QUIC

3. **Nginx Blog**
   - https://www.nginx.com/blog
   - 学习：负载均衡、反向代理

4. **Linux Man Pages**
   - man socket, man epoll, man sendfile
   - 最权威的API文档

### 实战练习

**Level 1：基础练习**
```
1. 手写Echo服务器（epoll ET模式）
2. 实现HTTP/1.1服务器（支持Keep-Alive）
3. 实现WebSocket服务器（完整握手和帧解析）
```

**Level 2：进阶练习**
```
4. 实现零拷贝文件服务器（sendfile）
5. 实现简单RPC框架（JSON协议）
6. 实现令牌桶限流器
```

**Level 3：高级挑战**
```
7. 实现HTTP/2服务器（使用nghttp2库）
8. 实现io_uring Echo服务器
9. 实现XDP包过滤器（DDoS防御）
10. 实现分布式服务注册中心（类etcd）
```

