---
feature: false
title: 初识 Rust(6) | 方法, 泛型, Trait, 生命周期, 集合类型
date: 2021-06-08 20:32:17
abstracts: 为类型实现方法. 如何使用泛型类型, 认识 Trait 和 Trait Object, 初步了解生命周期. 介绍集合类型, 动态数组, 基于动态数组的可变长度字符串, KV 存储.
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

# 方法

Rust 通过 `impl` 关键字在 `struct` , `enum` 或者 `trait` 对象的上下文实现方法调用语法, 一个对象可以有多个 `impl` 块. 关联函数的第一个参数通常为 `self` 参数, 一个指代方法或 Trait 类型的别名, 有 3 种变体:

- `self`, 允许实现者移动和修改对象, 对应的闭包特性为 `FnOnce`
- `&self`, 既不允许实现者移动对象也不允许修改, 对应的闭包特性为 `Fn`
- `&mut self`, 允许实现者修改对象但不允许移动, 对应的闭包特性为 `FnMut`

还有一个 `Self` 是用于主带当前的实例对象

```rust
/// 矩形结构体
struct Rectangular {
    width: u32,
    height: u32,
}
// 为该结构体创建方法
impl Rectangular {
    /// 创建矩形
    // 不含 self 参数的方法也称为静态方法
    fn new(width: u32, height: u32) -> Rectangular {
        Rectangular {
            width,
            height
        }
    }
    /// 快速创建正方形
    fn square(size: u32) -> Self {
        Rectangular {
            width: size,
            height: size
        }
    }
    /// 获得矩形宽
    fn get_width(&self) -> u32 {
        self.width
    }
    /// 获得矩形长
    fn get_height(&self) -> u32 {
        self.height
    }
    /// 求矩形面积
    fn area(&self) -> u32 {
        self.width * self.height
    }
    /// 求矩形对角线
    fn diagonal(&self) -> f64 {
        let x = self.width as f64;
        let y = self.height as f64;
        let z = x * x + y * y;
        z.sqrt()
    }
}
fn main() {
    let a = Rectangular::new(10, 20);
    let b = a.area();
    println!("这个矩形的面积是: {}",b);

    let c= Rectangular::square(20);
    let d = c.area();
    println!("这个正方形的面积是: {}",d);

    let e = Rectangular::new(3, 4);
    let f = e.diagonal();
    println!("这个矩形的对角线是:{}",f);

    let g = Rectangular::new(10,40);
    println!("这个矩形的长是: {}, 宽是: {}",g.get_width(),g.get_height());
}
```

# 泛型

Rust 作为一门强类型语言, 如果一个函数的参数类型定义为 `i32`, 那它就无法接受 `i16` 的参数. 但在编程中, 经常需要用一个函数处理不同类型的数据, 例如两个数的加法, 无论是整数还是浮点数, 甚至是自定义类型, 都能进行支持. 这时, 就需要用到 **泛型 (Generics)**. 熟悉 **面向对象** 的人可能了解 **多态**, 实际上泛型就是多态的一种, 在类型理论中称作 **参数多态**, 即对于给定参数可以有多种形式的函数或类型. 泛型用于表示任意类型, 但通常我们会对其加以约束

我们提到的任意类型加法, 可以参考下面的 "例子", 它还不能够编译, 原因就是缺少约束, 因为在实际中, 也不是任意两个事物就可以相加的

```rust
fn add<T>(a:T, b:T) -> T {
    a + b
}

println!("add i8: {}", add(2i8, 3i8));
println!("add i32: {}", add(20, 30));
println!("add f64: {}", add(1.23, 1.23));
```

## 泛型声明

还记得 `Option` 枚举的定义吗, 其中的 `T` 就是一个泛型, 使得这个枚举可以承载任何值. 在 Rust 中我们习惯使用 `T` 作为泛型

```rust
enum Option<T> {
    Some(T),
    None,
}
```

相信你已经注意到了, 泛型同样需要先声明再使用, 声明的语法就是 `<T>`, 泛型参数可以不止一个, 也可以作用于结构体和方法

```rust
fn point<T, U>(a: T, b: U) -> (T, U) {
    (a, b)
}
let couple = point(1, 2.0);

struct Point<T> {
    x: T,
    y: T,
}

// 要注意先在 impl 声明泛型, 再在 Point 使用泛型
impl<T> Point<T> {
    // Some function
}

// 不仅可以定义泛型, 也能为特定的类型实现特定的方法
impl Point<i32> {
    // Some function
}

let int_origin = Point { x: 0, y: 0 };
let float_origin = Point { x: 0.0, y: 0.0 };
```

