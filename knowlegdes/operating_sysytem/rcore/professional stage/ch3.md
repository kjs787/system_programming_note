


# 实现分时多任务系统

ch2 解决的问题是"一个应用怎么被跑起来"：内核把镜像拷到固定地址，伪造一份 `TrapContext`，
`__restore` + `sret` 掉进用户态；应用 `ecall` 回来，`__alltraps` 保存现场，处理完再原路返回。
整套机制有两个硬约束：

- **地址固定**：所有应用都被链接到 `0x80400000`，内存里同时只能有一份；
- **切换被动**：只有应用退出或出错，内核才有机会换下一个。

ch3 用三件事把这两个约束拆掉：

1. **多道程序放置**：编译期给第 i 个应用分配 `0x80400000 + i * 0x20000`，让它们互不重叠；
2. **多道程序加载**：内核初始化时一次性把全部应用搬进内存，`batch` 拆成 `loader` + `task`；
3. **任务切换 + 分时**：`__switch` 在内核态切换任务，时钟中断强制切换，形成时间片轮转。

一句话概括：**ch2 的 Trap 切换管"U 态 ↔ S 态"，ch3 新增的任务切换管"S 态内部的任务 A ↔ 任务 B"**。
两条通路叠加，才有"多个应用在并发"的假象。



## 多道程序放置与加载

### 为什么 ch2 的布局只能驻留一个应用

ch2 的用户程序在 `user/src/linker.ld` 里把起始地址写死成 `BASE_ADDRESS = 0x80400000`，
三个应用编译出来的 `_start` 全都落在这个地址上。批处理系统因此可以把三件事都简化成"固定地址"：

- 加载 = 把镜像拷到 `0x80400000`；
- 启动 = 从 `0x80400000` 开始执行；
- 换应用 = 覆盖同一段内存，再重新伪造 `TrapContext`。

这套约定成立的前提是"同一时刻内存里只有一个应用"。一旦想让两个应用同时驻留，
第一个冲突就是地址：两个应用都想让自己的 `.text` 落在 `0x80400000`，
而物理内存的一个字节只能放一份内容。

所以多道程序的第一步不是调度，而是**布局**：让第 i 个应用从 `0x80400000 + i * 0x20000` 开始。

> 补充：这里说的"地址"是物理地址。ch3 还没有页表，用户程序直接用物理地址运行，
> 因此"链接地址 = 加载地址 = 运行地址"，三者必须完全一致。等 ch4 引入地址空间，
> 每个应用才可以都以为自己独占 `0x0` 开始的内存。

### 用 -Ttext 给每个应用定制链接地址

既然每个应用的起始地址不同，就不能再让链接脚本写死地址，而要在**编译每一个应用时**把地址作为参数传进去。
`user/build.py` 的做法是：对每个应用单独跑一次 `cargo rustc`，用 `-Clink-args=-Ttext=...` 指定代码段地址。

```python
# user/build.py（示意，重点在最后一行参数）
APP_BASE = 0x80400000
APP_SIZE_LIMIT = 0x20000

for app_id, app in enumerate(apps):
    os.system(
        "cargo rustc --bin %s --release -- "
        "-Clink-args=-Ttext=%d" % (app, APP_BASE + app_id * APP_SIZE_LIMIT)
    )
```

这三层参数各管一段，缺一层地址就传不下去：

| 参数 | 作用 |
| --- | --- |
| `cargo rustc` | 和 `cargo build` 一样编译，但允许把 `--` 之后的参数**原样传给 rustc**（`cargo build` 不接受这些参数） |
| `-C link-args=...` | 把参数继续透传给链接器 `ld` |
| `-Ttext=0x80420000` | 让 `ld` 把 `.text` 段放到这个地址（等价于链接脚本里写 `. = 0x80420000; .text : {...}`） |

于是三个应用被链接到三个互不重叠的区间：

| 应用 | 链接/加载地址 | 可用区间（`APP_SIZE_LIMIT = 0x20000`，即 128 KiB） |
| --- | --- | --- |
| app0 | `0x80400000` | `0x80400000` ~ `0x80420000` |
| app1 | `0x80420000` | `0x80420000` ~ `0x80440000` |
| app2 | `0x80440000` | `0x80440000` ~ `0x80460000` |

两个前提，缺一个就前功尽弃：

1. 用户程序的 `linker.ld` 里**不能**再残留 `. = 0x80400000`。链接脚本里的显式地址优先级高于 `-Ttext`，
   留着它，`-Ttext` 就是白传，所有应用又会挤回同一个地址。
2. 每个应用的实际大小必须小于 `APP_SIZE_LIMIT`。超了就会压到下一个应用的头上，
   这种错误编译期完全不报，运行时表现为"莫名其妙的非法指令 / 访存错误"。

> 注解（qemu 内存上限）：物理内存是从 `0x80000000` 开始的一段（rCore 的 Makefile 一般给 `-m 128M`），
> `0x80400000 + num_app * 0x20000` 必须落在这段内存之内。应用太多或 `APP_SIZE_LIMIT` 太大，
> 拷贝时就会写到不存在的物理地址上 —— 表现就是教程里说的 core dumped。

### loader：启动时一次性加载全部应用

ch2 的 `batch` 子模块同时干了三件事：加载应用、保存应用信息、切换应用。
ch3 把它拆成两个模块，职责单一：

| 模块 | 职责 | 关键接口 |
| --- | --- | --- |
| `loader` | 启动时把**所有**应用搬进内存，并回答"第 i 个应用在哪、总共有几个" | `load_apps()`、`get_num_app()`、`get_base_i(i)`、`init_app_cx(i)` |
| `task` | 任务的状态、上下文、调度 | `TaskManager`、`__switch`、`run_first_task()` |

`load_apps` 在内核初始化时调用一次，之后所有应用都待在内存里：

