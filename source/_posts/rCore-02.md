---
feature: false
title: rCore 之旅(2)|基本应用程序与批处理系统
date: 2025-03-18 17:00:00
abstracts: 如果只能运行一个内核, 那是没有什么用的, 如果将一切功能都封装到内核中, 那么一旦某一功能出现错误将给内核带来毁灭性灾难. 所以在这一节, 我们将实现基本的批处理系统和几个基本的应用程序
tags:
    - OS
    - rCore
    - Rust
categories:
    - Course
cover: https://fontlos.com/icons/logo.png
---

如果只能运行一个内核, 那是没有什么用的, 如果将一切功能都封装到内核中, 那么一旦某一功能出现错误将给内核带来毁灭性灾难, 所以保护内核不被其他程序破坏, 我们引入了 **特权级** 机制, 将用户程序与内核隔离开

**批处理系统** 诞生于计算资源匮乏的年代, 基本上不具备交互能力, 只是将多个程序一起打包到计算机, 让他们逐个执行, 因其思想足够简单, 所以我们先以批处理系统的模式设计我们的用户程序, 然后再将它们链接到内核中执行

# 应用程序实现

如果你使用官方的框架, 需要切换到 `ch2` 分支, 先在项目根目录克隆我们的应用程序储存库

```sh
git clone https://github.com/LearningOS/rCore-Tutorial-Test-2025S.git user
```

这会克隆库并重命名文件夹为 `user`, 如果你使用官方框架, 此时可以通过以下命令测试一下

```sh
cd os
make run LOG=INFO
```

## 前置准备

而我们的教程继续在之前的项目基础上修改, 我们直接在 `rcore` 的根目录新建一个 Lib crate, 为了防止在编译时和项目顶层的 `.cargo` 配置文件冲突, 在编写代码时你可以将这个 crate 加入到 `workspace`, 但编译时需要删去

我们现在关心的是如何为内核构建批处理系统, 而不是构建应用本身, 而且应用的内容很多, 而且储存库都已经为大家实现好了, 对于高版本 Rust 可能有一些小的报错, 都很好修改, 所以这里我们直接给出代码和注释, 简单看一下就好

先添加几个依赖项和设置

```toml
[dependencies]
buddy_system_allocator = "0.6"
bitflags = "1.2.1"
spin = "0.9"
lock_api = "=0.4.6"
lazy_static = { version = "1.5.0", features = ["spin_no_std"] }

[profile.release]
opt-level = "z"
strip = true
lto = true
```

## 入口点

