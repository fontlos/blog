---
feature: false
title: Rust 数据结构与算法(5) | 树结构
date: 2025-03-27 14:00:00
abstracts: 在这一节中, 我们将接触到几种常见树结构, 从最基本的二叉搜索树开始, 逐步到 AVL 树和红黑树等自平衡树结构
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

# 普通二叉搜索树 (Binary Search Tree, BST)

有了前面学堆排序的经验, 对二叉树的概念应该有一定了解了

二叉搜索树是一种特殊的二叉树数据结构, 其中每个节点都满足以下性质

- 左子树中所有节点的值都小于当前节点的值
- 右子树中所有节点的值都大于当前节点的值
- 左右子树也分别是二叉搜索树

二叉树有节点, 有连线, 其实也像是一个图

提到二叉就会提到分而治之的思想, 二叉搜索树具有以下特点

- 高效查找: 平均时间复杂度为 O(log n), 比线性结构更高效
- 动态数据维护: 可以高效地插入和删除数据
- 有序数据存储: 中序遍历 BST 可以得到有序序列
- 作为更复杂数据结构的基础: 如 AVL 树, 红黑树等都是基于 BST 的扩展

二叉搜索树在数据库索引, 文件系统目录结构, 内存中的有序数据存储, 路由算法中的路由表都有很多应用

下面我们来从零实现一个二叉搜索树

## 数据结构定义

那么首先肯定是数据结构, 二叉搜索树的节点类似于双链表, 为了在编译时确定数据类型大小, 我们同样需要使用智能指针将数据分配到堆上, 只不过这次我们不使用 `NonNull`, 而是 `Box` 智能指针, 之前说过, 它是一个胖指针, 存储了更多的元信息, 表示唯一所有权, 适合表示树节点的父子关系, 即父节点销毁时子节点一并销毁. 总之树结构天然适合所有权语义, 不需要频繁的节点共享, 并且使用 `Box` 实现简单且安全

```rs
#[derive(Debug)]
struct TreeNode<T>
where
    // 要求我们的元素是可排序的
    T: Ord,
{
    value: T,
    left: Option<Box<TreeNode<T>>>,
    right: Option<Box<TreeNode<T>>>,
}

impl<T> TreeNode<T>
where
    T: Ord,
{
    fn new(value: T) -> Self {
        TreeNode {
            value,
            left: None,
            right: None,
        }
    }
}
```

类似链表, 我们也需要一个结构来表示树根

```rs
#[derive(Debug)]
struct BinarySearchTree<T>
where
    T: Ord,
{
    root: Option<Box<TreeNode<T>>>,
}

impl<T> BinarySearchTree<T>
where
    T: Ord,
{
    fn new() -> Self {
        BinarySearchTree { root: None }
    }
}
```

## 基本功能实现

基本结构并不复杂, 接下来我们为其实现插入和搜索这两个最基本的功能, 至于修改和删除, 就交给读者自行实现

我们首先为树节点实现这两个功能, 因为树根本质上就是树节点的包装

```rs
use std::cmp::Ordering;

// impl TreeNode
fn insert(&mut self, value: T) {
    //TODO
    // 逻辑很清晰, 就像我们之前说的那样, 小值往左走, 大值往右走
    match value.cmp(&self.value) {
        Ordering::Less => {
            // 存在则递归调用
            if let Some(ref mut left) = self.left {
                left.insert(value);
            // 不存在就直接插入
            } else {
                self.left = Some(Box::new(TreeNode::new(value)));
            }
        }
        Ordering::Greater => {
            if let Some(ref mut right) = self.right {
                right.insert(value);
            } else {
                self.right = Some(Box::new(TreeNode::new(value)));
            }
        }
        Ordering::Equal => {
            // 重复值处理: 这里我们选择不插入重复值
            // 也可以根据需求选择其他处理方式
        }
    }
}

// 辅助函数
// 递归查找节点
fn search(&self, value: T) -> bool {
    // 搜索也是同理, 小值去左边找, 大值去右边找
    match value.cmp(&self.value) {
        Ordering::Less => self.left.as_ref().map_or(false, |left| left.search(value)),
        Ordering::Greater => self.right.as_ref().map_or(false, |right| right.search(value)),
        Ordering::Equal => true,
    }
}
```