```rust
// os/src/loader.rs
pub fn load_apps() {
    extern "C" { fn _num_app(); }
    let num_app_ptr = _num_app as usize as *const usize;
    let num_app = get_num_app();
    let app_start = unsafe { core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1) };
    // clear i-cache first
    unsafe {
        core::arch::asm!("fence.i");
    }
    // load apps
    for i in 0..num_app {
        let base_i = get_base_i(i);
        // clear region
        (base_i..base_i + APP_SIZE_LIMIT)
            .for_each(|addr| unsafe { (addr as *mut u8).write_volatile(0) });
        // load app from data section to memory
        let src = unsafe {
            core::slice::from_raw_parts(app_start[i] as *const u8, app_start[i + 1] - app_start[i])
        };
        let dst = unsafe { core::slice::from_raw_parts_mut(base_i as *mut u8, src.len()) };
        dst.copy_from_slice(src);
    }
}
```

逐段说明：

1. `_num_app` 是链接器符号，由 `build.rs` 生成的 `link_app.S` 提供。它指向的数组布局是：

   ```text
   [应用个数 num_app][app_0 起点][app_1 起点] ... [app_{n-1} 起点][app_{n-1} 终点]
   ```

   注意**不是**"起点/终点成对"，而是"n 个起点 + 1 个终点"：第 i 段的长度由
   `app_start[i+1] - app_start[i]` 得到，所以 `app_start` 的长度要取 `num_app + 1`。

2. `fence.i` 用来同步取指端：刚写进内存的指令，必须让 CPU 的 I-cache / 取指流水线看到新内容。
   书里注释写的是 "clear i-cache first"，位置在拷贝循环之前；从语义上说"写入代码之后、执行之前"
   必须有一次 `fence.i`，所以更稳妥的写法是放在循环结束之后（或每次拷贝后执行一次）。

3. 先把 `[base_i, base_i + APP_SIZE_LIMIT)` 整段清零，再拷贝有效内容。清零有两个作用：
   抹掉上一轮加载的残留，以及把应用没有覆盖到的尾部填成 0，避免跳到应用之外时执行到垃圾指令。

4. 清零用 `write_volatile` 而不是普通的 `*mut u8 = 0`：普通写属于"死存储"，编译器有权优化掉；
   `write_volatile` 明确告诉编译器"这次写有副作用，必须真的发生"。

`base_i` 的计算必须与 `build.py` 完全一致，否则应用会跑在它没被链接到的地址上：

```rust
fn get_base_i(app_id: usize) -> usize {
    APP_BASE_ADDRESS + app_id * APP_SIZE_LIMIT   // 0x80400000 + i * 0x20000
}
```

> 补充：`_num_app` 数组里的"起点"是应用镜像在**内核 `.data` 段里的地址**（`link_app.S` 用 `.incbin`
> 把二进制塞进去），不是应用将来要运行的地址。真正的运行地址由 `get_base_i(i)` 算出来。
> 这两件事很容易混：一个是"镜像从哪读"，一个是"镜像写到哪"。

### 与 ch2 的对照

| 维度 | ch2（批处理） | ch3（多道程序） |
| --- | --- | --- |
| 应用地址 | 全部 `0x80400000` | `0x80400000 + i * 0x20000` |
| 何时加载 | 每次切换应用时只加载下一个 | 内核初始化时一次性全载 |
| 内存占用 | 同时只有 1 个 | 同时 n 个 |
| 模块划分 | `batch`（加载 + 切换 + 管理） | `loader` + `task`（拆开） |
| 切换依据 | 应用退出 / 出错 | 应用 `yield` 或时钟中断 |

### 避坑清单

1. `build.py` 的地址公式与 `loader::get_base_i` 不一致 → 应用被加载到与链接地址不同的地方，
   第一条 `jal` / `auipc` 就飞。
2. `APP_SIZE_LIMIT` 小于某个应用的实际大小 → 覆盖下一个应用，症状是"跑的明明是 app0，行为却像 app1"。
3. 用户 `linker.ld` 里残留写死的 `BASE_ADDRESS` → `-Ttext` 失效，所有应用又回到同一个地址。
4. 漏掉 `fence.i` → qemu 上偶尔能跑，换个版本就取到旧指令。
5. 内核镜像超过 `0x200000`（`0x80200000` ~ `0x80400000`）→ 内核尾部与 app0 重叠，
   链接完应该用 `size` / `readelf` 检查一下。
6. `_num_app` 数组长度取成 `num_app` 而不是 `num_app + 1` → 最后一段长度算错，拷贝越界。
7. 清零区间写成 `src.len()` 而不是 `APP_SIZE_LIMIT` → 应用尾部残留旧数据。


## 任务切换

### 两种切换的分工

ch2 已经有了一条完整的 "U 态 ↔ S 态" 通路（`__alltraps` / `__restore`）。
ch3 需要的是另一条：**两个应用各自的 S 态控制流之间的切换**。

| | Trap 切换（ch2） | 任务切换（ch3） |
| --- | --- | --- |
| 触发者 | `ecall`、缺页、非法指令、中断 | 内核主动调用 `__switch` |
| 方向 | U → S，或 S → U | S → S |
| 是否涉及特权级 | 是 | **否** |
| 保存内容 | 用户态现场：x0~x31 + `sstatus` + `sepc`（`TrapContext`，34*8 B） | 内核态续点：`ra` + `sp` + s0~s11（`TaskContext`，14*8 B） |
| 保存位置 | 该任务的内核栈顶 | 该任务的 `TaskControlBlock.task_cx` |
| 谁负责 | 硬件 + `__alltraps` 汇编 | `__switch` 汇编 + 编译器按调用约定配合 |
| 对应用是否透明 | 是 | 是 |

