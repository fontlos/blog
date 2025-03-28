---
feature: false
title: 初识 Rust(3) | 变量, 常量, 语句和表达式, 原生类型
date: 2021-05-08 12:10:27
abstracts: 对变量进行绑定, 解构与遮蔽. 什么是不可变变量, 为什么变量默认不可变, 与常量的区别是什么. 如何声明常量. 什么是语句和表达式. Rust 底层实现的原生类型有哪些
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

# 变量

## 变量绑定

在多数语言中, 我们可能会用类似 `var a = 1` 的语句将 `Value` **赋值 (assignment)** 给 `Variable`

而在 Rust 中我们使用 `let` 关键字将 `Value` **绑定 (Bind)** 到 `Variable`

为什么要引入一个新的名字呢? 这里就涉及 Rust 为了实现内存安全首创的最核心的语言机制 --- **所有权**

简单来讲, 任何 **Memory Object (内存对象)** 都有一个 **Owner (主人)**, 而且一般情况下一个内存对象有且完全属于一个主人, 绑定这个过程就是把这个内存对象绑定给一个变量

既然是绑定, 那么就可以很容易的想到, 内存对象是可以被迫离开原来的主人被绑定到一个新的变量的, 而且根据字面意思, 该内存对象之前的主人就会丧失对其的所有权, 比起赋值, 用绑定来描述这个过程更加形象, 不是吗?

至于所有权这个概念, 让我们在以后的部分加以解释, 现在让我们先来看几个例子:

```rust
fn main() {
    let a = 5; // 类型推断
    let b: i32 = 10; // 显式声明
    let c = 10i32; // 另一种显式声明, num + type
    let d = c; // 整数类型默认实现了 Copy trait, 所以下面变量 c 仍可使用

    //a = 10;   //报错, 变量默认不可更改

    // 编译器会对未使用的变量绑定产生警告；可以给变量名加上下划线前缀来消除警告。
    let _unused = 3u32;
}
```

Rust 通过静态类型确保类型安全. 变量绑定可以在声明时注明类型, 不过在多数情况下, 编译器能够从上下文推导出变量的类型, 从而大大减少了类型注释的工作

## 变量解构

`let` 不只是用于声明变量, 绑定变量. 实际上 `let` 是一种 **匹配模式 (Match Pattern)**, 拥有十分强大的功能

```rust
fn main() {
    let (a, mut b): (bool, bool) = (true, false);
    println!("a = {}, b = {}", a, b);

    // 解构赋值机制
    let (a, b);
    // _ 代表匹配一个值, 但是我们不关心且不需要这个值, 因此没有使用一个变量名而是使用了 _ 表示丢弃
    (a, b, _) = (1, 2, 3);
    println!("a = {}, b = {}", a, b);
}
```

## 变量遮蔽 (Shadowing)

其实在上面的例子我们就能看到, 连续声明了两次同名的变量. 很明显, Rust 允许声明同名变量, 这一过程称为 **遮蔽 (Shadowing)**, 在后面声明的变量会遮蔽掉前面声明的

```rust
fn main() {
    let a = 1;
    // 在main函数的作用域内对之前的变量进行遮蔽
    let a = a + 1;

    {
        // 在当前的花括号作用域内, 对之前的变量进行遮蔽, 不影响外部作用域
        let a = a * 2;
        println!("The value of a in the inner scope is: {}", a);
    }

    println!("The value of a is: {}", a);
}
```

这将会输出

```
The value of a in the inner scope is: 4
The value of x is: 2
```

但这并不代表我们改变了变量, 而是产生并绑定了一个新的变量, 涉及新的内存分配

变量遮蔽往往能节省变量名的使用, 让语义更加清晰

比如当我们接收了一段空格但只关心空格的数量

```rust
// 字符串类型
let spaces = "   ";
// usize数值类型
// 这样可以节省一个类似于 spaces_len 的变量名
let spaces = spaces.len();
```

## 默认不可变与可变变量

从上面的例子中可以看到 Rust 中的变量默认居然是 **不可变的**, 这似乎有违常识和字面意思

