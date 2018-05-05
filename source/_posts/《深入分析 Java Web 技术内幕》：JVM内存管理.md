---
title: 《深入分析 Java Web 技术内幕》：JVM 内存管理
date: 2017-04-28 13:14:02
tags: JVM
categories: [读书笔记, Java]
---
## 1. JVM 内存结构
在 Java 虚拟机规范中将 Java 运行时数据划分为 6 种，分别为：

- PC 寄存器数据；
- Java 栈；
- 堆；
- 方法区；
- 本地方法区；
- 运行时常量池。

<!--more-->

### 1.1 Java 栈
Java 栈总是和线程关联在一起，每当创建一个线程时，JVM 就会为这个线程创建一个对应的 Java 栈，在这个 Java 栈中又会含有多个栈帧（Frames），这些栈帧是与每个方法关联起来的，每运行一个方法就创建一个栈帧，每个栈帧会含有一些内部变量（在方法内定义的变量）、操作栈和方法返回值等信息。

每当一个方法执行完成时，这个栈帧就会弹出栈帧的元素作为这个方法的返回值，并清除这个栈帧，Java 栈的栈顶的栈帧就是当前正在执行的活动栈，也就是当前正在执行的方法，PC 寄存器也会指向这个地址。只有这个活动的栈帧的本地变量可以被操作栈使用，当在这个栈帧中调用另外一个方法时，与之对应的一个新的栈帧又被创建，这个新创建的栈帧又被放到 Java 栈的顶部，变为当前的活动栈帧。同样现在只有这个栈帧的本地变量才能被使用，当在这个栈帧中所有指令执行完成时这个栈帧移出 Java 栈，刚才的那个栈帧又变为活动栈帧，前面的栈帧的返回值又变为这个栈帧的操作栈中的一个操作数。如果前面的栈帧没有返回值，那么当前的栈帧的操作栈的操作数没有变化。

由于 Java 栈是与 Java 线程对应起来的，这个数据不是线程共享的，所以我们不用关心它的数据一致性问题，也不会存在同步锁的问题。

### 1.2 方法区
JVM 方法区是用于存储类结构信息的地方，class 文件会被 JVM 解析成几个部分，这些不同的部分在这个 class 被加载到 JVM 时，会被存储在不同的数据结构中，其中的常量池、域、方法数据、方法体、构造函数，包括类中的专用方法、实例初始化、接口初始化都存储在这个区域。

方法区这个存储区域也属于后面介绍的 Java 堆中的一部分，也就是我们通常所说的 Java 堆中的永久区。这个区域可以被所有的线程共享，并且它的大小可以通过参数来设置。

这个方法区存储区域的大小一般在程序启动后的一段时间内就是固定的了，JVM 运行一段时间后，需要加载的类通常都已经加载到 JVM 中了。但是有一种情况是需要注意的，那就是在项目中如果存在对类的动态编译，而且是同样一个类的多次编译，那么需要观察方法区的大小是否能满足类存储。

### 1.3 运行时常量池
在 JVM 规范中是这样定义运行时常量池这个数据结构的：Runtime Constant Pool 代表运行时每个 class 文件中的常量表。它包括几种常量：编译期的数字常量、方法或者域的引用（在运行时解析）。Runtime Constant Pool 的功能类似于传统编程语言的符号表，尽管它包含的数据比典型的符号表要丰富得多。每个Runtime Constant pool 都是在 JVM 的 Method area 中分配的，每个 Class 或者 Interface 的 Constant Pool 都是在 JVM 创建 class 或接口时创建的。

运行时常量池是方法区的一部分，所以它的存储也受方法区的规范约束，如果常量池无法分配，同样会 抛出 OutOfMemoryError。


### 1.4 本地方法栈
本地方法栈是为 JVM 运行 Native 方法准备的空间，它和前面介绍的 Java 栈的作用是类似的，由于很多 Native 方法都是用 C 语言实现的，所以它通常又叫 C 栈，除了在我们的代码中包含的常规的 Native 方法会使用这个存储空间，在 JVM 利用 JIT 技术时会将一些 Java 方法重新编译为 Native Code 代码，这些编译后的本地代码通常也是利用这个栈来跟踪方法的执行状态的。

## 2. 垃圾回收
### 2.1 如何检测垃圾
只要是某个对象不再被其他活动对象引用，那么这个对象就可以被回收了。这里的活动对象指的是能够被一个根对象集合到达的对象。根对象集合大都会包含如下一些元素：

- 在方法中局部变量区的元素的引用；
- 在 Java 操作栈中的对象的引用；
- 在常量池中的对象引用；
- 在本地方法中持有的对象的引用；
- 类的 Class 对象。

### 2.2 基于分代的垃圾回收算法
该算法的设计思路是：把对象按照寿命长短来分组，分为年轻代和年老代，新创建的对象被分在年轻代，如果对象经过几次回收后仍然存活，那么再把这个对象划分到年老代。年老代的收集频度不像年轻代那么频繁，这样就减少了每次垃圾收集时所要扫描的对象的数量，从而提高了垃圾回收效率。