一句话：**Trap 切换保存"用户程序被硬件打断时的那一刻"，任务切换保存"内核执行到一半时的那一刻"。**
两者叠加，一个任务才有完整的两层现场：用户态现场压在内核栈上，内核态续点存在 TCB 里。

### TaskContext：为什么只保存 ra / sp / s0~s11

```rust
// os/src/task/context.rs
#[repr(C)]
pub struct TaskContext {
    ra: usize,
    sp: usize,
    s: [usize; 12],
}
```

RISC-V 调用约定把通用寄存器分成两类：

| 类别 | 寄存器 | 谁负责保存 |
| --- | --- | --- |
| caller-saved（调用者保存） | `ra`、`t0~t6`、`a0~a7` | 调用者：调用之后还要用的值，自己压栈 |
| callee-saved（被调用者保存） | `s0~s11`、`sp` | 被调用者：它要用，就得先保存、返回前恢复 |

`__switch` 在 Rust 里被声明成一个普通的外部函数：

```rust
// os/src/task/switch.rs
core::arch::global_asm!(include_str!("switch.S"));

extern "C" {
    pub fn __switch(
        current_task_cx_ptr: *mut TaskContext,
        next_task_cx_ptr: *const TaskContext);
}
```

这意味着调用点前后，**编译器已经替我们保存了它认为还要用的 caller-saved 寄存器**
（`__switch` 可能改掉它们，编译器知道这一点）。所以 `__switch` 只需要负责 callee-saved 那一半，
再加上两个特殊角色：

- `ra`：`__switch` 返回后跳回哪里，也就是"续点"；
- `sp`：这个任务的内核栈在哪 —— **换任务，本质就是换内核栈**。

`TaskContext` 的内存布局必须与汇编里的偏移严格对应：

| 字段 | 偏移 | 大小 |
| --- | --- | --- |
| `ra` | 0 | 8 |
| `sp` | 8 | 8 |
| `s[0]` | 16 | 8 |
| … | … | … |
| `s[11]` | 104 | 8 |
| 合计 | — | 112 B |

`#[repr(C)]` 不能省：没有它，Rust 可以任意重排字段，汇编里的 `0(a0)`、`8(a0)` 就全错了。

### __switch 逐行

```asm
# os/src/task/switch.S
.altmacro
.macro SAVE_SN n
    sd s\n, (\n+2)*8(a0)
.endm
.macro LOAD_SN n
    ld s\n, (\n+2)*8(a1)
.endm

    .section .text
    .globl __switch
__switch:
    # __switch(
    #     current_task_cx_ptr: *mut TaskContext,   // a0
    #     next_task_cx_ptr: *const TaskContext     // a1
    # )
    # save kernel stack of current task
    sd sp, 8(a0)
    # save ra & s0~s11 of current execution
    sd ra, 0(a0)
    .set n, 0
    .rept 12
        SAVE_SN %n
        .set n, n + 1
    .endr
    # restore ra & s0~s11 of next execution
    ld ra, 0(a1)
    .set n, 0
    .rept 12
        LOAD_SN %n
        .set n, n + 1
    .endr
    # restore kernel stack of next task
    ld sp, 8(a1)
    ret
```

几个细节：

1. `a0` / `a1` 就是两个参数，这是 `extern "C"` 调用约定规定的，不需要额外搬运。
   `.altmacro` 是为了让 `.rept` 里的 `%n` 能展开成数字。
2. `(n+2)*8` 是因为 `s0` 在偏移 16，而 `n` 从 0 开始：`sn` 的偏移 = `(n + 2) * 8`。
3. 保存时先 `sp` 后 `ra`，顺序无所谓，两者互不依赖。
4. 恢复时 `ld sp, 8(a1)` **必须放在最后**：`sp` 一换，这个函数就"站在别人的栈上"了，
   后续任何压栈 / 出栈都会落到新任务的内核栈里。所以换 `sp` 是"移交控制权"的最后一步。
5. `ret` 等价于 `jalr x0, 0(ra)`。此时 `ra`、`sp`、`s0~s11` 都已经是新任务的值，
   于是 CPU 从新任务的续点继续执行 —— 而旧任务的这些值，早已躺进它的 `TaskContext` 里了。

### 一次切换的完整时序

```text
任务 A 的内核栈                              任务 B 的内核栈
┌──────────────────┐  ← sp_A                ┌──────────────────┐  ← sp_B
│ TrapContext_A    │                        │ TrapContext_B    │
├──────────────────┤                        ├──────────────────┤
│ __alltraps 栈帧   │                        │ __alltraps 栈帧   │
├──────────────────┤                        ├──────────────────┤
│ trap_handler 帧   │                        │ trap_handler 帧   │
├──────────────────┤                        ├──────────────────┤
│ run_next_task 帧  │                        │ run_next_task 帧  │
├──────────────────┤                        ├──────────────────┤
│ __switch 栈帧     │                        │ __switch 栈帧     │
└──────────────────┘                        └──────────────────┘

A 里调用 __switch(&mut A.task_cx, &B.task_cx)
  ├─ 把 sp_A / ra_A / s*_A 写进 A.task_cx
  ├─ 从 B.task_cx 读回 sp_B / ra_B / s*_B
  └─ ret → 跳到 ra_B
       └─ ra_B = B 上次调用 __switch 的下一条指令（首次则是 __restore）
```

关键点：`__switch` 不是中断，也不是 trap，它只是**一次普通的函数调用**。
所以 A 被换走时，从 `__alltraps` 到 `__switch` 的所有栈帧都还留在 A 的内核栈上；
B 恢复 `sp` 之后站的是 B 自己的栈，两边的"内核执行历史"互不覆盖。
这就是"**每个任务必须有独立内核栈**"的根本原因 —— 不是性能问题，是正确性问题。

