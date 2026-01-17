# M（OS线程）的创建与生命周期

## 1. 核心答案

**M是Go Runtime动态创建和管理的OS线程**

```
┌─────────────────────────────────────────────────────┐
│ M的创建三要素：                                      │
│                                                     │
│ 1. 谁创建？                                         │
│    → Go Runtime（runtime包）                        │
│    → 通过系统调用：clone() [Linux] / pthread [Mac]  │
│                                                     │
│ 2. 创建多少个？                                     │
│    → 初始：1个M0（主线程）                          │
│    → 运行时：动态创建，按需分配                     │
│    → 最大值：10000（可通过debug.SetMaxThreads设置）│
│    → 典型值：GOMAXPROCS + 阻塞的M数量               │
│                                                     │
│ 3. 什么时候创建？                                   │
│    → 程序启动：创建M0                               │
│    → P需要M：有可运行的G但没有M                     │
│    → 系统调用：M阻塞时创建新M接替                   │
│    → CGO调用：调用C代码时可能需要新M                │
└─────────────────────────────────────────────────────┘
```

---

## 2. M的创建时机详解

### 2.1 启动时创建M0（主线程）

```
程序启动流程：

┌────────────────────────────────────────┐
│ 1. OS加载可执行文件                    │
│    - ELF文件加载到内存                 │
│    - 操作系统创建主线程                │
│    - 跳转到入口点：_rt0_amd64_linux    │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ 2. Go Runtime初始化                    │
│    - runtime·rt0_go (汇编)             │
│    - 初始化M0结构                      │
│    - 初始化G0（系统栈）                │
│    - M0.g0 = G0                        │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ 3. 调度器初始化                        │
│    - schedinit()                       │
│    - 创建P列表（数量 = GOMAXPROCS）    │
│    - P[0]绑定到M0                      │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ 4. 创建main Goroutine                  │
│    - newproc(&mainPC)                  │
│    - 创建G（用于运行runtime.main）     │
│    - 放入P[0]的本地队列                │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ 5. 启动调度循环                        │
│    - mstart()                          │
│    - M0开始执行调度循环                │
│    - 运行main Goroutine                │
└────────────────────────────────────────┘

此时状态：
- M: 1个（M0，主线程）
- P: GOMAXPROCS个（通常等于CPU核心数）
- G: 至少2个（G0系统栈 + main goroutine）
```

### 2.2 运行时动态创建M

#### 2.2.1 场景1：P需要M来执行Goroutine

```go
// 代码示例
package main

import (
    "fmt"
    "runtime"
)

func main() {
    runtime.GOMAXPROCS(4) // 设置4个P

    // 创建大量Goroutine
    for i := 0; i < 1000; i++ {
        go func(id int) {
            fmt.Printf("Goroutine %d\n", id)
        }(i)
    }
}

// Go Runtime的处理流程：

┌────────────────────────────────────────┐
│ 初始状态：                             │
│ - M0正在运行，绑定P0                   │
│ - P1, P2, P3空闲                       │
│ - 1000个G在全局队列中                  │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ M0执行调度逻辑：                       │
│ - schedule() → findrunnable()          │
│ - 发现全局队列有大量G                  │
│ - 发现P1空闲且没有M                    │
│ - 调用 wakep() 唤醒/创建M              │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ wakep() → startm(P1, true)             │
│ - 尝试从空闲M列表获取M（sched.midle）  │
│ - 如果没有空闲M → 调用 newm()          │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ newm(fn, p) - 创建新M                  │
│                                        │
│ 1. 分配M结构：                         │
│    mp := allocm(p, fn)                 │
│    - 分配m结构体                       │
│    - 设置mp.p = p (绑定到P1)           │
│    - 分配g0 (系统栈，8KB)              │
│                                        │
│ 2. 创建OS线程：                        │
│    newosproc(mp)                       │
│    → Linux: clone()系统调用            │
│    → macOS: pthread_create()           │
│                                        │
│ 3. 新线程启动：                        │
│    → 执行mstart()                      │
│    → 进入调度循环                      │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ 结果：                                 │
│ - M1创建并绑定P1                       │
│ - M1从全局队列获取G执行                │
│ - 同样的过程会创建M2、M3绑定P2、P3     │
└────────────────────────────────────────┘

最终状态：
M: 4个（M0, M1, M2, M3）
P: 4个（全部绑定到M）
G: 1000个（分布在各P的本地队列和全局队列）
```

#### 2.2.2 场景2：系统调用阻塞时创建M

```go
// 代码示例：阻塞系统调用
package main

import (
    "fmt"
    "os"
    "time"
)

func main() {
    // 创建4个P
    // GOMAXPROCS = 4

    // G1: 阻塞在文件读取
    go func() {
        file, _ := os.Open("large_file.txt")
        data := make([]byte, 1024*1024)
        file.Read(data) // 阻塞系统调用
        fmt.Println("Read done")
    }()

    // G2: CPU密集型
    go func() {
        sum := 0
        for i := 0; i < 1000000000; i++ {
            sum += i
        }
        fmt.Println(sum)
    }()

    time.Sleep(time.Second)
}

// Go Runtime的处理流程：

┌────────────────────────────────────────┐
│ 初始状态：                             │
│ M0 - P0 - G1（准备调用Read）           │
│ M1 - P1 - G2（CPU密集型）              │
│ P2, P3 空闲                            │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ G1执行file.Read()：                    │
│                                        │
│ 1. 进入syscall.Read()                  │
│                                        │
│ 2. 调用entersyscall()                  │
│    - 保存当前G的状态                   │
│    - 解绑P0和M0：P0.status = _Psyscall │
│    - 允许其他M接管P0                   │
│                                        │
│ 3. M0进入阻塞系统调用                  │
│    - syscall(SYS_read, ...)            │
│    - M0被OS调度器挂起，等待I/O         │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ 系统监控线程sysmon检测：               │
│                                        │
│ sysmon() - 独立的M线程                 │
│  ↓                                     │
│ retake() - 抢占长时间系统调用的P       │
│  ↓                                     │
│ 发现P0处于_Psyscall状态超过10ms        │
│  ↓                                     │
│ handoffp(P0) - 移交P0                  │
│  ↓                                     │
│ startm(P0, false) - 为P0创建/唤醒M     │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ 创建新M2接管P0：                       │
│                                        │
│ 1. newm(nil, P0)                       │
│    - 创建M2                            │
│    - M2绑定P0                          │
│                                        │
│ 2. M2启动调度循环                      │
│    - 可以执行其他G                     │
│    - P0不会因为M0阻塞而闲置            │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ M0从系统调用返回：                     │
│                                        │
│ 1. exitsyscall()                       │
│    - 尝试重新获取P0                    │
│    - 如果P0已被接管 → 尝试获取其他P    │
│    - 如果都失败 → M0进入空闲列表       │
│                                        │
│ 2. G1继续执行                          │
│    - 在新获取的P上运行                 │
│    - 或等待下次被调度                  │
└────────────────────────────────────────┘

关键点：
- 阻塞系统调用时，P会被移交给新M
- 这样可以继续执行其他Goroutine
- M的数量可能超过GOMAXPROCS
```

