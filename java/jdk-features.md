### JDK8 新特性

- Lambda 表达式 - Lambda 允许把函数作为一个方法的参数（函数作为参数传递到方法中）。

- 方法引用 - 方法引用提供了非常有用的语法，可以直接引用已有Java类或对象（实例）的方法或构造器。与lambda联合使用，方法引用可以使语言的构造更紧凑简洁，减少冗余代码。

- 默认方法 - 默认方法就是一个在接口里面有了一个实现的方法。

- 新工具 - 新的编译工具，如：Nashorn引擎 jjs、 类依赖分析器jdeps。

- Stream API - 新添加的Stream API（java.util.stream） 把真正的函数式编程风格引入到Java中。

- Date Time API - 加强对日期与时间的处理。

- Optional 类 - Optional 类已经成为 Java 8 类库的一部分，用来解决空指针异常。

- Nashorn, JavaScript 引擎 - Java 8提供了一个新的Nashorn javascript引擎，它允许我们在JVM上运行特定的javascript应用。

### JDK9 新特性

- 模块系统 - 模块是一个包的容器，Java 9 最大的变化之一是引入了模块系统（Jigsaw 项目）。

- REPL (JShell) - 交互式编程环境。

- HTTP 2 客户端 - HTTP/2标准是HTTP协议的最新版本，新的 HTTPClient API 支持 WebSocket 和 HTTP2 流以及服务器推送特性。

- 改进的 Javadoc - Javadoc 现在支持在 API 文档中的进行搜索。另外，Javadoc 的输出现在符合兼容 HTML5 标准。

- 多版本兼容 JAR 包 - 多版本兼容 JAR 功能能让你创建仅在特定版本的 Java 环境中运行库程序时选择使用的 class 版本。

- 集合工厂方法 - List，Set 和 Map 接口中，新的静态工厂方法可以创建这些集合的不可变实例。

- 私有接口方法 - 在接口中使用private私有方法。我们可以使用 private 访问修饰符在接口中编写私有方法。

- 进程 API - 改进的 API 来控制和管理操作系统进程。引进 java.lang.ProcessHandle 及其嵌套接口 Info 来让开发者逃离时常因为要获取一个本地进程的 PID 而不得不使用本地代码的窘境。

- 改进的 Stream API - 改进的 Stream API 添加了一些便利的方法，使流处理更容易，并使用收集器编写复杂的查询。

- 改进 try-with-resources - 如果你已经有一个资源是 final 或等效于 final 变量,您可以在 try-with-resources 语句中使用该变量，而无需在 try-with-resources 语句中声明一个新变量。

- 改进的弃用注解 @Deprecated - 注解 @Deprecated 可以标记 Java API 状态，可以表示被标记的 API 将会被移除，或者已经破坏。

- 改进钻石操作符(Diamond Operator) - 匿名类可以使用钻石操作符(Diamond Operator)。

- 改进 Optional 类 - java.util.Optional 添加了很多新的有用方法，Optional 可以直接转为 stream。

- 多分辨率图像 API - 定义多分辨率图像API，开发者可以很容易的操作和展示不同分辨率的图像了。

- 改进的 CompletableFuture API - CompletableFuture 类的异步机制可以在 ProcessHandle.onExit 方法退出时执行操作。

- 轻量级的 JSON API - 内置了一个轻量级的JSON API。

- 响应式流（Reactive Streams）API - Java 9中引入了新的响应式流 API 来支持 Java 9 中的响应式编程。

### JDK10 新特性

- 局部变量类型推断（Local-Variable Type Inference）

- 将JDK资料整合到单个存储库中（Consolidate the JDK Forest into a Single Repository）

- 垃圾回收器接口（Garbage Collector Interface）- 通过引入简明的垃圾收集器(GC)接口，改进不同垃圾收集器的源代码隔离。

- G1的并行Full GC（Parallel Full GC for G1）- 通过使Full GC并行来改善G1最坏情况延迟。

- 应用程序类数据共享（Application Class-Data Sharing）- 为了改进启动和内存占用，可以扩展现有的类-数据共享(“CDS”)特性，以允许将应用程序类放在共享存档中。

- 局部线程握手协议（Thread-Local Handshakes）- 引入一种在线程上执行回调而不执行全局VM安全点的方法。使停止单个线程(而不是停止所有线程或不停止)成为可能且成本低廉。

- 删除javah（Remove the Native-Header Generation Tool (javah)）- 从JDK中删除javah工具。

- 额外的Unicode语言标签扩展（Additional Unicode Language-Tag Extensions）- 增强java.util.Locale和相关api，以实现BCP 47语言标记的额外Unicode扩展。

