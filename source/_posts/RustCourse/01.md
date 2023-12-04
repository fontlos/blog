---
feature: true
title: 初识 Rust(1) | Rust 环境配置
date: 2021-04-05 09:54:09
abstracts: 这是一个非官方的 Rust 语言简易教程, 目的是让新手快速熟练基本语法, 在实践中编写并加入了自己的理解和一些在其他教程中可能很少会被提到的琐碎之物
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

# 简介

**Rust**, 连续七年成为全世界最受欢迎的语言, 一门系统级, 多范式,赋予每个人构建可靠且高效软件能力的语言, 如今被越来越多的大公司所接受, 并成功进入 Windows, Linux, Android 等主流操作系统的内核

只要你喜欢编程, 无论是否用于参与工作, 相信学习这门语言都能让你受益匪浅

这是一个非官方的 Rust 语言简易教程, 在实践中编写并加入了自己的理解和一些在其他教程中可能很少会被提到的琐碎之物

# Rust环境配置

## 安装C++build tool

Rust 依赖 C 的链接器等组件, 所以电脑上至少要存在一个 C/C++ 编译环境

### Windows

### 安装VisualStudio

可以在 **单个组件** 选择只安装需要的组件, 关键词如下
- MSVC C/C++ 生成工具
- 适用于最新生成工具的 C++ ATL
- Windows SDK

担心出问题或者觉得麻烦也可以直接在 **工作负载** 选择 **使用 C++ 的桌面开发**

