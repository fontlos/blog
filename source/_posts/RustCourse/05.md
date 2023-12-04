---
feature: false
title: 初识 Rust(5) | 流程控制, 模式匹配, 错误处理
date: 2021-06-08 16:27:33
abstracts: 介绍流程控制, 用于构建程序结构. 了解 Match 模式匹配, 如何用 Match 进行错误处理
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

# 流程控制

有了流程控制, 我们才能把代码的结构组织起来. Rust 中的流程控制都是表达式, 可以被赋值给变量

## if-else 条件分支

`if` 是分支的一种, 可以与 `else` 和 `else if` 连用. 条件不需要小括号, 但条件后面必须跟一个代码块.

```rust
let x = 5;
let y = if x == 5 { true } else { false }
```

## for 迭代器循环

与 C 语言的三目运算符不同, `for` 的抽象结构如下

```rust
for item in collection {
    // Do something with item
}
```

`collection` 拥有三种不同形式, 可以是任何实现了 `Iterator trait` 的集合类型, 因此也可以通过 **Iterator (迭代器)** 的形式来使用:

1. `collection` 等价于 `IntoIterator::into_iter(collection)`, 转移所有权到 `for` 代码块的作用域
2. `&collection` 等价于 `collection.iter()`	不可变借用
3. `&mut collection` 等价于 `collection.iter_mut()`	可变借用

再简单提一下迭代器, 其还可以是以下两种形式

1. `1..10`, 表示 1 到 9 的整数
2. `1..=10`, 表示 1 到 10 的整数

```rust
for i in 1..=10 {
    println!("{i}");
}
```

如果想在循环中 **获取元素的索引** 可以使用 `enumerate` 方法

```rust
let a = [4, 3, 2, 1];
// 注意索引从 0 开始
for (i, v) in a.iter().enumerate() {
    println!("第 {} 个元素是 {}", i + 1, v);
}
```

如果仅仅希望将一个过程重复几次, 不需要额外声明变量, 可以用 `_` 来接收

```rust
for _ in 1..=10 {
    println!("Print ten times");
}
```

### 迭代器有哪些优势

先让我们看看这两种循环方式

```rust
// 第一种, 通过索引访问元素
let collection = [1, 2, 3, 4, 5];
for i in 0..collection.len() {
    let item = collection[i];
    // Do something with item
}

// 第二种, 通过迭代器直接访问元素
for item in collection {
    // Do something with item
}
```

首先是性能差距:

- 第一种做法为了避免悬垂引用, 对 `collection[index]` 的索引访问会进行 **边界检查(Bounds Checking)**, 确认该 `index` 确实在 `collection` 内, 导致运行时的性能损耗
- 而第二种迭代器的方式, 由编译器确保每个元素绝对有效, 就不会触发这种检查

安全性:

- 第一种方式对 `collection` 的索引访问是 **非连续的**, 有可能在两次访问之间 `collection` 发生了变化, 导致脏数据产生
- 而第二种迭代器的方式是连续访问, 通过所有权限制保证在访问过程中数据并不会发生变化


Rust 的 `for` 循环比 C 中的更加优秀强大, 无需任何条件限制, 也不需要通过索引来访问, 再加上 Rust 的 **零成本抽象**, 是  Rust 中最安全且高效的循环, 因此也是最常用的循环

## While 条件循环

`while` 抽象结构如下

```rust
while bool {
    code
}
```

之前我们说 `for` 是最安全的循环, 下面我们通过用 `while` 来模仿 `for` 的功能来说明为什么

```rust
let a = [1, 2, 3, 4, 5];
let mut index = 0;

while index < 5 {
    println!("the value is: {}", a[index]);
    index = index + 1;
}
```

我们通过维护一个索引, 如期打印出了 5 条语句. 但这这很容易出错, 一旦我们错误判断了索引的范围, 就会引发 Panic

而且在循环过程中, 也会不停的进行边界检查, 拖慢性能

对比一下上面的, 很容易看出, `for` 更加安全, 高效, 简洁