```rs
// app/src/lib.rs
#![no_std]
#![feature(linkage)]
#![feature(alloc_error_handler)]

#[macro_use]
pub mod console;
mod lang_items;
mod syscall;

extern crate alloc;
extern crate core;

use alloc::vec::Vec;
use buddy_system_allocator::LockedHeap;
pub use console::{flush, STDOUT};
pub use syscall::*;

// 因为其他地方使用了 static 全局变量, 需要一个堆, 暂时不重要
// 使用一个静态可变数组定义了一个 16KB 的堆
const USER_HEAP_SIZE: usize = 16384;
static mut HEAP_SPACE: [u8; USER_HEAP_SIZE] = [0; USER_HEAP_SIZE];
// 使用 buddy_system_allocator 全局分配器
#[global_allocator]
static HEAP: LockedHeap = LockedHeap::empty();
// 处理对分配错误的情况
#[alloc_error_handler]
pub fn handle_alloc_error(layout: core::alloc::Layout) -> ! {
    panic!("Heap allocation error, layout = {:?}", layout);
}

// 清理 BSS 段
fn clear_bss() {
    unsafe extern "C" {
        fn start_bss();
        fn end_bss();
    }
    unsafe {
        core::slice::from_raw_parts_mut(
            start_bss as usize as *mut u8,
            end_bss as usize - start_bss as usize,
        )
        .fill(0);
    }
}

// 入口点, 将被挂载到代码段的 `.text.entry`
#[unsafe(no_mangle)]
#[unsafe(link_section = ".text.entry")]
#[allow(static_mut_refs)]
pub extern "C" fn _start(argc: usize, argv: usize) -> ! {
    clear_bss();
    // 初始化堆
    unsafe {
        HEAP.lock()
            .init(HEAP_SPACE.as_ptr() as usize, USER_HEAP_SIZE);
    }
    let mut v: Vec<&'static str> = Vec::new();
    for i in 0..argc {
        let str_start =
            unsafe { ((argv + i * core::mem::size_of::<usize>()) as *const usize).read_volatile() };
        let len = (0usize..)
            .find(|i| unsafe { ((str_start + *i) as *const u8).read_volatile() == 0 })
            .unwrap();
        v.push(
            core::str::from_utf8(unsafe {
                core::slice::from_raw_parts(str_start as *const u8, len)
            })
            .unwrap(),
        );
    }
    exit(main(argc, v.as_slice()));
}

// 这是一个弱链接函数, 如果用户没有实现自己的 main 函数, 默认链接到这个, 并且直接在运行时报错
#[linkage = "weak"]
#[unsafe(no_mangle)]
fn main(_argc: usize, _argv: &[&str]) -> i32 {
    panic!("Cannot find main!");
}

// 下面是一些系统调用相关的

#[repr(C)]
#[derive(Debug, Default)]
pub struct TimeVal {
    pub sec: usize,
    pub usec: usize,
}

impl TimeVal {
    pub fn new() -> Self {
        Self::default()
    }
}

#[derive(Copy, Clone, PartialEq, Debug)]
pub enum TaskStatus {
    UnInit,
    Ready,
    Running,
    Exited,
}

#[repr(C)]
pub enum TraceRequest {
    Read,
    Write,
    Syscall,
}

// 用于控制台输出的系统调用
pub fn write(fd: usize, buf: &[u8]) -> isize {
    sys_write(fd, buf)
}

// 用于退出程序的系统调用
pub fn exit(exit_code: i32) -> ! {
    console::flush();
    sys_exit(exit_code);
}

pub fn yield_() -> isize {
    sys_yield()
}

pub fn get_time() -> isize {
    let mut time = TimeVal::new();
    match sys_get_time(&mut time, 0) {
        0 => ((time.sec & 0xffff) * 1000 + time.usec / 1000) as isize,
        _ => -1,
    }
}

pub fn sleep(period_ms: usize) {
    let start = get_time();
    while get_time() < start + period_ms as isize {
        sys_yield();
    }
}

pub fn trace(request: TraceRequest, id: usize, data: usize) -> isize {
    sys_trace(request as usize, id, data)
}

pub fn trace_read(addr: *const u8) -> Option<u8> {
    match trace(TraceRequest::Read, addr as usize, 0) {
        -1 => None,
        data => Some(data as u8),
    }
}

pub fn trace_write(addr: *const u8, data: u8) -> isize {
    trace(TraceRequest::Write, addr as usize, data as usize)
}

pub fn count_syscall(id: usize) -> isize {
    trace(TraceRequest::Syscall, id, 0)
}
```

因为后面有一些组件用了 static 全局变量, 所以我们需要一个简单的堆, 这个不重要暂时不去管它, 然后是熟悉的, 定义 `_start` 入口函数, 在里面清理 BSS 段, 初始化堆, 并将这个函数在编译后放在 `.text.entry` 代码段

值得注意的是我们有一个弱链接的 main 函数, 如果用户没有实现自己的 main 函数则在编译时会链接到这里, 功能只是在运行时报错.

再往下是一些系统调用, 包括一些后面才会用到的, 现在会用到的只有 `write` 和 `exit`

## 错误处理语言项

错误处理和之前也差不多

```rs
// app/src/lang_items.rs
use crate::exit;

#[panic_handler]
fn panic_handler(panic_info: &core::panic::PanicInfo) -> ! {
    let err = panic_info.message();
    if let Some(location) = panic_info.location() {
        println!(
            "Panicked at {}:{}, {}",
            location.file(),
            location.line(),
            err
        );
    } else {
        println!("Panicked: {}", err);
    }
    exit(-1);
}
```

## 基本系统调用

然后是系统调用的具体实现

