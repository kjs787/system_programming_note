---
name: refactor
trigger: /refactor
description: 将零散学习笔记重构为「背景/原理 → 代码实现 → 关键点分析 → 避坑指南」四段式的工程级知识条目
version: 1.0.0
args: "<文件路径 | 粘贴内容> [--level=light|full] [--no-diff] [--lang=c|rust|auto]"
---

# 技能：笔记标准化重构

## 1. 触发与用途

当用户输入 `/refactor` 时启用本技能，对一篇（或一段）学习笔记执行**结构化重写**，使其达到"可直接进入团队知识库"的标准。

默认参数：

| 参数 | 取值 | 说明 |
| --- | --- | --- |
| `--level` | `full`（默认）/ `light` | `full` 重写全文；`light` 只调整章节与补注释，保留原文措辞 |
| `--lang` | `auto`（默认）/ `c` / `rust` | 代码注释规则的语言分支，`auto` 按代码块自动判断 |
| `--no-diff` | 关闭变更清单 | 只输出正文 |
| `--no-sources` | 关闭关联推荐 | 只输出四段式正文 |

若用户未给参数，按默认值执行，并在输出头部注明所用参数。

## 2. 输入解析

1. 输入是路径（如 `knowlegdes/rust/advance_rust.md`）→ 读取全文；超过 800 行时按二级标题切分，逐节处理并在每节末尾合并。
2. 输入是粘贴内容 → 直接使用，若缺少标题则从首行提取主题名。
3. 输入为空 → 追问："请给出笔记路径或粘贴内容，并说明目标读者（自用速查 / 团队分享）。"
4. **保真原则**：不得删除原文的事实性内容；原文错误要保留"被纠正"的痕迹（见 §7 变更清单），原文缺失用 `> TODO: 待补充` 占位，**禁止编造** API、寄存器、路径或版本号。

## 3. 执行流程

| 步骤 | 动作 | 产物 |
| --- | --- | --- |
| S0 体检 | 通读原文，标记：主题、涉及子系统、代码语言、明显错误、术语不一致处 | 内部清单 |
| S1 归类 | 判断笔记属于：驱动 / 内核机制 / 用户态 / 工具链 / 语言特性，决定推荐的源码路径类别 | 主题标签 |
| S2 重组 | 按四段式重排段落；原文没有的内容用 `> TODO:` 占位，不硬凑 | 骨架 |
| S3 代码增强 | 为代码块补注释、补错误路径、补上下文说明；统一缩进与命名 | 代码块 |
| S4 术语校对 | 按 §6 表格逐条替换，中英混用处统一写法 | 术语一致 |
| S5 关联推荐 | 按 §7 映射表补充源码路径、man 手册、内核文档 | 「延伸阅读」 |
| S6 门禁 | 过 §9 的检查清单；不通过则回到 S2–S5 | 最终稿 |

## 4. 结构规范

### 4.1 文档头部

```markdown
# <主题名>

> **一句话摘要**：<这篇笔记解决什么问题，读者能获得什么>
>
> **适用环境**：<架构 / 内核版本 / 工具链 / 硬件平台，未知则写"通用">
> **前置知识**：<需要的背景，如"理解中断上下文""熟悉 C 指针">
> **原文来源**：<路径或"整理自某书/某文档">
```

### 4.2 四段式正文

必须按以下顺序、以下标题（一级标题用于篇，二级标题用于章，可按需降级）：

```markdown
## 一、背景与原理

### 1.1 要解决的问题
<为什么需要这个机制？不用它会怎样？一段话说清，不超过 5 行>

### 1.2 核心概念
<术语定义 + 关键数据结构/函数，可用表格>

### 1.3 工作机制
<分步骤讲清流程，配 Mermaid 图；涉及硬件时写清时序与寄存器/中断路径>
```

```markdown
## 二、代码实现

### 2.1 最小可运行示例
<完整可编译代码，含必要的头文件、错误处理、资源释放>

### 2.2 工程化版本（可选）
<加入并发保护、参数校验、日志、可配置项的版本，并说明与 2.1 的差异代价>
```

