---
feature: false
title: rCore 学习笔记(1)|从裸机程序开始
date: 2025-03-17 12:00:00
abstracts: rCore 系列课程的学习笔记, 在第一章将从环境配置开始, 移除标准库, 构建一个能运行在 Qemu 模拟器的裸机程序
tags:
    - OS
    - rCore
    - Rust
categories:
    - Course
cover: https://fontlos.com/icons/logo.png
---

# 环境准备

## 系统环境

为了避免不必要的环境错误, 我们在 Linux 上, 最好是 **Ubuntu** 发行版上进行开发, 对于 Windows, 可以使用 WSL2 进行, 注意必须是 WSL2, 因为 WSL1 本质上还是在进行 API 翻译, WSL2 才内置了一个真正的 Linux 内核. 至于如何安装可以查看 [往期文章](https://fontlos.com/post/2025-03-12), 只需要通过这个启用 WSL2 即可, 然后如果你不需要什么自定义选项, 在不安装任何发行版的情况下直接执行 `wsl` 命令就会自动安装一个 Ubuntu 发行版, 或者也可以前往微软应用商店安装

对于 MacOS 或其他不想使用 WSL 的情况, 可以使用虚拟机或 Docker, Docker 相关文件已经在储存库对应分支根目录了

## Rust 环境

首先需要安装一个基本的 C/Cpp 环境, 通过你的发行版的包管理工具安装 `gcc` 和 `make` 即可, 至于 `git` 一类基本工具请读者自行安装

对于 WSL, 可以使用 `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh` 命令来安装 Rust. 由于在国内访问比较慢, 可以先设置两个环境变量

```sh
export RUSTUP_DIST_SERVER="https://rsproxy.cn"
export RUSTUP_UPDATE_ROOT="https://rsproxy.cn/rustup"
```

更多环境配置细节和编辑器配置也可以参考 [往期文章](https://fontlos.com/post/RustCourse-01), 虽然是为 Windows 写的但也有一些参考价值

## Qemu 模拟器环境

对于多数 Linux 发行版都可以通过包管理工具简单的安装

```sh
# Ubuntu
apt-get install qemu-system
# Arch, 简单选择 full 即可
pacman -S qemu
```

但是高版本和官方储存库的代码会有一些不兼容问题, 你需要下载最新版本的 [`bootloader/rustsbi-qemu.bin`](https://github.com/rustsbi/rustsbi-qemu/releases) 替换掉默认的. 后面开发时也有一些常量值需要修改

```rs
// os/src/sbi.rs
const SBI_SHUTDOWN: usize = 0x53525354;
const SBI_SET_TIMER: usize = 0x54494D45;
```

这些到后面再说

或者你也可以手动编译一份 7.0 版本, 对于 Ubuntu 来说

```sh
# 安装编译所需的依赖包
sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
    gawk build-essential bison flex texinfo gperf libtool patchutils bc \
    zlib1g-dev libexpat-dev pkg-config  libglib2.0-dev libpixman-1-dev git tmux python3
# 下载源码包
# 如果下载速度过慢可以使用我们提供的百度网盘链接：https://pan.baidu.com/s/1z-iWIPjxjxbdFS2Qf-NKxQ
# 提取码 8woe
wget https://download.qemu.org/qemu-7.0.0.tar.xz
# 解压
tar xvJf qemu-7.0.0.tar.xz
# 编译安装并配置 RISC-V 支持
cd qemu-7.0.0
./configure --target-list=riscv64-softmmu,riscv64-linux-user
make -j$(nproc)
```

对于不同的系统可能有不同的需要安装的包, 按需安装即可, 提示缺什么可以查一查

最后验证一下是否安装成功, 我们主要需要 RiscV 相关的工具

```sh
qemu-system-riscv64 --version
```

## 环境测试

首先克隆源码

```sh
git clone https://github.com/LearningOS/rCore-Tutorial-Code-2024S
cd rCore-Tutorial-Code-2024S
git checkout ch1
```

这份源码比较旧了, 如果你使用高版本 Qemu, 我们需要做一些修改(也可以克隆我的 [fork](https://github.com/fontlos/rCore-Tutorial))

首先修改 `rust-toolchain.toml` 文件, 修改 `nightly` 后面的日期, 例如 `2025-03-16`, 因为部分特性在新版本 Rust 已经稳定

移除 `os/src/main.rs` 的 `#![feature(panic_info_message)]`, 这个特性已经稳定

移除 `os/src/lang_items.rs` 中 `info.message()` 后面的 `unwrap()` 方法

不要忘了替换 `bootloader/rustsbi-qemu.bin` 文件

尝试运行, 注意, 暂时不要尝试手动运行而不是用 Makefile, 这能帮你自动安装一些依赖

```sh
cd os
LOG=DEBUG make run
```

如果一切顺利, 将有以下输出

```
[rustsbi] RustSBI version 0.4.0-alpha.1, adapting to RISC-V SBI v2.0.0
.______       __    __      _______.___________.  _______..______   __
|   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
|  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
|      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
|  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
| _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|
[rustsbi] Implementation     : RustSBI-QEMU Version 0.2.0-alpha.3
[rustsbi] Platform Name      : riscv-virtio,qemu
[rustsbi] Platform SMP       : 1
[rustsbi] Platform Memory    : 0x80000000..0x88000000
[rustsbi] Boot HART          : 0
[rustsbi] Device Tree Region : 0x87e00000..0x87e012c4
[rustsbi] Firmware Address   : 0x80000000
[rustsbi] Supervisor Address : 0x80200000
[rustsbi] pmp01: 0x00000000..0x80000000 (-wr)
[rustsbi] pmp02: 0x80000000..0x80200000 (---)
[rustsbi] pmp03: 0x80200000..0x88000000 (xwr)
[rustsbi] pmp04: 0x88000000..0x00000000 (-wr)
[kernel] Hello, world!
[DEBUG] [kernel] .rodata [0x80202000, 0x80203000)
[ INFO] [kernel] .data [0x80203000, 0x80204000)
[ WARN] [kernel] boot_stack top=bottom=0x80214000, lower_bound=0x80204000
[ERROR] [kernel] .bss [0x80214000, 0x80215000)
```

前面打印了包括各种版本号在内的关键信息

- 平台内存范围: 0x80000000..0x88000000 (128MB 内存)
- 启动的 HART (Hardware Thread): 0 (单核 CPU)
- 设备树区域: 0x87e00000..0x87e012c4
- 固件地址: 0x80000000 (RustSBI 的加载地址)
- Supervisor 地址: 0x80200000 (内核的加载地址)
pmp01: 0x00000000..0x80000000 (禁止写和读)
pmp02: 0x80000000..0x80200000 (禁止所有访问)
pmp03: 0x80200000..0x88000000 (允许执行、写和读)
pmp04: 0x88000000..0x00000000 (禁止写和读)

这些信息表明 RustSBI 已成功初始化, 并将控制权移交给了内核. 后面有一些 PMP (Physical Memory Protection) 配置, 这些配置确保了内核只能访问其允许的内存区域 (0x80200000..0x88000000), 而其他区域被保护起来

至此, 恭喜你完成环境配置

## 一些补充

在 `os` 目录下 `make debug` 可以调试我们的内核, 这需要安装终端复用工具 `tmux` 和基于 riscv64 平台的 gdb 调试器, 详情见 [rCore-Tutorial-Guide-2024S](https://learningos.cn/rCore-Tutorial-Guide-2024S/0setup-devel-env.html#gdb)

# 让内核运行在裸机环境

在使用 Rust 创建一个新项目时, 例如 `cargo new rcore`

Cargo 会自动帮我们完成一个最简单的 Rust 程序

```rust
fn main() {
    println!("Hello, world!");
}
```

简单运行一下

```sh
cd rcore
cargo run
```

```
   Compiling rcore v0.1.0 (/root/Code/rcore)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.40s
     Running `target/debug/rcore`
Hello, world!
```

对于调试程序而言最常用的就是 `println!` 宏, 但即使是这么常用的简单的功能, 也不像表面上那么简单, 尤其是 Rust 的这个还是个宏而非函数, 表面三行代码下封装了许多细节, 宏会展开成函数调用, 其依赖于 Rust 标准库, 而标准库又依赖特定系统的系统调用, 系统调用取决于不同的系统和内核, 再往下依赖特定硬件的指令集, 层层抽象使得我们可以用少量的高级语言代码执行复杂的功能

可以通过下面的命令简单的查看一下当前平台的目标三元组

```sh
rustc -V -v
```
```
rustc 1.87.0-nightly (227690a25 2025-03-16)
binary: rustc
commit-hash: 227690a258492c84ae9927d18289208d0180e62f
commit-date: 2025-03-16
host: x86_64-unknown-linux-gnu
release: 1.87.0-nightly
LLVM version: 20.1.0
```

其中 `host` 字段就是我们所需的信息, 从前到后表示我们的编译链接目标是一个 `x86_64` 指令集的 CPU, 生产厂商未知, 建立在 Linux 的 GNU 运行时上

相应的, RiscV 的目标三元组是 `riscv64gc-unknown-none-elf`, 其中 `riscv64` 是指令集, 而后面的 `gc` 是指令集的扩展, 其中 `g` 是一个通用扩展, 包含了整数扩展 `i`, 乘除扩展 `m`, 原子扩展 `a`, 单双精度浮点扩展 `f` 和 `d`. `c` 对应了压缩扩展

如果直接尝试将这个程序编译链接到 RiscV 上

```sh
cargo run --target riscv64gc-unknown-none-elf
```

```
Compiling rcore v0.1.0 (/root/Code/rcore)
error[E0463]: can't find crate for `std`
  |
  = note: the `riscv64gc-unknown-none-elf` target may not support the standard library
  = note: `std` is required by `rcore` because it does not declare `#![no_std]`
  = help: consider building the standard library from source with `cargo build -Zbuild-std`

error: cannot find macro `println` in this scope
 --> src/main.rs:2:5
  |
2 |     println!("Hello, world!");
  |     ^^^^^^^

error: `#[panic_handler]` function required, but not found

For more information about this error, try `rustc --explain E0463`.
error: could not compile `rcore` (bin "rcore") due to 3 previous errors
```

很明显我们的裸机环境上显然没有标准库和系统调用这些功能, 不过除了标准库, Rust 还拥有一个核心库 Core, 它不依赖任何系统, 只包含 Rust 语言的核心功能, 所以要想运行我们的内核, 至少要移除对内核程序标准库的依赖且通过编译, 这通常被称为 `no_std` crate. 而且就像我们在上面说的, `println!` 宏在调试中相当常用, 因此下一步就是通过核心库和其他 `no_std` 库在用户态复活这个宏. (本质上我们就是在实现一个小的特定平台标准库)

最后把程序移植到内核态, 构建在裸机上支持输出的最小运行时环境

而为了加深理解这一过程, 在下面的内容我们将脱离 `rCore-Tutorial-Code-2024S` 这个框架, 自己动手开始实现这些功能

## 移除标准库

首先为了不在每次编译的时候加上 `target` 参数, 我们在当前项目根目录新建一个 `.cargo` 文件夹, 并在里面新建一个 `config.toml` 文件, 写入以下内容

```toml
[build]
target = "riscv64gc-unknown-none-elf"
```

这会让 Cargo 使用 `riscv64gc-unknown-none-elf` 作为目标三元组编译当前项目, 在一个目标三元组主机上编译另一个目标三元组程序的行为称为 **交叉编译**

然后在 `src/main.rs` 文件的第一行加上 `#![no_std]`, 这是一个 crate 级别的属性宏, 只能用在 `main.rs/lib.rs` 中, 告诉编译器这整个项目都不需要对标准库进行链接, 因此会改用核心库

再次尝试运行, 这次只需要 `cargo run`

```
   Compiling rcore v0.1.0 (/root/Code/rcore)
error: cannot find macro `println` in this scope
 --> src/main.rs:4:5
  |
4 |     println!("Hello, world!");
  |     ^^^^^^^

error: `#[panic_handler]` function required, but not found

error: could not compile `rcore` (bin "rcore") due to 2 previous errors
```

可以看到这次错误信息变成两个了, 我们一个一个看, 先看第一个, 说找不到 `println!` 宏, 因为这个宏就是由标准库提供的, 内部封装了一些特定平台系统调用, 我们先直接删掉它

然后看第二个错误, 找不到一个被 `#[panic_handler]` 属性宏标记的函数, 这是一个 **Lang Items(语言项)**, 是用来给编译器识别的特殊符号或函数, 用于实现语言核心功能, 例如这个函数将在程序发生 **Panic** 时被调用, 同样是标准库提供的, 默认会打印错误信息并退出程序. 因此这里需要我们手动提供一个函数

```rs
// 新建文件 src/lang_items,rs
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

`!` 是一个特殊的返回类型, 它表示永不返回, `panic!` 宏的返回值就是它, 目前在我们实现的 `panic` 函数中什么都不做, 接受一个参数但忽略, 在函数体内只是无限循环, 现在在我们的代码编辑器上应该是没有飘红的报错了, 再次运行

```
error: using `fn main` requires the standard library
  |
  = help: use `#![no_main]` to bypass the Rust generated entrypoint and declare a platform specific entrypoint yourself, usually with `#[no_mangle]`

error: could not compile `rcore` (bin "rcore") due to 1 previous error
```

出现了新的错误, 它说使用 `main` 函数需要标准库, 但我们又知道, `main` 函数是程序的入口点, 怎么能没有 `main` 函数呢? 下面的提示帮助了我们, 可以使用 `#![no_main]` 绕过 Rust 生成的入口点并自己声明一个特定于平台的入口点, 并且这个函数通常会使用 `#[no_mangle]` 来标记, 默认情况下编译器会修改函数名使其包含更多信息方便编译器, 但这会让函数名变得不可读, 而这个属性宏的意思就是告诉编译器不要这么做, 保持原符号就好. 这是一个 **Unsafe** 属性宏, 因此如果你使用高版本 Rust, 需要写成 `#[unsafe(no_mangle)]`

接下来在 `src/main.rs` 最上面也添加上 `#![no_main]`, 既然都绕过入口点了, 那 `main` 函数也没有存在的意义了, 暂时可以删掉

```rs
// src/main.rs
#![no_std]
#![no_main]

mod lang_items;
```

这时执行 `cargo build`, 没有错误, 至少我们的程序通过编译了

接下来可以用一些工具来分析一下我们的构建产物

```sh
# 文件格式
file target/riscv64gc-unknown-none-elf/debug/rcore
# target/riscv64gc-unknown-none-elf/debug/rcore: ELF 64-bit LSB executable, UCB RISC-V, RVC, double-float ABI, version 1 (SYSV), statically linked, with debug_info, not stripped

# 文件头信息
rust-readobj -h target/riscv64gc-unknown-none-elf/debug/rcore
# File: target/riscv64gc-unknown-none-elf/debug/rcore
# Format: elf64-littleriscv
# Arch: riscv64
# AddressSize: 64bit
# LoadName: <Not found>
# ElfHeader {
#   Ident {
#     Magic: (7F 45 4C 46)
#     Class: 64-bit (0x2)
#     DataEncoding: LittleEndian (0x1)
#     FileVersion: 1
#     OS/ABI: SystemV (0x0)
#     ABIVersion: 0
#     Unused: (00 00 00 00 00 00 00)
#   }
#   Type: Executable (0x2)
#   Machine: EM_RISCV (0xF3)
#   Version: 1
#   Entry: 0x0
#   ProgramHeaderOffset: 0x40
#   SectionHeaderOffset: 0x13C8
#   Flags [ (0x5)
#     EF_RISCV_FLOAT_ABI_DOUBLE (0x4)
#     EF_RISCV_RVC (0x1)
#   ]
#   HeaderSize: 64
#   ProgramHeaderEntrySize: 56
#   ProgramHeaderCount: 4
#   SectionHeaderEntrySize: 64
#   SectionHeaderCount: 12
#   StringTableSectionIndex: 10
# }

# 反汇编程序
rust-objdump -S target/riscv64gc-unknown-none-elf/debug/rcore
# target/riscv64gc-unknown-none-elf/debug/rcore:  file format elf64-littleriscv
```

通过 `file` 工具可以看到, 它似乎确实是 RiscV64 可执行程序, 但 `rust-readobj` 显示它的入口地址 `Entry` 是 `0`, 这显然不是一个正常的地址, 并且使用 `rust-objdump` 尝试反汇编时, 没有生成任何汇编代码

综上, 我们的可执行文件虽然格式合法, 但实际上是一个空程序, 原因是缺少了刚刚我们所说的编译器规定的入口函数 `_start`, 虽然绕过了它, 但我们没有提供一个新的

虽然程序暂时无法执行, 但至少我们已经摆脱了对标准库的依赖

## 让程序在用户态运行起来

接下来让我们添加这个入口函数, 还记得编译器在之前的提示吗, 需要用一个属性宏来标记

```rs
// src/main.rs
#[unsafe(no_mangle)]
extern "C" fn _start() {
    loop {}
}
```

这里 `extern "C"` 表示用 C 语言的 ABI 格式来导出这个函数, 在函数体内我们同样什么都不做, 仅仅是无限循环

再次编译并分析, 这里我们主要看反汇编信息

```sh
rust-objdump -S target/riscv64gc-unknown-none-elf/debug/rcore
# target/riscv64gc-unknown-none-elf/debug/rcore:  file format elf64-littleriscv

# Disassembly of section .text:

# 0000000000011158 <_start>:
# warning: address range table at offset 0x30 has a premature terminator entry at offset 0x40
# ;     loop {}
#    11158: a009          j       0x1115a <_start+0x2>
#    1115a: a001          j       0x1115a <_start+0x2>
```

可以看到成功导出了汇编, 其含义也正是一个死循环, 这代表我们的程序现在是一个合法的可以执行的程序了, 如果你愿意, 也可以尝试运行一下, 需要 Crtl-C 退出

```sh
qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/rcore
```

目前我们的程序无法正常退出, 即使注释掉入口文件里的无限循环, 尝试编译运行也会触发一个段错误, 因此我们需要使用操作系统的 `exit` 系统调用

```rs
// 在 RiscV 上用于表示 exit 的系统调用号
const SYSCALL_EXIT: usize = 93;

// 用于发起系统调用, 传入系统调用号和参数
fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret;
    unsafe {
        core::arch::asm!(
            // 在 RiscV 中 ecall 指令用于触发系统调用, 从用户态切换到内核态
            "ecall",
            // 表示 x10 寄存器既是输入也是输出, 用于传递第一个参数, 并接收系统调用的返回值
            inlateout("x10") args[0] => ret,
            // 用于传递后两个参数
            in("x11") args[1],
            in("x12") args[2],
            // 用于传递系统调用号, 执行时读取上面三个寄存器的参数
            in("x17") id,
        );
    }
    ret
}

// 发起 exit 系统调用, 传入一个退出状态码
pub fn sys_exit(xstate: i32) -> isize {
    syscall(SYSCALL_EXIT, [xstate as usize, 0, 0])
}

#[unsafe(no_mangle)]
extern "C" fn _start() {
    // 调用退出系统调用, 随便传入一个状态码比如 10
    sys_exit(10);
}
```

编译后, 在 bash 上可以执行

```sh
qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/rcore; echo $?
```

对于我的 nushell 则是

```sh
do { qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/rcore }
echo $env.LAST_EXIT_CODE
```

可以看到打印出了 10, 与我们传入的一样

## 复活打印输出功能