- 可选内存设备上的堆分配（Heap Allocation on Alternative Memory Devices）- 启用HotSpot VM在用户指定的替代内存设备(如NV-DIMM)上分配Java对象堆。

- 实验性的基于java的JIT编译器（Experimental Java-Based JIT Compiler）- 启用基于java的JIT编译器Graal作为Linux/x64平台上的实验性JIT编译器。

- 根证书（Root Certificates）- 在JDK中提供一组默认的根证书机构(CA)颁发证书。

### JDK11 新特性

- 改进Aarch64的固有函数（Improve Aarch64 Intrinsics）- 改进现有的字符串和数组的固有函数，并在AArch64处理器上为java.lang.Math的sin、cos和log函数实现新的固有函数。

- 删除Java EE和CORBA模块（Remove the Java EE and CORBA Modules）

- HTTP Client - Java 11后开始支持HTTP2，底层进行了大幅度的优化，并且现在完全支持异步非阻塞。

- Lambda参数的局部变量语法（Local-Variable Syntax for Lambda Parameters）- 允许在声明隐式类型化lambda表达式的形式参数时使用var。

- 支持Unicode 10版本

- 飞行记录仪（Flight Recorder）- 提供低开销的数据收集框架，用于对Java应用程序和HotSpot JVM进行故障排除。

- 启动单文件源代码程序（Launch Single-File Source-Code Programs）- 增强java启动器以运行作为单个java源代码文件提供的程序，包括通过“shebang”文件和相关技术从脚本中使用程序。

- 低开销堆分析（Low-Overhead Heap Profiling）- 提供一种低开销的Java堆分配抽样方法，可通过JVMTI访问。

- 实现传输层安全协议1.3（Transport Layer Security (TLS) 1.3）

- 试验地引入ZGC

- 弃用Nashorn JavaScript引擎

- 弃用Pack200工具和API

### JDK12 新特性

- 试验地引入Shenandoah低暂停垃圾回收器

- 微基准测试套件（Microbenchmark Suite）- 向JDK源代码中添加一套基本的微基准测试，使开发人员可以轻松地运行现有的微基准测试并创建新的微基准测试。

- Switch表达式(预览版)

- JVM常量API（JVM Constants API）- 引入一个API来对关键类文件和运行时工件(特别是可从常量池加载的常量)的标称描述进行建模。

- 保留一个AArch64端口 - 删除与arm64端口相关的所有源代码，同时保留32位ARM端口和64位aarch64端口。

- 默认的CDS档案（Default CDS Archives）- 增强JDK构建过程，在64位平台上使用默认类列表生成类数据共享(CDS，class data-sharing)归档。

- G1的可中止混合回收（Abortable Mixed Collections for G1）- 如果G1混合回收可能超过预设的暂停时间，则使其可中止。

- G1快速释放已申请但未使用的内存（Promptly Return Unused Committed Memory from G1）- 增强G1垃圾收集器，以便在空闲时自动将Java堆内存返回给操作系统。

### JDK13 新特性

- 动态CDS档案（Dynamic CDS Archives）- 扩展应用程序类数据共享，允许在Java应用程序执行结束时对类进行动态归档。归档的类将包括所有加载的应用程序类和库类，这些类没有出现在默认的基础层CDS归档中。

- 试验地增强ZGC - 增强ZGC以将未使用的堆内存返回给操作系统。

- 重新实现旧 Socket API（Reimplement the Legacy Socket API）- 将java.net.Socket和java.net.ServerSocket api使用的底层实现替换为更简单、更现代、易于维护和调试的实现。新的实现将更适用于用户模式线程（也就是纤程），目前正在Project Loom中进行探索。

- Switch表达式(第二次预览版)

- 文本块（Text Blocks）(预览版)

### JDK14 新特性

- instanceof的匹配模式（Pattern Matching for instanceof）(预览版)

- 打包工具(孵化器)

- G1的NUMA-Aware内存分配（NUMA-Aware Memory Allocation for G1）- 通过实现非均匀内存访问感知的内存分配来提高大型机器上的G1性能。

- JFR事件流（JFR Event Streaming）- 公开JDK Flight Recorder数据以进行持续监控。

- 非易失性映射字节缓冲区（Non-Volatile Mapped Byte Buffers）- 添加新的特定于jdk的文件映射模式，以便FileChannel API可以用于创建引用非易失性内存的MappedByteBuffer实例。

- 实用的NullPointerExceptions - 通过精确描述哪个变量为空，提高JVM生成的NullPointerExceptions的可用性。

