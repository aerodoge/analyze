# OS 如何读取硬件信息（CPU核心数量等）

## 核心答案

**OS通过多种硬件接口读取硬件信息：**

```
x86/x64架构：
1. CPUID指令 - CPU直接提供信息（最直接）
2. ACPI表 - 固件提供的硬件配置表（最全面）
3. MP表 - 多处理器配置表（旧方式）
4. MSR寄存器 - 特殊寄存器（高级功能）

ARM架构：
1. Device Tree（设备树）- 固件传递的硬件描述
2. ACPI表 - 新版ARM也支持ACPI
3. CP15协处理器寄存器 - ARM特有

读取时机：
内核启动早期（start_kernel → setup_arch）
```

---

## 1. x86/x64 架构：CPUID指令

### 1.1 CPUID指令原理

```
CPUID是x86 CPU提供的特殊指令，用于查询CPU的功能和信息

指令格式：
  输入：EAX = 功能号（查询什么信息）
  输出：EAX, EBX, ECX, EDX = 返回值（信息）

示例：查询CPU厂商
  mov eax, 0        ; 功能0：查询厂商ID
  cpuid             ; 执行CPUID指令
  ; 结果：
  ; EAX = 最大支持的功能号
  ; EBX = "Genu"  (Intel: GenuineIntel)
  ; EDX = "ineI"
  ; ECX = "ntel"
```

### 1.2 CPUID数据的存储位置

**重要问题：CPUID指令返回的这些硬件信息存储在哪里？**

```
CPUID返回的数据存储在CPU芯片内部的多个位置：

┌────────────────────────────────────────────────────┐
│ CPU芯片内部                                         │
├────────────────────────────────────────────────────┤
│                                                    │
│ 1. 微码ROM (Microcode ROM)                          │
│    ├─ 位置：CPU核心内部的只读存储器                    │
│    ├─ 容量：几KB到几十KB                             │
│    ├─ 存储内容：                                     │
│    │   - CPU厂商字符串 ("GenuineIntel")              │
│    │   - 基本功能标志位                              │
│    │   - CPUID指令的处理逻辑（微码）                  │
│    └─ 特点：制造时固化，无法修改                       │
│                                                    │
│ 2. 电子熔丝 (eFuses / One-Time Programmable)        │
│    ├─ 位置：CPU芯片内的熔丝阵列                       │
│    ├─ 容量：几百到几千bit                            │
│    ├─ 存储内容：                                    │
│    │   - CPU型号和步进 (Family, Model, Stepping)    │
│    │   - 最大频率                                   │
│    │   - 核心数量                                   │
│    │   - 功能禁用位 (哪些功能被禁用)                  │
│    │   - 唯一序列号 (PSN - Processor Serial Number) │
│    │   - 芯片测试时的binning结果                     │
│    └─ 特点：生产后期激光熔断，一次性写入                │
│                                                    │
│ 3. 配置寄存器 (Configuration Registers)             │
│    ├─ 位置：CPU内部的特殊寄存器                       │
│    ├─ 存储内容：                                    │
│    │   - 当前运行频率                               │
│    │   - 缓存配置                                   │
│    │   - TLB大小                                   │
│    │   - 超线程启用状态                              │
│    └─ 特点：启动时初始化，运行时可能动态变化             │
│                                                    │
│ 4. 硬件检测逻辑 (Hardware Detection Logic)           │
│    ├─ 位置：CPU内的硬件电路                           │
│    ├─ 检测内容：                                    │
│    │   - 实际启用的核心数量                           │
│    │   - 缓存大小（通过cache controller）            │
│    │   - APIC ID (每个逻辑核的唯一ID)                │
│    └─ 特点：动态检测，反映当前硬件状态                  │
│                                                    │
│ 5. 微码更新区 (Microcode Update RAM)                 │
│    ├─ 位置：CPU内的小块SRAM                          │
│    ├─ 用途：可通过BIOS/OS更新微码                     │
│    └─ 影响：可能修改CPUID返回的某些标志位               │
└────────────────────────────────────────────────────┘
```

#### 详细解释各个存储位置

**1. 微码ROM - 固化的CPU逻辑**

```
微码ROM是什么？

┌────────────────────────────────────────┐
│ CPU制造流程：                            │
│                                        │
│ 晶圆制造 → 光刻 → 蚀刻 → 金属层            │
│                    ↓                   │
│               微码ROM层                 │
│        ┌───────────────────┐           │
│        │ 0101010101010...  │           │
│        │ (微码指令)         │           │
│        └───────────────────┘           │
│                                        │
│ 微码ROM在制造时就被"硬编码"到硅片中         │
│ 类似于光盘的数据，无法修改                 │
└────────────────────────────────────────┘

存储内容示例：

地址 0x0000: "GenuineIntel" 字符串
地址 0x000C: 厂商ID = 0x8086
地址 0x0010: 基本功能标志
地址 0x0100: CPUID处理微码
...

CPUID指令执行时：
1. CPU硬件解码CPUID指令
2. 根据EAX中的功能号，查找微码ROM中的数据
3. 将数据加载到EAX/EBX/ECX/EDX寄存器
4. 返回给软件
```

