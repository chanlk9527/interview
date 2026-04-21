# EventBus 学习项目 — 背景与需求文档

## 1. 背景

### 1.1 什么是 EventBus？

EventBus（事件总线）是一种**发布-订阅（Pub/Sub）**通信模式的实现。它充当系统组件之间的中央通信通道，允许组件之间进行松耦合的异步通信。发布者将事件发送到总线，订阅者从总线接收感兴趣的事件，双方无需相互直接引用。

### 1.2 JetLinks EventBus 概述

[JetLinks](https://github.com/jetlinks/jetlinks-community) 是一个企业级物联网平台，其核心模块 `jetlinks-core` 中定义了 `EventBus` 接口，由 `jetlinks-supports` 模块中的 `BrokerEventBus` 实现。

JetLinks EventBus 的核心设计理念：

| 设计要素 | 说明 |
|---------|------|
| **基于 Topic 的路由** | 消息通过层级 Topic（如 `/device/product-01/device-001/message`）进行路由 |
| **树形 Topic 匹配** | 使用树结构存储订阅关系，支持 `*`（单层通配）和 `**`（多层通配） |
| **发布/订阅模式** | 发布者将事件发送到指定 Topic，所有匹配的订阅者都会收到消息 |
| **共享订阅** | 同一 subscriber ID 的多个实例中，只有一个会收到消息（负载均衡） |
| **订阅优先级** | 订阅者可设置优先级，优先级高的先收到消息 |
| **Broker 机制** | 支持集群间消息转发，通过 EventBroker 实现跨节点通信 |
| **响应式实现** | 基于 Project Reactor（Flux/Mono）实现异步非阻塞处理 |

### 1.3 本项目目标

本项目旨在通过**模仿 JetLinks EventBus 的设计思想**，创建一个非响应式（基于 `CompletableFuture` + 回调）的简化版 EventBus，以深入理解其核心原理。

**关键约束**：
- ❌ 不使用响应式编程（不使用 Reactor Flux/Mono）
- ❌ 不实现 Broker/集群功能（聚焦本地 EventBus）
- ✅ 使用标准 Java（JDK 17+）
- ✅ 保留 JetLinks 的核心设计精髓

---

## 2. 核心功能需求

### 2.1 Topic 系统

模仿 JetLinks 的层级 Topic 体系：

```
/device/product-01/device-001/message/property
/device/product-01/device-001/message/event
/device/product-01/*/message/**
/device/**/message
```

**功能点**：
- Topic 以 `/` 分隔层级
- 支持单层通配符 `*`（匹配路径中的单个层级）
- 支持多层通配符 `**`（匹配路径中的零个或多个层级）
- 使用树形数据结构（`Topic<T>` 树）高效存储和查找订阅关系

### 2.2 发布（Publish）

```java
// 伪代码示意
CompletableFuture<Long> result = eventBus.publish("/device/product-01/dev-001/message", payload);
```

**功能点**：
- 向指定 Topic 发布消息
- 通过 Topic 树查找所有匹配的订阅者
- 返回 `CompletableFuture<Long>`，在异步投递完成后给出成功投递数量
- 支持 `Encoder` 接口对消息进行序列化

### 2.3 订阅（Subscribe）

```java
// 伪代码示意
Disposable sub = eventBus.subscribe(
    Subscription.of("subscriber-1", "/device/**"),
    MyEvent.class,
    event -> { /* handle event */ }
);
// 稍后取消订阅
sub.dispose();
```

**功能点**：
- 通过 `Subscription` 对象描述订阅（订阅者 ID、Topic 列表、特性）
- 支持类型安全的事件反序列化（通过 `Decoder`）
- 支持取消订阅（通过 `Disposable`）
- 支持回调方式接收消息

### 2.4 共享订阅（Shared Subscription）

JetLinks 中，当多个订阅者使用相同的 subscriber ID 并启用 `shared` 特性时，同一条消息只会投递给其中一个订阅者（随机选择）。

**功能点**：
- `Feature.shared`：同一 subscriber ID 的多个 sink 中随机选一个投递
- 支持负载均衡语义

### 2.5 订阅优先级

```java
Subscription.builder()
    .subscriberId("high-priority")
    .topics("/device/**")
    .priority(1)  // 值越小优先级越高
    .build();
```

**功能点**：
- 订阅者可以设置 priority（默认 `Integer.MAX_VALUE`）
- 优先级高的订阅者先收到消息

### 2.6 编解码（Codec）

JetLinks 使用 `Encoder` 和 `Decoder` 将事件对象序列化为 `Payload` 以及从 `Payload` 反序列化：

**功能点**：
- `Encoder<T>`：将事件对象编码为 `Payload`（字节数据）
- `Decoder<T>`：将 `Payload` 解码为事件对象
- 内置 JSON 编解码支持
- `Codecs` 注册表：按类型查找对应的编解码器

---

## 3. 核心类设计（对比 JetLinks）

| JetLinks 原始类 | 我们的简化版 | 说明 |
|:---|:---|:---|
| `EventBus` 接口 | `EventBus` 接口 | 核心发布/订阅 API |
| `BrokerEventBus` | `DefaultEventBus` | 本地 EventBus 实现（去除 Broker 逻辑） |
| `Subscription` | `Subscription` | 订阅信息（subscriber、topics、features、priority） |
| `Subscription.Feature` | `Subscription.Feature` | 订阅特性枚举（简化：仅保留 `local` 和 `shared`） |
| `TopicPayload` | `TopicPayload` | 携带 Topic 信息的消息载体 |
| `Payload` | `Payload` | 消息数据的抽象（简化：基于 `byte[]`） |
| `Encoder<T>` | `Encoder<T>` | 编码器接口 |
| `Decoder<T>` | `Decoder<T>` | 解码器接口 |
| `Codecs` | `Codecs` | 编解码器注册表 |
| `Topic<T>` | `Topic<T>` | 树形 Topic 数据结构 |

---

## 4. 与 JetLinks 的差异

| 方面 | JetLinks 原版 | 我们的简化版 |
|:---|:---|:---|
| **编程模型** | 响应式（Flux/Mono） | `CompletableFuture` + 回调（`Consumer<T>`） |
| **Broker 支持** | 支持集群消息转发 | ❌ 仅支持本地 |
| **Payload 实现** | 基于 Netty ByteBuf（引用计数） | 基于 `byte[]`（无引用计数） |
| **线程模型** | Reactor Scheduler | Java `ExecutorService`（可选异步） |
| **Feature 集合** | local/broker/shared/sharedOldest/... | 简化为 local/shared |
| **序列化** | 自定义 + FastJSON | Jackson JSON |

---

## 5. 项目结构

```
eventbus/
├── EVENTBUS_READING_GUIDE.md         # 面向初学者的术语与阅读说明
├── pom.xml
└── src/
    ├── main/java/com/example/eventbus/
    │   ├── EventBus.java                  # 核心接口
    │   ├── DefaultEventBus.java           # 本地实现
    │   ├── Subscription.java              # 订阅信息
    │   ├── TopicPayload.java              # 消息载体
    │   ├── Payload.java                   # 消息数据抽象
    │   ├── Disposable.java                # 取消订阅接口
    │   ├── codec/
    │   │   ├── Encoder.java               # 编码器接口
    │   │   ├── Decoder.java               # 解码器接口
    │   │   ├── Codecs.java                # 编解码注册表
    │   │   └── JsonCodec.java             # JSON 编解码
    │   └── topic/
    │       ├── Topic.java                 # 树形 Topic 节点
    │       └── TopicUtils.java            # Topic 工具类（通配符匹配等）
    └── test/java/com/example/eventbus/
        ├── DefaultEventBusTest.java       # EventBus 单元测试
        ├── TopicTest.java                 # Topic 树测试
        └── CodecsTest.java               # 编解码测试
```

---

## 6. 技术栈

| 技术 | 版本 | 用途 |
|:---|:---|:---|
| Java | 17+ | 主语言 |
| Maven | 3.8+ | 构建工具 |
| Jackson | 2.15+ | JSON 序列化/反序列化 |
| JUnit 5 | 5.10+ | 单元测试 |
| SLF4J + Logback | 最新稳定版 | 日志 |
| Lombok | 最新稳定版 | 减少样板代码（可选） |

---

## 7. 实施计划

### Phase 1：基础设施
1. 初始化 Maven 项目
2. 实现 `Payload` / `TopicPayload`
3. 实现 `Encoder` / `Decoder` / `Codecs` / `JsonCodec`

### Phase 2：Topic 树
4. 实现 `TopicUtils`（路径解析、通配符判断）
5. 实现 `Topic<T>` 树结构（添加节点、查找匹配、订阅/取消订阅）

### Phase 3：EventBus 核心
6. 定义 `EventBus` 接口
7. 实现 `Subscription`（Builder 模式、Feature 枚举）
8. 实现 `DefaultEventBus`（发布、订阅、共享订阅、优先级）
9. 实现 `Disposable` 取消订阅机制

### Phase 4：测试与验证
10. 编写 `TopicTest`（通配符匹配）
11. 编写 `DefaultEventBusTest`（发布/订阅、共享订阅、优先级、取消订阅）
12. 编写 `CodecsTest`（JSON 序列化/反序列化）

---

## 8. 学习要点

通过本项目，你将深入理解：

1. **树形路由**：如何用树结构高效实现层级 Topic 的匹配和通配符支持
2. **发布-订阅模式**：解耦的消息通信机制
3. **共享订阅**：负载均衡场景下如何让同一组订阅者只有一个收到消息
4. **编解码抽象**：如何设计通用的序列化/反序列化接口
5. **订阅生命周期管理**：如何优雅地处理订阅和取消订阅
