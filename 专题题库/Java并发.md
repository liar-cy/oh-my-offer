# Java并发

- 题号前缀：JUC
- 范围：线程池、锁与线程状态、通信取消、volatile 与交替打印、AQS 队列同步器、线程创建方式、线程安全实现手段、线程顺序执行、锁的分类、CAS 与锁的关系、Go 协程在 Java 的对应方案（虚拟线程）、线程安全 LRU 与读多写少的并发优化、限制最大并发数的任务处理器实现。
- 最近更新：2026-09-24
- 说明：按本库面经整理。Java 以 17 为基准，框架差异在题内说明；补充练习不计入实际面试问题。

## 目录

- [[#JUC-001：ThreadLocal 是什么，有什么问题？|JUC-001：ThreadLocal 是什么，有什么问题？]]
- [[#JUC-002：线程池有哪些参数，任务流程及核心线程回收如何工作？|JUC-002：线程池有哪些参数，任务流程及核心线程回收如何工作？]]
- [[#JUC-003：synchronized 和 Lock 有什么区别？|JUC-003：synchronized 和 Lock 有什么区别？]]
- [[#JUC-004：synchronized 修饰普通方法和静态方法有什么区别？|JUC-004：synchronized 修饰普通方法和静态方法有什么区别？]]
- [[#JUC-005：ReentrantLock 如何实现公平锁与非公平锁？|JUC-005：ReentrantLock 如何实现公平锁与非公平锁？]]
- [[#JUC-006：Java 线程有哪些状态？|JUC-006：Java 线程有哪些状态？]]
- [[#JUC-007：sleep 和 wait 有什么区别？|JUC-007：sleep 和 wait 有什么区别？]]
- [[#JUC-008：线程之间有哪些通信与协作方式？|JUC-008：线程之间有哪些通信与协作方式？]]
- [[#JUC-009：线程池提交的任务能取消吗？|JUC-009：线程池提交的任务能取消吗？]]

- [[#JUC-010：volatile 有什么作用和局限？|JUC-010：volatile 有什么作用和局限？]]

- [[#JUC-011：如何交替打印 A1B2C3 并避免线程退出后永久等待？|JUC-011：如何交替打印 A1B2C3 并避免线程退出后永久等待？]]

- [[#JUC-012：AQS 的数据结构是什么，锁竞争与入队出队的并发安全怎么保证？|JUC-012：AQS 的数据结构是什么，锁竞争与入队出队的并发安全怎么保证？]]

- [[#JUC-013：继承 Thread 和实现 Runnable 有什么区别，创建线程有哪些方式？|JUC-013：继承 Thread 和实现 Runnable 有什么区别，创建线程有哪些方式？]]

- [[#JUC-014：Java 有哪些实现线程安全的手段？|JUC-014：Java 有哪些实现线程安全的手段？]]

- [[#JUC-015：如何保证多个线程按指定顺序执行？|JUC-015：如何保证多个线程按指定顺序执行？]]

- [[#JUC-016：Java 中有哪些锁，如何按不同维度分类？|JUC-016：Java 中有哪些锁，如何按不同维度分类？]]

- [[#JUC-017：Go 的协程在 Java 有类似方案吗？|JUC-017：Go 的协程在 Java 有类似方案吗？]]

- [[#JUC-018：什么是 CAS，它和锁有什么关系？|JUC-018：什么是 CAS，它和锁有什么关系？]]
- [[#JUC-019：线程安全的 LRU 缓存怎么实现，读多写少如何优化？|JUC-019：线程安全的 LRU 缓存怎么实现，读多写少如何优化？]]
- [[#JUC-020：如何写一个限制最大并发数的并发任务处理器？|JUC-020：如何写一个限制最大并发数的并发任务处理器？]]

### JUC-001：ThreadLocal 是什么，有什么问题？

**常见问法**

- 什么是 ThreadLocal？
- ThreadLocal 有什么问题？
- ThreadLocal 哪部分数据可能会存在内存泄露？

- ThreadLocal 的底层原理是什么？
- ThreadLocalMap 是怎么存储和寻址的？

#### 面试回答

ThreadLocal 为不同线程保存各自的变量值，适合线程内上下文，不是给共享对象加锁。常见问题是线程池复用导致上下文串用，以及值未及时清理造成内存滞留。使用请求上下文时应在 finally 中 remove；跨线程执行也不会自动获得普通 ThreadLocal 的值。

#### 技术细节

以 OpenJDK 17 为例，Thread 内持有 ThreadLocalMap，条目的 key 是 ThreadLocal 弱引用、value 是强引用。key 被回收后 value 可能仍随长寿命线程保留，内部清理不是及时性保证；若 ThreadLocal 仍被静态字段引用，key 也不会凭空消失。remove 应在使用该值的同一线程执行。InheritableThreadLocal 在创建子线程时继承，不能自动解决线程池任务传播；把同一个可变对象放入不同线程的本地槽位也不使对象线程安全。

存储与寻址（被追问“底层原理”时要能讲到这一层）：值不存放在 ThreadLocal 对象里，而是存放在每个 `Thread` 的 `threadLocals` 字段（`inheritableThreadLocals` 另有一份）指向的 `ThreadLocalMap` 中；ThreadLocal 实例只是定位用的 key。`set` 的首次路径是懒创建 map，随后以 `threadLocalHashCode & (len - 1)` 定位槽位——哈希值由内部常量增量（`0x61C88647`，黄金分割数）逐个 ThreadLocal 递增得来，目的是让同一线程内多个 ThreadLocal 的下标分散均匀；表长恒为 2 的幂，所以取模能用位与代替。冲突处理不用链表而是开放寻址的线性探测（命中已占槽则继续看下一个，遇 key 为 null 的过期槽位直接复用并顺手清理），`get` 走同一条探测链；负载因子约 2/3 触发 `rehash`，重建后按新表长重新散布。因为整张表只属于当前线程，读写都不需要加锁——这是它“快”的机制，也正是它只提供线程封闭、不提供共享可见性的原因。`get` 的快路径是从自己线程的 map 直接取，慢路径（map 未创建）才回退到 `getMap`／初始化。

#### 深挖追问

1. **为什么弱引用 key 仍可能内存滞留？**（补充练习）

   弱引用只影响 key，存活线程到 map 再到 value 的强引用链仍可能存在，必须按生命周期清理。

2. **为什么线程池特别容易串上下文？**（补充练习）

   线程比请求活得更久，下一任务可能拿到上一任务残留的值；在设置前定义默认值，并在 finally 清理。

3. **哪部分数据可能会存在内存泄露？**（面经实际出现；[[面经/帆软/二面/0002#Q03：ThreadLocal 哪部分数据可能存在内存泄露？|MJ035 · 帆软 · 二面 · Q03]]）

   泄露的是 ThreadLocalMap 中 key 已失效的 Entry 的 value：key 是弱引用被回收后，线程→Map→Entry→value 的强引用链仍让 value 存活，长寿命的线程池线程使其无法释放；被回收的只是 ThreadLocal 对象本身，不泄露。防御手段还是同线程 finally remove。

4. **除了内存泄漏，ThreadLocal 还会引发什么问题？**（面经实际出现；[[面经/小红书/一面/0001#Q06：ThreadLocal 是什么、底层原理如何，可能导致什么问题？|MJ040 · 小红书 · 一面 · Q06]]）

   三类：线程池复用导致的脏上下文与串号（比泄漏更致命，如用户身份错乱）；`InheritableThreadLocal` 在池化线程上只在线程创建时复制一次、任务提交时不传播（要靠显式捕获回放，如 TransmittableThreadLocal）；副本语义被误解——子线程拿到的是同一个对象引用，改内部状态照样互相影响，ThreadLocal 只保证“槽位隔离”不保证“对象线程安全”。

**面经来源**

- [[面经/小红书/一面/0001#Q06：ThreadLocal 是什么、底层原理如何，可能导致什么问题？|MJ040 · 小红书 · 一面 · Q06]]

- [[面经/北京某上市公司/一面/0001#Q01：ThreadLocal 是什么，有哪些问题？|MJ004 · 北京某上市公司 · 一面 · Q01]]
- [[面经/帆软/二面/0002#Q03：ThreadLocal 哪部分数据可能存在内存泄露？|MJ035 · 帆软 · 二面 · Q03]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 ThreadLocal API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/ThreadLocal.html)
- [OpenJDK 17u ThreadLocal 源码](https://raw.githubusercontent.com/openjdk/jdk17u/master/src/java.base/share/classes/java/lang/ThreadLocal.java)

### JUC-002：线程池有哪些参数，任务流程及核心线程回收如何工作？

**常见问法**

- 线程池最大等待时间是什么？
- 线程池怎么区分核心线程和非核心线程？
- 线程池有哪些参数，执行流程是什么？

- Java 为什么需要线程池？

- Java 的线程池有哪些核心参数？

- 要设计一个线程池，让高优先级的任务先执行，要怎么做？

#### 面试回答

ThreadPoolExecutor 常见七个参数是 corePoolSize、maximumPoolSize、keepAliveTime、unit、workQueue、threadFactory、handler。任务到来时通常先扩到核心数，再尝试入队，队列容纳不了时扩到最大数，仍不能接收则拒绝。keepAliveTime 管的是空闲线程回收，不是任务最大执行或排队时间。

#### 技术细节

这里按 Java 17 ThreadPoolExecutor。核心数与最大数是工作线程数量阈值，Worker 没有永久的“核心线程身份证”。创建时用不同上限判断；取任务时根据 allowCoreThreadTimeOut 或当前 workerCount 是否大于 corePoolSize，选择带超时 poll 或 take。默认保留核心数量，开启核心超时后也可缩至零；具体退出还需检查池状态和队列。

| 参数 | 含义 |
| --- | --- |
| corePoolSize | 通常优先创建到的工作线程数 |
| maximumPoolSize | 入队失败后仍可扩展到的上限 |
| keepAliveTime / unit | 可超时工作线程等待新任务的空闲期限及单位 |
| workQueue | 等待执行的任务队列 |
| threadFactory | 创建工作线程，配置名称等 |
| handler | 无法接收任务时的拒绝处理 |

无界队列通常使最大线程数难以发挥作用。execute 入队后还会复查关闭状态，并在没有工作线程时补建，不能只背四步而忽略并发关停。Future.get(timeout) 只限制调用者等待结果；awaitTermination 限制等待池终止，二者都不是 keepAliveTime。若要求排队期限或任务总时限，要另行设计 deadline 与协作取消。

线程池的目的：对平台线程复用创建成本，集中管理任务生命周期，并限制在途工作以保护 CPU、内存和下游。池不是 Java 执行并发任务的必要条件；池大小、队列和拒绝策略要匹配负载，过多线程会增加切换和资源竞争。

#### 深挖追问

1. **线程池最大等待时间是什么？**（面经实际出现；[[面经/北京某上市公司/一面/0001#Q04：线程池最大等待时间是什么，核心与非核心线程如何区分？|MJ004 一面 Q04]]）

   原题没指明 API；若指 keepAliveTime，就是线程空闲等任务的期限。若指 get(timeout) 或 awaitTermination，要分别说明等待对象。

2. **线程池如何区分核心和非核心线程？**（面经实际出现；[[面经/北京某上市公司/一面/0001#Q04：线程池最大等待时间是什么，核心与非核心线程如何区分？|MJ004 一面 Q04]]）

   按当前数量及回收策略判断，不给每个 Worker 保留永久分类；核心与非核心首先是容量和生命周期策略。

3. **Java 为什么需要线程池？**（面经实际出现；[[面经/百度/一面/0006#Q17：Java 为什么需要线程池？|MJ010 · 百度 · 一面 · Q17]]）

   常见平台线程服务用它复用线程、控制并发并统一调度；不意味着每种任务都必须使用线程池。

4. **如何设计让高优先级任务先执行的线程池？**（面经实际出现；[[面经/帆软/二面/0002#Q09：线程池有哪些核心参数，如何设计让高优先级任务先执行？|MJ035 · 帆软 · 二面 · Q09]]）

   把 workQueue 换成 PriorityBlockingQueue，任务实现 Comparable（优先级为主键、提交序号为次键保证同级 FIFO），出队即先取高优先级。三个坑：该队列无界使 maximumPoolSize 几乎不生效；只影响排队顺序、不抢占运行中的低优先级任务；低优先级会饥饿，需老化提权或双池隔离保底。

5. **怎么创建线程池，拒绝策略有哪些？**（面经实际出现；[[面经/美团/一面/0003#Q05：为什么要有线程池？怎么创建线程池，拒绝策略有哪些？|MJ037 · 美团 · 一面 · Q05]]）

   创建的正解是显式 `new ThreadPoolExecutor(...)` 七参；Executors 快捷工厂只是包装且有隐患——newFixedThreadPool／newSingleThreadExecutor 无界队列可积压 OOM，newCachedThreadPool／newScheduledThreadPool 线程数上限 Integer.MAX_VALUE 可线程爆炸（阿里手册口径禁用）；Spring 的 ThreadPoolTaskExecutor 是带生命周期管理的包装。拒绝策略四种：AbortPolicy（默认抛异常）、CallerRunsPolicy（提交方执行、形成背压）、DiscardPolicy（静默丢新任务）、DiscardOldestPolicy（弹队首再试）；生产实践自定义 RejectedExecutionHandler：日志＋报警＋降级入队／持久化补偿，选内置策略前先确认任务可丢。

**面经来源**

- [[面经/百度/一面/0006#Q17：Java 为什么需要线程池？|MJ010 · 百度 · 一面 · Q17]]

- [[面经/北京某上市公司/一面/0001#Q04：线程池最大等待时间是什么，核心与非核心线程如何区分？|MJ004 · 北京某上市公司 · 一面 · Q04]]
- [[面经/北京某上市公司/二面/0001#Q08：线程池有哪些参数，执行流程是什么？|MJ004 · 北京某上市公司 · 二面 · Q08]]
- [[面经/帆软/二面/0002#Q09：线程池有哪些核心参数，如何设计让高优先级任务先执行？|MJ035 · 帆软 · 二面 · Q09]]
- [[面经/美团/一面/0003#Q05：为什么要有线程池？怎么创建线程池，拒绝策略有哪些？|MJ037 · 美团 · 一面 · Q05]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 ThreadPoolExecutor API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)
- [OpenJDK 17u ThreadPoolExecutor 源码](https://raw.githubusercontent.com/openjdk/jdk17u/master/src/java.base/share/classes/java/util/concurrent/ThreadPoolExecutor.java)

### JUC-003：synchronized 和 Lock 有什么区别？

**常见问法**

- synchronized 和 Lock 有什么区别？

- synchronized 和 ReentrantLock 是干啥的，区别？

- 用过什么锁？synchronized 底层是什么，和对象头什么关系？

#### 面试回答

synchronized 是语言级监视器锁，退出同步块时自动释放；Lock 是接口，常以 ReentrantLock 为例比较，需要在 finally 中显式 unlock。两者都可提供互斥与可见性，ReentrantLock 还支持可中断获取、限时尝试、公平选项及多个 Condition。普通互斥优先考虑代码清晰，需要这些能力时再用显式锁。

#### 技术细节

不能把 Lock 接口直接等同于所有实现都可重入；本题比较对象是 synchronized 与 ReentrantLock。二者可重入，同一锁的解锁与后续成功加锁建立相应内存可见性。synchronized 等待进入监视器不能通过 interrupt 直接取消，lockInterruptibly 可以；lock() 本身不是可中断获取。公平性、tryLock 和 Condition 是行为差异，性能要结合版本和竞争程度测量。不要在未成功获得锁时调用 unlock。

#### 深挖追问

1. **如何确保 Lock 异常时释放？**（补充练习）

   成功 lock 后立即进入 try，finally unlock；使用 tryLock 时仅在返回 true 后解锁。

2. **synchronized 一定比 ReentrantLock 慢吗？**（补充练习）

   没有通用结论，JVM 优化、竞争模式及临界区长度都会影响，应按需要的语义选择。

3. **synchronized 能锁 String 或 Long 对象吗？**（面经实际出现；[[面经/携程/一面/0001#Q08：synchronized 能锁住 String 和 Long 对象吗？|MJ028 · 携程 · 一面 · Q08]]）

   语法上能（锁任何引用对象，long 会先装箱），但都不该用：String 字面量进常量池，内容相同的字符串全局同一把锁，会把不相干业务意外互斥，还可能与第三方库共享同一监视器；Long/Integer 包装类仅 −128～127 有缓存，超出范围每次装箱都是新对象，两个线程锁“相同的值”可能各锁各的，互斥静默失效，且包装类不可变，更新值等于换锁对象。正确做法是专有 `private final Object lock`，或按业务键做分段锁并自己掌控锁对象生命周期。

4. **synchronized 底层是怎么实现的，和对象头什么关系？**（面经实际出现；[[面经/传音控股/二面/0001#Q03：用过什么锁？synchronized 底层和对象头？|MJ030 · 传音控股 · 二面 · Q03]]）

   字节码层：代码块编译为 monitorenter/monitorexit（异常表保证退出），方法级用 ACC_SYNCHRONIZED 标志。运行层：锁状态记录在对象头 Mark Word（与哈希码、分代年龄复用同一字）——无锁→轻量级锁（CAS 把 Mark Word 拷入线程栈上的锁记录，自旋升级）→竞争加剧膨胀为重量级锁（依赖操作系统 mutex 的 monitor，未抢到者阻塞）；历史上还有偏向锁，因撤销成本与收益问题已在 JDK 15 起废弃（JEP 374）。均为 HotSpot 实现口径，JVMS 只规定语义，答题注明版本并按实现说明。

**面经来源**

- [[面经/北京某上市公司/一面/0001#Q05：synchronized 和 Lock 有什么区别？|MJ004 · 北京某上市公司 · 一面 · Q05]]
- [[面经/帆软/二面/0001#Q14：分布式场景下乐观锁、synchronized 锁的注意点，死锁、锁超时释放怎么处理？|MJ023 · 帆软 · 二面 · Q14]]
- [[面经/携程/一面/0001#Q07：synchronized 和 ReentrantLock 是干啥的，区别？|MJ028 · 携程 · 一面 · Q07]]
- [[面经/携程/一面/0001#Q08：synchronized 能锁住 String 和 Long 对象吗？|MJ028 · 携程 · 一面 · Q08]]
- [[面经/传音控股/二面/0001#Q03：用过什么锁？synchronized 底层和对象头？|MJ030 · 传音控股 · 二面 · Q03]]
- [[面经/美团/一面/0003#Q07：synchronized 和 ReentrantLock 的区别。|MJ037 · 美团 · 一面 · Q07]]
- [[面经/京东零售/一面/0001#Q07：Java 中有哪些锁机制？|MJ034 · 京东零售 · 一面 · Q07]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 ReentrantLock API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html)
- [Java 17 语言规范：线程与锁](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html)

### JUC-004：synchronized 修饰普通方法和静态方法有什么区别？

**常见问法**

- synchronized 加在普通方法和静态方法上有什么区别？

#### 面试回答

普通实例同步方法锁的是 this，静态同步方法锁的是声明该方法的类对应的 Class 对象。同一实例的同步方法互斥，不同实例通常不互斥；同一个 Class 的静态同步方法互斥。实例锁与类锁是不同对象，因此它们不会自动互斥。

#### 技术细节

关键是“锁对象身份相同”，不是方法名或是否访问相同字段。普通方法等价于围绕方法主体同步 this，静态方法锁声明类的 Class；同名类若由不同类加载器定义，Class 对象也不同。若多个实例访问同一 static 可变字段，却只锁各自 this，无法保护这份共享状态。未加同步的其他访问路径也不受这些锁自动约束。

#### 深挖追问

1. **两个不同对象调用静态同步方法会互斥吗？**（补充练习）

   只要实际调用同一个声明类的静态方法，锁的是同一 Class，是否经由不同对象表达式访问不改变锁身份。

2. **静态与实例同步方法怎样才能互斥？**（补充练习）

   显式使用同一个锁对象保护相应临界区，并保证所有相关访问遵循该协议。

**面经来源**

- [[面经/北京某上市公司/一面/0001#Q06：synchronized 加在普通方法和静态方法上有什么区别？|MJ004 · 北京某上市公司 · 一面 · Q06]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 语言规范：线程与锁](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html)

### JUC-005：ReentrantLock 如何实现公平锁与非公平锁？

**常见问法**

- Lock 怎么实现公平和非公平？

#### 面试回答

ReentrantLock 基于 AQS 状态和等待队列实现，状态记录持有次数并支持重入。公平模式在可获取时还会考虑是否有排在前面的等待者，非公平模式允许新线程直接竞争空闲锁。两者失败后都可能排队，不是非公平锁没有队列；公平也不保证操作系统调度绝对公平。

#### 技术细节

构造器 true 选择公平策略，默认是非公平。以典型 AQS 实现理解：竞争空闲状态使用 CAS；持有者重入增加计数，释放至零后才允许其他线程获得。公平的关键是获取资格检查而不是把线程启动顺序严格排序。无超时 tryLock() 即使在公平 ReentrantLock 上也可插队，带超时的尝试遵循相应公平规则。公平常以吞吐为代价降低饥饿风险，不保证每个等待者在固定时间内执行。

#### 深挖追问

1. **公平锁的 tryLock 一定公平吗？**（补充练习）

   无超时 tryLock 明确不遵守公平排队策略，不能把构造器选项当成所有 API 的统一规则。

2. **持有者重入需要重新排队吗？**（补充练习）

   不需要，重入增加持有计数，否则会自己等待自己。

**面经来源**

- [[面经/北京某上市公司/一面/0001#Q07：Lock 怎么实现公平和非公平？|MJ004 · 北京某上市公司 · 一面 · Q07]]
- [[面经/美团/一面/0003#Q08：ReentrantLock 底层原理；AQS 怎么实现、如何保证 state 的原子性？|MJ037 · 美团 · 一面 · Q08]]
- [[面经/京东零售/一面/0001#Q08：ReentrantLock 内部靠什么核心结构来保护共享资源？|MJ034 · 京东零售 · 一面 · Q08]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 ReentrantLock API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html)

### JUC-006：Java 线程有哪些状态？

**常见问法**

- 线程有哪些状态？

#### 面试回答

Java Thread.State 有 NEW、RUNNABLE、BLOCKED、WAITING、TIMED_WAITING、TERMINATED 六种。RUNNABLE 包含运行及等待 CPU 等情况；BLOCKED 特指等待进入 synchronized 监视器。wait、join、park 等可以进入等待状态，带时间限制则可能为 TIMED_WAITING。它们不是操作系统线程状态的一一映射。

#### 技术细节

start 前是 NEW，start 后可被调度执行；run 正常或异常结束后是 TERMINATED，不能再次 start。wait 后被唤醒还需重新获取监视器，可能转为 BLOCKED。ReentrantLock 竞争中 park 的线程通常是 WAITING，而非因“等锁”就必然 BLOCKED。线程状态只是瞬时采样，诊断还要结合堆栈、持锁者和 CPU，不能只凭状态断言死锁。

#### 深挖追问

1. **直接调用 run 会产生新线程吗？**（补充练习）

   不会，是当前线程的普通方法调用；start 才启动新的执行线程。

2. **RUNNABLE 就一定占满 CPU 吗？**（补充练习）

   不一定，它不等价于操作系统正在运行，需结合 CPU 采样判断。

**面经来源**

- [[面经/北京某上市公司/二面/0001#Q05：线程有哪些状态？|MJ004 · 北京某上市公司 · 二面 · Q05]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 Thread.State](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Thread.State.html)

### JUC-007：sleep 和 wait 有什么区别？

**常见问法**

- sleep 和 wait 有什么区别？

#### 面试回答

sleep 是 Thread 的静态方法，让当前线程暂停一段时间，不释放已持有的监视器；wait 是 Object 方法，调用者必须持有该对象监视器，等待时释放它，返回前重新获取。wait 用于条件协作，sleep 主要用于时间暂停，不能靠 sleep 建立线程间可见性。

#### 技术细节

wait 只释放目标对象监视器，不释放线程持有的其他锁。notify/notifyAll 也要持有同一个监视器，通知后不会立即把锁交出去。wait 可能虚假唤醒，因此用 while 检查条件，而不是 if；超时或通知都不代表业务条件一定成立。两者都可因中断抛 InterruptedException，通常应向上抛出或恢复中断标志后按取消协议退出，不吞掉异常继续忙等。

#### 深挖追问

1. **为什么 wait 必须写在条件循环里？**（补充练习）

   可能虚假唤醒或条件被别的线程先消费；重新获得锁后必须再检查。

2. **notify 后等待线程立刻运行吗？**（补充练习）

   不保证，它还要竞争锁并等待调度，通知者在离开同步块前仍持锁。

**面经来源**

- [[面经/北京某上市公司/二面/0001#Q06：sleep 和 wait 有什么区别？|MJ004 · 北京某上市公司 · 二面 · Q06]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 Object API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Object.html)
- [Java 17 语言规范：线程与锁](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html)

### JUC-008：线程之间有哪些通信与协作方式？

**常见问法**

- 线程通信方式有哪些？

- 线程之间是怎么通信的？

#### 面试回答

可用共享状态配合同步、volatile 或原子类表达状态，用 wait/notify、Condition 或 park/unpark 等待条件，也可以用 BlockingQueue 传递数据、Future 获取结果、CountDownLatch 等协调阶段。选择时同时考虑数据可见性、原子性和等待机制，不能只靠循环读普通变量或 sleep。

#### 技术细节

volatile 适合发布状态，但 i++ 等复合操作仍不原子。队列把放入前的动作安全发布给消费线程，并可通过有界容量形成背压。wait/Condition 要把状态检查与等待绑定在相同同步协议中，避免丢通知；park/unpark 的许可最多保留一个，不是可累加的计数信号量。一次性完成可用 latch，需要重复阶段可考虑 barrier/phaser，结果与异常传递可用 Future。共享对象也应避免在发布后无同步修改。

#### 深挖追问

1. **先 unpark 再 park 会丢通知吗？**（补充练习）

   许可可保留，下一次 park 可消费它，但多次 unpark 不会累积多个许可，仍需条件循环。

2. **生产者消费者用什么最直接？**（补充练习）

   通常用有界 BlockingQueue，明确满队列时阻塞、超时或拒绝策略。

**面经来源**

- [[面经/北京某上市公司/二面/0001#Q07：线程通信方式有哪些？|MJ004 · 北京某上市公司 · 二面 · Q07]]
- [[面经/美团/一面/0001#Q05：线程之间怎么通信？进程间的通信方式呢？|MJ031 · 美团 · 一面 · Q05]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 BlockingQueue](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/BlockingQueue.html)
- [Java 17 LockSupport](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/locks/LockSupport.html)

### JUC-009：线程池提交的任务能取消吗？

**常见问法**

- 线程池中提交的任务能取消吗？

#### 面试回答

通过 submit 得到 Future 后，可以尝试 cancel。未开始且取消成功的任务不会再执行；运行中的任务若用 cancel(true)，通常只是向工作线程发中断请求，任务必须协作响应，不保证立刻停止。cancel(false) 不请求中断，任务可能继续运行；Future 显示已取消也不代表业务副作用已经回滚。

#### 技术细节

以下以 ThreadPoolExecutor.submit 返回的 FutureTask 为背景，不把所有 Future 实现视为相同。取消与启动、完成存在竞争，应检查返回值；get 在取消后抛 CancellationException。循环任务检查中断，可中断阻塞则正确处理 InterruptedException；部分 I/O 还需要关闭资源或专用超时。队列中的取消包装器不一定立即物理移除，可按需要使用 purge。get(timeout) 超时不会自动取消任务，shutdownNow 也不等于安全强杀所有工作线程。

#### 深挖追问

1. **cancel(true) 返回 true 就说明线程已经退出吗？**（补充练习）

   不是，只表示取消状态转移成功；不响应中断的业务仍可能运行，要用自己的完成信号验证实际退出。

2. **已经写入数据库的数据会自动撤销吗？**（补充练习）

   不会，取消线程任务与数据库事务回滚是不同机制，需按事务边界和幂等策略处理。

**面经来源**

- [[面经/北京某上市公司/二面/0001#Q12：线程池中提交的任务能取消吗？|MJ004 · 北京某上市公司 · 二面 · Q12]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 Future API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/Future.html)
- [Java 17 ThreadPoolExecutor API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)

### JUC-010：volatile 有什么作用和局限？

**常见问法**

- volatile 有什么作用？

#### 面试回答

volatile 提供共享变量读写的可见性和相应有序性：对某 volatile 变量的写 happens-before 后续对它的读。它适合发布状态或不可变快照引用，但不能让 i++、先检查再更新等复合操作自动原子化。

#### 技术细节

按 Java SE 21 JMM 理解，发布线程在 volatile 写之前的操作，可经该同步关系对读取线程可见。volatile 引用不把被引用对象之后的任意修改都变成安全操作；多变量不变量应使用锁或适合的原子协议。

#### 深挖追问

1. **volatile 能替代锁吗？**（补充练习）

   简单状态发布可以；需要互斥更新或维护多个字段一致性时不能直接替代。

**面经来源**

- [[面经/百度/一面/0006#Q20：volatile 有什么作用？|MJ010 · 百度 · 一面 · Q20]]

**参考资料**（2026-09-13 查证；版本见正文）

- [JLS Java SE 21：线程与锁](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)

### JUC-011：如何交替打印 A1B2C3 并避免线程退出后永久等待？

**常见问法**

- 多线程交替打印 A1B2C3，任一线程退出后其余线程不永久阻塞。

#### 面试回答

先约定退出语义：本解采用任一工作线程退出则取消整个打印任务，其余线程及时结束，不要求继续补齐序列。用同一把锁保护轮次与取消状态，条件循环等待；正常打印后修改轮次并通知，异常、中断或提前返回时在 finally 发布取消并唤醒所有等待者。

#### 技术细节

假设两个线程各输出 A—Z 与 1—26，正常序列为 A1B2…Z26。等待条件必须同时检查轮次和取消；唤醒后再检查取消。线程包装器的 finally 必须覆盖所有退出路径，启动失败也由协调者取消。该协议保证协作退出且线程能被调度时无永久条件等待；进程被杀、线程被不安全强停或持锁永久阻塞不在保证内。若要求其他线程继续输出，必须另定存活集合与跳过轮次规则。

Java 参考核心实现（每次任务创建新实例；调用方创建并启动两个线程，分别调用 `run(0)` 和 `run(1)`）：

```java
final class Alternating {
    private final Object monitor = new Object();
    private int turn = 0;
    private boolean cancelled = false;

    void cancel() {
        synchronized (monitor) {
            cancelled = true;
            monitor.notifyAll();
        }
    }

    void run(int role) {
        try {
            for (int i = 0; i < 26; i++) {
                synchronized (monitor) {
                    while (!cancelled && turn != role) monitor.wait();
                    if (cancelled || Thread.currentThread().isInterrupted()) return;
                    if (role == 0) System.out.print((char) ('A' + i));
                    else System.out.print(i + 1);
                    turn = 1 - role;
                    monitor.notifyAll();
                    // 字母线程最后一次输出后，等数字线程完成或取消，
                    // 避免它立刻进入 finally 而让最终的 26 被跳过。
                    if (role == 0 && i == 25) {
                        while (!cancelled && turn != 0) monitor.wait();
                    }
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            cancel();
        }
    }
}
```

例子用标准输出演示，假设输出调用最终返回；生产中不要在该锁内调用可能永久挂起的外部 I/O。`finally` 取消涵盖正常结束、异常和中断；若某线程未启动成功，协调者需调用 `cancel()`。


#### 深挖追问

1. **任意线程退出后如何保证其余线程不永久等待？**（面经实际出现；[[面经/百度/三面/0001#Q01：多线程交替打印 A1B2C3，任一线程退出后其余线程不永久阻塞。|MJ010 · 百度 · 三面 · Q01]]）

   所有可控退出路径发布取消并通知所有等待者，等待循环检查取消；本解约定整体取消，非继续补齐。

2. **只在正常输出后 notify 为什么不够？**（补充练习）

   对方可能在异常或中断时退出，永远不会再通知；需要统一退出清理和取消条件。

**面经来源**

- [[面经/百度/三面/0001#Q01：多线程交替打印 A1B2C3，任一线程退出后其余线程不永久阻塞。|MJ010 · 百度 · 三面 · Q01]]

**参考资料**（2026-09-13 查证；版本见正文）

- [JLS Java SE 21：等待与通知](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)

### JUC-012：AQS 的数据结构是什么，锁竞争与入队出队的并发安全怎么保证？

**常见问法**

- AQS 的数据结构是什么，锁竞争怎么处理，入队出队的并发安全怎么保证？

- AQS 是怎么实现的，是怎么保证 state 变量的原子性的？

#### 面试回答

AQS（AbstractQueuedSynchronizer）核心＝一个 `state`（volatile int，锁/信号量/线程池工作数的语义由子类解释）＋一条 CLH 变体的 FIFO 双向等待队列。获取流程：子类 `tryAcquire` 用 CAS 改 state 抢占；失败则把线程包装成 Node（存线程、等待状态 waitStatus）CAS 入队尾，然后“前驱是 head 且再次 tryAcquire 成功”才出闸，否则按条件 park 挂起。竞争处理是“进入队列前自旋尝试一次＋入队后挂起等待”的混合：避免纯自旋烧 CPU，也避免每次获取都要唤醒。释放由 `tryRelease` 决定，成功后 `unparkSuccessor` 唤醒队首后继。ReentrantLock 的公平/非公平只差在获取时是否先检查队列（`hasQueuedPredecessors`），见 [[专题题库/Java并发#JUC-005：ReentrantLock 如何实现公平锁与非公平锁？|JUC-005：ReentrantLock 如何实现公平锁与非公平锁？]]。

#### 技术细节

入队（addWaiter）安全性靠两次 CAS 拼出：队列未初始化时 CAS 设置 head，随后 CAS 把新节点接到尾（先 `pred.next=node` 再 CAS `tail`，最后补 `node.prev`）；CAS 失败（有其他线程并发入队）就重读 tail 重试，中间被其他线程观察到“链接不完整”时，靠遍历方向与 waitStatus 检查容忍并修正。出队靠 setHead（把成功获取的节点设为新 head 并断开前驱引用帮助 GC）＋unparkSuccessor：唤醒后继时**从尾部向前**找有效等待节点，因为并发入队可能让 `head.next` 暂时为 null 或断链，从尾向前遍历（prev 链在此时已完整）更可靠。Node 的 waitStatus：SIGNAL 表示释放时要唤醒后继，CANCELLED 表示超时/中断后节点作废，CONDITION、PROPAGATE 服务条件队列与共享模式。独占与共享模式的区别在获取成功后是否继续向后传播唤醒（如 ReadWriteLock、Semaphore）。park/unpark 与 `LockSupport` 配合，唤醒后线程从 park 处返回再次参与 tryAcquire 竞争，而非直接获得锁——这也是非公平锁“ barging（插队）”的来源。

#### 深挖追问

1. **为什么唤醒后继要从尾向前遍历而不是 head.next？**（补充练习）

   并发入队的最后一步（设置 prev/next 链接）可能尚未完成，`head.next` 短暂为 null 或指向作废节点；prev 方向在 tail CAS 成功后已可用，从尾向前能找到最后一个有效等待者。

2. **AQS 的 state 一定是锁吗？**（补充练习）

   不是。state 只是一个 CAS 保护的 int，语义由子类定义：ReentrantLock 是重入次数、Semaphore 是许可数、ThreadPoolExecutor 合并存运行状态与工作线程数、CountDownLatch 是计数。

3. **AQS 怎么保证 state 变量的原子性？**（面经实际出现；[[面经/美团/一面/0003#Q08：ReentrantLock 底层原理；AQS 怎么实现、如何保证 state 的原子性？|MJ037 · 美团 · 一面 · Q08]]）

   两件套：state 声明为 volatile（读永远拿最新值、禁重排），写全部经 compareAndSetState——Unsafe／VarHandle 落到 CPU 原子指令（x86 lock cmpxchg，ARM LL/SC），“比较＋交换”整体不可分割。关键点：volatile 只保证单次读／写原子，抢锁是“读—判断—写”复合动作，必须靠 CAS 一次完成；失败就重读重试或由子类决定入队挂起，state 本身从不需要锁来保护。

**面经来源**

- [[面经/帆软/一面/0002#Q09：AQS 的数据结构是什么，锁竞争怎么处理，入队出队的并发安全怎么保证？|MJ027 · 帆软 · 一面 · Q09]]
- [[面经/京东零售/一面/0001#Q08：ReentrantLock 内部靠什么核心结构来保护共享资源？|MJ034 · 京东零售 · 一面 · Q08]]
- [[面经/美团/一面/0003#Q08：ReentrantLock 底层原理；AQS 怎么实现、如何保证 state 的原子性？|MJ037 · 美团 · 一面 · Q08]]

**参考资料**（查证：2026-09-22）

- [JDK 17 AbstractQueuedSynchronizer API 文档](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/locks/AbstractQueuedSynchronizer.html)
- [Doug Lea：The java.util.concurrent Synchronizer Framework](https://gee.cs.oswego.edu/dl/papers/aqs.pdf)（实现思路论文；细节以所用 JDK 源码为准）

### JUC-013：继承 Thread 和实现 Runnable 有什么区别，创建线程有哪些方式？

**常见问法**

- 多线程——继承 Thread 和实现 Runnable 接口有什么区别？

#### 面试回答

本质区别是“任务与执行载体是否分离”。继承 Thread：逻辑写在 run() 里，Thread 对象本身就是执行载体——受 Java 单继承限制，继承它就占用唯一继承位；任务与线程绑死，每个任务要 new 一个 Thread，多线程共享数据要靠外部构造，也无法交给线程池统一调度。实现 Runnable：只是“要跑的任务”，可以交给任意 Thread 执行或提交线程池，同一实例可复用、便于共享状态，也天然适配 Lambda。实现层面 Thread 本身就 implements Runnable，`new Thread(runnable)` 启动后线程执行的是传入任务的 run()。还有第三种 Callable＋`FutureTask`：`call()` 有返回值、可抛检查异常，配合 `ExecutorService.submit` 拿结果——生产上应优先“任务对象＋线程池”，不要散落 new Thread。

#### 技术细节

常被追问“start() 和 run() 区别”：run() 只是普通方法调用、仍在当前线程执行；start() 才向 JVM 申请新线程并触发线程状态从 NEW→RUNNABLE 的调度。“创建线程有几种方式”的严谨口径：创建线程本质只有构造 Thread（或其子类）再 start 一条路径；所谓“四种方式”（继承 Thread、实现 Runnable、Callable＋FutureTask、线程池提交）是“定义任务”的方式差异，线程池底层仍是 ThreadFactory 造 Thread。注意 Executors 快捷工厂的隐患（无界队列/无限线程数），生产用 ThreadPoolExecutor 显式参数（见 [[#JUC-002：线程池有哪些参数，任务流程及核心线程回收如何工作？|JUC-002：线程池参数与流程]]）。虚拟线程（Loom，JDK 21 起正式可用）另属一代模型：`Thread.ofVirtual()`／结构化并发，不改变上述接口关系。

#### 深挖追问

1. **Runnable 怎么拿到执行结果？**（补充练习）

   Runnable 的 run() 无返回值：要么改用 Callable＋Future 取结果，要么在任务内通过回调/共享 CompletableFuture 完成通知，异常要显式处理否则只进线程的未捕获异常处理器。

2. **创建一个新线程的主要成本（资源消耗）有哪些？**（面经实际出现）

   三条：① 内存——每条线程一个独立**栈**（默认约 1MB，`-Xss` 可调，主要是虚拟地址预留、按需触页）加上 JVM/OS 侧的 Thread 控制块与内核调度实体；② 系统调用——`start()` 要经 JNI 向 OS 创建线程，涉及用户态／内核态切换；③ 运行期——线程被调度时的**上下文切换**（保存/恢复寄存器、PC、栈指针，并冲击 CPU 缓存与 TLB，见 [[专题题库/操作系统#OS-009：线程切换需要保存哪些 CPU 上下文？|OS-009：线程切换的开销]]）。线程数过多还会放大内存占用与 GC Roots 扫描成本，这正是“用线程池复用、限制并发线程数”的根因；大量阻塞型任务可用 JDK 21 虚拟线程降低对载体线程的占用。
   来源：[[面经/京东零售/一面/0001#Q09：怎么创建一个新线程，它的主要成本（资源消耗）有哪些？|MJ034 · 京东零售 · 一面 · Q09]]

**面经来源**

- [[面经/传音控股/二面/0001#Q02：多线程——继承 Thread 和实现 Runnable 接口有什么区别？|MJ030 · 传音控股 · 二面 · Q02]]
- [[面经/京东零售/一面/0001#Q09：怎么创建一个新线程，它的主要成本（资源消耗）有哪些？|MJ034 · 京东零售 · 一面 · Q09]]

**参考资料**（查证：2026-09-22）

- [Java 17 Thread API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Thread.html)
- [Java 17 Runnable / Callable API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Runnable.html)

### JUC-014：Java 有哪些实现线程安全的手段？

**常见问法**

- Java 里面是怎么实现线程安全的？
- 怎么保证一个类在多线程下正确？

#### 面试回答

按“从不上锁到上重锁”的成本梯度分层答：① 不共享可变状态——栈封闭（对象只在方法内创建）、ThreadLocal、不可变对象（final 字段＋正确发布、record、只含常量的类），这是最优解；② 无锁的可变共享——volatile 保证可见性与禁止重排，Atomic 原子类与 LongAdder 靠 CAS 做原子更新，注意它们只保证单个操作原子，`i++` 这种“读—改—写”复合动作靠 volatile 并不安全；③ 互斥——synchronized（代码块/方法，静态方法锁 Class 对象）与 ReentrantLock（可公平、可中断、多 Condition），配 wait/notify 或 Condition 表达条件；④ 现成的并发组件——ConcurrentHashMap、CopyOnWriteArrayList、BlockingQueue，以及 `computeIfAbsent`／`merge`／`putIfAbsent` 这类原子复合操作，避免自己拼 check-then-act；⑤ 结构层面——用有界线程池、单线程写者＋队列把共享收敛为消息传递。
收口要说方法论：先明确类的**不变量**是什么、由哪个锁或协议保护，读写两端都走同一协议才叫安全；同时覆盖**发布安全**（对象构造完成前不外泄 this、final 语义）和**终止安全**（中断、shutdown 与资源释放），否则只是“看起来没报错”。

#### 技术细节

三个必须能展开的点：

1. **原子性、可见性、有序性分别由什么保证**：原子性靠锁与 CAS；可见性靠同步（synchronized/Lock 的解锁—加锁形成 happens-before）、volatile 或 final 正确初始化；有序性靠 JMM 的重排限制——单靠“加 volatile 字段”只解决可见性，不解决多个写者之间的竞争。
2. **锁该锁什么**：锁的对象必须是保护同一份不变量的所有线程共同看到的对象。常见错误是锁了每次新建的实例（`synchronized(new Object())`、锁 String 参数）、锁 `Integer` 装箱对象（受缓存影响且语义不确定）、读写分离用了两把不同的锁。读写锁（`ReentrantReadWriteLock`／StampedLock）适合读多写少，但 StampedLock 的乐观读要校验并处理重读，代码复杂度高。
3. **并发容器的边界**：`ConcurrentHashMap` 保证单个方法原子，不保证“先 containsKey 再 put”这种调用序列原子；迭代器是弱一致的，不抛 ConcurrentModificationException 但可能读到旧快照；`CopyOnWriteArrayList` 读便宜写昂贵，只适合读极多、写极少。

补充：安全发布的方式包括初始化 static 字段（含 holder 单例模式）、volatile 字段、final 字段（构造函数内 this 未逸出时）、经 Lock 保护的字段、以及传入 BlockingQueue。线程池复用线程会让 ThreadLocal 值跨任务残留，务必 `remove()`（见 [[#JUC-001：ThreadLocal 是什么，有什么问题？|JUC-001：ThreadLocal 是什么，有什么问题？]]）。虚拟线程场景下（JDK 21 起）长临界区的 `synchronized` 曾会钉住载体线程，JDK 24 的 JEP 491 已改进，但阻塞式锁仍建议改用 `ReentrantLock` 或无锁结构，并按所用 JDK 版本说明。

#### 深挖追问

1. **volatile 能替代锁吗？**（补充练习）

   不能。它解决可见性与重排，不解决复合动作的原子性；只有一个变量、且写入不依赖旧值（标志位、发布已构造好的对象引用）时够用，计数、边界检查、多字段不变量仍需锁或原子类的 CAS 循环。

2. **不可变对象一定线程安全吗？**（补充练习）

   引用不可变不等于状态不可变：`final List<String>` 字段本身不能改指向，但列表内容能被改，即“浅不可变”。真正不可变要求所有字段 final、类型不提供修改方法、构造器不逸出 this、且引用的对象也不可变（或用防御性拷贝／`List.copyOf`）。

3. **怎么判断一段代码需要加锁？**（补充练习）

   看是否存在“两个及以上线程访问同一状态、至少一个写、且访问不在同一同步协议内”。三条件缺一就不需要锁；反之即便“目前只有一个线程跑”也应写清不变量与保护策略，否则扩线程数时就是隐藏 bug。

4. **synchronized 和 ReentrantLock 怎么选？**（面经实际出现；参见 [[#JUC-003：synchronized 和 Lock 有什么区别？|JUC-003：synchronized 和 Lock 有什么区别？]]）

   默认用 synchronized：语法简单、不会忘记 unlock、JVM 层持续优化。需要公平策略、可中断获取、超时获取、多条件队列或非块结构化锁（跨方法加解锁）时才用 ReentrantLock。

**面经来源**

- [[面经/美团/一面/0001#Q21：Java 里怎么实现线程安全？|MJ031 · 美团 · 一面 · Q21]]
- [[面经/美团/一面/0003#Q06：多线程怎么保证线程安全？|MJ037 · 美团 · 一面 · Q06]]

**参考资料**（本次查证：2026-09-22）

- [JSR-133（Java 内存模型与 `volatile`／`final` 语义）](https://jcp.org/en/jsr/detail?id=133)
- [JEP 491：Synchronize Virtual Threads without Pinning（JDK 24）](https://openjdk.org/jeps/491)
- [Java 17 ConcurrentHashMap API（原子复合操作与弱一致迭代器）](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)

### JUC-015：如何保证多个线程按指定顺序执行？

**常见问法**

- 有 A、B、C 三个线程，如何保证它们的执行顺序？
- 怎么让线程 2 等线程 1 执行完再执行？

#### 面试回答

题意是“让 B 等 A 跑完、C 等 B 跑完”这种依赖顺序，最直接的答案是 **`Thread.join`**：主线程 `a.start(); a.join();` 等 A 结束后再 `b.start(); b.join();`，依次串起来；也可以在任务体里让后继线程 `join` 它的前驱。等价手段有：

- **单线程线程池**——`Executors.newSingleThreadExecutor()` 按 `submit` 顺序串行执行，天然有序、最省事。
- **`CountDownLatch`**——A 完成时 `countDown`，B 在开头 `await` 放行，适合“阶段/里程碑”式依赖。
- **`CompletableFuture`**——`supplyA().thenRunAsync(B).thenRunAsync(C)` 用链式回调表达依赖，还能组合并行分支。
- **`CyclicBarrier`／`Semaphore`／锁＋`Condition` 标志位**——更复杂的多阶段协作时用。

要点：`start()` 只保证“被调度”，不保证执行先后顺序，所以“先 start 的就先跑完”是错的直觉，必须显式建立“等待前驱完成”的关系。

#### 技术细节

`join` 底层是 `wait/notify`：线程终止时 JVM 会唤醒所有 `join` 它的线程；`join(timeout)` 可加超时避免前驱卡死时永久阻塞。单线程池的本质是把任务排进一条队列由一个 worker 顺序取，注意它用的是无界队列，任务堆积会占内存。`CountDownLatch` 计数只能一次性归零，需要重复放行阶段要用 `CyclicBarrier` 或 `Phaser`。若只是“按顺序提交”但允许并行执行，那考的就不是顺序而是编排，要区分清楚。

候选人最初答“把 A/B/C 放进公平等待队列一个个取”——公平队列解决的是“同一时刻谁先获得执行权”，并不能表达“A 完全结束后才能开始 B”的完成依赖，面试官不认可；回到 `join`／串行线程池／latch 这类“等待前驱完成”的机制才是本题要点。

#### 深挖追问

1. **`join` 和让线程 `sleep` 一段固定时间再启动下一个有什么区别？**（补充练习）

   `sleep` 是按“猜测的时长”错开，前驱若超时或提前完成都对不上，脆弱；`join` 是事件驱动的“等前驱真正终止”，与耗时无关，语义正确。

2. **`join` 为什么能被中断，中断后怎样？**（补充练习）

   `join` 抛 `InterruptedException`，表示等待被中断；处理时不要吞掉，要么恢复中断标志要么按业务终止，否则等待关系被破坏。

**面经来源**

- [[面经/字节/一面/0008#Q06：有 A、B、C 三个线程，如何保证它们的执行顺序？|MJ033 · 字节 · 一面 · Q06]]

**参考资料**（本次查证：2026-09-23）

- [Java 17 Thread.join API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Thread.html#join())
- [Java 17 CountDownLatch / Executors API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/package-summary.html)

### JUC-016：Java 中有哪些锁，如何按不同维度分类？

**常见问法**

- Java 中有哪些锁机制？
- Java 里锁的作用是什么？什么样的业务需要用锁来做？

#### 面试回答

不要背名字，按“分类维度”答更显结构：

- **乐观 vs 悲观**：悲观锁先加锁再操作（`synchronized`、`ReentrantLock`、DB 行锁）；乐观锁假设冲突少、用 CAS／版本号失败再重试（`Atomic*`、`Integer` 版本、`StampedLock` 乐观读）。
- **阻塞 vs 自旋**：拿不到就挂起（重量级锁、`Lock`）vs 忙等重试（自旋，适合临界区极短）。
- **可重入 vs 不可重入**：`synchronized`、`ReentrantLock` 可重入（同线程再次进入不死锁，靠持有计数）。
- **公平 vs 非公平**：`ReentrantLock(boolean fair)`，公平按队列顺序、非公平允许插队（吞吐更高、可能饥饿）。
- **独占 vs 共享**：写锁独占、读锁共享（`ReentrantReadWriteLock`、`StampedLock` 读写/乐观读）。
- **偏向/轻量/重量**：`synchronized` 在对象头 Mark Word 上的锁状态演进（偏向锁 JDK 15 起废弃）。
- 还有协作工具（`Semaphore`、`CountDownLatch`、`CyclicBarrier`）和分布式锁（Redis、ZooKeeper）——严格说部分是并发工具而非“互斥锁”，答时区分。

落点：锁保护的是“同一份共享可变状态的复合操作”，能不加锁就不加锁（用不可变、线程封闭、并发容器替代，见 JUC-014）。

#### 技术细节

`ReadWriteLock` 与 `StampedLock`：读写锁“读读并行、读写/写写互斥”，写锁可降级为读锁但读锁不能升级为写锁（会死锁）；`StampedLock` 提供乐观读（读时不加锁、读完 `validate` 校验期间是否被写），更快但不可重入、需处理线程中断。自旋锁要设上限与退避，否则空转烧 CPU。锁分类是横切维度，同一把锁可同时是“可重入＋非公平＋独占”（默认 `ReentrantLock`），别把它们当互斥选项。

#### 深挖追问

1. **`synchronized` 和 `ReentrantLock` 怎么选？**（面经实际出现）

   见 [[#JUC-003：synchronized 和 Lock 有什么区别？|JUC-003：synchronized 和 Lock 有什么区别]]——前者语法简单、自动释放、够用就用；后者要可中断获取、超时 `tryLock`、公平性、多 `Condition` 或读写分离时才值那点复杂度。

2. **偏向锁为什么被废弃？**（补充练习）

   它优化“始终单线程重入”的场景，但撤销需要到 safepoint、收益在现代并发下不显著且带来复杂度与性能毛刺，HotSpot 在 JDK 15 起默认禁用并标记废弃（与 [[#JUC-003：synchronized 和 Lock 有什么区别？|JUC-003]] 的锁状态口径一致，具体参数以所用 JDK 版本文档为准）。

**面经来源**

- [[面经/京东零售/一面/0001#Q07：Java 中有哪些锁机制？|MJ034 · 京东零售 · 一面 · Q07]]
- [[面经/帆软/一面/0003#Q05：Java 里锁的作用是什么，什么样的业务需要用锁？|MJ035 · 帆软 · 一面 · Q05]]

**参考资料**（本次查证：2026-09-23）

- [Java 17 concurrency（locks）package API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/locks/package-summary.html)

### JUC-017：Go 的协程在 Java 有类似方案吗？

**常见问法**

- 就包括你刚说 Go 的协程，Java 有类似的方案吗？
- 用 Go 写的并发服务改用 Java，怎么达到类似效果？

#### 面试回答

最接近的是 JDK 21 正式可用的虚拟线程（Project Loom，JEP 444）：由 JVM 而非操作系统调度的轻量线程，创建用 `Thread.ofVirtual()` 或 `Executors.newVirtualThreadPerTaskExecutor()`，每个任务一条、用完即弃，内存开销以 KB 计，可以开出百万级。关键机制与 goroutine 同构：线程在阻塞 IO 时不是挂住 OS 线程，而是把虚拟线程从载体线程（平台线程，跑在 ForkJoinPool 上）卸载，IO 完成后再挂到某个载体继续，所以“同步写法、异步性能”成立，不再需要回调地狱。JDK 21 之前的替代是响应式（WebFlux／Reactor）或 CompletableFuture 编排，能力等价但编程模型倒置；语言级协程有 Kotlin coroutines、Quasar 字节码方案。差异也要讲清：Java 没有 channel 那样的内建通信原语，协作靠 BlockingQueue 等；虚拟线程不该再包线程池，限流用 Semaphore；大量 ThreadLocal 在百万线程下内存放大，官方建议改用 Scoped Values（仍在孵化）。

#### 技术细节

- 版本口径：预览 JEP 425（JDK 19）→ JEP 436（20）→ 正式 JEP 444（21，LTS）。synchronized 长临界区与 native 帧会“钉住”载体线程（JEP 491 于 JDK 24 移除了 synchronized 这一限制），21～23 上高并发锁竞争场景仍建议 ReentrantLock 或缩小临界区——答题要报所用 JDK 版本。
- 结构化并发 `StructuredTaskScope` 与 Scoped Values 截至 JDK 25 仍在孵化迭代，生产主路径是虚拟线程本体。
- 与 GMP 对照（见 [[专题题库/Go#GO-001：是否会 Go，goroutine 的底层实现是什么？|GO-001]]）：虚拟线程≈G，载体线程≈M，ForkJoinPool 的 work-stealing≈P 队列＋窃取；差别在 Go 靠编译器插入调度点、Java 靠 JDK 库在阻塞点主动 yield，未走 JDK 阻塞 API 的“假 IO”（如自旋、第三方 socket 实现）不会自动让出。
- 池的语义变化：平台线程池的价值在“复用昂贵的线程”，虚拟线程便宜到不需要复用；要保护下游时限制的是并发任务数（Semaphore／固定许可），不是线程数。

#### 深挖追问

1. **虚拟线程能完全替代 goroutine 吗？**（补充练习）

   并发模型上接近，工程生态上不等价：Go 有语言级 channel、select、统一运行时与更小的部署产物；Java 侧虚拟线程只解决“阻塞廉价”，通信靠 JUC 组件，且老代码里的锁、ThreadLocal、连接池尺寸假设都要重新审视。

2. **为什么不能给虚拟线程再套一个固定大小线程池？**（补充练习）

   虚拟线程的开销主要在堆内栈对象，创建销毁都便宜；套固定池等于把并发度又钉回池大小，浪费了“每任务一线程”的模型。需要的是对下游资源（DB 连接、配额）显式设闸，用 Semaphore 表达，而不是复用线程。

**面经来源**

- [[面经/帆软/一面/0003#Q04：Go 的协程在 Java 里有类似的方案吗？|MJ035 · 帆软 · 一面 · Q04]]
- [[面经/帆软/一面/0003#Q03：如果改用 Java 开发同样的需求，架构上要注意什么、用什么技术达到类似效果？|MJ035 · 帆软 · 一面 · Q03]]

**参考资料**（本次查证：2026-09-23）

- [JEP 444: Virtual Threads (JDK 21)](https://openjdk.org/jeps/444)
- [JEP 491: Synchronize Virtual Threads without Pinning (JDK 24)](https://openjdk.org/jeps/491)
- [JEP 553: Structured Concurrency (Sixth Preview, JDK 25)](https://openjdk.org/jeps/553)

### JUC-018：什么是 CAS，它和锁有什么关系？

**常见问法**

- 讲一下 CAS 操作，CAS 和锁有关系吗？

#### 面试回答

CAS（Compare-And-Swap）是一条原子指令：给定内存位置、期望旧值和新值，仅当当前值等于期望值才写入新值并返回成功，否则失败；更新失败由调用方重读重试，形成 lock-free 的“乐观重试循环”。它和锁的关系一句话：CAS 不是锁，是无锁并发的硬件原语，但它是很多锁的实现零件——Atomic 整家族、AQS 抢 state、ReentrantLock 入队、synchronized 轻量级锁替换对象头 Mark Word、ConcurrentHashMap 的空表初始化与桶写入，底层全是 CAS。对比口径：锁是悲观的、失败让出 CPU 挂起；CAS 是乐观的、失败自旋重试，所以低竞争高吞吐、高竞争空转烧 CPU。局限也要报全：只保证单字原子，多字段不变量仍需锁；ABA 问题用版本号（AtomicStampedReference）解；拿不到进度时可能需要帮助器介入（无锁队列的复杂之处）。

#### 技术细节

- 硬件层：x86 的 `lock cmpxchg`（锁缓存行一致性），ARM／RISC-V 用 LL／SC（load-linked／store-conditional）对；JVM 侧入口是 `Unsafe.compareAndSwap*`，JDK 9+ 推荐 VarHandle（compareAndSet、getAndSet 等，带内存序选择）。
- Java 用法示例（自实现原子计数器，体会 retry 形态）：

```java
final AtomicInteger counter = new AtomicInteger();
void inc() {
    int old;
    while (!counter.compareAndSet(old = counter.get(), old + 1)) {
        // 失败即有并发修改，重读再试；Thread.onSpinWait() 可提示 CPU 自旋
    }
}
```

- ABA：值从 A 改回 A，CAS 误判“没变过”。对纯计数无所谓，对链表节点复用是致命的（head 弹出又压回相同值）；解法是版本戳 AtomicStampedReference 或每节点独立对象（Java 并发容器的做法）。
- CAS 自带同步语义：成功的 CAS 相当于一次 volatile 读＋volatile 写，happens-before 成立，这也是 Atomic 类不加锁还保证可见性的原因。
- 高竞争替代：LongAdder 把单点 CAS 分散成 Cell 数组按线程哈希落点，空间换竞争；这是“CAS 也有 contention 预算”的典型工程例证。

#### 深挖追问

1. **CAS 属于乐观锁还是悲观锁？**（补充练习；口径见 [[#JUC-016：Java 中有哪些锁，如何按不同维度分类？|JUC-016：锁的分类]]）

   通常说“乐观锁的一种实现”：假设冲突少、失败再重试；悲观锁是先占资源再操作。严格讲 CAS 本身只是原子指令，“乐观”是使用它的算法策略。

2. **自旋锁和 CAS 是一回事吗？**（补充练习）

   不是。自旋是“获取失败后的等待策略”（忙等），CAS 是“尝试更新的方式”；synchronized 的轻量级锁用 CAS 抢 Mark Word，抢不到后是否自旋、自旋多久由实现决定（自适应自旋）。反过来 Mutex 挂起等待也可以先 CAS 一次再睡——两者正交。

3. **为什么有了 CAS 还需要 AQS 的队列？**（补充练习）

   纯 CAS 重试在高竞争下吞吐塌方且不公平；AQS 用 CAS 处理快速路径，竞争者入 CLH 队列 park 挂起，把“抢不到就排队睡觉”制度化，兼得无锁快速路径与有界等待。见 [[#JUC-012：AQS 的数据结构是什么，锁竞争与入队出队的并发安全怎么保证？|JUC-012：AQS]]。

**面经来源**

- [[面经/帆软/一面/0003#Q06：讲一下 CAS 操作，CAS 和锁有什么关系？|MJ035 · 帆软 · 一面 · Q06]]

**参考资料**（本次查证：2026-09-23）

- [Java 21 VarHandle API（compareAndSet 与内存序）](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/invoke/VarHandle.html)
- [Java 21 java.util.concurrent.atomic 包](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/package-summary.html)

### JUC-019：线程安全的 LRU 缓存怎么实现，读多写少如何优化？

**常见问法**

- 上述 LRU 如果要改造成线程安全的，该怎么做？
- 在“读多写少”的并发场景下，怎么优化线程安全方案？
- 让你实现一个并发 LRU 缓存，怎么设计？

#### 面试回答

先说为什么不能偷懒：只把 `HashMap` 换成 `ConcurrentHashMap` 不够——LRU 的每次访问要同时改哈希表和链表，这是一组必须原子的复合操作，分开保护会暴露“map 里有节点但链表还没摘完”的中间态，链表直接断裂。基线做法是用一把锁（`synchronized` 或 `ReentrantLock`）把 get／put／淘汰整体串行化：正确性最好证明，代价是所有访问争同一把锁。

再说读多写少的真正陷阱：LRU 的“读”不只读，`get` 命中要把节点移到队首，本质是写操作，所以读写锁在这里收益有限——读锁保护不了改链表的动作，硬上 `ReadWriteLock` 很容易写成“多个线程同时在读锁下移动节点”的数据损坏。要优化按代价从低到高排：① 放弃严格 LRU 换近似算法，把读事件先记到每线程缓冲／环形队列，批量回放到访问顺序结构（Caffeine 的思路就是读写分离＋批量处理读事件＋用 W-TinyLFU 代替严格 LRU），把“每次读都改结构”变成“攒着改”；② 分段（striping），按 key 哈希切成若干把独立小 LRU，各自一把锁，竞争按段数下降，代价是容量与淘汰变成“每段近似”而不是全局精确；③ 顺序要求弱时，读路径做无锁快照（位置只周期性重排，或由单个写线程独占重排、读线程读不可变视图）。收口要说清收益与代价同时出现：换成近似算法就换掉了 LRU 的精确语义，分段就让淘汰不公平，并给出验证方式——并发压测看吞吐与锁等待，用双链表自检或不变量断言确认没有断链。

#### 技术细节

分层看成本（与 [[#JUC-014：Java 有哪些实现线程安全的手段？|JUC-014：Java 有哪些实现线程安全的手段]] 的梯度一致）：整锁最简单，单条命令的微秒级临界区在中等并发下就足够，但一个全局锁会让多核吞吐在某个线程数后不再上升；锁内不要做回源、序列化或任何 I/O，否则持锁时间被外部依赖支配，此时应改成“锁外取数据、锁内改结构”或用 `computeIfAbsent` 之类原子复合操作收敛边界（`ConcurrentHashMap` 保证单个方法原子，不保证“先查再放”这种调用序列原子）。

读写锁为什么常不适用：`ReadWriteLock` 的收益来自“读读并行”，前提是读路径真的只读（见 [[#JUC-016：Java 中有哪些锁，如何按不同维度分类？|JUC-016：锁的分类与读写锁边界]]）。访问顺序维护类结构的读会写，读锁形同虚设；而 `StampedLock` 的乐观读只适合“读完再校验有没有被写坏”的场景（读出快照后 `validate`），一旦读要改结构就不成立，且它不可重入、需处理中断。真要用读写锁的形态是：读锁只取 value，顺序更新异步补做——这已经属于近似 LRU。

分段与容量：段内独立容量会让总容量在段间分布不均（热 key 集中在一段时更早淘汰），需要接受或用少量全局统计做二次均衡；淘汰时只在本段找尾节点，锁粒度小但淘汰顺序不全局最优。无锁读的另一条路是“读多写少但顺序可容忍滞后”：节点里放访问计数或时间戳（一次 CAS 更新，不搬动链表），由后台线程周期性按计数重排链表，把结构改动从请求路径移走。

验证与度量的三件事：并发正确性（多写多读压测后遍历校验 map 与链表一致、无环无断链；可用线程交错的小规模随机测试跑不变量）、收益量化（不同段数／是否批量下的吞吐与 P99，以及锁等待与上下文切换）、语义偏差（命中率对比严格 LRU 掉了多少）。只说“加读写锁就快了”不算答案，面试官要的是知道读操作本身是写这个坑。

#### 深挖追问

1. **为什么不直接用 `ConcurrentLinkedQueue` 之类无锁容器拼一个 LRU？**（补充练习）

   无锁容器解决的是单个容器内部的并发，不解决“两个容器要一起变更”的跨容器原子性；LRU 需要按 key O(1) 定位并摘除任意节点，队列不支持，硬做会退化成扫描或再加一层 map，一致性反而更难。

2. **分段锁和 `ConcurrentHashMap` 的分段有什么相同与不同？**（补充练习）

   相同是按 key 哈希收敛竞争粒度；不同是 JDK 8 之后的 `ConcurrentHashMap` 已改用桶级 `synchronized`＋CAS 而不是固定 Segment 分段（见 [[专题题库/Java集合#JAVA-COL-002：HashMap 和 ConcurrentHashMap 有什么区别？|JAVA-COL-002]]），而自研分片 LRU 通常保留显式段数组，段数与段内容量是设计参数。

3. **读事件批量回放会不会让淘汰变慢，怎么处理？**（面经实际出现；[[面经/阿里云/一面/0001#Q04：把 LRU 改造成线程安全该怎么做；读多写少的并发场景下怎么优化线程安全方案？|MJ039 · 阿里云 · 一面 · Q04]] 的延伸）

   会：读事件积压期间顺序是滞后的，可能淘汰掉刚被访问的 key。缓解是限制缓冲长度并设阈值触发排空、对热点 key 单独保护（准入窗口＋频率门槛，这也是 TinyLFU 的准入思想），并用命中率指标验证偏差是否可接受。

**面经来源**

- [[面经/阿里云/一面/0001#Q04：把 LRU 改造成线程安全该怎么做；读多写少的并发场景下怎么优化线程安全方案？|MJ039 · 阿里云 · 一面 · Q04]]

### JUC-020：如何写一个限制最大并发数的并发任务处理器？

**常见问法**

- 手撕：给定 100 个任务 ID，控制最大并发数为 3，模拟并发调用外部接口（如打印 ID）。
- 怎么限制同时执行的线程数／请求数？
- 并发数和 QPS 限制是一回事吗？

#### 面试回答

题意是“同时在飞的任务数不超过 3”，本质就是信号量或固定大小的工作池，两种写法都要会说。Java 版一：固定线程池——`ExecutorService pool = Executors.newFixedThreadPool(3)`，把 100 个任务提交进去，再 `shutdown()` ＋ `awaitTermination(timeout)` 等全部完成；线程数即并发数，代码最短。Java 版二：`Semaphore(3)`——每个任务 `acquire()`、`try/finally` 里 `release()`，配 `CountDownLatch` 或 `CompletableFuture.allOf` 收尾；Java 21 起可以直接用虚拟线程跑 100 个任务，靠许可控制并发（此时线程数不再是约束，许可才是），这版更贴近“并发数”的原意，也便于扩展成“每个任务自己带重试”。Go 版：`errgroup.Group` 加 `SetLimit(3)` 后循环 `g.Go(...)` 再 `g.Wait()`；或手写容量为 3 的带缓冲 channel 当令牌池，启动前 `tokens <- struct{}{}`、完成后 `<-tokens`。

写完要主动补三点，这题的区分度在这里：① 失败与取消——外部接口报错要设重试上限与退避，是否因某个任务失败而取消其余要说清并实现（Go 用 `context` 取消，Java 用 `Future.cancel`／中断），并且已获取的许可与启动的 goroutine／线程必须保证释放，否则并发数会“漏”、最终卡死；② 结果与可观测——要不要保持原顺序（按索引写回定长数组，而不是加锁追加）、统计成功失败数与耗时；③ 限并发不等于限速率——限制同时在飞是最常用的下游自我保护，真要限 QPS 得再加令牌桶或漏桶（见 REDIS-004 的算法口径），两者经常一起用。

#### 技术细节

实现细节的常见错误：`newFixedThreadPool` 用无界队列，任务全进队列只是“线程数受限”，如果任务本身会占大量内存或需要背压，应改用有界队列 ＋ 拒绝策略的 `ThreadPoolExecutor`；`Semaphore` 忘记在 `finally` 释放许可会永久减少可用并发；用 `CountDownLatch(100)` 计数时要保证每个任务一定 `countDown()`（异常路径也要），否则主线程一直等；提交任务与等待完成之间不要关闭线程池两次或漏掉 `shutdown`。

验证方法要能说出口：让模拟调用 `sleep` 固定时长，观察同时处于执行中的数量峰值是否恰为 3（用一个 `AtomicInteger` 记录当前活跃数并取最大值），总耗时是否符合 `ceil(100/3) × 单次耗时` 的量级；再补一个失败注入用例（让第 5、17 个任务抛异常）确认既不死锁也不吞结果。若被追问“100 万个任务怎么办”，方向是不要把 100 万个任务一次性物化再提交——改成流式生产＋固定消费者（生产者-消费者），或分批提交，让队列长度受控。

与线程池的关系值得点一句：固定线程池其实就是“以线程为许可”的信号量实现；两者的差别在弹性——`Semaphore` 允许任务由调用方线程或虚拟线程执行，只约束临界并发度，而线程池同时约束了执行资源。外部接口调用是 IO 密集，JDK 21 上“虚拟线程 ＋ Semaphore”通常比“3 个平台线程”更好：栈成本极低、阻塞不占载体线程，而并发上限仍然由许可精确控制（Go 侧的对应写法就是 goroutine ＋ 带缓冲 channel／errgroup 限流，见 [[专题题库/Go#GO-002：Go 的 channel 是什么、并发安全吗，和 Mutex 怎么取舍？|GO-002]]）。

#### 深挖追问

1. **为什么不直接开 100 个线程？**（补充练习）

   线程本身有栈与调度成本，100 个并发还可能压垮外部接口或被限流，超过下游容量时吞吐反而下降；限并发的意义是“按下游能承受的量调用”，不是“尽量快”。

2. **如果要求“任务提交不能阻塞主线程且要有超时”怎么改？**（面经实际出现；[[面经/字节/一面/0009#Q18：手撕：实现一个并发任务处理器——100 个任务 ID，最大并发数为 3，模拟并发调用外部接口（如打印 ID）。|MJ041 · 字节 · 一面 · Q18]] 的延伸）

   提交侧改成非阻塞：`tryAcquire(timeout)` 拿不到许可就走快速失败或入本地队列缓冲；每个任务配独立超时（`orTimeout`／`context.WithTimeout`），超时后取消并释放许可；整体再给一个 deadline，结束时统计完成／超时／失败三类计数，而不是让主线程无限等待。

3. **多个外部接口共享同一个并发上限怎么设计？**（补充练习）

   按资源分桶：每个下游一个独立 `Semaphore`（避免一个慢接口拖住另一个的额度），或者做一层带优先级的小调度器；若还要限总并发，再套一层父许可并按“先取父后取子、释放反向”的顺序避免死锁。

**面经来源**

- [[面经/字节/一面/0009#Q18：手撕：实现一个并发任务处理器——100 个任务 ID，最大并发数为 3，模拟并发调用外部接口（如打印 ID）。|MJ041 · 字节 · 一面 · Q18]]
