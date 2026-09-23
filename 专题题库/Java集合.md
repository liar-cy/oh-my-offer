# Java集合

- 题号前缀：JAVA-COL
- 范围：HashMap 扩容过程是什么，扩容时如何处理并发读写？；HashMap 和 ConcurrentHashMap 有什么区别？
- 最近更新：2026-09-18
- 说明：按本库面经整理；补充练习不计入原始面试问题。个人经历答案为框架，技术版本以题内说明为准。

## 目录

- [[#JAVA-COL-001：HashMap 扩容过程是什么，扩容时如何处理并发读写？|JAVA-COL-001：HashMap 扩容过程是什么，扩容时如何处理并发读写？]]
- [[#JAVA-COL-002：HashMap 和 ConcurrentHashMap 有什么区别？|JAVA-COL-002：HashMap 和 ConcurrentHashMap 有什么区别？]]

### JAVA-COL-001：HashMap 扩容过程是什么，扩容时如何处理并发读写？

**常见问法**

- Java 中 HashMap 扩容的过程是什么？
- HashMap 扩容时的读和写要怎么处理？
- Map 扩容后旧数据怎么处理？

#### 面试回答

以 JDK 8 为例，分四句说：

- 什么时候扩：元素数量超过阈值（容量 × 负载因子，默认 0.75）就扩容，容量直接翻倍。
- 扩完做什么：换一个新的桶数组，把旧节点迁移过去。
- 迁移怎么算位置：因为容量始终是 2 的幂，只看 `hash & oldCap` 是 0 还是 1 —— 为 0 留在原索引，为 1 移到“原索引 + oldCap”，不需要重新算 hash。
- 并发怎么办：普通 HashMap 本来就不支持并发修改，扩容期间的读写没有任何安全保障。要并发就用 ConcurrentHashMap，或者自己把所有读写收敛到同一把锁。

#### 技术细节

**扩容的触发条件**

- 常规路径是阈值 `capacity × loadFactor`，默认负载因子 0.75。
- 不止这一条路径：链表要树化时，如果桶总数还没到 `MIN_TREEIFY_CAPACITY`（64），会先扩容而不是建树；第一次 put 时表还是 null，也会按默认容量初始化。

**节点为什么不用重算 hash**

- 容量翻倍后，新旧下标只在“新增的那一位”上不同，所以 `e.hash & oldCap` 一个掩码就能判定去向：0 → 留在 `j`，1 → 移到 `j + oldCap`。
- 链表按这个规则一次拆成 low／high 两条链，并且保持节点原有的相对顺序；树桶有对应的拆分逻辑。
- 迁移用的是节点里已经存好的扰动后 hash，不会再对每个 key 调一次 `hashCode()`。

**版本差异**

- JDK 7 用头插法，并发扩容可能把链表接成环，之后 get 时死循环 —— 这是 JDK 7 的坑，不能原样套到 JDK 8（JDK 8 改成尾插并且拆链）。
- JDK 8 不再成环，但丢更新、读到旧值、size 不准这些问题一个都没解决。

**两个容易说过头的地方**

- 只给写操作加锁、读操作绕过锁，不是正确的同步协议：读侧缺少安全发布保证，仍可能看到半初始化的状态。
- ConcurrentHashMap 的扩容有自己的转发节点和协助迁移机制，不能当成 HashMap 自带的能力来讲。

#### 深挖追问

1. **预先设置容量能避免线程安全问题吗？**（补充练习）

   不能。把初始容量开大只是省掉几次扩容开销，并发写冲突和可见性问题跟容量无关，一个都没解决。

2. **扩容时的读和写应该怎么办？**（面经实际出现；[[面经/百度/一面/0001#Q03：HashMap 如何扩容，扩容时的并发读写怎么处理？|MJ002 Q03]]）

   正确的解法在选型和同步这一层：要么换 ConcurrentHashMap，要么所有读写走同一把锁。不要给普通 HashMap 想象出一套“读旧表、写新表”的安全协议，源码里没有这个保证。

**面经来源**

- [[面经/百度/一面/0001#Q03：HashMap 如何扩容，扩容时的并发读写怎么处理？|MJ002 · 百度 · 一面 · Q03]]
- [[面经/字节/一面/0001#Q18：HashMap 如何扩容并迁移旧数据？|MJ011 · 字节 · 一面 · Q18]]

**参考资料**（本次查证：2026-09-12）

- [OpenJDK 8u HashMap 源码](https://github.com/openjdk/jdk8u/blob/master/jdk/src/share/classes/java/util/HashMap.java)

### JAVA-COL-002：HashMap 和 ConcurrentHashMap 有什么区别？

**常见问法**

- HashMap 和 ConcurrentHashMap 有什么区别？
- ConcurrentHashMap 的实现原理是什么？

#### 面试回答

- HashMap：不做同步，多线程直接共用会丢数据；允许一个 null key、多个 null value；迭代时结构被改动会抛 ConcurrentModificationException。
- ConcurrentHashMap：线程安全，key 和 value 都不许为 null；单次调用是安全的，但这不代表“先 get 再 put”这种多步逻辑也自动原子 —— 那种场景要用 putIfAbsent 或 compute 系列。
- 选型：单线程、或者外层已经统一加锁，用 HashMap 就够；多个线程共用一份可变映射，用 ConcurrentHashMap。

#### 技术细节

**写操作怎么并发（以 OpenJDK 8u 为例）**

- 目标桶是空的：直接 CAS 放入，不加锁。
- 目标桶已有节点：只对这一条链／这棵树的头节点 `synchronized`，锁的粒度是一个桶，不是整张表。
- 桶正在被别的线程扩容迁移：通过 ForwardingNode（占位用的转发节点）识别，读会转到新表，写会先协助把这块迁移完再重试。
- 所以不能再用 JDK 7 的“Segment 分段锁”来描述它，那是老实现的口径。

**读和迭代**

- get 一般不加锁，靠 table 数组元素自身的可见性保证读到已发布的节点。
- 迭代器是弱一致的：能看到迭代期间的一部分修改，不保证是某个时刻的全表快照，也不会抛 CME。
- 对比：HashMap 的 fail-fast 只是“尽力检测”，检测到才抛异常，它不是线程安全机制。

**一条容易漏的边界**

- 并发安全说的是映射这层结构（key 落到哪个槽）。value 指向的对象内部该不安全还是不安全：往里 put 一个 ArrayList，遍历时别人往里加元素照样出问题。

#### 深挖追问

1. **用 ConcurrentHashMap 后，get 再 put 就安全了吗？**（补充练习）

   不安全。每次调用各自线程安全，但组合起来仍会丢更新：get 返回 null 到 put 之间，别的线程可能已经写了同一个 key，你的写就把对方覆盖了。改用 putIfAbsent／computeIfAbsent／merge／compute 这类原子的条件操作。注意它们只保证“这一个 key”上的操作原子，不提供跨 key 的事务。

**面经来源**

- [[面经/百度/一面/0003#Q16：HashMap 和 ConcurrentHashMap 有什么区别？|MJ006 · 百度 · 一面 · Q16]]
- [[面经/字节/一面/0001#Q19：ConcurrentHashMap 的实现原理是什么？|MJ011 · 字节 · 一面 · Q19]]

**参考资料**（查证：2026-09-13；适用版本见正文）

- [Java SE 21，ConcurrentHashMap API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)
- [OpenJDK 8u，ConcurrentHashMap 源码](https://github.com/openjdk/jdk8u/blob/master/jdk/src/share/classes/java/util/concurrent/ConcurrentHashMap.java)