**2. 电子熔丝 (eFuses) - 激光刻录的配置**

```
什么是eFuses？

制造过程中的binning（分级）：

Step 1: 晶圆制造
┌────────────────────────────────────┐
│ Intel 12代酷睿晶圆                   │
│ 所有芯片最初都是相同的设计：            │
│ - 设计：16个P核心 + 8个E核心          │
│ - 每个核心都制造出来                  │
└────────────────────────────────────┘

Step 2: 测试（Test & Binning）
┌────────────────────────────────────┐
│ 每个芯片在工厂测试：                  │
│                                    │
│ 芯片A:                              │
│ ✓ 16个P核心全部工作                  │ 
│ ✓ 所有缓存工作                       │
│ ✓ 可达5.0GHz                        │
│ → 标记为 i9-12900K                  │
│                                    │
│ 芯片B:                              │
│ ✓ 12个P核心工作                      │
│ ✗ 4个P核心有缺陷                     │
│ ✓ 最高4.5GHz                        │
│ → 标记为 i7-12700K                  │
│                                    │
│ 芯片C:                              │
│ ✓ 8个P核心工作                       │
│ ✗ 8个P核心有缺陷                     │
│ ✓ 最高4.0GHz                        │
│ → 标记为 i5-12600K                  │
└────────────────────────────────────┘

Step 3: 激光熔断 (Laser Fusing)
┌────────────────────────────────────┐
│ 用激光熔断特定的熔丝：                 │
│                                    │
│ 对于芯片B (i7-12700K):              │
│                                    │
│ eFuse Array:                       │
│ ┌─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐  │
│ │ │ │ │ │ │ │ │ │X│X│X│X│ │ │ │ │  │
│ └─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘  │
│  核心编号:                          │
│  0 1 2 3 4 5 6 7 8 9 A B C D E F   │
│                  ↑ ↑ ↑ ↑           │
│              熔断（禁用）            │
│                                    │
│ 频率熔丝:                           │
│ MaxFreq = 4.5GHz (二进制编码)       │
│                                    │
│ 型号熔丝:                           │
│ ModelID = 0x9A (i7-12700K)         │
└────────────────────────────────────┘

熔断后：
- 物理上核心8-11还在芯片上
- 但eFuse标记它们为"禁用"
- CPUID读取eFuse，返回"12核"
- 硬件启动时跳过禁用的核心
```

**eFuses的物理结构**：

```
┌────────────────────────────────────────────┐
│ eFuse单元（纳米级）                          │
├────────────────────────────────────────────┤
│                                            │
│ 未熔断状态（默认）：                          │
│                                            │
│   ───[熔丝]───                             │
│       完整    → 读取为 "1" (或 "0")          │
│                                            │
│ 熔断状态（激光）：                            │
│                                            │
│   ───  X  ───                              │
│      断开    → 读取为 "0" (或 "1")           │
│                                            │
│ 一次性操作，不可逆！                          │
└────────────────────────────────────────────┘

eFuse Array 示例（Intel CPU）：

Bit 0-7:   CPU Family    (0x06 = Core系列)
Bit 8-15:  Model ID      (0x9A = 12代酷睿)
Bit 16-19: Stepping      (0x03 = C步进)
Bit 20-27: Core Count    (0x0C = 12核)
Bit 28-35: Max Freq      (0x2D = 4500MHz)
Bit 36-43: Cache Size    (0x1E = 30MB L3)
Bit 44-51: TDP           (0x7D = 125W)
Bit 52-59: Feature Mask  (哪些指令集启用)
Bit 60-127: Reserved
Bit 128-255: Unique Serial Number
...

总计：几百到几千bit
```

**3. 配置寄存器 - 动态状态**

```
配置寄存器在CPU启动时初始化：

┌────────────────────────────────────────┐
│ CPU上电流程：                            │
│                                        │
│ 1. 复位                                │
│    - 所有寄存器清零                      │
│    - 进入固件（microcode）               │
│                                        │
│ 2. 读取eFuses                           │
│    - 读取核心数量                        │
│    - 读取频率限制                        │
│    - 读取功能启用状态                     │
│                                        │
│ 3. 初始化配置寄存器                       │
│    ┌─────────────────────────┐         │
│    │ MSR 0x35 (Core Count)   │         │
│    │ = 读取eFuse → 12         │         │
│    ├─────────────────────────┤         │
│    │ MSR 0xCE (Frequency)    │         │
│    │ = 计算当前频率 → 2400MHz  │         │
│    ├─────────────────────────┤         │
│    │ CPUID_Config_Reg        │         │
│    │ = 缓存大小、TLB等         │         │
│    └─────────────────────────┘         │
│                                        │
│ 4. 启动核心                             │
│    - 只启动eFuse标记为启用的核心           │
│    - 禁用的核心保持关闭                   │
│                                        │
│ 5. CPUID指令准备就绪                     │
│    - 可以读取这些配置寄存器                │
└────────────────────────────────────────┘
```

