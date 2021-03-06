# JVM内存模型

## 概述

java虚拟机的内存结构可以分为公有和私有两部分

公有指的是所有线程都共享的部分，指的是  Java堆、方法区、常量池

私有指的是每个线程的私有数据，包括：PC寄存器、JAVA虚拟机栈，本地方法栈

## JAVA虚拟机内存结构

JAVA的JVM内存分为：**堆、栈、方法区、本地方法栈、程序计数器**

1、程序计数器

​		内存空间小，线程私有。字节码解释器工作就是通过改变这个计数器的值来选取下一条需要执行指令的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖计数器完成

​		如果线程正在执行一个JAVA方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址；如果正在执行的时Native方法，这个计数器的值则为Undefined，此内存区域是唯一一个在Java虚拟机规范中没有规定任何`OutOfMemoryError`情况的区域



2、Java虚拟机栈

​		线程私有，生命周期和线程一致。描述的是Java方法执行的内存模型：每个方法在执行时都会创建一个栈帧用于存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用直至执行结束，就对应着一个栈帧从虚拟机栈中入栈到出栈的过程。

​		局部变量表：存放了编译期可知的各种基本类型、对象引用和`returnAddress`类型。

​		`StackOverflowError`：线程请求的栈深度大于虚拟机所允许的深度

​		`OutOfMemoryError`：如果虚拟机栈可以动态扩展，而扩展时无法申请到足够的内存，



3、本地方法栈

​		区别于Java虚拟机栈的是，Java虚拟机栈为虚拟机执行Java方法服务，而本地方法栈则为虚拟机使用到的Native方法服务。也会有`StackOverflowError和OutOfMemoryError`异常。



4、Java堆

​		对于绝大多数应用来说，这块区域是JVM所管理的内存中最大的一块，线程共享，主要是存放对象实例和数组。内部会划分出多个线程私有的分配缓冲区。可以位于物理上不连续的空间，但是逻辑上要连续。

​		`OutOfMemoryError`:如果堆中没有内存完成实力分配，并且堆也无法再扩展时，抛出该异常



5、方法区

​		属于共享内存区域，存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。



## JAVA对象的创建

类加载检查-->分配内存-->初始化零值-->设置对象头-->执行init方法

1、类加载检查：虚拟机遇到一条new指令时，先检查这个指令的参数能否在常量池中定位到一个类的符号引用，并检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有，则先进行类的加载过程。

2、分配内存：有两种方式

​		指针碰撞：假设Java堆中的内存是规整的，用过的内存在一边，空闲的在另一边，中间有一个指针多为分界点的指示器，所分配的内存就把那个指针向空闲那边挪动一段与对象大小相等的距离。

​		空闲列表：如果java堆中的内存不是规整的，虚拟机必须维护一个列表，记录哪些内存块是可用的，分配是从列表中找到一块足够大的空间划分给对象，并更新列表的记录。

​		在划分可用空间时，会遇到线程安全问题。解决这个问题有两种方案，第一种：对分配内存空间的动作进行同步处理--虚拟机采用CAS配上失败重试的方式保证更新操作的原子性。另一种是把内存分配的动作安装现场划分在不同的空间之中进行，即每个线程在java堆中预先分配一小块内存，称为本地线程分配缓冲（TLAB），哪个线程需要分配内存，就在哪个线程的TLAB上分配，只有TLAB用完并分配新的TLAB时，才需要同步锁定，是否使用TLAB，-XX：+UseTLAB参数来设定。

3、初始化零值：将分配到的内存空间都初始化为零值，如果用TLAB，则在TLAB分配时初始化为零值。

4、设置对象头：主要设置类的元数据信息、对象的哈希码、对象的GC分代年龄等信息。

5、执行init方法初始化。



## 对象的内存布局

在`HotSpot`虚拟机中，分为三块区域：对象头、实例数据和对其填充

​		对象头(Header)：包含两部分，第一部分用于存储对象自身的运行时数据，如哈希码、GC 分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等，32 位虚拟机占 32 bit，64 位虚拟机占 64 bit。官方称为 ‘Mark Word’。第二部分是类型指针，即对象指向它的类的元数据指针，虚拟机通过这个指针确定这个对象是哪个类的实例。另外，如果是 Java 数组，对象头中还必须有一块用于记录数组长度的数据，因为普通对象可以通过 Java 对象元数据确定大小，而数组对象不可以。

