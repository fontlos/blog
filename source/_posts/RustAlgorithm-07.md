---
feature: false
title: Rust 数据结构与算法(7) | 哈希与缓存
date: 2025-04-08 20:00:00
abstracts: 这是附加的一节, 不在计划之中. 哈希表是最重要的容器类型之一, 其核心之一是哈希函数, 在这一节中, 我们将实现一个基本的 MurmurHash3 散列算法, 并基于它和链表从零构建一个 HashMap. 然后基于哈希表, 我们实现一个简单的 LRU 缓存
tags:
    - Rust
categories:
    - Course
cover: https://fontlos.com/cover/ferris.png
---

这是附加的一节, 不在计划之中. ~~难度大, 而且我也可能讲不清 :(~~

哈希表是最重要的容器类型之一, 其核心之一是哈希函数, 在这一节中, 我们将实现一个基本的 MurmurHash3 散列算法, 并基于它和链表从零构建一个 HashMap

# 哈希函数

要实现哈希表, 首先需要一个 **哈希函数 (Hash Function)**, 哈希函数又称散列函数, 顾名思义, 将任意大小的数据散列映射到固定大小值的函数, 它的核心特点是

- 确定性: 相同的输入总是产生相同的输出
- 快速计算: 能够快速计算出哈希值
- 均匀分布: 输出值在值域范围内尽可能均匀分布
- 抗碰撞性: 难以找到两个不同的输入产生相同的输出 (哈希冲突)

哈希函数是哈希表快速查找数据结构的基础, 除此以外还用于:

- 数据校验: 如校验文件完整性 (MD5、SHA 等)
- 密码存储: 存储密码的哈希值而非明文
- 负载均衡: 一致性哈希算法
- 加密: 作为加密算法的基础组件

哈希函数的关键指标包括

- 哈希质量: 输出值的分布均匀性
- 冲突率: 不同输入产生相同输出的概率
- 计算速度: 生成哈希值的效率
- 抗碰撞性: 抵抗故意制造冲突的能力
- 雪崩效应: 输入微小变化导致输出巨大变化

## MurmurHash3

**MurmurHash3** 是我们这一节要实现的哈希函数. ~~之所以选这个是因为我最近在做短链应用正好实现了这个 :).~~ 这是由 Austin Appleby 设计的一系列 **非加密** 哈希函数, 主要特点就是速度快, 冲突率低, 实现简单. 我们这次实现的是其第三代版本的 **x64_128** 变体

我们直接随着代码来梳理算法原理, 首先来看函数签名

```rs
fn murmurhash3_x64_128(data: &[u8], seed: u64) -> u64
```

我们以字节数组形式输入数据, 需要一个初始种子, 最终输出一个 `u64` 数据类型, 这是因为我们最终合并了两个状态量, 者在多数系统上已经够用了, 然后我们来做一些基础的事

```rs
let len = data.len();
// 使用种子值初始化两个 64 位状态变量
let mut h1 = seed;
let mut h2 = seed;
```

然后我们将数据以 `16B` 为一个数据块进行处理

```rs
// 分割
let chunks = data.chunks_exact(16);
// 获取剩余不足 16B 的部分
let remainder = chunks.remainder();
```

下面需要循环处理每个数据块, 不过在此之前我们需要先给出几个常量, 它们都是精心设计过的, 例如前

```rs
// 这两个 **魔数** 经过精心选择, 具有良好的混合特性, 这里很难解释为什么要选这些数字了
const C1: u64 = 0x87c37b91114253d5;
const C2: u64 = 0x4cf5ad432745937f;
// 用于环绕左移位数控制
// 梅森素数, 2^6 -1 对 CPU 架构友好
const R1: u32 = 31;
// 避免与上面的相同对称, 避免对其 32 位边界产生规律性, 经过验证的雪崩效应
const R2: u32 = 27;
// 乘法常数, 与后面加法常数配合产生非线性, 中等大小的奇数, 非 2 的幂附近的值, 避免简单位模式
const M: u64 = 0x52dce729;
```

暂时可以先不去管它们的作用, 我们慢慢看, 先看循环的部分

