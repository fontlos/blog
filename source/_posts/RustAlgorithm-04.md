---
feature: false
title: Rust 数据结构与算法(4) | 链表
date: 2025-03-27 12:00:00
abstracts: 链表和数组, 栈一样是一种线性数据结构, 不过由于 Rust 的安全规则不允许我们实现自身嵌套的数据结构, 因此我们将会接触到 Unsafe Rust 的世界, 同时也会接触到 Rust 零成本抽象特点, 因此我将这一节往后调整了一些
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

链表和数组, 栈一样是一种线性数据结构, 不过由于 Rust 的安全规则不允许我们实现自身嵌套的数据结构, 因此我们将会接触到 Unsafe Rust 的世界, 同时也会接触到 Rust 零成本抽象特点, 因此我将这一节往后调整了一些

# 链表

链表是一种线性数据结构, 由一系列 **节点** 组成, 每个节点包含数据部分和指向其他节点的指针. 与数组不同, 链表中的元素在内存中不是连续存储的

链表拥有以下特点

- **动态大小**: 链表可以动态增长和缩小, 不像数组需要预先分配固定大小
- **高效插入/删除**: 在链表中插入或删除节点只需修改指针, 时间复杂度为O(1) (在已知位置时)
- **内存利用率**: 不需要连续内存空间, 可以更灵活地利用内存

链表实现要点

- 节点之间通过指针连接
- 需要处理头节点和尾节点的特殊情况
- 通常提供添加、删除、查找等基本操作

## 单链表

从简单的单链表开始, 单链表顾名思义, 单向排列, 每个节点仅包含指向下一节点的指针

包含以下内容

- **节点(Node)**: 包含 **数据(val)** 和指向下一个节点的 **指针(next)**
- **链表结构(LinkedList)**: 通常包含 **头指针(start), 尾指针(end) 和长度(length)** 等信息

### `NonNull<T>`

在构建数据结构之前先来认识一个智能指针

`NonNull<T>` 是 Rust 标准库提供的一个智能指针类型, 它表示一个**非空的**, **协变的(covariant)** 指针

核心特性

1. **非空保证**: 与普通的 `*mut T` 不同, `NonNull<T>` 保证指针永远不会是 `null`. 这使得它可以用于枚举的判别式中, 例如 `Option<NonNull<T>>` 与 `*mut T` 大小相同
    - 枚举判别式: Rust 中枚举类型需要存储一个 **标签** 来区分不同的变体, 这个标签称为 **判别式(discriminant)**
    - `Option` 使得在类型系统层面让我们区分有无数据, 只要你持有 `Some` 就一定不会是空指针, None 代表不存在数据, 清晰的区分了有无的概念

2. **协变性**: `NonNull<T>` 对于 `T` 是协变的, 而 `*mut T` 是不变的. 这使得它在构建协变类型时更有用, 但也增加了误用的风险
    - **协变**: 如果 `A` 是 `B` 的子类型, 那么 `F<A>` 是 `F<B>` 的子类型
    ```rs
    fn example1<'short>(x: *mut &'static i32) -> *mut &'short i32 {
        // x // 错误!
        x as *mut &'short i32 // 正确, 需要显式转换
    }
    fn example2<'short>(x: NonNull<&'static i32>) -> NonNull<&'short i32> {
        x // 可以安全转换，因为 NonNull 是协变的
        // 所以对于链表来说提供了 生命周期灵活性, 允许链表自然地处理不同生命周期的元素
    }
    ```
    - 同理还有 **不变性** 和 **逆变性**
    - 协变性使得构建如 `Box, Rc` 等容器类型更灵活, 但需要确保类型安全
    - 在我们的链表实现中也会为我们节省大量的生命周期标注和显式生命周期转换

