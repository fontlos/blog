---
feature: false
title: 初识 Rust(4) | 作用域, 所有权机制, 函数与返回值, 复合类型, Module 与可见性
date: 2021-06-08 12:10:27
abstracts: 回顾作用域, 初步了解作用域和堆栈相关知识, 熟悉 Rust 首创的所有权机制. 函数的初步认识. 介绍一下复合类型, 结构体, 枚举. 最后了解一下包, 模块, 以及结构体和枚举内部成员的可见性, 以及如何公开并引入成员
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

# 变量作用域

这是一个简单的概念, 在绝大多数编程语言中都有作用域, 且几乎相同, 让我们通过一个小例子来理解

```rust
{ // a 在这里尚未声明
    let a = 0; // a 的作用域从这里开始
    { // b 在这里尚未声明, a 在这里仍然有效
        let b = 0; // b 的作用域从这里开始
        // 使用 b
    } // b 的作用域到此结束, 不再有效
    // 使用 a
} // a 的作用域到此结束, 不再有效

```

# 所有权机制

程序的本质就是计算机按照一定的规则来操作内存, 如何申请新的内存, 释放不必要的内存, 保证需要的内存时刻可用, 成为了所有编程语言的重中之重

如何管理内存, 在编程语言的发展中摸索出了两条主流做法:

1. **手动管理内存**, 如 C/C++, 需要通过函数调用来申请和释放内存, 心智负担最大, 但性能最高, 新生的 Zig 语言更是一个典型, 甚至没有隐式的内存分配
2. **GC (垃圾回收机制)**,如 Java, 新生的 Go, 通过在程序运行期间不断寻找不再需要的内存并释放, 心智负担最小, 但性能一般较低

而 Rust 语言探索出了第三条方法, 就是我们要介绍的 **所有权机制**: 通过指定一系列规则, 让编译器在编辑期就做好绝大多数内存安全检查, 对于初学者心智负担较大, 但这种检查只发生在编译期, 因此对于程序运行时没有任何性能损失

由于这是一个新的概念, 无论有无编程基础都需要一段时间来习惯, 而一旦掌握这些规则, 写起来会越发顺手, 海阔天空

## 一段内存不安全的 C 代码

```c
int* return_one() {
    int one = 1; // one 作用域的开始
    char *hello = "hello"; // hello 作用域的开始
    return &one; // 尝试返回一个在函数内部创建的值得引用
} // one 和 hello 的作用域结束, 内存被回收销毁
```

虽然这段代码可以被编译, 但事实上充满了内存安全问题

函数内部创建 `one` 这个变量并存储在 **栈** 上, 但栈上的内存在离开作用域后会被系统回收, 而最后函数返回 `one` 的地址, 这段地址将会指向一片未知的空间, 这就是 **Dangling Pointer (悬垂引用)**, 最后获得的 `one` 究竟是什么是无法预知的

随后, 变量 `hello`, 这这常量字符串将会被编码到常量区, 但这个字符串并没有任何作用, 可位于常量区的内存将会在整个程序结束后才被回收

根据微软公开的报告, 近 **70%** 的 Bug 都是 **内存安全问题**, 可见这个问题影响之深远.

但随着 Rust 的出现, 这类问题几乎可以被杜绝, 这到底是如何做到的呢

## 预备知识: Stack and Heap (堆栈)

堆栈是内存的两种不同形式, 在大多数现代语言中无需了解, 但 Rust 作为一门偏底层的系统级语言, 以及为了更好的了解所有权机制, 了解堆栈, 知道内存分配在何处是十分重要的, 这将直接影响程序的性能

### 栈

存储在栈中的内存 **先进后出** 就像在一根螺丝上拧上一个个螺母, 不可能在最后拧上的螺母被取下之前去除螺丝上的第一个螺母

向栈中存入数据称为 **进栈或压栈 (Push)**, 取出数据称为 **出栈 (Pop)**

栈要求每一个数据的大小都是已知且固定的, 以方便顺序压入和取出

### 堆