```markdown
## 三、关键点分析

<用表格或编号列表，逐条回答"为什么这样写"：
- 数据结构选择的理由（如为什么用 kfifo 而不是链表）
- 上下文与锁的对应关系（进程上下文可用 mutex，中断上下文只能 spinlock）
- 内存与生命周期（谁分配、谁释放、失败路径）
- 性能与实时性代价（复杂度、最坏延迟、Cache 影响）>
```

```markdown
## 四、避坑指南

> 每条格式：**现象** → **根因** → **修复** → **验证**

1. **现象**：...
   **根因**：...
   **修复**：...
   **验证**：...
```

```markdown
## 延伸阅读

- 内核源码：`drivers/i2c/i2c-core-base.c` —— I2C 核心的注册与匹配流程
- 手册页：`man 2 ioctl` —— 命令码的编码规则（`_IOR`/`_IOW`）
- 内核文档：`Documentation/driver-api/i2c.rst`
```

### 4.3 可选变更清单

```markdown
## 重构记录

| 类型 | 原文 | 修改后 | 理由 |
| --- | --- | --- | --- |
| 术语 | 中断处理 | 中断上下文 / 上半部与下半部 | 区分执行环境与代码实体 |
| 代码 | 无返回值检查 | 增加 `if (IS_ERR(dev))` 分支 | 失败路径缺失 |
| 结构 | 混在一起 | 拆为"原理/实现"两节 | 降低阅读跳跃 |
```

## 5. 代码增强规则

### 5.1 通用

- 每个代码块必须有语言标注（`c` / `rust` / `bash` / `dts` / `asm`）。
- 注释解释**为什么**和**边界条件**，不复述语句本身。
- 保留原文的错误处理风格；原文缺失错误处理时**补上**，并在变更清单登记。
- 不改变代码语义；需要改写算法时单独给出"改进版"，与原文版本并存对比。

### 5.2 C / 内核代码

| 对象 | 必须补的注释 | 示例 |
| --- | --- | --- |
| 寄存器读写 | 位域含义、复位值、副作用 | `writel(val, base + CTRL);` → `/* CTRL[3:0]=分频, CTRL[7]=使能, 写 1 清零中断标志 */` |
| 指针运算 | 指向对象、元素大小、边界 | `/* p 为 u32*，p+4 前进 16 字节而非 4 字节 */` |
| 内存分配 | GFP 上下文、是否可睡眠、释放点 | `buf = kmalloc(len, GFP_ATOMIC); /* 中断上下文，不可睡眠 */` |
| 用户空间拷贝 | 方向与失败处理 | `if (copy_from_user(dst, src, n)) return -EFAULT;` |
| 锁 | 保护的字段、获取顺序、上下文限制 | `/* 保护 tx_ring，进程上下文持锁，禁止在 ISR 中调用 */` |
| 错误路径 | `goto` 标签与清理顺序 | `err_free_irq: free_irq(irq, dev);` |
| 硬件访问 | 是否需要屏障 / `readl_relaxed` | `/* 写完描述符后需 wmb() 再敲门铃 */` |

### 5.3 Rust / 嵌入式 Rust

| 对象 | 必须补的注释 | 示例 |
| --- | --- | --- |
| `unsafe` 块 | `// SAFETY:` 说明前置条件 | `// SAFETY: base 由 ioremap 返回且在本驱动生命周期内有效` |
| FFI 结构体 | `#[repr(C)]` 与字段对齐 | `#[repr(C)] struct regs { ctrl: u32, stat: u32 }` |
| 所有权转移 | 为什么 move / clone / borrow | `/* 所有权转移给中断处理闭包，主线程不再访问 */` |
| 并发容器 | `Arc<Mutex<T>>` 的锁粒度 | `/* 锁只保护计数器，不覆盖 DMA 提交路径 */` |
| `no_std` | panic 策略、分配器来源 | `/* 使用 bump allocator，仅在初始化阶段分配 */` |

### 5.4 反例（禁止）

- 给每一行加 `// 赋值`、`// 返回` 这类无信息注释。
- 把内核代码改成用户态风格（如用 `printf` 代替 `printk`、用 `malloc` 代替 `kmalloc`）。
- 在注释里写未经核实的寄存器偏移；不确定就写 `/* TODO: 查 TRM 确认偏移 */`。