然后我们只需要为上层包装也实现这两个功能即可

```rs
fn insert(&mut self, value: T) {
    //TODO
    if let Some(ref mut root) = self.root {
        root.insert(value);
    } else {
        self.root = Some(Box::new(TreeNode::new(value)));
    }
}

fn search(&self, value: T) -> bool {
    //TODO
    self.root.as_ref().map_or(false, |root| root.search(value))
}
```

就像我们说的, 二叉搜索树非常的简单, 但也非常的基础

# 什么是自平衡二叉搜索树

了解了普通的二叉搜索树, 下面我们可以进一步认识 **自平衡** 二叉搜索树了, 所谓自平衡, 就是通过一系列操作, 使树结构整体尽可能对称平衡

为什么需要自平衡二叉搜索树?

普通 BST 在极端情况下会退化成链表 (时间复杂度从O(log n) → O(n)), 例如按顺序插入 `1, 2, 3, 4, 5`, 这将永远插入右侧节点, 形成一条长链

而自平衡二叉搜索树通过 **自动调整树的结构** (旋转, 重新着色等操作), 始终保持树的平衡性, 确保最坏情况下操作时间复杂度仍为 O(log n)

这里我们主要介绍两种, **AVL 树** 与 **红黑树** (其中 AVL 只是发明人名字的缩写, 纪念 **G.M. Adelson-Velsky 和 E.M. Landis** 首次提出平衡二叉树的概念)

| 特性             | AVL树                      | 红黑树                  |
|-----------------|----------------------------|------------------------|
| 平衡标准         | 严格平衡 (左右子树高度差≤1)    | 弱平衡 (最长路径≤2倍最短路径) |
| 旋转频率         | 高 (插入/删除可能频繁调整)     | 低 (颜色翻转多于旋转)        |
| 查找效率         | 更优 (严格平衡)              | 稍逊                       |
| 插入/删除效率    | 平均需要更多旋转               | 更快 (适合频繁修改的场景)    |
| 典型应用         | 需要快速查找的场景             | 需要频繁插入删除的场景      |

# AVL 树

AVL 树依靠**平衡因子 (Balance Factor)** 来调整树的结构, 其中 `balance_factor = height(left) - height(right) ∈ {-1, 0, 1}`, 当插入/删除导致 ``|balance_factor| > 1`` 时, 通过 **旋转** 恢复平衡

我们先来介绍何为旋转,首先容易想到我们有四种旋转场景

|失衡情况|旋转方式|示意图|
|-----------------|--------------|------------------------|
|左子树更高且左左插入|  右旋         |  `[y] → [x]` 的左左结构|
|右子树更高且右右插入|  左旋         |  `[x] → [y]` 的右右结构|
|左子树更高且左右插入|  先左旋再右旋  |  `[z] → [y] → [x]` 的左右结构|
|右子树更高且右左插入|  先右旋再左旋  |  `[x] → [y] → [z]` 的右左结构|

先看两个含子树的完整过程

```
左子树 y 失衡
       y (bf=2)
      /
     x (bf=1)
    / \
   z   s

右旋, 新根右子节点成为原根左子节点
     x
    / \
   z   y
      /
     s

想象一下, y 垂了下来, 但 x 不能下分三个节点, 于是 s 顺势断开, 向右依附到 y 上
```

```
右子树 x 失衡
   x (bf=-2)
    \
     y (bf=-1)
    / \
   s   z

左旋
     y
    / \
   x   z
    \
     s
```

不过因为我们这里只实现插入操作, 所以你在实际操作中没有那么多子树, 看起来就像是之发生了节点位置旋转而没发生子树重新挂载, 比如对于下面这种情况的简化过程

```
z 失衡, 先左后右
     z (bf=2)
    /
   x (bf=-1)
    \
     y

先对 x 左旋后
     z
    /
   y
  /
 x

再对 z 右旋后
     y
    / \
   x   z
```

## 数据结构扩展

首先给我们的原始结构新增字段代表子树的高度