​		实例数据(Instance Data)：程序代码中所定义的各种类型的字段内容(包含父类继承下来的和子类中定义的)。

​		对齐填充(Padding)：不是必然需要，主要是占位，保证对象大小是某个字节的整数倍。



## 对象的访问定位

使用对象时，通过栈上的reference数据来操作堆上的具体对象。

1、通过句柄访问

​		java堆中会分配一块内存作为句柄池，`reference`存储的是句柄地址。

2、使用直接指针访问

​		`reference`中直接存储对象地址



# 内存分配策略

## 判断对象存活的算法

1、引用计数算法

（1）概念：给对象中添加一个引用计数器每当有一个地方引用它时，计数器值加1；当引用失效时，计数器就减1；任何时刻计数器为0的对象就是不可能再被使用的。

（2）java虚拟机里面没有选用引用计数算法来管理内存，其中最主要的原因是它很难解决对象之间相互循环引用的问题。

2、可达性分析算法

（1）概念：通过一系列的成为“GC Roots”的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连（用图论的话来说，就是从GC Roots到这个对象不可达）时，则证明此对象是不可用的。

 

## finalize方法

1、进行垃圾回收前可以调用的方法（有没有必要调用）。

2、可以进行一次自我拯救，在垃圾回收之前关联某一个对象就可以，但是只能一次，因为finalize方法只能被系统自动调用一次。

3、卡大型分析算法中不可达对象，是一个“缓刑”阶段。要真正宣告一个对象死亡，至少要经历两次标记过程。

 

## 垃圾收集算法

1、标记-清除算法：

（1）概念：算法氛围“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。

（2）不足：一是效率问题，标记和清除两个过程的效率都不高；另一个是空间问题，标记清除之后会产生大量不连续的内存碎片，空间碎片太多可能会导致以后在程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

 

2、复制算法

（1）概念：将可用内存按照容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。这样使得每次都是对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，只要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效。

（2）不足：将内存缩小为了原来的一半，未免太高了一点。



3、标记-整理算法

（1）概念：标记出所有需要回收的对象，然后清理掉，让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

（2）应对老年代的特点提出，应对，被使用的内存中所有对象都100%存活的极端情况。



4、分代收集算法

（1）概念：根据对象存活周期的不同将内存划分为几块。一般是把java堆氛围新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率高，没有额外空间对他进行分配担保，就必须使用标记-清理或者标记整理算法来进行回收。



## 内存分配与回收策略

1、对象优先在Eden分配

​		大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够空间进行分配时，虚拟机将发起一起Minor GC。

​		注：

​		（1）新生代GC（Minor GC）：指发生在新生代的垃圾收集动作，因为java对象大多都具备朝生夕灭的特性，所以Minor GC非常频繁，一般回收速度也比较快。

​		（2）老年代GC（Major GC/Full GC）：指发生在老年代的GC，出现了Major GC，经常会伴随至少一次的Minor GC（但非绝对的，在Parallel Scavenge收集器的收集策略里就有直接进行Major GC 的策略选择过程）。Major GC的速度一般会比Minor GC慢10倍以上。

2、大对象直接进入老年代

​		所谓的大对象就是需要大量连续内存空间的Java对象，最典型的大对象就是那种很长的字符串以及数组。

这样做的目的是避免在Eden区及两个Survivor区之间发生大量的内存复制。

3、长期存活的对象将进入老年代

​		虚拟机给每个对象定义了一个对象年龄（Age）计数器。

4、动态对象年龄判定

​		为了能更好地适应不同程序的内存状况，虚拟机并不是永远地要求对象的年龄必须达到了MaxTenuringThreshold才能晋升老年代，如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一般，年龄大于或等于改年龄的对象就可以直接进入老年代，无需等到MaxTenuringThreshold中要求的年龄.

5、空间分配担保

​		只要老年代的连续空间大于新生代对象总大小或者历次晋升的平均大小就会进行Minor GC，否则将进行Full GC。



