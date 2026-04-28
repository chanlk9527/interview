# 06 - Spring Boot & Spring Cloud

---

## 一、Spring Boot 核心

1. Spring Boot 相比传统 Spring 项目，真正降低了哪些工程成本？它又带来了哪些隐式复杂度？
2. 自动配置的触发链路是什么？`@Conditional`、AutoConfiguration imports、配置属性如何协作？
3. `@SpringBootApplication` 背后的组件扫描边界如何影响项目结构和启动结果？
4. `SpringApplication.run()` 启动过程中，环境准备、事件发布、上下文刷新、Runner 执行分别做了什么？
5. 配置加载优先级、Profile、环境变量、配置中心如何合并？如何排查“配置没生效”？
6. 如何设计一个生产可用的 Starter：默认值、条件装配、配置提示、可观测性、向后兼容要怎么做？
7. Actuator 应该暴露哪些端点？健康检查、指标、日志级别、Prometheus、探针如何设计安全边界？
8. 内嵌 Web 容器的线程池、连接数、超时、最大请求体如何配置和监控？

## 二、Spring Boot 3 与现代化升级

1. 从 Spring Boot 2 升级到 3，Jakarta 包名迁移、依赖兼容、Spring Security、Actuator 指标会遇到哪些问题？
2. GraalVM Native Image / AOT 适合哪些服务？反射、动态代理、资源文件会带来什么限制？
3. 虚拟线程在 Spring Boot 服务中适合哪些阻塞 IO 场景？数据库连接池和下游限流为什么仍然是瓶颈？
4. 配置属性绑定、校验和元数据如何提升可维护性？什么时候不应该把配置做得过度动态？
5. 如何设计启动性能优化方案：Bean 数量、类路径扫描、懒加载、AOT、容器镜像分别能做什么？

## 三、Spring Cloud 与微服务治理

1. 服务注册发现、负载均衡、配置中心、网关、熔断限流、链路追踪分别解决什么治理问题？
2. Nacos、Consul、Eureka 等注册中心如何选型？一致性模型、健康检查、跨机房能力要怎么评估？
3. Spring Cloud LoadBalancer 的核心流程是什么？客户端负载均衡如何结合权重、灰度和实例元数据？
4. OpenFeign/RestClient/WebClient 如何选型？超时、重试、连接池、序列化和错误解码如何统一治理？
5. Sentinel、Resilience4j、熔断、限流、隔离、降级分别解决什么问题？如何避免“降级把问题藏起来”？
6. Spring Cloud Gateway 的过滤器链、路由匹配、限流、鉴权、灰度发布如何设计？
7. 配置中心动态刷新有什么风险？哪些配置可以热更新，哪些必须重启或灰度？
8. Micrometer Tracing、OpenTelemetry、Prometheus、日志聚合如何串起一次跨服务请求？

## 四、微服务实践问题

1. 微服务拆分应该基于业务边界还是技术分层？如何判断拆得太细或太粗？
2. 同步调用、异步消息、事件驱动、批处理分别适合什么交互场景？
3. 服务间数据一致性如何设计？本地事务、Outbox、Saga、补偿、幂等分别承担什么角色？
4. 灰度发布、蓝绿发布、金丝雀发布如何配合网关、注册中心和配置中心？
5. 多服务故障时，如何防止重试风暴、级联超时和线程池被打满？
6. 微服务日志、指标、Trace、审计事件如何统一规范，方便排障而不是只增加噪音？
7. 容器化部署时，启动探针、就绪探针、优雅停机、资源限制如何影响服务稳定性？

## 五、Dubbo 与 RPC（结合项目经验）

1. Dubbo 的 Provider、Consumer、Registry、Protocol、Cluster 分别承担什么职责？
2. RPC 和 HTTP API 如何选型？性能、治理能力、跨语言、调试成本分别如何权衡？
3. Dubbo 的负载均衡、路由、容错、超时、重试如何组合才不会放大故障？
4. 序列化协议如何选择？兼容性、性能、安全、Schema 演进分别要考虑什么？
5. 注册中心异常、Provider 半宕机、Consumer 线程池打满时，Dubbo 调用链会出现什么现象？
6. Dubbo 和 Spring Cloud 在服务治理模型上有什么差异？混合架构中如何统一观测和治理？
