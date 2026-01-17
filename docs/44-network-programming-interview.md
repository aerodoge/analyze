# 网络编程面试高频问题详解

> 涵盖所有网络编程面试必考知识点，助你面试无障碍

---

## 目录

1. [粘包/拆包问题](#1-粘包拆包问题)
2. [四次挥手与连接关闭](#2-四次挥手与连接关闭)
3. [TIME_WAIT问题与优化](#3-time_wait问题与优化)
4. [心跳机制](#4-心跳机制)
5. [常用Socket选项](#5-常用socket选项)
6. [非阻塞IO和epoll](#6-非阻塞io和epoll)
7. [零拷贝技术](#7-零拷贝技术)
8. [拥塞控制和流量控制](#8-拥塞控制和流量控制)
9. [高并发优化技巧](#9-高并发优化技巧)
10. [面试常见问题汇总](#10-面试常见问题汇总)

---

## 1. 粘包/拆包问题

### 1.1 什么是粘包/拆包？

**核心概念**：TCP是**字节流协议**，没有消息边界的概念。应用层的多个数据包可能被合并或拆分。

```
应用层发送的数据包：
┌────────┐ ┌────────┐ ┌────────┐
│ 包A    │ │ 包B    │ │ 包C    │
│ (100B) │ │ (200B) │ │ (150B) │
└────────┘ └────────┘ └────────┘
     ↓          ↓          ↓
TCP层看到的是连续的字节流：
┌──────────────────────────────────┐
│ AAAAA...BBBBB...CCCCC...         │  ← 没有边界！
└──────────────────────────────────┘
     ↓
接收端可能收到：

情况1 - 正常接收（理想情况）：
┌────────┐ ┌────────┐ ┌────────┐
│ A      │ │ B      │ │ C      │
└────────┘ └────────┘ └────────┘

情况2 - 粘包：
┌────────────────┐ ┌────────┐
│ A + B          │ │ C      │  ← A和B粘在一起
└────────────────┘ └────────┘

情况3 - 拆包：
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ A前半   │ │ A后半  │ │ B      │ │ C      │  ← A被拆成两段
└────────┘ └────────┘ └────────┘ └────────┘

情况4 - 粘包+拆包：
┌──────────────┐ ┌────────────────┐
│ A + B前半    │ │ B后半 + C      │
└──────────────┘ └────────────────┘
```

### 1.2 为什么会发生粘包/拆包？

**粘包的原因**：

```c
// 发送端的Nagle算法
// 为了提高网络利用率，会合并小包

应用程序快速发送多次：
send(fd, "Hello", 5);       // 发送包1
send(fd, "World", 5);       // 发送包2
send(fd, "!", 1);           // 发送包3

TCP层：
┌─────────────────────────────┐
│ 检测到3个小包，合并发送：       │
│ "HelloWorld!" (一次发送)     │  ← 粘包！
└─────────────────────────────┘

原因：
1. Nagle算法（延迟发送，合并小包）
2. TCP缓冲区足够大，一次性发送多个包
3. 接收端没有及时读取，多个包积压在接收缓冲区
```

**拆包的原因**：

```c
// MSS限制导致拆包

应用层发送：
send(fd, large_data, 10000);  // 发送10KB数据

TCP层：
┌────────────────────────────┐
│ MSS = 1460 字节             │
│ 10000 字节需要拆成多个段：    │
│                            │
│ 段1: 1460 字节              │
│ 段2: 1460 字节              │
│ 段3: 1460 字节              │
│ ...                        │
│ 段7: 820 字节               │  ← 拆包！
└────────────────────────────┘

原因：
1. MSS（最大段大小）限制，通常1460字节
2. IP层MTU限制，通常1500字节
3. 接收端缓冲区不足，分多次读取
4. 网络拥塞，TCP滑动窗口变小
```

### 1.3 粘包/拆包的危害

**示例：聊天协议**

```c
// 协议设计：每条消息一行
// 格式：用户名:消息内容\n

客户端发送：
send("Alice:Hello\n");
send("Bob:Hi\n");
send("Alice:How are you?\n");

理想情况下接收：
"Alice:Hello\n"
"Bob:Hi\n"
"Alice:How are you?\n"

但实际可能收到（粘包）：
"Alice:Hello\nBob:Hi\nAlice:How are you?\n"  ← 3条消息粘在一起

或者（拆包）：
"Alice:Hel"
"lo\nBob:Hi\n"
"Alice:How are you?\n"  ← 第一条消息被拆开

如果解析不当：
- "Alice:Hel" 会被当作一条完整消息（错误！）
- 导致数据混乱、协议解析失败
```

### 1.4 解决方案

#### 方案1：固定长度

```c
// 每条消息固定100字节，不足补空格

#define MSG_LEN 100

// 发送端
void send_fixed_msg(int fd, const char *msg) {
    char buf[MSG_LEN];
    memset(buf, ' ', MSG_LEN);  // 填充空格

    int len = strlen(msg);
    if (len > MSG_LEN) len = MSG_LEN;
    memcpy(buf, msg, len);

    send(fd, buf, MSG_LEN);  // 固定发送100字节
}

// 接收端
void recv_fixed_msg(int fd) {
    char buf[MSG_LEN];
    int total = 0;

    // 必须读满100字节
    while (total < MSG_LEN) {
        int n = recv(fd, buf + total, MSG_LEN - total, 0);
        if (n <= 0) break;
        total += n;
    }

    if (total == MSG_LEN) {
        buf[MSG_LEN] = '\0';
        printf("收到消息: %s\n", buf);
    }
}
```

**优点**：
- 实现简单
- 解析容易

**缺点**：
- 浪费带宽（消息短也要发100字节）
- 无法处理超过固定长度的消息
- 不灵活

**适用场景**：
- 消息长度变化不大的场景
- 对带宽不敏感的内网通信

---

#### 方案2：特殊分隔符

```c
// 使用\n或特殊字符作为消息边界

#define DELIMITER '\n'
#define BUF_SIZE 4096

// 发送端
void send_delimited_msg(int fd, const char *msg) 
{
    char buf[BUF_SIZE];
    snprintf(buf, BUF_SIZE, "%s%c", msg, DELIMITER);
    send(fd, buf, strlen(buf));
}

// 接收端（需要维护缓冲区）
typedef struct 
{
    char buf[BUF_SIZE];
    int pos;  // 当前位置
} MsgBuffer;

void init_buffer(MsgBuffer *mb) 
{
    memset(mb->buf, 0, BUF_SIZE);
    mb->pos = 0;
}

int recv_delimited_msg(int fd, MsgBuffer *mb, char *out_msg, int max_len) 
{
    while (1) 
    {
        // 1. 先检查缓冲区中是否已有完整消息
        char *delimiter = memchr(mb->buf, DELIMITER, mb->pos);
        if (delimiter != NULL) 
        {
            // 找到分隔符，提取消息
            int msg_len = delimiter - mb->buf;
            if (msg_len > max_len - 1) msg_len = max_len - 1;

            memcpy(out_msg, mb->buf, msg_len);
            out_msg[msg_len] = '\0';

            // 移除已处理的消息（包括分隔符）
            int remaining = mb->pos - msg_len - 1;
            memmove(mb->buf, delimiter + 1, remaining);
            mb->pos = remaining;

            return msg_len;  // 成功读取一条消息
        }

        // 2. 缓冲区没有完整消息，继续接收
        if (mb->pos >= BUF_SIZE - 1) 
        {
            fprintf(stderr, "缓冲区溢出！\n");
            return -1;
        }

        int n = recv(fd, mb->buf + mb->pos, BUF_SIZE - mb->pos - 1, 0);
        if (n <= 0) {
            return n;  // 连接关闭或错误
        }

        mb->pos += n;
    }
}

// 使用示例
int main() 
{
    int fd = socket(...);
    MsgBuffer mb;
    init_buffer(&mb);

    char msg[1024];
    while (1) 
    {
        int ret = recv_delimited_msg(fd, &mb, msg, sizeof(msg));
        if (ret > 0) 
        {
            printf("收到消息: %s\n", msg);
        } 
        else if (ret == 0) 
        {
            printf("连接关闭\n");
            break;
        } 
        else 
        {
            printf("错误\n");
            break;
        }
    }
}
```

**优点**：
- 实现相对简单
- 消息长度可变
- 带宽利用率高

**缺点**：
- 消息内容不能包含分隔符（需要转义）
- 需要逐字节扫描查找分隔符

**适用场景**：
- 文本协议（HTTP、Redis、SMTP）
- 消息内容可控的场景

---

#### 方案3：长度前缀（最常用）

```c
// 消息格式：[4字节长度][消息内容]

#pragma pack(1)  // 按1字节对齐
typedef struct 
{
    uint32_t length;  // 消息长度（不包括length字段本身）
    char data[0];     // 柔性数组，实际长度由length决定
} Message;
#pragma pack()

// 发送端
int send_length_prefixed_msg(int fd, const char *msg) 
{
    uint32_t msg_len = strlen(msg);
    uint32_t total_len = sizeof(uint32_t) + msg_len;

    char *buf = malloc(total_len);
    if (!buf) return -1;

    // 写入长度（网络字节序）
    uint32_t net_len = htonl(msg_len);
    memcpy(buf, &net_len, sizeof(uint32_t));

    // 写入消息内容
    memcpy(buf + sizeof(uint32_t), msg, msg_len);

    // 发送
    int sent = 0;
    while (sent < total_len) 
    {
        int n = send(fd, buf + sent, total_len - sent, 0);
        if (n <= 0) 
        {
            free(buf);
            return -1;
        }
        sent += n;
    }

    free(buf);
    return msg_len;
}

// 接收端
typedef struct 
{
    char *buf;
    int capacity;
    int pos;
} RecvBuffer;

void init_recv_buffer(RecvBuffer *rb, int capacity) 
{
    rb->buf = malloc(capacity);
    rb->capacity = capacity;
    rb->pos = 0;
}

void free_recv_buffer(RecvBuffer *rb) 
{
    free(rb->buf);
}

int recv_length_prefixed_msg(int fd, RecvBuffer *rb, char **out_msg, uint32_t *out_len) 
{
    // 1. 先读取4字节的长度字段
    while (rb->pos < sizeof(uint32_t)) 
    {
        int n = recv(fd, rb->buf + rb->pos, sizeof(uint32_t) - rb->pos, 0);
        if (n <= 0) return n;
        rb->pos += n;
    }

    // 2. 解析长度
    uint32_t msg_len;
    memcpy(&msg_len, rb->buf, sizeof(uint32_t));
    msg_len = ntohl(msg_len);  // 网络字节序转主机字节序

    // 3. 检查长度合法性
    if (msg_len == 0 || msg_len > 10 * 1024 * 1024) // 最大10MB
    {  
        fprintf(stderr, "非法消息长度: %u\n", msg_len);
        return -1;
    }

    // 4. 扩展缓冲区（如果需要）
    uint32_t total_len = sizeof(uint32_t) + msg_len;
    if (total_len > rb->capacity) 
    {
        rb->buf = realloc(rb->buf, total_len);
        rb->capacity = total_len;
    }

    // 5. 读取消息内容
    while (rb->pos < total_len) 
    {
        int n = recv(fd, rb->buf + rb->pos, total_len - rb->pos, 0);
        if (n <= 0) return n;
        rb->pos += n;
    }

    // 6. 提取消息
    *out_msg = malloc(msg_len + 1);
    memcpy(*out_msg, rb->buf + sizeof(uint32_t), msg_len);
    (*out_msg)[msg_len] = '\0';
    *out_len = msg_len;

    // 7. 重置缓冲区
    rb->pos = 0;

    return msg_len;
}

// 使用示例
int main() 
{
    int fd = socket(...);
    RecvBuffer rb;
    init_recv_buffer(&rb, 4096);

    while (1) 
    {
        char *msg = NULL;
        uint32_t len = 0;

        int ret = recv_length_prefixed_msg(fd, &rb, &msg, &len);
        if (ret > 0) 
        {
            printf("收到消息(%u字节): %s\n", len, msg);
            free(msg);
        } 
        else if (ret == 0) 
        {
            printf("连接关闭\n");
            break;
        } 
        else 
        {
            printf("错误\n");
            break;
        }
    }

    free_recv_buffer(&rb);
}
```

**优点**：
- 消息长度可变
- 消息内容无限制（可以是二进制）
- 解析效率高（不需要逐字节扫描）
- 工业界最常用

**缺点**：
- 实现稍复杂
- 需要维护接收缓冲区

**适用场景**：
- 二进制协议
- 高性能场景
- 工业级应用（RPC框架、消息队列）

---

#### 方案4：应用层协议

```c
// 使用成熟的应用层协议，如HTTP、Protobuf

// HTTP协议示例
GET /api/user HTTP/1.1\r\n
Host: example.com\r\n
Content-Length: 123\r\n    ← 用长度前缀
\r\n
{...消息体...}

// Protobuf示例
message ChatMessage {
    string user = 1;
    string content = 2;
    int64 timestamp = 3;
}

// 传输时：[4字节长度][Protobuf编码的消息]
```

**优点**：
- 成熟稳定
- 跨语言支持
- 有现成的库

**缺点**：
- 引入额外依赖
- 可能有性能开销

**适用场景**：
- 微服务通信
- 跨语言系统
- 需要版本兼容的场景

### 1.5 面试常见问题

**Q1: TCP是字节流，UDP会有粘包问题吗？**

```
答：UDP不会有粘包问题，因为：

1. UDP是数据报协议，有明确的消息边界
2. 每次recvfrom()读取一个完整的数据报
3. 要么完整接收，要么丢失，不会拆分

但UDP有其他问题：
- 丢包：数据报可能丢失
- 乱序：数据报可能乱序到达
- 重复：数据报可能重复

示例：
发送端：
sendto(fd, "A", 1, ...);
sendto(fd, "B", 1, ...);
sendto(fd, "C", 1, ...);

接收端：
recvfrom() → "A"  (完整的一个数据报)
recvfrom() → "B"  (完整的一个数据报)
recvfrom() → "C"  (完整的一个数据报)

不会收到 "ABC" 粘在一起！
```

**Q2: 如何选择粘包解决方案？**

```
┌────────────────┬──────────┬──────────┬──────────┐
│ 方案            │ 性能     │ 灵活性    │ 适用场景   │
├────────────────┼──────────┼──────────┼──────────┤
│ 固定长度        │ ★★★★★     │ ★☆☆☆☆   │ 定长消息   │
├────────────────┼──────────┼──────────┼──────────┤
│ 特殊分隔符       │ ★★★☆☆    │ ★★★☆☆   │ 文本协议   │
├────────────────┼──────────┼──────────┼──────────┤
│ 长度前缀        │ ★★★★★    │ ★★★★★    │ 二进制协议 │
├────────────────┼──────────┼──────────┼──────────┤
│ 应用层协议       │ ★★★☆☆    │ ★★★★★    │ 跨语言    │
└────────────────┴──────────┴──────────┴──────────┘

推荐：长度前缀（工业界标准）
```

**Q3: Nagle算法是什么？如何禁用？**

```c
// Nagle算法：延迟发送小包，等待ACK或累积更多数据

问题场景：
send(fd, "H", 1);  // 只发1字节
send(fd, "e", 1);  // 又发1字节
send(fd, "l", 1);  // 又发1字节
...

Nagle算法会等待，直到：
1. 收到前一个包的ACK
2. 或累积到MSS大小
3. 然后一次性发送

导致：延迟增加（对交互式应用不利）

禁用Nagle算法：
int flag = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));

适用场景：
- 实时游戏（低延迟优先）
- 交互式Shell（SSH）
- 股票行情（实时性要求高）

不适用场景：
- 文件传输（吞吐量优先）
- 批量数据（可以累积）
```

---

## 2. 四次挥手与连接关闭

### 2.1 为什么需要四次挥手？

**核心原因**：TCP是**全双工**通信，连接的两端都需要独立关闭。

```
全双工通信示意：
客户端                          服务器
  │ ─────→ 发送数据 ─────→        │
  │                              │
  │ ←───── 发送数据 ←─────        │

两个独立的数据流：
1. 客户端 → 服务器
2. 服务器 → 客户端

关闭连接需要分别关闭这两个方向：
1. 客户端关闭发送方向（不再发数据）
2. 服务器确认
3. 服务器关闭发送方向（不再发数据）
4. 客户端确认
```

### 2.2 四次挥手详细流程

```
客户端                                    服务器
(ESTABLISHED)                        (ESTABLISHED)
   │                                        │
   │ close(fd) 主动关闭                      │
   │                                        │
   │  [第1次挥手] FIN                        │
   │  ─────────────────────────────────────→│
   │  TCP标志: FIN=1, ACK=1                  │
   │  seq = u (比如5000)                     │
   │  ack = v (比如3000)                     │
   │                                        │
(FIN_WAIT_1)                          (CLOSE_WAIT)
   │                                        │
   │            [第2次挥手] ACK              │
   │  ←─────────────────────────────────────│
   │  TCP标志: FIN=0, ACK=1                  │
   │  seq = v (3000)                        │
   │  ack = u+1 (5001，确认客户端FIN)         │
   │                                        │
(FIN_WAIT_2)                                │
   │                                   【服务器继续发送】
   │            ← 数据包 ←                   │
   │            ← 数据包 ←                   │
   │                                        │
   │                                   close(fd)
   │                                        │
   │            [第3次挥手] FIN              │
   │  ←─────────────────────────────────────│
   │  TCP标志: FIN=1, ACK=1                  │
   │  seq = w (比如3500，发送完所有数据后）     │
   │  ack = u+1 (5001)                      │
   │                                        │
(TIME_WAIT)                           (LAST_ACK)
   │                                        │
   │  [第4次挥手] ACK                        │
   │  ─────────────────────────────────────→│
   │  TCP标志: FIN=0, ACK=1                  │
   │  seq = u+1 (5001)                      │
   │  ack = w+1 (3501，确认服务器FIN)         │
   │                                        │
   │ ← 等待2MSL (约60秒) ←              (CLOSED)
   │                                        │
(CLOSED)
```

### 2.3 每次挥手详解

#### 第1次挥手：客户端发送FIN

```
客户端 → 服务器
┌─────────────────────────────────────────┐
│ TCP 报文头部：                            │
├─────────────────────────────────────────┤
│ 标志位：                                 │
│   FIN = 1    ← 请求关闭连接               │
│   ACK = 1    ← 同时确认数据               │
├─────────────────────────────────────────┤
│ 序列号 (seq):                            │
│   seq = 5000                            │
│   ↑ 当前发送序列号                        │
├─────────────────────────────────────────┤
│ 确认号 (ack):                            │
│   ack = 3000                            │
│   ↑ 确认接收到的数据                       │
└─────────────────────────────────────────┘

内核操作：
close(fd)  或  shutdown(fd, SHUT_WR)
    ↓
tcp_close()
    ↓
发送FIN包
    ↓
状态: ESTABLISHED → FIN_WAIT_1

含义：
"我（客户端）没有数据要发送了，
 但我还可以接收数据"
```

**关键点**：
- **FIN占用1个序列号**（和SYN一样）
- 客户端关闭**发送方向**，但**接收方向仍然打开**
- 此时客户端不能再send()，但可以recv()

#### 第2次挥手：服务器确认FIN

```
服务器 → 客户端
┌─────────────────────────────────────────┐
│ TCP 报文头部：                           │
├─────────────────────────────────────────┤
│ 标志位：                                 │
│   FIN = 0    ← 还不关闭                  │
│   ACK = 1    ← 确认客户端FIN             │
├─────────────────────────────────────────┤
│ 序列号 (seq):                            │
│   seq = 3000                            │
├─────────────────────────────────────────┤
│ 确认号 (ack):                            │
│   ack = 5001                            │
│   ↑ 计算: 客户端seq + 1 = 5000 + 1        │
│   含义: "我收到了你的FIN(5000)"            │
└─────────────────────────────────────────┘

内核操作：
收到FIN
    ↓
发送ACK
    ↓
状态: ESTABLISHED → CLOSE_WAIT

含义：
"我（服务器）知道你不再发送数据了，
 但我可能还有数据要发给你"
```

**关键状态**：
- 客户端：`FIN_WAIT_1` → `FIN_WAIT_2`
- 服务器：`ESTABLISHED` → `CLOSE_WAIT`

**此时的连接状态**：
```
客户端 → 服务器：关闭 ✗（不能发送）
服务器 → 客户端：打开 ✓（仍可发送）

这就是"半关闭"状态！
```

#### 第3次挥手：服务器发送FIN

```
服务器 → 客户端
┌─────────────────────────────────────────┐
│ TCP 报文头部：                           │
├─────────────────────────────────────────┤
│ 标志位：                                 │
│   FIN = 1    ← 服务器也关闭了             │
│   ACK = 1                               │
├─────────────────────────────────────────┤
│ 序列号 (seq):                            │
│   seq = 3500                            │
│   ↑ 可能已经发送了额外的数据(3000→3500)     │
├─────────────────────────────────────────┤
│ 确认号 (ack):                            │
│   ack = 5001                            │
│   ↑ 仍然确认客户端的FIN                    │
└─────────────────────────────────────────┘

内核操作：
应用程序调用: close(fd)
    ↓
tcp_close()
    ↓
发送FIN包
    ↓
状态: CLOSE_WAIT → LAST_ACK

含义：
"我（服务器）也没有数据要发送了"
```

**时间差**：
```
第2次挥手和第3次挥手之间可能有时间间隔：

t1: 收到客户端FIN → CLOSE_WAIT
  ↓
  ├─ 继续发送未完成的数据
  ├─ 处理当前请求
  ├─ 清理资源
  ↓ (可能几秒钟)
t2: 调用close() → 发送FIN → LAST_ACK

这就是为什么需要4次而不能合并成3次！
```

#### 第4次挥手：客户端确认FIN

```
客户端 → 服务器
┌─────────────────────────────────────────┐
│ TCP 报文头部：                            │
├─────────────────────────────────────────┤
│ 标志位：                                 │
│   FIN = 0                               │
│   ACK = 1    ← 确认服务器FIN              │
├─────────────────────────────────────────┤
│ 序列号 (seq):                            │
│   seq = 5001                            │
├─────────────────────────────────────────┤
│ 确认号 (ack):                            │
│   ack = 3501                            │
│   ↑ 计算: 服务器seq + 1 = 3500 + 1        │
└─────────────────────────────────────────┘

内核操作：
收到服务器FIN
    ↓
发送ACK
    ↓
状态: FIN_WAIT_2 → TIME_WAIT
    ↓
等待 2MSL (2 * 60秒 = 120秒)
    ↓
状态: TIME_WAIT → CLOSED
```

### 2.4 关键状态详解

#### TIME_WAIT状态

```
为什么需要TIME_WAIT？

问题场景：如果第4次挥手的ACK丢失了...

客户端                          服务器
  │                               │
  │ ← FIN (第3次挥手) ←            │
  │                               │
  │ → ACK (第4次挥手) → ✗ 丢失！    │
  │                               │
  │                               │ ← 等待ACK超时
  │                               │
  │ ← FIN (重传) ←                 │ ← 重传FIN
  │                               │

如果客户端立即CLOSED：
  - 重传的FIN到达，但连接已关闭
  - 客户端会回复RST（连接不存在）
  - 服务器认为连接异常中断 ✗

有TIME_WAIT的话：
  - 客户端还在等待，可以接收重传的FIN
  - 重新发送ACK
  - 服务器正常关闭 ✓
```

**TIME_WAIT的作用**：

1. **确保最后的ACK能到达**
   ```
   如果ACK丢失，服务器会重传FIN
   TIME_WAIT期间可以重新回复ACK
   ```

2. **避免旧连接的数据干扰新连接**
   ```
   旧连接的延迟数据包可能在新连接建立后才到达
   TIME_WAIT确保这些旧包全部过期（2MSL）

   MSL (Maximum Segment Lifetime):
   - IP包在网络中的最大生存时间
   - Linux默认60秒
   - 2MSL = 120秒
   ```

**TIME_WAIT时间计算**：
```
2MSL = 2 * 60秒 = 120秒

为什么是2倍MSL？
- 发送最后的ACK：最多MSL到达服务器
- 服务器重传FIN：最多MSL返回客户端
- 总共最多2*MSL

在这段时间内：
- 旧连接的所有数据包都会过期
- 新连接不会收到旧包
```

#### CLOSE_WAIT状态

```
CLOSE_WAIT表示：
"对方已经close()了，但我还没有close()"

常见问题：CLOSE_WAIT过多

原因：应用程序没有调用close()
┌─────────────────────────────────────┐
│ 客户端发送FIN → 服务器进入CLOSE_WAIT   │
│       ↓                             │
│ 但服务器的应用程序忘记了：              │
│   - 没有调用close(fd)                │
│   - 或者代码逻辑有bug                 │
│   - 或者线程卡住了                    │
│       ↓                             │
│ 连接一直处于CLOSE_WAIT                │
│       ↓                             │
│ 大量CLOSE_WAIT → 文件描述符泄漏        │
└─────────────────────────────────────┘

排查方法：
# 查看CLOSE_WAIT数量
netstat -an | grep CLOSE_WAIT | wc -l

# 找到占用的进程
lsof -i | grep CLOSE_WAIT

# 检查应用程序代码
- 是否每个accept()的连接都有对应的close()？
- 是否异常分支忘记close()？
- 是否有资源泄漏？

修复：
void handle_client(int client_fd) 
{
    // ... 处理逻辑 ...

    // 确保所有路径都close
    close(client_fd);  // ← 必须调用！
}

或使用RAII（C++）：
{
    SocketGuard guard(client_fd);  // 析构时自动close
    // ... 处理逻辑 ...
}  // ← 自动close
```

### 2.5 close() vs shutdown()

```c
// close(): 关闭连接的两个方向

int client_fd = accept(...);
// ... 通信 ...
close(client_fd);

行为：
1. 关闭发送和接收两个方向
2. 发送FIN包
3. 释放文件描述符
4. 如果有引用计数，减1（fork的情况）

// shutdown(): 只关闭一个方向（半关闭）

shutdown(fd, SHUT_RD);   // 关闭读（接收）方向
shutdown(fd, SHUT_WR);   // 关闭写（发送）方向，发送FIN
shutdown(fd, SHUT_RDWR); // 关闭两个方向，等同close但不释放fd

半关闭的应用场景：
┌──────────────────────────────────────────┐
│ HTTP客户端：                              │
│ 1. 发送完请求后shutdown(SHUT_WR)           │
│ 2. 告诉服务器："我发完了"                   │
│ 3. 但仍可以recv()接收响应                   │
│ 4. 接收完毕后close()                       │
└──────────────────────────────────────────┘

示例代码：
// HTTP客户端
send(fd, "GET /index.html HTTP/1.0\r\n\r\n", ...);
shutdown(fd, SHUT_WR);  // 发送FIN，但仍可接收

char buf[4096];
while (recv(fd, buf, sizeof(buf), 0) > 0) 
{
    // 接收响应...
}
close(fd);  // 最后关闭
```

**close() vs shutdown() 对比**：

```
┌──────────────┬──────────────┬──────────────┐
│ 特性          │ close()      │ shutdown()   │
├──────────────┼──────────────┼──────────────┤
│ 关闭方向      │ 双向          │ 可单向        │
├──────────────┼──────────────┼──────────────┤
│ 释放fd        │ 是           │ 否           │
├──────────────┼──────────────┼──────────────┤
│ 发送FIN       │ 是           │ SHUT_WR时    │
├──────────────┼──────────────┼──────────────┤
│ 引用计数      │ 影响          │ 不影响        │
├──────────────┼──────────────┼──────────────┤
│ fork后影响    │ 需要所有进程   │ 立即生效      │
│              │ 都close      │              │
└──────────────┴──────────────┴──────────────┘

fork()场景的区别：
父进程fork()子进程后，fd引用计数=2

// 父进程
int fd = accept(...);
int pid = fork();

if (pid == 0) 
{
    // 子进程处理
    // ...
    close(fd);  // 引用计数减1，变成1，不会发送FIN
} 
else 
{
    // 父进程
    close(fd);  // 引用计数减1，变成0，发送FIN
}

// 使用shutdown
if (pid == 0) 
{
    shutdown(fd, SHUT_RDWR);  // 立即发送FIN，不管引用计数
    close(fd);
}
```

### 2.6 异常关闭：RST

```
RST (Reset) 表示异常终止连接

何时发送RST？

1. 连接不存在
   客户端 → SYN → 服务器(8888端口未监听)
   服务器 → RST → 客户端  (Connection refused)

2. 程序崩溃
   客户端正在通信 → 程序crash → 内核清理
   内核 → RST → 服务器  (Connection reset by peer)

3. SO_LINGER选项
   struct linger l = {1, 0};  // 立即RST
   setsockopt(fd, SOL_SOCKET, SO_LINGER, &l, sizeof(l));
   close(fd);  // 发送RST而不是FIN

4. 接收缓冲区还有数据时close
   客户端发送数据 → 服务器未读取 → close()
   服务器 → RST → 客户端

FIN vs RST的区别：
┌──────────┬────────────────┬─────────────────┐
│          │ FIN (正常关闭)  │ RST (异常终止)    │
├──────────┼────────────────┼─────────────────┤
│ 优雅程度  │ 优雅            │ 粗暴             │
├──────────┼────────────────┼─────────────────┤
│ 数据丢失  │ 保证发完         │ 可能丢失         │
├──────────┼────────────────┼─────────────────┤
│ 缓冲区    │ 清空后关闭       │ 立即丢弃         │
├──────────┼────────────────┼─────────────────┤
│ 对方感知  │ read()返回0     │ read()返回-1     │
│          │                │ errno=ECONNRESET│
└──────────┴────────────────┴─────────────────┘
```

### 2.7 四次挥手演进时间表

```
时刻│ 方向 │ FIN │ ACK │  seq  │  ack  │ 客户端状态   │ 服务器状态
────┼──────┼─────┼─────┼───────┼───────┼─────────────┼──────────────
t0  │      │     │     │       │       │ ESTABLISHED  │ ESTABLISHED
────┼──────┼─────┼─────┼───────┼───────┼─────────────┼──────────────
t1  │ C→S  │  1  │  1  │ 5000  │ 3000  │ FIN_WAIT_1   │ CLOSE_WAIT
    │ FIN  │     │     │       │       │              │
────┼──────┼─────┼─────┼───────┼───────┼─────────────┼──────────────
t2  │ S→C  │  0  │  1  │ 3000  │ 5001  │ FIN_WAIT_2   │ CLOSE_WAIT
    │ ACK  │     │     │       │       │              │ (可能发送数据)
────┼──────┼─────┼─────┼───────┼───────┼─────────────┼──────────────
t3  │ S→C  │  1  │  1  │ 3500  │ 5001  │ TIME_WAIT    │ LAST_ACK
    │ FIN  │     │     │       │       │              │
────┼──────┼─────┼─────┼───────┼───────┼─────────────┼──────────────
t4  │ C→S  │  0  │  1  │ 5001  │ 3501  │ TIME_WAIT    │ CLOSED
    │ ACK  │     │     │       │       │(等待2MSL)    │
────┼──────┼─────┼─────┼───────┼───────┼─────────────┼──────────────
t5  │      │     │     │       │       │ CLOSED       │
    │      │     │     │       │       │(2MSL后)      │
────┴──────┴─────┴─────┴───────┴───────┴─────────────┴──────────────
```

### 2.8 面试常见问题

**Q1: 为什么连接建立是3次，关闭是4次？**

```
答：

建立连接（3次握手）：
┌────────────────────────────────────┐
│ 第1次: C → S: SYN                   │
│ 第2次: S → C: SYN + ACK             │ ← 合并！
│ 第3次: C → S: ACK                   │
└────────────────────────────────────┘

服务器的SYN和ACK可以合并在一个包里发送，
因为此时服务器可以立即响应。

关闭连接（4次挥手）：
┌────────────────────────────────────┐
│ 第1次: C → S: FIN                   │
│ 第2次: S → C: ACK                   │ ← 不能合并
│ 第3次: S → C: FIN                   │ ← 有时间差
│ 第4次: C → S: ACK                   │
└────────────────────────────────────┘

服务器的ACK和FIN不能合并，因为：
1. 收到客户端FIN后，需要立即ACK
2. 但服务器可能还有数据要发送
3. 需要等应用程序处理完、调用close()后才发FIN
4. 中间可能有几秒钟的间隔

特例：如果服务器没有数据要发送，可以合并成3次：
第1次: C → S: FIN
第2次: S → C: FIN + ACK  ← 合并
第3次: C → S: ACK

但这只是优化，标准流程仍是4次。
```

**Q2: TIME_WAIT过多怎么办？**

```
原因：
主动关闭方会进入TIME_WAIT状态（2MSL = 120秒）

场景1：客户端频繁短连接
客户端 connect() → 通信 → close() → TIME_WAIT (120秒)
短时间内创建大量连接 → 大量TIME_WAIT

场景2：服务器主动关闭客户端连接
服务器 close(client_fd) → TIME_WAIT (120秒)
高并发下 → 服务器大量TIME_WAIT

危害：
- 占用端口（65535个端口用完 → Cannot assign requested address）
- 占用内存（每个TIME_WAIT约3.3KB）
- 影响性能

解决方案：

1. 应用层：使用长连接
   HTTP/1.1 Keep-Alive
   连接复用，减少频繁创建/关闭

2. SO_REUSEADDR（推荐）
   int reuse = 1;
   setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR,
              &reuse, sizeof(reuse));

   作用：允许bind到TIME_WAIT状态的端口
   适用：服务器重启时立即绑定端口

3. net.ipv4.tcp_tw_reuse（推荐）
   sysctl -w net.ipv4.tcp_tw_reuse=1

   作用：允许TIME_WAIT socket用于新连接（客户端）
   条件：timestamp选项开启
   安全：只影响客户端，服务器不受影响

4. net.ipv4.tcp_tw_recycle（不推荐，已废弃）
   sysctl -w net.ipv4.tcp_tw_recycle=1

   作用：快速回收TIME_WAIT socket
   问题：NAT环境下会导致连接失败
   状态：Linux 4.12+ 已移除

5. 调整TIME_WAIT时间（不推荐）
   修改内核常量TCP_TIMEWAIT_LEN
   风险：违反TCP RFC，可能导致数据混乱

推荐方案：
- 服务器：SO_REUSEADDR
- 客户端：tcp_tw_reuse + 长连接
- 架构：让客户端主动关闭（服务器被动关闭不会TIME_WAIT）
```

**Q3: 如何判断连接是正常关闭还是异常断开？**

```c
// 读取返回值判断

ssize_t n = recv(fd, buf, sizeof(buf), 0);

if (n > 0) 
{
    // 正常接收到数据
    printf("收到%ld字节\n", n);
} 
else if (n == 0) 
{
    // 对方正常关闭（发送了FIN）
    printf("连接正常关闭\n");
    close(fd);
} 
else 
{   
    // n < 0
    // 发生错误，检查errno
    if (errno == EAGAIN || errno == EWOULDBLOCK) 
    {
        // 非阻塞IO，暂时没数据
        // 继续等待...
    } 
    else if (errno == EINTR) 
    {
        // 被信号中断，重试
        // 继续recv...
    } 
    else if (errno == ECONNRESET) 
    {
        // 对方发送RST（异常断开）
        printf("连接被对方重置\n");
        close(fd);
    } 
    else if (errno == ETIMEDOUT)
    {
        // 连接超时
        printf("连接超时\n");
        close(fd);
    } 
    else 
    {
        // 其他错误
        perror("recv");
        close(fd);
    }
}

// 写入时检测断开

ssize_t n = send(fd, buf, len, 0);

if (n < 0) 
{
    if (errno == EPIPE) 
    {
        // 对方已关闭，但本端还在写（收到RST）
        printf("对方已关闭连接\n");
        // 会收到SIGPIPE信号（如果没忽略）
    } 
    else if (errno == ECONNRESET) 
    {
        // 连接被重置
        printf("连接被重置\n");
    }
}

// 忽略SIGPIPE信号（避免程序崩溃）
signal(SIGPIPE, SIG_IGN);
```

---

## 3. TIME_WAIT问题与优化

### 3.1 TIME_WAIT状态详解

（此部分已在第2章详细讲解，这里补充更多实战内容）

### 3.2 查看和监控TIME_WAIT

```bash
# 1. 查看当前TIME_WAIT数量
netstat -an | grep TIME_WAIT | wc -l

# 或使用ss（更快）
ss -tan | grep TIME-WAIT | wc -l

# 2. 查看所有连接状态统计
netstat -an | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'

# 输出示例：
# ESTABLISHED 523
# TIME_WAIT 8234      ← 重点关注
# FIN_WAIT_2 12
# CLOSE_WAIT 5        ← 如果过多表示应用bug

# 3. 持续监控
watch -n 1 'ss -s'

# 输出：
# Total: 9000 (kernel 9050)
# TCP:   8500 (estab 500, closed 8000, orphaned 0, synrecv 0, timewait 8000)

# 4. 查看具体的TIME_WAIT连接
ss -tan state time-wait | head -20

# 5. 按本地端口分组统计
ss -tan state time-wait | awk '{print $4}' | cut -d: -f2 | sort | uniq -c | sort -rn | head

# 输出：
# 1234 8080    ← 8080端口有1234个TIME_WAIT
# 567 9090
# ...
```

### 3.3 内核参数优化

```bash
# 查看当前配置
sysctl -a | grep tcp.*time

# 重要参数：

# 1. tcp_tw_reuse（推荐开启）
sysctl -w net.ipv4.tcp_tw_reuse=1

# 作用：允许TIME_WAIT socket用于新的TCP连接
# 适用：客户端（频繁发起连接的场景）
# 条件：需要开启tcp_timestamps
# 安全：是，基于timestamp判断，不会混淆旧数据

# 2. tcp_timestamps（推荐开启）
sysctl -w net.ipv4.tcp_timestamps=1

# 作用：TCP时间戳选项，用于RTT测量和PAWS
# 副作用：每个包增加12字节开销
# 必要性：tcp_tw_reuse依赖此选项

# 3. tcp_max_tw_buckets（限制数量）
sysctl -w net.ipv4.tcp_max_tw_buckets=50000

# 作用：系统最多保留多少TIME_WAIT socket
# 默认值：通常是内存依赖的，几万到几十万
# 超出后：新的TIME_WAIT直接销毁（警告：可能导致旧连接数据混入）

# 4. tcp_fin_timeout（慎用）
sysctl -w net.ipv4.tcp_fin_timeout=30

# 作用：FIN_WAIT_2状态超时时间
# 默认：60秒
# 注意：不影响TIME_WAIT（TIME_WAIT固定2MSL=120秒）

# 5. ip_local_port_range（扩大端口范围）
sysctl -w net.ipv4.ip_local_port_range="10000 65000"

# 作用：客户端可用端口范围
# 默认：32768-60999
# 调整：10000-65000（增加5.5万个可用端口）

# 永久保存配置
cat >> /etc/sysctl.conf << EOF
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_max_tw_buckets = 50000
net.ipv4.ip_local_port_range = 10000 65000
EOF

sysctl -p  # 生效
```

### 3.4 应用层优化

```c
// 1. SO_REUSEADDR（服务器必备）

int listen_fd = socket(AF_INET, SOCK_STREAM, 0);

int reuse = 1;
if (setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse)) < 0) 
{
    perror("setsockopt SO_REUSEADDR");
}

// 作用：
// - 允许bind到TIME_WAIT状态的端口
// - 允许同一端口启动多个监听socket（配合SO_REUSEPORT）
// - 服务器重启时不会报"Address already in use"

struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(8888);
addr.sin_addr.s_addr = INADDR_ANY;

if (bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr)) < 0) 
{
    perror("bind");  // 如果没有SO_REUSEADDR，这里可能失败
}

// 2. SO_LINGER（谨慎使用）

struct linger l;

// 选项1：优雅关闭（默认）
l.l_onoff = 0;
l.l_linger = 0;
setsockopt(fd, SOL_SOCKET, SO_LINGER, &l, sizeof(l));
// 行为：close()发送FIN，进入TIME_WAIT

// 选项2：强制关闭（发送RST）
l.l_onoff = 1;
l.l_linger = 0;  // ← 关键：超时时间为0
setsockopt(fd, SOL_SOCKET, SO_LINGER, &l, sizeof(l));
close(fd);  // 发送RST，不进入TIME_WAIT，直接CLOSED
// 优点：立即关闭，不占用TIME_WAIT
// 缺点：
//   - 发送缓冲区的数据可能丢失
//   - 对方收到RST而不是FIN
//   - 不符合TCP优雅关闭

// 选项3：等待关闭
l.l_onoff = 1;
l.l_linger = 5;  // 最多等待5秒
setsockopt(fd, SOL_SOCKET, SO_LINGER, &l, sizeof(l));
close(fd);
// 行为：
//   - close()阻塞，等待数据发送完毕
//   - 或者等待5秒超时
//   - 超时后发送RST

// 推荐：大部分情况用默认（l_onoff=0）

// 3. 连接池（最佳实践）

typedef struct {
    int fd;
    time_t last_use;
    int in_use;
} PooledConnection;

PooledConnection pool[100];

// 获取连接（复用）
int get_connection(const char *host, int port) 
{
    // 1. 先找空闲连接
    for (int i = 0; i < 100; i++) 
    {
        if (pool[i].fd > 0 && !pool[i].in_use) 
        {
            pool[i].in_use = 1;
            pool[i].last_use = time(NULL);
            return pool[i].fd;  // 复用！
        }
    }

    // 2. 没有空闲连接，创建新的
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    connect(fd, ...);

    // 加入连接池
    for (int i = 0; i < 100; i++) 
    {
        if (pool[i].fd == 0) 
        {
            pool[i].fd = fd;
            pool[i].in_use = 1;
            pool[i].last_use = time(NULL);
            return fd;
        }
    }

    return fd;
}

// 归还连接（不关闭）
void release_connection(int fd) 
{
    for (int i = 0; i < 100; i++) 
    {
        if (pool[i].fd == fd) 
        {
            pool[i].in_use = 0;  // 标记为空闲，不close！
            pool[i].last_use = time(NULL);
            return;
        }
    }
}

// 清理过期连接
void cleanup_pool() 
{
    time_t now = time(NULL);
    for (int i = 0; i < 100; i++) 
    {
        if (pool[i].fd > 0 && !pool[i].in_use && now - pool[i].last_use > 300) // 5分钟未使用
        {  
            close(pool[i].fd);
            pool[i].fd = 0;
        }
    }
}

// 好处：
// - 连接复用，大幅减少TIME_WAIT
// - 减少三次握手开销
// - 提高性能
```

### 3.5 架构优化

```
策略1：让客户端主动关闭

传统方式（服务器主动关闭）：
客户端 → 请求 → 服务器
客户端 ← 响应 ← 服务器
                服务器: close() → TIME_WAIT ✗

改进方式（客户端主动关闭）：
客户端 → 请求 → 服务器
客户端 ← 响应 ← 服务器
客户端: close() → TIME_WAIT ✓（客户端有的是端口）

实现：
// 服务器告诉客户端"我发完了"
send(client_fd, response, len, 0);
shutdown(client_fd, SHUT_WR);  // 半关闭，发送FIN

// 等待客户端关闭（客户端收到FIN后主动close）
char buf[1];
recv(client_fd, buf, 1, 0);  // 读到EOF(返回0)

close(client_fd);  // 服务器被动关闭，不会TIME_WAIT ✓

策略2：使用HTTP/1.1 Keep-Alive

HTTP/1.0（短连接）：
客户端 → 请求1 → 服务器
       ← 响应1 ←
       close() → TIME_WAIT ✗

客户端 → 请求2 → 服务器（新连接）
       ← 响应2 ←
       close() → TIME_WAIT ✗

HTTP/1.1（长连接）：
客户端 → 请求1 → 服务器
       ← 响应1 ←
       → 请求2 →（复用连接）
       ← 响应2 ←
       → 请求3 →（复用连接）
       ← 响应3 ←
       close() → TIME_WAIT ✓（只有一次）

策略3：负载均衡

客户端 → 负载均衡器 → 后端服务器池
         (TIME_WAIT)

- 客户端到负载均衡器：长连接
- 负载均衡器到后端：连接池
- TIME_WAIT集中在负载均衡器
- 后端服务器干净
```

### 3.6 监控和告警

```python
#!/usr/bin/env python3
# 监控TIME_WAIT数量，超过阈值告警

import subprocess
import time

THRESHOLD = 10000  # TIME_WAIT超过1万告警

def get_time_wait_count():
    result = subprocess.run(['ss', '-tan', 'state', 'time-wait'],
                            capture_output=True, text=True)
    lines = result.stdout.strip().split('\n')
    return len(lines) - 1  # 减去标题行

def alert(count):
    print(f"[ALERT] TIME_WAIT过多: {count}")
    # 发送告警通知...
    # send_email() or send_slack() or ...

while True:
    count = get_time_wait_count()
    print(f"TIME_WAIT: {count}")

    if count > THRESHOLD:
        alert(count)

    time.sleep(60)  # 每分钟检查一次
```

---

## 4. 心跳机制

### 4.1 为什么需要心跳？

**核心问题**：如何检测连接是否还活着？

```
场景1：对方进程崩溃
客户端 → 正常运行
服务器 → 程序崩溃（Segfault）→ 内核发送RST
客户端 → 能检测到（recv返回错误）✓

场景2：对方机器断电（拔网线）
客户端 → 正常运行
服务器 → 突然断电（没发FIN/RST）
客户端 → 不知道服务器已经死了 ✗

此时：
- TCP连接仍然是ESTABLISHED状态
- send()可能成功（数据在本地缓冲区）
- 但数据永远发不到对方
- 资源一直占用

解决方案：心跳机制
定期发送心跳包，检测对方是否还活着
```

### 4.2 TCP Keepalive（传输层）

**内核提供的TCP保活机制**

```c
// 开启TCP Keepalive

int fd = socket(AF_INET, SOCK_STREAM, 0);

int keepalive = 1;  // 开启keepalive
setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &keepalive, sizeof(keepalive));

// 配置keepalive参数（Linux）

// 1. 空闲多久开始发送keepalive探测（秒）
int keepidle = 60;  // 60秒空闲后开始探测
setsockopt(fd, IPPROTO_TCP, TCP_KEEPIDLE, &keepidle, sizeof(keepidle));

// 2. 探测间隔（秒）
int keepintvl = 10;  // 每隔10秒发送一次
setsockopt(fd, IPPROTO_TCP, TCP_KEEPINTVL, &keepintvl, sizeof(keepintvl));

// 3. 探测次数
int keepcnt = 3;  // 最多探测3次
setsockopt(fd, IPPROTO_TCP, TCP_KEEPCNT, &keepcnt, sizeof(keepcnt));

// 工作流程：
// 1. 连接空闲60秒 → 发送第1个keepalive探测
// 2. 10秒后没收到回复 → 发送第2个探测
// 3. 再10秒后没收到回复 → 发送第3个探测
// 4. 再10秒后还没回复 → 认为连接死了，关闭连接
// 总计：60 + 10 + 10 + 10 = 90秒
```

**TCP Keepalive的工作原理**：

```
正常情况（对方存活）：
客户端                          服务器
  │ ← 60秒空闲 ←                  │
  │                              │
  │ → Keepalive探测 →             │
  │ (seq=当前-1, 0字节数据)        │
  │                              │
  │ ← ACK ←                      │ ← 服务器正常，回复ACK
  │                              │
  │ ✓ 连接OK，继续等待             │

对方死亡（网络断开）：
客户端                          服务器
  │ ← 60秒空闲 ←                  │
  │                              │
  │ → Keepalive探测 → ✗ (丢失)    │ (服务器已断开)
  │ ← 等待10秒 ←                  │
  │                              │
  │ → Keepalive探测 → ✗ (丢失)    │
  │ ← 等待10秒 ←                  │
  │                              │
  │ → Keepalive探测 → ✗ (丢失)    │
  │ ← 等待10秒 ←                  │
  │                              │
  │ ✗ 连接死了，close()            │
  │   errno = ETIMEDOUT          │
```

**系统默认配置**：

```bash
# 查看系统默认配置
sysctl net.ipv4.tcp_keepalive_time    # 默认7200秒(2小时)
sysctl net.ipv4.tcp_keepalive_intvl   # 默认75秒
sysctl net.ipv4.tcp_keepalive_probes  # 默认9次

# 修改系统默认值
sysctl -w net.ipv4.tcp_keepalive_time=600
sysctl -w net.ipv4.tcp_keepalive_intvl=10
sysctl -w net.ipv4.tcp_keepalive_probes=3
```

**TCP Keepalive的优缺点**：

```
优点：
✓ 内核实现，应用层无感知
✓ 开销小（只有探测时才发包）
✓ 标准协议，跨平台

缺点：
✗ 默认2小时才开始探测（太长）
✗ 不灵活（只能检测连接存活，不能携带业务数据）
✗ 可能被中间设备（NAT、防火墙）干扰
✗ 只能检测网络层死亡，检测不了应用层死锁

适用场景：
- 长连接（数据库连接池）
- 辅助手段（和应用层心跳配合）
- 兜底保护（防止僵尸连接）
```

### 4.3 应用层心跳（推荐）

**自己实现心跳协议**

```c
// 协议设计

#define MSG_TYPE_HEARTBEAT_REQ  0x01  // 心跳请求
#define MSG_TYPE_HEARTBEAT_RESP 0x02  // 心跳响应
#define MSG_TYPE_DATA           0x03  // 普通数据

#pragma pack(1)
typedef struct 
{
    uint32_t length;      // 消息长度
    uint8_t  type;        // 消息类型
    uint8_t  data[0];     // 消息内容
} Message;
#pragma pack()

// 服务器端：心跳检测
typedef struct 
{
    int fd;
    time_t last_heartbeat;  // 最后一次心跳时间
    int missed_count;       // 连续未收到心跳次数
} ClientInfo;

ClientInfo clients[1000];
int client_count = 0;

#define HEARTBEAT_INTERVAL 30  // 30秒未收到心跳算超时
#define MAX_MISSED 3           // 最多允许3次心跳丢失

// 处理心跳请求
void handle_heartbeat_req(int client_fd) 
{
    // 更新最后心跳时间
    for (int i = 0; i < client_count; i++) 
    {
        if (clients[i].fd == client_fd) 
        {
            clients[i].last_heartbeat = time(NULL);
            clients[i].missed_count = 0;
            break;
        }
    }
    // 回复心跳响应
    Message msg;
    msg.length = htonl(sizeof(msg.type));
    msg.type = MSG_TYPE_HEARTBEAT_RESP;
    send(client_fd, &msg, sizeof(msg.length) + sizeof(msg.type), 0);
}

// 定期检查心跳超时
void check_heartbeat_timeout() 
{
    time_t now = time(NULL);
    for (int i = 0; i < client_count; i++) 
    {
        if (now - clients[i].last_heartbeat > HEARTBEAT_INTERVAL) 
        {
            clients[i].missed_count++;
            if (clients[i].missed_count >= MAX_MISSED) 
            {
                printf("客户端%d心跳超时，关闭连接\n", clients[i].fd);
                close(clients[i].fd);
                // 从列表中移除
                clients[i] = clients[--client_count];
                i--;
            }
        }
    }
}

// 主循环
int main() 
{
    // ...创建socket、bind、listen...
    while (1) 
    {
        // epoll等待事件
        int nfds = epoll_wait(epfd, events, MAX_EVENTS, 1000);

        for (int i = 0; i < nfds; i++) {
            int fd = events[i].data.fd;
            // 读取消息
            Message msg;
            recv(fd, &msg, sizeof(msg.length) + sizeof(msg.type), 0);
            switch (msg.type) 
            {
            case MSG_TYPE_HEARTBEAT_REQ:
                handle_heartbeat_req(fd);
                break;
            case MSG_TYPE_DATA:
                // 处理业务数据...
                break;
            }
        }

        // 定期检查心跳超时
        static time_t last_check = 0;
        time_t now = time(NULL);
        if (now - last_check >= 10) {  // 每10秒检查一次
            check_heartbeat_timeout();
            last_check = now;
        }
    }
}

// 客户端：发送心跳
void *heartbeat_thread(void *arg) 
{
    int fd = *(int*)arg;
    while (1) {
        sleep(15);  // 每15秒发送一次心跳
        Message msg;
        msg.length = htonl(sizeof(msg.type));
        msg.type = MSG_TYPE_HEARTBEAT_REQ;
        if (send(fd, &msg, sizeof(msg.length) + sizeof(msg.type), 0) < 0) 
        {
            printf("发送心跳失败，连接可能已断开\n");
            break;
        }
    }

    return NULL;
}

int main() 
{
    int fd = socket(...);
    connect(fd, ...);

    // 启动心跳线程
    pthread_t tid;
    pthread_create(&tid, NULL, heartbeat_thread, &fd);

    // 主线程处理业务...
}
```

**应用层心跳的优点**：

```
✓ 灵活：间隔、超时时间自己控制
✓ 可扩展：心跳包可以携带业务信息
  - 客户端版本号
  - 负载信息
  - 在线用户数
  - ...

✓ 跨协议：不依赖TCP，UDP也可以用
✓ 检测应用层死锁：
  - TCP层连接还在，但应用层卡死
  - TCP Keepalive检测不出来
  - 应用层心跳可以检测

✓ 业务需求：
  - 在线状态显示（QQ、微信）
  - 连接池管理
  - 负载均衡
```

### 4.4 心跳策略对比

```
┌────────────────┬─────────────────┬─────────────────┐
│ 特性            │ TCP Keepalive   │ 应用层心跳       │
├────────────────┼─────────────────┼─────────────────┤
│ 实现难度        │ 简单(内核实现)    │ 需要自己实现      │
├────────────────┼─────────────────┼─────────────────┤
│ 检测间隔        │ 默认2小时(太长)   │ 自定义(秒级)      │
├────────────────┼─────────────────┼─────────────────┤
│ 灵活性          │ 低               │ 高              │
├────────────────┼─────────────────┼─────────────────┤
│ 携带业务信息     │ 不能             │ 可以            │
├────────────────┼─────────────────┼─────────────────┤
│ 检测应用层死锁   │ 不能             │ 可以             │
├────────────────┼─────────────────┼─────────────────┤
│ 跨平台          │ 是              │ 是               │
├────────────────┼─────────────────┼─────────────────┤
│ 适用场景        │ 辅助、兜底        │ 主要方案         │
└────────────────┴─────────────────┴─────────────────┘

推荐方案：
应用层心跳(主) + TCP Keepalive(辅助)

应用层心跳：
- 间隔：15-30秒
- 超时：3次未响应(45-90秒)
- 作用：快速检测应用层问题

TCP Keepalive：
- 配置：60秒开始、10秒间隔、3次探测
- 作用：兜底，防止极端情况
```

### 4.5 心跳优化

```c
// 优化1：心跳合并

// 不好的做法：空心跳
每15秒发送：
MSG_TYPE_HEARTBEAT_REQ (只有类型，没有数据)

// 更好的做法：piggyback
如果有业务数据要发送，不需要额外心跳
只在空闲时发送心跳

void send_data(int fd, const char *data, size_t len) 
{
    Message msg;
    msg.type = MSG_TYPE_DATA;
    // ...发送数据...
    last_send_time = time(NULL);  // 更新发送时间
}

void heartbeat_timer() 
{
    time_t now = time(NULL);
    if (now - last_send_time >= 15) // 15秒内没发过数据
    {  
        send_heartbeat();  // 才发心跳
    }
}

// 优化2：自适应心跳间隔
typedef struct 
{
    int fd;
    int heartbeat_interval;  // 动态调整
    int rtt;                 // 往返时延
} Connection;

void adjust_heartbeat_interval(Connection *conn) 
{
    // 根据RTT动态调整
    if (conn->rtt < 50) 
    {
        conn->heartbeat_interval = 30;  // 低延迟，30秒心跳
    } 
    else if (conn->rtt < 200) 
    {
        conn->heartbeat_interval = 20;  // 中延迟，20秒心跳
    } 
    else 
    {
        conn->heartbeat_interval = 10;  // 高延迟，10秒心跳
    }
}

// 优化3：批量心跳（服务器端）

// 如果服务器有1万个客户端连接
// 每个客户端每15秒发一次心跳
// 服务器每秒收到约666个心跳

// 优化：批量处理
void batch_process_heartbeats() 
{
    time_t now = time(NULL);

    // 每秒只处理一次，批量更新
    for (int i = 0; i < client_count; i++) 
    {
        if (clients[i].heartbeat_pending) 
        {
            clients[i].last_heartbeat = now;
            clients[i].heartbeat_pending = 0;
        }
    }
}

// 优化4：心跳请求/响应分离

// 客户端：只发请求，不等响应
send_heartbeat_req();  // 发送后不阻塞

// 服务器：异步回复
void handle_heartbeat_req(int fd) 
{
    update_client_heartbeat(fd);
    send_heartbeat_resp_async(fd);  // 异步发送
}
```

### 4.6 面试常见问题

**Q1: TCP Keepalive和应用层心跳的区别？**

```
答：（见4.4的对比表）

补充：选择建议
- 对实时性要求高：应用层心跳（游戏、IM）
- 连接池、数据库：两者都用
- 简单场景：TCP Keepalive够用
- 推荐：应用层心跳为主，TCP Keepalive为辅
```

**Q2: 心跳间隔设置多少合适？**

```
答：

考虑因素：
1. 网络环境
   - 内网：可以30-60秒
   - 公网：建议15-30秒
   - 移动网络：10-20秒（易断）

2. NAT超时
   - 很多NAT设备60-300秒会超时
   - 心跳间隔要小于NAT超时

3. 业务需求
   - IM（QQ、微信）：15-30秒
   - 游戏：5-10秒（实时性高）
   - 监控系统：30-60秒

4. 服务器压力
   - 1万连接 × 每15秒1次 = 666次/秒
   - 100万连接 × 每15秒1次 = 6.6万次/秒
   - 需要权衡

推荐配置：
间隔：15-30秒
超时：3次未响应（45-90秒）
```

**Q3: 如何检测客户端伪造心跳？**

```c
// 问题：恶意客户端一直发心跳，但不处理业务

typedef struct 
{
    int fd;
    time_t last_heartbeat;
    time_t last_business;  // ← 增加：最后一次业务时间
} ClientInfo;

void check_zombie_client() 
{
    time_t now = time(NULL);
    for (int i = 0; i < client_count; i++) 
    {
        // 有心跳，但很久没有业务数据
        if (now - clients[i].last_heartbeat < 60 && now - clients[i].last_business > 600) // 10分钟没业务
            {  
            printf("客户端%d可疑：只有心跳，没有业务\n", clients[i].fd);
            // 选择：
            // 1. 发送probe消息，要求回复
            // 2. 强制关闭
            // 3. 标记为异常
        }
    }
}
```

---

## 5. 常用Socket选项

### 5.1 SO_REUSEADDR vs SO_REUSEPORT

**SO_REUSEADDR**：

```c
int reuse = 1;
setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));

// 作用：
// 1. 允许bind到TIME_WAIT状态的端口
// 2. 允许多个socket bind到同一IP:PORT（需要都设置此选项）
// 3. 允许bind到通配地址和具体地址

// 场景1：服务器快速重启
// 问题：服务器关闭后，端口处于TIME_WAIT（120秒）
//      重启时bind()失败：Address already in use
// 解决：设置SO_REUSEADDR

// 场景2：多网卡绑定
// 允许：
bind(fd1, "0.0.0.0:8888");  // 通配地址
bind(fd2, "192.168.1.1:8888");  // 具体地址（需要SO_REUSEADDR）

// 场景3：UDP多播
// 允许多个进程接收同一个多播组
```

**SO_REUSEPORT**：

```c
int reuse = 1;
setsockopt(listen_fd, SOL_SOCKET, SO_REUSEPORT, &reuse, sizeof(reuse));

// 作用：（Linux 3.9+）
// 允许多个socket bind到完全相同的IP:PORT
// 内核会做负载均衡，分发连接到不同socket

// 典型应用：多进程服务器

// 进程1：
int fd1 = socket(...);
setsockopt(fd1, SOL_SOCKET, SO_REUSEPORT, ...);
bind(fd1, "0.0.0.0:8888");
listen(fd1, 128);

// 进程2：
int fd2 = socket(...);
setsockopt(fd2, SOL_SOCKET, SO_REUSEPORT, ...);
bind(fd2, "0.0.0.0:8888");  // 成功！和进程1一样的地址
listen(fd2, 128);

// 进程3、4、5... 同样操作

// 内核行为：
// 新连接到来 → 内核根据hash(source_ip, source_port)
//            → 分发到某个进程的fd
//            → 负载均衡！

// 优点：
// ✓ 避免惊群（每个进程独立的accept队列）
// ✓ 更好的CPU缓存局部性
// ✓ 热重启更平滑
```

**对比**：

```
┌─────────────────┬──────────────────┬──────────────────┐
│                 │ SO_REUSEADDR     │ SO_REUSEPORT     │
├─────────────────┼──────────────────┼──────────────────┤
│ TIME_WAIT端口    │ 允许             │ 允许              │
├─────────────────┼──────────────────┼──────────────────┤
│ 完全相同地址      │ 不允许(UDP允许)    │ 允许             │
├─────────────────┼──────────────────┼──────────────────┤
│ 内核负载均衡      │ 无               │ 有                │
├─────────────────┼──────────────────┼──────────────────┤
│ 多进程绑定        │ 不支持            │ 支持             │
├─────────────────┼──────────────────┼──────────────────┤
│ 适用场景         │ 快速重启           │ 多进程服务器       │
└─────────────────┴──────────────────┴──────────────────┘
```

### 5.2 TCP_NODELAY（禁用Nagle算法）

```c
int flag = 1;
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));

// Nagle算法：
// 目的：减少小包数量，提高网络利用率
// 规则：
//   1. 如果有未确认的数据，缓存新数据
//   2. 直到：
//      - 收到前一个包的ACK，或
//      - 累积到MSS大小
//   3. 然后一次性发送

// 问题场景：交互式应用

// 客户端发送请求（SSH、游戏）：
send(fd, "ls", 2);  // 只有2字节
// Nagle算法：等待ACK或更多数据...
//          ↓ 延迟！
// 服务器需要等待才能收到"ls"

// 解决：禁用Nagle（TCP_NODELAY）
// 优点：低延迟，实时性好
// 缺点：可能产生更多小包，网络利用率降低

// 适用场景：
// ✓ 实时游戏
// ✓ SSH/Telnet
// ✓ 股票行情
// ✓ 视频会议
// ✓ 任何对延迟敏感的应用

// 不适用：
// ✗ 文件传输（大块数据，吞吐量优先）
// ✗ HTTP（通常带Content-Length，一次性发送）
```

### 5.3 SO_RCVBUF / SO_SNDBUF（缓冲区大小）

```c
// 接收缓冲区
int rcvbuf = 256 * 1024;  // 256KB
setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf));

// 发送缓冲区
int sndbuf = 256 * 1024;  // 256KB
setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &sndbuf, sizeof(sndbuf));

// 默认大小（Linux）：
// cat /proc/sys/net/ipv4/tcp_rmem  # 接收: min default max
// 4096 87380 6291456  (最小4KB, 默认85KB, 最大6MB)

// cat /proc/sys/net/ipv4/tcp_wmem  # 发送
// 4096 16384 4194304  (最小4KB, 默认16KB, 最大4MB)

// 何时调整？

// 1. 高带宽长延迟网络（BDP优化）
// BDP = 带宽 × RTT
// 例如：100Mbps，RTT=100ms
// BDP = 100*1000*1000 / 8 * 0.1 = 1.25MB
// 缓冲区至少需要1.25MB才能跑满带宽

int bdp = 2 * 1024 * 1024;  // 2MB
setsockopt(fd, SOL_SOCKET, SO_RCVBUF, &bdp, sizeof(bdp));

// 2. 批量传输
// 发送大文件时，增大发送缓冲区
int sndbuf = 1024 * 1024;  // 1MB
setsockopt(fd, SOL_SOCKET, SO_SNDBUF, &sndbuf, sizeof(sndbuf));

// 注意：
// - 内核会将你设置的值翻倍（为什么？）
// - 实际缓冲区 = min(你设置的 * 2, 系统最大值)
// - 设置过大可能导致内存浪费
// - 设置过小导致性能下降
```

### 5.4 SO_LINGER（控制close行为）

```c
struct linger 
{
    int l_onoff;   // 0=关闭, 1=开启
    int l_linger;  // 延迟时间（秒）
};

struct linger l;

// 情况1：默认行为（l_onoff=0）
l.l_onoff = 0;
l.l_linger = 0;  // 忽略
setsockopt(fd, SOL_SOCKET, SO_LINGER, &l, sizeof(l));

close(fd);
// 行为：
// - 立即返回
// - 发送缓冲区数据在后台继续发送
// - 发送FIN
// - 进入TIME_WAIT（如果是主动关闭方）

// 情况2：强制关闭（l_onoff=1, l_linger=0）
l.l_onoff = 1;
l.l_linger = 0;  // 超时为0
setsockopt(fd, SOL_SOCKET, SO_LINGER, &l, sizeof(l));

close(fd);
// 行为：
// - 立即返回
// - 丢弃发送缓冲区中的数据
// - 发送RST（而不是FIN）
// - 立即CLOSED（不经过TIME_WAIT）
// 用途：
// ✓ 避免TIME_WAIT（但不符合TCP规范）
// ✗ 可能丢失数据

// 情况3：延迟关闭（l_onoff=1, l_linger>0）
l.l_onoff = 1;
l.l_linger = 5;  // 最多等待5秒
setsockopt(fd, SOL_SOCKET, SO_LINGER, &l, sizeof(l));

close(fd);
// 行为：
// - close()阻塞，等待发送缓冲区数据发送完毕
// - 或者等待l_linger秒超时
// - 超时后发送RST
// 用途：
// ✓ 确保数据发送完毕
// ✗ 阻塞close()，可能卡住程序

// 推荐：
// - 大部分情况用默认（l_onoff=0）
// - 特殊需求才用l_onoff=1
```

### 5.5 SO_KEEPALIVE（TCP保活）

```c
int keepalive = 1;
setsockopt(fd, SOL_SOCKET, SO_KEEPALIVE, &keepalive, sizeof(keepalive));

// 配置keepalive参数
int keepidle = 60;   // 60秒空闲后探测
int keepintvl = 10;  // 每隔10秒探测一次
int keepcnt = 3;     // 最多探测3次

setsockopt(fd, IPPROTO_TCP, TCP_KEEPIDLE, &keepidle, sizeof(keepidle));
setsockopt(fd, IPPROTO_TCP, TCP_KEEPINTVL, &keepintvl, sizeof(keepintvl));
setsockopt(fd, IPPROTO_TCP, TCP_KEEPCNT, &keepcnt, sizeof(keepcnt));
```

### 5.6 其他常用选项

```c
// 1. SO_RCVTIMEO / SO_SNDTIMEO（读写超时）

struct timeval timeout;
timeout.tv_sec = 5;   // 5秒超时
timeout.tv_usec = 0;

setsockopt(fd, SOL_SOCKET, SO_RCVTIMEO, &timeout, sizeof(timeout));
setsockopt(fd, SOL_SOCKET, SO_SNDTIMEO, &timeout, sizeof(timeout));

// recv()超过5秒没数据 → 返回-1, errno=EAGAIN

// 2. TCP_CORK（TCP木塞）

int cork = 1;
setsockopt(fd, IPPROTO_TCP, TCP_CORK, &cork, sizeof(cork));

// 作用：
// - 开启后，send()不会立即发送，而是积累数据
// - 直到：
//   1. 关闭TCP_CORK，或
//   2. 达到MSS大小
// - 然后一次性发送

// 场景：HTTP响应
setsockopt(fd, IPPROTO_TCP, TCP_CORK, &cork, sizeof(cork));  // 开启

send(fd, "HTTP/1.1 200 OK\r\n", ...);      // 不发送
send(fd, "Content-Length: 1024\r\n\r\n", ...);  // 不发送
send(fd, file_content, 1024);              // 不发送

cork = 0;
setsockopt(fd, IPPROTO_TCP, TCP_CORK, &cork, sizeof(cork));  // 一次性发送所有

// 对比TCP_NODELAY：
// TCP_NODELAY：禁用Nagle，立即发送
// TCP_CORK：延迟发送，积累数据

// 3. SO_BROADCAST（广播）

int broadcast = 1;
setsockopt(fd, SOL_SOCKET, SO_BROADCAST, &broadcast, sizeof(broadcast));

// UDP广播到255.255.255.255
struct sockaddr_in addr;
addr.sin_addr.s_addr = htonl(INADDR_BROADCAST);
sendto(fd, data, len, 0, (struct sockaddr*)&addr, sizeof(addr));

// 4. IP_TTL（生存时间）

int ttl = 64;
setsockopt(fd, IPPROTO_IP, IP_TTL, &ttl, sizeof(ttl));

// 每经过一个路由器，TTL减1
// TTL=0时，包被丢弃
// 默认：64或128
```

---

## 6. 非阻塞IO和epoll

### 6.1 阻塞 vs 非阻塞

**阻塞IO**：

```c
// 阻塞模式（默认）
int fd = socket(AF_INET, SOCK_STREAM, 0);
connect(fd, ...);

char buf[1024];
int n = recv(fd, buf, sizeof(buf), 0);  // ← 阻塞在这里

// 如果没有数据：
// - 线程/进程睡眠，让出CPU
// - 等待数据到达
// - 被内核唤醒
// - 返回数据

// 问题：一个线程只能处理一个连接
// 1000个客户端 → 需要1000个线程 → 资源浪费
```

**非阻塞IO**：

```c
// 设置为非阻塞
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);

// 或创建时设置
int fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);

// 非阻塞读取
int n = recv(fd, buf, sizeof(buf), 0);

if (n > 0) 
{
    // 读到数据
} 
else if (n == 0) 
{
    // 连接关闭
} 
else 
{  
    // n < 0
    if (errno == EAGAIN || errno == EWOULDBLOCK) 
    {
        // 没有数据，稍后重试
        // 不阻塞！
    } 
    else 
    {
        // 真正的错误
    }
}

// 优点：一个线程可以处理多个连接
// 需要配合：select/poll/epoll
```

### 6.2 select/poll/epoll对比

**select**：

```c
fd_set readfds;
FD_ZERO(&readfds);
FD_SET(fd1, &readfds);
FD_SET(fd2, &readfds);
FD_SET(fd3, &readfds);

struct timeval timeout = {5, 0};  // 5秒超时
int ready = select(max_fd + 1, &readfds, NULL, NULL, &timeout);

if (ready > 0) 
{
    if (FD_ISSET(fd1, &readfds)) 
    {
        // fd1可读
    }
    if (FD_ISSET(fd2, &readfds)) 
    {
        // fd2可读
    }
}

// 缺点：
// ✗ 有最大文件描述符限制（1024）
// ✗ 每次调用需要拷贝fd_set到内核
// ✗ 需要遍历所有fd检查FD_ISSET
// ✗ O(n)复杂度
```

**poll**：

```c
struct pollfd fds[1000];
fds[0].fd = fd1;
fds[0].events = POLLIN;  // 监听可读
fds[1].fd = fd2;
fds[1].events = POLLIN;
// ...

int ready = poll(fds, 1000, 5000);  // 5秒超时

if (ready > 0) 
{
    for (int i = 0; i < 1000; i++) 
    {
        if (fds[i].revents & POLLIN) 
        {
            // fds[i].fd可读
        }
    }
}

// 改进：
// ✓ 没有最大fd限制
// 缺点：
// ✗ 仍需要拷贝pollfd数组
// ✗ 仍需要遍历检查
// ✗ O(n)复杂度
```

**epoll**（Linux）：

```c
// 1. 创建epoll实例
int epfd = epoll_create1(0);

// 2. 添加fd到epoll
struct epoll_event ev;
ev.events = EPOLLIN;  // 监听可读
ev.data.fd = fd1;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd1, &ev);

ev.data.fd = fd2;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd2, &ev);

// 3. 等待事件
struct epoll_event events[100];
int ready = epoll_wait(epfd, events, 100, 5000);  // 5秒超时

// 4. 处理就绪的fd
for (int i = 0; i < ready; i++) 
{
    int fd = events[i].data.fd;
    if (events[i].events & EPOLLIN) 
    {
        // fd可读
        recv(fd, ...);
    }
}

// 优点：
// ✓ 没有fd数量限制
// ✓ 不需要每次拷贝fd列表
// ✓ 只返回就绪的fd，不需要遍历
// ✓ O(1)复杂度（获取就绪事件）
```

**对比表**：

```
┌──────────┬─────────┬─────────┬─────────┐
│          │ select  │ poll    │ epoll   │
├──────────┼─────────┼─────────┼─────────┤
│ 最大连接  │ 1024    │ 无限制   │ 无限制   │
├──────────┼─────────┼─────────┼─────────┤
│ fd拷贝    │ 每次    │ 每次     │ 一次     │
├──────────┼─────────┼─────────┼─────────┤
│ 遍历      │ O(n)    │ O(n)    │ O(1)    │
├──────────┼─────────┼─────────┼─────────┤
│ 触发方式  │ LT      │ LT      │ LT/ET   │
├──────────┼─────────┼─────────┼─────────┤
│ 跨平台    │ 是      │ 是       │ Linux   │
├──────────┼─────────┼─────────┼─────────┤
│ 适用场景  │ <100连接│ <1000    │ 高并发   │
└──────────┴─────────┴─────────┴─────────┘
```

### 6.3 epoll的LT vs ET模式

**LT (Level Triggered) 水平触发**：

```c
struct epoll_event ev;
ev.events = EPOLLIN;  // LT模式（默认）
ev.data.fd = fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

// 行为：
// 只要fd可读，epoll_wait()就会返回
// 即使你没有读取数据

// 示例：
// 1. 100字节数据到达
// 2. epoll_wait()返回fd
// 3. 你只read了50字节
// 4. 下次epoll_wait()仍然返回fd（还有50字节）
// 5. 直到读完所有数据

// 优点：
// ✓ 编程简单，不容易出错
// ✓ 即使没读完数据，也会再次通知

// 缺点：
// ✗ 如果不及时读取，会频繁触发
```

**ET (Edge Triggered) 边缘触发**：

```c
struct epoll_event ev;
ev.events = EPOLLIN | EPOLLET;  // ET模式
ev.data.fd = fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

// 行为：
// 只在状态变化时通知一次
// 必须读取所有数据（直到EAGAIN）

// 示例：
// 1. 100字节数据到达 → 可读状态变化
// 2. epoll_wait()返回fd
// 3. 你只read了50字节
// 4. 下次epoll_wait()不会返回fd！（状态没变化）
// 5. 除非新数据到达，才会再次触发

// 正确用法：
while (1) 
{
    int n = recv(fd, buf, sizeof(buf), 0);
    if (n > 0) 
    {
        // 处理数据...
    } 
    else if (n == 0) 
    {
        // 连接关闭
        break;
    } 
    else 
    {
        if (errno == EAGAIN) 
        {
            // 读完了，退出循环
            break;
        }
        // 错误处理
        break;
    }
}

// 优点：
// ✓ 效率高，减少epoll_wait()触发次数
// ✓ 高并发性能更好

// 缺点：
// ✗ 编程复杂，必须循环读到EAGAIN
// ✗ 容易出错（忘记循环读取 → 数据丢失）
```

**LT vs ET选择**：

```
推荐：
- 新手：LT（简单可靠）
- 高性能需求：ET（需要仔细编程）
- 实际项目：nginx使用ET，libevent默认LT
```

### 6.4 epoll惊群问题

```c
// 问题：多个进程/线程同时epoll_wait同一个listen_fd

// 进程1：
epoll_wait(epfd, ...);  // 等待

// 进程2：
epoll_wait(epfd, ...);  // 等待

// 进程3：
epoll_wait(epfd, ...);  // 等待

// 新连接到来 → 3个进程都被唤醒！
//            → 但只有1个进程能accept成功
//            → 其他2个进程白白被唤醒
//            → 惊群！

// 解决方案：

// 1. EPOLLEXCLUSIVE（Linux 4.5+）
struct epoll_event ev;
ev.events = EPOLLIN | EPOLLEXCLUSIVE;  // ← 独占模式
ev.data.fd = listen_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

// 只唤醒一个进程 ✓

// 2. SO_REUSEPORT（推荐）
// 每个进程独立的listen_fd
// 内核负载均衡分发连接
// 见5.1节

// 3. 应用层加锁
pthread_mutex_lock(&accept_mutex);
epoll_wait(epfd, ...);
accept(...);
pthread_mutex_unlock(&accept_mutex);

// 但性能差，不推荐
```

---

## 7. 零拷贝技术

### 7.1 传统IO的4次拷贝

```
场景：发送文件

read(file_fd, buf, size);
send(socket_fd, buf, size);

数据流向：
┌──────────────────────────────────────────┐
│ 1. 磁盘 → 内核缓冲区（DMA拷贝）              │
│ 2. 内核缓冲区 → 用户缓冲区（CPU拷贝）         │  ← read()
│ 3. 用户缓冲区 → Socket缓冲区（CPU拷贝）      │  ← send()
│ 4. Socket缓冲区 → 网卡（DMA拷贝）           │
└──────────────────────────────────────────┘

共：4次拷贝，2次DMA，2次CPU

CPU拷贝浪费：
- 数据从内核拷贝到用户空间，又拷贝回内核
- 完全没必要！
```

### 7.2 sendfile（2次拷贝）

```c
#include <sys/sendfile.h>

int file_fd = open("file.txt", O_RDONLY);
int socket_fd = socket(...);

// sendfile：直接在内核中传输
off_t offset = 0;
size_t count = 1024 * 1024;  // 1MB
ssize_t sent = sendfile(socket_fd, file_fd, &offset, count);

// 数据流向：
┌──────────────────────────────────────────┐
│ 1. 磁盘 → 内核缓冲区（DMA拷贝）              │
│ 2. 内核缓冲区 → Socket缓冲区（CPU拷贝）      │  ← 内核内拷贝
│ 3. Socket缓冲区 → 网卡（DMA拷贝）           │
└──────────────────────────────────────────┘

共：3次拷贝，2次DMA，1次CPU
省掉了用户空间的拷贝！

// 优点：
// ✓ 减少拷贝次数
// ✓ 减少用户态/内核态切换
// ✓ 零拷贝（Zero Copy）

// 缺点：
// ✗ 只支持文件 → socket
// ✗ 不能修改数据（直接传输）

// 适用场景：
// ✓ 文件服务器（HTTP、FTP）
// ✓ 静态资源（nginx）
// ✓ 视频点播
```

### 7.3 splice（管道零拷贝）

```c
#include <fcntl.h>

int pipefd[2];
pipe(pipefd);

// 文件 → 管道
ssize_t n = splice(file_fd, NULL, pipefd[1], NULL, size, SPLICE_F_MOVE);

// 管道 → socket
splice(pipefd[0], NULL, socket_fd, NULL, n, SPLICE_F_MOVE);

// 数据流向：
// 文件 → 管道缓冲区 → Socket
// 全程在内核空间，零拷贝！

// 优点：
// ✓ 真正的零拷贝
// ✓ 更灵活（可以多次splice）

// 缺点：
// ✗ 需要管道
// ✗ 代码稍复杂
```

### 7.4 mmap（内存映射）

```c
#include <sys/mman.h>

int fd = open("file.txt", O_RDONLY);
struct stat sb;
fstat(fd, &sb);

// 将文件映射到内存
char *addr = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);

// 直接send映射的内存
send(socket_fd, addr, sb.st_size, 0);

munmap(addr, sb.st_size);

// 数据流向：
┌──────────────────────────────────────────┐
│ 1. 磁盘 → 页缓存（按需加载）                 │
│ 2. 页缓存 → Socket缓冲区（CPU拷贝）         │
│ 3. Socket缓冲区 → 网卡（DMA拷贝）           │
└──────────────────────────────────────────┘

共：3次拷贝（比传统少1次）

// 优点：
// ✓ 可以随机访问文件内容
// ✓ 多个进程可以共享映射
// ✓ 适合大文件

// 缺点：
// ✗ 文件大小改变需要重新mmap
// ✗ 可能有缺页中断（page fault）
```

### 7.5 零拷贝技术对比

```
┌──────────┬───────┬────────┬──────────┬──────────┐
│ 技术      │ 拷贝  │ 系统调用 │ 灵活性    │ 适用场景  │
├──────────┼───────┼────────┼──────────┼──────────┤
│ 传统IO    │ 4次   │ 2次    │ 高        │ 通用     │
├──────────┼───────┼────────┼──────────┼──────────┤
│ sendfile │ 3次   │ 1次     │ 低       │ 文件发送  │
├──────────┼───────┼────────┼──────────┼──────────┤
│ splice   │ 0次   │ 2次     │ 中       │ 管道传输  │
├──────────┼───────┼────────┼──────────┼──────────┤
│ mmap     │ 3次   │ 2次     │ 高       │ 大文件    │
└──────────┴───────┴────────┴──────────┴──────────┘

推荐：
- 小文件：传统IO（简单）
- 中等文件：sendfile（nginx使用）
- 大文件：mmap（随机访问）
- 管道场景：splice
```

---

## 8. 拥塞控制和流量控制

### 8.1 流量控制（滑动窗口）

**目的**：防止发送方发得太快，接收方来不及处理。

```
接收窗口（rwnd）：

接收方                           发送方
┌─────────────────┐               
│ 接收缓冲区        │               
│ ┌─────────────┐ │  告诉发送方    
│ │ 空闲50KB     │ │ ──窗口=50KB─→  最多发送50KB
│ │             │ │               
│ │ 已用10KB     │ │               
│ └─────────────┘ │               
└─────────────────┘               

发送方：
- 最多发送50KB未确认数据
- 超过50KB，停止发送
- 等待ACK，窗口滑动

窗口探测：
如果窗口=0（接收缓冲区满）
发送方定期发送窗口探测包（1字节）
询问窗口是否变大
```

### 8.2 拥塞控制（4个算法）

**1. 慢启动（Slow Start）**

```
目的：探测网络容量

初始窗口：cwnd = 1 MSS (约1460字节)

每收到1个ACK：cwnd 加倍
┌────────────────────────────┐
│ RTT 1: cwnd = 1 MSS        │
│ RTT 2: cwnd = 2 MSS        │
│ RTT 3: cwnd = 4 MSS        │
│ RTT 4: cwnd = 8 MSS        │
│ RTT 5: cwnd = 16 MSS       │
│ ...                        │
│ 指数增长！                   │
└────────────────────────────┘

直到：
- 达到ssthresh（慢启动阈值），或
- 发生丢包

然后进入拥塞避免
```

**2. 拥塞避免（Congestion Avoidance）**

```
触发：cwnd >= ssthresh

每收到1个ACK：cwnd += 1/cwnd
┌────────────────────────────┐
│ RTT 1: cwnd = 16 MSS       │
│ RTT 2: cwnd = 17 MSS       │
│ RTT 3: cwnd = 18 MSS       │
│ RTT 4: cwnd = 19 MSS       │
│ ...                        │
│ 线性增长！（慢慢试探）         │
└────────────────────────────┘

直到发生丢包 → 进入快速重传/快速恢复
```

**3. 快速重传（Fast Retransmit）**

```
检测丢包：收到3个重复ACK

正常情况：
发送：1, 2, 3, 4, 5
接收：ACK 2, ACK 3, ACK 4, ACK 5 ✓

丢包情况（包3丢失）：
发送：1, 2, 3✗, 4, 5
接收：ACK 2, ACK 2, ACK 2, ACK 2
      ↑重复ACK（期望收到3，但没收到）

收到3个重复ACK 2：
→ 确定包3丢失
→ 立即重传包3（不等超时！）
→ 快速恢复
```

**4. 快速恢复（Fast Recovery）**

```
发生丢包后：

ssthresh = cwnd / 2  // 阈值减半
cwnd = ssthresh      // 窗口降到阈值
// 不回到慢启动！

继续拥塞避免（线性增长）
```

**完整的拥塞控制状态机**：

```
cwnd
  ↑
  │    ┌─慢启动(指数)─→ 拥塞避免(线性) ─→
  │    │                     ↓ 丢包
  │    │                     ↓
  │    └─快速恢复 ←─────────┘
  │
  └─→ 时间
```

---

## 9. 高并发优化技巧

### 9.1 系统参数调优

```bash
# 1. 增加文件描述符限制
ulimit -n 1000000  # 当前shell
echo "* soft nofile 1000000" >> /etc/security/limits.conf
echo "* hard nofile 1000000" >> /etc/security/limits.conf

# 2. TCP相关
sysctl -w net.ipv4.tcp_tw_reuse=1        # TIME_WAIT复用
sysctl -w net.ipv4.tcp_fin_timeout=30    # FIN_WAIT超时
sysctl -w net.ipv4.tcp_max_syn_backlog=8192  # SYN队列
sysctl -w net.core.somaxconn=8192        # ACCEPT队列

# 3. 端口范围
sysctl -w net.ipv4.ip_local_port_range="10000 65000"

# 4. 内核连接跟踪
sysctl -w net.netfilter.nf_conntrack_max=1000000
```

### 9.2 架构优化

```
1. Reactor模式（epoll）
   主线程epoll监听 → 工作线程处理

2. 多进程模型（nginx）
   SO_REUSEPORT + 多个worker进程

3. 协程/异步IO（Go、Node.js）
   单线程处理大量连接

4. 连接池
   复用连接，减少TIME_WAIT

5. 负载均衡
   分散连接到多台服务器
```

---

## 10. 面试常见问题汇总

### 10.1 基础问题

**Q1: TCP和UDP的区别？**

```
TCP：
✓ 面向连接（三次握手）
✓ 可靠传输（重传、确认）
✓ 有序传输
✓ 流量控制
✓ 拥塞控制
✗ 开销大
✗ 延迟较高

UDP：
✓ 无连接
✓ 不可靠（尽力而为）
✓ 开销小
✓ 延迟低
✗ 无序
✗ 可能丢包

选择：
- TCP：文件传输、HTTP、SMTP、SSH
- UDP：视频直播、DNS、游戏（实时性）
```

**Q2: listen的backlog是什么？**

```c
listen(fd, backlog);

// backlog：ACCEPT队列的最大长度
// 实际值：min(backlog, somaxconn)

// 太小：高并发下连接被拒绝
// 太大：占用内存

// 推荐值：
// - 低并发：128
// - 中并发：512
// - 高并发：1024-8192
```

**Q3: 如何检测连接断开？**

```c
// 1. recv()返回0
if (recv(fd, buf, sizeof(buf), 0) == 0) {
    // 对方正常关闭（FIN）
}

// 2. recv()返回-1, errno=ECONNRESET
// 对方异常关闭（RST）

// 3. send()返回-1, errno=EPIPE
// 对方已关闭，但本端还在写

// 4. 心跳超时
// 应用层心跳多次无响应
```

### 10.2 高频问题

**Q4: 为什么TIME_WAIT是2MSL？**

```
MSL = 60秒（Maximum Segment Lifetime）

2MSL的原因：
1. 最后的ACK可能丢失
   → 服务器重传FIN（最多MSL到达）
   → 客户端重发ACK（最多MSL返回）
   → 总共2*MSL

2. 保证旧连接的包全部过期
   → 发送方最多MSL到达接收方
   → 回复最多MSL返回发送方
   → 总共2*MSL
```

**Q5: epoll的ET和LT有什么区别？**

（见6.3节详细对比）

**Q6: 如何实现一个高性能服务器？**

```
1. IO模型：epoll（ET模式）
2. 线程模型：主线程accept，工作线程池处理
3. 非阻塞IO：避免阻塞调用
4. 零拷贝：sendfile发送文件
5. 连接池：复用连接
6. 心跳机制：检测死连接
7. 优雅关闭：shutdown而不是close
8. 系统调优：ulimit、sysctl参数
```

---

## 总结

本文档涵盖了网络编程面试的所有高频知识点：

1. **粘包/拆包**：长度前缀方案最常用
2. **四次挥手**：TIME_WAIT、CLOSE_WAIT状态
3. **TIME_WAIT优化**：tcp_tw_reuse、SO_REUSEADDR
4. **心跳机制**：应用层心跳为主
5. **Socket选项**：SO_REUSEPORT、TCP_NODELAY
6. **epoll**：ET模式更高效
7. **零拷贝**：sendfile、splice、mmap
8. **拥塞控制**：慢启动、拥塞避免
9. **高并发优化**：系统参数、架构设计
10. **常见问题**：TCP vs UDP、TIME_WAIT、epoll

面试前建议：
✓ 熟记三次握手、四次挥手的seq/ack值
✓ 理解粘包/拆包的4种解决方案
✓ 掌握TIME_WAIT的原因和优化
✓ 了解epoll的LT/ET区别
✓ 能写出基本的epoll服务器代码


---

## 11. 实战代码与调试技巧

### 11.1 完整的Echo服务器（三个版本）

#### 版本1：基础单线程阻塞IO

```c
// echo_server_v1.c - 最简单的单线程版本
// 编译: gcc echo_server_v1.c -o echo_v1

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define PORT 8888
#define BUFFER_SIZE 1024

int main() 
{
    int listen_fd, client_fd;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_len;
    char buffer[BUFFER_SIZE];
    
    // 1. 创建socket
    listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_fd < 0) 
    {
        perror("socket");
        exit(1);
    }
    
    // 2. 设置SO_REUSEADDR（重要！）
    int reuse = 1;
    if (setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse)) < 0) 
    {
        perror("setsockopt");
        exit(1);
    }
    
    // 3. 绑定地址
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);
    
    if (bind(listen_fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) 
    {
        perror("bind");
        exit(1);
    }
    
    // 4. 监听
    if (listen(listen_fd, 128) < 0) 
    {
        perror("listen");
        exit(1);
    }
    
    printf("Echo server listening on port %d...\n", PORT);
    
    // 5. 主循环
    while (1) 
    {
        // 接受连接
        client_len = sizeof(client_addr);
        client_fd = accept(listen_fd, (struct sockaddr*)&client_addr,  &client_len);
        if (client_fd < 0) 
        {
            perror("accept");
            continue;
        }
        printf("New client: %s:%d\n", inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port));
        // 处理客户端（echo back）
        while (1) 
        {
            int n = recv(client_fd, buffer, BUFFER_SIZE - 1, 0);
            if (n > 0) 
            {
                buffer[n] = '\0';
                printf("Received: %s", buffer);             
                // 回显
                send(client_fd, buffer, n, 0);                
            } 
            else if (n == 0) 
            {
                // 客户端关闭
                printf("Client disconnected\n");
                break;                
            } 
            else 
            {
                perror("recv");
                break;
            }
        }
        close(client_fd);
    }
    close(listen_fd);
    return 0;
}

// 缺点：
// ✗ 一次只能服务一个客户端
// ✗ 处理慢的客户端会阻塞其他客户端
// ✗ 不适合生产环境
//
// 优点：
// ✓ 代码简单，易于理解
// ✓ 适合学习和demo
```

#### 版本2：多线程版本

```c
// echo_server_v2.c - 多线程版本
// 编译: gcc echo_server_v2.c -o echo_v2 -lpthread

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define PORT 8888
#define BUFFER_SIZE 1024

// 线程参数
typedef struct 
{
    int client_fd;
    struct sockaddr_in client_addr;
} ClientInfo;

// 处理客户端的线程函数
void *handle_client(void *arg) 
{
    ClientInfo *info = (ClientInfo*)arg;
    int client_fd = info->client_fd;
    char buffer[BUFFER_SIZE];
    
    printf("Thread %lu: handling client %s:%d\n",
           pthread_self(), inet_ntoa(info->client_addr.sin_addr), ntohs(info->client_addr.sin_port));
    
    // Echo循环
    while (1) 
    {
        int n = recv(client_fd, buffer, BUFFER_SIZE - 1, 0);       
        if (n > 0) 
        {
            buffer[n] = '\0';
            printf("Thread %lu received: %s", pthread_self(), buffer);
            send(client_fd, buffer, n, 0);      
        } 
        else if (n == 0) 
        {
            printf("Thread %lu: client disconnected\n", pthread_self());
            break;            
        } 
        else 
        {
            perror("recv");
            break;
        }
    }
    
    close(client_fd);
    free(info);
    
    printf("Thread %lu: exiting\n", pthread_self());
    return NULL;
}

int main() 
{
    int listen_fd;
    struct sockaddr_in server_addr;
    
    // 创建socket
    listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_fd < 0) 
    {
        perror("socket");
        exit(1);
    }
    
    // SO_REUSEADDR
    int reuse = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));
    
    // 绑定
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);
    
    if (bind(listen_fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) 
    {
        perror("bind");
        exit(1);
    }
    
    // 监听
    if (listen(listen_fd, 128) < 0) 
    {
        perror("listen");
        exit(1);
    }
    
    printf("Multi-threaded echo server listening on port %d...\n", PORT);
    
    // 主循环：accept + 创建线程
    while (1) 
    {
        ClientInfo *info = malloc(sizeof(ClientInfo));
        socklen_t len = sizeof(info->client_addr);
        info->client_fd = accept(listen_fd, (struct sockaddr*)&info->client_addr, &len);
        if (info->client_fd < 0) 
        {
            perror("accept");
            free(info);
            continue;
        }
        // 创建线程处理
        pthread_t tid;
        pthread_attr_t attr;
        pthread_attr_init(&attr);
        pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);  // 分离线程
        
        if (pthread_create(&tid, &attr, handle_client, info) != 0) 
        {
            perror("pthread_create");
            close(info->client_fd);
            free(info);
        }
        pthread_attr_destroy(&attr);
    }
    
    close(listen_fd);
    return 0;
}

// 优点：
// ✓ 支持多个客户端并发
// ✓ 每个客户端独立处理
//
// 缺点：
// ✗ 每个连接一个线程，1万连接 = 1万线程
// ✗ 线程创建/销毁开销大
// ✗ 大量线程导致上下文切换频繁
// ✗ 内存占用高（每个线程栈约8MB）
```

#### 版本3：epoll + 线程池（生产级）

```c
// echo_server_v3.c - epoll + 线程池版本
// 编译: gcc echo_server_v3.c -o echo_v3 -lpthread

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <pthread.h>
#include <errno.h>
#include <fcntl.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/epoll.h>

#define PORT 8888
#define BUFFER_SIZE 1024
#define MAX_EVENTS 1000
#define THREAD_POOL_SIZE 8

// 设置非阻塞
int set_nonblocking(int fd) 
{
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags == -1) return -1;
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

// 任务队列
typedef struct Task 
{
    int client_fd;
    struct Task *next;
} Task;

typedef struct 
{
    Task *head;
    Task *tail;
    pthread_mutex_t lock;
    pthread_cond_t cond;
    int shutdown;
} TaskQueue;

TaskQueue task_queue;

// 初始化任务队列
void task_queue_init() 
{
    task_queue.head = task_queue.tail = NULL;
    pthread_mutex_init(&task_queue.lock, NULL);
    pthread_cond_init(&task_queue.cond, NULL);
    task_queue.shutdown = 0;
}

// 添加任务
void task_queue_push(int client_fd) 
{
    Task *task = malloc(sizeof(Task));
    task->client_fd = client_fd;
    task->next = NULL;
    
    pthread_mutex_lock(&task_queue.lock);
    
    if (task_queue.tail) 
    {
        task_queue.tail->next = task;
    } 
    else 
    {
        task_queue.head = task;
    }
    task_queue.tail = task;
    
    pthread_cond_signal(&task_queue.cond);  // 唤醒一个工作线程
    pthread_mutex_unlock(&task_queue.lock);
}

// 获取任务（阻塞）
Task *task_queue_pop() 
{
    pthread_mutex_lock(&task_queue.lock);
    
    while (task_queue.head == NULL && !task_queue.shutdown) 
    {
        pthread_cond_wait(&task_queue.cond, &task_queue.lock);
    }
    
    if (task_queue.shutdown) 
    {
        pthread_mutex_unlock(&task_queue.lock);
        return NULL;
    }
    
    Task *task = task_queue.head;
    task_queue.head = task->head->next;
    if (task_queue.head == NULL) 
    {
        task_queue.tail = NULL;
    }
    
    pthread_mutex_unlock(&task_queue.lock);
    return task;
}

// 工作线程
void *worker_thread(void *arg) 
{
    char buffer[BUFFER_SIZE];
    printf("Worker thread %lu started\n", pthread_self());
    while (1) 
    {
        Task *task = task_queue_pop();
        if (task == NULL) break;  // shutdown
        
        int fd = task->client_fd;
        free(task);
        
        // 处理这个客户端的数据（ET模式，需要循环读）
        while (1) 
        {
            int n = recv(fd, buffer, BUFFER_SIZE - 1, 0);           
            if (n > 0) 
            {
                buffer[n] = '\0';
                send(fd, buffer, n, 0);  // echo                
            } 
            else if (n == 0) 
            {
                // 连接关闭
                close(fd);
                break;              
            } 
            else 
            {
                if (errno == EAGAIN) 
                {
                    // ET模式：读完了
                    break;
                } 
                else 
                {
                    perror("recv");
                    close(fd);
                    break;
                }
            }
        }
    }
    
    printf("Worker thread %lu exiting\n", pthread_self());
    return NULL;
}

int main() 
{
    int listen_fd, epfd;
    struct sockaddr_in server_addr;
    struct epoll_event ev, events[MAX_EVENTS];
    
    // 1. 创建socket
    listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_fd < 0) 
    {
        perror("socket");
        exit(1);
    }
    
    // 2. SO_REUSEADDR
    int reuse = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));
    
    // 3. 绑定
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);
    
    if (bind(listen_fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) 
    {
        perror("bind");
        exit(1);
    }
    
    // 4. 监听
    if (listen(listen_fd, 128) < 0) 
    {
        perror("listen");
        exit(1);
    }
    
    // 5. 创建epoll
    epfd = epoll_create1(0);
    if (epfd < 0) 
    {
        perror("epoll_create1");
        exit(1);
    }
    
    // 6. 添加listen_fd到epoll
    ev.events = EPOLLIN;
    ev.data.fd = listen_fd;
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev) < 0) 
    {
        perror("epoll_ctl");
        exit(1);
    }
    
    // 7. 创建线程池
    task_queue_init();
    pthread_t workers[THREAD_POOL_SIZE];
    for (int i = 0; i < THREAD_POOL_SIZE; i++) 
    {
        pthread_create(&workers[i], NULL, worker_thread, NULL);
    }
    
    printf("Epoll + thread pool echo server listening on port %d...\n", PORT);
    printf("Thread pool size: %d\n", THREAD_POOL_SIZE);
    
    // 8. 主循环（epoll）
    while (1) 
    {
        int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        if (nfds < 0) 
        {
            if (errno == EINTR) continue;
            perror("epoll_wait");
            break;
        }
        
        for (int i = 0; i < nfds; i++) 
        {
            int fd = events[i].data.fd;            
            if (fd == listen_fd) 
            {
                // 新连接
                while (1) 
                {
                    struct sockaddr_in client_addr;
                    socklen_t len = sizeof(client_addr);
                    int client_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &len);                   
                    if (client_fd < 0) 
                    {
                        if (errno == EAGAIN) break;  // accept完了
                        perror("accept");
                        break;
                    }
                    
                    printf("New client: %s:%d (fd=%d)\n", inet_ntoa(client_addr.sin_addr), ntohs(client_addr.sin_port), client_fd);
                    
                    // 设置非阻塞
                    set_nonblocking(client_fd);
                    
                    // 添加到epoll（ET模式）
                    ev.events = EPOLLIN | EPOLLET;
                    ev.data.fd = client_fd;
                    epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);
                }      
            } 
            else 
            {
                // 客户端数据可读
                // 分发给线程池处理
                task_queue_push(fd);
            }
        }
    }
    
    // 清理
    task_queue.shutdown = 1;
    pthread_cond_broadcast(&task_queue.cond);
    
    for (int i = 0; i < THREAD_POOL_SIZE; i++) 
    {
        pthread_join(workers[i], NULL);
    }
    
    close(listen_fd);
    close(epfd);
    
    return 0;
}

// 优点：
// ✓ 高性能，支持数万并发连接
// ✓ 线程数量可控（不会无限增长）
// ✓ 内存占用低
// ✓ 生产级代码
//
// 复杂度：
// ✗ 代码较复杂
// ✗ 需要理解epoll、ET模式、线程池
```

**测试客户端**：

```c
// client.c - 测试客户端
// 编译: gcc client.c -o client

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define SERVER_IP "127.0.0.1"
#define SERVER_PORT 8888

int main() 
{
    int fd;
    struct sockaddr_in server_addr;
    char buffer[1024];
    
    fd = socket(AF_INET, SOCK_STREAM, 0);
    
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    inet_pton(AF_INET, SERVER_IP, &server_addr.sin_addr);
    
    if (connect(fd, (struct sockaddr*)&server_addr, sizeof(server_addr)) < 0) 
    {
        perror("connect");
        exit(1);
    }
    
    printf("Connected to server\n");
    printf("Type messages (Ctrl+D to exit):\n");
    
    while (fgets(buffer, sizeof(buffer), stdin) != NULL) 
    {
        send(fd, buffer, strlen(buffer), 0);
        int n = recv(fd, buffer, sizeof(buffer) - 1, 0);
        if (n > 0) 
        {
            buffer[n] = '\0';
            printf("Echo: %s", buffer);
        } 
        else 
        {
            printf("Server closed\n");
            break;
        }
    }
    
    close(fd);
    return 0;
}
```

### 11.2 TCP完整状态机（11个状态）

```
TCP状态转换图（完整版）

服务器端                                        客户端
─────────────────────────────────────────────────────────
                                              CLOSED
                                                │
                                                │ socket()
                                                ↓
                                              CLOSED
                                                │
                                                │ connect()
CLOSED                                          │ 发送SYN
  │                                             ↓
  │ socket()                                 SYN_SENT
  ↓                                             │
CLOSED                                          │ 收到SYN+ACK
  │                                             │ 发送ACK
  │ bind()                                      ↓
  ↓                                          ESTABLISHED
CLOSED                                          │
  │                                             │
  │ listen()                                    │
  ↓                                             │
LISTEN ←──────────────────────────────────────────────┐
  │                                             │     │
  │ 收到SYN                                      │     │
  │ 发送SYN+ACK                                  │     │
  ↓                                             │     │
SYN_RECV                                        │     │
  │                                             │     │
  │ 收到ACK                                      │     │
  ↓                                             │     │
ESTABLISHED ←─────────────────────────────────────────┘
  │                                             │
  │                                             │
  │ ┌─── 数据传输阶段 ───┐                        │
  │                                             │
  │                                             │
  │                                         close()
  │                                         发送FIN
  │ 收到FIN                                      ↓
  │ 发送ACK                                   FIN_WAIT_1
  ↓                                             │
CLOSE_WAIT                                      │ 收到ACK
  │                                             ↓
  │ 应用调用close()                           FIN_WAIT_2
  │ 发送FIN                                      │
  ↓                                             │ 收到FIN
LAST_ACK                                        │ 发送ACK
  │                                             ↓
  │ 收到ACK                                   TIME_WAIT
  ↓                                             │
CLOSED                                          │ 等待2MSL
                                                ↓
                                              CLOSED

特殊状态：
┌──────────────────────────────────────────────┐
│ CLOSING：双方同时close()的罕见情况              │
│ FIN_WAIT_1 → 收到FIN（而不是ACK） → CLOSING    │
│ CLOSING → 收到ACK → TIME_WAIT                 │
└──────────────────────────────────────────────┘
```

**11个状态详解**：

```
1. CLOSED：初始状态，没有连接

2. LISTEN：服务器监听状态
   - listen()后进入
   - 等待客户端连接

3. SYN_SENT：客户端发送SYN后
   - connect()后进入
   - 等待服务器SYN+ACK

4. SYN_RECV：服务器收到SYN后
   - 发送SYN+ACK后进入
   - 等待客户端ACK

5. ESTABLISHED：连接建立
   - 可以传输数据
   - 正常工作状态

6. FIN_WAIT_1：主动关闭，发送FIN后
   - close()后进入
   - 等待对方ACK

7. FIN_WAIT_2：收到对方ACK
   - 等待对方FIN
   - 半关闭状态（只能接收）

8. TIME_WAIT：收到对方FIN，发送ACK后
   - 等待2MSL（120秒）
   - 确保对方收到ACK

9. CLOSE_WAIT：被动关闭，收到FIN后
   - 发送ACK后进入
   - 等待应用程序close()
   - 【如果这个状态很多，说明应用有bug！】

10. LAST_ACK：被动关闭，发送FIN后
    - 等待对方ACK

11. CLOSING：双方同时close()（罕见）
    - 收到FIN但还没收到ACK
    - 发送ACK后进入TIME_WAIT
```

**状态查看命令**：

```bash
# 查看所有连接状态
netstat -tan | awk '{print $6}' | sort | uniq -c

# 输出示例：
#     523 ESTABLISHED
#       8 FIN_WAIT_2
#      12 LISTEN
#    8234 TIME_WAIT
#       5 CLOSE_WAIT  ← 如果这个数字很大，有bug！

# 查看特定状态的连接
ss -tan state established  # 只看ESTABLISHED
ss -tan state time-wait    # 只看TIME_WAIT
ss -tan state close-wait   # 只看CLOSE_WAIT
```

### 11.3 常见错误码处理

```c
// errno完整处理示例

#include <errno.h>
#include <string.h>

// recv()错误处理
ssize_t safe_recv(int fd, void *buf, size_t len, int flags) 
{
    while (1) 
    {
        ssize_t n = recv(fd, buf, len, flags);
        if (n >= 0) 
        {
            return n;  // 成功（n=0表示连接关闭）
        }
        
        // n < 0，检查errno
        switch (errno) 
        {
        case EINTR:
            // 被信号中断，重试
            continue;
            
        case EAGAIN:  // 等同于EWOULDBLOCK
            // 非阻塞IO，暂时没数据
            return -1;  // 调用者需要检查errno
            
        case ECONNRESET:
            // 连接被对方重置（收到RST）
            fprintf(stderr, "Connection reset by peer\n");
            return -1;
            
        case ETIMEDOUT:
            // 连接超时
            fprintf(stderr, "Connection timed out\n");
            return -1;
            
        case ENOTCONN:
            // socket未连接
            fprintf(stderr, "Socket is not connected\n");
            return -1;
            
        case EBADF:
            // 无效的文件描述符
            fprintf(stderr, "Bad file descriptor\n");
            return -1;
            
        default:
            // 其他错误
            perror("recv");
            return -1;
        }
    }
}

// send()错误处理
ssize_t safe_send(int fd, const void *buf, size_t len, int flags) 
{
    size_t sent = 0;
    while (sent < len) 
    {
        ssize_t n = send(fd, (char*)buf + sent, len - sent, flags);
        if (n > 0) 
        {
            sent += n;
            continue;
        }
        if (n == 0) 
        {
            // 不应该发生
            break;
        }
        
        // n < 0，检查errno
        switch (errno) 
        {
        case EINTR:
            // 被信号中断，重试
            continue;
            
        case EAGAIN:
            // 发送缓冲区满，稍后重试
            // 对于阻塞socket不应该发生
            usleep(1000);  // 等待1ms
            continue;
            
        case EPIPE:
            // 对方已关闭连接
            // 会触发SIGPIPE信号（如果没忽略）
            fprintf(stderr, "Broken pipe (peer closed)\n");
            return -1;
            
        case ECONNRESET:
            // 连接被重置
            fprintf(stderr, "Connection reset by peer\n");
            return -1;
            
        case EMSGSIZE:
            // 消息太大（UDP）
            fprintf(stderr, "Message too large\n");
            return -1;
            
        default:
            perror("send");
            return -1;
        }
    }
    
    return sent;
}

// connect()错误处理
int safe_connect(int fd, const struct sockaddr *addr, socklen_t addrlen) 
{
    if (connect(fd, addr, addrlen) == 0) 
    {
        return 0;  // 成功
    }
    
    switch (errno) 
    {
    case EINPROGRESS:
        // 非阻塞socket，连接正在进行中
        // 需要用select/poll/epoll等待可写
        return -1;  // 调用者需要等待
        
    case EISCONN:
        // 已经连接（不应该发生）
        return 0;
        
    case ETIMEDOUT:
        fprintf(stderr, "Connection timed out\n");
        return -1;
        
    case ECONNREFUSED:
        fprintf(stderr, "Connection refused (no server listening)\n");
        return -1;
        
    case ENETUNREACH:
        fprintf(stderr, "Network is unreachable\n");
        return -1;
        
    case EHOSTUNREACH:
        fprintf(stderr, "Host is unreachable\n");
        return -1;
        
    case EADDRINUSE:
        fprintf(stderr, "Address already in use\n");
        return -1;
        
    default:
        perror("connect");
        return -1;
    }
}

// accept()错误处理
int safe_accept(int listen_fd, struct sockaddr *addr, socklen_t *addrlen) 
{
    while (1) 
    {
        int client_fd = accept(listen_fd, addr, addrlen);
        if (client_fd >= 0) 
        {
            return client_fd;
        }
        
        switch (errno) 
        {
        case EINTR:
            // 被信号中断，重试
            continue;
            
        case EAGAIN:  // 等同于EWOULDBLOCK
            // 非阻塞listen_fd，暂时没有连接
            return -1;
            
        case EMFILE:
            // 进程打开文件太多
            fprintf(stderr, "Too many open files in process\n");
            return -1;
            
        case ENFILE:
            // 系统打开文件太多
            fprintf(stderr, "Too many open files in system\n");
            return -1;
            
        case ENOBUFS:
        case ENOMEM:
            // 内存不足
            fprintf(stderr, "Out of memory\n");
            return -1;
            
        default:
            perror("accept");
            return -1;
        }
    }
}

// 忽略SIGPIPE信号（推荐）
void ignore_sigpipe() 
{
    signal(SIGPIPE, SIG_IGN);
}

// 使用示例
int main() 
{
    ignore_sigpipe();  // 防止SIGPIPE导致程序崩溃
    int fd = socket(...);
    if (safe_connect(fd, ...) < 0) 
    {
        // 处理连接失败
    }
    char buf[1024];
    ssize_t n = safe_recv(fd, buf, sizeof(buf), 0);
    if (n < 0) 
    {
        if (errno == EAGAIN) 
        {
            // 暂时没数据，稍后重试
        } 
        else 
        {
            // 真正的错误
        }
    }
    
    safe_send(fd, "Hello", 5, 0);
}
```

**常见errno速查表**：

```
┌────────────────┬───────────────────────────────┐
│ errno          │ 含义                           │
├────────────────┼───────────────────────────────┤
│ EINTR          │ 被信号中断（重试）               │
│ EAGAIN         │ 非阻塞IO，暂时不可用（重试）      │
│ EWOULDBLOCK    │ 等同EAGAIN                    │
│ EINPROGRESS    │ 非阻塞connect正在进行           │
│ EISCONN        │ 已连接                        │
│ ECONNREFUSED   │ 连接被拒绝（无服务器监听）        │
│ ECONNRESET     │ 连接被重置（收到RST）            │
│ ETIMEDOUT      │ 连接超时                       │
│ EPIPE          │ 管道破裂（对方关闭）             │
│ EADDRINUSE     │ 地址已被占用                    │
│ EADDRNOTAVAIL  │ 地址不可用                      │
│ ENETUNREACH    │ 网络不可达                      │
│ EHOSTUNREACH   │ 主机不可达                      │
│ EMFILE         │ 进程打开文件太多                 │
│ ENFILE         │ 系统打开文件太多                 │
│ ENOBUFS        │ 缓冲区空间不足                   │
│ ENOMEM         │ 内存不足                        │
│ EBADF          │ 无效的文件描述符                 │
│ ENOTCONN       │ socket未连接                   │
│ EMSGSIZE       │ 消息太大（UDP）                 │
└────────────────┴───────────────────────────────┘
```

### 11.4 字节序问题

```c
// 大端 vs 小端

// 假设有32位整数 0x12345678

// 大端（Big-Endian）：高位字节在前
内存地址:  0x1000  0x1001  0x1002  0x1003
内存内容:   0x12    0x34    0x56    0x78
           ↑ 高位                  ↑ 低位

// 小端（Little-Endian）：低位字节在前
内存地址:  0x1000  0x1001  0x1002  0x1003
内存内容:   0x78    0x56    0x34    0x12
           ↑ 低位                  ↑ 高位

// 网络字节序 = 大端
// 大部分CPU（x86、ARM） = 小端

// 检测本机字节序
#include <stdio.h>

int main() 
{
    unsigned int x = 0x12345678;
    unsigned char *p = (unsigned char*)&x;
    
    if (p[0] == 0x12) 
    {
        printf("Big-Endian\n");
    } 
    else if (p[0] == 0x78) 
    {
        printf("Little-Endian\n");  // x86/x64通常是这个
    }
    
    return 0;
}

// 字节序转换函数

#include <arpa/inet.h>

// 主机字节序 → 网络字节序（发送前）
uint32_t htonl(uint32_t hostlong);    // 32位（long）
uint16_t htons(uint16_t hostshort);   // 16位（short）

// 网络字节序 → 主机字节序（接收后）
uint32_t ntohl(uint32_t netlong);
uint16_t ntohs(uint16_t netshort);

// h = host (主机)
// n = network (网络)
// s = short (16位)
// l = long (32位)

// 实际使用示例

// 发送端：发送端口号和消息长度
uint16_t port = 8888;
uint32_t msg_len = 1234;

uint16_t net_port = htons(port);          // 转网络字节序
uint32_t net_msg_len = htonl(msg_len);

send(fd, &net_port, sizeof(net_port), 0);
send(fd, &net_msg_len, sizeof(net_msg_len), 0);

// 接收端：接收端口号和消息长度
uint16_t net_port;
uint32_t net_msg_len;

recv(fd, &net_port, sizeof(net_port), 0);
recv(fd, &net_msg_len, sizeof(net_msg_len), 0);

uint16_t port = ntohs(net_port);          // 转主机字节序
uint32_t msg_len = ntohl(net_msg_len);

printf("Port: %u, Msg length: %u\n", port, msg_len);

// IP地址转换

// 点分十进制字符串 → 网络字节序二进制
const char *ip_str = "192.168.1.100";
struct in_addr addr;
inet_pton(AF_INET, ip_str, &addr);  // 推荐
// 或
inet_aton(ip_str, &addr);           // 旧版

// 网络字节序二进制 → 点分十进制字符串
char ip_str[INET_ADDRSTRLEN];
inet_ntop(AF_INET, &addr, ip_str, sizeof(ip_str));  // 推荐
// 或
char *str = inet_ntoa(addr);        // 旧版，不可重入

// 完整示例：发送自定义协议

#pragma pack(1)  // 按1字节对齐，避免padding
typedef struct 
{
    uint16_t version;    // 版本号
    uint16_t cmd;        // 命令
    uint32_t seq;        // 序列号
    uint32_t length;     // 数据长度
    char data[0];        // 柔性数组
} Protocol;
#pragma pack()

// 发送消息
void send_message(int fd, uint16_t cmd, uint32_t seq, const char *data, uint32_t data_len) 
{
    size_t total_len = sizeof(Protocol) + data_len;
    char *buf = malloc(total_len);
    Protocol *proto = (Protocol*)buf;
    proto->version = htons(1);           // ← 转网络字节序
    proto->cmd = htons(cmd);             // ← 转网络字节序
    proto->seq = htonl(seq);             // ← 转网络字节序
    proto->length = htonl(data_len);     // ← 转网络字节序
    memcpy(proto->data, data, data_len); 
    send(fd, buf, total_len, 0);
    free(buf);
}

// 接收消息
int recv_message(int fd, Protocol **out_proto) 
{
    // 1. 先接收头部
    Protocol header;
    if (recv(fd, &header, sizeof(Protocol), MSG_WAITALL) != sizeof(Protocol)) 
    {
        return -1;
    }
    
    // 2. 转换字节序
    header.version = ntohs(header.version);  // ← 转主机字节序
    header.cmd = ntohs(header.cmd);
    header.seq = ntohl(header.seq);
    header.length = ntohl(header.length);
    
    // 3. 接收数据部分
    size_t total_len = sizeof(Protocol) + header.length;
    Protocol *proto = malloc(total_len);
    memcpy(proto, &header, sizeof(Protocol));
    
    if (header.length > 0) 
    {
        if (recv(fd, proto->data, header.length, MSG_WAITALL) != header.length) 
        {
            free(proto);
            return -1;
        }
    }
    
    *out_proto = proto;
    return 0;
}

// 常见错误：

// 错误1：忘记转换
uint32_t len = 1234;
send(fd, &len, sizeof(len), 0);  // 错误！小端系统会发送 0xD2 0x04 0x00 0x00

// ✓ 正确：
uint32_t len = 1234;
uint32_t net_len = htonl(len);
send(fd, &net_len, sizeof(net_len), 0);  // 正确！发送 0x00 0x00 0x04 0xD2

// 错误2：重复转换
uint16_t port = htons(8888);
send(fd, &port, sizeof(port), 0);  // 已经转换过了
// 接收端：
uint16_t port;
recv(fd, &port, sizeof(port), 0);
port = ntohs(ntohs(port));  // 错误！转换了两次

// ✓ 正确：
uint16_t port = ntohs(port);  // 只转换一次

// 错误3：只转换部分字段
struct {
    uint32_t length;  // 需要转换
    char data[100];   // 字节数组不需要转换
} msg;

msg.length = htonl(100);  // ✓ 正确
// data不需要转换，因为是字节数组
```

### 11.5 调试工具实战

#### tcpdump抓包分析

```bash
# 1. 基础用法：抓取8888端口
sudo tcpdump -i eth0 port 8888

# 2. 详细信息 + 显示绝对序列号
sudo tcpdump -i eth0 port 8888 -nn -S -vv

# 参数说明：
# -i eth0: 指定网卡
# -nn: 不解析主机名和端口名
# -S: 显示绝对序列号（而不是相对）
# -vv: 非常详细的输出

# 3. 只抓SYN包（分析三次握手）
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0' port 8888

# 4. 保存到文件
sudo tcpdump -i eth0 port 8888 -w capture.pcap

# 5. 读取文件
tcpdump -r capture.pcap -nn -S

# 6. 过滤特定IP
sudo tcpdump -i eth0 host 192.168.1.100 and port 8888

# 7. 只看三次握手
sudo tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0' port 8888

# 实际输出分析：

# 第1次握手（SYN）
12:00:00.000000 IP 192.168.1.100.54321 > 192.168.1.1.8888: 
    Flags [S], seq 1000, win 65535, options [mss 1460], length 0
#   ↑ S = SYN
#      ↑ seq = 1000 (客户端ISN)
#           ↑ win = 65535 (接收窗口)
#                  ↑ mss = 1460 (最大段大小)

# 第2次握手（SYN+ACK）
12:00:00.000100 IP 192.168.1.1.8888 > 192.168.1.100.54321: 
    Flags [S.], seq 2000, ack 1001, win 65535, options [mss 1460], length 0
#   ↑ S. = SYN+ACK (点号表示ACK)
#       ↑ seq = 2000 (服务器ISN)
#            ↑ ack = 1001 (确认客户端seq+1)

# 第3次握手（ACK）
12:00:00.000200 IP 192.168.1.100.54321 > 192.168.1.1.8888: 
    Flags [.], seq 1001, ack 2001, win 65535, length 0
#   ↑ . = 纯ACK
#      ↑ seq = 1001 (使用服务器期望的值)
#           ↑ ack = 2001 (确认服务器seq+1)

# 8. 抓取四次挥手
sudo tcpdump -i eth0 'tcp[tcpflags] & (tcp-fin|tcp-rst) != 0' port 8888

# 9. 显示ASCII内容
sudo tcpdump -i eth0 port 8888 -A

# 10. 显示十六进制
sudo tcpdump -i eth0 port 8888 -X
```

#### netstat/ss查看连接

```bash
# netstat（传统工具）

# 1. 查看所有TCP连接
netstat -tan

# 参数：
# -t: TCP
# -a: 所有状态
# -n: 数字形式（不解析）

# 2. 查看监听端口
netstat -tln

# 3. 统计连接状态
netstat -tan | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'

# 输出：
# ESTABLISHED 523
# TIME_WAIT 8234
# CLOSE_WAIT 5
# FIN_WAIT_2 12

# 4. 查看程序占用的端口
netstat -tlnp | grep :8888

# ss（更快的替代品）

# 1. 查看所有TCP连接
ss -tan

# 2. 只看ESTABLISHED
ss -tan state established

# 3. 只看TIME_WAIT
ss -tan state time-wait

# 4. 只看LISTEN
ss -tln

# 5. 显示进程信息
ss -tlnp

# 6. 统计信息
ss -s

# 输出：
# Total: 9000 (kernel 9050)
# TCP:   8500 (estab 500, closed 8000, orphaned 0, synrecv 0, timewait 8000)

# 7. 按本地端口分组
ss -tan state time-wait | awk '{print $4}' | cut -d: -f2 | sort | uniq -c | sort -rn

# 8. 持续监控
watch -n 1 'ss -s'
```

#### strace跟踪系统调用

```bash
# 1. 跟踪程序的所有系统调用
strace ./echo_server

# 2. 只跟踪网络相关调用
strace -e trace=network ./echo_server

# 等价于：
strace -e socket,bind,listen,accept,connect,send,recv,sendto,recvfrom ./echo_server

# 3. 跟踪正在运行的进程
strace -p 1234  # PID=1234

# 4. 统计系统调用
strace -c ./echo_server

# 输出：
# % time     seconds  usecs/call     calls    errors syscall
# ------ ----------- ----------- --------- --------- ----------------
#  45.00    0.000900          45        20           read
#  35.00    0.000700          35        20           write
#  10.00    0.000200         100         2           accept
#   5.00    0.000100          50         2           epoll_wait

# 5. 保存到文件
strace -o trace.log ./echo_server

# 6. 显示时间戳
strace -t ./echo_server

# 7. 显示相对时间
strace -r ./echo_server

# 8. 只跟踪失败的调用
strace -Z ./echo_server

# 实际案例：查找为什么bind失败

$ strace -e bind ./echo_server
...
bind(3, {sa_family=AF_INET, sin_port=htons(8888), 
     sin_addr=inet_addr("0.0.0.0")}, 16) = -1 EADDRINUSE (Address already in use)
...

# 发现：EADDRINUSE，端口被占用
# 解决：kill占用的进程，或设置SO_REUSEADDR
```

#### lsof查看文件描述符

```bash
# 1. 查看进程打开的所有文件
lsof -p 1234  # PID=1234

# 2. 查看进程打开的socket
lsof -p 1234 -a -i

# 3. 查看谁在监听8888端口
lsof -i :8888

# 输出：
# COMMAND   PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
# echo_ser 1234 root    3u  IPv4  12345      0t0  TCP *:8888 (LISTEN)

# 4. 查看所有TCP连接
lsof -i TCP

# 5. 查看所有ESTABLISHED连接
lsof -i TCP -s TCP:ESTABLISHED

# 6. 查看文件描述符泄漏
# 正常情况：
# 0u (stdin)
# 1u (stdout)  
# 2u (stderr)
# 3u (listen socket)
# 4u, 5u, 6u... (客户端连接)

# 如果看到大量fd没有关闭 → 泄漏！

lsof -p 1234 | wc -l  # 统计fd数量

# 7. 持续监控
watch -n 1 'lsof -p 1234 | wc -l'

# 实际案例：查找CLOSE_WAIT泄漏

$ lsof -p 1234 -a -i | grep CLOSE_WAIT
echo_ser 1234 root   10u  IPv4  12350  TCP 192.168.1.1:8888->192.168.1.100:54321 (CLOSE_WAIT)
echo_ser 1234 root   11u  IPv4  12351  TCP 192.168.1.1:8888->192.168.1.100:54322 (CLOSE_WAIT)
...
# 发现大量CLOSE_WAIT → 应用没有close(fd)
```

### 11.6 性能测试

#### ab（Apache Bench）

```bash
# 安装
sudo apt-get install apache2-utils

# 基础用法
ab -n 10000 -c 100 http://localhost:8888/

# 参数：
# -n: 总请求数
# -c: 并发数

# 输出：
# Requests per second:    5000.00 [#/sec] (mean)  ← QPS
# Time per request:       20.000 [ms] (mean)      ← 平均延迟
# Transfer rate:          1000.00 [Kbytes/sec]

# 高级用法

# 1. 设置超时
ab -n 1000 -c 10 -s 30 http://localhost:8888/

# 2. 保持连接（HTTP Keep-Alive）
ab -n 1000 -c 10 -k http://localhost:8888/

# 3. POST请求
ab -n 1000 -c 10 -p data.txt -T 'application/json' http://localhost:8888/

# 4. 自定义Header
ab -n 1000 -c 10 -H "Authorization: Bearer token123" http://localhost:8888/
```

#### wrk（更强大）

```bash
# 安装
git clone https://github.com/wg/wrk.git
cd wrk && make && sudo cp wrk /usr/local/bin/

# 基础用法
wrk -t 4 -c 100 -d 30s http://localhost:8888/

# 参数：
# -t: 线程数
# -c: 连接数
# -d: 持续时间

# 输出：
# Running 30s test @ http://localhost:8888/
#   4 threads and 100 connections
#   Thread Stats   Avg      Stdev     Max   +/- Stdev
#     Latency     2.50ms    1.20ms  20.00ms   75.00%
#     Req/Sec    10.00k     500.00  11.00k    80.00%
#   1200000 requests in 30.00s, 200.00MB read
# Requests/sec:  40000.00  ← QPS
# Transfer/sec:    6.67MB

# 使用Lua脚本

# script.lua
wrk.method = "POST"
wrk.body   = '{"key":"value"}'
wrk.headers["Content-Type"] = "application/json"

# 运行
wrk -t 4 -c 100 -d 30s -s script.lua http://localhost:8888/
```

### 11.7 常见陷阱

```c
// 陷阱1：忘记检查返回值

send(fd, buf, len, 0);  // 可能失败！
recv(fd, buf, len, 0);  // 可能返回0或-1！

// 正确做法 ✓
if (send(fd, buf, len, 0) < 0) 
{
    perror("send");
    // 处理错误
}

ssize_t n = recv(fd, buf, len, 0);
if (n < 0) 
{
    // 错误
} 
else if (n == 0) 
{
    // 连接关闭
} 
else 
{
    // 成功
}

// 陷阱2：缓冲区太小

char buf[10];
recv(fd, buf, sizeof(buf), 0);  // 只能接收10字节

// 如果对方发送100字节，剩余90字节会：
// - TCP：留在内核缓冲区，下次recv读取 ✓
// - UDP：直接丢弃 ✗

// 正确做法 ✓
char buf[4096];  // 足够大

// 陷阱3：忘记设置SO_REUSEADDR ❌

bind(fd, ...);  // 如果端口TIME_WAIT，bind失败

// 正确做法 ✓
int reuse = 1;
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &reuse, sizeof(reuse));
bind(fd, ...);

// 陷阱4：ET模式忘记循环读取 ❌

// epoll ET模式
struct epoll_event ev;
ev.events = EPOLLIN | EPOLLET;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

// 错误：只读一次
char buf[1024];
recv(fd, buf, sizeof(buf), 0);  // 只读了1024字节
// 如果有2048字节，剩余1024字节不会再通知！

// 正确做法 ✓
while (1) 
{
    int n = recv(fd, buf, sizeof(buf), 0);
    if (n > 0) 
    {
        // 处理数据
    } 
    else if (n == 0) 
    {
        // 连接关闭
        break;
    } 
    else 
    {
        if (errno == EAGAIN) 
        {
            // 读完了
            break;
        }
        // 错误
        break;
    }
}

// 陷阱5：accept惊群

// 多进程同时accept同一个listen_fd
// 新连接到来 → 所有进程被唤醒 → 只有1个成功

// 解决1：SO_REUSEPORT（推荐）✓
int reuse = 1;
setsockopt(listen_fd, SOL_SOCKET, SO_REUSEPORT, &reuse, sizeof(reuse));
// 每个进程独立bind

// 解决2：EPOLLEXCLUSIVE ✓
ev.events = EPOLLIN | EPOLLEXCLUSIVE;

// 陷阱6：close而不是shutdown

send(fd, data, len, 0);
close(fd);  // 立即关闭，数据可能丢失！

// 正确做法 ✓
send(fd, data, len, 0);
shutdown(fd, SHUT_WR);  // 半关闭，确保数据发完
// 等待对方关闭
char buf[1];
recv(fd, buf, 1, 0);
close(fd);

// 陷阱7：忘记非阻塞connect需要等待

int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);

if (connect(fd, ...) < 0) 
{
    if (errno == EINPROGRESS) 
    {
        // 忘记等待！直接用fd会出错
    }
}

// 正确做法 ✓
if (connect(fd, ...) < 0) 
{
    if (errno == EINPROGRESS) 
    {
        // 等待可写
        fd_set wfds;
        FD_ZERO(&wfds);
        FD_SET(fd, &wfds);
        
        struct timeval timeout = {5, 0};
        if (select(fd + 1, NULL, &wfds, NULL, &timeout) > 0) 
        {
            // 检查连接是否成功
            int error;
            socklen_t len = sizeof(error);
            getsockopt(fd, SOL_SOCKET, SO_ERROR, &error, &len);
            
            if (error == 0) 
            {
                // 连接成功
            } 
            else 
            {
                // 连接失败
            }
        }
    }
}

// 陷阱8：UDP不检查返回地址

// 服务器
recvfrom(fd, buf, len, 0, NULL, NULL);  // 忘记记录客户端地址

// 无法回复！

// 正确做法 ✓
struct sockaddr_in client_addr;
socklen_t addr_len = sizeof(client_addr);

recvfrom(fd, buf, len, 0, 
         (struct sockaddr*)&client_addr, &addr_len);

// 回复
sendto(fd, reply, reply_len, 0,
       (struct sockaddr*)&client_addr, addr_len);

// 陷阱9：粘包处理错误

// 只read一次
char buf[1024];
int n = recv(fd, buf, sizeof(buf), 0);
// 可能收到2条消息粘在一起！

// 正确做法 ✓
// 使用长度前缀协议（见1.4节）

// 陷阱10：忘记初始化sockaddr

struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(8888);
// 忘记清零！addr.sin_addr和sin_zero可能有垃圾值

// 正确做法 ✓
struct sockaddr_in addr;
memset(&addr, 0, sizeof(addr));  // 先清零
addr.sin_family = AF_INET;
addr.sin_port = htons(8888);
addr.sin_addr.s_addr = INADDR_ANY;
```

### 11.8 真实案例

#### 案例1：CLOSE_WAIT泄漏排查

```
现象：
服务器运行几天后，性能下降
netstat发现大量CLOSE_WAIT状态

$ netstat -tan | grep CLOSE_WAIT | wc -l
5234  ← 5000多个CLOSE_WAIT！

排查步骤：

1. 确认是哪个进程
$ lsof -i | grep CLOSE_WAIT
myserver 1234 root   10u  TCP *:8888->*:* (CLOSE_WAIT)
myserver 1234 root   11u  TCP *:8888->*:* (CLOSE_WAIT)
...

2. 分析代码
void handle_client(int client_fd) 
{
    char buf[1024];
    while (1) 
    {
        int n = recv(client_fd, buf, sizeof(buf), 0);
        if (n <= 0) 
        {
            break;  // ← 退出循环，但没有close！
        }
        // 处理数据...
    }
    // BUG：忘记 close(client_fd);
}

3. 修复
void handle_client(int client_fd) 
{
    char buf[1024];
    while (1) 
    {
        int n = recv(client_fd, buf, sizeof(buf), 0);
        if (n <= 0) 
        {
            break;
        }
        // 处理数据...
    }
    close(client_fd);  // ← 添加这行
}

4. 验证
重启服务器，观察一段时间
$ watch -n 10 'netstat -tan | grep CLOSE_WAIT | wc -l'
0  ← 正常了
```

#### 案例2：TIME_WAIT过多导致端口耗尽

```
现象：
客户端频繁短连接
Cannot assign requested address

$ ss -tan state time-wait | wc -l
28232  ← 28000个TIME_WAIT！

$ cat /proc/sys/net/ipv4/ip_local_port_range
32768 60999  ← 可用端口：60999-32768=28231

端口耗尽！

解决方案：

1. 开启端口复用（临时缓解）
$ sudo sysctl -w net.ipv4.tcp_tw_reuse=1

2. 扩大端口范围
$ sudo sysctl -w net.ipv4.ip_local_port_range="10000 65000"
可用端口：55000个

3. 应用层改造（根本解决）
// 改用连接池（长连接）
Connection pool[100];

Connection *get_connection() 
{
    // 复用连接，不关闭
    for (int i = 0; i < 100; i++) 
    {
        if (!pool[i].in_use) 
        {
            pool[i].in_use = 1;
            return &pool[i];
        }
    }
    // 没有空闲，创建新的
    return create_new_connection();
}

void release_connection(Connection *conn) 
{
    conn->in_use = 0;  // 标记空闲，不close！
}

4. 验证
$ ss -tan state time-wait | wc -l
15  ← 大幅减少
```

#### 案例3：粘包导致的协议解析错误

```
现象：
客户端发送100条消息
服务器只收到95条

代码：
// 发送端
for (int i = 0; i < 100; i++) 
{
    char msg[100];
    snprintf(msg, sizeof(msg), "Message %d\n", i);
    send(fd, msg, strlen(msg), 0);
}

// 接收端（错误）
char buf[1024];
int n = recv(fd, buf, sizeof(buf), 0);
buf[n] = '\0';
printf("%s", buf);  // 可能收到多条消息粘在一起！

// 例如收到：
// "Message 0\nMessage 1\nMessage 2\nMessa"
// 第3条消息被截断了！

解决方案：
// 使用长度前缀协议

// 发送端
for (int i = 0; i < 100; i++) 
{
    char msg[100];
    int len = snprintf(msg, sizeof(msg), "Message %d", i);
    
    uint32_t net_len = htonl(len);
    send(fd, &net_len, sizeof(net_len), 0);  // 先发长度
    send(fd, msg, len, 0);                    // 再发内容
}

// 接收端
while (1) 
{
    // 1. 读取长度
    uint32_t net_len;
    if (recv(fd, &net_len, sizeof(net_len), MSG_WAITALL) != sizeof(net_len)) 
    {
        break;
    }
    
    uint32_t len = ntohl(net_len);
    
    // 2. 读取内容
    char *msg = malloc(len + 1);
    if (recv(fd, msg, len, MSG_WAITALL) != len) 
    {
        free(msg);
        break;
    }
    
    msg[len] = '\0';
    printf("%s\n", msg);
    free(msg);
}

验证：100条消息全部收到 ✓
```

---

## 附录A：网络编程常用系统调用速查表

```c
// ========== Socket基础 ==========

// 创建socket
int socket(int domain, int type, int protocol);
// domain: AF_INET(IPv4), AF_INET6(IPv6)
// type: SOCK_STREAM(TCP), SOCK_DGRAM(UDP), SOCK_RAW
// protocol: 通常填0
// 返回: socket文件描述符，失败返回-1

// 绑定地址
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
// 返回: 成功0，失败-1

// 监听
int listen(int sockfd, int backlog);
// backlog: ACCEPT队列最大长度，实际值=min(backlog, somaxconn)
// 返回: 成功0，失败-1

// 接受连接
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
// 返回: 新的连接socket，失败返回-1

// 连接服务器
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
// 返回: 成功0，失败-1
// 非阻塞socket会返回-1，errno=EINPROGRESS（需要等待）

// ========== 数据传输 ==========

// 发送数据（TCP）
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
ssize_t write(int sockfd, const void *buf, size_t count);
// flags: 通常填0，可选MSG_NOSIGNAL、MSG_DONTWAIT
// 返回: 实际发送字节数，失败返回-1

// 接收数据（TCP）
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t read(int sockfd, void *buf, size_t count);
// flags: 可选MSG_WAITALL、MSG_PEEK、MSG_DONTWAIT
// 返回: 实际接收字节数，0表示连接关闭，-1表示错误

// 发送数据（UDP）
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
               const struct sockaddr *dest_addr, socklen_t addrlen);
// 返回: 实际发送字节数，失败返回-1

// 接收数据（UDP）
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                 struct sockaddr *src_addr, socklen_t *addrlen);
// 返回: 实际接收字节数，失败返回-1

// ========== 连接管理 ==========

// 关闭socket
int close(int fd);
// 关闭读写两个方向，释放文件描述符
// 返回: 成功0，失败-1

// 半关闭
int shutdown(int sockfd, int how);
// how: SHUT_RD(关闭读), SHUT_WR(关闭写), SHUT_RDWR(关闭读写)
// 返回: 成功0，失败-1

// ========== Socket选项 ==========

// 设置socket选项
int setsockopt(int sockfd, int level, int optname,
               const void *optval, socklen_t optlen);
// level: SOL_SOCKET, IPPROTO_TCP, IPPROTO_IP
// 常用optname:
//   SOL_SOCKET: SO_REUSEADDR, SO_REUSEPORT, SO_KEEPALIVE, SO_RCVBUF, SO_SNDBUF
//   IPPROTO_TCP: TCP_NODELAY, TCP_KEEPIDLE, TCP_KEEPINTVL, TCP_KEEPCNT
// 返回: 成功0，失败-1

// 获取socket选项
int getsockopt(int sockfd, int level, int optname,
               void *optval, socklen_t *optlen);
// 返回: 成功0，失败-1

// ========== IO多路复用 ==========

// select
int select(int nfds, fd_set *readfds, fd_set *writefds,
           fd_set *exceptfds, struct timeval *timeout);
// 返回: 就绪的文件描述符数量，超时返回0，失败返回-1

// poll
int poll(struct pollfd *fds, nfds_t nfds, int timeout);
// 返回: 就绪的文件描述符数量，超时返回0，失败返回-1

// epoll_create
int epoll_create(int size);  // size参数已废弃
int epoll_create1(int flags);  // flags通常填0
// 返回: epoll文件描述符，失败返回-1

// epoll_ctl
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// op: EPOLL_CTL_ADD, EPOLL_CTL_MOD, EPOLL_CTL_DEL
// event->events: EPOLLIN, EPOLLOUT, EPOLLET, EPOLLONESHOT, EPOLLEXCLUSIVE
// 返回: 成功0，失败-1

// epoll_wait
int epoll_wait(int epfd, struct epoll_event *events,
               int maxevents, int timeout);
// 返回: 就绪的文件描述符数量，超时返回0，失败返回-1

// ========== 地址转换 ==========

// 点分十进制 → 网络字节序
int inet_pton(int af, const char *src, void *dst);
// af: AF_INET, AF_INET6
// 返回: 成功1，格式错误0，失败-1

// 网络字节序 → 点分十进制
const char *inet_ntop(int af, const void *src, char *dst, socklen_t size);
// 返回: 成功返回dst指针，失败返回NULL

// 主机名 → IP
struct hostent *gethostbyname(const char *name);  // 不可重入
int getaddrinfo(const char *node, const char *service,
                const struct addrinfo *hints,
                struct addrinfo **res);  // 推荐

// ========== 字节序转换 ==========

uint32_t htonl(uint32_t hostlong);   // 主机 → 网络（32位）
uint16_t htons(uint16_t hostshort);  // 主机 → 网络（16位）
uint32_t ntohl(uint32_t netlong);    // 网络 → 主机（32位）
uint16_t ntohs(uint16_t netshort);   // 网络 → 主机（16位）

// ========== 文件描述符操作 ==========

// 获取/设置文件描述符标志
int fcntl(int fd, int cmd, ... /* arg */ );
// 常用cmd:
//   F_GETFL: 获取状态标志
//   F_SETFL: 设置状态标志（O_NONBLOCK, O_ASYNC）
//   F_GETFD/F_SETFD: 获取/设置文件描述符标志（FD_CLOEXEC）

// 设置非阻塞
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);

// 或创建时设置
int fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);

// ========== 其他 ==========

// 获取socket错误
int error;
socklen_t len = sizeof(error);
getsockopt(fd, SOL_SOCKET, SO_ERROR, &error, &len);

// 获取对端地址
int getpeername(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

// 获取本地地址
int getsockname(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

// 零拷贝
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
ssize_t splice(int fd_in, loff_t *off_in, int fd_out,
               loff_t *off_out, size_t len, unsigned int flags);

// 内存映射
void *mmap(void *addr, size_t length, int prot, int flags,
           int fd, off_t offset);
int munmap(void *addr, size_t length);
```

---

## 附录B：面试高频问题快速回顾

### 🔥 必背的核心知识点

#### 1. 三次握手

```
客户端               服务器
  │ SYN, seq=x          │
  │ ────────────→       │  第1次：客户端发SYN
  │                     │
  │    SYN+ACK,         │
  │  seq=y, ack=x+1     │
  │ ←────────────       │  第2次：服务器回SYN+ACK
  │                     │
  │ ACK, seq=x+1,       │
  │   ack=y+1           │
  │ ────────────→       │  第3次：客户端发ACK
  │                     │
ESTABLISHED        ESTABLISHED

关键点：
- SYN占用1个序列号
- ack = seq + 1
- 为什么3次：防止旧连接
```

#### 2. 四次挥手

```
客户端               服务器
  │ FIN, seq=u          │
  │ ────────────→       │  第1次：客户端FIN
FIN_WAIT_1         CLOSE_WAIT
  │                     │
  │ ACK, ack=u+1        │
  │ ←────────────       │  第2次：服务器ACK
FIN_WAIT_2              │
  │                     │
  │ FIN, seq=w          │
  │ ←────────────       │  第3次：服务器FIN
TIME_WAIT          LAST_ACK
  │                     │
  │ ACK, ack=w+1        │
  │ ────────────→       │  第4次：客户端ACK
  │                 CLOSED
  │ (等待2MSL)          │
CLOSED

关键点：
- 为什么4次：半关闭
- TIME_WAIT: 2MSL=120秒
- CLOSE_WAIT多：应用bug
```

#### 3. 粘包/拆包解决方案

```
1. 固定长度 ★☆☆☆☆
   - 每条消息固定100字节
   - 浪费带宽

2. 特殊分隔符 ★★★☆☆
   - 用\n或\r\n分隔
   - 文本协议常用

3. 长度前缀 ★★★★★（推荐）
   - [4字节长度][消息内容]
   - 工业界标准

4. 应用层协议 ★★★★☆
   - HTTP、Protobuf
   - 跨语言
```

#### 4. TIME_WAIT优化

```
查看：
ss -tan state time-wait | wc -l

优化：
1. tcp_tw_reuse=1（推荐）
2. SO_REUSEADDR（服务器）
3. 连接池（根本解决）
4. 让客户端主动关闭
```

#### 5. epoll LT vs ET

```
LT（水平触发）：
- 只要有数据就通知
- 编程简单
- 默认模式

ET（边缘触发）：
- 状态变化才通知
- 必须循环读到EAGAIN
- 高性能
```

#### 6. TCP vs UDP

```
TCP：可靠、有序、连接
UDP：快速、无连接、可能丢包

选择：
- 文件传输、HTTP：TCP
- 视频直播、游戏：UDP
```

### 高频面试问答

**Q1: 为什么TIME_WAIT是2MSL？**
```
答：
1. 最后ACK可能丢失，服务器重传FIN（MSL）
   客户端重发ACK（MSL），总共2MSL
2. 保证旧连接包过期，不干扰新连接
```

**Q2: CLOSE_WAIT过多怎么办？**
```
答：应用程序bug，没调用close()
排查：lsof -p PID | grep CLOSE_WAIT
修复：确保每个accept的fd都close
```

**Q3: 如何检测连接断开？**
```
答：
1. recv()返回0：正常关闭（FIN）
2. recv()返回-1, errno=ECONNRESET：异常（RST）
3. 心跳超时：应用层心跳
4. TCP Keepalive：内核层
```

**Q4: listen的backlog设置多大？**
```
答：
- 低并发：128
- 中并发：512
- 高并发：1024-8192
实际值 = min(backlog, somaxconn)
```

**Q5: 粘包怎么解决？**
```
答：长度前缀（最常用）
格式：[4字节长度][消息内容]
发送：htonl(len) + data
接收：MSG_WAITALL读取len，再读取data
```

**Q6: epoll比select好在哪？**
```
答：
1. 无fd数量限制（select限制1024）
2. 不需要每次拷贝fd列表
3. 只返回就绪的fd，O(1)复杂度
4. 支持ET模式
```

**Q7: 非阻塞connect如何处理？**
```
答：
1. connect返回-1, errno=EINPROGRESS
2. 用select/poll/epoll等待可写
3. getsockopt(SOL_SOCKET, SO_ERROR)检查结果
```

**Q8: 如何实现高性能服务器？**
```
答：
1. IO模型：epoll ET模式
2. 线程模型：主线程+工作线程池
3. 零拷贝：sendfile
4. 系统调优：ulimit、tcp_tw_reuse
5. 连接池：复用连接
6. 负载均衡：SO_REUSEPORT
```

### 代码题必备

**Echo服务器（epoll版）伪代码**：

```c
// 1. 创建listen_fd
listen_fd = socket();
setsockopt(SO_REUSEADDR);
bind(listen_fd, port);
listen(listen_fd, backlog);

// 2. 创建epoll
epfd = epoll_create1(0);
epoll_ctl(EPOLL_CTL_ADD, listen_fd, EPOLLIN);

// 3. 主循环
while (1) 
{
    nfds = epoll_wait(epfd, events, MAX, -1);
    
    for (i = 0; i < nfds; i++) 
    {
        fd = events[i].data.fd;
        
        if (fd == listen_fd) 
        {
            // accept
            client_fd = accept(listen_fd, ...);
            set_nonblocking(client_fd);
            epoll_ctl(EPOLL_CTL_ADD, client_fd, EPOLLIN|EPOLLET);
        } 
        else 
        {
            // echo
            while ((n = recv(fd, buf, size, 0)) > 0) 
            {
                send(fd, buf, n, 0);
            }
            if (n == 0) close(fd);
            if (errno != EAGAIN) close(fd);
        }
    }
}
```

---

### 实践建议

**1. 循序渐进**
```
Week 1: 基础
- 理解三次握手、四次挥手
- 手写简单的echo服务器（阻塞IO）

Week 2: 进阶
- 学习epoll
- 手写epoll版echo服务器
- 压测（ab、wrk）

Week 3: 深入
- 解决粘包问题
- 实现长度前缀协议
- 心跳机制

Week 4: 实战
- 实现简单的聊天室
- 或HTTP服务器
- 或Redis协议客户端
```

**2. 动手实践**
```
✓ 手写代码（不要只看）
✓ 调试抓包（tcpdump、wireshark）
✓ 性能测试（ab、wrk）
✓ 阅读源码（Redis、Nginx）
✓ 模拟故障（拔网线、杀进程）
```

**3. 面试准备**
```
✓ 熟记本文档的核心知识点
✓ 能手写epoll服务器
✓ 能讲清楚粘包解决方案
✓ 能分析TIME_WAIT问题
✓ 准备1-2个实际项目经验
```

### 进阶主题（选读）

```
1. 零拷贝技术
   - sendfile、splice、mmap
   - DMA、Page Cache

2. 高性能网络编程
   - DPDK（绕过内核）
   - io_uring（新异步IO）
   - XDP（eBPF）

3. 协议实现
   - HTTP/1.1、HTTP/2
   - WebSocket
   - QUIC（HTTP/3）

4. 分布式系统
   - RPC框架（gRPC、Thrift）
   - 服务发现
   - 负载均衡

5. 网络安全
   - TLS/SSL
   - DDoS防御
   - 限流、降级
```

---

## 结语

面试前checklist：
- [ ] 能画出三次握手、四次挥手的seq/ack值
- [ ] 能手写epoll服务器（至少LT模式）
- [ ] 能讲清楚粘包的4种解决方案
- [ ] 能分析TIME_WAIT、CLOSE_WAIT问题
- [ ] 能解释LT vs ET的区别
- [ ] 能写出正确的错误处理代码
- [ ] 准备了1-2个实际项目经验

最后的建议：
1. **不要死记硬背**，理解原理
2. **一定要动手写代码**，光看不够
3. **用tcpdump抓包验证**，眼见为实
4. **阅读优秀源码**（Redis、Nginx）
5. **准备实际项目经验**，而不是只会demo

---