### 第一次被切进来：ra 为什么是 __restore

新任务的 `TaskContext` 不是"保存"出来的，而是**伪造**出来的：

```rust
impl TaskContext {
    pub fn goto_restore(kstack_ptr: usize) -> Self {
        extern "C" { fn __restore(); }
        Self {
            ra: __restore as usize,
            sp: kstack_ptr,
            s: [0; 12],
        }
    }
}
```

`sp` 指向内核栈上已经压好的 `TrapContext`，`ra` 指向 `__restore`。
于是 `__switch` 对这个任务第一次 `ret`，效果等于"跳进 `__restore`，并让它以为刚被调用"。
此时它的内核栈上没有 `__alltraps` 的帧，只有 `init_app_cx` 造好的那份 `TrapContext` ——
正好是 `__restore` 需要的全部东西。

这正是 ch2 "把启动应用伪造成一次 Trap 返回" 的技巧：**同一个 `__restore`，两种进入方式**。

### 避坑清单

1. `TaskContext` 字段顺序 / 类型与汇编偏移不一致（尤其是漏了 `#[repr(C)]`）→ 切过去就飞，
   而且通常表现为"莫名其妙的三级异常"，极难定位。
2. 少存一个 `s` 寄存器 → 大多数时候没问题，换个优化级别或编译器版本就出错
   （编译器恰好把某个跨调用的值放在那个 `s` 寄存器里）。
3. 漏了 `.globl __switch` 或没写 `global_asm!(include_str!("switch.S"))` → 链接期找不到符号
   （这个反而好查）。
4. 只保存 `ra` 不保存 `sp`（或反过来）→ 换栈没发生，两个任务共用一个内核栈，必然互相覆盖。
5. 以为任务切换会"顺便保存用户态现场" → 不会。用户态现场只在 `__alltraps` 进内核时保存一次，
   任务切换只搬运"内核态续点"。


## 管理多道程序

### 任务状态与任务控制块

内核要管任务，先得回答三个问题：任务现在是什么状态、它下次从哪里继续、它上次停在哪。
前两个答案落在 `TaskControlBlock` 里，第三个答案落在它的内核栈上。

```rust
// os/src/task/task.rs
#[derive(Copy, Clone, PartialEq)]
pub enum TaskStatus {
    UnInit,   // 未初始化
    Ready,    // 准备运行
    Running,  // 正在运行
    Exited,   // 已退出
}

#[derive(Copy, Clone)]
pub struct TaskControlBlock {
    pub task_status: TaskStatus,
    pub task_cx: TaskContext,
}
```

状态迁移只有四条边：

```text
        TASK_MANAGER 初始化
UnInit ────────────────────► Ready ◄──────────────┐
                               │                  │
              run_first_task / │                  │ sys_yield
              run_next_task    │                  │ 时钟中断
                               ▼                  │
                            Running ──────────────┘
                               │
                               │ sys_exit
                               ▼
                            Exited
```

`Exited` 是一个"终点"：本章没有资源回收机制，退出的任务只是把状态改成 `Exited`，
它的 TCB 仍然留在数组里，它用过的内核栈也不再使用（真的回收要等后面的章节）。

> 补充：`TaskControlBlock` 派生 `Copy` 不是为了"随便复制"，而是为了让
> `[TaskControlBlock { .. }; MAX_APP_NUM]` 这种"用同一个值填满数组"的写法能编译通过。
> 数组初始化要求元素可以按位复制，这是 Rust 的语法约束，不是设计意图。

### TaskManager：常量与变量分离

```rust
// os/src/task/mod.rs
pub struct TaskManager {
    num_app: usize,                     // 常量：初始化后不再变化
    inner: UPSafeCell<TaskManagerInner>,// 变量：执行中会变，需要互斥访问
}

struct TaskManagerInner {
    tasks: [TaskControlBlock; MAX_APP_NUM],
    current_task: usize,
}
```

这种"变量与常量分离"的写法有两个好处：

- `num_app` 只读，读它不需要拿锁、也不需要借用 `inner`，`find_next_task` 里算扫描区间时很方便；
- 可变部分集中在一个 `UPSafeCell` 里，`exclusive_access()` 的持有时间可以压到最短，
  避免"拿着借用去 `__switch`"这种致命操作（后面会看到 `drop(inner)` 就是为此）。

`tasks` 用定长数组而不是 `Vec`：本章内核还没有堆分配器，`MAX_APP_NUM` 在编译期就定死，
数组直接放进 `.bss`。代价是应用数量有上限，好处是完全静态、没有分配失败的可能。

### TASK_MANAGER 的初始化

```rust
// os/src/task/mod.rs
lazy_static! {
    pub static ref TASK_MANAGER: TaskManager = {
        let num_app = get_num_app();
        let mut tasks = [TaskControlBlock {
            task_cx: TaskContext::zero_init(),
            task_status: TaskStatus::UnInit,
        }; MAX_APP_NUM];
        for (i, t) in tasks.iter_mut().enumerate().take(num_app) {
            t.task_cx = TaskContext::goto_restore(init_app_cx(i));
            t.task_status = TaskStatus::Ready;
        }
        TaskManager {
            num_app,
            inner: unsafe {
                UPSafeCell::new(TaskManagerInner {
                    tasks,
                    current_task: 0,
                })
            },
        }
    };
}
```

逐行说明：

1. 第 5 行 `get_num_app()` 从 `_num_app` 数组的第 0 项读出应用总数 —— 与 `load_apps` 用的是同一个来源，
   所以"加载了几个"和"管理几个"永远一致。
2. 第 6~9 行先用 `UnInit` + `zero_init` 填满整个 `MAX_APP_NUM` 长度的数组。
   `num_app` 之外的那些槽位就停在 `UnInit`，`find_next_task` 只认 `Ready`，所以永远不会被选中。