但其实, 默认不可变的变量能带来很多好处, 首先方便了编译器的类型推断, 然后让逻辑更加清晰, 能一眼看出哪些变量会在接下来发生变化

对于某些情景, 可以避免一些 Bug, 比如一个被多次使用的变量, 我们本希望它不改变, 但却不小心在某一处代码改变了它

实际上我们日常的代码中, 真正发生了改变的变量并不是特别多, 很多时候我们真的也需要一个 **不可变的变量**, 默认不可变就更加方便编译器进行检查, 既提高了效率, 也提高了内存安全性

在变量名前加上 `mut` 关键字, 即可让变量变为可变变量, 简单且灵活

通过显示声明可变变量, 能够强制让我们在写代码时思考这个变量 **是否真的需要可变**

```rust
fn main() {
    let mut a :f64 = 2.0;
    pritnln!("{}",a);
    // 改变 a 绑定的值
    // 直接修改对应地址的内存, 因此比遮蔽效率更高
    a = a + 1.0;
    println!("{}",a);
    // 通过变量遮蔽重新将a绑定为不可变
    let a = a;

    // // 之前的例子
    // // 这是不可以的, 无法将一个 usize 类型绑定给 &str
    // let mut spaces = "   ";
    // spaces = spaces.len();
}
```
虽然变量默认不可变, 但不可以把不可变变量理解为常量. 变量是可能不会发生改变的量, 而常量是永远不会改变的量, 在 Rust 中也有专门的声明方式

# 常量

Rust 有两种常量, 可以在任意作用域声明, 包括全局作用域. 它们都需要显式的类型声明:
- `const`: 不可改变的值, 通常情况下我们使用这种常量
- `static`: 通常称为静态变量, 具有 `'static` **生命周期**, 从程序启动到程序结束, 即在整个程序运行期间都存在, 静态变量是全局的, 可以在整个程序的任何地方访问
    - 有个特例是字符串字面量 `&str`. 它可以不经改动就被赋给一个 `static` 变量, 因为它的类型标记 `&'static str` 就包含了所要求的生命周期 `'static`. 其他的引用类型都必须特地声明, 使之拥有 `'static` 生命周期
    - `static mut`: 可变静态变量, 一种特殊的静态变量, 可以在运行时修改其值, 但这不等同于变量, 只能在 `unsafe` 块中操作
    - 可变静态变量通常用于在整个程序的执行过程中共享和修改全局状态. 一般情况下, 使用可变静态变量要慎重, 因为全局状态的可变性可能导致并发和竞争条件的问题. 然而有些场景下确实需要在全局范围内维护一些状态, 比如一个全局计数器, 这时可变静态变量是一种合理的选择

在实际开发中, 最好将硬编码的值保存为常量, 这样即使后期需要修改, 也只需要修改一次

```rust
// 全局变量是在在所有其他作用域之外声明的。
static LANGUAGE: &'static str = "Rust";
const  THRESHOLD: i32 = 10;

fn is_big(n: i32) -> bool {
    // 在一般函数中访问常量
    n > THRESHOLD
}

fn main() {
    let n = 16;

    // 在 main 函数中访问常量
    println!("This is {}", LANGUAGE);
    println!("The threshold is {}", THRESHOLD);
    println!("{} is {}", n, if is_big(n) { "big" } else { "small" });

    // 报错！不能修改一个 `const` 常量
    THRESHOLD = 5;
    // ^ 注释掉此行
}
```

# 语句与表达式

**语句(Statement)** 会执行一些操作但是不会返回一个值, 而 **表达式(Expression)** 会在求值后返回一个值, 很多其它语言而言往往不区分这两个概念, 但对 Rust 这种基于语句和表达式的语言来说, 理解和区分语句和表达式是很重要的. 基于表达式是函数式语言的重要特征, 表达式总要返回值

## 语句

语句和表达式的区分是非常简单的, 通常来说, 语句后面有一个 `;`, 而表达式没有

```rust
let a = 0;
let b = (1,2);
```

形如这种, 它们完成了一个具体的操作, 但是并没有返回值, 因此是语句