## Const 泛型

上面所提到的泛型都是针对 **类型** 的, 那么有没有针对 **值** 的泛型呢? 可能有人对这个问题本身都无法理解, 值要怎么用泛型? 我们先从数组讲起

我们知道 `[i32; 1]` 和 `[i32; 2]` 是两个不同的数组类型, 无法被同一个固定类型的函数所接受, 但我们可以通过引用, 用 `&[i32]` 这个类型让固定类型的函数可以接受所有的 `i32` 元素数组. 随后, 我们又可以用 `&[T]` 这个类型, 让函数接受所有 (被同一系列 Trait 所约束) 的数组

比如为了打印数组, 我们最终可以实现:

```rust
fn display_array<T: std::fmt::Debug>(arr: &[T]) {
    println!("{:?}", arr);
}
```

但如果在某种情况下, 不适合或者干脆不能用引用的数组呢? 其实很早之前, 部分第三方库的参数不允许数组超过 32 个元素, 因为迫于着某种原因, 他们需要为每种长度的数组都单独实现一个函数 ... 可以说是非常痛苦了, 不过好在, 这种情况已经称为过去式了, 我们拥有了 Const 泛型这种针对于值的泛型, 正好可以很好的处理数组长度的问题

让我们不通过引用重新实现上面的函数

```rust
fn display_array<T: std::fmt::Debug, const N: usize>(arr: [T; N]) {
    println!("{:?}", arr);
}
```

我们定义了一个类型为 `[T; N]` 的数组, 其中 `T` 是一个基于类型的泛型参数, 重点是 `N` 这个泛型, 它就是我们所说的那个基于值的泛型参数! 因为它用来替代的是数组的长度. 声明语法如上所示, 它基于的值类型是 usize

在 Const 泛型参数之前, Rust 完全不适合复杂矩阵的运算, 自从有了 Const 泛型, 一切都将改变

## 泛型的性能

对于多态函数，有两种 **派分** 机制: **静态派分** 和 **动态派分**. 前者类似于 C++ 的模板, Rust 会生成适用于指定类型的特殊函数, 然后在被调用的位置进行替换, 好处是允许函数被内联调用, 性能没有损耗, 毕竟这就像是我们为每个类型都手动实现了对应的函数, 但是这会导致代码膨胀, 使最终得二进制文件变大. 后者类似于 Go 的 `interface`, Rust 通过引入 **Trait Object (特征对象)** 来实现, 在运行期查找 **虚表** 来选择执行的方法. Trait Object 具有和 Trait相同的名称, 通过 **转换** 或者强制 **多态化** 一个指向具体类型的指针来创建, 会带来一定的性能损耗

# Trait

为了描述类型可以实现的抽象接口, Rust 通过 **Trait (特征)** 来定义 **函数类型签名**, 特性就相当于其他语言中的接口

```rust
// 通过 trait 关键字定义特性
trait HasArea {
    // 这个特性使 area 函数必须接受一个 &eslf 类型, 返回一个 f64 类型
    fn area(&self) -> f64;
}

struct Circle {
    x: f64,
    y: f64,
    radius: f64,
}

impl HasArea for Circle {//将特性应用于该结构体
    fn area(&self) -> f64 {//实现特性
        std::f64::consts::PI * (self.radius * self.radius)
    }
}

struct Square {
    x: f64,
    y: f64,
    side: f64,
}

impl HasArea for Square {
    fn area(&self) -> f64 {
        self.side * self.side
    }
}

```

总而言之, **Trait 定义了一组可以被共享的行为, 只要实现了Trait, 就能使用这组行为**

## 特征约束

其实在上面的示例中已经有所体现. 这里让我们再举一个例子:

```rust
fn print_area<T: HasArea>(shape: T) {
    println!("This shape has an area of {}", shape.area());
}
```

可以看到函数 `print_area()` 中的泛型参数 `T` 被添加了一个名为 `HasArea` 的 **Trait Bound (特征约束)**, 用以确保任何实现了`HasArea` 的类型将拥有一个 `area` 方法. Trait Bound, 可以使用 `+` 运算符