>注: 可以选择 [MinGW](https://www.mingw-w64.org/), 但是 Windows下还是更推荐 MSVC 环境

### Mac & Linux

- Mac 需要安装 Xcode
```sh
xcode-select --install
# 如果已安装则会显示已安装
```
- Linux 需要安装 GCC 或 Clang, 用系统对应的包管理器直接安装即可

## 安装Rust

### 配置镜像源

因为Rust的服务器在国外, 安装速度较慢, 所以我们可以考虑使用镜像源

这里使用 **字节跳动** 源
```sh
# Windows 直接在系统变量下加入以下两条
# 注意: 不是 Path 环境变量
RUSTUP_UPDATE_ROOT
https://rsproxy.cn/rustup

RUSTUP_DIST_SERVER
https://rsproxy.cn

# Mac & Linux 可以在终端执行以下两行命令
# 或者将这两行粘贴到终端的配置文件中(如~/.bashrc)
export RUSTUP_UPDATE_ROOT=https://rsproxy.cn/rustup
export RUSTUP_DIST_SERVER=https://rsproxy.cn
```

### Windows

如果不想安装在 C 盘的话需要提前配置两个系统变量

再在系统变量中加入以下两条, 下面为示例
```sh
RUSTUP_HOME
D:/Rust/rustup

CARGO_HOME
D:/Rust/cargo
```
在官网 [Rust-lang](https://www.rust-lang.org/zh-CN) 下载 rustup-init.exe并运行

### Mac & Linux

直接在终端执行
```sh
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

## 环境配置

### 安装选项

运行安装程序后, 终端会输出 `Current installtion options`

- 输入 `1` 执行安装
- 输入 `2` 进行自定义选项
    - 其中最主要的配置就是 Rust 的版本, 包括:
        - **Stable**: 稳定版的 Rust, API 与语言特性不会更改, 保持向后兼容
        - **Beta**: 测试版的 Rust, 测试一些即将加入 Stable 的新功能
        - **Nightly**: 每日更新版的 Rust, 有最前沿的功能和不稳定特性
- 输入 `3` 取消安装

随后将 `CARGO_HOME/bin` 加入环境变量(如 Windows 的 Path)

最后, 可以在 `CARGO_HOME` 文件夹下新建 `config.toml` 添加一些全局配置

```toml
# 定义一些命令别名
[alias]
# cargo b
b = "build"
c = "check"
t = "test"
r = "run"

[build]
# 利用编译缓存加快编译
# 需要安装 sscache 并配置 SCCACHE_DIR
rustc-wrapper = "sccache"

[target.x86_64-pc-windows-msvc]
# Windows 上比 link 更快的链接器
linker = 'rust-lld.exe'

# 替换默认 crate 源
[source.crates-io]
replace-with = 'rsproxy-sparse'

# 字节跳动的 crate 源
[source.rsproxy]
registry = "https://rsproxy.cn/crates.io-index"

# 通过 sparse 算法加速检索
[source.rsproxy-sparse]
registry = "sparse+https://rsproxy.cn/index/"

[registries.rsproxy]
index = "https://rsproxy.cn/crates.io-index"

# 使用系统自带的 git-cli 加速检索
[net]
git-fetch-with-cli = true

[http]
check-revoke = false
```

### 验证安装
终端输入
```sh
rustc -V
cargo -V
```

### 安装编辑器

截止到目前, JetBrain 推出了 Rust 的专用 IDE [RustRover](https://www.jetbrains.com/rust/), 但还不是很完善

这里推荐[VSCode](https://code.visualstudio.com/) + [RA](https://github.com/rust-lang/rust-analyzer)

>Rust-Analyer(简称RA)是由Rust社区维护的RLS 2.0, RLS有的它更好, RLS没有的它还有, 原来的官方插件已废弃, 现在 RA 正式进入 Rust 官方 Repository, 所以更推荐安装这一个. 更多内容可以浏览它官网的[用户手册](https://rust-analyzer.github.io/manual.html)

下面是其他的可能需要的插件

- [Chinese (Simplified)](https://marketplace.visualstudio.com/items?itemName=MS-CEINTL.vscode-language-pack-zh-hans) 简体中文插件
- [Crates](https://marketplace.visualstudio.com/items?itemName=serayuzgur.crates) Rust Crate 版本检查插件
- [Even Better Toml](https://marketplace.visualstudio.com/items?itemName=tamasfe.even-better-toml) 更好的 Toml 语言插件
- [Codeium](https://marketplace.visualstudio.com/items?itemName=Codeium.codeium) 一个类似 Copilot 的 AI 代码提示插件
- [CodeLLDB](https://marketplace.visualstudio.com/items?itemName=vadimcn.vscode-lldb) 可以用于调试 Rust 代码, 但功能有限, 需要安装对应的组件 `rustup component add llvm-tools`

至此, Rust 语言环境配置完毕

# Ferris

既然是初识, 那让我们也来先了解一些 Rust 语言的故事, 比如, 先从封面那只可爱的小螃蟹说起

这只小螃蟹叫 **Ferris**, 是 Rust 社区为 Rust 语言制作的吉祥物

红色的外表难免让人产生一些疑问: 它为什么是个熟螃蟹呢?

对不起, 这不是熟的, 还请给 Ferris 道个歉!

除了大家所熟知的 Rust 游戏和我们今天介绍的 Rust 语言, 别忘了 Rust 最初的含义 --- 锈, 而锈中的代表物, 铁锈, `Fe₂O₃` 就是类似这种红色

社区中常常打着 **REIR (Rewrite Everything In Rust)** 的旗号, 而用 Rust 重写某个组件的过程常常被称为 **「氧化」** . 比如有一个为Python编写Rust扩展的库, 就叫PyO3, 也有许多库从诞生时名字就以 **Dio (二氧化物)** 开头.

至于为什么选个螃蟹当吉祥物, 或许这是因为 Rust 开发者们的一个自称 --- **Rustacean**, 因为这个词是 **「甲壳纲动物」** 这个单词 **Crustacean (krʌ'steʃən)** 去掉了首字母 C 演变而来的, 去掉了C 就露出了 Rust 这四个字母, 或许还含有着去掉 C 用 Rust 重写之意. 而在甲壳纲动物里对大家而言螃蟹应该是最熟悉的了吧。