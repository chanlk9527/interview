# 数组

数组相关算法题整理。

---

## 一、什么是数组

数组（Array）是一种**线性数据结构**，它用一组**连续的内存空间**来存储一组**相同类型**的数据。

核心特征：

- **连续内存**：元素在内存中紧挨着存放，没有额外的指针开销
- **类型统一**：同一个数组中所有元素的数据类型相同，每个元素占用的字节数一致
- **随机访问**：通过下标可以在 O(1) 时间内直接定位到任意元素

### 寻址公式

```
address(arr[i]) = baseAddress + i * elementSize
```

正是因为这个公式，数组才能做到 O(1) 随机访问——给定下标 `i`，一次乘法加一次加法就能算出目标地址。

### 数组 vs 链表（直觉对比）

| 操作 | 数组 | 链表 |
|------|------|------|
| 随机访问（按下标） | O(1) ✅ | O(n) |
| 头部插入/删除 | O(n) | O(1) ✅ |
| 尾部插入（不扩容） | O(1) ✅ | O(1) ✅ |
| 中间插入/删除 | O(n) | O(1)*（已知节点） |
| 内存布局 | 连续，缓存友好 | 分散，缓存不友好 |
| 额外空间 | 无 | 每个节点存指针 |

> 选数组还是链表，核心看**访问模式**：频繁随机读 → 数组；频繁插入删除 → 链表。

---

## 二、静态数组

静态数组是指**长度在创建时就确定，之后不可改变**的数组。

### Java 中的静态数组

```java
// 声明并初始化，长度固定为 5
int[] arr = new int[5];

// 声明并赋值
int[] arr2 = {1, 2, 3, 4, 5};
```

特点：

- 长度固定，创建后无法扩展或缩小
- 越界访问会抛出 `ArrayIndexOutOfBoundsException`
- 在栈上分配引用，在堆上分配实际数据（Java 中数组是对象）

### 时间复杂度

| 操作 | 时间复杂度 | 说明 |
|------|-----------|------|
| 按下标读取 `arr[i]` | O(1) | 寻址公式直接计算 |
| 按下标修改 `arr[i] = x` | O(1) | 同上 |
| 遍历 | O(n) | 逐个访问 |
| 按值查找（无序） | O(n) | 最坏遍历整个数组 |
| 按值查找（有序） | O(log n) | 二分查找 |
| 插入（中间位置） | O(n) | 需要搬移后续元素 |
| 删除（中间位置） | O(n) | 需要搬移后续元素 |

### 插入和删除为什么慢？

以在下标 `k` 处插入为例：

```
插入前: [1, 2, 3, 4, 5, _]
                ↑ 要在下标2插入 99

第一步: 把下标2及之后的元素全部后移一位
        [1, 2, _, 3, 4, 5]

第二步: 在下标2写入新值
        [1, 2, 99, 3, 4, 5]
```

最坏情况（头部插入）需要搬移 n 个元素，所以是 O(n)。

---

## 三、动态数组与扩容机制

### 什么是动态数组

动态数组是在静态数组基础上封装的一层抽象，**对外表现为长度可变**，内部通过**扩容**来实现。

Java 中的 `ArrayList`、C++ 中的 `vector`、Python 中的 `list` 都是动态数组。

### 核心思想

```
动态数组 = 静态数组 + 自动扩容 + size 记录实际元素个数
```

内部维护两个关键变量：

- `capacity`：底层静态数组的总长度（容量）
- `size`：当前实际存储的元素个数（`size <= capacity`）

### 扩容流程

当 `size == capacity` 时触发扩容：

```
1. 创建一个更大的新数组（通常是原来的 1.5 倍或 2 倍）
2. 把旧数组的所有元素拷贝到新数组
3. 让引用指向新数组，旧数组等待 GC 回收
```

图示：

```
扩容前 (capacity=4, size=4):
  [10, 20, 30, 40]    ← 满了，要加入 50

扩容中:
  旧数组: [10, 20, 30, 40]
  新数组: [10, 20, 30, 40, _, _]    ← capacity 变为 6（1.5倍）

扩容后 (capacity=6, size=5):
  [10, 20, 30, 40, 50, _]
```

### 扩容因子的选择

| 语言/实现 | 扩容因子 | 说明 |
|-----------|---------|------|
| Java ArrayList | 1.5 倍 | `newCapacity = oldCapacity + (oldCapacity >> 1)` |
| C++ vector（GCC） | 2 倍 | 实现相关 |
| Python list | ~1.125 倍 | 增长较保守 |
| Go slice | 小于 256 时 2 倍，之后约 1.25 倍 | 分段策略 |