**4. 硬件检测逻辑 - 实时检测**

```
某些CPUID信息是硬件动态检测的：

示例：缓存大小检测

┌────────────────────────────────────────┐
│ CPU启动时的缓存检测：                     │
│                                        │
│ 1. Cache Controller自检                 │
│    for (way = 0; way < 16; way++)      │
│     {                                  │
│        for (set = 0; set < 512; set++) │
│         {                              │
│            test_cache_line(way, set);  │
│            if (error)                  │
│            {                           │
│                disable_way(way);       │
│                bad_ways++;             │
│            }                           │
│        }                               │
│    }                                   │
│                                        │
│ 2. 计算实际可用缓存                       │
│    total_ways = 16 - bad_ways          │
│    usable_cache = total_ways × 64KB    │
│                                        │
│ 3. 更新CPUID返回值                       │
│    CPUID_Cache_Size = usable_cache     │
│                                        │
│ 这就是为什么有些芯片L3缓存"缺一块"          │
└────────────────────────────────────────┘

示例：APIC ID分配

┌────────────────────────────────────────┐
│ 每个逻辑核心的APIC ID是硬件分配的：         │
│                                        │
│ 启动时：                                │
│ 物理核心0 → 逻辑核心0 → APIC ID = 0       │
│          → 逻辑核心1 → APIC ID = 1       │
│ 物理核心1 → 逻辑核心2 → APIC ID = 2       │
│          → 逻辑核心3 → APIC ID = 3       │
│ ...                                    │
│                                        │
│ CPUID 0xB返回的APIC ID是硬件检测的        │
│ 不是预先存储的                           │
└────────────────────────────────────────┘
```

**5. 微码更新 - 可修改的部分**

```
为什么需要微码更新？

问题：CPU出厂后发现bug或安全漏洞
解决：通过微码更新修复

微码更新流程：

1. Intel发布微码更新包 (.dat文件)
2. BIOS厂商集成到BIOS
3. 或OS在启动时加载 (Linux: /lib/firmware/intel-ucode/)

更新机制：
┌────────────────────────────────────────┐
│ CPU内部：                               │
│                                        │
│ 原始微码ROM (只读)                       │
│ ┌────────────────┐                     │
│ │ 原始CPUID逻辑   │ ← 有bug              │
│ └────────────────┘                     │
│         ↓ 被覆盖                        │
│ 微码更新RAM (SRAM)                      │
│ ┌────────────────┐                     │
│ │ 修复的逻辑       │ ← 加载到这里         │
│ └────────────────┘                     │
│         ↓ 优先使用                      │
│    CPUID执行                            │
└────────────────────────────────────────┘

示例：Spectre漏洞修复
- 原始ROM: CPUID.7H.EDX[26] = 0 (无IBRS)
- 微码更新后: CPUID.7H.EDX[26] = 1 (支持IBRS)
- CPUID返回值变化！

查看微码版本：
$ cat /proc/cpuinfo | grep microcode
microcode  : 0xf0
```

#### CPUID数据存储总结

```
CPUID返回的不同类型信息，来自不同位置：

┌───────────────────────┬─────────────────────────┐
│ 信息类型               │ 存储位置                  │
├───────────────────────┼─────────────────────────┤
│ 厂商字符串             │ 微码ROM (固化)            │
│ Family/Model/Stepping │ eFuses (激光熔断)        │
│ 核心数量               │ eFuses + 硬件检测        │
│ 最大频率               │ eFuses                  │
│ 当前频率               │ 配置寄存器 (动态)          │
│ 功能标志 (SSE等)       │ eFuses + 微码更新         │
│ 缓存大小               │ 硬件检测 + eFuses         │
│ APIC ID              │ 硬件分配 (动态)            │
│ TLB大小               │ 配置寄存器                │
│ 序列号                │ eFuses (唯一)             │
│ 安全特性标志           │ 微码更新 (可修改)          │
└──────────────────────┴──────────────────────────┘

关键洞察：
1. 大部分信息在制造时就固化了 (ROM + eFuses)
2. 部分信息是启动时检测的 (配置寄存器)
3. 少量信息可以通过微码更新修改
4. 所有信息都在CPU芯片内部，不依赖外部存储
```

### 1.3 查询CPU核心数量

