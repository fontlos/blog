---
feature: false
title: 初识 Rust(7) | 生命周期, 格式化输出与文件 IO
date: 2021-06-08 22:14:48
abstracts: 简单认识一下宏. 介绍一下如何漂亮的打印各种类型以方便调试程序. 最后简单介绍一些文件IO
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

# 生命周期

生命周期是 Rust 中的一个特殊概念, 简而言之, 它标注了作用域的范围. 早期的 Rust 需要手动标注所有生命周期, 但后来人们发现有一些生命周期格式经常重复性出现, 于是 Rust 就制定了一些规则, 在非必要是通过编译器推理消除生命周期参数

生命周期的标注通常是单引号 `'` 加上一个小写字母, 下面一段会造成悬垂引用的代码, 让我们加上生命周期标注看看为什么

```rust
{
    let r;                // ---------+-- 'a
                          //          |
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
                          //          |
    println!("r: {}", r); //          |
}                         // ---------+
```

`r` 的生命周期范围是 `'a`, `x` 的生命周期范围是 `'b`, 很明显, `x` 的生命周期更短, 我们将 `x` 的引用赋值给 `r`, 当 `x` 的生命结束时, `r` 依然存在, 但此时这个被引用的 `x` 已经被 Drop 了, 因此就会报错

## 手动标注生命周期与生命周期消除

生命周期的标注只发生在引用中, 使用前像泛型那样须要事先声明, 使用时标注需要紧随 `&` 操作符, 各一个空格后跟上具体的引用类型, 而声明时不需要

```rust
fn func<'a>(arg: &'a str) -> &'a str {

}

struct Strs<'a> {
    strs: &'a str,
}
```

就像我们说过的, 生命周期的格式经常重复, 通过实践我们可以发现, 有些生命周期标注就没有存在的必要, 因此我们制定了三条规则, 满足以下条件的, 即使删除生命周期, 编译器也可以进行推导

1. 每一个引用都有自己的生命周期
    - 例如: `fn foo<'a>(x: &'a i32)`, `fn foo<'a, 'b>(x: &'a i32, y: &'b i32)`
2. 若只有一个输入生命周期, 那么该生命周期会被赋给所有的输出生命周期
    - 例如 `fn foo(x: &i32) -> &i32`, `x` 的生命周期会被自动赋给返回值 `&i32`, 即 `fn foo<'a>(x: &'a i32) -> &'a i32`
3. 若存在多个输入生命周期, 且其中一个是 `&self` 或 `&mut self`, 则 `&self` 的生命周期被赋给所有的输出生命周期
    - 拥有 `&self`参数, 说明该函数是一个 **方法**, 该规则让方法的使用便利度大幅提升
    - 若一个方法，它的返回值的生命周期就是跟参数 &self 的不一样, 这时答案就很简单了: 手动标注生命周期. 因为这些规则只是当你没标注时编译器默认加上的

让我们通过模拟编译器理解一下这些规则

```rust
// 单引用参数
fn foo(s: &str) -> &str
// 首先, 为每个参数标注一个生命周期

fn foo<'a>(s: &'a str) -> &str
// 函数只有一个输入生命周期, 被赋予所有的输出生命周期

fn foo<'a>(s: &'a str) -> &'a str
// 编译器自动为返回值添加生命周期, 检查通过

// 多引用参数
fn bar(x: &str, y: &str) -> &str
// 首先, 为每个参数标注一个生命周期

fn bar<'a, 'b>(x: &'a str, y: &'b str) -> &str
// 第二条规则失效, 因为输入生命周期有两个
// 第三条规则也不符合
// 编译器依然无法为返回值标注合适的生命周期
//标错并提示我们需要手动标注生命周期
```

注意: 通过函数签名指定生命周期参数只是给编译器的提示性标注, 并不能真正改变变量的作用域, 而是告诉编译器当不满足此约束条件时, 就拒绝编译通过, 比如下面的代码会报错

```rust
fn return_str<'a>(x: &str, y: &str) -> &'a str {
    let result = String::from("string");
    result.as_str()
}
```

## 不总是靠谱的生命周期检查

```rust
#[derive(Debug)]
struct Foo;

impl Foo {
    fn take_mutable(&mut self) -> &Self {
        &*self
    }
    fn take_immutable(&self) {}
}

fn main() {
    let mut foo = Foo;
    let bar = foo.take_mutable();
    foo.take_immutable();
    println!("{:?}", bar);
}
```