```rs
// app/src/syscall.rs
use super::TimeVal;

pub const SYSCALL_WRITE: usize = 64;
pub const SYSCALL_EXIT: usize = 93;
pub const SYSCALL_SLEEP: usize = 101;
pub const SYSCALL_YIELD: usize = 124;
pub const SYSCALL_GETTIMEOFDAY: usize = 169;
pub const SYSCALL_TRACE: usize = 410;

pub fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret: isize;
    unsafe {
        core::arch::asm!(
            "ecall",
            inlateout("x10") args[0] => ret,
            in("x11") args[1],
            in("x12") args[2],
            in("x17") id
        );
    }
    ret
}

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
    syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}

pub fn sys_exit(exit_code: i32) -> ! {
    syscall(SYSCALL_EXIT, [exit_code as usize, 0, 0]);
    panic!("sys_exit never returns!");
}

pub fn sys_sleep(sleep_ms: usize) -> isize {
    syscall(SYSCALL_SLEEP, [sleep_ms, 0, 0])
}

pub fn sys_yield() -> isize {
    syscall(SYSCALL_YIELD, [0, 0, 0])
}

pub fn sys_get_time(time: &mut TimeVal, tz: usize) -> isize {
    syscall(SYSCALL_GETTIMEOFDAY, [time as *const _ as usize, tz, 0])
}

pub fn sys_trace(trace_request: usize, id: usize, data: usize) -> isize {
    syscall(SYSCALL_TRACE, [trace_request, id, data])
}
```

## 打印输出功能

```rs
// app/src/console.rs
use alloc::collections::vec_deque::VecDeque;
use alloc::sync::Arc;
use core::fmt::{self, Write};
use spin::mutex::Mutex;

pub const STDOUT: usize = 1;

const CONSOLE_BUFFER_SIZE: usize = 256 * 10;

use super::write;
use lazy_static::*;

struct ConsoleBuffer(VecDeque<u8>);

lazy_static! {
    static ref CONSOLE_BUFFER: Arc<Mutex<ConsoleBuffer>> = {
        let buffer = VecDeque::<u8>::with_capacity(CONSOLE_BUFFER_SIZE);
        Arc::new(Mutex::new(ConsoleBuffer(buffer)))
    };
}

impl ConsoleBuffer {
    fn flush(&mut self) -> isize {
        let s: &[u8] = self.0.make_contiguous();
        let ret = write(STDOUT, s);
        self.0.clear();
        ret
    }
}

impl Write for ConsoleBuffer {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        for c in s.as_bytes().iter() {
            self.0.push_back(*c);
            if (*c == b'\n' || self.0.len() == CONSOLE_BUFFER_SIZE) && -1 == self.flush() {
                return Err(fmt::Error);
            }
        }
        Ok(())
    }
}

#[allow(unused)]
pub fn print(args: fmt::Arguments) {
    let mut buf = CONSOLE_BUFFER.lock();
    buf.write_fmt(args);
}

#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}

#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    }
}

pub fn flush() {
    let mut buf = CONSOLE_BUFFER.lock();
    buf.flush();
}
```

## 链接脚本

我们将所有应用都加载到 `0x80400000` 来运行

```ld
// app/src/linker.ld
OUTPUT_ARCH(riscv)
ENTRY(_start)

BASE_ADDRESS = 0x0;

SECTIONS
{
    . = BASE_ADDRESS;
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }
    . = ALIGN(4K);
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }
    . = ALIGN(4K);
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }
    .bss : {
        start_bss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
        end_bss = .;
    }
    /DISCARD/ : {
        *(.eh_frame)
        *(.debug*)
    }
}
```

这个 crate 是一个 Lib crate, 所以我们其实是在为用户态程序实现一个简单的标准库

## 用户态程序

来看几个简单的应用, 我们会发现都有 `extern crate app;` 这一行用于链接到我们的 Lib

```rs
// app/bin/ch2b_hello_world.rs
#![no_std]
#![no_main]

#[macro_use]
extern crate app;

#[unsafe(no_mangle)]
fn main() -> i32 {
    println!("Hello, world from user mode program!");
    0
}
```

再给出两个简单应用

```rs
// app/bin/ch2b_bad_address.rs
// 这将尝试访问一个非法地址然后被内核杀死
#![no_std]
#![no_main]

extern crate app;

#[unsafe(no_mangle)]
pub fn main() -> isize {
    unsafe {
        #[allow(clippy::zero_ptr)]
        (0x0 as *mut u8).write_volatile(0);
    }
    panic!("FAIL: T.T\n");
}
```