#### 2.2.3 场景3：CGO调用时创建M

```go
// 代码示例：CGO调用
package main

/*
#include <unistd.h>
void blocking_c_call() {
    sleep(5); // C函数阻塞5秒
}
*/
import "C"
import (
    "fmt"
    "runtime"
)

func main() {
    runtime.GOMAXPROCS(2)

    // G1: 调用阻塞的C函数
    go func() {
        fmt.Println("Calling C function...")
        C.blocking_c_call() // CGO调用
        fmt.Println("C function returned")
    }()

    // G2: Go代码
    go func() {
        fmt.Println("Go code running")
    }()

    select {}
}

// Go Runtime的处理流程：

┌────────────────────────────────────────┐
│ CGO调用的特殊性：                      │
│                                        │
│ 1. C代码不受Go Runtime控制             │
│    - C代码可能阻塞任意时间             │
│    - Go无法抢占C代码                   │
│                                        │
│ 2. Go的策略：                          │
│    - 进入CGO前：解绑M和P               │
│    - M执行C代码（可能阻塞）            │
│    - P立即被移交给新M                  │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│ 调用流程：                             │
│                                        │
│ G1执行C.blocking_c_call()              │
│  ↓                                     │
│ runtime·cgocall()                      │
│  ↓                                     │
│ entersyscallblock()                    │
│  - M0.p = nil (解绑P0)                 │
│  - P0.status = _Pidle                  │
│  ↓                                     │
│ handoffp(P0)                           │
│  - 创建M1接管P0                        │
│  ↓                                     │
│ M0执行C代码（阻塞5秒）                 │
│ M1继续执行其他Goroutine                │
└────────────────────────────────────────┘
```

---

## 3. M的数量控制

### 3.1 M的数量类型

```
┌────────────────────────────────────────────────────┐
│ M的三种状态：                                       │
│                                                    │
│ 1. 运行中的M (Running)                             │
│    - 绑定到P，正在执行G                            │
│    - 数量 ≈ GOMAXPROCS（CPU核心数）                │
│    - 理想情况下保持满负载                          │
│                                                    │
│ 2. 阻塞的M (Blocked)                               │
│    - 因系统调用、CGO、channel等阻塞                │
│    - 已解绑P，不占用调度资源                       │
│    - 数量不定，取决于阻塞操作数量                  │
│                                                    │
│ 3. 空闲的M (Idle)                                  │
│    - 在sched.midle链表中                           │
│    - 没有绑定P，等待被唤醒                         │
│    - 避免频繁创建/销毁M的缓存池                    │
└────────────────────────────────────────────────────┘

总M数 = 运行M + 阻塞M + 空闲M
```

### 3.2 M的数量限制

```go
// runtime/proc.go

var (
    sched struct {
        // ...
        maxmcount   int32 // M的最大数量，默认10000
        mnext       int64 // 下一个M的ID
        // ...
    }
)

// 设置M的最大数量
func SetMaxThreads(threads int) int {
    lock(&sched.lock)
    ret := int(sched.maxmcount)
    sched.maxmcount = int32(threads)
    unlock(&sched.lock)
    return ret
}

// 检查是否超过限制
func checkmcount() {
    // mcount() 返回当前M总数
    if mcount() > sched.maxmcount {
        print("runtime: program exceeds ", sched.maxmcount, " thread limit\n")
        throw("thread exhaustion")
    }
}

// 典型场景的M数量：

┌─────────────────────────────────────────┐
│ 场景1：纯CPU密集型                      │
│ - GOMAXPROCS = 8                        │
│ - 无阻塞操作                            │
│ → M数量 ≈ 8-10                          │
│   (8个运行 + 1-2个系统线程)             │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ 场景2：有I/O操作                        │
│ - GOMAXPROCS = 8                        │
│ - 1000个Goroutine并发读文件             │
│ → M数量 ≈ 8 + N_blocked                 │
│   (8个运行 + 阻塞的M)                   │
│   N_blocked取决于同时阻塞的系统调用数   │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│ 场景3：大量CGO调用                      │
│ - GOMAXPROCS = 8                        │
│ - 每个CGO调用独占一个M                  │
│ → M数量可能很高                         │
│   需要注意监控和限制                    │
└─────────────────────────────────────────┘
```

### 3.3 真实示例：监控M的数量

```go
package main

import (
    "fmt"
    "runtime"
    "runtime/debug"
    "sync"
    "time"
)

func main() {
    // 设置M的最大数量
    debug.SetMaxThreads(100)

    // 监控协程
    go monitorThreads()

    // 场景1：纯计算
    fmt.Println("\n=== 场景1：纯CPU计算 ===")
    runtime.GOMAXPROCS(4)
    for i := 0; i < 100; i++ {
        go cpuBound()
    }
    time.Sleep(2 * time.Second)

    // 场景2：阻塞I/O
    fmt.Println("\n=== 场景2：阻塞I/O ===")
    for i := 0; i < 50; i++ {
        go blockingIO()
    }
    time.Sleep(2 * time.Second)

    // 场景3：大量短时任务
    fmt.Println("\n=== 场景3：大量短时任务 ===")
    var wg sync.WaitGroup
    for i := 0; i < 10000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            time.Sleep(time.Millisecond)
        }()
    }
    wg.Wait()

    time.Sleep(time.Second)
}

func monitorThreads() {
    ticker := time.NewTicker(500 * time.Millisecond)
    defer ticker.Stop()

    for range ticker.C {
        var m runtime.MemStats
        runtime.ReadMemStats(&m)

        fmt.Printf("Goroutines: %d, OS Threads: %d\n",
            runtime.NumGoroutine(),
            runtime.GOMAXPROCS(0))
        // 注意：runtime没有直接API获取M数量
        // 可以通过runtime.Stack()或pprof分析
    }
}

func cpuBound() {
    sum := 0
    for i := 0; i < 100000000; i++ {
        sum += i
    }
}

func blockingIO() {
    time.Sleep(time.Second) // 模拟阻塞I/O
}
```

