# Spring

- 题号前缀：SPRING
- 范围：Spring Boot 自动装配的原理是什么？；IoC 是什么，容器如何创建和管理 Bean？；AOP 的原理是什么，为什么自调用可能失效？；@Transactional 何时不生效，如何正确调用事务方法？；@Resource 与 @Autowired 有什么区别，如何按名称注入？；Spring 如何处理循环依赖，如何解决？；什么是懒加载，@Lazy 在哪里生效？；@Transactional 方法里新开线程是否还在同一事务；常用注解及其处理者；@Autowired 的注入流程与反射实现；Bean 的完整生命周期。
- 最近更新：2026-09-24
- 说明：按本库面经整理；补充练习不计入原始面试问题。个人经历答案为框架，技术版本以题内说明为准。

## 目录

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

### SPRING-001：Spring Boot 自动装配的原理是什么？

**常见问法**

- Spring Boot 自动装配的原理是什么？

#### 面试回答

`@SpringBootApplication` 中的 `@EnableAutoConfiguration` 会导入自动配置候选类。Spring Boot 3 主要从 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 读取候选项，再通过 `@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty` 等条件决定哪些配置生效。生效的自动配置本质上仍是向 IoC 容器注册 Bean；开发者自己定义了同类 Bean 时，很多默认配置会自动退让。

#### 技术细节

现代 Boot 候选列表使用 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports；Boot 2.7 引入该方式，更老的常见实现通过 spring.factories 注册 EnableAutoConfiguration 候选，不能把旧方式套到所有版本。选择器导入候选，@ConditionalOnClass、@ConditionalOnMissingBean 等条件控制注册。组件扫描查找业务组件，自动配置导入框架配置，二者机制不同；starter 常用于聚合依赖，不等于自动配置类本身。

#### 深挖追问

1. **为什么自定义 Bean 后默认 Bean 不生效？**（补充练习）

   常见原因是 @ConditionalOnMissingBean 条件不再满足，可通过条件报告确认，而非认为覆盖总是按加载顺序发生。

2. **自动配置不生效怎么排查？**（补充练习）

   检查依赖和候选注册、属性开关、排除项以及条件报告，再核对当前版本的配置方式。

**面经来源**