```rust
use std::fmt::Debug;

fn foo<T: Clone, K: Clone + Debug>(x: T, y: K) {
    x.clone();
    y.clone();
    println!("{:?}", y);
}
```

也可以有条件的实现 Trait, 例如, 标准库为任何实现了 `Display trait` 的类型实现了 `ToString trait`

```rust
impl<T: Display> ToString for T {
    // --snip--
}
```

现在我们可以修改之前的例子了

```rust
fn add<T: std::ops::Add<Output = T>>(a:T, b:T) -> T {
    a + b
}

println!("add i8: {}", add(2i8, 3i8));
println!("add i32: {}", add(20, 30));
println!("add f64: {}", add(1.23, 1.23));
```

只有能够加和的类型才能调用这个函数

有的时候类型约束可能会非常长, 将会是我们的函数签名也变得非常长, 这种时候就可以通过使用 `where` 关键字

```rust
fn bar<T, K>(x: T, y: K)
    where T: Clone,
          K: Clone + Debug
{
    x.clone();
    y.clone();
    println!("{:?}", y);
}
```

`where` 从句还允许限定的左侧可以是任意类型, 而不仅仅是类型参数

定义在 Trait 中的完整方法称为 **默认方法**, 可以被该 Trait 的实现 **重载**. 此外, Trait之间也可以存在 **继承**

```rust
trait People {
    fn eat(&self);

    // 默认方法
    fn greet(&self) {
        println!("Hello!");
        }
}

// 继承
trait Student : People {
    fn study(&self);
}

struct Citizen;

impl People for Citizen {
    fn eat(&self) {
        println!("Delicious!");
    }
}

impl Student for Citizen {
    fn study(&self) {
        println!("Study!");
    }
}
```

如果同一个类型实现了两个不同 Trait , 但这两个 Trait 却拥有名称相同的方法, 可以使用 **显式调用** 来避免混淆

```rust
// 短形式
TraitName::method_name(valuable);

// 展开形式
<TypeName as TraitName>::method_name(valuable);
```

## 直接用 Trait 作为参数和返回值

首先是作为参数

```rust
fn display(item: &impl ToString) {
    println!("Display: {}", item.to_string());
}
```

这看上去非常易懂: **item 是任何实现了 ToString trait 的类型**

同样, 我们也可以返回一个实现了某个 Trait 的任意类型

```rust
fn people() -> impl People{
    Citizen
}
```

但是有一点要注意, 函数的所有分支返回的类型必须是相同的, 这是因为 Rust 要求类型的大小必须在编译期已知, 而返回两种不同的实现了同一 Trait 的类型就会导致无法确定返回值的具体大小, 比如下面的代码无法编译

```rust
fn people(is_man: bool) -> impl People{
    if is_man{
        Man
    } else {
        Woman
    }
}
```

## 自动派生

我们的文章中已经出现过 `#[derive(Debug)]`, 它将通过一个 **过程宏** 为一个类型自动实现 `Debug trait`

## 调用某些方法需要引入对应的 Trait

**如果你要使用一个特征的方法, 那么你需要将该特征引入当前的作用域中**, 比如下面:

```rust
use std::convert::TryInto;

fn main() {
    let a: i32 = 10;
    let b: u16 = 100;
    let b_ = b.try_into().unwrap();
    if a < b_ {
        println!("Ten is less than one hundred.");
    }
}
```

虽然没有使用 `TryInto trait`, 但是调用了其下的 `try_into()` 方法. 但不用担心, 事实上 Rust 已经通过 `prelude` 引入了 `TryInto trait` 了, 所以不用担心这些额外的代码, 可以尝试删掉最上面的一行

## Trait 的泛型与关联类型

Trait 也可以接受泛型参数。但更好的处理方式往往是使用 **关联类型**

```rust
// 泛型参数
trait Graph<N, E> {
    fn has_edge(&self, &N, &N) -> bool;
    fn edges(&self, &N) -> Vec<E>;
}

fn distance<N, E, G: Graph<N, E>>(graph: &G, start: &N, end: &N) -> u32 {}

// 关联类型
trait Graph {
    type N;
    type E;

    fn has_edge(&self, &Self::N, &Self::N) -> bool;
    fn edges(&self, &Self::N) -> Vec<Self::E>;
}

fn distance<G: Graph>(graph: &G, start: &G::N, end: &G::N) -> uint {}

struct Node;

struct Edge;

struct SimpleGraph;

impl Graph for SimpleGraph {
    type N = Node;
    type E = Edge;

    fn has_edge(&self, n1: &Node, n2: &Node) -> bool {}

    fn edges(&self, n: &Node) -> Vec<Edge> {}
}

let graph = SimpleGraph;
let object = Box::new(graph) as Box<Graph<N=Node, E=Edge>>;
```