```rs
// app/src/ch2b_power.rs
// 使用了循环和取模快速计算大数幂
#![no_std]
#![no_main]

#[macro_use]
extern crate app;

const LEN: usize = 100;

#[unsafe(no_mangle)]
fn main() -> i32 {
    let p = 3u64;
    let m = 998244353u64;
    let iter: usize = 200000;
    let mut s = [0u64; LEN];
    let mut cur = 0usize;
    s[cur] = 1;
    for i in 1..=iter {
        let next = if cur + 1 == LEN { 0 } else { cur + 1 };
        s[next] = s[cur] * p % m;
        cur = next;
        if i % 10000 == 0 {
            println!("power_3 [{}/{}]", i, iter);
        }
    }
    println!("{}^{} = {}(MOD {})", p, iter, s[cur], m);
    println!("Test power_3 OK!");
    0
}
```

然后使用一个简单的 py 脚本来帮我们快速的批量编译这些程序

```py
# app/build.py
import os
import subprocess

# 定义基础地址和步长
base_address = 0x80400000
step = 0x20000

app_dir = "src/bin"
elf_dir = "target/riscv64gc-unknown-none-elf/release"
bin_dir = "../bin"

os.makedirs(bin_dir, exist_ok=True)

app_id = 0
apps = os.listdir(app_dir)
apps.sort()

chapter = 2

for app in apps:
    app = app[: app.find(".")]

    if (chapter == 2 and app.startswith("ch2")) or (chapter == 3 and app.startswith("ch3")):
        os.system(
            "cargo rustc --bin %s --release -- -Clink-args=-Ttext=%x"
            % (app, base_address + step * app_id)
        )
        print(
            "[build.py]: application %s start with address %s"
            % (app, hex(base_address + step * app_id))
        )

        # 生成 ELF 文件的路径
        elf_file = app
        elf_path = os.path.join(elf_dir, elf_file)

        # 生成二进制文件的路径
        bin_file = elf_file + ".bin"
        bin_path = os.path.join(bin_dir, bin_file)

        # 使用 rust-objcopy 将 ELF 文件转换为二进制文件
        subprocess.run([
            "rust-objcopy",
            elf_path,
            "--strip-all",
            "-O", "binary",
            bin_path
        ], check=True)

        print(f"[build.py]: Converted {elf_file} to {bin_file}")

        # 如果是 chapter 3，增加 app_id
        if chapter == 3:
            app_id = app_id + 1

print("All files processed.")
```

执行的时候注意要在 `rcore/app` 路径下, 并且不能将 `app` 添加到 `rcore` 的工作空间

# 批处理系统实现

## 前置准备

首先新建下面几个模块

包装一下 `RefCell` 以便我们能安全的调用它

```rs
// src/sync/mod.rs

//! 同步和内部可变性基元

mod up;
pub use up::UPSafeCell;

// src/sync/up.rs

//! 内部可变性原语
use core::cell::{RefCell, RefMut};

pub struct UPSafeCell<T> {
    inner: RefCell<T>,
}

unsafe impl<T> Sync for UPSafeCell<T> {}

impl<T> UPSafeCell<T> {
    /// 应该保证只能用于单处理器
    pub unsafe fn new(value: T) -> Self {
        Self {
            inner: RefCell::new(value),
        }
    }
    /// 获取内部可变引用
    /// Panic: 如果数据已被借用
    pub fn exclusive_access(&self) -> RefMut<'_, T> {
        self.inner.borrow_mut()
    }
}
```

## 链接用户态程序到内核

在我们的 `rcore/src/main.rs` 嵌入的第一行全局汇编下面再加一行

```rs
core::arch::global_asm!(include_str!("link_app.S"));
```

然后创建这个文件, 这将用于链接应用程序, 这个后缀名的汇编会做一些预处理工作

```S
# src/link_app.S
    .align 3
    .section .data
    .global _num_app
_num_app:
    .quad 3
    .quad app_0_start
    .quad app_1_start
    .quad app_2_start
    .quad app_2_end

    .section .data
    .global app_0_start
    .global app_0_end
app_0_start:
    .incbin "bin/ch2b_hello_world.bin"
app_0_end:

    .section .data
    .global app_1_start
    .global app_1_end
app_1_start:
    .incbin "bin/ch2b_bad_address.bin"
app_1_end:

    .section .data
    .global app_2_start
    .global app_2_end
app_2_start:
    .incbin "bin/ch2b_power.bin"
app_2_end:
```