```rs
#[derive(Debug)]
struct TreeNode<T: Ord> {
    value: T,
    left: Option<Box<TreeNode<T>>>,
    right: Option<Box<TreeNode<T>>>,
    height: usize, // 新增: 节点高度
}

impl<T> TreeNode<T>
where
    T: Ord,
{
    fn new(value: T) -> Self {
        TreeNode {
            value,
            left: None,
            right: None,
            height: 1,
        }
    }
}
```

## 旋转相关操作

接下来我们实现具体功能, 一旦理解了 AVL 树的思想, 实现方案就不难了, 下面的内容都是为 `TreeNode` 实现的

首先是关于树高度的一些操作

```rs
/// 子树高度
fn height(node: &Option<Box<TreeNode<T>>>) -> usize {
    // 空节点高度为 0
    node.as_ref().map_or(0, |n| n.height)
}
/// 更新节点高度
fn update_height(&mut self) {
    self.height = 1 + Self::height(&self.left).max(Self::height(&self.right));
}
```

然后是计算平衡因子, 算法很简单

```rs
fn balance_factor(&self) -> i32 {
    Self::height(&self.left) as i32 - Self::height(&self.right) as i32
}
```

然后我们实现上述的那些旋转操作, 实际上只需要左旋和右旋, 另外两种可以组合得到

```rs
/// 右旋操作
fn rotate_right(mut self: Box<TreeNode<T>>) -> Box<TreeNode<T>> {
    // 用左节点作为新的子树树根
    let mut new_root = self.left.take().unwrap();
    self.left = new_root.right.take();
    self.update_height();
    new_root.right = Some(self);
    new_root.update_height();
    new_root
}

/// 左旋操作
fn rotate_left(mut self: Box<TreeNode<T>>) -> Box<TreeNode<T>> {
    let mut new_root = self.right.take().unwrap();
    self.right = new_root.left.take();
    self.update_height();
    new_root.left = Some(self);
    new_root.update_height();
    new_root
}
```

然后是具体的旋转逻辑

```rs
/// 平衡调整主逻辑
fn balance(mut self: Box<TreeNode<T>>) -> Box<TreeNode<T>> {
    let bf = self.balance_factor();
    // Left-Left 情况
    // 左子树更高且左子树的左子树不矮
    if bf > 1 && self.left.as_ref().unwrap().balance_factor() >= 0 {
        return self.rotate_right();
    }
    // Left-Right 情况
    // 左子树更高且左子树的右子树更高
    if bf > 1 && self.left.as_ref().unwrap().balance_factor() < 0 {
        self.left = Some(self.left.take().unwrap().rotate_left());
        return self.rotate_right();
    }
    // Right-Right 情况
    // 右子树更高且右子树的右子树不矮
    if bf < -1 && self.right.as_ref().unwrap().balance_factor() <= 0 {
        return self.rotate_left();
    }
    // Right-Left 情况
    // 右子树更高且右子树的左子树更高
    if bf < -1 && self.right.as_ref().unwrap().balance_factor() > 0 {
        self.right = Some(self.right.take().unwrap().rotate_right());
        return self.rotate_left();
    }

    // 无需旋转
    self
}
```

## 添加新的接口

最后提供一个接口用于插入新元素

```rs
/// AVL树专用插入方法
/// 因为每次插入的平衡操作可能会消耗掉原根节点, 所以我们需要返回新的根节点
fn insert_avl(mut self: Box<TreeNode<T>>, value: T) -> Box<TreeNode<T>> {
    // 1. 标准BST插入
    match value.cmp(&self.value) {
        Ordering::Less => {
            self.left = match self.left {
                Some(left) => Some(left.insert_avl(value)),
                None => Some(Box::new(TreeNode::new(value))),
            };
        }
        Ordering::Greater => {
            self.right = match self.right {
                Some(right) => Some(right.insert_avl(value)),
                None => Some(Box::new(TreeNode::new(value))),
            };
        }
        Ordering::Equal => return self, // 重复值不插入
    };

    // 2. 更新当前节点高度
    self.update_height();

    // 3. 平衡调整
    self.balance()
}

// 别忘了在 BinarySearchTree 结构同样提供一个接口

/// AVL 树插入入口
pub fn insert_avl(&mut self, value: T) {
    self.root = match self.root.take() {
        Some(root) => Some(root.insert_avl(value)),
        None => Some(Box::new(TreeNode::new(value))),
    };
}
```