关联类型是在 Trait 定义的语句块中, 申明一个自定义类型, 这样就可以在特征的方法签名中使用该类型, 通常当类型很复杂时, 关联类型能大大提高代码可读性.

或者在使用泛型时, 导致函数头部也必须增加泛型的声明, 而使用关联类型会好很多, 对比一下下面的代码

```rust
trait Container<A,B> {
    fn contains(&self,a: A,b: B) -> bool;
}
fn difference<A,B,C>(container: &C) -> i32
  where
    C : Container<A,B> {...}

trait Container{
    type A;
    type B;
    fn contains(&self, a: &Self::A, b: &Self::B) -> bool;
}
fn difference<C: Container>(container: &C) {}
```

还有一点是我们之前提过 `Self` 用来指代当前调用者的具体类型, 那么如果我们定义了 `type Item`, 就可以使用 `Self::Item` 来指代该类型实现中定义的 `Item` 类型

## 关于实现 Trait 的几条限制:

- 如果一个 Trait 不在当前作用域内, 它就不能被实现。
- 不管是 Trait 还是 `impl`, 默认都只能在当前的 Crate 内起作用
- 带有 Trait Bound 的泛型函数使用 **单态化实现**, 所以它是 **静态派分** 的
- 最重要的一点, 不能为一个从外部引入的类型实现一个从外部引入的 Trait, 这两个至少要有一个是在当前作用域定义的, 这时为了防止你破环第三方库的代码或者第三方库破环你的代码, 比如你不能为 `String` 类型实现 `Display trait`. 这被称为 **孤儿原则**

## 绕过孤儿原则

可以通过 New Type 模式, 在本地定义一个元组结构体包裹外部类型, 然后为这个元组结构体实现外部 Trait

# 容器类型

## 动态数组

之前我们提到, Rust 的数组是不可变的, 而可变的动态数组, 我们称之为 **Vec (向量)**. 动态数组是一种基于堆内存申请的连续动态数据类型，拥有 O(1) 时间复杂度的索引, 压入 (push), 弹出 (pop)

动态数组在连续的内存空间储存多个值, 因此访问其中某个元素的成本非常低, 但同样只能存储相同类型的元素, 如果需要存储不同类型的元素, 可以使用 **重装枚举类型** 或者接下来会提到的 **Trait Object (特征对象)**

通过前面几篇文章的铺垫, Vec 的学习就变得相当简单了, 这里直接通过一系列例子进行展示

```rust
// 创建空 Vec
// 如果仅仅是创建空 Vec ,需要手动注明类型, 但如果随后便添加了一个元素, 那么编译器就可以自动推导出类型
let v: Vec<i32> = Vec::new();
// 使用宏创建空 Vec
let v: Vec<i32> = vec![];
// 创建包含 5 个元素的 Vec
let v = vec![1, 2, 3, 4, 5];
// 创建 10 个 0 的 Vec
let v = vec![0; 10];
// 创建可变的Vec, 并 push 元素 3
let mut v = vec![1, 2];
v.push(3);
// 创建拥有两个元素的 Vec, 并 pop 一个元素
let mut v = vec![1, 2];
let two = v.pop();
// 创建包含 3 个元素的可变 Vec，并索引一个值和修改一个值
let mut v = vec![1, 2, 3];
let three = v[2];
v[1] = v[1] + 5;

// 通过 get 方法安全的索引元素, 返回 Option<T> 枚举, 这里直接用 unwrap 方法进行解包, 但更好的做法是通过 match 进行解构
let one = v.get(0).unwrap();

// 通过迭代器遍历 Vec
let mut v = vec![1, 2, 3];
for i in &v {
    println!("{i}");
}

// 迭代的同时修改 Vec
let mut v = vec![1, 2, 3];
for i in &mut v {
    *i += 10
}

// 通过重装枚举装入不同类型的值
enum Num {
    Int(i32),
    Float(f64)
}

let v = vec![Num::Int(0), Num::Float(3.14)]

// 通过特征对象装入不同的值
trait Print {
    fn display(&self);
}

struct Text(String);
impl Print for Text {
    fn display(&self) {
        println!("The text is: {}",self.0)
    }
}
struct Int(i32);
impl Print for Int {
    fn display(&self) {
        println!("The num is {}",self.0)
    }
}

let v: Vec<Box<dyn IpAddr>> = vec![
    Box::new(Text("Hello, world!".to_string())),
    Box::new(Int(1)),
];

for ip in v {
    ip.display();
}
```

