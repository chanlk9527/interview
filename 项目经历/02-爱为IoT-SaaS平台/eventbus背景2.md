# Topic 与 TopicUtils 说明

## 1. 先说结论

`Topic` 和 `TopicUtils` 一起构成了 EventBus 的“路由层”。

- `TopicUtils` 负责 Topic 字符串本身的规则处理
  - 标准化
  - 拆分层级
  - 通配符判断
  - 提供一个独立的“模式是否匹配 topic”算法
- `Topic<T>` 负责把“订阅模式 -> 值”存进一棵树里，并在发布时高效找出所有命中的值

可以把它们理解成分工明确的两层：

- `TopicUtils` 解决“字符串规则”
- `Topic<T>` 解决“数据结构和查找”

在当前项目里，`DefaultEventBus` 真正依赖的是 `Topic<T>` 这棵树来做订阅匹配；`TopicUtils.matches(...)` 更像一个独立、可测试的基础能力，用来表达和验证 Topic 规则。

## 2. 为什么 Topic 看起来复杂

因为这里的 Topic 不是普通字符串相等比较，而是“分层路径 + 通配符匹配”。

例如这些模式都合法：

```text
/device/dev-01/message
/device/*/message
/device/**/message
/device/**
/
```

它们的语义不同：

- 精确值：`/device/dev-01/message`
- 单层通配：`*`，只能吃掉一层
- 多层通配：`**`，可以吃掉零层、一层或多层

一旦支持 `**`，匹配就不再是简单的“从左到右一一比较”，因为 `**` 有分支选择：

- 它可以先匹配 0 层，继续看后面
- 也可以先吞掉 1 层，再继续保持自己还有效

这就是它看起来复杂的根本原因。复杂点不在“树”，而在“多层通配符会带来回溯/分支”。

## 3. TopicUtils 在做什么

文件位置：[TopicUtils.java](/C:/Users/chanlk/Work/projects/iotmock/eventbus/src/main/java/com/example/eventbus/topic/TopicUtils.java)

### 3.1 `normalize(String topic)`

作用：把外部传进来的 Topic 统一成系统内部格式。

它主要做了几件事：

- 去掉首尾空白
- 合并多余的 `/`
- 去掉空 segment
- 校验通配符是否合法
- 最终统一成以 `/` 开头的格式

例子：

```text
"device//dev-01/message/" -> "/device/dev-01/message"
" /device///dev-01/ "     -> "/device/dev-01"
"/"                       -> "/"
```

为什么这一步很重要：

- 避免同一个 Topic 因为格式不同被当成不同路径
- 让后续匹配逻辑不必反复处理脏输入
- 订阅时和发布时都使用同一套规范，行为更稳定

### 3.2 `validateSegment(String segment)`

作用：限制通配符只能是完整的 `*` 或 `**`。

也就是说这些是非法的：

```text
abc*
de*ice
***
foo**bar
```

这样做是为了让规则足够清晰，避免把 Topic 规则做成半个正则表达式系统。

### 3.3 `segments(String topic)`

作用：把标准化后的 Topic 拆成层级片段。

例子：

```text
"/device/dev-01/message" -> ["device", "dev-01", "message"]
"/" -> []
```

注意根路径 `/` 会被表示成空列表，这一点对后面的树结构和匹配算法都很重要。

### 3.4 `matches(String pattern, String topic)`

作用：独立判断“某个模式是否匹配某个实际 topic”。

这个方法没有走 `Topic<T>` 树，而是直接基于 segment 做递归匹配。

支持规则：

- 普通字符串：必须完全相等
- `*`：匹配恰好一层
- `**`：匹配零层或多层

例如：

```text
"/device/*/message"  匹配 "/device/dev-01/message"
"/device/*/message"  不匹配 "/device/product-01/dev-01/message"
"/device/**/message" 匹配 "/device/product-01/dev-01/message"
```

### 3.5 为什么 `matches(...)` 里要做 memo

`**` 会引入递归分支。比如当前模式是 `**` 时，会同时尝试两条路：

