# 简单了解ch2源码



## 批处理系统的思想

可以执行多个应用程序，执行完一个再加载下一个到内存中。

**操作系统（批处理系统）**：只负责机械地跳转到**预定的固定地址**（比如 `0x80000000`）去执行代码。它不需要知道程序里面是什么。

**程序员（或构建系统）**：负责通过**链接脚本（`linker.ld`）**，在编译阶段就“规定死”用户程序的布局，保证程序的第一条指令（`_start`）正好落在那个固定的预定位址上



## 批处理系统内存布局约束

我们使用链接脚本 `user/src/linker.ld` 规定用户程序的内存布局：

- 将程序的起始物理地址调整为 `0x80400000` ，三个应用程序都会被加载到这个物理地址上运行；
- 将 `_start` 所在的 `.text.entry` 放在整个程序的开头 `0x80400000`； 批处理系统在加载应用后，跳转到 `0x80400000`，就进入了用户库的 `_start` 函数；
- 提供了最终生成可执行文件的 `.bss` 段的起始和终止地址，方便 `clear_bss` 函数使用。

``` text
OUTPUT_ARCH(riscv)	
ENTRY(_start)
BASE_ADDRESS = 0x80200000;   

SECTIONS
{
    . = BASE_ADDRESS;
    skernel = .;
	
	//rustsbi对硬件初始化后会将pc指针跳转到0x80200000上，将内核镜像入口.text.entry放置在该地址上	
    stext = .;
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }

	//先经过页对齐（4k）
	//存放只读数据段
    . = ALIGN(4K);
    etext = .;
    srodata = .;
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }

    . = ALIGN(4K);
    erodata = .;
    sdata = .;
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }

	//存放已初始化数据
    . = ALIGN(4K);
    edata = .;
    .bss : {
        *(.bss.stack)
        sbss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
    }

	//存放未初始化数据
    . = ALIGN(4K);
    ebss = .;
    ekernel = .;

    /DISCARD/ : {
        *(.eh_frame)
    }
}
```



## entry.asm应用入口汇编代码

```asm
    .section .text.entry
    .globl _start #创建全局符号_start，链接器据此找到程序入口
_start:
    la sp, boot_stack_top #设置栈顶指针到sp寄存器
    call rust_main #跳转到rust_main代码进行运行

	#在.bss.stack段预留栈空间
    .section .bss.stack
    .globl boot_stack_lower_bound
boot_stack_lower_bound:
    .space 4096 * 16	#预留4k*16的栈空间
    .globl boot_stack_top
boot_stack_top:
```



## 实现应用程序

ch2将实现一个简单的批处理系统，将多个程序打包到一起输入计算机；当一个程序运行结束后，计算机会 *自动* 执行下一个程序。且要实现**特权级机制**，将用户态和内核态隔离，一个应用出错后不会导致整个内核的崩溃。

### 项目代码树

在本项目中，内核代码和用户代码被分隔到不同的文件夹中，内核代码在os/中，用户代码在user/中。

```
── os
│   ├── Cargo.toml
│   ├── Makefile (修改：构建内核之前先构建应用)
│   ├── build.rs (新增：生成 link_app.S 将应用作为一个数据段链接到内核)
│   └── src
│       ├── batch.rs(新增：实现了一个简单的批处理系统)
│       ├── console.rs
│       ├── entry.asm
│       ├── lang_items.rs
│       ├── link_app.S(构建产物，由 os/build.rs 输出)
│       ├── linker.ld
│       ├── logging.rs
│       ├── main.rs(修改：主函数中需要初始化 Trap 处理并加载和执行应用)
│       ├── sbi.rs
│       ├── sync(新增：包装了RefCell，暂时不用关心)
│       │   ├── mod.rs
│       │   └── up.rs
│       ├── syscall(新增：系统调用子模块 syscall)
│       │   ├── fs.rs(包含文件 I/O 相关的 syscall)
│       │   ├── mod.rs(提供 syscall 方法根据 syscall ID 进行分发处理)
│       │   └── process.rs(包含任务处理相关的 syscall)
│       └── trap(新增：Trap 相关子模块 trap)
│           ├── context.rs(包含 Trap 上下文 TrapContext)
│           ├── mod.rs(包含 Trap 处理入口 trap_handler)
│           └── trap.S(包含 Trap 上下文保存与恢复的汇编代码)
```

```
── user(新增：应用测例保存在 user 目录下)
   ├── Cargo.toml
   ├── Makefile
   └── src
      ├── bin
      │   ├── ...
      ├── console.rs
      ├── lang_items.rs
      ├── lib.rs(用户库 user_lib)
      ├── linker.ld(应用的链接脚本)
      └── syscall.rs(包含 syscall 方法生成实际用于系统调用的汇编指令，
                     各个具体的 syscall 都是通过 syscall 来实现的)
```



