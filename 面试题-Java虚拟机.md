
# 前言
最近在看周志明的《深入理解Java虚拟机》，想写个笔记便于后期面试复习，希望对大家有所帮助...

# 内存区域与内存溢出

### 1.Java内存区域是如何划分的？
```
运行时数据区：
线程共享：方法区、堆
线程私有：虚拟机栈  本地方法栈  程序计数器

堆内存：Java世界几乎所有的对象和数组都储存在这里。
划分出多个线程私有的分配缓冲区thread local allocation buffer以提升对象分配时的效率。
通过-Xmx -Xms设定大小
```
### 2.Java方法区是什么，新jdk版本有何不同？
```
虚拟机规范中把方法区作为堆的 一个逻辑部分。储存类信息（版本字段方法接口）、常量、静态变量、代码缓存
jdk 6开始使用本地内存实现方法区。

直接内存：jdk1.4中有了new IO,使用channel +buffer的IO方式分配 直接内存。然后通过一个储存在java堆的 DirectByteBuffer作为这块内存的引用操作这块内存，省去讲数据复制到堆内存的消耗。
```
### 3.说一下创建对象的流程，是怎么分配内存的？
```
当虚拟机遇到一个new 关键词（还有复制/反序列化）时，
- 先去方法区检查这个类是否已经被加载解析初始化、如果没有则执行类加载过程
- 在堆分配内存的时候，如果是带整理功能的垃圾回收期(PerNew)，采用指针碰撞，也就是指针往空闲的方向挪动一定距离，留出对象需要的空间即可。如果是CMS等就要采用 空闲列表的方式了。。
- 并发分配内存的线程安全问题：
	1.虚拟机采用CAS保证更新指针的原子性
	2.上面提到的线程私有的分配缓冲区thread local allocation buffer以提升对象分配效率
	可以使用 -XX:+/-UseTLAB设定
- 设置对象头信息：GC分代年龄、hash、偏向锁等
- new指令后面还会跟上invokespecial 执行<init>()方法，构造方法。

ps:对象类型数据（方法区） 和 实例数据（堆内存）分开存储，需要分别访问
```

### 4.有没有遇到过OOM，你是如何处理的？
```
1.堆内存OOM
    设置 -XX:+HeapDumpOnOutOfMemoryError 转储内存快照
    通过内存分析工具（Eclipse Memory Analyzer，还有在线的）进行分析
    1.1 堆内存空间不足，优化代码，扩大阈值
    1.2 内存泄露，查看泄露对象的GCRoot引用链
2.栈内存OOM
	除非在线程申请内存时可能出现OOM，运行时不会出现。
	-Xss 228k设置栈内存大小，比如win32最多给虚拟机进程分配 2G的内存，2G内存减去堆内存和方法区，剩下的就是本地方法栈和 虚拟机栈，如果为每个线程分配的栈内存大，那么可以创建的线程数量就要减少，所以减少堆内存可以增加栈内存。
	栈的深度一般是 1000~2000
3.方法区（运行时常量池是方法区的一部分）OOM
	String::intern()方法可以将对象包含的字符串添加到常量池。下面会有
	JDK6:-XX:PermSize  -XX:MaxPermSize限制永久代的大小(间接限制常量池)
	JDK7:运行时常量池被移动到堆内存了，只收到堆内存的限制，OOM的时候直接提示 堆内存OOM...方法区OOM最大可能是用了很多cglib运行时生成了大量的类。
    JDK8:元空间存了类信息。-XX:MetaspaceSize元空间的初始大小，达到这个值就开始GC卸载类。-XXMaxMetaspaceSize最大值，一般是-1，只限制与本地内存。
    -XX:+TraceClassUnloading查看类的加载和卸载信息
4.直接内存OOM
-XX:MaxDirectMemorySize 指定。默认和堆（Xmx）设定的值相同。当用DirectByteBuffer(NIO中的类)或者Unsafe.allocateMemory申请内存时，发现超过了这个值就会抛出异常。
现象一般就是oom拿到的dump文件很小，程序又使用了NIO。
可以考虑这方面排查，后面有案例
```
### 5.你们的堆内存设置多大，为什么要这么设置
```
-Xms2g -Xmx2g -XX:MaxDirectMemorySize=1024M  
堆的最大值和最小值设置为一样可以避免堆内存自动扩展，扩容一次大约消耗0.1s
设置的过大会增加GC STW的时间。
```
### 6.String "" 和String.valueOf() 创建对象有什么不同
```
https://tech.meituan.com/2014/03/06/in-depth-understanding-string-intern.html
String类型的常量池比较特殊。它的主要使用方法有两种：

直接使用双引号声明出来的String对象会直接存储在常量池中。
如果不是用双引号声明的String对象，可以使用String提供的intern方法。intern 方法会从字符串常量池中查询当前字符串是否存在，若不存在就会将当前字符串放入常量池中

所以说一般在代码中通过拷贝或者数据绑定拿到的 entity对象中的string是存在堆内存的对象
```