# 垃圾回收器

## serial收集器

​		这是一个单线程收集器。意味着它只会使用一个 CPU 或一条收集线程去完成收集工作，并且在进行垃圾回收时必须暂停其它所有的工作线程直到收集结束。

1、特点

   针对新生代；

   采用复制算法；

   单线程收集；

​    进行垃圾收集时，必须暂停所有工作线程，直到完成；      

​    即会"Stop The World"；

   Serial/Serial Old组合收集器运行示意图如下：

2、应用场景

   依然是HotSpot在Client模式下默认的新生代收集器；

   也有优于其他收集器的地方：

>    简单高效（与其他收集器的单线程相比）；
>
>    对于限定单个CPU的环境来说，Serial收集器没有线程交互（切换）开销，可以获得最高的单线程收集效率；
>
>    在用户的桌面应用场景中，可用内存一般不大（几十M至一两百M），可以在较短时间内完成垃圾收集（几十MS至一百多MS）,只要不频繁发生，这是可以接受的

3、设置参数

   "-XX:+UseSerialGC"：添加该参数来显式的使用串行垃圾收集器；

4、Stop TheWorld说明

   JVM在后台自动发起和自动完成的，在用户不可见的情况下，把用户正常的工作线程全部停掉，即GC停顿；

   会带给用户不良的体验；

## ParNew收集器

ParNew垃圾收集器是Serial收集器的多线程版本。

1、特点

   除了多线程外，其余的行为、特点和Serial收集器一样；

   如Serial收集器可用控制参数、收集算法、Stop The World、内存分配规则、回收策略等；

   两个收集器共用了不少代码；

   ParNew/Serial Old组合收集器运行示意图如下：

2、应用场景

   在Server模式下，ParNew收集器是一个非常重要的收集器，因为除Serial外，目前只有它能与CMS收集器配合工作；

   但在单个CPU环境中，不会比Serail收集器有更好的效果，因为存在线程交互开销。

3、设置参数

   "-XX:+UseConcMarkSweepGC"：指定使用CMS后，会默认使用ParNew作为新生代收集器；

   "-XX:+UseParNewGC"：强制指定使用ParNew；  

   "-XX:ParallelGCThreads"：指定垃圾收集的线程数量，ParNew默认开启的收集线程与CPU的数量相同；

4、为什么只有ParNew能与CMS收集器配合

   CMS是HotSpot在JDK1.5推出的第一款真正意义上的并发（Concurrent）收集器，第一次实现了让垃圾收集线程与用户线程（基本上）同时工作；

   CMS作为老年代收集器，但却无法与JDK1.4已经存在的新生代收集器Parallel Scavenge配合工作；

   因为Parallel Scavenge（以及G1）都没有使用传统的GC收集器代码框架，而另外独立实现；而其余几种收集器则共用了部分的框架代码；

   关于CMS收集器后面会详细介绍。

## Parallel Scavenge收集器

Parallel Scavenge垃圾收集器因为与吞吐量关系密切，也称为吞吐量收集器（Throughput Collector）。

1、特点

> （A）、有一些特点与ParNew收集器相似
>
>    新生代收集器；
>
>    采用复制算法；
>
>    多线程收集；
>
> （B）、主要特点是：它的关注点与其他收集器不同
>
>    CMS等收集器的关注点是尽可能地缩短垃圾收集时用户线程的停顿时间；
>
>    而Parallel Scavenge收集器的目标则是达一个可控制的吞吐量（Throughput）；
>
>    关于吞吐量与收集器关注点说明详见本节后面；