### 实现用户程序

用户程序在user/src/bin/xxx.rs中，如ch2_hello_world.rs：

```rust
#![no_std]
#![no_main]

#[macro_use]
extern crate user_lib;

/// 正确输出：
/// Hello world from user mode program!

#[no_mangle]
fn main() -> i32 {
    println!("Hello, world from user mode program!");
    0
}
```

可以看到它引用了一个外部库**"user_lib"**（在rust中称为crate），在`user/Cargo.toml`中对库的名字进行了定义：name = "user_lib"，这个库其实就是 `user/src` 目录下的 `lib.rs` 以及它引用的若干子模块，等价于其他语言的标准库。



`usr/lib.rs`中的`_start`函数即**用户态程序的“最小运行时入口”**：

​	lib.rs中的_start是所有用户程序的入口函数。`#[link_section = ".text.entry"]`该宏将 `_start` 函数编译后的汇编代码放在名为 `.text.entry` 的代码段中， 方便用户库链接脚本将它作为用户程序的入口。启动流程是（`_start` → `main` → `exit`），过 `exit` 系统调用返回批处理系统，由系统决定加载下一个程序。

```rust
#[no_mangle]
#[link_section = ".text.entry"]
pub extern "C" fn _start(argc: usize, argv: usize) -> ! {
    clear_bss();	//清零未初始化的全局/静态变量
    unsafe {
        HEAP.lock()
            .init(HEAP_SPACE.as_ptr() as usize, USER_HEAP_SIZE);	//初始化用户堆空间
    }
    let mut v: Vec<&'static str> = Vec::new();
    for i in 0..argc {
        let str_start =
            unsafe { ((argv + i * core::mem::size_of::<usize>()) as *const usize).read_volatile() };
        let len = (0usize..)
            .find(|i| unsafe { ((str_start + *i) as *const u8).read_volatile() == 0 })
            .unwrap();
        v.push(
            core::str::from_utf8(unsafe {
                core::slice::from_raw_parts(str_start as *const u8, len)
            })
            .unwrap(),
        );
    }
    exit(main(argc, v.as_slice()));		//执行用户程序并退出
}
```

```rust
#[linkage = "weak"]
#[no_mangle]
fn main(_argc: usize, _argv: &[&str]) -> i32 {
    //lib.rs中的弱链接main，会被用户程序中的强链接main覆盖
    //若用户程序忘记定义main，也能有个兜底报错
    panic!("Cannot find main!");
}
```

```rust
// 在bin/xxx.rs中的用户程序，它是强链接main
// _start中的exit会先调用这个，执行用户程序，再退出
#[no_mangle]
fn main() -> i32 {
    println!("Hello, world from user mode program!");
    0
}
```



### 实现系统调用

**用户态发起系统调用的底层入口`syscall`**：

```rust
// user/src/syacall.rs
pub fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret: isize;
    unsafe {
        core::arch::asm!(
            "ecall",
            inlateout("x10") args[0] => ret, //x10 = a0
            in("x11") args[1],	//x11 = a1
            in("x12") args[2],	//x12 = a2
            in("x17") id	//x17 = a7
        );
    }
    ret
}
```

这条汇编代码的执行结果是以寄存器 `a0~a2` 来保存系统调用的参数，以及寄存器 `a7` 保存 `syscall ID`， 返回值通过寄存器 `a0` 传递给局部变量 `ret`。



**调用包装器`syscall wrapper`**：

​	调用包装器**规定传入参数的约束**，为用户程序提供更简洁的接口。该函数会对传入的参数进行**数据处理与格式转换**，对`syacall`进行了封装，再统一传递给内核。

```rust
// user/src/syacall.rs
pub fn sys_close(fd: usize) -> isize {
    syscall(SYSCALL_CLOSE, [fd, 0, 0])
}

pub fn sys_read(fd: usize, buffer: &mut [u8]) -> isize {
    syscall(
        SYSCALL_READ,
        [fd, buffer.as_mut_ptr() as usize, buffer.len()],
    )
}

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
    syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}

```

在`user/src/lib.rs`中进一步进行封装，等待用户程序进行调用：

```rust
pub fn close(fd: usize) -> isize {
    if fd == STDOUT {
        console::flush();
    }
    sys_close(fd)
}

pub fn read(fd: usize, buf: &mut [u8]) -> isize {
    sys_read(fd, buf)
}

pub fn write(fd: usize, buf: &[u8]) -> isize {
    sys_write(fd, buf)
}
```



### 编译生成应用程序二进制码