3. 第 10~13 行只对前 `num_app` 个 TCB 做真正的初始化：
   `init_app_cx(i)` 在**第 i 个任务的内核栈**上压入一份 `TrapContext`（`sepc` = 该应用的入口地址，
   `sp` = 用户栈顶，`sstatus.SPP = User`），返回压入之后的栈顶；
   `goto_restore` 把这个栈顶存成 `sp`、把 `ra` 设成 `__restore`。这一步做完，
   每个任务都有了一份"随时可以被切进去"的伪造上下文。

```rust
// os/src/loader.rs
pub fn init_app_cx(app_id: usize) -> usize {
    KERNEL_STACK[app_id].push_context(
        TrapContext::app_init_context(get_base_i(app_id), USER_STACK.get_sp())
    ) as *const _ as usize
}
```

注意这里两处不对称，是本章的关键设计：

| 资源 | 每任务一份？ | 原因 |
| --- | --- | --- |
| 内核栈 `KERNEL_STACK[i]` | **是** | 任务切换发生在内核态，被切走的内核栈必须原样保留（`TrapContext` 就压在上面） |
| 用户栈 `USER_STACK` | 否，全局一份 | 同一时刻只有一个任务在 U 态运行，栈顶地址可以复用 |

> 补充：用户栈共用是本章的简化。严格说，如果任务 A 在时间片中途被切走、它的用户栈上还留着活跃栈帧，
> 任务 B 复用同一段内存就会把这些帧覆盖掉。本章的示例程序栈帧很浅、被切走时不依赖栈上局部状态，
> 所以看不出问题；真正给每个应用独立用户栈（以及独立地址空间）是 ch4 的事。

### 系统调用：sys_yield 与 sys_exit

```rust
// os/src/syscall/process.rs
pub fn sys_yield() -> isize {
    suspend_current_and_run_next();
    0
}

pub fn sys_exit(exit_code: i32) -> ! {
    println!("[kernel] Application exited with code {}", exit_code);
    exit_current_and_run_next();
    panic!("Unreachable in sys_exit!");
}
```

```rust
// os/src/task/mod.rs
pub fn suspend_current_and_run_next() {
    TASK_MANAGER.mark_current_suspended();   // Running → Ready
    TASK_MANAGER.run_next_task();
}

pub fn exit_current_and_run_next() {
    TASK_MANAGER.mark_current_exited();      // Running → Exited
    TASK_MANAGER.run_next_task();
}
```

两者的差别只有一个：**当前任务之后还能不能被选中**。
`sys_yield` 是"让出但还会回来"，`sys_exit` 是"再也不回来"。

`sys_exit` 的返回类型是 `!`，但 `run_next_task` 在类型系统里是会返回的（它只是"切走"，
并非真的 `loop {}`），所以后面必须跟一个 `panic!` 兜底 —— 万一真的返回了，说明调度出了 bug。

用户库侧还有个小坑：

```rust
// user/src/lib.rs
// yield 是 Rust 的关键字，函数只能换个名字
pub fn yield_() -> isize { sys_yield() }
```

### 调度：run_next_task 与 find_next_task

```rust
impl TaskManager {
    fn run_next_task(&self) {
        if let Some(next) = self.find_next_task() {
            let mut inner = self.inner.exclusive_access();
            let current = inner.current_task;
            inner.tasks[next].task_status = TaskStatus::Running;
            inner.current_task = next;
            let current_task_cx_ptr = &mut inner.tasks[current].task_cx as *mut TaskContext;
            let next_task_cx_ptr = &inner.tasks[next].task_cx as *const TaskContext;
            drop(inner);
            // before this, we should drop local variables that must be dropped manually
            unsafe { __switch(current_task_cx_ptr, next_task_cx_ptr); }
            // go back to user mode
        } else {
            panic!("All applications completed!");
        }
    }

    fn find_next_task(&self) -> Option<usize> {
        let inner = self.inner.exclusive_access();
        let current = inner.current_task;
        (current + 1..current + self.num_app + 1)
            .map(|id| id % self.num_app)
            .find(|id| inner.tasks[*id].task_status == TaskStatus::Ready)
    }
}
```

`find_next_task` 的三个细节：

- 从 `current + 1` 开始，天然实现"轮到下一个"，不会出现某个任务永远排不上队；
- 区间是右开的 `current + 1 .. current + num_app + 1`，长度正好 `num_app`，
  取模之后**每个任务恰好被扫到一次**（包括 `current` 自己）；
- 所以"只有一个 Ready 任务，而且就是自己"时也能扫回自己：单任务调 `yield` 不会 panic。

`run_next_task` 的三个细节：

1. **先改状态再切**：`next` 置 `Running`、`current_task = next`。这两步必须在 `__switch` 之前完成 ——
   一旦切走，当前函数要等别人切回来才继续执行，那时再改就已经晚了（`current_task` 早已是别人的）。
2. **用裸指针绕过借用检查**：`&mut inner.tasks[current]` 和 `&inner.tasks[next]` 同时存在时，
   借用检查器会认为"同一个数组被同时可变和不可变借用"。这里先 `as *mut` / `as *const` 把借用"断开"，
   再在 `unsafe` 里用。安全性来自两点：两个指针指向不同元素；`__switch` 期间当前任务被挂起，没人再动 `tasks`。
3. **`drop(inner)` 必须显式写**：函数还没返回，`inner` 是局部变量，它的借用会一直持续到函数结束。
   而 `__switch` 之后，当前任务被挂起、CPU 可能在"另一个任务"里再次调用 `exclusive_access()` ——
   如果这边的借用没释放，那边就会 panic。`UPSafeCell` 内部用计数模拟 `RefCell` 的借用规则，
   这种"跨任务借用"是它最典型的翻车场景：**报错在下一个任务里，根因在当前任务忘记 drop**。