> 2、应用场景
>
>    高吞吐量为目标，即减少垃圾收集时间，让用户代码获得更长的运行时间；
>
>    当应用程序运行在具有多个CPU上，对暂停时间没有特别高的要求时，即程序主要在后台进行计算，而不需要与用户进行太多交互；
>
>    例如，那些执行批量处理、订单处理、工资支付、科学计算的应用程序；
>
> 3、设置参数
>
>    Parallel Scavenge收集器提供两个参数用于精确控制吞吐量：
>
> > （A）、"-XX:MaxGCPauseMillis"
> >
> >    控制最大垃圾收集停顿时间，大于0的毫秒数；
> >
> >    MaxGCPauseMillis设置得稍小，停顿时间可能会缩短，但也可能会使得吞吐量下降；
> >
> >    因为可能导致垃圾收集发生得更频繁；
> >
> > （B）、"-XX:GCTimeRatio"
> >
> >    设置垃圾收集时间占总时间的比率，0<n<100的整数；
> >
> >    GCTimeRatio相当于设置吞吐量大小；
> >
> >    垃圾收集执行时间占应用程序执行时间的比例的计算方法是：
> >
> > >    1 / (1 + n)
> > >
> > >    例如，选项-XX:GCTimeRatio=19，设置了垃圾收集时间占总时间的5%--1/(1+19)；
> >
> >    默认值是1%--1/(1+99)，即n=99；

> > 垃圾收集所花费的时间是年轻一代和老年代收集的总时间；
> >
> > 如果没有满足吞吐量目标，则增加代的内存大小以尽量增加用户程序运行的时间；
>
>    此外，还有一个值得关注的参数：

> > （C）、"-XX:+UseAdptiveSizePolicy"
> >
> >    开启这个参数后，就不用手工指定一些细节参数，如：
> >
> > >    新生代的大小（-Xmn）、Eden与Survivor区的比例（-XX:SurvivorRation）、晋升老年代的对象年龄（-XX:PretenureSizeThreshold）等；
> >
> >    JVM会根据当前系统运行情况收集性能监控信息，动态调整这些参数，以提供最合适的停顿时间或最大的吞吐量，这种调节方式称为GC自适应的调节策略（GC Ergonomiscs）；  

> >    这是一种值得推荐的方式：
> >
> > >    (1)、只需设置好内存数据大小（如"-Xmx"设置最大堆）；
> > >
> > >    (2)、然后使用"-XX:MaxGCPauseMillis"或"-XX:GCTimeRatio"给JVM设置一个优化目标；
> > >
> > >    (3)、那些具体细节参数的调节就由JVM自适应完成；    
> >
> >    这也是Parallel Scavenge收集器与ParNew收集器一个重要区别； 

## Serial Old收集器

 Serial Old是 Serial收集器的老年代版本；

1、特点

   针对老年代；

   采用"标记-整理"算法（还有压缩，Mark-Sweep-Compact）；

   单线程收集；

   Serial/Serial Old收集器运行示意图如下：

2、应用场景

   主要用于Client模式；

   而在Server模式有两大用途：

>    （A）、在JDK1.5及之前，与Parallel Scavenge收集器搭配使用（JDK1.6有Parallel Old收集器可搭配）；
>
>    （B）、作为CMS收集器的后备预案，在并发收集发生Concurrent Mode Failure时使用（后面详解）；

## Parallel Old收集器

Parallel Old垃圾收集器是Parallel Scavenge收集器的老年代版本；

   JDK1.6中才开始提供；

1、特点

   针对老年代；

   采用"标记-整理"算法；

   多线程收集；

   Parallel Scavenge/Parallel Old收集器运行示意图如下：

2、应用场景

   JDK1.6及之后用来代替老年代的Serial Old收集器；

   特别是在Server模式，多CPU的情况下；

   这样在注重吞吐量以及CPU资源敏感的场景，就有了Parallel Scavenge加Parallel Old收集器的"给力"应用组合；

3、设置参数

   "-XX:+UseParallelOldGC"：指定使用Parallel Old收集器；

## CMS收集器

> 并发标记清理（Concurrent Mark Sweep，CMS）收集器也称为并发低停顿收集器（Concurrent Low Pause Collector）或低延迟（low-latency）垃圾收集器；
>
>    在前面ParNew收集器曾简单介绍过其特点；
>
> 1、特点
>
>    针对老年代；
>
>    基于"标记-清除"算法(不进行压缩操作，产生内存碎片)；      
>
>    以获取最短回收停顿时间为目标；
>
>    并发收集、低停顿；
>
>    需要更多的内存（看后面的缺点）；
>
> ​      
>
>    是HotSpot在JDK1.5推出的第一款真正意义上的并发（Concurrent）收集器；
>
>    第一次实现了让垃圾收集线程与用户线程（基本上）同时工作；
>
> 2、应用场景
>
>    与用户交互较多的场景；    
>
>    希望系统停顿时间最短，注重服务的响应速度；
>
>    以给用户带来较好的体验；
>
>    如常见WEB、B/S系统的服务器上的应用；

