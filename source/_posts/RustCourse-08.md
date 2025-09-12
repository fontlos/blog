---
feature: false
title: 初识 Rust(8) | 基建设施
date: 2025-09-12 17:00:00
abstracts: Cargo, WorkSpace, Rustfmt, Clippy, crate.io 以及 CI
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

> 关于 Cargo 和 WorkSpace 的内容在 [第二节](https://fontlos.com/post/RustCourse-02) 已经大致说过了, 这里再重新梳理一下

# Cargo

仅仅使用 Rustc 编译简单程序是没问题的, 不过随着项目的增长, Rustc 就会难以满足需要, 这种时候就需要包管理工具了

## Cargo 简介

代码管理对于编程来说一直是一个重要的问题, 各种不同的语言也都会采用不同的代码管理器, Rust 作为一枚现代语言, 综合了现有语言管理工具的优点, 为我们提供了一个大杀器 --- **Cargo**. Cargo 的强大有目共睹, 包括许多现代新兴的包管理器都或多或少的借鉴了 Cargo, 例如 Python 的 **Uv**

作为 Rust 的代码组织管理工具, Cargo 提供了一系列的工具. 从项目的建立, 构建到测试, 运行直至部署, 为 Rust 项目的管理提供尽可能完整的手段, 极大的减轻了开发复杂度. 同时与 Rust 语言及其编译器 Rustc 本身的各种特性紧密结合

## Cargo 简单使用

在我们安装 Rust 的时候就已经安装好了 Cargo, 直接在命令行运行一下看看

```sh
Rust's package manager

Usage: cargo [+toolchain] [OPTIONS] [COMMAND]
       cargo [+toolchain] [OPTIONS] -Zscript <MANIFEST_RS> [ARGS]...

Options:
  -V, --version                  Print version info and exit
      --list                     List installed commands
      --explain <CODE>           Provide a detailed explanation of a rustc error message
  -v, --verbose...               Use verbose output (-vv very verbose/build.rs output)
  -q, --quiet                    Do not print cargo log messages
      --color <WHEN>             Coloring [possible values: auto, always, never]
  -C <DIRECTORY>                 Change to DIRECTORY before doing anything (nightly-only)
      --locked                   Assert that `Cargo.lock` will remain unchanged
      --offline                  Run without accessing the network
      --frozen                   Equivalent to specifying both --locked and --offline
      --config <KEY=VALUE|PATH>  Override a configuration value
  -Z <FLAG>                      Unstable (nightly-only) flags to Cargo, see 'cargo -Z help' for details
  -h, --help                     Print help

Commands:
    build, b    Compile the current package
    check, c    Analyze the current package and report errors, but don't build object files
    clean       Remove the target directory
    doc, d      Build this package's and its dependencies' documentation
    new         Create a new cargo package
    init        Create a new cargo package in an existing directory
    add         Add dependencies to a manifest file
    remove      Remove dependencies from a manifest file
    run, r      Run a binary or example of the local package
    test, t     Run the tests
    bench       Run the benchmarks
    update      Update dependencies listed in Cargo.lock
    search      Search registry for crates
    publish     Package and upload this package to the registry
    install     Install a Rust binary
    uninstall   Uninstall a Rust binary
    ...         See all commands with --list

See 'cargo help <command>' for more information on a specific command.
```

可以看到有许多的命令, 后面跟着注释写明其功能. 你还可以使用 `cargo [COMMAND] --help` 进一步查看子命令的帮助信息

其中一些比较常用的, 例如 `build` 和 `run` 等还定义了单字母别名以方便使用

还有一些例如 `add`, `remove` 等命令, 原来由社区的 `cargo-edit` 提供, 后来被整合到了官方内部

下面我们简单的使用一下

### 创建项目

新建一个文件夹并在终端中打开, 输入以下命令

```sh
cargo new hello_cargo --bin
```

`--bin` 是一个参数, 代表这是一个 **Bin** crate, 最终将会被编译为二进制可执行文件, Cargo 默认创建 Bin crate, 所以该参数可以不加

### 查看目录结构

`hello_cargo` 文件夹下有一个 `src` 和一个 `Cargo.toml` 文件, src 文件夹下有一个 `main.rs` 文件, 它也在 hello_cargo 目录初始化了一个 `.git` 仓库, 以及一个 `.gitignore` 文件, 在 VSCode 中我们可以很轻易地使用 Git

```
hello_cargo
├── Cargo.toml
└── src
    └── main.rs
```

### 编辑 main.rs 并运行

Cargo 初始化的 main.rs 里会有一些默认内容, 我们把它改为

```rust
// hello_cargo/src/main.rs
fn main() {
    println!("Hello, Cargo!");
}
```

随后在终端输入以下命令

```sh
cargo build # Debug 模式
cargo build --release # Release 模式 --release 代表优化编译
```

也可以直接使用 `cargo run` 命令编译并运行

这两个命令分别会在以下文件夹下生成可执行文件

```
./target/debug/hello_cargo.exe
./target/release/hello_cargo.exe
```

## 其他Cargo命令

- `cargo add <NAME> <OPTIONS>`: 添加依赖
- `cargo remove <NAME>`: 删除依赖
- `cargo update`: 根据 `Cargo.toml` 重新检索并更新各种依赖项的信息, 并写入 `Cargo.lock`
- `cargo clean`: 清理 `target` 文件夹中的所有内容
- `cargo install <NAME> <OPTIONS>`: 安装 `crates.io` 可用于实际的生产的可执行文件
- `cargo check`: 代码检查工具, 不执行真正的 Build, 所以会快一点
- `cargo fmt`: 代码格式化工具
- `cargo clippy`: 检查和优化 Rust 代码的工具

## Cargo 配置

在 Cargo 的安装目录下, 你可以创建一个名为 `config.toml` 的文件用于全局配置 Cargo

例如, 在我们安装完 Rust 后, 通常要配置镜像源以方便我们拉取 Crate 依赖, 以 `rsproxy.cn` 为例, 其官网会建议我们在这个文件内写入以下内容

```toml
# 选择镜像源
[source.crates-io]
replace-with = 'rsproxy-sparse'

# 定义镜像源
[source.rsproxy]
registry = "https://rsproxy.cn/crates.io-index"
# 用稀疏搜索加快索引
[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"

# 切换索引源
[registries.rsproxy]
index = "https://rsproxy.cn/crates.io-index"

# 使用系统上安装的 Git 工具
[net]
git-fetch-with-cli = true
```

还有一个可能用到的配置是

```toml
[http]
check-revoke = false
```

这将禁用 **证书吊销检查**, 可以解决某些网络环境下出现的 SSL 证书验证问题

如果你安装了 [**Sccache**](https://github.com/mozilla/sccache), 一个由 Mozilla 开发的编译缓存管理器, 并配置好了相关环境变量, 你就可以使用以下配置为 Rust 添加一个全局的编译缓存, 这可以显著降低从头编译程序的时间, 相当于一个共享的 `target` 目录

```toml
[build]
rustc-wrapper = "sccache"
```

在列出 Cargo 命令时我们发现一些常见命令是拥有自己的别名的, 你同样可以定义自己的命令别名, 不过这些别名不能混合使用, 例如可以定义一个用于快速进行 `Release` 编译的命令别名

```toml
[alias]
br = "build --release"
rr = "run --release"
```

你还可以通过在项目根目录创建 `.cargo/config.toml` 文件来进行针对当前项目的配置, 例如对于 `rCore` 项目可能会需要以下的配置用于指定编译目标和 `rustflag`, 免去了每次都手动传入这些信息的麻烦

```toml
[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
rustflags = [
    "-Clink-arg=-Tsrc/linker.ld", "-Cforce-frame-pointers=yes"
]
```

## Rust 工具链

默认情况下, 我们安装的 Rust 工具链是 **Stable** 版本, Rust 总共有三个主要版本: **Stable**, **Beta**, **Nightly**. 特性依次增多, 稳定性依次降低

你可以在安装时就选择其他版本的工具链, 也可以在安装后手动安装其他版本的工具链, 这需要使用 **Rustup** 工具, 例如

```sh
rustup toolchain install nightly
```

这将自动安装适合你当前系统的 Nightly 版 Rust 工具链, 如果有必要你还可以指定版本, 比如 `nightly-2024-02-25`

你还可以通过以下命令切换全局默认工具链

```sh
rustup toolchain default nightly
```

如果不想切换全局工具链只需要临时使用, 可以在你的项目根目录添加 `rust-toolchain.toml` 文件并写入以下内容

```toml
[toolchain]
channel = "nightly"
```

对于每个工具链, 你还可以安装其他的编译目标. 首先切换到对应的工具链, 使用以下命令就可以列出工具链列表

```sh
>rustup target list
aarch64-apple-darwin
aarch64-apple-ios
aarch64-apple-ios-macabi
aarch64-apple-ios-sim
aarch64-linux-android (installed)
...
```

你可以通过 `rustup target add` 来安装对应的编译目标, 例如对于 rCore 项目, 你可能需要以下目标

```sh
rustup target add riscv64gc-unknown-none-elf
```

# Crate, Package 与 WorkSpace

我们说过 Cargo 是一个强大的包管理工具, 让我们先来介绍以下几个概念:

1. **Crate**: 通常翻译为 **包**, 包括 **Bin Crate (将被编译为可执行文件)** 和 **Lib Crate (将被编译为库文件)**
2. **Package**: 包的名字已经被 Crate 占用, 这里可以被理解为 **项目**, 一个 Package 包括至多一个 Lib Crate 或任意数量得 Bin Crate 或者两者同时存在. 这一概念容易和 Crate 混淆, 一个方便理解得例子是对于 Bin Crate, 当我们初始化一个 Bin Crate 时看起来就像一个 Package, 这是因为这个 Crate 和 Package 的名字是一样的, 都是项目名. 但别忘了我们还可以在项目中创建其他的不同名字的将被编译为可执行文件的 `.rs` 文件
3. **WorkSpace**: **工作空间**, 用于在大型项目中组织多个 Package

一个标准 Cargo Package 目录结构通常如下, 这些目录拥有特殊含义:

- `Cargo.toml` 和 `Cargo.lock` 文件位于项目根目录
- 源代码位于 `src`
- 默认的 Lib Crate 入口文件是 `src/lib.rs`
- 默认的 Bin Crate入口文件是 `src/main.rs`
- 其他可选的可执行文件位于 `src/bin/*.rs`
    - 这里每一个 `.rs` 文件均对应一个可执行文件, 即一个 Bin Crate
    - 可通过 `cargo run --bin <NAME>` 运行
- 外部测试源代码文件位于 `tests/*.rs`
    - 这里每一个 `.rs` 文件内都可以有多个测试函数和测试模块
    - 可通过 `cargo test` 运行里面的所有测试
    - 可通过 `cargo test <KEYWORD>` 来运行所有测试函数名包含指定关键字的函数
- 示例程序源代码文件位于 `examples/*.rs`
    - 可通过 `cargo run --example <NAME>` 运行
- 基准测试源代码文件位于 `benches`
- `target`: 由 Cargo 自动生成, 包含下载的依赖项和编译缓存等

## Cargo.toml

Cargo 的项目数据描述文件, 存储了项目的所有信息

```toml
[package]
name = "hello_cargo"
description = ""
version = "0.1.0"
authors = ["YourName <Your email>"]
edition = "2024"

[dependencies]
crate1 = "0.3"
# 通配符
crate2 = "0.2.*"
# 跳脱条件, 不会修改从左到右第一个非零版本号, 例如这个会匹配 0.1.0(包括) 到 0.2.0(不包括) 之间的最新版本
crate3 = "^0.1.0"
# 暂且叫它波浪号条件 (Tilde 条件) 吧
# 如果指定 major 版本, minor 版本和 patch 程序版本, 或仅指定 major 版本和 minor 版本, 则仅允许 patch 程序级别更改
# 如果仅指定 major 版本, 则允许进行 minor 和 patch 级别更改
# 例如这个会匹配 1.1.0(包括) 到 1.2.0(不包括) 之间的版本
crate4 = "~1.1.0"
crate5 = { version = "0.5.0"}
crate6 = { git = "https://github.com/user/repo" }
crate7 = { path = "path/to/crate" }
```

**TOML** 是 Rust 的官方配置文件格式, 由像 `[package]` 或 `[dependencies]` 这样的段落组成, 每一个段落又由多个字段组成, 这些段落和字段就描述了项目组织的基本信息

- `package` 段落
    - `name` 字段表明项目的名称. 当发布 crate 时, `crates.io` 将使用此字段标明的名称. 这也是编译时输出的二进制可执行文件的名称
    - `version` 字段是使用 **语义版本控制** 的 crate 版本号
    - `description` 字段是对项目的描述
    - `authors` 字段表明发布 crate 时的作者列表
    - `edition` 字段是对项目的 **Edition (版次)** 声明, 作为一门成熟的语言很重要的一点就是向后兼容. 但有的时候, 面对一些历史遗留问题, 做出一些不兼容的更改也是有需要的, Rust 通过不同的版次引入不兼容的新特性或者删除旧的特性
- `dependencies` 段落可以让你为项目添加依赖, 包括一下几种:
    - 基于 Rust 官方仓库, 通过版本说明来描述
    - 基于项目源代码的 Git 仓库地址, 通过 URL 来描述
    - 基于本地项目的绝对路径或者相对路径

### 条件编译 -- Feature

你可以通过以下方式声明一个 Feature

```toml
[features]
feature1 = []
feature2 = ["feature1"] # 这表示启用 feature2 的同时会启用 feature1
```

然后你就可以使用 `cfg` 标志来让某些代码是可选的

例如

```rust
#[cfg(feature = "feature1")]
println!("Feature 1!");
```

不只是 Feature, `cfg` 还能通过很多其他标志来进行条件编译, 比如 `target` 等

默认情况下没有 Feature 被开启, 你可以通过指定默认 Feature

```toml
[features]
default = ["feature1"]

feature1 = []
feature2 = ["feature1"]
```

此外你也可以通过`--features "feature1 feature2"` 来在编译运行时临时手动开启某些 Feature, 除此之外还有两个命令行 flag

- `--all-features`: 启用当前 Crate 所有 Feature
- `--no-default-features`: 禁用当前 Crate 默认 Feature

Feature 还可以做到可选依赖, 例如

```toml
[dependencies]
crate1 = { version = "1.0", optional = true }

[features]
feature1 = ["crate1"]
```

这样只有当开启 `feature1` 时, `crate1` 才会作为依赖

同样的, 你也可以开启或关闭依赖自身的 Feature

```toml
[dependencies]
crate1 = { version = "1.0", features = ["feature1"] }
crate1 = { version = "1.0", , default-features = false, features = ["feature2"] } # 这将禁用默认 Feature 并单独开启指定 Feature
```

### 编译配置 -- Profile

不同的编译配置可配置编译器的行为, 主要在优化方面, 默认情况下, Rust 有四种编译配置 `dev`, `release`, `test` (继承自 `dev`), `bench` (继承自 `release`)

> 可以发现 `release` 配置对应 `target/release` 文件夹, 但 `dev` 对应的却是 `target/debug` 文件夹, 其实这是个有趣的历史遗留问题

可以在 `Cargo.toml` 中调节这些行为, 默认配置如下

```toml
[profile.dev]
opt-level = 0
debug = true
split-debuginfo = '...'  # 平台相关
debug-assertions = true
overflow-checks = true
lto = false
panic = 'unwind'
incremental = true
codegen-units = 256
rpath = false

[profile.release]
opt-level = 3
debug = false
split-debuginfo = '...'
debug-assertions = false
overflow-checks = false
lto = false
panic = 'unwind'
incremental = false
codegen-units = 16
rpath = false
```
我们来看一些比较常用的

- `opt-level`: 控制优化级别, 通常越高级的优化会导致更慢的编译速度. 但优化级别与性能的关系可能会在你的意料之外, 所以需要自己权衡
  - `0`: 无优化
  - `1`: 基本优化
  - `2`: 一些优化
  - `3`: 全部优化
  - `"s"`: 优化二进制文件的大小
  - `"z"`: 优化二进制文件大小, 但也会关闭循环向量化

- `debug`: 控制最终二进制文件输出的 debug 信息量
  - `0` 或 `false`: 不输出任何 debug 信息
  - `1`: 行信息
  - `2`: 完整的 debug 信息

- `debug-assertions`: 开启或关闭其中一个条件编译选项 `#[cfg(debug_assertions)]`, 选项是布尔值

- `lto`: 控制 LLVM 的链接时优化 (Link Time Optimizations ). 大幅增加链接时间, 但可以生成更加优化的代码
  - `false`: 只对本地包进行 `"thin"` LTO 优化，若代码生成单元数为 `1` 或者 `opt-level` 为 `0`, 则不会进行任何 LTO 优化
  - `"thin"`: 相比 `"fat"` 来说，它仅牺牲了一点性能，但是换来了链接时间的可观减少
  - `true` 或 `"fat"`: 对依赖图中的所有包进行 `"fat"` LTO 优化
  - `"off"`: 禁用 LTO

- `panic`: 控制 panic 策略
  - `"unwind"`: 遇到 panic 后对栈进行展开
  - `"abort"`: 遇到 panic 后直接停止程序, 剩余清理工作交给操作系统或不处理

- `incremental`: 用于开启或关闭增量编译, 选项是布尔值

- `codegen-units`: 指定一个包会被分割为多少个代码生成单元. 分割较多, 可增加并行编译速度, 但会一定程度降低性能
  - 对于增量编译默认值是 256, 非增量编译是 16

### WorkSpace

而对于 WorkSpace, 它可以组织多个 Package, `Cargo.toml` 的内容可能是这样的

```toml
[workspace]
members = [
    "path/to/package1",
    "path/to/package2",
    "lib/*",
    "..."
]
exclude = [
    "lib/package3"
]
```

你可以在 WorkSpace 定义一些 Package 信息, 例如

```toml
[package]
version = "0.1.0"
```

然后你就可以在 WorkSpace 管理的 Package 下面的 `Cargo.toml` 使用它, 例如

```toml
[package]
version = { workspace = true }
```

这同样适用于统一管理依赖

```toml
# ./Cargo.toml
[workspace.dependencies]
crate = "1.0"

# ./lib/Cargo.toml
[dependencies]
crate = { workspace = true }
```

看起来非常简单, WorkSpace 下所有 Package 共用同一个 `target` 文件夹, 总体来看可以占用更少的磁盘空间, 加快编译速度

## Cargo.lock

该文件不需要直接修改, 是 Cargo 工具根据 `Cargo.toml` 生成的项目依赖详细清单文件, 如果不手动修改 `Cargo.toml` 或执行 `cargo update` 会锁定依赖项的版本, 即使包含通配符等

但是如果有时你在更新 Crate 版本后遇到了奇怪的编译问题, 可以尝试删除这个文件让 Cargo 重新生成

## 定义集成测试用例

Cargo 另一个重要的功能, 即将软件开发过程中必要且非常重要的测试环节进行集成, 并通过代码属性声明或者 `Cargo.toml` 文件描述来对测试进行管理

单元测试通常和代码耦合在一起, 使用 `#[test]` 属性宏来标记单个测试函数

集成测试也可以和代码耦合在一起, 可以通过 `#[cfg(test)]` 属性宏标记一个模块来包裹多个测试函数, 例如

```rust
#[test]
fn test1() {}

#[cfg(test)]
mod tests {
    #[test]
    fn test2_foo() {}

    #[ignore]
    #[test]
    fn test3() {}

    #[should_panic]
    #[test]
    fn test4_foo() { panic!() }
}
```

独立在外的集成测试可以通过 `Cargo.toml` 文件中的 `[[test]]` 段落进行描述

简单的示例:

```toml
[[test]]
name = "test1"
path = "tests/test1.rs"

[[test]]
name = "test2"
path = "tests/test2.rs"
```

上述例子中, `name` 字段定义了集成测试的名称, `path` 字段定义了集成测试文件相对于 `Cargo.toml` 的路径.

看看, 定义集成测试就是如此简单, 但根据我们之前提到的一点, 有以下注意事项:

1. 如果没有在 `Cargo.toml` 里定义集成测试的入口, 那么 `tests` 目录 (不包括子目录) 下的每个 `.rs` 文件都被当作集成测试入口
2. 如果在 `Cargo.toml` 里定义了集成测试入口, 那么定义的那些 `.rs` 文件就是入口, 不再默认指定任何集成测试入口

通过 `cargo test` 将运行所有未被忽略的测试, 指定关键词后会运行所有包含测试的文件并过滤出那些函数名包含指定关键词的测试, 例如在上面的例子中运行 `cargo test foo`

## 定义示例和可执行文件

Example 用例的描述以及 Bin 可执行文件的描述也是 Cargo 的常用功能

和测试不同, 这两个不能定义和主要代码耦合在一起, 必须要独立的文件, 而且必须包含 `main` 函数

```toml
[[example]]
name = "examlpe1"
path = "examples/examlpe1.rs"

[[bin]]
name = "bin1"
path = "bin/bin1.rs"
```

同样的

1. 如果没有在 `Cargo.toml` 里定义这些入口, 那么 `examples`(或 `src/bin`) 目录 (不包括子目录) 下的每个 `.rs` 文件都被当作入口
2. 如果在 `Cargo.toml` 里定义了入口, 那么定义的那些 `.rs` 文件就是入口, 不再默认指定任何集成测试入口

这些 Examples 和 Bins, 需要通过文件名 `cargo run --example <NAME>` 或者 `cargo run --bin <NAME>` 来运行

# Rustfmt 与 Clippy

# crate.io 与 CI
