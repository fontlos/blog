---
feature: false
title: Rust 数据结构与算法(2) | 排序
date: 2025-03-26 16:00:00
abstracts: 排序是值得单开一节的内容, 因为排序方案种类众多, 思路也比较复杂, 在这一节将介绍冒泡排序, 插入排序, 快速排序, 堆排序, 以及一个简化版的 TimSort
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

排序是值得单开一节的内容, 因为排序方案种类众多, 思路也比较复杂, 在这一节将介绍冒泡排序, 插入排序, 快速排序, 堆排序, 以及一个简化版的 TimSort

# 冒泡排序 (Bubble Sort)

这可以说是思路上最简单的排序方案之一了 (睡眠排序和猴子排序这种异类不算)

冒泡排序通过重复地遍历要排序的列表, 比较相邻元素并交换它们的位置 (如果顺序错误) 来工作, 这个过程重复进行, 直到列表被排序

```rs
fn bubble_sort<T: Ord>(array: &mut [T]) {
    for i in 0..array.len() {  // 外层循环控制排序轮数
        for j in 0..array.len() - i - 1 {  // 内层循环控制每轮比较次数
            if array[j] > array[j + 1] {  // 如果前一个元素大于后一个
                array.swap(j, j + 1);    // 交换它们的位置
            }
        }
    }
}
```

- 外层循环: 控制排序轮数, 每轮会将当前最大的元素 **冒泡** 到正确位置
- 内层循环: 比较相邻元素, 将较大的元素向后移动

时间复杂度为 O(n²)

# 插入排序 (Insertion Sort)

插入的思想也很简单, 冒泡是将最大的元素放到正确的未知, 插入是将当前元素放到相对正确的位置, 通过构建有序序列, 对于未排序数据, 在已排序序列中从后向前扫描, 找到相应位置并插入

```rs
fn insertion_sort<T: Ord>(array: &mut [T]) {
    for i in 1..array.len() {  // 从第二个元素开始 (索引1)
        let mut j = i;  // j 是当前要插入的元素位置
        while j > 0 && array[j] < array[j - 1] {  // 当前元素比前一个小
            array.swap(j, j - 1);  // 交换它们
            j -= 1;  // 继续向前比较
        }
    }
}
```

- 外层循环: 遍历每个待插入的元素, 假定 0 号位置就是最初的的有序部分
- 内层循环: 将当前元素与已排序部分比较并交换, 直到找到正确位置

时间复杂度为 O(n²)

# 快速排序 (Quick Sort)

这是一个经典且常用的排序方案, 到了这以后, 算法开始上难度了

核心思想是 **分而治之**: 选择一个 "基准" 元素, 将数组分为两个子数组, 小于基准的和大于基准的, 然后递归排序子数组, 递归的过程就是继续分治的过程

```rs
fn quick_sort<T: Ord>(array: &mut [T]) {
    if array.len() <= 1 {  // 基本情况: 空数组或单元素数组已排序
        return;
    }
    let pivot = partition(array);  // 获取基准位置
    quick_sort(&mut array[..pivot]);  // 递归排序左半部分
    quick_sort(&mut array[pivot + 1..]);  // 递归排序右半部分
}

fn partition<T: Ord>(array: &mut [T]) -> usize {
    let pivot = array.len() - 1;  // 选择最后一个元素作为基准
    let mut i = 0;  // i 是小于基准的元素的边界, 为了避免每次都移动基准
    for j in 0..pivot {  // 遍历除基准外的所有元素
        if array[j] <= array[pivot] {  // 当前元素小于等于基准
            array.swap(i, j);  // 把它放到i的位置
            i += 1;  // 移动i边界
        }
    }
    array.swap(i, pivot);  // 把基准放到正确位置
    i  // 返回基准的最终位置
}
```

`partition` 函数: 将数组分为两部分, 返回基准的最终位置
`quick_sort` 递归: 对左右两部分分别递归排序
`array.swap(i, j)`: 交换元素位置以维持分区不变式

时间复杂度：平均 O(nlogn), 最坏 O(n²) (当分区极度不平衡时)

# 堆排序 (Heap Sort)

我先用尽可能通俗易懂的方式解释一下堆排序

假如这里有一些大小不一的球, 我们尝试给球按大小排序, 首先我们把这些球堆在一起(**构建堆**), 像一个金字塔(**最大堆**), 最大的球总是在最上面(**大顶堆排序**, 同样的你也可以构建小顶堆)

然后(**开始排序**), 把最上面的球(**最大值**)拿走, 放到一边. 把最后一个球放到最上面(**交换最大值与数组末尾**), 重新调整球堆(**待排序数组长度减一**), 让最大的球再次位于最上面, 重复这个过程

堆排序就是利用堆这种数据结构: 首先构建最大堆, 然后重复从堆中提取最大元素放到数组末尾

然后先解释几个专有名词