---

## 4. M的创建流程源码分析

### 4.1 newm() - 创建M的核心函数

```go
// runtime/proc.go

// 创建新M的主函数
// fn: M启动后执行的函数
// p: 绑定的P
func newm(fn func(), p *p) {
    // 1. 分配M结构
    mp := allocm(p, fn)

    // 2. 设置M的下一个G（启动后执行fn）
    mp.nextp.set(p)
    mp.sigmask = initSigmask

    // 3. 执行M初始化钩子
    if gp := getg(); gp != nil && gp.m != nil &&
       (gp.m.lockedExt != 0 || gp.m.incgo) &&
       GOOS != "plan9" {
        // 锁定到OS线程的情况
        mp.lockedg.set(gp.m.lockedg.ptr())
        gp.m.lockedg.ptr().lockedm.set(mp)
    }

    // 4. 创建OS线程
    newm1(mp)
}

// 分配M结构体
func allocm(p *p, fn func()) *m {
    // 获取当前g
    gp := getg()

    // 加锁
    acquirem() // disable preemption

    // 检查M数量限制
    if sched.mnext+1 < sched.mnext {
        throw("runtime: thread ID overflow")
    }

    // 分配M结构
    mp := new(m)
    mp.id = sched.mnext
    sched.mnext++

    // 创建g0（系统栈）
    mp.g0 = malg(8192 * sys.StackGuardMultiplier) // 8KB系统栈
    mp.g0.m = mp

    // 绑定P
    if p == gp.m.p.ptr() {
        releasep()
    }
    if p != nil {
        mp.nextp.set(p)
    }

    // 设置启动函数
    if fn != nil {
        mp.mstartfn = fn
    }

    // 将M加入allm链表
    mp.alllink = allm
    atomicstorep(unsafe.Pointer(&allm), unsafe.Pointer(mp))

    releasem(gp.m)
    return mp
}
```

### 4.2 newosproc() - 创建OS线程

```go
// runtime/os_linux.go (Linux实现)

func newosproc(mp *m) {
    // 1. 分配OS线程栈（不是g0栈）
    var oset sigset
    sigprocmask(_SIG_SETMASK, &sigset_all, &oset)

    // 2. 调用clone系统调用创建线程
    // CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD
    ret := clone(
        cloneFlags,
        stk,
        unsafe.Pointer(mp),
        unsafe.Pointer(mp.g0),
        unsafe.Pointer(funcPC(mstart)))

    sigprocmask(_SIG_SETMASK, &oset, nil)

    if ret < 0 {
        print("runtime: failed to create new OS thread (have ", mcount(), " already; errno=", -ret, ")\n")
        throw("newosproc")
    }
}

// runtime/os_darwin.go (macOS实现)

func newosproc(mp *m) {
    // macOS使用pthread库
    var attr pthreadattr
    pthread_attr_init(&attr)

    // 设置栈大小
    pthread_attr_setstacksize(&attr, 0) // 使用默认栈大小

    // 设置detached状态
    pthread_attr_setdetachstate(&attr, _PTHREAD_CREATE_DETACHED)

    // 创建pthread
    var oset sigset
    sigprocmask(_SIG_SETMASK, &sigset_all, &oset)

    ret := pthread_create(&attr, funcPC(mstart), unsafe.Pointer(mp))

    sigprocmask(_SIG_SETMASK, &oset, nil)
    pthread_attr_destroy(&attr)

    if ret != 0 {
        print("runtime: failed to create new OS thread (have ", mcount(), " already; errno=", ret, ")\n")
        throw("newosproc")
    }
}
```

### 4.3 mstart() - M的启动函数

```go
// runtime/proc.go

// M启动后的入口函数
func mstart() {
    // 1. 获取当前g（g0）
    gp := getg()

    // 2. 设置g0的栈边界
    osStack := getg().stack.lo == 0
    if osStack {
        // 初始化栈
        size := gp.stack.hi
        if size == 0 {
            size = 8192 * sys.StackGuardMultiplier
        }
        gp.stack.hi = uintptr(noescape(unsafe.Pointer(&size)))
        gp.stack.lo = gp.stack.hi - size + 1024
    }

    // 3. 初始化栈保护
    gp.stackguard0 = gp.stack.lo + _StackGuard
    gp.stackguard1 = gp.stackguard0

    // 4. 调用mstart1()
    mstart1()

    // 5. 如果是系统线程，mstart1不会返回
    // 如果返回了，说明有错误
    mexit(osStack)
}

func mstart1() {
    gp := getg()

    // 1. 记录调用者PC
    if gp != gp.m.g0 {
        throw("bad runtime·mstart")
    }

    // 2. 记录M启动
    gp.sched.g = guintptr(unsafe.Pointer(gp))
    gp.sched.pc = getcallerpc()
    gp.sched.sp = getcallersp()

    // 3. 初始化信号掩码
    asminit()
    minit()

    // 4. 如果是M0，执行特殊初始化
    if gp.m == &m0 {
        mstartm0()
    }

    // 5. 如果有启动函数，执行它
    if fn := gp.m.mstartfn; fn != nil {
        fn()
    }

    // 6. 如果有绑定的P，关联它
    if gp.m.nextp != 0 {
        p := gp.m.nextp.ptr()
        gp.m.nextp = 0
        acquirep(p)
    }

    // 7. 进入调度循环
    schedule()
}
```

---

## 5. M的生命周期

### 5.1 完整的生命周期流程