### 第一次进入用户态

内核启动后总得"进入"第一个任务，但此时没有"前一个任务"可以保存。`run_first_task` 的做法是给 `__switch`
塞一个一次性的、用完就扔的任务上下文：

```rust
fn run_first_task(&self) -> ! {
    let mut inner = self.inner.exclusive_access();
    let task0 = &mut inner.tasks[0];
    task0.task_status = TaskStatus::Running;
    let next_task_cx_ptr = &task0.task_cx as *const TaskContext;
    drop(inner);
    let mut _unused = TaskContext::zero_init();
    // before this, we should drop local variables that must be dropped manually
    unsafe {
        __switch(&mut _unused as *mut TaskContext, next_task_cx_ptr);
    }
    panic!("unreachable in run_first_task!");
}
```

`_unused` 的存在理由值得单独说：

- `__switch` 的实现是"无条件先写 `a0` 指向的 `TaskContext`"，所以第一个参数必须指向一块**合法可写**的内存，
  否则就会把 boot 栈上的其他数据写坏（比如覆盖 `inner` 或返回地址）。
- 不能把 `&mut task0.task_cx` 同时当两个参数：`__switch` 会先把"当前上下文"写进去，
  再读"下一个上下文" —— 同一块内存的话，读到的就是刚被覆盖的值。
- 因此造一个局部变量 `_unused` 最省事：它在 boot 栈上有真实空间，内容随后被丢弃，
  下划线前缀只是避免 unused 警告。

`__switch` 恢复 `sp` 之后，`sp` 指向 `init_app_cx` 压好的 `TrapContext`，`ra` 是 `__restore` ——
后面就完全是 ch2 的流程了。

### __restore 的变化：删掉 mv sp, a0

ch2 的 `__restore` 第一句是 `mv sp, a0`，ch3 必须删掉。原因在于两个版本的进入方式不同：

| 版本 | 谁调用 `__restore` | 调用时 `sp` 在哪 | 调用时 `a0` 是什么 |
| --- | --- | --- | --- |
| ch2 | `run_next_app` 里的普通函数调用 | boot 栈 | `push_context` 返回的 `TrapContext` 地址 |
| ch3（首次启动） | `__switch` 恢复 `sp` 后 `ret` 过去 | **已经**是内核栈上 `TrapContext` 的地址 | `__switch` 的第二个参数（指向 `TaskContext`） |
| ch3（Trap 返回） | `__alltraps` 里 `call trap_handler` 之后顺序执行到 | 内核栈，`trap_handler` 是被调用者，不会改 `sp` | `trap_handler` 返回的 `cx`（等于 `sp`） |

三种情况下，`sp` 都已经正确指向 `TrapContext`，所以不需要再从 `a0` 搬一次。
更关键的是：**如果保留 `mv sp, a0`，首次启动时会把 `sp` 设成 `TaskContext` 的地址**，
`__restore` 随后从那里读 `sstatus` / `sepc` / 通用寄存器，读到的全是垃圾，`sret` 直接飞。

这里也顺带解释了一个 ch2 遗留的疑问：`trap_handler` 为什么必须把 `cx` 原样返回？
因为 `__alltraps` 里 `call trap_handler` 之后紧跟着就是 `__restore` 的代码，
编译器按调用约定把返回值放在 `a0`；删掉 `mv sp, a0` 之后，`a0` 虽然不再被使用，
但"返回 `cx`"保证了 `__restore` 的语义（"从 `a0` 指向的上下文恢复"）在两种进入方式下都成立。

> 补充：为什么不给 `__restore` 做两个入口（一个给首次启动、一个给 trap 返回）？
> 因为"复用"本身就是设计目标：让"启动应用"和"从 trap 返回"走同一条代码路径，
> 少一条分支就少一类 bug。

### 状态迁移总表

| 事件 | 当前任务状态变化 | 是否发生任务切换 | 谁来选下一个 |
| --- | --- | --- | --- |
| 内核初始化（`TASK_MANAGER`） | `UnInit → Ready` | 否 | — |
| `run_first_task` | `Ready → Running`（task0） | 是（从 boot 上下文切到 task0） | 直接指定 0 |
| `sys_yield` | `Running → Ready` | 是 | `find_next_task` |
| `sys_exit` | `Running → Exited` | 是 | `find_next_task` |
| 时钟中断 | `Running → Ready` | 是 | `find_next_task` |
| 被 `__switch` 切回 | `Ready → Running` | —（切换的终点） | — |
| 全部任务都 `Exited` | — | 否 | `panic!("All applications completed!")` |

### 避坑清单

1. `find_next_task` 的区间写成 `current..current + num_app` → 会把自己多扫一次、把某个任务漏掉，
   极端情况下"当前任务是唯一 Ready"时选不到自己，`yield` 直接 panic。
2. 忘记把 `next` 置 `Running`、或忘记更新 `current_task` → 下一次切换基于错误的任务，
   典型症状是"两个任务来回切但第三个永远不跑"。
3. 在 `__switch` 前忘记 `drop(inner)` → 借用 panic，而且报错位置在**下一个任务**里，极具迷惑性。
4. `TASK_MANAGER` 初始化时数组长度写成 `num_app` 而不是 `MAX_APP_NUM` → 类型不符 / 越界，
   正确写法是"先填满 `MAX_APP_NUM`，再覆盖前 `num_app` 个"。
5. `sys_exit` 里没有 `panic!` 兜底 → 如果 `run_next_task` 真的返回（例如所有任务都已退出），
   会继续执行一个已经退出的任务。
6. 内核栈只建一份（`static KERNEL_STACK: KernelStack`）→ 任务一切换，两边的 `TrapContext` 互相覆盖，
   表现为"切回来之后应用从奇怪的地方继续执行"。