```c
// arch/x86/kernel/cpu/common.c

// 查询CPU拓扑信息
void detect_ht(struct cpuinfo_x86 *c)
{
    u32 eax, ebx, ecx, edx;

    // 功能号1：基本CPU信息
    cpuid(1, &eax, &ebx, &ecx, &edx);

    // EBX[23:16] = 逻辑处理器数量（每个物理package）
    unsigned int logical_processors = (ebx >> 16) & 0xff;

    // EDX[28] = 超线程标志位
    if (edx & (1 << 28)) 
    {
        printk("HT detected: %d logical processors per package\n", logical_processors);
        c->x86_max_cores = logical_processors;
    }

    // 功能号4：缓存和拓扑信息（更详细）
    cpuid_count(4, 0, &eax, &ebx, &ecx, &edx);

    // EAX[31:26] = 共享此缓存的核心数 - 1
    unsigned int cores_per_package = ((eax >> 26) & 0x3f) + 1;

    c->x86_max_cores = cores_per_package;
}

// 功能号0xB：扩展拓扑枚举（Intel最新）
void detect_extended_topology(struct cpuinfo_x86 *c)
{
    u32 eax, ebx, ecx, edx;
    int level = 0;

    do 
    {
        // 功能号0xB，ECX=level
        cpuid_count(0xB, level, &eax, &ebx, &ecx, &edx);

        int level_type = (ecx >> 8) & 0xff;
        int num_logical = ebx & 0xffff;

        switch (level_type) 
        {
        case 1:  // SMT级别（超线程）
            c->x86_max_cores = num_logical;
            break;
        case 2:  // Core级别（物理核心）
            c->phys_proc_id = edx;  // Package ID
            break;
        }

        level++;
    } while ((ecx & 0xff00) != 0);
}

// AMD的方式（功能号0x80000008）
void amd_get_topology(struct cpuinfo_x86 *c)
{
    u32 eax, ebx, ecx, edx;

    // AMD扩展功能
    cpuid(0x80000008, &eax, &ebx, &ecx, &edx);

    // ECX[7:0] = 物理核心数 - 1
    c->x86_max_cores = (ecx & 0xff) + 1;
}
```

### 1.4 完整的CPU检测代码

```c
// arch/x86/kernel/cpu/common.c

void __init identify_boot_cpu(void)
{
    struct cpuinfo_x86 *c = &boot_cpu_data;

    // 1. 检测CPU厂商
    cpuid(0, &c->cpuid_level, (int *)&c->x86_vendor_id[0], (int *)&c->x86_vendor_id[8], (int *)&c->x86_vendor_id[4]);

    // 2. 检测CPU功能
    cpuid(1, &c->x86, &c->x86_capability[CPUID_1_EBX], &c->x86_capability[CPUID_1_ECX], &c->x86_capability[CPUID_1_EDX]);

    // 3. 检测拓扑信息
    if (c->cpuid_level >= 0xB) 
    {
        // 使用扩展拓扑枚举（最新最准确）
        detect_extended_topology(c);
    } 
    else if (c->cpuid_level >= 4) 
    {
        // 使用功能4（旧方式）
        detect_ht(c);
    }

    // 4. 检测缓存信息
    init_intel_cacheinfo(c);

    // 5. 打印信息
    printk(KERN_INFO "CPU: Physical Processor ID: %d\n", c->phys_proc_id);
    printk(KERN_INFO "CPU: Cores per package: %d\n", c->x86_max_cores);
}
```

---

## 2. ACPI 表（高级配置和电源接口）

### 2.1 ACPI 是什么？

```
ACPI（Advanced Configuration and Power Interface）：
- 由BIOS/UEFI在内存中创建的数据表
- 包含硬件配置信息（CPU、内存、设备等）
- OS通过读取这些表了解硬件

ACPI表在内存中的位置：
┌────────────────────────────────────────┐
│ 物理内存                                │
│                                        │
│ 0x00000000 - 0x000FFFFF (1MB)          │
│ ├── BIOS区域                            │
│ │   └── ACPI表（RSDP）                  │
│ │       └── 指向其他ACPI表               │
│                                        │
│ 0x00100000 - ... (高地址)               │
│ └── 内核加载地址                         │
└────────────────────────────────────────┘

主要ACPI表：
1. RSDP (Root System Description Pointer) - 根表指针
2. RSDT (Root System Description Table) - 根表
3. MADT (Multiple APIC Description Table) - 多处理器信息
4. SRAT (System Resource Affinity Table) - NUMA信息
5. SLIT (System Locality Information Table) - NUMA距离
```

### 2.2 通过MADT读取CPU信息

