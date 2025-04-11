---
feature: false
title: Rust 数据结构与算法(6) | 图与搜索
date: 2025-03-27 18:00:00
abstracts: 图与搜索算法密不可分, 例如广度优先搜索(BFS)和深度优先搜索(DFS)是两种最基本的图遍历算法, 它们构成了许多高级图算法的基础. 所以在这一节中, 我们将会先讲解图这个概念, 然后实现这两种搜索算法
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

# 图(Graph)

**图(Graph)** 与搜索算法密不可分, 例如 **广度优先搜索(BFS)** 和 **深度优先搜索(DFS)** 是两种最基本的图遍历算法, 它们构成了许多高级图算法的基础. 首先我们来认识一下图. 简单来说, 图就类似于地图, 有地点, 地点和地点之间有路径连接

图是由 **顶点(Vertex)和边(Edge)** 组成的非线性数据结构, 数学表示为 `G = (V, E)`, 其中

- `V` 是顶点集合 (在代码中用 `usize` 索引表示)
- `E` 是连接顶点的边集合

常见的图结构包括

- 无向图: 没有方向, 比如社交关系都是相互的
- 有向图: 拥有一个递进的方向, 比如说网页内的链接, 是按顺序层层跳转的
- 加权图: 边带有权重的图, 比如说在导航软件中, 为了计算最佳路线, 我们就要给每段路经赋予不同的权重

常用的用于表示图的结构包括

- **邻接表 (Adjacency List)**: 为每个顶点存储一个列表, 记录与之相连的顶点
- **邻接矩阵 (Adjacency Matrix)**: 使用二维数组表示顶点之间的连接关系
- **边列表 (Edge List)**: 简单地存储所有边的列表

我们说图与搜索联系紧密, 为什么图需要搜索? 图的搜索算法用于系统地探索图中的节点和边, 主要解决以下问题

- 检查图中是否存在特定节点
- 查找从一个节点到另一个节点的路径
- 检查图的连通性
- 发现图的结构特性

所有图搜索算法都遵循一个通用模式

- 从起始节点开始
- 逐步 "发现" 相邻节点
- 记录已访问节点避免重复
- 按照特定策略选择下一个要访问的节点, 例如下面我们会讲的广度优先搜索和深度优先搜索

那么下面让我们简单实现一个图, 以及图的相关函数

## 基本数据结构

```rs
pub struct UndirectedGraph {
    adjacency_table: HashMap<String, Vec<(String, i32)>>,
}
```

这是一种加权无向图, `HashMap` 的键是节点名称 (`String`), 值是该节点的邻接表, 存储了相邻节点和边的权重 (`Vec<(String, i32)>`)

## 实现基本功能

我们的题目定义了一个图的通用 Trait

```rs
pub trait Graph {
    // 基本功能, 新建, 获得可变邻接表或不可变邻接表
    fn new() -> Self;
    fn adjacency_table_mutable(&mut self) -> &mut HashMap<String, Vec<(String, i32)>>;
    fn adjacency_table(&self) -> &HashMap<String, Vec<(String, i32)>>;
    // 添加节点和边, 需要我们实现
    fn add_node(&mut self, node: &str) -> bool {
        //TODO
    }
    fn add_edge(&mut self, edge: (&str, &str, i32)) {
        //TODO
    }
    // 图是否包含节点
    fn contains(&self, node: &str) -> bool {
        self.adjacency_table().get(node).is_some()
    }
    // 获得所有节点
    fn nodes(&self) -> HashSet<&String> {
        self.adjacency_table().keys().collect()
    }
    // 获得所有的边
    fn edges(&self) -> Vec<(&String, &String, i32)> {
        let mut edges = Vec::new();
        for (from_node, from_node_neighbours) in self.adjacency_table() {
            for (to_node, weight) in from_node_neighbours {
                edges.push((from_node, to_node, *weight));
            }
        }
        edges
    }
}
```

然后我们的题目已经为无向图实现了这个 Trait

```rs
impl Graph for UndirectedGraph {
    fn new() -> UndirectedGraph {
        UndirectedGraph {
            adjacency_table: HashMap::new(),
        }
    }
    fn adjacency_table_mutable(&mut self) -> &mut HashMap<String, Vec<(String, i32)>> {
        &mut self.adjacency_table
    }
    fn adjacency_table(&self) -> &HashMap<String, Vec<(String, i32)>> {
        &self.adjacency_table
    }
}
```

此外, 题目中还给了一个用于错误处理的结构体

```rs
use std::fmt;
#[derive(Debug, Clone)]
pub struct NodeNotInGraph;
impl fmt::Display for NodeNotInGraph {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "accessing a node that is not in the graph")
    }
}
```

我们现在的策略是节点不存在时自动创建节点, 至于错误处理的版本读者可以自行实现