虽然 `take_mutable` 方法借用了 `&mut self`, 但是它最终返回的是一个 `&Self`, 因此理论上带来的结果只是一个不可变借用, 同时 `take_immutable` 也进行了不可变借用, 那根据借用规则, 这段代码是合理的, 只发生了两次不可变借用. 然而却无法编译通过, 编译器仍然会固执的认为 `take_mutable` 借用了可变的 `foo`, 所以下面不可以再发生不可变借用. 但是按照逻辑, 可变借用的作用域仅在 `take_mutable` 内, 离开该作用域回到 `main` 函数后应该已经不存在了

对于这个反直觉的事情, 可以用生命周期来解释

```rust
struct Foo;

impl Foo {
    fn take_mutable<'a>(&'a mut self) -> &'a Self {
        &'a *self
    }
    fn take_immutable<'a>(&'a self) {}
}

fn main() {
    'b: {
        let mut foo: Foo = Foo;
        'c: {
            let bar: &'c Foo = Foo::take_mutable::<'c>(&'c mut foo);
            'd: {
                Foo::take_immutable::<'d>(&'d foo);
            }
            println!("{:?}", bar);
        }
    }
}
```

我们发现 `&mut foo` 和 `bar` 的生命周期都是 `'c`. 还记得 **生命周期消除规则** 第三条吗, 这导致了 `take_mutable` 方法中参数 `&mut self` 和返回值 `&Self` 拥有相同的生命周期, 因此, 若返回值的生命周期在 `main` 函数有效, 那 `&mut self` 的借用也是在 `main` 函数有效, 于是就违背了可变借用与不可变借用不能同时存在的规则, 最终导致了编译错误

实际上, 上述代码逻辑上完全正确, 但是因为生命周期系统的的死板, 导致了编译错误, 不幸的是, 截止到现在, 遇到这种因为生命周期系统的不靠谱导致的编译错误没有什么特别好的解决办法, 基本只能去修改代码. 期待后续堆生命周期系统继续完善, 让它足够聪明来理解这个问题

## 无界生命周期

`Unsafe` 块经常会凭空产生引用或生命周期, 这些生命周期被称为是 **Unbound (无界)** 的

比如解引用一个 **Raw Pointer (裸指针)**, 它并没有任何生命周期, 然后通过 `unsafe` 关键字操作后, 它被进行了解引用, 变成了一个 Rust 的标准引用类型, 该类型必须要有生命周期, 也就是 `'a`, 这个生命周期就是凭空产生的, 因为输入参数根本就没有这个生命周期

```rust
fn foo<'a, T>(x: *const T) -> &'a T {
    unsafe {
        &*x
    }
}
```

这种生命周期由于没有受到任何约束, 因此它想要多大就多大, 它实际上比 `'static` 还要强大. 例如 `&'static &'a T` 是无效类型, 但是无界生命周期 `&'unbounded &'a T` 会被视为 `&'a &'a T` 从而通过编译检查, 因为它的大小完全取决于需要它多大

因此我们要尽量避免这种无界生命周期. 最简单方式就是在函数声明中运用生命周期消除规则. 若一个输出生命周期被消除了, 那么必定因为有一个输入生命周期与之对应

## 生命周期约束 HRTB

生命周期约束跟特征约束类似, 都是通过形如 `'a: 'b` 的语法, 来说明两个生命周期的长短关系, 比如这里就是 `'a >= 'b`

被引用者的生命周期必须要比引用长, 比如

```rust
struct Ref<'a, T: 'a> {
    r: &'a T
}
```

因为 `r` 引用了 `T`, 因此 `r` 的生命周期 `'a` 必须要比 `T` 的生命周期更短. 在早期版本的 Rust 中上述标注是必须的, 而在新版本中, 编译器可以自动推导 T: 'a 类型的约束, 因此我们只需这样写即可

```rust
struct Ref<'a, T> {
    r: &'a T
}
```

实际上这也被称作 **结构体生命周期消除**

## Impl 块生命周期消除

```rust
impl<'a> Foo for Bar<'a> {
    // 内部实际上没有用到 'a
}
```

如果发现写出了这样的代码, 那实际上可以改写成下面这样

```rust
impl Foo for Bar<'_> {
    // Methods ...
}
```

`'_` 生命周期表示 `BufReader` 有一个不使用的生命周期, 我们可以忽略它, 无需为它创建一个名称, 那既然用不到为何还要写出来呢? 别忘了, **生命周期参数也是类型的一部分**, 因此 `BufReader<'a>` 是一个完整的类型, 在实现它的时候, 你不能把 `'a` 给丢了