`.align` 表示对齐到 2 的 3 次方字节, 然后定义了应用数量和起始符号, 在下面具体嵌入了二进制程序, 具体路径可以根据实际情况填写

## 加载应用程序

新建一个 `batch` 模块用于管理应用程序

```rs
const MAX_APP_NUM: usize = 16;
const APP_BASE_ADDRESS: usize = 0x80400000;
const APP_SIZE_LIMIT: usize = 0x20000;

// 新建应用管理器并初始化一个实例
struct AppManager {
    num_app: usize,
    current_app: usize,
    app_start: [usize; MAX_APP_NUM + 1],
}

lazy_static::lazy_static! {
    // 这是之前的一个拥有内部可变性的容器, 在有多个可变引用的情况下报错, 这在我们的单核环境下够用了
    static ref APP_MANAGER: UPSafeCell<AppManager> = unsafe {
        UPSafeCell::new({
            // 获取 _num_app 符号, 即应用数量
            unsafe extern "C" {
                fn _num_app();
            }
            let num_app_ptr = _num_app as usize as *const usize;
            let num_app = num_app_ptr.read_volatile();
            let mut app_start: [usize; MAX_APP_NUM + 1] = [0; MAX_APP_NUM + 1];
            // 因为我们之前的汇编中将应用数量和应用起始地址放在了一起, 连续读取即可, 别忘了最后一个终止地址, 所以需要读应用数量 + 1 个
            let app_start_raw: &[usize] =
                core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1);
            app_start[..=num_app].copy_from_slice(app_start_raw);
            AppManager {
                num_app,
                current_app: 0,
                app_start,
            }
        })
    };
}

impl AppManager {
    pub fn print_app_info(&self) {
        println!("[kernel] num_app = {}", self.num_app);
        for i in 0..self.num_app {
            println!(
                "[kernel] app_{} [{:#x}, {:#x})",
                i,
                self.app_start[i],
                self.app_start[i + 1]
            );
        }
    }

    unsafe fn load_app(&self, app_id: usize) {
        if app_id >= self.num_app {
            println!("All applications completed!");
            crate::sbi::shutdown();
        }
        println!("[kernel] Loading app_{}", app_id);

        unsafe {
            // 清空所有应用程序数据
            core::slice::from_raw_parts_mut(APP_BASE_ADDRESS as *mut u8, APP_SIZE_LIMIT).fill(0);
            // 开始加载下一个应用程序
            let app_src = core::slice::from_raw_parts(
                self.app_start[app_id] as *const u8,
                self.app_start[app_id + 1] - self.app_start[app_id],
            );
            // 将应用程序数据复制到起始地址
            let app_dst =
                core::slice::from_raw_parts_mut(APP_BASE_ADDRESS as *mut u8, app_src.len());
            app_dst.copy_from_slice(app_src);
        }
        // 在应用程序执行完毕后, 加载下一个应用前, 手动清理指令缓存, 确保指令缓存与内存一致
        unsafe { core::arch::asm!("fence.i") };
    }

    pub fn get_current_app(&self) -> usize {
        self.current_app
    }

    pub fn move_to_next_app(&mut self) {
        self.current_app += 1;
    }
}
```

多数内容没什么好说的, 但可以看到加载应用的函数最后一行多了个汇编指令, 现代 CPU 通常会有指令缓存 Instruction Cache 用于加速指令的获取, 然而当程序动态加载新的指令到内存中时, 例如加载一个新的应用程序, 指令缓存中可能仍然保存着旧的数据, 如果没有刷新指令缓存, CPU 可能会继续执行缓存中的旧指令, 而不是从内存中加载的新指令

最后, 我们对外暴露几个用于初始化的函数, 到时候我们只需调用 `init` 函数即可完成应用程序管理器的初始化, 不过目前最后一个函数还有其他依赖项没有实现

```rs
pub fn init() {
    print_app_info();
}

pub fn print_app_info() {
    APP_MANAGER.exclusive_access().print_app_info();
}
```

## 特权级切换

为了隔离程序和内核, 我们引入了特权级机制, 而批处理系统在应用程序执行之前, 进行一些初始化工作, 并且监控应用程序的执行, 这些都涉及到了特权级的切换, 例如