## 实现具体功能

既然题目在 Trait 中留的不是函数签名, 而是带有花括号的, 那我这里就直接在 Trait 中实现这个默认方法, 实际代码中不建议这样操作

```rs
fn add_node(&mut self, node: &str) -> bool {
    // 检查是否存在节点, 不存在就自动创建
    if self.contains(node) {
        false
    } else {
        self.adjacency_table_mutable()
            .insert(node.to_string(), Vec::new());
        true
    }
}
fn add_edge(&mut self, edge: (&str, &str, i32)) {
    // 边的信息, 两个节点, 一个权重
    let (node1, node2, weight) = edge;

    // 确保两个节点都存在
    if !self.contains(node1) {
        self.add_node(node1);
    }
    if !self.contains(node2) {
        self.add_node(node2);
    }

    // 获取邻接表可变引用
    let adj_table = self.adjacency_table_mutable();

    // 添加 node1 -> node2 的边
    adj_table
        .get_mut(&node1.to_string())
        .unwrap()
        .push((node2.to_string(), weight));

    // 添加 node2 -> node1 的边 (因为我们是无向图, 两边不能感知到对方是否连接, 所以需要互相连线)
    adj_table
        .get_mut(&node2.to_string())
        .unwrap()
        .push((node1.to_string(), weight));
}
```

只要理解了图的结构, 这并不难, 我们已经实现了一个基本的图, 不过它也只是个图, 还没有任何功能. 下面让我们学习一些建立在图结构上的搜索算法

# 广度优先搜索 (Breadth-First Search, BFS)

广度优先搜索是一种用于 **遍历或搜索** **树或图** 的算法。它从根节点(或任意节点)开始, 首先访问所有相邻节点, 然后再依次访问这些相邻节点的相邻节点, 以此类推, 一层一层地向外横向扩展

而提到广度优先搜索, 了解的人就会想到 **深度优先搜索(DFS)**, 接下来我们也会介绍这个, 它就是上面的描述的对立面, 路径式探索, 纵向深入

BFS 在以下领域发挥重要作用

- 最短路径查找: 找到两点间的最短路径, 无权或有权
- 连通性检查: 判断图中两点是否连通
- 层级遍历: 按层次处理树或图结构
- 网络爬虫: 按层级抓取网页
- 社交网络分析: 查找特定距离内的朋友关系

BFS通常使用 **队列(queue)** 来实现, 遵循以下步骤

1. 将起始节点放入队列并标记为已访问
2. 从队列中取出一个节点 (真的就取出去不要了)
3. 访问该节点的所有未访问邻接节点, 将它们放入队列并标记为已访问
4. 重复步骤 2-3 直到队列为空 (即随后访问的一个节点没有未访问的邻居了)

## 基本数据结构

还是先从数据结构开始, 这里我们需要一个图结构, 先看定义

```rs
struct Graph {
    adj: Vec<Vec<usize>>,
}
```

还记得我们之前讲过的图吗, 这其实就是 **邻接表(Adjacency List)**

- 外层 Vec 的索引代表节点 ID (如节点 0, 1, 2...)
- 每个内层 Vec 只存储实际存在的边, 适合稀疏图
- 对比邻接矩阵, 存储了所有可能的点, 节省大量内存
- 对比边列表(`Vec<(usize, usize)>`)又能提高查询效率

这里给一个简单的示例

```rs
// 对应无向图：
// 0 -- 1
// |    |
// 2    3
adj: vec![
    vec![1, 2],  // 节点 0 的邻居
    vec![0, 3],  // 节点 1 的邻居
    vec![0],     // 节点 2 的邻居
    vec![1]      // 节点 3 的邻居
]
```

同时操作这个图也很简单

- 查询节点 `v` 的邻居: `adj[v]` `O(1)`
- 判断 `u-v` 是否相连: `adj[u].contains(&v)` `O(degree)`, 与邻居数量成正比
- 添加边 `(u,v)`: `adj[u].push(v)` `O(1)`
- 遍历所有边: 嵌套循环遍历 `adj`, `O(V+E)`

## 基本方法

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

## 实现具体功能

题目要求我们实现广度优先搜索并返回一个访问列表

就像我们前面说的, 首先我们需要一个 `Vec` 来跟踪记录访问列表, 然后用另一个 `Vec` 记录这些节点的访问情况, 然后用一个队列来跟踪需要访问的节点, 将首个节点压入, 然后开始不断地弹出队列中的节点, 每次循环中都会把该节点未访问的节点压入队列, 同时把访问过的节点标记为已访问

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

我们使用 `VecDeque` 作为队列, 因为它支持高效的 `头部删除(pop_front)` 和 `尾部插入(push_back)` 操作, 时间复杂度都是 `O(1)`

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

