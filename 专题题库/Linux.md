# Linux

- 题号前缀：LINUX
- 范围：Linux 如何查看 CPU 状态和系统负载？；select、poll、epoll 有什么区别？；Java 应用高 CPU 与负载异常排查。
- 最近更新：2026-09-22
- 说明：按本库面经整理；补充练习不计入原始面试问题。个人经历答案为框架，技术版本以题内说明为准。

## 目录

- [[#LINUX-001：Linux 如何查看 CPU 状态和系统负载？|LINUX-001：Linux 如何查看 CPU 状态和系统负载？]]
- [[#LINUX-002：select、poll、epoll 有什么区别？|LINUX-002：select、poll、epoll 有什么区别？]]

- [[#LINUX-003：如何提取第四列 userid 并统计 Top10？|LINUX-003：如何提取第四列 userid 并统计 Top10？]]

### LINUX-001：Linux 如何查看 CPU 状态和系统负载？

**常见问法**

- Linux 中怎么看 CPU 状态和负载？

#### 面试回答

我会先用 uptime 或 top 看 1、5、15 分钟负载，再用 top、mpstat -P ALL 1 看总体和各核 CPU，必要时用 pidstat -u 1 定位进程。load average 不是 CPU 百分比，Linux 还计入不可中断睡眠任务；高负载但 CPU 不高时，要结合 vmstat、iostat 和进程状态排查 I/O 或资源等待。

#### 技术细节

top 的 us、sy、id 等字段帮助区分用户态、内核态和空闲时间；wa 可作 I/O 等待线索，不能单独证明磁盘是根因。负载需结合可用 CPU 数量理解，容器的 CPU 配额和宿主机核数可能不同。mpstat、pidstat、iostat 通常来自 sysstat，未安装时可以先看 top 和 /proc。短时采样比单次快照更能判断趋势；记录采样窗口并区分 CPU 饱和与任务堆积。

#### 深挖追问

1. **8 核 load=8 就一定满负载吗？**（补充练习）

   不一定，负载包含不可中断等待任务，还需看 CPU 使用率、运行队列和可用 CPU 配额。

2. **单核满而总 CPU 不高怎么查？**（补充练习）

   用每核与线程视图定位热点线程，结合线程栈判断锁竞争、串行计算或绑定问题。

3. **Java 应用响应慢、CPU 负载高怎么排查？**（面经实际出现；[[面经/帆软/二面/0001#Q17：Java 应用响应慢、CPU 负载高怎么排查；CPU 低但负载高如何定位？|MJ023 · 帆软 · 二面 · Q17]]）

   `top` 找高 CPU 进程→`top -Hp <pid>` 找热点线程→线程号转 16 进制后 `jstack <pid>`（或 arthas thread -n）匹配栈顶；区分业务死循环、GC 风暴（GC 日志频繁 Full GC 同时表现为 CPU 高＋响应慢）与正则/序列化热点；容器环境注意 CPU 配额与宿主机视角差异。

4. **CPU 低但负载高，如何定位？**（面经实际出现；[[面经/帆软/二面/0001#Q17：Java 应用响应慢、CPU 负载高怎么排查；CPU 低但负载高如何定位？|MJ023 · 帆软 · 二面 · Q17]]）

   负载包含不可中断等待：`vmstat` 看 r/b 与 wa，`iostat` 查磁盘，`ps` 找 D 状态线程（NFS/磁盘/内核锁等待），或大量线程阻塞在慢依赖导致积压；根因常在 I/O、依赖变慢或线程池/队列配置，而不是计算。

**面经来源**

- [[面经/百度/一面/0001#Q01：Linux 中怎么看 CPU 状态和负载？|MJ002 · 百度 · 一面 · Q01]]
- [[面经/帆软/二面/0001#Q17：Java 应用响应慢、CPU 负载高怎么排查；CPU 低但负载高如何定位？|MJ023 · 帆软 · 二面 · Q17]]

**参考资料**（本次查证：2026-09-12）

- [Linux proc_loadavg(5)](https://man7.org/linux/man-pages/man5/proc_loadavg.5.html)

### LINUX-002：select、poll、epoll 有什么区别？

**常见问法**

- select、poll 和 epoll 有什么区别？
- I/O 多路复用中 select、poll、epoll 等有什么区别？

- 介绍 I/O 多路复用。

- 介绍 Java NIO 模型、多路复用。

#### 面试回答

I/O 多路复用让一个线程等待多个描述符就绪。select 和 poll 每次提交待检查集合并扫描结果；select 还有 fd_set 容量等限制，poll 用数组表示。epoll 在内核维护关注集合和就绪集合，适合连接多、活跃比例低的场景。它通知就绪，真正的数据读取仍要调用 read 或 recv，不等于异步 I/O。

#### 技术细节

select 需重建被修改的集合，常见 FD_SETSIZE 为 1024；poll 无该固定集合限制，但仍受进程资源限制并需遍历数组。epoll 使用 ctl 注册或修改、wait 取就绪事件；LT 在条件仍满足时继续通知，ET 通常配非阻塞 I/O 处理到 EAGAIN。不能说 epoll 所有操作都是 O(1)，大量活跃事件仍需逐个处理，少量连接未必有优势。写事件只在有待发送数据时关注，避免空转。

#### 深挖追问

1. **ET 为什么要一直读到 EAGAIN？**（补充练习）

   若尚有数据却停止读取，之后可能没有新的边缘通知；非阻塞读到暂不可读再等待可避免遗漏。

2. **epoll 返回可读后 read 一定不会阻塞吗？**（补充练习）

   存在并发消费等变化，应使用非阻塞描述符并正确处理 EAGAIN。

3. **Java NIO 是怎么实现多路复用的？**（面经实际出现；[[面经/传音控股/二面/0001#Q01：介绍 Java NIO 模型、多路复用。|MJ030 · 传音控股 · 二面 · Q01]]）

   三件套对应内核机制：Channel 设非阻塞模式后注册到 Selector（Linux 下底层即 epoll，关注集合与就绪事件由内核维护），select() 返回 SelectionKey 集合，再逐 key 处理 read/write/accept/connect 事件；Buffer 承载数据并有 position/limit 游标语义。对比 BIO“一连接一线程”，NIO 用少量线程服务大量连接，瓶颈从线程数转为事件处理效率；Netty 的 Reactor 主从线程组与原生 Transport（Linux native epoll）在此之上补齐半包/粘包与内存管理。注意 select 返回“就绪”不等于 read 不阻塞、数据一定完整。

**面经来源**

- [[面经/百度/一面/0005#Q04：介绍 I/O 多路复用。|MJ008 · 百度 · 一面 · Q04]]
- [[面经/百度/一面/0003#Q04：select、poll 和 epoll 有什么区别？|MJ006 · 百度 · 一面 · Q04]]
- [[面经/百度/一面/0001#Q02：I/O 多路复用中 select、poll、epoll 等有什么区别？|MJ002 · 百度 · 一面 · Q02]]
- [[面经/传音控股/二面/0001#Q01：介绍 Java NIO 模型、多路复用。|MJ030 · 传音控股 · 二面 · Q01]]

**参考资料**（本次查证：2026-09-12）

- [Linux epoll(7)](https://man7.org/linux/man-pages/man7/epoll.7.html)

### LINUX-003：如何提取第四列 userid 并统计 Top10？

**常见问法**

- 如何提取文件第四列 userid 并返回 Top10？

#### 面试回答

若 Top10 指出现次数最多的 userid，假设空白分隔、无表头，可用：

```sh
awk 'NF >= 4 {print $4}' input.txt | sort | uniq -c | sort -k1,1nr -k2,2 | head -n 10
```

先提取第四列，再排序让相同 ID 相邻，统计频次，按次数降序取前十；输出为次数和 userid。

#### 技术细节

原题未说明 Top10 口径；若是 userid 数值最大十项，应按第四列数值排序，不能套用频次统计。上述命令并列次数时按 ID 字符串排序，必要时统一设置 LC_ALL=C 确保排序可复现。有表头时加 NR > 1 条件；Tab 分隔可用 -F '\t'；含引号、嵌入逗号的 CSV 需使用 CSV 解析器。uniq 只合并相邻重复行，因此前面的 sort 必不可少。只有不足十个不同 ID 时输出全部。大文件可外部排序，或用 awk 哈希计数后排序，后者内存随不同 ID 数增长。

#### 深挖追问

1. **只要 userid，不要次数怎么输出？**（补充练习）

   在上述流水线后接 `awk '{print $2}'`，因为统计结果第一列是次数、第二列才是 ID。

**面经来源**

- [[面经/百度/二面/0001#Q07：如何提取文件第四列 userid 并返回 Top10？|MJ009 · 百度 · 二面 · Q07]]

**参考资料**（查证日期：2026-09-13；具体部署版本未知）

- [GNU Awk 手册：字段提取](https://www.gnu.org/software/gawk/manual/gawk.html)
- [GNU Coreutils：sort](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html)
