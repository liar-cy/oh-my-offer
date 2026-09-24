# Java集合

- 题号前缀：JAVA-COL
- 范围：HashMap 整体工作原理；HashMap 扩容过程是什么，扩容时如何处理并发读写？；HashMap 和 ConcurrentHashMap 有什么区别？；HashSet 和 TreeSet 的底层实现与增删改查性能区别。
- 最近更新：2026-09-24
- 说明：按本库面经整理；补充练习不计入原始面试问题。个人经历答案为框架，技术版本以题内说明为准。

## 目录

- [[#JAVA-COL-003：HashMap 整体是如何工作的？|JAVA-COL-003：HashMap 整体是如何工作的？]]
- [[#JAVA-COL-001：HashMap 扩容过程是什么，扩容时如何处理并发读写？|JAVA-COL-001：HashMap 扩容过程是什么，扩容时如何处理并发读写？]]
- [[#JAVA-COL-002：HashMap 和 ConcurrentHashMap 有什么区别？|JAVA-COL-002：HashMap 和 ConcurrentHashMap 有什么区别？]]
- [[#JAVA-COL-004：HashSet 和 TreeSet 在底层实现与增删改查性能上有什么区别？|JAVA-COL-004：HashSet 和 TreeSet 在底层实现与增删改查性能上有什么区别？]]

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
- [[面经/虾皮/一面/0002#Q09：谈谈 HashMap 的扩容机制。|MJ048 · 虾皮 · 一面 · Q09]]

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

### JAVA-COL-003：HashMap 整体是如何工作的？

**常见问法**

- 介绍一下 HashMap。

#### 面试回答

以 JDK 8 为例，一句结构＋四条主线：

- 结构：数组＋链表＋红黑树；数组是桶表，冲突挂在桶上。
- 定位：key 的 hashCode 高 16 位异或低 16 位做扰动，再用 `(n-1) & hash` 取桶下标；容量恒为 2 的幂。
- 冲突处理：同桶挂链表；链表长度到 8 且表容量到 64 才树化，红黑树退化到 6 变回链表。
- 扩容：元素数超过容量×负载因子（默认 0.75）就翻倍，迁移细节见 JAVA-COL-001。
- 特性：允许一个 null key；迭代无序；非线程安全，并发用 ConcurrentHashMap（见 JAVA-COL-002）。

#### 技术细节

**为什么容量必须是 2 的幂**

- `hash & (n-1)` 等价于取模但只是位运算；且扩容翻倍时，元素新位置只由“新启用的那一位”决定，一个 `hash & oldCap` 就能判留原位还是移到“原位＋oldCap”。
- 扰动（`h ^ h>>>16`）是让高位信息参与低位掩码，减少“ hashCode 只差高位”时的聚集冲突。

**put 的完整分支**

- 表未初始化：先按默认容量 16（或指定容量）建表。
- 桶为空：直接放新节点。
- 桶已有：先比哈希再 equals，命中就覆盖 value；未命中挂链或进树。
- 挂链后判断树化条件（见上），最后 `++size > threshold` 触发扩容。

**取值与键的约定**

- get 与 put 是同一套“先哈希定位、再 equals 确认”，所以 equals／hashCode 契约破坏后取不中（见 JAVA-012）。
- null key 固定放在下标 0 的桶；“HashMap 支持 null”说的是 value 也可以 null，判断“键不存在”和“值为 null”要用 `containsKey` 区分。

**构造参数怎么给**

- 已知条目数 n：初始容量约 `n/0.75 + 1`，避免中途扩容；实现里会取不小于该值、2 的幂的结果（`tableSizeFor` 语义）。
- 负载因子调大省内存换更长链；调小反之。默认 0.75 是空间时间的折中，面试不必现场改。

**版本差异**

- JDK 7：数组＋链表，头插法；并发扩容可能成环（get 死循环），是历史事故不是现行行为。
- JDK 8 起：尾插＋树化；新增 `computeIfAbsent`／`merge` 等计算接口（仍非线程安全前提下的多步原子保证）。

#### 深挖追问

1. **树化阈值为什么是 8，退化为什么是 6？**（补充练习）

   官方注释的口径：随机哈希下链长达 8 的概率约千万分之六（泊松分布），树化是给“哈希质量差到极端”兜底，把最坏 O(n) 变 O(log n)。退化用 6 是与 8 留出间隙，避免在临界点反复转换。

2. **HashMap 的迭代顺序稳定吗？**（补充练习）

   不稳定。它不是“随机”，而是由哈希和容量决定：同样内容、同样容量、同样插入序列结果可复现，但不反映插入顺序；扩容后顺序会变。要顺序用 LinkedHashMap。

3. **为什么 key 常用 String 而不是自定义可变对象？**（补充练习）

   String 不可变，hashCode 缓存且稳定；可变对象参与哈希的字段一旦修改就定位失效。自定义 key 要么保证字段不可变，要么别改。

**面经来源**

- [[面经/微步在线/一面/0001#Q02：介绍一下 HashMap。|MJ043 · 微步在线 · 一面 · Q02]]

**参考资料**（本次查证：2026-09-24；适用 JDK 8 及之后的实现）

- [OpenJDK 8u HashMap 源码](https://github.com/openjdk/jdk8u/blob/master/jdk/src/share/classes/java/util/HashMap.java)
- [Java SE 17 HashMap API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/HashMap.html)

### JAVA-COL-004：HashSet 和 TreeSet 在底层实现与增删改查性能上有什么区别？

**常见问法**

- Java 中的 HashSet 和 TreeSet 在底层实现以及性能方面有哪些区别？性能上可以说一下增删改查的复杂度。

#### 面试回答

一句话：HashSet 是“哈希换 O(1) 且无序”，TreeSet 是“红黑树换有序和 O(log n)”。

- HashSet：底层就是 HashMap——元素当 key，value 是共享哑值；add／remove／contains 平均 O(1)，冲突成链退化 O(n)、树化后 O(log n)；遍历顺序与插入无关；允许一个 null 元素。
- TreeSet：底层 TreeMap，元素当 key，红黑树按 Comparable／Comparator 排序；add／remove／contains 稳定 O(log n)，还白送有序遍历与范围操作（first／higher／subSet）；不接受 null（比较即 NPE），去重依据是 `compareTo==0` 而非 equals。
- 选型：只要去重与判存在 → HashSet；要有序遍历、极值、区间查询 → TreeSet，为 log n 与每元素树节点开销付费。
- 中间档补一句：LinkedHashSet＝哈希＋双向链表，O(1) 且保插入序——“要不要序、要哪种序”才是第一问。

#### 技术细节

**去重语义的暗坑**

- TreeSet 用比较结果当“相等”：`compareTo` 与 `equals` 不一致时，会出现“equals 不同的两个对象被当成重复丢掉”——比较器要么与 equals 一致，要么明确接受这个语义（Effective Java 的 SortedSet 警告）。
- HashSet 相反：先 hashCode 分桶再 equals 确认——所以元素类的 equals／hashCode 契约必须一起重写（见 JAVA-012）。

**“增删改查”措辞在 Set 上的对应**

- Set 没有“改”：改元素＝remove＋add；改到一半失败会丢元素，稳妥写法是新建集合替换引用。
- add 返回 boolean（是否真的加入）是两套都成立的行为契约，可用于“去重计数”式判重。

**复杂度要说前提**

- HashSet 的 O(1) 是“哈希均匀”前提下的平均情况，最坏 O(n)；TreeSet 的 O(log n) 是最坏情况保证。答“一个快一个慢”不分场景，会被追问打回。

#### 深挖追问

1. **TreeMap 的红黑树比 AVL 慢在哪，为什么还选红黑树？**（补充练习）

   单次查找 AVL 略快（更平衡、树更矮），但插入删除的旋转次数更少更便宜——红黑树用“近似平衡换写放大小”，匹配通用容器读写混多的画像。JDK 早期 TreeMap 真用过 AVL，后改红黑树是公开历史。

2. **ConcurrentSkipListSet 呢？**（补充练习）

   并发场景的“有序 Set”没有并发红黑树，用跳表实现（并发读写、O(log n) 无锁化）——与 ConcurrentHashMap 不同族，量大时注意其 size 非 O(1)。

**面经来源**

- [[面经/帆软/一面/0004#Q19：HashSet 和 TreeSet 在底层实现以及增删改查性能上有哪些区别？|MJ050 · 帆软 · 一面 · Q19]]

**参考资料**（本次查证：2026-09-24）

- [Java SE 17 HashSet／TreeSet API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/TreeSet.html)
- [OpenJDK 17u TreeSet 源码（TreeMap 包装）](https://raw.githubusercontent.com/openjdk/jdk17u/master/src/java.base/share/classes/java/util/TreeSet.java)