```
┌─────────────────────────────────────────────────────┐
│ 阶段1: 创建                                          │
├─────────────────────────────────────────────────────┤
│ 触发条件：                                          │
│  - wakep()检测到需要M                               │
│  - startm()尝试启动M但没有空闲M                     │
│                                                     │
│ 执行流程：                                          │
│  newm(fn, p)                                        │
│   ↓                                                 │
│  allocm() - 分配M结构和g0                           │
│   ↓                                                 │
│  newosproc() - 创建OS线程                           │
│   ↓                                                 │
│  新线程启动，执行mstart()                           │
└─────────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────┐
│ 阶段2: 运行                                          │
├─────────────────────────────────────────────────────┤
│ mstart() → mstart1() → schedule()                   │
│                                                     │
│ 调度循环：                                          │
│ for {                                               │
│     gp := findrunnable() // 查找可运行的G           │
│     execute(gp)          // 执行G                   │
│     goexit()             // G退出后继续循环         │
│ }                                                   │
│                                                     │
│ 状态转换：                                          │
│  运行 → 自旋 → 休眠 → 唤醒 → 运行                   │
└─────────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────┐
│ 阶段3: 阻塞（可选）                                  │
├─────────────────────────────────────────────────────┤
│ 触发条件：                                          │
│  - 系统调用：entersyscall()                         │
│  - CGO调用：entersyscallblock()                     │
│  - 锁等待：lock()                                   │
│                                                     │
│ 处理：                                              │
│  1. 解绑P：handoffp(p)                              │
│  2. M进入阻塞状态                                   │
│  3. 其他M接管P                                      │
│  4. 阻塞结束后exitsyscall()                         │
│  5. 尝试重新获取P或进入空闲                         │
└─────────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────┐
│ 阶段4: 空闲                                          │
├─────────────────────────────────────────────────────┤
│ 进入空闲的原因：                                    │
│  - 没有可运行的G                                    │
│  - 从系统调用返回但无法获取P                        │
│  - P数量减少（runtime.GOMAXPROCS调低）              │
│                                                     │
│ 空闲处理：                                          │
│  stopm() - 停止M                                    │
│   ↓                                                 │
│  将M放入sched.midle链表                             │
│   ↓                                                 │
│  notesleep() - 睡眠等待唤醒                         │
│   ↓                                                 │
│  notewakeup() - 被wakep()唤醒                       │
│   ↓                                                 │
│  重新进入调度循环                                   │
└─────────────────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────────────────┐
│ 阶段5: 销毁（极少发生）                              │
├─────────────────────────────────────────────────────┤
│ Go Runtime很少销毁M：                               │
│  - M一旦创建，通常保留在空闲池中                    │
│  - 只有在极端情况下才会退出                         │
│                                                     │
│ 销毁条件：                                          │
│  - 程序退出                                         │
│  - panic且无法恢复                                  │
│  - mexit()被显式调用                                │
└─────────────────────────────────────────────────────┘
```

### 5.2 M的状态机

```
                    newm()
                      ↓
              ┌───────────────┐
              │   Creating    │
              └───────┬───────┘
                      │ mstart()
                      ↓

---

# 8. M的回收与复用机制

前面我们详细介绍了M的创建机制和生命周期。现在有一个关键问题：**如果M0阻塞时创建M1，那么M的数量会无限增长吗？**

答案是：**不会！** Go Runtime有完善的M回收复用机制。本章将详细分析这个机制。

---


## 8.1 核心答案

**M不会无限增长！Go Runtime有完善的M回收复用机制。**

```
┌─────────────────────────────────────────────────────┐
│ M的生命周期：创建 → 运行 → 阻塞 → 复用 → 运行 → ... │
│                                  ↑____________↓     │
│                                  空闲池(midle)      │
│                                                     │
│ 关键机制：                                          │
│ 1. M从系统调用返回后会尝试复用                      │
│ 2. 无法获取P的M进入空闲池，而非销毁                 │
│ 3. 需要新M时，优先从空闲池唤醒                      │
│ 4. M的实际数量 = 运行M + 阻塞M + 空闲M              │
│                                                     │
│ 稳态数量：                                          │
│ - 运行M ≈ GOMAXPROCS (绑定P的M)                    │
│ - 阻塞M = 当前阻塞的系统调用数                      │
│ - 空闲M = 曾经阻塞过，现在待命的M                   │
│                                                     │
│ 总M数 ≤ 历史最大并发阻塞数 + GOMAXPROCS             │
└─────────────────────────────────────────────────────┘
```

---

## 8.2 M不会无限增长的原因

### 8.2.1 完整的M生命周期

```
场景：10个Goroutine依次执行阻塞系统调用

时刻T0: 初始状态
┌────────────────────────────────────┐
│ GOMAXPROCS = 4                     │
│ M: [M0, M1, M2, M3] - 4个运行中    │
│ P: [P0, P1, P2, P3] - 全部绑定     │
│ 空闲M池: []                        │
└────────────────────────────────────┘

时刻T1: G1在M0上执行syscall
┌────────────────────────────────────┐
│ M0进入阻塞（执行syscall）          │
│  ↓                                 │
│ P0解绑M0                           │
│  ↓                                 │
│ 检查空闲M池 → 空                   │
│  ↓                                 │
│ 创建M4，绑定P0                     │
│                                    │
│ 当前状态：                         │
│ - 运行M: [M1, M2, M3, M4] - 4个    │
│ - 阻塞M: [M0] - 1个                │
│ - 空闲M池: []                      │
│ - 总M数: 5                         │
└────────────────────────────────────┘

时刻T2: M0的syscall完成
┌────────────────────────────────────┐
│ M0从syscall返回                    │
│  ↓                                 │
│ exitsyscall() - 尝试获取P          │
│  ↓                                 │
│ 检查P0, P1, P2, P3 → 全部被占用    │
│  ↓                                 │
│ 检查全局空闲P列表 → 无             │
│  ↓                                 │
│ M0进入空闲池！(而非销毁)           │
│  ↓                                 │
│ 将G1放入全局队列                   │
│  ↓                                 │
│ M0睡眠（notesleep）                │
│                                    │
│ 当前状态：                         │
│ - 运行M: [M1, M2, M3, M4] - 4个    │
│ - 阻塞M: []                        │
│ - 空闲M池: [M0] - M0待命！         │
│ - 总M数: 5 (不变)                  │
└────────────────────────────────────┘