堆主要用于弥补栈的不足, 用于存储大小未知, 或者可能发生改变的数据

存入数据时, 堆会寻找一块足够大的空间, 将其标记为已使用, 并返回一个指向这里的 **内存地址 (也称指针 Pointer)**, 这个过程称为 **分配 (Allocating)**, 随后将返回的内存地址压入栈中, 后续通过访问栈中的指针来访问实际内存

### 性能对比

很容易发现, 栈是一种更高效的内存分配方式:

- 写入数据: 因为每次压入新的数据无需分配新的空间, 只需要在栈顶进行操作即可, 而堆分配需要寻找空间, 标记空间, 为下一次分配作准备等等
- 读取数据: 得益于 CPU 高速 Cache, 栈上的内存多数时候可以直接存放到 Catch, 减少 CPU 对内存的直接访问, 有时可以带来数十倍的性能差距. 而堆内存只能存储在内存中, 而且需要先访问栈获得指针才能使用

### 总结

在栈中分配的内存, 在函数调用时顺序压入, 在函数调用结束时逆序弹出, 内存失效, 此时再进行引用操作就会产生悬垂引用

而堆上的内存缺乏组织, 因此对堆内存的追踪管理时十分重要的, 否则将会产生 **内存泄漏**, 导致一部分内存永远无法通过程序自身进行回收

而 Rust 的所有权机制就为解决以上这些问题提供了强大的保障

最后, 再大多数语言, 以及 Rust 开发中, 都不是必须理解堆栈的原理

但理解这些, 对我们 **剖析所有权** 的工作原理有很大的帮助

## 所有权原则

Rust 为了实现所有权内存管理, 在编译器层面制定了三条规则:

> 1. 每个值都被一个变量所拥有, 称为 Owner
> 2. 每个值在同一时间仅能被一个 Owner 所拥有
> 3. 当Owner 离开作用域时, 这个值将被 **Drop (丢弃)**

这里我们先用 `String` 类型进行举例, 详细内容将会在后面的文章介绍

Rust 中最常用的字符串之一 `&str` 我们已经见过了, 它将被硬编码到程序里. 字符串字面量很好用, 但也有一些缺陷, 比如, 它无法被改变

这种时候就需要可变长度字符串 `String` 类型了

可以通过标准库中的函数来创建它

```rust
let hello = String::from("Hello");
```

下面让我们以 `String` 为基础了解一下变量绑定背后的事

### 转移所有权

先来看看原生类型

```rust
let a = 1;
let b = a;
```

很简单, 首先将值 `1` 绑定给 `a`, 接下来 **Copy (拷贝)** `a` 的值并绑定给 `b`, 这样 `a` 和 `b` 的值均为 `1`

因为原生类型都是大小固定的简单值, 在后面的文章我们会了解到他们都实现了 `Copy trait`, 因此这两个变量是通过自动拷贝进行变量绑定的, 整个过程完全在栈上完成. 对这种简单值得拷贝并不会带来性能影响而且速度非常快, 比如示例中得 `i32` 类型, 只需要复制 4 个字节得内存即可

接下来看看 `String` 类型

```rust
let a = String::from("Hello");
let b = a;
```

看起来似乎和上面完全一样, 但实际上背后的工作流程完全不同

`String` 是一个复杂类型, 为了实现可变, 必须被分配到堆上, 因此这个类型由 **指向堆内存得指针, 字符串长度, 字符串容量** 组成

显而易见, 如果要完全拷贝 `String` 与 **,实际储存在堆上的字节数组**, 涉及到内存分配, 对性能会带来很大的影响

但如果只拷贝 `String` 类型本身, 那么就只需要拷贝以上三个内存大小固定且已知得量即可, 方便快捷, 所以 Rust 毫无疑问的选择了这一种做法

但这带来一个新的问题, 想想所有权原则的第二条, 那么此时, 将会有两个指针指向同一片堆内存, 即一个值有了两个 Owner. 但是这又如何呢