注意, Vec 与其内部元素是共存亡的, 一旦 Vec 离开作用域, 其自身和内部元素都将被 Drop

### 索引与 get 方法的区别

为什么要存在两种获取元素的方式呢? 其实这是为了解决数组越界导致的空指针问题

```rust
let v = vec![1, 2, 3, 4, 5];

let does_not_exist = &v[100];
let does_not_exist = v.get(100);
```

上面这两种方法, 使用索引会因为找不到元素直接 `panic`. 而使用 `get` 方法后, 最终通过解包会得到 `Option::None`

但总之, Rust 给了我们选择, `get` 方法既冗长不美观, 又会带来轻微的性能损失, 如果可以保证数组不越界, 那还是用索引更加方便

### Vec 常用方法

初始化 vec 的更多方式:
```rust
let v = vec![0; 3];   // 默认值为 0，初始长度为 3
let v = Vec::from([0, 0, 0]); // 从普通数组生成
let mut v = [0, 1, 2].to_vec();
v.is_empty() //判断是否为空
v.insert(3, 3); // 在指定索引插入元素, 注意索引不能超过 Vec 的长度
v.remove(3) // 移除指定索引的元素并返回, 比如这里会返回 3

let mut v1 = [3, 4, 5].to_vec(); // append 会清空 v1, 需要增加可变声明
v.append(&mut v1); // 将 v1 的元素添加到 v
v.truncate(5); // 截断到指定长度，多余的元素被删除, v: [0, 1, 2, 3, 4]
v.retain(|x| *x > 0); // 仅保留满足条件的元素 v: [1, 2, 3, 4]
// 删除指定范围的元素，同时获取被删除元素的迭代器, v: [1, 2], m: [3, 4]
let mut m: Vec<_> = v.drain(2..=4).collect();
let v2 = v.split_off(1); // 指定索引处切分成两个 vec, v: [1], v2: [2]
let slice = &m[0..=1]; // 获取切片
v.clear() // 清空数组
```

动态数组在增加元素时如果容量不足就会导致 Vec 扩容 (目前的策略是重新申请一块 2 倍大小的内存, 再将所有元素拷贝到新的内存位置，同时更新指针数据), 频繁扩容或者当元素数量较多且需要扩容时, 大量的内存拷贝显然会降低程序的性能

可以考虑在初始化时就指定一个实际的预估容量, 尽量减少可能的内存拷贝

```rust
let mut v = Vec::with_capacity(10);
v.reserve(100); // 调整 v 的容量到至少 100
v.shrink_to_fit(); // 释放剩余的容量, 一般不会主动执行
```

### Vec 的排序
Rust 实现了两种排序算法, 稳定排序 `sort` 和 `sort_by`, 不稳定排序 `sort_unstable` 和 `sort_unstable_by`

**稳定** 指对相等的元素, 不会对其进行重新排序, 而不稳定算法不保证这点, 但速度更快, 内存占用更低
```rust
// 排列整数
let mut vec = vec![1, 3, 2, 5, 4];
vec.sort_unstable();
// 排列浮点数
let mut vec = vec![1.0, 0.9, 1.1, 2.8, 4f32];
vec.sort_unstable();
```

运行后发现后者直接报错了, 这是因为在浮点数当中存在一个 **NAN (Not A Number)** 的值无法与其他的浮点数比较

所以浮点数并没有实现全数值可比较的 `Ord trait`, 而是实现了部分可比较的 `PartialOrd trait`

所以如果确定数组不包含 NAN, 可以用 `partial_cmp` 来作为大小判断的依据

```rust
let mut vec = vec![1.0, 0.9, 1.1, 2.8, 4f32];
vec.sort_unstable_by(|a, b| a.partial_cmp(b).unwrap());
```

如法炮制, 来对结构体数组进行排序

```rust
struct Num(u32)

let mut num = vec![Num(2), Num(1), Num(3)];
num.sort_unstable_by(|a, b| b.0.cmp(&a.0));
```

