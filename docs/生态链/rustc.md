# rustc：Rust 编译器

> rustc 是 Rust 的编译器本体：读 crate 源码，做类型检查和所有权检查，再交给 LLVM 生成机器码。
> 日常开发几乎都通过 Cargo 间接调用它；直接敲 `rustc` 主要是为了理解它在干什么。

---

## 目录

1. [rustc 是什么](#1-rustc-是什么)
2. [和 rustup、Cargo 的关系](#2-和-rustupcargo-的关系)
3. [编译单个文件](#3-编译单个文件)
4. [crate 才是编译单元](#4-crate-才是编译单元)
5. [crate 类型](#5-crate-类型)
6. [常用参数](#6-常用参数)
7. [错误码与 --explain](#7-错误码与---explain)
8. [从 Cargo 把参数传给 rustc](#8-从-cargo-把参数传给-rustc)
9. [rustc 实际做了哪些事](#9-rustc-实际做了哪些事)
10. [常见误区](#10-常见误区)
11. [练习题](#11-练习题)

---

## 1. rustc 是什么

`rustc` 把一份 **crate** 编成库或可执行文件。它不是包管理器，也不管 `Cargo.toml` 里的依赖解析——那些是 Cargo 的事。

可以类比：

| 语言 | 编译器 | 项目工具 |
|------|--------|----------|
| C | `clang` / `gcc` | Make / CMake |
| Java | `javac` | Maven / Gradle |
| Rust | **rustc** | **Cargo** |

一句话：

> **rustc 编译；Cargo 组织项目并调用 rustc；rustup 决定用哪一套 rustc。**

所有权、借用、生命周期不通过，就是 **rustc 在编译期拒绝**，不会拖到运行时。相关概念见[所有权笔记](../ownership.md)。

---

## 2. 和 rustup、Cargo 的关系

你敲的 `rustc` 通常是 rustup 放在 `~/.cargo/bin/rustc` 的**代理**，真正的编译器在 `~/.rustup/toolchains/` 里。查路径：

```bash
rustc --version --verbose
rustup which rustc
```

Cargo 构建时会拼出一长串 rustc 命令：选 crate 根、接 `--extern`、设优化级别、写出到 `target/`。看它到底传了什么：

```bash
cargo build -v
cargo rustc -p demos -- --print cfg
```

日常流程：

```text
rustup  →  选出 rustc / cargo
cargo   →  解析依赖、安排编译顺序
rustc   →  编每一个 crate
```

工具链管理见 [rustup.md](rustup.md)，项目管理见 [cargo.md](cargo.md)。

---

## 3. 编译单个文件

不经过 Cargo 也能编：

```bash
# hello.rs
fn main() {
    println!("hello, rustc");
}
```

```bash
rustc hello.rs
./hello
```

直接调用 `rustc` 时，标准库通常由当前工具链的 sysroot 提供；查看它的位置：

```bash
rustc --print sysroot
```

指定输出名、优化、带调试信息：

```bash
rustc hello.rs -o hello
rustc -O hello.rs          # 优化，接近 cargo build --release
rustc -g hello.rs          # 调试信息，接近 cargo build
```

这只适合**没有第三方依赖**的小例子。一加 `rand` 这类 crates.io 包，就要自己下依赖、编依赖、再 `--extern` 链进来——这正是 Cargo 存在的理由。

---

## 4. crate 才是编译单元

rustc **一次编译一个 crate**，入口是 crate 根：

- 二进制：`src/main.rs` 或 `src/bin/foo.rs`
- 库：`src/lib.rs`

`mod foo;` 会让编译器去读 `foo.rs` 或 `foo/mod.rs`，这些文件属于**同一个 crate**，不会各自再调一次 rustc。

```text
crates/demos/
└── src/main.rs     ← cargo build -p demos 时，rustc 的入口
```

多个 crate（例如本仓库 workspace 里以后再加库）会 **分别** 调用 rustc，再通过 `--extern` 把编好的 `rlib` 链给下游。

语言版本用 **edition**，不是 rustc 的版本号：

```bash
rustc --edition 2024 hello.rs
```

`Cargo.toml` 里的 `edition = "2024"` 最终也会变成传给 rustc 的这个参数。edition 换了，rustup 不会自动升级编译器。

---

## 5. crate 类型

`--crate-type` 决定产物形态：

| 类型 | 产物 | 用途 |
|------|------|------|
| `bin` | 可执行文件 | 默认，有 `main` |
| `lib` / `rlib` | `.rlib` | Rust 之间链接（Cargo 编库的默认） |
| `dylib` | 动态库 | Rust 动态链接，少用 |
| `cdylib` | C 兼容动态库 | FFI、wasm 常见 |
| `staticlib` | C 兼容静态库 | 给 C/C++ 链 |
| `proc-macro` | 过程宏 |

```bash
rustc --crate-type lib lib.rs          # 通常得到 lib<name>.rlib
rustc --crate-type rlib lib.rs         # 明确生成 Rust 专用 rlib
rustc --crate-type cdylib lib.rs
```

`rlib` 主要供 Rust crate 之间链接；`cdylib`、`staticlib` 面向 C 等其他语言，除了产物格式，还要注意导出符号和 ABI。Cargo 里对应 `[lib] crate-type = ["cdylib"]` 等，不必手写 rustc。

---

## 6. 常用参数

完整列表：`rustc --help`。学习阶段这几个够用：

| 参数 | 作用 |
|------|------|
| `-o <文件>` | 输出路径 |
| `--crate-name <名>` | crate 名 |
| `--crate-type <类型>` | 产物类型 |
| `--edition <年>` | 2015 / 2018 / 2021 / 2024 |
| `-L <目录>` | 额外库搜索路径 |
| `--extern name=path` | 链接已编译的依赖 |
| `--target <triple>` | 交叉编译目标 |
| `--emit asm,llvm-ir,mir,obj,link` | 输出中间产物或最终链接结果 |
| `-C opt-level=3` | 优化级别（0/1/2/3/s/z） |
| `-C debuginfo=2` | 调试信息 |
| `--explain <E0xxx>` | 解释错误码 |
| `--print cfg` | 打印当前 cfg |
| `--print sysroot` | 打印工具链 sysroot 路径 |
| `--print target-list` | 列出支持的目标平台 |
| `--cfg <键>[="值"]` | 手动启用条件编译配置 |
| `--error-format=json` | 以 JSON 输出诊断信息，便于 IDE/CI 处理 |
| `-W` / `-A` / `-D` | 设置 lint 等级：warn / allow / deny |

看 rustc 认为当前平台开了哪些 `cfg`：

```bash
rustc --print cfg
```

交叉编译目标要先用 rustup 装好，再交给 rustc（或 `cargo build --target`）：

```bash
rustup target add wasm32-unknown-unknown
rustc --target wasm32-unknown-unknown --crate-type cdylib lib.rs
```

> `rustup target add` 只提供目标平台的 Rust 标准库。非 wasm 目标通常还需要对应的 linker、C/C++ 交叉编译器或系统 SDK；安装 target 本身不等于交叉编译环境已经准备完成。

---

## 7. 错误码与 --explain

编译失败时，rustc 会给出 `error[E0xxx]`。把编号丢回去能看到官方说明和示例：

```bash
rustc --explain E0382
```

`E0382` 就是“值已被 move，不能再用”——所有权笔记里最常见的那类错误。

建议：先读终端里 rustc 标出的**那一行代码**，再 `--explain`，不要一上来搜博客。

---

## 8. 从 Cargo 把参数传给 rustc

项目里不要绕过 Cargo 手搓 `--extern`。需要给 rustc 加旗标时：

```bash
# 只对某个包
cargo rustc -p demos -- -C target-cpu=native

# 整个构建过程（含依赖）
RUSTFLAGS="-C target-cpu=native" cargo build -p demos --release
```

`--` 后面才是 rustc 的参数，前面是 Cargo 的。这和 `cargo run -- --help` 是同一套规则。

`-C target-cpu=native` 会针对当前机器优化，生成的程序可能无法在其他机器上运行；适合本机性能测试，不适合作为通用发布产物。

`cargo build -v` 能看到完整 rustc 命令行，是理解 Cargo 如何驱动编译器的最快办法。

---

## 9. rustc 实际做了哪些事

简化流水线：

1. **解析**：源码 → AST
2. **展开**：宏、`#[derive]`
3. **名称解析与类型检查**
4. **借用检查**（所有权、生命周期）
5. **MIR**：中间表示，继续优化
6. **LLVM IR** → 目标文件（机器码）
7. **链接**：二进制或动态库还需要 linker 组合目标文件、依赖库和系统库

库 crate 可能在生成 `rlib` 等库产物后结束；二进制 crate 通常还要经过最后的链接步骤。日常感知最强的是第 3、4 步：报错几乎都来自这里。`--emit=mir` 或 `--emit=llvm-ir` 可以看见中间结果，一般学习阶段不必碰。

Cargo 的 debug / release 大致对应：

| Cargo | rustc 侧（概念上） |
|-------|-------------------|
| `cargo build` | 低优化 + 调试信息，编译快 |
| `cargo build --release` | `-O` / 高 `opt-level`，运行快、编译慢 |
| `cargo check` | 只做到类型/借用检查，不生成最终可执行文件 |

`cargo check` 快，就是因为它让 rustc **少做后面的代码生成和链接**。但 build script、过程宏及其依赖仍可能被编译和执行，因此它并不是完全不产生构建产物。

---

## 10. 常见误区

1. **用 rustc 编整个 Cargo 项目**  
   有 `Cargo.toml` 就用 `cargo build`。自己调 rustc 不会自动拉依赖、也不会读 workspace。

2. **把 `rustc hello.rs` 和 `cargo run` 当成两套语言**  
   同一套 rustc。差别只是谁来拼参数、谁来管 `target/`。

3. **edition 和编译器版本搞混**  
   edition 是语言门面；能不能用 2024 edition，取决于当前 rustc 够不够新。见 [rustup.md](rustup.md)。

4. **PATH 里不是 rustup 那份 rustc**  
   `which rustc` 应指向 `~/.cargo/bin/rustc`。系统包管理器装的旧 `rustc` 会让 Cargo 和编译器各说各话。

5. **对着 `cargo rustc --help` 找 `-C`**  
   `-C` 是 rustc 的 codegen 旗标，必须写在 `--` 后面。

6. **以为 `--target` 自动准备好交叉编译**

   `--target` 只选择目标平台；标准库、linker、系统库和 SDK 是否准备好，仍要分别确认。

---

## 11. 练习题

1. 写一个无依赖的 `hello.rs`，用 `rustc` 编出来并运行；再用 `rustc -O` 编一次，对比两个二进制大小。
2. 运行 `rustc --version --verbose` 和 `rustup which rustc`，说明“你敲的 rustc”和“真正的编译器”各在哪。
3. 对一段会 move 的代码触发编译错误，用 `rustc --explain` 查对应错误码（常见 `E0382`）。
4. 在本仓库执行 `cargo build -p demos -v`，从输出里找出 rustc 命令，标出 `--crate-name`、`--edition`、输出路径。
5. 解释：为什么 `cargo check -p demos` 比 `cargo build -p demos` 快，却不能当可执行文件来跑？
6. 运行 `rustc --print sysroot` 和 `rustc --print target-list`，说明它们分别回答什么问题。

---

## 延伸阅读

- 手册：[The rustc book](https://doc.rust-lang.org/rustc/)
- 错误码索引：[Compiler error index](https://doc.rust-lang.org/error_codes/error-index.html)
- 上一篇：[rustup：Rust 工具链管理器](rustup.md)
- 下一篇：[Cargo：Rust 的包管理与构建工具](cargo.md)