基本数据结构和方法与 BFS 相同, 以及这里多了个方法, 实际就是对我们需要实现的方法的上层包装

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

# Dijkstra (迪科斯彻) 最短路径算法

我们已经知道 BFS 可以自动的解决无权图的最短路径查找, 所以这里我们来实现一个用于有权图最短路径查找的经典算法 -- **Dijkstra 最短路径算法**

它适用于边权值为非负的有向或无向图, 算法的主要思想是:

- 维护一个到各顶点的当前最短距离集合, 初始时源点距离为0, 其他点距离为无穷大
- 每次从未处理的顶点中选择距离最小的顶点
- 对该顶点的所有邻居进行松弛操作 (即检查是否可以通过该顶点获得更短的路径)
- 重复上述过程直到所有顶点都被处理

总之我们会探索所有节点, 记录所有的最短路径

实现时注意要点如下:

- 使用优先队列 (最小堆) 代替普通队列, 以便每次取出距离最小的节点
- 需要处理边的权重
- 需要维护到每个节点的当前最短距离
- 记录路径信息 (可选)

## 基本数据结构

因为我们需要处理权重, 所以节点不能是单纯的 `usize` 了

```rs
// 定义一个结构体来表示图中的边
#[derive(Debug)]
struct Edge {
    node: usize,
    weight: usize,
}

// 定义图结构，现在使用带权重的边
struct WeightedGraph {
    adj: Vec<Vec<Edge>>,
}

// 用于优先队列的比较结构体
#[derive(Eq, PartialEq)]
struct State {
    distance: usize,
    position: usize,
}

// 让State可以被比较，实现最小堆
impl Ord for State {
    fn cmp(&self, other: &Self) -> Ordering {
        other.distance.cmp(&self.distance)
    }
}

impl PartialOrd for State {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}
```

然后为权重图实现基本方法

```rs
impl WeightedGraph {
    // 创建一个新的带权图
    fn new(n: usize) -> Self {
        WeightedGraph {
            adj: vec![vec![]; n],
        }
    }

    // 添加带权边
    fn add_edge(&mut self, src: usize, dest: usize, weight: usize) {
        self.adj[src].push(Edge { node: dest, weight });
        self.adj[dest].push(Edge { node: src, weight }); // 无向图需要双向添加
    }
}
```

最后实现两个关键方法

```rs
impl WeightedGraph {
    // Dijkstra算法实现
    fn dijkstra(&self, start: usize) -> (Vec<usize>, Vec<Option<usize>>) {
        // 初始时所有节点对入口的距离都是无穷大
        let mut distances = vec![MAX; self.adj.len()];
        // 用于记录前驱节点, 重建路径
        let mut previous = vec![None; self.adj.len()];

        // 使用 BinaryHeap 实现最小堆, 每次取出距离最小的节点
        let mut heap = BinaryHeap::new();

        // 起始节点距离为 0, 直接推入
        distances[start] = 0;
        heap.push(State {
            distance: 0,
            position: start,
        });

        // 最小堆每一次 pop 都是当前距离最小的节点
        while let Some(State { distance, position }) = heap.pop() {
            // 检查当前位置的节点的距离是否已经比记录的距离大
            // 如果是, 说明已经有更优路径, 跳过处理
            if distance > distances[position] {
                continue;
            }

            // 遍历当前节点的所有邻接边
            for edge in &self.adj[position] {
                // 计算通过当前节点到达邻居节点的距离
                let next_distance = distance + edge.weight;

                // 检查新计算的距离是否比之前记录的距离更小
                if next_distance < distances[edge.node] {
                    // 更新邻居节点的最短距离
                    // 记录邻居节点的前驱节点为当前节点, 用当前节点作为下标输入就能得到最短路径上上一个节点的下标
                    distances[edge.node] = next_distance;
                    previous[edge.node] = Some(position);
                    // 将邻居节点及其新距离加入优先队列
                    heap.push(State {
                        distance: next_distance,
                        position: edge.node,
                    });
                }
            }
        }

        (distances, previous)
    }

    // 辅助函数: 根据 previous 数组构建路径
    fn build_path(&self, previous: &[Option<usize>], target: usize) -> Vec<usize> {
        // 初始化路径向量
        // 设置当前节点为目标节点
        let mut path = Vec::new();
        let mut current = target;

        // 当前节点有前驱节点时继续
        while let Some(prev) = previous[current] {
            // 将当前节点加入路径
            path.push(current);
            // 将当前节点设置为它的前驱节点
            current = prev;
        }
        path.push(current); // 添加起点
        path.reverse();
        path
    }
}
```

最后可以简单的测试一下