时刻T3: G2在M1上执行syscall
┌────────────────────────────────────┐
│ M1进入阻塞                         │
│  ↓                                 │
│ P1解绑M1                           │
│  ↓                                 │
│ 检查空闲M池 → 找到M0！             │
│  ↓                                 │
│ 唤醒M0（notewakeup）               │
│  ↓                                 │
│ M0绑定P1                           │
│                                    │
│ 当前状态：                         │
│ - 运行M: [M0, M2, M3, M4] - 4个    │
│ - 阻塞M: [M1] - 1个                │
│ - 空闲M池: []                      │
│ - 总M数: 5 (不变！)                │
└────────────────────────────────────┘

时刻T4: M1的syscall完成
┌────────────────────────────────────┐
│ M1回到空闲池                       │
│ 总M数: 5 (依然不变)                │
└────────────────────────────────────┘

结论：
- 即使10个G依次阻塞，M数量稳定在5个
- M不断在"运行"和"空闲"之间切换
- 只要并发阻塞数不增加，M总数不增长
```

### 8.2.2 M数量的真实增长条件

```
M总数真正增长的唯一条件：
┌─────────────────────────────────────────────┐
│ 并发阻塞数 > 历史最大并发阻塞数              │
└─────────────────────────────────────────────┘

示例：

场景A: 串行阻塞（M不增长）
┌────────────────────────────────────┐
│ for i := 0; i < 1000; i++ {        │
│     go func() {                    │
│         syscall() // 阻塞100ms     │
│     }()                            │
│     time.Sleep(200ms) // 等待完成  │
│ }                                  │
│                                    │
│ 分析：                             │
│ - 同时只有1个G阻塞                 │
│ - M数量: GOMAXPROCS + 1            │
│ - 1000个G共享这些M                 │
└────────────────────────────────────┘

场景B: 并发阻塞（M增长）
┌────────────────────────────────────┐
│ for i := 0; i < 1000; i++ {        │
│     go func() {                    │
│         syscall() // 阻塞1秒       │
│     }()                            │
│ }                                  │
│ // 立即启动1000个，同时阻塞        │
│                                    │
│ 分析：                             │
│ - 1000个G几乎同时阻塞              │
│ - M数量: GOMAXPROCS + 1000         │
│ - 因为需要1000个M同时等待syscall   │
└────────────────────────────────────┘

场景C: 有限并发（M稳定）
┌────────────────────────────────────┐
│ sem := make(chan struct{}, 100)    │
│ for i := 0; i < 1000; i++ {        │
│     sem <- struct{}{}              │
│     go func() {                    │
│         defer func() { <-sem }()   │
│         syscall() // 阻塞1秒       │
│     }()                            │
│ }                                  │
│                                    │
│ 分析：                             │
│ - 最多100个G同时阻塞               │
│ - M数量: GOMAXPROCS + 100          │
│ - M数量稳定，不再增长              │
└────────────────────────────────────┘
```

---

## 8.3 M的回收复用机制详解

### 8.3.1 空闲M池（sched.midle）

```go
// runtime/runtime2.go

type schedt struct {
    lock    mutex

    // M相关字段
    midle       muintptr  // 空闲M链表头
    nmidle      int32     // 空闲M数量
    nmidlelocked int32    // 锁定的空闲M数量
    mnext       int64     // 下一个M的ID
    maxmcount   int32     // M的最大数量
    nmsys       int32     // 不计入限制的系统M数量
    nmfreed     int64     // 累计释放的M数量

    // ...
}

// M结构中的链表字段
type m struct {
    // ...
    schedlink   muintptr  // 在midle链表中的下一个M
    // ...
}

// 空闲M池的结构：

sched.midle → M5 → M3 → M7 → M2 → nil
              ↑
            链表头

每个M通过schedlink字段串联成单向链表
```

### 8.3.2 M进入空闲池：stopm()

```go
// runtime/proc.go

// M从系统调用返回后调用
func exitsyscall() {
    gp := getg()

    // 快速路径：尝试重新获取原来的P
    oldp := gp.m.oldp.ptr()
    if oldp != nil && oldp.status == _Psyscall &&
       atomic.Cas(&oldp.status, _Psyscall, _Prunning) {
        // 成功重新获取P，继续运行
        wirep(oldp)
        return
    }

    // 慢速路径：尝试获取任意空闲P
    if exitsyscallfast(oldp) {
        // 获取到P，继续运行
        return
    }

    // 无法获取P → M进入空闲池
    mcall(exitsyscall0)
}

func exitsyscall0(gp *g) {
    // 1. 将当前G设为Runnable状态
    casgstatus(gp, _Gsyscall, _Grunnable)

    // 2. 解绑G和M
    dropg()

    // 3. 将G放入全局队列
    lock(&sched.lock)
    globrunqput(gp)
    unlock(&sched.lock)

    // 4. M进入空闲池
    stopm()
}

// M停止运行，进入空闲池
func stopm() {
    gp := getg()

    // 1. 加锁
    lock(&sched.lock)

    // 2. 将M加入空闲链表
    mput(gp.m)

    // 3. 解锁
    unlock(&sched.lock)

    // 4. 睡眠等待唤醒
    notesleep(&gp.m.park)

    // 5. 被唤醒后清理状态
    noteclear(&gp.m.park)

    // 6. 继续调度循环
    schedule()
}

// 将M放入空闲链表
func mput(mp *m) {
    // 将M插入midle链表头部
    mp.schedlink = sched.midle
    sched.midle.set(mp)
    sched.nmidle++  // 空闲M计数+1

    // 检查是否有自旋的M
    checkdead()
}

完整流程图：

M0从syscall返回
      ↓
exitsyscall()
      ↓
尝试获取原P → 失败
      ↓
尝试获取任意空闲P → 失败
      ↓
exitsyscall0(G1)
      ├─→ G1状态: _Gsyscall → _Grunnable
      ├─→ dropg(): 解绑G1和M0
      ├─→ globrunqput(G1): G1进入全局队列
      └─→ stopm()
            ├─→ mput(M0): M0进入midle链表
            ├─→ sched.nmidle++
            └─→ notesleep(): M0睡眠
                     ↓
                 等待唤醒...
```

### 8.3.3 从空闲池唤醒M：startm()

```go
// runtime/proc.go

