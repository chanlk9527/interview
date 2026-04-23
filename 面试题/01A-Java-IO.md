# 01A - Java IO

---

## 一、IO 基础

1. Java IO 流的分类？按方向（输入/输出）、按数据（字节/字符）、按功能（节点/处理）？
2. InputStream、OutputStream、Reader、Writer 四大基类的关系？
3. 字节流和字符流的区别？什么时候用字节流，什么时候用字符流？
4. 字符编码问题？UTF-8、GBK、ISO-8859-1？乱码的原因和解决方案？
5. 装饰器模式在 Java IO 中的体现？BufferedInputStream 包装 FileInputStream？
6. 常用的 IO 流有哪些？FileInputStream、BufferedReader、InputStreamReader、PrintWriter 等？
7. flush() 和 close() 的区别？为什么要 flush？
8. RandomAccessFile 的作用？和普通流的区别？支持什么操作？

## 二、BIO（Blocking IO）

1. BIO 的工作模型？为什么叫阻塞 IO？
2. BIO 的 Socket 编程模型？ServerSocket 和 Socket？
3. BIO 的线程模型？一个连接一个线程的问题？
4. BIO 适合什么场景？为什么不适合高并发？

## 三、NIO（Non-blocking IO）

1. NIO 和 BIO 的核心区别？面向流 vs 面向缓冲区？阻塞 vs 非阻塞？
2. NIO 的三大核心组件？Channel、Buffer、Selector？
3. Channel 的常见实现？FileChannel、SocketChannel、ServerSocketChannel、DatagramChannel？
4. Buffer 的核心属性？capacity、position、limit、mark？flip()、clear()、compact() 的作用？
5. Buffer 的类型？HeapByteBuffer vs DirectByteBuffer？直接内存的优缺点？
6. Selector 的作用？多路复用的原理？
7. SelectionKey 的四种事件？OP_ACCEPT、OP_CONNECT、OP_READ、OP_WRITE？
8. NIO 的编程模型？一个线程处理多个连接的流程？
9. NIO 的 Scatter/Gather 是什么？
10. 为什么直接用 Java NIO 编程很复杂？（空轮询 Bug、半包粘包、异常处理等）

## 四、AIO（Asynchronous IO）

1. AIO 和 NIO 的区别？真正的异步非阻塞？
2. AIO 的核心类？AsynchronousSocketChannel、AsynchronousServerSocketChannel？
3. AIO 的两种使用方式？Future 方式 vs CompletionHandler 回调方式？
4. AIO 为什么在 Linux 上没有被广泛使用？（Linux 的 AIO 实现不成熟，epoll 已经够用）
5. 为什么 Netty 放弃了 AIO？

## 五、IO 模型对比

1. 五种 IO 模型？阻塞 IO、非阻塞 IO、IO 多路复用、信号驱动 IO、异步 IO？
2. 同步/异步和阻塞/非阻塞的区别？这是两个维度的概念？
3. select、poll、epoll 的区别？为什么 epoll 性能更好？
4. 水平触发（LT）和边缘触发（ET）的区别？
5. Reactor 模式和 Proactor 模式的区别？

## 六、零拷贝

1. 传统 IO 的数据拷贝流程？（4 次拷贝、4 次上下文切换）
2. 零拷贝是什么？为什么能提升性能？
3. mmap（内存映射）的原理？MappedByteBuffer？
4. sendfile 的原理？FileChannel.transferTo()？
5. Java 中实现零拷贝的方式？FileChannel.transferTo/transferFrom、MappedByteBuffer？
6. Netty 的零拷贝实现？CompositeByteBuf、wrap、FileRegion？
7. Kafka 为什么快？零拷贝在 Kafka 中的应用？

## 七、序列化

1. 什么是序列化和反序列化？为什么需要序列化？
2. Java 原生序列化？Serializable 接口的作用？serialVersionUID 的作用？
3. Externalizable 和 Serializable 的区别？
4. transient 关键字的作用？被 transient 修饰的字段能被序列化吗？
5. Java 原生序列化的缺点？（性能差、体积大、安全漏洞）
6. 常见的序列化框架对比？JSON（Jackson/Gson）、Protobuf、Hessian、Kryo、Avro？
7. Protobuf 的优势？为什么比 JSON 快？
8. 如何选择序列化方案？跨语言、性能、可读性的权衡？
9. 序列化在 RPC 框架中的应用？Dubbo 的序列化选型？

## 八、文件与 NIO.2

1. Files 和 Paths 工具类的常用方法？（Java 7 NIO.2）
2. WatchService 文件监听的使用？
3. 大文件读写的最佳实践？BufferedReader 逐行读取 vs FileChannel + MappedByteBuffer？
4. 文件锁（FileLock）的使用？共享锁和排他锁？