> 3、设置参数
>
>    "-XX:+UseConcMarkSweepGC"：指定使用CMS收集器；
>
> 4、CMS收集器运作过程
>
>    比前面几种收集器更复杂，可以分为4个步骤:

> > （A）、初始标记（CMS initial mark）
> >
> >    仅标记一下GC Roots能直接关联到的对象；
> >
> >    速度很快；
> >
> >    但需要"Stop The World"；
> >
> > （B）、并发标记（CMS concurrent mark）
> >
> >    进行GC Roots Tracing的过程；
> >
> >    刚才产生的集合中标记出存活对象；
> >
> >    应用程序也在运行；
> >
> >    并不能保证可以标记出所有的存活对象；
> >
> > （C）、重新标记（CMS remark）
> >
> >    为了修正并发标记期间因用户程序继续运作而导致标记变动的那一部分对象的标记记录；
> >
> >    需要"Stop The World"，且停顿时间比初始标记稍长，但远比并发标记短；
> >
> >    采用多线程并行执行来提升效率；
> >
> > （D）、并发清除（CMS concurrent sweep）
> >
> >    回收所有的垃圾对象；

>    整个过程中耗时最长的并发标记和并发清除都可以与用户线程一起工作；
>
>    所以总体上说，CMS收集器的内存回收过程与用户线程一起并发执行；
>
>    CMS收集器运行示意图如下：

5、CMS收集器3个明显的缺点

​           （A）、对CPU资源非常敏感

> >    并发收集虽然不会暂停用户线程，但因为占用一部分CPU资源，还是会导致应用程序变慢，总吞吐量降低。
> >
> >    CMS的默认收集线程数量是=(CPU数量+3)/4；
> >
> >    当CPU数量多于4个，收集线程占用的CPU资源多于25%，对用户程序影响可能较大；不足4个时，影响更大，可能无法接受。
> >
> >  
> >
> >    增量式并发收集器：
> >
> > >    针对这种情况，曾出现了"增量式并发收集器"（Incremental Concurrent Mark Sweep/i-CMS）；
> > >
> > >    类似使用抢占式来模拟多任务机制的思想，让收集线程和用户线程交替运行，减少收集线程运行时间；
> > >
> > >    但效果并不理想，JDK1.6后就官方不再提倡用户使用。

> > B）、无法处理浮动垃圾,可能出现"Concurrent Mode Failure"失败

> > > （1）、浮动垃圾（Floating Garbage）
> > >
> > >    在并发清除时，用户线程新产生的垃圾，称为浮动垃圾；
> > >
> > >    这使得并发清除时需要预留一定的内存空间，不能像其他收集器在老年代几乎填满再进行收集；
> > >
> > >    也要可以认为CMS所需要的空间比其他垃圾收集器大；
> > >
> > >    "-XX:CMSInitiatingOccupancyFraction"：设置CMS预留内存空间；
> > >
> > >    JDK1.5默认值为68%；
> > >
> > >    JDK1.6变为大约92%；        

> > > （2）、"Concurrent Mode Failure"失败
> > >
> > >    如果CMS预留内存空间无法满足程序需要，就会出现一次"Concurrent Mode Failure"失败；
> > >
> > >    这时JVM启用后备预案：临时启用Serail Old收集器，而导致另一次Full GC的产生；
> > >
> > >    这样的代价是很大的，所以CMSInitiatingOccupancyFraction不能设置得太大。