## 6. 术语校对表

按"原文表述 → 建议表述"替换，全篇统一；表格未覆盖但明显不规范的，按同一原则处理并登记。

| 原文常见写法 | 建议写法 | 说明 |
| --- | --- | --- |
| 中断处理 | 中断上下文 / 中断服务例程（ISR） | "中断处理"是动作，代码实体用 ISR，运行环境用"中断上下文"；涉及上下半部要写清 |
| 中断里可以睡眠 | 中断上下文禁止睡眠 | 原子上下文调用可能睡眠的函数会触发 `BUG: scheduling while atomic` |
| 用户态函数 | 用户空间 API / glibc 封装 | 与"内核空间"对称；"态"多指 CPU 特权级 |
| 内核态 | 内核空间（kernel space） | 谈地址空间用"空间"，谈特权级用"EL0/EL1"或"Ring 3/Ring 0" |
| 打印日志 / printf | `printk` / `pr_info` / `dev_info` | 内核无 stdio；优先用带设备上下文的 `dev_*` |
| malloc / free | `kmalloc`/`kfree`、`kzalloc`、`vmalloc`/`vfree` | 说明选择依据（连续物理内存 / 大块非连续） |
| 开个线程收数据 | 内核线程 `kthread` 或工作队列 `workqueue` | 中断上下文不能创建线程；软中断/任务let 也需说明 |
| 加个锁 | 用 `spinlock`/`mutex` 保护某字段（注明上下文） | 必须说明保护对象与可睡眠性 |
| 睡眠等待 | 阻塞等待（`wait_event_*` / `msleep`） | 与"忙等"`cpu_relax()` 区分 |
| 死锁 | 死锁 / 活锁 / 饥饿 | 三者机理不同，不可混用 |
| 内存泄漏 | 内存泄漏 / 内存碎片 / 野指针 | 分别对应未释放、分配器退化、悬垂引用 |
| 竞态 | 竞态条件（race condition）/ 数据竞争（data race） | 前者是时序导致的错误结果，后者是未同步的并发访问 |
| 系统调用函数 | 系统调用（syscall）/ 库函数 | `printf` 是库函数，最终可能调用 `write` |
| 中断号 | hwirq / virq（Linux IRQ 号）/ 中断描述符 | 三者层次不同，映射由 `irq_domain` 完成 |
| GPIO 编号 | GPIO 描述符（`gpiod_*`） | 全局 GPIO 编号已废弃，新代码用 consumer API |
| 寄存器地址 | 物理地址 / 虚拟地址（`ioremap`）/ `__iomem` 指针 | 内核不能直接解引用物理地址 |
| volatile 保证同步 | 用 `READ_ONCE`/`WRITE_ONCE`/屏障 | `volatile` 不提供原子性与内存序 |
| 设备节点 | `/dev` 下的设备节点（cdev）/ sysfs 属性 | 两者用途不同，不能互称 |
| I2C 设备 | I2C 从设备（client）/ 适配器（adapter）/ 控制器 | 总线控制器 ≠ 从设备 |
| 分包/粘包 | 消息边界（framing） | 串流协议需自定义分帧 |
| 句柄 | 文件描述符（fd）/ 内核对象指针 | 用户态用 fd，内核态用结构体指针 |

Rust 侧补充：`所有权`、`借用`、`生命周期`、`trait 对象`、`Send`/`Sync`、`Pin`、`no_std` 均保留英文原词首次出现时括注中文，之后统一用英文。

## 7. 关联推荐规则

### 7.1 按主题映射源码路径