## 闭包的生命周期消除规则

先来看一段简单的代码

```rust
fn foo(x: &i32) -> &i32 { x }
let bar = |x: &i32| -> &i32 { x };
```

乍一看, 这不一样吗? 编译下试试? 编译不通过! 错误原因是编译器无法推测返回的引用和传入的引用谁活得更久

回忆一下生命周期消除规则: 如果函数参数中只有一个引用类型, 那该引用的生命周期会被自动分配给所有的返回引用. 完全一致, 而且 `foo` 函数也没有报错, 那这是为什么呢?

这是因为对于函数的生命周期而言, 它的消除规则之所以能生效是因为它的 **生命周期完全体现在签名的引用类型** 上, 在函数体中无需任何体现, 因此编译器可以做各种编译优化, 也很容易根据参数和返回值进行生命周期的分析, 最终得出消除规则

可闭包的生命周期分散在参数和闭包函数体中, 编译器就必须深入到函数体中, 去分析和推导, 复杂度因此急剧提升

日常遇到这个问题, 还是老老实实用函数吧. 这个问题很难解决, 但不是无法解决, 比如通过使用 `Fn trait`

```rust
fn fun<T, F: Fn(&T) -> &T>(f: F) -> F {
   f
}
let closure_slision = fun(|x: &i32| -> &i32 { x });
```

## NLL (Non-Lexical Lifetime)

引用的生命周期正常来说应该从借用开始一直持续到作用域结束, 但是这种规则会让多引用共存的情况变得更复杂. 好在新版 Rust 提供了 **NLL**, 我们在之前已经提过这个概念, 这里再解释一下: 引用的生命周期从借用处开始, 一直持续到最后一次使用的地方

再来看一段关于 NLL 的代码解释

```rust
let mut u = 0i32;
let mut v = 1i32;
let mut w = 2i32;

// lifetime of `a` = α ∪ β ∪ γ
let mut a = &mut u;     // --+ α. lifetime of `&mut u`  --+ lexical "lifetime" of `&mut u`,`&mut u`, `&mut w` and `a`
use(a);                 //   |                            |
*a = 3; // <-----------------+                            |
...                     //                                |
a = &mut v;             // --+ β. lifetime of `&mut v`    |
use(a);                 //   |                            |
*a = 4; // <-----------------+                            |
...                     //                                |
a = &mut w;             // --+ γ. lifetime of `&mut w`    |
use(a);                 //   |                            |
*a = 5; // <-----------------+ <--------------------------+
```

这段代码一目了然, `a` 有三段生命周期：`α, β, γ`, 每一段生命周期都随着当前值的最后一次使用而结束

## 再借用

以 NLL 为基础, 让我们再了解一个高级概念: **Reborrow (再借用)**

直接先上码

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

impl Point {
    fn uodate(&mut self, x: i32, y: i32) {
        self.x = x;
        self.y = y;
    }
}

fn main() {
    let mut p = Point { x: 0, y: 0 };
    let r = &mut p;
    let rr: &Point = &*r;

    println!("{:?}", rr);
    r.update(10, 10);
    println!("{:?}", r);
}
```

乍一看, 同时有了可变引用 `r` 和不可变引用 `rr`, 这不是违反了借用规则吗? 但实际上并没有, 因为 `rr` 是对 `r` 的再借用

```rust
let mut p = Point { x: 0, y: 0 };
let r = &mut p;
// Reborrow! 此时借用 r 与 r 借用 p 不冲突
let rr: &Point = &*r;
// rr 最后一次使用在这里, 期间我们并没有使用原来的借用 r , 因此不会报错, 根据 NLL, rr 在这里离开作用域
println!("{:?}", rr);
// rr 已 Drop, 使用 r 完全没有问题
r.move_to(10, 10);
println!("{:?}", r);
```

## &'static 和 T: 'static

`'static` 在 Rust 中是相当常见的, 例如字符串字面值, 特征对象就具有 `'static` 生命周期. 除了 `&'static` 有时我们还可以把 `'static` 作为生命周期约束, 比如 `T: Display + 'static`

那么问题来了, 这两者有什么区别吗?

### &'static

`&'static` 对于生命周期有着非常强的要求: 这个引用必须要活到程序结束

对于字符串字面量来说, 它直接被打包到二进制文件中, 永远不会被 Drop, 因此它能跟程序活得一样久, 自然它的生命周期是 `'static`, 这针对的仅仅是引用, 而不是持有该引用的变量, 变量还是要该 Drop 就 Drop 的

