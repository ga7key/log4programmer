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

- 弃用RMI激活机制以备废除（Deprecate RMI Activation for Removal）

### JDK16 新特性

- 矢量的API（Vector API）(孵化器) - 提供孵化器模块的初始迭代。jdk.incubator.vector，表示在运行时可靠地在支持的CPU架构上编译为最优矢量硬件指令的矢量计算，从而获得优于等效标量计算的性能。

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

### JDK17 新特性

- 恢复始终严格的浮点语义（Restore Always-Strict Floating-Point Semantics）- 使浮点操作始终严格，而不是同时具有严格的浮点语义(strictfp)和略有不同的默认浮点语义。这将恢复语言和VM的原始浮点语义，与Java SE 1.2中引入严格和默认浮点模式之前的语义相匹配。

- 增强的伪随机数生成器（Enhanced Pseudo-Random Number Generators）- 为伪随机数生成器(PRNG)提供新的接口类型和实现，包括可跳转PRNG和另一类可分割PRNG算法(LXM)。

- 新的macOS渲染管道（New macOS Rendering Pipeline）- 使用Apple Metal API为macOS实现一个Java 2D内部渲染管道，作为现有管道的替代方案。

- macOS/AArch64 Port

- 弃用Applet API以备废除（Deprecate the Applet API for Removal）

- 强封装JDK内部（Strongly Encapsulate JDK Internals）- 强烈封装JDK的所有内部元素，除了关键的内部api(如sun.misc.Unsafe)。不再可能像JDK 9到JDK 16那样，通过单个命令行选项放松对内部元素的强封装。

- switch模式匹配（Pattern Matching for switch）(预览版) - 将模式匹配扩展到switch允许针对多个模式对表达式进行测试，每个模式都有一个特定的操作，这样就可以简明而安全地表达复杂的面向数据的查询。

- 移除RMI激活（Remove RMI Activation）

- 密封类（Sealed Classes）- 密封类和接口限制了哪些其他类或接口可以扩展或实现它们。

- 删除实验性AOT和JIT编译器（Remove the Experimental AOT and JIT Compiler）

- 弃用安全管理器以备废除（Deprecate the Security Manager for Removal）

- 外部函数和内存API（Foreign Function & Memory API）(孵化器) - 引入一个API，通过该API, Java程序可以与Java运行时之外的代码和数据进行互操作。通过有效地调用外部函数(即JVM外部的代码)和安全地访问外部内存(即JVM不管理的内存)，API使Java程序能够调用本机库并处理本机数据，而不会出现JNI的脆弱性和危险。

- 矢量的API（Vector API）(第二次孵化) - 引入API来表达矢量计算，在运行时可靠地在支持的CPU架构上编译为最优矢量指令，从而实现优于等效标量计算的性能。

- 上下文特定的反序列化过滤器（Context-Specific Deserialization Filters）- 允许应用程序通过jvm范围的过滤器工厂配置上下文特定的和动态选择的反序列化过滤器，该过滤器工厂被调用以为每个单独的反序列化操作选择过滤器。

### JDK18 新特性

- 默认的UTF-8字符集（UTF-8 by Default）- 指定UTF-8作为标准Java api的默认字符集。通过此更改，依赖于默认字符集的api将在所有实现、操作系统、语言环境和配置中保持一致。

- 简易Web服务器（Simple Web Server）- 提供一个命令行工具来启动一个只提供静态文件的最小web服务器。没有CGI或类似servlet的功能可用。这个工具对于原型设计、特别编码和测试非常有用，特别是在教育环境中。

- Java API文档中的代码片段（Code Snippets in Java API Documentation）- 为JavaDoc的Standard Doclet引入@snippet标记，以简化在API文档中包含示例源代码的过程。

- 用方法句柄重新实现核心反射（Reimplement Core Reflection with Method Handles）- 在 java.lang.invoke 的方法句柄之上，重构 java.lang.reflect的方法、构造函数和字段，使用方法句柄处理反射的底层机制将减少 java.lang.reflect 和 java.lang.invoke 两者的 API 维护和开发成本。

- 矢量的API（Vector API）(第三次孵化) - 引入API来表达矢量计算，在运行时可靠地在支持的CPU架构上编译为最优矢量指令，从而实现优于等效标量计算的性能。

- 互联网地址解析SPI（Internet-Address Resolution SPI）- 为主机名和地址解析定义一个服务提供者接口(SPI)，以便java.net.InetAddress可以使用平台内置解析器以外的解析器。

- 外部函数和内存API（Foreign Function & Memory API）(第二次孵化) - 引入一个API，通过该API, Java程序可以与Java运行时之外的代码和数据进行互操作。通过有效地调用外部函数(即JVM外部的代码)和安全地访问外部内存(即JVM不管理的内存)，API使Java程序能够调用本机库并处理本机数据，而不会出现JNI的脆弱性和危险。

- switch模式匹配（Pattern Matching for switch）(第二次预览) - 将模式匹配扩展到switch允许针对多个模式对表达式进行测试，每个模式都有一个特定的操作，这样就可以简明而安全地表达复杂的面向数据的查询。

- 弃用Finalization以备删除（Deprecate Finalization for Removal）- Finalization旨在帮助避免资源泄漏问题，然而这个功能存在延迟不可预测、行为不受约束，以及线程无法指定等缺陷，导致其安全性、性能、可靠性和可维护性方面都存在问题，因此将其弃用，用户可选择迁移到其他资源管理技术，例如try-with-resources语句和清理器。

### JDK19 新特性