| 笔记主题关键词 | 推荐源码 | 推荐文档 |
| --- | --- | --- |
| 字符设备 / cdev / file_operations | `fs/char_dev.c`、`include/linux/cdev.h`、`drivers/char/` | `Documentation/driver-api/driver-model/` |
| platform 驱动 / probe / 设备树匹配 | `drivers/base/platform.c`、`drivers/of/`、`include/linux/platform_device.h` | `Documentation/devicetree/bindings/` |
| 设备树 / DTS / compatible | `drivers/of/base.c`、`scripts/dtc/` | `Documentation/devicetree/usage-model.rst` |
| I2C | `drivers/i2c/i2c-core-base.c`、`drivers/i2c/busses/`、`include/linux/i2c.h` | `Documentation/i2c/` |
| SPI | `drivers/spi/spi.c`、`include/linux/spi/spi.h` | `Documentation/spi/` |
| GPIO / pinctrl | `drivers/gpio/gpiolib.c`、`drivers/pinctrl/`、`include/linux/gpio/consumer.h` | `Documentation/driver-api/gpio/` |
| 中断 / IRQ / 中断上下文 | `kernel/irq/manage.c`、`kernel/irq/chip.c`、`include/linux/interrupt.h` | `Documentation/core-api/genericirq.rst` |
| 工作队列 / 下半部 | `kernel/workqueue.c`、`kernel/softirq.c` | `Documentation/core-api/workqueue.rst` |
| 定时器 / 延时 | `kernel/time/timer.c`、`include/linux/jiffies.h` | `Documentation/timers/` |
| 内存分配 / slab / vmalloc | `mm/page_alloc.c`、`mm/slub.c`、`mm/vmalloc.c` | `Documentation/core-api/memory-allocation.rst` |
| 用户态内存 / mmap | `mm/mmap.c`、`mm/memory.c` | `Documentation/filesystems/`、`man 2 mmap` |
| 锁 / 自旋锁 / mutex / RCU | `kernel/locking/mutex.c`、`kernel/locking/spinlock.c`、`kernel/rcu/` | `Documentation/locking/` |
| 进程 / 线程 / 调度 | `kernel/fork.c`、`kernel/sched/core.c`、`kernel/exit.c` | `man 2 clone`、`man 7 sched` |
| VFS / 文件系统 | `fs/open.c`、`fs/read_write.c`、`include/linux/fs.h` | `Documentation/filesystems/vfs.rst` |
| 网络 / Socket / netdev | `net/core/`、`net/ipv4/`、`drivers/net/` | `Documentation/networking/`、`man 7 tcp` |
| USB | `drivers/usb/core/`、`drivers/usb/gadget/` | `Documentation/driver-api/usb/` |
| MMC / SD | `drivers/mmc/core/`、`include/linux/mmc/` | `Documentation/driver-api/mmc/` |
| DMA / 一致性 | `kernel/dma/`、`include/linux/dma-mapping.h` | `Documentation/core-api/dma-api.rst` |
| 时钟 / 电源 / regulator | `drivers/clk/`、`drivers/regulator/`、`drivers/base/power/` | `Documentation/driver-api/pm/` |
| Rust for Linux | `rust/kernel/`、`rust/kernel/device.rs`、`rust/kernel/driver.rs`、`rust/kernel/sync/` | `Documentation/rust/` |
| 启动流程 | `init/main.c`、`arch/arm64/kernel/head.S` | `Documentation/arch/arm64/` |
| 调试 / 跟踪 | `kernel/trace/`、`lib/dynamic_debug.c` | `Documentation/trace/` |

### 7.2 按系统调用映射手册页

| 系统调用 / 接口 | 手册页 |
| --- | --- |
| `open`/`close`/`read`/`write`/`lseek` | `man 2 open`、`man 2 read`、`man 2 write` |
| `ioctl` | `man 2 ioctl`（含 `_IO`/`_IOR`/`_IOW` 编码） |
| `mmap`/`mprotect`/`msync` | `man 2 mmap`、`man 2 mprotect` |
| `poll`/`select`/`epoll` | `man 2 poll`、`man 7 epoll` |
| `clone`/`fork`/`execve`/`wait` | `man 2 clone`、`man 2 execve`、`man 2 wait` |
| `pthread_*` | `man 3 pthread_create`、`man 7 pthreads` |
| `socket`/`bind`/`listen`/`accept` | `man 2 socket`、`man 7 tcp` |
| `signalfd`/`signalfd`/信号 | `man 7 signal`、`man 2 sigaction` |
| `timerfd`/`clock_gettime` | `man 2 timerfd_create`、`man 3 clock_gettime` |
| `futex` | `man 2 futex` |
| netlink | `man 7 netlink` |
| 能力 / 权限 | `man 7 capabilities` |
| `/proc`、`sysfs`、udev | `man 5 proc`、`man 5 sysfs`、`man 7 udev` |
| 终端 / tty | `man 4 tty` |