```rs
for chunk in chunks {
    // 将每 16 字节分为两个 64 位整数 (小端序)
    let mut k1 = u64::from_le_bytes(chunk[0..8].try_into().unwrap());
    let mut k2 = u64::from_le_bytes(chunk[8..16].try_into().unwrap());

    // 这里是核心部分
    // 就像所有哈希算法那样, 我们就是在用常量更新数据, 然后用更新的数据来更新状态量, 如此反复

    // 首先是环绕乘法, 它不会溢出, 而是执行环绕操作
    k1 = k1.wrapping_mul(C1);
    // 然后左移 31 位, 这与左移符号不通, 它是循环左移, 左侧移出的位会从右侧重新填入
    // 不丢失数据, 只改变位置. 当 n 过大时会自动取模
    k1 = k1.rotate_left(R1);
    k1 = k1.wrapping_mul(C2);
    // 异或混合
    h1 ^= k1;

    // 然后改变状态量, 操作类似
    h1 = h1.rotate_left(R2);
    h1 = h1.wrapping_add(h2);
    h1 = h1.wrapping_mul(M).wrapping_add(0x52dce729);

    k2 = k2.wrapping_mul(C2);
    k2 = k2.rotate_left(R2);
    k2 = k2.wrapping_mul(C1);
    h2 ^= k2;

    h2 = h2.rotate_left(R1);
    h2 = h2.wrapping_add(h1);
    h2 = h2.wrapping_mul(M).wrapping_add(0x38495ab5);
}
```

然后是处理尾部剩余数据, 这可能有一点复杂, 因为我们要根据不同长度做不同的处理

```rs
if !remainder.is_empty() {
    let mut k1 = 0;
    let mut k2 = 0;

    match remainder.len() {
        15 => k2 ^= (remainder[14] as u64) << 48,
        14 => k2 ^= (remainder[13] as u64) << 40,
        13 => k2 ^= (remainder[12] as u64) << 32,
        12 => k2 ^= (remainder[11] as u64) << 24,
        11 => k2 ^= (remainder[10] as u64) << 16,
        10 => k2 ^= (remainder[9] as u64) << 8,
        9 => k2 ^= remainder[8] as u64,
        _ => (),
    }

    if !remainder.is_empty() {
        k2 = k2.wrapping_mul(C2);
        k2 = k2.rotate_left(R2);
        k2 = k2.wrapping_mul(C1);
        h2 ^= k2;
    }

    match remainder.len() {
        8 => k1 ^= (remainder[7] as u64) << 56,
        7 => k1 ^= (remainder[6] as u64) << 48,
        6 => k1 ^= (remainder[5] as u64) << 40,
        5 => k1 ^= (remainder[4] as u64) << 32,
        4 => k1 ^= (remainder[3] as u64) << 24,
        3 => k1 ^= (remainder[2] as u64) << 16,
        2 => k1 ^= (remainder[1] as u64) << 8,
        1 => k1 ^= remainder[0] as u64,
        _ => (),
    }

    if !remainder.is_empty() {
        k1 = k1.wrapping_mul(C1);
        k1 = k1.rotate_left(R1);
        k1 = k1.wrapping_mul(C2);
        h1 ^= k1;
    }
}
```

然后最终混合两个状态量作为我们的哈希值

```rs
h1 ^= len as u64;
h2 ^= len as u64;

h1 = h1.wrapping_add(h2);
h2 = h2.wrapping_add(h1);

h1 = fmix64(h1);
h2 = fmix64(h2);

// 最终结果
// 经验证比直接相加有更好的分布效果
h1.wrapping_add(h2)
```

这用到了一个辅助函数

```rs
fn fmix64(mut k: u64) -> u64 {
    // 第一个常数侧重影响高位
    // 第二个常数侧重影响低位

    k ^= k >> 33;
    k = k.wrapping_mul(0xff51afd7ed558ccd);
    k ^= k >> 33;
    k = k.wrapping_mul(0xc4ceb9fe1a85ec53);
    k ^= k >> 33;
    k
}
```

上面看不懂没关系, ~~哈希算法本来就是让你看不懂结果的过程,~~ 现在我们的关键内容就是基于它, 设计我们的哈希表了, 不过在此之前要对这个散列函数进行简单包装, 并实现 `Hasher` Trait