想象一下, 如果这件事发生了, 当 `a` 离开作用域时, 对应的内存被 Drop, 而当 `b` 离开作用域时, 程序会尝试在已被 Drop 的内存上再次进行 Drop, 这被称为 **Double Free (二次释放)** 错误, 同样可能导致内存安全问题

因此, Rust 的解决方式是, 将 `a` 赋值给 `b` 的同时, 将 `a` 手中的 **所有权转交给** 了 `b`, 此时 `a` 不再有效, 原来的值只有 `b` 一个 Owner, 所以以下的代码无法运行

```rust
let a = String::from("Hello");
let b = a;
println!("{}, world!", a);
```

如果你有一些编程基础, 可能听过类似 **Shallow Copy (浅拷贝)** 和 **Deep Copy (深拷贝)**, 那么 `String` 类型的这种转移所有权的机制看起来有点像浅拷贝, 但是别忘了在 `a` 赋值给 `b` 后 `a` 的值就失效了, 因此这里我们用一个更形象的说法, 将这个操作称之为 **Move (移动)**

了解了这些, 就应该能更清楚的明白为什么 Rust 称呼 `let valuable = value` 为 **绑定** 而非 **赋值** 了吧

而事实上, Rust 中也有与深拷贝和浅拷贝对应的概念

### 克隆

**Clone** 对应着一般意义上的深拷贝, Rust 永远不会自动执行这一过程, 深拷贝必须是显示的, 比如 `String` 类型如果真的有必要进行深度复制, 可以使用 `clone` 方法, 当然, 这涉及到高昂的内存分配, 请谨慎使用

```rust
let a = String::from("Hello");
let b = a.clone();
println!("a = {}, b = {}", a, b);
```

对于其他类型的深拷贝, 可能需要实现 `Clone trait`

### 拷贝
**Copy** 对应着一般意义上的浅拷贝, 这只发生在栈上, 因此性能很高

让我们再看一遍字符串字面量

```rust
let a = "Hello";
let b = a;
println!("{}, world!", a);
```

如果参考之前的 `String` 的例子, 没有调用 `clone` 方法那这段代码也应该报错才是

但这和之前的例子有一个本质上的区别: 在 `String` 的例子中 `a` 是持有所有权的, 而这个例子中 `a` 只是引用了硬编码在二进制中的字符串 `"Hello"`, 并没有持有所有权

因此 `let b = a` 中, 仅仅是对该引用进行了拷贝, 此时 `a` 和 `b` 都引用了同一个字符串

能进行拷贝的值都需要实现 `Copy trait`, 实现了这个 Trait 的类型主要有以下这些: 任何基本类型的组合, 不需要分配内存或某种形式资源的类型

具体如下:
1. 所有整数类型, 如 `i32`
2. 布尔类型 `bool`
3. 所有浮点数类型, 如 `f64`
4. 字符类型 `char`
5. 元组, 当且仅当其包含所有元素的类型也都是 Copy 的时候
6. 不可变引用 `&T` ,例如上面的最后一个例子, 但是注意, 可变引用 `&mut T` 是不可以 Copy 的

### 引用与借用

在上面的例子中, 我们提到了一个概念 --- **引用**

事实上, 如果仅仅支持通过移动所有权来获得一个值, 程序将变得更加复杂, 所幸, Rust 也像大多数语言那样, 提供了对某个变量的引用. 只不过, 在 Rust 中, 我们更习惯将它称为 **Borrowing (借用)**, 就像字面意思, **有借有还**, 而且 **借** 能发生的前提是这个量是有主人的

通过 `&` 运算符可以获得一个借用, 或者说常规意义上的 **引用**, 这是一个指针类型, 指向了对象存储的内存地址, **引用仅仅允许使用该值, 而没有所有权**

引用默认是不可变的, 离开作用域不会导致值的 Drop

```rust
let x = 0;
{
    let y = &x;
    // Do something with y
} // y 离开作用域但无事发生
// x 仍然可用
// assert_eq 宏用于判断两个变量是否完全相同
assert_eq!(0, x);
```