但是也不是说我们只需要 `for` 循环, 不然 Rust 为什么要设计 `while` 循环, 根据不同的场景, 选择更合适的方式

## Loop 无限循环

C 或者Rust 都可以通过 `while true` 来实现一个无限循环, 但 Rust 又单独专门设计了一个无限循环 --- `loop`

```rust
/// 注: loop 同样返回一个 `!` 类型
loop {
    code
}
```

就 **循环** 这一概念而言, `loop` 毫无疑问是最纯粹的, 它仅仅是循环, 再无其他, 因此也是适用面最广的循环, 当然特定的场景下 `for` 或 `while` 才是更优解

使用 `loop` 一定要打起精神来, 不然一个无限消耗资源的循环可能会让设备内存溢出

对于 `loop`, 第一个问题就是, 如何结束循环? 接下来我们就会介绍两种控制循环的方式, 当然, 这对其他两种循环同样适用

## Break, Continue 与 Lable

对于循环, 可以使用 `break` 关键字强制退出循环, 也可以使用 `continue` 关键字结束本次循环并进入下一次循环, 这两个关键字后面也可以跟一个返回值

也可以使用标签退出指定循环

```rust
'outer: loop {
    'inner: loop {
        println!("退出外部循环")；
        break 'outer
    }
}
```

# 模式匹配

## match

### 基本操作

`match` 是一个强大的匹配模式, 接下来先看几个例子

```rust
let day = 5;

match day {
    0 | 6 => println!("休息日")，
    1 ... 5 => println!("工作日")，
    _ => println!("none"),
}
```

`|` 用于匹配多个值, `...` 或 `..=` 用于匹配一个范围, 包括开头结尾, 因为 `match` 进行的是穷举性匹配, 所以需要一个 `_` 匹配剩下的所有值

可以使用 `@` 来绑定一个变量

```rust
let a = 1;
match a {
    b @ 1...3 => println!("a = {}", b),
    // 也可以给绑定的变量起一个新的名字
    c: num @ 4...6 => println!("a = {}", num),
    // 绑定也可以匹配多个值, 但要记得加括号
    d @ (7 | 8) => println!("a = {}", d),
    _ => println!("Match failed"),
}
```

### 获得一个引用

之前的文章我们提到可以使用 `ref` 获得一个引用, 这里我们再次强调, 在模式匹配中只能通过 `ref` 获得一个引用, 而函数声明只能用 `&` 来获得一个引用

```rust
#![feature(core_intrinsics)]
enum Num<'a> {
    Nor(i32),
    NorRef(i32),
    Ref(&'a i32),
    RefRef(&'a i32),
}

fn print_data(data: &u32) {
    println!("log data: {}", data);
}

// fn print_data(data: ref u32){  // expected type, found keyword `ref`
//     println!("log data: {}", data);
// }

fn print_type<T>(_: T) {
    println!("log type name: {}", unsafe { std::intrinsics::type_name::<T>() })
}

fn log(num: Num) {
    match num {
        Favour::Nor(data) => {
            config(&data);
            print_type_name_of(data);
        },
        Favour::NorRef(ref data) => {
            config(data);
            print_type_name_of(data);
        },
        Favour::Ref(data) => {
            config(data);
            print_type_name_of(data);
        },
        Favour::RefRef(ref data) => {
            config(data);
            print_type_name_of(data);
        }
    }
}

fn main() {
    log(Favour::Nor(1));
    log(Favour::Ref(&2));
    log(Favour::NorRef(3));
    log(Favour::RefRef(&4));
}
```

输出结果如下:

```
log data: 1
log type name: u32
log data: 2
log type name: &u32
log data: 3
log type name: &u32
log data: 4
log type name: &&u32
```

通过 `ref mut` 在模式匹配中可以获得可变引用

```rust
let mut a = 1;
match a {
    ref mut x => println!(x),
}
```

### 解构复合类型

match 可用于解构复合类型, 如定长数组, 元组, 结构体或枚举 (实际上在上面的例子中已经体现)

