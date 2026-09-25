# Java集合

- 题号前缀：JAVA-COL
- 范围：HashMap 整体工作原理（含哈希冲突处理：链地址法与开放寻址的取舍）；HashMap 扩容过程是什么，扩容时如何处理并发读写？；HashMap 和 ConcurrentHashMap 有什么区别？；HashSet 和 TreeSet 的底层实现与增删改查性能区别；List 的常见实现与适用场景；HashMap 树化动机与红黑树选型；线程安全集合盘点与选择。
- 最近更新：2026-09-26
- 说明：按本库面经整理；补充练习不计入原始面试问题。个人经历答案为框架，技术版本以题内说明为准。

## 目录

- [[#JAVA-COL-003：HashMap 整体是如何工作的？|JAVA-COL-003：HashMap 整体是如何工作的？]]
- [[#JAVA-COL-001：HashMap 扩容过程是什么，扩容时如何处理并发读写？|JAVA-COL-001：HashMap 扩容过程是什么，扩容时如何处理并发读写？]]
- [[#JAVA-COL-002：HashMap 和 ConcurrentHashMap 有什么区别？|JAVA-COL-002：HashMap 和 ConcurrentHashMap 有什么区别？]]
- [[#JAVA-COL-004：HashSet 和 TreeSet 在底层实现与增删改查性能上有什么区别？|JAVA-COL-004：HashSet 和 TreeSet 在底层实现与增删改查性能上有什么区别？]]
- [[#JAVA-COL-005：Java 的 List 有哪些实现？分别适合哪些应用场景？|JAVA-COL-005：Java 的 List 有哪些实现？分别适合哪些应用场景？]]
- [[#JAVA-COL-006：为什么 HashMap 要把链表转成红黑树？红黑树相比其它树结构的优势是什么？|JAVA-COL-006：为什么 HashMap 要把链表转成红黑树？红黑树相比其它树结构的优势是什么？]]
- [[#JAVA-COL-007：Java 中哪些集合是线程安全的，怎么选择？|JAVA-COL-007：线程安全集合盘点]]

### JAVA-COL-001：HashMap 扩容过程是什么，扩容时如何处理并发读写？

**常见问法**

- Java 中 HashMap 扩容的过程是什么？
- HashMap 扩容时的读和写要怎么处理？
- Map 扩容后旧数据怎么处理？
- HashMap 在多线程环境下存在哪些问题？

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

**多线程直接使用 HashMap 的问题清单**

- 丢更新：两个线程都判断同一空桶再 put，后写的覆盖先写的；size 计数同样会乱。
- 可见性没有保证：没有 happens-before，另一线程读到旧表、旧值都属正常，别指望“写完立刻可见”。
- 结构性损坏：JDK 7 并发扩容头插可成环，get 死循环打满 CPU；JDK 8 不再成环，但前两条原样存在。
- fail-fast 不是安全机制：迭代中有人改结构只是“尽力检测到才抛” ConcurrentModificationException，它不提供任何保护。

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
- [[面经/海信/电话面/0001#Q06：HashMap 在多线程环境下存在哪些问题？|MJ073 · 海信 · 电话面 · Q06]]

**参考资料**（本次查证：2026-09-12）

- [OpenJDK 8u HashMap 源码](https://github.com/openjdk/jdk8u/blob/master/jdk/src/share/classes/java/util/HashMap.java)

### JAVA-COL-002：HashMap 和 ConcurrentHashMap 有什么区别？

**常见问法**

- HashMap 和 ConcurrentHashMap 有什么区别？
- ConcurrentHashMap 的实现原理是什么？
- ConcurrentHashMap 在项目中怎么用的？key 和 value 存的什么？
- ConcurrentHashMap 的锁是怎么加的？锁粒度是什么？
- 谈谈 ConcurrentHashMap 的底层实现。

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
- [[面经/收钱吧/一面/0001#Q10：ConcurrentHashMap 在项目中如何使用？key 和 value 存什么？|MJ068 · 收钱吧 · 一面 · Q10]]
- [[面经/海信/电话面/0001#Q07：ConcurrentHashMap 的锁是怎么加的？|MJ073 · 海信 · 电话面 · Q07]]
- [[面经/字节/三面/0002#Q09：谈谈 ConcurrentHashMap 的底层实现|MJ077 · 字节 · 三面 · Q09]]
- [[面经/小米/一面/0001#Q15：HashMap 和 ConcurrentHashMap 有什么区别？|MJ078 · 小米 · 一面 · Q15]]

**参考资料**（查证：2026-09-13；适用版本见正文）

- [Java SE 21，ConcurrentHashMap API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)
- [OpenJDK 8u，ConcurrentHashMap 源码](https://github.com/openjdk/jdk8u/blob/master/jdk/src/share/classes/java/util/concurrent/ConcurrentHashMap.java)

### JAVA-COL-003：HashMap 整体是如何工作的？

**常见问法**

- 介绍一下 HashMap。
- 谈谈 HashMap 的底层数据结构。
- 如果用哈希算法做映射发生了冲突，一般怎么处理？
- 开放寻址（线性探测）和拉链法有什么区别，各自什么时候更合适？
- 什么是哈希冲突？有哪些解决哈希冲突的方案？
- 用自定义对象（如 Student）作 HashMap 的 key 要注意什么？

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

4. **哈希冲突有哪些处理办法，拉链法和开放寻址怎么选？**（面经实际出现；[[面经/字节/一面/0011#Q11：ID 映射成短链字符串怎么做，哈希冲突怎么处理，雪花算法的 64 位整数怎么编码，进制转换怎么设计？|MJ066 · 字节 · 一面 · Q11]]。原素材未记回答，以下为参考）

   两类路线：

   - 链地址法（JDK HashMap 用的）：同桶挂链表，冲突再多也只影响一个桶；删除干净，代价是每桶一次对象分配与指针跳转，缓存不友好。
   - 开放寻址：冲突就在数组里往后找空槽（线性探测、二次探测、双重哈希）；内存紧凑、局部性好，但删除要留墓碑、装载因子必须压低，且有聚集问题。

   选型口径：小对象、读多、要求缓存友好或不允许额外指针 → 开放寻址（如 JDK 的 `ThreadLocalMap`）；冲突分布不可控、删除频繁、实现要简单 → 拉链。手写实现见 [[专题题库/算法与数据结构#ALG-049：如何手写一个基于线性探测的哈希表（put／get）？|ALG-049：线性探测哈希表]]。

**面经来源**

- [[面经/微步在线/一面/0001#Q02：介绍一下 HashMap。|MJ043 · 微步在线 · 一面 · Q02]]
- [[面经/字节/一面/0011#Q11：ID 映射成短链字符串怎么做，哈希冲突怎么处理，雪花算法的 64 位整数怎么编码，进制转换怎么设计？|MJ066 · 字节 · 一面 · Q11]]
- [[面经/海信/电话面/0001#Q03：谈谈 HashMap 的底层数据结构|MJ073 · 海信 · 电话面 · Q03]]
- [[面经/海信/电话面/0001#Q04：什么是哈希冲突？有哪些解决哈希冲突的方案？|MJ073 · 海信 · 电话面 · Q04]]
- [[面经/浙江大华/电话面/0001#Q06：HashMap 的原理|MJ080 · 浙江大华 · 电话面 · Q06]]
- [[面经/用友/一面/0002#Q07：Map＜Student, String＞ 允许吗、要注意什么？HashCode 和 Equals 是什么关系？|MJ090 · 用友 · 一面 · Q07]]

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

### JAVA-COL-005：Java 的 List 有哪些实现？分别适合哪些应用场景？

**常见问法**

- Java 的 List 有哪些实现？分别适合哪些应用场景？
- ArrayList 和 LinkedList 有什么区别，什么时候用哪个？
- 在对集合的 for 循环或者 for-each 循环中执行删除操作，会有什么问题吗？

#### 面试回答

结论：实现 List 的常用类是 ArrayList、LinkedList，并发场景用 CopyOnWriteArrayList，Vector 属于历史遗留；选型看“按下标多还是头尾插删多、要不要线程安全”。

- ArrayList：动态数组。随机访问 O(1)、尾部追加摊还 O(1)；中间插删要搬元素 O(n)。默认选择——绝大多数业务都在“遍历＋按下标取”。
- LinkedList：双向链表。头尾插删 O(1)、迭代器处 remove O(1)；但 `get(i)` 要从近端遍历 O(n)，且每个节点两个指针的内存开销大。
- CopyOnWriteArrayList：每次写复制整个数组，读完全无锁、迭代器是快照不抛 CME；适合读极多写极少（监听器表、配置白名单），写频繁或元素多就是灾难。
- Vector／Stack：方法级 synchronized 的旧同步容器，锁粒度粗，并发场景已被 j.u.c 容器和 Collections.synchronizedList 取代，如实带过即可。

#### 技术细节

**为什么“中间插删多用 LinkedList”在实践中基本不成立**

- 要先 `get(i)` 或遍历定位才能插入，定位本身 O(n)，链表省掉的只是搬元素那一步。
- 数组访问有 CPU 缓存局部性，链表是指针跳转；同规模实测 ArrayList 通常更快。
- 真要“频繁头尾操作”，那是队列／栈的需求，用 ArrayDeque（比 LinkedList 更快、接口更贴），而不是把它当 List 用。

**ArrayList 的扩容与两个经典坑**

- 扩容：新容量 = 旧容量 + 旧容量右移 1 位（约 1.5 倍），再 `Arrays.copyOf`；已知规模用带初始容量的构造器避免反复扩。
- 坑一：`Arrays.asList` 返回的是定长视图，`add/remove` 抛 UnsupportedOperationException，要可变列表得 `new ArrayList<>(...)` 包一层。
- 坑二：`remove(int)` 和 `remove(Object)` 不同——装箱元素列表里 `list.remove(1)` 删的是下标；删 Integer 对象要显式传 `Integer.valueOf(1)`。

**CopyOnWriteArrayList 的准确边界**

- 单条 add／set 原子（内部 ReentrantLock＋写时复制），但“先查再改”的多步逻辑仍会竞态，size 也可能是瞬时旧值。
- 数组按元素数复制，百万级列表每次写复制百万引用——“读多写少”要少到能数得过来。

**版本差异**

- JDK 1.2：ArrayList／LinkedList 登场；Vector 是 JDK 1.0 遗留。
- JDK 1.5：concurrent 包加入 CopyOnWriteArrayList。
- JDK 8：实现层配合 lambda 重构（`forEach`／spliterator 等默认方法进接口），对外行为契约不变。
- JDK 9：`List.of(...)` 提供真正的不可变 List——拒绝 null、不允许重复语义交给业务判断，与“定长视图”是两回事。

#### 深挖追问

1. **ArrayList 存基本类型有什么代价？**（补充练习）

   只能存包装类型：`ArrayList<Integer>` 每个元素是堆上对象＋列表里存引用，千万级规模下内存和拆装箱开销都远超 `int[]`。量大且是纯数值时直接考虑数组或专用库。

2. **`Collections.synchronizedList` 和 CopyOnWriteArrayList 怎么选？**（补充练习）

   前者是“每个方法一把大锁”，读写互斥、迭代还要手动再加锁；后者读无锁、写复制。写比例不明显低就都别选，考虑分段／并发结构或把列表变不可变＋整体替换。

3. **List 迭代中删元素怎么写才对？**（补充练习）

   `for-each` 中直接 `list.remove` 会触发 fail-fast 抛 CME；正确写法是迭代器的 `remove()`、或 JDK 8 的 `removeIf`、或倒序按下标删。ArrayList 与 LinkedList 都适用这条契约。

**面经来源**

- [[面经/海信/电话面/0001#Q02：Java 的 List 有哪些实现？分别适合哪些应用场景？|MJ073 · 海信 · 电话面 · Q02]]
- [[面经/浙江大华/电话面/0001#Q05：ArrayList 和 LinkedList 的区别|MJ080 · 浙江大华 · 电话面 · Q05]]
- [[面经/用友/一面/0002#Q06：在集合的 for 或 for-each 循环中执行删除操作，会有什么问题？|MJ090 · 用友 · 一面 · Q06]]

**参考资料**（本次查证：2026-09-25）

- [Java SE 17 List／ArrayList／CopyOnWriteArrayList API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/List.html)
- [Java SE 17 Arrays.asList 说明（定长视图）](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Arrays.html#asList(T...))
- [OpenJDK 17u ArrayList 源码（`newCapacity = oldCapacity + (oldCapacity >> 1)`）](https://raw.githubusercontent.com/openjdk/jdk17u/master/src/java.base/share/classes/java/util/ArrayList.java)

### JAVA-COL-006：为什么 HashMap 要把链表转成红黑树？红黑树相比其它树结构的优势是什么？

**常见问法**

- 为什么 HashMap 会把链表转为红黑树？选用红黑树相比其它树结构的优势是什么？
- 链表太长就树化，树化解决的是什么问题？

#### 面试回答

结论：树化是给“冲突堆成链”兜底——链上查找最坏 O(n)，红黑树把它压回 O(log n)；选红黑树，是在 AVL、普通 BST、B 树之间取“查找不太差、改动不昂贵、不会退化”的平衡点。

- 触发条件（JDK 8）：所在链长度达到 8，且桶数组容量不小于 64；表还小则先扩容而不建树。
- 相比普通 BST：红黑树保证最长路径不超过最短的 2 倍，不会被插入顺序带偏退化成链——那等于白树化。
- 相比 AVL：AVL 更矮查得略快，但插删的旋转／再平衡更频繁；桶里读写混布，红黑树插入至多 2 次、删除至多 3 次旋转更划算。
- 相比 B／B+ 树：那是为“磁盘块多分叉”设计的，内存里逐行比较的单键场景没有收益，实现反而复杂。

#### 技术细节

**树化与退化的完整口径**

- 常量：`TREEIFY_THRESHOLD=8`、`UNTREEIFY_THRESHOLD=6`、`MIN_TREEIFY_CAPACITY=64`，都是 HashMap 的静态常量。
- 阈值 8 的依据（源码注释口径）：随机哈希下桶内元素数近似 λ=0.5 的泊松分布，链长达到 8 的概率约千万分之六——树化防的是“哈希质量差／被恶意构造”的极端，不是常态。
- 退化用 6 是与 8 留间隙，避免在临界点反复互转；扩容拆分树桶时，两侧若都退化到 6 以下也顺势转回链表。
- 树化后单桶查找 O(log n)，前提是 key 可比较或哈希够散： Comparable 缺失时红黑树用 `tieBreakOrder`（先比 class 名再比 identityHashCode）定序——所以自定义 key 实现 Comparable 能让树桶更快。

**“红黑树 vs AVL”说清楚换的是什么**

- 红黑树的不变量（黑高一致、无连续红）比 AVL 的严格平衡松，树可能高出一点，查找平均略慢。
- 换来的是修改局部化：一次插入的修复通常只波及近端少数节点，不触发链式再平衡——桶链表反复挂链／脱链的形态更吃这个。
- JDK 自己也是同样取舍：TreeMap 同样是红黑树；早期版本用过 AVL 后换掉是公开历史（见 JAVA-COL-004 深挖）。

**容易说过头的地方**

- 别讲成“HashMap 查找整体变 O(log n)”：定位桶仍是 O(1)，只有“超长链的那一个桶”内查找变 O(log n)。
- 别把树化说成性能优化：它是防最坏情况的兜底，树节点比普通节点更大更重，正常短链下 HashMap 根本不会用到树。
- “为什么不是跳表”这类追问可如实答：JDK 选择在容器家族里统一用红黑树（与 TreeMap 同族）；并发有序结构才见跳表（ConcurrentSkipListMap），两者比较对象不同。

#### 深挖追问

1. **为什么退化阈值是 6 而不是 8？**（补充练习；树化阈值一侧见 [[#JAVA-COL-003：HashMap 整体是如何工作的？|JAVA-COL-003]] 的深挖追问 1）

   留缓冲带。若加删一个元素就在 7／8 附近反复触发改结构，链表与树的相互转换成本比省下的查找还贵；退化线取 6，配合扩容拆树时的批量检查，避免抖动。

2. **树化后 put 还快吗？会不会更慢？**（补充练习）

   单桶内比长链快；但树节点的分配、旋转比链表追加重。所以短链（≤8）时 HashMap 宁可挂在链上——这也是“树化是兜底不是提速”的含义。

3. **元素都不实现 Comparable 会怎样？**（补充练习）

   树仍能建（tieBreakOrder 兜底定序），但同 hash 桶内比较失去“按键值短路”的机会，查找退化成一层层等价判断。自定义 key 想吃到树化收益就实现 Comparable，或者——修好 hashCode 让链根本别长到 8。

**面经来源**

- [[面经/海信/电话面/0001#Q05：为什么 HashMap 要把链表转为红黑树？红黑树相比其它树结构的优势是什么？|MJ073 · 海信 · 电话面 · Q05]]

**参考资料**（本次查证：2026-09-25；适用 JDK 8 及之后的实现）

- [OpenJDK 8u HashMap 源码（树化常量与注释）](https://github.com/openjdk/jdk8u/blob/master/jdk/src/share/classes/java/util/HashMap.java)
- [Java SE 17 HashMap API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/HashMap.html)

### JAVA-COL-007：Java 中哪些集合是线程安全的，怎么选择？

**常见问法**

- [[面经/浙江大华/电话面/0001#Q04：哪些集合是线程安全的？|MJ080 · 浙江大华 · 电话面 · Q04]]
- Vector、Hashtable、ConcurrentHashMap 这些都线程安全吗？程度一样吗？

#### 面试回答

结论：线程安全的集合按年代分三档，都叫安全，实现与代价完全不同。

- 三档：JDK 1.0 的全表锁遗留类（Vector／Hashtable／Stack）；JUC 的细粒度并发类（ConcurrentHashMap、CopyOnWrite 系、BlockingQueue）；“不改就是安全”的不可变集合。

- 遗留类：`Vector`、`Hashtable`、`Stack` 方法级 `synchronized`，一次锁整张表；复合操作（先 get 再 add）仍不原子；新代码不用。
- 并发优化类：`ConcurrentHashMap`（CAS＋桶头 synchronized，size 分片计数）、`CopyOnWriteArrayList`／`CopyOnWriteArraySet`（写时复制，读零开销）、`BlockingQueue` 家族（`ArrayBlockingQueue`／`LinkedBlockingQueue`／`PriorityBlockingQueue`，锁＋Condition 实现阻塞语义）、`ConcurrentSkipListMap`（跳表做有序并发）。
- 包装类：`Collections.synchronizedList/Map` 只是把每个方法加一把互斥锁——单方法安全，遍历与复合操作仍要调用方自己锁该包装对象。
- 不可变类：`List.of`／`Map.of`（JDK 9＋）创建后不能改，安全来自“没有写”；`Collections.unmodifiableList` 是只读视图，底层被改则视图跟着变——两者不同，别说混。

#### 技术细节

**“线程安全”要拆开问三件事**

- 单操作原子性：三档全都满足（不可变是空满足）。
- 复合操作正确性：全都不自动满足——`if (!map.containsKey(k)) map.put(...)` 即使 ConcurrentHashMap 也要换成 `putIfAbsent`／`computeIfAbsent`。
- 迭代一致性：遗留类与包装类要么 fail-fast、要么全程锁表；CHM 与 CopyOnWrite 是弱一致／快照遍历，不抛并发修改异常但可能读到旧态。

**选择口径**

- 读多写极少的小列表：`CopyOnWriteArrayList`（监听器列表是典型）；每次写复制整数组，大列表高频写直接出局。
- 共享键值映射：`ConcurrentHashMap`；需要“不存在才建且只建一次”用 `computeIfAbsent`，注意函数里不能再操作同一个 map。
- 生产者消费者：直接用 `BlockingQueue`，把“等待元素／等待空间”交给队列，而不是自己条件变量轮询。
- 只在初始化时构建、之后只读的查找表：构建完包成不可变（`Map.copyOf`／`unmodifiableMap`）发布，比并发容器更快更省。
- `Collections.synchronizedList` 的正当用途只剩一个：把旧代码里的 ArrayList 低成本过渡到单锁安全，且迭代处记得 `synchronized (list)`。

**容易说过头的地方**

- 别说“Hashtable 和 Vector 不安全”——它们安全，只是粒度粗；淘汰理由是性能与扩展性，不是正确性。
- ConcurrentHashMap 的 `size()` 是基表＋CounterCell 的估算和，并发写时只是近似；要精确计数得自己加原子变量。
- 线程安全集合解决不了跨集合不变量（两个 map 同步增删）——那需要外部锁或换一种数据结构。

#### 深挖追问

1. **为什么 JDK 9 的 List.of 允许安全发布而 Arrays.asList 不行？**（补充练习）

   `List.of` 真不可变（改即抛 `UnsupportedOperationException`），可以当常量公开；`Arrays.asList` 是数组视图——set 能写回原数组、size 不可变，既不算只读也不算线程安全。

2. **BlockingQueue 的 offer／put／take 在满和空时各是什么行为？**（补充练习）

   `put`／`take` 阻塞等待；`offer(e, timeout, unit)` 带超时；`offer`／`poll` 立即返回布尔。有界队列要显式定满时策略（阻塞、丢弃还是拒绝），这在线程池配置里就是 workQueue 行为（见 JUC-002）。

**面经来源**

- [[面经/浙江大华/电话面/0001#Q04：哪些集合是线程安全的？|MJ080 · 浙江大华 · 电话面 · Q04]]

**参考资料**（本次查证：2026-09-25）

- [Java SE 17 Collections 概览（Synchronized Wrappers 与 Unmodifiable Views 文档）](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/Collections.html)
- [Java SE 17 Concurrent 包概览（并发集合与队列）](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/concurrent/package-summary.html)

**相关题目**

[[#JAVA-COL-002：HashMap 和 ConcurrentHashMap 有什么区别？|JAVA-COL-002：CHM 细节]]、[[#JAVA-COL-005：Java 的 List 有哪些实现？分别适合哪些应用场景？|JAVA-COL-005：List 实现]]、[[专题题库/Java并发#JUC-014：Java 有哪些实现线程安全的手段？|JUC-014：线程安全手段]]、[[专题题库/Java并发#JUC-021：CopyOnWriteArrayList 是怎么保证线程安全的？为什么没有 CopyOnWriteLinkedList？|JUC-021：写时复制]]
