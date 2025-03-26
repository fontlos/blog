---
feature: true
title: Rust 数据结构与算法(3) | 搜索
date: 2025-03-26 18:00:00
abstracts: 在这一节中, 我们将完成三道关于搜索的算法题, 分别是二叉搜索树, 广度优先搜索, 和深度优先搜索
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

在这一节中, 我们将完成三道关于搜索的算法题, 分别是二叉搜索树, 广度优先搜索, 和深度优先搜索

# 二叉搜索树 (Binary Search Tree, BST)

有了前面学堆排序的经验, 对二叉树的概念应该有一定了解了

二叉搜索树是一种特殊的二叉树数据结构, 其中每个节点都满足以下性质

- 左子树中所有节点的值都小于当前节点的值
- 右子树中所有节点的值都大于当前节点的值
- 左右子树也分别是二叉搜索树

提到二叉就会提到分而治之的思想, 二叉搜索树具有以下特点

- 高效查找: 平均时间复杂度为 O(log n), 比线性结构更高效
- 动态数据维护: 可以高效地插入和删除数据
- 有序数据存储: 中序遍历BST可以得到有序序列
- 作为更复杂数据结构的基础: 如 AVL 树, 红黑树等都是基于 BST 的扩展

二叉搜索树在数据库索引, 文件系统目录结构, 内存中的有序数据存储, 路由算法中的路由表都有很多应用

下面我们来从零实现一个二叉搜索树

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

# 广度优先搜索 (Breadth-First Search, BFS)

广度优先搜索是一种用于 **遍历或搜索** **树或图** 的算法。它从根节点(或任意节点)开始, 首先访问所有相邻节点, 然后再依次访问这些相邻节点的相邻节点, 以此类推, 一层一层地向外横向扩展

而提到广度优先搜索, 了解的人就会想到 **深度优先搜索(DFS)**, 接下来我们也会介绍这个, 它就是上面的描述的对立面, 路径式探索, 纵向深入

BFS 在以下领域发挥重要作用

- 最短路径查找: 在无权图中找到两点间的最短路径
- 连通性检查: 判断图中两点是否连通
- 层级遍历: 按层次处理树或图结构
- 网络爬虫: 按层级抓取网页
- 社交网络分析: 查找特定距离内的朋友关系

BFS通常使用 **队列(queue)** 来实现, 遵循以下步骤

1. 将起始节点放入队列并标记为已访问
2. 从队列中取出一个节点 (真的就取出去不要了)
3. 访问该节点的所有未访问邻接节点, 将它们放入队列并标记为已访问
4. 重复步骤 2-3 直到队列为空 (即随后访问的一个节点没有未访问的邻居了)

还是先从数据结构开始, 这里我们需要一个图结构, 先看定义

```rs
struct Graph {
    adj: Vec<Vec<usize>>,
}
```

结构非常简单, 但是又有点看不懂为什么要这样, 下面我们来解释一些

在解释它之前先解释一下上层概念, 什么是 **图(Graph)**? 简单来说, 图就类似于地图, 有地点, 地点和地点之间有路径连接

图是由 **顶点(Vertex)和边(Edge)** 组成的非线性数据结构, 数学表示为 `G = (V, E)`, 其中

- `V` 是顶点集合 (在代码中用 `usize` 索引表示)
- `E` 是连接顶点的边集合

常见的图结构包括

- 无向图: 没有方向, 比如社交关系都是相互的
- 有向图: 拥有一个递进的方向, 比如说网页内的链接, 是按顺序层层跳转的
- 加权图: 边带有权重的图, 比如说在导航软件中, 为了计算最佳路线, 我们就要给每段路经赋予不同的权重

有了这些基本认识, 现在介绍一下我们上面的数据结构, 这种结构称为 **邻接表(Adjacency List)**

- 外层 Vec 的索引代表节点 ID (如节点 0, 1, 2...)
- 每个内层 Vec 只存储实际存在的边, 适合稀疏图
- 对比邻接矩阵(你可以理解为一个二维坐标平面), 存储了所有可能的点, 节省大量内存
- 对比边列表(`Vec<(usize, usize)>`)又能提高查询效率

这里给一个简单的示例

```rs
// 对应无向图：
// 0 -- 1
// |    |
// 2    3
adj: vec![
    vec![1, 2],  // 节点0的邻居
    vec![0, 3],  // 节点1的邻居
    vec![0],     // 节点2的邻居
    vec![1]      // 节点3的邻居
]
```

同时操作这个图也很简单

- 查询节点 `v` 的邻居: `adj[v]` O(1)
- 判断 `u-v` 是否相连: `adj[u].contains(&v)` O(degree), 与邻居数量成正比
- 添加边 `(u,v)`: `adj[u].push(v)` O(1)
- 遍历所有边: 嵌套循环遍历 `adj`, O(V+E)

这样我们就能为这个结构添加两个基本方法了

```rs
fn new(n: usize) -> Self {
    Graph {
        adj: vec![vec![]; n],
    }
}

// Add an edge to the graph
fn add_edge(&mut self, src: usize, dest: usize) {
    self.adj[src].push(dest);
    // 对于无向图, 我们需要双向边
    self.adj[dest].push(src);
}
```