### 7.3 推荐原则

- 每篇 3–6 条，按"源码 → 手册 → 文档"排序；每条附一句"为什么值得看"。
- 路径必须真实存在于对应内核版本；跨版本差异要注明（如"6.1 起 `…` 改名"）。
- 不确定路径是否存在时，写"参考 `drivers/<子系统>/` 目录"，不编造具体文件名。

## 8. 输出格式

1. **一行重构摘要**：`已将 <原文> 重构为四段式，补充 N 处注释、修正 M 处术语、新增 K 条延伸阅读。`
2. **正文**：按 §4 模板输出，代码块带语言标注。
3. **重构记录**（默认开启）：§4.3 表格，`--no-diff` 时省略。
4. 输出为完整 Markdown，可直接覆盖原文件；若用户要求"只给片段"，则只输出改动部分并标注插入位置。

## 9. 质量门禁

全部通过才算完成：

- [ ] 四个一级段落齐全且顺序正确，无"其他/杂项"兜底段
- [ ] 每个代码块有语言标注，能独立编译或明确标注"片段"
- [ ] 寄存器/指针/内存分配处均有针对性注释（至少命中原文出现的位置）
- [ ] 所有"中断上下文/内核空间/用户空间/竞态条件"等术语全篇一致
- [ ] 每条避坑指南具备 现象→根因→修复→验证 四要素
- [ ] 延伸阅读链接路径真实、与主题相关
- [ ] 无编造的 API、地址、版本号；不确定处有 `TODO` 标注
- [ ] Mermaid 图语法正确（节点文案用 `[]` 包裹，避免特殊字符破坏解析）
- [ ] 原文事实性内容无遗漏；删改项均在重构记录中登记

## 10. 降级策略

| 情况 | 处理 |
| --- | --- |
| 原文过短（< 200 字） | 只做结构与术语整理，`背景与原理` 允许 3 行以内，不强行扩写 |
| 原文是纯命令记录（如 `dmesg` 输出） | 归入"现象记录"，转写为"背景 → 命令 → 输出解读 → 结论" |
| 原文含大段源码粘贴（> 300 行） | 只保留关键函数，其余折叠为"完整实现见 `<路径>`"并补注释 |
| 原文主题跨多个子系统 | 按子系统拆分小节，每节独立四段式，不混写 |
| 用户要求保留原格式 | 尊重用户，但至少补齐"避坑指南"与"延伸阅读" |

## 11. 示例（片段）

**重构前**

````markdown
## i2c读写
用i2c_transfer发消息，注意中断处理里不能睡眠。
````

**重构后**

````markdown
## 二、代码实现

```c
/* 一次完整事务：写寄存器地址 + 读回数据，必须放在同一个 transfer 中，
 * 否则中间可能被其他 I2C 主机/从设备打断（重复起始条件丢失）。 */
struct i2c_msg msgs[2] = {
	{ .addr = client->addr, .flags = 0,        .len = 1, .buf = &reg },
	{ .addr = client->addr, .flags = I2C_M_RD, .len = 1, .buf = &val },
};

/* 进程上下文可睡眠，此处用 i2c_transfer 而非 i2c_transfer_atomic */
ret = i2c_transfer(client->adapter, msgs, ARRAY_SIZE(msgs));
if (ret != ARRAY_SIZE(msgs))
	return ret < 0 ? ret : -EIO;   /* 部分成功也要当失败处理 */
```

## 四、避坑指南

1. **现象**：偶发 `-EIO`，重试即成功。
   **根因**：把"写寄存器地址"和"读数据"拆成两次 `i2c_transfer`，中间丢了重复起始条件。
   **修复**：合并为一次 `i2c_transfer`（如上）。
   **验证**：逻辑分析仪抓 SCL/SDA，确认读操作前有 `Sr` 而非 `P`。
````