- 启动应用程序时, 初始化应用的用户态上下文, 并能切换到用户态执行应用程序
- 应用程序发起系统调用后, 需要切换到内核态进行处理
- 应用程序执行出错时, 批处理操作系统要杀死该应用并加载运行下一个应用
- 应用程序执行结束时, 批处理操作系统要加载运行下一个应用

我们暂时只考虑用户态(U)程序触发中断并切换到监管态(S)的情况

下面是与特权级切换相关的 **控制状态寄存器 CSR**

- `sstatus` **(Supervisor Status Register)**: SPP 等字段给出中断 Trap 发生之前 CPU 处在哪个特权级等信息
    - SPP (Supervisor Previous Privilege): 记录 Trap 发生之前 CPU 所处的特权级。
        - 值为 0 表示 Trap 来自用户态 (U-mode),值为 1 表示 Trap 来自监管态 (S-mode)
    - SIE (Supervisor Interrupt Enable): 表示监管态下的中断是否启用
        - 当 Trap 发生时, SIE 会被清零, 禁用中断
    - SPIE (Supervisor Previous Interrupt Enable): 保存 Trap 发生之前的 SIE 值, 用于在 Trap 返回时恢复中断状态
    - 用途: 在 Trap 发生时, 保存当前状态; 在 Trap 返回时, 恢复状态
- `sepc` **(Supervisor Exception Program Counter)**: 当 Trap 是一个异常的时候，记录 Trap 发生之前执行的最后一条指令的地址
    - 当 Trap 是异常时, sepc 会保存触发异常的指令地址
    - 当 Trap 是中断时, sepc 会保存下一条即将执行的指令地址
    - 在 Trap 处理完成后, 可以通过 sepc 返回到原来的执行流
- `scause` **(Supervisor Cause Register)**: Trap 的原因
    - 最高位
        - `1`: 中断
        - `0`: 异常
    - 剩余位表示信息
- `stval` **(Supervisor Trap Value Register)**: Trap 附加信息
- `stvec` **(Supervisor Trap Vector Base Address Register)**: 控制 Trap 处理代码的入口地址
    - BASE: Trap 处理程序的基地址
    - MODE: Trap 处理模式
        - `0`: 直接模式, 所有 Trap 都跳转到 BASE
        - `1`: 向量模式, 根据 Trap 原因跳转到 BASE + 4 * cause, 但我们不使用
    - 用途: 在 Trap 发生时, CPU 会跳转到 stvec 指定的地址执行 Trap 处理程序

特权级切换的一部分过程由硬件直接完成

例如从用户态陷入内核态时, CPU 检测到 Trap

- sstatus 的 SPP 字段会被修改为 CPU 当前的特权级
- sepc 会被修改为 Trap 处理完成后默认会执行的下一条指令的地址
- scause/stval 分别会被修改成这次 Trap 的原因以及相关的附加信息
- CPU 会跳转到 stvec 所设置的 Trap 处理入口地址, 并将当前特权级设置为 S, 然后从Trap 处理入口地址处开始执行

而当 CPU 完成 Trap 处理准备返回的时候, 需要通过一条 S 特权级的特权指令 `sret` 来完成, 这一条指令具体完成以下功能:

- CPU 会将当前的特权级按照 sstatus 的值恢复状态
- 将 sepc 的值写入程序计数器(PC), 返回到 Trap 发生前的执行流

## 用户栈与内核栈

在 Trap 触发时, CPU 会切换到 S 特权级并跳转到 stvec 所指示的位置, 但是在正式进入 S 特权级的 Trap 处理之前, 我们必须保存原控制流的寄存器状态, 这一般通过栈来完成. 但我们需要用专门为操作系统准备的内核栈, 而不是应用程序运行时用到的用户栈. 我们只需简单包装两个数组

