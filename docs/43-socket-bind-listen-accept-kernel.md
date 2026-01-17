# socket、bind、listen、accept 内核实现详解

> 从内核角度深入理解 TCP 服务器的建立过程

---

## 目录

1. [概述](#1-概述)
2. [socket() - 创建套接字](#2-socket---创建套接字)
3. [bind() - 绑定地址](#3-bind---绑定地址)
4. [listen() - 开始监听](#4-listen---开始监听)
5. [accept() - 接受连接](#5-accept---接受连接)
6. [完整流程](#6-完整流程)
7. [内核数据结构](#7-内核数据结构)
8. [三次握手详解](#8-三次握手详解)

---

## 1. 概述

### 1.1 TCP 服务器的建立流程

```c
// 典型的 TCP 服务器代码
int main() 
{
    // 1. 创建 socket
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);

    // 2. 绑定地址和端口
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8888);
    addr.sin_addr.s_addr = INADDR_ANY;
    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));

    // 3. 开始监听
    listen(listen_fd, 128);

    // 4. 接受连接
    while (1) 
    {
        int client_fd = accept(listen_fd, NULL, NULL);
        // 处理连接...
    }
}
```

### 1.2 这四个函数在内核中做了什么？

```
┌─────────────────────────────────────────────────┐
│ socket()                                        │
├─────────────────────────────────────────────────┤
│ - 分配socket内核对象                              │
│ - 分配文件描述符                                  │
│ - 初始化socket状态为CLOSED                        │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│ bind()                                          │
├─────────────────────────────────────────────────┤
│ - 绑定IP地址和端口号                               │
│ - 加入到内核的端口哈希表                            │
│ - 状态仍为CLOSED                                 │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│ listen()                                        │
├─────────────────────────────────────────────────┤
│ - 创建accept队列（SYN队列和ACCEPT队列）            │
│ - 修改socket状态为LISTEN                         │
│ - 注册到内核监听哈希表                             │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│ accept()                                        │
├─────────────────────────────────────────────────┤
│ - 从ACCEPT队列取出已完成的连接                      │
│ - 创建新的socket对象                              │
│ - 分配新的文件描述符                               │
│ - 返回给应用程序                                  │
└─────────────────────────────────────────────────┘
```

### 1.3 重要概念：准备阶段 vs 工作阶段

**关键理解**：bind() 和 listen() **主要是做初始化和设置工作**，它们本身并不处理网络数据包。

#### 两个阶段的划分

```
准备阶段（设置数据结构）：
┌─────────────────────────────────────┐
│ socket()  - 创建内核对象             │
│ bind()    - 绑定地址，插入哈希表      │  ← 都是准备工作
│ listen()  - 创建队列，改状态          │  ← 不处理网络包
└─────────────────────────────────────┘
              ↓
真正干活的阶段（处理网络包）：
┌─────────────────────────────────────┐
│ 三次握手  - 内核协议栈自动处理        │
│   ├─ 收到 SYN：创建 request_sock     │  ← 这里才开始处理网络包
│   ├─ 发送 SYN+ACK                   │
│   └─ 收到 ACK：移到 ACCEPT 队列      │
│                                     │
│ accept()  - 从队列取连接             │
│                                     │
│ 数据传输  - 收发数据包               │
└─────────────────────────────────────┘
```

#### bind() 到底做了什么

```c
// 核心工作：建立"端口 → socket"的映射关系
int inet_bind(struct socket *sock, struct sockaddr *uaddr, int addr_len)
{
    // 1. 检查端口是否可用（扫描 bhash 哈希表）
    // 2. 分配端口号
    // 3. 把 socket 插入到 bhash[hash(port)]
    //    ↓
    //    这样当网络包到达时，内核能通过端口号快速找到对应的 socket
}
```

**为什么必须做这个准备**：
- 网络包到达时，内核只知道目标端口号（比如 8080）
- 内核需要通过 `bhash` 哈希表查找：端口 8080 → 哪个 socket？
- 如果没有 bind()，就没有这个映射关系，包就不知道给谁

#### listen() 到底做了什么

```c
// 核心工作：创建队列 + 改状态 + 注册到监听表
int inet_listen(struct socket *sock, int backlog)
{
    // 1. 创建 SYN 队列（大小: tcp_max_syn_backlog）
    // 2. 创建 ACCEPT 队列（大小: min(backlog, somaxconn)）
    // 3. 状态改为 TCP_LISTEN
    // 4. 插入到 listening_hash 哈希表
    //    ↓
    //    这样当 SYN 包到达时，内核知道这是一个监听 socket
}
```

**为什么必须做这个准备**：
- SYN 包到达时，内核需要知道这个 socket 是监听模式（而不是客户端）
- 需要有队列来存储半连接（SYN_RECV）和全连接（ESTABLISHED）
- 如果没有 listen()，内核收到 SYN 包会直接回 RST（拒绝连接）

#### 真正"干活"的是谁？

**内核协议栈的中断处理函数**（当网卡收到 SYN 包时）：

```c
tcp_v4_rcv()                    // 收包入口
  ↓
tcp_v4_do_rcv()
  ↓
tcp_rcv_state_process()
  ↓
tcp_v4_conn_request()          // 处理 SYN
  ↓
inet_csk_reqsk_queue_hash_add() // 放入 SYN 队列 ← 使用 listen() 创建的队列
  ↓
tcp_make_synack()              // 构造 SYN+ACK 包
  ↓
ip_build_and_send_pkt()        // 发送 SYN+ACK
```

#### 对比总结

```
bind() 和 listen()：
├─ 执行时机：应用程序主动调用（一次性）
├─ 工作内容：创建数据结构、设置状态、建立索引
└─ 是否处理网络包：否（仅准备工作）

三次握手处理：
├─ 执行时机：SYN 包到达时触发（每个连接都会执行）
├─ 工作内容：解析包、回包、状态机转换、放入队列
└─ 是否处理网络包：是（真正的网络通信）
```

#### 形象类比

```
socket() + bind() + listen() = 开一家餐厅
├─ socket()  : 租下店面
├─ bind()    : 挂上门牌号（让客人能找到）
└─ listen()  : 摆好桌椅、准备菜单（准备接客）

三次握手 = 客人进门、点菜、上菜
├─ SYN      : 客人敲门
├─ SYN+ACK  : 服务员说"请进"
└─ ACK      : 客人入座

accept() = 服务员带客人到座位

数据传输 = 客人吃饭
```

**结论**：bind() 和 listen() 确实"只是"做初始化和设置，但这些设置是**必不可少的基础设施**：
- bind() 建立了"端口→socket"的索引，让内核能路由网络包
- listen() 创建了连接队列，让三次握手有地方存放结果
- 真正处理网络包的是内核协议栈的中断处理函数（详见第8章）

---

## 2. socket() - 创建套接字

### 2.1 用户空间调用

```c
#include <sys/socket.h>

int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
//                     ↑        ↑            ↑
//                   协议族    socket类型    协议号
//                  (IPv4)     (流式/TCP)   (0=自动)
```

**参数说明**：

```
AF_INET:      IPv4 协议族
AF_INET6:     IPv6 协议族

SOCK_STREAM:  流式socket (TCP)
SOCK_DGRAM:   数据报socket (UDP)
SOCK_RAW:     原始socket

protocol:     协议号，通常填0表示自动选择
```

### 2.2 系统调用机制详解

**重要**：socket() 不是直接调用 __sys_socket()，而是通过**系统调用机制**间接调用。

#### 完整的调用链路

```
用户空间：
┌─────────────────────────────────────┐
│ 应用程序                             │
│   int fd = socket(AF_INET, ...)     │
└─────────────────────────────────────┘
         ↓ 调用
┌─────────────────────────────────────┐
│ glibc (C 标准库)                     │
│   socket() wrapper 函数              │
│   {                                 │
│       return syscall(__NR_socket,   │ ← 触发系统调用
│                      family,        │
│                      type,          │
│                      protocol);     │
│   }                                 │
└─────────────────────────────────────┘
         ↓ 系统调用 (syscall 指令)
─────────────────────────────────────────
         ↓ 从用户态切换到内核态
内核空间：
┌─────────────────────────────────────┐
│ 系统调用表                           │
│   sys_call_table[__NR_socket]       │
│     ↓                               │
│   sys_socket() (SYSCALL_DEFINE3生成) │
│   {                                 │
│       return __sys_socket(family,   │ ← 这里调用
│                          type,      │
│                          protocol); │
│   }                                 │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│ __sys_socket()                      │
│   {                                 │
│       sock_create(...);             │
│       sock_map_fd(...);             │
│       ...                           │
│   }                                 │
└─────────────────────────────────────┘
```

#### 用户空间的 socket() 函数

```c
// glibc: sysdeps/unix/sysv/linux/socket.c
int socket(int family, int type, int protocol)
{
    // 这是一个很薄的封装，直接触发系统调用
    return INLINE_SYSCALL(socket, 3, family, type, protocol);
}

// 底层实现（x86_64）
int socket(int family, int type, int protocol)
{
    long ret;

    // 通过 syscall 指令进入内核
    asm volatile (
        "syscall"
        : "=a" (ret)
        : "0" (__NR_socket),  // RAX: 系统调用号 (198)
          "D" (family),       // RDI: 第1个参数
          "S" (type),         // RSI: 第2个参数
          "d" (protocol)      // RDX: 第3个参数
        : "rcx", "r11", "memory"
    );

    return (int)ret;
}
```

**系统调用号**：

```c
// include/uapi/asm-generic/unistd.h
#define __NR_socket 198  // x86_64 上的系统调用号
```

#### 内核的系统调用表

```c
// 内核启动时建立的系统调用表
const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
    [0 ... __NR_syscall_max] = &sys_ni_syscall,

    // ...
    [__NR_socket] = sys_socket,  // 指向 SYSCALL_DEFINE3 生成的函数
    [__NR_bind]   = sys_bind,
    [__NR_listen] = sys_listen,
    [__NR_accept] = sys_accept4,
    // ...
};
```

#### SYSCALL_DEFINE3 宏展开

```c
// include/linux/syscalls.h
#define SYSCALL_DEFINE3(name, ...) \
    SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)

// 展开后生成：
asmlinkage long sys_socket(int family, int type, int protocol)
{
    long ret;

    // 参数检查、权限验证等
    // ...

    // 调用实际的实现函数
    ret = __sys_socket(family, type, protocol);  // ← 这里调用

    // 错误处理、审计日志等
    // ...

    return ret;
}
```

#### 为什么要分层？

```
┌──────────────────────────────────────────────┐
│ 应用层: socket() - glibc 提供                 │
├──────────────────────────────────────────────┤
│ 作用：                                        │
│  - 提供统一的 POSIX API                       │
│  - 隐藏系统调用细节                            │
│  - 跨平台兼容性                               │
└──────────────────────────────────────────────┘
                ↓ 系统调用
┌──────────────────────────────────────────────┐
│ 系统调用层: sys_socket() - 内核提供           │
├──────────────────────────────────────────────┤
│ 作用：                                        │
│  - 统一的入口点                               │
│  - 参数验证和边界检查                          │
│  - 权限检查（如需要 CAP_NET_RAW）             │
│  - 审计日志（SELinux, AppArmor）              │
└──────────────────────────────────────────────┘
                ↓
┌──────────────────────────────────────────────┐
│ 实现层: __sys_socket() - 内核实现             │
├──────────────────────────────────────────────┤
│ 作用：                                        │
│  - 真正的业务逻辑                             │
│  - 可以被内核其他地方复用                      │
│  - 避免重复的安全检查开销                      │
└──────────────────────────────────────────────┘
```

#### 系统调用的开销

```
用户态 → 内核态切换的开销：

1. 保存用户态 CPU 寄存器状态
2. 切换到内核栈
3. 查找系统调用表
4. 执行系统调用处理函数
5. 切换回用户栈
6. 恢复用户态 CPU 寄存器状态

开销：大约 50-100 个 CPU 周期

可以用 strace 观察：
$ strace -e socket ./your_program
socket(AF_INET, SOCK_STREAM, IPPROTO_IP) = 3
```

#### 简化的系统调用流程

```c
// 用户空间 (glibc)
socket(AF_INET, SOCK_STREAM, 0)
    ↓ syscall 指令
// ─── 用户态/内核态边界 ───
    ↓
// 内核空间
sys_socket(AF_INET, SOCK_STREAM, 0)  // SYSCALL_DEFINE3 生成
    ↓
__sys_socket(AF_INET, SOCK_STREAM, 0)  // 实际实现
```

### 2.3 内核实现

```c
// net/socket.c
int __sys_socket(int family, int type, int protocol)
{
    struct socket *sock;
    int retval;

    // 1. 分配socket内核对象
    retval = sock_create(family, type, protocol, &sock);
    if (retval < 0)
        return retval;

    // 2. 分配文件描述符，并关联socket
    retval = sock_map_fd(sock, 0);
    if (retval < 0)
        goto out_release;

    return retval;

out_release:
    sock_release(sock);
    return retval;
}
```

### 2.4 关键步骤详解

#### 步骤 1: 分配socket内核对象

```c
// net/socket.c
int sock_create(int family, int type, int protocol, struct socket **res)
{
    struct socket *sock;

    // 1.1 分配socket结构体
    sock = sock_alloc();
    sock->type = type;

    // 1.2 根据协议族调用对应的create函数
    // 对于AF_INET (IPv4)，调用inet_create()
    err = pf->create(net, sock, protocol, kern);
    if (err < 0)
        goto out_module_put;

    *res = sock;
    return 0;
}
```

**`struct socket` 数据结构**：

```c
// include/linux/net.h
struct socket 
{
    socket_state           state;      // socket状态 (SS_UNCONNECTED, SS_CONNECTED等)
    short                  type;       // socket类型 (SOCK_STREAM, SOCK_DGRAM)
    unsigned long          flags;
    struct file            *file;      // 关联的文件对象
    struct sock            *sk;        // 指向传输层sock结构
    const struct proto_ops *ops;       // 协议操作函数集
};
```

#### 步骤 2: 创建传输层sock结构

```c
// net/ipv4/af_inet.c
static int inet_create(struct net *net, struct socket *sock, int protocol)
{
    struct sock *sk;
    struct inet_protosw *answer;

    // 2.1 根据type和protocol找到协议处理函数
    // 对于SOCK_STREAM，找到TCP协议
    list_for_each_entry_rcu(answer, &inetsw[sock->type], list) 
    {
        if (protocol == answer->protocol) 
        {
            // 找到了TCP协议
            break;
        }
    }

    // 2.2 分配sock结构（TCP对应tcp_sock）
    sk = sk_alloc(net, PF_INET, GFP_KERNEL, answer->prot, kern);

    // 2.3 初始化sock
    sock_init_data(sock, sk);

    // 2.4 设置协议操作函数
    sock->ops = answer->ops;  // inet_stream_ops (TCP)

    // 2.5 调用协议的初始化函数
    if (sk->sk_prot->init) 
    {
        err = sk->sk_prot->init(sk);  // tcp_v4_init_sock()
    }

    return 0;
}
```

**`struct sock` 数据结构** (TCP的核心结构)：

```c
// include/net/sock.h
struct sock 
{
    // 接收队列
    struct sk_buff_head sk_receive_queue;
    // 发送队列
    struct sk_buff_head sk_write_queue;

    // 协议操作函数
    struct proto        *sk_prot;

    // socket 状态 (TCP_CLOSE, TCP_LISTEN, TCP_ESTABLISHED 等)
    unsigned char       sk_state;

    // 绑定的本地地址和端口
    __be32              sk_rcv_saddr;   // 本地IP
    __u16               sk_num;         // 本地端口

    // 远程地址和端口
    __be32              sk_daddr;       // 远程IP
    __be16              sk_dport;       // 远程端口

    // 接收缓冲区大小
    int                 sk_rcvbuf;
    // 发送缓冲区大小
    int                 sk_sndbuf;

    // ...
};
```

#### 步骤 3: 分配文件描述符

```c
// net/socket.c
static int sock_map_fd(struct socket *sock, int flags)
{
    struct file *newfile;
    int fd;

    // 3.1 从进程的文件描述符表中分配一个空闲的fd
    fd = get_unused_fd_flags(flags);
    if (unlikely(fd < 0)) 
    {
        return fd;
    }

    // 3.2 创建file对象
    newfile = sock_alloc_file(sock, flags, NULL);

    // 3.3 将file对象安装到fd
    fd_install(fd, newfile);

    return fd;
}
```

**文件描述符表**：

```
进程的文件描述符表：
┌────┬──────────────┐
│ 0  │ stdin        │
├────┼──────────────┤
│ 1  │ stdout       │
├────┼──────────────┤
│ 2  │ stderr       │
├────┼──────────────┤
│ 3  │ listen_fd ─────→ struct file ─→ struct socket ─→ struct sock
│    │              │                                   (TCP socket)
├────┼──────────────┤
│ 4  │ ...          │
└────┴──────────────┘
```

### 2.5 socket() 完成后的状态

```
内存中的对象关系：
┌─────────────────┐
│ 用户空间         │
│                 │
│ int listen_fd   │  ← 文件描述符 (比如 3)
└─────────────────┘
        ↓
┌──────────────────────────────┐
│ 内核空间                      │
│                              │
│ fd_table[3]                  │
│      ↓                       │
│ struct file                  │  ← 文件对象
│      ↓                       │
│ struct socket                │  ← socket 对象
│   state: SS_UNCONNECTED      │
│   type:  SOCK_STREAM         │
│   ops:   inet_stream_ops     │
│      ↓                       │
│ struct sock                  │  ← 传输层 socket (tcp_sock)
│   sk_state: TCP_CLOSE        │
│   sk_num:   0  (端口未绑定)    │
│   sk_rcv_saddr: 0  (IP未绑定) │
└──────────────────────────────┘
```

---

## 3. bind() - 绑定地址

### 3.1 用户空间调用

```c
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(8888);           // 端口 8888
addr.sin_addr.s_addr = INADDR_ANY;     // 0.0.0.0 (所有网卡)

int ret = bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
```

**地址结构**：

```c
struct sockaddr_in 
{
    sa_family_t    sin_family;  // AF_INET
    in_port_t      sin_port;    // 端口号 (网络字节序)
    struct in_addr sin_addr;    // IP 地址
    char           sin_zero[8]; // 填充字节
};

// INADDR_ANY = 0.0.0.0 表示监听所有网卡
```

### 3.2 系统调用入口

```c
// 用户空间
bind(listen_fd, &addr, sizeof(addr))
    ↓
// 系统调用
SYSCALL_DEFINE3(bind, int, fd, struct sockaddr __user *, umyaddr, int, addrlen)
    ↓
// 内核处理
__sys_bind(fd, umyaddr, addrlen)
```

### 3.3 内核实现

```c
// net/socket.c
int __sys_bind(int fd, struct sockaddr __user *umyaddr, int addrlen)
{
    struct socket *sock;
    struct sockaddr_storage address;
    int err;

    // 1. 根据fd找到socket对象
    sock = sockfd_lookup_light(fd, &err, &fput_needed);
    if (!sock)
        goto out;

    // 2. 从用户空间拷贝地址
    err = move_addr_to_kernel(umyaddr, addrlen, &address);
    if (err >= 0) 
    {
        // 3. 调用协议相关的bind函数
        // 对于TCP，调用inet_bind()
        err = sock->ops->bind(sock, (struct sockaddr *)&address, addrlen);
    }

    fput_light(sock->file, fput_needed);
out:
    return err;
}
```

### 3.4 TCP的bind实现

```c
// net/ipv4/af_inet.c
int inet_bind(struct socket *sock, struct sockaddr *uaddr, int addr_len)
{
    struct sock *sk = sock->sk;
    struct sockaddr_in *addr = (struct sockaddr_in *)uaddr;
    unsigned short snum;  // 端口号
    int err = 0;

    // 1. 检查地址族
    if (addr->sin_family != AF_INET)
        goto out;

    // 2. 获取端口号 (网络字节序转主机字节序)
    snum = ntohs(addr->sin_port);

    // 3. 检查端口是否可用
    // 如果端口已被占用，返回EADDRINUSE
    lock_sock(sk);

    // 4. 调用传输层的bind函数
    // 对于TCP，调用inet_csk_get_port()
    err = sk->sk_prot->get_port(sk, snum);
    if (err) 
    {
        inet->inet_saddr = 0;
        release_sock(sk);
        goto out;
    }

    // 5. 绑定成功，保存地址和端口
    inet->inet_saddr = addr->sin_addr.s_addr;  // 本地IP
    inet->inet_rcv_saddr = addr->sin_addr.s_addr;
    inet->inet_sport = htons(inet->inet_num);  // 本地端口

    release_sock(sk);
out:
    return err;
}
```

### 3.5 端口绑定详解

```c
// net/ipv4/inet_connection_sock.c
int inet_csk_get_port(struct sock *sk, unsigned short snum)
{
    struct inet_hashinfo *hashinfo = sk->sk_prot->h.hashinfo;
    struct inet_bind_hashbucket *head;
    struct inet_bind_bucket *tb = NULL;

    // 1. 如果端口号为 0，内核自动分配端口
    if (!snum) 
    {
        snum = inet_csk_find_open_port(sk);
        if (!snum)
            goto fail;
    }

    // 2. 检查端口是否可用
    head = &hashinfo->bhash[inet_bhashfn(net, snum, hashinfo->bhash_size)];
    spin_lock(&head->lock);

    // 2.1 遍历哈希桶，查找是否有相同端口
    inet_bind_bucket_for_each(tb, &head->chain) 
    {
        if (net_eq(ib_net(tb), net) && tb->port == snum) 
        {
            // 找到了相同端口
            if (!inet_csk_bind_conflict(sk, tb, true)) 
            {
                // 没有冲突，可以复用 (SO_REUSEADDR)
                goto tb_found;
            }
            goto fail_unlock;
        }
    }

    // 2.2 没有找到，创建新的bind bucket
    tb = inet_bind_bucket_create(hashinfo->bind_bucket_cachep, net, head, snum);
    if (!tb)
        goto fail_unlock;

tb_found:
    // 3. 绑定成功，加入哈希表
    inet_bind_hash(sk, tb, snum);

    spin_unlock(&head->lock);
    return 0;

fail_unlock:
    spin_unlock(&head->lock);
fail:
    return 1;
}
```

### 3.6 内核的端口哈希表

```
内核维护的端口绑定哈希表：
┌────────────────────────────────────────┐
│ inet_hashinfo.bhash[]                  │
├────────────────────────────────────────┤
│ [0] → NULL                             │
├────────────────────────────────────────┤
│ [1] → NULL                             │
├────────────────────────────────────────┤
│ ...                                    │
├────────────────────────────────────────┤
│ [hash(8888)] → inet_bind_bucket        │
│                  port: 8888            │
│                  owners: [sock1, ...]  │
├────────────────────────────────────────┤
│ ...                                    │
└────────────────────────────────────────┘

哈希函数：hash(port) = port % bhash_size
```

### 3.7 bind() 完成后的状态

```
内核对象状态：
┌─────────────────────────┐
│ struct sock             │
│   sk_state: TCP_CLOSE   │
│   sk_num:   8888        │    ← 已绑定端口
│   sk_rcv_saddr: 0.0.0.0 │    ← 已绑定IP (所有网卡)
└─────────────────────────┘
        ↓
┌───────────────────┐
│ 端口哈希表          │
│ bhash[hash(8888)] │ → inet_bind_bucket
│                   │     port: 8888
│                   │     sk: 指向上面的 sock
└───────────────────┘
```

---

## 4. listen() - 开始监听

### 4.1 用户空间调用

```c
int backlog = 128;  // 队列长度
int ret = listen(listen_fd, backlog);
```

**backlog参数**：

```
backlog的含义：
  - 旧版内核 (< 2.2)：SYN队列的最大长度
  - 新版内核 (>= 2.2)：完成连接队列 (ACCEPT队列) 的最大长度

实际值受系统参数限制：
  /proc/sys/net/core/somaxconn (默认128)
  /proc/sys/net/ipv4/tcp_max_syn_backlog (默认1024)
```

### 4.2 系统调用入口

```c
// 用户空间
listen(listen_fd, backlog)
    ↓
// 系统调用
SYSCALL_DEFINE2(listen, int, fd, int, backlog)
    ↓
// 内核处理
__sys_listen(fd, backlog)
```

### 4.3 内核实现

```c
// net/socket.c
int __sys_listen(int fd, int backlog)
{
    struct socket *sock;
    int err;

    // 1. 根据fd找到socket对象
    sock = sockfd_lookup_light(fd, &err, &fput_needed);
    if (sock) 
    {
        // 2. 限制backlog不超过系统最大值
        somaxconn = sock_net(sock->sk)->core.sysctl_somaxconn;
        if ((unsigned int)backlog > somaxconn)
            backlog = somaxconn;

        // 3. 调用协议相关的listen函数
        // 对于TCP，调用inet_listen()
        err = sock->ops->listen(sock, backlog);

        fput_light(sock->file, fput_needed);
    }
    return err;
}
```

### 4.4 TCP的listen实现

```c
// net/ipv4/af_inet.c
int inet_listen(struct socket *sock, int backlog)
{
    struct sock *sk = sock->sk;
    unsigned char old_state;
    int err;

    lock_sock(sk);

    err = -EINVAL;
    // 1. 检查socket类型和状态
    if (sock->state != SS_UNCONNECTED || sock->type != SOCK_STREAM)
        goto out;

    old_state = sk->sk_state;
    // 2. 只有TCP_CLOSE或TCP_LISTEN状态才能调用listen
    if (!((1 << old_state) & (TCPF_CLOSE | TCPF_LISTEN)))
        goto out;

    // 3. 如果已经是LISTEN状态，只更新backlog
    if (old_state != TCP_LISTEN) 
    {
        // 3.1 初始化accept队列
        err = inet_csk_listen_start(sk, backlog);
        if (err)
            goto out;
    }

    // 4. 更新backlog
    sk->sk_max_ack_backlog = backlog;
    err = 0;

out:
    release_sock(sk);
    return err;
}
```

### 4.5 初始化accept队列

```c
// net/ipv4/inet_connection_sock.c
int inet_csk_listen_start(struct sock *sk, int backlog)
{
    struct inet_connection_sock *icsk = inet_csk(sk);
    struct inet_sock *inet = inet_sk(sk);

    // 1. 初始化 accept 队列
    reqsk_queue_alloc(&icsk->icsk_accept_queue);

    // 2. 设置队列的最大长度
    icsk->icsk_accept_queue.listen_opt->max_qlen_log = backlog;

    // 3. 修改 socket 状态为 LISTEN
    sk->sk_state = TCP_LISTEN;

    // 4. 将 socket 加入到监听哈希表
    // 这样内核收到 SYN 包时能快速找到这个 socket
    if (!sk->sk_prot->get_port(sk, inet->inet_num)) {
        inet->inet_sport = htons(inet->inet_num);

        sk_dst_reset(sk);
        err = sk->sk_prot->hash(sk);  // 调用 inet_hash()

        if (likely(!err))
            return 0;
    }

    sk->sk_state = TCP_CLOSE;
    return err;
}
```

### 4.6 accept队列详解

**两个队列**：

```
内核为每个监听socket维护两个队列：

┌─────────────────────────────────────────────────┐
│ SYN队列 (半连接队列)                              │
├─────────────────────────────────────────────────┤
│ 存储收到SYN，但还未完成三次握手的连接                 │
│ 状态: SYN_RECV                                   │
│ 数据结构: request_sock                           │
│ 最大长度: tcp_max_syn_backlog                    │
│                                                 │
│ [request_sock] → [request_sock] → ...           │
│   client_ip:port      client_ip:port            │
│   state: SYN_RECV     state: SYN_RECV           │
└─────────────────────────────────────────────────┘
                    ↓ (收到ACK)
┌─────────────────────────────────────────────────┐
│ ACCEPT队列 (完成连接队列)                          │
├─────────────────────────────────────────────────┤
│ 存储已完成三次握手，等待accept()取走的连接            │
│ 状态: ESTABLISHED                                │
│ 数据结构: sock                                   │
│ 最大长度: backlog                                │
│                                                 │
│ [sock] → [sock] → [sock] → ...                  │
│   state: ESTABLISHED                            │
└─────────────────────────────────────────────────┘
                    ↓
            accept()取走
```

**队列结构**：

```c
// include/net/request_sock.h
struct request_sock_queue 
{
    struct request_sock     *rskq_accept_head;  // ACCEPT队列头
    struct request_sock     *rskq_accept_tail;  // ACCEPT队列尾
    rwlock_t                syn_wait_lock;
    u8                      rskq_defer_accept;
    struct listen_sock      *listen_opt;        // SYN队列
};

struct listen_sock 
{
    u8                      max_qlen_log;       // log2(backlog)
    u32                     nr_table_entries;   // SYN队列哈希表大小
    struct request_sock     *syn_table[0];      // SYN队列哈希表
};
```

**完成连接队列 (ACCEPT队列) 深入理解**：

```
为什么叫"完成连接队列"？

TCP三次握手已完成
连接状态是ESTABLISHED
可以收发数据了
但应用程序还没有通过accept()取走

所以：
  - 从内核角度：连接已完成
  - 从应用角度：还在等待中
```

**完整的工作流程**：

```
时间线：

t1: 客户端发起连接
    客户端 → [SYN] → 服务器
    │
    ├─ 服务器内核收到SYN
    ├─ 创建request_sock
    └─ 加入SYN队列 ← 半连接状态

t2: 服务器回复
    服务器 → [SYN+ACK] → 客户端

t3: 客户端确认
    客户端 → [ACK] → 服务器
    │
    ├─ 服务器内核收到ACK
    ├─ 三次握手完成！
    ├─ 从SYN队列移除request_sock
    ├─ 创建新的sock对象
    ├─ 状态设为ESTABLISHED
    └─ 加入ACCEPT队列 ← 完成连接队列！

t4: 应用程序调用accept()
    │
    ├─ 从ACCEPT队列取出sock
    ├─ 分配文件描述符
    └─ 返回给应用程序
```

**ACCEPT队列满了会怎样？**

```c
// 场景：高并发下，accept()处理不过来

ACCEPT队列状态：
┌──────────────────────────────────┐
│ 最大长度: backlog = 128           │
│ 当前长度: 128 (满了！)             │
│                                  │
│ [sock] → [sock] → ... → [sock]   │
│  (128个已完成的连接在等待)          │
└──────────────────────────────────┘

此时又有新的客户端完成三次握手：
  客户端 → [ACK] → 服务器
  │
  ├─ 内核发现ACCEPT队列已满
  │
  └─ 两种处理方式：

      方式1 (tcp_abort_on_overflow=0, 默认):
          ├─ 丢弃ACK，不回复
          ├─ 客户端会重传ACK
          └─ 更温和，给应用程序时间处理

      方式2 (tcp_abort_on_overflow=1):
          ├─ 发送RST拒绝连接
          ├─ 客户端立即收到"Connection refused"
          └─ 快速失败，避免客户端长时间等待
```

**系统参数控制**：

```bash
# 查看ACCEPT队列满时的行为
cat /proc/sys/net/ipv4/tcp_abort_on_overflow
# 0 = 丢弃 ACK，等待重传 (默认)
# 1 = 发送 RST，拒绝连接

# 修改backlog的最大值
cat /proc/sys/net/core/somaxconn
# 默认 128，可以调大
echo 1024 > /proc/sys/net/core/somaxconn

# 查看当前ACCEPT队列统计
ss -lnt
# Recv-Q: 当前ACCEPT队列长度
# Send-Q: ACCEPT 队列最大长度 (backlog)

# 示例输出：
# State   Recv-Q Send-Q Local Address:Port
# LISTEN  5      128    0.0.0.0:8888
#         ↑      ↑
#      当前5个  最大128个
```

**高并发场景的问题**：

```
假设每秒有1000个新连接：
  1. 每个连接完成三次握手后进入ACCEPT队列
  2. 如果accept()处理速度慢（每秒只处理100个）
  3. ACCEPT队列会快速积累：
     - 第1秒：100个在队列中
     - 第2秒：1000个在队列中
     - 第3秒：队列满(128)，开始丢弃连接
  4. 客户端开始出现连接失败或延迟

解决方案：
┌──────────────────────────────────────┐
│ 1. 增大backlog                        │
│    listen(fd, 1024);                 │
│    sysctl -w net.core.somaxconn=1024 │
├──────────────────────────────────────┤
│ 2. 加快accept()处理                   │
│    - 多线程/多进程accept               │
│    - 使用epoll ET模式循环accept        │
├──────────────────────────────────────┤
│ 3. 监控队列状态                        │
│    - watch -n1 'ss -lnt'             │
│    - 监控Recv-Q是否接近Send-Q          │
└──────────────────────────────────────┘
```

**代码示例**：

```c
// 示例：监控 ACCEPT 队列
void monitor_accept_queue(int listen_fd) 
{
    struct tcp_info info;
    socklen_t len = sizeof(info);

    if (getsockopt(listen_fd, IPPROTO_TCP, TCP_INFO, &info, &len) == 0) 
    {
        // info 包含队列信息
        printf("ACCEPT queue state: %u/%u\n",
               info.tcpi_unacked,    // 当前队列长度
               info.tcpi_sacked);     // 最大队列长度
    }
}

// 高并发服务器的accept循环
void high_performance_accept(int listen_fd) 
{
    while (1) 
    {
        // ET模式下，循环accept直到EAGAIN
        while (1) 
        {
            int client_fd = accept(listen_fd, NULL, NULL);

            if (client_fd < 0) 
            {
                if (errno == EAGAIN) 
                {
                    // ACCEPT队列空了
                    break;
                }
                perror("accept");
                break;
            }

            // 快速处理：分发给工作线程
            dispatch_to_worker(client_fd);
        }
    }
}
```

### 4.7 监听哈希表

```c
// net/ipv4/inet_hashtables.c
int __inet_hash(struct sock *sk, struct sock *osk)
{
    struct inet_hashinfo *hashinfo = sk->sk_prot->h.hashinfo;
    struct inet_listen_hashbucket *ilb;

    if (sk->sk_state != TCP_LISTEN) 
    {
        // 连接状态的socket，加入到ehash (established hash)
        __inet_hash_nolisten(sk, osk);
        return 0;
    }

    // 监听状态的socket，加入到lhash (listening hash)
    ilb = &hashinfo->listening_hash[inet_sk_listen_hashfn(sk)];

    spin_lock(&ilb->lock);
    __sk_nulls_add_node_rcu(sk, &ilb->head);
    sock_prot_inuse_add(sock_net(sk), sk->sk_prot, 1);
    spin_unlock(&ilb->lock);

    return 0;
}
```

**监听哈希表结构**：

```
内核的监听socket哈希表：
┌────────────────────────────────────────────────┐
│ inet_hashinfo.listening_hash[]                 │
├────────────────────────────────────────────────┤
│ [0] → NULL                                     │
├────────────────────────────────────────────────┤
│ [1] → sock (监听8888)                           │
│        sk_state: TCP_LISTEN                    │
│        sk_num: 8888                            │
│        icsk_accept_queue: {SYN队列, ACCEPT队列} │
├────────────────────────────────────────────────┤
│ [2] → sock (监听80) → sock (监听 8080)          │
├────────────────────────────────────────────────┤
│ ...                                            │
└────────────────────────────────────────────────┘

当收到SYN包时：
  1. 根据目标端口查找监听哈希表
  2. 找到对应的监听socket
  3. 将连接加入SYN队列
```

### 4.8 listen() 完成后的状态

```
内核对象状态：
┌────────────────────────────┐
│ struct sock                │
│   sk_state: TCP_LISTEN     │  ← 状态变为LISTEN
│   sk_num:   8888           │
│   sk_max_ack_backlog: 128  │  ← backlog
│      ↓                     │
│ struct inet_connection_sock│
│      ↓                     │
│ icsk_accept_queue:         │
│   ┌─────────────────────┐  │
│   │ SYN 队列 (空)        │  │
│   │ [空]                │  │
│   └─────────────────────┘  │
│   ┌─────────────────────┐  │
│   │ ACCEPT 队列 (空)     │  │
│   │ [空]                │  │
│   └─────────────────────┘  │
└────────────────────────────┘
        ↓
┌────────────────────────────┐
│ 监听哈希表                   │
│ listening_hash[hash(8888)] │ → 指向上面的sock
└────────────────────────────┘
```

---

## 5. accept() - 接受连接

### 5.1 用户空间调用

```c
struct sockaddr_in client_addr;
socklen_t addr_len = sizeof(client_addr);

int client_fd = accept(listen_fd, (struct sockaddr*)&client_addr, &addr_len);

// client_fd是新连接的socket
// client_addr包含客户端的IP和端口
```

**accept()的作用**：

```
从ACCEPT队列中取出一个已完成三次握手的连接：
  1. 创建新的socket对象
  2. 分配新的文件描述符
  3. 返回给应用程序
```

### 5.2 系统调用入口

```c
// 用户空间
accept(listen_fd, &client_addr, &addr_len)
    ↓
// 系统调用
SYSCALL_DEFINE3(accept, int, fd, struct sockaddr __user *, upeer_sockaddr, int __user *, upeer_addrlen)
    ↓
// 实际调用accept4
SYSCALL_DEFINE4(accept4, int, fd, struct sockaddr __user *, upeer_sockaddr, int __user *, upeer_addrlen, int, flags)
    ↓
// 内核处理
__sys_accept4(fd, upeer_sockaddr, upeer_addrlen, flags)
```

### 5.3 内核实现

```c
// net/socket.c
int __sys_accept4(int fd, struct sockaddr __user *upeer_sockaddr, int __user *upeer_addrlen, int flags)
{
    struct socket *sock, *newsock;
    struct file *newfile;
    int err, newfd;

    // 1. 根据fd找到监听socket
    sock = sockfd_lookup_light(fd, &err, &fput_needed);
    if (!sock)
        goto out;

    // 2. 创建新的socket对象
    newsock = sock_alloc();
    if (!newsock)
        goto out_put;

    newsock->type = sock->type;
    newsock->ops = sock->ops;

    // 3. 分配新的文件描述符
    newfd = get_unused_fd_flags(flags);
    if (unlikely(newfd < 0)) 
    {
        err = newfd;
        sock_release(newsock);
        goto out_put;
    }

    // 4. 创建新的file对象
    newfile = sock_alloc_file(newsock, flags, sock->sk->sk_prot_creator->name);

    // 5. 调用协议相关的accept函数
    // 对于TCP，调用inet_accept()
    err = sock->ops->accept(sock, newsock, sock->file->f_flags, false);
    if (err < 0)
        goto out_fd;

    // 6. 如果提供了地址参数，拷贝客户端地址到用户空间
    if (upeer_sockaddr) 
    {
        if (newsock->ops->getname(newsock, (struct sockaddr *)&address, &len, 2) < 0) 
        {
            err = -ECONNABORTED;
            goto out_fd;
        }
        err = move_addr_to_user(&address, len, upeer_sockaddr, upeer_addrlen);
        if (err < 0)
            goto out_fd;
    }

    // 7. 安装文件描述符
    fd_install(newfd, newfile);
    err = newfd;

out_put:
    fput_light(sock->file, fput_needed);
out:
    return err;

out_fd:
    fput(newfile);
    put_unused_fd(newfd);
    goto out_put;
}
```

### 5.4 TCP的accept实现

```c
// net/ipv4/af_inet.c
int inet_accept(struct socket *sock, struct socket *newsock, int flags, bool kern)
{
    struct sock *sk1 = sock->sk;
    int err = -EINVAL;

    // 1. 调用传输层的accept函数
    // 对于TCP，调用inet_csk_accept()
    struct sock *sk2 = sk1->sk_prot->accept(sk1, flags, &err, kern);

    if (!sk2)
        goto do_err;

    lock_sock(sk2);

    // 2. 设置新socket的状态
    sock_rps_record_flow(sk2);
    WARN_ON(!((1 << sk2->sk_state) &
              (TCPF_ESTABLISHED | TCPF_SYN_RECV |
               TCPF_CLOSE_WAIT | TCPF_CLOSE)));

    // 3. 关联新socket和newsock
    sock_graft(sk2, newsock);

    newsock->state = SS_CONNECTED;
    err = 0;
    release_sock(sk2);
do_err:
    return err;
}
```

### 5.5 从ACCEPT队列取出连接

```c
// net/ipv4/inet_connection_sock.c
struct sock *inet_csk_accept(struct sock *sk, int flags, int *err, bool kern)
{
    struct inet_connection_sock *icsk = inet_csk(sk);
    struct request_sock_queue *queue = &icsk->icsk_accept_queue;
    struct request_sock *req;
    struct sock *newsk;
    int error;

    lock_sock(sk);

    // 1. 检查是否是监听socket
    if (sk->sk_state != TCP_LISTEN) 
    {
        error = -EINVAL;
        goto out_err;
    }

    // 2. 从ACCEPT队列取出一个连接
    req = reqsk_queue_remove(queue, sk);
    if (!req) 
    {
        // 2.1 队列为空
        if (flags & O_NONBLOCK) 
        {
            // 非阻塞模式，立即返回EAGAIN
            error = -EAGAIN;
            goto out_err;
        }

        // 2.2 阻塞模式，等待连接到来
        error = inet_csk_wait_for_connect(sk, timeo);
        if (error)
            goto out_err;

        // 等待结束，重新尝试取出连接
        req = reqsk_queue_remove(queue, sk);
    }

    // 3. 取出成功，获取新的sock
    newsk = req->sk;

    // 4. 清理request_sock
    reqsk_put(req);

    release_sock(sk);
    return newsk;

out_err:
    newsk = NULL;
    *err = error;
    release_sock(sk);
    return newsk;
}
```

### 5.6 阻塞等待

```c
// net/ipv4/inet_connection_sock.c
static int inet_csk_wait_for_connect(struct sock *sk, long timeo)
{
    struct inet_connection_sock *icsk = inet_csk(sk);
    DEFINE_WAIT(wait);
    int err;

    for (;;) 
    {
        // 1. 将当前进程加入到 socket 的等待队列
        prepare_to_wait_exclusive(sk_sleep(sk), &wait, TASK_INTERRUPTIBLE);

        release_sock(sk);

        // 2. 如果 ACCEPT 队列为空，让出 CPU，进程睡眠
        if (reqsk_queue_empty(&icsk->icsk_accept_queue))
            timeo = schedule_timeout(timeo);

        lock_sock(sk);

        // 3. 被唤醒，检查条件
        err = 0;
        if (!reqsk_queue_empty(&icsk->icsk_accept_queue))
            break;  // ACCEPT 队列有连接了，退出循环

        err = -EINVAL;
        if (sk->sk_state != TCP_LISTEN)
            break;

        err = sock_intr_errno(timeo);
        if (signal_pending(current))
            break;  // 收到信号，退出

        err = -EAGAIN;
        if (!timeo)
            break;  // 超时，退出
    }

    finish_wait(sk_sleep(sk), &wait);
    return err;
}
```

**等待队列机制**：

```
当 ACCEPT 队列为空时：
┌─────────────────────────────────────────┐
│ 进程 A 调用 accept()                     │
│   ↓                                     │
│ ACCEPT 队列为空                          │
│   ↓                                     │
│ 进程 A 加入 socket 的等待队列              │
│   ↓                                     │
│ 进程 A 睡眠 (状态: TASK_INTERRUPTIBLE)    │
└─────────────────────────────────────────┘
                ↓ (等待...)
┌─────────────────────────────────────────┐
│ 客户端连接到来，完成三次握手                 │
│   ↓                                     │
│ 连接加入 ACCEPT 队列                      │
│   ↓                                     │
│ 内核唤醒等待队列中的进程                    │
│   ↓                                     │
│ 进程 A 被唤醒                             │
│   ↓                                     │
│ accept() 从队列取出连接，返回              │
└─────────────────────────────────────────┘
```

### 5.7 accept() 完成后的状态

```
监听socket(listen_fd = 3):
┌──────────────────────────────┐
│ struct sock                  │
│   sk_state: TCP_LISTEN       │
│   sk_num:   8888             │
│   icsk_accept_queue:         │
│     ACCEPT 队列: [少了一个连接] │
└──────────────────────────────┘

新连接socket(client_fd = 4):
┌─────────────────────────────┐
│ struct sock                 │
│   sk_state: TCP_ESTABLISHED │  ← 已建立连接
│   sk_num:   8888            │  ← 本地端口
│   sk_rcv_saddr: 192.168.1.1 │  ← 本地IP
│   sk_daddr: 192.168.1.100   │  ← 客户端IP
│   sk_dport: 54321           │  ← 客户端端口
└─────────────────────────────┘

文件描述符表：
┌────┬──────────────┐
│ 3  │ listen_fd ─────→ 监听socket
├────┼──────────────┤
│ 4  │ client_fd ─────→ 新连接socket ← accept()返回
└────┴──────────────┘
```

---

## 6. 完整流程

### 6.1 时间线

```
时刻t1: socket()
┌─────────────────────────────────────────┐
│ 用户空间: listen_fd = socket(...)        │
│                                         │
│ 内核空间:                                │
│   - 分配socket, sock对象                 │
│   - 分配fd = 3                           │
│   - sk_state: TCP_CLOSE                 │
└─────────────────────────────────────────┘

时刻t2: bind()
┌─────────────────────────────────────────┐
│ 用户空间: bind(listen_fd, addr, ...)     │
│                                         │
│ 内核空间:                                │
│   - sk_num: 8888                        │
│   - sk_rcv_saddr: 0.0.0.0               │
│   - 加入端口哈希表                        │
│   - sk_state: 仍为TCP_CLOSE              │
└─────────────────────────────────────────┘

时刻t3: listen()
┌─────────────────────────────────────────┐
│ 用户空间: listen(listen_fd, 128)         │
│                                         │
│ 内核空间:                                │
│   - 创建SYN队列和ACCEPT队列               │
│   - sk_state: TCP_LISTEN                │
│   - 加入监听哈希表                        │
│   - 现在可以接受连接了！                   │
└─────────────────────────────────────────┘

时刻 t4: 客户端发起连接 (三次握手)
┌─────────────────────────────────────────┐
│ 客户端: connect(server_ip, 8888)         │
│   ↓                                     │
│ 发送SYN包 → 服务器                        │
│   ↓                                     │
│ 内核收到SYN:                             │
│   1. 查找监听哈希表，找到listen_fd         │
│   2. 创建request_sock，加入SYN队列        │
│   3. 回复SYN+ACK                         │
│   ↓                                     │
│ 收到ACK ← 客户端                          │
│   ↓                                     │
│ 内核收到 ACK:                            │
│   1. 从SYN队列移除                       │
│   2. 创建新的sock，加入ACCEPT队列          │
│   3. 唤醒等待的accept()                   │
└─────────────────────────────────────────┘

时刻t5: accept()
┌─────────────────────────────────────────┐
│ 用户空间: client_fd = accept(listen_fd)  │
│                                         │
│ 内核空间:                                │
│   - 从ACCEPT队列取出连接                  │
│   - 创建新的socket对象                    │
│   - 分配fd = 4                           │
│   - sk_state: TCP_ESTABLISHED           │
│   - 返回client_fd = 4                    │
└─────────────────────────────────────────┘
```

### 6.2 完整的状态转换

```
socket 状态机：

  socket()
     ↓
┌──────────┐
│TCP_CLOSE │  ← 初始状态
└──────────┘
     ↓ bind()
┌──────────┐
│TCP_CLOSE │  ← 绑定地址后，状态不变
└──────────┘
     ↓ listen()
┌──────────┐
│TCP_LISTEN│  ← 进入监听状态，可以接受连接
└──────────┘
     ↓ (收到SYN)
┌──────────┐
│SYN_RECV  │  ← 半连接状态 (在SYN队列中)
└──────────┘
     ↓ (收到ACK，完成三次握手)
┌───────────────┐
│TCP_ESTABLISHED│  ← 完成连接 (在ACCEPT队列中)
└───────────────┘
     ↓ accept()
┌───────────────┐
│TCP_ESTABLISHED│  ← 新的socket，交给应用程序
└───────────────┘
```

### 6.3 内存布局

```
完整的内存对象关系：

用户空间：
┌─────────────────┐
│ listen_fd = 3   │
│ client_fd = 4   │
└─────────────────┘
        ↓
内核空间：
┌────────────────────────────────────────────────────┐
│ 文件描述符表 (fd_table)                              │
│                                                    │
│ [3] → file ─→ socket ─→ sock (监听 socket)          │
│                          sk_state: TCP_LISTEN      │
│                          sk_num: 8888              │
│                          icsk_accept_queue         │
│                                                    │
│ [4] → file ─→ socket ─→ sock (连接 socket)          │
│                          sk_state: TCP_ESTABLISHED │
│                          sk_num: 8888              │
│                          sk_daddr: 192.168.1.100   │
│                          sk_dport: 54321           │
└────────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────┐
│ 哈希表                                           │
│                                                 │
│ 端口哈希表 (bhash):                               │
│   [hash(8888)] → 监听 socket                     │
│                                                 │
│ 监听哈希表 (listening_hash):                      │
│   [hash(8888)] → 监听 socket                     │
│                                                 │
│ 连接哈希表 (ehash):                               │
│   [hash(本地:8888, 远程:54321)] → 连接 socket     │
└─────────────────────────────────────────────────┘
```

---

## 7. 内核数据结构

### 7.0 连接的本质：数据结构 + 状态 + 缓冲区

**核心概念**：从内核的角度看，所谓的"连接"本质上就是**一组数据结构 + 状态 + 缓冲区**。

#### 从不同层面看"连接"

```
应用层（程序员视角）：
├─ 一个文件描述符(fd)
└─ 可以read()/write()的对象

内核层（内核视角）：
├─ 一组数据结构（struct sock, struct tcp_sock）
├─ TCP状态（ESTABLISHED, CLOSE_WAIT等）
├─ 发送/接收缓冲区
├─ 定时器（重传、keepalive）
└─ 在哈希表中的索引

网络层（协议视角）：
└─ 四元组（源IP:端口 <-> 目标IP:端口）
```

#### 不同阶段使用不同的数据结构

内核为了节省内存，在连接的不同阶段使用不同大小的数据结构：

```c
// 1. 半连接（SYN_RECV状态）- 轻量级结构
struct request_sock 
{
    struct sock_common      skc;
    struct request_sock    *dl_next;         // 链表指针
    u16                     mss;             // MSS
    u8                      num_retrans;     // 重传次数
    u8                      syn_ack_timeout; // SYN+ACK 超时
    // ... 很精简，只存储握手需要的信息
};
// 大小：约128字节

// 2. 完整连接（ESTABLISHED 状态）- 完整结构
struct tcp_sock 
{
    struct inet_connection_sock inet_conn;
    // TCP状态
    u32 rcv_nxt;        // 下一个期望接收的序列号
    u32 snd_nxt;        // 下一个要发送的序列号
    u32 snd_una;        // 第一个未确认的序列号
    // 缓冲区
    struct sk_buff_head out_of_order_queue;  // 乱序队列
    struct sk_buff     *send_head;           // 发送队列头
    // 拥塞控制
    u32 snd_cwnd;       // 拥塞窗口
    u32 snd_ssthresh;   // 慢启动阈值
    // 定时器
    struct timer_list retransmit_timer;  // 重传定时器
    struct timer_list delack_timer;      // 延迟ACK定时器

    // ... 非常多的字段
};
// 大小：约1800+字节
```

#### 为什么要用轻量级的request_sock？

**防御SYN Flood攻击**：

```
SYN Flood攻击场景：
┌─────────────────────────────────────┐
│ 攻击者发送100万个SYN包                 │
└─────────────────────────────────────┘
         ↓
如果每个连接都用tcp_sock：
├─ 1,000,000 × 1800 字节 = 1.7 GB内存
└─ 服务器内存耗尽 ✗

使用request_sock：
├─ 1,000,000 × 128 字节 = 122 MB内存
└─ 还能扛住 ✓
```

#### "连接"的完整生命周期

```c
// t1: 收到SYN包
tcp_v4_conn_request()
{
    // 分配轻量级结构
    struct request_sock *req = reqsk_alloc();
    req->skc.state = TCP_SYN_RECV;

    // 放入SYN队列
    inet_csk_reqsk_queue_hash_add(sk, req);

    // 这时的"连接"只是一个128字节的结构体
}

// t2: 收到ACK，三次握手完成
tcp_v4_syn_recv_sock()
{
    // 升级！分配完整的 sock
    struct sock *newsk = tcp_create_openreq_child(sk, req);
    // newsk 是 1800+ 字节的完整结构

    // 初始化缓冲区、定时器等
    tcp_init_buffer_space(newsk);
    tcp_init_metrics(newsk);

    // 从SYN队列移除request_sock
    inet_csk_reqsk_queue_unlink(sk, req);

    // 放入ACCEPT队列
    inet_csk_reqsk_queue_add(sk, req, newsk);

    // 这时的"连接"变成了完整的tcp_sock
}

// t3: accept()取走连接
sys_accept4()
{
    // 从ACCEPT队列取出newsk
    struct sock *newsk = inet_csk_accept(sk);

    // 分配文件描述符
    int newfd = sock_map_fd(newsk);

    // 返回给应用程序
    return newfd;
    // 应用程序看到的是一个fd，内核里是tcp_sock
}
```

#### 一个ESTABLISHED连接包含什么

```
一个完整的连接在内核中包括：

1. 核心数据结构
   ├─ struct socket (VFS层)
   ├─ struct sock (通用socket层)
   └─ struct tcp_sock (TCP协议层)

2. 状态信息
   ├─ TCP状态（ESTABLISHED, FIN_WAIT_1等）
   ├─ 序列号（snd_nxt, rcv_nxt）
   └─ 窗口大小（rcv_wnd, snd_wnd）

3. 缓冲区
   ├─ 发送缓冲区（sk_write_queue）
   ├─ 接收缓冲区（sk_receive_queue）
   └─ 乱序队列（out_of_order_queue）

4. 定时器
   ├─ 重传定时器（retransmit_timer）
   ├─ 延迟ACK定时器（delack_timer）
   ├─ Keepalive定时器（keepalive_timer）
   └─ TIME_WAIT定时器（timewait_timer）

5. 拥塞控制状态
   ├─ 拥塞窗口（snd_cwnd）
   ├─ 慢启动阈值（snd_ssthresh）
   └─ RTT估计（srtt, mdev）

6. 哈希表索引
   ├─ 在ehash中的位置（通过四元组哈希）
   └─ 在bhash中的位置（通过端口号哈希）
```

#### 形象类比

```
"连接" 就像一份"客户档案"：

餐厅的客户：
├─ 档案袋（数据结构）
├─ 客户状态（等位/就餐中/结账）
├─ 订单列表（发送/接收缓冲区）
├─ 计时器（上菜时间、催单时间）
└─ 档案柜中的位置（哈希表索引）

内核的连接：
├─ tcp_sock（数据结构）
├─ TCP 状态（ESTABLISHED/CLOSE_WAIT）
├─ sk_buff 队列（发送/接收缓冲区）
├─ timer（重传定时器、keepalive定时器）
└─ ehash 索引（四元组哈希）
```

#### 总结

**"连接"从内核角度看就是数据结构**，但不仅仅是结构体，而是包括：

1. **数据结构**：存储连接的所有信息
2. **状态机**：TCP 状态（ESTABLISHED, FIN_WAIT_1 等）
3. **缓冲区**：待发送和已接收的数据
4. **定时器**：各种超时和重传机制
5. **索引**：在哈希表中的位置，用于快速查找

**关键点**：
- 半连接用轻量级的 `request_sock`（128 字节）
- 完整连接用完整的 `tcp_sock`（1800+ 字节）
- 应用程序看到的是 `fd`，内核管理的是这一整套数据结构
- "建立连接"本质上就是在内核中创建这些数据结构并初始化状态

---

### 7.1 核心数据结构关系

```c
// 三层结构：
// 1. struct file (VFS 层)
// 2. struct socket (socket 层)
// 3. struct sock (传输层)

┌──────────────────────────────────────┐
│ struct file                          │  ← VFS 层
│   f_op: socket_file_ops              │
│   private_data ─┐                    │
└─────────────────┼────────────────────┘
                  ↓
┌──────────────────────────────────────┐
│ struct socket                        │  ← socket 层
│   state: SS_CONNECTED                │
│   type:  SOCK_STREAM                 │
│   ops:   inet_stream_ops  ←┐         │
│   sk ─┐                    │         │
└───────┼────────────────────┼─────────┘
        ↓                    │
┌──────────────────────────────────────┐
│ struct sock (或 struct tcp_sock)     │  ← 传输层
│   sk_state: TCP_ESTABLISHED          │
│   sk_prot: tcp_prot  ←───────────────┘
│   sk_receive_queue                   │
│   sk_write_queue                     │
└──────────────────────────────────────┘
```

### 7.2 操作函数集

```c
// socket 层操作函数
const struct proto_ops inet_stream_ops = {
    .family        = PF_INET,
    .bind          = inet_bind,
    .listen        = inet_listen,
    .accept        = inet_accept,
    .connect       = inet_stream_connect,
    .socketpair    = sock_no_socketpair,
    .sendmsg       = inet_sendmsg,
    .recvmsg       = inet_recvmsg,
    // ...
};

// 传输层操作函数 (TCP)
struct proto tcp_prot = {
    .name              = "TCP",
    .close             = tcp_close,
    .connect           = tcp_v4_connect,
    .disconnect        = tcp_disconnect,
    .accept            = inet_csk_accept,
    .ioctl             = tcp_ioctl,
    .init              = tcp_v4_init_sock,
    .destroy           = tcp_v4_destroy_sock,
    .shutdown          = tcp_shutdown,
    .setsockopt        = tcp_setsockopt,
    .getsockopt        = tcp_getsockopt,
    .sendmsg           = tcp_sendmsg,
    .recvmsg           = tcp_recvmsg,
    .get_port          = inet_csk_get_port,
    .hash              = inet_hash,
    // ...
};
```

### 7.3 完整的 TCP socket 结构

```c
// 实际上 TCP socket 使用 tcp_sock，它包含了所有层次的结构
struct tcp_sock {
    struct inet_connection_sock inet_conn;  // 包含 inet_sock
    // TCP 特有的字段
    u32     rcv_nxt;        // 下一个期望接收的序号
    u32     snd_nxt;        // 下一个要发送的序号
    u32     snd_una;        // 第一个未确认的序号
    u32     snd_wnd;        // 发送窗口
    u32     rcv_wnd;        // 接收窗口
    // ...
};

struct inet_connection_sock {
    struct inet_sock icsk_inet;  // 包含 sock
    // 连接管理相关
    struct request_sock_queue icsk_accept_queue;  // accept 队列
    // ...
};

struct inet_sock {
    struct sock sk;  // 基础 sock
    // IP 层相关
    __be32 inet_saddr;   // 本地 IP
    __be16 inet_sport;   // 本地端口
    __be32 inet_daddr;   // 远程 IP
    __be16 inet_dport;   // 远程端口
    // ...
};

struct sock {
    // socket 层指针
    struct socket *sk_socket;

    // 协议操作函数
    struct proto *sk_prot;

    // 队列
    struct sk_buff_head sk_receive_queue;
    struct sk_buff_head sk_write_queue;

    // 状态
    unsigned char sk_state;

    // 缓冲区大小
    int sk_rcvbuf;
    int sk_sndbuf;

    // ...
};
```

---

## 8. 三次握手详解

### 8.0 三次握手基础：SYN、ACK、seq、ack详解

#### TCP报文的关键字段

```
TCP报文头部（20字节）：
┌────────────────────────────────────────┐
│ 源端口 (16位) │ 目标端口 (16位)        │
├────────────────────────────────────────┤
│ 序列号 seq (32位)                       │  ← 本次发送数据的序列号
├────────────────────────────────────────┤
│ 确认号 ack (32位)                       │  ← 期望对方下次发送的序列号
├────────────────────────────────────────┤
│ 头部长度 │ 保留 │ 标志位 │ 窗口大小    │
│          │      │ UAPRSF │             │
│          │      │ RCSSYI │             │
│          │      │ GKHTNN │             │
│          │      │  └─ SYN: 同步标志    │
│          │      │    └─ ACK: 确认标志  │
├────────────────────────────────────────┤
│ 校验和            │ 紧急指针            │
├────────────────────────────────────────┤
│ 选项（可选）                            │
└────────────────────────────────────────┘
```

**核心字段说明**：
- **SYN标志位**：同步标志，值为0或1，用于建立连接
- **ACK标志位**：确认标志，值为0或1，表示ack字段有效
- **seq（序列号）**：本次发送数据的起始序列号
- **ack（确认号）**：期望对方下次发送的序列号（只有ACK=1时有效）

#### 完整的三次握手流程（详细版）

```
客户端                                    服务器
(CLOSED)                                (LISTEN)
   │                                        │
   │  [第1次握手] SYN                        │
   │  ─────────────────────────────────────→│
   │  TCP标志: SYN=1, ACK=0                  │
   │  seq = 1000 (客户端随机生成的ISN)        │
   │  ack = 0 (ACK=0时此字段无意义)           │
   │  数据长度: 0                             │
   │                                        │
(SYN_SENT)                            (SYN_RECV)
   │                                        │
   │            [第2次握手] SYN+ACK           │
   │  ←─────────────────────────────────────│
   │  TCP标志: SYN=1, ACK=1                  │
   │  seq = 2000 (服务器随机生成的ISN)        │
   │  ack = 1001 (确认客户端SYN: 1000+1)     │
   │  数据长度: 0                             │
   │                                        │
   │  [第3次握手] ACK                        │
   │  ─────────────────────────────────────→│
   │  TCP标志: SYN=0, ACK=1                  │
   │  seq = 1001 (使用服务器期望的值)         │
   │  ack = 2001 (确认服务器SYN: 2000+1)     │
   │  数据长度: 0 (也可以携带数据)            │
   │                                        │
(ESTABLISHED)                        (ESTABLISHED)
```

**ISN (Initial Sequence Number)**: 初始序列号，每次建立连接时随机生成，防止旧连接的数据干扰新连接。

#### 第1次握手详解

```
客户端 → 服务器
┌─────────────────────────────────────────┐
│ TCP 报文头部：                           │
├─────────────────────────────────────────┤
│ 源端口: 54321                            │
│ 目标端口: 8888                           │
├─────────────────────────────────────────┤
│ 序列号 (seq):                            │
│   seq = 1000                            │
│   ↑ 客户端随机选择的初始序列号 (ISN)       │
│   用于标识后续发送的数据                  │
├─────────────────────────────────────────┤
│ 确认号 (ack):                            │
│   ack = 0                               │
│   ↑ 因为ACK=0，这个字段无意义              │
├─────────────────────────────────────────┤
│ 标志位：                                 │
│   SYN = 1    ← 请求建立连接               │
│   ACK = 0    ← 没有确认信息               │
│   FIN = 0                               │
│   RST = 0                               │
│   PSH = 0                               │
│   URG = 0                               │
├─────────────────────────────────────────┤
│ 窗口大小: 65535 (接收窗口)               │
│ 选项: MSS=1460, SACK_PERM, ...          │
│ 数据长度: 0 (SYN包不携带数据)            │
└─────────────────────────────────────────┘

客户端状态变化: CLOSED → SYN_SENT
服务器收到SYN后: LISTEN → SYN_RECV
```

**关键点**：
- SYN占用1个序列号（即使没有数据）
- seq=1000表示：我的初始序列号是1000
- 下次发送数据将从seq=1001开始

**内核操作**（服务器端）：
```c
tcp_v4_conn_request()
{
    // 1. 分配轻量级半连接结构
    struct request_sock *req = reqsk_alloc();

    // 2. 保存客户端信息
    req->rcv_isn = 1000;  // 保存客户端的ISN
    req->state = TCP_SYN_RECV;
    req->client_ip = 192.168.1.100;
    req->client_port = 54321;

    // 3. 放入 SYN 队列（半连接队列）
    inet_csk_reqsk_queue_hash_add(sk, req);

    // 4. 生成服务器的ISN
    u32 server_isn = secure_tcp_sequence_number();  // 比如 2000

    // 5. 构造并发送 SYN+ACK
    tcp_make_synack(sk, req, server_isn);
    // SYN=1, ACK=1, seq=2000, ack=1001
}
```

#### 第2次握手详解

```
服务器 → 客户端
┌─────────────────────────────────────────┐
│ TCP 报文头部：                           │
├─────────────────────────────────────────┤
│ 源端口: 8888                             │
│ 目标端口: 54321                          │
├─────────────────────────────────────────┤
│ 序列号 (seq):                            │
│   seq = 2000                            │
│   ↑ 服务器随机选择的初始序列号 (ISN)       │
├─────────────────────────────────────────┤
│ 确认号 (ack):                            │
│   ack = 1001                            │
│   ↑ 计算: 客户端seq + 1 = 1000 + 1       │
│   含义: "我收到了你的1000，期望你下次从1001开始" │
├─────────────────────────────────────────┤
│ 标志位：                                 │
│   SYN = 1    ← 服务器也要建立连接          │
│   ACK = 1    ← 确认收到客户端的SYN         │
│   FIN = 0                               │
│   RST = 0                               │
├─────────────────────────────────────────┤
│ 窗口大小: 65535                          │
│ 选项: MSS=1460, ...                     │
│ 数据长度: 0                              │
└─────────────────────────────────────────┘

服务器状态: SYN_RECV (等待客户端ACK)
客户端收到后: SYN_SENT → ESTABLISHED
```

**为什么 ack = seq + 1？**
```
理解确认号的计算：

1. SYN标志占用1个序列号
   - 客户端的SYN包: seq=1000
   - SYN占用序列号1000
   - 下一个可用序列号是1001

2. 服务器用 ack=1001 告诉客户端：
   "我已经收到了序列号1000的SYN，
    现在期望你从序列号1001开始发送数据"

3. 同理，服务器的SYN也占用1个序列号：
   - 服务器的SYN包: seq=2000
   - SYN占用序列号2000
   - 下一个可用序列号是2001
```

#### 第3次握手详解

```
客户端 → 服务器
┌─────────────────────────────────────────┐
│ TCP 报文头部：                           │
├─────────────────────────────────────────┤
│ 源端口: 54321                            │
│ 目标端口: 8888                           │
├─────────────────────────────────────────┤
│ 序列号 (seq):                            │
│   seq = 1001                            │
│   ↑ 使用服务器期望的序列号                 │
│   (就是第2次握手中服务器ack的值)           │
├─────────────────────────────────────────┤
│ 确认号 (ack):                            │
│   ack = 2001                            │
│   ↑ 计算: 服务器seq + 1 = 2000 + 1       │
│   含义: "我收到了你的2000，期望你下次从2001开始" │
├─────────────────────────────────────────┤
│ 标志位：                                 │
│   SYN = 0    ← 不再是SYN包                │
│   ACK = 1    ← 确认收到服务器的SYN+ACK     │
│   FIN = 0                               │
├─────────────────────────────────────────┤
│ 窗口大小: 65535                          │
│ 数据长度: 0 (可以携带数据，称为"TFO")      │
└─────────────────────────────────────────┘

客户端状态: ESTABLISHED
服务器收到后: SYN_RECV → ESTABLISHED
```

**内核操作**（服务器端）：
```c
tcp_v4_syn_recv_sock()
{
    // 1. 验证ACK
    if (th->ack != 2001) {
        // ACK错误，发送RST
        tcp_send_reset(sk);
        return NULL;
    }

    // 2. 从 SYN 队列移除 request_sock
    inet_csk_reqsk_queue_unlink(sk, req);

    // 3. 升级！创建完整的 tcp_sock (从128字节升级到1800+字节)
    struct sock *newsk = tcp_create_openreq_child(sk, req);
    newsk->sk_state = TCP_ESTABLISHED;

    // 4. 初始化序列号状态
    struct tcp_sock *tp = tcp_sk(newsk);
    tp->rcv_nxt = 1001;  // 期望接收的下一个序列号
    tp->snd_nxt = 2001;  // 下次发送的序列号
    tp->snd_una = 2001;  // 第一个未确认的序列号（2001之前都已确认）

    // 5. 放入 ACCEPT 队列（全连接队列）
    inet_csk_reqsk_queue_add(sk, req, newsk);

    // 6. 唤醒阻塞在 accept() 的进程
    sk->sk_data_ready(sk);

    return newsk;
}
```

#### 序列号演进时间表

```
时刻 │ 方向 │ SYN │ ACK │  seq  │  ack  │ 占用序列号 │ 说明
─────┼──────┼─────┼─────┼───────┼───────┼───────────┼──────────────
t1   │ C→S  │  1  │  0  │ 1000  │   0   │ 1000      │ 客户端发起连接
     │      │     │     │       │       │           │ SYN占用seq=1000
─────┼──────┼─────┼─────┼───────┼───────┼───────────┼──────────────
t2   │ S→C  │  1  │  1  │ 2000  │ 1001  │ 2000      │ 服务器确认并发送SYN
     │      │     │     │       │       │           │ 确认1000，期望1001
─────┼──────┼─────┼─────┼───────┼───────┼───────────┼──────────────
t3   │ C→S  │  0  │  1  │ 1001  │ 2001  │ 无        │ 客户端确认服务器SYN
     │      │     │     │       │       │           │ 确认2000，期望2001
─────┼──────┼─────┼─────┼───────┼───────┼───────────┼──────────────
t4   │ C→S  │  0  │  1  │ 1001  │ 2001  │1001-1100  │ 客户端发送100字节
     │ 数据 │     │     │       │       │           │ seq从1001开始
─────┼──────┼─────┼─────┼───────┼───────┼───────────┼──────────────
t5   │ S→C  │  0  │  1  │ 2001  │ 1101  │ 无        │ 服务器确认收到
     │ ACK  │     │     │       │       │           │ ack=1001+100=1101
─────┼──────┼─────┼─────┼───────┼───────┼───────────┼──────────────
t6   │ S→C  │  0  │  1  │ 2001  │ 1101  │2001-2200  │ 服务器发送200字节
     │ 数据 │     │     │       │       │           │ seq从2001开始
─────┼──────┼─────┼─────┼───────┼───────┼───────────┼──────────────
t7   │ C→S  │  0  │  1  │ 1101  │ 2201  │ 无        │ 客户端确认收到
     │ ACK  │     │     │       │       │           │ ack=2001+200=2201
─────┴──────┴─────┴─────┴───────┴───────┴───────────┴──────────────
```

**关键规律**：
- **SYN和FIN占用1个序列号**（即使不携带数据）
- **数据占用的序列号 = 数据长度**
- **纯ACK不占用序列号**
- **ack = 已接收的最大seq + 已接收的数据长度**

#### 为什么需要三次握手？（不是两次或四次）

**为什么不是两次握手？**
```
场景：防止旧连接请求导致资源浪费

┌────────────────────────────────────────┐
│ 客户端发送SYN(seq=1000)                 │
│   → 网络延迟，包在网络中滞留             │
│                                        │
│ 客户端超时，重发SYN(seq=2000)            │
│   → 立即到达服务器                       │
│   → 连接正常建立并使用                   │
│   → 传输完数据后关闭连接                 │
│                                        │
│ 旧的SYN(seq=1000)终于到达！              │
├────────────────────────────────────────┤
│ 如果只有两次握手：                       │
│   服务器收到旧SYN → 立即建立连接 ✗       │
│   但客户端已经关闭了！                   │
│   服务器白白分配资源（内存、端口等）      │
│   资源泄漏！                            │
├────────────────────────────────────────┤
│ 三次握手的好处：                        │
│   服务器收到旧SYN → 发送SYN+ACK          │
│   客户端收到 → 发现是旧连接 → 发RST拒绝  │
│   服务器收到RST → 释放资源 ✓            │
│   避免资源浪费！                        │
└────────────────────────────────────────┘
```

**为什么不是四次握手？**
```
三次已经足够：
┌────────────────────────────────────────┐
│ 需要确认的信息：                        │
│ 1. 客户端 → 服务器方向可达 (第1次)       │
│ 2. 服务器 → 客户端方向可达 (第2次)       │
│ 3. 客户端确认服务器收到 (第3次)          │
│                                        │
│ 第2次握手同时完成了两件事：              │
│ - 服务器发送自己的SYN                   │
│ - 服务器确认客户端的SYN (ACK)            │
│                                        │
│ 如果分开成四次：                        │
│ 1. 客户端: SYN                          │
│ 2. 服务器: ACK                          │
│ 3. 服务器: SYN                          │
│ 4. 客户端: ACK                          │
│                                        │
│ 完全没必要！合并2和3可以减少1次RTT       │
└────────────────────────────────────────┘
```

#### 序列号的作用

**1. 保证数据有序**
```
发送端发送:
  seq=1000, len=100  (数据A)
  seq=1100, len=200  (数据B)
  seq=1300, len=150  (数据C)

网络乱序到达:
  seq=1100, len=200  (B先到)
  seq=1300, len=150  (C后到)
  seq=1000, len=100  (A最后到)

接收端根据seq重排:
  1000 → 1100 → 1300 ✓
  恢复正确顺序：A → B → C
```

**2. 检测丢包**
```
接收端状态: rcv_nxt = 1100 (期望接收1100)

收到包: seq=1300, len=100
  ↓
发现: 1100 到 1299 之间的数据丢失了！
  ↓
动作:
  - 将seq=1300的包放入乱序队列
  - 发送重复ACK (ack=1100)
  - 通知发送端重传1100-1299
```

**3. 防止重复数据**
```
接收端状态: rcv_nxt = 1500

收到旧包: seq=1000, len=100
  ↓
判断: 1000 < 1500，这是已经接收过的旧数据
  ↓
动作: 丢弃，发送ACK(ack=1500)确认当前进度
```

**4. 防止注入攻击**
```
攻击者想注入伪造数据：
┌────────────────────────────────────┐
│ 攻击者抓包看到正常通信:             │
│   seq=5000, ack=3000              │
│                                   │
│ 攻击者伪造包:                      │
│   seq=5100, ack=3000, data="hack" │
│   ↓                               │
│ 接收端状态: rcv_nxt=5000           │
│   ↓                               │
│ 判断: seq=5100 > rcv_nxt=5000     │
│       中间有数据丢失，放入乱序队列   │
│   ↓                               │
│ 真实的seq=5000包到达               │
│   ↓                               │
│ 判断: seq=5100是乱序包，但等待超时  │
│       丢弃伪造包！ ✓               │
│                                   │
│ 攻击难度：                         │
│   - ISN是32位随机数 (4,294,967,296种可能) │
│   - 需要精确猜中当前的seq和ack      │
│   - 几乎不可能成功                 │
└────────────────────────────────────┘
```

#### 抓包验证

**使用tcpdump抓取三次握手**：
```bash
# 抓取8888端口的握手包
sudo tcpdump -i eth0 port 8888 -nn -S -vv

# 参数说明:
# -nn: 不解析主机名和端口名
# -S:  显示绝对序列号（不是相对序列号）
# -vv: 详细输出
```

**输出示例**：
```
# 第1次握手 - 客户端发送SYN
12:00:00.000000 IP 192.168.1.100.54321 > 192.168.1.1.8888:
  Flags [S], seq 1000, win 65535, options [mss 1460,sackOK,TS val 123456 ecr 0], length 0

  解读:
  - Flags [S]: SYN=1, ACK=0
  - seq 1000: 客户端ISN=1000
  - win 65535: 接收窗口大小
  - mss 1460: 最大段大小
  - length 0: 不携带数据

# 第2次握手 - 服务器发送SYN+ACK
12:00:00.000100 IP 192.168.1.1.8888 > 192.168.1.100.54321:
  Flags [S.], seq 2000, ack 1001, win 65535, options [mss 1460,sackOK,TS val 234567 ecr 123456], length 0

  解读:
  - Flags [S.]: SYN=1, ACK=1 (点号表示ACK)
  - seq 2000: 服务器ISN=2000
  - ack 1001: 确认客户端seq+1
  - length 0: 不携带数据

# 第3次握手 - 客户端发送ACK
12:00:00.000200 IP 192.168.1.100.54321 > 192.168.1.1.8888:
  Flags [.], seq 1001, ack 2001, win 65535, options [TS val 123457 ecr 234567], length 0

  解读:
  - Flags [.]: SYN=0, ACK=1 (点号表示纯ACK)
  - seq 1001: 使用服务器期望的序列号
  - ack 2001: 确认服务器seq+1
  - length 0: 不携带数据（也可以携带）

# 第4个包 - 客户端发送数据
12:00:00.000300 IP 192.168.1.100.54321 > 192.168.1.1.8888:
  Flags [P.], seq 1001:1101, ack 2001, win 65535, length 100

  解读:
  - Flags [P.]: PSH=1, ACK=1
  - seq 1001:1101: 发送100字节，占用序列号1001-1100
  - ack 2001: 仍然确认2001
  - length 100: 携带100字节数据
```

**使用wireshark查看**：
```
打开wireshark后，过滤器输入: tcp.port == 8888

三次握手在wireshark中的显示：
┌─────────┬──────┬──────────────┬───────────────┐
│ No.     │ Info │ Seq          │ Ack           │
├─────────┼──────┼──────────────┼───────────────┤
│ 1       │ SYN  │ seq=0 (rel)  │               │
│         │      │ seq=1000(abs)│               │
├─────────┼──────┼──────────────┼───────────────┤
│ 2       │SYN,  │ seq=0 (rel)  │ ack=1 (rel)   │
│         │ ACK  │ seq=2000(abs)│ ack=1001(abs) │
├─────────┼──────┼──────────────┼───────────────┤
│ 3       │ ACK  │ seq=1 (rel)  │ ack=1 (rel)   │
│         │      │ seq=1001(abs)│ ack=2001(abs) │
└─────────┴──────┴──────────────┴───────────────┘

注: wireshark默认显示相对序列号，从0开始更易读
    可以右键 → Protocol Preferences → 取消"Relative sequence numbers"
    查看绝对序列号
```

---

### 8.1 完整的三次握手流程

```
客户端                                服务器
  │                                      │
  │  ① SYN (seq=100)                     │
  ├──────────────────────────────────────→
  │                                      │ 收到 SYN
  │                                      │ ↓
  │                                      │ 内核处理:
  │                                      │ 1. 查找监听哈希表
  │                                      │    找到 listen_fd (port=8888)
  │                                      │ 2. 创建 request_sock
  │                                      │ 3. 加入 SYN 队列
  │                                      │ 4. 状态: SYN_RECV
  │                                      │
  │  ② SYN+ACK (seq=200, ack=101)        │
  ←──────────────────────────────────────┤
  │                                      │
收到 SYN+ACK                              │
  │                                      │
  │  ③ ACK (ack=201)                     │
  ├──────────────────────────────────────→
  │                                      │ 收到 ACK
  │                                      │ ↓
  │                                      │ 内核处理:
  │                                      │ 1. 从 SYN 队列移除 request_sock
  │                                      │ 2. 创建新的 sock
  │                                      │ 3. 状态: TCP_ESTABLISHED
  │                                      │ 4. 加入 ACCEPT 队列
  │                                      │ 5. 唤醒阻塞在 accept() 的进程
  │                                      │
连接建立 (TCP_ESTABLISHED)                │ 连接建立 (TCP_ESTABLISHED)
  │                                      │
  │                                      │ accept() 被唤醒
  │                                      │ ↓
  │                                      │ 从 ACCEPT 队列取出 sock
  │                                      │ ↓
  │                                      │ 返回 client_fd
```

### 8.2 内核收到 SYN 包的处理

```c
// net/ipv4/tcp_ipv4.c
int tcp_v4_rcv(struct sk_buff *skb)
{
    struct tcphdr *th;
    struct sock *sk;

    // 1. 解析 TCP 头部
    th = tcp_hdr(skb);

    // 2. 查找 socket
    sk = __inet_lookup_skb(&tcp_hashinfo, skb, th->source, th->dest);

    if (!sk) {
        // 没找到连接状态的 socket，查找监听 socket
        sk = __inet_lookup_listener(net, &tcp_hashinfo, skb,
                                     th->source, th->dest);
    }

    if (!sk)
        goto no_tcp_socket;

    // 3. 根据 socket 状态处理
    if (sk->sk_state == TCP_LISTEN) {
        // 监听状态，处理 SYN 包
        ret = tcp_v4_do_rcv(sk, skb);
    } else if (sk->sk_state == TCP_ESTABLISHED) {
        // 连接状态，处理数据包
        tcp_v4_do_rcv(sk, skb);
    }
    // ...
}

// 处理 SYN 包
int tcp_v4_conn_request(struct sock *sk, struct sk_buff *skb)
{
    struct request_sock *req;

    // 1. 检查 SYN 队列是否已满
    if (inet_csk_reqsk_queue_is_full(sk) && !isn) {
        // SYN 队列满了，丢弃 SYN 包
        goto drop;
    }

    // 2. 创建 request_sock
    req = inet_reqsk_alloc(&tcp_request_sock_ops, sk, !want_cookie);

    // 3. 填充 request_sock
    tcp_openreq_init(req, &tmp_opt, skb, sk);

    // 4. 加入 SYN 队列
    inet_csk_reqsk_queue_hash_add(sk, req, TCP_TIMEOUT_INIT);

    // 5. 发送 SYN+ACK
    err = tcp_v4_send_synack(sk, dst, &fl4, req, &foc);

    return 0;

drop:
    NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_LISTENDROPS);
    return 0;
}
```

### 8.3 内核收到 ACK 包的处理

```c
// net/ipv4/tcp_ipv4.c
struct sock *tcp_v4_syn_recv_sock(const struct sock *sk, struct sk_buff *skb,
                                   struct request_sock *req,
                                   struct dst_entry *dst)
{
    struct inet_request_sock *ireq;
    struct inet_sock *newinet;
    struct tcp_sock *newtp;
    struct sock *newsk;

    // 1. 创建新的 sock 对象
    newsk = tcp_create_openreq_child(sk, req, skb);
    if (!newsk)
        goto exit_nonewsk;

    // 2. 设置新 sock 的地址信息
    newinet = inet_sk(newsk);
    ireq = inet_rsk(req);
    newinet->inet_daddr = ireq->ir_rmt_addr;      // 远程 IP
    newinet->inet_rcv_saddr = ireq->ir_loc_addr;  // 本地 IP
    newinet->inet_saddr = ireq->ir_loc_addr;
    newinet->inet_dport = ireq->ir_rmt_port;      // 远程端口

    // 3. 设置状态为 TCP_SYN_RECV
    tcp_set_state(newsk, TCP_SYN_RECV);

    // 4. 加入到连接哈希表 (ehash)
    __inet_hash_nolisten(newsk, NULL);

    // 5. 将 newsk 加入到 ACCEPT 队列
    inet_csk_reqsk_queue_add(sk, req, newsk);

    return newsk;

exit_nonewsk:
    dst_release(dst);
exit:
    NET_INC_STATS_BH(sock_net(sk), LINUX_MIB_LISTENDROPS);
    return NULL;
}

// 加入 ACCEPT 队列
void inet_csk_reqsk_queue_add(struct sock *sk, struct request_sock *req,
                               struct sock *child)
{
    struct request_sock_queue *queue = &inet_csk(sk)->icsk_accept_queue;

    spin_lock(&queue->rskq_lock);
    req->sk = child;
    req->dl_next = NULL;

    // 加入队列尾部
    if (queue->rskq_accept_head == NULL)
        queue->rskq_accept_head = req;
    else
        queue->rskq_accept_tail->dl_next = req;

    queue->rskq_accept_tail = req;
    spin_unlock(&queue->rskq_lock);

    // 唤醒阻塞在 accept() 的进程
    sk->sk_data_ready(sk);
}
```

### 8.4 SYN Flood 攻击防护

**SYN Cookie 机制**：

```
正常情况：
  收到 SYN → 创建 request_sock → 加入 SYN 队列 → 占用内存

SYN Flood 攻击：
  大量伪造的 SYN 包 → SYN 队列被占满 → 正常连接无法建立

SYN Cookie 防护：
  收到 SYN → 不创建 request_sock → 不占用内存
          → 在 SYN+ACK 的序号中编码连接信息
          → 收到 ACK 时，从序号中解码出连接信息
          → 此时才创建 sock

开启方法：
  /proc/sys/net/ipv4/tcp_syncookies = 1
```

---

## 9. 总结

### 9.1 四个函数的核心作用

| 函数 | 主要作用 | 内核关键操作 | 状态变化 |
|------|---------|-------------|---------|
| **socket()** | 创建 socket | 分配 socket、sock 对象<br>分配文件描述符 | → TCP_CLOSE |
| **bind()** | 绑定地址和端口 | 设置 sk_num、sk_rcv_saddr<br>加入端口哈希表 | TCP_CLOSE |
| **listen()** | 开始监听 | 创建 SYN 队列和 ACCEPT 队列<br>加入监听哈希表 | → TCP_LISTEN |
| **accept()** | 接受连接 | 从 ACCEPT 队列取出连接<br>创建新 socket 和 fd | 返回 TCP_ESTABLISHED |

### 9.2 关键数据结构

```
struct file         - VFS 层，文件对象
  ↓
struct socket       - socket 层，socket 抽象
  ↓
struct sock         - 传输层，协议相关的 socket
  (tcp_sock)

request_sock        - 半连接对象 (SYN 队列)
request_sock_queue  - accept 队列 (SYN 队列 + ACCEPT 队列)
```

### 9.3 哈希表

```
端口哈希表 (bhash):
  - 用于检查端口是否已被绑定
  - bind() 时插入

监听哈希表 (listening_hash):
  - 用于快速查找监听 socket
  - listen() 时插入
  - 收到 SYN 包时查询

连接哈希表 (ehash):
  - 用于快速查找已建立的连接
  - 三次握手完成时插入
  - 收到数据包时查询
```

### 9.4 队列

```
SYN 队列 (半连接队列):
  - 存储收到 SYN，未完成三次握手的连接
  - request_sock 对象
  - 状态: SYN_RECV

ACCEPT 队列 (完成连接队列):
  - 存储已完成三次握手的连接
  - sock 对象
  - 状态: ESTABLISHED
  - accept() 从这里取出
```

### 9.5 记忆要点

```
✅ 核心流程：

socket() → 创建对象，分配 fd
bind()   → 绑定地址，加入端口哈希表
listen() → 创建队列，加入监听哈希表，状态变 LISTEN
accept() → 从队列取出，创建新 socket 和 fd

✅ 三次握手：

收到 SYN     → 创建 request_sock → 加入 SYN 队列
收到 ACK     → 创建 sock → 加入 ACCEPT 队列
accept() 调用 → 从 ACCEPT 队列取出

✅ 为什么需要两个队列？

SYN 队列：存储未完成的连接（防止占用太多资源）
ACCEPT 队列：存储已完成的连接（等待 accept() 取走）
```

这就是 socket、bind、listen、accept 在内核层面的完整实现！🎯
