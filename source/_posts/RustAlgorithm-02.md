---
feature: false
title: Rust 数据结构与算法(2) | 堆
date: 2025-03-26 16:00:00
abstracts: 堆是一种特殊的完全二叉树数据结构, 在这一节中, 我们将实现一种底层的二叉堆, 学习什么是堆化, 并可以自由选择是最大堆还是最小堆模式, 在后续的排序算法课程中也将基于此实现一个简单的堆排序
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

# 堆

堆是一种特殊的完全二叉树数据结构, 在这一节中, 我们将实现一种底层的二叉堆, 学习什么是堆化, 并可以自由选择是最大堆还是最小堆模式, 在后续的排序算法课程中也将基于此实现一个简单的堆排序

- **堆(Heap)**: 一种树形结构, 我们这里是 **二叉堆**, 即个父节点至多有两个子节点
- **堆化(Heapify)**: 调整堆使其满足堆的性质
    - **最大堆(max-heap)**: 每个节点的值都大于或等于其子节点的值
    - **最小堆(min-heap)**: 每个节点的值都小于或等于其子节点的值
- **父/子节点**: 这个很好理解, 树型结构中上层是父节点, 下层是子节点

堆通常用数组来实现, 因为完全二叉树的特性使得我们可以用简单的索引关系来表示父子节点关系

核心操作

- 插入(Add): `O(log n)`
- 删除(Next/Extract): `O(log n)`
- 查看堆顶(Peek): `O(1)`

索引计算: 我们这里以 `0` 为根节点, 所以实际计算中看上去少加了 `1`, 以及我们默认不使用 `0` 号位置, 直接用默认值占位

- 父节点索引: `parent_idx = idx / 2`
- 左子节点索引: `left_child_idx = idx * 2`
- 右子节点索引: `right_child_idx = idx * 2 + 1`

维护堆的性质关键就在于 **上浮(Swim)** 和 **下沉 (Sink)** 操作, 就是让我们需要的元素按某种顺序向上移动或向下移动

至于堆的常见应用, 除开堆排序外, 还有

- 优先队列实现: 一种抽象数据类型, 每个元素都有优先级, 优先级高的元素先出队
- 图算法中的 **Dijkstra (最短路径)** 和 **Prim (最小生成树)** 算法
- 求 **Top K** 问题: 从海量数据中找出前 `K` 大/小的元素
- 中位数查找问题: 两个堆, 最大堆存较小的一半数, 最小堆存较大的一半数, 保持两堆大小平衡 (差值 ≤ 1). 若两堆大小相等, 取两个堆顶的平均值, 否则取元素多的那个堆的堆顶

## 基本数据结构与基本方法

```rs
pub struct Heap<T>
where
    T: Default,
{
    count: usize,   // 元素数量
    items: Vec<T>,  // 实际存储
    comparator: fn(&T, &T) -> bool, // 比较函数, 用于决定是最大堆还是最小堆
    // 这是题目中给定的定义
}
```

然后根据上面的定义, 我们给出基本方法

```rs
impl<T> Heap<T>
where
    T: Default,
{
    pub fn new(comparator: fn(&T, &T) -> bool) -> Self {
        Self {
            count: 0,
            // 初始化时用默认值占位, 所以我们需要的堆顶在 1 号位置
            items: vec![T::default()],
            comparator,
        }
    }

    pub fn len(&self) -> usize {
        self.count
    }

    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }

    fn parent_idx(&self, idx: usize) -> usize {
        idx / 2
    }

    // 检查给定节点是否有至少一个子节点, 对于上浮和下沉操作至关重要
    // 左子节点不存在那么一定不存在右子节点
    fn children_present(&self, idx: usize) -> bool {
        self.left_child_idx(idx) <= self.count
    }

    fn left_child_idx(&self, idx: usize) -> usize {
        idx * 2
    }

    fn right_child_idx(&self, idx: usize) -> usize {
        self.left_child_idx(idx) + 1
    }
}
```

## 关键方法实现

因为我们这里的堆是从零构建的, 和堆排序自下而上堆化不同, 我们只要在每次插入或弹出新元素时就维护好堆的秩序即可 (即上浮下沉)

那么这里给出添加新元素的方法, 在这里我们首先将元素添加到末尾, 然后层层比较, 让元素上浮, 最终位于正确位置. 这次我们就使用循环来解决, 而在后面的堆排序中我们会使用递归的方式

```rs
pub fn add(&mut self, value: T) {
    self.items.push(value);
    self.count += 1;
    let mut idx = self.count;  // 新元素的索引

    // 上浮过程
    while idx > 1 {
        let parent_idx = self.parent_idx(idx);
        // 调用比较函数
        if (self.comparator)(&self.items[idx], &self.items[parent_idx]) {
            self.items.swap(idx, parent_idx);
            // 如果一直能比较, 那么最多比较到节点 1 比较就结束了
            idx = parent_idx;
        } else {
            // 如果以及找到正确位置了, 那么就停下来
            break;
        }
    }
}
```

我们用一段例子来解释这个过程, 比如有一个如下的最小堆

```
       1 (idx = 1)
     /   \
    3     5 (idx = 3)
   / \
  4   8 (idx = 4,5)
```

现在插入数据 2

```
      1
    /   \
   3     5
  / \   /
 4   8 2 (idx = 6)
```

比较 idx(6) 和父节点 idx(3), 发现需要上浮, 交换

```
      1
    /   \
   3     2 (idx = 3)
  / \   /
 4   8 5 (idx = 6) 这里交换了节点索引
```

然后继续尝试和父节点比较, 发现无需移动, 节点插入成功

## 为堆结构实现迭代器