这种设计的思路是把堆划分成若干个子堆，每个子堆对应一个年龄代。

JVM 将整个堆划分为 Young 区、Old 区和 Perm 区，分别存放不同年龄的对象，这三个区存放的对象有如下区别。

- Young 区又分为 Eden 区和两个 Survivor 区，其中所有新创建的对象都在 Eden 区，当 Eden 区满后会触发 minor GC 将 Eden 区仍然存活的对象复制到其中一个
  Survivor 区中，另外一个 Survivor 区中的存活对象也复制到这个 Survivor 中，以
  保证始终有一个 Survivor 区是空的。
- Old 区存放的是 Young 区的 Survivor 满后触发 minor GC 后仍然存活的对象，当 Eden 区满后会将对象存放到 Survivor 区中，如果 Survivor 区仍然存不下这些对象，GC 收集器会将这些对象直接存放到 Old 区。如果在 Survivor 区中的对象足
  够老，也直接存放到 Old 区。如果 Old 区也满了，将会触发 Full GC，回收整个堆
  内存。
- Perm 区存放的主要是类的 Class 对象，如果一个类被频繁地加载，也可能会导致 Perm 区满，Perm 区的垃圾回收也是由 Full GC 触发的。

Sun 对堆中的不同代的大小也给出了建议，一般建议 Young 区的大小为整个堆的 1/4，而 Young 区中 Survivor 区一般设置为整个 Young 区的 1/8。

### 2.3 Serial Collector
Serial Collector 是 JVM 在client 模式下默认的 GC 方式。可以通过 JVM 配置参数 `-XX:+UseSerialGC` 来指定 GC 使用该收集算法。我们指定所有的对象都在 Young 区的 Eden中创建，但是如果创建的对象超过 Eden 区的总大小，或者超过了 `PretenureSizeThreshold` 配置参数配置的大小，就只能在 Old 区分配了。

当 Eden 空间不足时就触发了 Minor GC，触发 Minor GC 时首先会检查之前每次 Minor GC 时晋升到 Old 区的平均对象大小是否大于 Old 区的剩余空间，如果大于，则将直接触发 Full GC，如果小于，则要看 `HandlePromotionFailure` 参数（`-XX:-HandlePromotionFailure`）的值。如果为 true，仅触发 Minor GC，否则再触发一次 Full GC。其实这个规则很好理解，如果每次晋升的对象大小都超过了 Old 区的剩余空间，那么说明当前的 Old 区的空间已经不能满足新对象所占空间的大小，只有触发 Full GC 才能获得更多的内存空间。

当 Minor GC时，除了将 Eden 区的非活动对象回收以外，还会把一些老对象也复制到 Old 区中。这个老对象的定义是通过配置参数 `MaxTenuringThreshold` 来控制的，如 `-XX:MaxTenuringThreshold=10`，则如果这个对象已经被 Minor GC 回收过 10 次后仍然存活，那么这个对象在这次 Minor GC 后直接放入 Old 区。还有一种情况，当这次 Minor GC 时 Survivor 区中的 To Space 放不下这些对象时，这些对象也将直接放入 Old 区。如果 Old 区或者 Perm 区空间不足，将会触发 Full GC，Full GC 会检查 Heap 堆中的所有对象，清除所有垃圾对象，如果是 Perm 区，会清除已经被卸载的 classloader 中加载的类的信息。

### 2.4 Parallel Collector
Parallel GC 根据 Minor GC 和 Full GC 的不同分为三种，分别是 ParNewGC、ParallelGC 和 ParallelOldGC。

1. ParNewGC

  可以通过 `-XX:+UseParNewGC` 参数来指定， 它的对象分配和回收策略与 Serial
  Collector 类似，只是回收的线程不是单线程的，而是多线程并行回收。在 Parallel Collector 中还有一个 `UseAdaptiveSizePolicy` 配置参数，这个参数是用来动态控制 Eden、From Space 和 To Space 的 `TenuringThreshold` 大小的，以便于控制哪些对象经过多少次回收后可以直接放入 Old 区。

