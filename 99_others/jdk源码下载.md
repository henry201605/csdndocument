​	最近在研究多线程的内容，在看Java源码的时候有很多是调用native的本地方法，就想着下载一下Hotspot源码从而能更深入的了解一下底层原理。好了废话不多说，下面介绍一下Hotspot源码下载方法。

## 1、登录

打开`openjdk`官网;http://hg.openjdk.java.net/，选择要下载的版本，本文以jdk8为例。![image-20200525150342334](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200525150342334.png)

## 2、选择jdk版本

从上图中选择jdk8，点击进去

![image-20200525150540281](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200525150540281.png)

```shell
openjdk
—— corba：不流行的多语言、分布式通讯接口
—— hotspot：Java 虚拟机
—— jaxp：XML 处理
—— jaxws：一组 XML web services 的 Java API
—— jdk：java 开发工具包
—— —— 针对操作系统的部分
—— —— share：与平台无关的实现
—— langtools：Java 语言工具
—— nashorn：JVM 上的 JavaScript 运行时
```

## 3、选择Hotspot

![image-20200525151324837](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200525151324837.png)

4、点击browse

![image-20200525151402564](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200525151402564.png)

通过上图可以看到目录，下面是展示的目录的具体含义：

```shell
├─agent                            Serviceability Agent的客户端实现
├─make                             用来build出HotSpot的各种配置文件
├─src                              HotSpot VM的源代码
│  ├─cpu                            CPU相关代码（汇编器、模板解释器、ad文件、部分runtime函数在这里实现）
│  ├─os                             操作系相关代码
│  ├─os_cpu                         操作系统+CPU的组合相关的代码
│  └─share                          平台无关的共通代码
│      ├─tools                        工具
│      │  ├─hsdis                      反汇编插件
│      │  ├─IdealGraphVisualizer       将server编译器的中间代码可视化的工具
│      │  ├─launcher                   启动程序“java”
│      │  ├─LogCompilation             将-XX:+LogCompilation输出的日志（hotspot.log）整理成更容易阅读的格式的工具
│      │  └─ProjectCreator             生成Visual Studio的project文件的工具
│      └─vm                           HotSpot VM的核心代码
│          ├─adlc                       平台描述文件（上面的cpu或os_cpu里的*.ad文件）的编译器
│          ├─asm                        汇编器接口
│          ├─c1                         client编译器（又称“C1”）
│          ├─ci                         动态编译器的公共服务/从动态编译器到VM的接口
│          ├─classfile                  类文件的处理（包括类加载和系统符号表等）
│          ├─code                       动态生成的代码的管理
│          ├─compiler                   从VM调用动态编译器的接口
│          ├─gc_implementation          GC的实现
│          │  ├─concurrentMarkSweep      Concurrent Mark Sweep GC的实现
│          │  ├─g1                       Garbage-First GC的实现（不使用老的分代式GC框架）
│          │  ├─parallelScavenge         ParallelScavenge GC的实现（server VM默认，不使用老的分代式GC框架）
│          │  ├─parNew                   ParNew GC的实现
│          │  └─shared                   GC的共通实现
│          ├─gc_interface               GC的接口
│          ├─interpreter                解释器，包括“模板解释器”（官方版在用）和“C++解释器”（官方版不在用）
│          ├─libadt                     一些抽象数据结构
│          ├─memory                     内存管理相关（老的分代式GC框架也在这里）
│          ├─oops                       HotSpot VM的对象系统的实现
│          ├─opto                       server编译器（又称“C2”或“Opto”）
│          ├─prims                      HotSpot VM的对外接口，包括部分标准库的native部分和JVMTI实现
│          ├─runtime                    运行时支持库（包括线程管理、编译器调度、锁、反射等）
│          ├─services                   主要是用来支持JMX之类的管理功能的接口
│          ├─shark                      基于LLVM的JIT编译器（官方版里没有使用）
│          └─utilities                  一些基本的工具类
└─test                             单元测试
```

4、下载

点击左侧“zip”进行下载。

![image-20200525151825116](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200525151825116.png)

因为是国外网站，有时候下载可能遇到问题，如果下载不下来，可以到我提供的网盘进行下载：

链接：https://pan.baidu.com/s/1ogmNSkncUFVbd8ig3dtAcA 
提取码：90x3