由于 `let` 是语句, 因此不能将 `let` 语句赋值给其它值, 如下形式是错误的

```rust
let b = (let a = 0);
```

但是, `let` 作为表达式已经是试验功能了, 也许不久的将来, 上面的代码可以被真正编译通过

## 表达式

表达式会进行求值, 然后返回一个值

表达式可以成为语句的一部分, 例如 `let a = 0;` 中 `0` 就是一个表达式, 它所求的值就是 `0` (有些反直觉, 但是确实是表达式)

调用函数, 宏都是表达式, 用花括号包裹最终返回一个值的语句块也是表达式. 总之, 有返回值就是表达式:

```rust
let y = {
    let x = 3;
    x + 1
};

println!("The value of y is: {}", y);
```

如果在上面的 `x+1` 后面加上一个 `;` 就表示丢弃返回值, 将会返回元类型 `()`

# 原生类型

在上面的例子中, 我们见到了 `i32` `f64` `bool` 等数据类型, 它们是 Rust 的原生类型, Rust 的原生类型有以下几类:

- 布尔类型: 只有两个值, `true` 和 `false`
- 字符类型: 表示单个 Unicode 字符, 储存为 `u8`
- 数值类型: 有符号整型 (`i8` `i16` `i32` `i64` `i128` `isize`), 无符号整型 (`u8` `u16` `u32` `u64` `u128` `usize`) 和浮点型 (`f32` `f64`)
- 字符串类型: 其底层为不定长类型 `str`, 更常用的是字符串切片 `&str` 和堆分配字符串 `String`, 其中 `String` 并不是原生类型, 字符串切片是静态的, 有固定大小且不可改变, 堆分配字符串是可变的
- 数组: 有固定大小, 且元素为同一类型, 可表示为 `[T; N]`
- 切片: 引用数组的一部分数据且无需复制, 可表示为 `&[T]`
- 元组: 有固定大小, 元素类型可不同的有序列表
- 指针: 最底层是裸指针 `*const T` 和 `*mut T`, 解引用它们是不安全的, 需要放到 `unsafe` 块里
- 函数: 本质是一个函数指针
- 元类型: 其唯一的值是 `()`

```rust
// 如果没有必要, 以后的示例将不会有 fn main() {} 等
// 下面按顺序展示一下这些类型
let t = true;
let f: bool = false;

let c = 'c';

let x = 100;
let y: u32 = 123_456;
let z: f64 = -1.2e+3; // 浮点数可用科学计数法
let zero = z.abs(); // 取绝对值操作
let bin = 0b111_000;
let oct = 0o1234_5670;
let hex = 0xf23a9;

let strs = "Hello world";
let strs: &'static str = "Hello, world!";
// 这不是原生类型哦
let mut string = strs.to_string();

let a = [0, 1, 2, 3];
let b = &a[1..3];
let mut ten_zeros: [i64; 10] = [0; 10];

let tuple: (i32, &str) = (50, "hello");
let (fifty, _) = tuple;
let hello = tuple.1;

let x = 5;
let p = &x as *const i32;
let point_at = unsafe { *p };

fn func(x:i32) -> i32 {
    x
}
let function: fn(i32) -> i32 = func;
```

有几点是需要特别注意的:

- 数值类型可以使用 `_` 来增加可读性
- Rust 支持单字节字符(`u8`) `b'H'` 和字节数组字符串(`&[u8; N]`) `b"Hello"`, **仅限于 ASCII 字符**. 使用 `r#"..."#` 标记来表示原始字符串, 不需要对特殊字符进行转义
- `str` 类型很少使用, `&str` 类型使用的较多, 本质是 `[u8]` 类型的切片 `&[u8]`, 是一种大小固定的类型, 之前提到过常见的字符串字面值就是带有 `'static` 生命周期的 `&str` 类型
- 使用 `&` 符号将 `String` 类型转换成 `&str` 类型很容易, 但由于 `String` 不是原生类型, 使用 `to_string()` 方法将 `&str` 转换到 `String` 类型 **涉及到高昂的分配内存**, 除非很有必要否则不要这么做. 也可以使用 `String::from("")` 直接生成一个 String 类型
- 数组的长度是 **不可变的**, 动态的数组(**Vec**) 将会在之后提到, 可以通过 `vec![]` 宏或者 `Vec::new()` 声明
- 元组可以使用 `==` 和 `!=` 运算符来判断是否相同
- 不多于 32 个元素的数组和不多于 12 个元素的元组在值传递时是自动复制的
- Rust 不提供原生类型之间的隐式转换, 只能使用 `as` 关键字显式转换
- 可以使用 `type` 关键字定义某个类型的别名, 并且应该采用大驼峰命名法(形如 `UpperCamelCase`), 这在解决非常长的变量名时非常有用, 最常见的时 `impl` 块中的 `Self` 别名

