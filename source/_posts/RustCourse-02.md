---
feature: false
title: 初识 Rust(2) | 你好, 世界!
date: 2021-04-05 12:36:07
abstracts: Rust 与 Cargo 的基本操作, 注释与文档
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

# Hello Rust

## Hello World

学习一门语言的传统都是打印 **Hello World**, 下面让我们用 Rust 的方式向世界问好

## 创建一个 Rust 文件

打开 **VSCode**, 新建一个文件夹用于存放我们的代码, 在里面新建一个 `main.rs` 文件 (Rust的习惯后缀名为 `.rs` 虽然在只用 Rustc 编译时别的后缀名也能通过)

关于文件命名, Rust采用 **蛇形命名法**, 如果名字有多个单词, **无需** 有大写字母, 而是采用 `_` 来分隔每一个单词, 如 `hello_world.rs`, 尽量避免使用 **ASCII** 字符以外的字符, 不要以数字开头

## 编写并运行第一个 Rust 程序

```rust
// main.rs
fn main() {
    println!("Hello World!");
}
```

在 main.rs 文件上单击右键, 选择 **在终端中打开**, 然后执行以下命令

```sh
rustc main.rs && main
```

终端上会输出

```
Hello World!
```

## 分析这个程序

好了, 我们已经创建了第一个 Rust 程序了, 但这段代码到底是什么意思呢? 现在让我们来分析一下:

1. `fn` 表示定义一个 **函数**, `main` 是这个函数的名字, 花括号 `{}` 里的语句则是这个函数的内容, Rust 要求所有函数体都要用花括号包裹起来. 一般来说, 将左花括号与函数声明置于同一行并以空格分隔, 是良好的代码风格
2. 名字为 **main** 的函数在 Rust 里有特殊的作用, 即程序的入口, 程序就是从这里开始执行的
3. `println!()`是一个宏, 它的功能是打印圆括号`()`中的内容并换行, `!` 是宏的标志, 如果是调用函数, 则没有 `!`
4. 在 Rust 中, 语句的末尾一般用分号 `;` 作为结束标志
5. Rust 程序的编译与运行是彼此独立的, 在运行 Rust 程序之前, 必须先使用 Rust 编译器编译它, 即输入 rustc 命令并传入源文件名称
6. Rust 是一种 **预编译静态类型** 语言, 这意味着你可以编译程序, 并将可执行文件送给其他人, 他们甚至不需要安装 Rust 就可以运行. 这与解释性语言不同

# Hello Cargo

仅仅使用 Rustc 编译简单程序是没问题的, 不过随着项目的增长, Rustc 就会难以满足需要, 这种时候就需要包管理工具了

## Cargo 简介

代码管理对于编程来说一直是一个重要的问题, 各种不同的语言也都会采用不同的代码管理器, Rust 作为一枚现代语言, 综合了现有语言管理工具的优点, 为我们提供了一个大杀器 --- **Cargo**

作为 Rust 的代码组织管理工具, Cargo 提供了一系列的工具. 从项目的建立, 构建到测试, 运行直至部署, 为 Rust 项目的管理提供尽可能完整的手段. 同时与 Rust 语言及其编译器 Rustc 本身的各种特性紧密结合

## Cargo 入门

在我们安装 Rust 的时候就已经安装好了 Cargo

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

# Crate 与 Package

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
    - 可通过 `cargo run --bin <NAME>` 来运行
- 外部测试源代码文件位于 `tests/*.rs`
    - 这里每一个 `.rs` 文件内都可以有多个测试函数和测试模块
    - 可通过 `cargo test` 运行里面的所有测试
    - 可通过 `cargo test <KEYWORD>` 来运行测试函数名包含指定关键字的函数
- 示例程序源代码文件位于 `examples/*.rs`
    - 需要在 `Cargo.toml` 文件手动指定, 通过 `cargo run --example <NAME>` 运行
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
edition = "2021"

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

而对于 WorkSpace, 它可以组织多个 Package, `Cargo.toml` 的内容可能是这样的

