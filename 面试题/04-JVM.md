# 04 - JVM

---

## 一、运行时内存与对象模型

1. JVM 运行时内存区域如何服务于线程执行、对象分配和类元数据管理？
2. 堆、线程栈、元空间、直接内存分别受哪些参数和系统资源限制？OOM 信息如何对应到具体区域？
3. 对象头、字段、对齐填充会如何影响对象大小？为什么高并发缓存要关心对象布局？
4. TLAB、逃逸分析、标量替换如何影响对象分配成本？哪些代码写法会阻碍这些优化？
5. 直接内存和堆内存如何取舍？在 NIO、Netty、序列化缓冲中如何排查直接内存泄漏？

### 基础补充题

1. JVM 运行时数据区主要包括哪些部分？
2. 程序计数器为什么是线程私有的？
3. 方法区和堆分别主要存放什么数据？
4. Java 对象在内存中通常由哪几部分组成？

## 二、对象生命周期与引用

1. 从 `new` 到构造完成，对象创建经历哪些关键步骤？哪些步骤可能触发类加载或内存分配失败？
2. 可达性分析如何判断对象存活？GC Roots 通常来自哪些地方？
3. 强引用、软引用、弱引用、虚引用分别适合什么场景？为什么软引用不适合做稳定缓存？
4. 对象年龄、晋升、动态年龄判断会如何影响老年代压力？
5. ThreadLocal、静态集合、监听器、缓存、连接池常见的引用链泄漏如何定位？

### 基础补充题

1. 对象什么时候会变成不可达？
2. GC Roots 常见类型有哪些？
3. 强引用和弱引用最基础的区别是什么？
4. `finalize()` 为什么不建议作为资源释放手段？

## 三、垃圾回收与收集器选型

1. 标记-复制、标记-清除、标记-整理解决的问题和代价分别是什么？
2. 分代收集在现代 JVM 中仍然有哪些价值？为什么有些低延迟收集器会弱化分代假设？
3. Young GC、Mixed GC、Full GC 的触发原因和业务影响分别是什么？
4. G1 的 Region、Remembered Set、Mixed GC 如何工作？它适合怎样的服务画像？
5. ZGC / Shenandoah 这类低延迟 GC 解决了什么问题？读屏障、染色指针/转发指针的代价是什么？
6. 容器化部署下，堆、非堆、线程栈、直接内存如何共同影响 Pod 内存水位？
7. 选择 GC 时，你会如何根据吞吐、延迟、堆大小、对象生命周期、JDK 版本和运维能力做权衡？

### 基础补充题

1. 什么是 Minor GC、Major GC 和 Full GC？
2. 新生代和老年代划分的基本依据是什么？
3. 常见垃圾收集算法有哪些？
4. 吞吐量优先和低延迟优先的 GC 目标有什么区别？

## 四、类加载、模块与运行时扩展

1. 类加载的加载、链接、初始化分别做什么？哪些代码会触发类初始化？
2. 双亲委派模型解决了什么安全和一致性问题？在 SPI、应用服务器、插件化中为什么会出现定制类加载？
3. 自定义类加载器如何设计隔离边界？热部署、脚本引擎、插件系统会遇到哪些类冲突问题？
4. JPMS 模块化、反射访问限制、`--add-opens` 对框架和升级 JDK 有什么影响？
5. 如何排查 `ClassNotFoundException`、`NoClassDefFoundError`、`LinkageError`、类版本冲突？

### 基础补充题

1. 类加载过程通常分为哪几个阶段？
2. 双亲委派模型的基本流程是什么？
3. `ClassNotFoundException` 和 `NoClassDefFoundError` 最基础的区别是什么？
4. 什么情况下会触发类初始化？

## 五、JVM 调优与可观测性

1. JVM 调优前为什么要先定义目标？吞吐、延迟、成本、稳定性之间如何取舍？
2. 你会如何为线上服务建立 JVM 基线：启动参数、GC 日志、JFR、线程池指标、内存水位、容器限制？
3. 如何排查 OOM？堆、元空间、直接内存、线程数过多分别应该看哪些证据？
4. 如何排查 CPU 飙高、接口偶发超时、频繁 GC、内存缓慢上涨这几类线上问题？
5. GC 日志、JFR、async-profiler、Arthas、MAT 分别适合回答什么问题？
6. 升级 JDK 或切换 GC 时如何设计压测、灰度、指标对比和回滚方案？
7. 你在项目中做过哪些 JVM 相关优化？如何证明优化确实有效？

### 基础补充题

1. `-Xms` 和 `-Xmx` 分别表示什么？
2. 为什么生产环境通常会开启 GC 日志？
3. 堆 dump 文件通常用来分析什么问题？
4. 线程 dump 文件通常用来分析什么问题？

## 六、线上问题排查

### 6.1 CPU 问题排查

**1. 线上服务 CPU 突然飙到 100%，你的排查步骤是什么？请详细描述从发现到定位的完整流程。**

完整排查流程分五步：

1. **确认进程**：用 `top` 命令找到 CPU 占用最高的 Java 进程，记下 PID
2. **定位线程**：用 `top -Hp <pid>` 查看该进程内各线程的 CPU 占用，找到最耗 CPU 的线程 ID（TID）
3. **转换线程 ID**：将十进制的 TID 转为十六进制，`printf "%x\n" <tid>`，比如线程 ID 12345 转为 0x3039
4. **导出线程栈**：用 `jstack <pid> > thread_dump.txt` 导出线程快照
5. **匹配分析**：在 thread dump 中搜索十六进制线程 ID（nid=0x3039），查看该线程的堆栈信息，定位到具体代码行

实际操作中建议多次 jstack（间隔几秒取3次），对比同一线程的堆栈是否一直停在同一位置，如果是，大概率就是问题代码。

也可以用 Arthas 一步到位：`thread -n 3` 直接显示 CPU 占用最高的3个线程及其堆栈。

**2. 如何用 `top -Hp <pid>` 配合 `jstack` 定位具体是哪个线程导致 CPU 飙高？线程 ID 的十六进制转换怎么做？**

```bash
# 第一步：找到 Java 进程 PID
top -c    # 找到 CPU 最高的 java 进程，假设 PID=20558

# 第二步：查看进程内线程级 CPU 占用
top -Hp 20558    # 找到 CPU 最高的线程，假设 TID=20607

# 第三步：十进制转十六进制
printf "%x\n" 20607    # 输出 507f

# 第四步：导出线程栈
jstack 20558 > thread_dump.txt

# 第五步：搜索匹配
grep -A 30 "nid=0x507f" thread_dump.txt
```

