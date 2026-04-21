# EventBus 面试常见追问与应答策略

## 背景

本文档整理了作为面试者，在介绍 EventBus 项目时可能遇到的典型追问，以及推荐的应答思路。核心目标是帮助面试者清晰地向面试官解释：这个 EventBus 是什么、不是什么、为什么要自己做。

---

## 1. "这和 Kafka / RabbitMQ 有什么区别？"

这是最常见的第一个追问。面试官可能没有仔细区分进程内事件总线和分布式消息中间件。

### 一句话定位

> EventBus 是进程内的方法调用路由器，Kafka 是跨进程的持久化消息管道。它们解决的问题层次完全不同。

### 类比

- Kafka 像邮局——信件投递、存储、确认签收、支持多个收件人各自按自己的节奏取信。
- EventBus 像办公室里喊一嗓子——同一个进程里，谁关心谁就听到，喊完就没了。

### 三个维度对比

| 维度 | EventBus（本项目） | Kafka / RabbitMQ |
|------|-------------------|-----------------|
| 部署边界 | 单进程内存，和业务代码跑在同一个 JVM | 独立集群，跨网络通信 |
| 消息生命周期 | 发完即忘，不落盘，没有重放 | 持久化到磁盘，支持 offset 回溯 |
| 交付保证 | 无 ACK、无重试、subscriber 挂了消息就丢了 | at-least-once / exactly-once，有 dead letter |

### 推荐话术

> 这两个东西解决的问题不在一个层面。
>
> Kafka 解决的是跨服务、跨进程的异步通信——消息要落盘、要保序、要支持消费者各自维护 offset、要处理网络分区。它本质上是一个分布式日志系统。
>
> 我做的 EventBus 解决的是同一个 JVM 进程内模块之间的解耦。比如 IoT 平台里，设备接入模块收到一条属性上报，规则引擎、告警、日志这些模块都要处理。如果直接互相调用就耦合死了，所以用一个进程内的 pub/sub 路由器来分发。
>
> 它不需要网络、不需要持久化、不需要 ACK。核心难点在 topic 通配符匹配的数据结构设计（我用了 trie 树）、shared subscription 的投递语义、以及并发安全。
>
> 如果要类比的话，它更接近 Guava EventBus 或 Spring ApplicationEvent，而不是 Kafka。

### 如果继续追问"那为什么不直接用 Kafka"

> 因为场景不需要。这些模块跑在同一个进程里，走网络绕一圈 Kafka 再回来，延迟从微秒级变成毫秒级，还多了一个外部依赖要运维。进程内通信用进程内方案，跨进程通信才用 Kafka，各司其职。

---

## 2. "那为什么不用 Guava EventBus 或 Spring ApplicationEvent？"

这是面试官确认你定位清楚之后的自然追问：既然是进程内方案，为什么不用现成的？

### 核心回答

> Guava EventBus 和 Spring ApplicationEvent 都是进程内 pub/sub，但它们的路由模型太简单了，撑不住我们的场景。我们需要的是一个基于层级 topic 的路由器，不是一个基于类型的事件广播器。

### 对比 Guava EventBus

| 维度 | Guava EventBus | 本项目 EventBus |
|------|---------------|----------------|
| 路由方式 | 按 Java 类型匹配（发 OrderEvent，所有订阅 OrderEvent 的 handler 收到） | 按层级 topic + 通配符匹配 |
| 通配符 | 不支持 | 支持 `*` 单层、`**` 多层 |
| 共享订阅 | 不支持 | 同组内只投一个，负载均衡语义 |
| 优先级 | 不支持 | 订阅者可设优先级，高优先级先收到 |
| 取消订阅 | 要手动 unregister 整个对象 | 返回 Disposable，细粒度取消单个订阅 |

> Guava EventBus 本质上是个类型驱动的观察者模式工具。它没有 topic 概念，没法表达 `/device/product-01/*/message/**` 这种层级路由。在 IoT 场景里，设备有产品、设备 ID、消息类型这些层级，用类型匹配根本表达不了。

### 对比 Spring ApplicationEvent

| 维度 | Spring Event | 本项目 EventBus |
|------|-------------|----------------|
| 路由方式 | 按事件类型 + 可选 SpEL 条件过滤 | 按层级 topic 树 + 通配符 |
| 通配符路由 | 不支持 | 核心能力 |
| 共享订阅 | 不支持 | 支持 |
| 与 Spring 耦合 | 强绑定 Spring 容器 | 独立组件，无框架依赖 |
| 动态订阅/取消 | 不方便，listener 通常是 bean 生命周期绑定的 | 运行时随时订阅、随时 dispose |

