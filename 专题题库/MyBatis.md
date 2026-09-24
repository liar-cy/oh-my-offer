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