jstack 输出中每个线程的 `nid` 就是操作系统层面的线程 ID（十六进制），与 `top -Hp` 看到的 TID 对应。

**3. CPU 飙高的常见原因有哪些？**

| 原因 | 特征 | 排查方式 |
|---|---|---|
| **死循环/无限循环** | 单线程 CPU 100%，堆栈固定在某段代码 | jstack 多次采样，堆栈不变 |
| **频繁 Full GC** | GC 线程占满 CPU，业务线程被 STW | `jstat -gcutil` 看 FGC 频率和耗时 |
| **正则表达式回溯** | 线程卡在 `java.util.regex` 相关方法 | jstack 看到 `Pattern.match` 相关堆栈 |
| **锁竞争激烈** | 大量线程 BLOCKED，少数线程 RUNNABLE | jstack 看 BLOCKED 线程等待的锁 |
| **序列化/反序列化** | JSON/XML 解析大对象消耗 CPU | trace 定位方法耗时 |
| **加密/压缩运算** | 密集计算型操作 | 火焰图定位热点方法 |
| **线程数过多** | 上下文切换频繁，`vmstat` 的 cs 列很高 | `jstack` 统计线程数，检查线程池配置 |

**4. 如何区分是业务代码导致的 CPU 高还是 GC 导致的 CPU 高？**

两种方法：

**方法一：看 GC 指标**
```bash
jstat -gcutil <pid> 1000    # 每秒输出一次 GC 统计
```
如果 FGC（Full GC 次数）在快速增长，FGCT（Full GC 总耗时）占比很高，说明是 GC 导致的。

**方法二：看线程名**
用 `top -Hp` 找到 CPU 最高的线程，再用 jstack 匹配：
- 如果线程名是 `GC task thread#0`、`VM Thread` 等 → GC 导致
- 如果线程名是业务线程（如 `http-nio-8080-exec-1`）→ 业务代码导致

**方法三：用 Arthas**
```bash
dashboard    # 直接看 GC 信息和线程 CPU 占用的综合面板
```

**5. `jstack` 输出中线程状态分别代表什么？排查时重点关注哪些？**

| 状态 | 含义 | 说明 |
|---|---|---|
| **RUNNABLE** | 正在运行或等待 CPU 时间片 | 包括在执行 Java 代码和执行 native 方法（如 IO） |
| **BLOCKED** | 等待获取 synchronized 锁 | 线程想进入 synchronized 块但锁被其他线程持有 |
| **WAITING** | 无限期等待 | 调用了 `Object.wait()`、`Thread.join()`、`LockSupport.park()` |
| **TIMED_WAITING** | 有超时的等待 | 调用了 `Thread.sleep()`、`Object.wait(timeout)`、`LockSupport.parkNanos()` |

排查重点：
- **CPU 高**：重点看 **RUNNABLE** 状态的线程，它们在消耗 CPU
- **响应慢/卡死**：重点看 **BLOCKED** 和 **WAITING** 状态的线程，分析它们在等什么锁或什么资源
- **死锁**：jstack 会在末尾自动输出 `Found one Java-level deadlock`，直接搜索即可
- 如果大量线程处于 BLOCKED 且都在等同一把锁，说明存在锁竞争热点

#### 基础补充题

1. CPU 使用率和系统负载（load average）有什么区别？
2. `top`、`ps`、`jstack` 分别能提供哪些信息？
3. 为什么定位 CPU 高时通常要多次采样线程栈？
4. 业务线程长时间 RUNNABLE 可能说明什么？

### 6.2 内存问题排查

**1. 线上服务频繁 Full GC，你怎么排查？从哪些指标入手？**

排查步骤：

1. **确认 GC 频率和耗时**：
   ```bash
   jstat -gcutil <pid> 1000    # 每秒输出一次
   ```
   重点看：FGC（Full GC 次数）是否在快速增长，FGCT（Full GC 总耗时）是否很高，O（老年代使用率）是否持续接近 100%。

2. **分析 GC 日志**：优先使用统一日志参数 `-Xlog:gc*:file=gc.log:time,uptime,level,tags`，看每次 GC 前后堆内存、老年代/Region、暂停时间和触发原因。如果回收后老年代占用依然很高，说明有大量不可回收对象（可能是内存泄漏）。

3. **导出堆转储分析**：
   ```bash
   jmap -dump:live,format=b,file=heap.hprof <pid>
   ```
   用 MAT 分析哪些对象占用最多内存，找到 GC Root 引用链。

4. **常见原因**：
   - 内存泄漏：对象被长生命周期引用持有，无法回收（如 static 集合、缓存未设上限、监听器未注销）
   - 大对象直接进入老年代：超过 `-XX:PretenureSizeThreshold` 的对象直接分配到老年代
   - 新生代太小：对象过早晋升到老年代，`-XX:MaxTenuringThreshold` 设置不合理
   - Metaspace 触发 Full GC：元空间不足也会触发 Full GC

**2. 如何区分内存泄漏和内存溢出？排查思路有什么不同？**

| 对比项 | 内存泄漏（Memory Leak） | 内存溢出（OOM） |
|---|---|---|
| **定义** | 对象不再使用但无法被 GC 回收 | 申请内存时堆空间不足 |
| **表现** | 内存使用缓慢增长，Full GC 后老年代占用越来越高 | 直接抛出 OutOfMemoryError |
| **关系** | 泄漏积累到一定程度会导致溢出 | 溢出不一定是泄漏导致的（也可能是堆设置太小或瞬时大对象） |

排查思路差异：
- **内存泄漏**：多次 dump 堆快照对比，看哪些对象在持续增长；用 MAT 的 Leak Suspects 报告找到泄漏的引用链
- **内存溢出**：先看是哪种 OOM（堆、元空间、直接内存、线程栈），再针对性排查；如果是堆 OOM，先确认堆大小是否合理，再看是否有泄漏

**3. 堆内存 OOM（`java.lang.OutOfMemoryError: Java heap space`）的常见原因和排查方法？**

常见原因：
- **内存泄漏**：最常见，对象被意外持有无法回收（static Map 不断 put、ThreadLocal 未 remove、监听器未注销）
- **堆设置过小**：`-Xmx` 设置不合理，业务增长后内存不够
- **大对象/大查询**：一次性从数据库查出大量数据加载到内存，或者读取大文件到内存
- **缓存无上限**：本地缓存（如 HashMap 做缓存）没有设置最大容量和淘汰策略