> Spring Event 的问题是两个：一是路由能力弱，它本质还是按类型广播，SpEL 条件过滤性能也不好；二是和 Spring 容器强绑定，订阅者必须是 Spring Bean，动态订阅和取消很别扭。我们的场景里设备随时上下线，订阅关系是动态变化的，需要细粒度的生命周期管理。

### 收尾

> 所以核心原因是：我们需要的是一个基于层级 topic 的路由器，不是一个基于类型的事件广播器。Guava 和 Spring 的模型从根上就不匹配这个需求。自己做一个，反而代码量不大，但语义完全可控。

---

## 3. 应答策略总结

面试中回答这类"为什么不用 XXX"的问题，核心逻辑是三层递进：

1. **先定位层次差异**：Kafka 是跨进程的，我这个是进程内的，不在一个层面。
2. **再定位模型差异**：Guava/Spring 虽然也是进程内，但它们是类型驱动的广播器，我需要的是 topic 层级路由器。
3. **最后点明具体功能缺口**：通配符匹配、shared subscription、优先级、动态生命周期管理——这些现有方案都不支持。

这样回答的好处：
- 不是"我想练手造轮子"，而是"现有方案功能不够"
- 具体指出了哪里不够，展示了技术调研能力
- 暗示你对 Guava 和 Spring 的内部机制都了解，不是没调研就动手
- 展示了架构选型的判断力——知道什么场景该用什么工具


## 4. "你的 IoT 平台不是集群部署吗？难道没有跨节点通信？这个 EventBus 只是进程内的？"

这个追问比较尖锐，面试官在试探你是不是只做了个玩具。你需要承认集群场景确实存在，但把架构分层讲清楚。

### 核心回答

> 对，生产环境确实是集群部署，跨节点通信是必须的。但架构上我们是分层设计的：
>
> **本地 EventBus 是底层路由引擎**，负责单节点内的 topic 匹配和投递。它不关心消息从哪来。
>
> **上面有一层 Broker 机制**，负责跨节点的消息转发。当一个节点收到消息，Broker 判断"这条消息在其他节点上有没有人订阅"，如果有，就通过网络转发过去。对端节点收到后，再交给本地 EventBus 做路由投递。

### EventBus 与 Broker 的职责划分

| 层次 | 职责 | 关心的问题 |
|------|------|-----------|
| EventBus（本地层） | 本节点内谁该收到这条消息 | topic 匹配、优先级、shared 投递、并发安全 |
| Broker（集群层） | 消息要不要转发到其他节点 | 网络通信、序列化、节点发现、重试 |

> 这样分层的好处是：本地路由逻辑不被网络、序列化、重试这些分布式问题污染。EventBus 的 topic 匹配、优先级、shared subscription 这些语义，在单节点和集群模式下是完全一致的。

### 架构示意

```
┌─────────────────────────────────────────────┐
│                  Node A                      │
│                                             │
│  [模块1] ──publish──→ [EventBus] ──→ [模块2] │
│                          │                  │
│                          ↓                  │
│                      [Broker] ──── 网络 ────→ Node B 的 Broker
│                                             │        ↓
└─────────────────────────────────────────────┘   Node B 的 EventBus
                                                       ↓
                                                  [模块3]
```

### 如果继续追问 Broker 怎么实现

> JetLinks 原版的做法是定义一个 EventBroker 接口，每个节点启动时向 EventBus 注册自己的 Broker。发布消息时，EventBus 除了投递给本地订阅者，还会把消息交给所有注册的 Broker，由 Broker 决定要不要转发。
>
> 底层通信可以用各种方式——集群内部用 Redis Pub/Sub、用 RSocket、用 gRPC 都行。Broker 只是一个转发策略的抽象，具体传输协议是可插拔的。
>
> 我这个项目聚焦在本地 EventBus 这一层，Broker 层没有实现。但设计上是预留了扩展点的——EventBus 的 subscribe 有一个 Feature 枚举，其中 `local` 表示只接收本地消息，如果去掉这个 feature 就意味着也接收远程转发过来的消息。

### 回答这个问题要传达的三个信息

1. **你知道集群场景存在**——不是闭着眼睛只做单机
2. **架构是分层的**——EventBus 和 Broker 职责分离，不是一坨
3. **你的项目有明确的 scope**——聚焦本地路由层，Broker 层是有意不做的，不是不知道要做