```rs
// src/batch.rs
const USER_STACK_SIZE: usize = 4096 * 2;
const KERNEL_STACK_SIZE: usize = 4096 * 2;

#[repr(align(4096))]
struct KernelStack {
    data: [u8; KERNEL_STACK_SIZE],
}

#[repr(align(4096))]
struct UserStack {
    data: [u8; USER_STACK_SIZE],
}

// 以全局变量形式存在 BSS 段
static KERNEL_STACK: KernelStack = KernelStack {
    data: [0; KERNEL_STACK_SIZE],
};
static USER_STACK: UserStack = UserStack {
    data: [0; USER_STACK_SIZE],
};

impl KernelStack {
    // 获取栈顶指针
    // 栈的顶部地址 = 栈的起始地址 + 栈的大小
    fn get_sp(&self) -> usize {
        self.data.as_ptr() as usize + KERNEL_STACK_SIZE
    }
    pub fn push_context(&self, cx: TrapContext) -> &'static mut TrapContext {
        // 当前栈顶减去 TrapContext 的大小, 得到新的栈顶, 即 TrapContext 的指针
        let cx_ptr = (self.get_sp() - core::mem::size_of::<TrapContext>()) as *mut TrapContext;
        unsafe {
            *cx_ptr = cx;
        }
        unsafe { cx_ptr.as_mut().unwrap() }
    }
}

impl UserStack {
    fn get_sp(&self) -> usize {
        self.data.as_ptr() as usize + USER_STACK_SIZE
    }
}
```

定义 Trap 上下文

```rs
// src/trap/context.rs
use riscv::register::sstatus::{self, SPP, Sstatus};

#[repr(C)]
pub struct TrapContext {
    /// 通用寄存器 x[0..31]
    pub x: [usize; 32],
    pub sstatus: Sstatus,
    pub sepc: usize,
}
```

为什么需要保存所有通用寄存器

- 特权级切换
    - 应用程序控制流和内核控制流运行在不同的特权级 (用户态和内核态)
    - 特权级切换时, 寄存器的值可能会被覆盖, 因此需要保存
- 编程语言差异
    - 应用程序和内核可能由不同的编程语言编写, 调用约定可能不同
    - 保存所有寄存器可以避免因调用约定不同而导致的问题
- Trap 处理复杂性
    - Trap 处理过程中可能会调用多个模块, 难以确定哪些寄存器会被使用
    - 为了确保安全, 保存所有寄存器是最稳妥的做法

例外情况

- x0 寄存器
    - 硬编码为 0, 值不会改变, 无需保存
    - 但仍为其预留空间, 以便统一处理
- tp (x4) 线程寄存器
    - 除非手动使用, 否则一般不会被用到
    - 但仍为其预留空间, 以便统一处理

控制状态寄存器

- scause 和 stval: 在 Trap 处理的第一时间就会被使用或保存, 因此它们的值不会被覆盖或丢失, 无需在 TrapContext 中保存
- sstatus 和 sepc: 在 Trap 处理的全程都有意义. 在 Trap 嵌套的情况下, 它们的值可能会被覆盖. 因此，需要在 TrapContext 中保存, 并在 Trap 返回时恢复

其他 CSR 交给硬件自行更新

实现给用户态使用的系统调用

```rs
// src/syscall/mod.rs

//! 用户态程序通过 ecall 指令触发系统调用
//! 处理器会产生一个 "U-mode 环境调用" 异常
//! 异常由内核中的 trap_handler 处理, 并最终调用 syscall() 函数

const SYSCALL_WRITE: usize = 64;
const SYSCALL_EXIT: usize = 93;

mod fs;
mod process;

use fs::*;
use process::*;

pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
    match syscall_id {
        SYSCALL_WRITE => sys_write(args[0], args[1] as *const u8, args[2]),
        SYSCALL_EXIT => sys_exit(args[0] as i32),
        _ => panic!("Unsupported syscall_id: {}", syscall_id),
    }
}

// src/syscall/fs.mod

const FD_STDOUT: usize = 1;

/// 将长度为 'len' 的 buf 写入文件描述符为 'fd' 的文件中
pub fn sys_write(fd: usize, buf: *const u8, len: usize) -> isize {
    log::trace!("kernel: sys_write");
    match fd {
        FD_STDOUT => {
            let slice = unsafe { core::slice::from_raw_parts(buf, len) };
            let str = core::str::from_utf8(slice).unwrap();
            crate::print!("{}", str);
            len as isize
        }
        _ => {
            panic!("Unsupported fd in sys_write!");
        }
    }
}

// src/syscall/process.rs

use crate::batch::run_next_app;

/// 任务退出并返回退出码
pub fn sys_exit(exit_code: i32) -> ! {
    log::trace!("[kernel] Application exited with code {}", exit_code);
    run_next_app()
}
```