排查方法：
```bash
# 1. 确保开启了 OOM 时自动 dump
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof

# 2. 用 MAT 打开 hprof 文件
# 3. 查看 Dominator Tree（支配树），找到占用内存最大的对象
# 4. 查看 Leak Suspects 报告，MAT 会自动分析可疑的泄漏点
# 5. 通过 GC Root 引用链找到是谁持有了这些对象
```

**4. 元空间 OOM（`java.lang.OutOfMemoryError: Metaspace`）的常见原因？什么场景下会出现？**

元空间存储类的元数据（类信息、方法信息、常量池等），OOM 说明加载的类太多。

常见场景：
- **动态生成大量类**：CGLib、Javassist 等字节码增强框架动态创建类，如果没有控制好会不断生成新类
- **反射滥用**：`Method.invoke()` 在一定次数后会生成 DelegatingClassLoader 加载动态类
- **JSP 热部署**：每次修改 JSP 都会生成新的类
- **OSGi/自定义类加载器**：类加载器未正确卸载，导致类无法被回收
- **Spring AOP 代理过多**：大量 Bean 被代理，每个代理类都是动态生成的

排查方法：
```bash
# 查看加载的类数量
jstat -class <pid>

# 用 Arthas 查看类加载器统计
classloader -t

# 增大元空间并设置上限
-XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m
```

**5. 直接内存 OOM（`java.lang.OutOfMemoryError: Direct buffer memory`）怎么排查？Netty 场景下如何避免？**

直接内存是通过 `ByteBuffer.allocateDirect()` 或 Netty 的 `UnpooledByteBufAllocator` 分配的堆外内存，不受 `-Xmx` 控制，受 `-XX:MaxDirectMemorySize` 限制。

排查方法：
```bash
# 1. 开启 NMT（Native Memory Tracking）
-XX:NativeMemoryTracking=detail

# 2. 查看内存分布
jcmd <pid> VM.native_memory detail

# 3. 监控直接内存使用
# 通过 JMX 的 java.nio:type=BufferPool,name=direct 查看
```

Netty 场景下避免的方法：
- **使用池化分配器**：`PooledByteBufAllocator.DEFAULT`，Netty 4.1+ 默认就是池化的
- **及时释放 ByteBuf**：确保每个 ByteBuf 都被正确 `release()`，引用计数归零
- **Handler 中注意传递**：如果 Handler 中读取了 ByteBuf 但不往下传，必须手动释放
- **开启泄漏检测**：`-Dio.netty.leakDetection.level=PARANOID`（开发/测试环境用）
- **设置合理的 MaxDirectMemorySize**：`-XX:MaxDirectMemorySize=512m`

**6. `jmap -dump` 导出堆转储文件后，如何用 MAT 分析？重点看哪些报告？**

```bash
# 导出堆转储
jmap -dump:live,format=b,file=heap.hprof <pid>
# 注意：live 参数会先触发一次 Full GC，只 dump 存活对象
```

MAT 分析重点看的报告：

1. **Leak Suspects Report（泄漏嫌疑报告）**：MAT 自动分析并给出可能的内存泄漏点，包括占用内存最大的对象和引用链，这是最先看的报告

2. **Dominator Tree（支配树）**：按对象的 retained size（对象本身 + 它独占引用的所有对象大小）排序，快速找到"大户"

3. **Histogram（直方图）**：按类统计对象数量和大小，看哪个类的实例数异常多

4. **Thread Overview（线程概览）**：查看每个线程持有的对象，定位是哪个线程的操作导致内存问题

5. **Path to GC Roots**：对可疑对象右键查看到 GC Root 的引用链，找到是谁持有了它导致无法回收

**7. 线上不方便 `jmap -dump` 怎么办？**

`jmap -dump` 的问题：会触发 Full GC（使用 `live` 参数时），对线上服务有影响；大堆 dump 时间很长，期间服务可能无响应。

替代方案：

1. **提前配置 JVM 参数**（推荐，最佳实践）：
   ```bash
   -XX:+HeapDumpOnOutOfMemoryError
   -XX:HeapDumpPath=/data/logs/heap.hprof
   ```
   OOM 时自动 dump，不需要人工介入。

2. **使用 Arthas**：
   ```bash
   heapdump /tmp/heap.hprof          # dump 所有对象
   heapdump --live /tmp/heap.hprof   # 只 dump 存活对象（会触发 Full GC）
   ```
   好处是不需要知道 PID，Arthas 会自动 attach。

3. **使用 jcmd**（推荐替代 jmap）：
   ```bash
   jcmd <pid> GC.heap_dump /tmp/heap.hprof
   ```
   比 jmap 更安全，是 Oracle 官方推荐的方式。

4. **核心转储（Core Dump）**：通过 `gcore <pid>` 生成核心转储，再用 `jmap` 从 core 文件中提取堆信息，不影响运行中的进程。

**8. 如何排查堆外内存泄漏？有哪些工具可以用？**

堆外内存包括：直接内存（DirectByteBuffer）、JNI 分配的内存、线程栈、元空间、Code Cache 等。堆外泄漏的特征是 RSS（进程实际物理内存）持续增长，但堆内存使用正常。

排查工具和方法：

1. **NMT（Native Memory Tracking）**：
   ```bash
   # 启动时开启
   -XX:NativeMemoryTracking=detail
   
   # 设置基线
   jcmd <pid> VM.native_memory baseline
   
   # 一段时间后对比
   jcmd <pid> VM.native_memory detail.diff
   ```
   可以看到 JVM 各区域（Heap、Thread、Class、Code、Internal 等）的内存变化。

2. **pmap**：
   ```bash
   pmap -x <pid>    # 查看进程的内存映射
   ```
   对比多次 pmap 输出，看哪些内存段在增长。

3. **gperftools（Google Performance Tools）**：
   ```bash
   # 通过 LD_PRELOAD 注入 tcmalloc
   LD_PRELOAD=/usr/lib/libtcmalloc.so java -jar app.jar
   ```
   可以追踪 native 内存分配的调用栈，生成内存分配的火焰图。

4. **Arthas + memory 命令**：查看 JVM 各区域内存使用情况。

常见堆外泄漏原因：
- DirectByteBuffer 未释放（Netty ByteBuf 引用计数未归零）
- JNI 代码中 malloc 后未 free
- 线程数持续增长（每个线程默认占 1MB 栈空间）
- 使用了 Unsafe.allocateMemory 未释放