```rust
use std::{slice::from_raw_parts, str::from_utf8_unchecked};

fn get_memory_location() -> (usize, usize) {
    // “Hello World” 是字符串字面量, 因此它的生命周期是 `'static`
    // 但持有它的变量 string 的生命周期完全取决于作用域
    let string = "Hello World!";
    let pointer = string.as_ptr() as usize;
    let length = string.len();
    (pointer, length)
    // string 被 Drop, 但其对应的数据依然存在
}

fn get_str_at_location(pointer: usize, length: usize) -> &'static str {
    unsafe { from_utf8_unchecked(from_raw_parts(pointer as *const u8, length)) }
}

let (pointer, length) = get_memory_location();
let message = get_str_at_location(pointer, length);
println!(
    "The {} bytes at 0x{:X} stored: {}",
    length, pointer, message
);
```

### T: 'static

比起 `&'static`, 这种形式的约束就有些复杂了

首先, 在以下两种情况下, `T: 'static` 对 `T` 的约束与 `&'static` 有相同的意义

```rust
use std::fmt::Debug;

fn print<T: Debug + 'static>( input: T) {
    println!("'static value passed in is: {:?}", input);
}

// 或者 impl 类型
// fn print( input: impl Debug + 'static ) {
//     println!("'static value passed in is: {:?}", input);
// }

fn main() {
    let i = 0;

    print(&i);
}
```

这会报错, 原因很简单: &i 的生命周期无法满足 'static 的约束

但只需要小小的修改一下函数签名

```rust
// print_impl 同理
fn print<T: Debug + 'static>( input: &T) {
    println!("'static value passed in is: {:?}", input);
}
```

居然就修好了, 这是因为我们约束的是 `T`, 但使用的是 `&T`, 因此编译器不会检查 `T` 的约束, 只要确保 `&T` 的生命周期符合规则即可, 而这段代码显然是符合的. 这也说明了 `'static` 这个约束有多么脆弱

那么 ``static` 到底针对谁, 是 `&'static` 这个引用还是该引用指向的数据活得跟程序一样久呢

答案是引用指向的数据, 而引用本身是要遵循其作用域范围的, 就像这个简单的例子

```rust
{
    let static_str = "I'm static";
    println!("{}", static_str);
    // 该变量被 Drop, 但是数据依然存在
}
println!("{}", static_str);
```

最后, 一个经验是: 如果你需要添加 &'static 来让代码工作, 那很可能是设计上出问题了

# 格式化输出宏

在上一篇文章我们一上来就接触到了一个在其它语言初学中不会上来就学的东西 --- **宏**

格式化输出都需要宏可见 Rust 是比较依赖宏的语言

Rust 的宏基于 **AST** 语法树而非 C++ 中简单的字符串替换, 所以更加强大, 甚至可以扩展 Rust 自身的 "语法"

将字符打印到控制台的操作由 `std::fmt` 里面的一系列宏来处理:
- `format!`: 格式化一系列字符串和参数为 `String`
- `print!`: 与 `format!` 类似, 但将最终结果输出到控制台标准输出
- `println!`: 与 `print!` 类似, 但输出结果追加一个换行符
- `eprint!`: 与 `format!` 类似, 但将文本输出到控制台标准错误
- `eprintln!`: 与 `eprint!` 类似, 但输出结果追加一个换行符

之前的文章中我们也进行了许多打印操作, 但 `println!` 宏远比你想象的强大得多

```rust
fn main() {
    // `{}` 会被任意变量内容所替换
    // 变量内容会转化成字符串
    println!("{} days", 31);

    // 不加后缀的话, 31 就自动成为 i32 类型
    // 你可以添加后缀来改变 31 的类型

    // 用变量替换字符串有多种写法
    // 比如可以使用位置参数
    println!("{0}, this is {1}. {1}, this is {0}", "A", "B");

    // 可以使用命名参数
    println!(
        "{name} {age} {id}",
        name="A",
        age=12,
        id=123
    );

    // 可以在 `:` 后面指定特殊的格式
    println!("{} 的二进制是 {:b}", 10, 10);

    // 你可以按指定宽度来右对齐文本
    // 下面语句输出 "     1", 5 个空格后面连着 1
    println!("{number:>width$}", number=1, width=6);

    // 你可以在数字左边补 0. 下面语句输出 "000001"
    println!("{number:>0width$}", number=1, width=6);

    // println! 会检查使用到的参数数量是否正确

    #[allow(dead_code)]
    struct Structure(i32);
    // 像结构体这样的自定义类型需要更复杂的方式来处理
    // 下面语句无法运行
    // println!("{}", Structure(3));
}
```

