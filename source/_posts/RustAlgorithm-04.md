---
feature: true
title: Rust 数据结构与算法(4) | 栈
date: 2025-03-27 12:00:00
abstracts: 在这一节中, 我们将介绍栈的概念, 如何实现一个基本的栈和其他相关功能, 包括数组实现的方案和队列实现的栈, 最后我们再根据栈实现一个更好的队列
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

都来学习 Rust 了, 那么对于堆栈不说了解也应该是听说过了. 接下来让我们先从比较简单的栈结构来学习. 在这一节中, 我们将介绍栈的概念, 如何实现一个基本的栈和其他相关功能, 包括数组实现的方案和队列实现的栈, 最后我们再根据栈实现一个更好的队列

# 栈 (Stack)

栈是一种遵循 **后进先出 (LIFO, Last In First Out)** 原则的线性数据结构, 它只允许在一端 (称为 **栈顶**) 进行插入和删除操作. 一种经典的形象比喻是一叠盘子, 只能往盘子上面放新的盘子, 取用盘子时也只能从最上面的盘子开始取用

栈的应用可谓是相当多了, 最基本的有

- 函数调用栈: 后调用的函数先返回, 递归算法实现
- 表达式求值: 如括号匹配, 这也是我们这一小节需要实现的功能
- 撤销(Undo)操作: 如浏览器的前进/后退功能

栈的结构和数组类似, 也是一段连续线性的存储空间, 栈的操作也和数组类似, 主要就是 **推入(Push)** 和 **弹出(Pop)**, 但它只能在顶部推入新元素. 这使得栈可以非常高效的加入新元素, 因为我们可以很轻易的知道栈顶的位置

此外我们也需要一些其他的基本操作, 例如查看但不弹出栈顶元素(Peek), 判断栈是否为空, 清空栈等

我们会发现这些行为都和数组很像, 因此在这一节中, 我们先基于动态数组 `Vec` 实现一个基本的栈结构, 并基于这个栈实现一个括号匹配的功能

## 基于动态数组的栈

```rs
#[derive(Debug)]
struct Stack<T> {
    size: usize,
    data: Vec<T>,
}
```

一目了然, 不必多说, 然后我们实现一些基本功能, 其实这些功能都可以借助数组已有的功能

```rs
impl<T> Stack<T> {
    fn new() -> Self {
        Self {
            size: 0,
            data: Vec::new(),
        }
    }
    fn is_empty(&self) -> bool {
        self.size == 0
    }
    fn len(&self) -> usize {
        self.size
    }
    fn clear(&mut self) {
        self.size = 0;
        self.data.clear();
    }
    fn push(&mut self, val: T) {
        self.data.push(val);
        self.size += 1;
    }
    // 注意弹出时栈可能为空
    fn pop(&mut self) -> Option<T> {
        // TODO
        if self.size == 0 {
            None
        } else {
            self.size -= 1;
            self.data.pop()
        }
    }
    fn peek(&self) -> Option<&T> {
        if 0 == self.size {
            return None;
        }
        self.data.get(self.size - 1)
    }
    fn peek_mut(&mut self) -> Option<&mut T> {
        if 0 == self.size {
            return None;
        }
        self.data.get_mut(self.size - 1)
    }
}
```

一个基本的栈结构就完成了, 是不是很简单. 然后, 我们基于栈, 实现一个基本的括号匹配方法.

```rs
fn bracket_match(bracket: &str) -> bool {
    //TODO
    let mut stack = Stack::new();
    // 定义括号匹配规则
    let bracket_pairs = [('(', ')'), ('[', ']'), ('{', '}')];
    for c in bracket.chars() {
        // 如果是左括号，压入栈
        if ['(', '[', '{'].contains(&c) {
            stack.push(c);
        }
        // 如果是右括号, 先找到右括号对应的左括号
        else if let Some(&(left, _)) = bracket_pairs.iter().find(|&&(_, right)| right == c) {
            // 检查对应左括号与栈顶是否匹配
            if stack.pop() != Some(left) {
                return false;
            }
        }
    }
    // 最后检查栈是否为空
    stack.is_empty()
}
```

那么栈的这种结构天然适合关于栈顶的操作, 可如果我们就是想对栈中间的内容进行读取或修改呢? 还记得链表的部分吗, 我们同样可以为栈实现迭代器 Trait