```rs
struct Murmur3Hasher {
    seed: u64,
    buffer: Vec<u8>,
}

impl Murmur3Hasher {
    fn new(seed: u64) -> Self {
        Murmur3Hasher {
            seed,
            buffer: Vec::new(),
        }
    }
}

impl std::hash::Hasher for Murmur3Hasher {
    fn finish(&self) -> u64 {
        murmurhash3_x64_128(&self.buffer, self.seed)
    }

    fn write(&mut self, bytes: &[u8]) {
        self.buffer.extend_from_slice(bytes);
    }
}
```

# HashMap

## 基本数据结构

我们的哈希表底层基于链表, 这是为了解决哈希冲突

- 每个数据桶位置维护一个链表
- 冲突的键值对追加到链表尾部
- 实现简单，稳定可靠

这里我们使用标准库提供的

```rs
use std::collections::LinkedList;

pub struct HashMap<K, V> {
    buckets: Vec<LinkedList<(K, V)>>,
    size: usize,
    capacity: usize,
    // 触发扩容的负载因子
    load_factor: f32,
}
```

更好的方案是开放寻址法, 发生冲突时寻找下一个空闲位置, 缓存更友好但实现复杂, 不适合教学演示的目的. 此外还有一些小链表转数组, 极端链表转平衡树的优化操作, 为了简便也不实现了

然后就是为它实现一些基础功能了, 都比较容易理解

```rs
impl<K, V> HashMap<K, V>
where
    K: Eq + std::hash::Hash + Clone,
    V: Clone
{
    pub fn new() -> Self {
        // 初始容量选择 2 的幂次方, 比较方便
        let initial_capacity = 16;
        HashMap {
            // 初始化所有桶为空链表
            buckets: vec![LinkedList::new(); initial_capacity],
            size: 0,
            capacity: initial_capacity,
            // 基于泊松分布和工程经验
            load_factor: 0.75,
        }
    }

    pub fn with_capacity(capacity: usize) -> Self {
        HashMap {
            buckets: vec![LinkedList::new(); capacity],
            size: 0,
            capacity,
            load_factor: 0.75,
        }
    }

    pub fn contains_key(&self, key: &K) -> bool {
        self.get(key).is_some()
    }

    pub fn len(&self) -> usize {
        self.size
    }

    pub fn is_empty(&self) -> bool {
        self.size == 0
    }

    // 作为关联函数方便使用
    fn hash(key: &K) -> u64 {
        use std::hash::Hasher;
        let mut hasher = Murmur3Hasher::new(0);
        key.hash(&mut hasher);
        hasher.finish()
    }
}
```

## 实现关键功能

首先是一个扩容方法, 采用以下策略:

- 容量加倍 (保持 2 的幂次方)
- 创建新桶数组
- 重新哈希所有现有元素到新位置
- 替换旧桶数组
- 扩容是 `O(n)` 操作，但分摊到多次插入仍是 `O(1)`
- 加倍策略减少频繁扩容
- 一次性重新哈希避免渐进式 rehash 的复杂性

```rs
fn resize(&mut self) {
    let new_capacity = self.capacity * 2;
    let mut new_buckets = vec![LinkedList::new(); new_capacity];

    for bucket in self.buckets.drain(..) {
        for (key, value) in bucket {
            let hash = Self::hash(&key);
            let index = (hash as usize) % new_capacity;
            new_buckets[index].push_back((key, value));
        }
    }

    self.buckets = new_buckets;
    self.capacity = new_capacity;
}
```

然后是较为简单的获取, 没太多需要解释的地方, 分桶查找

```rs
pub fn get(&self, key: &K) -> Option<&V> {
    let hash = Self::hash(key);
    let index = (hash as usize) % self.capacity;

    for entry in self.buckets[index].iter() {
        if &entry.0 == key {
            return Some(&entry.1);
        }
    }
    None
}

pub fn get_mut(&mut self, key: &K) -> Option<&mut V> {
    let hash = Self::hash(key);
    let index = (hash as usize) % self.capacity;

    for entry in self.buckets[index].iter_mut() {
        if &entry.0 == key {
            return Some(&mut entry.1);
        }
    }
    None
}
```

插入时我们先检查是否需要扩容, 然后按照标准库的风格, 如果键不存在就插入, 如果存在就替换旧值并返回