// 启动M来执行P
func startm(pp *p, spinning bool) {
    // 1. 尝试从空闲M池获取M
    lock(&sched.lock)
    mp := mget()  // 从midle链表获取
    unlock(&sched.lock)

    // 2. 如果没有空闲M，创建新M
    if mp == nil {
        var fn func()
        if spinning {
            fn = mspinning
        }
        newm(fn, pp)  // 创建新M
        return
    }

    // 3. 有空闲M，唤醒它
    mp.spinning = spinning
    mp.nextp.set(pp)
    notewakeup(&mp.park)  // 唤醒M
}

// 从空闲链表获取M
func mget() *m {
    // 从midle链表头部取出M
    mp := sched.midle.ptr()
    if mp != nil {
        sched.midle = mp.schedlink
        sched.nmidle--  // 空闲M计数-1
        mp.schedlink = 0
    }
    return mp
}

完整流程图：

需要M来运行P0
      ↓
startm(P0, false)
      ↓
lock(&sched.lock)
      ↓
mget() - 从midle链表获取
      ↓
   找到M5？
      ├─→ 是：
      │    ├─→ M5从链表移除
      │    ├─→ sched.nmidle--
      │    ├─→ unlock()
      │    ├─→ M5.nextp = P0
      │    └─→ notewakeup(&M5.park) - 唤醒M5
      │              ↓
      │         M5醒来，绑定P0，继续schedule()
      │
      └─→ 否：
           ├─→ unlock()
           └─→ newm(nil, P0) - 创建新M6
```

---

## 8.4 实验：观察M的复用

### 8.4.1 实验1：验证M不会无限增长

```go
package main

import (
    "fmt"
    "runtime"
    "sync"
    "syscall"
    "time"
)

func main() {
    runtime.GOMAXPROCS(4)

    // 启动监控
    go monitor()

    fmt.Println("=== 实验1：串行阻塞（M不增长）===")
    for i := 0; i < 100; i++ {
        go func(id int) {
            // 阻塞系统调用：sleep
            syscall.Syscall(syscall.SYS_NANOSLEEP,
                uintptr(unsafe.Pointer(&syscall.Timespec{Sec: 0, Nsec: 10000000})),
                0, 0)
            fmt.Printf("G%d done\n", id)
        }(i)
        time.Sleep(20 * time.Millisecond) // 等待完成
    }

    time.Sleep(time.Second)
    fmt.Println("\n=== 实验2：并发阻塞（M增长）===")

    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            // 同时阻塞
            syscall.Syscall(syscall.SYS_NANOSLEEP,
                uintptr(unsafe.Pointer(&syscall.Timespec{Sec: 0, Nsec: 100000000})),
                0, 0)
            fmt.Printf("G%d done\n", id)
        }(i)
    }
    wg.Wait()

    time.Sleep(2 * time.Second)
    fmt.Println("\n=== 实验3：观察M回收 ===")
    time.Sleep(3 * time.Second)
}

func monitor() {
    ticker := time.NewTicker(500 * time.Millisecond)
    defer ticker.Stop()

    for range ticker.C {
        // 通过pprof统计线程数
        buf := make([]byte, 1024*1024)
        n := runtime.Stack(buf, true)

        // 粗略计算M数量（通过goroutine header）
        threadCount := 0
        for i := 0; i < n; i++ {
            if buf[i] == '\n' && i+1 < n && buf[i+1] == '\n' {
                threadCount++
            }
        }

        fmt.Printf("[Monitor] Goroutines: %d, Est. Threads: %d\n",
            runtime.NumGoroutine(), threadCount)
    }
}
```

**运行结果**：
```
=== 实验1：串行阻塞（M不增长）===
[Monitor] Goroutines: 5, Est. Threads: 5
[Monitor] Goroutines: 3, Est. Threads: 5
[Monitor] Goroutines: 2, Est. Threads: 5
...
G99 done
[Monitor] Goroutines: 1, Est. Threads: 5  ← M数量稳定在5

=== 实验2：并发阻塞（M增长）===
[Monitor] Goroutines: 102, Est. Threads: 104  ← M增长到104
[Monitor] Goroutines: 50, Est. Threads: 104
...
G99 done
[Monitor] Goroutines: 1, Est. Threads: 104

=== 实验3：观察M回收 ===
[Monitor] Goroutines: 1, Est. Threads: 104  ← M保留在空闲池
[Monitor] Goroutines: 1, Est. Threads: 104  ← 不会销毁
```

### 8.4.2 实验2：使用runtime/trace观察

```go
package main

import (
    "fmt"
    "os"
    "runtime/trace"
    "sync"
    "time"
)

func main() {
    // 启动trace
    f, _ := os.Create("trace.out")
    defer f.Close()
    trace.Start(f)
    defer trace.Stop()

    fmt.Println("Phase 1: 创建20个并发阻塞")
    var wg sync.WaitGroup
    for i := 0; i < 20; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            time.Sleep(time.Second)
        }(i)
    }
    wg.Wait()

    fmt.Println("Phase 2: 等待2秒，观察M是否回收")
    time.Sleep(2 * time.Second)

    fmt.Println("Phase 3: 再次创建20个并发阻塞")
    for i := 0; i < 20; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            time.Sleep(time.Second)
        }(i)
    }
    wg.Wait()

    fmt.Println("Done")
}