3. **内存布局**: 由于 **空指针优化**, `Option<NonNull<T>>` 和 `NonNull<T>` 具有相同的大小和对齐方式
    - 空指针优化: 由于 `NonNull<T>` 保证不为 `null`, 编译器可以将 `Option::None` 表示为 `null` 指针, 而不需要额外的判别式
    - 大小相同: `Option<NonNull<T>>` 不需要额外空间存储 `None` 状态，因为它可以用 `null` 指针表示 `None`, 所以它的大小与 `*mut T` 相同
    - 这使得在内存层面通过 `Option` 包装和空指针优化实现了 **零成本抽象**

主要用途

- 构建安全抽象 (如 `Box`、`Rc`、`Arc`、`Vec` 和 `LinkedList`) 的内部实现
- 在需要非空指针保证的场合替代 `*mut T`
- 在需要协变指针的场合使用

安全注意事项

1. **悬垂指针**: 虽然指针非空，但仍可能指向已释放的内存 (悬垂指针)
2. **协变风险**: 如果类型不应该协变, 例如如果类型内部有可变性, 协变可能不安全, 需要额外字段 (如 `_marker: PhantomData<Cell<T>>`) 来提供不变性
3. **共享引用转换**: 虽然 `NonNull<T>` 实现了从 `&T` 的 `From` 转换, 但通过共享引用派生的指针进行修改仍然是未定义行为, 除非在 `UnsafeCell<T>` 内部
    - `&T` 到 `NonNull<T>` 的转换是允许的
    - 但不能通过这样的指针修改数据, 除非在 `UnsafeCell` 中
    - 违反会导致未定义行为 (Undefined Behavior)

### 数据结构定义

```rs
use std::ptr::NonNull;

#[derive(Debug)]
struct Node<T> {
    val: T,
    // 指向下一个节点的指针
    next: Option<NonNull<Node<T>>>,
}

#[derive(Debug)]
struct LinkedList<T> {
    length: u32,
    // 指向链表第一个节点的指针
    start: Option<NonNull<Node<T>>>,
    // 指向链表最后一个节点的指针
    end: Option<NonNull<Node<T>>>,
}
```

回顾一下上面的内容

- 类型系统层面: 通过 `NonNull` 表明这是一个永远不为 `null` 的指针
    - 当确实有下一个节点时, 使用 `Some(non_null_ptr)`
    - 没有下一个节点时, 使用 `None`

- 内存表示层面: 通过 Option 包装 + 空指针优化, 实现零成本抽象
    - `Option<NonNull>` 会被优化为直接用 `null` 指针表示 `None`
    - 非 `null` 指针表示 `Some`

我们来看看为什么要这样设计

- `NonNull` 的保证: 当你确实持有指针时, 它绝对不会是 `null`
    ```rs
    let node = unsafe { non_null_ptr.as_ref() }; // 安全，因为知道不是 null
    ```
- `Option` 的语义: 存在或不存在下一个节点, 使得节点不会无穷嵌套没有尽头
- 空指针优化: `Option<NonNull>` 与 `*mut T` 大小相同 (在 64 位系统都是 8 字节)
- 与其他方案对比
    |方案|优点|缺点|
    |-|-|-|
    |`Option<Box<Node<T>>>`       |完全安全           |有堆分配开销, 胖指针包含额外数据会更大|
    |`Option<Rc<Node<T>>>`        |共享所有权         |有引用计数开销|
    |`*mut Node<T>`	              |最小开销	          |完全 `unsafe`, 可能为 `null`|
    |`Option<NonNull<Node<T>>>`	  |零成本抽象非空保证   |仍需 `unsafe` 操作|

接下来我们实现一些基本的功能

```rs
impl<T> Node<T> {
    fn new(t: T) -> Node<T> {
        Node { val: t, next: None }
    }
}

impl<T> Default for LinkedList<T> {
    fn default() -> Self {
        Self::new()
    }
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            length: 0,
            start: None,
            end: None,
        }
    }
}
```

### 添加一些基本方法

这些很基础, 我们就不去解释了, 接下来给 `LinkedList` 添加更多方法