#### 基础补充题

1. 堆内存、元空间、直接内存分别可能抛出哪些 OOM？
2. 内存泄漏和内存占用高是一回事吗？
3. 为什么分析 OOM 前要先看具体错误信息？
4. Heap Dump 和 GC 日志分别适合回答什么问题？

### 6.3 线程问题排查

**1. 如何排查线程死锁？`jstack` 能自动检测死锁吗？输出是什么样的？**

`jstack` 可以自动检测死锁。执行 `jstack <pid>` 后，如果存在死锁，输出末尾会有：

```
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f8b2c003f08 (object 0x000000076ab75e10, a java.lang.Object),
  which is held by "Thread-0"
"Thread-0":
  waiting to lock monitor 0x00007f8b2c006358 (object 0x000000076ab75e20, a java.lang.Object),
  which is held by "Thread-1"

Java stack information for the threads listed above:
===================================================
"Thread-1":
    at com.example.DeadLockDemo.methodB(DeadLockDemo.java:25)
    - waiting to lock <0x000000076ab75e10>
    - locked <0x000000076ab75e20>
"Thread-0":
    at com.example.DeadLockDemo.methodA(DeadLockDemo.java:15)
    - waiting to lock <0x000000076ab75e20>
    - locked <0x000000076ab75e10>
```

排查步骤：
1. `jstack <pid>` 或 `jcmd <pid> Thread.print` 导出线程栈
2. 搜索 `deadlock` 关键字
3. 如果工具没有直接检测到（比如某些 ReentrantLock 死锁），手动搜索 BLOCKED/WAITING 状态的线程，分析它们等待的锁和持有的锁
4. Arthas 的 `thread -b` 命令可以直接找到阻塞其他线程的"罪魁祸首"线程

**2. 线程池满了导致请求超时，怎么排查？如何确定是线程池配置不合理还是下游服务慢？**

排查步骤：

1. **确认线程池状态**：
   - 用 Arthas 的 `thread` 命令查看线程数和状态
   - 如果有暴露线程池指标（如 Spring Boot Actuator），查看 `activeCount`、`queueSize`、`poolSize`

2. **区分原因**：
   - **线程池配置不合理**：线程都在 RUNNABLE 状态忙碌执行，队列满了 → 需要增大线程池或队列
   - **下游服务慢**：线程大多在 TIMED_WAITING 或 WAITING 状态，堆栈显示在等待 HTTP 响应、数据库查询等 → 问题在下游，需要加超时控制或优化下游

3. **用 Arthas trace 定位**：
   ```bash
   trace com.example.XxxService xxxMethod    # 看方法内部各步骤耗时
   ```

4. **检查线程池配置是否合理**：
   - IO 密集型：线程数 = CPU 核数 × 2（或更多）
   - CPU 密集型：线程数 = CPU 核数 + 1
   - 队列不要用无界队列（`LinkedBlockingQueue` 不设容量），否则永远不会触发拒绝策略，内存会被撑爆

**3. 大量线程处于 WAITING 或 BLOCKED 状态，可能是什么原因？怎么定位？**

**BLOCKED 状态**（等待 synchronized 锁）：
- 某个 synchronized 方法/块执行时间过长，其他线程排队等待
- 定位：jstack 中搜索 `BLOCKED`，查看 `waiting to lock <地址>`，再搜索 `locked <同一地址>` 找到持有锁的线程

**WAITING 状态**（无限期等待）：
- `Object.wait()`：等待 notify，可能是生产者-消费者模型中生产者停了
- `LockSupport.park()`：等待 ReentrantLock 或 Condition
- `Thread.join()`：等待另一个线程结束
- 线程池中的空闲线程等待任务（`LinkedBlockingQueue.take()`）→ 这是正常的

定位方法：
1. jstack 导出线程栈，统计各状态线程数量
2. 对 BLOCKED/WAITING 线程，查看堆栈中的等待对象
3. 找到持有锁的线程，分析它为什么长时间不释放

**4. 如何排查线程数持续增长的问题？**

```bash
# 监控线程数变化
jstat -class <pid>
# 或者
jcmd <pid> Thread.print | grep "java.lang.Thread.State" | wc -l

# 用 Arthas 实时监控
dashboard    # 看 Thread 面板的线程数变化
thread       # 列出所有线程，按名称分组统计
```

常见原因：
- **线程池未正确关闭**：每次请求创建新的线程池但没有 shutdown，线程池中的核心线程不会自动销毁
- **无限创建线程**：直接 `new Thread().start()` 而不是用线程池
- **定时任务重复创建**：每次调用都 `new ScheduledExecutorService()` 而不是复用
- **第三方库的线程泄漏**：某些 SDK 内部创建线程但没有提供关闭方法

排查方法：
1. 用 jstack 多次采样，对比新增的线程名称模式
2. 根据线程名找到创建线程的代码（如 `pool-123-thread-1` 说明创建了 123 个线程池）
3. 代码中搜索 `new Thread`、`Executors.new`、`new ThreadPoolExecutor` 等关键字

#### 基础补充题

1. 线程的 RUNNABLE、BLOCKED、WAITING、TIMED_WAITING 分别是什么状态？
2. 为什么线程池队列堆积会导致接口超时？
3. 死锁产生的四个必要条件是什么？
4. 为什么线程数不是越多越好？

### 6.4 GC 问题排查

**1. 如何开启和分析 GC 日志？现代 JDK 的统一日志参数怎么写？**

**统一日志框架（JEP 158）**：
```bash
-Xlog:gc*:file=/data/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=20M
```

关注点：
- 暂停时间、GC 原因、回收前后堆大小、晋升/复制量
- 是否出现 Full GC、Evacuation Failure、Humongous Allocation 等异常信号
- 日志级别和 tag 是否足够：常见 tag 包括 `gc`、`gc+heap`、`gc+age`、`gc+ergo`
- 如果仍维护旧运行环境，需要知道历史参数只作为兼容知识，不应成为新项目默认方案

**2. GC 日志中哪些指标最关键？**