7. 在 `run_first_task` 里把 `_unused` 换成 `&mut task0.task_cx` → 见上文，`__switch` 会先写坏它再读它。


## 分时多任务系统

### 从协作式到抢占式

ch3 前半段的切换全靠应用自己调 `sys_yield`——应用不配合，内核就永远拿不回 CPU。
这叫**协作式调度**，它把"是否让出"的决定权交给了被调度的对象，显然不可靠。

现代调度器基本都是**抢占式**的：每个应用最多连续执行一个**时间片**（time slice，毫秒量级），
时间一到，内核强制把它换下去。本章用最简单的**时间片轮转**（RR, Round-Robin）：
所有任务排成一个环，每个跑一个时间片，依次轮转。

抢占需要一个"应用无法拒绝"的定时器，这就是 RISC-V 的 `mtime` / `mtimecmp`。

### 硬件计时器：mtime 与 mtimecmp

| 名字 | 位置 | 宽度 | 谁能访问 | 作用 |
| --- | --- | --- | --- | --- |
| `mtime` | CLINT 的内存映射寄存器 | 64 位 | M 态（S 态只读，通过 `time::read()`） | 按固定频率自增的计数器 |
| `mtimecmp` | CLINT 的内存映射寄存器 | 64 位 | **只有 M 态** | 比较值：`mtime >= mtimecmp` 时触发时钟中断 |

触发链路是这样的：

```text
mtime 追平 mtimecmp
      │
      ▼
CLINT 向 hart 发 M 态定时器中断
      │
      ▼
M 态的 SEE（RustSBI）接管，把它转成 S 态的 STIP（supervisor timer interrupt pending）
      │
      ▼
S 态在"允许中断"时进入 stvec → __alltraps → trap_handler，scause = Interrupt(SupervisorTimer)
```

S 态**不能直接写 `mtimecmp`**（那是 M 态的寄存器），只能通过 SBI 调用请求 M 态代劳：

```rust
// os/src/sbi.rs
const SBI_SET_TIMER: usize = 0;

pub fn set_timer(timer: usize) {
    sbi_call(SBI_SET_TIMER, timer, 0, 0);
}
```

`set_timer` 的参数是"**下一次中断的绝对时刻**"，不是增量。这个区别后面会写进避坑清单：
传增量的话，`mtimecmp` 会被设成一个早已过去的时刻，中断要么立刻连发、要么永远不再来。

### set_next_trigger：每 10 ms 一次

```rust
// os/src/timer.rs
use riscv::register::time;

pub fn get_time() -> usize {
    time::read()                       // 读 mtime
}

const TICKS_PER_SEC: usize = 100;      // 每秒 100 个时间片 → 每片 10 ms

pub fn set_next_trigger() {
    set_timer(get_time() + CLOCK_FREQ / TICKS_PER_SEC);
}
```

`CLOCK_FREQ` 是平台相关的时钟频率（每秒 `mtime` 增加多少），写在 `config` 子模块里。
若 `CLOCK_FREQ = 12_500_000`，则增量为 `125_000` tick，正好 10 ms。

三个要点：

1. 定时器是 **one-shot** 的：触发一次之后就失效，必须在每次中断处理里重新设置，
   否则只会抢占一次，之后系统退回协作式。
2. 用 `CLOCK_FREQ / TICKS_PER_SEC` 而不是写死 tick 数，是为了与平台解耦：
   换一块板子只改 `CLOCK_FREQ`。
3. 第一次触发要在进入用户态之前就设好，否则第一个应用会一直跑到自己退出。

### get_time_us 与 sys_get_time

```rust
// os/src/timer.rs
const MICRO_PER_SEC: usize = 1_000_000;

pub fn get_time_us() -> usize {
    time::read() / (CLOCK_FREQ / MICRO_PER_SEC)
}
```

整数除法把 tick 换算成微秒。精度取决于 `CLOCK_FREQ / 1_000_000`：
12.5 MHz 时除数是 12，精度约 0.96 μs；10 MHz 时除数是 10，精度 0.1 μs —— 都够用。

应用侧通过一个新的系统调用拿到时间：

```rust
/// 功能：获取当前的时间，保存在 TimeVal 结构体 ts 中，_tz 在我们的实现中忽略
/// 返回值：返回是否执行成功，成功则返回 0
/// syscall ID：169
fn sys_get_time(ts: *mut TimeVal, _tz: usize) -> isize;
```

```rust
// os/src/syscall/process.rs
#[repr(C)]
pub struct TimeVal {
    pub sec: usize,
    pub usec: usize,
}
```

实现就是算一次 `get_time_us`，再通过裸指针写回用户内存：

```rust
pub fn sys_get_time(ts: *mut TimeVal, _tz: usize) -> isize {
    let us = get_time_us();
    unsafe {
        *ts = TimeVal { sec: us / 1_000_000, usec: us % 1_000_000 };
    }
    0
}
```

> 补充：这里和 `sys_write` 一样，**完全没有校验用户指针**。`ts` 指向内核地址、非法地址、
> 或者一个只读页，内核都会照写不误 —— 这是本章"能跑就行"的实现，安全边界要等地址空间那一章。

### 三层中断开关：sie.STIE / sstatus.SIE / sstatus.SPIE

"时钟中断能不能进来"由三层开关串联决定，任何一层关着都进不来：

| 开关 | 位置 | 含义 | 谁来设置 |
| --- | --- | --- | --- |
| `sie.STIE` | `sie` CSR 的 bit5 | S 态定时器中断这一**类**是否使能（按中断源屏蔽） | 软件：`sie::set_stimer()` |
| `sstatus.SIE` | `sstatus` 的 bit1 | S 态的**全局**中断使能（总闸） | 硬件在 trap 入口清零、`sret` 恢复；软件也可手动设 |
| `sstatus.SPIE` | `sstatus` 的 bit5 | `SIE` 的备份，trap 入口保存、`sret` 恢复 | 硬件 |