```rs
pub fn add(&mut self, obj: T) {
    // 创建一个新的节点, 并用 Box 分配堆内存
    let mut node = Box::new(Node::new(obj));
    // 确保新节点的 next 指针为 None
    node.next = None;
    // 将 Box 转换为原始指针, 再包装成 NonNull 指针
    let node_ptr = Some(unsafe { NonNull::new_unchecked(Box::into_raw(node)) });
    // 检查链表是否为空
    match self.end {
        // 如果链表为空, 新节点就是首节点
        None => self.start = node_ptr,
        // 否则将当前尾节点的 next 指向新节点
        Some(end_ptr) => unsafe { (*end_ptr.as_ptr()).next = node_ptr },
    }
    // 更新尾指针为新节点
    self.end = node_ptr;
    // 增加链表长度
    self.length += 1;
}

// 公开方法, 调用私有递归方法
pub fn get(&mut self, index: i32) -> Option<&T> {
    self.get_ith_node(self.start, index)
}

// 递归查找第 index 个节点
fn get_ith_node(&mut self, node: Option<NonNull<Node<T>>>, index: i32) -> Option<&T> {
    // 检查当前节点指针
    match node {
        // 如果节点不存在返回 None
        None => None,
        // 如果节点存在
        Some(next_ptr) => match index {
            // 看看是否是第 index 个节点
            0 => Some(unsafe { &(*next_ptr.as_ptr()).val }),
            // 否则继续递归查找下一个节点
            _ => self.get_ith_node(unsafe { (*next_ptr.as_ptr()).next }, index - 1),
        },
    }
}
```

以上就是题目中已经给出的内容, 其他与链表无关的辅助功能这里就不写了, 比如格式化输出

### 合并有序单链表

然后就是我们算法教程的第一道题了, **合并两个有序单链表成一个新的有序单链表**

首先给出一种简单的方案, 既然提供了 `add` 函数, 那我们就借来用一下

```rs
pub fn merge(list_a: LinkedList<T>, list_b: LinkedList<T>) -> Self
    // 合并要求我们的数据是可比较的
where
    T: std::cmp::PartialOrd,
{
    let mut merged = LinkedList::new();

    // 封装获取节点值和移动节点的逻辑, 最小化 unsafe
    let get_val = |node_ptr: NonNull<Node<T>>| unsafe { &(*node_ptr.as_ptr()).val };
    let take_next = |node_ptr: NonNull<Node<T>>| unsafe {
        let next = (*node_ptr.as_ptr()).next;
        let val = std::ptr::read(&(*node_ptr.as_ptr()).val);
        (val, next)
    };

    // 获取 a, b 的头指针作为开始
    let mut a_ptr = list_a.start;
    let mut b_ptr = list_b.start;

    // 开始循环
    while let (Some(a_node), Some(b_node)) = (a_ptr, b_ptr) {
        // 获取 a, b 当前指针的值作比较, 选择先推入哪一个, 同时更新对应的指针
        if get_val(a_node) <= get_val(b_node) {
            // 这里直接获取值
            let (val, next) = take_next(a_node);
            // 使用提供的函数
            merged.add(val);
            a_ptr = next;
        } else {
            let (val, next) = take_next(b_node);
            merged.add(val);
            b_ptr = next;
        }
    }

    // 处理剩余节点
    let mut process_remaining = |mut ptr| {
        while let Some(node) = ptr {
            let (val, next) = take_next(node);
            merged.add(val);
            ptr = next;
        }
    };

    // 循环 a, b 剩余的内容
    process_remaining(a_ptr);
    process_remaining(b_ptr);

    merged
}
```

这种实现比较简单粗暴, 直接从目标地址把值读出来, 然后使用提供了的 `add` 函数将按逻辑推入. 虽然能通过测试, 但会有很多问题, 最简单的就是性能问题, 观察 `add` 函数也发现这涉及了内存分配, 而我们没有增加新的数据, 这显然是一种浪费