```rs
// 获取所有权并消耗栈
pub struct IntoIter<T>(Stack<T>);
impl<T: Clone> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        if !self.0.is_empty() {
            self.0.size -= 1;
            self.0.data.pop()
        } else {
            None
        }
    }
}
// 只读栈
pub struct Iter<'a, T: 'a> {
    stack: Vec<&'a T>,
}
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<Self::Item> {
        self.stack.pop()
    }
}
// 可变内容栈
pub struct IterMut<'a, T: 'a> {
    stack: Vec<&'a mut T>,
}
impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;
    fn next(&mut self) -> Option<Self::Item> {
        self.stack.pop()
    }
}

// impl Stack

fn into_iter(self) -> IntoIter<T> {
    IntoIter(self)
}

fn iter(&self) -> Iter<T> {
    let mut iterator = Iter { stack: Vec::new() };
    for item in self.data.iter() {
        iterator.stack.push(item);
    }
    iterator
}

fn iter_mut(&mut self) -> IterMut<T> {
    let mut iterator = IterMut { stack: Vec::new() };
    for item in self.data.iter_mut() {
        iterator.stack.push(item);
    }
    iterator
}
```

其实本质上全算是将栈再次转化回数组了

## 基于队列(Queue)的栈

这里我们使用另一种方式来实现一个栈 -- **队列(Queue)**

首先让我们介绍一下队列. 顾名思义, 队列就是一条队伍, 同样是线性连续的数据结构, 不过和栈相反, 它的原则是 **先进先出(FIFO, First In First Out)**, 它允许在一端(称为队尾)进行插入操作, 在另一端(称为队头)进行弹出操作.

还记得吗, 我们在上一节实现 BFS 时已经用到过队列了, 那么队列的应用就包括

- 任务调度系统: 打印机任务队列
- 消息队列
- 广度优先搜索 (BFS) 算法
- 网络数据包缓冲

队列同样包括类似于数组的基本操作, 只有两个例外

- `enqueue`: 将元素加入队尾
- `dequeue`: 从队头移除元素

在这里, 我们同样使用动态数组作为队列的底层结构, 但注意, 由于底层结构的选择, 这里会导致队列弹出首元素的时间复杂度为 O(n), 因为需要弹出第一个元素

先给出基本数据结构

```rs
#[derive(Debug)]
pub struct Queue<T> {
    elements: Vec<T>,
}

impl<T> Queue<T> {
    pub fn new() -> Queue<T> {
        Queue {
            elements: Vec::new(),
        }
    }

    pub fn enqueue(&mut self, value: T) {
        self.elements.push(value)
    }

    pub fn dequeue(&mut self) -> Result<T, &str> {
        if !self.elements.is_empty() {
            // 移除队头元素(注意这是O(n)操作)
            Ok(self.elements.remove(0usize))
        } else {
            Err("Queue is empty")
        }
    }

    pub fn peek(&self) -> Result<&T, &str> {
        match self.elements.first() {
            Some(value) => Ok(value),
            None => Err("Queue is empty"),
        }
    }

    pub fn size(&self) -> usize {
        self.elements.len()
    }

    pub fn is_empty(&self) -> bool {
        self.elements.is_empty()
    }
}

impl<T> Default for Queue<T> {
    fn default() -> Queue<T> {
        Queue {
            elements: Vec::new(),
        }
    }
}
```

逻辑都比较简单, 然后给出使用双队列实现栈的方案, 其中一个是主队列, 一个是辅助队列, 具体谁是主队列取决于两个队列的内部情况

```rs
pub struct Stack<T> {
    q1: Queue<T>,
    q2: Queue<T>,
}
impl<T> Stack<T> {
    pub fn new() -> Self {
        Self {
            q1: Queue::<T>::new(),
            q2: Queue::<T>::new(),
        }
    }
    pub fn push(&mut self, elem: T) {
        //TODO
        // 压入元素时优先压入主队列
        // 哪个队列不是空的, 哪个队列就是主队列
        // 如果都是空的默认压入一号队列
        // pop 操作可以保证正常情况下两个队列至少有一个为空
        if !self.q1.is_empty() {
            self.q1.enqueue(elem);
        } else {
            self.q2.enqueue(elem);
        }
    }
    pub fn pop(&mut self) -> Result<T, &str> {
        //TODO
        if self.is_empty() {
            return Err("Stack is empty");
        }
        // 确定哪个队列是非空的(主队列)
        let (full, empty) = if !self.q1.is_empty() {
            (&mut self.q1, &mut self.q2)
        } else {
            (&mut self.q2, &mut self.q1)
        };
        // 将主队列中的元素(除了最后一个)全部转移到空队列
        // 可能你会觉得这很麻烦, 效率很低
        // 没错, 因为我们题目给出的队列的底层结构是动态数组
        // 弹出首个元素的时间复杂度是 O(n)
        // 如果我们的队列实现可以做到压入和弹出都是 O(1)
        // 那么你就能理解这种实现方案了
        while full.size() > 1 {
            if let Ok(val) = full.dequeue() {
                empty.enqueue(val);
            }
        }
        // 弹出并返回主队列的最后一个元素
        full.dequeue()
    }
    // 只有当两个队列都为空时栈为空
    pub fn is_empty(&self) -> bool {
        //TODO
        self.q1.is_empty() && self.q2.is_empty()
    }
}
```