- **堆(Heap)**: 一种树形结构, 最大堆指每个父节点都比子节点大, 我们这里使用 **二叉堆**, 即个父节点至多有两个子节点
- **堆化(Heapify)**: 调整堆使其满足堆的性质
- **父/子节点**: 这个很好理解, 树型结构中上层是父节点, 下层是子节点

```rs
// 这次我们先看辅助函数, 负责将数组的一部分 "堆化"
fn heapify<T: Ord>(array: &mut [T], root: usize, end: usize) {
    let mut largest = root;  // 假设根节点最大
    // 因为通常来说, 每层元素的个数是上一层的两倍, 所以上层节点的序号的二倍就是本层节点序号的开始
    // 通过 +1 和 +2 获得本层两个节点的索引
    // 这相当于将一个二叉堆平铺在数组上
    let left = 2 * root + 1;  // 左子节点索引
    let right = 2 * root + 2;  // 右子节点索引

    // 我们只处理这一层, 用 end 作为边界
    // 找出 root, left, right 中最大的
    if left < end && array[left] > array[largest] {
        largest = left;
    }
    if right < end && array[right] > array[largest] {
        largest = right;
    }

    if largest != root {  // 如果最大不是 root
        array.swap(root, largest);  // 交换它们
        heapify(array, largest, end);  // 递归堆化受影响的子树
    }
}

fn heap_sort<T: Ord>(array: &mut [T]) {
    if array.len() <= 1 {  // 基本情况
        return;
    }

    // 在这里, 我们相对于把数组结构视为一颗无序二叉树
    // 构建最大堆
    for i in (0..array.len() / 2).rev() {  // 从最后一个非叶子节点开始
        heapify(array, i, array.len());  // 堆化
    }

    // 提取元素
    for i in (1..array.len()).rev() {  // 从后往前
        array.swap(0, i);  // 把当前最大元素放到数组末尾
        heapify(array, 0, i);  // 对剩余元素重新堆化
    }
}
```

- `heapify` 函数: 维护堆的性质, 确保父节点大于子节点
- 构建堆: 从最后一个非叶子节点开始向前堆化
- 排序阶段: 重复提取最大元素并堆化剩余部分

时间复杂度: O(nlogn)

然后我们再用实际例子分析一下这个过程, 先介绍排序函数, 因为一旦理解了排序函数, 通过层层递归, 堆化函数一定能变成给一个仅包含一个父节点和至多两个子节点的堆进行堆化

前几行不用解释, 从构建最大堆开始, 首先是如何找到最后一个非叶子节点, 所谓叶子节点就是没有子节点的节点, 举个例子, 假设我们有这样一个数组

```
[3, 7, 5, 9, 4, 8, 2]
```

我们直接按数组顺序将其视作二叉树

```
        3
      /   \
     7     5
    / \   / \
   9   4 8   2
```

对于最后一个节点 `n-1` (这里 n 为 7, 所以我们的索引为 6), 我们可以计算它的父节点, 也就是最后一个非叶子节点 `(n-2)/2`(注意这里是整数除法, 四舍五入) (对应索引 2, 对应数据 `5`)

至于为什么这样计算, 假设最后一个非叶子节点有两个子节点, 那么根据计算公式就可以知道, 它的父节点是 `(n-2)/2`, 如果最后一个非叶子节点有一个子节点, 那么我们可以知道 `(n-1)/2` 才是正确结果, 可是由于四舍五入, `(n-1)/2 = (n-2)/2 + 1/2`, 那么 `(n-2)/2` 比正确结果少了 `1/2`, 但正好可以通过四舍五入得到正确结果. 因此无论如何, `n/2 - 1` 都将正确的得到最后一个非叶子节点, 又因为 `..` 不包含结束值, 所以 `(0..array.len() / 2).rev()` 就将从最后一个非叶子节点开始逆序迭代

我们找到了最后一个非叶子节点, 这里索引是 2, 对应数据 `5`, 我们先对这个节点进行堆化. 发现左子节点更大, 交换得到

```
        3
      /   \
     7     8
    / \   / \
   9   4 5   2
```

然后对上一个节点, 索引为 1, 对应数据 `7`, 进行堆化, 同理可得

```
        3
      /   \
     9     8
    / \   / \
   7   4 5   2
```

最后, 我们对索引 0 的节点进行堆化

```
        9
      /   \
     3     8
    / \   / \
   7   4 5   2
```

注意, 要点来了, 为了避免上层堆化过程影响下层堆化, 这里会 **递归的再次调用堆化函数**, 因为我们交换了 0 号节点和 1 号节点, 所以这里会再次检查之前的更大的节点, 即 `largest != root` 时, 我们交换这两个元素的位置, 并且再次检查原来 `largest` 位置, 即原来的元素 `9` 的位置, 交换过后, 现在已经是 `3`, 检查这个父节点下面的子节点是否需要堆化, 然后我们发现确实需要!

```
        9
      /   \
     7     8
    / \   / \
   3   4 5   2
```