```c
// drivers/acpi/tables.c

// 查找ACPI表
struct acpi_table_header *acpi_find_table(char *signature)
{
    // 1. 找到RSDP（Root System Description Pointer）
    struct acpi_table_rsdp *rsdp;
    rsdp = acpi_find_rsdp();  // 在BIOS区域扫描签名"RSD PTR "

    // 2. 通过RSDP找到RSDT（Root System Description Table）
    struct acpi_table_rsdt *rsdt;
    rsdt = (struct acpi_table_rsdt *)rsdp->rsdt_physical_address;

    // 3. 在RSDT中查找指定的表（如MADT）
    for (int i = 0; i < rsdt->entry_count; i++) 
    {
        struct acpi_table_header *table;
        table = (struct acpi_table_header *)rsdt->entry[i];

        if (memcmp(table->signature, signature, 4) == 0) 
        {
            return table;
        }
    }

    return NULL;
}

// 解析MADT（Multiple APIC Description Table）
void __init acpi_parse_madt(void)
{
    struct acpi_table_madt *madt;

    // 1. 查找MADT表
    madt = (struct acpi_table_madt *)acpi_find_table("APIC");
    if (!madt) 
    {
        printk("No MADT found\n");
        return;
    }

    // 2. 遍历MADT中的子结构
    u8 *p = (u8 *)(madt + 1);  // 跳过表头
    u8 *end = (u8 *)madt + madt->header.length;

    while (p < end) 
    {
        struct acpi_subtable_header *header;
        header = (struct acpi_subtable_header *)p;

        switch (header->type) 
        {
        case ACPI_MADT_TYPE_LOCAL_APIC:  // 本地APIC（每个CPU核心一个）
            {
                struct acpi_madt_local_apic *lapic;
                lapic = (struct acpi_madt_local_apic *)p;

                if (lapic->lapic_flags & ACPI_MADT_ENABLED) 
                {
                    // 找到一个CPU核心！
                    int cpu_id = lapic->processor_id;
                    int apic_id = lapic->id;

                    printk("CPU %d: APIC ID %d\n", cpu_id, apic_id);

                    // 注册CPU
                    set_cpu_possible(cpu_id, true);
                    set_cpu_present(cpu_id, true);
                    nr_cpus++;
                }
            }
            break;

        case ACPI_MADT_TYPE_IO_APIC:  // I/O APIC（中断控制器）
            // 处理I/O APIC
            break;

        // 其他类型...
        }

        p += header->length;
    }

    printk("ACPI: %d CPUs detected\n", nr_cpus);
}

// MADT表结构示例：
struct acpi_table_madt 
{
    struct acpi_table_header header;     // 签名："APIC"
    u32 address;                         // 本地APIC地址
    u32 flags;
    // 后面跟着多个子结构（Local APIC, I/O APIC等）
};

struct acpi_madt_local_apic 
{
    struct acpi_subtable_header header;  // type=0（Local APIC）
    u8 processor_id;                     // CPU ID
    u8 id;                               // APIC ID
    u32 lapic_flags;                     // 标志（enabled/disabled）
};
```

---

## 3. 完整的硬件发现流程

### 3.1 内核启动时的硬件检测

```c
// init/main.c

asmlinkage __visible void __init start_kernel(void)
{
    // ... 早期初始化 ...

    // 1. 架构相关初始化（关键！）
    setup_arch(&command_line);
}

// arch/x86/kernel/setup.c

void __init setup_arch(char **cmdline_p)
{
    // 1. 早期ACPI初始化
    acpi_boot_table_init();   // 查找ACPI表

    // 2. CPUID检测（引导CPU）
    early_identify_cpu(&boot_cpu_data);

    // 3. 解析ACPI MADT表（发现所有CPU）
    acpi_boot_init();         // 解析MADT，发现所有CPU核心

    // 4. 设置Per-CPU区域
    setup_per_cpu_areas();    // 为所有发现的CPU分配内存

    // 5. SMP初始化
    smp_prepare_cpus(nr_cpus);  // 准备启动其他CPU
}

// arch/x86/kernel/acpi/boot.c

int __init acpi_boot_init(void)
{
    // 1. 解析MADT表（CPU信息）
    acpi_table_parse(ACPI_SIG_MADT, acpi_parse_madt);

    // 2. 处理每个Local APIC
    acpi_table_parse_madt(ACPI_MADT_TYPE_LOCAL_APIC,
                          acpi_parse_lapic, 0);

    // 3. 处理I/O APIC
    acpi_table_parse_madt(ACPI_MADT_TYPE_IO_APIC,
                          acpi_parse_ioapic, 0);

    // 4. 打印发现的CPU数量
    printk(KERN_INFO "ACPI: %d CPUs available\n",
           num_present_cpus());

    return 0;
}
```

### 3.2 数据流向图

```
系统启动流程：

T1: BIOS/UEFI初始化
    ├── 检测硬件（CPU、内存、设备）
    ├── 在内存中创建ACPI表
    │   └── MADT表包含所有CPU信息
    └── 跳转到内核入口

T2: 内核start_kernel()
    ↓
T3: setup_arch()
    ├── acpi_boot_table_init()
    │   └── 在内存0x0-0xFFFFF扫描RSDP签名
    │       找到RSDP → RSDT → MADT
    ↓
T4: early_identify_cpu()
    ├── 执行CPUID指令（引导CPU）
    │   cpuid(0): 厂商ID = "GenuineIntel"
    │   cpuid(1): 功能信息
    │   cpuid(4): 缓存拓扑
    │   cpuid(0xB): 扩展拓扑
    └── 填充boot_cpu_data结构
    ↓
T5: acpi_boot_init()
    ├── 解析MADT表
    │   遍历Local APIC条目：
    │   ├── CPU 0: APIC ID=0 (enabled)
    │   ├── CPU 1: APIC ID=2 (enabled)
    │   ├── CPU 2: APIC ID=4 (enabled)
    │   ...
    │   └── CPU 19: APIC ID=38 (enabled)
    └── 设置CPU位图
        set_cpu_possible(0-19, true)
        nr_cpu_ids = 20
    ↓
T6: setup_per_cpu_areas()
    └── 为20个CPU分配Per-CPU内存区域
    ↓
T7: sched_init()
    └── 为20个CPU初始化运行队列
    ↓
T8: smp_prepare_cpus()
    └── 启动其他19个CPU核心
```