为了方便理解, 我们这里举一些例子

```
现有数据 1, 2, 3, 4, 5, 按顺序压入, 假设队列是一个管道, 左进右出

队列 A: 5, 4, 3, 2, 1
队列 B:

为了先进后出, 后进先出, 即弹出 5, 我们需要让 5 右侧的元素不要挡着, 首先把 A 队列的 1 弹出来, 然后顺势压入 B 队列

队列 A: 5, 4, 3, 2
队列 B: 1

重复这一过程, 直到队列 A 只剩下一个元素

队列 A: 5
队列 B: 4, 3, 2, 1

这时再弹出队列 A 的最后一个元素, 我们就完成了出栈操作, 此时 A 为空, B 为新的主队列, 可以看到元素的顺序保持不变

而且对于一个好的队列实现, 压入和弹出的操作都是 O(1), 所以看起来麻烦, 实际上并不会有太大性能问题

比如说我们实现过的双链表作为队列, 再比如说, 我们前面看到过的标准库提供的 VecDeque
```

## 基于栈的队列

没错, 这两个东西彼此双生, 用栈实现队列和用队列实现栈都是经典的算法, 下面我们就用栈给出一种高效的队列实现方案

但是栈先进后出的特性实际上已经可以被数组模拟, 所以我们直接使用两个动态数组来实现一个队列, 下面先给出代码实现

```rs
#[derive(Debug)]
pub struct Queue<T> {
    in_stack: Vec<T>,   // 用于入队操作的栈
    out_stack: Vec<T>,  // 用于出队操作的栈
}

impl<T> Queue<T> {
    pub fn new() -> Self {
        Queue {
            in_stack: Vec::new(),
            out_stack: Vec::new(),
        }
    }

    /// 入队操作 - O(1)时间复杂度
    pub fn enqueue(&mut self, elem: T) {
        self.in_stack.push(elem);
    }

    /// 出队操作 - 均摊O(1)时间复杂度
    pub fn dequeue(&mut self) -> Option<T> {
        // 如果out_stack为空，将in_stack的所有元素转移到out_stack
        if self.out_stack.is_empty() {
            while let Some(elem) = self.in_stack.pop() {
                self.out_stack.push(elem);
            }
        }
        self.out_stack.pop()
    }

    /// 查看队首元素 - 均摊O(1)时间复杂度
    pub fn peek(&mut self) -> Option<&T> {
        // 同样需要先确保out_stack有元素
        if self.out_stack.is_empty() {
            while let Some(elem) = self.in_stack.pop() {
                self.out_stack.push(elem);
            }
        }
        self.out_stack.last()
    }

    /// 返回队列大小 - O(1)时间复杂度
    pub fn size(&self) -> usize {
        self.in_stack.len() + self.out_stack.len()
    }

    /// 检查队列是否为空 - O(1)时间复杂度
    pub fn is_empty(&self) -> bool {
        self.in_stack.is_empty() && self.out_stack.is_empty()
    }
}
```

我们可以看到, 有一个栈专门负责压入元素, 有一个栈专门负责弹出元素

我们也给出一个示例

```
现有数据 1, 2, 3, 4, 5, 现在我们按照数组顺序入栈, 右边是栈顶

s1: 1, 2, 3, 4, 5
s2:

现在我们需要弹出元素, 记住队列是先进先出, 所以我们需要弹出栈底的元素, 那么就把第一个栈所有元素逐个弹出, 并压入第二个栈

s1:
s2: 5, 4, 3, 2, 1

可以看到 s2 的数据被倒过来了, 这时只需要弹出 s2 的栈顶元素, 我们就弹出了第一个元素

那么此时怎么入队呢? 直接压入第一个栈就好了, 比如加入数据 6

s1: 6
s2: 5, 4, 3, 2

那么我们要再出队一个数据呢? 我们知道第二个栈已经是倒序了, 所以我们直接再次从栈顶弹出一个元素, 它就是倒数第二个元素
```

这样一来我们就实现了一个入队出队均摊下来都是 O(1) 的队列
