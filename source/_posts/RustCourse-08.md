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

# Cargo

我们都知道 Rust 的编译器是 Rustc, 对于单文件简单程序使用 Rustc 编译是没问题的, 不过随着项目的增长, Rustc 就会难以满足需要

代码管理对于编程来说一直是一个重要的问题, Rust 作为一枚现代语言, 综合了现有语言管理工具的优点, 为我们提供了一个强大的代码组织管理工具 --- **Cargo**. Cargo 提供了一系列的工具. 从项目的建立, 构建, 测试, 运行直至部署, 为 Rust 项目的管理提供尽可能完整的手段, 极大的减轻了开发复杂度. 同时与 Rust 语言及其编译器 Rustc 本身的各种特性紧密结合

Cargo 的强大有目共睹, 包括许多现代新兴的包管理器都或多或少的借鉴了 Cargo, 例如 Python 的 **Uv**

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

下面我们简单的使用一下, 在终端中打开一个文件夹, 输入以下命令

```sh
cargo new hello_cargo --bin
```

`--bin` 是一个参数, 代表这是一个 **Bin** crate, 最终将会被编译为二进制可执行文件, Cargo 默认创建 Bin crate, 所以该参数可以不加

查看目录结构. `hello_cargo` 文件夹下有一个 `src` 和一个 `Cargo.toml` 文件, src 文件夹下有一个 `main.rs` 文件, 它也在 hello_cargo 目录初始化了一个 `.git` 仓库, 以及一个 `.gitignore` 文件

```
hello_cargo
├── Cargo.toml
└── src
    └── main.rs
```

Cargo 初始化的 main.rs 里会有一些默认内容, 我们把它改为

```rust
// hello_cargo/src/main.rs
fn main() {
    println!("Hello, Cargo!");
}
```

随后在终端输入以下命令进行编译

```sh
cargo build # Debug 模式
cargo build --release # Release 模式 --release 代表优化编译
```

这两个命令分别会在以下文件夹下生成可执行文件

```
./target/debug/hello_cargo.exe
./target/release/hello_cargo.exe
```

也可以直接使用 `cargo run` 命令编译并运行, 当然你同样可以加上 `--release` flag, 可以在终端看到