1. `**` 先匹配 0 层，模式向后走一步
2. `**` 继续吞掉当前层，topic 向后走一步，但模式停在原位

如果不做 memo，同一组状态可能反复计算。这里用 `(patternIndex, topicIndex)` 做缓存键，是一个典型的动态规划/记忆化搜索写法。

它的作用不是改变语义，而是：

- 避免重复计算
- 降低 `**` 带来的回溯成本
- 让匹配在复杂模式下依然可控

## 4. Topic<T> 在做什么

文件位置：[Topic.java](/C:/Users/chanlk/Work/projects/iotmock/eventbus/src/main/java/com/example/eventbus/topic/Topic.java)

`Topic<T>` 是一个“按路径分层的前缀树”。

它存的不是单纯的 Topic 字符串，而是：

```text
Topic pattern -> 一组绑定值
```

在 EventBus 里，这个 `T` 实际上就是订阅者包装对象 `SubscriberSink<?>`。

所以你可以把它理解成：

```text
订阅模式 -> 订阅者
```

### 4.1 为什么要用树，不直接遍历所有订阅

如果不用树，发布一条消息时，最直接的方式是：

1. 遍历所有订阅模式
2. 一个个调用 `TopicUtils.matches(pattern, topic)`
3. 收集命中的订阅者

这样写最简单，但订阅一多，发布时每次都要全表扫描。

`Topic<T>` 的价值是把共享前缀组织起来：

```text
/device/dev-01/message
/device/dev-02/message
/device/*/message
/device/**
```

这些模式都共享 `/device` 前缀。树结构能复用这部分路径，匹配时也只沿着相关分支走，不必每次从所有模式里重新筛一遍。

### 4.2 内部结构

每个节点 `Node<T>` 里有两部分核心数据：

- `children`：下一层 segment 到子节点的映射
- `values`：正好绑定在当前路径上的值

可以粗略想成这样：

```text
root
└── device
    ├── **
    │   └── values: ["all-device"]
    ├── *
    │   └── message
    │       └── values: ["single-level"]
    └── dev-01
        └── message
            └── values: ["exact"]
```

### 4.3 `add(String pattern, T value)`

作用：把一个 Topic 模式注册进树里。

处理过程很直接：

1. 先把模式拆成 segments
2. 从根节点出发逐层向下
3. 没有子节点就创建
4. 走到最后一个 segment 后，把 value 放进当前节点的 `values`

例如注册：

```text
"/device/*/message"
```

树里就会形成：

```text
root -> "device" -> "*" -> "message"
```

### 4.4 `remove(String pattern, T value)`

作用：把某个模式上的某个值解绑。

它不只是删掉 `values` 里的元素，还会顺便做一件事：

- 如果某个子节点已经没有 `values`，也没有 `children`，就把这个空节点剪掉

这相当于树的“垃圾回收”，避免残留无意义节点。

### 4.5 `match(String topic)`

作用：找出所有与某个实际 Topic 匹配的值。

这是整个类最关键的能力。返回结果不是只找一个，而是找“全部命中项”。

例如当前树里有：

```text
"/device/**"            -> "all-device"
"/device/*/message"     -> "single-level"
"/device/dev-01/message" -> "exact"
```

匹配：

```text
"/device/dev-01/message"
```

最终会同时命中：

```text
"all-device"
"single-level"
"exact"
```

这正是 `TopicTest` 里验证的行为。

## 5. `Topic.match(...)` 为什么更复杂

最复杂的部分在 `collect(...)`。

原因是它不是单纯检查“是否匹配”，而是要在一棵混合了：

- 精确节点
- `*` 节点
- `**` 节点

的树中，把所有可能命中的值都找出来。

### 5.1 它会同时探索三类分支

对于当前 topic 的某一层，比如 `dev-01`，它会看：

1. 有没有精确子节点 `dev-01`
2. 有没有单层通配子节点 `*`
3. 有没有多层通配子节点 `**`

也就是说，一次匹配不一定只有一条路径，而可能有多条路径同时成立。

### 5.2 `**` 的两种语义都要支持

