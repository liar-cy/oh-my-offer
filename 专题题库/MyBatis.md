# MyBatis

- 题号前缀：MYBATIS
- 范围：MyBatis 工作原理与接口方法到 SQL 执行的转换；MyBatis 中 #{ } 与 ${ } 有什么区别？；如何实现自定义 MyBatis 拦截器？
- 最近更新：2026-09-24
- 说明：按本库面经整理。Java 以 17 为基准，框架差异在题内说明；补充练习不计入实际面试问题。

## 目录

- [[#MYBATIS-003：MyBatis 的工作原理是什么？接口方法调用是如何转成 SQL 执行的？|MYBATIS-003：MyBatis 的工作原理是什么？接口方法调用是如何转成 SQL 执行的？]]
- [[#MYBATIS-001：MyBatis 中 ＃{ } 与 ${ } 有什么区别？|MYBATIS-001：MyBatis 中 #{ } 与 ${ } 有什么区别？]]
- [[#MYBATIS-002：如何实现自定义 MyBatis 拦截器？|MYBATIS-002：如何实现自定义 MyBatis 拦截器？]]

### MYBATIS-001：MyBatis 中 ＃{ } 与 ${ } 有什么区别？

**常见问法**

- MyBatis 中 $ 和 # 有什么区别？

#### 面试回答

一句话：#{} 是参数占位符，${} 是字符串拼接。

- `#{value}`：通常生成 PreparedStatement（预编译语句）的参数占位符，实际值由 JDBC 绑定。
- `${value}`：把内容直接拼进 SQL 文本，不能自动防止注入。
- 普通条件值用 `#{}`。
- 表名、列名这类位置没法当普通值来占位，就从受控白名单里选出名字再构造 SQL，不直接拼用户输入。

#### 技术细节

**两者各自做了什么**

- 例如 `WHERE name = #{name}` 对应一条带参数的 SQL，MyBatis 用 TypeHandler 绑定实际值。
- `${name}` 则在 SQL 解析阶段就替换字符串，通常不会替你添加引号和转义。

**排序列怎么办**

- ORDER BY 的列名写成 `#{}` 不会把参数变成 SQL 标识符，动态列用 choose 等映射出允许的排序项。

**模糊查询怎么办**

- LIKE 可以由应用构造 `%关键字%` 再作为参数绑定，或者用 CONCAT 配合参数。
- 别为了方便做模糊查询就直接改成原样拼接。

**一条边界**

- 预编译绑定降低的是注入风险，但它只管到绑定的值；其他动态 SQL 片段仍须逐个检查。

#### 深挖追问

1. **排序列可以用 #{column} 吗？**（补充练习）

   不行。它绑定的是值而非标识符，实现不了动态列选择；要用白名单映射出允许的列名。

2. **#{} 就保证整个 SQL 没有注入吗？**（补充练习）

   不保证。它只保护正确绑定的值位置，同一句里混入 ${} 或其他不可信 SQL 拼接仍有风险。

**面经来源**

- [[面经/字节/一面/0010#Q15：什么是 SQL 注入？有哪些防范措施？|MJ042 · 字节 · 一面 · Q15]]
- [[面经/北京某上市公司/二面/0001#Q10：MyBatis 中 $ 和 ＃ 有什么区别？|MJ004 · 北京某上市公司 · 二面 · Q10]]

**参考资料**（本次查证：2026-09-12）

- [MyBatis 3 Mapper XML](https://mybatis.org/mybatis-3/sqlmap-xml.html)

### MYBATIS-002：如何实现自定义 MyBatis 拦截器？

**常见问法**

- 自定义拦截器如何实现？

#### 面试回答

结合前一道 MyBatis 问题，我会先按 MyBatis 插件来答，分三步：

- 实现 Interceptor 接口，用 `@Intercepts`、`@Signature` 声明要拦截的目标类型、方法和参数签名。
- 在 intercept 里写前置和后置逻辑，中间调用 `invocation.proceed()` 执行被拦截的原方法。
- 最后通过配置或框架集成把插件注册进去。

能拦的目标是 Executor、StatementHandler、ParameterHandler、ResultSetHandler 上的指定方法，签名必须匹配。

#### 技术细节

**先说前提**

- 原文只写“自定义拦截器”，不能确认一定指 MyBatis，因此此条明确按上下文解释。

**能拦什么**

- 插件是对框架对象创建代理，不等于能任意拦截业务类。
- 目标只能是那四个对象的指定方法，签名必须匹配得上。

**写 intercept 时的三个坑**

- 统计耗时用 try/finally 包住，保留异常的向外传播。
- 不调用 proceed 会短路原行为；重复调用可能把原方法执行多遍。
- 拦截器是共享实例，不应在字段里保存每次请求的可变状态。

**改 SQL 要多想一步**

- 多个插件的顺序会影响嵌套调用的层次。
- 改了 SQL 还要同步参数映射，并考虑缓存键，不能只拼一段字符串。

**如果面试官指的是 Spring MVC**

- 那是另一套：实现 HandlerInterceptor，按需要选 preHandle、postHandle、afterCompletion，再用 WebMvcConfigurer.addInterceptors 注册路径规则。
- 它是 HTTP 处理链，与 MyBatis 的 SQL 插件不同。
- 原文没给拦截目标，暂不额外创建猜测的面试题。

#### 深挖追问

1. **为什么要调用 invocation.proceed？**（补充练习）

   为了继续执行被拦截的原方法。不调用就是有意替代或短路原逻辑，必须明确自己返回什么、留下哪些副作用。

2. **拦截器里能保存 startTime 字段吗？**（补充练习）

   不能。共享实例的字段会串请求，并发时互相覆盖；这类值放在单次调用的局部变量里。

**面经来源**

- [[面经/北京某上市公司/二面/0001#Q11：自定义拦截器如何实现？|MJ004 · 北京某上市公司 · 二面 · Q11]]

**参考资料**（本次查证：2026-09-12）

- [MyBatis 3 插件配置](https://mybatis.org/mybatis-3/configuration.html)
- [Spring MVC 拦截器配置](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/interceptors.html)

### MYBATIS-003：MyBatis 的工作原理是什么？接口方法调用是如何转成 SQL 执行的？

**常见问法**

- 谈谈 Mybatis 的原理。
- Java 中明明你调用的是这个接口方法，Mybatis 是怎么转变成去执行一个 SQL 语句的？

#### 面试回答

先给一句总定位：MyBatis 是半自动 ORM——SQL 自己写，参数绑定和结果映射由框架做。执行链路四段：

- 启动解析：读 XML／注解配置，每条 SQL 变成一个 MappedStatement，以“接口全限定名.方法名”为 key 注册进 Configuration。
- 拿到“实现”：Mapper 接口没有实现类；`getMapper` 返回的是 JDK 动态代理对象（MapperProxy）。
- 方法转 SQL：调接口方法进入代理的 invoke，用接口名＋方法名查出 MappedStatement，结合实参解析动态 SQL 得到 BoundSql；`#{}` 此时已是 JDBC 占位符。
- 执行与映射：Executor → StatementHandler 调 JDBC；参数由 ParameterHandler 设入，结果集由 ResultSetHandler 经 TypeHandler 映射成对象返回。

收尾一句：接口方法与 SQL 之间的桥，就是 namespace＋id 与方法名的命名约定，加上一层代理按名字查注册表。

#### 技术细节

**各对象管什么**

- SqlSessionFactory：由 Configuration 构建，进程级单例。
- SqlSession：一次会话门面，内部持有 Executor；用完要关（Spring 托管时除外）。
- Executor：调度入口，管一级／二级缓存和 update／query；实现有 SIMPLE／REUSE（语句复用）／BATCH（批处理）。
- ParameterHandler／ResultSetHandler／TypeHandler：分别管设参、取结果、Java 类型与 JDBC 类型的转换。

**代理这层的具体机制**

- MapperRegistry 为每个接口建 MapperProxyFactory；`getMapper` 生成 MapperProxy（实现 InvocationHandler）。
- invoke 里排除 Object 方法（equals／hashCode／toString 本地执行）、default 方法直接调用；其余封装成 MapperMethod 去执行。
- 方法可重载（MappedStatement 按“接口名.方法名”缓存，同名同 id 即可共用一条 SQL）。

**动态 SQL 何时定型**

- XML 里的 `<if>`／`<foreach>` 在启动期构建成 DynamicSqlSource；每次 `getBoundSql(实参)` 才拼出这一次真正执行的 SQL 文本。
- 所以同一方法不同参数可能得到不同 prepared statement；`${}` 的替换就发生在拼文本这一步（对比见 MYBATIS-001）。

**与 Spring 集成后的样子**

- `@MapperScan`／`@Mapper` 把代理对象注册成 Bean，业务里注入的接口字段就是 MapperProxy。
- SqlSessionTemplate 是线程安全代理：同一次 Spring 事务内复用同一 SqlSession（一级缓存随事务生效），并让异常翻译成 Spring DataAccessException。
- 事务由 DataSourceTransactionManager（或 MyBatis-Plus 配置）统一管，`@Transactional` 提交回滚，不手写 commit。

#### 深挖追问

1. **为什么接口不写实现类也能被调用？**（面经实际出现；由“Mybatis 原理”追问“接口怎么转成执行 SQL”，见来源）

   动态代理生成的实现只有一层壳：壳内不写业务，只按“接口全限定名＋方法名”去注册表取配置好的 MappedStatement 执行。SQL 逻辑在配置期就备好了，代理只负责查表和跑腿。

2. **XML 里 SQL 写错，什么时候才报出来？**（补充练习）

   看错误类型：XML 语法／namespace 不合法多在启动解析期；SQL 语义错误要等真正执行、数据库拒绝时抛 SQLException 链路；方法名和 id 对不上则在首次调用时报“找不到 MappedStatement”。

3. **一级缓存会脏读吗？**（补充练习）

   同一 SqlSession 内重复 query 直接返回缓存，期间别的连接提交了变更也看不到。所以要么短会话，要么该查询显式 `flushCache`／开 `useCache=false`；跨服务场景靠 Redis 之类外部层不构成 MyBatis 一级缓存。

4. **两级缓存分别存的是什么？为什么生产常关二级缓存？**（面经实际出现；[[面经/虾皮/一面/0002#Q12：MyBatis 的两级缓存是干嘛的？分别存的是什么？|MJ048 · 虾皮 · 一面 · Q12]]）

   目的同为减少回库，范围与内容不同：

   - 一级：SqlSession 级、默认开；存“本会话执行过的语句（含参数、SQL 文本指纹）→ 结果对象引用”，同会话重复查询直接命中；提交／关闭／`clearCache` 失效。
   - 二级：namespace 级、跨会话共享、需 `<cache>`＋`cacheEnabled` 显式开；存序列化后的结果记录（按缓存 id＋记录槽组织），取用是副本，要求结果类可序列化。
   - 写入时机：语句执行完成后才进二级缓存——事务未结束前对外不可见。
   - 关掉它的三个理由：多 namespace 关联同一张表会读到旧值；多实例各存各的、天然不同步；失效粒度和内存占用不可控——共享缓存的职责交给 Redis 这层显式做更稳。

**面经来源**

- [[面经/微步在线/一面/0001#Q13：谈谈 Mybatis 的原理。调用的是接口方法，是怎么转变成执行 SQL 的？|MJ043 · 微步在线 · 一面 · Q13]]
- [[面经/虾皮/一面/0002#Q12：MyBatis 的两级缓存是干嘛的？分别存的是什么？|MJ048 · 虾皮 · 一面 · Q12]]

**参考资料**（本次查证：2026-09-24）

- [MyBatis 3 执行流程与配置文档（Executor／MapperProxy 相关章节）](https://mybatis.org/mybatis-3/configuration.html)
- [MyBatis 3 Java API（SqlSessionFactory／Mapper）](https://mybatis.org/mybatis-3/java-api.html)
- [MyBatis Spring 集成文档（SqlSessionTemplate 与事务）](https://mybatis.org/spring/3/index.html)
