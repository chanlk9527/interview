# 05 - Spring 框架

---

## 一、IoC 与依赖注入

1. 什么是 IoC（控制反转）？什么是 DI（依赖注入）？
2. Spring IoC 容器的初始化流程？
3. BeanFactory 和 ApplicationContext 的区别？
4. 依赖注入的方式？构造器注入 vs Setter 注入 vs 字段注入？推荐哪种？
5. @Autowired 和 @Resource 的区别？
6. Spring 如何解决循环依赖？三级缓存机制？

## 二、Bean 生命周期与作用域

1. Spring Bean 的完整生命周期？（实例化 → 属性填充 → 初始化 → 使用 → 销毁）
2. BeanPostProcessor 和 BeanFactoryPostProcessor 的区别？
3. @PostConstruct、InitializingBean、init-method 的执行顺序？
4. Bean 的作用域有哪些？singleton、prototype、request、session？
5. 单例 Bean 的线程安全问题？

## 三、AOP

1. 什么是 AOP？核心概念（切面、切点、通知、连接点）？
2. Spring AOP 的实现原理？JDK 动态代理 vs CGLIB 代理？
3. Spring AOP 和 AspectJ 的区别？
4. @Aspect 注解的使用？五种通知类型？
5. AOP 在项目中的实际应用场景？（日志、权限、事务、缓存）
6. AOP 失效的常见场景？（自调用、private 方法、final 类）

## 四、事务管理

1. Spring 事务的实现原理？
2. @Transactional 注解的属性？propagation、isolation、rollbackFor？
3. Spring 事务的传播行为有哪些？各自的含义？
4. Spring 事务失效的常见场景？（自调用、异常被吞、非 public 方法等）
5. 编程式事务和声明式事务的区别？
6. 分布式事务怎么处理？（详见分布式章节）

## 五、Spring MVC

1. Spring MVC 的请求处理流程？DispatcherServlet 的工作原理？
2. @Controller 和 @RestController 的区别？
3. 拦截器（Interceptor）和过滤器（Filter）的区别？
4. @RequestBody 和 @ResponseBody 的原理？消息转换器？
5. 全局异常处理？@ControllerAdvice + @ExceptionHandler？
6. Spring MVC 如何处理跨域（CORS）？

## 六、Spring 设计思想

1. Spring 中用到了哪些设计模式？
2. FactoryBean 和 BeanFactory 的区别？
3. Spring 的事件机制？ApplicationEvent 和 ApplicationListener？
4. Spring SPI 机制？spring.factories 的作用？