简要介绍一下应用程序的构建，在 `user` 目录下 `make build`：

1. 对于 `src/bin` 下的每个应用程序， 在 `target/riscv64gc-unknown-none-elf/release` 目录下生成一个同名的 ELF 可执行文件；
2. 使用 objcopy 二进制工具删除所有 ELF header 和符号，得到 `.bin` 后缀的纯二进制镜像文件。 它们将被链接进内核，并由内核在合适的时机加载到内存。





## 实现批处理操作系统

### 将用户程序二进制镜像链接到内核

在 `os/src/main.rs`中引入了一段汇编代码：

**把所有用户程序的二进制镜像，静态嵌入到内核的 `.data` 段里，并给内核提供一张“应用目录表”**

```rust
//在/os/src/main.rs:40
global_asm!(include_str!("link_app.S"));
```

```asm

    .align 3	
    .section .data		#这段代码在 .data 段中构建了一个特定的数据结构
    .global _num_app
_num_app:	#_num_app 标签，是batch.rs代码中 extern "C" { fn _num_app(); } 寻找的符号入口
    .quad 7	#有7个用户程序
    .quad app_0_start	#存放每个用户程序的起始地址
    .quad app_1_start
    .quad app_2_start
    .quad app_3_start
    .quad app_4_start
    .quad app_5_start
    .quad app_6_start
    .quad app_6_end	#存放每个用户程序的结束地址

    .section .data
    .global app_0_start
    .global app_0_end
app_0_start:	
    .incbin "../user/build/bin/ch2b_bad_address.bin"	#直接嵌入二进制代码
app_0_end:

    .section .data
    .global app_1_start
    .global app_1_end
app_1_start:
    .incbin "../user/build/bin/ch2b_bad_instructions.bin"
app_1_end:

    .section .data
    .global app_2_start
    .global app_2_end
app_2_start:
    .incbin "../user/build/bin/ch2b_bad_register.bin"
app_2_end:

    .section .data
    .global app_3_start
    .global app_3_end
app_3_start:
    .incbin "../user/build/bin/ch2b_hello_world.bin"
app_3_end:

    .section .data
    .global app_4_start
    .global app_4_end
app_4_start:
    .incbin "../user/build/bin/ch2b_power_3.bin"
app_4_end:

    .section .data
    .global app_5_start
    .global app_5_end
app_5_start:
    .incbin "../user/build/bin/ch2b_power_5.bin"
app_5_end:

    .section .data
    .global app_6_start
    .global app_6_end
app_6_start:
    .incbin "../user/build/bin/ch2b_power_7.bin"
app_6_end:

```



### 加载程序二进制码

在`/os/src/batch.rs`中实现应用管理器`AppManager`：

```rust
struct AppManager {
    num_app: usize,
    current_app: usize,
    app_start: [usize; MAX_APP_NUM + 1],
}
```

用来初始化`AppManager`实例**`APP_MANAGER: UPSafeCell<AppManager>`**：

```rust
lazy_static! {	//lazy_static! 允许你定义一个静态变量，并在第一次被访问时才执行初始化闭包,不用在编译阶段确定
    		   //只有在使用时才会初始化APP_MANAGER
    static ref APP_MANAGER: UPSafeCell<AppManager> = unsafe {
        UPSafeCell::new({
            extern "C" {	
                fn _num_app();	//引用了一个外部的c语言函数: _num_app()
            }
            let num_app_ptr = _num_app as usize as *const usize;
            let num_app = num_app_ptr.read_volatile();
            let mut app_start: [usize; MAX_APP_NUM + 1] = [0; MAX_APP_NUM + 1];
            let app_start_raw: &[usize] =
                core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1);
            app_start[..=num_app].copy_from_slice(app_start_raw);
            AppManager {
                num_app,
                current_app: 0,
                app_start,
            }
        })
    };
}
```

找到 `link_app.S` 中提供的符号 `_num_app` ，并从这里开始解析出应用数量以及各个应用的开头地址。 用容器 `UPSafeCell` 包裹 `AppManager` 是为了防止全局对象 `APP_MANAGER` 被重复获取。



`AppManager`的`load_app`是核心方法，将要运行的**下一个应用程序**加载到固定位置`0x80400000`上：