因此, 我们可以通过直接移动指针的方案来优化性能

```rs
pub fn merge(list_a: LinkedList<T>, list_b: LinkedList<T>) -> Self
    where
        T: std::cmp::PartialOrd,
    {
    let mut merged = LinkedList::new();

    let mut add_node = |node: NonNull<Node<T>>| {
        // 断开原链接, 防止节点同时属于两个链表, take 将 node 的 next 设为 none 并取出 next, 这是一个原子操作
        let next = unsafe{ (*node.as_ptr()).next.take() };
        // SAFETY:
        // merged.end 如果是 Some, 必定是之前添加的合法节点
        match merged.end {
            None => merged.start = Some(node),
            Some(end) => unsafe{ (*end.as_ptr()).next = Some(node) },
        }
        merged.end = Some(node);
        merged.length += 1;
        next
    };

    let mut a_ptr = list_a.start;
    let mut b_ptr = list_b.start;

    // 主合并循环
    while let (Some(a), Some(b)) = (a_ptr, b_ptr) {
        if unsafe {(*a.as_ptr()).val <= (*b.as_ptr()).val} {
            a_ptr = add_node(a)
        } else {
            b_ptr = add_node(b)
        };
    }

    // 处理剩余节点
    while let Some(curr) = a_ptr {
        a_ptr = add_node(curr);
    }
    while let Some(curr) = b_ptr {
        b_ptr = add_node(curr);
    }

    merged
}
```

逻辑比较清晰, 关键内容写在注释里了

### 为链表实现一些其他功能

完成了我们的习题, 对于链表, 我们可以再实现一些其他功能 -- 最经典的 **增删改查 (CRUD)**

我们的链表比较简单, 没有额外的键, 所以要实现这些功能我们需要假定每一个值都是唯一的, 当然你也可以扩展这个链表的数据结构, 来使链表支持重复元素

我们先实现一个基础的 **查**, 这样才方便我们实现其他功能, 和 `get` 不一样, 我们不查询索引, 而是查询对应的值

```rs
pub fn find(&self, value: &T) -> Option<NonNull<Node<T>>>
where
    T: std::cmp::PartialOrd,
{
    let mut current = self.start;
    while let Some(node) = current {
        unsafe {
            if &(*node.as_ptr()).val == value {
                return Some(node);
            }
            current = (*node.as_ptr()).next;
        }
    }
    None
}
```

逻辑很清楚, 基本没有什么需要解释的地方, 然后基于查, 我们就能很简单的实现更新了

```rs
/// 更新指定节点的值
pub fn update(&mut self, node: NonNull<Node<T>>, new_value: T) {
    unsafe {
        (*node.as_ptr()).val = new_value;
    }
}
```

你同样可以实现一个基于索引的更新

然后是删除

```rs
/// 删除指定节点
pub fn remove(&mut self, target: NonNull<Node<T>>) -> Option<T> {
    unsafe {
        // 处理头节点特殊情况
        if Some(target) == self.start {
            let node = Box::from_raw(target.as_ptr());
            self.start = node.next;
            if self.end == Some(target) {
                self.end = None;
            }
            self.length -= 1;
            return Some(node.val);
        }

        // 查找前驱节点
        let mut prev = None;
        let mut current = self.start;
        while let Some(node) = current {
            if (*node.as_ptr()).next == Some(target) {
                prev = Some(node);
                break;
            }
            current = (*node.as_ptr()).next;
        }

        if let Some(prev_node) = prev {
            // 将要删除的节点的 next 给前面的节点的 next
            let target_node = Box::from_raw(target.as_ptr());
            (*prev_node.as_ptr()).next = target_node.next;
            // 如果要删除的是最后一格节点, 那么需要更新尾指针
            if Some(target) == self.end {
                self.end = Some(prev_node);
            }
            self.length -= 1;
            Some(target_node.val)
        } else {
            None
        }
    }
}
```