- Records(预览版) - 提供了一种紧凑的语法，声明贫血的、不可变数据的类。

- Switch表达式

- 弃用Solaris和SPARC端口

- 删除并发标记扫描(CMS)垃圾收集器

- 试验地将ZGC垃圾回收器移植到macOS和Windows

- 弃用parallelscscraper + SerialOld GC组合

- 删除Pack200 Tools和API

- 文本块（Text Blocks）(第二次预览版)

- 外部内存访问API(孵化器) - 引入一个API，允许Java程序安全有效地访问Java堆之外的外部内存。

### JDK15 新特性

- 密封类（Sealed Classes）(预览版) - 密封类和接口限制了哪些其他类或接口可以扩展或实现它们。

- 隐藏类（Hidden Classes）- 隐藏类不能被其他类的字节码直接使用。隐藏类是为在运行时生成类并通过反射间接使用它们的框架所使用的。隐藏类可以定义为访问控制嵌套的成员，并且可以独立于其他类卸载。

- 移除Nashorn JavaScript引擎

- 重新实现旧的DatagramSocket API - 将java.net.DatagramSocket和java.net.MulticastSocket api的底层实现替换为更简单、更现代、易于维护和调试的实现。新的实现将很容易适应虚拟线程的工作，目前正在Project Loom中进行探索。

- 废弃并禁用偏向锁（Deprecate and Disable Biased Locking）

- instanceof的匹配模式（Pattern Matching for instanceof）(第二次预览)

- ZGC（可伸缩的低延迟垃圾收集器）

- 文本块（Text Blocks）

- Shenandoah（一个低暂停的垃圾收集器）

- 删除Solaris和SPARC端口

- 外部内存访问API(第二次孵化) - 引入一个API，允许Java程序安全有效地访问Java堆之外的外部内存。

- Records(第二次预览) - 提供了一种紧凑的语法，声明贫血的不可变数据的类。

- 废弃RMI激活机制（Deprecate RMI Activation for Removal）

### JDK16 新特性

- 向量的API（Vector API）(孵化器) - 提供孵化器模块的初始迭代。jdk.incubator.vector，表示在运行时可靠地在支持的CPU架构上编译为最优矢量硬件指令的矢量计算，从而获得优于等效标量计算的性能。

- 启用c++ 14语言特性（Enable C++14 Language Features）- 允许在JDK c++源代码中使用c++ 14语言特性，并对HotSpot代码中可以使用哪些特性给出具体的指导。

- 从Mercurial迁移到Git，Github（Migrate from Mercurial to Git，GitHub）

- ZGC并发线程栈处理（ZGC: Concurrent Thread-Stack Processing）- 将ZGC线程堆栈处理从安全点移到并发阶段。

- unix域套接字通道（Unix-Domain Socket Channels）- 在java.nio.channels包中为socket channel和server-socket channel api添加unix域(AF_UNIX)套接字支持。扩展继承的通道机制以支持unix域套接字通道和服务器套接字通道。

- 弹性元空间（Elastic Metaspace）- 将未使用的HotSpot类元数据(即元空间)内存更及时地返回给操作系统，减少元空间占用，简化元空间代码，从而降低维护成本。

- 外部链接器API（Foreign Linker API）- 引入一个API，提供对native代码的静态类型纯java访问。这个API与外部内存API (JEP 393)一起，将大大简化绑定到本机库的过程，否则容易出错。

- 对基于值的类的警告（Warnings for Value-Based Classes）- 将基础类型包装类指定为基于值的类，并弃用它们的构造函数以移除，从而提示新的弃用警告。对于在Java平台中任何基于值的类的实例上进行不正确的同步尝试，提供警告。

- 打包工具（Packaging Tool）- 提供jpackage工具，用于打包自包含的Java应用程序。

- 外部内存访问API(第三次次孵化) - 引入一个API，允许Java程序安全有效地访问Java堆之外的外部内存。

- instanceof的匹配模式（Pattern Matching for instanceof）- 匹配模式合并了判断、强制类型转换操作，以更简洁和安全的方式表达。

- Records - 提供了一种紧凑的语法，声明贫血的、不可变数据的类。

- 默认强封装JDK内部组件（Strongly Encapsulate JDK Internals by Default）- 默认情况下，强封装JDK的所有内部元素，但关键的内部api(如sun.misc.Unsafe)除外。允许最终用户选择宽松的强封装（JDK9以来的默认设置）。

- 密封类（Sealed Classes）(第二次预览) - 密封类和接口限制了哪些其他类或接口可以扩展或实现它们。