``` rust
    unsafe fn load_app(&self, app_id: usize) {
        // 边界检查
        if app_id >= self.num_app {
            println!("All applications completed!");
            use crate::board::QEMUExit;
            crate::board::QEMU_EXIT_HANDLE.exit_success();
        }
        println!("[kernel] Loading app_{}", app_id);
        // 清空应用加载区，将预留给应用程序的内存区域（从 APP_BASE_ADDRESS 开始，大小为 APP_SIZE_LIMIT）全部清零。
        core::slice::from_raw_parts_mut(APP_BASE_ADDRESS as *mut u8, APP_SIZE_LIMIT).fill(0);
        // 复制应用的二进制数据
        let app_src = core::slice::from_raw_parts(	
            self.app_start[app_id] as *const u8,
            self.app_start[app_id + 1] - self.app_start[app_id],
        );
        
        let app_dst = core::slice::from_raw_parts_mut(APP_BASE_ADDRESS as *mut u8, app_src.len());
        app_dst.copy_from_slice(app_src);
        //清零指令缓存 (i-cache)
        asm!("fence.i");
    }
```

对`core::slice::from_raw_parts`的使用：

​	这类方法的主要目的就是将裸指针`data`和切片长度`len`，转换为rust切片，如此绕过安全检查并调用`fill()，copy_from_slice()`等方法。

```rust
//你需要保证：
//data 是有效的指针，指针必须对齐到 T 的对齐要求。
//data 指向的内存至少能容纳 len 个 T，len是元素个数而非字节数。
//如果是 from_raw_parts_mut，这块内存必须真正可变
//不能同时存在多个可变引用，也不能在不可变引用存在时创建可变引用；
//必须在unsafe代码块中
unsafe{
    core::slice::from_raw_parts_mut(APP_BASE_ADDRESS as *mut u8, APP_SIZE_LIMIT).fill(0);

    let app_src = core::slice::from_raw_parts(self.app_start[app_id] as *const u8, self.app_start[app_id + 1] - self.app_start[app_id],);
    let app_dst = core::slice::from_raw_parts_mut(APP_BASE_ADDRESS as *mut u8, app_src.len());
    app_dst.copy_from_slice(app_src);
}

```



`batch` 子模块对外暴露出如下接口：

- `init` ：调用 `print_app_info` 的时第一次用到了全局变量 `APP_MANAGER` ，它在这时完成初始化(lazy)。对于`APP_MANAGER`中的数据`AppManager`，一次只有一个可变借用；
- `run_next_app` ：批处理操作系统的核心操作，即加载并运行下一个应用程序。 批处理操作系统完成初始化，或者应用程序运行结束/出错后会调用该函数。



## 实现特权级的切换

### 特权级切换时，保存状态的寄存器

| CSR 名  | 该 CSR 与 Trap 相关的功能                                    |
| ------- | ------------------------------------------------------------ |
| sstatus | `SPP` 等字段给出 Trap 发生之前 CPU 处在哪个特权级（S/U）等信息 |
| sepc    | 当 Trap 是一个异常的时候，记录 Trap 发生之前执行的最后一条指令的地址 |
| scause  | 描述 Trap 的原因，通过 `scause` 判断是不是系统调用           |
| stval   | 给出 Trap 附加信息                                           |
| stvec   | 控制 Trap 处理代码的入口地址                                 |

特权级切换的具体过程一部分由硬件直接完成，另一部分则需要由操作系统来实现。

**CSR 就是 RISC-V 的“控制和状态寄存器”，用来保存和处理 CPU 的运行状态**。



这是`riscv`常见处理系统调用而导致的trap的方法：

![image-20261023172003740](C:\Users\86567\AppData\Roaming\Typora\typora-user-images\image-20261023172003740.png)



### 特权级切换时硬件的工作

当 CPU 执行完一条指令并准备从用户特权级 陷入（ `Trap` ）到 S 特权级的时候，硬件会自动完成如下这些事情：

- `sstatus` 的 `SPP` 字段会被修改为 CPU 当前的特权级（U/S）。
- `sepc` 会被修改为 Trap 处理完成后默认会执行的下一条指令的地址。
- `scause/stval` 分别会被修改成这次 Trap 的原因以及相关的附加信息。
- CPU 会跳转到 `stvec` 所设置的 Trap 处理入口地址，并将当前特权级设置为 S ，然后从Trap 处理入口地址处开始执行。

而当 CPU 完成 Trap 处理准备返回的时候，需要通过一条 S 特权级的特权指令 `sret` 来完成，这一条指令具体完成以下功能：

- CPU 会将当前的特权级按照 `sstatus` 的 `SPP` 字段设置为 U 或者 S ；
- CPU 会跳转到 `sepc` 寄存器指向的那条指令，然后继续执行。



### 硬件没有做的事：软件必须补上的部分

上面这四条只完成了“跳转 + 记账”，真正麻烦的三件事硬件都不管：

- **没有切换栈指针** `sp`：Trap 刚发生时 `sp` 仍然指向用户栈；
- **没有保存任何通用寄存器**：x1~x31 里全是用户程序的值；
- **没有保存 `sstatus` / `sepc` 的旧值**：`SPP`、`SIE`、`sepc` 已经被硬件改写，原值没了；