然后我们可以尝试一些测试

```rs
#[test]
fn test_avl_insert() {
    let mut avl = BinarySearchTree::new();
    avl.insert_avl(3);
    avl.insert_avl(2);
    avl.insert_avl(1); // 触发右旋
    assert_eq!(avl.root.as_ref().unwrap().value, 2);
    assert_eq!(avl.root.as_ref().unwrap().height, 2);

    avl.insert_avl(4);
    avl.insert_avl(5); // 触发左旋
    assert_eq!(avl.root.as_ref().unwrap().right.as_ref().unwrap().value, 4);
}
```

# 红黑树

其实红黑树顾名思义, 就是对节点提出了颜色的概念, 相比AVL树的严格平衡, 红黑树提供了一种弱平衡方案, 仍且能保证插入/删除平均只需O(1)次旋转和 O(log n) 的查询效率

红黑树有如下五条约束

- 每个节点是红色或黑色
- 根节点必须是黑色
- 所有叶子节点 (特殊的 Nil 节点, 只作为占位符) 是黑色
- 红色节点的子节点必须为黑色 (不能有连续红节点)
- 从任一节点到其叶子节点的路径包含相同数量的黑色节点 (不包含节点自身, 但包含叶子节点)(黑高相同)

红黑树的本质实际上是一种多叉树 2-3-4 树 的二叉等价形式, 不过在这里我们不细究了, 只是简单介绍一下为什么这五条原则能保证红黑树的弱平衡, 以及操作的具体细节

这五条约束的目的是尝试在严格平衡 (AVL) 和完全不平衡 (普通BST) 之间找到一个平衡, 允许最长路径(红黑相间)不超过最短路径(纯黑)的 2 倍, 通过颜色约束 (无连续红节点) 和黑高一致保证基本平衡

- 根节点黑: 所有路径起点一致
- 红节点子节点黑: 控制路径膨胀
- 黑高相同: 限制最长路径 (平衡核心)

红黑树同样拥有旋转操作, 旋转的触发条件就取决于上述约束, 不过除了旋转, 红黑树还需要 **修复**, 就如同堆需要堆化来维持堆的性质, 红黑树也需要颜色修复来维持红黑树的性质, 并且修复的触发条件拥有优先级

旋转过程中可以遵循两条规则

- 新根继承原根颜色: 维持全局颜色分布
- 原根降级为红色: 避免黑高增加

1. 左倾修复 (Left-Leaning Fix)
    - 条件: 当前节点的右子为红且左子为黑
    - 操作: 左旋当前节点
    - 目的: 将右红子节点转为左红, 维持左倾倾向
    - 理由: 红黑树倾向于左倾结构, 右红子节点可能导致后续插入破坏平衡

```
旋转, 并将新根左节点挂在原根右节点
修复前:
      B [z]
       \
       R [y]
      /
    R [x]

左旋后:
      B [y]
     /
   R [z]
     \
      R [x]
```

2. 连续左红修复 (Left-Red Conflict)
    - 条件: 当前节点的左子和左孙均为红
    - 操作: 右旋当前节点
    - 目的: 消除连续左红节点 (对应 2-3 树中临时 4-节点)
    - 理由: 连续两个左红节点会导致黑高计算失衡, 右旋将其转为平衡的临时 4-节点结构

```
旋转, 并将新根右节点挂在原根左节点
修复前:
      B [z]
     /
   R [y]
   /
 R [x]

右旋后:
      B [y]
        \
       R [z]
        /
      R [x]
```

3. 颜色翻转 (Color Flip)
    - 条件: 当前节点的左右子节点均为红
    - 操作: 将子节点变黑, 当前节点变红
    - 目的: 模拟 2-3-4 树的节点分裂
    - 理由: 将临时 4-节点 (两个红子) 拆分为三个 2-节点, 红色上溢可能传递到上层继续修复

```
翻转前:
      B [y]
     / \
   R [x] R [z]

翻转后:
      R [y]
     / \
   B [x] B [z]
```

下面是一段完整过程, 插入 `[3, 1, 5, 2]`, 同样因为我们这里只实现插入操作, 在实际操作中没有那么多子树, 看起来就像是之发生了节点位置旋转而没发生子树重新挂载