---

## 4. 实际内存中的 ACPI 表

### 4.1 ACPI 表的内存布局

```
物理内存（示例）：
┌──────────────────────────────────────────────────┐
│ 0x000F0000 - 0x000FFFFF (BIOS区域)               │
│ ┌────────────────────────────────────────────┐   │
│ │ 0x000F1234: RSDP (Root System Desc Pointer)│   │
│ │ ┌────────────────────────────────────────┐ │   │
│ │ │ Signature: "RSD PTR "                  │ │   │
│ │ │ Checksum: 0x42                         │ │   │
│ │ │ OEM ID: "INTEL "                       │ │   │
│ │ │ Revision: 2                            │ │   │
│ │ │ RSDT Address: 0x1000 (物理地址)         │ │   │
│ │ │ XSDT Address: 0x2000 (64位)            │ │   │
│ │ └────────────────────────────────────────┘ │   │
│ └────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ 0x00001000: RSDT (Root System Description Table) │
│ ┌────────────────────────────────────────────┐   │
│ │ Signature: "RSDT"                          │   │
│ │ Length: 64 bytes                           │   │
│ │ Revision: 1                                │   │
│ │ Entry[0]: 0x3000 (指向FACP)                 │   │
│ │ Entry[1]: 0x4000 (指向APIC/MADT)            │   │
│ │ Entry[2]: 0x5000 (指向SRAT)                 │   │
│ │ Entry[3]: 0x6000 (指向HPET)                 │   │
│ └────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│ 0x00004000: MADT (Multiple APIC Desc Table)      │
│ ┌────────────────────────────────────────────┐   │
│ │ Signature: "APIC"                          │   │
│ │ Length: 512 bytes                          │   │
│ │ Local APIC Address: 0xFEE00000             │   │
│ │ Flags: 0x01                                │   │
│ │                                            │   │
│ │ Sub-structures:                            │   │
│ │ ┌────────────────────────────────────────┐ │   │
│ │ │ Type: 0 (Local APIC)                   │ │   │
│ │ │ Length: 8                              │ │   │
│ │ │ Processor ID: 0                        │ │   │
│ │ │ APIC ID: 0                             │ │   │
│ │ │ Flags: 0x01 (Enabled)                  │ │   │
│ │ └────────────────────────────────────────┘ │   │
│ │ ┌────────────────────────────────────────┐ │   │
│ │ │ Type: 0 (Local APIC)                   │ │   │
│ │ │ Processor ID: 1                        │ │   │
│ │ │ APIC ID: 2                             │ │   │
│ │ │ Flags: 0x01 (Enabled)                  │ │   │
│ │ └────────────────────────────────────────┘ │   │
│ │ ... (重复20次，每个CPU一个)                   │   │
│ └────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

### 4.2 内核读取 ACPI 表的代码

```c
// drivers/acpi/tables.c

// 1. 查找RSDP（在BIOS区域扫描）
static struct acpi_table_rsdp *acpi_find_rsdp(void)
{
    void *start = (void *)0x000E0000;  // BIOS区域起始
    void *end   = (void *)0x00100000;  // BIOS区域结束
    void *p;

    // 扫描内存，查找"RSD PTR "签名（每16字节对齐）
    for (p = start; p < end; p += 16) 
    {
        if (memcmp(p, "RSD PTR ", 8) == 0) 
        {
            // 验证校验和
            struct acpi_table_rsdp *rsdp = p;
            u8 checksum = 0;
            for (int i = 0; i < 20; i++) 
            {
                checksum += ((u8 *)rsdp)[i];
            }
            if (checksum == 0) 
            {
                printk("ACPI: RSDP found at %p\n", p);
                return rsdp;
            }
        }
    }

    return NULL;
}

// 2. 通过RSDP读取RSDT
static struct acpi_table_rsdt *acpi_get_rsdt(void)
{
    struct acpi_table_rsdp *rsdp = acpi_find_rsdp();
    if (!rsdp)
        return NULL;

    // RSDP包含RSDT的物理地址
    void *rsdt_phys = (void *)(unsigned long)rsdp->rsdt_physical_address;

    // 映射物理地址到虚拟地址（early_ioremap）
    struct acpi_table_rsdt *rsdt;
    rsdt = early_ioremap(rsdt_phys, sizeof(*rsdt));

    return rsdt;
}

// 3. 在RSDT中查找MADT
static struct acpi_table_madt *acpi_get_madt(void)
{
    struct acpi_table_rsdt *rsdt = acpi_get_rsdt();

    // 遍历RSDT的条目
    int entries = (rsdt->header.length - sizeof(rsdt->header)) / sizeof(u32);