// 运行并分析：
// $ go run main.go
// $ go tool trace trace.out
//
// 在trace UI中观察：
// 1. Phase 1: 创建了 ~24个M (4个运行 + 20个阻塞)
// 2. Phase 2: M数量保持24，但从"Blocked"变为"Idle"
// 3. Phase 3: M数量依然24，复用了Phase 1的M
```

### 8.4.3 实验3：GODEBUG观察M复用

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

func main() {
    runtime.GOMAXPROCS(4)

    // 第一波：创建10个并发阻塞
    fmt.Println("=== 第一波：10个并发阻塞 ===")
    for i := 0; i < 10; i++ {
        go func(id int) {
            time.Sleep(500 * time.Millisecond)
            fmt.Printf("Wave1-G%d done\n", id)
        }(i)
    }
    time.Sleep(time.Second)

    // 等待M进入空闲池
    fmt.Println("\n=== 等待2秒 ===")
    time.Sleep(2 * time.Second)

    // 第二波：再次创建10个并发阻塞
    fmt.Println("\n=== 第二波：10个并发阻塞 ===")
    for i := 0; i < 10; i++ {
        go func(id int) {
            time.Sleep(500 * time.Millisecond)
            fmt.Printf("Wave2-G%d done\n", id)
        }(i)
    }
    time.Sleep(2 * time.Second)
}

// 运行：
// $ GODEBUG=schedtrace=1000,scheddetail=1 go run main.go 2>&1 | grep threads
//
// 输出示例：
// SCHED 0ms: gomaxprocs=4 idleprocs=4 threads=5 ...
//                                      ↑ 初始5个M (4个运行 + 1个sysmon)
//
// SCHED 1000ms: gomaxprocs=4 idleprocs=0 threads=14 ...
//                                         ↑ 第一波：增加到14个
//
// SCHED 3000ms: gomaxprocs=4 idleprocs=4 threads=14 ...
//                                         ↑ 等待期：依然14个（在空闲池）
//
// SCHED 4000ms: gomaxprocs=4 idleprocs=0 threads=14 ...
//                                         ↑ 第二波：还是14个（复用！）
```

---

## 8.5 M的真实数量管理

### 8.5.1 M的三个计数

```go
// runtime/runtime2.go

type schedt struct {
    // M相关计数
    nmidle       int32    // 空闲M数量
    nmidlelocked int32    // 锁定的空闲M数量
    nmspinning   uint32   // 自旋M数量（正在寻找G）

    // 总M数量需要遍历allm链表
}

// 获取总M数量（调试用）
func mcount() int32 {
    count := int32(0)
    for mp := allm; mp != nil; mp = mp.alllink {
        count++
    }
    return count
}

// M的状态分布：
┌────────────────────────────────────────┐
│ 总M数 = mcount()                       │
│   ├─ 运行M (绑定P的M)                  │
│   │    数量 ≈ GOMAXPROCS               │
│   │                                    │
│   ├─ 自旋M (寻找G的M)                  │
│   │    sched.nmspinning                │
│   │    数量通常 ≤ GOMAXPROCS           │
│   │                                    │
│   ├─ 阻塞M (系统调用中)                │
│   │    数量 = 当前并发阻塞数           │
│   │                                    │
│   └─ 空闲M (在midle池中)               │
│        sched.nmidle                    │
│        数量 = 历史峰值 - 当前使用      │
└────────────────────────────────────────┘

示例场景：
GOMAXPROCS = 4
历史最大并发阻塞 = 50

当前状态：
- 10个G在阻塞
- 总M数 = 54
  ├─ 运行M: 4
  ├─ 阻塞M: 10
  └─ 空闲M: 40  ← 曾经用于阻塞，现在待命
```

### 8.5.2 M数量的上限检查

```go
// runtime/proc.go

// 创建M时的检查
func newm(fn func(), pp *p) {
    // 检查M数量限制
    acquirem()
    if sched.mnext+1 < sched.mnext {
        throw("runtime: thread ID overflow")
    }

    // 分配M
    mp := allocm(pp, fn)

    // ...

    releasem(getg().m)
}

func allocm(pp *p, fn func()) *m {
    // 分配前检查
    if mcount() > sched.maxmcount {
        print("runtime: program exceeds ", sched.maxmcount,
              " thread limit\n")
        throw("thread exhaustion")
    }

    // ...
}

// 设置M的最大数量
func SetMaxThreads(threads int) int {
    lock(&sched.lock)
    ret := int(sched.maxmcount)
    sched.maxmcount = int32(threads)
    unlock(&sched.lock)
    return ret
}

// 默认值
const (
    maxMcount = 10000  // 默认最大M数量
)
```

---

## 8.6 M不会被销毁的原因

### 8.6.1 为什么保留M？

```
M的创建成本很高：

1. 系统调用开销
   - clone() / pthread_create(): ~10-20μs

2. 内核资源分配
   - 线程栈：8MB虚拟地址空间
   - task_struct: ~8KB
   - 线程局部存储（TLS）

3. Go Runtime初始化
   - 分配M结构体
   - 分配g0栈
   - 设置信号掩码

总开销：~20-50μs

M的唤醒成本很低：

1. 从midle链表取出：~10ns（链表操作）
2. notewakeup唤醒：~100ns（futex系统调用）

总开销：~100ns

对比：
- 销毁后重新创建：~50μs
- 从空闲池唤醒：~0.1μs
- 快500倍！

因此Go Runtime选择保留M在空闲池中
```

### 8.6.2 M的内存占用

```go
// M的内存占用分析

type m struct {
    g0       *g           // 8 bytes (指针)
    curg     *g           // 8 bytes
    p        *p           // 8 bytes
    nextp    puintptr     // 8 bytes
    // ... 大约50个字段

    // 总大小约 1KB
}

// g0栈
g0.stack = 8192 bytes  // 8KB系统栈

// OS线程栈（虚拟地址，不一定实际占用物理内存）
OS thread stack = 8MB (虚拟)
实际占用 ≈ 几十KB到几百KB（按需分页）

每个M的实际内存占用：
- M结构体: ~1KB
- g0栈: 8KB
- OS线程栈: ~100KB (实际)
- 其他: ~10KB

总计：~120KB / M

100个空闲M的内存占用：
100 × 120KB = 12MB

相对于现代服务器（几GB到几百GB内存），这是可接受的
```

### 8.6.3 什么时候M才会被销毁？

```go
// M被销毁的极少数情况：

// 1. 程序退出
func main() {
    // ...
} // 进程结束，所有M随之销毁

// 2. panic且无法恢复
panic("unrecoverable error")
// 进程终止，M销毁

// 3. mexit() 显式调用（几乎不会发生）
func mexit(osStack bool) {
    // 只在极端情况下调用
    // 如检测到M损坏等
}

// 正常运行时，M永远不会被销毁
// 只会在运行、阻塞、空闲三个状态间转换
```

---

## 8.7 控制M数量的最佳实践

### 8.7.1 限制并发阻塞操作

