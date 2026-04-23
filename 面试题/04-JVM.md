# 04 - JVM

---

## 一、内存区域

1. JVM 运行时数据区有哪些？各自的作用？
2. 堆和栈的区别？
3. 方法区（元空间）存什么？为什么 JDK 8 用元空间替代了永久代？
4. 直接内存（Direct Memory）是什么？和堆内存的区别？
5. 对象在内存中的布局？对象头包含什么？

## 二、对象创建与内存分配

1. 对象的创建过程？（类加载检查 → 分配内存 → 初始化零值 → 设置对象头 → init）
2. 内存分配方式？指针碰撞 vs 空闲列表？
3. TLAB（Thread Local Allocation Buffer）是什么？
4. 对象的访问定位方式？句柄 vs 直接指针？

## 三、垃圾回收

1. 如何判断对象是否可以被回收？引用计数法 vs 可达性分析？
2. Java 的四种引用类型？强引用、软引用、弱引用、虚引用？
3. 常见的垃圾回收算法？标记-清除、标记-复制、标记-整理？
4. 分代收集理论？为什么要分新生代和老年代？
5. Minor GC、Major GC、Full GC 的区别？什么时候触发？
6. 常见的垃圾收集器？各自的特点？
   - Serial / Serial Old
   - ParNew / Parallel Scavenge / Parallel Old
   - CMS（Concurrent Mark Sweep）
   - G1（Garbage First）
   - ZGC / Shenandoah
7. CMS 和 G1 的区别？各自的优缺点？
8. G1 的 Region 划分和回收过程？
9. ZGC 的核心特性？染色指针和读屏障？

## 四、类加载机制

1. 类加载的过程？加载 → 验证 → 准备 → 解析 → 初始化？
2. 双亲委派模型是什么？为什么要用双亲委派？
3. 如何打破双亲委派模型？有哪些实际案例？（SPI、Tomcat、OSGi）
4. 自定义类加载器的实现？

## 五、JVM 调优

1. 常用的 JVM 参数有哪些？（-Xms、-Xmx、-Xss、-XX:MetaspaceSize 等）
2. 如何排查 OOM（OutOfMemoryError）？常见的 OOM 场景？
3. 如何排查 CPU 飙高问题？（top → jstack → 分析线程栈）
4. 如何排查内存泄漏？（jmap → MAT/VisualVM 分析堆转储）
5. GC 日志怎么看？常用的 GC 日志参数？
6. JVM 调优的一般思路和步骤？
7. 你在项目中做过哪些 JVM 调优？效果如何？

## 六、线上问题排查

### 6.1 CPU 问题排查

1. 线上服务 CPU 突然飙到 100%，你的排查步骤是什么？请详细描述从发现到定位的完整流程。
2. 如何用 `top -Hp <pid>` 配合 `jstack` 定位具体是哪个线程导致 CPU 飙高？线程 ID 的十六进制转换怎么做？
3. CPU 飙高的常见原因有哪些？（死循环、频繁 GC、正则回溯、锁竞争等）
4. 如何区分是业务代码导致的 CPU 高还是 GC 导致的 CPU 高？
5. `jstack` 输出中线程状态 RUNNABLE、BLOCKED、WAITING、TIMED_WAITING 分别代表什么？排查时重点关注哪些？

### 6.2 内存问题排查

1. 线上服务频繁 Full GC，你怎么排查？从哪些指标入手？
2. 如何区分内存泄漏和内存溢出？排查思路有什么不同？
3. 堆内存 OOM（`java.lang.OutOfMemoryError: Java heap space`）的常见原因和排查方法？
4. 元空间 OOM（`java.lang.OutOfMemoryError: Metaspace`）的常见原因？什么场景下会出现？
5. 直接内存 OOM（`java.lang.OutOfMemoryError: Direct buffer memory`）怎么排查？Netty 场景下如何避免？
6. `jmap -dump` 导出堆转储文件后，如何用 MAT（Memory Analyzer Tool）分析？重点看哪些报告？
7. 线上不方便 `jmap -dump` 怎么办？（`-XX:+HeapDumpOnOutOfMemoryError` 参数、arthas 的 `heapdump` 命令）
8. 如何排查堆外内存泄漏？有哪些工具可以用？（NMT、pmap、gperftools）

### 6.3 线程问题排查

1. 如何排查线程死锁？`jstack` 能自动检测死锁吗？输出是什么样的？
2. 线程池满了导致请求超时，怎么排查？如何确定是线程池配置不合理还是下游服务慢？
3. 大量线程处于 WAITING 或 BLOCKED 状态，可能是什么原因？怎么定位？
4. 如何排查线程数持续增长的问题？（线程池未正确关闭、无限创建线程等）

### 6.4 GC 问题排查

1. 如何开启和分析 GC 日志？`-Xlog:gc*`（JDK 9+）和 `-XX:+PrintGCDetails`（JDK 8）的区别？
2. GC 日志中哪些指标最关键？（GC 暂停时间、GC 频率、回收前后堆大小变化）
3. Young GC 时间过长可能是什么原因？怎么优化？
4. CMS 的 Concurrent Mode Failure 是什么？怎么解决？
5. G1 的 Mixed GC 和 Full GC 分别在什么情况下触发？如何避免 G1 的 Full GC？
6. 如何用 GCViewer 或 GCEasy 等工具分析 GC 日志？

### 6.5 常用排查工具

1. JDK 自带的排查工具有哪些？各自的用途？（jps、jstat、jinfo、jmap、jstack、jcmd）
2. `jstat -gcutil <pid>` 输出的各列含义是什么？如何通过它判断 GC 是否正常？
3. Arthas 的常用命令有哪些？（dashboard、thread、trace、watch、stack、tt、profiler）
4. 如何用 Arthas 的 `trace` 命令定位方法耗时？和 `watch` 命令有什么区别？
5. 如何用 Arthas 在不重启服务的情况下动态修改日志级别或查看方法返回值？
6. async-profiler 是什么？如何用它生成火焰图来定位性能瓶颈？
7. 如何用 `jcmd` 替代 `jmap` 和 `jstack`？`jcmd` 有哪些优势？

### 6.6 实战场景题

1. 线上服务响应变慢但 CPU 和内存都正常，你怎么排查？（网络、IO、锁、线程池、下游依赖等）
2. 服务启动后运行一段时间就 OOM，但堆内存设置得很大，可能是什么原因？
3. 某个接口偶尔超时，大部分时候正常，你怎么定位？（GC STW、锁竞争、线程池排队等）
4. 发布新版本后 Full GC 频率明显增加，你怎么排查是哪段代码引起的？
5. 容器环境下 JVM 被 OOM Killer 杀掉，但堆内存没有超限，可能是什么原因？（堆外内存、线程栈、NMT 排查）
6. 线上出现 `StackOverflowError`，如何快速定位是哪个递归调用导致的？
7. 数据库连接池耗尽导致服务不可用，如何从 JVM 层面排查连接泄漏？

## 七、JIT 编译

1. 解释执行和编译执行的区别？
2. 热点代码检测？方法调用计数器和回边计数器？
3. C1 和 C2 编译器的区别？分层编译？
4. JIT 的常见优化手段？（方法内联、逃逸分析、标量替换等）