```sh
   Compiling hello_cargo v0.1.0 (path\to\hello_cargo)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.02s
     Running `target\debug\hello_cargo.exe`
Hello, Cargo!
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

在 Cargo 的安装目录下, 你可以创建一个名为 `config` 或 `config.toml` 的文件用于全局配置 Cargo, 前者的优先级更高

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

这将禁用 **证书吊销检查**, 可以解决某些网络环境下出现的 SSL 证书验证问题, 但关闭验证也可能会导致不安全

如果你安装了 [**Sccache**](https://github.com/mozilla/sccache) (Shared Compilation Cache), 一个由 Mozilla 开发的共享编译缓存管理器, 并配置好了相关环境变量, 你就可以使用以下配置为 Rust 添加一个全局的编译缓存, 在缓存命中的情况下这可以显著降低从头编译程序的时间, 相当于一个共享的 `target` 目录

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

如果想更临时一点, 你还可以使用命令行参数, 例如 `cargo +nightly build`

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
2. **Package**: 包的名字已经被 Crate 占用, 这里可以被理解为 **项目**, 一个 Package 包括至多一个 Lib Crate 或至少一个 Bin Crate 或者两者同时存在. 这一概念容易和 Crate 混淆, 举个例子, 对于 Bin Crate, 当我们初始化它时看起来就像一个 Package, 这是因为这个 Crate 和 Package 的名字是一样的, 都是项目名, 但别忘了我们还可以在这个 Package 中创建更多其他 Bin Crate 或者一个 Lib Crate, 例如下面的结构
   ```
   hello_cargo
   ├── Cargo.toml
   └── src
       ├── lib.rs
       └── main.rs
   ```
3. **WorkSpace**: **工作空间**, 用于在大型项目中组织多个 Package

一个标准 Cargo Package 目录结构通常如下, 这些目录拥有特殊含义:

首先是基本结构

- `Cargo.toml`: 项目的数据描述文件
- `src`: 源代码目录
  - `src/lib.rs`: 默认的 Lib Crate 入口文件
  - `src/main.rs`: 默认的 Bin Crate 入口文件
  - `src/bin/*.rs`: 其他可选的 Bin Crate 入口文件
    - 这里每一个 `.rs` 文件均对应一个可执行文件
    - 可通过 `cargo run --bin <NAME>` 运行

然后是可选结构

- `build.rs`: 构建脚本
- `tests/*.rs`: 外部测试源代码文件
  - 这里每一个 `.rs` 文件内都可以有多个测试函数和测试模块
  - 可通过 `cargo test` 运行里面的所有测试
  - 可通过 `cargo test <KEYWORD>` 来运行所有函数名包含指定关键字的测试
- `examples/*.rs`: 示例程序源代码文件
  - 这里每一个 `.rs` 文件实际上也对应一个可执行文件
  - 可通过 `cargo run --example <NAME>` 运行
- `benches/*.rs`: 基准测试源代码文件

最后是编译器自动生成的结构

- `Cargo.lock`: 项目依赖详细清单文件
- `target`: 包含下载的依赖项和编译缓存等

下面我们详细介绍一下这些

## Cargo.toml 与 Cargo.lock

`Cargo.toml` 是项目的数据描述文件, 存储了项目的所有信息

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

**TOML** 是 Rust 的官方配置文件格式, 我们在之前介绍 Cargo 配置时就已经见过它了, 由像 `[package]` 或 `[dependencies]` 这样的段落组成, 每一个段落又由多个字段组成, 这些段落和字段就描述了项目组织的基本信息

- `package` 段落
    - `name` 字段表明项目的名称. 当发布 crate 时, `crates.io` 将使用此字段标明的名称. 这也是编译时输出的二进制可执行文件的名称
    - `version` 字段是使用 **语义版本控制** 的 crate 版本号
    - `description` 字段是对项目的描述
    - `authors` 字段表明发布 crate 时的作者列表
    - `edition` 字段是对项目的 **Edition (版次)** 声明, 作为一门成熟的语言很重要的一点就是向后兼容. 但有的时候, 面对一些历史遗留问题, 做出一些不兼容的更改也是有需要的, Rust 通过不同的版次引入不兼容的新特性或者删除旧的特性, 截止到目前, Rust 有 `2015`, `2018`, `2021`, `2024` 四个版次
- `dependencies` 段落可以让你为项目添加依赖, 包括一下几种:
    - 基于 Rust 官方仓库, 通过版本说明来描述
    - 基于项目源代码的 Git 仓库地址, 通过 URL 来描述
    - 基于本地项目的绝对路径或者相对路径
    - 来自当前工作空间

`Cargo.lock` 文件不需要直接修改, 是 Cargo 工具根据 `Cargo.toml` 生成的项目依赖详细清单文件, 如果不手动修改 `Cargo.toml` 或执行 `cargo update` 会锁定依赖项的版本, 即使包含通配符等

但是如果有时你在更新 Crate 版本后遇到了奇怪的编译问题, 可以尝试删除这个文件让 Cargo 重新生成

## 条件编译 -- Feature

你可以通过以下方式声明一个 Feature

```toml
[features]
feature1 = []
feature2 = ["feature1"] # 这表示启用 feature2 的同时会启用 feature1
```

然后你就可以使用 `cfg` 标志来让某些代码是可选的, 例如

```rust
#[cfg(feature = "feature1")]
println!("Feature 1!");
```

不只是 Feature, `cfg` 还能通过很多其他标志来进行条件编译, 比如 `target` 等. 你还可以通过 `not()`, `any()`, `all()` 等来组合条件

默认情况下没有 Feature 被开启, 你可以通过指定默认 Feature

```toml
[features]
default = ["feature1"]

feature1 = []
feature2 = ["feature1"]
```

此外你也可以通过`cargo run --features "feature1 feature2"` 来在编译运行时临时手动开启某些 Feature, 除此之外还有两个命令行 flag

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
crate2 = { version = "1.0", , default-features = false, features = ["feature2"] } # 这将禁用默认 Feature 并单独开启指定 Feature
```

## 编译配置 -- Profile

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

- `codegen-units`: 指定一个包会被分割为多少个代码生成单元. 分割较多, 可增加并行编译速度, 但会一定程度降低性能.

## 工作空间 -- WorkSpace

WorkSpace 可以组织多个 Package, 你同样需要在根目录创建一个 `Cargo.toml` 文件, 例如

```toml
[workspace]
resolver = "2" # 建议启用第二版解析器, 这在 2021 版次引入, 在 2024 版次已经成为默认值可以无需声明
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

其中 `members` 里面就是当前 WorkSpace 所包含并管理的 Package. 你可以写明相对路径, 也可以使用通配符, 当你使用通配符时也可以使用 `exclude` 来排除掉一些 Package

你可以在正常 Cargo 命令后面新增一个 `--package [PACKAGE]` flag 来操作指定 Package

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

## 定义集成测试用例

Cargo 另一个重要的功能, 即对软件开发过程中非常重要的测试环节进行集成, 并通过代码属性声明或者 `Cargo.toml` 文件描述来对测试进行管理

单元测试通常和代码耦合在一起, 使用 `#[test]` 来标记单个测试函数

集成测试也可以和代码耦合在一起, 可以通过 `#[cfg(test)]` 标记一个模块来包裹多个测试函数, 例如

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

此外, 测试代码以及下面介绍的示例代码也可以依赖外部 Crate, 不过需要定义在 `[dev-dependencies]` 字段下

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

## 构建脚本

根目录下的 `build.rs` 是一个特殊的文件, 它相当于在编译之前被执行的 Rust 代码. 可以对你接下来真正要编译的项目代码进行预处理操作

作为一个可执行文件那么肯定需要包含 `main` 函数. 同样的, 它也可以依赖外部 Crate, 需要定义在 `build-dependencies` 字段下

一个常见的使用场景是给可执行文件添加图标, 对于 Windows 平台

```toml
[target.'cfg(windows)'.build-dependencies]
embed-resource = "3.0.2"
```

```rust
#[cfg(target_os = "windows")]
fn main() {
    embed_resource::compile("./static/icon/icon.rc", embed_resource::NONE).manifest_optional().unwrap();
}

#[cfg(not(target_os = "windows"))]
fn main() {}
```

# Rustfmt

**Rustfmt** 是 Rust 官方提供的自动 **代码格式化** 工具. 它的核心目标是基于一套明确的, 社区共识的风格指南来格式化 Rust 代码，从而增强代码可读性, 促进代码一致性

在你安装 Rust 时 Rustfmt 就一并安装了, 我们可以像使用 Rustc 那样简单的使用它来格式化单个文件. 那么当然, 在日常开发中我们很少会单独使用它, 而是通过我们的 Cargo 来统一格式化整个项目的代码, 只需要简单的一行 `cargo fmt`, 这将会就地格式化你所有的代码. 如果你只想检查格式而非修改, 可以使用 `cargo fmt -- --check`

Rustfmt 已经有了一套非常优秀的默认风格. 当然如果你需要, 你同样可以在项目根目录创建一个 `rustfmt.toml` 来配置它, 这里我们展示一些常见选项, 你也可以使用 `rustfmt --help=config` 来查看所有可配置项

```toml
# 缩进风格: 空格或制表符 (tab)
hard_tabs = false
# 每个缩进级别的空格数
tab_spaces = 4

# 最大行宽: 超过此宽度的代码会被换行
max_width = 100

# 将多个 `use` 语句合并为一个（例如：`use std::{io, fs};`）
merge_imports = true

# 匹配表达式（match）的样式："SameLine" 或 "Default"
match_arm_blocks = false

# 错误后尽最大努力格式化, 而不是中止
error_on_line_overflow = false
error_on_unformatted = false

# 控制 where 子句的位置
where_single_line = false

# 对导入进行重新排序和分组
reorder_imports = true
```

有些时候你可能需要 Rustfmt 忽略掉部分代码块, 这时只需要使用 `#[rustfmt::skip]` 进行标记, 例如一个你精心排列整齐的矩阵

```rust
#[rustfmt::skip]
const PC1: [usize; 56] = [
    57, 49, 41, 33, 25, 17, 09,
    01, 58, 50, 42, 34, 26, 18,
    10, 02, 59, 51, 43, 35, 27,
    19, 11, 03, 60, 52, 44, 36,
    63, 55, 47, 39, 31, 23, 15,
    07, 62, 54, 46, 38, 30, 22,
    14, 06, 61, 53, 45, 37, 29,
    21, 13, 05, 28, 20, 12, 04,
];
```

# Clippy

Clippy 是 Rust 编译器的一个官方插件, 它是一个强大的 Linter 工具, 比 `cargo check` 和 `cargo fmt` 更进一步, 其核心作用包括

- 捕捉常见错误和反模式: 识别那些虽然能通过编译, 但可能存在潜在问题, 不够优雅或效率低下的代码
- 推行最佳实践: 鼓励使用更符合 Rust 语言习惯, 更安全, 更高效的写法
- 提升代码质量: 充当一位审查员, 帮助你写出更好的 Rust 代码
- 辅助学习 Rust: 包括 Rust 编译器也是, 通过其警告和建议, 新手可以快速了解 Rust 的惯用写法和各种陷阱

在你安装 Rust 时 Clippy 就一并安装了, 默认包含了很多的 Lint, 你可以使用 `cargo clippy` 来很方便的调用它

例如对于我们上面的矩阵, 当你调用它时就会提示

```sh
warning: this is a decimal constant
 --> src\main.rs:3:37
  |
3 |             57, 49, 41, 33, 25, 17, 09,
  |                                     ^^
  |
  = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#zero_prefixed_literal
  = note: `#[warn(clippy::zero_prefixed_literal)]` on by default
help: if you mean to use a decimal constant, remove the `0` to avoid confusion
  |
3 -             57, 49, 41, 33, 25, 17, 09,
3 +             57, 49, 41, 33, 25, 17, 9,
```

它的意思是说在类 C 语言中, 前导零表示这个数是八进制, 可能会对有这方面经验的人造成误解, 提醒你去掉前导零. 不过 Rust 本身没有这个约定. 所以对于我们辛辛苦苦排列整齐的矩阵, 你可以使用 `#[allow(clippy::zero_prefixed_literal)]` 来关掉这个警告

> 原文的描述也很有趣 "In some languages (including the infamous C language and most of its family), this marks an octal constant. In Rust however, this is a decimal constant. This could be confusing for both the writer and a reader of the constant."

Clippy 还有很多更强大的功能, 例如对于下面这个函数

```rust
fn sub_bytes(&self, state: &mut [u8; 16]) {
    for i in 0..16 {
        state[i] = self.sub_byte(state[i]);
    }
}
```

这将会提示你

```sh
warning: the loop variable `i` is only used to index `state`
   --> src\crypto\aes.rs:109:18
    |
109 |         for i in 0..16 {
    |                  ^^^^^
    |
    = help: for further information visit https://rust-lang.github.io/rust-clippy/master/index.html#needless_range_loop
    = note: `#[warn(clippy::needless_range_loop)]` on by default
help: consider using an iterator
    |
109 -         for i in 0..16 {
109 +         for <item> in state.iter_mut().take(16) {
    |
```

它的意思是让你使用迭代器以更安全的操作可变引用, 而无需这个无意义的索引变量, 可简单修改为

```rust
fn sub_bytes(&self, state: &mut [u8; 16]) {
    for s in state.iter_mut() {
        *s = self.sub_byte(*s);
    }
}
```

对于真正的开发环境, 定期运行 Lint 检查是很有必要的, 还可以使用 `cargo clippy -- -D warnings` 让检查结果不再是警告而是错误, 来强制团队解决这些问题

此外, 你可以使用 `cargo fix --clippy` 来尝试自动修复这些警告

关于 Lint 规则, 由于实在太多, 感兴趣可以在 [官方文档](https://rust-lang.github.io/rust-clippy/master/index.html) 自行查看

# crates.io

因为 `crates.io` 的访问速度在国内比较慢, 所以我们学 Rust 的第一步总是先配置镜像源. 但别忘了, 它才是 Rust 官方的 Crate 注册管理中心. 注意别丢了 URL 中的 "s"

多数公开的 Crate 都托管在上面, 日常你就可以在上面寻找自己所需的 Crate.

你自己也可以发布自己的 Crate, 只需要在上面注册一个账号, 获取登录 Token, 使用 `cargo login [TOKEN]` 完成登录, 就可以使用 `cargo publish` 发布啦. 当然建议先使用 `cargo publish --dry-run` 检查一下代码库有没有错误或警告, 这也会把代码库打包到一个 `target/package/*.crate` 文件. 一般要求这个文件不能超过 10MB, 你可以在 Cargo.toml 中配置声明包含或排除哪些文件

```toml
[package]
exclude = [
    "videos/*",
]

include = [
    "**/*.rs",
    "Cargo.toml",
]
```

Crate 一经发布无法修改无法删除, 只能通过新的版本号覆盖上去. 所以发布前需要慎重考虑一下. 并且 Crate 的名字遵循先到先得的原则.

同时如果发布的是 Lib Crate, 记得做好详细的文档注释, 它会被自动渲染成漂亮的文档托管在 `docs.rs` 上

你也可以在本地提前渲染文档进行查看

1. `rustdoc *.rs`: 用于单个文件
2. `cargo doc`: 用于整个项目, 最好加上 `--no-depth` 参数, 否则将给你所有的依赖项也生成文档

# CI 持续集成

**CI (Continuous Integration)** 是软件开发中的重要环节. 简单来说, 这是一个自动化服务, 会定期对项目代码进行拉取, 执行自动化检查, 构建, 发布等任务. 它有一个统一的运行环境, 无需反复配置, 同时也无需手动反复执行那些琐碎的命令. 包括我们上面提到的 `cargo fmt`, `cargo clippy`, `cargo check`, `cargo build`, `cargo publish` 都可以通过持续集成来完成, 稳定高效且可靠

有许多厂商都提供了持续集成服务, 在这里我们主要介绍 **Github Actions**. 我们先简述一下其几个核心概念

- GitHub Actions: 每个项目都可以拥有一个 Actions, 其可以包含多个工作流
- Workflow: 工作流, 描述了一次持续集成的过程
- Job: 作业, 一个工作流可以包含多个作业, 作业之间可以并行执行, 也可以有依赖关系, 例如先为多个平台并行构建, 都完成之后一并发布
- Step: 步骤, 每个作业由多个步骤组成, 按照顺序一步一步完成
- Action: 动作, 每个步骤可以包含多个动作

作为 Github 的官方服务, Actions 与 Github 进行了深度整合, 其本身就可以是一个 Github 项目. 因此你可以直接引用使用别人制作好的 Action, 也可以发布自己的 Action 给别人使用, 只需要简单的 `Username/RepoName@Version`

为自己的项目启用 Actions 只需要在项目根目录创建 `.github/workflows/*.yml`, 这里面每个 `.yml` 文件都将被识别成一个独立的 Workflow, YML 文件通过缩进区分层级, 因此编写时要注意缩进.

首先在 Github 创建一个项目, 至于其他的密钥配置就不多赘述了, 确保你拥有推送权限. 注意一点, 如果你希望你的 Workflow 能够发布 Release 及拥有写入权限, 需要导航到项目的 **Settings - Actions - General - Workflow permissions** 勾选 **Read and write permissions** 并保存, 然后将项目拉取到本地

下面以我实际使用的一个为例

```yml
# 工作流默认名称, 当你手动执行时就会显示这个名字
name: Build and Release

# 工作流触发时刻
on:
  # 手动触发
  workflow_dispatch:
  # 当发生 push 时
  push:
    # push 的内容需要是 tag
    tags:
      # 要求 tag 的格式
      - 'v*.*.*'

jobs:
  # 一个名为 build 的作业
  build:
    # 运行的系统环境
    runs-on: windows-latest
    # 声明整个作业的全局环境变量
    env:
      RELEASE_NAME: ""
      TAG_NAME: ""
      PRERELEASE: ""
      RELEASE_BODY: ""

    # 第一个步骤
    steps:
    # 步骤的名字, 也会显示在 Actions 日志中
    - name: Checkout code
      # 引用其他人的 Action
      uses: actions/checkout@v4
      # 对该 Action 添加的额外参数
      with:
        fetch-depth: 0

    - name: Setup Rust
      uses: actions-rust-lang/setup-rust-toolchain@v1
      with:
        toolchain: stable
        override: true

    - name: Build for Windows
      # 可以通过管道直接执行命令
      run: |
        cargo build --release
        tar -acf ./Defender-rs.zip -C ./target/release defender.exe defender_core.dll
        cargo build --lib
        Copy-Item -Force .\target\release\defender.exe .\target\debug\defender.exe
        tar -acf ./Defender-rs-debug.zip -C ./target/debug defender.exe defender_core.dll

    # 判断是否为预发布
    - name: Determine Release Type
      # id 用于后期获取指定 step 的信息
      id: determine_release
      # 修改使用的 shell
      shell: bash
      # 这里用到了一些 Github 内置的环境变量
      run: |
        if [ "${{ github.event_name }}" == "workflow_dispatch" ]; then
          echo "RELEASE_NAME=Defender-rs Nightly Build.$(date -u +'%Y.%m.%d')" >> $GITHUB_ENV
          echo "TAG_NAME=nightly" >> $GITHUB_ENV
          echo "PRERELEASE=true" >> $GITHUB_ENV
        else
          echo "RELEASE_NAME=Defender-rs Release Build.${{ github.ref_name }}" >> $GITHUB_ENV
          echo "TAG_NAME=${{ github.ref_name }}" >> $GITHUB_ENV
          echo "PRERELEASE=false" >> $GITHUB_ENV
        fi

    - name: Read Release Note
      id: read_release_note
      shell: bash
      run: |
        if [ -f "./Release.md" ]; then
          notes_content=$(cat "./Release.md")
          echo "content<<EOF" >> $GITHUB_OUTPUT
          echo "$notes_content" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
        else
          echo "content=No release notes provided." >> $GITHUB_OUTPUT
          echo "::warning file=./Release.md::Release notes file not found. Using default message."
        fi

    - name: Generate Changelog from PRs
      id: generate_changelog
      if: env.PRERELEASE == 'false'
      uses: mikepenz/release-changelog-builder-action@v5
      with:
        configurationJson: |
          {
            "categories": [
              {
                "title": "## What's Changed",
                "labels": []
              }
            ],
            "pr_template": "- #{{TITLE}} by @#{{AUTHOR}} (##{{NUMBER}})",
            "template": "#{{CHANGELOG}}",
            "pr_trim_body": true,
            "empty_template": "## No significant changes"
          }
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

    - name: Construct Release Body
      id: construct_body
      shell: bash
      # 这里就使用了一个来自之前 step 的输出内容
      run: |
        release_body="${{ steps.read_release_note.outputs.content }}"
        if [[ "${{ env.PRERELEASE }}" == "false" && -n "${{ steps.generate_changelog.outputs.changelog }}" ]]; then
          release_body="${release_body}

          ${{ steps.generate_changelog.outputs.changelog }}"
        fi
        echo "body<<EOF" >> $GITHUB_OUTPUT
        echo "$release_body" >> $GITHUB_OUTPUT
        echo "EOF" >> $GITHUB_OUTPUT

    - name: Create Release
      id: create_release
      uses: softprops/action-gh-release@v1
      with:
        name: ${{ env.RELEASE_NAME }}
        tag_name: ${{ env.TAG_NAME }}
        body: ${{ steps.construct_body.outputs.body }}
        draft: false
        prerelease: ${{ env.PRERELEASE == 'true' }}
        files: |
          ./Defender-rs.zip
          ./Defender-rs-debug.zip
```