2. ParallelGC

  在 Server 下默认的 GC 方式，可以通过 `-XX:+UseParallelGC` 参数来强制指定，并行回收的线程数可以通过 `-XX:ParallelGCThreads` 来指定，这个值有个计算公式，如果 CPU 和核数小于 8，线程数可以和核数一样，如果大于 8，值为 $3+(cpu core*5)/8$。

  可以通过 `-Xmn` 来控制 Young 区的大小，如 `-Xman10m`，即设置 Young 区的大小为 10MB。在 Young 区内的 Eden、From Space 和 To Space 的大小控制可以通过 `SurvivorRatio` 参数来完成，如设置成 `-XX:SurvivorRatio=8`，表示 Eden 区与 From Space 的大小为 8:1，如果 Young 区的总大小为 10 MB，那么 Eden、s0 和 s1 的大小分别为 8 MB、1 MB 和 1 MB。但在默认情况下以 `-XX:InitialSurivivorRatio` 设置的为准，这个值默认也为 8，表示的是 Young:s0 为 8:1。当在 Eden 区中申请内存空间时，如果 Eden 区不够，那么看当前申请的空间是否大于等于 Eden 的一半，如果大于则这次申请的空间直接在 Old 中分配，如果小于则触发 Minor GC。在触发 GC 之前首先会检查每次晋升到 Old 区的平均大小是否大于 Old 区的剩余空间，如大于则再触发 Full GC。在这次触发 GC 后仍然会按照这个规则重新检查一次。也就是如果满足上面这个规则，Full GC 会执行两次。

  在 Young 区的对象经过多次 GC 后有可能仍然存活，那么它们晋升到 Old 区的规则可以通过如下参数来控制：`AlwaysTenure`，默认为false，表示只要 Minor GC 时存活就晋升到 old；`NeverTenure`，默认为 false，表示永远不晋升到old 区。如果在上面两个都没设置的情况下设置 `UseAdaptiveSizePolicy`，启动时以 `InitialTenuringThreshold` 值作为存活次数的阈值，在每次 GC 后会动态调整，如果不想使用 `UseAdaptiveSizePolicy`，则以 `MaxTenuringThreshold`
  为准，不使用 `UseAdaptiveSizePolicy` 可以设置为 `-XX:-UseAdaptiveSizePolicy`。如果 Minor GC 时 To Space 不够，对象也将会直接放到 Old 区。当 Old 或者 Perm 区空间不足时会触发 Full GC，如果配置了参数 `ScavengeBeforeFullGC`，在 Full GC 之前会先触发Minor GC。

3. ParallelOldGC
  可以通过 `-XX:+UseParallelOldGC` 参数来强制指定， 并行回收的线程数可以通过
  `-XX:ParallelGCThreads` 来指定，这个数字的值有个计算公式，如果 CPU 和核数小于 8，线程数可以和核数一样，如果大于 8，值为 $3+(cpu core*5)/8$。

  它与 ParallelGC 有何不同呢？其实不同之处在 Full GC 上，前者 Full GC 进行的动作为清空整个 Heap 堆中的垃圾对象，清除 Perm 区中已经被卸载的类信息，并进行压缩。而后者是清除 Heap 堆中的部分垃圾对象，并进行部分的空间压缩。GC 垃圾回收都是以多线程方式进行的，同样也将暂停所有的应用程序。

### 2.5 CMS Collector
可通过 `-XX:+UseConcMarkSweepGC` 来指定，并发的线程数默认为 4（并行GC 线程数 +3），也可通过 `ParallelCMSThreads` 来指定。

CMS GC 与上面讨论的 GC 不太一样，它既不是上面所说的 Minor GC，也不是 Full GC，它是基于这两种 GC 之间的一种 GC。它的触发规则是检查 Old 区或者 Perm 区的使用率，当达到一定比例时就会触发 CMS GC，触发时会回收 Old 区中的内存空间。这个比例可以通过 `CMSInitiatingOccupancyFraction` 参数来指定， 默认是 92%， 这个默认值是通过

$$ ((100-MinHeapFreeRatio)+(double)(CMSTriggerRatio*MinHeapFreeRatio)/100.0)/100.0$$

计算出来的，其中的 `MinHeapFreeRatio` 为 40、`CMSTriggerRatio` 为 80。如果让 Perm 区也使用 CMS GC 可以通过 `-XX:+CMSClassUnloadingEnabled` 来设定，Perm 区的比例默认值也是 92%，这个值可以通过 `CMSInitiatingPermOccupancyFraction` 设定。这个默认值也是通过一个公式计算出来的：

$$
((100-MinHeapFreeRatio)+(double)(CMSTriggerPermRatio*MinHeapFreeRatio)/

100.0)/100.0
$$
其中 `MinHeapFreeRatio` 为 40，`CMSTriggerPermRatio` 为 80。

触发 CMS GC 时回收的只是 Old 区或者 Perm 区的垃圾对象，在回收时和前面所说的
Minor GC 和 Full GC 基本没有关系。在这个模式下的 Minor GC 触发规则和回收规则与 Serial Collector 基本一致，不同之处只是GC 回收的线程是多线程而已。

触发 Full GC 是在这两种情况下发生的：一种是 Eden 分配失败，Minor GC 后分配到 ToSpace，To Space 不够再分配到 Old 区，Old 区不够则触发 Full GC；另外一种情况是，当CMS GC 正在进行时向Old 申请内存失败则会直接触发 Full GC。

这里还需要特别提醒一下，在Hotspot 1.6 中使用这种 GC 方式时在程序中显式地调用
了 `System.gc`，且设置了 `ExplicitGCInvokesConcurrent` 参数，那么使用 NIO 时可能会引发内存泄漏。