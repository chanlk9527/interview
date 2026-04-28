# 05 - Spring 框架

---

## 一、IoC 与组件设计

1. IoC 容器解决了什么工程问题？它和“new 一个对象”相比真正改变了什么？
2. 构造器注入、Setter 注入、字段注入如何取舍？为什么生产代码更推荐显式依赖？
3. Bean 的命名、条件装配、Profile、配置属性如何配合，支撑多环境和多租户差异？
4. `BeanFactory`、`ApplicationContext`、`Environment`、`ResourceLoader` 分别承担什么职责？
5. 复杂项目里如何避免 Bean 过多、依赖链过深、启动变慢和隐式耦合？

## 二、Bean 生命周期与扩展点

1. Bean 从定义读取到可用对象，中间有哪些关键阶段？哪些扩展点最常用？
2. `BeanPostProcessor`、`BeanFactoryPostProcessor`、`ImportBeanDefinitionRegistrar` 分别适合改什么？
3. 循环依赖为什么只在部分场景可被解决？构造器注入、代理对象、事务 Bean 会带来什么边界？
4. 单例 Bean 的线程安全如何保证？无状态设计、不可变配置、并发容器各适合什么场景？
5. 如何排查 Bean 没有注入、注入错实现、条件装配未生效、启动顺序异常这类问题？

## 三、AOP 与代理

1. Spring AOP 适合解决哪些横切关注点？哪些问题不应该靠 AOP 隐式处理？
2. JDK 动态代理、CGLIB、AspectJ、Byte Buddy 的差异是什么？Spring 通常如何选择代理方式？
3. `@Transactional`、`@Cacheable`、`@Async` 为什么会因为自调用、非 public 方法、代理绕过而失效？
4. 多个切面同时存在时，如何控制顺序、异常传播和性能影响？
5. 你如何设计一个审计日志、幂等或限流切面，让它可测试、可观测、可降级？

## 四、事务管理

1. Spring 声明式事务的本质是什么？事务边界到底包住了哪些代码？
2. 传播行为、隔离级别、只读事务、超时、回滚规则如何影响真实业务一致性？
3. 为什么“方法加了 `@Transactional`”仍然可能不回滚？异常类型、捕获方式、代理边界分别有什么影响？
4. 长事务会带来哪些数据库和应用层问题？如何缩短事务边界？
5. 本地事务、消息最终一致性、补偿事务、Outbox Pattern 分别适合什么场景？
6. 如何为事务问题补充日志、指标和链路追踪，避免线上只看到“数据不一致”？

## 五、Web MVC 与接口边界

1. Spring MVC 请求处理链路中，`DispatcherServlet`、HandlerMapping、HandlerAdapter、MessageConverter 各做什么？
2. 参数绑定、校验、类型转换、国际化错误消息如何设计，才能让接口边界清晰？
3. Filter、Interceptor、AOP、ControllerAdvice 分别适合放哪些逻辑？
4. 全局异常处理如何和错误码、HTTP 状态码、日志、告警、TraceId 配合？
5. 大请求体、文件上传、流式响应、SSE、异步请求在 Spring MVC 中有哪些资源风险？
6. CORS、鉴权、CSRF、请求重放在后端接口中如何分层处理？

## 六、Spring 设计思想与可维护性

1. Spring 中哪些设计模式真正影响日常开发？工厂、模板方法、观察者、代理分别体现在哪里？
2. `FactoryBean` 和普通 Bean 的差异是什么？什么场景值得用它而不是直接 `@Bean`？
3. Spring 事件机制适合做什么？为什么不应把它当成可靠消息队列？
4. 如何写一个可维护的 Starter 或公共组件？自动配置、条件装配、配置元数据、向后兼容如何设计？
5. 如何降低 Spring 项目的“魔法感”：命名规范、启动日志、测试切片、架构边界分别能做什么？