链表的删除比数组要高效, 因为这几乎只涉及到指针的更改

为了写起来简单我直接用 `unsafe` 包装整个函数了, 这在实际开发中通常是不建议的, 我们要最小化 `unsafe` 内容

最后就是增加了, 链表的新增插入也比数组要高效, 只涉及到一个新节点的内存分配, 而无需移动后面所有值

```rs
/// 在指定节点后插入新值
pub fn insert_after(&mut self, node: NonNull<Node<T>>, value: T) {
    unsafe {
        let new_node = Box::new(Node {
            val: value,
            next: (*node.as_ptr()).next,
        });
        let new_node_ptr = NonNull::new(Box::into_raw(new_node));
        (*node.as_ptr()).next = new_node_ptr;
        // 更新尾指针
        if self.end == Some(node) {
            self.end = new_node_ptr;
        }
        self.length += 1;
    }
}

/// 因为头节点前面没有其他节点了, 所以需要一个单独的函数在链表头部插入, 这很类似于 add 函数在链表尾部插入内容
pub fn push_front(&mut self, value: T) {
    let mut new_node = Box::new(Node::new(value));
    new_node.next = self.start;
    let new_node_ptr = unsafe { NonNull::new_unchecked(Box::into_raw(new_node)) };
    // 要注意它是不是尾节点
    if self.end.is_none() {
        self.end = Some(new_node_ptr);
    }
    self.start = Some(new_node_ptr);
    self.length += 1;
}
```

最后我们可以新增一个测试函数

```rs
#[test]
fn test_crud() {
    let mut list = LinkedList::new();
    list.add(1);
    list.add(2);
    list.add(3);

    // 查找
    let node = list.find(&2).unwrap();

    // 更新
    list.update(node, 4);
    assert_eq!(list.get(1), Some(&4));

    // 插入
    list.insert_after(node, 5);
    assert_eq!(list.length, 4);
    assert_eq!(list.get(2), Some(&5));

    // 删除
    let val = list.remove(node).unwrap();
    assert_eq!(val, 4);
    assert_eq!(list.length, 3);

    // 头部插入
    list.push_front(0);
    assert_eq!(list.get(0), Some(&0));
    assert_eq!(list.length, 4);
}
```

### 链表的迭代器

对于数组, 向量, 哈希表这些类型, 我们都经常使用迭代器这个结构, 函数式写起来又爽又高效, 但实际上是标准库在背后做了大量工作. 其实我们也可以为链表尝试实现一些迭代器

```rs
// 不可变迭代器
pub struct LinkedListIter<'a, T> {
    current: Option<NonNull<Node<T>>>,
    _marker: std::marker::PhantomData<&'a Node<T>>,
}

impl<'a, T> Iterator for LinkedListIter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.current.map(|node| unsafe {
            let node_ref = &*node.as_ptr();
            self.current = node_ref.next;
            &node_ref.val
        })
    }
}

// 可变迭代器
pub struct LinkedListIterMut<'a, T> {
    current: Option<NonNull<Node<T>>>,
    _marker: std::marker::PhantomData<&'a mut Node<T>>,
}

impl<'a, T> Iterator for LinkedListIterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.current.map(|node| unsafe {
            let node_ref = &mut *node.as_ptr();
            self.current = node_ref.next;
            &mut node_ref.val
        })
    }
}
```

这两个迭代器就可以允许我们使用 `for` 循环来遍历或修改链表的值了.

除此之外, 对于有序链表的合并, 我们知道, 链表不同于数组, 链表中插入元素是十分高效的, 因此我们也可以通过向其中一个链表有条件的插入另一个链表来做到这个功能. 这里也可以使用迭代器辅助一下