> 扩容因子越大，扩容次数越少，但浪费的空间越多；反之扩容频繁但空间利用率高。1.5~2 倍是工程实践中的平衡点。

### 均摊时间复杂度

单次扩容的代价是 O(n)（拷贝所有元素），但扩容不是每次 `add` 都发生。

假设从空数组开始，连续插入 n 个元素，扩容发生在 size 为 1, 2, 4, 8, ..., n 时：

```
总拷贝次数 = 1 + 2 + 4 + 8 + ... + n ≈ 2n
```

均摊到每次插入：**O(2n) / n = O(1)**

所以动态数组尾部插入的**均摊时间复杂度是 O(1)**。

---

## 四、Java ArrayList 源码要点

### 关键字段

```java
// 底层存储数组
transient Object[] elementData;

// 实际元素个数
private int size;

// 默认初始容量
private static final int DEFAULT_CAPACITY = 10;
```

### add 方法核心逻辑

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // 检查是否需要扩容
    elementData[size++] = e;           // 放入元素，size+1
    return true;
}
```

### grow 扩容方法

```java
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); // 1.5 倍
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

### 常见面试追问

**Q：ArrayList 默认容量是多少？**
A：默认 10。但如果用无参构造器创建，初始其实是空数组 `{}`，第一次 `add` 时才扩容到 10。

**Q：为什么扩容 1.5 倍而不是 2 倍？**
A：1.5 倍扩容后，之前释放的内存块加起来可能大于新数组大小，有机会被复用；2 倍扩容则永远无法复用之前的内存。1.5 倍是空间利用率和扩容频率的折中。

**Q：ArrayList 是线程安全的吗？**
A：不是。多线程环境下应使用 `Collections.synchronizedList()` 或 `CopyOnWriteArrayList`。

---

## 五、数组的常见操作与技巧

### 1. 初始化

```java
// 默认值初始化（int 默认 0，boolean 默认 false，引用默认 null）
int[] arr = new int[10];

// 填充特定值
Arrays.fill(arr, -1);

// 二维数组
int[][] matrix = new int[3][4];
```

### 2. 拷贝

```java
// System.arraycopy —— 最快，底层 native 方法
System.arraycopy(src, srcPos, dest, destPos, length);

// Arrays.copyOf —— 内部调用 System.arraycopy
int[] newArr = Arrays.copyOf(arr, newLength);

// clone
int[] newArr2 = arr.clone();
```

### 3. 排序

```java
Arrays.sort(arr);                          // 基本类型：双轴快排
Arrays.sort(arr, (a, b) -> b - a);         // 对象类型：TimSort（稳定）
```

### 4. 二分查找

```java
int index = Arrays.binarySearch(arr, target); // 前提：数组已排序
```

### 5. 转换

```java
// 数组 → List
List<Integer> list = Arrays.stream(arr).boxed().collect(Collectors.toList());

// List → 数组
int[] arr = list.stream().mapToInt(Integer::intValue).toArray();
```

---

## 六、数组在算法题中的常见考法

| 类型 | 典型题目 | 核心思路 |
|------|---------|---------|
| 双指针 | 两数之和、三数之和、移除元素 | 排序 + 左右指针 / 快慢指针 |
| 滑动窗口 | 最小覆盖子串、长度最小的子数组 | 维护窗口的左右边界 |
| 前缀和 | 和为K的子数组、区间求和 | `prefix[i] = prefix[i-1] + arr[i]` |
| 二分查找 | 搜索旋转排序数组、寻找峰值 | 有序性 + 缩小搜索范围 |
| 原地操作 | 移除元素、合并有序数组 | 不使用额外空间，覆盖写入 |
| 矩阵 | 螺旋矩阵、旋转图像、搜索二维矩阵 | 边界控制、方向模拟 |
| 哈希辅助 | 两数之和、存在重复元素 | 用 HashMap 记录已遍历信息 |

---

## 七、小结

```
数组的本质：连续内存 + 相同类型 → O(1) 随机访问

静态数组：长度固定，增删慢（O(n)），查改快（O(1)）
动态数组：静态数组 + 自动扩容，尾部插入均摊 O(1)

面试核心考点：
  ① 寻址公式与随机访问原理
  ② 扩容机制（1.5倍、均摊O(1)）
  ③ 数组 vs 链表的取舍
  ④ 各类算法技巧（双指针、滑动窗口、前缀和...）
```