```
我们先让新节点默认为红色
R3

然后强制根节点为黑色
B3

插入新节点, 仍然默认为红色
  B3
 /
R1

再插入新节点
  B3
 / \
R1 R5

触发操作 3, 左右子节点都为红色, 反转颜色
  R3
 / \
B1 B5

然后强制根节点为黑色
  B3
 / \
B1 B5

插入最后一个节点, 触发左旋, B1 节点的右子为红且左子为黑(Nil)
    B3
   / \
  B1 B5
   \
   R2

左旋
    B3
   / \
  R2 B5
 /
B1
```

## 数据结构扩展

我们还是从原始的二叉搜索树开始, 首先扩展数据结构

```rs
#[derive(Debug, PartialEq)]
enum Color {
    Red,
    Black,
}

#[derive(Debug)]
struct TreeNode<T: Ord> {
    value: T,
    left: Option<Box<TreeNode<T>>>,
    right: Option<Box<TreeNode<T>>>,
    color: Color, // 新增颜色标记
}

impl<T: Ord> TreeNode<T> {
    fn new(value: T) -> Self {
        TreeNode {
            value,
            left: None,
            right: None,
            color: Color::Red, // 新节点默认为红色
        }
    }
}
```

## 旋转与修复

同样的, 接下来我们要在 `TreeNode` 上实现一系列操作

```rs
/// 判断节点是否为红色 (空节点视为黑色)
fn is_red(node: &Option<Box<Self>>) -> bool {
    node.as_ref().map_or(false, |n| n.color == Color::Red)
}

/// 修复红黑树性质的三种情况
fn fixup(mut self: Box<Self>) -> Box<Self> {
    // 右子红且左子黑
    if Self::is_red(&self.right) && !Self::is_red(&self.left) {
        self = self.rotate_left();
    }
    // 左子红且左子的左子红
    if Self::is_red(&self.left) && Self::is_red(&self.left.as_ref().unwrap().left) {
        self = self.rotate_right();
    }
    // 左右子均红
    if Self::is_red(&self.left) && Self::is_red(&self.right) {
        self.flip_colors();
    }
    self
}

/// 左旋操作
fn rotate_left(mut self: Box<Self>) -> Box<Self> {
    let mut new_root = self.right.take().unwrap();
    self.right = new_root.left.take();
    new_root.color = self.color;
    self.color = Color::Red;
    new_root.left = Some(self);
    new_root
}

/// 右旋操作
fn rotate_right(mut self: Box<Self>) -> Box<Self> {
    let mut new_root = self.left.take().unwrap();
    self.left = new_root.right.take();
    new_root.color = self.color;
    self.color = Color::Red;
    new_root.right = Some(self);
    new_root
}

/// 颜色翻转
fn flip_colors(&mut self) {
    self.color = match self.color {
        Color::Red => Color::Black,
        Color::Black => Color::Red,
    };
    self.left.as_mut().unwrap().color = Color::Black;
    self.right.as_mut().unwrap().color = Color::Black;
}
```

## 添加新的接口

最后同样提供一个接口用于插入新元素

```rs
/// 红黑树插入入口
pub fn insert_rb(mut self: Box<Self>, value: T) -> Box<Self> {
    match value.cmp(&self.value) {
        Ordering::Less => {
            self.left = match self.left {
                Some(left) => Some(left.insert_rb(value)),
                None => Some(Box::new(Self::new(value))),
            };
        }
        Ordering::Greater => {
            self.right = match self.right {
                Some(right) => Some(right.insert_rb(value)),
                None => Some(Box::new(Self::new(value))),
            };
        }
        Ordering::Equal => return self, // 重复值不插入
    }
    self.fixup() // 插入后修复红黑树性质
}

// 别忘了在 BinarySearchTree 结构同样提供一个接口

pub fn insert_rb(&mut self, value: T) {
    match self.root.take() {
        Some(root) => {
            let mut new_root = root.insert_rb(value);
            new_root.color = Color::Black; // 根节点始终为黑
            self.root = Some(new_root);
        }
        None => {
            let mut node = TreeNode::new(value);
            node.color = Color::Black; // 根节点强制为黑
            self.root = Some(Box::new(node));
        }
    }
}
```