```rs
#[test]
fn test_dijkstra() {
    let mut graph = WeightedGraph::new(5);
    graph.add_edge(0, 1, 4);
    graph.add_edge(0, 2, 1);
    graph.add_edge(1, 2, 2);
    graph.add_edge(1, 3, 5);
    graph.add_edge(2, 3, 1);
    graph.add_edge(2, 4, 3);
    graph.add_edge(3, 4, 1);

    let (distances, previous) = graph.dijkstra(0);

    // 检查距离
    assert_eq!(distances[0], 0);
    assert_eq!(distances[1], 3);
    assert_eq!(distances[2], 1);
    assert_eq!(distances[3], 2);
    assert_eq!(distances[4], 3);

    // 检查路径
    assert_eq!(graph.build_path(&previous, 4), vec![0, 2, 3, 4]);
    assert_eq!(graph.build_path(&previous, 1), vec![0, 2, 1]);
}

#[test]
fn test_disconnected_graph() {
    let mut graph = WeightedGraph::new(4);
    graph.add_edge(0, 1, 1);
    graph.add_edge(2, 3, 1);

    let (distances, _) = graph.dijkstra(0);

    assert_eq!(distances[0], 0);
    assert_eq!(distances[1], 1);
    assert_eq!(distances[2], MAX); // 不可达
    assert_eq!(distances[3], MAX); // 不可达
}
```

# DFS 寻路算法

Dijkstra 同样可以用于寻路, 而且只要有就一定能找到最短路径, 但是需要全图搜索比较慢, 接下来我们再使用 DFS 实现一个迷宫寻路算法, 不保证最短, 但能更快的找到通路

## 基本数据结构

```rs
struct Maze {
    grid: Vec<Vec<char>>,
    rows: usize,
    cols: usize,
}
```

## 实现基本方法

要想构建一个迷宫, 直接使用邻接矩阵是最方便的

```rs
impl Maze {
    fn new(grid: Vec<Vec<char>>) -> Self {
        let rows = grid.len();
        let cols = if rows > 0 { grid[0].len() } else { 0 };
        Maze { grid, rows, cols }
    }

    // 和之前一样, 这只是对 DFS 的上层包装
    fn solve(&self, start: (usize, usize), end: (usize, usize)) -> Option<Vec<(usize, usize)>> {
        let mut visited = HashSet::new();
        let mut path = Vec::new();

        if self.dfs(start, end, &mut visited, &mut path) {
            Some(path)
        } else {
            None
        }
    }
}
```

## 实现关键方法

```rs
impl Maze {
    // 这里我们同样递归的调用
    fn dfs(
        &self,
        current: (usize, usize),
        end: (usize, usize),
        visited: &mut HashSet<(usize, usize)>,
        path: &mut Vec<(usize, usize)>,
    ) -> bool {
        // 基本情况, 到达终点了
        if current == end {
            path.push(current);
            return true;
        }

        // 这是针对我们下面的测试给出的判定条件, 用 # 代表墙壁, 如果访问过或者遇到墙壁, 这就不是通路
        if visited.contains(&current) || self.grid[current.0][current.1] == '#' {
            return false;
        }

        visited.insert(current);
        path.push(current);

        // 定义一些方向
        let directions = [(-1, 0), (1, 0), (0, -1), (0, 1)];
        for (dr, dc) in directions.iter() {
            let r = current.0 as i32 + dr;
            let c = current.1 as i32 + dc;

            // 保证坐标仍然在迷宫内
            if r >= 0 && r < self.rows as i32 && c >= 0 && c < self.cols as i32 {
                let next = (r as usize, c as usize);
                // 优先搜索邻居
                if self.dfs(next, end, visited, path) {
                    return true;
                }
            }
        }

        // 如果没有通路就返回上一格
        path.pop();
        false
    }
}
```

最后给出一个简单的测试

```rs
#[test]
fn test_dfs_maze() {
    let grid = vec![
        vec!['S', '.', '.', '#', '.'],
        vec!['#', '#', '.', '#', '.'],
        vec!['.', '.', '.', '.', '.'],
        vec!['.', '#', '#', '#', '.'],
        vec!['.', '.', '.', '.', 'E'],
    ];

    let maze = Maze::new(grid);
    if let Some(path) = maze.solve((0, 0), (4, 4)) {
        println!("Find:");
        for (i, (r, c)) in path.iter().enumerate() {
            println!("Step {}: ({}, {})", i, r, c);
        }
    } else {
        println!("Failed to find a path.");
    }
}
```

来看运行结果

```
Find:
Step 0: (0, 0)
Step 1: (0, 1)
Step 2: (0, 2)
Step 3: (1, 2)
Step 4: (2, 2)
Step 5: (2, 1)
Step 6: (2, 0)
Step 7: (3, 0)
Step 8: (4, 0)
Step 9: (4, 1)
Step 10: (4, 2)
Step 11: (4, 3)
Step 12: (4, 4)
```

可以看到这并不是最短路径
