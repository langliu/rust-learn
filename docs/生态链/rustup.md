# rustup：Rust 工具链管理器

> rustup 负责**安装、切换、更新** Rust 编译器及相关工具。Cargo 管项目，rustup 管“你机器上有哪几套 Rust”。
> 本文命令在终端直接运行即可；不依赖本仓库里的 crate。

---

## 目录

1. [rustup 是什么](#1-rustup-是什么)
2. [安装](#2-安装)
3. [装完之后多了什么](#3-装完之后多了什么)
4. [工具链：stable / beta / nightly](#4-工具链stable--beta--nightly)
5. [组件（Components）](#5-组件components)
6. [编译目标（Targets）](#6-编译目标targets)
7. [覆盖与 rust-toolchain.toml](#7-覆盖与-rust-toolchaintoml)
8. [常用命令](#8-常用命令)
9. [文档与本机 std](#9-文档与本机-std)
10. [常见误区](#10-常见误区)
11. [练习题](#11-练习题)

---

## 1. rustup 是什么

Rust 的发布方式不是“下一个安装包覆盖上一个”，而是**多套工具链并存**：

- 日常用 **stable**
- 尝新用 **nightly**
- 某个老项目钉死 **1.76.0**

[rustup](https://rustup.rs) 就是管这些工具链的官方程序。关系可以记成：

| 工具 | 管什么 |
|------|--------|
| **rustup** | 机器上的 `rustc` / `cargo` 版本、组件、交叉编译目标 |
| **cargo** | 当前项目的依赖、构建、测试、发布（见 [cargo.md](cargo.md)） |
| **rustc** | 把一份 crate 编译成库或可执行文件 |

一句话：

> **rustup 决定“用哪套 Rust”；cargo / rustc 在这套 Rust 里干活。**

---

## 2. 安装

官方入口：[https://rustup.rs](https://rustup.rs)

macOS / Linux：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

按提示选默认安装即可。装完后确认：

```bash
rustup --version
rustc --version
cargo --version
```

`rustc`、`cargo` 对不上或提示 command not found，把 `~/.cargo/bin` 加进 `PATH`（安装脚本一般会改 shell 配置，新开一个终端再试）。

国内下载慢时，可先设镜像再跑安装脚本（以中科大为例）：

```bash
export RUSTUP_UPDATE_ROOT=https://mirrors.ustc.edu.cn/rust-static/rustup
export RUSTUP_DIST_SERVER=https://mirrors.ustc.edu.cn/rust-static
```

---

## 3. 装完之后多了什么

默认会装 **stable** 工具链，并在 `~/.cargo/bin` 放一组**代理程序**：

```
~/.cargo/bin/rustc
~/.cargo/bin/cargo
~/.cargo/bin/rustup
~/.cargo/bin/rustfmt
~/.cargo/bin/clippy-driver
...
```

你敲 `rustc` 时，并不是直接调用某个版本的编译器，而是 rustup 的代理根据规则选出**当前生效的工具链**，再转交给它。

看此刻到底用了哪套：

```bash
rustup show
rustup which rustc
```

真正的编译器在 `~/.rustup/toolchains/` 下，按工具链名分目录。两个环境变量值得知道：

| 变量 | 默认 | 内容 |
|------|------|------|
| `RUSTUP_HOME` | `~/.rustup` | 工具链、设置 |
| `CARGO_HOME` | `~/.cargo` | 代理、cargo 配置、registry 缓存 |

---

## 4. 工具链：stable / beta / nightly

| 渠道 | 节奏 | 用途 |
|------|------|------|
| **stable** | 每 6 周发一版 | 日常开发、本仓库默认 |
| **beta** | 下一份 stable 的预览 | 提前验证即将稳定的改动 |
| **nightly** | 每天构建 | 不稳定特性、部分实验工具 |

还可以钉死具体版本或带日期的 nightly：

```text
stable
nightly
1.84.0
nightly-2025-03-01
```

安装与切换：

```bash
rustup toolchain install nightly
rustup toolchain install 1.84.0
rustup default stable              # 全局默认
rustup toolchain list
```

临时用另一套，不必改默认：

```bash
cargo +nightly -V
rustup run nightly rustc -V
```

`cargo +nightly` 这种写法是 rustup 代理识别的，不是 Cargo 自己的参数。

---

## 5. 组件（Components）

一套工具链里除了 `rustc`、`cargo`，还可以按需加组件：

| 组件 | 作用 |
|------|------|
| `rustfmt` | 格式化（`cargo fmt`） |
| `clippy` | linter（`cargo clippy`） |
| `rust-src` | 标准库源码，IDE 跳进 `Vec` 等需要它 |
| `rust-analyzer` | 语言服务（Zed 等编辑器会用） |
| `rust-docs` | 本机标准库文档 |
| `llvm-tools` | `llvm-cov` 等底层工具 |

```bash
rustup component add rustfmt clippy rust-src
rustup component list                  # 看已装 / 可装
rustup component remove llvm-tools
```

组件是**绑在某一套工具链上的**。给 nightly 加了 clippy，不等于 stable 也有。给指定工具链装：

```bash
rustup component add clippy --toolchain nightly
```

默认安装配置（profile）决定“第一套工具链带哪些组件”：

```bash
rustup set profile default    # 含 rustfmt、clippy 等，推荐
rustup set profile minimal    # 只含 rustc / cargo / rust-std，CI 常用
```

---

## 6. 编译目标（Targets）

`rustc` 默认编你这台机器的架构（host）。交叉编译要先加 **target**：

```bash
rustup target add wasm32-unknown-unknown
rustup target add aarch64-unknown-linux-gnu
rustup target list --installed
```

然后：

```bash
cargo build --target wasm32-unknown-unknown
```

target 同样挂在当前工具链上。只装了 wasm 目标、却用 `cargo +nightly` 编，如果 nightly 没加过这个 target，仍会失败。

---

## 7. 覆盖与 rust-toolchain.toml

生效优先级（高 → 低，简化理解）：

1. 命令行：`cargo +nightly build`
2. 当前目录（或父目录）的 `rust-toolchain.toml` / `rust-toolchain`
3. `rustup override set` 写在目录上的覆盖
4. `rustup default` 的全局默认

项目里推荐把版本写进仓库，别人 clone 下来 rustup 会自动下载对应工具链。

`rust-toolchain.toml` 示例：

```toml
[toolchain]
channel = "stable"
components = ["rustfmt", "clippy"]
# targets = ["wasm32-unknown-unknown"]
```

也可以只钉版本：

```toml
[toolchain]
channel = "1.84.0"
```

目录覆盖（不进 git，只影响你这台机器）：

```bash
cd /path/to/old-project
rustup override set 1.76.0
rustup override list
rustup override unset
```

本仓库若未放 `rust-toolchain.toml`，就用你的全局 default（一般是 stable）。

---

## 8. 常用命令

| 命令 | 作用 |
|------|------|
| `rustup show` | 当前工具链、覆盖、已装列表 |
| `rustup update` | 更新已装渠道（stable/beta/nightly） |
| `rustup update stable` | 只更新 stable |
| `rustup check` | 看看有没有新版本 |
| `rustup default stable` | 设全局默认 |
| `rustup toolchain install nightly` | 安装一套工具链 |
| `rustup toolchain uninstall nightly` | 卸掉一套，腾磁盘 |
| `rustup component add clippy` | 给当前工具链加组件 |
| `rustup target add <triple>` | 加交叉编译目标 |
| `rustup which cargo` | 当前 `cargo` 的真实路径 |
| `rustup self update` | 更新 rustup 自己 |
| `rustup self uninstall` | 卸载 rustup 及工具链 |

`rustup update` 更新的是**工具链**；`rustup self update` 更新的是 **rustup 这个管理器**。日常前者更常用。

---

## 9. 文档与本机 std

```bash
rustup doc            # 打开当前工具链的本地文档首页
rustup doc --std      # 直接打开标准库
rustup doc --cargo    # Cargo Book 本地版
```

不联网也能查 `Option`、`Iterator`。编辑器跳进标准库源码则依赖 `rust-src` 组件。

---

## 10. 常见误区

1. **用系统包管理器再装一份 Rust**  
   Homebrew / apt 的 `rustc` 往往滞后，还会和 rustup 抢 `PATH`。学习环境只保留 rustup 这一条。

2. **以为 `cargo install rustc` 能换编译器**  
   换版本走 rustup，不要用 cargo 装编译器。

3. **`edition = "2024"` 和工具链搞混**  
   `Cargo.toml` 里的 edition 是**语言版本**；能不能用这个 edition，取决于当前 `rustc` 够不够新。edition 换了，rustup 不会自动升级。

4. **给错了工具链装组件**  
   `cargo +nightly clippy` 失败时，检查是否执行过 `rustup component add clippy --toolchain nightly`。

5. **PATH 里混了多份 cargo**  
   `which -a rustc` 应首先指向 `~/.cargo/bin/rustc`。若前面还有 `/usr/bin/rustc`，优先修 PATH。

6. **把 `RUSTUP_HOME` 当项目目录**  
   工具链很占空间（每套数百 MB 到 1GB+），可 `rustup toolchain uninstall` 清不用的 nightly / 旧版本。

---

## 11. 练习题

1. 运行 `rustup show`，写出当前默认工具链名称，以及 `rustup which rustc` 的路径。
2. 安装 nightly，用 `cargo +nightly -V` 和 `cargo -V` 对比两个版本，然后把全局默认改回 stable。
3. 确认 `clippy`、`rustfmt` 已安装，在本仓库执行 `cargo fmt --all -- --check` 和 `cargo clippy --workspace`。
4. 在任意临时目录放一个只含 `channel = "stable"` 的 `rust-toolchain.toml`，再 `rustup show`，看“active toolchain”从哪来。
5. 解释：为什么本机同时装着 stable 和 nightly 时，敲 `cargo build` 仍然用 stable？

---

## 延伸阅读

- 安装：[rustup.rs](https://rustup.rs)
- 手册：[The rustup book](https://rust-lang.github.io/rustup/)
- 工具链文件：[Overrides](https://rust-lang.github.io/rustup/overrides.html)
- 下一篇：[Cargo：Rust 的包管理与构建工具](cargo.md)