```rs
// 消耗迭代器
pub struct LinkedListIntoIter<T> {
    current: Option<NonNull<Node<T>>>,
}

impl<T> Iterator for LinkedListIntoIter<T> {
    type Item = NonNull<Node<T>>;

    fn next(&mut self) -> Option<Self::Item> {
        self.current.map(|node| unsafe {
            let next = (*node.as_ptr()).next;
            (*node.as_ptr()).next = None; // 断开连接
            self.current = next;
            node
        })
    }
}

// impl LinkedList

fn into_iter(self) -> LinkedListIntoIter<T> {
    LinkedListIntoIter {
        current: self.start,
    }
}

fn append_node(&mut self, node: NonNull<Node<T>>) {
    unsafe{ (*node.as_ptr()).next = None };
    match self.end {
        None => self.start = Some(node),
        Some(end) => unsafe{ (*end.as_ptr()).next = Some(node) },
    }
    self.end = Some(node);
    self.length += 1;
}

pub fn merge(&mut self, other: LinkedList<T>) where T: PartialOrd {
    let mut other_iter = other.into_iter();
    let mut current = &mut self.start;

    while let Some(self_node) = current.as_ref().map(|n| unsafe { &*n.as_ptr() }) {
        if let Some(other_node_ptr) = other_iter.current {
            unsafe {
                if (*other_node_ptr.as_ptr()).val <= self_node.val {
                    // 插入 other 的节点到当前节点前
                    let other_node = other_iter.next().unwrap();
                    (*other_node.as_ptr()).next = *current;
                    *current = Some(other_node);
                    self.length += 1;
                    continue;
                }
            }
        }
        current = unsafe { &mut (*current.unwrap().as_ptr()).next };
    }

    // 处理剩余的 other 节点
    for node in other_iter {
        self.append_node(node);
    }
}
```

有人可能会问为什么不同时把被更新的链表也作为迭代器, 这是因为由于所有权和借用规则, 以及防止结构被破坏, 我们不能在迭代迭代器的同时插入或删除元素

## 双链表

学习了单链表, 那么双链表的理解就不难了, 双链表是链表的一种, 只有节点部分与单链表不同, 每个节点包含

1. 数据部分 (`val`)
2. 指向下一个节点的指针 (`next`)
3. 指向前一个节点的指针 (`prev`)

它们有如下如别

|特性|单链表|双链表|
|-|-|-|
|节点结构    |只有next指针          |有next和prev两个指针|
|遍历方向    |只能单向遍历(从头到尾)  |可以双向遍历(从头到尾或从尾到头)|
|空间占用    |较小                 |较大(每个节点多一个指针)|
|插入/删除   |需要知道前驱节点       |可以直接操作前后节点|
|实现复杂度  |较简单                |较复杂(需要维护两个指针)|

双链表拥有以下特性

1. **双向遍历能力**
    - 可以从任意方向遍历链表
    - 适合需要反向查找的场景

2. **更高效的节点操作**
    - 删除任意节点时间复杂度O(1) (已知节点位置时)
    - 不需要像单链表那样从头遍历找前驱节点

3. **更灵活的数据结构基础**
    - 许多复杂数据结构(如双端队列)的基础
    - 实现LRU缓存等算法的理想选择

### 数据结构实现

基本上只有节点结构定义和相关函数的改变

```rust
struct Node<T> {
    val: T,
    next: Option<NonNull<Node<T>>>,
    prev: Option<NonNull<Node<T>>>, // 新增的prev指针
}
```

其他函数的更改可以参考单链表的实现

这一节的题目是反转双链表, 这非常简单了, 只需要对每一个节点交换前后指针就可以了

```rust
pub fn reverse(&mut self) {
    let mut current = self.start;
    while let Some(mut node_ptr) = current {
        let node = unsafe{node_ptr.as_mut()};
        // 由于第一个节点没有 prev 指针, 所以我们从第二个节点开始交换 prev 和 next 指针
        current = node.next;
        std::mem::swap(&mut node.prev, &mut node.next);
    }
    // 最后交换链表的头尾指针
    std::mem::swap(&mut self.start, &mut self.end);
}
```

至于双链表的增删改查, 就交给读者自行实现吧