另外，`sret` 除了恢复特权级和 pc，还会附带做：`SIE ← SPIE`、`SPIE ← 1`、将特权级切换为U`SPP ← U`。

所以 `stvec` 指向的第一条指令必须立刻完成“换栈 + 保存现场”，而且过程中不能弄丢任何一个寄存器的值——这就引出了 `sscratch` 的用法。



补充：

SSP:`sstatus`中的一个比特位`bit8`，记录trap前的特权级

SIE：`sstatus`中的一个比特位`bit1`，S 态全局中断使能，该位决定当前是否启用中断。

SPIE：`sstatus`中的一个比特位`bit5`，SIE的备份，用于trap发生时存储当前`SIE`，用于`SIE`恢复处理。



### 整体流程

```text
U 态:  app 执行 ecall / 访存出错
         │
         │ ① 硬件自动: 特权级→S, pc←stvec, sepc←出错指令, scause/stval 记录原因
         │             sstatus.SPP←U, SPIE←SIE, SIE←0
         │            ⚠ sp 不变，仍指向用户栈
         ▼
S 态:  __alltraps
         │ ② csrrw 交换 sp/sscratch  →  sp 指向内核栈
         │ ③ 在内核栈上开 34*8 的帧，把 x1~x31、sstatus、sepc 全存进去
         │ ④ a0 = 这个帧的地址，call trap_handler
         ▼
       trap_handler (Rust)
         │ ⑤ 读 scause 分发: 系统调用 / 缺页 / 非法指令 / 其他
         │ ⑥ 直接改栈上上下文里的 sepc、a0 等字段
         │ ⑦ 原样返回这个上下文（a0 保持不变）
         ▼
       __restore
         │ ⑧ 从栈上恢复 sstatus/sepc/通用寄存器
         │ ⑨ 回收帧，再 csrrw 交换 → sp 回到用户栈，sscratch 回到内核栈
         │ ⑩ sret: 特权级←SPP, pc←sepc，回到 U 态
         ▼
U 态:  从 ecall 的下一条指令继续
```

核心其实只有一句话：**“Trap 入口先用 `sscratch` 把栈从用户栈换成内核栈，在内核栈上保存一个固定布局的 `TrapContext`，交给 Rust 处理，再用同一份上下文执行一次 `sret` 返回”**。汇编只负责“换栈 + 搬运上下文”，处理逻辑全在 Rust 里。



### 换栈：sscratch 与那个必须始终成立的不变量

换栈至少需要一个空闲寄存器来中转，可 Trap 刚发生时 32 个通用寄存器全是用户的值，一个都不能破坏。`sscratch` 就是硬件预先留好的“藏起来的那一个”。

约定（**不变量，破坏它第一次 Trap 就会崩**）：

> U 态运行时 `sscratch` 永远保存内核栈顶；一旦进入内核，`sscratch` 就变成用户栈顶。

| 时刻 | `sp` | `sscratch` |
| --- | --- | --- |
| U 态运行中 | 用户栈顶 | 内核栈顶 |
| `__alltraps` 第一条 `csrrw` 之后 | 内核栈顶 | 用户栈顶 |
| `__restore` 最后一条 `csrrw` 之后 | 用户栈顶 | 内核栈顶 |

```asm
csrrw sp, sscratch, sp   # 原子交换: sp←旧sscratch, sscratch←旧sp
```

一条指令完成换栈，不占用任何通用寄存器。入口处和返回处各一次，完全对称。



### 内核栈与 TrapContext：内存布局就是 ABI

内核栈和用户栈都只是字节数组的包装，实例化在 `.bss` 段里，4096 对齐；`get_sp` 返回数组末尾地址（RISC-V 栈向下增长）：

```rust
// os/src/batch.rs
#[repr(align(4096))]
struct KernelStack { data: [u8; KERNEL_STACK_SIZE] }
#[repr(align(4096))]
struct UserStack   { data: [u8; USER_STACK_SIZE] }

impl UserStack {
    fn get_sp(&self) -> usize {
        self.data.as_ptr() as usize + USER_STACK_SIZE
    }
}
```

Trap 时保存的上下文，主要保存 32 个通用寄存器和 Trap 时重要的 CSR：