# 垃圾回收器

### 1.如何判断对象是否可被回收？
```
1.引用计数法（几乎不用，因为循环引用的问题）
2.可达性分析算法，GCRoots不可达的对象会被垃圾回收。
也叫追踪式垃圾收集。
```
### 2.GCRoot有哪些？
```
1.虚拟机栈（本地变量表）
2.方法区中的类 信息中 的静态变量 引用的对象。
3.方法区中的常量引用的对象
4.Native方法引用的对象
5.JVM自身的基本数据类型、异常对象、类加载器
6.被Synchronized持有的对象
```
### 3.阐述一次GC的触发过程
```
1.为对象分配空间时，如果发现Eden区已经到达GC回收阈值（可设置），直接开始youngGC(minorGC)，将Eden区域和一个survivor区域（也就是90%的新生代）的可回收对象清理掉。
剩余的对象移动到另一块survivor区域，
如果放得下，则youngGC(minorGC)结束。
如果放不下，则通过 老年代 分配担保机制，直接进入老年代，
如果放得下，则GC结束。
如果放不下，则触发Old GC(majorGC),这次GC也成了FULL GC
FULL GC后如果老年代还是放不下，首先老年代扩容，扩容失败 OOM
```
### 4.介绍一下常见的垃圾回收算法 以及算法的应用。
```
标记复制
缺点：1.可用内存变为原来的一半 2.如果回收后存活的对象多，那移动对象将增加开销
优点：停顿时间比较短（仅标记过程需要STW）

标记清除
缺点：1.执行效率随着对象数量的增加而下降 2.碎片化问题，当分配大对象时又会触发GC

标记整理
将存活对象都移动到内存空间的一端，清理其他区域。


CMS就是基于标记清除算法，在内存碎片过多的时候会用标记整理 来解决碎片问题
```
### 5.为什么要使用分代收集算法？
```
基于2个假说：
1.绝大多数对象都是昭生夕灭的，新生代98%的对象熬不过第一轮GC
2.熬过多次GC的对象就越难以消亡
把容易消亡的对象集中在一起 把老年对象也集中在一起，可以以较低的代价回收大量的空间.

可能问题：跨代 且互相引，无法分代收集。解决办法就是不管那些跨带的，等他们升入老年代。
```
### 聊一下你们用的垃圾回收器
```
一般的搭配 ：
parallel scavenge + parallel old（jdk6开始有，ps的老年代版本）
parNew + CMS
G1
------------------------------------------------------------------------
parallel scavenge:吞吐量优先 -XX:MaxGCPauseMillis（最大停顿时间）  GCTimeRatio(设置吞吐量，默认99)
-XX:+UseAdaptiveSizePolicy： 新生代大小自适应，只需要设置-Xmx值，ps自动调整Eden survivor大小来达到最大停顿时间和吞吐量目标。

parNew:可以理解是CMS搭配的年轻代收集器。-XX:ParallelCGThreads限制并发线程数量，否则默认等于CPU核心数量。采用复制算法 会STW

CMS:1.初始标记（stw）  2.并发标记  3.重新标记（stw、增量更新）  4.并发清除
无法收集浮动垃圾
-XX:CMSInitiatingOcc-pancyFraction 触发垃圾收集的 堆内存百分比（默认老年代的92%）
在并发标记、清除的过程中、老年代内存不足，就会出现并发失败，这时虚拟机会STW，然后让serial Old来进行收集，增加STW时间。
空间碎片问题：-XX:CMSFullGCBeforeCompaction多次gc后进行碎片整理
------------------------------------------------------------------------
G1:Mixed GC收集整个新生代和老年的全功能垃圾回收器，jdk9默认。
把堆内存划分为大小相等的Region，每个Region可以扮演Eden survivor或者老年代
超过Region容量大小（-XX:G1HeapRegionSize设定 1MB-32MB）一半的被认为是大对象，大对象放在N个连续的 Humongous Region中。
G1维护了一个优先级列表，记录每个region回收所获得的空间以及所需时间的经验值，每次根据允许的停顿时间（MaxGCPauseMillis 默认200ms）回收最有价值的Region，单个region划分一部分作为回收过程中的新对象分配。
运行步骤：1.初始标记（minorGC时完成短暂stw）2.并发标记 3.最终标记（stw）4.筛选回收（对回收价值进行排序，选择n个region构成回收集，把存活的对象移动（复制）到空的region，清理掉旧的region的全部空间，移动过程中stw）
整体上看是标记 整理，2个region之间是标记复制。
采用增量回收缓解大内存停顿时间长的问题，但是完全解决还得看ZGC

缺点：
夸Region问题:记忆其他Region，导致G1占用heap 10~20%的内存
如果回收赶不上分配的速度，G1也会长时间STW。
如果期望停顿时间设置的太小，每次筛选出来的region集合比较少，清理跟不上分配速度导致 长时间fullGC,所以期望停顿一般设置在 200-300ms
------------------------------------------------------------------------
ZCC:将region分成 小region 中region 大region.
染色指针技术：Windowsx86-64不支持
1.并发标记  2.并发预备重分配 3.并发重分配 4.并发重映射
优势：几乎整个收集过程全程可并发，短暂停顿只与GcRoots大小相关  与堆内存大小无关，因而实现任何堆停顿时间小于10ms。吞吐量逼近parallel scavenge，停顿时间领先ps g1两个量级

缺点：没有分代，大量浮动垃圾，并发收集周期长，解决方案是增加堆内存大小，给zgc更多喘息时间
```
### 如何选择垃圾回收器，选择的经验？

