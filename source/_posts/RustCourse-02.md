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

打开 **VSCode**, 新建一个文件夹用于存放我们的代码, 在里面新建一个 `main.rs` 文件 (Rust的习惯后缀名为 `.rs` 虽然在只用 Rustc 编译时别的后缀名也能通过)

关于文件命名, Rust采用 **蛇形命名法**, 如果名字有多个单词, **无需** 有大写字母, 而是采用 `_` 来分隔每一个单词, 如 `hello_world.rs`, 尽量避免使用 **ASCII** 字符以外的字符, 不要以数字开头

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

对于单文件简单程序使用 Rustc 编译是没问题的, 不过随着项目的增长, Rustc 就会难以满足需要

代码管理对于编程来说一直是一个重要的问题, Rust 作为一枚现代语言, 综合了现有语言管理工具的优点, 为我们提供了一个强大的代码组织管理工具 --- **Cargo**. Cargo 提供了一系列的工具. 从项目的建立, 构建, 测试, 运行直至部署, 为 Rust 项目的管理提供尽可能完整的手段, 极大的减轻了开发复杂度. 同时与 Rust 语言及其编译器 Rustc 本身的各种特性紧密结合

Cargo 的强大有目共睹, 包括许多现代新兴的包管理器都或多或少的借鉴了 Cargo, 例如 Python 的 **Uv**

## Cargo 简单使用

在我们安装 Rust 的时候就已经安装好了 Cargo. 下面我们简单的使用一下, 在终端中打开一个文件夹, 输入以下命令

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

> 关于 Cargo 的更多内容以及其他 Rust 基建设施, 请查看 [初识 Rust(8) | 基建设施](https://fontlos.com/post/RustCourse-08)

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

/// # Add One
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