```rust
let point = (0, 2);
match point {
    (0, y) => println!("这个点在y轴上, 纵坐标为 {}", y),
    (x, 0) => println!("这个点在x轴上, 横坐标为 {}", x),
    (0, 0) => println!("这个点是原点"),
    _ => println!("这个点不在坐标轴上"),
}
```

### 忽略变量

可以使用 `..` 来忽略变量

```rust
struct Point {
    x:i32,
    y:i32,
}

let point = Point {
    x:10,
    y:10
}

match point {
    Point {x,..} =>println!("x is {}",x),
}

enum Int{
    Value(i32),
    N
}

let value = Int::Value(10);

match value{
    Int::Value(i) if i>5=>println!("这个数字大于5"),
    Int::Value(..)=>println!("是一个数字"),
    Int::N=>println!("不是数字"),
}
```

但要注意 `..` 必须是无歧义的, 比如下面的代码无法运行

```rust
let numbers = (2, 4, 8, 16, 32);

match numbers {
    // error: `..` can only be used once per tuple pattern
    (.., second, ..) => {
        println!("Some numbers: {}", second)
    },
}
```

### 变量遮蔽

无论是 `match` 还是接下来会提到的 `if let`, 它们都将开辟一个新的代码块, 这将会绑定新变量, 同名外部变量会被暂时遮蔽:

```rust
let age = Some(0); // 此时 age 是 Some(T) 类型
match age { // match 作用域开始
    // 下一行 age 是 i32 类型, 但或许我们本想使用 Some(T) 类型
    Some(age) =>  println!("匹配出来的age是{}",age),
    _ => ()
}// match 作用域结束
// 此时 age 是 Some(T) 类型
```

### 匹配守卫

**Match Guard** 可以让我们在一个 `match` 分支模式之后额外添加一个 `if` 条件, 进行进一步更加精准的匹配, 这个条件也可以使用在匹配模式中创建的变量. 记住, 模式的优先级大于匹配守卫, 只有先满足模式, 才会考虑是否满足匹配守卫

```rust
let num = Some(4);

match num {
    Some(x) if x > 0 => println!("A positive num: {}", x), // 只有这一行会打印输出
    Some(x) => println!("A negative num: {}", x),
    None => println!("No num"),
}
```

当匹配模式无法提供类如 `if x > 0` 的表达能力时可以考虑这种方式

匹配守卫还能解决上面变量遮蔽导致无法使用外部变量的问题

```rust
let x = Some(5);
let y = 10;

match x {
    Some(50) => println!("50"),
    Some(n) if n == y => println!("Matched, n = {}", n),
    _ => println!("Default case, x = {:?}", x),
}
```

上面的代码只有第三个匹配分支会打印输出. 因为第二个匹配分支中的模式不会像 `Some(y)` 那样引入一个覆盖外部 `y` 的新变量. 这意味着可以在匹配守卫中使用外部的 `y`. 而匹配守卫 `if n == y` 并不是一个模式所以没有引入新变量, 这个 `y` 正是 **外部的** `y` 而不是新的覆盖变量 `y`

也可以在匹配守卫中使用 `|` 运算符来指定多个模式, 匹配守卫的条件会作用于所有的模式

```rust
let x = 4;
let y = false;

match x {
    // 匹配守卫并非只作用于 6
    4 | 5 | 6 if y => println!("yes"),
    _ => println!("no"),
}
```

### 拓展: match! 宏

这里含有许多未提到的知识, 可以先看以后的文章再回过头来看

考虑以下这种情况

有一个动态数组，里面存有以下枚举：

```rust
enum Bin {
    Zero,
    One
}

fn main() {
    let bin = vec![Bin::One, Bin::One, Bin::Zero];
}
```
如果想对 `bin` 进行过滤，只保留类型是 `Bin::One` 的元素, 这种做法是不行的

```rust
bin.iter().filter(|b| b == Bin::One);
```

因为无法将 x 直接跟一个枚举成员进行比较. 虽然也可以用 `match` 来解决, 但在迭代器链式调用中略显啰嗦, 好在 Rust 标准库提供了一个非常实用的宏: `matches!`, 可以将一个 **表达式** 跟 **模式** 进行匹配, 并根据是否匹配成功返回 `true` 或 `false`. 因此上面的代码可以改成这样