```
垃圾回收器的三个指标：内存占用、延迟、吞吐量。

选择：能用新版本jdk 能用ZGC,就用ZGC,延迟低
小内存（4-6G）上CMS优于G1。6-8G以上选择G1.因为G1维护的优先集合、夸region卡表都要消耗更多内存，且也更消耗计算资源

记录好虚拟机和垃圾回收器日志
-Xlog:gc*(jdk9之后  9之前是 -XX:+PrintGC)
查看GC前后堆的变化 9之前是 -XX:+PrintHeapAtGC jdk9之后是-Xlog:gc+heap =debug:
查看停顿时间：-XX:+PrintGCApplicationStoppedTime  jdk9之后是 -Xlog:safepoint

permanent :永久代
def new :年轻代
```

### 为什么会发生stop the world(STW)?
```
可达性分析算法 耗时最长的查找引用链可以与用户线程并发执行。
但是根节点（GCRoots）的枚举过程中必须要STW，否则无法保证准确性，引用关系会不断变化。

引用链变化导致1.浮动垃圾，该标记的没有标记到
2.对象消失，不该标记的，被标记回收

线程会在执行过程中不停的去轮训 垃圾收集 标志位，一但发现标志位为真，就找到最近的安全点主动挂起。
一般执行native代码的线程不会被block，因为执行native代码不会改变对象的引用关系。
```