```rs
pub fn insert(&mut self, key: K, value: V) -> Option<V> {
    // 检查是否需要扩容
    if (self.size as f32) >= (self.capacity as f32 * self.load_factor) {
        self.resize();
    }

    let hash = Self::hash(&key);
    let index = (hash as usize) % self.capacity;

    // 检查是否已存在相同的 key
    for entry in self.buckets[index].iter_mut() {
        if entry.0 == key {
            let old_value = std::mem::replace(&mut entry.1, value);
            return Some(old_value);
        }
    }

    // 插入新键值对
    self.buckets[index].push_back((key, value));
    self.size += 1;
    None
}
```

```rs
pub fn remove(&mut self, key: &K) -> Option<V> {
    let hash = Self::hash(key);
    let index = (hash as usize) % self.capacity;

    // 先找到目标链表
    let prev_link = &mut self.buckets[index];
    while !prev_link.is_empty() {
        // 最好情况, 第一个节点就是我们要的, 直接弹出即可
        if prev_link.front().unwrap().0 == *key {
            // 找到目标节点, 直接移除
            self.size -= 1;
            return Some(prev_link.pop_front().unwrap().1);
        }
        // 移动到下一个节点, 我们直接切掉零号节点
        let mut split_list = prev_link.split_off(1);
        // 然后交换变量, 这是为了绕过同一作用域内只能由一个可变借用的规则
        std::mem::swap(prev_link, &mut split_list);
    }
    None
}
```

更高效的方式是使用 **链表游标**, 不过截止到现在, 这需要启用一个不稳定特性 `#![feature(linked_list_cursors)]`, 因此只给出实现, 不具体展开了, 而且逻辑已经很明确了

```rs
pub fn remove(&mut self, key: &K) -> Option<V> {
    let hash = Self::hash(key);
    let index = (hash as usize) % self.capacity;

    let mut cursor = self.buckets[index].cursor_front_mut();
    while let Some(entry) = cursor.current() {
        if entry.0 == *key {
            self.size -= 1;
            return Some(cursor.remove_current().unwrap().1);
        }
        cursor.move_next();
    }

    None
}
```

最后给出一个简单的测试

```rs
#[test]
fn test_hash_map() {
    let mut map = HashMap::new();
    assert_eq!(map.len(), 0);
    assert!(map.is_empty());

    map.insert("key1", "value1");
    assert_eq!(map.len(), 1);
    assert!(!map.is_empty());
    assert_eq!(map.get(&"key1"), Some(&"value1"));
    assert!(map.contains_key(&"key1"));

    map.insert("key2", "value2");
    assert_eq!(map.len(), 2);
    assert_eq!(map.get(&"key2"), Some(&"value2"));

    assert_eq!(map.remove(&"key1"), Some("value1"));
    assert_eq!(map.len(), 1);
    assert_eq!(map.get(&"key1"), None);

    *map.get_mut(&"key2").unwrap() = "new_value";
    assert_eq!(map.get(&"key2"), Some(&"new_value"));
}
```

# LRU

**LRU (Least Recently Used, 最近最少使用)** 是一种经典的缓存淘汰策略. 核心思想就在字面上: 当缓存空间不足时, 优先淘汰那些最久未被访问的数据

通常有指定的缓存容量限制, 为了快速查找缓存中的键值对, HashMap 是一个不错的选择, 同时为了维护键的访问顺序, 最近访问的放在前面, 久未访问的放在后面, 我们可以使用链表完成

## 基本数据结构

这里我们同样使用标准库提供的版本, 当然用上面的哈希表实现的也可以

```rs
use std::collections::{HashMap, LinkedList};
use std::hash::Hash;

pub struct LRUCache<K, V> {
    capacity: usize,
    map: HashMap<K, V>,
    order: LinkedList<K>,
}

impl<K, V> LRUCache<K, V>
where
    K: Eq + Hash + Clone,
{
    pub fn new(capacity: usize) -> Self {
        assert!(capacity > 0, "Capacity must be greater than 0");
        LRUCache {
            capacity,
            map: HashMap::with_capacity(capacity),
            order: LinkedList::new(),
        }
    }

    pub fn len(&self) -> usize {
        self.map.len()
    }

    pub fn is_empty(&self) -> bool {
        self.map.is_empty()
    }
}
```

## 实现关键功能

为了维护访问顺序, 我们先实现一个简单的辅助函数