- Record模式（Record Patterns）(预览版) - 可以嵌套Record模式和Type模式，以支持强大的、声明性的和可组合的数据导航和处理形式。

- Linux/RISC-V Port

- 外部函数和内存API（Foreign Function & Memory API）(预览版) - 引入一个API，通过该API, Java程序可以与Java运行时之外的代码和数据进行互操作。通过有效地调用外部函数(即JVM外部的代码)和安全地访问外部内存(即JVM不管理的内存)，API使Java程序能够调用本机库并处理本机数据，而不会出现JNI的脆弱性和危险。

- 虚拟线程（Virtual Threads）(预览版) - 向Java平台引入虚拟线程。虚拟线程是轻量级线程，可以显著减少编写、维护和观察高吞吐量并发应用程序的工作量。

- 矢量的API（Vector API）(第四次孵化) - 引入API来表达矢量计算，在运行时可靠地在支持的CPU架构上编译为最优矢量指令，从而实现优于等效标量计算的性能。

- switch模式匹配（Pattern Matching for switch）(第三次预览) - 将模式匹配扩展到switch允许针对多个模式对表达式进行测试，每个模式都有一个特定的操作，这样就可以简明而安全地表达复杂的面向数据的查询。

- 结构化并发性（Structured Concurrency）(孵化器) - 通过引入结构化并发的API来简化多线程编程。结构化并发将在不同线程中运行的多个任务视为单个工作单元，从而简化了错误处理和取消，提高了可靠性并增强了可观察性。

### JDK21 新特性

- 字符串模板（String Templates）(预览版)- 字符串模板通过将文本与嵌入表达式和模板处理器耦合来产生专门的结果，从而补充了Java现有的字符串文字和文本块。

- 序列化集合（Sequenced Collections）- 引入新的接口来表示具有定义的相遇顺序的集合。每个这样的集合都有定义良好的第一个元素、第二个元素，依此类推，直到最后一个元素。它还提供了统一的api来访问它的第一个和最后一个元素，并以相反的顺序处理它的元素。

- ZGC分代（Generational ZGC）- 通过扩展Z Garbage Collector(ZGC)来为年轻对象和老对象维护单独的代，从而提高应用程序性能。这将允许ZGC更频繁地收集年轻的对象——它们往往会在年轻时死亡。

- Record模式（Record Patterns）- 可以嵌套Record模式和Type模式，以支持强大的、声明性的和可组合的数据导航和处理形式。

- switch模式匹配（Pattern Matching for switch）- 将模式匹配扩展到switch允许针对多个模式对表达式进行测试，每个模式都有一个特定的操作，这样就可以简明而安全地表达复杂的面向数据的查询。

- 外部函数和内存API（Foreign Function & Memory API）(第三次预览) - 引入一个API，通过该API, Java程序可以与Java运行时之外的代码和数据进行互操作。通过有效地调用外部函数(即JVM外部的代码)和安全地访问外部内存(即JVM不管理的内存)，API使Java程序能够调用本机库并处理本机数据，而不会出现JNI的脆弱性和危险。

- 未命名模式和变量（Unnamed Patterns and Variables）- 未命名模式匹配记录组件而不声明组件的名称或类型，以及未命名变量，这些变量可以初始化但不能使用。两者都由下划线_表示。

- 虚拟线程（Virtual Threads）- 向Java平台引入虚拟线程。虚拟线程是轻量级线程，可以显著减少编写、维护和观察高吞吐量并发应用程序的工作量。

- 未命名类和实例主方法（Unnamed Classes and Instance Main Methods）(预览版) - 发展Java语言，使学生可以编写他们的第一个程序，而不需要了解为大型程序设计的语言特性。学生不必使用单独的Java方言，而是可以为单类程序编写精简的声明，然后随着技能的提高，无缝地扩展程序，使用更高级的功能。

- 作用域值（Scoped Values）(预览版) - 引入有作用域的值，这些值可以在不使用方法参数的情况下安全有效地共享给方法。它们比线程局部变量更可取，特别是在使用大量虚拟线程时。实际上，作用域值是隐式方法参数。这就“好像”调用序列中的每个方法都有一个额外的、不可见的参数。没有任何方法声明这个参数，只有能够访问作用域值对象的方法才能访问它的值(数据)。有作用域的值可以通过一系列中间方法将数据从调用者安全地传递给远程被调用者，这些中间方法不为数据声明参数，也不能访问数据。

- 矢量的API（Vector API）(第六次孵化) - 引入API来表达矢量计算，在运行时可靠地在支持的CPU架构上编译为最优矢量指令，从而实现优于等效标量计算的性能。

- 弃用Windows 32位x86端口以备废除（Deprecate the Windows 32-bit x86 Port for Removal）

- 准备禁止动态加载代理（Prepare to Disallow the Dynamic Loading of Agents）- 在将代理动态加载到正在运行的JVM中时发出警告。这些警告的目的是让用户为将来的版本做好准备，该版本默认情况下不允许动态加载代理，以便在默认情况下提高完整性。在启动时加载代理的可服务性工具不会在任何版本中发出警告。

- 密钥封装机制API（Key Encapsulation Mechanism API）- 引入密钥封装机制(kem)的API，这是一种使用公钥加密保护对称密钥的加密技术。

- 结构化并发性（Structured Concurrency）(预览版) - 通过引入结构化并发的API来简化多线程编程。结构化并发将在不同线程中运行的多个任务视为单个工作单元，从而简化了错误处理和取消，提高了可靠性并增强了可观察性。