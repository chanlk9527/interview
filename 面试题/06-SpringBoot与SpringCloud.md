# 06 - Spring Boot & Spring Cloud

---

## 一、Spring Boot 核心

1. Spring Boot 的核心优势？和传统 Spring 项目的区别？
2. 自动配置（Auto-Configuration）的原理？@EnableAutoConfiguration 做了什么？
3. @SpringBootApplication 包含哪些注解？各自的作用？
4. Spring Boot 的启动流程？SpringApplication.run() 做了什么？
5. 配置文件加载顺序？application.yml vs application.properties？Profile 机制？
6. Spring Boot Starter 的原理？如何自定义一个 Starter？
7. Spring Boot Actuator 的作用？常用的监控端点？
8. 内嵌 Tomcat 的原理？如何切换为 Undertow 或 Jetty？

## 二、Spring Cloud 核心组件

1. Spring Cloud 的整体架构？核心组件有哪些？
2. 服务注册与发现：Eureka vs Nacos vs Consul 的对比？
3. 负载均衡：Ribbon / Spring Cloud LoadBalancer 的策略？
4. 服务调用：Feign / OpenFeign 的原理？
5. 熔断降级：Hystrix vs Sentinel 的对比？
6. 网关：Spring Cloud Gateway vs Zuul 的区别？Gateway 的过滤器机制？
7. 配置中心：Spring Cloud Config vs Nacos Config？动态刷新原理？
8. 链路追踪：Sleuth + Zipkin / SkyWalking 的原理？

## 三、微服务实践问题

1. 微服务拆分的原则？怎么判断服务粒度是否合适？
2. 微服务间的通信方式？同步（HTTP/RPC）vs 异步（MQ）？
3. 微服务的数据一致性怎么保证？
4. 微服务的版本管理和灰度发布？
5. 微服务的日志聚合方案？ELK / EFK？
6. 微服务的容器化部署？Docker + K8s？

## 四、Dubbo（结合项目经验）

1. Dubbo 的整体架构？Provider、Consumer、Registry、Monitor？
2. Dubbo 的服务注册与发现流程？
3. Dubbo 支持的协议？dubbo 协议 vs http 协议？
4. Dubbo 的负载均衡策略？
5. Dubbo 的容错机制？Failover、Failfast、Failsafe 等？
6. Dubbo 的序列化方式？Hessian2 vs Protobuf？
7. Dubbo 和 Spring Cloud 的区别？如何选型？