```rust
bin.iter().filter(|b| matches!(b, Bin::One));
```

还有许多更强大的功能

```rust
let f = 'f';
// assert! 宏用于判断一个表达式是否为 true
// 同样可以指定多个模式
assert!(matches!(foo, 'A'..='Z' | 'a'..='z'));

let four = Some(4);
// 同样可以使用匹配守卫
assert!(matches!(four, Some(x) if x > 2));
```

## if let 与 while let

`if let` (单次) 与 `while let` (循环) 相当于精简版的 `match`, 用于解决一些匹配项很少或者只关心其中几个值时的匹配

```rust
enum Int{
    Value(i32),
    N
}

let a = Int::Value(10);

if let Int::Value(v) = a {
    println!("The value is {}", v);
}

let mut b = Int::Value(1);

while let Int::Value(v) = b {
    if v>3 {
        println!("b is over 3");
        b = N
    } else {
        println!("b is {:?}, add one",i);
        b = Int::Value(i+1);
    }
}
```

实际上类似 `match`, `if let` 也支持变量绑定, 而且还有一种称为 **绑定时解构** 的操作

还记得我们之前也通过 `let` 解构过复合类型吗, 我们说过 `let` 也是一种匹配模式, 因此也拥有一些强大的功能, 比如, 绑定时解构

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

// 绑定新变量 `p`，同时对 `Point` 进行解构
let point = Point {x: 10, y: 5};
if let p @ Point {x: 10, y} = point {
    println!("x is 10 and y is {} in {:?}", y, p);
} else {
    println!("x was not 10 :(");
}

// 对了, 还记得我们之前也通过 `let` 解构过复合类型吗
// 我们还说过 `let` 也是一种匹配模式, 所以 ...
let p @ Point {x: px, y: py } = Point {x: 10, y: 23};
println!("x: {}, y: {}", px, py);
println!("{:?}", p);
```

# 可能存在的值

**Null (空值)** 是一个让人又爱又恨的东西, 很多时候它都太重要了, 但是又由于空值过于灵活, 带来了无穷无尽的内存安全问题

作为一门现代语言, 没有空值不行, 但作为一门注重安全的语言, 有空值又不好, 于是 Rust 通过枚举巧妙地解决了这个问题, 构造了一个 **Option (可能存在的值)**

`Option` 在结构上非常类似我们上面所举的最后一个例子

```rust
enum Option<T> {
    Some(T),
    None,
}
```

其中 `T` 是一个 **泛型类型**, 表示 **任意符合限制的类型**, 我们将在后面的文章提到. 总之, 一个 `Option`, 要么存在 `Some(T)` 要么是 `None`

> 注: `Option` 的变体成员 `Some` 和 `None` 都通过 `prelude` 导入默认作用域了, 可以直接使用, 但不要忘记它们是来自 `Option` 的哦

上面提到的 **匹配模式** 就是用于解构 `Option` 的绝佳方式

```rust
let v = Some(1);
match v {
    Some(1) => println!("One!"),
    _ => (),
}

// 当然在我们仅仅关心一个值时不要忘了这个更加简洁的写法
if let Some(1) = v{
    println!("One!");
}
```

# 错误处理

## 不可恢复的错误与 panic! 宏的简单介绍

面对复杂的生产环境, 我们的程序几乎不可能是毫无错误的, 比如程序自身逻辑出现严重问题. 即使程序本身正常, 谁知道用户会做出什么奇怪的操作呢, 比如一个手滑删掉了程序所依赖的动态库, 这种时候就不是很适合让我们的程序靠自身来解决这个问题, 被称为 **Panic (不可恢复的错误)**

造成 Panic 有两种方式

### 被动触发

```rust
fn main() {
    let array = [1, 2, 3];
    println!("{}", v[10]);
}
```

一个经典的数组越界错误, 这时我们的程序就会被动的触发 Panic. 这在编程语言中无一例外, 都会报出严重的异常, 部分语言包括 Rust 甚至导致程序直接崩溃关闭。

报错信息如下

```
$ cargo run
   Compiling ...
    Finished ...
     Running ...
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 10', src/main.rs:3:20
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