| 指标 | 含义 | 关注点 |
|---|---|---|
| **GC 暂停时间（Pause Time）** | 单次 GC 的 STW 时间 | Young GC 通常 < 50ms，Full GC < 1s 为宜 |
| **GC 频率** | 单位时间内 GC 次数 | Young GC 几秒一次正常，Full GC 不应频繁出现 |
| **回收前后堆大小** | GC 前后各区域的内存变化 | 如果 Full GC 后老年代占用依然很高，可能有泄漏 |
| **晋升大小（Promotion）** | 从新生代晋升到老年代的数据量 | 晋升过多说明新生代太小或对象存活时间长 |
| **GC 原因（GC Cause）** | 触发 GC 的原因 | Allocation Failure（正常）、Metadata GC Threshold、Ergonomics 等 |

一个简单的判断标准：
- GC 总耗时占应用运行时间的比例 < 5% 是健康的
- 如果 > 10% 说明 GC 压力很大，需要调优

**3. Young GC 时间过长可能是什么原因？怎么优化？**

正常 Young GC 应该在 10-50ms 以内，如果超过 100ms 就需要关注。

可能原因：
- **新生代太大**：Eden 区过大，一次 Young GC 需要扫描和复制的对象太多 → 适当减小新生代
- **存活对象太多**：大量对象在 Young GC 时仍然存活，复制到 Survivor 区的开销大 → 检查是否有短生命周期的大对象
- **引用处理耗时**：大量软引用、弱引用、Finalizer 需要处理 → 减少不必要的特殊引用
- **卡表（Card Table）扫描**：老年代引用新生代的对象多，扫描 Remembered Set 耗时 → G1 中可以通过调整 Region 大小优化
- **Survivor 区太小**：对象在 Survivor 区放不下，直接晋升老年代，增加老年代压力

优化方向：
```bash
# 调整新生代大小
-Xmn256m    # 或者用比例 -XX:NewRatio=2

# 调整 Survivor 区比例
-XX:SurvivorRatio=8    # Eden:S0:S1 = 8:1:1

# 调整晋升年龄
-XX:MaxTenuringThreshold=15
```

**4. G1 或低延迟 GC 下出现长暂停，你会优先排查哪些原因？**

长暂停不一定等于“GC 参数不对”，要先确认暂停类型和业务时间点是否重合。

常见原因：
- **对象分配速率过高**：Young GC 频率升高，复制和晋升压力变大
- **存活对象过多**：一次回收要扫描/复制大量存活对象，暂停时间上升
- **Humongous Object**：大对象直接占用连续 Region，容易触发额外回收或 Full GC
- **引用处理过重**：软引用、弱引用、Finalizer/Cleaner、类卸载耗时异常
- **容器内存压力**：堆外内存、线程栈、JIT Code Cache、Native 内存挤压可用空间
- **日志或诊断误判**：业务超时可能由锁竞争、线程池排队、下游抖动造成，需要和 GC 日志、JFR、链路追踪对齐时间线

排查思路：
```bash
# 查看 GC 原因、暂停阶段和堆变化
-Xlog:gc*,safepoint:file=/data/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=20M

# 结合 JFR 观察分配热点、锁竞争、线程阻塞和文件/网络 IO
jcmd <pid> JFR.start name=profile settings=profile duration=120s filename=/tmp/app.jfr
```

**5. G1 的 Mixed GC 和 Full GC 分别在什么情况下触发？如何避免 G1 的 Full GC？**

**Mixed GC 触发条件**：
- 当老年代占用达到 `-XX:InitiatingHeapOccupancyPercent`（默认 45%）时，G1 启动并发标记
- 并发标记完成后，G1 会进行 Mixed GC，同时回收年轻代和部分老年代 Region（选择垃圾最多的 Region 优先回收，这就是 Garbage First 的含义）

**Full GC 触发条件**（G1 的 Full GC 在较新 JDK 中已有多线程优化，但仍然是需要重点避免的异常信号）：
- **Evacuation Failure**：复制存活对象时没有足够的空闲 Region → 最常见的原因
- 并发标记没来得及完成，老年代就满了
- 大对象（Humongous Object）分配失败，没有足够的连续 Region

避免 G1 Full GC 的方法：
```bash
# 1. 降低并发标记触发阈值
-XX:InitiatingHeapOccupancyPercent=35

# 2. 增加并发标记线程数
-XX:ConcGCThreads=4

# 3. 增大堆内存，给 G1 更多缓冲空间
-Xmx4g

# 4. 避免大对象：G1 中超过 Region 大小一半的对象是 Humongous Object
# 调整 Region 大小（1MB~32MB，必须是2的幂）
-XX:G1HeapRegionSize=16m

# 5. 调整 Mixed GC 的回收力度
-XX:G1MixedGCCountTarget=8          # 一次并发标记后最多做8次 Mixed GC
-XX:G1OldCSetRegionThresholdPercent=10  # 每次 Mixed GC 最多回收10%的老年代 Region
```

**6. 如何用 GCViewer 或 GCEasy 等工具分析 GC 日志？**

**GCEasy（https://gceasy.io）**：在线工具，上传 GC 日志即可
- 自动生成 GC 暂停时间分布图、堆内存使用趋势图
- 给出调优建议（如是否需要增大堆、是否有内存泄漏迹象）
- 支持多种 GC 收集器的日志格式
- 适合快速分析，不需要安装

**GCViewer**：开源桌面工具
```bash
java -jar gcviewer.jar gc.log
```
- 可视化展示 GC 暂停时间、吞吐量、堆内存变化
- 支持对比多个 GC 日志（调优前后对比）

分析时重点关注：
1. **吞吐量**：应用运行时间 / 总时间，目标 > 95%
2. **最大暂停时间**：是否有异常长的 GC 暂停
3. **Full GC 频率**：正常情况下不应频繁出现
4. **堆内存趋势**：如果 Full GC 后堆占用逐次升高，说明有内存泄漏
5. **晋升速率**：对象从新生代到老年代的速率是否过快

#### 基础补充题

1. Young GC 频繁通常说明什么？
2. Full GC 频繁通常需要优先检查哪些方向？
3. GC 暂停时间过长会对接口响应产生什么影响？
4. 为什么调整 GC 参数前要先看 GC 日志和业务指标？

### 6.5 常用排查工具

**1. JDK 自带的排查工具有哪些？各自的用途？**