---


## 5. "如果做一个基于 Redis 的 Broker，你会怎么设计？"

这个追问考察你能不能从本地设计自然延伸到分布式落地，同时展示你对 Redis Pub/Sub 特性和局限性的理解。

### 核心思路

> Redis Broker 的本质是把 Redis 当作节点之间的消息中转站。每个节点既是 publisher 也是 subscriber。Redis 只负责"把消息送到节点"，精确的 topic 路由仍然由各节点本地的 EventBus 完成。

### 消息流转全景

```
Node A publish("/device/dev-01/property", payload)
    │
    ├──→ 本地 EventBus 投递给 Node A 的本地订阅者
    │
    └──→ Redis Broker 序列化后 PUBLISH 到 Redis Channel
              │
              ↓
         Redis Server 广播给所有订阅了该 Channel 的节点
              │
              ├──→ Node B 收到 → 本地 EventBus 做 topic 匹配 → 投递
              └──→ Node C 收到 → 本地 EventBus 做 topic 匹配 → 投递
```

### 三个关键设计决策

**1. Redis Channel 怎么映射 Topic？**

不能把每个具体 topic 都映射成一个 Redis Channel，否则 channel 数量会爆炸。常见做法：

- 简单方案：用一个统一 channel（如 `eventbus:broadcast`），消息体里带原始 topic，对端收到后由本地 EventBus 做精确匹配
- 优化方案：按业务域分几个 channel（如 `eventbus:device`、`eventbus:rule`），减少无关节点的处理量

> Redis 只管广播到节点，精确的通配符路由还是本地 EventBus 的事。

**2. 消息格式**

序列化后的消息至少包含三个字段：

| 字段 | 用途 |
|------|------|
| sourceNodeId | 来源节点标识，避免自己发的消息又被自己消费 |
| topic | 原始 topic 路径，对端用来做本地路由 |
| payload | 消息体的字节数组 |

**3. 避免无效转发（可选优化）**

> 如果 Node B 上根本没人订阅 `/device/**`，那这类消息就不需要发给 Node B。做法是每个节点把自己的订阅 pattern 注册到 Redis（比如用一个 Set），Broker 发消息前先判断哪些节点有匹配的订阅，只转发给需要的节点。
>
> 但这会增加复杂度和一致性问题。简单版先全量广播，让对端自己过滤即可。

### 伪代码示意

```java
public class RedisBroker implements EventBroker {

    private final RedisTemplate<String, byte[]> redis;
    private final String nodeId;
    private final EventBus localEventBus;

    // 本地 EventBus 发布时调用，把消息转发到集群
    @Override
    public void forward(String topic, Payload payload) {
        BrokerMessage msg = new BrokerMessage(nodeId, topic, payload.getBytes());
        redis.convertAndSend("eventbus:broadcast", serialize(msg));
    }

    // 监听 Redis Channel，收到远程消息后交给本地 EventBus
    @Override
    public void onMessage(String channel, byte[] data) {
        BrokerMessage msg = deserialize(data);
        if (msg.getSourceNodeId().equals(nodeId)) {
            return; // 忽略自己发出的消息
        }
        localEventBus.publishFromRemote(msg.getTopic(), msg.getPayload());
    }
}
```

### Redis Pub/Sub 的局限性与选型建议

| 特性 | Redis Pub/Sub | Redis Streams | Kafka |
|------|:---:|:---:|:---:|
| 消息持久化 | ❌ | ✅ | ✅ |
| 断线重连后补发 | ❌ | ✅ | ✅ |
| 消费者组 / ACK | ❌ | ✅ | ✅ |
| 部署复杂度 | 低 | 中 | 高 |

> Redis Pub/Sub 的明确局限是：消息不持久化，没有 ACK。节点断线期间的消息会丢失。
>
> 如果业务能容忍偶尔丢消息（比如设备状态上报，下一次上报会覆盖），Redis Pub/Sub 够用且最简单。如果不能容忍，可以升级到 Redis Streams（支持消费者组和 ACK）或直接上 Kafka。
>
> 所以 Broker 接口设计成可插拔的，底层传输根据业务对可靠性的要求来选型。

### 一句话总结

> Redis Broker 就是：用 Redis 做节点间的广播通道，每个节点收到远程消息后，复用本地 EventBus 的 topic 匹配能力做精确投递。职责边界清晰——Redis 管传输，EventBus 管路由。

---