报错信息很详细, 这多亏于 Rust 强大到一骑绝尘的编译器, 编译器手把手叫你写代码可不是乱吹的, 告诉我们崩溃线程, 崩溃原因, 对应的源码位置, 以及如何查看更详细报错信息的命令

被动触发是最常见的 Panic 方式. 主动报错总是好的, 总不会有人希望代码里藏着一个随时可能会爆的雷吧

### 主动调用

有时候处于一些特殊原因, 比如程序依赖的重要文件丢失了, 但既靠程序自己来还原可能会带来功能膨胀, 这时我们就可以手动调用 `panic!` 宏, 调用后程序会打印出一个错误信息,随后开始进行栈展开, 最后清理并退出程序.

> 切记, 一定是不可恢复的错误才能调用 `panic!`, 就像酒吧总不能因为客人点份炒饭就爆炸吧

### Backtrace 栈展开

实际开发中, 因为函数的层层调用, 错误往往涉及到很长的调用链甚至会深入第三方库, 如果没有栈展开技术, 错误将难以追踪溯源. 还是来看这个简单的错误

```rust
fn main() {
    let v = vec![1, 2, 3];
    v[10];
}
```

```
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 10', src/main.rs:3:5
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

有人可能不以为然, 但实际上数组越界是一个非常严重的问题, 比如 C 语言中, 数组越界一样可以访问对应地址的内存, 但结果可就不是一个数组的元素了, 这种情况被称为缓冲区溢出, 并可能会导致安全漏洞.

还是那句话, 主动报错总是好的, 比如这种情况, 如果真的访问到了一个未知的值, 这在很多时候会导致程序上的逻辑 Bug, 而众所周知, 逻辑 Bug 是最难被发现和修复的, 因此程序直接崩溃, 告诉我们问题所在, 然后我们进行修复, 这才是最合理的开发流程, 而不是把选择掩耳盗铃

现在, 我们已经知道错误发生的位置了, 为了获取更详细的信息, 让我们按照提示使用添加一个临时环境变量, 再次运行程序:  `RUST_BACKTRACE=1 cargo run` (Linux/Mac shell) 或 `$env:RUST_BACKTRACE=1 ; cargo run` (Windows PowerShell)

```
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 10', src/main.rs:4:5
stack backtrace:
   0: rust_begin_unwind
             at /rustc/59eed8a2aac0230a8b53e89d4e99d55912ba6b35/library/std/src/panicking.rs:517:5
   1: core::panicking::panic_fmt
             at /rustc/59eed8a2aac0230a8b53e89d4e99d55912ba6b35/library/core/src/panicking.rs:101:14
   2: core::panicking::panic_bounds_check
             at /rustc/59eed8a2aac0230a8b53e89d4e99d55912ba6b35/library/core/src/panicking.rs:77:5
   3: <usize as core::slice::index::SliceIndex<[T]>>::index
             at /rustc/59eed8a2aac0230a8b53e89d4e99d55912ba6b35/library/core/src/slice/index.rs:184:10
   4: core::slice::index::<impl core::ops::index::Index<I> for [T]>::index
             at /rustc/59eed8a2aac0230a8b53e89d4e99d55912ba6b35/library/core/src/slice/index.rs:15:9
   5: <alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index
             at /rustc/59eed8a2aac0230a8b53e89d4e99d55912ba6b35/library/alloc/src/vec/mod.rs:2465:9
   6: world_hello::main
             at ./src/main.rs:4:5
   7: core::ops::function::FnOnce::call_once
             at /rustc/59eed8a2aac0230a8b53e89d4e99d55912ba6b35/library/core/src/ops/function.rs:227:5
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

上面打印的内容就是一次 **栈展开 (栈回溯)**, 包含了函数调用的顺序 (**逆序**), 排在最顶部最后一个调用的函数是 `rust_begin_unwind`，该函数的目的就是进行栈展开, 并呈现这些信息给我们