为什么在堆化最下层子节点的时候没有提呢, 因为那时候这些最下层子节点已经没有其他子节点了

如此, 我们成功构建了最大堆. 我们不在乎子节点如何排序, 只要满足堆的结构就好, 并且知道此时最大值 9 被我们找到了且放在堆顶

然后我们开始循环提取最大元素, 即在第二个循环的位置

在循环中我们看到, 首先我们交换对顶和最后一个叶子节点

```
        2
      /   \
     7     8
    / \   /
   3   4 5   9
```

现在轮到 `end` 参数发力了, 通过它, 我们将不会在堆化的过程中考虑刚刚被我们找到的最大元素 9, 所以我去掉了那条线, 然后对这个新的堆进行堆化, 过程和上面一样, 我们发现索引 2 (8) 无需变动, 索引 1 (7) 无需变动, 索引 0 (2) 需要变动, 和右边的索引 1 (8) 交换后, 又一次影响到了索引 1 的子树, 我们对其堆化, 得到

```
        8
      /   \
     7     5
    / \   /
   3   4 2   9
```

我们又一次找到了最大的数据 8, 让它和我们保护的位置, 即上一次找到的最大元素的前一个节点交换位置, 得到

```
        2
      /   \
     7     5
    / \
   3   4 8   9
```

可见我们每一次都会将最大元素移动到正确的位置, 同时堆这个结构也能让我们以较低的时间复杂度进行操作, 因为二叉也是一种分而治之

# Timsort

Timsort 是一种混合排序, 结合了不同方案的优点, 在 Java 等语言中作为默认的排序方案, 实现起来比较复杂, 所以这里只给出大致原理和简化版实现

结合了归并排序和插入排序的优点. 它寻找已有序的 **Run**, 用插入排序扩展短 Run, 然后合并 Run

看不懂没关系, 让我尽可能通俗的解释一下

TimSort 就像整理一副扑克牌, 我们先看看能不能找到顺子(**数据中已经有序的小段**), 如果找到的顺子太短了, 就从牌堆里找一找给它补长一点(**通过插入排序补全小段**). 顺子和顺子也许可以凑成更大的顺子, 层层合并, 最终我们就得到了一副排序好的牌

接下来解释一些专有名词

- `Run`: 已经有序的小段
- `MIN_RUN`: 最小 `Run` 长度, 小于它时使用插入排序
- 分段排序: 将数组分成 `MIN_RUN` 大小的块并用插入排序
- 合并阶段：按大小倍增的方式合并相邻的已排序 `Run`
- `merge` 函数：标准的两路归并，需要克隆元素

这个算法很优秀, 在最差的情况下也能保证 O(nlogn) 的时间复杂度, 对部分有序数据接近O(n)

```rs
fn tim_sort<T: Ord + Clone>(array: &mut [T]) {
    const MIN_RUN: usize = 32;  // 最小 Run 长度

    let len = array.len();
    if len <= MIN_RUN {  // 小数组直接插入排序
        // 这个我们之前实现过
        insertion_sort(array);
        return;
    }

    // 将数组分成 MIN_RUN 大小的块并排序
    for i in (0..len).step_by(MIN_RUN) {
        let end = std::cmp::min(i + MIN_RUN, len);
        insertion_sort(&mut array[i..end]);
    }

    // 合并已排序的 Run
    // 初始块大小
    let mut size = MIN_RUN;
    // 只要块仍然比整个数组小
    while size < len {
        // 每次处理两个相邻的块
        for left in (0..len).step_by(2 * size) {
            // 计算中间点和右边界
            // 第一个块的结尾
            let mid = std::cmp::min(left + size, len);
            // 第二个块的结尾
            let right = std::cmp::min(left + 2 * size, len);
            // 如果有两个块可以合并
            if mid < right {
                // 合并它们
                merge(array, left, mid, right);
            }
        }
        // 每次合并后对小块大小的要求大小翻倍
        size *= 2;
    }
}

fn merge<T: Ord + Clone>(array: &mut [T], left: usize, mid: usize, right: usize) {
    let left_part = array[left..mid].to_vec();  // 复制左半部分
    let right_part = array[mid..right].to_vec();  // 复制右半部分

    let mut i = 0;  // 左部分索引
    let mut j = 0;  // 右部分索引
    let mut k = left;  // 合并位置索引

    // 这和我们合并链表的方式相同
    // 合并两个已排序数组
    while i < left_part.len() && j < right_part.len() {
        if left_part[i] <= right_part[j] {
            array[k] = left_part[i].clone();
            i += 1;
        } else {
            array[k] = right_part[j].clone();
            j += 1;
        }
        k += 1;
    }

    // 复制剩余元素
    while i < left_part.len() {
        array[k] = left_part[i].clone();
        i += 1;
        k += 1;
    }

    while j < right_part.len() {
        array[k] = right_part[j].clone();
        j += 1;
        k += 1;
    }
}
```
