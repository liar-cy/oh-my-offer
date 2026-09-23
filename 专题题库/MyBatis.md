# MyBatis

- 题号前缀：MYBATIS
- 范围：MyBatis 中 #{ } 与 ${ } 有什么区别？；如何实现自定义 MyBatis 拦截器？
- 最近更新：2026-09-24
- 说明：按本库面经整理。Java 以 17 为基准，框架差异在题内说明；补充练习不计入实际面试问题。

## 目录

- [[#MYBATIS-001：MyBatis 中 ＃{ } 与 ${ } 有什么区别？|MYBATIS-001：MyBatis 中 #{ } 与 ${ } 有什么区别？]]
- [[#MYBATIS-002：如何实现自定义 MyBatis 拦截器？|MYBATIS-002：如何实现自定义 MyBatis 拦截器？]]

### MYBATIS-001：MyBatis 中 ＃{ } 与 ${ } 有什么区别？

**常见问法**

- MyBatis 中 $ 和 # 有什么区别？

#### 面试回答

#{value} 通常生成 PreparedStatement 参数占位符，由 JDBC 绑定数据值；${value} 是把内容直接拼进 SQL 文本，不能自动防止注入。普通条件值用 #{}，表名、列名等不能作为普通值占位时，应从受控白名单选择再构造 SQL，不直接拼用户输入。

#### 技术细节

例如 `WHERE name = #{name}` 对应带参数的 SQL，MyBatis 用 TypeHandler 绑定实际值；`${name}` 则在 SQL 解析阶段替换字符串，通常不会替你添加引号和转义。ORDER BY 的列名用 #{} 不会把参数变成 SQL 标识符，可用 choose 等映射允许的排序项。LIKE 可由应用构造 `%关键字%` 再绑定，或用 CONCAT 配合参数；避免为模糊查询直接改用原样拼接。使用预编译绑定降低注入风险，但其他动态 SQL 片段仍须检查。

#### 深挖追问

1. **排序列可以用 #{column} 吗？**（补充练习）

   它绑定的是值而非标识符，不能实现动态列选择；使用白名单映射允许的列名。

2. **#{} 就保证整个 SQL 没有注入吗？**（补充练习）

   只保护正确绑定的值位置，混入 ${} 或其他不可信 SQL 拼接仍有风险。

**面经来源**

- [[面经/字节/一面/0010#Q15：什么是 SQL 注入？有哪些防范措施？|MJ042 · 字节 · 一面 · Q15]]
- [[面经/北京某上市公司/二面/0001#Q10：MyBatis 中 $ 和 ＃ 有什么区别？|MJ004 · 北京某上市公司 · 二面 · Q10]]

**参考资料**（本次查证：2026-09-12）

- [MyBatis 3 Mapper XML](https://mybatis.org/mybatis-3/sqlmap-xml.html)

### MYBATIS-002：如何实现自定义 MyBatis 拦截器？

**常见问法**

- 自定义拦截器如何实现？

#### 面试回答

结合前一道 MyBatis 问题，我会先按插件回答：实现 Interceptor，使用 @Intercepts、@Signature 声明要拦截的目标类型、方法和参数签名，在 intercept 中执行前后逻辑并调用 invocation.proceed，最后通过配置或框架集成注册插件。目标可包括 Executor、StatementHandler、ParameterHandler、ResultSetHandler 的指定方法，签名必须匹配。

#### 技术细节

原文只写“自定义拦截器”，不能确认一定指 MyBatis，因此此条明确按上下文解释。插件对框架对象创建代理，不等于能任意拦截业务类。统计耗时用 try/finally 保留异常传播，不调用 proceed 会短路原行为，重复调用可能重复执行；共享拦截器不应在字段里保存每次请求的可变状态。插件顺序会影响嵌套调用，改 SQL 还需同步参数映射并考虑缓存键，不能只拼一段字符串。

若面试官指 Spring MVC：实现 HandlerInterceptor，选择 preHandle、postHandle、afterCompletion，再用 WebMvcConfigurer.addInterceptors 注册路径规则；这是 HTTP 处理链，与 MyBatis SQL 插件不同。原文没给拦截目标，暂不额外创建猜测的面试题。

#### 深挖追问

1. **为什么要调用 invocation.proceed？**（补充练习）

   继续被拦截的原方法；不调用意味着有意替代或短路，必须明确返回值与副作用。

2. **拦截器里能保存 startTime 字段吗？**（补充练习）

   共享实例会串请求，应在单次调用的局部变量中保存，避免并发覆盖。

**面经来源**

- [[面经/北京某上市公司/二面/0001#Q11：自定义拦截器如何实现？|MJ004 · 北京某上市公司 · 二面 · Q11]]

**参考资料**（本次查证：2026-09-12）

- [MyBatis 3 插件配置](https://mybatis.org/mybatis-3/configuration.html)
- [Spring MVC 拦截器配置](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/interceptors.html)
