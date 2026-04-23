# 12 - Netty 与网络编程

---

## 一、IO 模型

1. 五种 IO 模型？阻塞IO、非阻塞IO、IO多路复用、信号驱动IO、异步IO？
2. select、poll、epoll 的区别？为什么 epoll 性能更好？
3. 水平触发（LT）和边缘触发（ET）的区别？
4. Java NIO 的核心组件？Channel、Buffer、Selector？

## 二、Netty 核心

1. Netty 的整体架构？为什么选择 Netty 而不是直接用 Java NIO？
2. Reactor 线程模型？单 Reactor 单线程、单 Reactor 多线程、主从 Reactor？
3. Netty 的线程模型？BossGroup 和 WorkerGroup？EventLoop？
4. Channel 和 ChannelPipeline 的关系？Handler 的执行顺序？
5. ChannelInboundHandler 和 ChannelOutboundHandler 的区别？
6. Bootstrap 和 ServerBootstrap 的区别？

## 三、编解码与协议

1. TCP 粘包/拆包问题？产生原因和解决方案？
2. Netty 提供的编解码器？LengthFieldBasedFrameDecoder、DelimiterBasedFrameDecoder？
3. 自定义协议的设计？魔数、版本号、长度字段、消息体？
4. Protobuf 编解码的使用？

## 四、内存管理

1. ByteBuf 和 Java NIO ByteBuffer 的区别？
2. 池化（Pooled）和非池化（Unpooled）ByteBuf 的区别？
3. 堆内存（HeapByteBuf）和直接内存（DirectByteBuf）的区别？
4. CompositeByteBuf 的作用？零拷贝？
5. 引用计数和内存泄漏检测？

## 五、高级特性

1. Netty 的零拷贝实现？FileRegion、CompositeByteBuf、wrap？
2. 心跳检测机制？IdleStateHandler 的使用？
3. 断线重连的实现方案？
4. Netty 的流量整形？（TrafficShaping）
5. Netty 的优雅关闭？

## 六、实战场景

1. Netty 在你的 IoT 平台中是怎么用的？
2. 如何管理百万级长连接？
3. 如何做连接的集群化管理？
4. Netty 的性能调优经验？