进入 `**` 子节点以后，`collect(...)` 需要同时支持：

1. `**` 匹配 0 层
2. `**` 匹配 1 层或更多层

所以你会看到两种行为：

- 进入 `**` 子节点时，不立刻消费当前层，表示“先试试匹配 0 层”
- 如果当前节点本身就是 `**`，又会继续用同一个节点消费下一层，表示“再多吞一层”

这两个动作合在一起，才完整表达了 `**` 的语义。

### 5.3 为什么需要 `visited`

`collect(...)` 里用 `State(node, index)` 记录访问状态。

这是因为 `**` 会造成“留在当前节点，但 topic index 继续向前”的搜索路径。如果没有去重，某些状态组合可能被重复走到，甚至出现无限递归风险。

`visited` 的意义是：

- 同一个节点
- 同一个 topic 下标

只处理一次。

这和 `TopicUtils.matches(...)` 里的 memo 本质上是同一个思路，只是一个用在“树搜索”，一个用在“模式匹配”。

## 6. 这两个类在 EventBus 里是怎么配合的

链路大致是这样：

1. 创建订阅时，`Subscription` 会先调用 `TopicUtils.normalize(...)` 规范订阅模式
2. `DefaultEventBus.subscribe(...)` 把订阅者注册进 `Topic<SubscriberSink<?>>`
3. 发布消息时，`TopicPayload` 会先把实际 topic 做标准化
4. `DefaultEventBus.publish(...)` 再调用 `topicTree.match(topic)` 找到所有匹配订阅者

对应代码可以看：

- [Subscription.java](/C:/Users/chanlk/Work/projects/iotmock/eventbus/src/main/java/com/example/eventbus/Subscription.java)
- [TopicPayload.java](/C:/Users/chanlk/Work/projects/iotmock/eventbus/src/main/java/com/example/eventbus/TopicPayload.java)
- [DefaultEventBus.java](/C:/Users/chanlk/Work/projects/iotmock/eventbus/src/main/java/com/example/eventbus/DefaultEventBus.java)

所以整体关系不是：

```text
EventBus 直接靠字符串做匹配
```

而是：

```text
TopicUtils 负责统一规则
Topic<T> 负责存储和查找
DefaultEventBus 负责在发布/订阅流程里使用它们
```

## 7. 一个完整例子

假设注册了 3 个订阅：

```text
"/device/**"
"/device/*/message"
"/device/dev-01/message"
```

发布：

```text
"/device/dev-01/message"
```

处理过程大概是：

1. `TopicPayload` 先把 topic 规范成 `/device/dev-01/message`
2. `Topic.match(...)` 从根节点开始
3. 先进入 `device`
4. 接下来同时尝试：
   - 精确分支 `dev-01`
   - 单层通配 `*`
   - 多层通配 `**`
5. 最终收集到 3 个绑定值
6. `DefaultEventBus` 再继续做 shared、priority 等投递策略

这里要特别注意：

- `Topic` 只负责回答“谁匹配”
- 不负责回答“谁先投递”
- 也不负责 shared 组选一个

后两者是在 `DefaultEventBus` 里处理的

## 8. 如果你要抓重点，记住这几句就够了

- `TopicUtils` 是 Topic 规则引擎，处理格式、层级和通配符语义
- `Topic<T>` 是 Topic 索引结构，用树保存订阅关系并做高效匹配
- 真正让代码变复杂的是 `**`，因为它可以匹配零层到多层
- `memo` 和 `visited` 都是在控制 `**` 带来的搜索分支和重复计算
- `Topic` 负责“找出所有命中项”，`DefaultEventBus` 负责“决定怎么投递这些命中项”

## 9. 对应测试

如果你想顺着仓库里的例子继续看，最适合配合阅读的是：

- [TopicTest.java](/C:/Users/chanlk/Work/projects/iotmock/eventbus/src/test/java/com/example/eventbus/TopicTest.java)

这个测试已经覆盖了最关键的三件事：

- Topic 标准化
- 通配符匹配
- Topic 树的命中与删除