- [[面经/用友/一面/0001#Q07：Spring Boot 自动装配的原理是什么？|MJ001 · 用友 · 一面 · Q07]]

**参考资料**（本次查证：2026-09-19）

- [Spring Boot 自动配置](https://docs.spring.io/spring-boot/reference/using/auto-configuration.html)
- [Spring Boot 自定义自动配置](https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html)

### SPRING-002：IoC 是什么，容器如何创建和管理 Bean？

**常见问法**

- IoC 是什么，具体原理是什么？

#### 面试回答

IoC 是把对象创建和依赖管理交给容器，DI 是实现依赖装配的方式。Spring 先读取配置形成 BeanDefinition，再在需要时实例化对象、注入依赖，执行初始化与后处理，最后按作用域管理使用和销毁。业务类声明依赖即可，不必到处手动 new 并组装对象。

#### 技术细节

ApplicationContext 在 BeanFactory 基础上提供更完整的应用能力。要区分 BeanDefinition 阶段的扩展与 Bean 实例阶段的 BeanPostProcessor；后者可返回代理对象。构造器注入使依赖明确，但互相依赖的构造器无法靠提前暴露实例解决。某些单例属性注入循环在特定设置下可通过早期引用处理，不能推出所有循环依赖都能解决；更合理的处理通常是调整职责或依赖方向。

#### 深挖追问

1. **Bean 默认单例是否线程安全？**（补充练习）

   单例只说明容器中的实例数量；共享可变状态仍需要同步或无状态设计。

2. **IoC 和 AOP 怎么关联？**（补充练习）

   容器后处理器可在 Bean 创建过程中返回代理，使调用进入切面逻辑。

**面经来源**

- [[面经/用友/一面/0001#Q08：IoC 和 AOP 是什么，具体原理是什么？|MJ001 · 用友 · 一面 · Q08]]

**参考资料**（本次查证：2026-09-12）

- [Spring IoC 容器](https://docs.spring.io/spring-framework/reference/core/beans/introduction.html)

### SPRING-003：AOP 的原理是什么，为什么自调用可能失效？

**常见问法**

- AOP 是什么，具体原理是什么？

- 动态代理有哪几种？

- AOP 是什么，底层原理是什么？

- 公共字段填充、限流这类功能为什么要引入 AOP 和注解？

#### 面试回答

Spring AOP 通常通过代理在方法调用前后织入事务、日志等逻辑，代理可采用 JDK 动态代理或 CGLIB 子类方式。调用经过代理才会进入拦截器链；同一个对象内部用 this 调用方法通常绕过代理，因此相关切面可能不生效。具体代理默认值要看 Spring 与 Boot 配置。

#### 技术细节

切点决定匹配哪些方法，通知定义何时执行，代理把这些通知组成调用链。接口代理和子类代理的能力边界不同；CGLIB 不能通过覆盖 final、private 方法完成增强。声明式事务还受异常回滚规则、可见性和调用入口影响，不能只看是否存在 @Transactional。优先将需要被拦截的行为拆到独立 Bean；不要为解决自调用随意在业务中获取当前代理。AspectJ 编织与基于代理的 Spring AOP 也不同。

补充：从实现机制回答“动态代理有哪几种”时，JDK Proxy 以接口为代理契约，调用转交 InvocationHandler；CGLIB 属于运行时生成子类的方式。Byte Buddy 等也能构造代理，不应将“两种常见机制”说成全 Java 只有两个方案。静态代理是手工写代理类，AspectJ 编织则不是同一种代理调用路径。代理对象和目标对象的身份、可见方法与 this 调用路径决定增强是否发生。

#### 深挖追问

1. **为什么加了事务注解还不回滚？**（补充练习）

   排查是否经过代理、是否抛出符合规则的异常、异常是否被吞掉以及传播设置，不能只归因于数据库。

2. **JDK 代理一定比 CGLIB 快吗？**（补充练习）

   不能脱离版本与调用场景下结论；选型先满足接口和代理能力需求，再针对实际负载测量。

3. **公共字段（创建时间、更新人）为什么要用 AOP 填充，怎么实现？**（面经实际出现；[[面经/京东健康/二面/0001#Q02：公共字段的填充为什么要引入 AOP，怎么实现的，有什么作用？|MJ018 · 京东健康 · 二面 · Q02]]）

   这类字段每个写操作都要填，散写易漏且口径不一。切面拦截写方法或带标记注解的入口，环绕通知从登录上下文与系统时间取值填入实体；MyBatis 场景也可用 MyBatis-Plus MetaObjectHandler／Executor 拦截器在 SQL 执行前统一填充。作用是横切逻辑集中、业务无感、字段口径统一；注意同类自调用不走代理会失效，异步线程里上下文（当前用户）需显式传递。

**面经来源**

- [[面经/京东健康/二面/0001#Q02：公共字段的填充为什么要引入 AOP，怎么实现的，有什么作用？|MJ018 · 京东健康 · 二面 · Q02]]
- [[面经/京东健康/一面/0001#Q10：限流部分怎么实现的，为什么要用 AOP 和注解，有哪些作用？|MJ018 · 京东健康 · 一面 · Q10]]

- [[面经/百度/一面/0006#Q15：AOP 是什么，底层原理是什么？|MJ010 · 百度 · 一面 · Q15]]

- [[面经/北京某上市公司/一面/0001#Q09：动态代理有哪几种？|MJ004 · 北京某上市公司 · 一面 · Q09]]
- [[面经/用友/一面/0001#Q08：IoC 和 AOP 是什么，具体原理是什么？|MJ001 · 用友 · 一面 · Q08]]

**参考资料**（本次查证：2026-09-12）

- [Spring AOP 代理机制](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)

### SPRING-004：@Transactional 何时不生效，如何正确调用事务方法？

**常见问法**

- @Transactional 注解什么时候失效？
- 如何在非方法内调用另一个事物方法？

#### 面试回答

Spring 声明式事务默认靠 AOP 代理拦截方法调用。常见失效情况有：对象是手动 `new` 的、同类中用 `this` 调用事务方法、代理无法拦截目标方法、异常被捕获后没再抛出，或异常类型不符合回滚规则。比较稳妥的做法是把事务方法放在独立的 Spring Bean 中，通过注入的代理对象调用；需要在同类中明确控制事务边界时，可以使用 `TransactionTemplate`。

#### 技术细节

注解是元数据，需要事务基础设施与正确事务管理器。默认配置下 RuntimeException、Error 通常触发回滚，受检异常通常不触发，可通过 rollbackFor 等调整；Spring 6.2+ 还允许全局改变默认回滚策略，应看配置。类代理的 private 方法不能通过覆盖增强；Spring 6.0 起 protected／包可见方法在类代理下默认可支持，接口代理入口仍需 public 接口方法，不能套用“非 public 全失效”。

常见 JDBC 事务绑定当前线程，不自动传播到新线程。外层普通方法调用另一个 Bean 的代理事务方法，可开启事务；同类直接调用则不会应用被调方法自己的注解。若外层已经有事务，内部数据库操作仍可能参与外层事务，这与内部注解被拦截是两回事。可优先拆独立服务，必要时用 TransactionTemplate 明确代码块边界。

原文“如何在非方法内调用另一个事物方法”语义不完整：若指非事务方法，按以上跨 Bean／自调用区别回答；若真指构造器或初始化阶段，应将事务工作放到代理就绪后的入口，不依赖初始化中的代理事务。

#### 深挖追问

1. **异常被 catch 后为什么不回滚？**（补充练习）

   如果异常未传播到事务拦截器且未标记 rollback-only，拦截器可能看到正常返回；按业务规则重新抛出或显式标记，不能机械 catch 所有异常。

2. **调用另一个事务方法一定新开事务吗？**（补充练习）

   不一定，REQUIRED 常加入已有事务，REQUIRES_NEW 才按配置挂起外层并新建；前提是调用经过代理。

**面经来源**

- [[面经/北京某上市公司/一面/0001#Q08：@Transactional 何时失效，如何从其他入口调用事务方法？|MJ004 · 北京某上市公司 · 一面 · Q08]]
- [[面经/京东零售/一面/0001#Q10：如何使用 Spring 控制事务？|MJ034 · 京东零售 · 一面 · Q10]]

**参考资料**（本次查证：2026-09-19）

- [Spring @Transactional 使用规则](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html)
- [Spring 声明式事务回滚规则](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html)

### SPRING-005：@Resource 与 @Autowired 有什么区别，如何按名称注入？

**常见问法**

- @Resource 和 @Autowired 有什么区别？
- 如何使用 @Autowired 按名称注入？

#### 面试回答

@Autowired 属于 Spring，主要按类型解析依赖，可结合 @Qualifier 限定候选；@Resource 属于 Jakarta 注解体系，在 Spring 中可用 name 明确按 Bean 名称注入，未指定时通常先用字段或属性名再按适用规则回退。若用 @Autowired 指定命名 Bean，常写 @Autowired 配 @Qualifier("beanName")，仍要求类型匹配。

#### 技术细节

现代 Spring 使用 jakarta.annotation.Resource，旧项目可能是 javax.annotation.Resource。@Autowired 可用于构造器、字段和方法，单构造器场景通常可省略；@Resource 通常用于字段和 setter，不作为构造器参数注解。@Qualifier 的核心是缩小按类型候选集，Bean 名可作为回退匹配值，不是忽略类型直接取任意对象。字段或参数同名回退还受候选优先级和参数名元数据影响，显式 qualifier 更易读。

```java
// 片段：假设容器中有名为 orderService 的 OrderService Bean
@Autowired
@Qualifier("orderService")
private OrderService service;
```

也可用 `@Resource(name = "orderService")` 表达明确的名称依赖。

#### 深挖追问

1. **有两个同类型 Bean，@Primary 与 @Qualifier 怎么选？**（补充练习）

   @Primary 提供默认优先候选，@Qualifier 在具体注入点限定用途；依赖特定角色时明确限定。

2. **@Autowired 能只按名称忽略类型吗？**（补充练习）

   不能，Qualifier 仍在类型兼容的候选中筛选；需按 Bean 身份表达时可使用 @Resource(name=...)。

**面经来源**

- [[面经/北京某上市公司/一面/0001#Q10：@Resource 与 @Autowired 有什么区别，如何按名称注入？|MJ004 · 北京某上市公司 · 一面 · Q10]]
- [[面经/小红书/一面/0001#Q03：@Autowired 的底层原理；实现时怎么通过反射获取成员变量、用哪个反射方法取 Field、又用哪个方法检查它是否被 @Autowired 标记？|MJ040 · 小红书 · 一面 · Q03]]

**参考资料**（本次查证：2026-09-12）

- [Spring @Autowired 与 Qualifier](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/autowired-qualifiers.html)
- [Spring @Resource 注入](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/resource.html)

### SPRING-006：Spring 如何处理循环依赖，如何解决？

**常见问法**

- 如何解决循环依赖？

#### 面试回答

循环依赖是 A 依赖 B、B 又依赖 A。首先通过拆分职责、抽公共服务或事件协作消除环；某些场景可用注入点 @Lazy 或 ObjectProvider 延后获取。容器在允许循环引用时，能用早期引用处理部分单例属性注入循环，但构造器循环、原型循环或复杂代理情形并非都能解决。

#### 技术细节

典型单例解析涉及完整单例、早期单例引用、创建早期引用的工厂三层缓存。A 先实例化再填属性，B 需要 A 时可获取 A 的早期引用；工厂可让后处理器提供合适代理，避免最终代理与早期对象不一致。构造器环在对象尚未实例化时就互等，不能套用这一流程。是否允许循环引用要看 Spring Boot 版本和配置，不把开关当作设计修复。注入点延迟代理可推迟依赖解析，但若构造或初始化中立即调用，又可能触发原来的环。

与 [[#SPRING-002：IoC 是什么，容器如何创建和管理 Bean？|IoC 生命周期]]、[[#SPRING-007：什么是懒加载，@Lazy 在哪里生效？|懒加载]] 联合复习。

#### 深挖追问

1. **三级缓存是为所有循环依赖准备的吗？**（补充练习）

   不是，主要解释特定单例提前暴露及代理协调；不能解决所有作用域与构造器循环。

2. **开允许循环引用就可以了吗？**（补充练习）

   只改变容器策略，仍需检查半初始化对象、代理一致性及业务职责，优先消除环。

**面经来源**

- [[面经/北京某上市公司/一面/0001#Q11：如何解决循环依赖？|MJ004 · 北京某上市公司 · 一面 · Q11]]
- [[面经/小红书/一面/0001#Q04：请讲一下 Spring 中一个 Bean 的生命周期。|MJ040 · 小红书 · 一面 · Q04]]

**参考资料**（本次查证：2026-09-12）

- [Spring 依赖注入与循环依赖](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html)

### SPRING-007：什么是懒加载，@Lazy 在哪里生效？

**常见问法**

- 懒加载是什么？

#### 面试回答

Spring 懒加载是把 Bean 创建从容器启动时推迟到第一次需要它时。Bean 定义上 @Lazy 控制初始化时机，注入点上 @Lazy 通常注入延迟解析代理。若一个非懒加载单例在启动时直接依赖该 Bean，即使被依赖 Bean 标了懒加载，也可能因必须满足依赖而被提前创建。

#### 技术细节

默认非懒加载单例通常随容器预实例化；懒加载可缩短启动路径，但把初始化开销和配置错误推迟到首次使用。注入点代理在实际调用时解析目标，与只给目标 Bean 标 @Lazy 不是同一个效果。ObjectProvider.getObject 可更显式地延迟获取。用于循环依赖时要看触发时点，不能在构造器里立即使用代理后又宣称依赖已经解开。这里按 Spring Bean 懒加载回答，不与 ORM 懒加载或网页资源懒加载混为一谈。

#### 深挖追问

1. **懒加载会让 Bean 变成多例吗？**（补充练习）

   不会，scope 与初始化时机是不同维度，懒加载单例创建后仍按单例管理。

2. **第一次调用慢怎么办？**（补充练习）

   把首次初始化延迟纳入响应预算，对确需预热的关键路径显式预热，或不使用懒加载。

**面经来源**

- [[面经/北京某上市公司/一面/0001#Q12：懒加载是什么？|MJ004 · 北京某上市公司 · 一面 · Q12]]

**参考资料**（本次查证：2026-09-12）

- [Spring 延迟初始化](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-lazy-init.html)
- [Spring @Lazy API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Lazy.html)

### SPRING-008：@Transactional 方法里新开线程执行，还在同一事务中吗？

**常见问法**

- 主方法有 `@Transactional`，内部依次调用 `serviceA.updateA()` 和 `serviceB.updateB()`；把 `serviceA.updateA()` 放到一个新线程执行，A 表和 B 表的更新还会在同一个事务中吗？

#### 面试回答

不会在同一个事务里。Spring 的声明式事务上下文绑定在**当前线程**上——`TransactionSynchronizationManager` 用 `ThreadLocal` 保存事务状态，数据库连接也是从线程绑定的资源里取的。主方法开事务后，`serviceB.updateB()` 在同一线程、复用它的事务连接；一旦把 `updateA()` 交给新线程，新线程的 `ThreadLocal` 是空的，拿不到那个连接，于是 `updateA()` 要么以自动提交独立执行、要么自开新事务，**不会**随主事务一起提交或回滚。结果是原子性被打破：主方法后续抛异常回滚时，`updateB` 回滚了，`updateA` 却已独立生效。

要处理得明确取舍：① 接受“两个独立事务”，为失败写补偿/对账（最终一致）；② 在新线程内用编程式事务自己管理边界并处理异常；③ 若只是想“事务提交后再异步做副作用”，用 `TransactionSynchronization.afterCommit` 或事务事件在提交后触发，而不是在事务中途甩给新线程。

#### 技术细节

传播行为（`REQUIRES_NEW`/`NESTED`）解决的是**同一调用线程内**、经过代理的事务嵌套，不跨线程传播——`@Transactional` 不会把 `ThreadLocal` 上下文带进子线程。同理，`@Async` 与 `@Transactional` 组合时，异步方法一定在新线程、必然脱离调用方事务。连接绑定靠 `TransactionSynchronizationManager` 维护的 `DataSource → Connection` 的 `ThreadLocal` 映射，这是“同一线程复用同一连接”的基础，也是“换线程即换（或丢）事务”的根因。

#### 深挖追问

1. **那怎样才能让新线程里的操作也纳入原事务？**（面经实际追问）

   实际上做不到“真正同一物理事务跨线程回滚”；能做的要么改回同线程（不用新线程），要么把它设计成独立事务＋失败补偿，或用事务提交后回调保证“只在成功后异步执行”。想并行执行又要一致，通常是业务层拆分＋对账，而不是指望 Spring 事务跟着线程走。
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

#### 面试回答

报名单没有区分度，按“用途分类＋各带一句机制”讲：容器与配置类（`@SpringBootApplication`＝`@EnableAutoConfiguration`＋`@Configuration`＋`@ComponentScan`；`@Configuration`／`@Bean`、`@Component` 及派生的 `@Service`／`@Repository`／`@Controller`、`@Import`、`@Conditional*`）由配置类解析与自动装配处理；注入类（`@Autowired`、`@Value`、`@Resource`）由 `BeanPostProcessor` 在属性填充阶段处理；Web 类（`@RestController`、`@RequestMapping` 及其派生、`@PathVariable`、`@RequestBody`、`@Valid`）由 `DispatcherServlet` 与处理器适配器／参数解析器处理；横切类（`@Transactional`、`@Aspect` 家族、`@Cacheable`、`@Async`）靠代理生效，因此有“自调用与私有方法失效”这类共同坑；生命周期与顺序类（`@PostConstruct`／`@PreDestroy`、`@Lazy`、`@Primary`、`@Order`）分别落在初始化回调、按需创建、候选仲裁与排序环节。收尾给一句判断标准：注解只是声明式外衣，要掌握的是它在哪个扩展点被谁处理——被问“为什么我的注解没生效”时，答案几乎都在“没走代理／没交给容器／时机不对”这三类里。

#### 技术细节

分类背后的处理者值得能点名，因为追问会往下钻一层：组件扫描把 `@Component` 系的定义注册成 `BeanDefinition`；自动装配靠 `@EnableAutoConfiguration` 导入 `AutoConfigurationImportSelector` 读取候选配置并受 `@Conditional*` 过滤（见 [[#SPRING-001：Spring Boot 自动装配的原理是什么？|SPRING-001：Spring Boot 自动装配的原理是什么？]]）；`@Autowired` 的注入与反射细节见 [[#SPRING-010：@Autowired 的注入流程与反射实现细节？|SPRING-010：@Autowired 的注入流程与反射实现细节]]；`@Transactional` 的传播行为、隔离级别与失效场景见 [[#SPRING-004：@Transactional 何时不生效，如何正确调用事务方法？|SPRING-004：@Transactional 何时不生效]]；AOP 代理与自调用见 [[#SPRING-003：AOP 的原理是什么，为什么自调用可能失效？|SPRING-003：AOP 的原理与自调用失效]]。

`@Value` 与 `@ConfigurationProperties` 的取舍是高频延伸：前者逐个绑定、支持 SpEL 但散落在字段上；后者批量绑定到类型安全的配置类、支持校验与 IDE 提示，成组配置优先后者。`@Primary` 与 `@Qualifier` 解决同类型多候选的仲裁（见 SPRING-005），`@Order`／`@Priority` 影响切面与列表注入顺序。`@PostConstruct`（JSR-305／jakarta 注解）在属性填充后执行，`InitializingBean#afterPropertiesSet` 与自定义 `init-method` 依次在其后——三者时机不要混着说。

答题时把范围收在自己用过的部分：与其列 30 个名词，不如挑 5—6 个能讲到处理者与失效场景的。项目里没用到的（如 `@Cacheable`、`@Async`）就说“了解机制但项目未使用”，被追问线程与事务的关系时能给出口径（见 [[#SPRING-008：@Transactional 方法里新开线程执行，还在同一事务中吗？|SPRING-008：事务与新线程]]）。

#### 深挖追问

1. **`@Component` 和 `@Bean` 有什么区别，什么时候必须用 `@Bean`？**（补充练习）

   `@Component` 由扫描在类上声明、构造与依赖交给容器推断；`@Bean` 写在配置类方法上，适合第三方类（无法改源码加注解）、需要构造参数或初始化逻辑、以及同一类型要多实例／条件创建的场景。

2. **`@Autowired` 能用在哪些位置，哪种推荐？**（补充练习）

   字段、构造器、setter 与方法参数都可；推荐构造器注入——依赖显式、可声明为 final、便于测试且不会绕过校验，缺点是循环依赖时没有字段注入的“提前暴露”余地（见 [[#SPRING-006：Spring 如何处理循环依赖，如何解决？|SPRING-006：循环依赖]]）。

3. **为什么 `@Transactional` 标在 `private` 方法上不生效？**（补充练习）

   代理只能拦截可被代理调用到的方法（Spring 的注解解析与 CGLIB／JDK 代理都要求方法可见并可覆写），`private`／`final` 方法或自调用都不走代理，事务增强因此不会应用。

**面经来源**

- [[面经/小红书/一面/0001#Q02：Spring 框架常用的注解有哪些？|MJ040 · 小红书 · 一面 · Q02]]

### SPRING-010：@Autowired 的注入流程与反射实现细节？

**常见问法**

- @Autowired 注解的底层原理是什么？
- 实现依赖注入时，是怎么通过反射获取成员变量的？具体用什么反射方法获取 Field？
- 获取到字段后，用什么方法检查它是否被 @Autowired 标记？
- 让你自己实现一个最小版的依赖注入，怎么写？

#### 面试回答

`@Autowired` 由 `AutowiredAnnotationBeanPostProcessor`（实现 `InstantiationAwareBeanPostProcessor`）在 Bean 实例化之后的属性填充阶段处理：`postProcessProperties` 取出该 Bean 的注入元数据，逐个元素把依赖解析出来并写回。解析时把字段包成 `DependencyDescriptor` 交给 `DefaultListableBeanFactory.doResolveDependency`，按类型查候选；多候选依次用 `@Qualifier`、`@Primary`、优先级、再回退到字段名匹配；`required=true` 且没有候选就抛 `NoSuchBeanDefinitionException`。构造器注入不在这一步，是实例化阶段由容器选定构造器时完成。

三个反射细节：① 拿成员变量用 `Class#getDeclaredFields()`——包含本类声明的私有字段但不含父类，父类字段靠沿继承链向上遍历，Spring 内部用 `ReflectionUtils.doWithFields` 做这个遍历；② 方法注入用 `getDeclaredMethods()`、构造器用 `getDeclaredConstructors()`，拿到的都是 `java.lang.reflect` 的元对象；③ 判断注解用 `field.isAnnotationPresent(Autowired.class)` 语义最直接，但 Spring 实际用 `AnnotationUtils.findAnnotation(field, Autowired.class)`，因为还要支持元注解（自己定义的注解里再标 `@Autowired`）与合成注解，`isAnnotationPresent` 只看直接标注。写私有字段前必须 `field.setAccessible(true)`（Spring 封装成 `ReflectionUtils.makeAccessible`），最后 `field.set(bean, value)` 完成注入。性能上不会每次启动后还在反射扫描：首次解析出的注入元素被缓存为 `InjectionMetadata`，之后只是遍历缓存。

自己实现最小版时，上述流程能直接照抄：容器启动后遍历每个 Bean 的 `getDeclaredFields()`，命中注解的字段按类型或名称从 Bean 定义表取实例、`setAccessible` 后写入；但要补三件事才谈得上可用——循环依赖（字段注入可提前暴露未完成对象解决，构造器注入无解）、多候选仲裁（`@Qualifier`／`@Primary`）、以及代理对象上的注解查找（`findAnnotation` 而不是 `isAnnotationPresent`）。

#### 技术细节

字段注入的代价常被追问：字段可以为 null 通过编译、脱离容器无法构造（测试要反射或用 `ReflectionTestUtils`）、依赖变多时不报错也不显形、循环依赖被“静默容忍”。构造器注入把这些问题换成显式失败与 final 字段，这也是 Spring 官方推荐构造器注入的原因。

反射调用的两个隐性成本：`setAccessible` 与 `getDeclaredFields` 都有开销，Spring 用缓存（`ReflectionUtils` 的声明字段缓存、`InjectionMetadata`）把重复反射降到最低；Java 9 模块系统下访问非导出包会额外受限，跨模块的类要 `--add-opens` 才能反射，这是“同样代码在 JDK 17 报 InaccessibleObjectException”的常见原因。注解元数据还有保留策略要求：`@Autowired` 是 `RUNTIME` 保留，否则运行时反射读不到——被问“自定义注解为什么读不到”，先看 `@Retention`。

与 `@Resource` 的分派差别（按名优先、由 `CommonAnnotationBeanPostProcessor` 处理）见 [[#SPRING-005：@Resource 与 @Autowired 有什么区别，如何按名称注入？|SPRING-005：@Resource 与 @Autowired 有什么区别]]；`@Value` 占位符解析发生在同一阶段的 `DependencyDescriptor` 解析里，SpEL 也在此求值。

#### 深挖追问

1. **`@Autowired` 和 `getBean()` 拿对象有区别吗？**（补充练习）

   同一个容器、同一份单例，差别在时机与表达：`@Autowired` 由容器在装配期注入、依赖是声明式的；`getBean()` 是主动拉取，容易把容器当服务定位器用而隐藏真实依赖，还会绕开部分后置处理时机。

2. **为什么 `@Autowired` 能注入到被 CGLIB 代理的类字段里？**（补充练习）

   注入发生在目标实例上、代理在其后（初始化后处理）生成；父类字段声明在目标类层次中，`findAnnotation`／`doWithFields` 会沿继承链查找，所以代理不会让注解“消失”。若注入的是接口类型的限定 Bean，真正决定注入结果的是候选解析与 `@Qualifier`。

3. **同类中多个 Bean 满足同一类型，容器怎么选？**（补充练习）

   依次看 `@Qualifier` 匹配、`@Primary` 优先、`@Priority` 顺序，最后用注入点名称匹配 Bean 名；仍不确定就抛 `NoUniqueBeanDefinitionException`——所以给同类多实例的字段起与 Bean 名一致的名字能“碰巧生效”，但不要依赖这一点（显式 `@Qualifier` 更稳）。

**面经来源**

- [[面经/小红书/一面/0001#Q03：@Autowired 的底层原理；实现时怎么通过反射获取成员变量、用哪个反射方法取 Field、又用哪个方法检查它是否被 @Autowired 标记？|MJ040 · 小红书 · 一面 · Q03]]

**参考资料**（查证：2026-09-23）

- [Spring Framework Reference：Annotation-based container configuration](https://docs.spring.io/spring-framework/reference/core/beans/annotation.html)
- [Java 17 `Class#getDeclaredFields`／`Field#setAccessible`](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/lang/reflect/Field.html)

### SPRING-011：Spring Bean 的完整生命周期是怎样的？

**常见问法**

- 请讲一下 Spring 中一个 Bean 的生命周期。
- 一个 Bean 从创建到销毁经历了哪些步骤？

#### 面试回答

以单例 Bean 为主线讲六段：① 实例化——读 `BeanDefinition`，推断或选定构造器创建原始实例（构造器注入在这一步完成）；② 属性填充——`BeanPostProcessor` 介入，`@Autowired`／`@Value` 在这里写入，循环依赖也在这一步靠提前暴露半成品对象解决；③ Aware 回调——容器把自身资源交给 Bean，如 `BeanNameAware`、`BeanFactoryAware`、`ApplicationContextAware`；④ 初始化前——`postProcessBeforeInitialization`，随后 `@PostConstruct`、`InitializingBean#afterPropertiesSet`、自定义 `init-method` 依次执行；⑤ 初始化后——`postProcessAfterInitialization`，AOP 代理通常在这一步生成并替换原始对象，所以“自调用绕过代理”类问题都源于此；⑥ 使用与销毁——容器关闭时按 `@PreDestroy`、`DisposableBean#destroy`、`destroy-method` 顺序回收。

再补三点显示深度：原型（prototype）Bean 容器只负责创建与装配、不执行销毁回调，客户端要自己管生命周期；代理可能在初始化后被替换，注入到其他 Bean 的应是最终对象，这正是三级缓存里要提前暴露“可能已代理”的引用（`getEarlyBeanReference`）的原因；最后是定位思路——实例化失败多在构造与依赖解析阶段，`BeanCurrentlyInCreationException` 是循环依赖，字段为 null 通常是这个对象被 `new` 出来而没交给容器。

#### 技术细节

三段式主干好记：创建（实例化＋装配）→ 初始化（回调与前后处理）→ 销毁（清理）。其中的关键区别常被追问：`BeanPostProcessor` 是“对所有 Bean 生效的横切扩展点”（AOP、`@Autowired` 都靠它），`BeanFactoryPostProcessor` 则在 Bean 定义阶段生效、拿到的是定义而非实例（自动装配的候选配置处理、属性占位符替换都发生在这里，见 [[#SPRING-001：Spring Boot 自动装配的原理是什么？|SPRING-001：Spring Boot 自动装配的原理是什么？]]）。

初始化三兄弟的顺序与语义：`@PostConstruct`（jakarta 注解，由 `InitDestroyAnnotationBeanPostProcessor` 触发，最先）、`afterPropertiesSet`（接口方法，容器强约束）、`init-method`（配置里指定的字符串方法名，最灵活也最易拼错）。业务代码统一用 `@PostConstruct` 即可，需要被 XML／配置覆盖时才用 `init-method`。销毁侧同理有 `@PreDestroy` 与 `destroy` 方法／`destroy-method`。

提前初始化与延迟初始化的边界：`@Lazy` 让 Bean 首次使用时才创建（见 [[#SPRING-007：什么是懒加载，@Lazy 在哪里生效？|SPRING-007：什么是懒加载]]），SmartInitializingSingleton／`ApplicationRunner` 之类则用于“全部单例就绪之后”的启动动作——放在 `@PostConstruct` 里做依赖其他 Bean 的启动逻辑是常见错误，因为此时邻居 Bean 可能还没装配完。循环依赖的完整机制与为什么构造器注入救不了见 [[#SPRING-006：Spring 如何处理循环依赖，如何解决？|SPRING-006：Spring 如何处理循环依赖]]。

#### 深挖追问

1. **为什么 AOP 代理不在实例化时就生成，而要放在初始化之后？**（补充练习）

   增强依赖 Bean 自身配置就绪（注解、属性、初始化回调结果），且 `postProcessAfterInitialization` 是容器约定的“最终对象”产出点；提前生成会让初始化回调作用在代理而非目标对象上，语义更难解释。

2. **三级缓存分别在什么时候被用到？**（补充练习）

   正常无环流程只走单例池（一级）与创建中标记；只有出现循环依赖、需要把未完成对象提前暴露时，才用到二级（早期引用）和三级（对象工厂 `ObjectFactory`，用于按需生成代理），这样既解环又保证代理只创建一次。

3. **Bean 生命周期问题怎么在真实项目里定位？**（补充练习）

   按“哪一段炸”反查：找不到符号或依赖 null→该对象未经容器创建；启动阶段循环依赖异常→按 SPRING-006 处理；初始化里读不到其他 Bean 状态→时机太早，改用 SmartInitializingSingleton／事件；关闭时报连接池已关→销毁顺序问题，需要显式控制依赖或提前 flush。

**面经来源**

- [[面经/小红书/一面/0001#Q04：请讲一下 Spring 中一个 Bean 的生命周期。|MJ040 · 小红书 · 一面 · Q04]]

**参考资料**（查证：2026-09-23）

- [Spring Framework Reference：Container lifecycle／Bean 生命周期回调](https://docs.spring.io/spring-framework/reference/core/beans/factory-nature.html)