```rust
let decimal = 65.4321_f32;
let integer = decimal as u8;
let character = integer as char;

type Color = (u8, u8, u8);
let black: Color = (0, 0, 0);
```

## 基本计算

### 整数

对于整数, Rust 只提供了最基本的运算, 甚至没有乘方运算符, 需要通过方法调用

```rs
let a: i32 = 10;
let b: i32 = 3;
let c: i32 = -1;

// 基本运算
let sum = a + b;
let diff = a - b;
let product = a * b;
let quotient = a / b;
let remainder = a % b;
let power = a.pow(b as u32);
let absolute = c.abs();

// 比较运算
let eq = a == b;
let ne = a != b;
let lt = a < b;
let gt = a > b;
let le = a <= b;
let ge = a >= b;

// 位运算
let x: u8 = 0b1010;
let y: u8 = 0b1100;

let and = x & y;
let or = x | y;
let xor = x ^ y;
let left_shift = x << 2;
let right_shift = y >> 1;
let not = !x;
```

### 浮点数

对于浮点数, Rust 针对 `f32` 和 `f64` 类型提供了比较丰富的函数

```rs
let x: f64 = 3.5;
let y: f64 = 2.0;

let power = x.powf(y);
let sqrt = x.sqrt();
let gt = x > y;
let eq = x == y;

let x = 3.14159f32;
let sin_x = x.sin();
```

你可能会好奇, 为什么对于浮点数我还会说针对 `f32` 和 `f64`, 这又体现出 Rust 的严谨性了. 如果你不指明, 那么小数的类型是 `float`, 例如下面的代码就无法编译

```rs
let x = 3.14159;
let sin_x = x.sin();
```

此外, 对于浮点数的比较不总是可靠的, 因为浮点数中还存在一个特殊的值, `NaN` (Not a Number), 它无法进行比较, 不等于任何值, 不大于或小于任何值, 因此除了不等运算之外只要它参与比较就会返回 `false`

可以用 `is_nan` 方法来检测

```rs
let x = 0.0 / 0.0; // 产生 NaN
if x.is_nan() {
    println!("x is NaN");
}
```

可以用一种安全比较的方法来比较浮点数

```rs
let a = 1.0;
let b = f64::NAN;
match a.partial_cmp(&b) {
    Some(std::cmp::Ordering::Less) => println!("a < b"),
    Some(std::cmp::Ordering::Greater) => println!("a > b"),
    Some(std::cmp::Ordering::Equal) => println!("a == b"),
    None => println!("NaN"),
}
```

### 布尔类型

```rs
let a = true;
let b = false;

let and = a && b;
let or = a || b;
let not = !a;

let eq = a == b;
let ne = a != b;
```

### 字符类型

```rs
let c1 = 'a';
let c2 = 'b';

// 比较 ASCII 编码
let eq = c1 == c2;
let lt = c1 < c2;

let code = c1 as u32; // 97
let c3 = char::from_u32(98).unwrap(); // 'b'
```

### 混合计算

Rust 不提供隐式类型转换, 混合类型运算时必须手动写明类型转换的过程, 通过 `as` 关键字

```rs
let i: i32 = 10;
let f: f64 = 3.5;

// 需要显式转换
let sum = i as f64 + f; // 13.5

let u: u32 = 20;
// 需要显式转换
let product = i as i64 * u as i64; // 200

// 布尔值可以转换为整数
let b = true;
let int_val = b as i32; // 1
```