    for (int i = 0; i < entries; i++) 
    {
        void *table_phys = (void *)(unsigned long)rsdt->entry[i];

        // 映射表头
        struct acpi_table_header *header;
        header = early_ioremap(table_phys, sizeof(*header));

        // 检查签名
        if (memcmp(header->signature, "APIC", 4) == 0) 
        {
            // 找到MADT！
            struct acpi_table_madt *madt;
            madt = early_ioremap(table_phys, header->length);
            return madt;
        }

        early_iounmap(header, sizeof(*header));
    }

    return NULL;
}
```

---

## 5. 查看系统的 ACPI 表

### 5.1 Linux 用户空间查看

```bash
# 查看所有ACPI表
$ ls /sys/firmware/acpi/tables/
APIC  BGRT  DSDT  FACP  FACS  HPET  MCFG  SRAT  SSDT  WAET

# 读取MADT（APIC）表
$ sudo cat /sys/firmware/acpi/tables/APIC | hexdump -C
00000000  41 50 49 43 90 00 00 00  03 76 49 4e 54 45 4c 20  |APIC.....vINTEL |
00000010  54 65 6d 70 6c 61 74 65  00 00 00 00 49 4e 54 4c  |Template....INTL|
00000020  24 07 13 20 00 00 e0 fe  01 00 00 00 00 08 00 00  |$.. ............|
00000030  00 00 01 00 00 08 01 02  00 00 01 00 00 08 02 04  |................|
         ↑         ↑  ↑  ↑
      Type=0    Len=8 PID APIC ID

# 使用acpidump工具（更友好）
$ sudo acpidump > acpi_tables.dat
$ acpixtract -a acpi_tables.dat
$ iasl -d APIC.dat  # 反汇编MADT表

# 输出示例：
/*
 * Intel ACPI Component Architecture
 * MADT (Multiple APIC Description Table)
 *
 * Local APIC Address : 0xFEE00000
 *
 * Sub-tables:
 *   [000h 0000  1]      Sub-table Type : 00 [Processor Local APIC]
 *   [001h 0001  1]             Length : 08
 *   [002h 0002  1]      Processor ID : 00
 *   [003h 0003  1]             Local APIC ID : 00
 *   [004h 0004  4]       Flags (decoded below) : 00000001
 *                         Processor Enabled : 1
 *
 *   [008h 0008  1]      Sub-table Type : 00 [Processor Local APIC]
 *   [009h 0009  1]             Length : 08
 *   [00Ah 0010  1]      Processor ID : 01
 *   [00Bh 0011  1]             Local APIC ID : 02
 *   [00Ch 0012  4]       Flags (decoded below) : 00000001
 *                         Processor Enabled : 1
 *   ...
 */
```

### 5.2 查看内核检测到的CPU

```bash
# 查看CPU信息
$ lscpu
Architecture:        x86_64
CPU(s):              20           ← 内核检测到20个逻辑核心
Thread(s) per core:  2
Core(s) per socket:  10
Socket(s):           1

# 查看/proc/cpuinfo（每个CPU一项）
$ cat /proc/cpuinfo
processor       : 0      ← ACPI MADT中的Processor ID
physical id     : 0      ← Package ID
core id         : 0      ← Core ID
apicid          : 0      ← ACPI MADT中的APIC ID

processor       : 1
physical id     : 0
core id         : 0      ← 与processor 0相同core id（超线程伙伴）
apicid          : 2      ← APIC ID不同

processor       : 2
physical id     : 0
core id         : 1      ← 不同的core（物理核心1）
apicid          : 4

# 查看内核日志
$ dmesg | grep -E "CPU|ACPI"
[    0.000000] ACPI: RSDP 0x00000000000F1234 000024 (v02 INTEL)
[    0.000000] ACPI: RSDT 0x0000000000001000 000040 (v01 INTEL  Template 00000001 INTL 20130517)
[    0.000000] ACPI: MADT 0x0000000000004000 000200 (v03 INTEL  Template 00000001 INTL 20130517)
[    0.000000] ACPI: Local APIC address 0xfee00000
[    0.000000] ACPI: LAPIC (acpi_id[0x00] lapic_id[0x00] enabled)
[    0.000000] ACPI: LAPIC (acpi_id[0x01] lapic_id[0x02] enabled)
...
[    0.000000] ACPI: 20 CPUs available
```

---

## 6. ARM 架构：设备树（Device Tree）

### 6.1 设备树简介

```
ARM系统没有类似BIOS/ACPI的标准固件接口
解决方案：设备树（Device Tree Blob, DTB）

设备树：
- 一个描述硬件的文本文件（编译成二进制）
- 由Bootloader（如U-Boot）传递给内核
- 包含CPU、内存、设备的所有信息

