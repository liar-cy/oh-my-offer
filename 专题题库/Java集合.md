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

以 JDK 8 为例，HashMap 通常在元素数超过阈值后扩为原容量两倍，重新分配桶数组，再迁移节点。由于容量是二次幂，节点按 hash 与 oldCap 的按位与结果留在原索引或移动到原索引加 oldCap。普通 HashMap 本来就不支持并发修改，扩容时读写也没有安全保障；并发场景应使用 ConcurrentHashMap 或统一加锁。

#### 技术细节

阈值通常为 capacity×loadFactor，默认负载因子 0.75；极小容量的树化请求等也可能触发扩容。链表节点可分成 low/high 两组并保持组内顺序，树桶有相应拆分逻辑。迁移使用已存的扰动 hash，并不要求对每个 key 再调用 hashCode。JDK 7 某些并发扩容可能形成链表环，不应原样套到 JDK 8；JDK 8 仍可能丢更新和发生可见性问题。仅给写操作加锁而让读操作绕过锁也不构成正确的同步协议。ConcurrentHashMap 扩容有其转发与协助迁移机制，不能视作 HashMap 自带能力。

#### 深挖追问

1. **预先设置容量能避免线程安全问题吗？**（补充练习）

   不能，避免某些扩容只减少开销，并未解决并发写冲突和可见性。

2. **扩容时的读和写应该怎么办？**（面经实际出现；[[面经/百度/一面/0001#Q03：HashMap 如何扩容，扩容时的并发读写怎么处理？|MJ002 Q03]]）

   从容器选型或统一同步解决，不直接给普通 HashMap 想象出“读旧表、写新表”的安全协议。

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

HashMap 不支持无同步的并发修改，允许一个 null key 和多个 null value；ConcurrentHashMap 提供线程安全的并发访问，不允许 null 键值。ConcurrentHashMap 的单次操作安全，不等于多步业务逻辑自动原子化；先判断再插入应使用 putIfAbsent 或合适的 compute 方法。HashMap 适合单线程或由外部统一同步的场景，共享并发映射通常选 ConcurrentHashMap。

#### 技术细节

实现以 OpenJDK 8u 为例：ConcurrentHashMap 通过 CAS、桶级 synchronized 和可见性约束协调更新，空桶可 CAS 插入，冲突桶按相应路径同步，扩容有转发节点和协助迁移；不能套用 JDK 7 的固定 Segment 分段锁描述。读取通常不阻塞；迭代弱一致，并非某个时刻的全表快照。HashMap 的 fail-fast 只是尽力检测，不是线程安全机制。映射安全也不保证其 value 对象内部线程安全。

#### 深挖追问

1. **用 ConcurrentHashMap 后，get 再 put 就安全了吗？**（补充练习）

   每次调用线程安全，但组合可能丢更新；使用原子条件更新、compute／merge 或业务同步，并明确这些操作只解决对应键的更新，不提供任意跨键事务。

**面经来源**

- [[面经/百度/一面/0003#Q16：HashMap 和 ConcurrentHashMap 有什么区别？|MJ006 · 百度 · 一面 · Q16]]
- [[面经/字节/一面/0001#Q19：ConcurrentHashMap 的实现原理是什么？|MJ011 · 字节 · 一面 · Q19]]

**参考资料**（查证：2026-09-13；适用版本见正文）

- [Java SE 21，ConcurrentHashMap API](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)
- [OpenJDK 8u，ConcurrentHashMap 源码](https://github.com/openjdk/jdk8u/blob/master/jdk/src/share/classes/java/util/concurrent/ConcurrentHashMap.java)