### 赋值运算符

类似于多数语言, Rust 也有赋值运算符

```rs
let mut a = 5;
// 加法赋值
a += 3; // a = 8
// 减法赋值
a -= 2; // a = 6
// 乘法赋值
a *= 4; // a = 24
// 除法赋值
a /= 3; // a = 8
// 取余赋值
a %= 5; // a = 3
// 位运算赋值
a <<= 1; // a = 6
a |= 0b1010; // a = 14
```

### 数据溢出处理

在不同的模式下编译, Rust 对数据溢出的处理也是不同的

```rs
// 默认情况下，Debug 模式会 Panic，Release 模式会环绕
let max_u8 = 255u8;
// let overflow = max_u8 + 1; // Debug 模式下会 Panic

// 使用 wrapping 方法
let wrapped = max_u8.wrapping_add(1); // 0

// 使用 checked 方法
if let Some(result) = max_u8.checked_add(1) {
    println!("No overflow: {}", result);
} else {
    println!("Overflow occurred");
}

// 使用 overflowing 方法
let (result, overflowed) = max_u8.overflowing_add(1); // (0, true)
```

### 类型转换

最后再集中介绍一下基本的类型转换

```rs
// 安全转换
let i: i32 = 42;
let f: f64 = i as f64;

// 截断转换
let f: f64 = 3.99;
let i: i32 = f as i32; // 3

// 可能丢失精度的转换
let big: i64 = 1_000_000_000_000;
let small: i32 = big as i32; // -727379968 (溢出)

// 布尔转换
let b: bool = true;
let i: i32 = b as i32; // 1

// 字符转换
let c: char = 'A';
let u: u32 = c as u32; // 65
```

## 数组与切片

这两个在原生类型中稍复杂一点, 下面让我们具体的学习一下这些类型

数组用于储存相同类型的数据集, `[T; N]` 表示一个 `T` 类型, `N` 个元素的数组, 数组的大小必须固定, 需要在编译的时候确定下来

```rust
let mut array :[i32; 3]= [0; 3];
array[1] = 1;
array[2] = 2;
println!("{}", array[2]);
```

`array[1] = 2` 的意思是将索引为 `1` 的元素的值改为 `2`, 需要注意的是 Rust 就像大多数语言那样, 数组第一个元素的索引为 **0**

**切片(Slice)** 类型和数组类似, 但其大小在编译时是不确定的. 切片是一个 **双字对象**, 第一个字是一个指向数据的**指针**, 第二个字是切片的 **长度**. 这个 "字" 的宽度和 `usize` 相同, Slice 可以用来借用数组的一部分, 类型标记为 `&[T]`

```rust
use std::mem;

// 此函数借用一个 slice
fn analyze_slice(slice: &[i32]) {
    println!("The first element of this slice is {}", slice[0]);
    println!("The slice has {} element", slice.len());
}

fn main() {
    let xs: [i32; 5] = [1, 2, 3, 4, 5];
    let ys: [i32; 500] = [0; 500];

    println!("The first element of this array is {}", xs[0]);
    println!("The second element of this array is  {}", xs[1]);

    println!("The length of this array is {}", xs.len());

    // 数组是在栈中分配的
    println!("The array occupied {} bytes", mem::size_of_val(&xs));

    println!("borrow the whole array as a slice");
    // 直接借用整个数组
    analyze_slice(&xs);

    // 也可以指向数组的一部分
    println!("borrow a section of the array as a slice");
    // 下标 1-3
    analyze_slice(&ys[1 .. 4]);
    // 下标 1-4
    analyze_slice(&ys[1 ..= 4]);
    // 下标 1-末尾
    analyze_slice(&ys[1 ..]);
    // 下标 0-4
    analyze_slice(&ys[.. 4]);
    // 借用整个数组
    analyze_slice(&ys[..]);

    // 越界的下标会引发致命错误 (panic)
    println!("{}", xs[5]);
}
```