Rust 中还提供了一个专门的 **Reference (引用)** 关键字 `ref`, 这与 `&` 有什么区别呢? 举一个例子, 就像我们写作文引用名人名言那样, 有的时候实在想不出来了, 那就引用一个不存在的名人吧 (你说对吧, 沃斯机所得先生). `ref` 就允许在对象被声明之前先获得一个引用

```rust
// 下面就展示了两者的区别
let ref a: i32;
a = &3;
let b = &2;
```

但是需要注意一点, `ref` 只能用于变量声明, 不能用来声明类型, 比如 `&i32` 是一个合法的引用类型, 但 `ref i32` 不是

事实上, 和 `ref` 与接下来要提到的 `*` 一样, `&` 同样可以用于变量声明, 比如:

```rust
// &a 这个整体是一个 &i32 类型
let &a = 0;
```

关于 `ref` 与 `&` 的细节问题可以看下面这个例子

```rust
// 注意, 目前这段代码只能在 Nightly 版本运行
#![feature(core_intrinsics)]

fn main() {
    let x = &false;
    print_type_name_of(x);

    let &y = &false;
    print_type_name_of(y);

    let ref z = &false;
    print_type_name_of(z);
}

fn print_type_name_of<T>(_: T) {
    println!("The type of the value is {}", unsafe { std::intrinsics::type_name::<T>() })
}
```

运行结果如下:

```
The type of the value is &bool
The type of the value is bool
The type of the value is &&bool
```

其中第二条是最有趣的, 可以理解为 **y 的引用是一个 Bool 类型的引用, 所以 y 是 Bool 类型**, 似乎也有一点解引用的味道

小小的总结一下:
> 作用于表达式, `&` 表示借用, `ref` 无效
> 作用于变量绑定, `&` 类似于下面会提到的 `*`, `ref` 表示引用类型
> 作用于类型声明, `&` 表示引用类型, `ref` 无效
> 作用于模式匹配, `&` 无效, `ref` 表示引用类型 (这将会在后面的文章提到)

### 解引用

可以通过 `*` 运算符进行解引用, 来访问原始对象

```rust
let x = 0;
let y = &x;
assert_eq!(0, *y);
```

### 可变引用

通过 `&mut` (或者 `ref mut`) 来获得一个可变引用, 当然, 被引用的值也应该是可变的

```rust
let mut a = 1;
let b = &mut a;
*b = 2
println!("{}", b);
```

但要注意一点, 在 **不存在可变引用** 的情况下不可变引用可以存在 **任意多个**

而一旦存在可变引用, 就 **不能存在** 不可变引用, 且 **同一个作用域** 只能有一个可变引用

前一个很好理解, 毕竟我们也不希望一个不可变的引用在某处突然莫名其妙的被修改了

后一个对新手来说也是一个重难点, 被称为 **Borrow Checker (编译器借用检查机制)**

这种限制的好处就是使 Rust 在编译期就避免数据竞争, 数据竞争可由以下行为造成:
- 两个或更多的指针同时访问同一数据
- 至少有一个指针被用来写入数据
- 没有同步数据访问的机制

数据竞争会导致 **UB(未定义行为)**, 这种行为很可能超出我们的预期, 难以在运行时追踪, 并且难以诊断和修复, 所有 Rust 为了避免了这种情况直接拒绝编译存在数据竞争的代码

但有的时候, 出于一些需要, 可能需要存在对变量的可变引用和不可变引用, 只要这两个不是同时发生就好

> 在 Rust 1.31 以后, 变量的作用域持续于整个 `{}`, 而引用不同, 在最后一次被使用后即离开作用域

```rust
let mut s = String::from("hello");

let ref1 = &s;
println!("{}, world!", r1);
// 在 Rust 1.31 以后, r1 作用域在这里结束, 所以下面我们可以创建一个可变引用

let r2 = &mut s;
println!("{}, world!", r2);
```

这种编译器优化行为被称为 **Non-Lexical Lifetimes (NLL)**, 专门用于找到某个引用在作用域结束前就不再被使用的代码位置