然后, 为了能更好的访问堆, 比如我们可能需要按从大到小的顺序读取堆的元素, 我们可能需要对它进行迭代, 所以我们也为堆实现一个迭代器 Trait, 不过在此之前, 我们还需要实现一个方法, 用于找到当前节点最小(或最大)的子节点, 函数名是题目中给出来的, 我就不改了. 这个方法对于实现下沉操作很重要, 而实现迭代器就需要这种下沉操作

```rs
fn smallest_child_idx(&self, idx: usize) -> usize {
    let left = self.left_child_idx(idx);
    let right = self.right_child_idx(idx);

    // 右节点不存在那么左节点就是我们需要的, 不论是找最大的还是最小的
    if right > self.count {
        left
    } else {
        // 调用我们的比较函数, 也许是找小的, 也许是找大的
        if (self.comparator)(&self.items[left], &self.items[right]) {
            left
        } else {
            right
        }
    }
}
```

现在我们可以实现一个迭代器了, 先给出代码

```rs
impl<T> Iterator for Heap<T>
where
    T: Default,
{
    type Item = T;

    // 取出堆顶元素 (索引1的元素)
    // 此时堆没有顶部节点了, 所以将最后一个元素移到堆顶, 简单方便
    // 执行 "下沉"(sink) 操作, 恢复堆的性质
    fn next(&mut self) -> Option<T> {
        //TODO
        if self.count == 0 {
            return None;
        }

        // 取出堆顶元素
        // 与 remove 不同, swap_remove 将取出指定元素并使用最后一个元素取代它, 而不是移动后续所有元素
        // 有更高的性能, 平均 O(1)
        let top = self.items.swap_remove(1);
        self.count -= 1;

        if self.count > 0 {
            // 下沉过程, 从堆顶开始
            let mut idx = 1;
            // 如果存在子节点
            while self.children_present(idx) {
                // 获取我们需要的最大或最小子节点
                let child_idx = self.smallest_child_idx(idx);
                // 如果符合要求就下沉
                if !(self.comparator)(&self.items[idx], &self.items[child_idx]) {
                    self.items.swap(idx, child_idx);
                    idx = child_idx;
                } else {
                    break;
                }
            }
        }

        Some(top)
    }
}
```

我们还是给出一个示例来, 依然是之前的最小堆

```
       1
     /   \
    3     2
   / \   /
  4   8 5
```

首先弹出 `idx(1)`, 即数据 1, 然后用最后一个元素 idx(6) 即数据 5 来替代原来堆顶的位置

```
       5
     /   \
    3     2
   / \
  4   8
```

然后我们需要自上而下的比较这对堆造成的影响, 把错误的元素进行下沉, 首先比较 idx(1) 和它的最小的子节点 idx(3) 即 2, 发现需要移动

```
       2
     /   \
    3     5 (交换节点序号)
   / \
  4   8
```

然后检查移动过的节点, 这里是 idx(3), 看看之前的移动有没有破坏它的只需, 发现并没有, 因为它已经没有子节点了, 即使有也会循环继续交换. 因此此时堆已经恢复秩序, 无需调整

最后, 我们只需要再用一些结构包裹这个堆, 就可以实现具体的最大堆和最小堆了

```rs
pub struct MinHeap;

impl MinHeap {
    #[allow(clippy::new_ret_no_self)]
    pub fn new<T>() -> Heap<T>
    where
        T: Default + Ord,
    {
        Heap::new(|a, b| a < b)
    }
}

pub struct MaxHeap;

impl MaxHeap {
    #[allow(clippy::new_ret_no_self)]
    pub fn new<T>() -> Heap<T>
    where
        T: Default + Ord,
    {
        Heap::new(|a, b| a > b)
    }
}

// 或者

impl<T> Heap<T>
where
    T: Default + Ord,
{
    /// Create a new MinHeap
    pub fn new_min() -> Self {
        Self::new(|a, b| a < b)
    }

    /// Create a new MaxHeap
    pub fn new_max() -> Self {
        Self::new(|a, b| a > b)
    }
}
```

## Top-K 算法

最后让我们使用我们的堆结构实现一个 Top-K-Min 函数, 找出一组数值之中最小的 K 个值

```rs
/// 获取最小的 K 个元素
pub fn top_k_min<T: Ord + Default>(nums: Vec<T>, k: usize) -> Vec<T> {
    if k == 0 || nums.is_empty() {
        return vec![];
    }

    let mut heap = MaxHeap::new(); // 最大堆保留最小的 K 个元素

    for num in nums {
        if heap.len() < k {
            heap.add(num);
        } else {
            // 只要比堆顶小就踢掉堆顶自己成为新的堆顶. 然后重新堆化
            if &num < heap.items.get(1).unwrap() {
                heap.next(); // 移除当前堆顶
                heap.add(num);
            }
        }
    }

    // 由于是最大堆, 所以最终结果将是降序
    heap.collect()
}
```

然后给出一些测试用例

```rs
#[test]
fn test_top_k_min() {
    let nums = vec![4, 1, 3, 12, 7, 5];
    // 由于最大堆的行为会导致结果是反转的
    assert_eq!(top_k_min(nums.clone(), 3), vec![4, 3, 1]);
    assert_eq!(top_k_min(nums.clone(), 5), vec![7, 5, 4, 3, 1]);
    assert_eq!(top_k_min(nums.clone(), 0), vec![]);
}

#[test]
fn test_edge_cases() {
    // 元素全部相同
    assert_eq!(top_k_min(vec![2, 2, 2], 2), vec![2, 2]);
    // K 大于数组长度
    assert_eq!(top_k_min(vec![1, 2], 5), vec![2, 1]);
}
```