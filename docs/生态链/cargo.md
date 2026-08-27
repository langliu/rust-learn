# Cargo：Rust 的包管理与构建工具

> Cargo 是 Rust 官方的包管理器、构建系统和项目脚手架。学 Rust 几乎等于同时学 Cargo：创建项目、加依赖、跑测试、发 crate，都走它。
> 本文命令均可在本仓库根目录或任意 crate 目录下验证。

---

## 目录

1. [Cargo 是什么](#1-cargo-是什么)
2. [安装与版本](#2-安装与版本)
3. [项目与包：crate / package / workspace](#3-项目与包crate--package--workspace)
4. [创建项目](#4-创建项目)
5. [Cargo.toml 速览](#5-cargotoml-速览)
6. [常用命令](#6-常用命令)
7. [依赖管理](#7-依赖管理)
8. [构建产物与 target](#8-构建产物与-target)
9. [特性（Features）](#9-特性features)
10. [工作区（Workspace）](#10-工作区workspace)
11. [测试、文档与发布](#11-测试文档与发布)
12. [常见误区](#12-常见误区)
13. [练习题](#13-练习题)

---

## 1. Cargo 是什么

Rust 编译器本体是 `rustc`。直接调 `rustc src/main.rs` 能编译单个文件，但真实项目还要：

- 组织多个源文件
- 拉取 crates.io 上的第三方库
- 区分 debug / release
- 跑测试、生成文档、发布版本

这些事都交给 **Cargo**。可以把它想成：

| 角色 | 类比 |
|------|------|
| 脚手架 | `npm create` / `cargo new` |
| 依赖管理 | `npm` / `pip` / Maven |
| 构建系统 | `make` / Gradle |
| 任务入口 | `cargo build` / `test` / `run` / `doc` |

一句话：

> **写 Rust 代码用 rustc；管 Rust 项目用 Cargo。**

---

## 2. 安装与版本

Cargo 随 [rustup](https://rustup.rs) 一起安装，不必单独装。工具链怎么装、怎么切，见 [rustup.md](rustup.md)。

```bash
rustc --version
cargo --version
rustup show
```

常用工具链切换：

```bash
rustup update                 # 更新 stable
rustup toolchain install nightly
rustup default stable         # 默认用 stable
```

项目里也可以固定工具链，在根目录放 `rust-toolchain.toml`：

```toml
[toolchain]
channel = "stable"
```

---

## 3. 项目与包：crate / package / workspace

这三个词经常混用，先分清：

| 概念 | 含义 |
|------|------|
| **crate** | 一次 `rustc` 编译的单元。二进制 crate 产出可执行文件，库 crate 产出 `rlib` |
| **package** | 一个 `Cargo.toml` 描述的包，至少包含一个 crate（`src/lib.rs` 和/或 `src/main.rs`） |
| **workspace** | 多个 package 共享同一份 `Cargo.lock` 和 `target/` 的仓库布局 |

本仓库就是 workspace：根 `Cargo.toml` 只有 `[workspace]`，真正的 crate 放在 `crates/` 下。

一个 package 可以同时有库和二进制：

```
my-pkg/
├── Cargo.toml
└── src/
    ├── lib.rs          # 库 crate
    └── main.rs         # 默认二进制 crate
```

多个二进制可放在 `src/bin/`：

```
src/bin/foo.rs    → cargo run --bin foo
src/bin/bar.rs    → cargo run --bin bar
```

---

## 4. 创建项目

```bash
cargo new hello                 # 二进制项目
cargo new greet --lib           # 库项目
cargo init                      # 在已有目录初始化
```

`cargo new hello` 大约生成：

```
hello/
├── Cargo.toml
├── .gitignore
└── src/main.rs
```

`Cargo.toml` 起步内容类似：

```toml
[package]
name = "hello"
version = "0.1.0"
edition = "2024"
```

`edition` 是语言版本（2015 / 2018 / 2021 / 2024），不是编译器版本。换 edition 不会自动升级 `rustc`。

---

## 5. Cargo.toml 速览

`Cargo.toml` 是清单（manifest），常见段落：

```toml
[package]
name = "guessing_game"
version = "0.1.0"
edition = "2024"
description = "猜数字小游戏"
license = "MIT"

[dependencies]
rand = "0.9"

[dev-dependencies]
pretty_assertions = "1"

[build-dependencies]
cc = "1"
```

| 段落 | 何时用 |
|------|--------|
| `[dependencies]` | 正常编译、运行需要 |
| `[dev-dependencies]` | 仅测试、示例、基准需要 |
| `[build-dependencies]` | `build.rs` 编译期脚本需要 |
| `[workspace]` | 声明工作区成员 |
| `[profile.release]` | 调优化级别、LTO 等 |

版本号遵循 **SemVer**。`"0.9"` 等价于 `^0.9`：允许 `0.9.x`，不允许 `0.10`。

锁定的精确版本写在 **`Cargo.lock`**：

- 应用程序：应提交 `Cargo.lock`，保证别人构建出同一份依赖树
- 纯库：通常不提交，让下游自己解析

---

## 6. 常用命令

| 命令 | 作用 |
|------|------|
| `cargo build` | 开发构建（`target/debug`） |
| `cargo build --release` | 优化构建（`target/release`） |
| `cargo run` | 构建并运行默认二进制 |
| `cargo run --bin NAME` | 运行指定二进制 |
| `cargo check` | 只做类型检查，最快的“能不能过编译” |
| `cargo test` | 跑单元/集成测试 |
| `cargo clippy` | 官方 linter（需 rustup component） |
| `cargo fmt` | 按 rustfmt 格式化 |
| `cargo doc --open` | 生成并打开 API 文档 |
| `cargo tree` | 看依赖树 |
| `cargo clean` | 清掉 `target/` |
| `cargo update` | 在 `Cargo.toml` 约束内更新 lock |

开发时优先 `cargo check`：它不生成可执行文件，反馈更快。

传参给程序本身时，用 `--` 隔开：

```bash
cargo run --bin quiz -- --help
```

`--` 前面是 Cargo 的参数，后面是程序的参数。

---

## 7. 依赖管理

### 从 crates.io 添加

```bash
cargo add rand
cargo add serde --features derive
cargo add tokio --features full
```

等价于手写：

```toml
[dependencies]
rand = "0.9"
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

### 其他来源

```toml
[dependencies]
# Git
anyhow = { git = "https://github.com/dtolnay/anyhow", branch = "master" }

# 本地路径（workspace 内最常见）
greeting = { path = "../greeting" }

# 工作区统一版本
serde = { workspace = true }
```

### 删依赖

```bash
cargo remove rand
```

### 看为什么拉进来某个包

```bash
cargo tree -i libc
```

---

## 8. 构建产物与 target

默认产物目录是项目（或 workspace）根下的 `target/`：

```
target/
├── debug/           # cargo build
├── release/         # cargo build --release
└── doc/             # cargo doc
```

workspace 共享一个 `target/`，所以本仓库根目录的 `target/` 属于整个工作区，不要提交（已在 `.gitignore`）。

指定别的输出目录：

```bash
CARGO_TARGET_DIR=/tmp/rust-out cargo build
```

---

## 9. 特性（Features）

crate 常用 feature 做可选功能，避免默认依赖过重。

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
```

自己的库也可以声明：

```toml
[features]
default = ["std"]
std = []
json = ["dep:serde"]
```

启用方式：

```bash
cargo build --features json
cargo build --no-default-features --features json
```

原则：**默认 feature 保持精简**；可选能力拆成独立 feature。

---

## 10. 工作区（Workspace）

本仓库根 `Cargo.toml` 类似：

```toml
[workspace]
members = ["crates/*"]
resolver = "3"

[workspace.dependencies]
# 所有 crate 共享的依赖版本，子 crate 用 `{ workspace = true }` 引用
```

好处：

1. 一次 `cargo test --workspace` 跑全部成员
2. 共享 `Cargo.lock` 和 `target/`，依赖版本一致、编译缓存复用
3. `[workspace.dependencies]` 避免每个 crate 各写一套版本号

在某个成员目录执行 `cargo run` 时，Cargo 会自动找到根 workspace。指定成员：

```bash
cargo run -p guessing_game
cargo test -p greeting
```

---

## 11. 测试、文档与发布

### 测试

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }
}
```

```bash
cargo test
cargo test it_works          # 按名字过滤
cargo test -- --nocapture    # 打印 stdout
```

集成测试放 `tests/*.rs`，每个文件是独立 crate。

### 文档

源码里的 `///` 会变成 docs.rs 风格文档，还可放可运行示例：

```rust
/// 把两个数加起来。
///
/// ```
/// assert_eq!(add(1, 2), 3);
/// ```
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

```bash
cargo test --doc             # 跑文档里的示例
cargo doc --open
```

### 发布到 crates.io

```bash
cargo login
cargo publish
```

发布前确认：`name` 未被占用、`license` 已填、公开 API 有文档。学习仓库不必发布。

---

## 12. 常见误区

1. **把 `Cargo.lock` 和 `Cargo.toml` 搞反**  
   toml 是约束（“我要 rand 0.9 这一代”），lock 是快照（“实际解析到 0.9.2 + 传递依赖 …”）。

2. **在 workspace 成员里各自 `cargo new` 出第二份 lock**  
   成员应只有自己的 `Cargo.toml`，lock 只留在 workspace 根。

3. **依赖版本写死 `=1.2.3` 过多**  
   应用可以锁得紧；库应对下游宽松，让 Cargo 解析兼容版本。

4. **一上来就 `--release`**  
   release 编译慢、调试难。日常开发用 debug；测性能、发二进制再用 release。

5. **忘记 `--` 传参**  
   `cargo run --bin quiz --help` 看到的是 Cargo 的 help，不是 quiz 的。

---

## 13. 练习题

1. 在 `crates/` 下新建一个二进制 crate `hello_cargo`，用 `cargo run -p hello_cargo` 打印 `hello, cargo`。
2. 给它加上 `rand` 依赖，打印一个 `1..=10` 的随机数。
3. 用 `cargo tree -p hello_cargo` 画出依赖树，指出 `rand` 还拉进了哪些包。
4. 写一个 `#[test]`，断言你的打印辅助函数返回的字符串包含 `hello`。
5. 解释：为什么本仓库的 `cargo build` 在根目录执行，产物却都进同一个 `target/`？

---

## 延伸阅读

- 官方书：[The Cargo Book](https://doc.rust-lang.org/cargo/)
- 依赖规范：[Specifying Dependencies](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html)
- 工作区：[Workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html)
- crates 索引：[crates.io](https://crates.io)