当然, 我们也可以手动实现 `Ord trait` 来作为排序依据, 但这还不够, 因为实现这个 Trait 还需要实现 `Eq, PartialEq, PartialOrd trait` 好消息是我们可以 `derive` 这些属性

```rust
#[derive(Ord, Eq, PartialEq, PartialOrd)]
struct Num(u32)

let mut num = vec![Num(2), Num(1), Num(3)];
num.sort_unstable()
```

## 可变长度字符串

`String` 是一个带有的 `vec:Vec<u8>` 成员的结构体, 可以理解为 `str` 类型的动态形式. 它们的关系相当于 `[T]` 和 `Vec<T>` 的关系, 所以 `String` 类型也有类似 `push` 和 `pop` 的方法

```rust
// 创建一个空的字符串
let mut s = String::new();
// 从 `&str` 类型转化成 `String` 类型
let mut hello = String::from("Hello, ");
// 压入字符和压入字符串切片
hello.push('w');
hello.push_str("orld!");

// 弹出字符。
let mut s = String::from("foo");
assert_eq!(s.pop(), Some('o'));
assert_eq!(s.pop(), Some('o'));
assert_eq!(s.pop(), Some('f'));
assert_eq!(s.pop(), None);
```

## KV 存储

**KV 键值对** 数据结构并提供了平均复杂度为 **O(1)** 的查询方法, 当我们希望通过一个 Key 去查询值时, 该类型非常有用, 下面我们介绍其中一种比较常用的, **HashMap (哈希表)**

HashMap 并不在 `prelude` 中, 需要我们手动引入, 类似 Vec, 可以使用 `new` 方法来, 然后通过 `insert` 方法插入键值对

```rust
use std::collections::HashMap;

let mut map = HashMap::new();

// 自动类型推断, 很明显是 HashMap<&str,i32>
map.insert("one", 1);
map.insert("two", 2);
```

可以通过迭代器将其他类型高效的转化成 HashMap

```rust
use std::collections::HashMap;

let price_list = vec![
    ("苹果".to_string(), 4),
    ("橘子".to_string(), 3),
    ("香蕉".to_string(), 5),
];

let price_map: HashMap<_,_> = price_list.into_iter().collect();
```

创建 HashMap 有三点需要注意:

1. 若类型实现 `Copy trait`, 会被复制进 HashMap
2. 若没实现 `Copy trait`, 所有权将被转移给 HashMap
3. 如果将引用放入 HashMap, 请确保该引用的 **生命周期** 至少跟 HashMap 一样长

### 一些常用操作

```rust
// 通过 Key 获取 Value
// get 方法会返回一个 Option 类型, 如果未查询到会返回 None
// 查询到会返回一个引用类型, 比如这里是 Option<i32>
price_map.get("苹果");

// 如果想直接获得 i32 类型可以用以下的方法
// 因为 i32 是 Copy 的, 通过 copied 方法可以返回 Option<i32>
// 随后通过 unwrap_or 方法解包, 如果是 Some 则返回, 如果是 None, 则返回 0
price_map.get("橘子").copied().unwrap_or(0);

// 迭代循环 HashMap
for (key, value) in &scores {
    println!("{} 的价格是 {}", key, value);
}

// 通过相同的 Key 可以更新对应的 Value
price_map.insert("香蕉".to_string(), 5);

// 查询一个 Key, 如果不存在就插入, 如果存在则无事发生
price_map.entry("葡萄".to_string()).or_insert(7);
```

# 特征对象

上面我们提到函数可以以 Trait 为返回值, 但有一个限制就是函数的所有分支返回的类型必须是相同的, 而这种限制就会让这种方法变得非常鸡肋. 有什么解决办法吗?

一种方法是可以采用 Rust 独有的重装枚举. Hmm, 这确实能解决眼下的问题, 但如果你无法提前知道返回的 Trait 的所有情况呢?

其实在上一篇文章中答案就已经显露了, 为了在 `main` 函数中使用 `?` 运算符, 我们通过 `Box<dyn Error>` 让 `main` 函数可以返回任何实现了 `Error` 的错误类型, 其中的 `dyn Trait` 就是我们接下来要讲的 **Trait Object (特征对象)**, 我们在前文 **泛型的性能** 就提到过, Trait Object 的类型是在运行时确定的, 因此会带来一定的性能损耗

