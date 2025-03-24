---
feature: false
title: rCore 之旅(3)|分时多任务系统
date: 2025-03-20 19:00:00
abstracts: 对于一个批处理系统, 每次运行程序都需要先加载程序到指定位置, 执行程序, 卸载程序, 然后加载下一个程序. 这未免有些麻烦了, 而且还会有装载卸载程序的额外开销, 所以在这一节, 我们来实现一个分时多任务系统, 它能并发地执行多个用户程序
tags:
    - OS
    - rCore
    - Rust
categories:
    - Course
cover: https://fontlos.com/icons/logo.png
---

对于一个批处理系统, 每次运行程序都需要先加载程序到指定位置, 执行程序, 卸载程序, 然后加载下一个程序. 这未免有些麻烦了, 而且还会有装载卸载程序的额外开销, 所以在这一节, 我们来实现一个分时多任务系统, 它能并发地执行多个用户程序, 包含以下功能

- 一次性加载所有用户程序, 减少任务切换开销
- 支持任务切换机制, 保存切换前后程序上下文
- 支持程序主动放弃处理器, 实现 yield 系统调
- 以时间片轮转算法调度用户程序, 实现资源的时分复用

并且随着我们的 OS 慢慢增大, 暂时无法同时展示出所有代码了, 有机会的话, 可以拆分成更多章节然后写上完整流程

# 多个应用程序同时加载

在之前的批处理系统, 每个应用程序都用同一个地址, 分先后进行装载卸载, 如果要实现一次性加载多个应用程序, 那么就不能使用同样的地址了, 不过事实上我们在上一节给出的 `build.py` 已经解决了这个问题, 把 `chapter` 变量设置为3, 就会让每个程序的起始地址递增. 不过单个应用程序不能太大, 并且 Qemu 本身也不支持同时加载太多程序

首先我们不需要之前的 `batch` 了, 将其拆分为 `loader` 和 `task` 分别负责加载所有程序和调度这些程序

我们不需要之前的 `AppManager` 了, 不过要做的事也和之前差不多

```rs
/// 获取指定索引的 app 基地址, 这两个常数定义在 config 模块, 值和 build.py 脚本相同
fn get_base_i(app_id: usize) -> usize {
    APP_BASE_ADDRESS + app_id * APP_SIZE_LIMIT
}

/// app 数量
pub fn get_num_app() -> usize {
    unsafe extern "C" {
        fn _num_app();
    }
    unsafe { (_num_app as usize as *const usize).read_volatile() }
}

/// 加载 app
pub fn load_apps() {
    unsafe extern "C" {
        fn _num_app();
    }
    let num_app_ptr = _num_app as usize as *const usize;
    let num_app = get_num_app();
    let app_start = unsafe { core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1) };
    for i in 0..num_app {
        let base_i = get_base_i(i);
        // 清理对应区域
        (base_i..base_i + APP_SIZE_LIMIT)
            .for_each(|addr| unsafe { (addr as *mut u8).write_volatile(0) });
        // 加载应用
        let src = unsafe {
            core::slice::from_raw_parts(app_start[i] as *const u8, app_start[i + 1] - app_start[i])
        };
        let dst = unsafe { core::slice::from_raw_parts_mut(base_i as *mut u8, src.len()) };
        dst.copy_from_slice(src);
    }

    unsafe {
        asm!("fence.i");
    }
}

/// 为用户程序初始化一个中断上下文
pub fn init_app_cx(app_id: usize) -> usize {
    KERNEL_STACK[app_id].push_context(TrapContext::app_init_context(
        get_base_i(app_id),
        USER_STACK[app_id].get_sp(),
    ))
}
```

# 任务切换

这是现代操作系统的核心机制之一, 即让运行的程序主动交出 CPU 的使用权, 而内核要做的就是, 执行另一个程序, 并保存这些程序的运行时上下文, 以便恢复程序运行

任务切换不涉及特权级的切换, 部分工作可以由编译器完成而非完全由硬件内核完成. 并且这对应用是透明的, 应用程序无需感知这些切换

任务切换实际上是两个不同应用程序在内核中的 Trap 控制流之间的切换. 当一个应用程序通过 Trap 进入内核态时, 内核可以调用一个特殊的 `__switch` 函数来切换任务

在 `__switch` 函数执行期间, 当前的任务控制流 A 会被暂停并切换出去, CPU 转而执行另一个任务的控制流 B, 类似递归, 当 `__switch` 返回时, 控制流 A 最终会从某个控制流 C 切换回来继续执行

任务切换需要保存和恢复 CPU 的某些寄存器, 这些寄存器构成了 **任务上下文**, 主要包括:
- `ra`: 返回地址, 记录了 `__switch` 函数返回后应该跳转到的地址。
- `sp`: 栈指针, 指向当前任务的栈
- `s0~s11`: 被调用者保存的寄存器, 这些寄存器在函数调用时需要由被调用者保存和恢复

结构如下

```rs
pub struct TaskContext {
    ra: usize,
    sp: usize,
    s: [usize; 12],
}
```

下面给出 `__switch` 函数的实现, 实际传入的参数就是当前和要切换的 TaskContext, 通过 a0 和 a1 寄存器传入

```S
# src/switch.S
# 同样定义两个宏, 用于保存和恢复寄存器
.altmacro
.macro SAVE_SN n
    sd s\n, (\n+2)*8(a0)
.endm
.macro LOAD_SN n
    ld s\n, (\n+2)*8(a1)
.endm
    .section .text
    .globl __switch
__switch:
    # 当前任务栈指针 sp 保存到 a0+8的地址
    sd sp, 8(a0)
    # 保存任务返回地址
    sd ra, 0(a0)
    # 保存关键寄存器
    .set n, 0
    .rept 12
        SAVE_SN %n
        .set n, n + 1
    .endr
    # 从 a1 恢复下一个任务的上下文
    ld ra, 0(a1)
    .set n, 0
    .rept 12
        LOAD_SN %n
        .set n, n + 1
    .endr
    # 从 a1+8 恢复下一个任务的栈指针
    ld sp, 8(a1)
    # 返回到下一个任务执行流
    ret
```

可以看到这也是由汇编实现的, 这主要由于我们需要精准操作寄存器, 避免编译器优化干扰打乱任务切换逻辑. 而且这个函数的功能是切换到另一个任务的上下文, 而不是简单地调用一个函数. 这种操作需要直接修改栈指针和返回地址, 而 Rust 的函数调用机制无法实现这种底层控制

然后我们使用 Rust 代码包装这个函数

```rs
use super::TaskContext;
use core::arch::global_asm;

global_asm!(include_str!("switch.S"));

unsafe extern "C" {
    // 保存当前, 切换到下一个
    pub fn __switch(current_task_cx_ptr: *mut TaskContext, next_task_cx_ptr: *const TaskContext);
}
```

# 多任务调度