```toml
[workspace]
members = [
    "path/to/package1",
    "path/to/package2",
    "..."
]
```

看起来非常简单, WorkSpace 下所有 Package 共用同一个 `target` 文件夹, 总体来看可以占用更少的磁盘空间, 加快编译速度

## Cargo.lock

该不需要直接修改, 是 Cargo 工具根据 `Cargo.toml` 生成的项目依赖详细清单文件, 如果不手动修改 `Cargo.toml` 或执行 `cargo update` 会锁定依赖项的版本, 即使包含通配符等

## 定义集成测试用例

Cargo 另一个重要的功能, 即将软件开发过程中必要且非常重要的测试环节进行集成, 并通过代码属性声明或者 `Cargo.toml` 文件描述来对测试进行管理

单元测试通常和代码耦合在一起, 使用 `#[test]` 属性宏来标记单个测试函数

集成测试也可以和代码耦合在一起, 可以通过 `#[cfg(test)]` 属性宏标记一个模块来包裹多个测试函数

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

1. 如果没有在 `Cargo.toml` 里定义集成测试的入口, 那么 `tests` 目录 (不包括子目录) 下的每个 `.rs` 文件被当作集成测试入口
2. 如果在 `Cargo.toml` 里定义了集成测试入口, 那么定义的那些 `.rs` 文件就是入口, 不再默认指定任何集成测试入口

## 定义示例和可执行文件

Example 用例的描述以及 Bin 用例的描述也是 Cargo 的常用功能, 其中之前提到的特殊的 Bin 文件夹可被自动识别无需手动配置

```toml
[[example]]
name = "examlpe1"
path = "examples/examlpe1.rs"

[[bin]]
name = "bin1"
path = "bin/bin1.rs"
```

对于 `[[example]]` 和 `[[bin]]` 段落中声明的 Examples 和 Bins, 需要通过 `cargo run --example <NAME>` 或者 `cargo run --bin <NAME>` 来运行

## 其他Cargo命令

- `cargo add <NAME> <OPTIONS>`: 添加依赖
- `cargo remove <NAME>`: 删除依赖
- `cargo clean`: 清理 `target` 文件夹中的所有内容
- `cargo update`: 根据 `Cargo.toml` 重新检索并更新各种依赖项的信息, 并写入 `Cargo.lock`
- `cargo install <NAME> <OPTIONS>`: 安装 `crates.io` 可用于实际的生产的可执行文件
- `cargo fmt`: 代码格式化工具
- `cargo check`: 代码检查工具, 不执行真正的 Build, 所以会快一点

从这开始, 请把 Cargo 当作习惯, 对于简单项目, Cargo 并不比 Rustc 提供了更多的优势, 不过随着开发的深入, 终将证明其价值

# 注释与文档

最后, 让我们学习一下 Rust 的注释

Rust 中有三种注释, 前两种分别为:

1. 行注释 `//...`
2. C 语言风格的块注释 `/*...*/`

```rust
// 创建一个绑定
let x = 5;
let y = 6; // 创建另一个绑定

/*这是一段块注释*/ let a = 1; /*块注释不影响块以外的代码*/

/*
块注释可以有很多行
*/
```

Rust的第三种注释是 **文档注释**, 文档注释又分为两小种, 支持 **Markdown** 语法:

1. `//!` 模块注释, 用来描述包含它的项, 一般用在模块文件的头部
2. `///` 用来描述的它后面接着的项

```rust
//! 这是一个模块

/// #Add One
/// ## Example
/// ```rust
/// let one = 1;
/// assert_eq!(2, add_one(one));
/// ```
fn add_one(x: i32) -> i32 {
    x + 1
}
```

## 生成文档文件

Rust 工具链可以很轻易地从文档注释中生成漂亮的 **HTML** 文档:

1. `rustdoc *.rs`: 用于单个文件
2. `cargo doc`: 用于整个项目, 最好加上 `--no-depth` 参数, 否则将给你所有的依赖项也生成文档