硬件在 Trap 入口和出口做的这两件事，是理解"为什么本章不用手动开中断"的关键：

```text
进入 Trap:   SPIE ← SIE        // 先把总闸状态存起来
             SIE  ← 0          // 关总闸，处理过程中不响应 S 态中断
             SPP  ← 当前特权级

sret 返回:   SIE  ← SPIE       // 恢复总闸
             SPIE ← 1
             特权级 ← SPP
```

推论有两个：

1. 内核态执行期间默认**不响应** S 态中断，所以本章不需要手动 `sstatus::set_sie()`，
   也不会出现嵌套中断（嵌套意味着内核栈可能被第二次 trap 覆盖，处理起来复杂得多）。
2. 如果时钟中断在**内核态**到达，它不会被丢弃，而是 pending 在那里；等 `sret` 把 `SIE` 恢复成 1、
   特权级回到 U 之后，它立刻被响应。**抢占点因此天然落在"从内核返回用户态"的那一刻**，
   这正好是我们想要的位置。

但 `sie.STIE` 这一层没人替我们开 —— 必须在启动第一个应用之前显式设置：

```rust
// os/src/trap/mod.rs
use riscv::register::sie;

pub fn enable_timer_interrupt() {
    unsafe { sie::set_stimer(); }
}
```

漏掉这一句的典型症状：系统能跑、应用能切换（靠 `sys_yield`），但 `power` 系列这种不主动让出的应用
会一个跑到底，看起来像"时间片根本没生效"。

### 抢占点：trap_handler 多一个分支

```rust
// os/src/trap/mod.rs
match scause.cause() {
    Trap::Exception(Exception::UserEnvCall) => { /* 系统调用 */ }
    Trap::Interrupt(Interrupt::SupervisorTimer) => {
        set_next_trigger();                 // 先续上下一枪
        suspend_current_and_run_next();     // 再把当前任务换下去
    }
    // ...
}
```

初始化时按顺序做三件事：

```rust
// os/src/main.rs
trap::enable_timer_interrupt();   // 打开 sie.STIE
timer::set_next_trigger();        // 装上第一枪（10 ms 后）
task::run_first_task();           // 进入第一个应用
```

### 一个时间片的完整时序

```text
U 态: app A 正在执行
        │
        │ ① mtime 追平 mtimecmp → STIP 置位 → 硬件跳 stvec
        │    （sepc = 被打断的那条 U 指令，SPP=U，SPIE←SIE=1，SIE←0）
        ▼
S 态: __alltraps → 在 A 的内核栈上保存 TrapContext_A
        │
        ▼
      trap_handler: scause = SupervisorTimer
        │ ② set_next_trigger()：把 mtimecmp 推到 10 ms 之后
        │ ③ suspend_current_and_run_next()：A 置 Ready，选 B，__switch
        ▼
      B 的内核栈 → __restore(B 的 TrapContext) → sret
        │
        ▼
U 态: app B 继续执行（B 上次也是在自己的 sepc 处被打断的）
```

注意 ③ 里的 `__switch` 对 A 来说是"挂起"：A 的内核栈上从 `__alltraps` 到 `__switch` 的帧全部保留，
`TrapContext_A.sepc` 指向它被打断的那条指令。等 A 再次被选中，`__switch` 返回到
`run_next_task` 的下一行，一路退栈回到 `__restore`，恢复 `TrapContext_A`，`sret` 回到 U 态 ——
**从被打断的那条指令继续，应用完全感知不到自己被暂停过**。这就是"对应用透明"的具体含义。

### 遗留问题：通向 ch4

本章的"多道程序"还停留在"共享同一片物理内存"的阶段，几个明显的坑留在后面填：

| 遗留问题 | 后果 | 后续章节 |
| --- | --- | --- |
| 没有地址空间隔离 | 应用可以读写内核、读写别的应用 | ch4 引入页表，每个应用一套地址空间 |
| 用户栈全局共用 | 被切走的任务，栈帧可能被后来的任务覆盖 | ch4 每个应用独立用户栈 |
| 应用地址 / 数量写死 | `0x20000` 上限、`MAX_APP_NUM` 上限 | 后续引入更灵活的加载方式 |
| 任务退出不回收 | 内核栈、TCB 永久占用 | 后续实现资源回收 |
| 用户指针不校验 | `sys_write` / `sys_get_time` 可被任意地址攻击 | 后续加入地址检查 / 页表权限 |

### 避坑清单

1. 忘了 `sie::set_stimer()` → `STIE = 0`，时钟中断永远进不来，退化成协作式调度。
2. 忘了在 `SupervisorTimer` 分支里 `set_next_trigger()` → 只抢占一次，之后系统又变成"跑完一个换下一个"。
3. `set_timer` 传了**增量**而不是绝对时刻 → `mtimecmp` 设成了过去的时间，中断行为完全失控。
4. `CLOCK_FREQ` 与实际平台不符 → 时间片变成 100 ms 或 1 ms，表现为"切换太慢"或"几乎全在切换"。
5. 在 `trap_handler` 里手动开 `sstatus.SIE` 却没处理嵌套 → 内核栈被第二次 trap 覆盖，
   表现是随机崩溃、且崩溃点与真实原因无关。
6. 把时钟中断当成异常处理（漏了 `Trap::Interrupt(_)` 分支）→ 走到 `_ => panic!("Unsupported trap ...")`。
7. 修改 `TICKS_PER_SEC` 后忘了同步检查 `CLOCK_FREQ` 的整除关系 → 除出来的 tick 数取整误差累积，
   时间片实际长度与预期不符。