> 注: 获取到栈回溯信息需要开启 `debug` 标志, 只要在使用 `cargo run` 或 `cargo build` 不加 `--release` Flag 即可, 这两个操作默认是 Debug 运行方式. 而且栈展开信息在不同操作系统或者 Rust 版本上也有所不同

### Panic 终止方式

当出现 `panic!` 时, 程序提供了两种方式来处理终止流程: 栈展开和直接终止。

- **栈展开** 是默认的方式, 这意味着 Rust 会回溯栈上数据和函数调用, 做更多的善后工作. 好处是可以给出充分的报错信息和栈调用信息, 便于事后的问题复盘
- **直接终止** 顾名思义就是不清理数据直接退出程序, 善后工作交与操作系统来负责

大多数情况使用默认选择就好, 但当你关心最终编译出的二进制可执行文件大小或者进行嵌入式开发时, 那么可以尝试去使用直接终止的方式, 例如下面的配置修改 `Cargo.toml` 文件, 实现在 Release 模式下遇到 Panic 直接终止

```toml
[profile.release]
panic = 'abort'
```

### Panic 后会怎么样

如果是 Main 线程 Panic, 则程序会终止, 如果是其它子线程, 则该线程终止, 不会影响 Main 线程. 因此, 尽量不要在 Main 线程中做太多任务, 可以将这些任务交由子线程去做, 防止整个程序崩溃

## 可恢复的错误与 Result 枚举

就像上面提到的, 遇到一些简单的错误时, 我们应该尝试着用一种更温和的方式捕获并解决这个问题, 而不是让我们的程序直接崩溃, 比如 `Result` 枚举, 其定义如下

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

类似于 `Option`, `Ok` 变体代表成功执行, `T` 泛型代表执行成功后返回的内容, `Err` 变体代表执行失败, `E` 泛型存放着错误信息. 而这两个变体也通过 `prelude` 导入了默认作用域, 可以直接使用

`Result` 与 `match` 等匹配模式结合起来, 就构成了 Rust 中优雅强大健壮的错误处理系统

举一个尝试打开文件的例子

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open("hello.txt");

    let f = match f {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the file: {:?}", e),
            },
            other_error => panic!("Problem opening the file: {:?}", other_error),
        },
    };
}
```

简单明了, 上面代码在匹配出 Error 并没有直接选择 Panic, 而是对 Error 进行了进一步匹配解析

- 如果文件存在且成功打开就返回文件句柄
- 如果是文件不存在错误 `ErrorKind::NotFound` 就创建文件, 这里创建文件. `File::create` 也是返回 `Result`, 因此继续用 `match` 进行匹配
    - 创建成功, 将新的文件句柄赋值给 `f`
    - 如果失败, 则 Panic
- 剩下的错误，一律 Panic

事实上这样写也有一点啰嗦. 在初识 Rust 后的一些进阶学习中, 我们会讲到 **组合器** 这个强大的工具

有时, 在代码原型设计阶段我们不想处理错误, 或者我们不需要处理这个错误, 或者我们能保证这个操作不会触发错误, 那么我们就可以通过以下两个方法简化错误处理

```rust
use std::fs::File;

let f = File::open("hello.txt").unwrap();
let f = File::open("hello.txt").expect("Failed to open hello.txt");
```

其中 `unwrap` 方法代表直接解构 `Result`, 成功就返回, 不成功就 Panic. `expect` 方法与之类似, 只不过可以附带一段额外的错误信息

## 错误传播

有时我们可能不需要在某个函数内部就地解决问题, 需要把错误层层向上传递, 让上层决策者决定如何处理问题, 这个时候我们可以考虑将错误信息封装进 `Result` 作为函数返回值移交给上层调用者

比如我们想要将文件的内容读取到字符串中

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_file() -> Result<String, io::Error> {
    let f = File::open("hello.txt");
    let mut f = match f {
        // 打开文件成功, 将 file 句柄绑定给 f
        Ok(file) => file,
        // 打开文件失败, 将错误返回(向上传播)
        Err(e) => return Err(e),
    };
    // 创建动态字符串
    let mut s = String::new();
    // 从 f 文件句柄读取数据并写入动态字符串
    match f.read_to_string(&mut s) {
        // 读取成功, 返回 Ok 封装的字符串
        Ok(_) => Ok(s),
        // 读取失败, 将错误向上传播
        Err(e) => Err(e),
    }
}
```