这种类型在 UI 库中比较常见, 因为在 UI 中, 我们需要绘制组件, 我们不可能为每个组件都实现一个绘制函数, 而且有的库也允许用户封装自己的组件, 这就更无法预知其类型了, 这时就要用到 Trait Object

下面我们举一段例子

```rust
pub trait Draw {
    fn draw(&self);
}
```

任何实现了 `Draw trait` 的类型都是可以并且需要被绘制的

```rust
struct Text {}
impl Draw for Text {
    fn draw(&self) {
        // Drawing
    }
}
struct Image {}
impl Draw for Image {
    fn draw(&self) {
        // Drawing
    }
}
```

随后, 我们需要一个 View Tree, 这里我们用 Vec 来代替, 储存这些可以被绘制的对象. 但这个 Vec 应该是什么类型呢? 我们既需要绘制 `Text`, 也需要绘制 `Imagin`, 可我们不能填入两个类型. 但因为它们都实现了 `Draw trait`, 那可不可以把拥有 `Draw` 特征的对象填入呢? 答案当然是肯定的, 这就是 Trait Object

再提一次, Trait Object 指向实现了 `Draw trait` 的类型的实例, 也就是指向了 `Text` 或者 `Imagin` 的实例, 这种映射关系是存储在一张表中, 可以在运行时通过特征对象找到具体调用的类型方法

可以通过 `&` 引用或者 `Box<T>` **智能指针** 的方式来创建特征对象

> 关于智能指针会在以后的进阶教程中提到. 这里简单概括一下, `Box<T>` 能把任意类型分配到堆上并返回一个指针

关于创建的具体过程让我们先举一个小例子

```rust
trait Draw {
    fn draw(&self) -> String;
}

impl Draw for u8 {
    fn draw(&self) -> String {
        format!("u8: {}", *self)
    }
}

// 若 T 实现了 Draw trait, 则调用该函数时传入的 Box<T> 可以被隐式转换成函数参数签名中的 Box<dyn Draw>
fn draw1(x: Box<dyn Draw>) {
    // 由于实现了 Deref trait, Box<T> 会自动解引用为 T, 然后调用该值对应的类型上定义的 `draw` 方法
    x.draw();
}

fn draw2(x: &dyn Draw) {
    x.draw();
}

fn main() {
    let num = 8u8;
    draw1(Box::new(num));
    draw2(&num);
}
```

下面我们可以完善这段代码了

```rust
pub struct Screen {
    pub components: Vec<Box<dyn Draw>>,
}

// 实现一个 run 方法启动渲染
impl Screen {
    pub fn run(&self) {
        for component in self.components.iter() {
            component.draw();
        }
    }
}
```

有两点需要注意
- 可以发现 `dyn` 关键字只用在 Trait Object 的类型声明上, 在创建时无需使用
- 之所以能够将 Trait Object 作为参数, 是因为 `&dyn` 或者 `Box<dyn>` 都是一种指针, 可以在编译器确定指针的大小, 而如果直接使用 `dyn Draw`, `Draw` 类型的大小还是无法在编译器确定, 进而无法编译

那么使用 Trait Object 的好处是, 即使用户自己创建了一个组件, 只要它实现了 `Draw trait`, 就可以被加入到我们的 `Screen components` 中. 而如果使用泛型, 那么我们的 UI 就只能接受 `Text` 或者 `Imagin` 两种组件了

在动态类型语言和 Go 的 `interface` 中, 有一个重要概念: **Duck Typing (鸭子类型)**, 就是只关心值有什么特征, 而不关心它实际是什么. 原例子是, 当一个东西走起来像鸭子, 叫起来像鸭子, 那么它就是一只鸭子, 即使它真的不是鸭子, 我们也当它是鸭子

## 特征对象的限制

不是所有 Trait 都能拥有 Trait Object, 只有对象安全的 Trait 才行

如果一个对象时安全的, 它的 Trait 下的所有方法需要有以下特征:
- 方法的返回类型不能是 `Self`
- 方法没有任何泛型参数

对象安全对于 Trait Object 是必须的, 因为就像鸭子类型所讲的, 一旦有了特征对象, 我们就不再关心其具体类型了, 但如果 Trait 方法返回了 `Self` 类型, 但是特征对象忘记了其真正的类型, 那这个 `Self` 的处境就非常尴尬, 连它自己都不知道自己是什么了. 对于泛型参数也是同理