> > （C）、产生大量内存碎片
> >
> >    由于CMS基于"标记-清除"算法，清除后不进行压缩操作；
> >
> >    前面[《Java虚拟机垃圾回收(二) 垃圾回收算法》](http://blog.csdn.net/tjiyu/article/details/53983064)"标记-清除"算法介绍时曾说过：
> >
> > >    产生大量不连续的内存碎片会导致分配大内存对象时，无法找到足够的连续内存，从而需要提前触发另一次Full GC动作。
> >
> >    解决方法：        
> >
> > > （1）、"-XX:+UseCMSCompactAtFullCollection"
> > >
> > >    使得CMS出现上面这种情况时不进行Full GC，而开启内存碎片的合并整理过程；
> > >
> > >    但合并整理过程无法并发，停顿时间会变长；
> > >
> > >    默认开启（但不会进行，结合下面的CMSFullGCsBeforeCompaction）；
> > >
> > > （2）、"-XX:+CMSFullGCsBeforeCompaction"
> > >
> > >    设置执行多少次不压缩的Full GC后，来一次压缩整理；
> > >
> > >    为减少合并整理过程的停顿时间；
> > >
> > >    默认为0，也就是说每次都执行Full GC，不会进行压缩整理；

> >    由于空间不再连续，CMS需要使用可用"空闲列表"内存分配方式，这比简单实用"碰撞指针"分配内存消耗大；
> >
> >    更多关于内存分配方式请参考：《[Java对象在Java虚拟机中的创建过程](http://blog.csdn.net/tjiyu/article/details/53923392)》
>
>    总体来看，与Parallel Old垃圾收集器相比，CMS减少了执行老年代垃圾收集时应用暂停的时间；
>
>    但却增加了新生代垃圾收集时应用暂停的时间、降低了吞吐量而且需要占用更大的堆空间；

## G1收集器

> G1（Garbage-First）是JDK7-u4才推出商用的收集器；
>
> 1、特点

> > （A）、并行与并发
> >
> >    能充分利用多CPU、多核环境下的硬件优势；
> >
> >    可以并行来缩短"Stop The World"停顿时间；
> >
> >    也可以并发让垃圾收集与用户程序同时进行；
> >
> > （B）、分代收集，收集范围包括新生代和老年代  
> >
> >    能独立管理整个GC堆（新生代和老年代），而不需要与其他收集器搭配；
> >
> >    能够采用不同方式处理不同时期的对象；
> >
> > ​        
> >
> >    虽然保留分代概念，但Java堆的内存布局有很大差别；
> >
> >    将整个堆划分为多个大小相等的独立区域（Region）；
> >
> >    新生代和老年代不再是物理隔离，它们都是一部分Region（不需要连续）的集合；
> >
> >    更多G1内存布局信息请参考：
> >
> > >    《垃圾收集调优指南》 9节：[http://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html#garbage_first_garbage_collection](http://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc.html%23garbage_first_garbage_collection)

> > （C）、结合多种垃圾收集算法，空间整合，不产生碎片
> >
> >    从整体看，是基于标记-整理算法；
> >
> >    从局部（两个Region间）看，是基于复制算法；
> >
> >    这是一种类似火车算法的实现；
> >
> >  
> >
> >    都不会产生内存碎片，有利于长时间运行；
> >
> > （D）、可预测的停顿：低停顿的同时实现高吞吐量
> >
> >    G1除了追求低停顿处，还能建立可预测的停顿时间模型；
> >
> >    可以明确指定M毫秒时间片内，垃圾收集消耗的时间不超过N毫秒；

> 2、应用场景
>
>    面向服务端应用，针对具有大内存、多处理器的机器；
>
>    最主要的应用是为需要低GC延迟，并具有大堆的应用程序提供解决方案；
>
>    如：在堆大小约6GB或更大时，可预测的暂停时间可以低于0.5秒；
>
> ​      
>
>    用来替换掉JDK1.5中的CMS收集器；
>
>    在下面的情况时，使用G1可能比CMS好：
>
> >    （1）、超过50％的Java堆被活动数据占用；
> >
> >    （2）、对象分配频率或年代提升频率变化很大；
> >
> >    （3）、GC停顿时间过长（长于0.5至1秒）。

>    是否一定采用G1呢？也未必：
>
> >    如果现在采用的收集器没有出现问题，不用急着去选择G1；
> >
> >    如果应用程序追求低停顿，可以尝试选择G1；
> >
> >    是否代替CMS需要实际场景测试才知道。

> 3、设置参数
>
>    "-XX:+UseG1GC"：指定使用G1收集器；
>
>    "-XX:InitiatingHeapOccupancyPercent"：当整个Java堆的占用率达到参数值时，开始并发标记阶段；默认为45；
>
>    "-XX:MaxGCPauseMillis"：为G1设置暂停时间目标，默认值为200毫秒；
>
>    "-XX:G1HeapRegionSize"：设置每个Region大小，范围1MB到32MB；目标是在最小Java堆时可以拥有约2048个Region；
>
>    更多关于G1参数设置请参考：
>
> >    《垃圾收集调优指南》 10.5节：[http://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html#important_defaults](http://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/g1_gc_tuning.html%23important_defaults)
>
> 4、为什么G1收集器可以实现可预测的停顿
>
>    G1可以建立可预测的停顿时间模型，是因为：

> >    可以有计划地避免在Java堆的进行全区域的垃圾收集；
> >
> >    G1跟踪各个Region获得其收集价值大小，在后台维护一个优先列表；
> >
> >    每次根据允许的收集时间，优先回收价值最大的Region（名称Garbage-First的由来）；
> >
> >    这就保证了在有限的时间内可以获取尽可能高的收集效率；

> 5、一个对象被不同区域引用的问题
>
>    一个Region不可能是孤立的，一个Region中的对象可能被其他任意Region中对象引用，判断对象存活时，是否需要扫描整个Java堆才能保证准确？
>
>    在其他的分代收集器，也存在这样的问题（而G1更突出）：
>
> >    回收新生代也不得不同时扫描老年代？
>
>    这样的话会降低Minor GC的效率；
>
>    解决方法：

> >    无论G1还是其他分代收集器，JVM都是使用Remembered Set来避免全局扫描：
> >
> > >    每个Region都有一个对应的Remembered Set；
> > >
> > >    每次Reference类型数据写操作时，都会产生一个Write Barrier暂时中断操作；
> > >
> > >    然后检查将要写入的引用指向的对象是否和该Reference类型数据在不同的Region（其他收集器：检查老年代对象是否引用了新生代对象）；
> > >
> > >    如果不同，通过CardTable把相关引用信息记录到引用指向对象的所在Region对应的Remembered Set中；
> > >
> > > ​          
> >
> > >    当进行垃圾收集时，在GC根节点的枚举范围加入Remembered Set；
> > >
> > >    就可以保证不进行全局扫描，也不会有遗漏。

> 6、G1收集器运作过程
>
>    不计算维护Remembered Set的操作，可以分为4个步骤（与CMS较为相似）。

> > （A）、初始标记（Initial Marking）
> >
> >    仅标记一下GC Roots能直接关联到的对象；
> >
> >    且修改TAMS（Next Top at Mark Start）,让下一阶段并发运行时，用户程序能在正确可用的Region中创建新对象；
> >
> >    需要"Stop The World"，但速度很快；
> >
> > （B）、并发标记（Concurrent Marking）
> >
> >    进行GC Roots Tracing的过程；
> >
> >    刚才产生的集合中标记出存活对象；
> >
> >    耗时较长，但应用程序也在运行；
> >
> >    并不能保证可以标记出所有的存活对象；
> >
> > （C）、最终标记（Final Marking）
> >
> >    为了修正并发标记期间因用户程序继续运作而导致标记变动的那一部分对象的标记记录；
> >
> >    上一阶段对象的变化记录在线程的Remembered Set Log；
> >
> >    这里把Remembered Set Log合并到Remembered Set中；
> >
> > ​          
> >
> >    需要"Stop The World"，且停顿时间比初始标记稍长，但远比并发标记短；
> >
> >    采用多线程并行执行来提升效率；
> >
> > （D）、筛选回收（Live Data Counting and Evacuation）
> >
> >    首先排序各个Region的回收价值和成本；
> >
> >    然后根据用户期望的GC停顿时间来制定回收计划；
> >
> >    最后按计划回收一些价值高的Region中垃圾对象；
> >
> > ​          
> >
> >    回收时采用"复制"算法，从一个或多个Region复制存活对象到堆上的另一个空的Region，并且在此过程中压缩和释放内存；
> >
> >    可以并发进行，降低停顿时间，并增加吞吐量；