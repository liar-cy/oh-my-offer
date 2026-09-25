# Java并发

- 题号前缀：JUC
- 范围：线程池、锁与线程状态、通信取消、volatile 与交替打印、AQS 队列同步器、线程创建方式、线程安全实现手段、线程顺序执行、锁的分类、CAS 与锁的关系、Go 协程在 Java 的对应方案（虚拟线程）、线程安全 LRU 与读多写少的并发优化、限制最大并发数的任务处理器实现、CopyOnWriteArrayList 线程安全机制与无链表版的取舍、CountDownLatch 与 CyclicBarrier／Semaphore 的区分、synchronized 锁升级机制。
- 最近更新：2026-09-25
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
- [[#JUC-021：CopyOnWriteArrayList 是怎么保证线程安全的？为什么没有 CopyOnWriteLinkedList？|JUC-021：CopyOnWriteArrayList 是怎么保证线程安全的？为什么没有 CopyOnWriteLinkedList？]]
- [[#JUC-022：CountDownLatch 解决什么问题，和 CyclicBarrier、Semaphore 怎么区分？|JUC-022：CountDownLatch 解决什么问题，和 CyclicBarrier、Semaphore 怎么区分？]]
- [[#JUC-023：分段锁＋编程式事务怎么配合实现（库存分桶案例）？|JUC-023：分段锁与编程式事务的配合实现]]
- [[#JUC-024：synchronized 的锁升级机制是什么？|JUC-024：synchronized 的锁升级机制是什么？]]

### JUC-001：ThreadLocal 是什么，有什么问题？

**常见问法**

- 什么是 ThreadLocal？
- ThreadLocal 有什么问题？
- ThreadLocal 哪部分数据可能会存在内存泄露？

- ThreadLocal 的底层原理是什么？
- ThreadLocalMap 是怎么存储和寻址的？
- ThreadLocal 主要拿来做什么？希望跨线程传递 ThreadLocal，有哪些方法？

#### 面试回答

ThreadLocal 是给每个线程各自存一份变量值，用来放线程内的上下文；它不是给共享对象加锁的那种手段。

- 两类常见问题：线程池复用导致上下文串用；值没及时清理造成内存滞留。
- 用法：放请求上下文时，要在 `finally` 里 `remove`。
- 边界：换到另一个线程执行，不会自动拿到普通 ThreadLocal 的值。

#### 技术细节

**结构与引用强度（以 OpenJDK 17 为例）**

- 每个 `Thread` 内部持有一个 `ThreadLocalMap`。
- 条目的 key 是对 `ThreadLocal` 的弱引用，value 是强引用。
- key 被回收后，value 仍可能跟着长寿命的线程留下来；map 内部的清理不是一个及时性保证。
- 反过来，如果那个 `ThreadLocal` 还被静态字段引用着，key 也不会凭空消失。
- `remove()` 必须在用到这个值的同一个线程里调用。

**值到底存在哪里（被追问“底层原理”时要讲到这一层）**

- 值不存在 `ThreadLocal` 对象里，而是存在每个 `Thread` 的 `threadLocals` 字段所指向的 `ThreadLocalMap` 中（`inheritableThreadLocals` 另有一份）。
- `ThreadLocal` 实例本身只是用来定位的 key。

**槽位怎么算出来**

- `set` 的首次路径是懒创建 map，然后用 `threadLocalHashCode & (len - 1)` 定位槽位。
- 哈希值来自一个内部常量增量（`0x61C88647`，黄金分割数），每来一个新 `ThreadLocal` 就累加一次，目的是让同一线程里多个 `ThreadLocal` 的下标分散均匀。
- 表长恒为 2 的幂，所以取模可以用位与代替。

**冲突怎么处理**

- 不用链表，而是开放寻址里的线性探测：命中已占用的槽就继续看下一个；遇到 key 为 `null` 的过期槽位，直接复用并顺手清理。
- `get` 走的是同一条探测链。
- 负载因子约 2/3 时触发 `rehash`，重建后按新的表长重新散布。

**为什么它不需要加锁**

- 整张表只属于当前线程，所以读写都不用加锁。
- 这正是它“快”的机制，也正是它只提供线程封闭、不提供共享可见性的原因。
- `get` 的快路径是从自己线程的 map 直接取；只有慢路径（map 还没创建）才会回退到 `getMap`／初始化。

**继承与共享的两条边界**

- `InheritableThreadLocal` 是在创建子线程时继承的，所以它不能自动解决线程池里的任务传播问题。
- 把同一个可变对象放进不同线程的本地槽位，也不会让这个对象变得线程安全。

#### 深挖追问

1. **为什么弱引用 key 仍可能内存滞留？**（补充练习）

   弱引用只作用在 key 上。线程到 map 再到 value 这条强引用链仍然可能一直活着，所以必须按生命周期自己去清理。

2. **为什么线程池特别容易串上下文？**（补充练习）

   因为线程比请求活得更久，下一个任务可能拿到上一个任务残留的值。做法是：设置之前先定好默认值，用完在 `finally` 里清理。

3. **哪部分数据可能会存在内存泄露？**（面经实际出现；[[面经/帆软/二面/0002#Q03：ThreadLocal 哪部分数据可能存在内存泄露？|MJ035 · 帆软 · 二面 · Q03]]）

   泄露的是 `ThreadLocalMap` 里 key 已经失效的那个 Entry 的 value。

   - 原因：key 是弱引用，被回收之后，线程 → Map → Entry → value 这条强引用链还在，value 就释放不掉；线程池里长寿命的线程会让它一直留着。
   - 不泄露的部分：被回收的只是 `ThreadLocal` 对象本身。
   - 防御手段还是那一条：在同一个线程里、在 `finally` 中 `remove`。

4. **除了内存泄漏，ThreadLocal 还会引发什么问题？**（面经实际出现；[[面经/小红书/一面/0001#Q06：ThreadLocal 是什么、底层原理如何，可能导致什么问题？|MJ040 · 小红书 · 一面 · Q06]]）

   三类：

   - 脏上下文与串号：线程池复用导致，比泄漏更致命，比如用户身份错乱。
   - 传播失效：`InheritableThreadLocal` 在池化线程上只在线程创建时复制一次，任务提交时不会传播，要靠显式捕获回放（如 TransmittableThreadLocal）。
   - 副本语义被误解：子线程拿到的是同一个对象引用，改内部状态照样互相影响。ThreadLocal 只保证“槽位隔离”，不保证“对象线程安全”。

**面经来源**

- [[面经/小红书/一面/0001#Q06：ThreadLocal 是什么、底层原理如何，可能导致什么问题？|MJ040 · 小红书 · 一面 · Q06]]

- [[面经/北京某上市公司/一面/0001#Q01：ThreadLocal 是什么，有哪些问题？|MJ004 · 北京某上市公司 · 一面 · Q01]]
- [[面经/帆软/二面/0002#Q03：ThreadLocal 哪部分数据可能存在内存泄露？|MJ035 · 帆软 · 二面 · Q03]]
- [[面经/收钱吧/一面/0001#Q17：ThreadLocal 主要做什么？跨线程传递 ThreadLocal 有哪些方法？|MJ068 · 收钱吧 · 一面 · Q17]]

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

`ThreadPoolExecutor` 常见的七个参数：corePoolSize、maximumPoolSize、keepAliveTime、unit、workQueue、threadFactory、handler。

任务来了怎么走（四步）：

- 先扩到核心线程数。
- 核心数满了，再尝试入队。
- 队列容纳不了，才扩到最大线程数。
- 还是接收不下，就走拒绝策略。

一句澄清：`keepAliveTime` 管的是空闲线程的回收，不是任务的最大执行时间，也不是排队时间。

#### 技术细节

这里按 Java 17 的 `ThreadPoolExecutor` 讲。

**核心线程数不是身份标签**

- 核心数和最大数都只是工作线程数量的阈值。
- Worker 身上没有一张永久的“核心线程身份证”。
- 创建线程时用不同的上限来判断，取任务时再决定回收策略。

**空闲线程怎么决定要不要超时**

- 取任务时看 `allowCoreThreadTimeOut`，或者看当前 `workerCount` 是否大于 `corePoolSize`。
- 据此选择带超时的 `poll`，还是不带超时的 `take`。
- 默认会保留核心数量；开启核心线程超时之后，也可以缩到零。
- 线程真正退出前，还要再检查池的状态和队列。

| 参数 | 含义 |
| --- | --- |
| corePoolSize | 通常优先创建到的工作线程数 |
| maximumPoolSize | 入队失败后仍可扩展到的上限 |
| keepAliveTime / unit | 可超时工作线程等待新任务的空闲期限及单位 |
| workQueue | 等待执行的任务队列 |
| threadFactory | 创建工作线程，配置名称等 |
| handler | 无法接收任务时的拒绝处理 |

**流程之外还要补的三点**

- 队列选无界时，最大线程数通常就发挥不出作用了。
- `execute` 入队之后还会复查关闭状态，并且在没有工作线程时补建一个。不能只背那四步，把并发关停漏掉。
- 三个“超时”别混：`Future.get(timeout)` 只限制调用者等多久拿到结果；`awaitTermination` 限制的是等待池终止的时间；这两个都不是 `keepAliveTime`。
- 如果业务要求的是排队期限或任务总时限，那得自己设计 deadline 加协作取消。

**线程池到底解决什么**

- 复用平台线程、摊薄创建成本。
- 集中管理任务生命周期。
- 限制在途工作量，保护 CPU、内存和下游。
- 注意：池不是 Java 跑并发任务的必要条件。池大小、队列和拒绝策略都要跟负载匹配，线程开太多反而会增加切换和资源竞争。

#### 深挖追问

1. **线程池最大等待时间是什么？**（面经实际出现；[[面经/北京某上市公司/一面/0001#Q04：线程池最大等待时间是什么，核心与非核心线程如何区分？|MJ004 一面 Q04]]）

   原题没指明是哪个 API，所以要先反问口径：

   - 若指 `keepAliveTime`，那是线程空闲等着新任务的期限。
   - 若指 `get(timeout)` 或 `awaitTermination`，两者的等待对象不同，要分开说明。

2. **线程池如何区分核心和非核心线程？**（面经实际出现；[[面经/北京某上市公司/一面/0001#Q04：线程池最大等待时间是什么，核心与非核心线程如何区分？|MJ004 一面 Q04]]）

   它不是给每个 Worker 打一个永久分类标签，而是按当前数量和回收策略来判断。核心与非核心的差别，首先体现在容量和生命周期策略上。

3. **Java 为什么需要线程池？**（面经实际出现；[[面经/百度/一面/0006#Q17：Java 为什么需要线程池？|MJ010 · 百度 · 一面 · Q17]]）

   在常见的平台线程服务里，它用来复用线程、控制并发并统一调度。但这也并不意味着每种任务都必须走线程池。

4. **如何设计让高优先级任务先执行的线程池？**（面经实际出现；[[面经/帆软/二面/0002#Q09：线程池有哪些核心参数，如何设计让高优先级任务先执行？|MJ035 · 帆软 · 二面 · Q09]]）

   做法：把 `workQueue` 换成 `PriorityBlockingQueue`，任务实现 `Comparable`——优先级作主键、提交序号作次键来保证同级 FIFO，这样出队时天然先取到高优先级任务。

   三个坑要一起说：

   - 这个队列是无界的，导致 `maximumPoolSize` 几乎不生效。
   - 它只影响排队顺序，不会抢占已经在跑的低优先级任务。
   - 低优先级任务会饥饿，需要老化提权，或者用双池隔离来保底。

5. **怎么创建线程池，拒绝策略有哪些？**（面经实际出现；[[面经/美团/一面/0003#Q05：为什么要有线程池？怎么创建线程池，拒绝策略有哪些？|MJ037 · 美团 · 一面 · Q05]]）

   创建：正解是显式写 `new ThreadPoolExecutor(...)` 七个参数。`Executors` 的快捷工厂只是包装，而且有隐患——`newFixedThreadPool`／`newSingleThreadExecutor` 用无界队列，任务积压可能 OOM；`newCachedThreadPool`／`newScheduledThreadPool` 的线程数上限是 `Integer.MAX_VALUE`，可能线程爆炸（阿里手册口径是禁用）。Spring 的 `ThreadPoolTaskExecutor` 则是带生命周期管理的包装。

   拒绝策略有四种：

   - `AbortPolicy`：默认，直接抛异常。
   - `CallerRunsPolicy`：由提交方线程执行，顺便形成背压。
   - `DiscardPolicy`：静默丢掉新任务。
   - `DiscardOldestPolicy`：弹掉队首再试一次。

   生产实践：自定义 `RejectedExecutionHandler` 做日志＋报警＋降级入队／持久化补偿。选内置策略之前，先确认这类任务真的可以丢。

6. **核心 8、最大 10 都在跑，新任务能否不入队、先把线程跑满再排队？**（面经实际出现；[[面经/微步在线/一面/0001#Q11：线程池有哪些核心参数？能否不入队、先把线程跑满再入队？|MJ043 · 微步在线 · 一面 · Q11]]）

   默认流程改不了顺序：`execute` 写死“核心 → 入队 → 扩到最大”，想跳过入队只能改造队列。两种做法：

   - 换 `SynchronousQueue`：它不存储任务、只做直接交接。核心线程都忙时入队即失败，线程池就会继续建线程到 max；建满后再来任务走拒绝。代价是失去缓冲，突发直接触顶。
   - 自定义队列重写 `offer()`：当前工作线程数没到 `maximumPoolSize` 就返回 false，逼线程池先建线程；跑满了 offer 才真正入队（《Java 并发编程艺术》的公开做法，入队失败后要注意任务别丢，需重试入队）。

   收尾补设计意图：默认“先入队”是因为平台线程创建与切换贵，宁可排队也别放大线程数；要不要先扩线程，取决于任务是 IO 密集（等待多、可多开）还是 CPU 密集（开了更抢核）。

**面经来源**

- [[面经/百度/一面/0006#Q17：Java 为什么需要线程池？|MJ010 · 百度 · 一面 · Q17]]

- [[面经/北京某上市公司/一面/0001#Q04：线程池最大等待时间是什么，核心与非核心线程如何区分？|MJ004 · 北京某上市公司 · 一面 · Q04]]
- [[面经/北京某上市公司/二面/0001#Q08：线程池有哪些参数，执行流程是什么？|MJ004 · 北京某上市公司 · 二面 · Q08]]
- [[面经/帆软/二面/0002#Q09：线程池有哪些核心参数，如何设计让高优先级任务先执行？|MJ035 · 帆软 · 二面 · Q09]]
- [[面经/美团/一面/0003#Q05：为什么要有线程池？怎么创建线程池，拒绝策略有哪些？|MJ037 · 美团 · 一面 · Q05]]
- [[面经/微步在线/一面/0001#Q11：线程池有哪些核心参数？能否不入队、先把线程跑满再入队？|MJ043 · 微步在线 · 一面 · Q11]]
- [[面经/收钱吧/一面/0001#Q16：线程池有哪些核心参数？每个参数的作用？|MJ068 · 收钱吧 · 一面 · Q16]]
- [[面经/钉钉/电话面/0001#Q10：谈谈对 Java 多线程／高并发的认识，以及如何创建、使用、什么场景用？|MJ053 · 钉钉 · 电话面 · Q10]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 ThreadPoolExecutor API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)
- [OpenJDK 17u ThreadPoolExecutor 源码](https://raw.githubusercontent.com/openjdk/jdk17u/master/src/java.base/share/classes/java/util/concurrent/ThreadPoolExecutor.java)

### JUC-003：synchronized 和 Lock 有什么区别？

**常见问法**

- synchronized 和 Lock 有什么区别？

- synchronized 和 ReentrantLock 是干啥的，区别？

- 用过什么锁？synchronized 底层是什么，和对象头什么关系？

- 为什么选择加 synchronized 锁呢？而不是其它的锁？
- synchronized 和 ReentrantLock 的区别？
- 谈谈 synchronized 的原理（底层实现、和对象头的关系）。

#### 面试回答

- `synchronized`：语言级的监视器锁，退出同步块时自动释放。
- `Lock`：是一个接口，通常拿 `ReentrantLock` 来对比，必须在 `finally` 里显式 `unlock`。
- 相同点：两者都能提供互斥和可见性。
- 多出来的能力：`ReentrantLock` 支持可中断获取、限时尝试、公平选项，还能有多个 `Condition`。

选型就一句：普通互斥优先选写法清晰的 `synchronized`，确实需要上面那些能力才换显式锁。

#### 技术细节

**先把比较对象说清**

- 不能把 `Lock` 接口直接等同于“所有实现都可重入”。本题的比较对象是 `synchronized` 与 `ReentrantLock`。
- 这两者都可重入；同一把锁的解锁和后续成功加锁之间，会建立起相应的内存可见性。

**等待时能不能被中断**

- 等 `synchronized` 进监视器，不能用 interrupt 直接取消。
- `lockInterruptibly()` 可以。但要注意：`lock()` 本身不是可中断获取。

**两句别说过头的话**

- 公平性、`tryLock`、`Condition` 讲的是行为差异；性能要结合具体版本和竞争程度去测量。
- 没有成功拿到锁的时候，不要去调 `unlock`。

#### 深挖追问

1. **如何确保 Lock 异常时释放？**（补充练习）

   成功 `lock` 之后立刻进入 try，在 `finally` 里 `unlock`。用 `tryLock` 时，只有在返回 `true` 之后才需要解锁。

2. **synchronized 一定比 ReentrantLock 慢吗？**（补充练习）

   没有通用结论。JVM 的优化、竞争模式、临界区长度都会影响结果，所以应当按自己需要的语义来选，而不是按“谁更快”。

3. **synchronized 能锁 String 或 Long 对象吗？**（面经实际出现；[[面经/携程/一面/0001#Q08：synchronized 能锁住 String 和 Long 对象吗？|MJ028 · 携程 · 一面 · Q08]]）

   语法上能——任何引用对象都能锁，`long` 会先装箱。但两种都不该用：

   - String：字面量会进常量池，内容相同的字符串全局就是同一把锁，会把不相干的业务意外互斥掉，而且还可能跟第三方库共用同一个监视器。
   - Long／Integer：包装类只有 −128～127 有缓存，超出范围每次装箱都是新对象。两个线程锁“相同的值”，很可能各锁各的，互斥静默失效。包装类本身又不可变，更新值等于换了锁对象。

   正确做法：用专有的 `private final Object lock`，或者按业务键做分段锁，并且自己掌控锁对象的生命周期。

4. **synchronized 底层是怎么实现的，和对象头什么关系？**（面经实际出现；[[面经/传音控股/二面/0001#Q03：用过什么锁？synchronized 底层和对象头？|MJ030 · 传音控股 · 二面 · Q03]]）

   分两层说。

   - 字节码层：同步代码块编译成 `monitorenter`／`monitorexit`（靠异常表保证一定能退出）；方法级则用 `ACC_SYNCHRONIZED` 标志。
   - 运行层：锁状态记在对象头的 Mark Word 里（这一字与哈希码、分代年龄复用）。演进链条是无锁 → 轻量级锁 → 重量级锁。轻量级锁是用 CAS 把 Mark Word 拷进线程栈上的锁记录，靠自旋升级；竞争加剧后膨胀成重量级锁，依赖操作系统 mutex 的 monitor，没抢到的人直接阻塞。
   - 历史上还有偏向锁，因为撤销成本和收益不匹配，已在 JDK 15 起废弃（JEP 374）。

   口径提示：以上都是 HotSpot 的实现细节，JVMS 只规定语义。答题时要注明版本，并按实现来说明。

**面经来源**

- [[面经/北京某上市公司/一面/0001#Q05：synchronized 和 Lock 有什么区别？|MJ004 · 北京某上市公司 · 一面 · Q05]]
- [[面经/帆软/二面/0001#Q14：分布式场景下乐观锁、synchronized 锁的注意点，死锁、锁超时释放怎么处理？|MJ023 · 帆软 · 二面 · Q14]]
- [[面经/携程/一面/0001#Q07：synchronized 和 ReentrantLock 是干啥的，区别？|MJ028 · 携程 · 一面 · Q07]]
- [[面经/途虎养车/一面/0001#Q04：synchronized 和 ReentrantLock 的区别|MJ070 · 途虎养车 · 一面 · Q04]]
- [[面经/携程/一面/0001#Q08：synchronized 能锁住 String 和 Long 对象吗？|MJ028 · 携程 · 一面 · Q08]]
- [[面经/传音控股/二面/0001#Q03：用过什么锁？synchronized 底层和对象头？|MJ030 · 传音控股 · 二面 · Q03]]
- [[面经/美团/一面/0003#Q07：synchronized 和 ReentrantLock 的区别。|MJ037 · 美团 · 一面 · Q07]]
- [[面经/京东零售/一面/0001#Q07：Java 中有哪些锁机制？|MJ034 · 京东零售 · 一面 · Q07]]
- [[面经/拼多多/一面/0002#Q05：项目里为什么用 synchronized 加锁？锁的机制保护什么？为什么选 synchronized 而不是其它锁？|MJ044 · 拼多多 · 一面 · Q05]]
- [[面经/帆软/一面/0004#Q03：乐观锁和悲观锁分别有什么特点？各自在什么场景下使用？|MJ050 · 帆软 · 一面 · Q03]]
- [[面经/海信/电话面/0001#Q08：谈谈 synchronized 的原理|MJ073 · 海信 · 电话面 · Q08]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 ReentrantLock API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html)
- [Java 17 语言规范：线程与锁](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html)

### JUC-004：synchronized 修饰普通方法和静态方法有什么区别？

**常见问法**

- synchronized 加在普通方法和静态方法上有什么区别？

#### 面试回答

锁的对象不一样：

- 普通实例同步方法：锁的是 `this`。
- 静态同步方法：锁的是声明这个方法的类所对应的那个 Class 对象。

由此推出三句结论：

- 同一个实例上的同步方法互斥，不同实例之间通常不互斥。
- 同一个 Class 的静态同步方法彼此互斥。
- 实例锁和类锁是两个不同的对象，所以它们之间不会自动互斥。

#### 技术细节

**判断标准只有一个：锁对象身份是否相同**

- 跟方法名无关，也跟“是不是访问了同一个字段”无关。
- 普通同步方法等价于围绕方法体 `synchronized (this)`；静态同步方法锁的是声明类的 Class。
- 一个边界：同名的类如果由不同的类加载器定义，它们的 Class 对象也不同，那就不是同一把锁。

**两个常见误用**

- 多个实例访问同一个 `static` 可变字段，却各自锁自己的 `this`——这份共享状态根本没被保护住。
- 没加同步的其他访问路径，也不会被这两类锁自动约束。

#### 深挖追问

1. **两个不同对象调用静态同步方法会互斥吗？**（补充练习）

   会。只要实际调用的是同一个声明类上的静态方法，锁的就是同一个 Class。用不同的对象表达式去访问它，并不会改变锁的身份。

2. **静态与实例同步方法怎样才能互斥？**（补充练习）

   得显式用同一个锁对象去保护对应的临界区，并且保证所有相关的访问都走这一套协议。靠 `this` 和 Class 自动互斥是不存在的。

3. **`synchronized (obj)` 里能锁 String 或 Integer 对象吗？**（面经实际出现；[[面经/微步在线/一面/0001#Q09：synchronized 修饰普通方法和静态方法有什么区别？锁代码块能锁 String 或 Integer 对象吗？|MJ043 · 微步在线 · 一面 · Q09]]）

   语法上锁任何对象引用都行，但这两类都不能用，问题出在“锁对象身份不受你控制”：

   - String：字面量驻留在字符串池，全程序同内容常量共享一把锁——不相关的模块被隐式串联，轻则无谓互斥，重则凑出死锁；换成 `new String` 又每次都是新对象，各锁各的，形同没锁。
   - Integer：`valueOf` 在 -128～127 返回缓存对象，锁会被意外共享；超出范围每次装箱是新对象。更根本的是数值一变、引用就换，锁保护不住共享状态。
   - 正确做法：`private final Object lock = new Object()`，专锁专用、不对外暴露、不再赋值。锁 Boolean／Long 等包装类同理（见 JAVA-005 缓存）。

**面经来源**

- [[面经/北京某上市公司/一面/0001#Q06：synchronized 加在普通方法和静态方法上有什么区别？|MJ004 · 北京某上市公司 · 一面 · Q06]]
- [[面经/微步在线/一面/0001#Q09：synchronized 修饰普通方法和静态方法有什么区别？锁代码块能锁 String 或 Integer 对象吗？|MJ043 · 微步在线 · 一面 · Q09]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 语言规范：线程与锁](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html)

### JUC-005：ReentrantLock 如何实现公平锁与非公平锁？

**常见问法**

- Lock 怎么实现公平和非公平？
- 你刚才讲了好多概念，又是公平锁、非公平锁……谈一谈 ReentrantLock 的公平锁和非公平锁。

#### 面试回答

底层都是 AQS 的状态加等待队列：state 记录持有次数，所以支持重入。

- 公平模式：即使锁可以获取，也还要看前面有没有排队的等待者。
- 非公平模式：允许新来的线程直接去抢空闲锁。

两个别搞混的点：

- 两种模式抢不到锁之后都会排队，非公平锁并不是没有队列。
- 公平也只是锁内部的排队公平，不保证操作系统调度绝对公平。

#### 技术细节

**怎么选模式**

- 构造器传 `true` 就是公平策略，不传默认是非公平。

**一次获取发生了什么（按典型 AQS 实现理解）**

- 抢空闲状态用 CAS。
- 已经持有的线程再进来，只是把重入计数加一。
- 计数释放到零之后，才允许别的线程拿到锁。

**公平到底“公平”在哪**

- 关键在获取资格的检查，而不是把线程的启动顺序严格排序。

**一个容易答错的 API 差异**

- 无超时的 `tryLock()` 即使在公平的 `ReentrantLock` 上也可以插队。
- 带超时的尝试才遵循对应的公平规则。

**收益与代价**

- 公平通常拿吞吐换饥饿风险下降，但它并不保证每个等待者都能在固定时间内执行到。

#### 深挖追问

1. **公平锁的 tryLock 一定公平吗？**（补充练习）

   不一定。无超时的 `tryLock` 明确不遵守公平排队策略，所以不能把构造器那个选项当成所有 API 的统一规则。

2. **持有者重入需要重新排队吗？**（补充练习）

   不需要。重入只是把持有计数加一。真要重新排队，就会变成自己等自己了。

**面经来源**

- [[面经/北京某上市公司/一面/0001#Q07：Lock 怎么实现公平和非公平？|MJ004 · 北京某上市公司 · 一面 · Q07]]
- [[面经/美团/一面/0003#Q08：ReentrantLock 底层原理；AQS 怎么实现、如何保证 state 的原子性？|MJ037 · 美团 · 一面 · Q08]]
- [[面经/京东零售/一面/0001#Q08：ReentrantLock 内部靠什么核心结构来保护共享资源？|MJ034 · 京东零售 · 一面 · Q08]]
- [[面经/拼多多/一面/0002#Q08：谈谈 ReentrantLock 的公平锁和非公平锁。|MJ044 · 拼多多 · 一面 · Q08]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 ReentrantLock API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html)

### JUC-006：Java 线程有哪些状态？

**常见问法**

- 线程有哪些状态？

#### 面试回答

`Thread.State` 一共六种：NEW、RUNNABLE、BLOCKED、WAITING、TIMED_WAITING、TERMINATED。

- RUNNABLE 范围比字面意思宽，既包括正在运行，也包括等待 CPU 等情况。
- BLOCKED 是特指等待进入 `synchronized` 监视器。
- `wait`、`join`、`park` 这些会进入等待状态；带了时间限制就可能是 TIMED_WAITING。
- 别背错：这六种和操作系统线程状态不是一一映射的关系。

#### 技术细节

**两端的状态**

- `start` 之前是 NEW；`start` 之后才可能被调度执行。
- `run` 不管是正常返回还是异常结束，之后都是 TERMINATED，而且不能再次 `start`。

**几个容易判错的状态跳转**

- 被 `notify` 唤醒之后，还要重新去获取监视器，所以这时可能转为 BLOCKED。
- 在 `ReentrantLock` 竞争中 park 住的线程通常是 WAITING。别一概而论成“只要在等锁就是 BLOCKED”。

**诊断时怎么用状态**

- 线程状态只是那一瞬间的采样。要判断问题，还得结合堆栈、持锁者和 CPU，不能只凭一个状态就断言死锁。

#### 深挖追问

1. **直接调用 run 会产生新线程吗？**（补充练习）

   不会。直接调 `run` 只是当前线程的一次普通方法调用；只有 `start` 才会启动新的执行线程。

2. **RUNNABLE 就一定占满 CPU 吗？**（补充练习）

   不一定。它并不等价于“操作系统此刻正在运行这个线程”，要结合 CPU 采样来判断。

**面经来源**

- [[面经/北京某上市公司/二面/0001#Q05：线程有哪些状态？|MJ004 · 北京某上市公司 · 二面 · Q05]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 Thread.State](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Thread.State.html)

### JUC-007：sleep 和 wait 有什么区别？

**常见问法**

- sleep 和 wait 有什么区别？

#### 面试回答

两个方法的归属和用途都不同：

- `sleep`：`Thread` 的静态方法，让当前线程暂停一段时间，不释放已经持有的监视器。
- `wait`：`Object` 的方法，调用者必须先持有这个对象的监视器；等待期间会释放它，返回前再重新获取。

一句话分工：`wait` 用来做条件协作，`sleep` 主要就是暂停时间。别指望靠 `sleep` 建立线程之间的可见性。

#### 技术细节

**释放的是哪一把锁**

- `wait` 只释放目标对象的监视器，线程手里其他锁一把都不会放。
- `notify`／`notifyAll` 同样要先持有这个监视器；而且通知完不会立刻把锁交出去。

**为什么必须写 while**

- `wait` 可能被虚假唤醒，所以检查条件要用 `while`，不能用 `if`。
- 被通知或者超时了，都不代表业务条件一定成立。

**中断怎么处理**

- 两个方法都可能因为中断抛 `InterruptedException`。
- 常规做法是往上抛，或者恢复中断标志之后按取消协议退出。不要把异常吞掉再继续忙等。

#### 深挖追问

1. **为什么 wait 必须写在条件循环里？**（补充练习）

   因为可能虚假唤醒，也可能条件被别的线程先消费掉了。重新拿到锁之后必须再检查一遍，这就是要写循环的原因。

2. **notify 后等待线程立刻运行吗？**（补充练习）

   不保证。它还得先去竞争锁，然后等调度。而通知者在离开同步块之前，仍然是持锁的。

**面经来源**

- [[面经/北京某上市公司/二面/0001#Q06：sleep 和 wait 有什么区别？|MJ004 · 北京某上市公司 · 二面 · Q06]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 Object API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Object.html)
- [Java 17 语言规范：线程与锁](https://docs.oracle.com/javase/specs/jls/se17/html/jls-17.html)

### JUC-008：线程之间有哪些通信与协作方式？

**常见问法**

- 线程通信方式有哪些？

- 线程之间是怎么通信的？

- 共享屏幕用自己的 IDE 写一个简单的 Java 线程通信 demo。

#### 面试回答

按用途分四类手段：

- 表达状态：共享状态配合同步，或者用 `volatile`、原子类。
- 等待条件：`wait`／`notify`、`Condition`、`park`／`unpark`。
- 传递数据：`BlockingQueue`。
- 拿结果、协调阶段：`Future` 拿结果，`CountDownLatch` 之类协调阶段。

选的时候三件事一起考虑：数据可见性、原子性、等待机制。别用“循环读普通变量”或者 `sleep` 凑合。

#### 技术细节

**各自的能力和边界**

- `volatile`：适合把状态发布出去，但 `i++` 这类复合操作仍然不是原子的。
- 队列：能把“放入之前”的那些动作安全发布给消费线程，而且可以用有界容量形成背压。
- `wait`／`Condition`：要把状态检查和等待绑定在同一套同步协议里，否则会丢通知。
- `park`／`unpark`：许可最多保留一个，它不是可以累加的计数信号量。
- `Future`：用来传结果和异常。

**协调工具怎么选**

- 一次性的完成事件用 latch；需要重复的阶段协作，可以考虑 barrier／phaser。

**一条通用注意**

- 共享对象发布出去之后，也应避免无同步地修改它。

#### 深挖追问

1. **先 unpark 再 park 会丢通知吗？**（补充练习）

   不会丢，但也不累加。许可可以保留下来，下一次 `park` 就能消费它；不过多次 `unpark` 不会攒出多个许可，所以外面仍然要写条件循环。

2. **生产者消费者用什么最直接？**（补充练习）

   通常直接用有界的 `BlockingQueue`。同时要把队列满了之后的行为定清楚：是阻塞、超时，还是走拒绝策略。

3. **现场手写线程通信 demo，最短写法和讲评点是什么？**（面经实际出现；[[面经/韶音科技/二面/0001#Q06：共享屏幕，用自己的 IDE 写一个简单的 Java 线程通信 demo。|MJ047 · 韶音科技 · 二面 · Q06]]）

   最短可靠形态：共享锁对象＋布尔标志，`synchronized` 内 `while` 检查条件→`wait()`，干活后翻标志→`notifyAll()`；参考实现见 [[面经/韶音科技/二面/0001#Q06：共享屏幕，用自己的 IDE 写一个简单的 Java 线程通信 demo。|MJ047 二面 Q06]] 的双线程交替打印。

   边写要边说三件事：`wait` 释放锁而 `sleep` 不释放；条件必须 `while` 复查防虚假唤醒与丢通知；两个线程锁的是同一个对象才谈得上通信。写完主动给升级路径：生产用有界 `BlockingQueue` 或 `Lock`＋双 `Condition` 精确唤醒（交替打印进阶见 JUC-011）。

**面经来源**

- [[面经/北京某上市公司/二面/0001#Q07：线程通信方式有哪些？|MJ004 · 北京某上市公司 · 二面 · Q07]]
- [[面经/美团/一面/0001#Q05：线程之间怎么通信？进程间的通信方式呢？|MJ031 · 美团 · 一面 · Q05]]
- [[面经/韶音科技/二面/0001#Q06：共享屏幕，用自己的 IDE 写一个简单的 Java 线程通信 demo。|MJ047 · 韶音科技 · 二面 · Q06]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 BlockingQueue](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/BlockingQueue.html)
- [Java 17 LockSupport](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/locks/LockSupport.html)

### JUC-009：线程池提交的任务能取消吗？

**常见问法**

- 线程池中提交的任务能取消吗？

#### 面试回答

能取消，但只是“请求”。`submit` 拿到 `Future` 之后可以调 `cancel`：

- 任务还没开始、而且取消成功：它就不会再执行了。
- 任务已经在跑、用 `cancel(true)`：通常只是给工作线程发一个中断请求。任务必须自己协作响应，不保证立刻停下来。
- `cancel(false)`：不请求中断，任务可能继续跑完。
- 最后一条边界：`Future` 显示已取消，并不代表业务副作用已经回滚。

#### 技术细节

以下以 `ThreadPoolExecutor.submit` 返回的 `FutureTask` 为背景，不要把所有 `Future` 实现当成一样的。

**API 层面要注意的**

- 取消和“启动”“完成”之间存在竞争，所以一定要检查 `cancel` 的返回值。
- 取消之后再调 `get`，抛的是 `CancellationException`。

**任务内部怎么配合**

- 循环体里要检查中断；能被中断的阻塞，就老老实实处理 `InterruptedException`。
- 有一部分 I/O 不吃中断，还需要关闭资源或者用专门的超时。

**三个别高估取消的地方**

- 队列里那些已取消的包装器不一定会被立刻物理移除，需要的话可以自己调 `purge`。
- `get(timeout)` 超时并不会自动取消任务。
- `shutdownNow` 也不等于把工作线程安全强杀掉。

#### 深挖追问

1. **cancel(true) 返回 true 就说明线程已经退出吗？**（补充练习）

   不是。返回 `true` 只说明取消状态转移成功了。业务如果不响应中断，照样还在跑；要确认它真的退出了，得用你自己的完成信号去验证。

2. **已经写入数据库的数据会自动撤销吗？**（补充练习）

   不会。取消线程任务和数据库事务回滚是两套机制。要按事务边界和幂等策略自己处理。

**面经来源**

- [[面经/北京某上市公司/二面/0001#Q12：线程池中提交的任务能取消吗？|MJ004 · 北京某上市公司 · 二面 · Q12]]

**参考资料**（本次查证：2026-09-12）

- [Java 17 Future API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/Future.html)
- [Java 17 ThreadPoolExecutor API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)

### JUC-010：volatile 有什么作用和局限？

**常见问法**

- volatile 有什么作用？
- 介绍一下内存屏障的工作原理。
- `volatile int a;` 然后执行 `a++`，是线程安全的吗？
- volatile 的原理是什么（内存屏障／可见性怎么实现）？
- volatile 的原理是什么（内存屏障／可见性怎么实现）？

#### 面试回答

`volatile` 管两件事：共享变量读写的可见性，以及相应的有序性。规则一句话：对某个 volatile 变量的写，happens-before 后续对它的读。

- 适合：发布状态、发布不可变快照的引用。
- 不适合：别指望它把 `i++`、“先检查再更新”这类复合操作自动变原子。

#### 技术细节

**安全发布为什么成立**

- 按 Java SE 21 的 JMM（Java 内存模型）理解：发布线程在这次 volatile 写之前的那些操作，可以经由这条同步关系，对被读取线程可见。

**内存屏障的实现口径**

- volatile 写前后、读前后由编译器插入屏障，禁止跨屏障重排序：LoadLoad／StoreStore 管普通读写与 volatile 的相互乱序，LoadStore 防 volatile 读与后续普通写乱序，StoreLoad 最重——volatile 写之后再读别的变量必须看到最新值。
- x86 是强内存序（TSO）：前三类基本免费，StoreLoad 用带锁指令／`mfence` 类代价补齐；写 volatile 同时经缓存一致性协议让旧副本失效，可见性与有序性是一套机制两面。
- 口径提醒：屏障语义以 JMM 规范为准，具体指令映射是 HotSpot／平台实现，答完主动声明版本与平台差异。

**两条边界**

- volatile 修饰的是引用本身。被引用对象后来发生的任意修改，并不会因此就变成安全操作。
- 涉及多个变量的不变量，要用锁，或者用合适的原子协议来维护。

#### 深挖追问

1. **volatile 能替代锁吗？**（补充练习）

   简单的状态发布可以。但只要需要互斥更新，或者要维护多个字段之间的一致性，就不能用它直接替代锁。

2. **`volatile int a`，两个线程各执行一次 `a++`，结果一定等于 2 吗？**（面经实际出现；[[面经/拼多多/二面/0001#Q03：volatile 的作用是什么？内存屏障的工作原理？volatile int a 执行 a++ 线程安全吗？|MJ049 · 拼多多 · 二面 · Q03]]）

   不一定。a++ 是“读—加—写”三步：volatile 保证每次读到的都是最新值，但两线程可以同读到 0、各自写回 1，更新丢失。修正用 `AtomicInteger.incrementAndGet()`（CAS）或加锁——口诀：只要出现“先检查再更新”或复合更新，volatile 不够（CAS 方案见 JUC-018）。

**面经来源**

- [[面经/百度/一面/0006#Q20：volatile 有什么作用？|MJ010 · 百度 · 一面 · Q20]]
- [[面经/拼多多/二面/0001#Q03：volatile 的作用是什么？内存屏障的工作原理？volatile int a 执行 a++ 线程安全吗？|MJ049 · 拼多多 · 二面 · Q03]]
- [[面经/收钱吧/一面/0001#Q18：volatile 有什么作用？原理是什么？|MJ068 · 收钱吧 · 一面 · Q18]]

**参考资料**（2026-09-13 查证；版本见正文）

- [JLS Java SE 21：线程与锁](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)

### JUC-011：如何交替打印 A1B2C3 并避免线程退出后永久等待？

**常见问法**

- 多线程交替打印 A1B2C3，任一线程退出后其余线程不永久阻塞。

#### 面试回答

先说清楚退出语义：本解采用“任一工作线程退出，就取消整个打印任务”，其余线程及时结束，不要求继续把序列补齐。

协议四句：

- 用同一把锁保护轮次和取消状态，等待写在条件循环里。
- 正常打印完，改轮次并通知。
- 异常、中断或者提前返回时，在 `finally` 里发布取消。
- 同时唤醒所有等待者，避免有人还在等。

#### 技术细节

**题目的设定**

- 假设两个线程分别输出 A—Z 和 1—26，正常序列就是 A1B2…Z26。

**等待条件怎么写**

- 必须同时检查轮次和取消两件事。
- 被唤醒之后，还要再检查一次取消。

**取消要覆盖所有路径**

- 线程包装器的 `finally` 要覆盖全部退出路径。
- 连启动失败这种情况，也要由协调者去发布取消。

**这个协议保证到哪一步**

- 保证的是：在协作退出、并且线程能被调度的前提下，不会出现永久的条件等待。
- 不保证的：进程被杀、线程被不安全强停、或者持锁永久阻塞，这几种都在保证之外。

**如果要换语义**

- 若改成“其余线程继续输出”，那就得另外定义存活集合和跳过轮次的规则。

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

**这份代码的前提**

- 例子用标准输出演示，前提是输出调用最终会返回。生产中不要在这把锁里面去调可能永久挂起的外部 I/O。
- `finally` 里的取消涵盖三种结束：正常结束、异常、中断。
- 如果某个线程压根没启动成功，就要由协调者去调 `cancel()`。


#### 深挖追问

1. **任意线程退出后如何保证其余线程不永久等待？**（面经实际出现；[[面经/百度/三面/0001#Q01：多线程交替打印 A1B2C3，任一线程退出后其余线程不永久阻塞。|MJ010 · 百度 · 三面 · Q01]]）

   做法是所有可控的退出路径都发布取消、并通知全部等待者，等待那边则用循环去检查取消标志。注意本解的约定是整体取消，不是继续补齐。

2. **只在正常输出后 notify 为什么不够？**（补充练习）

   因为对方可能在异常或中断时退出，那就永远不会再发通知了。所以必须有统一的退出清理，以及一个取消条件。

**面经来源**

- [[面经/百度/三面/0001#Q01：多线程交替打印 A1B2C3，任一线程退出后其余线程不永久阻塞。|MJ010 · 百度 · 三面 · Q01]]

**参考资料**（2026-09-13 查证；版本见正文）

- [JLS Java SE 21：等待与通知](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)

### JUC-012：AQS 的数据结构是什么，锁竞争与入队出队的并发安全怎么保证？

**常见问法**

- AQS 的数据结构是什么，锁竞争怎么处理，入队出队的并发安全怎么保证？

- AQS 是怎么实现的，是怎么保证 state 变量的原子性的？

#### 面试回答

AQS（AbstractQueuedSynchronizer）的核心就两样东西：

- 一个 `state`：volatile int。锁、信号量、线程池工作数这些语义，都由子类自己解释。
- 一条 CLH 变体的 FIFO（先进先出）双向等待队列。

获取的流程是一条链：

- 子类的 `tryAcquire` 用 CAS 改 state 来抢占。
- 抢不到，就把当前线程包装成 Node（里面存线程和等待状态 waitStatus），CAS 接到队尾。
- 之后要满足“前驱是 head，并且再次 `tryAcquire` 成功”才算出闸，否则按条件 park 挂起。

竞争策略是混合式的：入队前先自旋尝试一次，入队后就挂起等待。这样既避免纯自旋烧 CPU，也避免每次获取都得去唤醒别人。

释放由 `tryRelease` 决定，成功了再用 `unparkSuccessor` 唤醒队首的后继。

至于 ReentrantLock 的公平／非公平，只差在获取时是否先检查队列（`hasQueuedPredecessors`），见 [[专题题库/Java并发#JUC-005：ReentrantLock 如何实现公平锁与非公平锁？|JUC-005：ReentrantLock 如何实现公平锁与非公平锁？]]。

#### 技术细节

**入队（addWaiter）的并发安全靠两次 CAS 拼出来**

- 队列还没初始化时，先 CAS 设置 head；随后 CAS 把新节点接到尾部。
- 接尾的顺序是固定的三步：先 `pred.next=node`，再 CAS 换 `tail`，最后补 `node.prev`。
- CAS 失败说明有别的线程在并发入队，那就重读 tail 再重试。
- 中间这个“链接不完整”的状态可能被别的线程观察到，AQS 是靠遍历方向和 waitStatus 检查来容忍并修正它的。

**出队靠 setHead 加 unparkSuccessor**

- `setHead`：把成功获取的节点设成新的 head，同时断开前驱引用，顺手帮助 GC。
- 唤醒后继时是**从尾部向前**找有效的等待节点。
- 原因：并发入队可能让 `head.next` 暂时为 null 或者断链；而这时 prev 链已经完整了，所以从尾往前更可靠。

**Node 的 waitStatus 分别是什么意思**

- SIGNAL：释放的时候要唤醒自己的后继。
- CANCELLED：超时或中断之后，这个节点作废了。
- CONDITION、PROPAGATE：分别服务于条件队列和共享模式。

**独占模式和共享模式差在哪**

- 差别就在获取成功之后，要不要继续向后传播唤醒。`ReadWriteLock`、`Semaphore` 走的是共享这一边。

**被唤醒不等于拿到锁**

- `park`／`unpark` 是配合 `LockSupport` 用的。
- 线程被唤醒后，是从 `park` 处返回、再重新参与 `tryAcquire` 竞争，而不是直接获得锁。
- 这也正是非公平锁里 barging（插队）现象的来源。

#### 深挖追问

1. **为什么唤醒后继要从尾向前遍历而不是 head.next？**（补充练习）

   因为并发入队的最后一步（设置 prev／next 链接）可能还没完成，这时 `head.next` 会短暂为 null，或者指向一个已作废的节点。而 prev 方向在 tail 的 CAS 成功之后就已经可用了，从尾向前能找到最后一个有效等待者。

2. **AQS 的 state 一定是锁吗？**（补充练习）

   不是。state 只是一个由 CAS 保护的 int，语义完全由子类定义：

   - `ReentrantLock`：重入次数。
   - `Semaphore`：许可数。
   - `ThreadPoolExecutor`：合并存放运行状态与工作线程数。
   - `CountDownLatch`：计数。

3. **AQS 怎么保证 state 变量的原子性？**（面经实际出现；[[面经/美团/一面/0003#Q08：ReentrantLock 底层原理；AQS 怎么实现、如何保证 state 的原子性？|MJ037 · 美团 · 一面 · Q08]]）

   就两件套：

   - state 声明为 volatile：读永远拿最新值，并且禁止重排。
   - 写全部经 `compareAndSetState`：走 `Unsafe`／`VarHandle`，最后落到 CPU 的原子指令上（x86 的 `lock cmpxchg`，ARM 的 LL／SC），“比较＋交换”整体不可分割。

   关键点要说清：volatile 只保证单次读／写是原子的，而抢锁是一个“读—判断—写”的复合动作，必须靠 CAS 一次完成。失败之后要么重读重试，要么由子类决定入队挂起。state 本身从来不需要用锁去保护。

**面经来源**

- [[面经/帆软/一面/0002#Q09：AQS 的数据结构是什么，锁竞争怎么处理，入队出队的并发安全怎么保证？|MJ027 · 帆软 · 一面 · Q09]]
- [[面经/京东零售/一面/0001#Q08：ReentrantLock 内部靠什么核心结构来保护共享资源？|MJ034 · 京东零售 · 一面 · Q08]]
- [[面经/美团/一面/0003#Q08：ReentrantLock 底层原理；AQS 怎么实现、如何保证 state 的原子性？|MJ037 · 美团 · 一面 · Q08]]
- [[面经/虾皮/一面/0002#Q11：谈谈 AQS。|MJ048 · 虾皮 · 一面 · Q11]]

**参考资料**（查证：2026-09-22）

- [JDK 17 AbstractQueuedSynchronizer API 文档](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/locks/AbstractQueuedSynchronizer.html)
- [Doug Lea：The java.util.concurrent Synchronizer Framework](https://gee.cs.oswego.edu/dl/papers/aqs.pdf)（实现思路论文；细节以所用 JDK 源码为准）

### JUC-013：继承 Thread 和实现 Runnable 有什么区别，创建线程有哪些方式？

**常见问法**

- 多线程——继承 Thread 和实现 Runnable 接口有什么区别？

#### 面试回答

本质区别只有一句：任务与执行载体是否分离。

- 继承 `Thread`：逻辑写在 `run()` 里，Thread 对象自己就是执行载体。问题有三点——受 Java 单继承限制，继承它就用掉了唯一的继承位；任务和线程绑死，每个任务都得 `new` 一个 Thread，多线程要共享数据只能靠外部构造；而且没法交给线程池统一调度。
- 实现 `Runnable`：它只是“要跑的任务”。可以交给任意 Thread 执行，也可以提交线程池；同一个实例能复用，便于共享状态，还天然适配 Lambda。
- 实现关系上：`Thread` 本身就 implements `Runnable`，`new Thread(runnable)` 启动后，线程执行的是传进去那个任务的 `run()`。

还有第三种：`Callable` ＋ `FutureTask`。`call()` 有返回值、可以抛检查异常，配合 `ExecutorService.submit` 拿结果。

生产上的口径：优先“任务对象＋线程池”，不要散落着 `new Thread`。

#### 技术细节

**常被追问：start() 和 run() 的区别**

- `run()` 只是一次普通方法调用，仍然在当前线程里执行。
- `start()` 才会向 JVM 申请新线程，并触发线程状态从 NEW→RUNNABLE 的调度。

**“创建线程有几种方式”要说得严谨**

- 创建线程本质上只有一条路径：构造 `Thread`（或它的子类），然后 `start`。
- 所谓“四种方式”（继承 Thread、实现 Runnable、Callable＋FutureTask、线程池提交），差别其实在“怎么定义任务”，不在怎么造线程。线程池底层仍然是靠 `ThreadFactory` 造 Thread。

**两个补充口径**

- 注意 `Executors` 快捷工厂的隐患（无界队列／无限线程数），生产上要用 `ThreadPoolExecutor` 显式给参数（见 [[#JUC-002：线程池有哪些参数，任务流程及核心线程回收如何工作？|JUC-002：线程池参数与流程]]）。
- 虚拟线程（Loom，JDK 21 起正式可用）属于另一代模型：用 `Thread.ofVirtual()`／结构化并发。它不改变上面这套接口关系。

#### 深挖追问

1. **Runnable 怎么拿到执行结果？**（补充练习）

   `Runnable` 的 `run()` 没有返回值，所以有两条路：要么改用 `Callable` ＋ `Future` 去取结果；要么在任务内部通过回调／共享的 `CompletableFuture` 来通知完成。另外异常要显式处理，否则它只会进到线程的未捕获异常处理器里。

2. **创建一个新线程的主要成本（资源消耗）有哪些？**（面经实际出现）

   三条成本：

   - 内存：每条线程都有一个独立的**栈**（默认约 1MB，`-Xss` 可调，主要是虚拟地址预留、按需触页），再加上 JVM／OS 侧的 Thread 控制块与内核调度实体。
   - 系统调用：`start()` 要经 JNI 向 OS 创建线程，中间涉及用户态／内核态切换。
   - 运行期：线程被调度时会发生**上下文切换**——保存／恢复寄存器、PC、栈指针，还会冲击 CPU 缓存与 TLB，见 [[专题题库/操作系统#OS-009：线程切换需要保存哪些 CPU 上下文？|OS-009：线程切换的开销]]。

   再补一层：线程数过多还会放大内存占用和 GC Roots 的扫描成本，这正是“用线程池复用、限制并发线程数”的根因。如果任务是大量阻塞型的，可以用 JDK 21 的虚拟线程来降低对载体线程的占用。
   来源：[[面经/京东零售/一面/0001#Q09：怎么创建一个新线程，它的主要成本（资源消耗）有哪些？|MJ034 · 京东零售 · 一面 · Q09]]

**面经来源**

- [[面经/传音控股/二面/0001#Q02：多线程——继承 Thread 和实现 Runnable 接口有什么区别？|MJ030 · 传音控股 · 二面 · Q02]]
- [[面经/京东零售/一面/0001#Q09：怎么创建一个新线程，它的主要成本（资源消耗）有哪些？|MJ034 · 京东零售 · 一面 · Q09]]
- [[面经/钉钉/电话面/0001#Q10：谈谈对 Java 多线程／高并发的认识，以及如何创建、使用、什么场景用？|MJ053 · 钉钉 · 电话面 · Q10]]

**参考资料**（查证：2026-09-22）

- [Java 17 Thread API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Thread.html)
- [Java 17 Runnable / Callable API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Runnable.html)

### JUC-014：Java 有哪些实现线程安全的手段？

**常见问法**

- Java 里面是怎么实现线程安全的？
- 怎么保证一个类在多线程下正确？

#### 面试回答

按“从不上锁到上重锁”的成本梯度分层答，五层：

- 不共享可变状态：栈封闭（对象只在方法内创建）、`ThreadLocal`、不可变对象（final 字段＋正确发布、`record`、只含常量的类）。这是最优解。
- 无锁的可变共享：`volatile` 保证可见性与禁止重排；`Atomic` 原子类和 `LongAdder` 靠 CAS 做原子更新。注意它们只保证单个操作原子，`i++` 这种“读—改—写”复合动作靠 volatile 并不安全。
- 互斥：`synchronized`（代码块／方法，静态方法锁 Class 对象）与 `ReentrantLock`（可公平、可中断、多 `Condition`），再配 `wait`／`notify` 或 `Condition` 来表达条件。
- 现成的并发组件：`ConcurrentHashMap`、`CopyOnWriteArrayList`、`BlockingQueue`，以及 `computeIfAbsent`／`merge`／`putIfAbsent` 这类原子复合操作——避免自己去拼 check-then-act。
- 结构层面：用有界线程池，或者“单线程写者＋队列”，把共享收敛成消息传递。

收口要说方法论，三件事：

- 先明确这个类的**不变量**是什么、由哪把锁或哪个协议来保护；读写两端都走同一套协议才叫安全。
- 覆盖**发布安全**：对象构造完成前不外泄 `this`、用好 final 语义。
- 覆盖**终止安全**：中断、shutdown 和资源释放。否则只能算“看起来没报错”。

#### 技术细节

三个必须能展开的点。

**原子性、可见性、有序性分别由什么保证**

- 原子性：靠锁与 CAS。
- 可见性：靠同步——`synchronized`／`Lock` 的“解锁—加锁”会形成 happens-before；`volatile` 或者 final 的正确初始化也可以。
- 有序性：靠 JMM 的重排限制。
- 一句澄清：单靠“给字段加 volatile”只解决可见性，解决不了多个写者之间的竞争。

**锁到底该锁什么**

- 锁的对象，必须是“保护同一份不变量”的所有线程共同看到的那个对象。
- 常见错误有三种：锁了每次新建的实例（`synchronized(new Object())`、锁 String 参数）、锁 `Integer` 装箱对象（受缓存影响，语义还不确定）、读写分离却用了两把不同的锁。
- 读写锁（`ReentrantReadWriteLock`／`StampedLock`）适合读多写少；但 `StampedLock` 的乐观读要校验、还要处理重读，代码复杂度更高。

**并发容器的边界**

- `ConcurrentHashMap`：保证单个方法原子，但不保证“先 `containsKey` 再 `put`”这种调用序列原子。
- 迭代器：是弱一致的——不抛 `ConcurrentModificationException`，但可能读到旧的快照。
- `CopyOnWriteArrayList`：读很便宜、写很昂贵，只适合读极多、写极少的场景。

**安全发布有哪几种方式**

- 初始化 static 字段（holder 单例模式就属于这一类）。
- volatile 字段。
- final 字段（前提是构造函数内 `this` 没有逸出）。
- 经 `Lock` 保护的字段。
- 传进 `BlockingQueue`。

**两个工程上的坑**

- 线程池会复用线程，所以 `ThreadLocal` 的值会跨任务残留，务必要 `remove()`（见 [[#JUC-001：ThreadLocal 是什么，有什么问题？|JUC-001：ThreadLocal 是什么，有什么问题？]]）。
- 虚拟线程场景下（JDK 21 起），长临界区的 `synchronized` 曾经会钉住载体线程；JDK 24 的 JEP 491 已经改进了这一点。但阻塞式锁仍然建议换成 `ReentrantLock` 或无锁结构，并且按所用的 JDK 版本来说明。

#### 深挖追问

1. **volatile 能替代锁吗？**（补充练习）

   不能。它解决的是可见性与重排，不解决复合动作的原子性。

   - 够用的情况：只有一个变量，而且写入不依赖旧值——比如标志位、发布一个已经构造好的对象引用。
   - 不够用的情况：计数、边界检查、多字段不变量，这些仍然要靠锁，或者用原子类的 CAS 循环。

2. **不可变对象一定线程安全吗？**（补充练习）

   不一定。引用不可变不等于状态不可变：`final List<String>` 这个字段本身改不了指向，但列表里的内容能改，这就是“浅不可变”。

   真正的不可变要同时满足四条：所有字段 final、类型不提供修改方法、构造器不逸出 `this`、引用的对象本身也不可变（或者做防御性拷贝／用 `List.copyOf`）。

3. **怎么判断一段代码需要加锁？**（补充练习）

   看三个条件是否同时成立：两个及以上线程访问同一状态、其中至少一个是写、而且这些访问不在同一套同步协议里。三条缺一条就不需要锁。

   反过来，即便“目前只有一个线程在跑”，也应当把不变量和保护策略写清楚，否则以后把线程数加大时，这里就是隐藏的 bug。

4. **synchronized 和 ReentrantLock 怎么选？**（面经实际出现；参见 [[#JUC-003：synchronized 和 Lock 有什么区别？|JUC-003：synchronized 和 Lock 有什么区别？]]）

   默认用 `synchronized`：语法简单、不会忘记 unlock，JVM 层还在持续优化它。

   只有需要这些能力时才换 `ReentrantLock`：公平策略、可中断获取、超时获取、多条件队列，或者非块结构化的锁（要跨方法加解锁）。

**面经来源**

- [[面经/美团/一面/0001#Q21：Java 里怎么实现线程安全？|MJ031 · 美团 · 一面 · Q21]]
- [[面经/美团/一面/0003#Q06：多线程怎么保证线程安全？|MJ037 · 美团 · 一面 · Q06]]
- [[面经/钉钉/电话面/0001#Q10：谈谈对 Java 多线程／高并发的认识，以及如何创建、使用、什么场景用？|MJ053 · 钉钉 · 电话面 · Q10]]

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
- **`CountDownLatch`**——A 完成时 `countDown`，B 在开头 `await` 放行，适合“阶段／里程碑”式依赖。
- **`CompletableFuture`**——`supplyA().thenRunAsync(B).thenRunAsync(C)` 用链式回调表达依赖，还能组合并行分支。
- **`CyclicBarrier`／`Semaphore`／锁＋`Condition` 标志位**——更复杂的多阶段协作时用。

要点：`start()` 只保证“被调度”，不保证执行先后顺序，所以“先 start 的就先跑完”是错的直觉，必须显式建立“等待前驱完成”的关系。

#### 技术细节

**join 这一层**

- `join` 底层就是 `wait/notify`：线程终止时，JVM 会唤醒所有 `join` 它的线程。
- 用 `join(timeout)` 可以加超时，避免前驱卡死时把自己永久挂住。

**其他手段的注意点**

- 单线程池的本质：任务排进一条队列，由一个 worker 顺序取。注意它用的是无界队列，任务堆积会占内存。
- `CountDownLatch` 的计数只能一次性归零。要重复放行多个阶段，得换 `CyclicBarrier` 或 `Phaser`。

**先把题意分清楚**

- 如果只是“按顺序提交”但允许并行执行，那考的就不是顺序，而是编排，这两件事要区分清楚。

**候选人最初的答法为什么不行**

- 他答的是“把 A／B／C 放进公平等待队列一个个取”。但公平队列解决的是“同一时刻谁先拿到执行权”，表达不了“A 完全结束之后 B 才能开始”这种完成依赖，面试官不认可。
- 回到 `join`／串行线程池／latch 这类“等待前驱完成”的机制，才是本题的要点。

#### 深挖追问

1. **`join` 和让线程 `sleep` 一段固定时间再启动下一个有什么区别？**（补充练习）

   本质区别在“按时间错开”还是“按事件等待”。`sleep` 猜的是一个时长，前驱超时了或者提前干完，全都对不上，很脆弱；`join` 等的是“前驱真正终止”这个事件，跟耗时无关，语义才是对的。

2. **`join` 为什么能被中断，中断后怎样？**（补充练习）

   `join` 会抛 `InterruptedException`，表示这次等待被中断了。处理时别把它吞掉：要么恢复中断标志，要么按业务终止。否则“等前驱”这个关系就被破坏了。

**面经来源**

- [[面经/字节/一面/0008#Q06：有 A、B、C 三个线程，如何保证它们的执行顺序？|MJ033 · 字节 · 一面 · Q06]]

**参考资料**（本次查证：2026-09-23）

- [Java 17 Thread.join API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/Thread.html#join())
- [Java 17 CountDownLatch / Executors API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/package-summary.html)

### JUC-016：Java 中有哪些锁，如何按不同维度分类？

**常见问法**

- Java 中有哪些锁机制？
- Java 里锁的作用是什么？什么样的业务需要用锁来做？
- 除了 synchronized 还了解哪些锁？公平锁、非公平锁、轻量级锁、自旋锁分别讲一讲，synchronized 属于哪种？
- 自旋去获取锁会消耗 CPU 吗？适合什么样的场景？

#### 面试回答

不要背名字，按“分类维度”答更显结构：

- **乐观 vs 悲观**：悲观锁先加锁再操作（`synchronized`、`ReentrantLock`、DB 行锁）；乐观锁假设冲突少、用 CAS／版本号失败再重试（`Atomic*`、`Integer` 版本、`StampedLock` 乐观读）。
- **阻塞 vs 自旋**：拿不到就挂起（重量级锁、`Lock`）vs 忙等重试（自旋，适合临界区极短）。
- **可重入 vs 不可重入**：`synchronized`、`ReentrantLock` 可重入（同线程再次进入不死锁，靠持有计数）。
- **公平 vs 非公平**：`ReentrantLock(boolean fair)`，公平按队列顺序、非公平允许插队（吞吐更高、可能饥饿）。
- **独占 vs 共享**：写锁独占、读锁共享（`ReentrantReadWriteLock`、`StampedLock` 读写／乐观读）。
- **偏向／轻量／重量**：`synchronized` 在对象头 Mark Word 上的锁状态演进（偏向锁 JDK 15 起废弃）。
- 还有协作工具（`Semaphore`、`CountDownLatch`、`CyclicBarrier`）和分布式锁（Redis、ZooKeeper）——严格说部分是并发工具而非“互斥锁”，答时区分。

落点：锁保护的是“同一份共享可变状态的复合操作”，能不加锁就不加锁（用不可变、线程封闭、并发容器替代，见 JUC-014）。

#### 技术细节

**读写锁：ReadWriteLock**

- 规则是“读读并行、读写／写写互斥”。
- 一条方向性限制：写锁可以降级成读锁，但读锁不能升级成写锁——那样会死锁。

**乐观读：StampedLock**

- 它提供乐观读：读的时候不加锁，读完再用 `validate` 校验这期间有没有被写过。
- 代价是更快但不可重入，而且需要处理线程中断。

**自旋锁**

- 一定要设自旋上限和退避，否则就是空转烧 CPU。

**最容易答错的一点：这些维度是横切的**

- 同一把锁可以同时满足“可重入＋非公平＋独占”，默认构造的 `ReentrantLock` 就是这样。
- 所以别把它们当成互斥的选项来背。

#### 深挖追问

1. **`synchronized` 和 `ReentrantLock` 怎么选？**（面经实际出现）

   见 [[#JUC-003：synchronized 和 Lock 有什么区别？|JUC-003：synchronized 和 Lock 有什么区别]]。简单说：前者语法简单、自动释放，够用就用；后者要等到你确实需要可中断获取、超时 `tryLock`、公平性、多 `Condition` 或者读写分离时，才值得那点复杂度。

2. **偏向锁为什么被废弃？**（补充练习）

   它优化的是“始终只有一个线程重入”这种场景。问题有三个：撤销需要走到 safepoint、在现代并发下收益不显著、还额外带来复杂度和性能毛刺。所以 HotSpot 从 JDK 15 起把它默认禁用并标记废弃。

   这一条与 [[#JUC-003：synchronized 和 Lock 有什么区别？|JUC-003]] 的锁状态口径一致，具体参数以所用 JDK 版本的文档为准。

3. **synchronized 在上述分类里各属于哪种？**（面经实际出现；[[面经/拼多多/一面/0002#Q06：Java 里除了 synchronized 还了解哪些锁？公平锁、非公平锁、轻量级锁、自旋锁各是什么，synchronized 属于哪种？|MJ044 · 拼多多 · 一面 · Q06]]）

   一把锁同时落在多个维度，逐维贴标签即可：

   - 悲观锁（先锁再操作）、可重入、独占。
   - 公平性：没有公平选项，获取近似非公平。
   - 等待方式：混合——轻量级阶段可自旋，膨胀成重量级后挂起阻塞。
   - 锁状态：HotSpot 实现层经历“无锁 → 轻量级 → 重量级”升级（偏向锁 JDK 15 起废弃）。
   - 口径提示：轻量级／自旋是 JVM 实现细节，答完声明“这是 HotSpot 的说法”。

4. **轻量级锁自旋获取会消耗 CPU 吗？什么样的场景自旋才有效？**（面经实际出现；[[面经/拼多多/一面/0002#Q07：轻量级锁自旋获取会消耗 CPU 吗？有什么问题？自旋适合什么场景、什么时候无效？|MJ044 · 拼多多 · 一面 · Q07]]）

   会：自旋就是忙等循环，等待期间核不闲下来。它是拿“烧 CPU”换“免挂起”——省掉上下文切换和唤醒延迟，前提是等得短。

   - 有效：临界区短且确定（几条指令的字段更新）、线程数不明显超过核数。
   - 无效：锁内有 IO／远程调用等长操作；线程数远超核数（自旋者占着核，持锁者反而排不上队，越等越久）；高竞争下重试互相抵消。
   - HotSpot 因此做自适应自旋（按历史成功率调整圈数），竞争加剧直接膨胀成重量级去挂起；手写自旋锁必须设上限加退避。
   - 同一道理适用于 CAS：低竞争吞吐优势，高竞争空转，解法是分段降冲突而不是旋得更久。

**面经来源**

- [[面经/京东零售/一面/0001#Q07：Java 中有哪些锁机制？|MJ034 · 京东零售 · 一面 · Q07]]
- [[面经/帆软/一面/0003#Q05：Java 里锁的作用是什么，什么样的业务需要用锁？|MJ035 · 帆软 · 一面 · Q05]]
- [[面经/拼多多/一面/0002#Q05：项目里为什么用 synchronized 加锁？锁的机制保护什么？为什么选 synchronized 而不是其它锁？|MJ044 · 拼多多 · 一面 · Q05]]
- [[面经/拼多多/一面/0002#Q06：Java 里除了 synchronized 还了解哪些锁？公平锁、非公平锁、轻量级锁、自旋锁各是什么，synchronized 属于哪种？|MJ044 · 拼多多 · 一面 · Q06]]
- [[面经/拼多多/一面/0002#Q07：轻量级锁自旋获取会消耗 CPU 吗？有什么问题？自旋适合什么场景、什么时候无效？|MJ044 · 拼多多 · 一面 · Q07]]
- [[面经/帆软/一面/0004#Q03：乐观锁和悲观锁分别有什么特点？各自在什么场景下使用？|MJ050 · 帆软 · 一面 · Q03]]
- [[面经/帆软/一面/0004#Q16：多线程编程中锁有什么作用？|MJ050 · 帆软 · 一面 · Q16]]

**参考资料**（本次查证：2026-09-23）

- [Java 17 concurrency（locks）package API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/locks/package-summary.html)

### JUC-017：Go 的协程在 Java 有类似方案吗？

**常见问法**

- 就包括你刚说 Go 的协程，Java 有类似的方案吗？
- 用 Go 写的并发服务改用 Java，怎么达到类似效果？

#### 面试回答

最接近的方案是 JDK 21 正式可用的虚拟线程（Project Loom，JEP 444）。

- 它是什么：由 JVM 而不是操作系统调度的轻量线程。创建用 `Thread.ofVirtual()` 或 `Executors.newVirtualThreadPerTaskExecutor()`，每个任务一条、用完即弃。
- 有多轻：内存开销以 KB 计，可以开出百万级。
- 为什么和 goroutine 同构：线程阻塞在 IO 时，不会挂住 OS 线程，而是把虚拟线程从载体线程（也就是跑在 ForkJoinPool 上的平台线程）卸载下来；IO 完成后再挂到某个载体线程继续跑。
- 所以效果是：“同步写法、异步性能”成立，不用再写回调地狱。

如果不到 JDK 21，替代方案有：响应式（WebFlux／Reactor）或者 CompletableFuture 编排——能力等价，但编程模型是倒置的；语言级协程则有 Kotlin coroutines 和 Quasar 这类字节码方案。

三个差异也要主动讲清：

- Java 没有 channel 那样内建的通信原语，协作要靠 `BlockingQueue` 这些。
- 虚拟线程不该再包一层线程池，要限流就用 `Semaphore`。
- 大量 `ThreadLocal` 在百万线程下会把内存放大，官方建议改用 Scoped Values（仍在孵化）。

#### 技术细节

**版本口径**

- 演进路线：预览 JEP 425（JDK 19）→ JEP 436（20）→ 正式 JEP 444（21，LTS）。
- 一个坑：`synchronized` 的长临界区和 native 帧会“钉住”载体线程。JEP 491 在 JDK 24 移除了 synchronized 这个限制，所以 21～23 上高并发锁竞争的场景仍然建议改用 `ReentrantLock` 或者缩小临界区。答题时要把所用的 JDK 版本报清楚。

**还没定型的部分**

- 结构化并发 `StructuredTaskScope` 和 Scoped Values，截至 JDK 25 仍在孵化迭代。生产上的主路径就是虚拟线程本体。

**和 Go 的 GMP 怎么对照**

- 对应关系（见 [[专题题库/Go#GO-001：是否会 Go，goroutine 的底层实现是什么？|GO-001]]）：虚拟线程≈G，载体线程≈M，ForkJoinPool 的 work-stealing≈P 队列＋窃取。
- 关键差别：Go 是靠编译器插入调度点，Java 是靠 JDK 库在阻塞点主动 yield。
- 推论：没有走 JDK 阻塞 API 的“假 IO”（比如自旋、第三方 socket 实现）不会自动让出。

**池的语义变了**

- 平台线程池的价值在于“复用昂贵的线程”；虚拟线程便宜到根本不需要复用。
- 要保护下游时，你限制的是并发任务数（用 `Semaphore`／固定许可），而不是线程数。

#### 深挖追问

1. **虚拟线程能完全替代 goroutine 吗？**（补充练习）

   并发模型上很接近，但工程生态不等价。

   - Go 那一侧：语言级的 channel、select，统一的运行时，还有更小的部署产物。
   - Java 这一侧：虚拟线程只解决“阻塞很廉价”这一件事，通信仍然要靠 JUC 组件。老代码里的锁、`ThreadLocal`、以及对连接池尺寸的假设，都得重新审视。

2. **为什么不能给虚拟线程再套一个固定大小线程池？**（补充练习）

   因为虚拟线程的开销主要在堆内的栈对象上，创建和销毁都很便宜。再套一层固定池，等于把并发度重新钉回池大小，白白浪费了“每任务一线程”这个模型。真正需要做的是给下游资源（DB 连接、配额）显式设闸，用 `Semaphore` 来表达，而不是去复用线程。

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

CAS（Compare-And-Swap，比较并交换）就是一条原子指令：

- 给三个东西——内存位置、期望的旧值、新值。
- 只有当前值等于期望值时才写入新值并返回成功，否则失败。
- 失败之后由调用方重读再重试，这就构成了 lock-free（无锁）的“乐观重试循环”。

它和锁的关系一句话讲清：CAS 不是锁，它是无锁并发的硬件原语；但它同时是很多锁的实现零件。`Atomic` 整家族、AQS 抢 state、`ReentrantLock` 入队、`synchronized` 轻量级锁替换对象头 Mark Word、`ConcurrentHashMap` 的空表初始化和桶写入，底层全是 CAS。

和锁的对比口径：

- 锁是悲观的：拿不到就让出 CPU 挂起。
- CAS 是乐观的：失败就自旋重试。所以低竞争时吞吐高，高竞争时就是空转烧 CPU。

三个局限要主动报全：

- 只保证单个字原子，多字段的不变量仍然要靠锁。
- ABA 问题要用版本号来解（`AtomicStampedReference`）。
- 长时间拿不到进度时，可能需要帮助器介入——这也是无锁队列复杂的地方。

#### 技术细节

**硬件层靠哪条指令**

- x86：`lock cmpxchg`（锁住缓存行一致性）。
- ARM／RISC-V：用 LL／SC 一对指令，也就是 load-linked／store-conditional。

**JVM 侧的入口**

- 早期是 `Unsafe.compareAndSwap*`。
- JDK 9+ 推荐改用 `VarHandle`（`compareAndSet`、`getAndSet` 等，还能选内存序）。

**Java 用法示例**（自实现原子计数器，体会一下 retry 的形态）：

```java
final AtomicInteger counter = new AtomicInteger();
void inc() {
    int old;
    while (!counter.compareAndSet(old = counter.get(), old + 1)) {
        // 失败即有并发修改，重读再试；Thread.onSpinWait() 可提示 CPU 自旋
    }
}
```

**ABA 问题**

- 现象：值从 A 改成 B、又改回 A，CAS 就误判成“压根没变过”。
- 什么时候无所谓：纯计数。什么时候致命：链表节点复用——head 被弹出又被压回相同的值。
- 两种解法：加版本戳（`AtomicStampedReference`），或者让每个节点都是独立对象（Java 并发容器走的就是这条路）。

**CAS 自带同步语义**

- 一次成功的 CAS，相当于做了一次 volatile 读加一次 volatile 写，happens-before 是成立的。
- 这也解释了为什么 `Atomic` 类不加锁还能保证可见性。

**竞争很高时换什么**

- 换 `LongAdder`：它把单点 CAS 分散成一个 `Cell` 数组，按线程哈希决定落点，用空间换竞争。
- 这是“CAS 也有 contention（竞争）预算”的一个典型工程例证。

#### 深挖追问

1. **CAS 属于乐观锁还是悲观锁？**（补充练习；口径见 [[#JUC-016：Java 中有哪些锁，如何按不同维度分类？|JUC-016：锁的分类]]）

   通常的说法是“乐观锁的一种实现”：先假设冲突很少，失败了再重试；悲观锁则是先把资源占住再操作。但严格讲，CAS 本身只是一条原子指令，“乐观”是使用它的那套算法策略。

2. **自旋锁和 CAS 是一回事吗？**（补充练习）

   不是，这两个维度是正交的。

   - 自旋说的是“获取失败之后怎么等”（忙等）；CAS 说的是“怎么尝试更新”。
   - 举例：`synchronized` 的轻量级锁用 CAS 去抢 Mark Word，但抢不到之后要不要自旋、自旋多久，由实现决定（自适应自旋）。
   - 反过来，Mutex 这种挂起等待的锁，也可以先 CAS 一次再睡。

3. **为什么有了 CAS 还需要 AQS 的队列？**（补充练习）

   因为纯 CAS 重试在高竞争下吞吐会塌方，而且不公平。AQS 的做法是分工：用 CAS 处理快速路径，竞争者则进 CLH 队列 park 挂起，把“抢不到就排队睡觉”制度化，这样既保住无锁快速路径，又有界等待。见 [[#JUC-012：AQS 的数据结构是什么，锁竞争与入队出队的并发安全怎么保证？|JUC-012：AQS]]。

**面经来源**

- [[面经/帆软/一面/0003#Q06：讲一下 CAS 操作，CAS 和锁有什么关系？|MJ035 · 帆软 · 一面 · Q06]]
- [[面经/拼多多/二面/0001#Q03：volatile 的作用是什么？内存屏障的工作原理？volatile int a 执行 a++ 线程安全吗？|MJ049 · 拼多多 · 二面 · Q03]]

**参考资料**（本次查证：2026-09-23）

- [Java 21 VarHandle API（compareAndSet 与内存序）](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/lang/invoke/VarHandle.html)
- [Java 21 java.util.concurrent.atomic 包](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/atomic/package-summary.html)

### JUC-019：线程安全的 LRU 缓存怎么实现，读多写少如何优化？

**常见问法**

- 上述 LRU 如果要改造成线程安全的，该怎么做？
- 在“读多写少”的并发场景下，怎么优化线程安全方案？
- 让你实现一个并发 LRU 缓存，怎么设计？

#### 面试回答

先说为什么不能偷懒：只把 `HashMap` 换成 `ConcurrentHashMap` 是不够的。

- 原因：LRU 每次访问都要同时改哈希表和链表，这是一组必须原子的复合操作。
- 分开保护的后果：会暴露“map 里有节点、但链表还没摘完”的中间态，链表直接断裂。
- 基线做法：用一把锁（`synchronized` 或 `ReentrantLock`）把 get／put／淘汰整体串行化。好处是正确性最好证明，代价是所有访问都来争同一把锁。

再说读多写少场景真正的陷阱：LRU 的“读”其实不只是读。

- `get` 命中时要先把节点移到队首，这本质上是写操作。
- 所以读写锁在这里收益有限：读锁保护不了改链表的动作，硬上 `ReadWriteLock` 很容易写成“多个线程同时在读锁下移动节点”的数据损坏。

要优化，按代价从低到高排三档：

- 放弃严格 LRU，换近似算法：读事件先记到每线程缓冲／环形队列，再批量回放到访问顺序结构。Caffeine 的思路就是读写分离＋批量处理读事件＋用 W-TinyLFU 代替严格 LRU，本质是把“每次读都改结构”变成“攒着改”。
- 分段（striping）：按 key 哈希切成若干把独立的小 LRU，每段一把锁，竞争按段数下降。代价是容量和淘汰变成“每段近似”，不再是全局精确。
- 顺序要求弱的时候：读路径做无锁快照——位置只周期性重排，或者由单个写线程独占重排、读线程只读不可变视图。

收口时必须把收益和代价一起说：换成近似算法，就换掉了 LRU 的精确语义；做了分段，淘汰就不再公平。验证方式也要给出——并发压测看吞吐和锁等待，再用双链表自检或不变量断言，确认没有断链。

#### 技术细节

**分层看成本**（与 [[#JUC-014：Java 有哪些实现线程安全的手段？|JUC-014：Java 有哪些实现线程安全的手段]] 的梯度一致）

- 整锁最简单：单条命令的微秒级临界区，在中等并发下就已经够用。
- 但一把全局锁会让多核吞吐在某个线程数之后不再上升。
- 锁里面不要做回源、序列化或任何 I/O，否则持锁时间会被外部依赖支配。
- 这时要改成“锁外取数据、锁内改结构”，或者用 `computeIfAbsent` 这类原子复合操作把边界收敛住。（提醒：`ConcurrentHashMap` 只保证单个方法原子，不保证“先查再放”这种调用序列原子。）

**读写锁为什么常常不适用**

- `ReadWriteLock` 的收益来自“读读并行”，前提是你的读路径真的只读（见 [[#JUC-016：Java 中有哪些锁，如何按不同维度分类？|JUC-016：锁的分类与读写锁边界]]）。
- 访问顺序维护这类结构，读的时候本身要写，读锁就形同虚设。
- `StampedLock` 的乐观读也只适合“读完再校验有没有被写坏”的场景（读出快照之后 `validate`）；一旦读要改结构就不成立了，而且它不可重入、还需要处理中断。
- 真要用读写锁，能用的形态是：读锁只取 value，顺序更新异步补做——但这已经属于近似 LRU 了。

**分段与容量**

- 段内独立容量会让总容量在段之间分布不均：热 key 集中在某一段时，那一段会更早淘汰。要么接受，要么用少量全局统计做二次均衡。
- 淘汰时只在本段里找尾节点，锁粒度是小了，但淘汰顺序不再是全局最优。
- 无锁读还有另一条路，适合“读多写少但可以容忍顺序滞后”：节点里放访问计数或时间戳（一次 CAS 更新，不动链表），再由后台线程周期性按计数重排链表，把结构改动从请求路径上移走。

**验证与度量要说三件事**

- 并发正确性：多写多读压测之后，遍历校验 map 和链表一致、无环无断链；也可以用控制线程交错的小规模随机测试去跑不变量。
- 收益量化：不同段数、是否批量下的吞吐与 P99，还有锁等待和上下文切换。
- 语义偏差：命中率相比严格 LRU 掉了多少。

只说“加个读写锁就快了”不算答案，面试官要的是你知道“读操作本身就是写”这个坑。

#### 深挖追问

1. **为什么不直接用 `ConcurrentLinkedQueue` 之类无锁容器拼一个 LRU？**（补充练习）

   两层原因。第一，无锁容器解决的是单个容器内部的并发，解决不了“两个容器要一起变更”的跨容器原子性。第二，LRU 需要按 key O(1) 定位并摘除任意节点，队列本身不支持这个操作；硬做就会退化成扫描，或者再加一层 map，一致性反而更难保证。

2. **分段锁和 `ConcurrentHashMap` 的分段有什么相同与不同？**（补充练习）

   相同点：都是按 key 哈希来收敛竞争粒度。

   不同点：JDK 8 之后的 `ConcurrentHashMap` 已经改用桶级 `synchronized` ＋ CAS，不再是固定的 Segment 分段（见 [[专题题库/Java集合#JAVA-COL-002：HashMap 和 ConcurrentHashMap 有什么区别？|JAVA-COL-002]]）；而自研的分片 LRU 通常会保留一个显式的段数组，段数和段内容量是自己定的设计参数。

3. **读事件批量回放会不会让淘汰变慢，怎么处理？**（面经实际出现；[[面经/阿里云/一面/0001#Q04：把 LRU 改造成线程安全该怎么做；读多写少的并发场景下怎么优化线程安全方案？|MJ039 · 阿里云 · 一面 · Q04]] 的延伸）

   会。读事件积压的那段时间里顺序是滞后的，可能把刚被访问过的 key 淘汰掉。

   缓解手段有三条：

   - 限制缓冲长度，并设阈值触发排空。
   - 对热点 key 单独保护：准入窗口＋频率门槛，这也是 TinyLFU 的准入思想。
   - 用命中率指标去验证偏差能不能接受。

**面经来源**

- [[面经/阿里云/一面/0001#Q04：把 LRU 改造成线程安全该怎么做；读多写少的并发场景下怎么优化线程安全方案？|MJ039 · 阿里云 · 一面 · Q04]]

### JUC-020：如何写一个限制最大并发数的并发任务处理器？

**常见问法**

- 手撕：给定 100 个任务 ID，控制最大并发数为 3，模拟并发调用外部接口（如打印 ID）。
- 怎么限制同时执行的线程数／请求数？
- 并发数和 QPS 限制是一回事吗？

#### 面试回答

题意是“同时在飞的任务数不超过 3”，本质就是信号量或者固定大小的工作池，两种写法都要会说。

- Java 版一（固定线程池）：`ExecutorService pool = Executors.newFixedThreadPool(3)`，把 100 个任务提交进去，再 `shutdown()` ＋ `awaitTermination(timeout)` 等全部完成。线程数就是并发数，代码最短。
- Java 版二（`Semaphore(3)`）：每个任务 `acquire()`，在 `try/finally` 里 `release()`，收尾配 `CountDownLatch` 或 `CompletableFuture.allOf`。Java 21 起还可以直接用虚拟线程跑 100 个任务、靠许可控制并发（这时线程数不再是约束，许可才是）。这版更贴近“并发数”的原意，也方便扩展成“每个任务自己带重试”。
- Go 版：`errgroup.Group` 加 `SetLimit(3)`，循环 `g.Go(...)` 之后 `g.Wait()`；或者手写一个容量为 3 的带缓冲 channel 当令牌池，启动前 `tokens <- struct{}{}`、完成后 `<-tokens`。

写完要主动补三点，这题的区分度全在这里：

- 失败与取消：外部接口报错要设重试上限和退避；要不要因为某个任务失败就取消其余任务，得说清并实现（Go 用 `context` 取消，Java 用 `Future.cancel`／中断）。另外，已经拿到的许可和已经启动的 goroutine／线程必须保证释放，否则并发数会“漏”，最后卡死。
- 结果与可观测：要不要保持原顺序（做法是按索引写回定长数组，而不是加锁追加）；还要统计成功数、失败数和耗时。
- 限并发不等于限速率：限制“同时在飞的数量”是最常用的下游自我保护；真要限 QPS，得再加令牌桶或漏桶（口径见 REDIS-004）。两者经常一起用。

#### 技术细节

**实现细节的四个常见错误**

- 用 `newFixedThreadPool`：它带的是无界队列，任务全堆在队列里，只做到“线程数受限”。如果任务本身占大量内存或者需要背压，应该改用有界队列 ＋ 拒绝策略的 `ThreadPoolExecutor`。
- `Semaphore` 忘了在 `finally` 里释放许可：可用并发数会被永久减少。
- 用 `CountDownLatch(100)` 计数时，要保证每个任务一定 `countDown()`，异常路径也不例外，否则主线程会一直等下去。
- 提交任务和等待完成之间，不要把线程池关闭两次，也不要漏掉 `shutdown`。

**验证方法要能主动说出口**

- 让模拟调用 `sleep` 一个固定时长，然后看同时处于执行中的数量峰值是不是恰好为 3：做法是用一个 `AtomicInteger` 记录当前活跃数并取最大值。
- 再看总耗时是否符合 `ceil(100/3) × 单次耗时` 这个量级。
- 最后补一个失败注入用例（比如让第 5、17 个任务抛异常），确认既不死锁也不吞结果。
- 如果被追问“100 万个任务怎么办”，方向是：不要把 100 万个任务一次性物化出来再提交。改成流式生产＋固定消费者（生产者-消费者），或者分批提交，让队列长度受控。

**它和线程池的关系，值得点一句**

- 固定线程池其实就是一个“以线程为许可”的信号量实现。
- 两者的差别在弹性：`Semaphore` 允许任务由调用方线程或者虚拟线程去执行，只约束临界并发度；线程池则连执行资源一起约束了。
- 具体到这题：调外部接口是 IO 密集，JDK 21 上“虚拟线程 ＋ Semaphore”通常比“3 个平台线程”更好——栈成本极低、阻塞时不占载体线程，而并发上限仍然由许可精确控制。Go 侧的对应写法就是 goroutine ＋ 带缓冲 channel／errgroup 限流，见 [[专题题库/Go#GO-002：Go 的 channel 是什么、并发安全吗，和 Mutex 怎么取舍？|GO-002]]。

#### 深挖追问

1. **为什么不直接开 100 个线程？**（补充练习）

   因为线程本身就有栈和调度成本。而且 100 个并发还可能把外部接口压垮、或者被对方限流，超过下游容量之后吞吐反而会下降。要记住限并发的意义是“按下游能承受的量去调用”，不是“尽量快”。

2. **如果要求“任务提交不能阻塞主线程且要有超时”怎么改？**（面经实际出现；[[面经/字节/一面/0009#Q18：手撕：实现一个并发任务处理器——100 个任务 ID，最大并发数为 3，模拟并发调用外部接口（如打印 ID）。|MJ041 · 字节 · 一面 · Q18]] 的延伸）

   三层各改一处：

   - 提交侧改成非阻塞：用 `tryAcquire(timeout)`，拿不到许可就走快速失败，或者进本地队列缓冲。
   - 单个任务配独立超时：`orTimeout`／`context.WithTimeout`，超时之后取消任务并释放许可。
   - 整体再给一个 deadline：结束时统计完成／超时／失败三类计数，而不是让主线程无限等待。

3. **多个外部接口共享同一个并发上限怎么设计？**（补充练习）

   按资源分桶：每个下游放一个独立的 `Semaphore`，这样一个慢接口不会把另一个的额度拖住；或者在上面做一层带优先级的小调度器。

   如果还要限总并发，就再套一层父许可，并且严格按“先取父、后取子；释放时反向”的顺序来，避免死锁。

**面经来源**

- [[面经/字节/一面/0009#Q18：手撕：实现一个并发任务处理器——100 个任务 ID，最大并发数为 3，模拟并发调用外部接口（如打印 ID）。|MJ041 · 字节 · 一面 · Q18]]

### JUC-021：CopyOnWriteArrayList 是怎么保证线程安全的？为什么没有 CopyOnWriteLinkedList？

**常见问法**

- 知道 CopyOnWriteArrayList 吗？它是怎么保证线程安全的？
- List 有 ArrayList 和 LinkedList，你认为有没有 CopyOnWriteLinkedList？为什么没有？

#### 面试回答

线程安全靠“写互斥＋整表替换的原子发布”，四句说完：

- 写操作先拿 ReentrantLock，把底层数组复制一份，在新数组上完成增删改。
- 改完一次性替换数组引用，这个替换对所有读线程是原子可见的。
- 读操作完全无锁、无等待，读到的一定是某个完整时刻的数组快照。
- 所以它是弱一致：不保证“写完立刻人人可见”，但每个时刻看到的都是自洽的旧表或新表，不会看到半成品。

“没有 CopyOnWriteLinkedList”是成本收益问题，不是语法不可能：

- 数组版收益是读无锁＋按下标直取；链表没有随机读，收益先塌一半。
- 复制成本：数组是一次内存拷贝，链表要逐个 new 节点，更贵。
- 需求端：并发频繁插删已有 ConcurrentLinkedQueue／synchronizedList 覆盖，COW 链表两头不占。

#### 技术细节

**锁到底保护了什么**

- 只保护写－写互斥：两个写线程各自复制再替换，后替换的会把先替换的更新整表抹掉。
- 读一列都不碰锁；数组字段由 `volatile`（setArray 语义）发布，保证可见性。
- 迭代器创建时锁定当前数组引用，之后该迭代器永远看这份快照，所以不抛 ConcurrentModificationException，但 add／remove 要显式调在列表上。

**代价与适用边界**

- 每次写 O(n) 复制：写多时 CPU 和 GC 都吃不消，元素又大又密集写入直接排除。
- 内存翻倍瞬间：复制期间新旧两份数组共存。
- 典型场景：读远多于写、且能容忍短暂旧数据——事件监听器列表、白名单、低频更新的配置。
- 对比：`Collections.synchronizedList` 读也要锁但没有复制成本；`ConcurrentLinkedQueue` 是节点级 CAS、只有队列语义。要“并发 Map”则去看 ConcurrentHashMap，别把 COW 思想说成通用并发容器方案。

**为什么面试官会连着问“没有 COW 链表”**

- 考的是对复制成本和数据访问模式的对应关系，不是背 API。
- 答题要点：说“数组一次 memcpy、支持随机读；链表逐节点复制、无随机读；COW 的收益在链表上兑现不了，需求又被别的类接住了”。不要答成“技术上无法实现”。

#### 深挖追问

1. **CopyOnWriteArrayList 的迭代器能删元素吗？**（补充练习）

   不能。快照迭代器的 add／set／remove 直接抛 UnsupportedOperationException；删除要调列表自身的方法，作用于“下一次替换”。

2. **写了新元素，另一个线程的 get(0) 能读到吗？**（补充练习）

   取决于替换是否已完成：替换前读到旧快照，替换后读到新数组。没有“读到一半”的状态，但有窗口期——这就是弱一致的含义。

**面经来源**

- [[面经/微步在线/一面/0001#Q10：CopyOnWriteArrayList 怎么保证线程安全？为什么没有 CopyOnWriteLinkedList？|MJ043 · 微步在线 · 一面 · Q10]]

**参考资料**（本次查证：2026-09-24）

- [Java SE 17 CopyOnWriteArrayList API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/CopyOnWriteArrayList.html)
- [OpenJDK 17u CopyOnWriteArrayList 源码](https://raw.githubusercontent.com/openjdk/jdk17u/master/src/java.base/share/classes/java/util/concurrent/CopyOnWriteArrayList.java)

### JUC-022：CountDownLatch 解决什么问题，和 CyclicBarrier、Semaphore 怎么区分？

**常见问法**

- 谈谈 CountDownLatch。

#### 面试回答

- 解决什么：“等 N 件事都完成”的闩：构造给计数 N，完成方每做完一件 `countDown()` 减一，等待方 `await()` 阻塞到归零被唤醒。
- 三个关键性质：一次性——归零后不能重置复用；`countDown()` 不阻塞——减完就走，干活方不等待；允许多个线程同时 `await`，一起被放行。
- 底层一句：AQS 共享模式——state 存计数，`tryAcquireShared` 判“是否为 0”，`tryReleaseShared` 减到 0 时向队首传播唤醒。
- 三者一句话区分：Latch 是“等 N 件事完成”；CyclicBarrier 是“等 N 个人到齐”（可复用、还能在汇合点跑 barrierAction）；Semaphore 是“抢通行许可”（限并发数）。
- 一个必说细节：工作线程里 `countDown()` 必须放 finally——有任务抛异常不计数，`await()` 就永久挂起。

#### 技术细节

**典型用法与等待者方向**

- 常见形态是“一个（或多个）等待者 ⇐ 多个干活方”：主线程等多个子任务就绪、服务启动等多个组件初始化完成（Spring 启动里也用它等异步组件）。
- `await(timeout, unit)` 版本必带超时：闩坏了（少一次 countDown）时至少能带着告警脱身。

**和 CyclicBarrier 的选型判据**

- 等待的是“别的工作完成”（计数方≠等待方）→ Latch。
- 等待的是“一组线程彼此到齐，然后一起继续”（参与方自己 await）→ Barrier；它可循环使用。
- 归零语义相反：Latch 减到 0 放行；Barrier 是人数到 N 放行并复位。

**Semaphore 别混进来**

- Semaphore 不表达“完成”，表达“额度”：拿不到许可就等，释放才放行下一批——用来限并发（见 JUC-020），拿它做“等完成”是用错工具。

**易错边界**

- `await()` 前要确认计数初值覆盖的是“事件数”而不是“线程数”——一个线程发多个事件、或事件数动态时，初值要跟着算清。
- 它不提供结果传递：要拿任务结果用 `Future`／`CompletableFuture`，Latch 只管“发生了没有”。

#### 深挖追问

1. **多个线程 await，会被一次性全部唤醒吗？**（补充练习）

   会。计数归零那次释放以共享模式向同步队列传播唤醒，所有等待者陆续拿到“已归零”的许可醒来；不存在只放一个的语义。

2. **能复用怎么办？**（补充练习）

   JDK 没有给 Latch 提供 reset——要么每轮新建一个，要么换 Phaser／CyclicBarrier；“多阶段汇合”场景 Phaser 更贴（register／arriveAndAdvance 阶段推进）。

**面经来源**

- [[面经/虾皮/一面/0002#Q10：谈谈 CountDownLatch。|MJ048 · 虾皮 · 一面 · Q10]]

**参考资料**（本次查证：2026-09-24）

- [Java SE 17 CountDownLatch API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/CountDownLatch.html)
- [OpenJDK 17u AbstractQueuedSynchronizer 源码（共享获取与释放）](https://raw.githubusercontent.com/openjdk/jdk17u/master/src/java.base/share/classes/java/util/concurrent/locks/AbstractQueuedSynchronizer.java)

### JUC-023：分段锁＋编程式事务怎么配合实现（库存分桶案例）？

**常见问法**

- [[面经/收钱吧/一面/0001#Q09：项目中分段锁＋编程式事务是如何实现的？|MJ068 · 收钱吧 · 一面 · Q09]]
- 一个大库存行并发扣减太慢，怎么拆？

#### 面试回答

结论：两件事各管一段——分段锁把“一行争用”拆成“N 行争用”，编程式事务把锁持有时间压到“一次 DB 更新”以内；配合起来治热点行更新。

- 分段：把总库存拆成 N 个桶行（`stock_bucket_0..N-1`），请求先选一个有余量的桶，再在该桶上做扣减；任一时刻争用的是桶行锁，不是全局一行。
- 编程式事务：用 `TransactionTemplate.execute(...)` 只包住“条件扣减桶库存＋写流水”两条 SQL；事务边界在代码里看得见，不会被 `@Transactional` 圈进 RPC／发 MQ。
- 竞态处理：“选桶”和“扣减”之间桶可能被抢空，所以扣减必须 `update ... set stock=stock-? where id=? and stock>=?`，影响行数 0 就换桶重试——内存计数只做提示，不做裁决。
- 收尾：这套组合把单行吞吐放大近 N 倍，代价是“查真实余量”要 SUM 多桶，对账与回补逻辑按桶写。

#### 技术细节

**TransactionTemplate 的几条纪律**

- 回滚语义：execute 回调里抛 RuntimeException 自动回滚；受检异常要么包成 Runtime，要么显式 `status.setRollbackOnly()`——这是编程式事务最常见的“以为会滚其实没滚”。
- 传播行为可在模板上设（`PROPAGATION_REQUIRES_NEW` 单独开一层），用它把“流水必落库、主流程可回滚”这类需求表达出来，比拆 Bean 自调用干净。
- `timeout` 只对事务生效，别忘了它；大事务的第一刀就是“出事务的调用全部挪走”。

**为什么事务要短**

- 事务时长≈行锁持有时间：事务里夹一个 300ms 的 RPC，等于让该桶所有并发排队 300ms。
- 分段锁解决“争用面”，短事务解决“争用时”；只做其一都不够。

**与 ConcurrentHashMap 的“分段锁”别混**

- JDK 7 的 CHM 用 Segment（继承 ReentrantLock）分段锁表数组；JDK 8 起改为 CAS＋桶头节点 synchronized（见 JAVA-COL-002）。
- 面试如果问的是集合实现，答上面那句即可；业务“分段锁”是行分片思想，两者同源不同物。

**边界与代价**

- 桶间不均：按 hash 固定路由会让热商品集中一桶，可用“随机起点轮询找桶”或定时再平衡（桶间搬运）。
- 回补要按桶回：退款／取消归还库存时回到原桶，别让空桶恒空。

#### 深挖追问

1. **能不能不用分段，直接乐观锁重试？**（补充练习）

   低争用时 version 乐观锁更简单；热点行下冲突率高，重试风暴比锁等待更贵——分段把冲突面打散后再配条件更新是常见终态。

2. **N 取多少？**（补充练习）

   按“单行可承受 TPS × N ≥ 峰值”估，并留对账成本余量；桶太多会让“查余量／回补”放大成扫多行，一般几个到几十个。

**面经来源**

- [[面经/收钱吧/一面/0001#Q09：项目中分段锁＋编程式事务是如何实现的？|MJ068 · 收钱吧 · 一面 · Q09]]

**参考资料**（本次整理为通用工程口径；TransactionTemplate 语义以 Spring 当期文档为准）

- [Spring Framework：事务管理（TransactionTemplate／编程式事务）](https://docs.spring.io/spring-framework/reference/data-binding/transaction.html)

**相关题目**

[[专题题库/MySQL#MYSQL-007：如何优化秒杀库存更新与热点行锁竞争？|MYSQL-007：热点行锁治理]]、[[专题题库/Spring#SPRING-004：@Transactional 何时不生效，如何正确调用事务方法？|SPRING-004：事务失效场景]]、[[专题题库/Java集合#JAVA-COL-002：HashMap 和 ConcurrentHashMap 有什么区别？|JAVA-COL-002：CHM 的锁粒度演进]]

### JUC-024：synchronized 的锁升级机制是什么？

**常见问法**

- 谈谈 synchronized 的锁升级机制。
- 偏向锁、轻量级锁、重量级锁是怎么转换的？
- 为什么 synchronized 需要锁升级？

#### 面试回答

结论：锁升级是 HotSpot 让对象头 Mark Word 按竞争激烈程度「无锁 →（偏向）→ 轻量级 → 重量级」单向演进，动机是低竞争时不惊动操作系统。

- 轻量级锁：CAS 把 Mark Word 拷进线程栈上的锁记录，抢不到先自旋——用几条指令的忙等换掉挂起／唤醒。
- 膨胀成重量级：自旋耗到预算或竞争加剧，锁膨胀为 monitor（依赖 OS 互斥），没抢到的线程阻塞排队，不再空转。
- 偏向锁：为“同一个线程反复进出”设计的历史路径，撤销要走 safepoint、多线程下负收益，JDK 15 起按 JEP 374 废弃默认关闭——现在答题先说“无锁→轻量级→重量级”，偏向锁作历史补充。
- 一句纠偏：一般只升不降，别说成“竞争消失自动降级”。

#### 技术细节

**轻量级锁的加解锁过程**

- 线程抢锁时先把对象头 Mark Word 拷贝到栈帧里的“锁记录”（displaced header，被挤掉的头）。
- CAS 尝试把对象头改成指向这条锁记录的指针：成功就持有轻量级锁；重入则同线程再次 CAS 到自己的另一个锁记录。
- CAS 失败且不是自己重入，说明有竞争：自适应自旋若干轮（按历史成功率调整），仍失败就膨胀。
- 解锁反向做：CAS 把锁记录里的 Mark Word 写回对象头；写不回说明期间已有人膨胀成重量级，转入重量级解锁并唤醒 EntryList 里的等待者。

**膨胀成重量级之后**

- 对象头指向一个 monitor 结构（HotSpot 内部实现），底层挂 OS 的互斥原语；没抢到锁的线程挂进 EntryList 由内核调度。
- 重量级的开销主要在“挂起＋唤醒”两次用户态／内核态切换——这正是要用轻量级自旋省掉的钱。
- `wait()` 的线程进的是 monitor 另一条等待队列（WaitSet），与 EntryList 的抢锁队列分开管（见 JUC-007 sleep／wait 区别）。

**Mark Word 的复用与牵连**

- 锁状态位和哈希码、分代年龄复用对象头同一字（JUC-003 深挖 4 同口径）：一旦调用过 `Object.hashCode()`，历史上就会破坏偏向状态——这也是偏向锁被废弃时公开列出的牵连之一。
- 所以“Mark Word 存锁状态”要限定说法：存的是指向锁记录／monitor 的指针加低位状态标记，不是单独开辟的字段。

**容易答错的地方**

- 把“锁升级”和悲观／乐观、独占／共享这些分类维度混着背——那是横切的分类（JUC-016），升级只是 synchronized 在 HotSpot 里的状态演进。
- 说“一定经历偏向→轻量→重量”：现行版本没有偏向锁了，链路是“无锁→轻量级→重量级”。
- 说“解锁后自动降回轻量级”：膨胀过的 monitor 一般不会退回，科普文的“降级”多半是把轻量级解锁时“写回 Mark Word”误读成了降级。
- 口径声明：以上全是 HotSpot 实现细节，JVMS 只定义互斥与可见性语义，答题注明版本。

#### 深挖追问

1. **为什么要升级，不直接一律用重量级锁？**（补充练习）

   为绝大多数“基本不竞争／竞争一闪而过”的对象省内核切换：无竞争时一次 CAS 就完事，短临界区自旋几轮就拿到。真高竞争再付重量级的钱——分级定价是这套机制存在的唯一理由。

2. **自旋等锁不也浪费 CPU 吗？**（面经实际出现；[[面经/拼多多/一面/0002#Q07：轻量级锁自旋获取会消耗 CPU 吗？有什么问题？自旋适合什么场景、什么时候无效？|MJ044 · 拼多多 · 一面 · Q07]]）

   会，所以 HotSpot 用自适应自旋限制圈数，并设膨胀阈值：临界区长（锁内有 IO）、线程数远超核数时自旋只会饿死持锁者，直接膨胀挂起更划算。展开见 [[#JUC-016：Java 中有哪些锁，如何按不同维度分类？|JUC-016]] 的深挖追问 4。

3. **偏向锁为什么被废弃？**（补充练习）

   它赌的是“这把锁始终只有一个线程重入”，撤销却要走到 safepoint，现代并发代码里收益不抵复杂度和性能毛刺；JDK 15 起默认禁用并标记废弃（JEP 374）。与 [[#JUC-016：Java 中有哪些锁，如何按不同维度分类？|JUC-016]] 深挖追问 2 口径一致。

**面经来源**

- [[面经/海信/电话面/0001#Q09：谈谈 synchronized 的锁升级机制|MJ073 · 海信 · 电话面 · Q09]]

**参考资料**（本次查证：2026-09-25；实现细节以所用 JDK 版本 HotSpot 源码为准）

- [JEP 374：Deprecate and Disable Biased Locking（JDK 15）](https://openjdk.org/jeps/374)
- [OpenJDK HotSpot markWord 头文件（17u）](https://github.com/openjdk/jdk17u/blob/master/src/hotspot/share/oops/markWord.hpp)