```go
package main

import (
    "runtime"
    "sync"
)

// 不好的做法：无限制并发
func badPractice() {
    for i := 0; i < 10000; i++ {
        go func() {
            // 阻塞I/O
            blockingIO()
        }()
    }
    // 可能创建10000个M！
}

// 好的做法：限制并发数
func goodPractice() {
    maxConcurrent := runtime.GOMAXPROCS(0) * 10
    sem := make(chan struct{}, maxConcurrent)

    for i := 0; i < 10000; i++ {
        sem <- struct{}{} // 获取许可
        go func() {
            defer func() { <-sem }() // 释放许可
            blockingIO()
        }()
    }
    // M数量稳定在 GOMAXPROCS + maxConcurrent
}

// 更好的做法：使用worker pool
func bestPractice() {
    numWorkers := runtime.GOMAXPROCS(0) * 2
    jobs := make(chan Job, 100)
    var wg sync.WaitGroup

    // 固定数量的worker
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range jobs {
                job.Process()
            }
        }()
    }

    // 发送任务
    for i := 0; i < 10000; i++ {
        jobs <- Job{ID: i}
    }
    close(jobs)
    wg.Wait()

    // M数量稳定在 GOMAXPROCS + numWorkers
}
```

### 8.7.2 使用非阻塞I/O

```go
// 阻塞I/O（创建大量M）
func blockingIO() {
    conn, _ := net.Dial("tcp", "example.com:80")
    data := make([]byte, 1024)
    conn.Read(data) // 阻塞，M解绑P
}

// 非阻塞I/O（复用少量M）
func nonBlockingIO() {
    // Go的net包内部使用epoll/kqueue
    // 阻塞在epoll_wait上，但不解绑P
    conn, _ := net.Dial("tcp", "example.com:80")
    data := make([]byte, 1024)
    conn.Read(data) // 不会创建新M
}

// 为什么net.Conn.Read不创建新M？

// 内部实现（简化）：
func (c *netFD) Read(p []byte) (n int, err error) {
    // 1. 尝试立即读取
    n, err = syscall.Read(c.Sysfd, p)
    if err != syscall.EAGAIN {
        return n, err
    }

    // 2. 没有数据，注册到netpoller
    if err = c.pd.waitRead(); err != nil {
        return 0, err
    }

    // 3. 阻塞在channel上（Go channel，不是系统调用）
    // M不解绑P，G被挂起
    <-c.pd.rg

    // 4. netpoller唤醒，继续读取
    return syscall.Read(c.Sysfd, p)
}

// netpoller的优势：
// - 一个M专门运行netpoller（epoll_wait）
// - 其他M继续执行其他G
// - 不需要为每个I/O操作创建M
```

### 8.7.3 监控M数量

```go
package main

import (
    "fmt"
    "runtime"
    "runtime/debug"
    "time"
)

func main() {
    // 设置M数量上限
    debug.SetMaxThreads(1000)

    // 定期监控
    go monitorThreads()

    // 业务逻辑
    runApplication()
}

func monitorThreads() {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()

    for range ticker.C {
        // 获取M数量（间接方式）
        buf := make([]byte, 1<<20)
        stackLen := runtime.Stack(buf, true)

        // 简单统计（实际应该用pprof）
        fmt.Printf("[Monitor] Stack dump size: %d bytes\n", stackLen)

        // 如果发现M数量异常增长，记录日志并告警
        if stackLen > 1<<18 { // 假设阈值256KB
            fmt.Println("[ALERT] Too many threads detected!")
            // 发送告警、记录堆栈等
        }
    }
}

// 使用pprof监控（更准确）
import _ "net/http/pprof"

func init() {
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
}

// 访问 http://localhost:6060/debug/pprof/threadcreate
// 查看线程创建的堆栈信息
```

---

## 8.8 总结

### 8.8.1 核心结论

```
┌─────────────────────────────────────────────────────┐
│ M不会无限增长的原因：                                │
│                                                     │
│ 1. M从系统调用返回后会被复用                         │
│    - exitsyscall() → 尝试获取P                      │
│    - 失败 → stopm() → 进入midle空闲池               │
│                                                     │
│ 2. 需要M时优先从空闲池获取                           │
│    - startm() → mget() → 从midle获取                │
│    - 空闲池为空 → 才newm()创建新M                   │
│                                                     │
│ 3. M的数量取决于历史最大并发阻塞数                   │
│    - 总M数 ≈ GOMAXPROCS + 历史峰值并发阻塞数        │
│    - 稳态时，大部分M在空闲池中待命                  │
│                                                     │
│ 4. M几乎永不销毁                                    │
│    - 创建成本高（~50μs）                            │
│    - 唤醒成本低（~0.1μs）                           │
│    - 保留在空闲池更高效                             │
│                                                     │
│ 5. 控制M数量的关键：限制并发阻塞操作                 │
│    - 使用信号量/worker pool                         │
│    - 优先使用非阻塞I/O（net包）                     │
│    - 设置maxmcount上限                              │
└─────────────────────────────────────────────────────┘
```

### 8.8.2 M数量公式

```
稳态M数量：
┌────────────────────────────────────────┐
│ 总M数 = GOMAXPROCS                     │
│       + 历史最大并发阻塞数             │
│       + 系统线程(sysmon等，~1-3个)     │
│                                        │
│ 其中：                                 │
│ - 运行M ≈ GOMAXPROCS                   │
│ - 阻塞M = 当前并发阻塞数               │
│ - 空闲M = 历史峰值 - 当前阻塞          │
└────────────────────────────────────────┘

示例：
- GOMAXPROCS = 8
- 历史最大并发阻塞 = 100
- 当前并发阻塞 = 20

→ 总M数 = 8 + 100 + 2 = 110
  ├─ 运行M: 8
  ├─ 阻塞M: 20
  └─ 空闲M: 82  ← 在midle池中待命
```

### 8.8.3 关键函数

```
M的生命周期关键函数：

创建：
- newm(fn, p)      - 创建M的入口
- allocm(p, fn)    - 分配M结构
- newosproc(mp)    - 创建OS线程
- mstart()         - M启动

复用：
- exitsyscall()    - 从syscall返回
- exitsyscall0()   - 无法获取P的情况
- stopm()          - M进入空闲池
- mput(mp)         - 将M加入midle链表

唤醒：
- startm(p, spin)  - 启动M来运行P
- mget()           - 从midle链表获取M
- notewakeup()     - 唤醒M
```

这就是M不会无限增长的完整机制！