接下来我们将学习如何打印像结构体那样的复杂数据类型

所有的类型, 若想用 `std::fmt` 的格式化打印出来, 都要求实现它. 自动的实现只为一些类型提供, 比如 `std` 库中的类型. 所有其他类型 都 **必须** 手动实现

`fmt::Debug` 派生宏使这项工作变得相当简单, 所有类型都能推导 `fmt::Debug` 的实现. 但是 `fmt::Display` 需要手动实现

## Debug trait

所有 Std 类型都天生可以使用 `{:?}` 来打印
```rust
// 推导 `Structure` 的 `fmt::Debug` 实现
#[derive(Debug)]
struct Structure(i32);

// 将 `Structure` 放到结构体 `Deep` 中. 然后使 `Deep` 也能够打印
#[derive(Debug)]
struct Deep(Structure);

fn main() {
    // 使用 `{:?}` 打印和使用 `{}` 类似
    println!("一年有{:?}个月", 12);
    println!("{0:?}是这个演员的名字", "Slater");

    // `Structure` 也可以打印！
    println!("打印结构体{:?}", Structure(3));

    // 使用 `derive` 的一个问题是不能控制输出的形式
    // 假如我只想展示一个 `7` 怎么办？
    println!("打印结构体{:?}", Deep(Structure(7)));
}

```

`fmt::Debug` 使这些内容可以打印, 但是牺牲了一些美感. Rust 通过 `{:#?}` 提供了 "美化打印" 的功能

```rust
#[derive(Debug)]
struct Person<'a> {
    name: &'a str,
    age: u8
}

fn main() {
    let name = "Peter";
    let age = 27;
    let peter = Person { name, age };

    // 美化打印
    println!("{:#?}", peter);
}
```

## Display trait

`fmt::Debug` 通常看起来不太简洁, 因此自定义输出的外观经常是更可取的. 这需要通过手动实现 `fmt::Display` 来做到. `fmt::Display` 采用 `{}` 标记

```rust
#![allow(unused_variables)]
fn main() {
    // fmt::Display 需要手动导入
    use std::fmt;

    struct Structure(i32);

    impl fmt::Display for Structure {
        fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
            // 仅将 self 的第一个元素写入到给定的输出流 `f`. 返回 `fmt:Result`
            // 结果表明操作成功或失败. `write!`的用法和 `println!` 很相似
            write!(f, "{}", self.0)
        }
    }
}
```

`fmt::Display` 的效果可能比 `fmt::Debug` 简洁, 但对于 `std` 库来说, 模棱两可的类型该如何显示呢? 这并不是一个问题, 因为对于任何 **非** 泛型的 **容器** 类型, `fmt::Display` 都能够实现。

```rust
use std::fmt;

// 带有两个数字的结构体. 推导出 `Debug`, 以便与 `Display` 的输出进行比较
#[derive(Debug)]
struct MinMax(i64, i64);

// 实现 `MinMax` 的 `Display`
impl fmt::Display for MinMax {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        // 使用 `self.number` 来表示各个数据
        write!(f, "({}, {})", self.0, self.1)
    }
}

// 为了比较, 定义一个含有具名字段的结构体
#[derive(Debug)]
struct Point2D {
    x: f64,
    y: f64,
}

// 类似地对 `Point2D` 实现 `Display`
impl fmt::Display for Point2D {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        // 自定义格式, 使得仅显示 `x` 和 `y` 的值
        write!(f, "x: {}, y: {}", self.x, self.y)
    }
}

fn main() {
    let minmax = MinMax(0, 14);

    println!("Compare structures:");
    println!("Display: {}", minmax);
    println!("Debug: {:?}", minmax);

    let big_range =   MinMax(-300, 300);
    let small_range = MinMax(-3, 3);

    println!("The bigger range is {big} ,the smaller range is {small}",
        small = small_range,
        big = big_range
    );

    let point = Point2D { x: 3.3, y: 7.2 };

    println!("Compare points:");
    println!("Display: {}", point);
    println!("Debug: {:?}", point);
}

```

# 输入输出流

## 控制台输入