或者需要多个可变引用, 就像上面提到的 "同一作用域", 我们可以通过 `{}` 开辟新的作用域来实现这个需求

```rust
let mut hello = String::from("hello");

{
    let ref1 = &mut s;

} // r1 在这里被 Drop, 所以我们可以创建一个新的可变引用

let ref2 = &mut s;
```

### 悬垂引用

在本文的开头就提到了这个概念. Rust 编译器的 **借用检查** 可以确保引用永远也不会变成悬垂状态: 当你获取数据的引用后, 编译器会检查以确保数据不会在引用结束前被释放. 要想释放数据, 必须先停止其引用的使用

```rust
let reference_to_nothing = {
    let strs = String::from("hello");
    // 下面这一行无法运行!
    &strs;
    // 改成这样移动所有权才可以
    // strs
}
```

### 总结

> 1. 每个值都被一个变量所拥有, 称为 Owner
> 2. 每个值在同一时间仅能被一个 Owner 所拥有
> 3. 当Owner 离开作用域时, 这个值将被 **Drop (丢弃)**
> 4. 同一时刻, 对一个值要么拥有一个可变引用, 要么拥有任意个不可变引用

介绍完所有权, 就方便我们介绍函数与一些复杂的类型了

# 函数

## 函数签名与声明

```rust
fn function_name(arg: arg_type, ...) -> return_type{
    body
}
```

函数通过 `fn` 关键字声明, 后跟一个函数名称, 采用蛇形命名法, 小括号是 **必须** 的, 内部的参数是可选的, 参数需要有参数名称和参数类型, 多个参数通过 `,` 隔开. `-> return_type` 表示返回一个值, 同样是可选的, 不需要名称, 只需要注明类型, 但注意, 一次 **最多只能返回一个值**, 多个返回值可以通过 **返回元组** 来实现, 最后必须跟上一个 `{}` 函数体

函数声明的位置随意, 全局甚至在另一个函数内部, 即使在调用函数后才声明也无所谓, 只要有定义即可, Rust 不关心我们把函数放在哪

函数默认会将最后一条表达式返回, 如果有需要也可以通过 `return` 关键字在任意位置提前返回

```rust
fn return_one() -> i32 {
    let one = 1;
    one
}

fn return_two() -> i32 {
    return 2;
    println!("This will print nothing");
}
```

## 函数调用

函数调用很简单, 只需要函数名后跟小括号, 内部按顺序传入参数即可

```rust
fn add(num1: i32, num2: i32) -> i32 {
    num1 + num2
}

let two = return_one(1, 1);
```

## 特殊的返回类型

### 无返回值

实际上对于每一个无返回值的函数, 都隐式的返回了一个 **单元类型()**, 本质是一个零长度的元组, 没有任何作用, 仅仅表示一个函数没有返回值

实际上以分号 `;` 结尾的表达式同样隐式的返回了这个类型

```rust
fn return_nothing(){
    println!("Return nothing");
}
// 显式指定, 但谁会闲得无聊这样做呢
fn return_nothing() -> () {
    println!("Return nothing");
}
```

### 永不返回的发散函数
当用 `!` 作函数返回类型的时候, 表示该函数永不返回 (diverge function), 这种语法往往用做会导致程序崩溃的函数

```rust
fn dead_end() -> ! {
    panic!("Never return");
}
```

## 函数传值与所有权

将值传入函数传出一样会发生 **移动** 或 **复制**

```rust
fn main() {
    let strs = String::from("hello"); // strs 进入作用域
    takes_ownership(strs); // strs 的所有权移动到函数里
    // strs 不再有效
    let num = 0; // num 进入作用域
    copy(num); // num 应该移动函数里, 但 i32 实现了 Copy 所以在后面可继续使用 num

} // 这里 num 先被 Drop, 然后是 strs, 但 strs 的所有权已被移走

fn takes_ownership(strs: String) { // strs 进入作用域
    println!("{}", strs);
} // strs 被 Drop

fn copy(num: i32) { // num 进入作用域
    println!("{}", some_integer);
} // num 被 Drop
```

