## cargo基本认识

cargo是一个**包管理器**，提供一系列工具完善rust开发流程。

### 创建一个cargo项目

使用 cargo new 项目名创建cargo项目

```
cargo new 项目名

//项目结构如是：
.
├── .git
├── .gitignore
├── Cargo.toml
└── src
    └── main.rs
```

**使用cargo run编译并运行rust项目**

是cargo build 和 ./target/release/world_hello的连续操作。

cargo build命令会快速编译程序，但代码运行速度会变慢，因为**编译器不会对代码做任何优化**。

使用 cargo run --release 和 cargo build --release获得高性能代码

**快速编译命令**

$ cargo check

## Cargo.toml和Cargo.lock

Cargo.toml是cargo特有的项目项目描述文件，存储项目所有的元配置信息。

Cargo.lock是项目依赖表

**自定义项目依赖**

- 基于 Rust 官方仓库 `crates.io`，通过版本说明来描述
- 基于项目源代码的 git 仓库地址，通过 URL 来描述
- 基于本地项目的绝对路径或者相对路径，通过类 Unix 模式的路径来描述

```
[dependencies]
rand = "0.3"
hammer = { version = "0.5.0"}
color = { git = "https://github.com/bjz/color-rs" }
geometry = { path = "crates/geometry" }
```