示例（.dts文件）：
/ {
    #address-cells = <2>;
    #size-cells = <2>;

    cpus {
        #address-cells = <1>;
        #size-cells = <0>;

        cpu@0 {
            device_type = "cpu";
            compatible = "arm,cortex-a72";
            reg = <0>;
            enable-method = "psci";
        };

        cpu@1 {
            device_type = "cpu";
            compatible = "arm,cortex-a72";
            reg = <1>;
            enable-method = "psci";
        };

        cpu@2 {
            device_type = "cpu";
            compatible = "arm,cortex-a72";
            reg = <2>;
            enable-method = "psci";
        };

        cpu@3 {
            device_type = "cpu";
            compatible = "arm,cortex-a72";
            reg = <3>;
            enable-method = "psci";
        };
    };

    memory@0 {
        device_type = "memory";
        reg = <0x0 0x00000000 0x0 0x80000000>;  // 2GB内存
    };
};
```

### 6.2 内核解析设备树

```c
// arch/arm64/kernel/setup.c

void __init setup_arch(char **cmdline_p)
{
    // 1. 设备树地址由Bootloader传递（在寄存器x0中）
    void *dt_virt = fixmap_remap_fdt(__fdt_pointer);

    // 2. 解析设备树
    unflatten_device_tree();

    // 3. 查找CPU节点
    of_scan_flat_dt(early_init_dt_scan_cpus, NULL);
}

// drivers/of/fdt.c

int __init early_init_dt_scan_cpus(unsigned long node,
                                   const char *uname, int depth, void *data)
{
    // 查找"cpus"节点的子节点
    if (strcmp(uname, "cpus") == 0)
        return 0;  // 进入cpus节点

    // 检查是否是cpu节点
    const char *type = of_get_flat_dt_prop(node, "device_type", NULL);
    if (!type || strcmp(type, "cpu") != 0)
        return 0;

    // 读取CPU ID
    const __be32 *reg = of_get_flat_dt_prop(node, "reg", NULL);
    int cpu_id = be32_to_cpup(reg);

    printk("Found CPU%d\n", cpu_id);

    // 注册CPU
    set_cpu_possible(cpu_id, true);
    set_cpu_present(cpu_id, true);

    return 0;
}
```

---

## 7. 面试回答模板

**问题**："OS怎么知道有多少个核心？在哪里读的硬件信息？"

**标准答案**：

"OS通过多种硬件接口读取CPU核心数量等硬件信息，具体方式取决于架构：

**x86/x64架构（主要方式）**：

1. **CPUID指令**（最直接）：
   - CPU提供的特殊指令，查询CPU功能
   - 执行`cpuid(0xB)`获取扩展拓扑信息
   - 返回物理核心数、逻辑核心数、超线程信息
   - 时机：内核启动时`early_identify_cpu()`

2. **ACPI MADT表**（最全面）：
   - BIOS/UEFI在物理内存0x0-0xFFFFF区域创建的表
   - 包含所有CPU的信息（APIC ID、状态等）
   - 内核通过扫描"RSD PTR "签名找到RSDP → RSDT → MADT
   - 解析MADT的Local APIC条目，每个条目代表一个逻辑核心
   - 时机：`setup_arch()` → `acpi_boot_init()`

3. **MP表**（旧方式）：
   - 多处理器配置表，ACPI之前的标准
   - 现代系统不常用

**ARM架构**：

1. **设备树（Device Tree）**：
   - Bootloader传递的硬件描述文件
   - 包含cpus节点，每个子节点代表一个CPU
   - 内核解析设备树，调用`of_scan_flat_dt()`

2. **ACPI**：
   - 新版ARM服务器也支持ACPI（类似x86）

**读取时机**：
内核启动早期，`start_kernel()` → `setup_arch()`阶段，这时还没有调度器，只有引导CPU在运行。

**数据流向**：
```
BIOS/UEFI创建ACPI表 → 内核扫描物理内存找到RSDP →
解析RSDT → 找到MADT → 遍历Local APIC条目 →
注册CPU（set_cpu_possible）→ 分配Per-CPU区域 →
初始化运行队列 → 启动其他CPU
```

**验证方式**：
- 用户空间：`lscpu`、`cat /proc/cpuinfo`
- 查看ACPI表：`/sys/firmware/acpi/tables/APIC`
- 内核日志：`dmesg | grep ACPI`
"

---

## 8. 总结

```
关键点：
1. OS不是"猜"CPU数量，而是通过硬件接口读取
2. x86: CPUID指令（CPU提供）+ ACPI表（BIOS提供）
3. ARM: 设备树（Bootloader提供）
4. 读取时机：内核启动早期（调度器初始化之前）
5. 数据存储：设置CPU位图（cpu_possible_mask等）

核心数据结构：
- cpu_possible_mask: 所有可能的CPU（包括未上线的）
- cpu_present_mask: 当前存在的CPU
- cpu_online_mask: 当前在线的CPU
- cpu_active_mask: 当前激活的CPU（可以被调度器使用）

这些位图在解析ACPI/设备树后被设置，指导后续的Per-CPU区域分配和调度器初始化。
```

**核心思想是：OS通过标准化的硬件接口（CPUID、ACPI、设备树）读取硬件信息，而不是通过某种"魔法"猜测。
这些接口在不同架构上有不同的实现，但目的都是让OS能准确地了解硬件配置。**
