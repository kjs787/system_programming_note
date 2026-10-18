# system_programming_note

## 项目简介

这是一个系统编程方向的个人学习笔记仓库，记录从语言基础到操作系统内核的完整学习路径。内容以「原理讲解 + 代码示例 + 源码阅读」为主，既作为自己的知识索引，也希望能给同样在学习系统编程的人提供参考。

笔记主要覆盖以下几个方向：

- **C++**：语言基础、面向对象、STL 与泛型编程
- **Rust**：语法基础、所有权与生命周期、并发与异步、智能指针，以及 Cargo 工具链
- **操作系统**：OSTEP 理论、xv6 源码剖析、rCore 实践（no_std 开发、内核各阶段）
- **Linux 系统编程**：系统调用、文件 IO、进程与线程、Socket 网络编程
- **驱动开发**：Linux 驱动的组成框架、编译加载与内核配置

编写约定：

- 使用 Obsidian 维护，支持双链与 LaTeX 公式
- 代码块尽量保持可直接运行，原理部分配合图示与源码片段
- 单篇笔记按「概念 → 机制 → 源码/示例 → 小结」的顺序组织

## 项目目录

```text
system_programming_note
├── agent
│   └── dsh_config
│       ├── agent.md
│       └── skills
│           └── standardize.md
├── knowlegdes
│   ├── cpp
│   │   ├── cpp_fundamentals.md
│   │   ├── OOP.md
│   │   └── STL.md
│   ├── driver development
│   │   └── dirver_fudamentals.md
│   ├── operating_sysytem
│   │   ├── linux
│   │   │   └── linux_basics
│   │   │       ├── linux_system_programming.md
│   │   │       └── socket_programming.md
│   │   ├── OSTEP
│   │   │   └── virtualizing.md
│   │   ├── rcore
│   │   │   ├── basic stage
│   │   │   │   └── no_std_dev.md
│   │   │   └── professional stage
│   │   │       ├── ch0.md
│   │   │       └── ch1.md
│   │   └── xv6
│   │       ├── All Resources Needed.md
│   │       ├── Calls and Traps.md
│   │       ├── File Systems.md
│   │       ├── Interrupts and Drivers.md
│   │       ├── Locks.md
│   │       ├── OS Interface.md
│   │       ├── OS Structure.md
│   │       ├── Page Tables.md
│   │       └── Scheduling.md
│   └── rust
│       ├── advance_rust.md
│       ├── meet_cargo.md
│       └── rust_fundamentals.md
├── LICENSE
└── README.md
```

### 目录说明

| 目录 | 内容 |
| --- | --- |
| `knowlegdes/cpp` | C++ 语言基础、面向对象（封装/继承/多态、设计模式）、STL 容器与算法、空间配置器 |
| `knowlegdes/rust` | Rust 语法基础、所有权与借用、泛型与 trait、生命周期、错误处理；进阶部分含闭包、迭代器、智能指针、并发、内联汇编与 async |
| `knowlegdes/operating_sysytem/OSTEP` | 《操作系统导论》读书笔记，目前整理到进程抽象、受限直接执行与进程调度 |
| `knowlegdes/operating_sysytem/xv6` | xv6 源码阅读：OS 接口与结构、调用与陷阱、中断与驱动、页表、锁、调度、文件系统 |
| `knowlegdes/operating_sysytem/rcore` | rCore 训练营实践：no_std 开发、堆分配器、syscall 封装、并发原语，以及专业阶段各章节 |
| `knowlegdes/operating_sysytem/linux/linux_basics` | Linux 系统编程与 Socket 网络编程 |
| `knowlegdes/driver development` | Linux 驱动开发基础：驱动框架、编译加载、menuconfig 与内置编译 |
| `agent` | DSH 配置：`agent.md` 为常驻角色设定（嵌入式 Linux 系统架构师），`skills/standardize.md` 为 `/refactor` 笔记重构技能 |

## 正在做的事

1. 将其他平台的笔记转移到本仓库中
2. 学习 rust 基本语法
3. 抽空重构一些笔记

## 许可证

本项目采用 [MIT License](LICENSE)。