上面的代码很好的实现了我们的需求, 但还有一个问题 ,有些过于冗长. 幸运的是, Rust为我们提供了一个很甜的 **语法糖** --- `?` 运算符

下面让我们来看看这颗语法糖到底有多甜, 同样的功能用 `?` 重新实现:

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut f = File::open("hello.txt")?;
    let mut s = String::new();
    f.read_to_string(&mut s)?;
    Ok(s)
}
```

简洁而高效, 实际上 `?` 的本质是一个宏, 实现了与 `match` 类似的功能, 但在某些方面更加强大

比如 `?` 是可以链式调用的, 上面的代码可以进一步缩短:

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut s = String::new();
    File::open("hello.txt")?.read_to_string(&mut s)?;
    // 事实上 Rust 标准库提供了 `fs::read_to_string()` 函数提供了以上一条龙服务
    Ok(s)
}
```

### 关于 ? 的拓展

比如一个设计良好的系统中, 肯定有自定义的错误特征, 错误之间存在着上下级关系, 例如标准库中的 `std::io::Error` 和 `std::error::Error`, 前者是 `IO` 相关的错误结构体, 而后者是一个最最通用的 **标准错误特征**, 同时前者实现了后者, 因此 `std::io::Error` 可以转换为 `std:error::Error`

明白了以上的错误转换就能更好地理解 `?` 的强大了, 首先它可以自动进行类型提升

```rust
fn open_file() -> Result<File, Box<dyn std::error::Error>> {
    let mut f = File::open("hello.txt")?;
    Ok(f)
}
```

上面代码中 `File::open` 报错时返回的错误是 `std::io::Error` 类型, 但是 `open_file` 函数最终返回了一个实现了 `std::error::Error` 的 Trait Object. 一个错误类型通过 `?` 返回成了另一个错误类型. 这是因为标准库中有一个 `From trait`, 该 Trait 有一个 `from` 方法用于把一个类型转成另外一个类型, 而 `?` 可以自动调用该方法, 然后进行隐式类型转换. 因此只要函数返回的错误 `ReturnError` 实现了 `From<OtherError> trait`, 那么 `?` 就会自动把 `OtherError` 转换为 `ReturnError`

这种转换非常好用, 意味着你可以构建一个大而全的 `ReturnError` 来覆盖所有错误类型, 只需为各种子错误类型实现这种转换即可

实际上 `?` 不仅仅可以用于 `Result` 的传播, 还能用于 `Option` 的传播. 成功返回 `Some(T)`, 失败返回 `None`

最后有两点需要注意:
- `?` 需要一个变量来承载正确的值, 只有发生了错误才能直接返回, 因此类似 `func()?` 的表达式作为最终的返回值是不可以的
- `?` 只能在以 `Result` 为返回值的函数中使用. 那是不是代表着 `main` 函数与 `?` 无缘了呢? 不会的, 事实上 Rust 也支持返回 `Result` 的 `main` 函数:

```rust
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    Ok(())
}
```

这样就能使用 `?` 了. 可以看到返回的错误是 `Box<dyn Error>` , 因为 `std::error:Error` 是 Rust 中抽象层次最高的错误, 所以就算 `main` 函数中调用任何标准库函数发生错误, 都可以进行返回。

至于 `main` 函数可以有多种返回值, 那是因为实现了 `std::process::Termination trait` , 只是目前为止该特征尚未稳定

最后提一下, 事实上在 `?` 之前, Rust 有一个 `try!` 宏用于捕获错误, 只是如今这个宏在所有方面都不如 `?`, 因此还是不要使用的好