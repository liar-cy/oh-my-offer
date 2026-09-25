# Maven

- 题号前缀：MAVEN
- 范围：Maven 生命周期与常用命令（compile／package／install）、测试跳过参数差异、坐标与仓库、依赖仲裁与常用诊断命令。
- 最近更新：2026-09-25
- 说明：按本库面经整理；补充练习不计入原始面试问题。个人经历答案为框架，技术版本以题内说明为准。

## 目录

- [[#MAVEN-001：mvn compile、package、install 有什么区别？|MAVEN-001：生命周期三命令的区别]]
- [[#MAVEN-002：-DskipTests 和 -Dmaven.test.skip=true 有什么区别？|MAVEN-002：两种跳过测试的参数差异]]

### MAVEN-001：mvn compile、package、install 有什么区别？

**常见问法**

- [[面经/途虎养车/一面/0001#Q13：mvn compile、package、install 三个命令的区别|MJ070 · 途虎养车 · 一面 · Q13]]
- Maven 的生命周期有哪些阶段？

#### 面试回答

结论：三者是同一条构建生命线上的三个先后阶段，区别＝“构建推进到哪一步、产物放到哪里”。

- 生命周期主干：`validate → compile → test → package → verify → install → deploy`；执行靠后的阶段会自动先完成它前面的阶段。
- `mvn compile`：只把主代码编译到 `target/classes`，不跑测试、不打包。
- `mvn package`：编译＋跑测试＋打成 jar／war 放到 `target/`——产物只在本项目目录里。
- `mvn install`：在 package 之后，把产物和 pom 装进本地仓库（`~/.m2/repository`），本机其他工程才能按 GAV 依赖它；多模块联调最常用。
- 再往上是 `deploy`：推到远程仓库（私服／Nexus），团队共享——三兄弟里没有一个会把东西推上去。

#### 技术细节

**产物落点对比**

- compile：`target/classes`（字节码）。
- package：`target/*.jar｜*.war`（＋sources／javadoc 附件按插件配置）。
- install：`~/.m2/repository/<groupPath>/<artifactId>/<version>/...`。

**容易说不准的三处**

- “install 会上传服务器”——错，只到本机仓库；上传是 deploy。
- “package 不跑测试”——默认跑；跳测试是 MAVEN-002 那两个参数的事。
- 生命周期阶段是“有序前缀执行”：`mvn test` 一定先 compile；但直接调插件目标（如 `mvn surefire:test`）不保证前置阶段，构建脚本里要分清。

**多模块常配参数**

- `mvn -pl 模块 -am install`：只构建目标模块及其依赖方（also-make），大仓提速明显。
- `-o`（offline）只吃本地仓库，网络抖动时排障用；`-U` 强制更新 SNAPSHOT。
- 依赖排查：`mvn dependency:tree -Dverbose` 看仲裁结果，版本冲突先 `dependencyManagement` 统一再谈排除。

#### 深挖追问

1. **SNAPSHOT 和 RELEASE 在 install 时有什么不同？**（补充练习）

   SNAPSHOT 每次构建可覆盖本地仓库里的时间戳版本，install 后依赖方立刻拿到新构建；RELEASE 版本不允许重复覆盖，正式发版靠 deploy 到私服。

2. **CI 里常用哪个？**（补充练习）

   组件库：`mvn deploy`（或 release 插件）；应用：`mvn verify`／`package` 出产物交给镜像构建；“CI 里长期 `-DskipTests`”是把质量门拆了，见 [[#MAVEN-002：-DskipTests 和 -Dmaven.test.skip=true 有什么区别？|MAVEN-002]]。

**面经来源**

- [[面经/途虎养车/一面/0001#Q13：mvn compile、package、install 三个命令的区别|MJ070 · 途虎养车 · 一面 · Q13]]

**参考资料**（查证日期：2026-09-25；适用 Maven 3.x）

- [Maven：Introduction to the Build Lifecycle（默认生命周期阶段清单）](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)
- [Maven：Dependency Mechanism（本地仓库与 GAV）](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html)

**相关题目**

[[#MAVEN-002：-DskipTests 和 -Dmaven.test.skip=true 有什么区别？|MAVEN-002：跳过测试的参数]]

### MAVEN-002：-DskipTests 和 -Dmaven.test.skip=true 有什么区别？

**常见问法**

- [[面经/途虎养车/一面/0001#Q14：-DskipTests 和 -Dmaven.test.skip=true 的区别|MJ070 · 途虎养车 · 一面 · Q14]]
- 打包时想跳过测试，两个参数怎么选？

#### 面试回答

结论：一个“跳执行”、一个“连编译都跳”——`-DskipTests` 仍编译测试源码只是不运行；`-Dmaven.test.skip=true` 让测试源码根本不参与编译。

- `-DskipTests`：surefire／failsafe 跳过用例执行；`target/test-classes` 照常产出，测试代码写错会立刻暴露。
- `-Dmaven.test.skip=true`：maven-compiler-plugin 一并跳过测试编译——更快，但测试目录烂在锅里也不报错。
- 还有个近亲 `-Dskip`：那是 surefire 的“连测试类都不编译”总开关，语义最重，别和上面两个混用着说。
- 口径：临时赶构建用 `-DskipTests`（保留编译这一层保护）；CI 长期跳测试＝拆质量门，要跑指定用例用 `-Dtest=XxxTest#method`。

#### 技术细节

**属性作用面**

- `skipTests` 由 surefire／failsafe 插件解释；`maven.test.skip` 由 compiler 与 surefire 共同解释（编译＋执行都跳）。
- 两者都可写进 pom 的 `<properties>` 持久生效——写进 pom 就要评审，别在代码库里留下“永远不跑测试”的隐形配置。

**验证是否真的生效**

- 看构建日志：跳执行时 surefire 打印 “Tests are skipped.”；跳编译时不出现 `Compiling N source files ... test` 行。
- 看产物：`target/test-classes` 存在与否，是两参数最直观的差异证据。

**跑一部分的正确姿势**

- `-Dtest=OrderServiceTest`、`-Dtest='*IT'`、failsafe 的 `-Dit.test=`；配合 `-DfailIfNoTests=false` 在多模块下不误报。

#### 深挖追问

1. **为什么本地跳测试、CI 不跳，还会线上炸？**（补充练习）

   本地少跑一轮测试，合码时就全靠 CI 兜底。

   - 若 CI 也配了 skip，测试等于从未在集成环境跑过。
   - 这是流程问题，不是参数选择问题（质量门流程见 [[专题题库/软件工程#ENG-001：使用 AI／vibe coding 开发项目的流程是什么？|ENG-001 的工程纪律部分]]）。

**面经来源**

- [[面经/途虎养车/一面/0001#Q14：-DskipTests 和 -Dmaven.test.skip=true 的区别|MJ070 · 途虎养车 · 一面 · Q14]]

**参考资料**（查证日期：2026-09-25；适用 Maven Surefire／Compiler 插件当期文档）

- [Maven Surefire：How to skip tests only, but still compile](https://maven.apache.org/surefire/maven-surefire-plugin/examples/skip-test.html)
- [Maven Compiler Plugin：maven.test.skip 属性](https://maven.apache.org/plugins/maven-compiler-plugin/testCompile-mojo.html)

**相关题目**

[[#MAVEN-001：mvn compile、package、install 有什么区别？|MAVEN-001：生命周期与默认跑测试]]
