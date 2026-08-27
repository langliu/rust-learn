# rust-learn

Rust 学习仓库：用 Cargo workspace 放练习 crate，用 `docs/` 记笔记。

官网（中文）：[https://rust-lang.org/zh-CN/](https://rust-lang.org/zh-CN/)

## 仓库结构

```
rust-learn/
├── Cargo.toml          # workspace 清单
├── crates/             # 练习用 crate（每个子目录一个 package）
└── docs/
    ├── ownership.md    # 所有权笔记
    └── 生态链/
        ├── rustup.md   # rustup 介绍
        └── cargo.md    # Cargo 介绍
```

根 `Cargo.toml` 只声明工作区，真正的包在 `crates/`：

```toml
[workspace]
members = ["crates/*"]
resolver = "3"
```

## 环境

需要 [rustup](https://rustup.rs)（会带上 `rustc` 和 `cargo`）：

```bash
rustc --version
cargo --version
```

推荐编辑器：[Zed](https://zed.dev)（内置 rust-analyzer，打开本仓库即可跳转、补全、诊断）。

## 常用命令

在仓库根目录执行：

```bash
cargo check --workspace          # 最快的编译检查
cargo test --workspace           # 跑全部成员测试
cargo run -p <crate-name>        # 运行某个二进制 crate
```

新建练习 crate：

```bash
cargo new crates/hello --name hello
cargo new crates/greet --lib --name greet
```

## 笔记

| 文档 | 内容 |
|------|------|
| [docs/ownership.md](docs/ownership.md) | 所有权、移动、借用、切片、生命周期初见 |
| [docs/生态链/rustup.md](docs/生态链/rustup.md) | rustup：工具链、组件、target、覆盖 |
| [docs/生态链/cargo.md](docs/生态链/cargo.md) | Cargo：包、依赖、workspace、常用命令 |

## 约定

- 练习代码放 `crates/`，不要在仓库根再放一份 `src/`
- `Cargo.lock` 留在 workspace 根，成员目录不各自生成 lock
- `target/` 已 gitignore，不要提交构建产物