返回值也同理

```rust
fn main() {
    let str1 = gives_ownership(); // gives_ownership 将返回值所有权移给 str1, str1进入作用域
    // str1 的所有权被移动到 takes_and_gives_back 中, 且也将返回值移给 str2
    let str2 = takes_and_gives_back(str1);
} // str1 已被移出, str2 被Drop

fn gives_ownership() -> String {
    let strs = String::from("hello"); // strs 进入作用域.
    strs // 返回 strs 并移出所有权给函数调用者
}

fn takes_and_gives_back(strs: String) -> String { // strs 进入作用域

    strs  // 返回 strs 并移出所有权给函数调用者
}
```

但这样总是把值传来传去会让语法变得啰嗦, 很不优雅, 这时我们可以传递引用

```rust
fn main() {
    let mut strs = String::from("hello");

    let len = calculate_length(&strs); // &strs 作用域结束

    println!("The length of '{}' is {}.", strs, len);

    change(&mut strs);

    println!("{}", strs);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

## 匿名函数

Rust 使用 **Closure (闭包)** 创建匿名函数

```rust
let out = 12; // 闭包可使用外部变量
let num1 = |i,j|i+j+out; // 闭包可根据上下文自动推导类型
let num2 = |i:i32,j:i32|i*j+out; // 也可以指定类型
let a = 1;
let b = 2;
let c = num1(a,b);
let d = num2(a,b);
println!("{},{}",c,d);
```

其中 `||` 之间是闭包的参数, 其后是函数的主体, 闭包 `num1` 借用了它作用域中的 `let` 绑定 `out`. 如果要让闭包获得所有权, 可以使用 `move` 关键字

```rust
let mut num = 5;

{
    let mut add = move |x: i32| num += x;   // 闭包通过 move 获取了 num 的所有权
    add(5);
}
// 下面的 num 在被 move 之后还能继续使用是因为其实现了 Copy trait
assert_eq!(5, num);
```

# 高阶函数

Rust 支持高阶函数, 允许闭包作为参数

```rust
fn add_one(x: i32) -> i32 { x + 1 }

fn apply<F>(f: F, y: i32) -> i32 // 接受一个类型F与i32, 返回i32
    where F: Fn(i32) -> i32// 对类型 F 的约束
{
    f(y) * y // 函数体
}

fn factory(x: i32) -> Box<Fn(i32) -> i32> {
    //返回一个函数
    Box::new(move |y| x + y)
}

fn main() {
    let transform: fn(i32) -> i32 = add_one;//函数指针
    let f0 = add_one(2i32) * 2;
    let f1 = apply(add_one, 2);
    let f2 = apply(transform, 2);
    println!("{}, {}, {}", f0, f1, f2);//这三个是相等的

    let closure = |x: i32| x + 1;
    let c0 = closure(2i32) * 2;
    let c1 = apply(closure, 2);
    let c2 = apply(|x| x + 1, 2);
    println!("{}, {}, {}", c0, c1, c2);

    let box_fn = factory(1i32);
    let b0 = box_fn(2i32) * 2;
    let b1 = (*box_fn)(2i32) * 2;
    let b2 = (&box_fn)(2i32) * 2;
    println!("{}, {}, {}", b0, b1, b2);

    let add_num = &(*box_fn);
    let translate: &Fn(i32) -> i32 = add_num;
    let z0 = add_num(2i32) * 2;
    let z1 = apply(add_num, 2);
    let z2 = apply(translate, 2);
    println!("{}, {}, {}", z0, z1, z2);
}
```

# 复合类型

## 结构体

**Struct** 是一种记录类型, 所包含的每个 **Field (域)** 都有一个名称, 每个结构体也都有一个名称, 通常以 **大写字母** 开头, 使用 **驼峰命名法**. 元组结构体是由 **元组** 和 **结构体** 混合构成, 元组结构体有名称, 但是它的域没有. 当元组结构体只有一个域时, 称为**New Type (新类型)**. 没有域的结构体, 称为类单元结构体. 结构体中的值默认是不可变的, 需要给结构体加上 `mut` 使其可变

```rust
struct Student {
    name: String,
    grade: i8,
    class: i8,
    id: i8,
}
let mut bob = Student {
    name: String::from("Bob"),
    class: 1,
    grade: 2,
    id: 123,
};
bob.grade = 1;