```rust
//os/src/trap/context.rs

use riscv::register::sstatus::{self, Sstatus, SPP};
/// Trap Context
#[repr(C)]
pub struct TrapContext {
    /// general regs[0..31]
    pub x: [usize; 32],
    /// CSR sstatus      
    pub sstatus: Sstatus,
    /// CSR sepc
    pub sepc: usize,
}

impl TrapContext {
    /// set stack pointer to x_2 reg (sp)
    pub fn set_sp(&mut self, sp: usize) {
        self.x[2] = sp;
    }
    /// init app context
    pub fn app_init_context(entry: usize, sp: usize) -> Self {
        let mut sstatus = sstatus::read(); // CSR sstatus
        sstatus.set_spp(SPP::User); //previous privilege mode: user mode
        let mut cx = Self {
            x: [0; 32],
            sstatus,
            sepc: entry, // entry point of app
        };
        cx.set_sp(sp); // app's user stack pointer
        cx // return initial Trap Context of app
    }
}
```

**这个结构体的偏移就是汇编里的硬编码常量**，所以必须 `#[repr(C)]`、字段必须定长（`Sstatus` 是 `usize` 的包装，8 字节）：

| 偏移 | 内容 | 说明 |
| --- | --- | --- |
| `0*8` | x0 | 恒为 0，不保存，仅占位 |
| `1*8` | x1 (ra) | 保存 |
| `2*8` | x2 (sp) | 保存，但来源是 `sscratch` 而非当前 sp |
| `3*8` | x3 (gp) | 保存 |
| `4*8` | x4 (tp) | U 态不用，仅占位 |
| `5*8 ~ 31*8` | x5~x31 | `.rept` 循环保存 |
| `32*8` | sstatus | 保存 |
| `33*8` | sepc | 保存 |
| 合计 | `34*8 = 272` 字节 | 与 `addi sp, sp, -34*8` 对应 |

几个设计取舍：

- **为什么全存通用寄存器**：内核 Trap 路径会调用大量 Rust 代码，没法静态分析出哪些寄存器没被用到，索性全存。
- **为什么存 sstatus/sepc，却不存 scause/stval**：`scause/stval` 在 `trap_handler` 开头读一次就用完，被覆盖也无所谓；而 `sstatus/sepc` 在整个内核执行期间都有意义（`sret` 还要用），还可能被 Trap 嵌套覆盖，必须存到栈上再恢复。
- **为什么 x0/tp 仍留槽位**：为了偏移整齐，后面加字段不用重排。



### 保存现场：`__alltraps`

```asm
.altmacro                # %n 形式的参数替换需要它
.macro SAVE_GP n
    sd x\n, \n*8(sp)
.endm

.align 2                 # 4 字节对齐，stvec Direct 模式要求地址低 2 位为 0
__alltraps:
    csrrw sp, sscratch, sp   # ① 换栈：sp→内核栈顶，sscratch→用户栈顶
    addi  sp, sp, -34*8      # ② 在内核栈上开 TrapContext 大小的帧

    sd x1, 1*8(sp)           # ③ ra
    sd x3, 3*8(sp)           #    gp
    # x2 先跳过：现在 sp 是内核栈，用户 sp 还藏在 sscratch 里
    # x4(tp) 跳过：U 态不用

    .set n, 5                # ④ x5~x31
    .rept 27
        SAVE_GP %n
        .set n, n+1
    .endr

    csrr t0, sstatus         # ⑤ 此时 t0/t1/t2 已存进栈，可以随便用
    csrr t1, sepc
    sd   t0, 32*8(sp)
    sd   t1, 33*8(sp)

    csrr t2, sscratch        # ⑥ t2 = 用户 sp，补存到 x2 的槽位
    sd   t2, 2*8(sp)

    mv   a0, sp              # ⑦ 第一个参数 = &mut TrapContext
    call trap_handler        #    从这里开始才是"正常"的 Rust 函数调用
```

1. **顺序不能乱**：必须先把 t0/t1/t2（属于 x5~x31）存进栈，才能拿它们当临时寄存器去读 CSR；反过来先读再存，临时寄存器就被覆盖了。
2. **x2 要特殊处理**：直接 `sd x2, 2*8(sp)` 存的是内核栈指针，是错的；所以要 `t2 ← sscratch` 绕一步。
3. **`a0` 被覆盖没关系**：x10 早就在第 ④ 步存好了。
4. 在 `call trap_handler` 之前，内核栈是“手工”用的（\(sp-272\)）；之后才交给编译器的调用约定。

栈帧示意（内核栈向下增长）：

```text
高地址  ← KERNEL_STACK.get_sp()  (内核栈顶，即 sscratch 的原值)
        ┌──────────────────┐
 +264   │ sepc             │
 +256   │ sstatus          │
  ...   │ ...              │
  +16   │ x2 = 用户 sp     │
   +8   │ x1 (ra)          │
   +0   │ x0 (未使用)      │  ← sp 指向这里，也就是 cx 的地址
        ├──────────────────┤
        │ trap_handler 栈帧 │  (在 TrapContext 下方生长，返回时自动回收)
低地址  └──────────────────┘
```