| 工具 | 用途 | 常用命令 |
|---|---|---|
| **jps** | 列出 Java 进程 | `jps -lv`（显示主类全名和 JVM 参数） |
| **jstat** | 监控 GC、类加载、JIT 编译等统计信息 | `jstat -gcutil <pid> 1000`（每秒输出 GC 统计） |
| **jinfo** | 查看/修改 JVM 参数 | `jinfo -flags <pid>`（查看所有 JVM 参数） |
| **jmap** | 堆内存相关：dump 堆、查看堆统计 | `jmap -heap <pid>`、`jmap -dump:format=b,file=heap.hprof <pid>` |
| **jstack** | 导出线程栈快照 | `jstack <pid>`、`jstack -l <pid>`（包含锁信息） |
| **jcmd** | 统一诊断入口，可查看线程、堆、JFR、NMT、VM 参数等 | `jcmd <pid> help`（查看支持的命令） |
| **jfr** | 低开销事件采集，适合分析分配、锁、IO、GC、线程等 | `jcmd <pid> JFR.start ...` |
| **jhsdb** | 服务性调试工具，适合进程挂死或 core dump 场景 | `jhsdb jmap --heap --pid <pid>` |
| **jconsole** | 图形化监控工具 | 直接运行 `jconsole` |
| **jvisualvm** | 更强大的图形化监控和分析工具 | 直接运行 `jvisualvm` |

**2. `jstat -gcutil <pid>` 输出的各列含义是什么？如何通过它判断 GC 是否正常？**

```
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00  45.23  67.89  34.56  95.12  92.34   156    1.234    3    0.567    1.801
```

| 列 | 含义 |
|---|---|
| S0/S1 | Survivor 0/1 区使用率（%） |
| E | Eden 区使用率（%） |
| O | 老年代使用率（%） |
| M | 元空间使用率（%） |
| CCS | 压缩类空间使用率（%） |
| YGC | Young GC 次数 |
| YGCT | Young GC 总耗时（秒） |
| FGC | Full GC 次数 |
| FGCT | Full GC 总耗时（秒） |
| GCT | GC 总耗时（秒） |

判断标准：
- **O 列持续接近 100%**：老年代快满了，Full GC 后也降不下来 → 内存泄漏
- **FGC 快速增长**：频繁 Full GC → 需要排查原因
- **YGCT/YGC > 50ms**：平均每次 Young GC 超过 50ms → Young GC 耗时过长
- **FGCT/FGC > 1s**：平均每次 Full GC 超过 1 秒 → 需要优化
- **E 列快速从 0 到 100 循环**：对象分配速率很高

**3. Arthas 的常用命令有哪些？**

| 命令 | 用途 | 示例 |
|---|---|---|
| **dashboard** | 实时面板：线程、内存、GC 概览 | `dashboard` |
| **thread** | 线程相关：查看线程列表、CPU 占用、死锁 | `thread -n 3`（CPU 最高的3个线程）、`thread -b`（找阻塞线程） |
| **trace** | 方法调用链路耗时 | `trace com.example.Service method` |
| **watch** | 观察方法的入参、返回值、异常 | `watch com.example.Service method "{params, returnObj}" -x 2` |
| **stack** | 查看方法的调用栈 | `stack com.example.Service method` |
| **tt** | 时间隧道：记录方法调用，可回放 | `tt -t com.example.Service method`、`tt -p -i 1001` |
| **profiler** | 生成火焰图 | `profiler start`、`profiler stop --format html` |
| **sc** | 查看类信息 | `sc -d com.example.Service`（类的详细信息） |
| **sm** | 查看方法信息 | `sm com.example.Service`（列出所有方法） |
| **jad** | 反编译类 | `jad com.example.Service` |
| **ognl** | 执行 OGNL 表达式 | `ognl '@com.example.Config@getValue()'` |
| **heapdump** | 导出堆转储 | `heapdump /tmp/heap.hprof` |
| **logger** | 查看/修改日志级别 | `logger --name ROOT --level debug` |

**4. 如何用 Arthas 的 `trace` 命令定位方法耗时？和 `watch` 命令有什么区别？**

**trace**：追踪方法内部的调用链路，显示每个子方法的耗时，用于定位"慢在哪里"。

```bash
# 追踪 UserService.getUser 方法内部耗时
trace com.example.UserService getUser

# 输出示例：
# +---[3.2ms] com.example.UserDao.findById()
# +---[0.1ms] com.example.UserConverter.toDTO()
# +---[150ms] com.example.RemoteService.callExternal()   ← 瓶颈在这里
```

**watch**：观察方法的入参、返回值、异常、执行耗时，用于"看到了什么"。

```bash
# 观察方法的入参和返回值
watch com.example.UserService getUser "{params, returnObj}" -x 2

# 只在方法抛异常时观察
watch com.example.UserService getUser "{params, throwExp}" -e -x 2
```

核心区别：
- `trace` 关注**方法内部的调用链路和耗时分布**，帮你找到性能瓶颈
- `watch` 关注**方法的输入输出数据**，帮你看到运行时的实际值
- 两者经常配合使用：先用 trace 找到慢方法，再用 watch 看具体参数

**5. 如何用 Arthas 在不重启服务的情况下动态修改日志级别或查看方法返回值？**

**动态修改日志级别**：
```bash
# 查看当前日志级别
logger

# 修改 ROOT logger 级别为 DEBUG
logger --name ROOT --level debug

# 修改指定包的日志级别
logger --name com.example.service --level debug

# 指定 classloader（多 classloader 环境）
logger --name ROOT --level debug -c 18b4aac2
```

**查看方法返回值**：
```bash
# 用 watch 观察返回值
watch com.example.UserService getUser returnObj -x 3

# 用 tt 记录并回放
tt -t com.example.UserService getUser    # 开始记录
# 触发几次调用后
tt -l                                     # 列出记录
tt -i 1001                               # 查看第1001次调用的详细信息（入参、返回值、异常）
tt -p -i 1001                            # 重放第1001次调用
```

**6. async-profiler 是什么？如何用它生成火焰图来定位性能瓶颈？**

async-profiler 是一个低开销的 Java 采样分析器，可以采集 CPU、内存分配、锁竞争等信息，生成火焰图。它的优势是使用 AsyncGetCallTrace API，不受安全点偏差（Safepoint Bias）影响，结果更准确。

使用方式：

```bash
# 独立使用
# 1. 采集 CPU 火焰图（持续30秒）
./profiler.sh -d 30 -f cpu_flamegraph.html <pid>

# 2. 采集内存分配火焰图
./profiler.sh -d 30 -e alloc -f alloc_flamegraph.html <pid>

# 3. 采集锁竞争火焰图
./profiler.sh -d 30 -e lock -f lock_flamegraph.html <pid>
```

```bash
# 通过 Arthas 使用（更方便）
profiler start                          # 开始采集
profiler status                         # 查看状态
profiler stop --format html --file /tmp/flamegraph.html    # 停止并生成火焰图
```