struct Color(u8, u8, u8);
let android_green = Color(0xa4, 0xc6, 0x39);
// 解构
let Color(red, green, blue) = android_green;

struct Inches(i32);
let length = Inches(10);
let Inches(integer_length) = length;

struct EmptyStruct;
let empty = EmptyStruct;
```

## 枚举

**枚举** 是一种代表一系列子数据类型的集合, 可用于分类, 但是相较于其他语言, 还可以携带数据类型, 可以被称为 **重装枚举**. 枚举可以通过 `::`来获得每个元素的名称

```rust
enum Book {
    Pbook(u32),
    Ebook { url: String },
}
let book = Book::Pbook(1010);
let _ebook = Book::Ebook {
    url: String::from("https://xx.xx"),
};
```

与结构体一样, 枚举中的元素默认不能使用关系运算符进行比较 (如 `==`, `!=`, `>=`), 也不支持像 `+` 和 `*` 这样的双目运算符, 均需要自己实现, 或者使用 `match` 进行匹配。

### C 语言风格的枚举

Rust 的枚举也可以像 C 语言那样

```rust
// 拥有隐式辨别值 (implicit discriminator, 从 0 开始) 的 enum
enum Number {
    Zero,
    One,
    Two,
}

// 拥有显式辨别值 (explicit discriminator) 的 enum
enum Color {
    Red = 0xff0000,
    Green = 0x00ff00,
    Blue = 0x0000ff,
}

fn main() {
    // `enum` 可以转成整型。
    println!("zero 是 {}", Number::Zero as i32);//as 用来类型转换
    println!("one 是 {}", Number::One as i32);

    println!("玫瑰是 #{:06x} 色的", Color::Red as i32);
    println!("天空是 #{:06x} 色的", Color::Blue as i32);
}
```

# Module 与可见性

当项目越来越大, 把所有代码都塞进 `main.rs` 或 `lib.rs` 几乎是不可能的, 或者有的时候我们需要人为控制代码的可见性, 这个时候我们就需要 **Module (模块)** 来管理我们的项目结构

可以通过 `mod` 关键字 + `mod_name` 来开辟一个新的 Module, 默认情况下父 Module 或同级 Module 无法访问子 Module, 而子 Module 可以访问所有的父 Module

如果是在单个文件中, `mod_name` 后面直接跟一个 `{}`, 并在内部放入其成员

```rust
mod a_module {
    // Some code
}
```

当然, Module 最重要的功能还是将代码分到不同的文件中, 这种情况下 `mod_name` 后面直接加 `;` 即可, 我们在 `main.rs` 中声明三个 Module

```rust
mod mod1;
mod mod2;
mod mod3;
```

下面是一段文件树示例

```
src
├─ main.rs
├─ mod3.rs // 内容: mod file3;
├─ mod3
│  └─ file3.rs
├─ mod2
│  ├─ mod.rs // 内容: mod file2;
│  └─ file2.rs
└─ mod1.rs
```

我们能发现有三种不同的方式用于将代码分入不同的文件:

1. 直接在 `src` 文件夹下新建与 Module 同名的文件
2. 在 `src` 文件夹下新建与 Module 同名的文件夹并在内部新建 `mod.rs` 文件, 同时这个文件夹内也可以新建其他 Module 并在 `mod.rs` 文件中声明
3. 同时在 `src` 文件夹中新建与 Module 同名的文件和文件夹, 而这个同名的文件可以起到上一条中 `mod.rs` 文件的作用

Crate 本身就是其下所有模块的父模块, Module 层层嵌套就组成了模块树

## 通过路径引用模块

路径通过 `::` 操作符进行分隔, 分为两种:

- 相对路径:
    - 由当前模块的同级模块出发, 引用其内部成员. `current_mod::inner_function()`
    - 由父模块出发引用其他子模块内部成员, 需要 `super` 关键字. `super::another_mod::inner_function()`
    - 由当前模块出发, 引入自己的内部成员, 需要 `self` 关键字. `self::my_function()`. 比如在 Module 内部的一个函数需要引用这个 Module 的另一个函数时
- 绝对路径: 就像之前说的, Crate 就是最高的父模块, 可以直接由 Crate 出发一步步走向内部成员, 需要 `crate` 关键字. `crate::sub_mod::inner_function()`

## 代码可见性

下面这段代码看似合理, 但实际上并不能运行, 就像我们之前说的, Module 默认是对父 Module 不可见的

```rust
struct Int(i32)