# 虚拟机监控和Debug工具
### jps JVM Process Status
```
jps -lv(l:主类全程  v虚拟机参数)
```
### jstat JVM statistics  monitoring tool
```
jstat -gc jps-id  250 20 每隔250ms查询一次  一共查询20次
-class查看类装载 
-gcutil查看垃圾回收信息 详见书本 142页
```
### jinfo 
```
查看配置 等于System.getProperties()
```
### jmap Memory Map for Java
```
jmap -dump :format=b,file=xxxx.bin 内存dump
```
### jstack Stack Trace for Java
```
➜  ~ jstack -l 53751(jps 获取的id)
等于代码中Thread.getAllStackTraces() 获取全部的线程堆栈

```
### jhsdb 分析对象
```

```
### jconsole
```
内存 标签  相当于 jstat 可以看GC啥的
线程 相当于  jstack

```
# 调优案例分析

### 大内存的机器频繁FullGC
```
问题：4C 16G jdk5 -Xmx 12G 单词fullGC耗时 15s
大内存的机器无法dump内存。dump产生10G的文件，分析困难

解法：尽可能保证对象不要进入老年代
最值得考虑的方案：上ZGC
或者半夜定时GC、定时重启.
拆分成多个2G内存的实例部署，且垃圾回收器改成cms，本地缓存改成集中式缓存redis

```
### 堆外内存（直接内存）OOM
```
i5+4G 32位windows 机器 -Xmx 1.6G -XX:HeapDumpOnOOM设置,但是OOM后没有产生dump文件
jstat 监控 eden区 survivor 老年代都很稳定

win32位给进程的限制是2G，分配了1.4G，则只有最大0.4G给对外内存
且堆外内存空间不足无法主动触发GC,只有在FullGC后顺便清理，可以catch 异常手动System.gc()

socket缓冲区receive占用37k send缓冲区25k。连接过多导致IOException:too many open files
```
### Runtime.getRuntime.exec()创建进程
```
dtrace发现 fork 系统调用消耗大量CPU资源

Runtime.getRuntime.exec()创建一个和当前虚拟机有一样环境变量的进程，再用这个进程去执行shell命令，然后退出进程。

应该改为使用JavaAPI 不要用exec()
```
### 服务器虚拟机进程崩溃
```
异常日志,程序进程自动关闭后留下 hs_err_pid###.log文件
java.net.SocketException:Connection reset
SocketInputStream.java:168

异步大量的rpc调用，响应时间超过3分钟，导致等待线程和Socket连接越来越多，最终虚拟机崩溃

解决：改成mq

```
### 定时加载80M的数据导致minorGC 耗时500ms
```
-Xms4g -Xms8g -Xmn1g Eden800M Survivor 100*2 Old 5.8G permanent65M
分析GC日志可知,耗时主要是minorGC后有100M对象存活，移动这部分对象负担太重，可以去掉survivor空间，让这些对象一次minorGC后直接进入老年代

提高空间利用率：存的数据是HashMap<Long,Long>
消耗的内存为(long 24byte *2 + Entry(32byte)+ hashMap Ref = 8byte = 88byte),
可以改成使用基本的数据类型。
Long对象 8字节mark Word 8字节 Klass 8字节long内容
```
### 进入安全点超时导致 用户线程长时间等待。
```
Time: user sys real user是用户态cpu时间，real是时钟时间，因为CPU核数较多，所以同时有多个cpu参与执行、cpu时间一般会数倍于 real 时间。
real是真正垃圾收集的耗时

thread were stopped 2.36478 seconds这个是用户线程停顿的时间
远大于 real时间。

回顾一下原理，GC线程想要stw，就要向所有用户线程发送block信号，原则上线程只能由自己中断不能被其他人打断。
这叫做 主动式中断，设置一个标志位，线程执行过程中不断去轮询，发现中断就主动挂起。

for (int i;)这种是可数循环，不会设置安全点
for (long i;)不可数循环，会设置安全点
所以当for(int i)执行中 开始gc，会等for(int i)执行完才能中断线程。
```