`trap_handler` 自己的栈帧长在 TrapContext **下面**，返回时自动收回，所以返回后 `sp` 精确指回 TrapContext 起点，`__restore` 里 `addi sp, +34*8` 才能刚好回到栈顶。



### 分发处理：`trap_handler`

```rust
// os/src/trap/mod.rs
#[no_mangle]
pub fn trap_handler(cx: &mut TrapContext) -> &mut TrapContext {
    let scause = scause::read();
    let stval  = stval::read();
    match scause.cause() {
        Trap::Exception(Exception::UserEnvCall) => {
            cx.sepc += 4;                                   // ①
            cx.x[10] = syscall(cx.x[17],
                [cx.x[10], cx.x[11], cx.x[12]]) as usize;   // ②
        }
        Trap::Exception(Exception::StoreFault)
        | Trap::Exception(Exception::StorePageFault) => { /* 打印并 run_next_app() */ }
        Trap::Exception(Exception::IllegalInstruction) => { /* 同上 */ }
        _ => panic!("Unsupported trap {:?}, stval = {:#x}!", scause.cause(), stval),
    }
    cx                                                      // ③
}
```

① **`cx.sepc += 4` 改的是栈上副本，不是 CSR**。硬件把 `sepc` 设成了 ecall 本身，要让应用从下一条指令继续，必须 `+4`；但真正的 `sepc` CSR 要等 `__restore` 里 `csrw sepc, t1` 才更新。“改上下文”和“改 CSR”是两个分开的动作。

② a7 是 syscall ID，a0~a2 是参数，返回值写回 `cx.x[10]`（即 a0）。**必须从栈上的上下文取值**，因为进入内核后寄存器可能已被 Rust 代码改过。

③ **返回 `cx` 是一条暗线**：`__restore` 第一条指令是 `mv sp, a0`，它默认 `a0 == cx == sp`。若返回 `()`，a0 就是垃圾值，`__restore` 立刻踩飞。这里故意原样返回，让 RISC-V 调用约定把指针放回 a0。



### 恢复现场：`__restore`

```asm
.macro LOAD_GP n
    ld x\n, \n*8(sp)
.endm

__restore:
    mv sp, a0                # ① 两种情况都成立（见后文）
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    ld t2, 2*8(sp)           #    t2 = 用户 sp
    csrw sstatus, t0         # ② 先恢复 CSR
    csrw sepc,    t1
    csrw sscratch, t2        #    用户 sp 先塞进 sscratch

    ld x1, 1*8(sp)           # ③ 再恢复通用寄存器
    ld x3, 3*8(sp)
    .set n, 5
    .rept 27
        LOAD_GP %n           #    这里会把 t0/t1/t2 覆盖掉——无所谓，CSR 已写完
        .set n, n+1
    .endr

    addi sp, sp, 34*8        # ④ 回收帧，sp 回到内核栈顶
    csrrw sp, sscratch, sp   # ⑤ 交换：sp←用户栈顶，sscratch←内核栈顶
    sret                     # ⑥ 特权级←SPP，pc←sepc，SIE←SPIE，SPIE←1
```

1. **先 CSR 后通用寄存器**：恢复通用寄存器会把 t0/t1/t2 覆盖成用户的值，之后再想 `csrw sstatus` 就没有干净寄存器可用了。顺序反了必崩。
2. **为什么没有 `ld x2, 2*8(sp)`**：此刻 `sp` 正指着这块 TrapContext 当基址，一旦恢复 x2，基址当场失效，后面的 `addi` / `csrrw` 全错。所以用“先把用户 sp 塞进 `sscratch`，最后用一次 `csrrw` 换进 `sp`”的写法——和入口处完全对称，这是本段第二个巧妙之处。
3. `sret` 是 RISC-V 中**唯一**能降低特权级的指令，所以哪怕只是“第一次启动应用”，也必须走 `sret`。



### 首次启动：把“启动应用”伪装成“一次 Trap 返回”

既然降特权级只能靠 `sret`，启动应用的思路就是：在内核栈上造一个“看起来刚从 U 态 Trap 进来”的上下文，然后走一遍 `__restore`。

```rust
// os/src/batch.rs
pub fn run_next_app() -> ! {
    let mut app_manager = APP_MANAGER.exclusive_access();
    let current_app = app_manager.get_current_app();
    unsafe { app_manager.load_app(current_app); }
    app_manager.move_to_next_app();
    drop(app_manager);                 // ① 手动释放，否则后面的代码再也拿不到它

    extern "C" { fn __restore(cx_addr: usize); }
    unsafe {
        __restore(KERNEL_STACK.push_context(
            TrapContext::app_init_context(APP_BASE_ADDRESS, USER_STACK.get_sp())
        ) as *const _ as usize);
    }
    panic!("Unreachable in batch::run_current_app!");
}
```