mod inner {
    fn return_one() -> Int {
        Int(1)
    }
}

let one = inner::return_one();
```

可以通过 `pub` 关键字使内部成员公开

```rust
mod inner {
    pub fn return_one() -> Int {
        Int(1)
    }
}
```

注意, Struct 即使对外公开, 其内部 Field 仍然默认私有, 也需要 `pub` 关键字才能访问, 而 Enum 类型一旦公开, 其内部所有变体自动公开

```rust
mod inner {
    pub struct Point {
        pub x: i32,
        y: i32
    }

    pub enum Bool {
        True,
        False
    }
}

let p = inner::Point {
    // x 是 pub 的, 可以访问
    x: 0,
    // y 是 private 的, 无法访问, 此行会报错, 应给给 Point 的 y 也加上 pub
    y: 0
};

let b = inner::Bool::True;
```

`pub` 关键字还可以在后面加上一些限制

```rust
pub(crate) // 只在当前 Crate 公开
pub(super) // 只在当前父 Module 公开
pub(self) // 只在当前 Module 公开
pub(in a::b) // 在指定模块公开
```

如果是多重嵌套的 Module, 为了访问最里面的 Module, 同样需要在 `mod` 前加上 `pub` 关键字

```rust
mod out_mod{
    pub mod inner_mod{
        pub fn func(){
            // Do something
        }
    }
}

out_mod::inner_mod::func()
```

## 引入作用域

上面我们已经成功地将代码拆分到了不同的文件中. 但是有一个问题, 比如 `out_mod::inner_mod::func()`, 要是每次调用函数都要用上这一坨, 那任谁也受不了

这种时候, 我们就可以通过 `use` 关键字将一个成员引入我们的作用域, 使其在当前 Module 以及所有子 Module 持续可用

还是上面的例子

```rust
mod out_mod{
    pub mod inner_mod{
        pub fn func(){
            // Do something
        }
    }
}

use out_mod::inner_mod::func;

fn use1() {
    func()
}

fn use2() {
    func()
}
// ...
```

我们知道 Crate 本身就是一个顶级 Module, 因此我们也可以使用 `use` 将第三方库中的代码引入我们的作用域

可以通过 `{}` 一次引入多个成员, 用 `,` 隔开, 可以通过 `*` 引入一个模块下所有的公开成员. 同样可以通过 `self` 关键字引入 Module 自身

```rust
use crate1::mod1::{self, mod2::*, item};

mod1::some_func();
item();
```

## 重导出

可以通过 `pub` 关键字将 `use` 引入的成员在当前 Module 再次公开, 以让 Module 可以通过这个 Module 来访问那些成员

```rust
mod mod1 {
    fn func() {
        // Do something
    }
}

mod mod2 {
    pub use mod1::func;
}

mod2::func;
```

这对于集中管理公开成员非常有帮助, 事实上, Rust 为每一个项目都隐式的添加了 `use std::prelude::*`, 它就通过重导出其他模块的常用成员, 可以非常方便快捷的引入这些常用的功能. 在编写 Lib Crate 时也可以创建一个 `prelude` Module 来集中管理常用成员