接下来是需要我们实现的功能, 要求我们实现广度优先搜索并返回一个访问列表, 先看代码

```rs
use std::collections::VecDeque;

fn bfs_with_return(&self, start: usize) -> Vec<usize> {
    // 初始化访问顺序记录器
    let mut visit_order = vec![];
    // 创建访问标记数组, 防止重复访问或无限循环, 初始都为 false
    // self.adj.len() 获取图中节点总数
    let mut visited = vec![false; self.adj.len()];

    // 创建双端队列作为 BFS 队列
    let mut queue = VecDeque::new();

    // 标记起始节点为已访问, 并加入队列
    visited[start] = true;
    queue.push_back(start);

    // 主循环: 当队列不为空时持续处理, 后面推入前面取出, 保证层级顺序
    while let Some(current) = queue.pop_front() {
        // 将当前节点加入访问顺序
        visit_order.push(current);
        // 遍历当前节点的所有邻居
        // &self.adj[current] 获取当前节点的邻居列表
        // 使用 &neighbor 避免所有权转移
        for &neighbor in &self.adj[current] {
            // 如果邻居未被访问过
            if !visited[neighbor] {
                // 标记为已访问
                visited[neighbor] = true;
                // 加入队列尾部 (保证层级顺序)
                queue.push_back(neighbor);
            }
        }
    }
    // 返回访问顺序
    visit_order
}
```

我们使用 `VecDeque` 作为队列, 因为它支持高效的 `头部删除(pop_front)` 和 `尾部插入(push_back)` 操作, 时间复杂度都是O(1)

- 首先初始化创建访问顺序数组, 访问标记数组和队列
- 起始处理: 将起始节点标记为已访问并加入队列
- 主循环
    - 从队列头部取出一个节点
    - 将该节点加入访问顺序
    - 遍历该节点的所有邻居, 将未访问的邻居标记为已访问并加入队列
    - 终止条件: 当队列为空时, 表示所有可达节点都已访问完毕

有的人可能好奇, 就为了一个访问顺序列表, 这东西有什么用啊? 它最基础的用处, 就像这句话说的, 没错, 就是为了获取从一个入口进去之后的访问列表, 基于这个, 我们可以发展出许多其他应用

首先就是最短路径查找, 对于无权图, BFS 天然的就能保证, 从某一入口进入后第一次访问的节点, 就是访问该节点的最短路径

然后, 还能做一些连通性判断, 原理也很简单, 只要访问列表的长度和节点数量相同, 就代表所有的节点是彼此连通的

# 深度优先搜索 (Depth-First Search, DFS)

深度优先搜索同样是一种用于遍历或搜索树或图的算法. 它沿着树的深度遍历树的节点, 尽可能深的搜索树的分支. 当节点 v 的所在边都已被探寻过, 搜索将回溯到发现节点 v 的那条边的起始节点. 这一过程一直进行到已发现从源节点可达的所有节点为止. 有点像玩游戏时收集全结局的样子

正如上面所说, DFS 的核心思想是, 尽可能深地探索图的分支，直到无法继续前进才回溯. 这种策略使得 DFS 特别适合寻找图中的路径或检测环等问题

除此之外 DFS 还在以下领域发挥作用

- 遍历图的所有节点
- 拓扑排序
- 解决迷宫问题
- 连通分量分析

通常使用递归或显式栈来实现. 这里我们使用递归实现

基本数据结构和方法与 BFS 相同, 以及这里多了个方法

```rs
// Perform a depth-first search on the graph, return the order of visited nodes
fn dfs(&self, start: usize) -> Vec<usize> {
    let mut visited = HashSet::new();
    let mut visit_order = Vec::new();
    self.dfs_util(start, &mut visited, &mut visit_order);
    visit_order
}
```

这要求我们需要使用 `HashSet<usize>` 来记录已访问的节点, 集合能保证数据的唯一性, 并且它的查找和插入操作都是平均 O(1) 时间复杂度, 然后同样用 `Vec<usize>` 来存储访问顺序

这个实现比较简单, 我们直接先给出代码

```rs
fn dfs_util(&self, v: usize, visited: &mut HashSet<usize>, visit_order: &mut Vec<usize>) {
    // 标记当前节点为已访问
    visited.insert(v);
    // 将当前节点加入访问顺序
    visit_order.push(v);
    // 遍历当前节点的所有邻接节点
    for &neighbor in &self.adj[v] {
        // 如果邻居节点没有被访问, 那么不管其他邻居节点了, 我们立刻深入这个邻居节点继续递归调用
        // 这里选择递归实现, 更直观体现DFS的自然回溯特性
        // 但是如果图很深可能会调用栈溢出
        // 这同样可以避免无限循环
        if !visited.contains(&neighbor) {
            self.dfs_util(neighbor, visited, visit_order);
        }
    }
}
```

主要内容都体现在注释里了, 读者可以自行实现一个迭代的方案