```rust
use std::io; // 手动导入 `io`

fn read_input() -> io::Result<()> {
    //创建空字符串
    let mut input = String::new();
    io::stdin().read_line(&mut input)?;
    println!("Your input: {}", input.trim());
    Ok(())
}
fn main() {
    read_input();
}
```

## 控制台输出

对我们来说这些已经太熟悉了, 这里只再提一些细节. 标准化的输出是 **行缓冲** 的, 这就导致标准化的输出在遇到一个新行之前并不会被隐式刷新. 换句话说 `print!` 和 `println!` 二者的效果并不总是相同的. 比如:

```rust
use std::io;
fn main() {
    print!("Waitting: ");
    let mut input = String::new();
    io::stdin()
        .read_line(&mut input)
        .expect("Read failed");
    print!("Your input: {}\n", input);
}
```

在这段代码运行时则不会先出现预期的提示字符串, 因为行没有被刷新. 如果想要达到预期的效果就要显示的刷新, 即在提示字符串下加一行 `io::stdout().flush().unwrap();`

## 文件输入

文件输入流指向了文件而不是控制台, 一般通过 match 处理潜在错误

```rust
use std::error::Error;
use std::fs::File;
use std::io::prelude::*;
use std::path::Path;

fn main() {
    // 创建一个文件路径
    let path = Path::new("test.txt");
    let display = path.display();

    // 打开文件只读模式, 返回一个 `io::Result<File>` 类型
    let mut file = match File::open(&path) {
        // 处理打开文件可能潜在的错误
        Err(err) => panic!("无法打开 {}, 错误: {}", display,Error::description(&err)),
        Ok(file) => file,
    };

    // 文件输入数据到字符串, 并返回 `io::Result<usize>` 类型
    let mut s = String::new();
    match file.read_to_string(&mut s) {
        Err(err) => panic!("无法读取 {}, 错误: {}", display,Error::description(&err)),
        Ok(_) => print!("{} 的内容为:\n{}", display, s),
    }
}
```

# 文件输出

文件输出流重定向到文件中

```rust
use std::error::Error;
use std::io::prelude::*;
use std::fs::File;
use std::path::Path;

fn main() {
    let path = Path::new("out/test.txt");
    let display = path.display();

    // 用只写模式创建并打开一个文件, 并返回 `io::Result<File>` 类型
    let mut file = match File::create(&path) {
        Ok(file) => file,
        Err(err) => panic!("无法创建文件 {}, 错误: {}", display, Error::description(&err)),
    };

    file.write(b"写入文本").unwrap();
}
```

# OpenOptions

Rust 还为我们提供了一个方便的配置用于统一前面两个操作, 下面看一些例子

```rust
use std::fs::OpenOptions;

// 打开一个文件
let file = OpenOptions::new() // 创建一组可供配置的空白新选项, 每个选项的默认值都是 false
    .read(true) // 启用 读 模式
    .write(true) // 启用 写 模式
    .create(true) // 如果不存在则创建文件, 存在就返回文件
    .open("test.txt"); // 文件路径
```

如果该文件已经存在, 则对该文件的任何写调用都将覆盖其内容, 而不会将其截断

下面还有一些常用选项

- `append`: 追加模式, 写入将追加到文件中, 而不是覆盖. 请注意, 设置 `.write(true).append(true)` 与仅设置 `.append(true)` 具有相同的效果. 下面是一些注意事项
    - 对于大多数文件系统, 操作系统保证所有写操作都是 **Atom (原子)** 的: 不会浪费任何写操作, 因为另一个进程会同时进行写操作
    - 使用追加模式时, 确保一次完成将所有在一起的数据写入文件
    - 如果同时使用读取和追加的访问权限打开文件, 请注意: 在打开之后以及每次写入之后, 读取位置可能设置在文件末尾. 所以在写入之前, 保存当前位置, 可以使用 `seek(SeekFrom::Current(0))`, 并在下次读取之前恢复它
- `truncate`: 截断模式, 如果成功打开文件, 则会将文件长度截断为 0. 注意, 必须同时开启 `write` 才能使用此操作
- `create_new`: 创建新文件模式. 与 `create` 不同的是, 如果文件已存在, 即使是符号链接, 它也不会返回, 而是直接报错, 只有文件不存在才会创建并返回. 通过这样的方式确保打开的一定是新文件
    - 这个选项是有实际用处的, 因为它是原子的. 否则, 在检查文件是否存在与创建新文件之间, 文件可能是由另一个进程创建的
    - 注意: 如果开启了此选项, 则 `create` 和 `truncate` 将被忽略