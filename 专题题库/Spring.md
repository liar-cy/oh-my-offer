# Spring

- 题号前缀：SPRING
- 范围：Spring Boot 有哪些关键特性；Spring Boot 自动装配的原理是什么？；IoC 是什么，容器如何创建和管理 Bean？；AOP 的原理是什么，为什么自调用可能失效？；@Transactional 何时不生效，如何正确调用事务方法？；@Resource 与 @Autowired 有什么区别，如何按名称注入？；Spring 如何处理循环依赖，如何解决？；什么是懒加载，@Lazy 在哪里生效？；@Transactional 方法里新开线程是否还在同一事务；常用注解及其处理者；@Autowired 的注入流程与反射实现；Bean 的完整生命周期；基于 Boot 开发会用到的工具链；Spring 对 WebSocket 的支持（两条技术路线与业务侧消息确认）；Spring MVC 一次请求经过的组件链路。
- 最近更新：2026-09-26
- 说明：按本库面经整理；补充练习不计入原始面试问题。个人经历答案为框架，技术版本以题内说明为准。

## 目录

- [[#SPRING-012：Spring Boot 有哪些关键特性？|SPRING-012：Spring Boot 有哪些关键特性？]]
- [[#SPRING-013：基于 Spring Boot 开发会用到哪些工具？|SPRING-013：基于 Spring Boot 开发会用到哪些工具？]]
- [[#SPRING-014：Spring Session 中存储的是什么数据？|SPRING-014：Spring Session 存储内容与工作机制]]
- [[#SPRING-015：如何在 Spring 中实现一个简单的事务？|SPRING-015：声明式与编程式事务]]
- [[#SPRING-001：Spring Boot 自动装配的原理是什么？|SPRING-001：Spring Boot 自动装配的原理是什么？]]
- [[#SPRING-002：IoC 是什么，容器如何创建和管理 Bean？|SPRING-002：IoC 是什么，容器如何创建和管理 Bean？]]
- [[#SPRING-003：AOP 的原理是什么，为什么自调用可能失效？|SPRING-003：AOP 的原理是什么，为什么自调用可能失效？]]
- [[#SPRING-004：@Transactional 何时不生效，如何正确调用事务方法？|SPRING-004：@Transactional 何时不生效，如何正确调用事务方法？]]
- [[#SPRING-005：@Resource 与 @Autowired 有什么区别，如何按名称注入？|SPRING-005：@Resource 与 @Autowired 有什么区别，如何按名称注入？]]
- [[#SPRING-006：Spring 如何处理循环依赖，如何解决？|SPRING-006：Spring 如何处理循环依赖，如何解决？]]
- [[#SPRING-007：什么是懒加载，@Lazy 在哪里生效？|SPRING-007：什么是懒加载，@Lazy 在哪里生效？]]
- [[#SPRING-008：@Transactional 方法里新开线程执行，还在同一事务中吗？|SPRING-008：@Transactional 方法里新开线程执行，还在同一事务中吗？]]
- [[#SPRING-009：Spring 常用注解有哪些，分别由哪个扩展点处理？|SPRING-009：Spring 常用注解有哪些，分别由哪个扩展点处理？]]
- [[#SPRING-010：@Autowired 的注入流程与反射实现细节？|SPRING-010：@Autowired 的注入流程与反射实现细节？]]
- [[#SPRING-011：Spring Bean 的完整生命周期是怎样的？|SPRING-011：Spring Bean 的完整生命周期是怎样的？]]

- [[#SPRING-016：Spring 用哪些类和注解支持 WebSocket？可靠消息需要业务层自己确认吗？|SPRING-016：Spring 的 WebSocket 支持与消息确认]]
- [[#SPRING-017：Spring MVC 处理一个请求会经过哪些组件？|SPRING-017：Spring MVC 请求处理流程]]

### SPRING-001：Spring Boot 自动装配的原理是什么？

**常见问法**

- Spring Boot 自动装配的原理是什么？
- [[面经/小米/二面/0001#Q07：自定义一个组件集成到 SpringBoot 中，要做哪些操作？|MJ079 · 小米 · 二面 · Q07]]

#### 面试回答

一句话：自动装配就是“先导入一批候选配置类，再用条件筛掉不适用的”。

- 入口：`@SpringBootApplication` 里的 `@EnableAutoConfiguration`，它负责导入自动配置的候选类。
- 候选从哪读：Spring Boot 3 主要从 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 取候选项。
- 谁能生效：靠 `@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty` 这类条件逐个判定。
- 生效之后是什么：本质上仍然是往 IoC 容器里注册 Bean，没有另一套运行时机制。
- 为什么我定义了自己的就不生效：很多默认配置自带“你没有我才配”的条件，你定义了同类 Bean，它会自动退让。

#### 技术细节

**候选清单放在哪里（版本差异）**

- Spring Boot 2.7 引入了 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`，现代 Boot 的候选列表就写在这个文件里。
- 更老的常见实现是通过 `spring.factories` 注册 `EnableAutoConfiguration` 的候选；这是版本差异，不能把旧方式套到所有版本上。

**谁导入、谁决定生效**

- 由选择器（import selector）把候选类导入进来。
- 导入不等于注册：`@ConditionalOnClass`、`@ConditionalOnMissingBean` 这些条件才控制最终注不注册。

**两组容易混的说法**

- 组件扫描找的是业务组件，自动配置导入的是框架配置，两者机制不同。
- starter 常见的用法是聚合依赖，它不等于自动配置类本身。

#### 深挖追问

1. **为什么自定义 Bean 后默认 Bean 不生效？**（补充练习）

   常见原因是 `@ConditionalOnMissingBean` 的条件不再满足了。到底卡在哪个条件，用条件报告确认，不要认定“覆盖总是按加载顺序发生”。

2. **自动配置不生效怎么排查？**（补充练习）

   按顺序查：依赖在不在、候选有没有注册上、属性开关、排除项，最后看条件报告；再核对当前版本用的是哪种配置方式。

3. **自定义一个组件集成到 Spring Boot，要做哪些操作？**（面经实际出现；[[面经/小米/二面/0001#Q07：自定义一个组件集成到 SpringBoot 中，要做哪些操作？|MJ079 · 小米 · 二面 · Q07]]）

   反向走一遍自动装配链条即可：

   - 写 `@AutoConfiguration` 配置类，把组件注册成 `@Bean`。
   - 用 `@ConfigurationProperties` 暴露 `xxx.*` 配置项。
   - 加 `@ConditionalOnClass`／`@ConditionalOnMissingBean` 让使用者能覆盖。
   - 在 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 登记配置类全限定名（2.7 前写 `spring.factories`）。

   发布形态按官方惯例拆两个构件：`xxx-spring-boot-autoconfigure` 放装配代码，`xxx-spring-boot-starter` 只聚依赖不含代码——使用者加一个 starter 依赖就能用。

**面经来源**

- [[面经/用友/一面/0001#Q07：Spring Boot 自动装配的原理是什么？|MJ001 · 用友 · 一面 · Q07]]
- [[面经/微步在线/一面/0001#Q12：谈谈 SpringBoot 的自动装配原理。|MJ043 · 微步在线 · 一面 · Q12]]
- [[面经/虾皮/一面/0002#Q13：谈谈 SpringBoot 的启动过程。|MJ048 · 虾皮 · 一面 · Q13]]
- [[面经/小米/二面/0001#Q07：自定义一个组件集成到 SpringBoot 中，要做哪些操作？|MJ079 · 小米 · 二面 · Q07]]

**参考资料**（本次查证：2026-09-19）

- [Spring Boot 自动配置](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)
- [Spring Boot 自定义自动配置](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html)

### SPRING-002：IoC 是什么，容器如何创建和管理 Bean？

**常见问法**

- IoC 是什么，具体原理是什么？
- 谈谈对 Spring IOC 的理解。
- 谈谈 Spring 的实现原理；Spring 是怎么根据 XML 或注解创建出对象的？

#### 面试回答

IoC（Inversion of Control，控制反转）是把对象创建和依赖管理交给容器，DI（依赖注入）是实现依赖装配的具体方式。

- 容器做的事：先读配置形成 BeanDefinition，再在需要时实例化对象、注入依赖，执行初始化与后处理，最后按作用域管理使用和销毁。
- 业务类的感受：声明自己依赖什么就行，不必到处手动 new 再手工组装对象。

#### 技术细节

**容器的分层**

- `ApplicationContext` 在 `BeanFactory` 的基础上提供更完整的应用能力。

**两个扩展点的作用阶段不同**

- BeanDefinition 阶段的扩展，拿到的是定义而不是实例。
- Bean 实例阶段的 `BeanPostProcessor` 拿到的是实例，并且可以返回代理对象。

**注入方式和循环依赖的关系**

- 构造器注入让依赖关系很明确，但互相依赖的构造器没法靠“提前暴露实例”解决——这时对象还没造出来。
- 某些单例的属性注入循环，在特定设置下可以通过早期引用处理；不能由此推出所有循环依赖都能解决。
- 更合理的处理通常是调整职责或依赖方向。

#### 深挖追问

1. **Bean 默认单例是否线程安全？**（补充练习）

   不保证。单例只说明容器里的实例数量，共享可变状态仍然需要同步，或者改成无状态设计。

2. **IoC 和 AOP 怎么关联？**（补充练习）

   关联点就是容器的后处理：Bean 创建过程中后处理器可以返回代理对象，之后的调用就会进入切面逻辑。

3. **IOC 在做项目的过程中，有哪些应用场景？**（面经实际出现；[[面经/海信/电话面/0001#Q15：IOC 在做项目的过程中有哪些应用场景？|MJ073 · 海信 · 电话面 · Q15]]。原素材未记回答，以下为参考）

   按“换实现不改调用方、横切逻辑统一织入、对象交给容器管”三类归拢，每类要落到自己项目里的真实例子：

   - 一个接口多个实现：Service 只依赖接口，【支付渠道／存储／消息发送】的实现靠配置或 `@Primary`／`@Qualifier` 切换；测试时注入 Mock 替身——这是 DI 最直接的红利。
   - 横切能力：事务、日志、限流用注解＋代理统一织入（AOP 建立在容器后处理之上），不用每个方法手写。
   - 重资源与配置：连接池、线程池做成 Bean 由容器创建和销毁；`@Value`／配置类把环境差异注入出去，代码不写死。
   - 可加分的一句：用 `ApplicationEventPublisher` 发事件，协作方之间不直接依赖。

   别只报“解耦”三个字——每个【】换成真实用法才站得住。

4. **XML 和注解两种配置，容器分别怎么把它们变成对象？**（面经实际出现；[[面经/浙江大华/二面/0001#Q05：谈谈 Spring 的实现原理；Spring 是怎么根据 XML 或注解创建出对象的？|MJ081 · 浙江大华 · 二面 · Q05]]）

   两条解析入口、一条创建流水线：

   - XML：`XmlBeanDefinitionReader` 解析 `<bean>` 元素，把 class、构造参数、属性值、init／destroy 方法记成 `BeanDefinition` 注册进容器。
   - 注解：
     - `ClassPathBeanDefinitionScanner` 扫描 `@Component`／`@Service` 派生注解生成定义。
     - `@Configuration`＋`@Bean` 方法把返回类型注册为定义。
     - `@Autowired` 注入本身由 `AutowiredAnnotationBeanPostProcessor` 在属性填充阶段反射完成。
   - 汇合：两者最终都在 `BeanDefinitionRegistry` 里同质——`refresh()` 预实例化单例时统一走“实例化→填充→初始化→代理”流程（见 SPRING-011），所以行为一致、可混用。
   - 反射点：真正“造对象”的是 `BeanUtils.instantiateClass`（构造器反射）或工厂方法调用——所谓 Spring 创建对象，本质是容器替你反射 new 并管住后续。

**面经来源**

- [[面经/用友/一面/0001#Q08：IoC 和 AOP 是什么，具体原理是什么？|MJ001 · 用友 · 一面 · Q08]]
- [[面经/海信/电话面/0001#Q14：谈谈对 Spring IOC 的理解|MJ073 · 海信 · 电话面 · Q14]]
- [[面经/浙江大华/二面/0001#Q05：谈谈 Spring 的实现原理；Spring 是怎么根据 XML 或注解创建出对象的？|MJ081 · 浙江大华 · 二面 · Q05]]
- [[面经/招银网络科技/一面/0001#Q04：谈谈你对 IOC 和 AOP 的理解|MJ082 · 招银网络科技 · 一面 · Q04]]

**参考资料**（本次查证：2026-09-12）

- [Spring IoC 容器](https://docs.spring.io/spring-framework/reference/core/beans/introduction.html)

### SPRING-003：AOP 的原理是什么，为什么自调用可能失效？

**常见问法**

- AOP 是什么，具体原理是什么？

- 动态代理有哪几种？

- AOP 是什么，底层原理是什么？

- 公共字段填充、限流这类功能为什么要引入 AOP 和注解？
- AOP 的作用是什么？主要解决什么问题？好处是什么？
- 谈谈 AOP 的原理以及应用场景。
- 除了用 AOP 实现日志输出，还有什么方式记录日志并减少对业务代码的侵入？

#### 面试回答

Spring AOP 的做法是在目标对象外面套一层代理，把事务、日志这类逻辑织到方法调用的前后。

- 代理两种常见实现：JDK 动态代理（基于接口）和 CGLIB（运行时生成子类）。
- 生效前提：调用必须经过代理对象，才会进入拦截器链。
- 自调用为什么失效：同一个对象内部用 `this` 调方法，绕过了代理，对应的切面自然不生效。
- 默认用哪种代理，要看具体的 Spring 与 Boot 配置，别背成一个固定答案。

#### 技术细节

**三个概念怎么配合**

- 切点决定匹配哪些方法，通知定义什么时候执行。
- 代理负责把这些通知串成一条调用链。

**两种代理的能力边界**

- 接口代理和子类代理能做的事不一样，不能当成可互换。
- CGLIB 是靠子类覆盖生效的，所以 `final`、`private` 方法没法通过覆盖来增强。
- 增强到底会不会发生，看三件事：调用的是代理对象还是目标对象、方法可见不可见、这一下是不是走了 `this`。

**“动态代理有哪几种”怎么答**

- 从实现机制讲：JDK Proxy 以接口为代理契约，方法调用转交给 `InvocationHandler`；CGLIB 属于运行时生成子类的方式。
- 别说成“全 Java 只有两个方案”：Byte Buddy 之类也能构造代理。
- 另外两类要区分开：静态代理是手工写代理类；AspectJ 编织不是同一种代理调用路径，它和基于代理的 Spring AOP 本身就是两套机制。

**声明式事务不能只看注解**

- 有没有 `@Transactional` 只是其中一面，还受异常回滚规则、方法可见性、调用入口的影响。

**自调用的正确处理方式**

- 优先把需要被拦截的行为拆到独立 Bean。
- 不要为了绕开自调用，随手在业务代码里获取当前代理。

#### 深挖追问

1. **为什么加了事务注解还不回滚？**（补充练习）

   依次排查四件事：调用有没有经过代理、抛的异常符不符合回滚规则、异常有没有被吞掉、传播行为是怎么设置的。不能一句话归因到数据库。

2. **JDK 代理一定比 CGLIB 快吗？**（补充练习）

   不能脱离版本和调用场景下结论。先按接口和代理能力的要求选型，再针对实际负载去测。

3. **公共字段（创建时间、更新人）为什么要用 AOP 填充，怎么实现？**（面经实际出现；[[面经/京东健康/二面/0001#Q02：公共字段的填充为什么要引入 AOP，怎么实现的，有什么作用？|MJ018 · 京东健康 · 二面 · Q02]]）

   - 为什么：这类字段每个写操作都要填，散着写容易漏，口径还不统一。
   - 怎么做：切面拦截写方法或带标记注解的入口，在环绕通知里从登录上下文和系统时间取值填进实体。
   - MyBatis 场景：也可以用 MyBatis-Plus 的 `MetaObjectHandler`／`Executor` 拦截器，在 SQL 执行前统一填充。
   - 作用：横切逻辑集中、业务代码无感、字段口径统一。
   - 两个坑：同类自调用不走代理，会失效；异步线程里拿不到当前用户，上下文要显式传递。

4. **除了 AOP，还有什么方式记录日志并减少对业务代码的侵入？**（面经实际出现；[[面经/浙江大华/二面/0001#Q07：除了用 AOP 实现日志输出，还有什么方式记录日志并减少对业务代码的侵入？|MJ081 · 浙江大华 · 二面 · Q07]]）

   按“能拦到哪一层”给梯度答：

   - 入口层：Servlet Filter 或 Spring MVC `HandlerInterceptor` 统一记请求 URL、参数、耗时与结果码——覆盖所有对外接口，但看不到方法内部与业务分支。
   - 日志框架层：logback／log4j2 的 pattern 自动带类名、行号、线程，`MDC` 放 traceId 让每行日志可串链路（Filter 里放、异步线程用 TaskDecorator 传递）——业务只打日志不拼格式。
   - 字节码层：`-javaagent` 探针（SkyWalking／Pinpoint，或自写 ByteBuddy transformer）在方法进出时增强——真正零业务代码，代价是运维接入与性能评估。
   - 结构层：Spring 事件把“记录动作”从主流程解耦、装饰器包一层计时——仍要显式发事件或包对象，侵入介于中间。
   - 选型句：
     - 接口审计日志用拦截器就够。
     - 全链路方法级观测上 APM 探针。
     - AOP 适合“少量关键方法＋需要业务上下文”的日志。

**面经来源**

- [[面经/京东健康/二面/0001#Q02：公共字段的填充为什么要引入 AOP，怎么实现的，有什么作用？|MJ018 · 京东健康 · 二面 · Q02]]
- [[面经/京东健康/一面/0001#Q10：限流部分怎么实现的，为什么要用 AOP 和注解，有哪些作用？|MJ018 · 京东健康 · 一面 · Q10]]
- [[面经/字节/一面/0012#Q02：Java 的 AOP 是什么？作用是什么？主要解决什么问题？好处是什么？|MJ075 · 字节 · 一面 · Q02]]

- [[面经/百度/一面/0006#Q15：AOP 是什么，底层原理是什么？|MJ010 · 百度 · 一面 · Q15]]

- [[面经/北京某上市公司/一面/0001#Q09：动态代理有哪几种？|MJ004 · 北京某上市公司 · 一面 · Q09]]
- [[面经/虾皮/一面/0002#Q08：谈谈对 Spring 中 JDK 动态代理的理解。|MJ048 · 虾皮 · 一面 · Q08]]
- [[面经/用友/一面/0001#Q08：IoC 和 AOP 是什么，具体原理是什么？|MJ001 · 用友 · 一面 · Q08]]
- [[面经/浙江大华/二面/0001#Q06：谈谈 AOP 的原理以及应用场景|MJ081 · 浙江大华 · 二面 · Q06]]
- [[面经/浙江大华/二面/0001#Q07：除了用 AOP 实现日志输出，还有什么方式记录日志并减少对业务代码的侵入？|MJ081 · 浙江大华 · 二面 · Q07]]
- [[面经/招银网络科技/一面/0001#Q04：谈谈你对 IOC 和 AOP 的理解|MJ082 · 招银网络科技 · 一面 · Q04]]

**参考资料**（本次查证：2026-09-12）

- [Spring AOP 代理机制](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)

### SPRING-004：@Transactional 何时不生效，如何正确调用事务方法？

**常见问法**

- @Transactional 注解什么时候失效？
- 如何在非方法内调用另一个事物方法？
- Test 类里 a（无注解）调用加了 @Transactional 的 b，事务生效吗？a 也加上呢？

#### 面试回答

Spring 声明式事务默认是靠 AOP 代理拦截方法调用的，一句话：没走代理就没有事务。常见的失效情况有——

- 这个对象是你手动 `new` 出来的，不在容器里。
- 同类里用 `this` 调事务方法，绕过了代理。
- 代理拦不到目标方法。
- 异常被捕获之后没有再抛出去。
- 抛出的异常类型不符合回滚规则。

做法上稳妥的说法：把事务方法放到独立的 Spring Bean 里，通过注入进来的代理对象调用；确实要在同类中自己控制事务边界时，用 `TransactionTemplate`。

#### 技术细节

**注解只是元数据**

- 注解本身不干活，要靠事务基础设施和一个正确的事务管理器。

**回滚规则（有版本差异）**

- 默认配置下，`RuntimeException`、`Error` 通常触发回滚，受检异常通常不触发；可以用 `rollbackFor` 等属性调整。
- Spring 6.2 起还允许全局改变默认回滚策略，所以这里的结论要看配置。

**方法可见性（也有版本差异）**

- 类代理靠覆盖方法做增强，`private` 方法覆盖不了。
- Spring 6.0 起，`protected`／包可见方法在类代理下默认可支持；走接口代理时，入口仍需是 public 的接口方法。
- 别把它简化成“非 public 全失效”。

**线程与调用入口**

- 常见 JDBC 事务绑定在当前线程上，不会自动传播到新线程。
- 外层普通方法去调另一个 Bean 的代理事务方法，事务可以开起来；同类里直接调用，就不会应用被调方法自己的注解。
- 外层已经有事务时，内部的数据库操作仍可能参与外层事务 —— 这和“内部注解有没有被拦截”是两回事。
- 处理上优先拆独立服务，必要时用 `TransactionTemplate` 明确代码块的边界。

**原文题意不完整的地方**

- 原文“如何在非方法内调用另一个事物方法”语义不完整：如果指非事务方法，就按上面跨 Bean／自调用的区别回答；如果真指构造器或初始化阶段，应把事务工作放到代理就绪之后的入口去做，不要依赖初始化中的代理事务。

#### 深挖追问

1. **异常被 catch 后为什么不回滚？**（补充练习）

   异常没有传播到事务拦截器、也没标记 rollback-only，拦截器看到的就是正常返回。要按业务规则重新抛出或显式标记回滚，不能机械地 catch 所有异常。

2. **调用另一个事务方法一定新开事务吗？**（补充练习）

   不一定。`REQUIRED` 通常是加入已有事务，`REQUIRES_NEW` 才会按配置挂起外层并新建。前提还是那一条：调用要经过代理。

**面经来源**

- [[面经/北京某上市公司/一面/0001#Q08：@Transactional 何时失效，如何从其他入口调用事务方法？|MJ004 · 北京某上市公司 · 一面 · Q08]]
- [[面经/京东零售/一面/0001#Q10：如何使用 Spring 控制事务？|MJ034 · 京东零售 · 一面 · Q10]]
- [[面经/淘天/一面/0002#Q21：如何在 Spring 中实现一个简单的事务？|MJ071 · 淘天 · 一面 · Q21]]
- [[面经/淘天/一面/0002#Q22：Test 类里 a 调用加了 @Transactional 的 b，事务生效吗？a 也加上呢？|MJ071 · 淘天 · 一面 · Q22]]

**参考资料**（本次查证：2026-09-19）

- [Spring @Transactional 使用规则](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [Spring 声明式事务回滚规则](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html)

### SPRING-005：@Resource 与 @Autowired 有什么区别，如何按名称注入？

**常见问法**

- @Resource 和 @Autowired 有什么区别？
- 如何使用 @Autowired 按名称注入？
- 一个接口有两个实现类，Controller 层用 @Autowired 注入使用，会存在什么问题吗？如何解决？

#### 面试回答

两个注解来自不同体系，找依赖的出发点也不一样：

- `@Autowired`：Spring 自己的注解，主要按类型解析依赖，可以配合 `@Qualifier` 限定候选。
- `@Resource`：属于 Jakarta 注解体系，在 Spring 里可以用 `name` 明确按 Bean 名称注入；不指定 name 时，通常先拿字段或属性名去匹配，再按适用规则回退。
- 想用 `@Autowired` 点名某个 Bean：写成 `@Autowired` 配 `@Qualifier("beanName")`，但这仍然要求类型匹配。

#### 技术细节

**包名有版本差异**

- 现代 Spring 用 `jakarta.annotation.Resource`，旧项目里可能是 `javax.annotation.Resource`。

**能标注的位置不同**

- `@Autowired` 可用于构造器、字段和方法；只有一个构造器的场景通常还能省略它。
- `@Resource` 通常用于字段和 setter，不作为构造器参数的注解。

**@Qualifier 不是“按名字直接取对象”**

- 它的核心作用是缩小按类型筛出来的候选集合。
- Bean 名可以作为回退匹配的值，但不是忽略类型、随便取一个对象。
- 字段或参数的同名回退，还会受候选优先级和参数名元数据影响；写显式的 qualifier 更容易读。

```java
// 片段：假设容器中有名为 orderService 的 OrderService Bean
@Autowired
@Qualifier("orderService")
private OrderService service;
```

也可用 `@Resource(name = "orderService")` 表达明确的名称依赖。

#### 深挖追问

1. **有两个同类型 Bean，@Primary 与 @Qualifier 怎么选？**（补充练习）

   分工不同：`@Primary` 负责在候选里定一个默认优先的 Bean，`@Qualifier` 负责在具体注入点上限定“我要哪一个”。这个依赖有明确角色时，用限定写死更清楚。

2. **@Autowired 能只按名称忽略类型吗？**（补充练习）

   不能。`@Qualifier` 仍然只在类型兼容的候选里筛选，不会跳过类型检查。真要按 Bean 身份表达时，用 `@Resource(name = ...)`。

3. **一个接口有两个实现类，`@Autowired` 注入会存在什么问题？**（面经实际出现；[[面经/用友/一面/0002#Q11：一个接口两个实现类，Controller 层用 @Autowired 注入会有什么问题？怎么解决？|MJ090 · 用友 · 一面 · Q11]]）

   问题：按类型解析出现两个候选 Bean，注入点又没限定，容器无法决定选谁，启动时抛 `NoUniqueBeanDefinitionException`（`required=true`）；字段名恰好等于某个 Bean 名时会“碰巧注入成功”，但这是回退匹配，不是解法。

   - 首选在具体注入点限定：`@Autowired` ＋ `@Qualifier("beanName")`，或改 `@Resource(name = "beanName")`。
   - 有明确主选时用 `@Primary` 定默认，其他注入点仍可点名覆盖；两者并存时限定符优先于 `@Primary`。
   - 需要全量策略时用 `List<PayService>`／`Map<String, PayService>`（key 为 Bean 名）注入后按渠道路由——这是策略模式的标准写法，比在每处 `@Qualifier` 更耐改。
   - 想避免字符串 Bean 名，自定义带 `@Qualifier` 的注解（如 `@Alipay`），可编译期检查。
   - 结构层面：一个接口对应两个实现且调用点必须点名，往往说明职责该拆成两个接口。


**面经来源**

- [[面经/北京某上市公司/一面/0001#Q10：@Resource 与 @Autowired 有什么区别，如何按名称注入？|MJ004 · 北京某上市公司 · 一面 · Q10]]
- [[面经/小红书/一面/0001#Q03：@Autowired 的底层原理；实现时怎么通过反射获取成员变量、用哪个反射方法取 Field、又用哪个方法检查它是否被 @Autowired 标记？|MJ040 · 小红书 · 一面 · Q03]]
- [[面经/用友/一面/0002#Q11：一个接口两个实现类，Controller 层用 @Autowired 注入会有什么问题？怎么解决？|MJ090 · 用友 · 一面 · Q11]]

**参考资料**（本次查证：2026-09-12）

- [Spring @Autowired 与 Qualifier](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/autowired-qualifiers.html)
- [Spring @Resource 注入](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/resource.html)

### SPRING-006：Spring 如何处理循环依赖，如何解决？

**常见问法**

- 如何解决循环依赖？
- 什么是循环依赖？Spring 是如何解决循环依赖的？

#### 面试回答

循环依赖就是 A 依赖 B、B 又依赖 A。回答分三层：

- 首选：把环本身消掉——拆分职责、抽出公共服务，或者改成事件协作。
- 延后拿：某些场景可以在注入点用 `@Lazy`，或者用 `ObjectProvider`，把真正解析依赖的时刻推后。
- 容器兜底：允许循环引用时，容器能用早期引用处理一部分单例的属性注入循环。但构造器循环、原型（prototype）循环、复杂代理情形并非都能解决。

#### 技术细节

**三层缓存分别放什么**

- 完整单例、早期单例引用、以及创建早期引用用的工厂，一共三层。

**为什么这种环能解开**

- A 先实例化、再填属性；这中间 B 需要 A 时，可以拿到 A 的早期引用。
- 那个工厂的作用是：让后处理器提供合适的代理，避免最终代理和早期对象对不上。

**解不开的情况**

- 构造器环在对象还没实例化时就在互相等，套用不了上面的流程。

**两个容易说过头的地方**

- 是否允许循环引用，要看 Spring Boot 的版本和配置；打开开关不等于修好了设计问题。
- 注入点上的延迟代理确实能推迟依赖解析，但你在构造或初始化过程中就立刻调用它，又可能把原来的环触发一遍。

与 [[#SPRING-002：IoC 是什么，容器如何创建和管理 Bean？|IoC 生命周期]]、[[#SPRING-007：什么是懒加载，@Lazy 在哪里生效？|懒加载]] 联合复习。

#### 深挖追问

1. **三级缓存是为所有循环依赖准备的吗？**（补充练习）

   不是。它解释的是特定单例怎么提前暴露、以及代理怎么协调；换作用域或者是构造器循环，它解决不了。

2. **开允许循环引用就可以了吗？**（补充练习）

   那只是改了容器的处理策略。开了之后仍要检查：会不会拿到半初始化的对象、代理是否一致、业务职责本身合不合理。结论仍是优先消除环。

**面经来源**

- [[面经/北京某上市公司/一面/0001#Q11：如何解决循环依赖？|MJ004 · 北京某上市公司 · 一面 · Q11]]
- [[面经/小红书/一面/0001#Q04：请讲一下 Spring 中一个 Bean 的生命周期。|MJ040 · 小红书 · 一面 · Q04]]
- [[面经/海信/电话面/0001#Q16：什么是循环依赖？Spring 是如何解决循环依赖的？|MJ073 · 海信 · 电话面 · Q16]]

**参考资料**（本次查证：2026-09-12）

- [Spring 依赖注入与循环依赖](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)

### SPRING-007：什么是懒加载，@Lazy 在哪里生效？

**常见问法**

- 懒加载是什么？

#### 面试回答

Spring 的懒加载，就是把 Bean 的创建从“容器启动时”推迟到“第一次真正需要它时”。同一个 `@Lazy` 写在两个位置，含义不一样：

- 写在 Bean 定义上：控制这个 Bean 的初始化时机。
- 写在注入点上：注入进来的是一个延迟解析的代理，第一次调用时才去取真目标。
- 一个容易忽略的前提：如果某个非懒加载单例在启动时就直接依赖它，那即使被依赖的 Bean 标了懒加载，也可能因为要满足依赖而被提前创建。

#### 技术细节

**默认行为与代价**

- 默认情况下，非懒加载的单例通常随容器预实例化。
- 懒加载能缩短启动路径，但代价是把初始化开销和配置错误一起推迟到首次使用时才暴露。

**两个位置的效果不是一回事**

- 注入点上的代理是在实际调用时才解析目标，这和只给目标 Bean 标 `@Lazy` 不是同一个效果。
- 想更显式地延迟获取，可以用 `ObjectProvider.getObject`。

**用在循环依赖上要看触发时点**

- 关键是“什么时候真正去解析”。不能在构造器里立刻用这个代理，然后宣称依赖已经解开了。

**别和同名概念混**

- 这里说的是 Spring Bean 的懒加载，不要和 ORM 的懒加载、网页资源的懒加载混为一谈。

#### 深挖追问

1. **懒加载会让 Bean 变成多例吗？**（补充练习）

   不会。scope 管的是实例数量，懒加载管的是什么时候创建，两个维度互不影响；懒加载的单例创建之后仍按单例管理。

2. **第一次调用慢怎么办？**（补充练习）

   把首次初始化的耗时算进响应预算里。确需预热的关键路径就显式预热，或者干脆不给它加懒加载。

**面经来源**

- [[面经/北京某上市公司/一面/0001#Q12：懒加载是什么？|MJ004 · 北京某上市公司 · 一面 · Q12]]

**参考资料**（本次查证：2026-09-12）

- [Spring 延迟初始化](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-lazy-init.html)
- [Spring @Lazy API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Lazy.html)

### SPRING-008：@Transactional 方法里新开线程执行，还在同一事务中吗？

**常见问法**

- 主方法有 `@Transactional`，内部依次调用 `serviceA.updateA()` 和 `serviceB.updateB()`；把 `serviceA.updateA()` 放到一个新线程执行，A 表和 B 表的更新还会在同一个事务中吗？

#### 面试回答

不会在同一个事务里。原因是 Spring 的声明式事务上下文绑定在**当前线程**上：

- `TransactionSynchronizationManager` 用 `ThreadLocal` 保存事务状态，数据库连接也是从线程绑定的资源里取的。
- 主方法开事务后，`serviceB.updateB()` 还在同一线程，复用的是那个事务连接。
- 一旦把 `updateA()` 交给新线程，新线程的 `ThreadLocal` 是空的，拿不到那个连接。
- 于是 `updateA()` 要么以自动提交的方式独立执行，要么自己新开一个事务，**不会**随主事务一起提交或回滚。

后果是原子性被打破：主方法后面抛异常回滚时，`updateB` 回滚了，`updateA` 却已经独立生效。

怎么取舍，三条路选一条：

- 接受“这是两个独立事务”，为失败写补偿／对账，走最终一致。
- 在新线程内部用编程式事务自己管理边界，并把异常处理掉。
- 如果只是想“事务提交后再异步做副作用”，用 `TransactionSynchronization.afterCommit` 或事务事件在提交后触发，而不是在事务中途甩给新线程。

#### 技术细节

**传播行为不跨线程**

- `REQUIRES_NEW`／`NESTED` 解决的是同一调用线程内、经过代理的事务嵌套。
- 它不会跨线程传播：`@Transactional` 不会把 `ThreadLocal` 上下文带进子线程。

**和 @Async 组合时**

- 异步方法一定在新线程执行，因此必然脱离调用方的事务。

**根因在连接绑定**

- 连接绑定靠 `TransactionSynchronizationManager` 维护的 `DataSource → Connection` 的 `ThreadLocal` 映射。
- 这既是“同一线程复用同一连接”的基础，也是“换线程就换（或丢）事务”的根因。

#### 深挖追问

1. **那怎样才能让新线程里的操作也纳入原事务？**（面经实际追问）

   实际上做不到“真正的同一物理事务跨线程回滚”。能做的是：改回同一线程（不用新线程）、把它设计成独立事务＋失败补偿、或者用事务提交后的回调保证“只在成功后才异步执行”。

   既要并行执行又要保证一致时，通常靠业务层拆分＋对账，而不是指望 Spring 事务跟着线程走。

   来源：[[面经/京东零售/一面/0001#Q11：@Transactional 主方法里把其中一个 service 调用放到新线程执行，A、B 两张表还在同一事务吗？|MJ034 · 京东零售 · 一面 · Q11]]

**面经来源**

- [[面经/京东零售/一面/0001#Q11：@Transactional 主方法里把其中一个 service 调用放到新线程执行，A、B 两张表还在同一事务吗？|MJ034 · 京东零售 · 一面 · Q11]]

**参考资料**（本次查证：2026-09-23）

- [Spring 声明式事务的实现（TransactionSynchronizationManager 与线程绑定）](https://docs.spring.io/spring-framework/reference/data-access/transaction.html)
- [Spring `@Async` 与事务](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)（异步在新线程执行，具体行为以所用版本文档为准）

### SPRING-009：Spring 常用注解有哪些，分别由哪个扩展点处理？

**常见问法**

- Spring 框架常用的注解有哪些？
- 你平时用过哪些 Spring 注解，它们分别在什么阶段生效？
- AOP 相关注解有哪些？

#### 面试回答

报名字没有区分度，我的讲法是“按用途分类，每类再带一句机制”：

- 容器与配置类：`@SpringBootApplication`（＝`@EnableAutoConfiguration`＋`@Configuration`＋`@ComponentScan`）、`@Configuration`／`@Bean`、`@Component` 及派生的 `@Service`／`@Repository`／`@Controller`、`@Import`、`@Conditional*`。这一类由配置类解析与自动装配处理。
- 注入类：`@Autowired`、`@Value`、`@Resource`。由 `BeanPostProcessor` 在属性填充阶段处理。
- Web 类：`@RestController`、`@RequestMapping` 及其派生、`@PathVariable`、`@RequestBody`、`@Valid`。由 `DispatcherServlet` 与处理器适配器／参数解析器处理。
- 横切类：`@Transactional`、`@Aspect` 家族、`@Cacheable`、`@Async`。都靠代理生效，所以共用“自调用与私有方法失效”这一类坑。
- 生命周期与顺序类：`@PostConstruct`／`@PreDestroy`、`@Lazy`、`@Primary`、`@Order`，分别落在初始化回调、按需创建、候选仲裁、排序这几个环节。

收尾给一句判断标准：注解只是声明式的外衣，真正要掌握的是它在哪个扩展点、被谁处理。被问“为什么我的注解没生效”时，答案几乎都在“没走代理／没交给容器／时机不对”这三类里。

#### 技术细节

**处理者要能点名，追问会往下钻一层**

- 组件扫描：把 `@Component` 系的定义注册成 `BeanDefinition`。
- 自动装配：靠 `@EnableAutoConfiguration` 导入 `AutoConfigurationImportSelector` 读取候选配置，并受 `@Conditional*` 过滤（见 [[#SPRING-001：Spring Boot 自动装配的原理是什么？|SPRING-001：Spring Boot 自动装配的原理是什么？]]）。
- `@Autowired` 的注入与反射细节见 [[#SPRING-010：@Autowired 的注入流程与反射实现细节？|SPRING-010：@Autowired 的注入流程与反射实现细节]]。
- `@Transactional` 的传播行为、隔离级别与失效场景见 [[#SPRING-004：@Transactional 何时不生效，如何正确调用事务方法？|SPRING-004：@Transactional 何时不生效]]。
- AOP 代理与自调用见 [[#SPRING-003：AOP 的原理是什么，为什么自调用可能失效？|SPRING-003：AOP 的原理与自调用失效]]。

**几组高频延伸的取舍**

- `@Value` 逐个绑定、支持 SpEL（Spring 表达式语言），但配置散落在字段上。
- `@ConfigurationProperties` 批量绑定到类型安全的配置类，还支持校验与 IDE 提示；成组的配置优先用它。
- `@Primary` 与 `@Qualifier` 解决同类型多候选的仲裁（见 SPRING-005）。
- `@Order`／`@Priority` 影响切面顺序与列表注入顺序。

**三个初始化回调的时机**

- `@PostConstruct`（JSR-305／jakarta 注解）在属性填充之后执行。
- `InitializingBean#afterPropertiesSet` 与自定义 `init-method` 依次排在它后面。
- 三者的时机不要混着说。

**答题范围怎么收**

- 与其列 30 个名词，不如挑 5—6 个能讲到处理者与失效场景的。
- 项目里没用到的（如 `@Cacheable`、`@Async`）就说“了解机制但项目未使用”。
- 被追问线程与事务的关系时能给出口径（见 [[#SPRING-008：@Transactional 方法里新开线程执行，还在同一事务中吗？|SPRING-008：事务与新线程]]）。

#### 深挖追问

1. **`@Component` 和 `@Bean` 有什么区别，什么时候必须用 `@Bean`？**（补充练习）

   - `@Component`：写在类上、由扫描发现，怎么构造、依赖给谁都由容器推断。
   - `@Bean`：写在配置类的方法上。三类情况必须用它：
     - 第三方类改不了源码、加不了注解。
     - 需要自己传构造参数或写初始化逻辑。
     - 同一类型要多实例／按条件创建。

2. **`@Autowired` 能用在哪些位置，哪种推荐？**（补充练习）

   字段、构造器、setter 与方法参数都可以，推荐构造器注入。

   - 好处：依赖显式、可以声明为 final、便于测试，也不会绕过校验。
   - 代价：碰到循环依赖时，没有字段注入那种“提前暴露”的余地（见 [[#SPRING-006：Spring 如何处理循环依赖，如何解决？|SPRING-006：循环依赖]]）。

3. **为什么 `@Transactional` 标在 `private` 方法上不生效？**（补充练习）

   事务增强要靠代理去覆盖方法，而 Spring 的注解解析与 CGLIB／JDK 代理都要求方法可见并且可覆写。`private`／`final` 方法覆盖不了，自调用又不走代理，所以事务增强根本不会应用上去。

**面经来源**

- [[面经/小红书/一面/0001#Q02：Spring 框架常用的注解有哪些？|MJ040 · 小红书 · 一面 · Q02]]
- [[面经/招银云创/二面/0001#Q05：怎么理解注解？|MJ056 · 招银云创 · 二面 · Q05]]
- [[面经/招银网络科技/一面/0001#Q05：AOP 有哪些注解？|MJ082 · 招银网络科技 · 一面 · Q05]]
- [[面经/用友/一面/0002#Q10：Spring 有哪些常用注解？|MJ090 · 用友 · 一面 · Q10]]

### SPRING-010：@Autowired 的注入流程与反射实现细节？

**常见问法**

- @Autowired 注解的底层原理是什么？
- 实现依赖注入时，是怎么通过反射获取成员变量的？具体用什么反射方法获取 Field？
- 获取到字段后，用什么方法检查它是否被 @Autowired 标记？
- 让你自己实现一个最小版的依赖注入，怎么写？

#### 面试回答

`@Autowired` 由 `AutowiredAnnotationBeanPostProcessor`（实现 `InstantiationAwareBeanPostProcessor`）处理，时机是 Bean 实例化之后的属性填充阶段：

- `postProcessProperties` 取出这个 Bean 的注入元数据，逐个元素把依赖解析出来再写回去。
- 解析时把字段包成 `DependencyDescriptor`，交给 `DefaultListableBeanFactory.doResolveDependency`，按类型查候选。
- 多个候选时的顺序：先看 `@Qualifier`，再看 `@Primary`，再看优先级，最后回退到按字段名匹配。
- `required=true` 且一个候选都没有，就抛 `NoSuchBeanDefinitionException`。
- 注意构造器注入不在这一步：它是实例化阶段由容器选定构造器时完成的。

三个反射细节：

- 取成员变量用 `Class#getDeclaredFields()`：它包含本类声明的私有字段，但不含父类的；父类字段要沿继承链向上遍历，Spring 内部用的是 `ReflectionUtils.doWithFields` 做这个遍历。
- 方法注入用 `getDeclaredMethods()`，构造器用 `getDeclaredConstructors()`，拿到的都是 `java.lang.reflect` 里的元对象。
- 判断有没有标 `@Autowired`：语义上最直接的是 `field.isAnnotationPresent(Autowired.class)`，但 Spring 实际用的是 `AnnotationUtils.findAnnotation(field, Autowired.class)`。原因是它还要支持元注解（在自己定义的注解里再标 `@Autowired`）和合成注解，而 `isAnnotationPresent` 只看直接标注。

最后一步是写入：写私有字段之前必须先 `field.setAccessible(true)`（Spring 封装成 `ReflectionUtils.makeAccessible`），再用 `field.set(bean, value)` 完成注入。

性能上不用担心“每次启动后还在反射扫描”：首次解析出的注入元素会被缓存成 `InjectionMetadata`，之后只是遍历缓存。

自己实现最小版时，上面的流程可以直接照抄：容器启动后遍历每个 Bean 的 `getDeclaredFields()`，命中注解的字段按类型或名称从 Bean 定义表里取实例，`setAccessible` 之后写进去。但要补三件事才谈得上可用：

- 循环依赖：字段注入可以靠提前暴露未完成对象解决，构造器注入无解。
- 多候选仲裁：`@Qualifier`／`@Primary`。
- 代理对象上的注解查找：用 `findAnnotation`，而不是 `isAnnotationPresent`。

#### 技术细节

**字段注入的代价（常被追问）**

- 字段可以先为 null 也通过编译。
- 脱离容器就没法构造：测试要用反射，或者用 `ReflectionTestUtils`。
- 依赖变多时不报错也不显形。
- 循环依赖被“静默容忍”。
- 构造器注入把这几点换成了显式失败加 final 字段，这就是 Spring 官方推荐构造器注入的原因。

**反射的两个隐性成本**

- `setAccessible` 和 `getDeclaredFields` 都有开销。Spring 靠缓存把重复反射降到最低：`ReflectionUtils` 的声明字段缓存、`InjectionMetadata`。
- Java 9 模块系统下，访问非导出包会额外受限：跨模块的类要加 `--add-opens` 才能反射。这就是“同样代码在 JDK 17 抛 `InaccessibleObjectException`”的常见原因。

**注解本身还要能读到**

- 注解有保留策略要求：`@Autowired` 是 `RUNTIME` 保留，否则运行时反射读不到。被问“自定义注解为什么读不到”，先看 `@Retention`。

**和相邻注解的关系**

- 与 `@Resource` 的分派差别（按名优先、由 `CommonAnnotationBeanPostProcessor` 处理）见 [[#SPRING-005：@Resource 与 @Autowired 有什么区别，如何按名称注入？|SPRING-005：@Resource 与 @Autowired 有什么区别]]。
- `@Value` 的占位符解析发生在同一阶段的 `DependencyDescriptor` 解析里，SpEL 也在这里求值。

#### 深挖追问

1. **`@Autowired` 和 `getBean()` 拿对象有区别吗？**（补充练习）

   拿到的是同一个容器里的同一份单例，差别在时机和表达方式：`@Autowired` 是容器在装配期注入，依赖是声明式的；`getBean()` 是主动去拉，容易把容器当成服务定位器用，把真实依赖藏起来，而且会绕开部分后置处理的时机。

2. **为什么 `@Autowired` 能注入到被 CGLIB 代理的类字段里？**（补充练习）

   因为顺序是“先注入、后代理”：注入发生在目标实例上，代理是在初始化后的处理阶段才生成的。字段声明在目标类的层次里，`findAnnotation`／`doWithFields` 会沿继承链查找，所以代理不会让注解“消失”。如果注入的是接口类型的限定 Bean，真正决定注入结果的是候选解析和 `@Qualifier`。

3. **同类中多个 Bean 满足同一类型，容器怎么选？**（补充练习）

   依次看：`@Qualifier` 匹不匹配、有没有 `@Primary`、`@Priority` 的顺序，最后用注入点的名字去匹配 Bean 名。仍选不出来就抛 `NoUniqueBeanDefinitionException`。

   所以给同类多实例的字段起一个和 Bean 名一致的名字，确实可能“碰巧生效”，但别依赖这一点，显式写 `@Qualifier` 更稳。

**面经来源**

- [[面经/小红书/一面/0001#Q03：@Autowired 的底层原理；实现时怎么通过反射获取成员变量、用哪个反射方法取 Field、又用哪个方法检查它是否被 @Autowired 标记？|MJ040 · 小红书 · 一面 · Q03]]

**参考资料**（查证：2026-09-23）

- [Spring Framework Reference：Annotation-based container configuration](https://docs.spring.io/spring-framework/reference/core/beans/annotation.html)
- [Java 17 `Class#getDeclaredFields`／`Field#setAccessible`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/reflect/Field.html)

### SPRING-011：Spring Bean 的完整生命周期是怎样的？

**常见问法**

- 请讲一下 Spring 中一个 Bean 的生命周期。
- 一个 Bean 从创建到销毁经历了哪些步骤？
- [[面经/小米/一面/0001#Q11：Spring 中的类在启动之后会执行哪些方法、用到哪些注解？|MJ078 · 小米 · 一面 · Q11]]
- 构造方法和 @Autowired 哪个先执行？
- @PostConstruct 和实现 InitializingBean 重写 init 方法，哪个先执行？

#### 面试回答

以单例 Bean 为主线，分六段讲：

- 实例化：读 `BeanDefinition`，推断或选定构造器，创建原始实例。构造器注入在这一步完成。
- 属性填充：`BeanPostProcessor` 介入，`@Autowired`／`@Value` 在这里写入。循环依赖也是在这一步靠提前暴露半成品对象解决的。
- Aware 回调：容器把自身的资源交给 Bean，比如 `BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware`。
- 初始化前：先走 `postProcessBeforeInitialization`，随后 `@PostConstruct`、`InitializingBean#afterPropertiesSet`、自定义 `init-method` 依次执行。
- 初始化后：走 `postProcessAfterInitialization`。AOP 代理通常在这一步生成并替换原始对象，所以“自调用绕过代理”这类问题都源于此。
- 使用与销毁：容器关闭时，按 `@PreDestroy`、`DisposableBean#destroy`、`destroy-method` 的顺序回收。

再补三点显示深度：

- 原型（prototype）Bean：容器只负责创建与装配，不执行销毁回调，客户端要自己管生命周期。
- 代理可能在初始化之后才被替换，而注入给其他 Bean 的应是最终对象 —— 这正是三级缓存要提前暴露“可能已经代理过”的引用（`getEarlyBeanReference`）的原因。
- 定位思路：实例化失败多在构造与依赖解析阶段。
  - `BeanCurrentlyInCreationException` 对应循环依赖。
  - 字段为 null 通常是这个对象被 `new` 出来而没交给容器。

#### 技术细节

**主干三段与两个扩展点**

- 主干好记：创建（实例化＋装配）→ 初始化（回调与前后处理）→ 销毁（清理）。
- `BeanPostProcessor` 是对所有 Bean 生效的横切扩展点，AOP、`@Autowired` 都要靠它。
- `BeanFactoryPostProcessor` 在 Bean 定义阶段生效，拿到的是定义而不是实例；自动装配的候选配置处理、属性占位符替换都发生在这里（见 [[#SPRING-001：Spring Boot 自动装配的原理是什么？|SPRING-001：Spring Boot 自动装配的原理是什么？]]）。

**初始化三兄弟的顺序与语义**

- `@PostConstruct`：jakarta 注解，由 `InitDestroyAnnotationBeanPostProcessor` 触发，排在最先。
- `afterPropertiesSet`：接口方法，由容器强约束。
- `init-method`：配置里用字符串指定的方法名，最灵活也最易拼错。
- 业务代码统一用 `@PostConstruct` 就行，需要被 XML／配置覆盖时才用 `init-method`。
- 销毁侧是同样一套：`@PreDestroy` 与 `destroy` 方法／`destroy-method`。

**提前初始化与延迟初始化的边界**

- `@Lazy` 让 Bean 到首次使用时才创建（见 [[#SPRING-007：什么是懒加载，@Lazy 在哪里生效？|SPRING-007：什么是懒加载]]）。
- SmartInitializingSingleton／`ApplicationRunner` 这类，是给“全部单例就绪之后”的启动动作用的。
- 常见错误是把依赖其他 Bean 的启动逻辑写进 `@PostConstruct`：那时邻居 Bean 可能还没装配完。
- 循环依赖的完整机制、以及为什么构造器注入救不了，见 [[#SPRING-006：Spring 如何处理循环依赖，如何解决？|SPRING-006：Spring 如何处理循环依赖]]。

#### 深挖追问

1. **为什么 AOP 代理不在实例化时就生成，而要放在初始化之后？**（补充练习）

   因为增强要等 Bean 自身配置就绪：注解、属性、初始化回调的结果。而且 `postProcessAfterInitialization` 才是容器约定的“最终对象”产出点。提前生成，初始化回调就会作用在代理而不是目标对象上，语义更难解释。

2. **三级缓存分别在什么时候被用到？**（补充练习）

   正常无环的流程只走单例池（一级）加“创建中”的标记。只有出现循环依赖、需要把未完成对象提前暴露时，才用到二级（早期引用）和三级（对象工厂 `ObjectFactory`，用来按需生成代理）；这么分层是为了既解开环，又保证代理只创建一次。

3. **Bean 生命周期问题怎么在真实项目里定位？**（补充练习）

   按“哪一段炸的”反查：

   - 找不到符号、或者依赖是 null：这个对象没经过容器创建。
   - 启动阶段抛循环依赖异常：按 SPRING-006 处理。
   - 初始化里读不到其他 Bean 的状态：时机太早，改用 SmartInitializingSingleton 或事件。
   - 关闭时报连接池已关：销毁顺序问题，需要显式控制依赖或者提前 flush。

4. **构造方法和 @Autowired 哪个先执行？**（面经实际出现；[[面经/小米/一面/0001#Q13：构造方法和 @Autowired 哪个先执行？|MJ078 · 小米 · 一面 · Q13]]）

   构造方法先。生命周期顺序是“实例化（调构造）→ 属性填充（@Autowired 由 `AutowiredAnnotationBeanPostProcessor` 反射写入）”。

   推论：构造方法体里读 `@Autowired` 字段一定是 null。构造器注入是例外视角：依赖先解析成参数、再随构造一次性传入，对象诞生即依赖齐全——这也是推荐构造器注入、循环依赖对它无解的共同原因。

5. **@PostConstruct 和 InitializingBean#afterPropertiesSet 哪个先执行？**（面经实际出现；[[面经/小米/一面/0001#Q14：@PostConstruct 注解和实现 InitializingBean 重写 init 方法，哪个先执行？|MJ078 · 小米 · 一面 · Q14]]）

   `@PostConstruct` 先。`InitDestroyAnnotationBeanPostProcessor` 在初始化前置回调里先触发它，随后才是容器强约束的 `afterPropertiesSet`，最后是配置指定的 `init-method`。三者时机都在“属性填充之后、代理生成之前”，业务代码统一用 `@PostConstruct` 即可；销毁侧顺序对称为 `@PreDestroy` → `destroy()` → destroy-method。

**面经来源**

- [[面经/小红书/一面/0001#Q04：请讲一下 Spring 中一个 Bean 的生命周期。|MJ040 · 小红书 · 一面 · Q04]]
- [[面经/虾皮/一面/0002#Q13：谈谈 SpringBoot 的启动过程。|MJ048 · 虾皮 · 一面 · Q13]]
- [[面经/小米/一面/0001#Q11：Spring 中的类在启动之后会执行哪些方法、用到哪些注解？|MJ078 · 小米 · 一面 · Q11]]
- [[面经/小米/一面/0001#Q13：构造方法和 @Autowired 哪个先执行？|MJ078 · 小米 · 一面 · Q13]]
- [[面经/小米/一面/0001#Q14：@PostConstruct 注解和实现 InitializingBean 重写 init 方法，哪个先执行？|MJ078 · 小米 · 一面 · Q14]]
- [[面经/浙江大华/二面/0001#Q05：谈谈 Spring 的实现原理；Spring 是怎么根据 XML 或注解创建出对象的？|MJ081 · 浙江大华 · 二面 · Q05]]

**参考资料**（查证：2026-09-23）

- [Spring Framework Reference：Container lifecycle／Bean 生命周期回调](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html)

### SPRING-012：Spring Boot 有哪些关键特性？

**常见问法**

- 谈谈 SpringBoot 有哪些关键的特性？
- Spring 有哪些核心特性，展开讲讲？（问法未指明 Boot，先澄清范围再按 Boot 口径展开）

#### 面试回答

按“各解决什么问题”报主特性，不背形容词：

- 自动装配：starter 带依赖，`@EnableAutoConfiguration` 按条件自动注册默认配置——解决“Spring 配置量大、上手慢”（原理见 SPRING-001）。
- 起步依赖与版本仲裁：`spring-boot-starter-parent` 统一管理传递依赖版本——解决“两个库各带一个 Jackson”的冲突地狱。
- 内嵌容器：Tomcat／Jetty／Undertow 打进可执行 jar，`java -jar` 就跑——解决“打 war、装容器、对齐版本”的部署重活。
- 约定优于配置：默认值覆盖大多数场景，`application.yml` 只写例外；改环境靠 profile。
- 生产就绪：Actuator 提供健康检查、指标、信息端点，天然对接监控与编排探针。

收尾一句：这些合起来把“从脚手架到上线”的路径标准化了，微服务才可能批量生产。

#### 技术细节

**每个特性和老流程的对照**

- 自动装配对比的是以前手抄一堆 `@Bean` 配置／XML；starter 聚合的是 dependencyManagement 里的版本对齐。
- 内嵌容器改变部署形态：应用自己监听端口，水平扩容就是多起进程；war 外置容器的共享部署方式基本不再用。
- 约定不排斥定制：AutoConfiguration 全部可覆盖（自己定义同类 Bean 即退让，见 SPRING-001），profile 解决多环境差异。

**易混点**

- starter ≠ 自动配置类：starter 常见作用只是“聚合一组依赖＋一份配置约定”，真正干活的是它带进来的 autoconfigure 包（Boot 官方模块常常成对：`xxx-spring-boot-starter` 与 `xxx-spring-boot-autoconfigure`）。
- Actuator 端点默认暴露要收敛（health／info 之外按需开，走安全层），这是面试提一嘴能加分的工程点。

**版本口径**

- Boot 3.x 基线 JDK 17、命名空间 `jakarta.*`；谈特性遇到细节差异（如 imports 文件替代 spring.factories）以所用版本文档为准。

#### 深挖追问

1. **不用 Spring 能做出这些效果吗？Boot 是不是黑魔法？**（补充练习）

   都不是魔法：自动装配本质是“读清单＋条件判断＋注册 Bean”，还是 IoC 容器那一套（SPRING-002）。Boot 只是把重复劳动模板化；理解模板背后的机制，排查问题时才出得去模板。

2. **Boot 应用启动时容器里发生了什么？**（面经实际出现；[[面经/虾皮/一面/0002#Q13：谈谈 SpringBoot 的启动过程。|MJ048 · 虾皮 · 一面 · Q13]]）

   `SpringApplication.run` 的主干按顺序走：

   1. 推断应用类型与加载初始化器／监听器。
   2. 准备 Environment（配置文件与 profile）。
   3. 创建并 refresh 容器：BeanFactoryPostProcessor 含自动配置评估、注册 BeanPostProcessor、实例化单例。
   4. 内嵌容器随刷新启动监听端口。
   5. Runner 回调。
   6. 发布 `ApplicationReadyEvent`。

   全程事件贯穿，失败路径发 Failed 事件。单例实例化细节见 SPRING-011，逐条展开版另见 [[面经/虾皮/一面/0002#Q13：谈谈 SpringBoot 的启动过程。|MJ048 一面 Q13]] 的面试回答。

**面经来源**

- [[面经/拼多多/一面/0002#Q12：谈谈 SpringBoot 有哪些关键特性。|MJ044 · 拼多多 · 一面 · Q12]]
- [[面经/虾皮/一面/0002#Q13：谈谈 SpringBoot 的启动过程。|MJ048 · 虾皮 · 一面 · Q13]]
- [[面经/网易云音乐/一面/0001#Q05：Spring 有哪些核心特性？展开讲讲|MJ067 · 网易云音乐 · 一面 · Q05]]

**参考资料**（本次查证：2026-09-24）

- [Spring Boot Overview（官方文档：features／starters／Actuator）](https://docs.spring.io/spring-boot/reference/)
- [Spring Boot 自动配置](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)

### SPRING-013：基于 Spring Boot 开发会用到哪些工具？

**常见问法**

- 基于 SpringBoot 做开发的过程中，会涉及用到哪些工具？
- 你平时开发一个 Boot 项目，工作流里都有什么？

#### 面试回答

结论：按开发流程分组讲，每样带一句用途——面试官要听的是工作流，不是工具清单。

- 构建与依赖：Maven／Gradle；依赖冲突用 `mvn dependency:tree` 看树，用 `exclusion`／`dependencyManagement` 收敛版本。
- 编码与热反馈：IDEA、Lombok 省样板代码、`spring-boot-devtools` 做类路径变更自动重启。
- 配置与接口：`application.yml`＋profile 分环境、`@ConfigurationProperties` 绑配置、springdoc／Knife4j 出 OpenAPI 文档、Postman 或 curl 联调。
- 数据层：MySQL 客户端、HikariCP（Boot 默认连接池）、MyBatis／MyBatis-Plus 与 SQL 日志、MyBatis Generator 出基础 CRUD。
- 中间件与可观测：redis-cli／RDME 看缓存、MQ 管理控制台看堆积、Actuator 暴露健康与指标给监控系统。
- 测试与定位：JUnit 5＋Mockito＋Testcontainers、Logback 日志、线上定位用 Arthas、性能问题用 JDK 自带的 jcmd／JFR 或 VisualVM、压测用 JMeter。
- 交付：Git＋分支规范、Docker 起本地依赖、`spring-boot-maven-plugin` 打可执行 jar 进 CI。

#### 技术细节

**每样工具解决 Boot 开发的哪一步**

- devtools：自动重启比冷启动快，是因为它用两套类加载器——不变的三方库留在 base 类加载器，只重建 restart 类加载器；IDEA 里要触发它得真正执行 Build（保存文件不会）。
- devtools 还会顺手改掉一批开发期默认值，例如把模板与静态资源缓存关掉、开 H2 控制台；打包运行时这些自动失效，所以它不会把「缓存关闭」带到生产。
- Actuator：给探针和监控用的端点集合（health、info、metrics 等）。默认只暴露 `health`，其余要靠 `management.endpoints.web.exposure.include` 显式打开。
- 配置绑定：`@ConfigurationProperties` 比一串 `@Value` 好维护，配合配置元数据处理器还能在 IDE 里得到 yml 提示。
- Testcontainers：把 MySQL／Redis／Kafka 以容器方式拉起来跑集成测试，解决「本地过了 CI 挂」「mock 与真实行为不一致」两类问题。

**版本与选型差异**

- 接口文档：Boot 3／Spring 6（jakarta 命名空间）走 springdoc-openapi；老的 springfox 不兼容，别在新项目里选它。
- Boot 3.x 基线 JDK 17；当前官方文档已把 devtools 内嵌的 LiveReload 标为 deprecated（Boot 4.1.0 起），新项目别再依赖它刷新浏览器。
- JDK 自带诊断工具（jcmd、JFR、jstack）不需要额外装 agent，面试里能说出这一条比只报第三方工具更稳。

**安全与生产边界（最容易被追问）**

- Actuator 端点默认收敛，只开需要的；`env`／`beans`／`heapdump` 这类不要暴露到公网，走管理端口＋鉴权。
- devtools 不参与生产运行，也不要把依赖传递给下游模块（Maven 标 `optional`，Gradle 用 `developmentOnly`）。
- Arthas 这类在线诊断工具在生产上是「读」为主，但要走审批和录屏，别在高峰期随便 `redefine` 热替换线上类。

**容易被挑刺的说法**

- 别把「用过的工具」说成「懂的原理」：报 Lombok 就会被问编译期注解处理，报 Arthas 就会被问它怎么做字节码增强，报不动的就不要列。
- 清单不要念成流水账：挑 2～3 个能讲清「为什么用它、替我解决了什么」的展开，其余一句带过。

#### 深挖追问

1. **Actuator 和 Micrometer 是什么关系？**（补充练习）

   Actuator 负责暴露端点与健康检查，Micrometer 负责以统一 API 采集指标、再对接 Prometheus／Datadog 等后端；两者常配合出现，但不是同一个东西。

2. **项目里怎么定位一次线上 CPU 飙高？**（补充练习）

   通用链路是：`top` 找进程 → `top -Hp` 找线程 → 线程号转 16 进制 → `jstack` 或 Arthas `thread` 定位栈；再结合 GC 日志与最近发布判断是死循环、频繁 Full GC 还是正则回溯。

3. **为什么不用 IDE 直接跑，要用 Maven 插件打包？**（补充练习）

   CI 与本地行为要一致；`spring-boot-maven-plugin` 的 repackage 产出含内嵌容器与启动类的可执行 jar，`java -jar` 即可运行，也是容器镜像里最常见的入口。

**面经来源**

- [[面经/阳光电源/一面/0001#Q07：基于 Spring Boot 开发的过程中，会用到哪些工具？|MJ065 · 阳光电源 · 一面 · Q07]]

**参考资料**（查证日期：2026-09-25）

- [Spring Boot 文档：Developer Tools（自动重启、属性默认值、生产禁用）](https://docs.spring.io/spring-boot/reference/using/devtools.html)
- [Spring Boot 文档：Actuator Endpoints（默认仅暴露 health 与 exposure 配置）](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html)
- [Spring Boot 文档：Running your application（打包运行与构建插件）](https://docs.spring.io/spring-boot/reference/using/running-your-application.html)
- [springdoc 官方站点（OpenAPI 3 与 Spring Boot 集成）](https://springdoc.org/)
- [Apache Maven Dependency Plugin：tree 目标](https://maven.apache.org/plugins/maven-dependency-plugin/tree-mojo.html)
- [MyBatis Generator 文档](https://mybatis.org/generator/)
- [Alibaba Arthas 项目主页](https://github.com/alibaba/arthas)
- [Apache JMeter 官网](https://jmeter.apache.org/)

### SPRING-014：Spring Session 中存储的是什么数据？

**常见问法**

- [[面经/收钱吧/一面/0001#Q07：Spring Session 中存储的是什么数据？|MJ068 · 收钱吧 · 一面 · Q07]]
- 用 Spring Session 后，Redis 里那个 session key 存的是什么？
- 为什么考虑在用户登录的时候用 Spring Session 去整合登录状态？它是怎么执行的？

#### 面试回答

结论：存的是服务端会话状态——Session 元数据＋attribute 键值对；客户端 Cookie 里只有会话 ID，对象本体都在共享存储。

- 元数据：session id、creationTime、lastAccessedTime、maxInactiveInterval（空闲超时秒数）、principalName（配合 Spring Security 存的登录用户标识，用于按用户查会话）。
- attribute：`session.setAttribute` 写入的业务对象——典型是登录用户信息、【购物车／向导步骤】等跨请求状态。
- 定位机制：`SessionRepositoryFilter` 包住请求，从 Cookie（默认名 `SESSION`）／Header／URL 取 id，去 `SessionRepository` 换回 Session，响应完成时写回。
- 用 Redis 存储时 attribute 要序列化（Jackson／JDK），所以属性必须可序列化、体积要控制——Session 是每请求热点，不是数据仓库。

#### 技术细节

**Redis 里的形态**

- `RedisIndexedSessionRepository`：一个 session 主 hash（`spring:session:sessions:{id}`，存元数据与 attributeEntries）＋ 几个索引 set／key（`sessions:expires`、`expirations`），支撑按 principal 查询与会话失效事件。
- 过期处理：依赖 Redis key 过期通知＋定期清理双路；maxInactiveInterval 换算成 TTL，续期靠最后访问时间刷新。

**与容器 Session 的关系**

- Spring Session 替换的是 `HttpSession` 的存储与生命周期管理，Tomcat 内存那份不再使用；因此多实例天然共享登录态、单实例重启不丢会话。
- Cookie 属性（HttpOnly／Secure／SameSite）仍要按安全要求配——存储搬了，凭证暴露面没变（Cookie／Session／Token 区分见 NET-012）。

**放什么不该放什么**

- 适合：小的、稳定的、每请求都要的标识类数据（用户 ID、权限快照版本号）。
- 不适合：大对象（完整用户档案、列表缓存）——序列化开销＋网络传输让每个请求都变慢；放共享缓存按 key 取。

#### 深挖追问

1. **为什么登录态迁移到 Spring Session 后要做序列化改造？**（面经实际出现的延伸；[[面经/收钱吧/一面/0001#Q08：序列化的作用、二进制与 JSON 序列化的差异、发生在网络的哪一层|MJ068 · 收钱吧 · 一面 · Q08]]）

   容器 Session 同进程放对象引用就行；外置存储必须把对象编码成字节。类结构变更（加字段、改类名）会造成旧 Session 反序列化失败、用户被登出，所以 attribute 类型要有版本兼容策略或放扁平的 Map／DTO。

2. **principalName 有什么用？**（补充练习）

   Spring Security 把认证信息放进 Session 时，Spring Session 同步记录 principalName，支持“查某用户的全部会话”——改密后强制下线、单点踢人这类需求靠它。

3. **为什么登录态选 Spring Session，而不是 JWT 或容器会话复制？**（面经实际出现；[[面经/京东JDS/一面/0001#Q05：为什么用 Spring Session 整合用户登录状态？它是怎么执行的？|MJ088 · 京东 JDS · 一面 · Q05]]）

   按“要不要服务端可控”和“改造成本”两条线选：

   - 对比容器 Session：容器那份在单进程内存里，多实例要么粘滞要么复制（Tomcat 集群复制开销大且不同步所有属性），重启会丢会话；Spring Session 存 Redis 后各实例共享、重启不踢人，代价是每请求一次网络与序列化。
   - 对比 JWT：JWT 自包含、验证不打服务端，但签发出去就收不回——改密踢人、封号、权限变更都要额外维护黑名单或缩短有效期，撤销成本反而更高。会话＋共享存储天然支持强制下线（配合 principalName 查该用户全部会话）。
   - 选型口径：需要即时撤销、服务端可控登录态（后台系统、支付类）→ Session ＋ Spring Session；跨域无 Cookie、纯接口且可容忍有效期内不可撤销 → JWT／Token（见 WEBSEC-004）。
   - 生产上常见组合是两层：短期 access token ＋ 服务端可撤销的 refresh／会话记录，本质仍是把“可撤销状态”放回服务端。


**面经来源**

- [[面经/收钱吧/一面/0001#Q07：Spring Session 中存储的是什么数据？|MJ068 · 收钱吧 · 一面 · Q07]]
- [[面经/京东JDS/一面/0001#Q05：为什么用 Spring Session 整合用户登录状态？它是怎么执行的？|MJ088 · 京东 JDS · 一面 · Q05]]

**参考资料**（查证日期：2026-09-25；适用 Spring Session 文档）

- [Spring Session Reference（SessionRepository、Redis 存储、Cookie/Header、过期机制）](https://docs.spring.io/spring-session/reference/)

**相关题目**

[[专题题库/计算机网络#NET-012：Cookie、Session 和 Token 有什么区别？|NET-012：会话三件套]]、[[专题题库/Java基础#JAVA-020：二进制序列化和 JSON 序列化有什么差异？各自适配什么场景？|JAVA-020：序列化选型]]

### SPRING-015：如何在 Spring 中实现一个简单的事务？

**常见问法**

- [[面经/淘天/一面/0002#Q21：如何在 Spring 中实现一个简单的事务？|MJ071 · 淘天 · 一面 · Q21]]
- 除了 @Transactional，还能怎么控制事务？

#### 面试回答

结论：声明式最省事——方法加 `@Transactional`，边界由代理管理；要精确控制就用编程式 `TransactionTemplate`，边界写在代码里看得见。

- 声明式三件配套：`@EnableTransactionManagement`（Boot 自动装配已带）、数据源对应的事务管理器（`DataSourceTransactionManager`）、方法注解；MyBatis／JPA 场景事务管理器由 starter 配好。
- 关键属性：`propagation`（默认 REQUIRED）、`isolation`、`timeout`、`readOnly`、`rollbackFor`——受检异常默认不回滚，接口声明了 checked 异常时必须显式指定。
- 编程式：注入 `PlatformTransactionManager` 构造 `TransactionTemplate`，`execute(status -> {...})` 里抛 RuntimeException 自动回滚，或 `status.setRollbackOnly()` 主动标记。
- 收尾一句：两种方式底层是同一套抽象（事务定义＋事务管理器＋具体管理器），`@Transactional` 只是把调用模板藏进了拦截器。

#### 技术细节

**最小可运行清单**

- 依赖 `spring-tx`＋数据源。
- 配置类上 `@EnableTransactionManagement`。
- 需要时 `proxyTargetClass=true` 切 CGLIB。
- 生效前提是调用经过代理：同类自调用、`final`／`private` 方法、异常被 catch 吞掉都会“看起来加了没生效”（SPRING-004 全清单）。

**Test 类这种场景怎么判**

- JUnit 里 `new` 出来的对象、或 `this.b()` 自调用，都没有代理——b 上的注解不生效；a 加不加注解都不改变这一点。
- 正确做法：把被测方法放进 `@Service` Bean，测试里 `@Autowired` 注入后调用。
- 另有一套东西要分清：测试方法加 Spring Test 的 `@Transactional`，是整个测试跑在托管事务里、默认结束回滚——这是测试回滚特性，不是业务代理生效的证据。

**TransactionTemplate 的适用信号**

- 事务段要刻意收窄：把 RPC／发 MQ／缓存写挪出 `execute` 块，事务时长即可测——JUC-023 的库存分桶就是这种写法。
- 传播行为想临时改：新建 template 实例设 `PROPAGATION_REQUIRES_NEW`，比在业务方法上叠注解清晰。
- 手动回滚标记不抛异常、调用方无感——适合“失败记日志、外层决定是否补偿”的流程。

**底层三件套（被追问“原理”时给这个）**

- `TransactionDefinition`（属性集合）→ `PlatformTransactionManager`（getTransaction／commit／rollback 的统一接口）→ 具体管理器绑定资源（JDBC 连接、JPA EntityManager）。
- 代理入口是 `TransactionInterceptor`：开启事务→执行方法→按异常决定提交／回滚→清理与同步。

#### 深挖追问

1. **一个方法里想“部分回滚”怎么做？**（补充练习）

   拆成两个事务：内层 `REQUIRES_NEW` 独立提交（日志／记账必留），外层失败不影响内层；或用 TransactionTemplate 分两段手动控制——不存在“单个事务回滚一半”。

2. **readOnly=true 有什么用？**（补充练习）

   它是给持久层与数据库的提示（Hibernate flush 模式、连接设为只读），能把优化交给引擎并挡住误写；它本身不提供任何隔离或锁语义，别当并发保护说。

**面经来源**

- [[面经/淘天/一面/0002#Q21：如何在 Spring 中实现一个简单的事务？|MJ071 · 淘天 · 一面 · Q21]]

**参考资料**（查证日期：2026-09-25；适用 Spring Framework 6.x 文档口径）

- [Spring Framework：Transaction Management（声明式与编程式、属性清单）](https://docs.spring.io/spring-framework/reference/data-binding/transaction.html)
- [Spring Framework：@Transactional 的代理机制与自调用限制](https://docs.spring.io/spring-framework/reference/data-binding/transaction/annotation-driven.html)

**相关题目**

[[#SPRING-004：@Transactional 何时不生效，如何正确调用事务方法？|SPRING-004：失效场景与修法]]、[[#SPRING-003：AOP 的原理是什么，为什么自调用可能失效？|SPRING-003：代理机制]]

### SPRING-016：Spring 用哪些类和注解支持 WebSocket？可靠消息需要业务层自己确认吗？

**常见问法**

- [[面经/小米/一面/0001#Q04：WebSocket 在 Spring 框架中涉及哪些类或注解；客户端与服务端通信需要在业务侧写代码做消息确认吗？|MJ078 · 小米 · 一面 · Q04]]
- 你们项目的长连接是怎么在 Spring 里写出来的？
- 项目做集群部署，各节点都要基于 WebSocket 做协同处理，你会怎么部署？

#### 面试回答

结论：Spring 提供两条路线——原生 WebSocket（`@EnableWebSocket`＋`WebSocketHandler`）适合自定义协议消息；STOMP 消息代理（`@MessageMapping`／`@SendTo`）适合“订阅目的地”式的即时通讯。要不要业务侧确认按“消息丢了会怎样”判断：通知类可以不做，涉钱涉指令必须做。

核心类与注解速报：

- 原生线：
  - `@EnableWebSocket` 开启。
  - 实现 `WebSocketHandler`（常用继承 `TextWebSocketHandler`）写 `afterConnectionEstablished`／`handleTextMessage`。
  - `HandshakeInterceptor` 在 HTTP 升级前鉴权。
  - `WebSocketConfigurer#registerWebSocketHandlers` 注册端点路径与 `setAllowedOrigins`。
- STOMP 线：
  - `@EnableWebSocketMessageBroker`。
  - `configureMessageBroker` 设应用前缀与代理前缀。
  - `@Controller`＋`@MessageMapping` 收、`@SendTo`／`SimpMessagingTemplate#convertAndSendToUser` 发。
  - `@SubscribeMapping` 订阅即答。
- 会话管理：`WebSocketSession` 非线程安全，发送要加锁；userId→session 映射自己维护（多实例要放 Redis 等共享存储并用广播转发）。

#### 技术细节

**两条线怎么选**

- 自定义帧格式、点对点推送、和已有二进制协议打通——原生线。
- 前端用 SockJS／STOMP 客户端、要“房间／用户”订阅模型、想复用消息代理（SimpleBroker 或外部 RabbitMQ STOMP）——消息线。
- 两条线底层都是 `spring-websocket` 模块，握手都走 HTTP Upgrade（原理见 [[专题题库/计算机网络#NET-022：WebSocket 的底层原理是什么？连接是怎么建立的？|NET-022]]）。

**“要不要消息确认”分三层说**

- 传输层：TCP 保证已交付字节的可靠与有序；这层不用写代码。
- 协议层：WebSocket 有 ping／pong 探活与 close 帧，但没有逐条消息的应用层 ACK；连接活着≠对端业务处理成功。
- 业务层：会“丢”的两种场景要确认——服务端推送后客户端崩溃没消费；客户端发送时连接刚断，写进socket 的数据随连接作废。
- 落地形态：服务端回执消息（带原消息 ID＋状态）、客户端超时重发、双方按消息 ID 幂等去重；或者更常用的“序号＋重连补拉”——重连后客户端带最后已确认序号，服务端从存储补发缺口。
- 结论句：确认机制是业务可靠投递设计，不是 WebSocket 的义务；即时展示类（弹幕、在线状态）直说“不做确认，丢一条无所谓”。

**容易说过头的地方**

- 别说“WebSocket 全双工所以天然可靠”——全双工说的是通信方向，不是投递语义。
- `@MessageMapping` 收不到“客户端没回 ACK”这件事——代理只保证把帧路由进方法。
- 多实例部署下内存 Map 存 session 会推送丢失：session 不可跨进程迁移，必须共享路由表＋节点间广播。

#### 深挖追问

1. **心跳和断线重连谁负责？**（补充练习；[[面经/收钱吧/二面/0001#Q05：WebSocket 的底层原理与连接建立过程|MJ069 · 收钱吧 · 二面 · Q05]] 的延伸）

   两端都要：服务端定时 ping（或业务空包）探测半开连接，中间层（Nginx）默认 60 秒空闲会掐连接，心跳间隔要小于它；重连由客户端负责退避重试＋恢复后重建订阅与补拉数据。

2. **Spring 的 WebSocket 和 Socket 是什么关系？**（补充练习）

   WebSocket 是跑在 TCP 之上的应用层消息协议，与 Socket（API）不同层，见 [[专题题库/计算机网络#NET-021：WebSocket 和 Socket 分别属于哪一层，是什么关系？|NET-021]]。

**面经来源**

- [[面经/小米/一面/0001#Q04：WebSocket 在 Spring 框架中涉及哪些类或注解；客户端与服务端通信需要在业务侧写代码做消息确认吗？|MJ078 · 小米 · 一面 · Q04]]
- [[面经/用友/一面/0002#Q04：项目做集群部署，各节点都要基于 WebSocket 做协同处理，怎么部署、会遇到哪些问题、怎么解决？|MJ090 · 用友 · 一面 · Q04]]

**参考资料**（本次查证：2026-09-25；适用 Spring Framework 6.x 文档口径）

- [Spring Framework Reference：WebSocket 服务端（原生与消息代理两条线）](https://docs.spring.io/spring-framework/reference/web/websocket.html)
- [Spring Framework Reference：WebSocket 消息（STOMP）](https://docs.spring.io/spring-framework/reference/web/websocket/stomp.html)

**相关题目**

[[专题题库/计算机网络#NET-022：WebSocket 的底层原理是什么？连接是怎么建立的？|NET-022：握手与帧]]、[[专题题库/计算机网络#NET-016：SSE 和 WebSocket 有什么区别，如何选择？|NET-016：选型对比]]

### SPRING-017：Spring MVC 处理一个请求会经过哪些组件？

**常见问法**

- [[面经/小米/一面/0001#Q10：Spring 处理一个请求会经过哪些模块？|MJ078 · 小米 · 一面 · Q10]]
- 一个 HTTP 请求在 Spring 里是怎么被处理的？

#### 面试回答

结论：主干是 `DispatcherServlet.doDispatch` 串起“HandlerMapping 找处理器 → HandlerAdapter 调 Controller → 返回值经消息转换器或视图解析器出响应”，过滤器、拦截器、异常解析器横切在前后。

按时序报模块：

1. Servlet 容器收下请求，先过 Filter 链（跨域、鉴权、编码）。
2. 进入前端控制器 `DispatcherServlet`。
3. `HandlerMapping`：按 URL（与请求条件）定位 Handler，返回“处理器＋拦截器链”的执行链。
4. `HandlerInterceptor#preHandle`：任一返回 false 就中断。
5. `HandlerAdapter`：适配并反射调用 Handler 方法；参数由 `HandlerMethodArgumentResolver` 逐个解析，`@RequestBody` 走 `HttpMessageConverter` 反序列化。
6. 返回值处理：`@ResponseBody` 经 `HttpMessageConverter`（Jackson 等）序列化直接写响应；页面请求返回 ModelAndView，交给 `ViewResolver` 解析视图并渲染。
7. 异常路径：处理过程抛错走 `HandlerExceptionResolver`（含 `@ExceptionHandler`／`@ControllerAdvice`）。
8. `postHandle` → 视图渲染完成后 `afterCompletion`；响应写回容器。

#### 技术细节

**为什么要有 HandlerAdapter 这层**

- 早期 SpringMVC 允许 Controller 实现不同类型接口（`Controller` 接口、HttpRequestHandler 等），适配器统一“怎么调用”；今天注解 POJO 占绝对主流，但结构保留。
- `@RequestMapping` 的注册本质：启动期把方法解析成 `RequestMappingInfo`→HandlerMethod 映射存进 `RequestMappingHandlerMapping`。

**每步的扩展点（追问“想插入自己的逻辑怎么办”）**

- 请求进出：Filter（容器级）、Interceptor（MVC 级）、`RequestBodyAdvice`／`ResponseBodyAdvice`（_body 加工）、`HandlerMethodArgumentResolver` 自定义参数。
- 异常：`@ControllerAdvice＋@ExceptionHandler` 全局兜底。
- 异步：返回 `Callable`／`DeferredResult` 时 `DispatcherServlet` 释放容器线程，靠 `AsyncContext` 完成时再派发——长任务转异步是这层支持的。

**容易混淆的边界**

- Tomcat 收连接、走连接器与线程池发生在进入 Spring 之前；这题问的是“进入 Spring 之后”。
- Gateway／安全过滤器链（Spring Security 的 `FilterChainProxy`）都挂在 Filter 阶段，不是 MVC 自己的组件。
- Spring Boot 的嵌入式容器默认把 `DispatcherServlet` 注册到 `/`；接口 404 而 actuator 正常，多半是映射没进 `HandlerMapping`。

#### 深挖追问

1. **拦截器和过滤器怎么选？**（补充练习）

   Filter 是 Servlet 规范、拿到的是裸 request／response，适合跨域、日志、鉴权这类框架外关注点。

   Interceptor 在 MVC 体系内，能拿到 HandlerMethod（知道要执行哪个 Controller 方法），适合登录态注入、接口级权限与耗时统计。需要 Spring 上下文依赖注入时只能 Interceptor（或用容器级 Filter 代理）。

2. **自研框架／RPC 场景下被问“你的 DispatcherServlet 链路怎么监控”？**（补充练习）

   在 Interceptor 或 `RequestBodyAdvice` 打点阶段耗时，配合 APM 的 servlet 插桩看入口 RT；慢接口按 Q10 的分层法继续下钻。

**面经来源**

- [[面经/小米/一面/0001#Q10：Spring 处理一个请求会经过哪些模块？|MJ078 · 小米 · 一面 · Q10]]

**参考资料**（本次查证：2026-09-25；适用 Spring Framework 6.x 文档口径）

- [Spring Framework Reference：Spring MVC 的 DispatcherServlet 与请求处理](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet.html)
- [Spring Framework Reference：MVC 配置与拦截器](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-config/interceptors.html)

**相关题目**

[[#SPRING-009：Spring 常用注解有哪些，分别由哪个扩展点处理？|SPRING-009：注解与处理扩展点]]、[[#SPRING-003：AOP 的原理是什么，为什么自调用可能失效？|SPRING-003：AOP 代理]]