火焰图阅读方法：
- **X 轴**：方法的采样占比（越宽说明 CPU 占用越多）
- **Y 轴**：调用栈深度（从下到上是调用链）
- **颜色**：无特殊含义，只是为了区分不同方法
- **重点关注**：顶部最宽的"平顶"方法，它们是真正消耗 CPU 的热点

**7. 如何用 `jcmd` 替代 `jmap` 和 `jstack`？`jcmd` 有哪些优势？**

`jcmd` 是现代 JDK 中最常用的统一诊断入口之一，很多场景可以替代或补充 `jmap`、`jstack`、`jinfo` 等分散命令。

替代关系：

| 原命令 | jcmd 等价命令 |
|---|---|
| `jstack <pid>` | `jcmd <pid> Thread.print` |
| `jmap -dump:format=b,file=heap.hprof <pid>` | `jcmd <pid> GC.heap_dump heap.hprof` |
| `jmap -heap <pid>` | `jcmd <pid> GC.heap_info` |
| `jmap -histo <pid>` | `jcmd <pid> GC.class_histogram` |
| `jinfo -flags <pid>` | `jcmd <pid> VM.flags` |
| `jstat` 的部分功能 | `jcmd <pid> PerfCounter.print` |

jcmd 独有的能力：
```bash
jcmd <pid> VM.native_memory detail       # 查看 Native 内存分布（需开启 NMT）
jcmd <pid> VM.system_properties          # 查看系统属性
jcmd <pid> VM.uptime                     # 查看 JVM 运行时间
jcmd <pid> VM.command_line               # 查看启动命令行
jcmd <pid> Compiler.directives_print     # 查看 JIT 编译指令
jcmd <pid> help                          # 查看所有支持的命令
```

jcmd 的优势：
- **更安全**：通过 JVM 内部的诊断命令协议通信，比 jmap 的 attach 机制更稳定
- **功能更全**：一个工具覆盖多个工具的功能
- **可扩展**：支持 JFR（Java Flight Recorder）等高级功能
- **官方推荐**：Oracle 明确建议用 jcmd 替代 jmap 和 jstack

#### 基础补充题

1. `jps` 和 `ps` 在查 Java 进程时有什么区别？
2. `jstack`、`jmap`、`jstat` 分别解决什么基础问题？
3. Arthas 相比 JDK 自带工具常见优势是什么？
4. 线上使用诊断工具时为什么要评估性能影响和权限风险？

### 6.6 实战场景题

**1. 线上服务响应变慢但 CPU 和内存都正常，你怎么排查？**

CPU 和内存正常但响应慢，问题通常在 IO 等待、锁、线程池或下游依赖上。排查思路：

1. **检查线程状态**：`jstack` 或 Arthas `thread`，看是否有大量线程处于 WAITING/BLOCKED/TIMED_WAITING
   - 大量 BLOCKED → 锁竞争
   - 大量 TIMED_WAITING 在 `SocketInputStream.read` → 下游服务慢
   - 大量 WAITING 在 `LinkedBlockingQueue.take` → 线程池队列排队

2. **检查网络和下游**：
   - 用 `ss -tnp` 或 `netstat` 查看 TCP 连接状态，是否有大量 CLOSE_WAIT（对端关闭但本地未关闭）
   - 检查下游服务（数据库、Redis、第三方 API）的响应时间

3. **检查 IO**：`iostat -x 1` 查看磁盘 IO，`%util` 接近 100% 说明磁盘是瓶颈

4. **检查 GC**：虽然内存"正常"，但可能有频繁的短暂 GC 暂停，`jstat -gcutil` 确认

5. **用 Arthas trace 定位**：`trace` 具体接口方法，看耗时分布在哪个环节

6. **检查连接池**：数据库连接池、HTTP 连接池是否耗尽，导致请求排队等待连接

**2. 服务启动后运行一段时间就 OOM，但堆内存设置得很大，可能是什么原因？**

堆设置很大还 OOM，需要区分是哪种 OOM：

- **堆 OOM（Java heap space）**：
  - 内存泄漏：对象被长生命周期引用持有，GC 无法回收，随时间累积
  - 缓存无上限：本地缓存（HashMap/ConcurrentHashMap）不断增长
  - Session 对象堆积：大量用户 Session 未过期清理
  - 大查询结果集：分页查询写错导致全表加载

- **元空间 OOM（Metaspace）**：
  - 动态生成类过多（CGLib、反射、Groovy 脚本等）
  - 没有设置 `-XX:MaxMetaspaceSize`，默认无上限但实际受物理内存限制

- **直接内存 OOM（Direct buffer memory）**：
  - NIO/Netty 的 ByteBuffer 未正确释放
  - `-XX:MaxDirectMemorySize` 设置过小

- **线程栈 OOM（unable to create new native thread）**：
  - 线程数过多，每个线程默认 1MB 栈空间
  - 操作系统的线程数限制（`ulimit -u`）

排查方法：先看 OOM 的具体错误信息，确定是哪种类型，再针对性分析。

**3. 某个接口偶尔超时，大部分时候正常，你怎么定位？**

"偶尔"超时说明不是代码逻辑问题，而是某种间歇性因素。常见原因和排查：

1. **GC STW**：
   - 查看 GC 日志，看超时时间点是否与 GC 暂停重合
   - 特别关注 Full GC、G1 并发标记的 Remark 阶段、Mixed GC 和 safepoint 暂停

2. **锁竞争**：
   - 高并发时偶尔出现锁等待超时
   - 用 Arthas `thread -b` 找阻塞线程

3. **线程池排队**：
   - 线程池满了，请求在队列中等待
   - 监控线程池的 `queueSize` 和 `activeCount`

4. **下游服务抖动**：
   - 数据库慢查询、Redis 大 Key、第三方 API 偶尔超时
   - 用 trace 看方法内部耗时分布

5. **网络抖动**：
   - 容器环境下网络偶尔波动
   - 检查 TCP 重传率

6. **JIT 编译**：
   - 服务刚启动时，热点代码还在解释执行，偶尔触发 JIT 编译会有短暂暂停
   - 可以用 `-XX:+PrintCompilation` 确认

排查技巧：用 Arthas 的 `tt` 命令记录每次调用，对比正常和超时的调用，看差异在哪里。

**4. 发布新版本后 Full GC 频率明显增加，你怎么排查是哪段代码引起的？**

