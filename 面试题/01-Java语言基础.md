# 01 - Java 语言基础

---

## 一、数据类型与基础语法

1. Java 基本数据类型有哪些？各占多少字节？
2. 自动装箱与拆箱的原理？Integer 缓存池（-128~127）是怎么回事？
3. String、StringBuilder、StringBuffer 的区别？String 为什么是不可变的？
4. String 的 intern() 方法有什么作用？字符串常量池在哪里？
5. == 和 equals() 的区别？hashCode() 和 equals() 的关系？
6. final、finally、finalize 的区别？
7. 值传递和引用传递？Java 是值传递还是引用传递？

## 二、面向对象

1. 面向对象三大特性（封装、继承、多态）分别怎么理解？
2. 重载（Overload）和重写（Override）的区别？
3. 抽象类和接口的区别？什么时候用抽象类，什么时候用接口？
4. Java 为什么不支持多继承？接口的默认方法（default method）解决了什么问题？
5. 深拷贝和浅拷贝的区别？如何实现深拷贝？
6. Object 类有哪些常用方法？

## 三、异常处理

1. Java 异常体系结构？Throwable、Error、Exception 的继承关系？
2. Error 和 Exception 的区别？分别举几个常见的例子？
3. Checked Exception 和 Unchecked Exception 的区别？为什么有这两种设计？
4. RuntimeException 有哪些常见子类？NullPointerException、IllegalArgumentException、IndexOutOfBoundsException 等？
5. try-catch-finally 的执行顺序？finally 中的 return 会怎样？try 中 return 了 finally 还会执行吗？
6. try-with-resources 是什么？解决了什么问题？AutoCloseable 接口的作用？
7. 多个 catch 块的匹配顺序？为什么子类异常要放在前面？
8. throw 和 throws 的区别？
9. 自定义异常的最佳实践？什么时候该自定义异常？
10. 异常对性能的影响？为什么不建议用异常做流程控制？
11. 异常链（Exception Chaining）是什么？Cause 的作用？
12. Spring 中的全局异常处理方案？@ControllerAdvice + @ExceptionHandler？
13. 项目中你是怎么设计异常体系的？业务异常和系统异常怎么区分？

## 四、泛型

1. 什么是泛型？泛型的好处是什么？
2. 泛型擦除是什么？有什么影响？
3. 上界通配符（? extends T）和下界通配符（? super T）的区别？PECS 原则？
4. 泛型方法和泛型类的区别？

## 五、注解

1. 什么是注解？注解的本质是什么？（特殊的接口）
2. 注解的分类？源码级、编译级、运行时注解的区别？
3. 元注解有哪些？@Target、@Retention、@Documented、@Inherited、@Repeatable 各自的作用？
4. @Retention 的三种策略？SOURCE、CLASS、RUNTIME 的区别？什么时候用哪种？
5. 如何自定义一个注解？自定义注解的属性有什么限制？
6. 注解是怎么被解析的？编译期处理（APT）vs 运行时反射？
7. 注解处理器（Annotation Processor）的原理？Lombok 是怎么实现的？
8. Spring 中常用注解的原理？@Component、@Autowired、@Transactional 是怎么生效的？
9. 注解和 XML 配置的对比？各自的优缺点？
10. 如何实现一个自定义的 @Log 注解来做方法级别的日志记录？

## 六、反射

1. 什么是反射？反射能做什么？
2. 获取 Class 对象的几种方式？Class.forName()、.class、getClass() 的区别？
3. 通过反射创建对象的方式？Constructor.newInstance() vs Class.newInstance()？
4. 通过反射获取和调用方法？getMethod() vs getDeclaredMethod()？
5. 通过反射访问私有字段和方法？setAccessible(true) 的作用和安全性？
6. 反射的性能问题？比直接调用慢多少？如何优化？（缓存 Method 对象、MethodHandle、生成代理类）
7. 反射在框架中的应用？Spring IoC、MyBatis Mapper 代理、JUnit 测试？
8. 反射和泛型擦除的关系？如何通过反射获取泛型的实际类型？
9. 动态代理的实现原理？JDK 动态代理 vs CGLIB 的区别？
10. MethodHandle 和反射的区别？Java 7 引入 MethodHandle 的意义？

## 七、SPI 机制

1. 什么是 SPI（Service Provider Interface）？和 API 有什么区别？
2. Java SPI 的工作原理？ServiceLoader 的加载流程？
3. META-INF/services 目录下的配置文件格式？
4. Java SPI 的优缺点？（无法按需加载、不支持排序和优先级）
5. Java SPI 的经典应用？JDBC Driver 的自动加载、SLF4J 日志门面？
6. Dubbo SPI 和 Java SPI 的区别？Dubbo SPI 做了哪些增强？（按需加载、支持 AOP/IoC、@SPI 注解）
7. Spring SPI 机制？spring.factories 的作用？和 Java SPI 的区别？
8. Spring Boot 自动配置的底层就是 SPI 思想，具体怎么实现的？
9. 如何自己实现一个简单的 SPI 机制？
10. SPI 在你项目中有用到吗？（支付渠道扩展、设备协议适配等场景）

## 八、Java 8+ 新特性

1. Lambda 表达式的本质是什么？函数式接口？
2. Stream API 的常用操作？中间操作和终端操作的区别？
3. Optional 类的作用和使用方式？
4. CompletableFuture 的使用场景和常用方法？
5. Java 8 的日期时间 API（LocalDate、LocalDateTime）？
6. Java 9-17 有哪些值得关注的新特性？（模块化、Records、Sealed Classes、Pattern Matching 等）

## 九、IO 与 NIO

> 已独立成篇，详见 [01A-Java-IO.md](./01A-Java-IO.md)
