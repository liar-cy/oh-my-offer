# Caffeine

- 题号前缀：CAFFEINE
- 范围：从 Caffeine 获取对象是深拷贝还是浅拷贝？
- 最近更新：2026-09-12
- 说明：按本库面经整理。Java 以 17 为基准，框架差异在题内说明；补充练习不计入实际面试问题。

## 目录

- [[#CAFFEINE-001：从 Caffeine 获取对象是深拷贝还是浅拷贝？|CAFFEINE-001：从 Caffeine 获取对象是深拷贝还是浅拷贝？]]

### CAFFEINE-001：从 Caffeine 获取对象是深拷贝还是浅拷贝？

**常见问法**

- 从 Caffeine 获取对象是深拷贝还是浅拷贝？

#### 面试回答

对普通本地 Caffeine Cache，命中时通常返回当前缓存值的对象引用，不自动做深拷贝，也没有创建浅拷贝外层对象。修改返回的可变对象，可能直接改变其他读取者看到的缓存内容。缓存容器线程安全不代表里面的可变对象也线程安全，可优先存不可变值或明确做防御性复制。

#### 技术细节

限定普通 Cache／LoadingCache，未叠加自定义序列化、复制适配器等。put 保存值，getIfPresent 或命中 get 取回当前值；加载、刷新和显式替换可能更新引用，所以不是“同一 key 永远同一个对象”。源码读取节点 value 并返回，未默认执行 clone 或序列化往返。若通过 Spring Cache 或其他封装使用，要检查封装层策略。深浅拷贝概念参见 [[专题题库/Java基础#JAVA-003：深拷贝、浅拷贝与引用赋值有什么区别？|Java 对象复制]]，不要把仅共享引用误称浅拷贝。

#### 深挖追问

1. **获取后改 List，其他线程会看到变化吗？**（补充练习）

   可能访问的就是同一个 List；同时还存在对象内部数据竞争和可见性问题，应采用不可变值或统一同步。

2. **直接修改缓存对象会触发正常更新语义吗？**（补充练习）

   不能据此假定刷新、过期或权重都会按 put 更新；需要更新时显式替换值，并检查配置语义。

**面经来源**

- [[面经/北京某上市公司/二面/0001#Q04：深拷贝和浅拷贝有什么区别，从 Caffeine 获取对象属于哪一种？|MJ004 · 北京某上市公司 · 二面 · Q04]]

**参考资料**（本次查证：2026-09-12）

- [Caffeine 官方缓存填充说明](https://github.com/ben-manes/caffeine/wiki/Population)
- [Caffeine BoundedLocalCache 源码](https://raw.githubusercontent.com/ben-manes/caffeine/master/caffeine/src/main/java/com/github/benmanes/caffeine/cache/BoundedLocalCache.java)