1. **对比 GC 日志**：新旧版本的 GC 日志对比，看老年代增长速率、对象晋升速率的变化

2. **对比堆快照**：
   - 分别在新旧版本运行一段时间后 dump 堆
   - 用 MAT 的 Histogram 对比，看哪些类的对象数量/大小明显增加

3. **代码 diff 分析**：
   - 查看本次发布的代码变更，重点关注：
     - 新增的缓存、集合类字段
     - 修改了数据查询逻辑（是否查了更多数据）
     - 新增了定时任务或后台线程
     - 引入了新的第三方库

4. **用 Arthas profiler 对比**：
   - 生成内存分配火焰图（`profiler start -e alloc`），看哪些方法分配了大量对象

5. **灰度验证**：如果有条件，灰度回滚到旧版本确认 Full GC 恢复正常，确认是新代码引起的

**5. 容器环境下 JVM 被 OOM Killer 杀掉，但堆内存没有超限，可能是什么原因？**

容器的 OOM Killer 看的是进程的 RSS（实际物理内存），而不是 JVM 堆大小。JVM 进程的内存 = 堆 + 元空间 + 线程栈 + 直接内存 + Code Cache + GC 开销 + JNI + 其他。

常见原因：
- **堆外内存未计入**：`-Xmx` 只限制堆大小，但 DirectByteBuffer、Netty 的堆外内存不受此限制
- **线程栈**：每个线程默认 1MB（`-Xss`），1000 个线程就是 1GB
- **元空间**：未设置 `-XX:MaxMetaspaceSize`，可能无限增长
- **Code Cache**：JIT 编译的代码缓存，默认 240MB
- **容器内存限制设置不合理**：容器 limit 设置得太接近 `-Xmx`，没有给非堆内存留余量

解决方案：
```bash
# 1. 容器内存 = Xmx + 非堆预留（建议预留 Xmx 的 50%-100%）
# 例如 Xmx=2g，容器 limit 至少设 3g-4g

# 2. 用 NMT 查看内存分布
-XX:NativeMemoryTracking=summary
jcmd <pid> VM.native_memory summary

# 3. 限制各区域内存
-Xmx2g                          # 堆
-XX:MaxMetaspaceSize=256m        # 元空间
-XX:MaxDirectMemorySize=512m     # 直接内存
-XX:ReservedCodeCacheSize=128m   # Code Cache
-Xss512k                        # 线程栈（减小单个线程栈大小）

# 4. 确认当前运行时的容器感知能力和参数
-XX:+UseContainerSupport
-XX:MaxRAMPercentage=75.0        # 堆占容器内存的75%
```

**6. 线上出现 `StackOverflowError`，如何快速定位是哪个递归调用导致的？**

`StackOverflowError` 说明线程栈溢出，通常是递归调用没有正确的终止条件。

快速定位方法：

1. **看异常堆栈**：StackOverflowError 的堆栈信息会显示重复的方法调用模式
   ```
   java.lang.StackOverflowError
       at com.example.TreeService.traverse(TreeService.java:45)
       at com.example.TreeService.traverse(TreeService.java:48)
       at com.example.TreeService.traverse(TreeService.java:48)
       ... (重复数百次)
   ```
   重复出现的方法就是递归调用点。

2. **常见场景**：
   - 树/图的遍历没有终止条件或存在环
   - 对象的 `toString()`、`hashCode()`、`equals()` 方法中存在循环引用
   - JSON 序列化时对象存在双向引用（A 引用 B，B 引用 A）
   - 递归算法的边界条件写错

3. **解决方案**：
   - 修复递归终止条件
   - 用迭代替代递归（用显式栈模拟递归）
   - 增大线程栈大小（治标不治本）：`-Xss2m`
   - JSON 序列化加 `@JsonIgnore` 或 `@JsonManagedReference` 打破循环引用

**7. 数据库连接池耗尽导致服务不可用，如何从 JVM 层面排查连接泄漏？**

连接池耗尽的表现：请求超时，日志中出现 `Cannot get a connection, pool error Timeout waiting for idle object` 等错误。

排查步骤：

1. **确认连接池状态**：
   - 如果用 Druid，访问监控页面查看活跃连接数、等待线程数
   - 如果用 HikariCP，通过 JMX 或 Actuator 查看 `activeConnections`、`pendingThreads`

2. **用 jstack 分析**：
   ```bash
   jstack <pid> | grep -A 20 "TIMED_WAITING"
   ```
   看是否有大量线程在等待获取数据库连接（堆栈中出现 `getConnection`、`borrowObject` 等）

3. **用 Arthas 定位**：
   ```bash
   # 追踪获取连接的方法，看哪些调用获取了连接但长时间未归还
   trace javax.sql.DataSource getConnection
   
   # 用 watch 观察连接的获取和关闭
   watch java.sql.Connection close "{target}" -x 1
   ```

4. **常见泄漏原因**：
   - 代码中获取连接后没有在 finally 块中关闭
   - 事务超时但连接未释放
   - 使用了手动事务管理但异常路径没有 rollback + close
   - 慢查询占用连接时间过长

5. **连接池配置检查**：
   ```properties
   # HikariCP 推荐配置
   maximumPoolSize=20              # 最大连接数
   minimumIdle=5                   # 最小空闲连接
   connectionTimeout=30000         # 获取连接超时（30秒）
   maxLifetime=1800000             # 连接最大存活时间（30分钟）
   leakDetectionThreshold=60000   # 连接泄漏检测阈值（60秒未归还则告警）
   ```
   开启 `leakDetectionThreshold` 后，HikariCP 会在日志中打印泄漏连接的获取堆栈，直接定位到泄漏代码。

#### 基础补充题

1. 接口慢、CPU 高、内存高三类问题的第一反应分别应该看什么？
2. 为什么线上问题排查要记录时间点、版本和流量变化？
3. 排查偶发问题时为什么需要日志、指标、链路追踪一起看？
4. 回滚、限流、降级和排查之间如何确定优先级？

## 七、JIT 编译

1. 解释执行和编译执行的区别？
2. 热点代码检测？方法调用计数器和回边计数器？
3. C1 和 C2 编译器的区别？分层编译？
4. JIT 的常见优化手段？（方法内联、逃逸分析、标量替换等）

### 基础补充题

1. 什么是热点代码？
2. 为什么 JVM 既有解释执行也有编译执行？
3. 方法内联为什么能提升性能？
4. JIT 编译会不会影响服务刚启动时的性能表现？