```rs
// 简单的实现: 先移除再添加到尾部, 缺点是每次都需要重建链表
// 注意: 对于大型链表这显然不是最高效的方式, 但对于演示目的足够
fn move_to_back(&mut self, key: &K) {
    // 创建新链表, 将目标 key 放到最后
    let mut new_order = LinkedList::new();
    let mut found = false;
    // 将旧链表中的元素转移到新链表
    while let Some(k) = self.order.pop_front() {
        if &k == key {
            found = true;
        } else {
            new_order.push_back(k);
        }
    }
    // 如果找到 key, 将其放到链表尾部
    if found {
        new_order.push_back(key.clone());
    }
    // 将其他元素放回原链表
    while let Some(k) = new_order.pop_front() {
        self.order.push_back(k);
    }
}
```

或者我们同样可以像实现哈希表那样, 使用链表游标, 但需要开启不稳定特性

```rs
fn move_to_back(&mut self, key: &K) {
    let mut cursor = self.order.cursor_front_mut();
    while let Some(k) = cursor.current() {
        if k == key {
            cursor.remove_current();
            self.order.push_back(key.clone());
            break;
        }
        cursor.move_next();
    }
}
```

在生产中还是建议使用 `linked-hash-map` 这些经过考验的 Crate

然后我们就可以实现缓存的读取和写入了

```rs
pub fn get(&mut self, key: &K) -> Option<&V> {
    if self.map.contains_key(key) {
        // 将访问的 key 移动到链表尾部 (表示最近使用)
        self.move_to_back(key);
        self.map.get(key)
    } else {
        None
    }
}

pub fn put(&mut self, key: K, value: V) {
    // 如果已存在就更新现有值, 否则就插入新值, 同时考虑是否要销毁旧值
    if self.map.contains_key(&key) {
        // 更新现有值
        self.map.insert(key.clone(), value);
        self.move_to_back(&key);
    } else {
        if self.map.len() >= self.capacity {
            // 移除最久未使用的元素（链表头部）
            if let Some(oldest_key) = self.order.pop_front() {
                self.map.remove(&oldest_key);
            }
        }
        // 插入新值
        self.map.insert(key.clone(), value);
        self.order.push_back(key);
    }
}
```

最后给出一些测试

```rs
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_lru_cache_basic() {
        let mut cache = LRUCache::new(2);

        assert_eq!(cache.get(&"a"), None);

        cache.put("a", 1);
        assert_eq!(cache.len(), 1);
        assert_eq!(cache.get(&"a"), Some(&1));

        cache.put("b", 2);
        assert_eq!(cache.get(&"a"), Some(&1));
        assert_eq!(cache.get(&"b"), Some(&2));
    }

    #[test]
    fn test_lru_cache_eviction() {
        let mut cache = LRUCache::new(2);

        cache.put("a", 1);
        cache.put("b", 2);
        cache.put("c", 3); // 这会使得键 "a" 被移除

        assert_eq!(cache.get(&"a"), None);
        assert_eq!(cache.get(&"b"), Some(&2));
        assert_eq!(cache.get(&"c"), Some(&3));
    }

    #[test]
    fn test_lru_cache_update() {
        let mut cache = LRUCache::new(2);

        cache.put("a", 1);
        cache.put("b", 2);
        assert_eq!(cache.get(&"a"), Some(&1)); // 访问 "a" 使其成为最近使用的

        cache.put("c", 3); // 这会使得键 "b" 被移除，而不是 "a"
        assert_eq!(cache.get(&"b"), None);
        assert_eq!(cache.get(&"a"), Some(&1));
        assert_eq!(cache.get(&"c"), Some(&3));
    }

    #[test]
    fn test_lru_cache_complex() {
        let mut cache = LRUCache::new(3);

        cache.put("a", 1);
        cache.put("b", 2);
        cache.put("c", 3);
        assert_eq!(cache.get(&"a"), Some(&1)); // a -> c, b, a

        cache.put("d", 4); // 移除 b (最久未使用)
        assert_eq!(cache.get(&"b"), None);
        assert_eq!(cache.get(&"a"), Some(&1));
        assert_eq!(cache.get(&"c"), Some(&3));
        assert_eq!(cache.get(&"d"), Some(&4));

        cache.put("e", 5); // 移除 a (虽然a被访问过, 但之后 c 和 d 也被访问过)
        assert_eq!(cache.get(&"a"), None);
    }
}
```