```rust
impl KernelStack {
    fn get_sp(&self) -> usize {
        self.data.as_ptr() as usize + KERNEL_STACK_SIZE
    }
    pub fn push_context(&self, cx: TrapContext) -> &'static mut TrapContext {
        let cx_ptr = (self.get_sp() - core::mem::size_of::<TrapContext>()) as *mut TrapContext;
        unsafe { *cx_ptr = cx; }
        unsafe { cx_ptr.as_mut().unwrap() }
    }
}
```

`app_init_context` 里三个字段各管一件事：

| 字段 | 取值 | 作用 |
| --- | --- | --- |
| `sepc` | `0x80400000` | `sret` 后 pc 跳到这里（链接脚本固定死的入口） |
| `x[2]` | `USER_STACK.get_sp()` | 换栈后 sp 指向用户栈顶 |
| `sstatus.SPP` | `User` | 决定 `sret` 后特权级降为 U |

其余寄存器清零，相当于给应用一个“干净”的初始状态。

**`push_context` 的隐含要求**：它把 `cx` 放在 `get_sp() - 272` 并返回该地址。这样 `__restore` 里 `mv sp, a0` 之后，`addi sp, +34*8` 恰好回到 `KERNEL_STACK.get_sp()`——正是 `sscratch` 应有的值。**若 `push_context` 留了任何填充或没对齐，`sscratch` 就会被设成错误地址，第二次 Trap 直接崩**。

> 补充：`sret` 会把 `SIE ← SPIE`、`SPIE ← 1`。但注意 S 级中断在 **U 态运行时本来就是放行的**（放行条件是“当前特权级 < S”，与 `SIE` 无关），所以 `SPIE = 0` 并不会让 U 态失去时钟中断，它影响的是“`sret` 返回到 S 态运行”时的中断开关。ch2 里不设置也无妨——此时 `sie` 中的定时器中断源根本没打开。



### 两条路径的对照

`__restore` 开头那句看起来奇怪的 `mv sp, a0`，是因为它要同时服务两条完全不同的调用路径：

| | 普通 Trap 返回 | 首次启动应用 |
| --- | --- | --- |
| 调用者 | `__alltraps` 中 `call trap_handler` 返回后顺序执行 | `run_next_app` 中 `call __restore` |
| 进入时 `sp` | 已指向 TrapContext | 指向 `run_next_app` 的栈帧 |
| `a0` | `trap_handler` 返回的 `cx`（== `sp`） | `push_context` 返回的地址 |
| `mv sp, a0` 的作用 | **幂等的空操作** | **丢弃 `run_next_app` 的栈帧，把 sp 拉到新上下文** |
| 上下文来源 | 真实 Trap 时保存的 | `app_init_context` 手工构造的 |

一次 `mv` 统一两条路径，代价是一条指令。



### 避坑清单

1. **`sscratch` 不变量被破坏**：任何一处漏掉 `csrrw`，第一次 Trap 就会把上下文写到用户栈上，或 `sret` 后 `sp` 指向内核栈。
2. **`34*8` 与 `size_of::<TrapContext>()` 不同步**：后续章节给 `TrapContext` 加 `kernel_sp` / `kernel_satp` 字段时，必须同步改 `trap.S` 里的 `-34*8` 和所有偏移，否则恢复出来的寄存器全部错位。
3. **`TrapContext` 丢了 `#[repr(C)]` 或字段不定长**：汇编硬编码偏移立即失效。
4. **`trap_handler` 返回 `()`**：`a0` 变垃圾值，`__restore` 的 `mv sp, a0` 直接踩飞。
5. **`__restore` 里先恢复通用寄存器再写 CSR**：t0/t1/t2 被覆盖，CSR 写不进去。
6. **忘记 `sepc += 4`**：应用会无限重复执行同一条 `ecall`。
7. **单内核栈的局限**：这里 `KERNEL_STACK` 全局唯一，只适用于“同一时刻只有一个应用”的批处理模型；要多任务或内核可重入，就得每个任务一个内核栈（第 3 章 `__switch` 与 `kernel_sp` 要解决的问题）。
8. **只处理了四种 Trap**：时钟中断、LoadFault 等都会走到 `panic!`；`SIE` 在 Trap 期间被硬件清零，这阶段本来也不会被时钟中断打断。
9. **`sys_write` 不校验用户指针**：越界的 `buf` 会让内核直接读非法地址，这是刻意留到地址空间章节解决的漏洞。
10. **`.align 2` 是 4 字节**（\(2^2\)），不是 2 字节。
