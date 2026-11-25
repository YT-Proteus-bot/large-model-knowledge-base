---
title: Java 知识库：从零基础到工程实践
topic: 教学知识库
sub_topic: Java
doc_type: kb-doc
level: 入门
chapters: 11
chars: 128110
build_date: '2026-11-25'
---

# Java 知识库：从零基础到工程实践

从 JDK 环境与基础语法讲到面向对象、集合、泛型异常反射、IO/NIO、并发编程、JVM 原理、Java 8+ 新特性，再到 Spring 生态与工程实践中的经典坑。

> 适合人群：零基础或刚入门的开发者、需要带新人的工程师；每章独立成节，可按需跳读，也适合整篇灌入 RAG 系统做检索。

## 目录

1. 环境与工具链
2. 语法基础与类型系统
3. 面向对象
4. 集合框架
5. 泛型、异常、注解与反射
6. IO 与 NIO
7. 并发编程
8. JVM 原理
9. Java 8+ 新特性
10. 主流框架与工程
11. 工程实践与常见坑

## 1. 环境与工具链

Java 的报错有一个稳定特征：提示看起来像代码写错，根因却在「编译这份源码的 JDK」与「运行它的 JVM」不是同一个。同一份源码在同事机器上能跑、在 CI 上抛 `UnsupportedClassVersionError`，差别通常只有三处：版本号、`JAVA_HOME` 的指向、依赖解析的结果。把环境与工具链跑通要按顺序落实五件事——定 JDK 版本、装好并验证路径、配好 IDE、看懂依赖坐标、走通一次从编译到运行，每一步都应当能用命令自证，而不是凭「能跑就行」。

### 1.1 JDK 版本选择与 LTS 策略

三个概念先拆开。JVM（Java Virtual Machine，执行字节码的虚拟机）负责运行程序；JRE（Java Runtime Environment，JVM 加标准类库）只能运行、不能编译；JDK（Java Development Kit，JRE 加 javac 编译器、jlink、jcmd、jstack 等开发诊断工具）才是开发环境。只装了 JRE 的机器上 `java -version` 一切正常，`javac -version` 却报「不是内部或外部命令」，就是这个区别造成的。

Java 从 9 起改为每 6 个月发布一版（3 月与 9 月），其中每两年指定一个长期支持版（LTS，Long Term Support，供应商提供多年安全更新的版本），其余为非 LTS，只维护到下一个版本发布。选型的实际结论只有一句：新项目选最新的 LTS，存量项目由依赖生态的硬约束决定，不要为了「用新语法」而升级。

| 版本 | 发布 | 类型 | class 文件版本 | 值得注意的变化 |
| --- | --- | --- | --- | --- |
| 8 | 2014-03 | LTS | 52 | Lambda、Stream、java.time |
| 11 | 2018-09 | LTS | 55 | 局部变量 var、标准 HttpClient |
| 17 | 2021-09 | LTS | 61 | record、sealed、instanceof 模式匹配转正、强封装 JDK 内部 API |
| 21 | 2023-09 | LTS | 65 | 虚拟线程、switch 模式匹配、分代 ZGC |
| 25 | 2025-09 | LTS | 69 | 紧凑源文件与实例 main 方法、模块导入声明、紧凑对象头 |

class 文件版本这一列最实用，它能反查一切版本错配。看到 `UnsupportedClassVersionError: com/example/App has been compiled by a more recent version of the Java Runtime (class file version 65.0)`，65 就意味着这个类由 JDK 21 编译，运行它的 JVM 必须不低于 21；55 对应 11，52 对应 8，61 对应 17。反过来，用 21 编译、用 17 运行，报错信息完全一样，只是数字不同。

发行版（Distribution，基于同一份 OpenJDK 源码、由不同厂商编译打包的 JDK）之间的差别在许可与支持，不在语言特性，都通过 TCK 兼容性测试，换发行版不需要改代码。Oracle JDK 自 17 起按 NFTC 许可可免费用于开发与测试，生产使用需要订阅；Oracle JDK 8 从 8u211 起商用即需付费，这是很多老项目被法务点名的原因。可免费商用的选择有 Eclipse Temurin（Adoptium）、Amazon Corretto、Alibaba Dragonwell、Azul Zulu、Microsoft Build of OpenJDK，生产环境任选其一并固定小版本即可。

框架的版本约束要和 JDK 一起看：Spring Boot 3.x 要求 JDK 17 及以上，Spring Boot 2.7 支持 8 到 17，想用虚拟线程必须 21 及以上。升级路径上，8 直接到 17 是常见选择，但如果项目依赖大量只在 8 上验证过的老库，先到 11 再到 17 更容易定位问题；跨过 17 直接上 21 时，要注意强封装带来的反射报错，那些在 8 上「随便反射 JDK 内部类」的代码会在运行时抛 `InaccessibleObjectException`。

预览特性（Preview Feature，语法已可用但尚未定稿、必须加 `--enable-preview` 才能编译的 API）不能进生产：它只在当个版本有效，下一个版本可能改动签名，而且要求编译与运行的 JDK 版本完全相同。判断顺序应当是：先看依赖框架要求的最低 JDK，再看团队运维的镜像与监控代理支持到哪个版本，最后才考虑语言特性带来的收益。为了 record 或模式匹配把整套运行环境抬到最新的非 LTS 版本，通常不划算。

新建项目在 21 与 25 之间取舍时，看的也是生态成熟度：主流框架、中间件客户端与 APM 代理对 21 的验证最充分，出了问题容易搜到答案；25 的维护窗口更长，适合依赖面窄、从零起步的新服务。同一个组织内不建议同时维护三个以上大版本，JDK 每多一个版本，构建镜像、流水线模板与运维基线就要各维护一份。

### 1.2 安装与环境变量

`JAVA_HOME` 指向 JDK 根目录，也就是包含 `bin`、`lib`、`conf` 的那一层，不是 `bin` 目录本身，也不是 JRE 目录；`PATH` 里加入 `%JAVA_HOME%\bin`。两个变量分工明确：Maven、Gradle 命令行、Tomcat 启动脚本读 `JAVA_HOME` 找 JDK，手敲 `java` 命令靠 `PATH` 查找。Maven 报 `JAVA_HOME should point to a JDK not a JRE`，就是这一条被违反了。

Windows 上最常见的「版本对不上」不是变量没设，而是 Oracle 安装器会把 `C:\Program Files\Common Files\Oracle\Java\javapath` 写进 `PATH`，而且往往排在自定义路径之前。javapath 里放着 `java.exe`、`javaw.exe`、`javac.exe` 几个转发程序，都指向最后一次安装的 JVM。结果是 `JAVA_HOME` 写着 21，`java -version` 却输出 8。判断方法是把三处输出放在一起对照：

```powershell
$env:JAVA_HOME                # 应指向 JDK 根目录
java -version                 # 第一行的版本号
javac -version                # 必须与上一行完全一致
java -XshowSettings:properties -version 2>&1 | Select-String 'java.home'
```

`java.home` 若不在 `$env:JAVA_HOME` 之下，说明 `PATH` 上有别的 java 抢先。修法是把 `%JAVA_HOME%\bin` 提到 `PATH` 最前，或删掉 javapath 条目：

```powershell
# 系统级设置需管理员权限，改完必须重开终端，已打开的窗口读不到新值
[Environment]::SetEnvironmentVariable('JAVA_HOME','C:\dev\jdk-21.0.5','Machine')
$old = [Environment]::GetEnvironmentVariable('Path','Machine')
[Environment]::SetEnvironmentVariable('Path',"%JAVA_HOME%\bin;$old",'Machine')
```

```bash
# Linux / macOS：写入 ~/.bashrc 或 ~/.zshrc 才会对所有会话生效
export JAVA_HOME=/opt/jdk-21.0.5
export PATH="$JAVA_HOME/bin:$PATH"
readlink -f "$(command -v java)"   # 应落在 $JAVA_HOME/bin/java
```

多版本共存的稳妥做法：Linux 与 macOS 用 SDKMAN!（`sdk list java`、`sdk use java 17.0.13-tem`），Debian 系也可以用 update-alternatives 管理符号链接；Windows 上更直接，保留多个 JDK 目录，切换时只改 `JAVA_HOME`，再让每个项目在 IDE 里单独指定 SDK。不要靠「重装另一个版本」来换版本，重装会顺手改掉 javapath 与注册表项，问题只是被推迟到下次。

容器里不要重复设 `JAVA_HOME`。`eclipse-temurin:21-jre` 这类官方镜像已经设好该变量与 `PATH`，再覆盖反而容易写错；而自己解压 JDK 的镜像必须显式补上 `ENV JAVA_HOME=/opt/java/openjdk` 与 `ENV PATH=$JAVA_HOME/bin:$PATH`，否则依赖 shell 的启动脚本拿不到 `jcmd`、`jstack` 这些诊断工具。CI 中注入 JVM 参数的通用做法是用 `JAVA_TOOL_OPTIONS` 环境变量，它会被任何 JVM 启动时自动读取，并在标准错误打印一行 `Picked up JAVA_TOOL_OPTIONS: ...`——这行提示本身不是报错，但经常被日志采集当成错误关键字，需要提前在告警规则里排除。

### 1.3 IDEA 配置要点

IntelliJ IDEA 里决定能否正常编译的是五处设置，它们各自独立，改一处不会自动同步另一处。

第一处是 Project Structure 里的 Project SDK 与 Language level，两者应与实际 JDK 一致（SDK 选 21，Language level 选 21）。第二处是 Modules 里的 SDK，多数时候保留为「Project SDK」跟随项目；一旦被单独改成别的版本，IDE 内编译与命令行 Maven 就会用不同 JDK，出现「IDE 里能跑、`mvn` 编不过」。第三处是 `Settings > Build, Execution, Deployment > Build Tools > Maven`：Maven home path 建议选 Bundled 或与 CI 同版本的本地安装；JDK for importer 与 Runner 的 JRE 都指向同一个 JDK；User settings file 决定读哪一份 `settings.xml`，私服配置写错时依赖永远拉不下来。第四处是 `Settings > Build > Compiler > Java Compiler` 的 target bytecode version，用 Maven 管理时应以 pom 里的 `maven.compiler.release` 为准。第五处是 `Editor > File Encodings`，全部设为 UTF-8，并勾选 Transparent native-to-ASCII conversion（只影响 `.properties` 里的中文转义）。

另有三处设置影响「能跑但行为诡异」。`Build Tools > Maven > Runner` 勾上 Delegate IDE build/run actions to Maven，让 IDE 的构建与运行走同一套 pom，避免 IDE 内置编译器与 maven-compiler-plugin 版本差异带来的匿名内部类、注解处理器不一致；`Build > Compiler > Annotation Processors` 勾上 Enable annotation processing，Lombok 与 MapStruct 靠它生成代码，不勾则在源码里满屏红字但命令行编译正常；`Build > Compiler` 里勾上 Build project automatically 之后，热部署才生效，但它与 Maven 增量编译偶有冲突，出现「改了没生效」时先手动 Rebuild。

工程协作上，`.idea/`、`*.iml`、`target/`、`out/` 都不入库，只提交 `pom.xml` 或 `build.gradle.kts` 以及 `.mvn/`、`gradlew` 与 wrapper 配置。项目路径不要含中文与空格，Windows 上 `C:\Users\张三\Desktop\demo` 这类路径会让部分工具在拼命令行参数时出错。索引卡顿时把 `target` 目录在 Project Structure 里标记为 Excluded，IDEA 不再扫描编译产物。

跨机器一致性靠 wrapper 而不是靠口头约定：`mvn wrapper:wrapper -Dmaven=3.9.9` 会生成 `mvnw`、`mvnw.cmd` 与 `.mvn/wrapper/maven-wrapper.properties`，团队统一执行 `./mvnw clean package`，CI 也用它，构建工具版本就不再取决于某台机器装了什么。Gradle 同理，`gradlew` 与 `gradle/wrapper/gradle-wrapper.properties` 一起入库。IDEA 的 Run Configuration 里另有一处值得固定：VM options 加 `-Dfile.encoding=UTF-8`，避免从 IDE 直接运行时输出编码与终端不一致，日志里的中文变成问号。

编辑器里满屏红色 `Cannot resolve symbol` 时，按三个方向依次排除：先在 `pom.xml` 里确认依赖确实被引入（`mvn -q dependency:list` 能看到），再确认 Project SDK 与 pom 里的 `release` 一致，最后执行 `File > Invalidate Caches / Restart` 清索引。三种原因的表现几乎一模一样，只有固定顺序排查才不至于反复删缓存。

### 1.4 Maven 与 Gradle：坐标、依赖与取舍

Maven 用坐标（Coordinate，`groupId` + `artifactId` + `version` 三段定位一个构件）描述依赖，Gradle 用同构的 `group:name:version` 字符串，两者背后是同一套仓库体系：Maven Central 与公司私服（Nexus、Artifactory）提供远端，本地缓存分别在 `~/.m2/repository` 与 `~/.gradle/caches`。版本号带 `-SNAPSHOT` 表示可变快照，构建工具会按更新策略重新拉取；正式版本一旦发布就不该覆盖，覆盖会让同一份 pom 在不同机器上解析出不同内容。

作用域（Scope）决定依赖在哪个阶段可见，Maven 的五个取值需要记牢：

| scope | 编译主代码 | 编译测试代码 | 运行 | 打进产物 | 典型用途 |
| --- | --- | --- | --- | --- | --- |
| compile | 是 | 是 | 是 | 是 | 默认值，日志门面、工具库 |
| provided | 是 | 是 | 否 | 否 | servlet-api，由容器提供 |
| runtime | 否 | 是 | 是 | 是 | JDBC 驱动、日志实现 |
| test | 否 | 是 | 仅测试期 | 否 | JUnit、Mockito |
| import | 仅用于 dependencyManagement | — | — | — | 导入 BOM 统一版本 |

`provided` 有个隐蔽后果：依赖不进入产物，部署到没有该库的环境就会在首次调用时抛 `NoClassDefFoundError`。Lombok 更特殊，它只需要在编译期运行注解处理器，正确做法是用 `provided` 加 `annotationProcessorPaths`，而非放进 `compile`。

依赖冲突的调解规则只有两条：路径最短者胜；路径长度相同时，先声明者胜。传递依赖不会报错，只会静默选中一个版本，所以「本地正常、线上报 `NoSuchMethodError`」几乎都是某个传递依赖把版本降了下去。定位靠依赖树：

```bash
mvn -q dependency:tree -Dincludes=com.fasterxml.jackson.core   # 只看某个 group 的解析结果
mvn -q dependency:tree -Dverbose                                # 列出被省略与冲突的节点
```

固定版本的优先级从低到高是：传递依赖 < `dependencyManagement` < 直接声明。多模块项目在父 pom 的 `dependencyManagement` 写死版本，子模块省略 `version`；跨项目统一版本用 `import` 作用域引 BOM（Bill of Materials，只含版本清单的 pom），例如 `spring-boot-dependencies`。排除某个传递依赖用 `<exclusions>`，但排除之后要自己补上所需版本，否则问题从「版本冲突」变成「类缺失」。

一个具体例子能说明这条规则的杀伤力：项目自身没有声明 Jackson，两个内部 SDK 分别传递依赖 `jackson-databind:2.9` 与 `2.12`，两条路径的深度相同，于是「先声明者胜」生效，实际进 classpath 的可能是更旧的 2.9。症状是运行时 `NoSuchMethodError`，栈顶指向业务代码，看上去完全不像环境问题。处置是在 `dependencyManagement` 里把 Jackson 统一钉到目标版本，并让 CI 的依赖检查失败即中断，而不是靠谁记得加排除。

私服配置写在 `~/.m2/settings.xml` 的 `<mirrors>` 中，把 `central` 指向公司仓库；需要认证的仓库在 `<servers>` 里配 id 与凭据，该 id 必须与 mirror 或 pom 里的仓库 id 完全一致，写错只会得到 401，且提示里不会告诉你差在哪。网络受限时用 `mvn -o` 走离线模式，只用本地仓库已有构件；快照版本要强制刷新时用 `mvn -U`。这两个开关能把「有网却拉不下来」与「改了快照但没生效」两类问题分开。

```xml
<properties>
  <maven.compiler.release>21</maven.compiler.release>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
<dependencies>
  <dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>2.0.16</version>
  </dependency>
  <dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.11.0</version>
    <scope>test</scope>   <!-- 只在测试编译与测试运行时可见 -->
  </dependency>
</dependencies>
```

两种工具的取舍可以压缩成几个维度：

| 维度 | Maven | Gradle |
| --- | --- | --- |
| 配置形式 | XML，声明式，结构固定 | Groovy / Kotlin DSL，可编程 |
| 约定 | 生命周期与目录强约定，难偏离 | 约定可覆盖，灵活但依赖自律 |
| 增量与缓存 | 一般 | 任务级增量、构建缓存，明显更快 |
| 大型多模块 | 构建时间随模块数近似线性增长 | 复用与并行更好 |
| 依赖可见性 | 统一 compile 类路径 | `api` 与 `implementation` 显式区分 |
| 学习成本 | 低，报错直观 | 高，脚本本身要维护 |

Gradle 最值得注意的差异是可见性：`api` 会传递给依赖本模块的下游，`implementation` 不会。把实现细节写成 `api`，下游编译期就会看到不该看到的类型，改一次实现引发连锁重编。版本对齐推荐用版本目录（`gradle/libs.versions.toml`）而不是各模块各写一个常量。

```kotlin
plugins { java; application }
java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }  // 锁定编译用 JDK，不受机器默认 JDK 影响
dependencies {
    implementation("org.slf4j:slf4j-api:2.0.16")
    testImplementation("org.junit.jupiter:junit-jupiter:5.11.0")
}
application { mainClass = "com.example.App" }
```

### 1.5 标准目录结构与编译到运行

Maven 与 Gradle 默认共用同一套目录约定（Gradle 的 java 插件直接沿用 Maven 布局），这是 Java 生态最容易迁移的部分：

```text
demo/
├── pom.xml                                   # 或 build.gradle.kts + settings.gradle.kts
├── .mvn/wrapper/                             # 可选，锁定 Maven 版本
└── src/
    ├── main/java/com/example/App.java        # 生产代码，目录层级必须与 package 一致
    ├── main/resources/application.properties # 配置文件，打包后位于 classpath 根
    ├── test/java/com/example/AppTest.java    # 测试代码，不进产物
    └── test/resources/                       # 测试专用配置，优先级高于 main
```

三个约定要记住：`src/main/java` 下的文件必须放在与 `package` 声明一致的目录里；`resources` 下的文件在运行时经 classpath 读取，路径不以 `src` 开头；测试类与生产类同包名，才能访问包级私有成员。

不借助构建工具手动跑一遍，能看清每一步产物是什么：

```java
// src/main/java/com/example/App.java
package com.example;

public class App {
    public static void main(String[] args) {
        System.out.println("running on JDK " + System.getProperty("java.version"));
    }
}
```

```bash
# 1) 编译：-d 指定输出目录，字节码按包名分层生成 target/classes/com/example/App.class
javac -encoding UTF-8 --release 21 -d target/classes $(find src/main/java -name '*.java')
# 2) 运行：-cp 指向 class 根目录，之后是全限定类名，不带 .class 后缀
java -cp target/classes com.example.App
# 3) 打 jar：--main-class 写进 MANIFEST，-C 切换目录避免把 classes 这层带进包内
jar --create --file app.jar --main-class com.example.App -C target/classes .
java -jar app.jar
```

Windows 下第一条要换成 PowerShell 写法 `javac -encoding UTF-8 --release 21 -d target/classes (Get-ChildItem -Recurse -Filter *.java src/main/java).FullName`，其余命令一致。

`--release 21` 同时做三件事：设定字节码版本、把可编译期使用的标准库 API 限制在 21 以内、按该版本的系统符号表编译。老写法 `-source 8 -target 8` 只改字节码版本，仍允许调用 21 才有的方法，运行到那一行才抛 `NoSuchMethodError`，这是「编译通过、上线崩溃」的经典来源。Maven 中对应 `maven.compiler.release`，不要用 `source` 与 `target` 两个属性替代。

JDK 9 之后还有一条并行的模块路径（Module Path）通道：源码里出现 `module-info.java` 时，构建工具会把编译切到模块模式，`requires` 未声明的包在编译期就不可见，运行时报 `java.lang.module.FindException`，而不是熟悉的 `ClassNotFoundException`。多数业务项目仍跑在 classpath 上，碰到这类报错先确认是否某个依赖自带了模块描述符。做精简镜像时可以用 `jdeps --print-module-deps --ignore-missing-deps app.jar` 列出运行所需模块，再用 jlink 裁掉用不到的部分。

`mvn package` 产出的 `target/demo-1.0.0.jar` 默认不含任何第三方依赖，`java -jar` 一碰框架类就抛 `NoClassDefFoundError`。要得到可直接分发的整包，用 maven-shade-plugin（把依赖解包合并进同一个 jar）、maven-assembly-plugin（依赖放进 `lib/` 并在 Manifest 里写 Class-Path），或 Spring Boot 的 `spring-boot-maven-plugin`（生成含嵌套 jar 的可执行包）。发布前用 `unzip -l app.jar | head -20` 看一眼，只看到自己的类就说明打包方式选错了。

### 1.6 常见环境问题与排查顺序

| 现象 | 根因 | 处置 |
| --- | --- | --- |
| `UnsupportedClassVersionError`，class file version 65.0 | 运行时 JVM 低于编译用 JDK | 统一 `--release` 与运行 JDK，容器基础镜像同步换 |
| `错误: 找不到或无法加载主类` | `-cp` 未含根目录、包名与目录不符、类名多写了 `.class` | 用 `java -cp target/classes 全限定类名` 复核 |
| `ClassNotFoundException` / `NoClassDefFoundError` | 依赖未进 classpath，或静态初始化块抛异常导致类初始化失败 | 先查依赖树，再看异常链第一个 `Caused by` |
| `OutOfMemoryError: Metaspace` | 动态生成类过多（反射、CGLIB、热部署反复加载） | 设 `-XX:MaxMetaspaceSize=512m`，并排查重复创建代理 |
| 控制台中文乱码 | 源文件编码与 `-encoding` 不一致；17 及以前 Windows 默认 GBK | 统一 UTF-8；JDK 18 起默认字符集已是 UTF-8 |
| `Could not resolve dependencies` / `PKIX path building failed` | 私服地址错、代理做证书拦截 | 核对 `settings.xml` 的 mirror 与 server，导入公司根证书 |
| Maven 报 `JAVA_HOME should point to a JDK not a JRE` | `JAVA_HOME` 指向 JRE 或 JDK 的 `bin` | 改为 JDK 根目录，用 `mvn -v` 回显确认 |

排查顺序固定为四步，不要跳步。第一步，`java -version` 与 `javac -version` 两行版本是否一致；第二步，`JAVA_HOME` 是否指向 JDK 根目录；第三步，`mvn -v`（或 `./gradlew --version`）回显的 Java home 是否与第二步相同；第四步，`mvn dependency:tree` 看可疑依赖实际解析到的版本。四步覆盖了绝大部分「环境问题」，跳过任一步都会把版本问题误判成代码问题。

对已经在跑的进程，还有两个只认现场的手段：`jcmd <pid> VM.system_properties` 会打印该进程真实的 `java.home`、`java.class.path` 与全部系统属性，比对启动脚本更可靠；`jcmd <pid> VM.version` 给出 JVM 版本与参数来源。一台机器上跑着多个服务、版本各不相同的时候，这两个命令是唯一能确证「到底用的是哪个 JDK」的证据。

一个真实案例：本地 JDK 21、pom 里写 `release 21`，编译打包都正常，CI 用 `maven:3.9-eclipse-temurin-17` 镜像构建，直接在编译阶段报 `release version 21 not supported`。根因不是代码，而是构建镜像的 JDK 与 `release` 不匹配。处置有两条路：换基础镜像到 21，或把 `release` 降到 17——后者要求先确认没有用到 17 之后才有的 API，否则问题会从编译期推迟到运行期。

### 1.7 常见坑

- **把 JRE 当 JDK，或 `JAVA_HOME` 指到 `bin`**：`java` 能用、`javac` 找不到，Maven 直接拒绝启动。判断依据是 `JAVA_HOME` 下是否有 `bin/javac`，而不是环境变量看起来像不像。
- **用 `-source` / `-target` 代替 `--release`**：字节码版本降下来了，编译期却仍允许调用高版本 API，直到运行到那一行才崩，且崩在现场业务代码里，排查成本远高于编译期报错。
- **IDE 与命令行用的不是同一套 Maven 配置**：IDEA 默认读 Bundled Maven 自带的 `conf/settings.xml`，命令行读 `~/.m2/settings.xml`，两者私服与镜像不同，表现为「IDE 能下依赖、`mvn` 下不下来」。统一 `Maven home path` 与 `User settings file` 即可。
- **把 `target/`、`.idea/`、本地 jar 一起提交**：`system` 作用域的本地 jar 不参与依赖调解，别人拉下来必然缺类；产物目录入库则会让「源码改了但编译结果没变」的假象反复出现。`.gitignore` 里写好 `target/`、`out/`、`.idea/`、`*.iml` 是最省事的防线。

## 2. 语法基础与类型系统

Java 的语法和 C 系语言相似，差异藏在类型系统里：基本类型与引用类型的分野、自动装箱、字符串不可变、写死的数值提升规则。它们决定了同一行代码在哪儿悄悄建了对象、在哪儿丢了精度。

### 2.1 基本类型与包装类

Java 的类型只有两类：基本类型（Primitive Type，直接存数值、不持有对象引用的八种内置类型）和引用类型（Reference Type，变量里存的是指向堆中对象的地址）。

| 类型 | 位数 | 默认值 | 包装类 |
| --- | --- | --- | --- |
| byte | 8 | 0 | Byte |
| short | 16 | 0 | Short |
| int | 32 | 0 | Integer |
| long | 64 | 0L | Long |
| float | 32 | 0.0f | Float |
| double | 64 | 0.0d | Double |
| char | 16 | '\u0000' | Character |
| boolean | JVM 相关 | false | Boolean |

默认值只对成员变量成立，局部变量没有默认值，未赋值就使用会直接编译失败——这是 Java 故意让「忘记初始化」在编译期暴露。

```java
static int count;        // 成员变量：默认 0
static String name;      // 引用类型默认 null
int local;               // 局部变量没有默认值
// System.out.println(local); // 编译错误：might not have been initialized
```

char 是 16 位无符号的，能直接存一个汉字：`char c = '中';`。

### 2.2 装箱、拆箱与缓存

装箱（Boxing，把基本类型值包装成对应包装类对象的过程）由编译器在需要时自动插入，叫自动装箱；反向过程叫拆箱（Unboxing）。编译器插入的是 `Integer.valueOf(int)` 和 `Integer.intValue()`，不是 new。`valueOf` 对 -128 到 127 走 IntegerCache，返回同一个对象；Float 和 Double 没有缓存。

```java
Integer a = 127, b = 127;
Integer c = 128, d = 128;
System.out.println(a == b);   // true：命中缓存，同一个对象
System.out.println(c == d);   // false：超出范围，两个不同对象
```

铁律是包装类比较值一律用 `equals` 或 `Objects.equals`。更隐蔽的是拆箱引发的空指针：

```java
Map<String, Integer> map = new HashMap<>();
int n = map.get("missing");        // 拆箱 null → NullPointerException
int safe = Objects.requireNonNullElse(map.get("missing"), 0);

Integer x = null;
int r = false ? x : 0;             // 两分支类型统一为 int，仍要拆箱 → NPE
```

真实排查步骤：一是看堆栈，异常行出现 `Integer.intValue()` 而代码里没有这个调用，就是编译器插入的拆箱；二是用 `javap -c -p 类名` 反编译，在对应行找 `invokevirtual java/lang/Integer.intValue`，定位到具体变量；三是回溯来源，`Map.get`、`Optional`、JSON 反序列化都可能给 null；四是判空或用 `Objects.requireNonNullElse` 修复。`Long sum = 0L; sum += i;` 每次累加都新建对象，百万级循环里换成 long 即可。

### 2.3 String、StringBuilder 与字符串常量池

String 是不可变（Immutable，对象创建后内容无法修改）的引用类型，内部用 byte 数组存字符。不可变让它可安全共享、天然线程安全、hashCode 可缓存一次，因此适合做 HashMap 的键。

字符串常量池（String Pool，JVM 中存放字符串字面量的区域，同一字面量只保留一份）决定了 `==` 的结果：

```java
String s1 = "java", s2 = "java";
String s3 = new String("java");
String s4 = s1 + "va";              // 编译期常量折叠
System.out.println(s1 == s2);       // true：同一个常量池对象
System.out.println(s1 == s3);       // false：new 在堆上另建对象
System.out.println(s1 == s3.intern()); // true：intern 返回池中对象
```

`+` 只在单次表达式里安全，写进循环就是灾难：每次迭代都新建 StringBuilder 并复制已有内容，复杂度退化成 O(n²)。

```java
String result = "";
for (int i = 0; i < 10000; i++) result += i;   // 错误：反复复制已有内容

StringBuilder sb = new StringBuilder(40000);   // 正确：循环外建一个并预估容量
for (int i = 0; i < 10000; i++) sb.append(i);
String ok = sb.toString();
```

StringBuilder 非线程安全，StringBuffer 每个方法都加了 synchronized，单线程一律用前者。两个易错点：`split` 的参数是正则，按点号切分要写 `split("\\.")`；`replace` 是字面量替换，`replaceAll` 才是正则替换。

### 2.4 数组

数组（Array，长度固定、元素类型相同、内存连续的引用类型容器）有三个要点：长度创建后不可变；`length` 是字段不是方法，写 `arr.length` 而不是 `arr.length()`；元素有默认值，规则同成员变量。

```java
int[] primes = {2, 3, 5, 7};
int[][] jagged = {{1}, {1, 2}};          // 二维数组是「数组的数组」，每行可不等长
System.out.println(primes.length);       // 4，注意没有括号
Object[] objs = new String[1];           // 数组协变：编译通过
objs[0] = 42;                            // 运行期抛 ArrayStoreException
```

拷贝有三种：`System.arraycopy` 最快但要自控长度，`Arrays.copyOf` 能扩容，`clone()` 是浅拷贝；`Arrays.binarySearch` 要求数组已排序。

### 2.5 运算符与类型转换

自动类型提升（Widening Conversion，取值范围小的类型自动转成大范围类型）的顺序是 byte → short → int → long → float → double，char 参与运算时先转 int；long 转 float 方向是「变宽」，但只有 24 位有效尾数，大整数会丢精度。强制类型转换（Narrowing Conversion，显式截断高位）则可能溢出。

```java
byte b = (byte) 300;      // 44：只保留低 8 位
short s = 1;
s += 1;                   // 通过：复合赋值隐含强制转换
// s = s + 1;             // 编译错误：int 不能直接赋给 short
System.out.println(7 / 2);  // 3，整数除法向零取整
System.out.println(-7 % 3); // -1，符号跟被除数
```

浮点数不能用 `==` 比较，要用误差范围或 `Double.compare`；金额一律用 BigDecimal。`&&` 短路、`&` 不短路：`text != null && text.length() > 0` 安全，换成 `&` 立刻 NPE。

### 2.6 流程控制

条件分支用 if / else if / else；按次数循环用 for，按条件循环用 while，至少要执行一次用 do-while。多分支的等值判断用 switch。

switch 支持 byte、short、char、int 及其包装类、String（Java 7 起）和枚举，不支持 long 与 double。最常犯的错是漏写 `break` 造成贯穿（Fall-through，命中一个分支后继续执行后面的分支）：`case 1: case 2: case 3:` 这种合并写法只在最后写一次 `break`，少写就会把后面的分支一起执行；`default` 放最后可读性最好。

增强 for 是语法糖，编译后对集合走迭代器、对数组走下标；循环体内增删集合元素会抛 ConcurrentModificationException，删元素要用迭代器的 `remove()`。跳出多层嵌套可用标签：循环前写 `outer:`，循环内写 `break outer;`。

### 2.7 方法、重载与可变参数

方法签名（Signature）只包含方法名加参数类型列表，返回类型、参数名、throws 都不参与，所以不能靠返回类型区分重载（Overload，同一类中方法名相同、参数列表不同的多个方法）。重载解析分三阶段：第一阶段不允许装箱拆箱和可变参数、只允许基本类型拓宽；第二阶段允许装箱拆箱；第三阶段才允许可变参数，前一阶段命中就不再往下走。

```java
static void print(long v)    { }   // 阶段一命中
static void print(Integer v) { }   // 阶段二
static void print(int... v)  { }   // 阶段三
print(1);                          // 走 long：int→long 拓宽即可，不必装箱
```

Java 只有值传递（Pass-by-Value，调用时把实参的值复制一份交给形参）：传引用类型时复制的是引用本身，能改对象内容，不能让形参改指新对象。可变参数（Varargs，形参表末尾用 `类型...` 声明、编译后就是该类型数组）必须放最后一个，一个方法只能有一个；`sum()` 传零个参数可以，传 `(int[]) null` 就会在遍历时 NPE。

### 2.8 包与访问修饰符

包（Package，组织类、划分命名空间并避免类名冲突的机制）的声明必须在源文件第一行（注释除外），命名用全小写反向域名，如 `com.example.order`。

| 修饰符 | 本类 | 同包 | 子类（跨包） | 其他包 |
| --- | --- | --- | --- | --- |
| private | 可访问 | 不可 | 不可 | 不可 |
| 默认（不写） | 可访问 | 可访问 | 不可 | 不可 |
| protected | 可访问 | 可访问 | 可访问 | 不可 |
| public | 可访问 | 可访问 | 可访问 | 可访问 |

protected 跨包时有个细节：子类只能通过自身类型的引用访问继承来的成员，用父类类型的引用访问会编译失败。

### 2.9 命名与编码规范入门

- 类、接口、枚举用大驼峰（UpperCamelCase），如 OrderService；方法、变量用小驼峰（lowerCamelCase），方法名动词开头，如 calculateTotal。
- 常量全大写加下划线且必须是 static final，如 MAX_RETRY_COUNT。
- 布尔值用 is / has / can 开头，如 isActive，不要用 flag、status2。
- 禁止魔法值（代码里直接出现的字面量数字或字符串），抽成常量或枚举。
- 缩进 4 空格，行宽不超过 120，方法不超过 80 行、参数不超过 5 个。

### 2.10 语法与类型速查表

| 现象 | 根因 | 正确做法 |
| --- | --- | --- |
| 两个 128 的 Integer 用 `==` 得 false | 超出 IntegerCache 范围 | 用 `equals` 或 `Objects.equals` |
| 三目表达式抛 NPE | 两分支类型统一触发拆箱 | 显式判空或改用基本类型 |
| 循环拼字符串越来越慢 | 每次迭代新建 StringBuilder 并复制 | 循环外用 StringBuilder 并预估容量 |
| switch 该进一个分支却进了多个 | 漏写 break 造成贯穿 | 每个分支补 break |
| `s = s + 1` 编译失败 | 只有复合赋值才有隐式强转 | 写成 `s += 1` |
| 数组赋值抛 ArrayStoreException | 数组协变把类型检查推到运行期 | 别把子类型数组赋给父类型数组变量 |

## 3. 面向对象

面向对象（Object-Oriented Programming，OOP，用「对象」组织程序、把数据与操作数据的方法捆绑在一起并支持继承与多态的编程范式）是 Java 的骨架。Java 没有顶层函数，所有代码必须写在类里，连 main 方法也是某个类的静态方法。初学者的困难通常不在语法，而在几条看不见的规则：new 的那一刻发生了什么、父类构造器里调用子类方法为什么读到 null、接口与抽象类怎么选、什么时候用继承会埋雷。这些规则不弄明白，代码照样编译通过、照样上线，然后以事故的形式在半年后暴露出来。

### 3.1 类与对象：模板、实例与引用

类（Class，把字段与方法打包在一起的类型定义，是创建对象的模板）与对象（Object，按类模板在堆上分配出来的实例）是同一件事的两面：类是图纸，对象是按图纸造出来的房子，可以造很多栋。字段（Field，类中声明的变量，描述对象的状态）与实例方法（Instance Method，能访问 this 与实例字段的方法）合起来决定对象的行为。

执行 `new Account("张三", 10000)` 分三步：在堆上分配内存并把字段清零、执行构造器、返回对象引用。变量里存的不是对象本身，而是引用（Reference，指向堆中对象的地址值）。所以 `Account b = a` 复制的是引用，a 与 b 指向同一个对象，通过 b 改状态，读 a 也能看到。

```java
public class Account {
    private String owner;           // 实例字段：每个对象各一份
    private long balanceCents;      // 金额以「分」为单位，避开浮点误差

    public Account(String owner, long balanceCents) {
        this.owner = owner;                 // this 指向当前对象
        this.balanceCents = balanceCents;
    }

    public void deposit(long cents) {
        if (cents <= 0) throw new IllegalArgumentException("金额必须为正");
        this.balanceCents += cents;         // 先校验再改，状态始终合法
    }

    public long getBalanceCents() { return balanceCents; }
}

Account a = new Account("张三", 10_000);
Account b = a;                              // 复制引用，不是复制对象
b.deposit(500);
System.out.println(a.getBalanceCents());    // 10500
```

Java 的参数传递只有一种：值传递（Pass by Value，调用方法时把实参的值复制一份给形参）。基本类型复制数值本身，引用类型复制引用的值。这解释了两个高频误解：

```java
static void reassign(Account x) { x = new Account("李四", 0); } // 只改本地副本，调用方引用不变
static void mutate(Account x)   { x.deposit(100); }             // 与调用方同一对象，改动可见
```

字段有默认值（int 为 0、boolean 为 false、引用为 null），局部变量没有，不初始化就编译报错。把「字段自动清零」当成「局部变量也安全」，是空指针的第一来源。

### 3.2 构造器与初始化顺序

构造器（Constructor，与类同名、没有返回类型、在 new 时执行的特殊方法）负责把对象从「全零状态」推进到合法状态。四条硬规则：构造器不被继承；`this(...)` 与 `super(...)` 只能出现在第一行且只能二选一；一个构造器都不写时编译器补一个无参构造器，一旦自己写了就没了；构造器可以重载。

第三条是框架报错的常见根因：给实体类手写了带参构造器后忘了补无参构造器，MyBatis、Jackson 这类靠反射调无参构造器建对象的框架会抛 `NoSuchMethodException` 或 `InvalidDefinitionException`。

初始化顺序按类与实例两条线走，且都由书写顺序决定：

| 阶段 | 内容 | 执行次数 |
| --- | --- | --- |
| 1 | 父类静态字段与静态块 | 类加载时一次 |
| 2 | 子类静态字段与静态块 | 类加载时一次 |
| 3 | 父类实例字段与实例块 | 每次 new |
| 4 | 父类构造器 | 每次 new |
| 5 | 子类实例字段与实例块 | 每次 new |
| 6 | 子类构造器体 | 每次 new |

```java
class Parent {
    static { System.out.println("1 父类静态块"); }
    { System.out.println("3 父类实例块"); }
    Parent() { System.out.println("4 父类构造器"); }
}
class Child extends Parent {
    static { System.out.println("2 子类静态块"); }
    private int x = init();                       // 5：字段初始化与实例块按书写顺序
    private static int init() { System.out.println("5 子类字段初始化"); return 0; }
    { System.out.println("6 子类实例块"); }
    Child() { super(); System.out.println("7 子类构造器"); }
}
new Child();   // 输出 1 2 3 4 5 6 7；再 new 一次只输出 3 4 5 6 7
```

子类字段在父类构造器执行完之后才初始化，这条顺序制造了一个经典陷阱：父类构造器里调用一个可被重写的方法，运行时落到子类实现，而子类字段此刻还是 null。

```java
abstract class Base {
    Base() { System.out.println("name=" + name()); }   // ❌ 构造器里调用可重写方法
    protected String name() { return "base"; }
}
class Sub extends Base {
    private String tag = "sub";
    @Override protected String name() { return tag; }  // 父类构造器执行时 tag 尚未赋值
}
new Sub();   // 输出 name=null
```

对策是把这类调用从构造器移出去，改成显式的 `init()` 或在工厂方法里调用；父类构造器只做本类字段的初始化。

### 3.3 封装：让非法状态无法被构造

封装（Encapsulation，隐藏内部实现、只暴露必要入口，使对象状态始终满足自身约束）的价值不在「字段加 private 比较好看」，而在于把校验集中到一处。字段全 public 的类，任何调用方都能把余额改成负数；改成私有字段加校验方法后，非法状态在入口就被拦掉，后续所有逻辑都不用再防御。

访问控制有四级，判断依据是本类、同包、子类、其他包：

| 修饰符 | 本类 | 同包 | 不同包子类 | 其他包 |
| --- | --- | --- | --- | --- |
| private | 可访问 | 不可 | 不可 | 不可 |
| 无修饰符（包私有） | 可访问 | 可访问 | 不可 | 不可 |
| protected | 可访问 | 可访问 | 可访问 | 不可 |
| public | 可访问 | 可访问 | 可访问 | 可访问 |

protected 有个细节：不同包的子类只能通过「子类类型的引用」访问继承来的 protected 成员，用父类引用访问会编译失败。实际开发中包私有是最被低估的修饰符，同包的协作类用它暴露内部方法，包外完全不可见，比 public 安全得多。

封装成熟的标志是能守住不变式（Invariant，对象在整个生命周期内必须始终成立的约束条件），并防住可变引用的外泄：

```java
public final class Period {
    private final LocalDate start;
    private final LocalDate end;
    private final List<String> tags;

    public Period(LocalDate start, LocalDate end, List<String> tags) {
        if (end.isBefore(start)) throw new IllegalArgumentException("结束日期早于开始");
        this.start = start;
        this.end = end;
        this.tags = List.copyOf(tags);   // 入参拷贝：外部改原 List 不影响本对象
    }
    public List<String> getTags() { return tags; }   // 不可变副本，可直接返回
}
```

如果字段必须持有可变集合又要对外暴露，就返回 `new ArrayList<>(tags)` 或 `Collections.unmodifiableList(tags)`。另外不要给每个字段机械生成 setter：为「余额」「创建时间」这类字段提供 setter，等于把刚封好的门重新打开。

### 3.4 继承、重写与动态绑定

继承（Inheritance，子类通过 extends 获得父类字段与方法的机制，Java 只支持单继承）表达 is-a 关系。多态（Polymorphism，同一处调用在运行时按对象实际类型选择不同实现的能力）是继承最主要的收益：调用方只依赖父类或接口，新增子类不用改调用代码。

三个容易混的概念必须拆开看：

- 重载（Overload）：同名方法、参数列表不同，编译期按实参的静态类型选定，与返回值无关；
- 重写（Override）：子类重新实现父类方法，方法名与参数列表完全一致，运行期按实际类型选定；
- 字段不参与多态：字段在编译期按静态类型解析，子类同名字段只是把父类字段「隐藏」了。

```java
class Animal {
    String name = "animal";
    String sound() { return "..."; }
}
class Dog extends Animal {
    String name = "dog";
    @Override String sound() { return "汪汪"; }
}
Animal a = new Dog();
System.out.println(a.name);     // animal：字段看静态类型
System.out.println(a.sound());  // 汪汪：方法看实际类型
```

方法调用的动态绑定（Dynamic Binding，运行期根据对象实际类型决定调用哪个实现）由虚方法表实现：每个类一张方法表，对象头里存类指针，调用时查表取地址。字段访问没有这层间接，所以「字段被隐藏」的行为差异根源就在这里。

| 维度 | 重载 Overload | 重写 Override |
| --- | --- | --- |
| 出现位置 | 同一个类或继承链上 | 子类、实现类 |
| 方法名与参数 | 名同、参数列表必须不同 | 名与参数列表必须完全相同 |
| 决定时机 | 编译期，看静态类型 | 运行期，看实际类型 |
| 返回类型 | 无约束 | 可协变（子类型） |
| 访问权限 | 无约束 | 不能比父类更严格 |
| 异常 | 无约束 | 不能抛出更宽的受检异常 |

private、static、final 方法都不能被重写：private 方法在子类里写同名方法属于新方法，加 @Override 会直接编译报错；static 方法只能被隐藏，通过父类引用调用时永远走父类版本。@Override 注解的真正价值是把「以为在重写、实际写成了重载」从运行期 bug 变成编译错误，务必每个重写方法都加。

### 3.5 接口与抽象类：契约与半成品

接口（Interface，只声明能力契约、不持有实例状态的类型，Java 8 起可含 default 与 static 方法，Java 9 起可含 private 方法）回答「能做什么」；抽象类（Abstract Class，用 abstract 声明、不能实例化、可混有抽象方法与具体实现与实例字段的类）回答「是什么」，并提供可复用的半成品实现。

| 维度 | 接口 | 抽象类 |
| --- | --- | --- |
| 实例字段 | 不能有（字段隐式为 public static final 常量） | 可以有任意实例字段 |
| 方法构成 | 抽象方法、default、static、private 方法 | 抽象方法与具体方法混用 |
| 构造器 | 没有 | 有，供子类 super 调用 |
| 一个类可继承几个 | 可实现多个接口 | 只能继承一个父类 |
| 语义 | can-do，横向能力 | is-a，纵向分类 |
| 演进风险 | 加 default 方法基本兼容已有实现 | 加抽象方法会破坏所有子类 |

```java
public interface PayChannel {
    String code();                              // 抽象方法：实现类必须提供
    long pay(long amountCents);

    default boolean supports(String c) {        // default 方法：公共逻辑只写一次
        return code().equalsIgnoreCase(c);
    }
    static PayChannel of(String code) {         // 静态工厂挂在接口上
        return switch (code) {
            case "wx" -> new WeChatPay();       // 实现类省略
            case "ali" -> new AliPay();
            default -> throw new IllegalArgumentException("未知渠道: " + code);
        };
    }
}

public abstract class AbstractPayChannel implements PayChannel {
    @Override public final long pay(long amountCents) {   // 模板方法：固定流程，禁止改
        if (amountCents <= 0) throw new IllegalArgumentException("金额非法");
        return doPay(amountCents);                          // 差异化步骤交给子类
    }
    protected abstract long doPay(long amountCents);
}
```

这段代码展示了三件事：default 方法让接口可以带公共实现而不破坏已有实现类；static 工厂方法让创建逻辑跟着契约走；抽象类用 final 模板方法锁住流程、把可变部分留成 abstract 钩子。选择规则很直接：需要多态替换且无共享状态就用接口；有共享状态或需要一个受控的扩展骨架就用抽象类；两者常常组合，抽象类实现接口。

多接口带来的默认方法冲突要认识：两个接口提供了同名同参的 default 方法，实现类必须自己重写并可用 `InterfaceA.super.method()` 指定，否则编译不通过。这是有意设计，避免歧义悄悄通过。

### 3.6 static 与 final：归属与不可变

static（静态成员，属于类而不属于某个实例，类加载时初始化一次，被所有实例共享）与 final（终态，变量只可赋值一次、方法不可重写、类不可继承）是两个正交的修饰符，组合含义要分清：

| 修饰对象 | final 的含义 | 典型用途 |
| --- | --- | --- |
| 局部变量、方法参数 | 只能赋值一次 | 满足 lambda 捕获要求 |
| 实例字段 | 构造完成后不可再指向别的对象 | 依赖注入、值对象字段 |
| static 字段 | 类级别常量 | 配置上限、错误码 |
| 方法 | 子类不能重写 | 模板方法的固定步骤、安全校验 |
| 类 | 不能被继承 | String、Integer、record |

```java
public class Counter {
    private static int created;                 // 类级别，所有实例共享
    private final int id;                       // 每个实例一份，创建后不变
    private static final int MAX = 10_000;      // 编译期常量，会被内联进调用方字节码

    public Counter() {
        if (created >= MAX) throw new IllegalStateException("超出上限");
        this.id = ++created;
    }
    public static int created() { return created; }   // 静态方法不能访问 id
}
```

static 的三条硬约束与一个隐性成本：静态方法里不能使用 this、实例字段、实例方法（编译报「non-static variable id cannot be referenced from a static context」）；静态方法不能被重写，只能被隐藏；静态字段是事实上的全局变量，测试之间、请求之间互相污染。

`static final` 还有一个编译期内联的坑：`public static final int MAX = 100` 属于编译期常量，值会被直接写进调用方的 class 文件。只改声明这个常量的类而不重新编译调用方，调用方用的仍是旧值 100。表现是「改了配置没生效、清缓存重启还是旧值」，本质是漏了增量编译。需要热改的配置不要用编译期常量表达，改为从配置中心读取的实例字段。

final 修饰引用只保证引用不变，不保证对象内容不变：`final List<String> list = new ArrayList<>()` 仍然可以 add。要真正不可变，字段类型本身要不可变（String、Integer、record、List.of 的结果）。

### 3.7 内部类与匿名类

内部类（Inner Class，定义在另一个类内部的类）按位置与是否持有外部实例分四种：

| 形态 | 声明方式 | 持有外部实例 | 典型场景 |
| --- | --- | --- | --- |
| 成员内部类 | `class Inner {}` | 是 | 需要读写外部实例字段的辅助对象 |
| 静态嵌套类 | `static class Nested {}` | 否 | 构建器 Builder、Map.Entry |
| 局部内部类 | 方法体内 `class X {}` | 是 | 极少使用 |
| 匿名内部类 | `new Runnable() {...}` | 是 | 一次性的接口实现 |

```java
int threshold = 100;                                  // 必须 effectively final 才能被捕获
Comparator<String> byLen = new Comparator<>() {       // 匿名类：可写多个方法、可带状态
    @Override public int compare(String a, String b) { return a.length() - b.length(); }
};
Runnable task = () -> System.out.println("lambda 只能实现函数式接口: " + threshold);
```

匿名类在 Java 8 之后大部分场景被 lambda 取代，因为它只能实现函数式接口（Functional Interface，只有一个抽象方法的接口）。匿名类仍然不可替代的地方：需要实现多个方法、需要持有可变状态、需要覆盖接口之外的父类方法。

最需要警惕的是成员内部类与匿名类都隐式持有外部类引用（Outer.this）。把这个实例放进长生命周期的容器——静态 Map、线程池队列、监听器列表——外部类对象就永远无法被回收，这是后端与 Android 中最常见的泄漏形态之一。排查手段是把监听器改注册为静态嵌套类，或只把需要的字段作为参数传进去，而不是把整个外部对象拖进去。

### 3.8 枚举：类型安全的常量

枚举（Enum，用 enum 声明、实例在编译期固定且数量有限、天然线程安全的类）是替代 int 常量的标准做法。它是完整的类：可以带字段、构造器、方法、实现接口，只是实例由 JVM 在类初始化时一次性创建并保证唯一。

```java
public enum OrderStatus {
    CREATED("待支付"), PAID("已支付"), SHIPPED("已发货"), CLOSED("已关闭");

    private final String label;
    OrderStatus(String label) { this.label = label; }   // 构造器隐式 private
    public String label() { return label; }
    public boolean canShip() { return this == PAID; }   // 枚举比较用 ==，安全且快

    private static final Map<String, OrderStatus> BY_NAME =
            Arrays.stream(values()).collect(Collectors.toUnmodifiableMap(Enum::name, e -> e));
    public static OrderStatus of(String name) {
        OrderStatus s = BY_NAME.get(name);
        if (s == null) throw new IllegalArgumentException("未知状态: " + name);
        return s;
    }
}
```

四个必须知道的点。第一，`values()` 每次调用都返回数组副本，在高频路径上反复调用会持续分配对象，应缓存成静态 Map。第二，`valueOf` 找不到时抛 IllegalArgumentException，不是返回 null，别写成「判空即安全」。第三，`ordinal()` 是声明顺序，可以拿来做数组下标，但绝不能存进数据库：任何人插入一个常量，历史数据的含义就整体错位，持久化要用 `name()`。第四，枚举的序列化被 JVM 特殊处理，反序列化得到的仍是同一个实例，所以「单元素枚举」是《Effective Java》推荐的单例写法，天然防反射与反序列化破坏。

switch 对枚举编译成跳转表，性能好，但传 null 会直接 NPE，进入 switch 之前要判空。

### 3.9 record：不可变数据载体

record（记录类，Java 16 正式引入，用一行声明不可变数据载体的语法糖）专治「一个类只有字段、构造器、getter、equals、hashCode、toString」这类样板代码。

```java
public record Point(int x, int y) {
    public Point {                                  // 紧凑构造器：只写校验，赋值自动补
        if (x < 0 || y < 0) throw new IllegalArgumentException("坐标不能为负");
    }
    public int manhattan() { return x + y; }        // 可以追加自己的方法
}

Point p = new Point(3, 4);
System.out.println(p.x() + "," + p.y() + "," + p.manhattan());  // 3,4,7
System.out.println(p.equals(new Point(3, 4)));                  // true，自动生成的 equals
```

编译器自动生成：private final 字段、全参构造器、与组件同名的访问器（是 `x()` 而不是 `getX()`）、基于全部组件的 equals 与 hashCode、可读的 toString。record 隐式 final，既不能被继承也不能继承别的类（可以实现接口）。

| 特性 | record | 普通类 |
| --- | --- | --- |
| 字段 | 组件即 private final，不允许额外实例字段 | 任意声明 |
| 不可变程度 | 浅不可变，引用内容仍可变 | 自行控制 |
| 访问器命名 | `x()` | 自定，通常是 getX() |
| 继承 | 隐式 final | 可继承、可被继承 |
| 适用场景 | DTO、值对象、方法返回多值、Map 的 key | 需要可变状态或框架要求无参构造 |

两个坑。其一是浅不可变：`record Order(List<Item> items)` 里的 items 仍可被外部修改，必须在紧凑构造器里写 `items = List.copyOf(items)`。其二是框架兼容：Jackson 2.12 起支持 record，MyBatis 需要能定位全参构造并开启构造器映射，Spring 的某些旧版本在把 record 当请求体时要求额外配置；对接老框架前先确认版本，否则会在启动或反序列化时报「找不到无参构造器」。

### 3.10 组合优于继承

组合（Composition，用一个字段持有另一个对象、把职责委托给它，表达 has-a）与继承（is-a）是复用的两条路。继承是白盒复用：子类依赖父类的实现细节，父类一调整，子类就可能悄悄坏掉；而且父类方法内部调用另一个可重写方法时，实际行为取决于子类实现，这叫自调用（self-use）问题。

经典反例：用继承给 HashSet 加计数。

```java
class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;
    @Override public boolean add(E e) { addCount++; return super.add(e); }
    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();            // 以为只加 2
        return super.addAll(c);          // HashSet.addAll 内部又调用 add，被重复计数
    }
}
// addAll(List.of("a","b")) 之后 addCount 是 4，不是 2
```

改成组合加转发，行为就完全可控：

```java
public class InstrumentedSet<E> {
    private final Set<E> inner;                  // 组合：只依赖 Set 接口
    private int addCount = 0;
    public InstrumentedSet(Set<E> inner) { this.inner = inner; }
    public boolean add(E e) { addCount++; return inner.add(e); }
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();                    // 结果稳定为 2
        return inner.addAll(c);
    }
    public int addCount() { return addCount; }
}
```

判断标准只有一句话：只有当子类确实「是一种」父类、并且父类在设计上就为继承留了扩展点（方法有文档、允许重写、不依赖内部自调用）时才用继承；只是为了复用几行代码，一律改成组合加接口委托。工程上常见的折中是：继承抽象类拿默认实现，但对外只暴露接口类型，把继承藏在实现内部。

### 3.11 常见坑与排查顺序

排查面向对象问题最有效的手段不是反复读代码，而是最小复现：把可疑的类单独抽出来，写一个 main 方法加打印，确认构造顺序与字段实际取值，再逐条对照 3.2 的初始化顺序表与 3.4 的重写规则表。这类事故绝大多数不是「语法写错」，而是「初始化时机」与「相等性契约」两处认知偏差。以下四条覆盖了日常八成的情况。

1. 构造器里调用可重写方法。现象是子类字段读出 null 或 0。根因是父类构造器执行时子类字段尚未初始化。对策是把调用移到 init() 或工厂方法，父类构造器只初始化自己的字段。
2. 重写了 equals 却没重写 hashCode。现象是对象明明相等，放进 HashSet 后 contains 返回 false、HashMap 里 get 不到。根因是两者必须遵守同一份契约：equals 相等则 hashCode 必须相等。对策是用 IDE 或 record 生成，或直接用 `Objects.hash(...)`。
3. 用 == 比较包装类型、String 或枚举。Integer 在 -128 到 127 之间有缓存池，小数值 `==` 为 true、超出范围为 false，测试环境与生产环境的数值差异会让这个 bug 时隐时现。引用类型一律用 equals 或 `Objects.equals`，只有枚举可以放心用 ==。
4. 把可变对象放进 HashSet 或作为 HashMap 的 key，之后又修改它。哈希值变了，桶位置也变了，再 get 就是找不到。对策是 key 使用不可变类型（String、Integer、record），或在放入后不再修改。

## 4. 集合框架

### 4.1 两条分支：Collection 与 Map

集合框架（Collections Framework，JDK 中用来存放和操作一组对象的接口与实现的总称）是日常代码里出现频率最高的库。它的顶层分成两条互不相交的分支：Collection 管「一组单个元素」，Map 管「键到值的映射」。Map 不继承 Collection 不是设计疏漏——Collection 的抽象是「一堆元素，可以 add、可以迭代」，Map 的抽象是「一张查找表，按 key 取 value」，把 Map 塞进 Collection 会让 `add(E)` 和 `iterator()` 失去语义。需要把 Map 当集合使用时，通过 `keySet()`、`values()`、`entrySet()` 拿到三个视图，它们才是 Collection 的子类型；其中 `entrySet()` 一次遍历就能同时取到 key 和 value，是遍历 Map 唯一推荐的方式。

```text
Iterable
└── Collection
    ├── List          有序、可重复、可按索引访问
    │   ├── ArrayList
    │   ├── LinkedList（同时实现 Deque）
    │   └── Vector（已不推荐，方法级 synchronized）
    ├── Set           不重复
    │   ├── HashSet
    │   ├── LinkedHashSet
    │   └── SortedSet ── NavigableSet ── TreeSet
    └── Queue         队列语义
        ├── Deque ── ArrayDeque / LinkedList
        ├── PriorityQueue
        └── BlockingQueue ── ArrayBlockingQueue / LinkedBlockingQueue

Map（不继承 Collection）
├── HashMap
├── LinkedHashMap
├── TreeMap（实现 SortedMap / NavigableMap）
├── Hashtable（已过时，全表 synchronized）
├── ConcurrentHashMap
└── EnumMap
```

| 接口 | 顺序 | 重复元素 | null | 线程安全 | 典型实现 |
| --- | --- | --- | --- | --- | --- |
| List | 插入顺序，有索引 | 允许 | 允许 | 否 | ArrayList |
| Set | HashSet 无序、LinkedHashSet 插入序、TreeSet 排序 | 不允许 | HashSet 允许一个 null | 否 | HashSet |
| Queue / Deque | 入队顺序；PriorityQueue 按优先级 | 允许 | ArrayDeque 不允许 null | 否 | ArrayDeque |
| Map | HashMap 无序、LinkedHashMap 插入或访问序、TreeMap 按键排序 | 键不允许重复 | HashMap 允许一个 null 键 | 否 | HashMap |
| ConcurrentMap | 无序 | 键不允许重复 | 键值都不允许 null | 是 | ConcurrentHashMap |

写代码时声明成接口类型（`List<String> list = new ArrayList<>();`）而不是实现类型，是为了让「换实现」只改一行：需要有序去重时把右侧换成 `TreeSet`，需要线程安全时换成 `CopyOnWriteArrayList`，调用方的方法签名不用动。

### 4.2 ArrayList 与 LinkedList：数组和双向链表

ArrayList 内部就是一个 `Object[] elementData`。默认构造 `new ArrayList<>()` 并不分配数组，而是指向一个共享的空数组，第一次 `add` 时才扩到 10；满了之后按 1.5 倍扩（`oldCapacity + (oldCapacity >> 1)`），再用 `Arrays.copyOf` 把旧数据整段复制过去。1.5 倍是折中值：扩容太频繁会反复复制，一次翻倍又容易留下大块回收不掉的内存。这条实现细节解释了为什么「已知规模就传初始容量」是有效优化——预分配把多次复制换成一次分配。

LinkedList 是双向链表，每个节点 `Node<E>` 持有 `item`、`prev`、`next` 三个引用，外加头尾指针 `first`/`last`。`get(i)` 并不像很多人以为的那样直接跳到第 i 个，而是做了一次折半优化：`i < size / 2` 从头走，否则从尾走，但仍然是 O(n)。

| 操作 | ArrayList | LinkedList |
| --- | --- | --- |
| `get(i)` 随机访问 | O(1) | O(n)，从头或尾走 i 步 |
| `add(E)` 尾部追加 | 均摊 O(1)，扩容那次 O(n) | O(1) |
| `add(0, E)` 头部插入 | O(n)，整个数组后移 | O(1)，改两个指针 |
| `add(i, E)` 中间插入 | O(n-i)，搬移后半段 | O(n) 定位 + O(1) 改指针 |
| `remove(i)` | O(n-i) 搬移 | O(n) 定位 |
| 内存占用 | 每元素一个引用（4 或 8 字节）+ 预留空槽 | 每节点多两个引用，约为数组的 3 倍以上 |
| 遍历性能 | 连续内存，缓存友好 | 指针跳转，缓存命中差 |

```java
import java.util.ArrayList;
import java.util.LinkedList;
import java.util.List;

public class ListCost {
    public static void main(String[] args) {
        int n = 10_000;                                  // 想跑得快就用 1 万；10 万时第二个计时会跑很久
        List<Integer> array = new ArrayList<>(n);        // 预分配，省掉扩容复制
        List<Integer> linked = new LinkedList<>();

        long t0 = System.nanoTime();
        for (int i = 0; i < n; i++) array.add(0, i);     // 头插：每次搬移整个数组，O(n^2)
        long t1 = System.nanoTime();
        for (int i = 0; i < n; i++) linked.add(0, i);    // 头插：只改两个指针，O(n)
        long t2 = System.nanoTime();
        System.out.printf("ArrayList 头插 %,d ns%nLinkedList 头插 %,d ns%n", t1 - t0, t2 - t1);

        t0 = System.nanoTime();
        for (int i = 0; i < n; i++) array.get(i);
        t1 = System.nanoTime();
        for (int i = 0; i < n; i++) linked.get(i);       // 每次定位都走一遍链，整体 O(n^2)
        t2 = System.nanoTime();
        System.out.printf("ArrayList 遍历 %,d ns%nLinkedList 遍历 %,d ns%n", t1 - t0, t2 - t1);
    }
}
```

默认选 ArrayList。LinkedList 的优势只在「已经持有节点位置」（迭代器遍历中插入删除）和纯头尾操作上，而要当队列或栈用时，`ArrayDeque` 用环形数组实现，比 LinkedList 更快也更省内存，所以 LinkedList 在业务代码里的合理出场机会比教科书暗示的少得多。

### 4.3 HashMap：桶数组、链表与红黑树

HashMap 的结构是「桶数组 + 链表 + 红黑树」。数组的每个格子是一个桶，key 先经哈希算出桶下标，落进同一个桶的元素用链表串起来，链表过长时转成红黑树（Red-Black Tree，一种自平衡二叉查找树，查询稳定 O(log n)）。

| 参数 | 默认值 | 含义 |
| --- | --- | --- |
| `DEFAULT_INITIAL_CAPACITY` | 16 | 桶数组初始长度，必须是 2 的幂 |
| `DEFAULT_LOAD_FACTOR` | 0.75 | 负载因子，元素数超过「容量 × 0.75」就扩容 |
| `TREEIFY_THRESHOLD` | 8 | 单桶链表长度达到 8 时尝试树化 |
| `UNTREEIFY_THRESHOLD` | 6 | 树节点数降到 6 时退回链表 |
| `MIN_TREEIFY_CAPACITY` | 64 | 数组长度不足 64 时不树化，改为扩容 |
| `MAXIMUM_CAPACITY` | 1 << 30 | 数组长度上限 |

```java
// JDK 8 的哈希扰动与寻址，面试和排错都绕不开这两行
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// 实际下标：n 为数组长度，必为 2 的幂
int index = (n - 1) & hash;   // 等价于 hash % n，但只有位运算
```

为什么要把 hashCode 的高 16 位异或到低位：数组长度通常远小于 2^16（默认 16，上限 2^30），`(n - 1) & hash` 只用得到 hash 的低几位，高位信息被直接丢弃。如果对象的 `hashCode` 低位分布很差（自己手写 `return a * 31 + b` 很容易出现），冲突就会全挤在少数桶里。一次位移加一次异或的代价，换来高位也参与定位，这是「扰动函数（Hash Spreading）」的全部动机。

为什么容量必须是 2 的幂：只有 n 为 2 的幂时，`n - 1` 的二进制低位全为 1，`(n - 1) & hash` 才能均匀铺满每个桶且等价于取模；否则总会有些桶永远落不到。所以 `new HashMap<>(1000)` 里的 1000 会被 `tableSizeFor` 向上取整成 1024，传进去的数字只是「期望容量」而不是真实数组长度。

`put` 的流程值得背下来，排查问题时全靠它：

1. 数组为空则先 `resize()` 分配（默认 16），并把阈值设为「容量 × 0.75」。
2. 用扰动后的 hash 算出下标，桶为空就直接放一个新节点。
3. 桶首节点的 hash 相等且 `equals` 为 true，覆盖它的 value 并返回旧值。
4. 首节点是树节点，走红黑树插入。
5. 否则遍历链表尾插；链长达到 8 且数组长度达到 64 就树化，长度不足 64 只扩容不树化。
6. `++size` 超过阈值则扩容，容量与阈值同时翻倍。

树化要同时满足两个条件：链表长度到 8，数组长度到 64。后者常被忽略——数组还小时，`treeifyBin` 会改成调用 `resize()`，因为扩容可能把那批冲突节点拆到两个桶里，比建一棵树更划算。阈值取 8 有概率依据：hash 均匀时一个桶内节点数达到 8 的概率约为 6e-8，正常业务几乎不会触发；一旦真的树化，说明要么数据规模极大，要么 `hashCode` 有明显缺陷（大量 key 落到同一个值）。

扩容时容量翻倍，JDK 8 不需要重新计算 hash，只看多出来的那一位：

```java
// JDK 8 HashMap.resize() 拆桶的核心逻辑（简化）
if ((e.hash & oldCap) == 0) {
    // 多出的那一位是 0：索引不变，留在原位置 j
} else {
    // 是 1：索引变成 j + oldCap
}
```

容量翻倍后，`n - 1` 只多了一个为 1 的高位，所以每个节点的新位置只有两种可能：原地下标 j，或 j + 原容量。JDK 8 用尾插保持链表相对顺序；JDK 7 是头插，并会重新计算 `indexFor`，这个差异在并发场景下后果完全不同。

负载因子 0.75 是空间与冲突的折中：调到 0.5，冲突更少、查询更快，但一半桶是空的，内存占用和扩容次数都上去了；调到 0.9，空间利用率高，链表变长、查询退化，也更容易触发树化。0.75 配合 2 的幂容量，能让绝大多数桶里只停 0 到 1 个节点，这是业务代码不该去改的默认值——真需要动它，通常说明该换数据结构了。

初始容量的正确算法是「元素数 ÷ 0.75 再向上取整」，而不是直接填元素数：

```java
Map<String, Integer> m1 = new HashMap<>(1000);              // 实际容量 1024，装到 769 个就扩容
Map<String, Integer> m2 = new HashMap<>(1000 * 4 / 3 + 1);  // 1334 -> 2048，可装 1536 个才扩容
```

`HashMap` 允许一个 null 键（固定落在 0 号桶），也允许任意多个 null 值；它不是线程安全的，任何「一边写一边读」的用法都要换实现。

### 4.4 并发写 HashMap 会发生什么

这是 Java 里最经典的生产事故之一，两个 JDK 版本的表现完全不同。

JDK 7 会死循环。扩容的 `transfer` 用头插把旧数组的节点搬到新数组，多线程同时扩容时，两个线程对同一批节点的 `next` 指针互相覆盖，链表被接成环。此后任何落到这个桶的 `get` 都会沿着环一直走下去，永远遇不到 `next == null`：CPU 单核打满、线程不返回、不抛异常、日志无输出，重启后一切正常。

JDK 8 改成尾插后不会再成环，但依然不安全：两个线程同时 `put` 到同一个空桶，都读到 null 并各自创建节点，后写的覆盖先写的，一条数据静默消失；`++size` 不是原子操作，最终 size 偏小；扩容期间别的线程可能读到 null。症状从「服务卡死」变成「偶尔少一条数据、size 对不上」，反而更难查。

排查死循环的标准步骤（Linux 现场）：

1. `top -Hp <pid>` 找出占满 CPU 的线程号（TID）。
2. `printf '%x\n' <tid>` 把十进制 TID 转成十六进制。
3. `jstack <pid> | grep -A 30 'nid=0x<hex>'` 打印该线程的栈。
4. 栈顶是 `java.util.HashMap.getEntry` 或 `HashMap.get`、线程状态 RUNNABLE 且栈深固定不变，基本可判定成环。
5. 处置：把该字段换成 `ConcurrentHashMap`，重启恢复；`jmap -histo:live <pid>` 可以顺带确认哪些大 Map 是嫌疑人。

| 场景 | 该用什么 | 说明 |
| --- | --- | --- |
| 多线程读写同一个映射 | `ConcurrentHashMap` | 桶级锁 + CAS，读不加锁 |
| 一次性构建、之后只读 | `Map.copyOf` 或 `Collections.unmodifiableMap` | 不可变对象天然线程安全 |
| 老代码只能小改 | `Collections.synchronizedMap` | 全表一把锁，复合操作仍需外部 synchronized |
| 本地缓存、要淘汰策略 | Caffeine | 自带 LRU/LFU 与过期，比手写可靠 |

`Collections.synchronizedMap` 最容易被误用：它只保证单个方法原子，`if (!map.containsKey(k)) map.put(k, v);` 这种「先查后写」的两步组合依然会出问题，必须自己用 `synchronized (map)` 包住，或者直接用 `ConcurrentHashMap.putIfAbsent`。同理，包装后的 `for-each` 遍历仍会抛 CME，因为 `iterator()` 本身没有加锁，整个循环必须手动写进 `synchronized (map)` 块里。

### 4.5 HashSet、LinkedHashMap、TreeMap 的适用场景

HashSet 内部就是一个 HashMap，元素当 key，value 是一个占位常量：

```java
private transient HashMap<E, Object> map;
private static final Object PRESENT = new Object();   // 所有 value 共享这一个对象，省内存

public boolean add(E e) {
    return map.put(e, PRESENT) == null;               // put 返回 null 说明这个 key 之前不存在
}
```

所以 HashSet 的去重规则完全等于 HashMap 的 key 规则：`hashCode` 相同、`equals` 为 true 就算同一个元素；也因此它允许一个 null、不保证顺序、`add`/`contains` 平均 O(1)、非线程安全。`LinkedHashSet` 只是把内部的 HashMap 换成 LinkedHashMap。

LinkedHashMap 继承 HashMap，每个 Entry 多了 `before`/`after` 两个指针，额外维护一条双向链表。默认按插入顺序遍历；构造时传 `accessOrder = true`，每次 `get` 都会把节点移到链表尾部，于是链表头就是「最久未访问」。配合重写 `removeEldestEntry` 就得到一个 LRU 缓存：

```java
import java.util.LinkedHashMap;
import java.util.Map;

public class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LruCache(int capacity) {
        super(capacity, 0.75f, true);   // 第三个参数 true = 按访问顺序排序
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;        // 每次 put 后回调，超容量就淘汰链表头
    }

    public static void main(String[] args) {
        LruCache<String, Integer> cache = new LruCache<>(3);
        cache.put("a", 1);
        cache.put("b", 2);
        cache.put("c", 3);
        cache.get("a");                  // a 被移到队尾，顺序变成 b, c, a
        cache.put("d", 4);               // 超容量，淘汰最久未访问的 b
        System.out.println(cache.keySet());   // [c, a, d]
    }
}
```

TreeMap 用红黑树按键排序，`get`/`put` 是 O(log n)，换来的是有序遍历和范围查询能力：

```java
import java.util.TreeMap;

TreeMap<Integer, String> tm = new TreeMap<>();
tm.put(3, "c");
tm.put(1, "a");
tm.put(5, "e");
System.out.println(tm.firstKey());                       // 1
System.out.println(tm.floorKey(4));                      // 3，小于等于 4 的最大键
System.out.println(tm.ceilingKey(4));                    // 5，大于等于 4 的最小键
System.out.println(tm.subMap(1, true, 5, false));        // {1=a, 3=c}，左闭右开
```

TreeMap 的键不能为 null（自然排序时会调用 `compareTo` 而抛 NPE），键必须实现 `Comparable` 或在构造时传入 `Comparator`；`TreeSet` 同理。

| 需求 | 选择 | 理由 |
| --- | --- | --- |
| 单纯去重、不在意顺序 | HashSet | 平均 O(1)，最省事 |
| 去重且保留插入顺序 | LinkedHashSet | 内部就是 LinkedHashMap |
| 按键排序、要范围查询 | TreeMap / TreeSet | O(log n) 换有序，HashMap 给不了 |
| 按插入顺序遍历映射 | LinkedHashMap | 默认模式即可 |
| 实现 LRU 淘汰 | LinkedHashMap（accessOrder=true） | 重写 removeEldestEntry |
| 统计词频、分组 | HashMap + `merge` / `computeIfAbsent` | 一次调用完成判空与写入 |

```java
Map<String, Integer> counter = new HashMap<>();
for (String w : words) {
    counter.merge(w, 1, Integer::sum);        // 不存在放 1，存在则累加，避免 get 判空再 put
}

Map<String, List<String>> groups = new HashMap<>();
for (String w : words) {
    groups.computeIfAbsent(w.substring(0, 1), k -> new ArrayList<>()).add(w);
}
```

### 4.6 迭代器与 fail-fast

for-each 是语法糖：对 Collection 编译成 `iterator()` 加 `hasNext()`/`next()`，对数组编译成下标循环。所以「遍历时修改集合」出问题的根源在迭代器，而不在 for-each 语法本身。

fail-fast（快速失败，遍历过程中发现集合被结构性修改就立即抛异常）靠一个 `modCount` 计数器实现：`add`/`remove`/`clear` 每做一次结构性修改就把它加一；迭代器创建时记下 `expectedModCount`，每次 `next()` 比较两者，不相等就抛 `ConcurrentModificationException`。

三个必须知道的边界：它只在 `next()`/`remove()` 时检查，所以「改了但没抛」完全可能；`list.set(i, x)` 不改变元素个数，不算结构性修改，不会触发；fail-fast 是 bug 检测机制而不是并发控制手段，多线程下它既可能漏报也可能误报，不能拿它当同步方案。

遍历中删除只有两种正确写法：

```java
// 写法一：用迭代器自己的 remove，它会同步 expectedModCount
for (Iterator<String> it = list.iterator(); it.hasNext(); ) {
    if (it.next().startsWith("a")) {
        it.remove();
    }
}

// 写法二：JDK 8 的 removeIf，内部同样走迭代器
list.removeIf(s -> s.startsWith("a"));
```

错误写法是 `for (String s : list) { if (s.startsWith("a")) list.remove(s); }`：第一次删完 `modCount` 变了，下一轮 `next()` 立即抛异常。注意单线程也会抛——CME 不是并发专属。

还有一种不抛异常的错法，危害更大：按索引正序删除。`for (int i = 0; i < list.size(); i++) { if (cond) list.remove(i); }` 删掉第 i 个之后，后面的元素整体前移，原来的第 i+1 个落到 i 号位置，而下一轮 `i` 已经自增，等于凭空跳过一个元素。这种错不会报错，只会让结果「少删了几个」，排查时极难发现。要么改成倒序遍历 `for (int i = list.size() - 1; i >= 0; i--)`，要么直接用迭代器。

另外三个高频混淆点：`Arrays.asList(...)` 返回的是 Arrays 内部的固定长度列表，`add`/`remove` 抛 `UnsupportedOperationException`，但 `set` 可以；`List.of(...)` 是真正不可变的，连 `set` 都抛；`subList(a, b)` 是原列表的视图而不是拷贝，改视图等于改原列表，原列表发生结构性修改后再操作视图会抛 CME。

### 4.7 Comparable 与 Comparator

Comparable（自然排序接口，由类自己实现 `compareTo`，规定「这类对象之间怎么比」）只有一个方法；Comparator（外部比较器，把比较规则从类里抽出来）是函数式接口，同一个类可以有任意多个。类天生只有一种「自然顺序」时用 Comparable，需要多种排序规则或用第三方类（没法改源码）时用 Comparator。

两者的返回值约定完全一致：小于返回负数，等于返回 0，大于返回正数。`Collections.sort`、`TreeMap`/`TreeSet`、`Stream.sorted` 全部依赖这个约定，写反了不会报错，只会排错序或者在 TreeMap 里查不到数据。

```java
import java.util.Comparator;

public class Student implements Comparable<Student> {
    private final String name;
    private final int score;

    public Student(String name, int score) {
        this.name = name;
        this.score = score;
    }

    public String getName() { return name; }
    public int getScore() { return score; }

    @Override
    public int compareTo(Student o) {
        return Integer.compare(this.score, o.score);   // 不要写 this.score - o.score，会溢出
    }
}

// 一个类可以有多个 Comparator，按需组合
Comparator<Student> byScoreDesc = Comparator.comparingInt(Student::getScore).reversed();
Comparator<Student> byName = Comparator.comparing(Student::getName, Comparator.nullsFirst(String::compareTo));
Comparator<Student> combined = byScoreDesc.thenComparing(byName);   // 分数相同时再按名字
students.sort(combined);
```

`this.score - o.score` 是最常见的溢出坑：当分数是 `Integer.MIN_VALUE` 而另一个是正数时，相减溢出成负数，排序结果会莫名其妙地错乱。统一用 `Integer.compare` 或 `Comparator.comparingInt`。

与 equals 的一致性在 TreeSet/TreeMap 里是硬约束：它们用 `compareTo` 判断相等，返回 0 就视为重复元素，根本不会再看 `equals`。经典反例是 BigDecimal——`new BigDecimal("1.0").equals(new BigDecimal("1.00"))` 为 false（精度 scale 不同），但 `compareTo` 返回 0，于是同样的两个对象放进 HashSet 是两个元素、放进 TreeSet 只有一个。用 BigDecimal 当 HashMap 的键时取不到值，往往就是这个差异造成的。

排序稳定性也要留意：`List.sort`、`Collections.sort`、`Arrays.sort(Object[])` 用 TimSort，稳定（相等元素保持原有相对顺序）；`Arrays.sort(int[])` 等基本类型重载用双轴快排，不稳定。多关键字排序之所以能「先按 A 排、再按 B 排」，前提就是稳定排序。

### 4.8 并发集合概览

ConcurrentHashMap 是并发场景的主力。JDK 7 用 Segment（默认 16 段，每段继承 ReentrantLock）做分段锁；JDK 8 去掉了 Segment，改成「空桶用 CAS 插入 + 非空桶用 synchronized 锁住桶首节点」，锁粒度细到单个桶，并发度随容量自然增长。

它不允许 null 键和 null 值，这是并发语义逼出来的规定：`get(k)` 返回 null 时，调用方无法区分「键不存在」和「键存在但值是 null」，唯一的补救是先 `containsKey` 再 `get`，而这两步之间状态可能已被别的线程改掉，判断永远不可靠。为了消灭这个二义性，直接把 null 禁掉。

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.putIfAbsent("a", 1);                          // 不存在才放，原子
map.computeIfAbsent("b", k -> 0);                 // 不存在则计算并放入，原子
map.merge("a", 1, Integer::sum);                  // 存在则合并，原子
map.compute("b", (k, v) -> v == null ? 1 : v + 1);
```

| 实现 | 并发机制 | 适用场景 | 注意 |
| --- | --- | --- | --- |
| ConcurrentHashMap | CAS + 桶级 synchronized | 高并发键值读写 | 不允许 null；`size()` 是近似值 |
| CopyOnWriteArrayList | 写时复制整个数组 | 读多写极少，如监听器列表 | 写操作 O(n) 且内存翻倍；迭代器看不到新元素 |
| ConcurrentLinkedQueue | 无锁 CAS 链表 | 高并发队列 | `size()` 需要遍历，慎用 |
| ArrayBlockingQueue | 一把锁 + 两个 Condition | 有界生产者-消费者 | 容量固定，满时 `put` 阻塞 |
| LinkedBlockingQueue | put/take 两把锁 | 吞吐更高的阻塞队列 | 默认容量 `Integer.MAX_VALUE`，等于无界 |
| Collections.synchronizedMap | 全表一把锁 | 低并发、改造老代码 | 迭代时必须手动 `synchronized (map)` |

CopyOnWriteArrayList 的迭代器是创建那一刻的数组快照，遍历全程不会抛 CME，代价是读不到之后 `add` 的元素，且每次写都要复制整个数组——它适合「配置项、监听器」这类写操作一年几次的场景，绝不适合当普通列表用。

ConcurrentHashMap 的 `size()` 在并发下是估计值：JDK 8 用 `baseCount` 加一个 `CounterCell` 数组把计数打散到多个缓存行，累加时可能读到中间状态，所以结果只是近似；`mappingCount()` 返回 long，解决超大表的溢出问题，但同样是估计值。需要精确计数，只能在业务层另用一个 `LongAdder` 或数据库计数，不能指望 `size()`。

### 4.9 排查速查与常见坑

| 现象 | 原因 | 处置 |
| --- | --- | --- |
| 遍历时抛 ConcurrentModificationException | for-each 中直接调用 `list.remove` | 改用 `iterator.remove()` 或 `removeIf` |
| `get` 明明 put 过却返回 null | 键是可变对象，put 后字段被改，桶位置变了 | 键改用 String、Integer 或不可变 record |
| 服务 CPU 100% 且线程不退出 | JDK 7 HashMap 并发扩容成环 | `jstack` 确认栈顶为 `HashMap.get`，换 ConcurrentHashMap |
| 线上偶尔少数据、size 偏小 | JDK 8 HashMap 并发 put 覆盖、`size` 非原子 | 同上，换并发容器 |
| TreeSet 里去重了本该保留的对象 | `compareTo` 返回 0 被当成重复 | 检查比较器，必要时先比业务主键 |
| 扩容比预期早发生 | 初始容量按元素数填，没算负载因子 | 用 `(int)(size / 0.75f) + 1` |

四条踩中概率最高的坑按顺序记：

第一条，拿可变对象当 HashMap 的键。`put` 之后只要改动了参与 `hashCode` 计算的字段，这个键的桶位置就变了，再 `get` 一定找不到；更糟的是它连删都删不掉，因为 `remove` 同样按新哈希去找，这条 entry 会一直留在表里，成为缓慢增长的内存泄漏。键一律用不可变类型。

第二条，在 for-each 里 `remove`。单线程照样抛 CME，正确的两条路是 `iterator.remove()` 与 `list.removeIf(...)`。

第三条，以为 `new HashMap<>(1000)` 能装 1000 个。它的真实容量是 1024，装到第 769 个就扩容并重新分配、重新拆桶，本该避免的性能损耗一点没省。要装 n 个元素，容量按 `n / 0.75` 向上取整。

第四条，以为 HashMap 或 HashSet 的元素顺序是随机的、或者等于插入顺序。JDK 8 里顺序由哈希值决定，小整数键看起来像有序纯属巧合；需要顺序就显式选 LinkedHashMap（插入序）或 TreeMap（排序），否则换 JDK 版本或改数据量后顺序就会变。

## 5. 泛型、异常、注解与反射

Java 里最容易被写错、也最容易被反复追问的四块内容，恰好都卡在编译期与运行期的交界处：泛型在编译期提供类型安全，在运行期却被整体擦除；异常是唯一能中断正常控制流的机制，也是最常被悄悄吞掉的错误来源；注解本身不执行任何逻辑，只是一份贴在代码元素上的元数据；反射让程序在运行期查看并修改自身结构，Spring 的依赖注入、AOP、Jackson 的序列化几乎全部建立在它之上。这四件事单独看都不复杂，组合起来才是框架的真相与线上事故的源头。

### 5.1 泛型类、泛型方法与类型边界

泛型（Generics，把类型写成参数、由使用方在声明处代入、并由编译器完成类型检查的机制）的本质是「把类型检查从运行期提前到编译期」。没有泛型时，集合里装的都是 Object，取出来必须强转，类型错了只能等到运行期抛 ClassCastException。

```java
// 泛型类：T 由使用方在声明变量时确定
public class Box<T> {
    private T value;
    public void set(T value) { this.value = value; }
    public T get() { return value; }
}

// 泛型方法：类型参数写在返回值之前，与所在类是否泛型无关
public static <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;   // 有了 extends 边界，才能调用 compareTo
}

// 多边界：类必须写在第一位，其余用 & 连接接口
static <T extends Number & Comparable<T>> void sortDesc(List<T> list) { /* ... */ }
```

使用时的菱形推断让右侧不必重复类型：`Box<String> box = new Box<>();`。边界（Bound，用 extends 给类型参数限定的上界）的作用是双向的：既限制可传入的类型，也让编译器知道「T 至少具备哪些方法」，否则方法体里连 `a.compareTo(b)` 都调不了。

泛型的类型参数必须是引用类型，因此有三条绕不过去的硬限制，每一条都有明确原因：

1. 类型参数不能是基本类型。`Box<int>` 不合法，只能写 `Box<Integer>`。擦除后类型参数要被替换成 Object 之类的引用类型，Object 装不下 int，所以装箱无法避免。
2. 不能创建泛型数组。`new T[10]`、`new List<String>[10]` 都不合法。数组在运行期知道并检查自己的元素类型，泛型在运行期没有元素类型信息，两者直接矛盾。确需数组时只能 `(T[]) new Object[10]` 并接受 @SuppressWarnings，或传入 `Class<T>` 用 `Array.newInstance(clazz, 10)` 生成。
3. 静态成员不能使用类的类型参数。`static T instance;` 不合法，而 `static <T> T identity(T t)` 合法。原因是静态成员属于类本身，类只有一个，而 T 是每个实例化各自不同的，静态位置无法确定 T 是什么。

### 5.2 通配符与 PECS 原则

泛型不具备协变性：`List<Integer>` 不是 `List<Number>` 的子类型，尽管 Integer 是 Number 的子类。原因很实际——如果允许这种赋值，就能往一个实际装着 Integer 的列表里塞进 Double，运行时必然出问题。于是 Java 用通配符（Wildcard，用 ? 表示未知类型实参的写法）来表达「方向可控的灵活性」。

| 写法 | 名称 | 读出的类型 | 能写入的元素 | 典型用法 |
| --- | --- | --- | --- | --- |
| `List<?>` | 无界通配 | Object | 只能写 null | 只遍历、只统计，不关心元素类型 |
| `List<? extends Number>` | 上界通配 | 至少是 Number | 一律禁止（null 除外） | 求和、打印、作为数据源 |
| `List<? super Integer>` | 下界通配 | Object | Integer 及其子类 | 收集结果、作为数据目的地 |

上界通配为什么不能写：编译器只知道元素是 Number 的某个未知子类型，写入 Integer 可能与实际的 `List<Double>` 冲突，写入 Double 又可能与 `List<Integer>` 冲突，所以一律拒绝；而读出时至少能保证是 Number，是安全的。

PECS（Producer Extends, Consumer Super：生产者用 extends，消费者用 super）是一条记忆规则——如果参数是给你提供数据的，用 extends；如果参数是接收你的数据的，用 super。JDK 的 `Collections.copy` 是标准范例：

```java
// 源只被读取（生产者）→ extends；目标只被写入（消费者）→ super
public static <T> void copy(List<? super T> dest, List<? extends T> src)
```

```java
List<Integer> ints = Arrays.asList(1, 2, 3);
List<Number> nums = new ArrayList<>();
nums.addAll(ints);            // OK：addAll 的形参是 Collection<? extends E>
// nums = ints;               // 编译错误：List<Integer> 不能赋给 List<Number>
```

一个容易忽略的细节：`List<Object>` 与 `List<?>` 不同。前者能往里 add 任何对象，后者除了 null 什么都加不进去；反过来 `List<?>` 可以接收 `List<String>`，`List<Object>` 不行。

### 5.3 类型擦除与桥接方法

类型擦除（Type Erasure，编译器生成字节码时把类型参数替换为它的最左边界、无边界时替换为 Object，并在必要处插入强制转换的机制）是理解 Java 泛型一切怪异行为的钥匙。擦除之后，`List<String>` 与 `List<Integer>` 在运行期是同一个类：

```java
System.out.println(new ArrayList<String>().getClass() == new ArrayList<Integer>().getClass()); // true
// if (obj instanceof List<String>) { }   // 编译错误：擦除后无法判断泛型实参
```

擦除带来的第二个后果是不能用类型参数区分重载，因为擦除后签名相同：

```java
// 编译错误：name clash，两个方法擦除后都是 print(List)
void print(List<String> list) {}
void print(List<Integer> list) {}
```

第三部分是桥接方法（Bridge Method，编译器为实现擦除后的父类型契约，在子类中自动生成的、带强制转换的合成方法）。看这段代码：

```java
import java.lang.reflect.Method;

public class ErasureDemo {
    static class StringBox implements Comparable<StringBox> {
        String v;
        StringBox(String v) { this.v = v; }
        @Override public int compareTo(StringBox o) { return v.compareTo(o.v); }
    }

    public static void main(String[] args) {
        for (Method m : StringBox.class.getDeclaredMethods()) {
            System.out.println(m + " bridge=" + m.isBridge() + " synthetic=" + m.isSynthetic());
        }
    }
}
```

输出包含两个方法（顺序由 JVM 决定）：

```text
public int ErasureDemo$StringBox.compareTo(ErasureDemo$StringBox) bridge=false synthetic=false
public int ErasureDemo$StringBox.compareTo(java.lang.Object) bridge=true synthetic=true
```

`Comparable<T>` 擦除后方法签名是 `compareTo(Object)`，而 StringBox 实现的是 `compareTo(StringBox)`。如果编译器不额外生成 `compareTo(Object)`，当代码用 `Comparable c = box; c.compareTo(other)` 做多态调用时就会找不到方法。桥接方法就是那个「接住 Object、强转为 StringBox、再转发」的中间层，它解释了为什么反射遍历方法时经常看到两个同名方法。

擦除并不等于泛型信息完全消失：声明层面的泛型签名被保存在 class 文件的 Signature 属性里，运行期可以读回，这正是各类 ORM 与序列化框架还原实体类型的依据。

```java
class BaseDao<T> {}
class UserDao extends BaseDao<User> {}

// 读取父类声明中的实际类型参数
ParameterizedType pt = (ParameterizedType) UserDao.class.getGenericSuperclass();
Class<?> entity = (Class<?>) pt.getActualTypeArguments()[0];   // class User
```

### 5.4 异常体系与 try-with-resources

异常（Exception，用来描述程序运行中出现、可以恢复或至少可以被记录与上报的问题，继承自 Throwable 的对象）分为三个分支，是否受编译器强制处理是划分它们的核心标准。

| 分类 | 基类 | 代表类 | 编译器强制处理 | 典型场景 |
| --- | --- | --- | --- | --- |
| 受检异常（Checked Exception） | Exception，不含 RuntimeException | IOException、SQLException、InterruptedException | 是，必须 catch 或声明 throws | 外部资源、可预期的失败 |
| 非受检异常（Unchecked Exception） | RuntimeException | NullPointerException、IllegalArgumentException、ClassCastException | 否 | 编程错误、参数校验 |
| 错误（Error） | Error | OutOfMemoryError、StackOverflowError、NoClassDefFoundError | 否 | JVM 与类加载层面的严重问题，通常无力处理 |

受检异常的初衷是强制调用方在编译期就面对「可能失败」，代价是污染方法签名、并在层层向上传递时被包装成信息更少的异常。工程上的常见取舍是：对外部资源失败用受检异常（如 IO），对参数与状态错误用非受检异常。

受检异常与接口演进之间还存在天然冲突：给一个已被大量实现的方法新增受检异常，等于让所有调用方编译失败。这也是 JDK 在 Stream 与 Lambda 相关 API 中几乎不声明受检异常的原因——函数式接口的抽象方法一旦声明 throws，就再也无法作为 lambda 的目标类型。可用的折中是：底层保留受检异常以强制处理，跨层边界处统一包装为非受检的领域异常，并由最外层集中记录与映射。

try-catch-finally 的行为有两个反直觉点。第一，finally 中修改局部变量不影响已经计算好的返回值：

```java
static int f() {
    int x = 1;
    try {
        return x;        // 先把 1 暂存为返回值
    } finally {
        x = 100;         // 只改了局部变量，返回值仍是 1
        // return 2;     // 若在此处 return，会覆盖 try 的返回值，甚至吞掉异常
    }
}
```

第二，手写 finally 关闭资源会引发异常覆盖：如果 try 块抛出业务异常，而 close() 也抛异常，后者的堆栈会盖掉前者，真正的根因彻底丢失。try-with-resources（TWR，自动关闭实现 AutoCloseable 的资源的语法，Java 7 引入）正是为解决这个问题而生：

```java
try (Connection conn = ds.getConnection();
     PreparedStatement ps = conn.prepareStatement(SQL)) {   // 资源必须是 AutoCloseable
    ps.setLong(1, userId);
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) { /* 处理结果 */ }
    }
} catch (SQLException e) {
    for (Throwable s : e.getSuppressed()) {                  // close 的异常挂在 suppressed 上
        log.warn("关闭资源失败", s);
    }
    throw new DataAccessException("查询用户失败", e);         // 保留 cause，堆栈不断链
}
```

TWR 的三个关键规则：资源按声明的逆序关闭；主异常保留在抛出位置，close 阶段的异常被抑制（suppressed）挂在主异常上而不覆盖它；被管理的变量隐含 final，Java 9 起可直接使用已声明且 effectively final 的变量。多类型同构处理用 multi-catch：`catch (IOException | SQLException e)`，此时 e 隐含 final，类型是两者的共同父类。

### 5.5 自定义异常与处理策略

自定义异常（Custom Exception，为表达业务语义而定义的异常类）应该携带足够的上下文，并且永远保留原始 cause：

```java
public class BizException extends RuntimeException {
    private final String code;

    public BizException(String code, String message) {
        super(message);
        this.code = code;
    }

    // 包装底层异常时必须传 cause，否则堆栈在这里断掉
    public BizException(String code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    public String getCode() { return code; }
}
```

三条经过检验的经验。第一，分层归口：DAO 层把 SQLException 包装成 DataAccessException（技术异常），Service 层抛 BizException（业务异常），Controller 层或全局异常处理器统一映射为错误码与 HTTP 状态码，其余各层不各自翻译异常。第二，禁止空 catch 与只打印不处理，也禁止 `throw new BizException("失败")` 这种丢掉 cause 的写法——线上看到的是最外层消息，排错靠的却是最内层堆栈。第三，不要把异常当流程控制：构造异常要填充堆栈（fillInStackTrace 遍历调用栈），在循环里靠异常判断「记录不存在」会让耗时上升一个数量级，改用返回 Optional 或判空分支。

一次真实排错的例子：日志里只有一行 `java.lang.NullPointerException`，没有任何堆栈行号，同一个方法其他时候却有完整堆栈。原因是 JVM 的快速抛出优化（`-XX:+OmitStackTraceInFastThrow`，默认开启）会在同一个隐式异常被高频抛出后，复用预分配的无堆栈异常对象。定位手段是启动参数临时加 `-XX:-OmitStackTraceInFastThrow` 复现，拿到行号后去查那一行的返回值判空；根因通常是循环里对可能为 null 的查询结果直接调用了方法。另外，Lambda 与 Stream 里的函数式接口不能声明受检异常，遇到 IOException 必须在 lambda 内部 try-catch 包装成非受检异常再抛出。

### 5.6 注解与元注解

注解（Annotation，以 @ 形式附着在类、方法、字段、参数等元素上、本身不改变程序行为、需要由编译器或运行期反射读取才生效的结构化元数据）。一句容易记错的话：注解自己什么也不做，@Transactional 能开事务是因为 Spring 读了它并生成了代理，不是因为注解本身有魔力。

元注解（Meta-annotation，专门用来修饰注解定义的注解）决定了自定义注解在什么时候、什么地方可见：

| 元注解 | 作用 | 关键取值与说明 |
| --- | --- | --- |
| @Retention | 保留到哪个阶段 | SOURCE（仅源码，如 @Override）、CLASS（写入 class 文件但反射不可见，默认值）、RUNTIME（运行期可反射读取） |
| @Target | 可标注的元素种类 | METHOD、FIELD、TYPE、PARAMETER、CONSTRUCTOR、ANNOTATION_TYPE 等，不写则几乎处处可标 |
| @Documented | 是否进入 javadoc | 标记型，无属性 |
| @Inherited | 子类是否继承父类的该注解 | 只对类注解生效，对接口实现、方法覆写不生效 |
| @Repeatable | 同一元素可否重复标注 | 必须同时提供一个容器注解 |

```java
@Retention(RetentionPolicy.RUNTIME)   // 少了 RUNTIME，反射永远读不到，这是注解失效的第一原因
@Target({ElementType.METHOD, ElementType.TYPE})
@Inherited
public @interface ApiLog {
    String value() default "";        // 属性只能是基本类型、String、Class、枚举、注解或其数组
    int slowMs() default 500;
}
```

注解的属性写法像方法，但本质是编译期常量：默认值必须是常量表达式，不能是 `new Date()` 这类运行期求值；属性类型不能是包装类之外的任意对象，也不能是泛型。SOURCE 级注解（@Override、@SuppressWarnings）由编译器消费，用于检查与消警；CLASS 级注解常见于字节码增强与 Lombok 这类注解处理器；只有 RUNTIME 级注解才能在运行期通过反射读取，框架的扩展点几乎全部建立在这一级上。

注解与 XML 外部配置的取舍也值得了解：注解的优势是就近、类型安全、能被 IDE 校验与重构，劣势是硬编码、改动后必须重新编译、无法对同一份代码表达两套配置；外部配置的优势是运行期可换、可集中审计，劣势是缺少编译期检查。许多框架两者都支持，常见的落地约定是「结构性配置用注解、环境相关参数用外部配置」。组合注解（把多个注解合并成一个使用，例如 @SpringBootApplication 由 @SpringBootConfiguration、@EnableAutoConfiguration、@ComponentScan 组合而成）也只能依赖 RUNTIME 保留与元注解机制才能被框架识别。

### 5.7 反射：Class 对象与动态调用

反射（Reflection，在运行期检查并操作类、字段、方法、构造器的能力，统一入口是 java.lang.Class 对象）。获取 Class 有三种方式，语义不同：`Service.class` 不触发初始化、编译期即可确定；`obj.getClass()` 依赖已有实例；`Class.forName("com.x.Service")` 会触发类的静态初始化，因此常被 JDBC 驱动注册这类场景使用。

框架使用反射的动机可以归为三类：一是把「字符串到代码」的映射建立起来，配置文件里的类名、MyBatis Mapper 接口方法与 SQL 的对应都属于这一类；二是绕过编译期的静态绑定做通用处理，例如把任意对象的字段逐个取出序列化成 JSON；三是补回泛型擦除丢失的信息，从字段的泛型声明里还原集合的元素类型。三类用途的共同前提都是「编译期并不知道具体类型」，也正因如此，反射代码无法被编译器校验，拼错方法名只能等到运行期才暴露。

```java
import java.lang.annotation.*;
import java.lang.reflect.*;

public class ReflectionDemo {
    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    @interface Timed {}

    static class Service {
        @Timed
        public String hello(String name) { return "hi, " + name; }
        private int secret = 42;
    }

    public static void main(String[] args) throws Exception {
        Class<?> clazz = Class.forName("ReflectionDemo$Service");           // 1. 拿到 Class
        Object svc = clazz.getDeclaredConstructor().newInstance();          // 2. 反射创建实例
        Method m = clazz.getMethod("hello", String.class);                  // 3. 按签名精确定位方法
        if (m.isAnnotationPresent(Timed.class)) {
            long t0 = System.nanoTime();
            Object r = m.invoke(svc, "world");
            System.out.println(r + " cost=" + (System.nanoTime() - t0) + "ns");
        }
        Field f = clazz.getDeclaredField("secret");                         // 4. 访问私有字段
        f.setAccessible(true);                                             // 不加会抛 IllegalAccessException
        System.out.println(f.getInt(svc));                                 // 42
    }
}
```

四个必须记住的细节：`getMethod` 只返回 public 且包含继承来的方法，`getDeclaredMethod` 返回本类声明的全部方法（含 private）但不含继承；基本类型参数必须写 `int.class` 而不是 `Integer.class`；`Class.newInstance()` 已被废弃，应使用 `getDeclaredConstructor().newInstance()`，后者会把构造器抛出的异常正常包装；反射调用中目标方法抛出的异常会被包成 InvocationTargetException，真实异常在 `getCause()` 里，这是反射排错中最常见的困惑点。

反射的性能代价来自每次调用的访问权限检查、参数装箱以及无法内联。下表是同一台机器上千万次调用的相对量级，用于建立直觉而非精确基准：

| 调用方式 | 相对耗时 | 说明 |
| --- | --- | --- |
| 直接调用 | 1x | JIT 内联后接近零额外开销 |
| `Method.invoke` | 10~50x | 每次做访问检查、可变参数装箱、难以内联 |
| 反射 + `setAccessible(true)` + 缓存 Method 对象 | 5~20x | 省掉查找与部分检查 |
| `MethodHandle` / `LambdaMetafactory` | 1~3x | 生成可被 JIT 内联的调用点，主流框架的做法 |

反射的风险同样明确：它绕过封装与 final 约束，可以改掉单例的私有实例；JDK 9 之后的模块系统默认封闭非导出包，对 JDK 内部类 `setAccessible(true)` 会抛 InaccessibleObjectException，需要 `--add-opens java.base/java.lang=ALL-UNNAMED`；反射调用在 IDE 中无法跳转与重构，编译期也检查不出方法名拼错；GraalVM native image 需要为反射目标提供配置，否则运行期找不到方法。

### 5.8 动态代理：JDK 与 CGLIB

动态代理（Dynamic Proxy，在运行期生成一个代理类，把对目标方法的调用统一转发给 InvocationHandler 处理的机制）是 AOP、事务、缓存、RPC 客户端、MyBatis Mapper 接口的共同底座。增强逻辑的织入时机有三种：编译期（用 AspectJ 编译器直接改字节码）、类加载期（javaagent 在类加载时改写字节码）、运行期（生成代理对象，Spring AOP 采用这一种）。运行期代理的好处是不需要特殊编译器与启动参数，代价是只能拦截经代理对象进入的外部调用。

```java
import java.lang.reflect.*;

public class JdkProxyDemo {
    interface UserService { String find(long id); }

    static class UserServiceImpl implements UserService {
        public String find(long id) { return "user-" + id; }
    }

    static class TimingHandler implements InvocationHandler {
        private final Object target;
        TimingHandler(Object target) { this.target = target; }

        @Override public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            long t0 = System.nanoTime();
            try {
                return method.invoke(target, args);       // 转发给真实对象
            } catch (InvocationTargetException e) {
                throw e.getCause();                       // 拆包，否则异常类型被统一成包装类型
            } finally {
                System.out.println(method.getName() + " cost=" + (System.nanoTime() - t0) + "ns");
            }
        }
    }

    public static void main(String[] args) {
        UserService target = new UserServiceImpl();
        UserService proxy = (UserService) Proxy.newProxyInstance(
                UserService.class.getClassLoader(),        // 类加载器
                new Class<?>[]{UserService.class},         // 必须提供接口，这是 JDK 代理的硬约束
                new TimingHandler(target));
        System.out.println(proxy.find(7L));
    }
}
```

JDK 动态代理的硬约束是「目标必须实现接口」，因为生成的代理类继承 Proxy 并实现指定接口，无法继承目标类。CGLIB（Code Generation Library，通过 ASM 生成目标类的子类并覆写方法来织入增强的字节码库，Spring 已将其重打包内置）走的正是继承路线，因此目标类不能是 final、被代理的方法不能是 final 或 private。

| 维度 | JDK 动态代理 | CGLIB |
| --- | --- | --- |
| 实现方式 | 生成实现同一组接口的类 | 生成目标类的子类并覆写方法 |
| 前提条件 | 目标类必须实现接口 | 目标类与被代理方法不能是 final |
| 代理对象类型 | 只能按接口使用，强转成实现类会抛 ClassCastException | 是目标类的子类，可按类类型使用 |
| 额外依赖 | JDK 内置，无需第三方 | 需要字节码库（Spring 内置） |
| Spring AOP 默认 | 目标有接口时使用 | 目标无接口时使用；Spring Boot 2.x 起 proxy-target-class 默认为 true，一律用 CGLIB |

动态代理的两个经典失效场景值得单独记住。一是自调用：在同一个类的方法 A 里用 `this.methodB()` 调用带 @Transactional 的方法 B，调用不经过代理对象，事务注解完全失效，解决办法是注入自身代理、拆到另一个 Bean，或改用 AspectJ 织入。二是 JDK 代理下按实现类注入：代理对象只实现了接口，用 `@Autowired UserServiceImpl` 会启动失败，应注入接口类型。此外，代理对象的 equals/hashCode/toString 也会被转发进 handler，实现时要么显式处理，要么委托给 target，否则会出现两个「相等」的代理互不相等这类怪现象。

### 5.9 常见坑

- 泛型只在编译期存在：`instanceof List<String>`、`new T()`、`catch (MyException<String>)` 全部不成立，运行期拿不到实参类型。需要在运行期使用类型时，显式传入 `Class<T>`，或用 `TypeReference`、`getGenericSuperclass()` 这类保留在 class 文件 Signature 属性里的声明信息。
- 注解不生效先查三处：`@Retention` 是否写成了 RUNTIME（默认 CLASS 读不到）、`@Target` 是否包含当前元素、以及调用是否被 `this` 自调用绕过了代理。
- catch 块的三条禁忌：空 catch、只 `printStackTrace()` 不落日志、抛新异常时丢掉 cause。统一写成 `throw new BizException(code, "可读消息", e)`，否则线上只有最外层消息，根因永久丢失。
- 反射相关的两个必查项：`Method.invoke` 抛出的 InvocationTargetException 必须 `getCause()` 拆包才能看到真实异常；JDK 17+ 对 JDK 内部类调用 `setAccessible(true)` 需要显式 `--add-opens`，否则直接抛 InaccessibleObjectException。

## 6. IO 与 NIO

程序与外部世界交换数据只有一条物理通路：字节。硬盘上的文件、网卡上的报文、终端上的按键，进入 JVM 后都是字节序列，而程序想用的是字符、对象、整数。IO（Input/Output，输入输出，指程序与文件、网络、内存等外部数据源之间搬运数据的过程）这一章要解决的，就是在这条通路上正确地搬运、编码、缓冲与并发。Java 提供了三套 API：`java.io` 的流式阻塞模型、`java.nio` 的缓冲区加通道模型、以及 Java 7 起在 `java.nio.file` 下的 `Path`/`Files` 工具类。选型顺序可以粗略记成：日常文件读写先用 `Files`，高并发网络连接用 NIO，连接数少、逻辑简单时用 BIO。

### 6.1 IO 体系的四个基类

`java.io` 包的类有几百个，但顶层只有四个抽象基类，按两个维度交叉分类：数据单位是字节还是字符，方向是输入还是输出。

| 基类 | 数据单位 | 方向 | 典型子类 |
| --- | --- | --- | --- |
| `InputStream` | 字节（8 位） | 读入程序 | `FileInputStream`、`ByteArrayInputStream`、`BufferedInputStream` |
| `OutputStream` | 字节（8 位） | 写出程序 | `FileOutputStream`、`ByteArrayOutputStream`、`BufferedOutputStream` |
| `Reader` | 字符（16 位 char） | 读入程序 | `FileReader`、`StringReader`、`BufferedReader` |
| `Writer` | 字符（16 位 char） | 写出程序 | `FileWriter`、`StringWriter`、`BufferedWriter` |

命名有稳定规律：前缀说明数据源或目的地（`File`、`ByteArray`、`Piped`），中间说明附加功能（`Buffered` 缓冲、`Data` 读写基本类型、`Object` 序列化、`Print` 格式化输出），后缀是基类名。看到 `BufferedInputStream` 就知道它是「给字节输入加缓冲」。

`java.io` 的设计是装饰器模式（Decorator Pattern，用包装对象在运行时给另一个对象叠加职责）。`new BufferedReader(new InputStreamReader(new FileInputStream(f), charset))` 三层包装分别负责「读字节」「解码成字符」「按行缓冲」，每层只做一件事，可以自由组合。理解了这一点，就不会去找「既能指定编码又能按行读文件的 FileReader」——它不存在，正确做法是把三层拼起来，或者直接用 `Files.newBufferedReader`。

那么什么时候用字节流、什么时候用字符流？判断标准只有一条：文件用文本编辑器打开能看懂，就用字符流；看不懂，就用字节流。图片、zip 包、class 文件、加密数据都没有「字符」这个概念，用字符流读它们会在解码阶段损坏内容。反过来，用字节流读文本不会报错，但会得到一半正确、一半乱码的结果——ASCII 范围内的英文数字完全正常，中文变成乱码，因为一个汉字在 UTF-8 里占三个字节，逐字节处理就是把它拆成三块。这种「部分正确」比彻底报错更难排查，所以文本必须走字符流。

四个基类的核心方法签名值得背下来，因为子类只是换了数据源，语义完全一致：`InputStream.read()` 返回 `0~255` 的 int，流末尾返回 `-1`；`read(byte[] b)` 返回实际读到的字节数；`Reader.read(char[] cbuf)` 返回实际读到的字符数。**返回 -1 才是结束，返回 0 不代表结束**（0 只可能出现在长度为 0 的数组或非阻塞通道上）。

### 6.2 编码：乱码的唯一来源

字符编码（Character Encoding，把字符与字节序列相互映射的规则集）是文本 IO 里所有乱码的根因。同一份中文，用 UTF-8 编码后每个汉字占 3 个字节，用 GBK 编码后占 2 个字节；字节流本身不带任何「我是什么编码」的标记，写的一方和解码的一方约定不同，屏幕上就会出现 `锟斤拷`、`????` 或 `�`。

字节与字符的转换发生在 `InputStreamReader` / `OutputStreamWriter` 里，也发生在 `String.getBytes()` 与 `new String(byte[])` 里。这四个位置**都不指定字符集时，会用平台默认字符集**：Windows 中文版在 JDK 18 之前默认 GBK，Linux 容器常常是 POSIX/C，JDK 18 起由 JEP 400 统一为 UTF-8。同一个 jar 在开发机正常、在容器里乱码，差别就在这。

```java
// 转码：从 GBK 文件读到 UTF-8 文件，显式指定两端字符集
try (Reader r = new InputStreamReader(new FileInputStream("gbk.txt"), StandardCharsets.GBK);
     Writer w = new OutputStreamWriter(new FileOutputStream("utf8.txt"), StandardCharsets.UTF_8)) {
    char[] buf = new char[4096];
    int n;
    while ((n = r.read(buf)) != -1) {
        w.write(buf, 0, n);   // 关键：必须写 n 个，写整个 buf 会把上次残留的字符一起写出去
    }
}
```

还有一个容易忽略的机制：`InputStream.read()` 每次只交付一个字节，如果按字节逐个转字符，中文字符必然在字节边界上被切开，屏幕上出现 `�`。`InputStreamReader` 之所以必须存在，是因为它在内部保留了「半个字符」的状态：缓冲区读到底时若发现某个字符的字节还没凑齐，会把这几个字节留到下一次读取再一起解码。手写的字节转字符代码总在中文字符上出错，缺的就是这份跨块状态。

排查乱码有固定顺序，不要靠猜。第一步，用十六进制看原始字节：`UTF-8` 的中文首字节落在 `0xE4~0xE9`，GBK 首字节落在 `0x81~0xFE` 且第二字节在 `0x40~0xFE`，`EF BB BF` 是 UTF-8 BOM，`FF FE` 是 UTF-16LE BOM。第二步，确认读取端用的字符集，而不是写入端声称的字符集。第三步，若文件带了 BOM，`Files.readString` 会把 BOM 解成 `\uFEFF` 放在字符串开头，JSON 解析器报「Unexpected character」就是这个原因，处理方式是在解析前判断 `text.startsWith("\uFEFF")` 并截掉首字符，或保存时选「UTF-8 无 BOM」。

`Files.readString` 与 `Files.writeString` 默认使用 UTF-8，不跟随 `-Dfile.encoding`，所以用它读一个 GBK 文件会直接抛 `MalformedInputException: Input length = 1`，而不是产出乱码——这个异常反而更容易定位问题。

### 6.3 缓冲流与关闭顺序

`FileInputStream.read()` 每调用一次就是一次系统调用。一个 100 MB 的文件逐字节读，意味着上亿次内核态与用户态切换，耗时从零点几秒涨到几分钟。缓冲流（Buffered Stream，在内存里攒一批数据再一次性读写，默认 8192 字节）就是为消除这个开销而存在的。

| 读取方式 | 系统调用次数（100 MB） | 耗时量级 | 内存占用 |
| --- | --- | --- | --- |
| `read()` 逐字节 | 约 1 亿次 | 数十秒到数分钟 | 极小 |
| `read(byte[8192])` 自建缓冲 | 约 1.3 万次 | 0.2 秒级 | 8 KB |
| `BufferedInputStream.read()` 逐字节 | 约 1.3 万次 | 0.3 秒级 | 8 KB |
| `Files.readAllBytes` | 少量 | 0.1 秒级 | 等于文件大小 |

结论：**逐字节读也要套 BufferedInputStream**，这样循环写法自然，性能损失很小；自建 `byte[]` 适合追求极致且代码可控的场景；`readAllBytes` 只用于小文件，100 MB 的文件直接占 100 MB 堆，并发几个就 OOM（OutOfMemoryError，内存溢出错误）。

写入端有更隐蔽的问题：`BufferedOutputStream` 把数据攒在内存里，不 `flush()` 也不 `close()`，程序退出时缓冲区里的数据直接丢失，文件长度为 0。`try-with-resources`（自动关闭资源的语法，退出代码块时按声明的逆序调用 `close()`）同时解决了「忘记关闭」和「关闭顺序」两件事：

```java
try (var out = new BufferedOutputStream(new FileOutputStream("app.log"))) {
    out.write("启动完成\n".getBytes(StandardCharsets.UTF_8));
}   // 退出时先 flush 再 close；若中途异常也会关闭，且关闭异常被抑制为 suppressed

// 关闭顺序的验证：包装流关闭会连带关闭被包装的流，但反过来不会
try (var fin = new FileInputStream("a.txt");
     var bin = new BufferedInputStream(fin)) {
    // 异常时先关 bin（含 flush），再关 fin
}
```

一个反直觉的细节：`System.out` 是 `PrintStream`，它把所有 `IOException` 内部吞掉，只置一个标记位。写日志到文件时若用 `PrintStream`，磁盘满、权限不足都不会抛异常，表现是「日志莫名少了」。判断方式是 `out.checkError()`，或者干脆别用 `PrintStream`，改用日志框架。

### 6.4 Files 与 Path：日常文件操作的首选

`Path`（路径的抽象，只描述位置，不要求文件存在）替换了 `java.io.File`；`Files`（静态工具类，提供全部文件系统操作）替换了 `File` 上的那些返回 boolean 的方法。核心区别是**错误处理方式**：`file.delete()` 失败只返回 `false`，不给原因；`Files.delete(path)` 失败抛 `NoSuchFileException` 或 `AccessDeniedException`，堆栈里写清是哪个路径出了什么问题。

```java
Path p = Path.of("data", "app.log");          // 分隔符由 API 处理，不要手拼 "/" 或 "\\"
Files.createDirectories(p.getParent());       // 逐级创建父目录，已存在不报错

// 追加写，显式指定字符集和打开模式
Files.writeString(p, "第一行\n", StandardCharsets.UTF_8,
        StandardOpenOption.CREATE, StandardOpenOption.APPEND);

// 小文件一次读完；大文件用流式读取，避免 readAllBytes 把整个文件塞进堆
try (var lines = Files.lines(p, StandardCharsets.UTF_8)) {
    lines.filter(l -> l.contains("ERROR")).forEach(System.out::println);
}   // Files.lines 持有文件句柄，必须关闭，否则句柄泄漏

// 原子替换：写临时文件再改名，避免其他进程读到写了一半的文件
Path tmp = Files.createTempFile(p.getParent(), "app", ".tmp");
Files.writeString(tmp, newContent, StandardCharsets.UTF_8);
Files.move(tmp, p, StandardCopyOption.REPLACE_EXISTING, StandardCopyOption.ATOMIC_MOVE);
```

相对路径还有一个必须知道的前提：`Path.of("config/app.yml")` 是相对 JVM 的工作目录（系统属性 `user.dir`）解析的，既不相对类文件所在目录，也不相对项目根目录。在 IDE 里运行与用 `java -jar` 启动，工作目录往往不同，同一个路径就一个找得到、一个抛 `NoSuchFileException`。定位这类问题的第一行代码是打印 `Path.of(".").toAbsolutePath().normalize()`，看解析结果到底落在哪里；要分发的程序不该依赖相对路径，配置文件用 `-Dapp.home=` 显式传入，classpath 内的资源用 `ClassLoader.getResourceAsStream` 读取，因为打进 jar 之后它已经不是文件系统里的一个路径了。

`Files` 的常用方法按用途分组：判断与查询用 `exists`、`size`、`isDirectory`、`getLastModifiedTime`；复制移动用 `copy`、`move`，跨文件系统时 `ATOMIC_MOVE` 会抛 `AtomicMoveNotSupportedException`，要捕获后降级为普通 `move`；遍历用 `Files.list`（只一层，返回 `Stream<Path>`，需关闭）与 `Files.walk`（递归，默认不跟随符号链接，可用 `FileVisitOption.FOLLOW_LINKS` 打开）；删除非空目录必须倒序，`Files.walk` 的结果要 `sorted(Comparator.reverseOrder())` 再逐个 `delete`。

`Files.lines` 与 `Files.list` 返回的流都属于「必须显式关闭」的资源，它们背后是打开的文件句柄或目录句柄。在长跑服务里漏关，几百次之后就会撞上 `Too many open files`（Linux 默认单进程 1024 个文件描述符），而堆内存看起来毫无异常。

### 6.5 序列化与 serialVersionUID

序列化（Serialization，把对象及其引用图转换成字节序列，以便写文件或在网络中传输）由 `Serializable` 标记接口开启，反序列化是其逆过程。它曾经是 Java 分布式通信的默认方案，现在只推荐用于短期的本地缓存；跨语言、跨版本、长期存储都应该用 JSON 或 Protobuf。

```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L; // 显式声明版本号
    private final String name;
    private transient String token;                  // transient 字段不写盘，反序列化后为 null

    public User(String name, String token) { this.name = name; this.token = token; }
}

// 写出：注意套缓冲流，否则每个字段一次系统调用
try (var oos = new ObjectOutputStream(new BufferedOutputStream(new FileOutputStream("user.bin")))) {
    oos.writeObject(new User("ann", "tk-123"));
}
```

`serialVersionUID` 是这条链路上最容易出事的参数。不显式声明时，JVM 会根据类名、字段、方法签名算出一个哈希值作为版本号；只要给类加一个字段、改一个方法修饰符，这个值就变。此时用旧字节流反序列化新类，会抛：

```
java.io.InvalidClassException: com.example.User; local class incompatible:
stream classdesc serialVersionUID = -1, local class serialVersionUID = 8234711...
```

显式写死 `serialVersionUID` 后，增删字段不会断开兼容，新增字段反序列化时为默认值（`null`/`0`）。但这只解决「能不能读」，不解决「语义是否正确」：删掉的字段所承载的业务含义需要自己迁移。

`Serializable` 还有四个必须知道的语义。静态字段不参与序列化，因为它属于类而不是对象。父类不可序列化时，反序列化会调用父类的无参构造器，父类没有无参构造器就会失败。反序列化**不执行本类的构造器**，所以单例、不可变对象、构造器里的校验逻辑全部被绕过，防护办法是写 `private Object readResolve()` 返回唯一的实例，或在 `readObject` 里重新校验。反序列化任意字节等同于「按字节流执行构造逻辑」，是历史上远程代码执行漏洞的主要来源，JDK 9 起可用 `ObjectInputFilter` 做类白名单：

```java
// 只允许反序列化白名单内的类，其余直接拒绝
var filter = ObjectInputFilter.Config.createFilter("com.example.dto.*;java.base/*;!*");
try (var ois = new ObjectInputStream(new FileInputStream("user.bin"))) {
    ois.setObjectInputFilter(filter);
    Object o = ois.readObject();
}
```

另外，JDK 16 起 record 类不会自动实现 `Serializable`，需要显式声明；并且 record 的序列化走 `ObjectStreamClass` 的专用路径，不依赖反射写字段。

### 6.6 NIO 的 Buffer 与 Channel

NIO（New IO，Java 1.4 引入的面向缓冲区与通道的 IO 模型）与 `java.io` 的根本区别在数据流向：流是单向的、逐字节推进的；NIO 的通道（Channel，可同时读写的双向数据管道）必须配合缓冲区（Buffer，一块带读写游标的内存区域）使用，一次读写一批。

`Buffer` 有三个必须区分的游标：`capacity` 是容量，创建后固定；`position` 是下一个读或写的位置；`limit` 是可读或可写的上界。所谓「切换读写模式」，就是调整 `position` 与 `limit`：

| 操作 | position | limit | 含义 |
| --- | --- | --- | --- |
| `allocate(16)` | 0 | 16 | 写模式，可写 16 字节 |
| `put(...)` 5 字节 | 5 | 16 | 已写 5 字节 |
| `flip()` | 0 | 5 | 读模式，可读 5 字节 |
| `clear()` | 0 | 16 | 回到写模式，丢弃未读数据 |
| `compact()` | 剩余长度 | 16 | 把未读数据移到开头，再进入写模式 |

```java
ByteBuffer buf = ByteBuffer.allocate(16);
buf.put("hello".getBytes(StandardCharsets.UTF_8)); // position=5
buf.flip();                                        // limit=5, position=0；漏掉这一步，下面读到 0 字节
byte[] out = new byte[buf.remaining()];
buf.get(out);                                      // out = "hello"
buf.clear();                                       // 复用前必须 clear 或 compact
```

`Channel` 的四个常用实现是 `FileChannel`（文件）、`SocketChannel`（TCP 客户端）、`ServerSocketChannel`（TCP 服务端）、`DatagramChannel`（UDP）。读写方法的返回值要按「可能不完整」处理，这是 NIO 与 IO 最大的思维差异：

```java
try (FileChannel ch = FileChannel.open(Path.of("a.bin"), StandardOpenOption.READ)) {
    ByteBuffer buf = ByteBuffer.allocate(8192);
    while (ch.read(buf) != -1) {        // 返回 -1 表示文件末尾，可能返回 0
        buf.flip();                     // 切换为读模式
        while (buf.hasRemaining()) {
            process(buf.get());         // 处理缓冲区里已读到的内容
        }
        buf.clear();                    // 清空后继续填
    }
}
```

还有一个性能参数：`ByteBuffer.allocateDirect(n)` 在堆外分配，`SocketChannel` 与 `FileChannel` 只有在 direct buffer 上才能避免一次额外的内存拷贝，所以网络框架普遍用堆外缓冲池。代价是分配与回收比堆内慢得多，且不受 GC 堆大小约束，`OutOfMemoryError: Direct buffer memory` 要用 `-XX:MaxDirectMemorySize` 限制。

### 6.7 内存映射与文件复制

内存映射（Memory-mapped File，用 `FileChannel.map` 把文件的一段映射进进程虚拟地址空间，读写像访问数组，由缺页中断负责把页从磁盘载入）是随机访问大文件最快的办法：不需要 `read`/`write` 系统调用，也不需要用户态缓冲区。

```java
try (FileChannel ch = FileChannel.open(Path.of("index.dat"),
        StandardOpenOption.READ, StandardOpenOption.WRITE)) {
    MappedByteBuffer m = ch.map(FileChannel.MapMode.READ_WRITE, 0, ch.size());
    int first = m.getInt(0);       // 按偏移随机读，像数组一样
    m.putInt(0, first + 1);        // 修改写回映射区
    m.force();                     // 强制刷盘；不调用则断电可能丢数据
}
```

三个限制必须记住：单次映射长度不能超过 `Integer.MAX_VALUE`（约 2 GB），更大的文件要按段多次 `map`；映射区的释放依赖 `Cleaner`，时机不确定，Windows 上映射未解除时文件删不掉、覆盖不了，表现为 `FileSystemException: 另一个程序正在使用此文件`；映射内存不计入 GC 堆，堆 dump 与 `-Xmx` 都看不到它。

文件复制有三种写法，差别在数据拷贝次数：

| 方式 | 用户态拷贝 | 适用场景 | 注意事项 |
| --- | --- | --- | --- |
| `Files.copy` | 由 JDK 选择最优实现 | 通用，代码最简 | 默认不覆盖，需 `REPLACE_EXISTING` |
| `FileChannel.transferTo` | 尽量零拷贝（`sendfile`） | 大文件、文件到套接字 | 单次传输有上限，必须循环 |
| `byte[]` 缓冲循环 | 两次 | 需要边复制边处理 | 缓冲 64 KB 起，注意 `write` 用实际长度 |

```java
// 方式一：一行完成，内部会选择合适的系统调用
Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING);

// 方式二：转移字节，循环处理单次上限，避免大文件一次传不完
try (FileChannel in = FileChannel.open(src, StandardOpenOption.READ);
     FileChannel out = FileChannel.open(dst, StandardOpenOption.CREATE,
             StandardOpenOption.WRITE, StandardOpenOption.TRUNCATE_EXISTING)) {
    long total = in.size(), pos = 0;
    while (pos < total) {
        long n = in.transferTo(pos, total - pos, out); // 返回实际传输字节数
        if (n <= 0) break;                             // 防止 n=0 时死循环
        pos += n;
    }
}
```

`transferTo` 转到另一个 `FileChannel` 时，能否真正零拷贝取决于操作系统与 JDK 实现；转到 `SocketChannel` 时在 Linux 上会走 `sendfile`，这正是静态文件服务器与消息队列发送大量数据的常用手段。

### 6.8 BIO、NIO、AIO 与网络 IO 模型

网络编程里「模型」指的是线程与连接的对应关系。BIO（Blocking IO，阻塞式 IO，读操作没数据就挂起线程直到有数据）、NIO（Non-blocking IO，非阻塞 IO，配合多路复用器由一个线程监听大量连接）与 AIO（Asynchronous IO，异步 IO，由操作系统完成读写后回调通知）是三代方案。

| 模型 | 线程与连接 | 阻塞点 | 连接数上限 | 典型代表 |
| --- | --- | --- | --- | --- |
| BIO | 1 连接 1 线程 | `accept`、`read` | 受线程数限制，数百 | `ServerSocket`、Tomcat 传统模式 |
| NIO | 1 线程管多连接 | `selector.select()` | 数千到数万 | Netty、Redis 客户端 |
| AIO | 回调线程池 | 无 | 理论最高 | `AsynchronousSocketChannel` |

Linux 上 NIO 的底层是 epoll；AIO 在 Linux 上长期由线程池模拟，JDK 22 起才有基于 io_uring 的实现，这也是 Netty 不采用 AIO 的原因——它的优势在 Linux 上并不成立。AIO 的写法是 `AsynchronousSocketChannel` 读写时传 `CompletionHandler` 或拿回 `Future`，调用线程立刻返回，结果在回调里交付；它回调嵌套深、异常路径多、调试困难，初学者不必急着用。

`Selector` 自身的状态不是线程安全的。一个线程阻塞在 `select()` 里时，另一个线程对同一个 `Selector` 调用 `register` 不会立即生效，还可能互相干扰。跨线程注册的标准做法是先 `selector.wakeup()` 打断阻塞的 `select`，让事件循环在下一轮统一处理注册任务——所有 NIO 框架里「单个事件循环线程加任务队列」的结构都来自这一条约束。另外 `select()` 可能返回 0（超时、被 `wakeup()` 唤醒、伪唤醒），循环必须容忍空转。

NIO 服务端的骨架是所有框架的共同起点，理解它才能读懂 `Selector` 的坑：

```java
try (ServerSocketChannel server = ServerSocketChannel.open();
     Selector selector = Selector.open()) {
    server.bind(new InetSocketAddress(8080));
    server.configureBlocking(false);                       // 必须非阻塞才能注册
    server.register(selector, SelectionKey.OP_ACCEPT);
    ByteBuffer buf = ByteBuffer.allocate(1024);
    while (true) {
        selector.select();                                  // 阻塞到有事件就绪
        Iterator<SelectionKey> it = selector.selectedKeys().iterator();
        while (it.hasNext()) {
            SelectionKey key = it.next();
            it.remove();                                    // 必须移除，否则同一事件被反复处理
            if (key.isAcceptable()) {
                SocketChannel client = server.accept();
                client.configureBlocking(false);
                client.register(selector, SelectionKey.OP_READ);
            } else if (key.isReadable()) {
                SocketChannel client = (SocketChannel) key.channel();
                buf.clear();
                int n = client.read(buf);                       // 非阻塞读，可能返回 0
                if (n == -1) { key.cancel(); client.close(); continue; } // -1 是对方关闭
                buf.flip();
                while (buf.hasRemaining()) client.write(buf);   // 简化版：未处理写不完的情况
            }
        }
    }
}
```

这段代码能跑，但离生产还差三处，每一处都是真实的故障来源。第一，`OP_WRITE` 不能一开始就注册：套接字发送缓冲区通常有空间，`OP_WRITE` 会一直就绪，`select` 立刻返回，CPU 空转到 100%。正确做法是只在 `write` 返回 0（缓冲区写满）时注册 `OP_WRITE`，写完立即取消。第二，非阻塞 `write` 可能只写出一部分，剩下的部分必须挂在连接上等下次可写，这就是各框架都要实现「写缓冲队列」的原因。第三，TCP 是字节流不是消息，一次 `read` 读到的可能是半个请求或两个半请求，必须自己定长度前缀或分隔符来做粘包拆包。

### 6.9 常见坑

- **`w.write(buf)` 写成全量**：`Writer.write(char[])` 会把整个数组写出去，包括上一次循环残留的字符，产物末尾出现重复内容或多余空字符。凡是从流里读到的 `n`，写出时一律 `write(buf, 0, n)`。
- **Buffer 忘记 `flip()`**：写完之后 position 停在末尾、limit 等于 capacity，此时直接 `get()` 读到的是未写入的空字节；`flip()` 把 limit 设为 position 再归零 position，是 NIO 里最高频的漏写。读之前 `flip`，读完 `clear`，写满之前 `compact`，三条记住就够了。
- **依赖平台默认字符集**：`new String(bytes)`、`new FileReader(f)`、`new InputStreamReader(in)` 都不指定编码，开发机（GBK）与容器（UTF-8）行为不同，乱码只在部署后出现。所有跨越「字节↔字符」边界的地方都显式传 `StandardCharsets.UTF_8`。
- **小文件 API 用在大文件上**：`Files.readAllBytes`、`Files.readString` 会按文件大小一次性申请数组，读一个 2 GB 的日志直接 OOM；`Files.lines` 反过来只占常量内存，但它是必须关闭的资源，漏关在长跑服务里最终表现为 `Too many open files`。
- **忘记 flush 与 close**：`BufferedOutputStream`、`BufferedWriter`、`PrintWriter` 都把数据留在内存，程序正常结束也可能因未 `close` 而丢内容，异常路径下更是必然丢失。统一用 `try-with-resources` 包住所有流，让关闭与异常处理脱钩。

## 7. 并发编程

并发（Concurrency，多个任务在同一时间段内交替推进）与并行（Parallelism，多个任务在同一时刻真正同时执行）经常被混着说：单核 CPU 上只能并发，多核才能并行。并发编程要解决的不只是「让代码跑得更快」，而是三件事：算得快（用满多核）、响应快（不阻塞主流程）、扛得住（限流与隔离）。代价是同一份数据被多个执行流交替读写之后，正确性不再由代码的书写顺序保证，而要由内存模型与同步机制保证。线程本身、锁与内存可见性、原子类与 AQS、线程池、异步编排、死锁排查，是这条链上依次要打通的六个环节。

### 7.1 线程的创建、状态与协作

进程（Process，操作系统分配内存与文件句柄等资源的单位）与线程（Thread，CPU 调度的最小单位）的关系是：一个进程里可以有多个线程，它们共享堆内存与方法区，各自独占虚拟机栈和程序计数器。共享堆是并发出问题的物理根源：两个线程拿到的是同一个对象的同一个字段，谁先写谁后写没有天然顺序。

创建线程有四种写法，实际生产只用第四种：

```java
// 1. 实现 Runnable：只描述任务，不绑定线程，推荐
Runnable task = () -> System.out.println("运行在 " + Thread.currentThread().getName());
new Thread(task, "worker-1").start();      // start 才是启动新线程，run 只是普通方法调用

// 2. 实现 Callable + FutureTask：能返回结果、能抛出受检异常
FutureTask<Integer> future = new FutureTask<>(() -> 1 + 1);
new Thread(future).start();
Integer sum = future.get();                // 阻塞直到任务算完

// 3. 继承 Thread：白占一个继承位，不推荐
// 4. 交给线程池：线程复用，可控并发度，生产环境唯一正确姿势
ExecutorService pool = Executors.newFixedThreadPool(4);
pool.execute(task);
pool.shutdown();
```

线程有六种状态，排查线上问题时看线程栈里的 `java.lang.Thread.State` 就是在看这张表：

| 状态 | 含义 | 典型进入方式 | 离开方式 |
| --- | --- | --- | --- |
| NEW | 已创建未启动 | `new Thread()` 之后 | 调用 `start()` |
| RUNNABLE | 可运行（含正在等 CPU 与等 IO） | `start()` 之后 | 任务结束或转入阻塞 |
| BLOCKED | 等监视器锁 | 抢 `synchronized` 锁失败 | 抢到锁 |
| WAITING | 无限期等待 | `wait()`、`join()`、`park()` | `notify()` 或被中断 |
| TIMED_WAITING | 限时等待 | `sleep(n)`、`wait(n)`、`join(n)` | 超时或被中断 |
| TERMINATED | 已结束 | `run()` 正常返回或抛异常 | —— |

两个必须记牢的区别：`Thread.sleep()` 不释放已持有的锁，`Object.wait()` 会释放；`interrupt()` 是协作式中断，它只把中断标志置为 true，并不会真的掐断线程，只有线程阻塞在 `sleep`、`wait`、`join` 上时才会抛 `InterruptedException` 并清除标志。老 API `stop()`、`suspend()` 会留下不一致的中间状态，早已废弃。

要优雅停止一个循环任务，正确写法是「可见的标志位 + 中断兜底」：

```java
public class Worker implements Runnable {
    private volatile boolean running = true;   // volatile 保证主线程的修改对工作线程可见

    @Override public void run() {
        while (running) {
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) { // 收到中断：恢复标志位，让上层知道被中断过
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    public void shutdown() { running = false; }
}
```

还有一个容易忽略的属性：守护线程（Daemon Thread，不阻止 JVM 退出、随用户线程一起结束的线程）。`new Thread(...)` 默认是非守护线程，必须在 `start()` 之前调用 `setDaemon(true)` 才生效，启动后再改会抛 `IllegalThreadStateException`。垃圾回收、心跳上报这类后台辅助任务适合设为守护线程；写文件、发消息这类必须做完的工作绝不能是守护线程，否则主线程一结束它就被强行终止。

### 7.2 可见性、有序性与 volatile

现代 CPU 有多级缓存，每个核心先改自己缓存里的副本，何时刷回主存不确定；编译器和 JIT 还会在不改变单线程语义的前提下重排指令。于是出现两种违反直觉的现象：可见性问题（一个线程改了值，另一个线程永远读不到）与有序性问题（代码写的顺序不是实际执行的顺序）。

下面这段代码在服务器模式下可能永远打印不出结果：

```java
public class VisibilityBug {
    private static boolean flag = false;    // 缺少 volatile
    private static int counter = 0;

    public static void main(String[] args) throws Exception {
        new Thread(() -> {
            while (!flag) { }               // JIT 可能把 flag 提升到寄存器，循环里不再读主存
            System.out.println("counter=" + counter);
        }).start();
        Thread.sleep(100);
        counter = 42;
        flag = true;                        // 主线程的写，子线程可能看不见
    }
}
```

`volatile` 提供两条语义：写入立即刷回主存、读取强制从主存读，因此可见性得到保证；同时禁止该变量读写与前后指令重排，因此有序性得到部分保证。它不给的正是初学者最容易误解的那一条：不保证原子性。`volatile int n; n++` 是「读—加—写」三步，两个线程交错执行照样丢更新。

这些保证在规范里被表述为 happens-before（先行发生，Java 内存模型对跨线程可见性的契约：A 先行发生于 B，则 A 的结果对 B 可见）。需要记住的六条规则：

- 程序顺序：同一线程内，前面的操作先行发生于后面的操作。
- volatile 规则：对 volatile 变量的写，先行发生于后续对该变量的读。
- 锁规则：解锁先行发生于后续对同一把锁的加锁。
- 线程启动：`start()` 之前的操作对子线程可见。
- 线程终止：线程内所有操作先行发生于其他线程从 `join()` 成功返回。
- 传递性：A 先行发生于 B、B 先行发生于 C，则 A 先行发生于 C。

最容易被写错的地方是双重检查锁（Double-Checked Locking，先判空再进锁再判空，为了避免每次都加锁的单例写法）：

```java
public class Singleton {
    private static volatile Singleton instance;   // 少了 volatile，可能发布构造未完成的对象
    public static Singleton getInstance() {
        if (instance == null) {                   // 第一次检查：绝大多数调用不加锁直接返回
            synchronized (Singleton.class) {
                if (instance == null) {           // 第二次检查：防止多个线程同时通过第一关
                    instance = new Singleton();   // 分配内存、初始化、赋值引用，后两步可能被重排
                }
            }
        }
        return instance;
    }
}
```

如果没有 `volatile`，`new` 的三步可能被重排成「先赋值引用、后初始化」，另一个线程在第一关读到非 null 就返回了一个字段全是默认值的半成品对象。

### 7.3 synchronized 与锁升级

`synchronized` 是 JVM 内置的互斥手段，三种用法锁的对象不同：修饰实例方法锁 `this`，修饰静态方法锁该类的 `Class` 对象，修饰代码块锁括号里指定的对象。它可重入（同一线程再次进入同一把锁的临界区不会把自己锁死），释放由 JVM 保证，即使抛异常也会退出监视器。

锁的粒度要贴在「必须一起成立」的最小范围上。转账这类操作的正确做法不只是加锁，还要固定加锁顺序，否则两个方向相反的转账会互相等锁：

```java
public class TransferService {
    public void transfer(Account from, Account to, long cents) {
        // 按 id 排序决定加锁顺序：A→B 与 B→A 都会先锁 id 小的那个，不会形成环路
        Account first  = from.id < to.id ? from : to;
        Account second = from.id < to.id ? to   : from;
        synchronized (first) {
            synchronized (second) {
                if (from.balance < cents) throw new IllegalStateException("余额不足");
                from.balance -= cents;          // 两次写在同一临界区内，对其他线程要么都可见要么都不可见
                to.balance   += cents;
            }
        }
    }
}
```

三个关于锁对象的经验：不要用字符串常量或包装类实例当锁，`synchronized("lock")` 锁的是常量池里那个全局唯一对象，别的代码只要用同一字面量就和你串成了同一把锁；锁对象引用必须是 final 或不可变，否则锁替换了，不同线程就锁在了不同对象上；能用 `synchronized(this)` 以外的私有锁对象就别用 this，因为外部代码也能拿到 this 去加锁，锁的范围会失控。

`synchronized` 在历史上做过锁升级优化，目的是让「绝大多数没有竞争」的场景不付出操作系统调用开销，升级方向单向不可逆：

| 阶段 | 触发条件 | 实现手段 | 开销 |
| --- | --- | --- | --- |
| 无锁 | —— | 普通对象头 | 最低 |
| 偏向锁 | 只有一个线程反复进出 | 在对象头 Mark Word 记线程 ID（JDK 15 起默认关闭并废弃） | 极低 |
| 轻量级锁 | 有少量竞争 | 线程栈上建锁记录，CAS 抢 Mark Word | 自旋消耗 CPU |
| 重量级锁 | 自旋失败或竞争激烈 | 操作系统互斥量，竞争线程挂起 | 涉及内核态切换 |

在「只加锁」这个需求上，`synchronized` 与 `ReentrantLock`（JUC 包提供的显式可重入锁）能力对比如下，绝大多数业务代码用前者就够：

| 维度 | synchronized | ReentrantLock |
| --- | --- | --- |
| 实现层次 | JVM 监视器指令 monitorenter/exit | JDK 层 AQS + CAS |
| 释放方式 | 自动，异常也释放 | 必须写在 finally 里手动 unlock |
| 等锁可中断 | 不支持 | `lockInterruptibly()` |
| 超时放弃 | 不支持 | `tryLock(3, TimeUnit.SECONDS)` |
| 公平性 | 非公平 | 构造时可指定公平锁 |
| 条件队列 | 单一 wait/notify | 一把锁可挂多个 `Condition` |
| 适用场景 | 普通临界区 | 需要超时、可中断、多条件唤醒 |

### 7.4 CAS、原子类与 AQS

CAS（Compare-And-Swap，比较并交换）是一条 CPU 原子指令：只有内存当前值等于期望值时才写入新值并返回 true，否则返回 false。它是无锁编程的地基，`AtomicInteger` 的自增就是对它做循环重试：

```java
public class CasCounter {
    private final AtomicInteger count = new AtomicInteger(0);
    public void increment() { count.incrementAndGet(); }   // 失败就重读重试，直到成功
}
```

CAS 有三个必须知道的短板。一是 ABA 问题：值从 A 变 B 又变回 A，CAS 看不出中间发生过变化，需要 `AtomicStampedReference` 这类带版本号的引用。二是高竞争下大量线程空转重试，CPU 白白烧掉，此时 `LongAdder` 用分段累加把热点打散，效果远好于 `AtomicLong`。三是只能保证一个变量的原子性，多个字段要一起原子更新得用 `AtomicReference` 包一个不可变对象。

AQS（AbstractQueuedSynchronizer，抽象队列同步器）是 JUC 里大部分同步器的公共骨架，核心只有三样东西：一个 `volatile int state` 表示同步状态、一个 CLH 变体的双向等待队列、以及 CAS 抢 state。子类通过重写 `tryAcquire`、`tryRelease` 等模板方法定义「什么算获取成功」，`ReentrantLock` 用它实现 state 计数（可重入就是 state 加一）、`Semaphore` 用它实现许可数量、`CountDownLatch` 用它实现倒计数。理解这一点，读 JUC 源码就不再是死记硬背。

### 7.5 同步工具与并发容器

几个高频工具各自解决一类协作问题：`CountDownLatch`（一次性门闩，主线程等 N 个子任务全部完成）、`CyclicBarrier`（可重复使用的栅栏，N 个线程互相等齐再一起走）、`Semaphore`（信号量，控制同时进行的线程数量，用于限流）、`Condition`（条件队列，配合 `ReentrantLock` 实现精确唤醒）。

```java
ExecutorService pool = Executors.newFixedThreadPool(8);
CountDownLatch latch = new CountDownLatch(8);          // 计数只能减不能加，也不能重置
List<String> results = new CopyOnWriteArrayList<>();   // 多线程同时写、读操作远多于写
for (int i = 0; i < 8; i++) {
    int shard = i;
    pool.submit(() -> {
        try {
            results.add("分片 " + shard + " 完成");
        } finally {
            latch.countDown();                          // 必须放 finally，任务异常也要放行
        }
    });
}
latch.await(3, TimeUnit.SECONDS);                       // 一定带超时，避免永久阻塞
pool.shutdown();
```

容器选择同样有明确的边界：

| 容器 | 并发策略 | 适用场景 | 注意 |
| --- | --- | --- | --- |
| ConcurrentHashMap | 读无锁，写只锁单个桶 | 高并发读写映射 | 复合操作要用 `computeIfAbsent` 等方法，先 get 再 put 有竞态 |
| CopyOnWriteArrayList | 写时复制整个数组 | 读多写极少（配置、监听器列表） | 写开销 O(n)，数据量大时不要用 |
| ArrayBlockingQueue | 数组 + 单锁 + 双条件 | 有界队列，线程池首选 | 容量固定，满了阻塞或拒绝 |
| LinkedBlockingQueue | 链表 + 双锁 | 高吞吐任务队列 | 默认容量 `Integer.MAX_VALUE`，等于无界 |
| SynchronousQueue | 不存储元素，直接交接 | `newCachedThreadPool` 内部使用 | 没有消费者时提交即失败 |

### 7.6 线程池七参数与调优

线程池（ThreadPool，预先创建并复用一组线程来执行提交任务的容器）的价值是三点：复用线程省掉创建销毁开销、用队列削峰、用参数约束住资源上限。`ThreadPoolExecutor` 的构造器有七个参数，每个都要有理由：

| 参数 | 作用 | 常见取值与陷阱 |
| --- | --- | --- |
| corePoolSize | 核心线程数，默认常驻 | CPU 密集型取核数，IO 密集型可放大 |
| maximumPoolSize | 最大线程数 | 队列无界时这个值永远不会生效 |
| keepAliveTime | 非核心线程空闲存活时间 | 默认对核心线程无效，除非开启 allowCoreThreadTimeOut |
| unit | 时间单位 | `TimeUnit.SECONDS` 最常用 |
| workQueue | 任务队列 | 必须用有界队列，`LinkedBlockingQueue` 不传容量即无界 |
| threadFactory | 线程工厂 | 一定要自定义线程名，否则 jstack 里全是 pool-1-thread-n |
| handler | 拒绝策略 | 默认 AbortPolicy 直接抛异常 |

任务提交后的处理顺序固定：核心线程未满就新建核心线程；核心满了进队列排队；队列满了才把线程扩到 maximum；连 maximum 都满了就走拒绝策略。这个顺序解释了一个经典困惑——为什么把 maximumPoolSize 调大不生效？因为用的是无界队列，队列永远不会满，线程数永远停在 core。

四种拒绝策略里只有两种值得用：`AbortPolicy`（抛 `RejectedExecutionException`，让调用方感知到系统过载）与 `CallerRunsPolicy`（由提交任务的线程自己执行，天然形成背压，把压力传回上游）；`DiscardPolicy` 与 `DiscardOldestPolicy` 静默丢任务，等于埋一颗数据丢失的雷。

```java
int cpu = Runtime.getRuntime().availableProcessors();
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    cpu,                                     // 核心线程数：CPU 密集型约等于核数
    cpu * 2,                                 // 最大线程数：IO 等待占比高时才需要放大
    60L, TimeUnit.SECONDS,                   // 非核心线程空闲一分钟后回收
    new ArrayBlockingQueue<>(1000),          // 有界队列，容量按峰值 QPS × 可接受排队时长估算
    new ThreadFactory() {                    // 线程带业务前缀，日志与线程栈一眼可辨
        private final AtomicInteger seq = new AtomicInteger(1);
        @Override public Thread newThread(Runnable r) {
            return new Thread(r, "order-pool-" + seq.getAndIncrement());
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy() // 打满时由提交线程执行，倒逼上游降速
);
```

线程数没有万能公式，只有两个粗略起点：CPU 密集型取 `核数 + 1`；IO 密集型取 `核数 × (1 + 平均等待时间 / 平均计算时间)`。最终值必须由压测得出，并且要配合队列容量与超时一起看。

关闭线程池也不能只调一个方法：`shutdown()` 停止接收新任务，但会把已提交的任务执行完，属于优雅关闭；`shutdownNow()` 在停止接收的同时中断所有工作线程并返回尚未开始的任务列表，用于快速止损。完整写法是先关闭入口、再等一段时间、最后兜底强杀：

```java
pool.shutdown();                                   // 不再接新任务
if (!pool.awaitTermination(30, TimeUnit.SECONDS)) { // 给在途任务 30 秒处理时间
    List<Runnable> dropped = pool.shutdownNow();    // 超时则中断，并拿到没来得及跑的任务
    log.warn("强制关闭，丢弃 {} 个任务", dropped.size());
}
```

线程池一旦进入 `TERMINATED` 就不可复用，应用重启或有状态的动态扩缩容场景必须重新创建实例。

### 7.7 CompletableFuture 编排

`CompletableFuture`（Java 8 引入的异步编排工具，把「稍后才有结果」的计算表达成可组合的对象）用来解决多个 RPC 或数据库调用串行等待的问题：本来 200ms + 300ms 的两次调用，并行之后总耗时接近较慢的那个。

```java
ExecutorService pool = orderPool;   // 业务自定义线程池，不要用默认的公共池
CompletableFuture<User> userFuture =
    CompletableFuture.supplyAsync(() -> userService.load(userId), pool);
CompletableFuture<List<Order>> orderFuture =
    CompletableFuture.supplyAsync(() -> orderService.listByUser(userId), pool);

UserDetail detail = userFuture
    .thenCombine(orderFuture, (user, orders) -> new UserDetail(user, orders)) // 两个都完成才合并
    .orTimeout(2, TimeUnit.SECONDS)                                           // 超时兜底
    .exceptionally(ex -> UserDetail.empty(userId))                            // 任一环节异常则降级
    .join();                                                                  // 阻塞取结果
```

四个高频坑：不传线程池时默认用 `ForkJoinPool.commonPool()`，并行度只有 CPU 核数减一，一个慢 IO 就能拖垮整个 JVM 的公共池；只有以 `Async` 结尾的方法才会切线程，`thenApply` 在上游完成线程上同步执行，耗时逻辑必须写 `thenApplyAsync` 并显式传池；`exceptionally` 只能捕获上游异常，下游自己的异常它管不到；`join()` 抛的是非受检的 `CompletionException`，与 `get()` 抛受检的 `ExecutionException` 不同，边界代码里要包一层转换。

需要等一批互不依赖的调用全部返回时，用 `allOf` 汇总；如果只想拿到最快返回的那个结果（比如主备两个数据源），用 `applyToEither`。前者等到全部完成，其中一个失败会让整体进入异常状态，因此批量场景通常还要对每个 future 单独挂 `exceptionally`，把失败降级成默认值，避免一次失败拖垮整批。

### 7.8 死锁的复现与排查

死锁（Deadlock，两个或多个线程各自持有对方需要的锁、并永久等待对方释放）满足四个条件：互斥、持有并等待、不可抢占、循环等待。破坏任意一个即可，工程上最省事的是破坏循环等待——给所有锁排一个全局顺序，按序加锁。

```java
public class DeadlockDemo {
    private static final Object A = new Object();
    private static final Object B = new Object();

    public static void main(String[] args) {
        new Thread(() -> {                     // t1：先锁 A 再锁 B
            synchronized (A) { sleep(100); synchronized (B) { System.out.println("t1"); } }
        }, "t1").start();
        new Thread(() -> {                     // t2：先锁 B 再锁 A，与 t1 方向相反
            synchronized (B) { sleep(100); synchronized (A) { System.out.println("t2"); } }
        }, "t2").start();
    }
}
```

线上排查按下面的顺序走，通常五分钟内能定位：

1. 用 `jps -l` 或 `ps -ef | grep java` 找到目标进程 PID。
2. 抓线程栈：`jstack -l <pid> > /tmp/dump.txt`，JDK 9+ 也可以用 `jcmd <pid> Thread.print -l`。容器里没装 JDK 工具时，`kill -3 <pid>` 会把线程栈打到应用的标准输出。
3. 在输出里搜关键字 `Found one Java-level deadlock`，JVM 会直接列出涉及的线程和锁。
4. 逐条对读：`- waiting to lock <0x000000076ab0a0a8>` 表示卡在这把锁上，`- locked <0x000000076ab0a0a0>` 表示已经持有。同一个地址在 A 线程是 locked、在 B 线程是 waiting to lock，就是环。
5. 对照线程名（`t1`、`t2` 或自定义的 `order-pool-3`）回到业务代码里找对应的方法与行号，确认加锁顺序。

死锁之外还有两种常被混淆的「卡住」：线程池核心线程全被占满且任务在等一个提交到同一个池的子任务，线程栈表现为大量线程停在队列的 `take` 上，属于线程池饥饿，解法是拆池或去掉嵌套提交；CPU 打到 100% 且线程栈里某个线程长期停在 `RUNNABLE` 同一行，属于自旋或死循环，与锁无关。

### 7.9 收尾速查与常见坑

| 症状 | 首选工具 | 关键判断依据 |
| --- | --- | --- |
| 接口耗时抖动、线程数飙升 | 线程池监控 + `jstack` | 队列长度持续增长说明消费能力不足 |
| 线程全部卡住 | `jstack` 找 deadlock 段 | 锁地址在同一线程 locked、另一线程 waiting to lock |
| 计数不准、状态偶尔回退 | 代码审查 `volatile` 自增 | `n++` 用了 volatile 却没加锁 |
| CPU 高但吞吐低 | `top -Hp` + `jstack` | 多个线程在同一处自旋重试 |

四个真实高频坑：

- 用 `Executors.newFixedThreadPool()` 创建线程池：内部是无界 `LinkedBlockingQueue`，任务堆积到堆内存耗尽，表现为 OOM 而不是拒绝，排查时容易误判成内存泄漏。
- 把 `volatile` 当原子用：`volatile int count` 加 `count++`，压测时少算几个数，线上变成订单金额对不上。
- 线程池里用 `ThreadLocal` 不清理：线程被复用，上一个请求的用户信息串到下一个请求，属于数据串号兼内存泄漏，必须 `try { ... } finally { threadLocal.remove(); }`。
- `catch (InterruptedException e) {}` 吞掉中断：线程永远退不出循环，`shutdown()` 之后进程挂住不结束；正确做法是恢复中断标志或向上抛出。

## 8. JVM 原理

JVM（Java Virtual Machine，Java 虚拟机，一套规定字节码如何被加载、验证、执行与回收内存的规范）是「一次编写，到处运行」真正兑现的地方。同一份 `.class` 文件在 x86 的 Linux、ARM 的 macOS、Windows 上跑出相同结果，靠的是各平台各自实现的虚拟机把同一套字节码翻译成机器动作。

很多人把 JVM 当成面试专用知识，实际上四类线上事故能直接追到它：堆里的对象只增不减，最终报 `OutOfMemoryError`；依赖冲突时同一个类被两个加载器各加载一份，抛出的异常与源码完全对不上号；垃圾回收的停顿让接口 P99 每隔几分钟尖刺一次；共享变量少了一层同步，压测不复现、生产偶尔错。下面几节按「内存分几块 → 对象长什么样 → 类怎么进来 → 垃圾怎么走 → 线程之间看到什么 → 出事用哪个命令看」的顺序展开。

### 8.1 运行时内存区域

判断一个区域是「线程私有」还是「所有线程共享」，比记住名字更重要，因为它同时决定并发问题的来源和溢出报错的方向。

| 区域 | 归属 | 存放内容 | 溢出形态 | 受堆参数控制 |
| --- | --- | --- | --- | --- |
| 程序计数器（Program Counter Register） | 线程私有 | 当前线程执行到哪条字节码 | 规范中唯一不抛溢出的区域 | 否 |
| 虚拟机栈（VM Stack） | 线程私有 | 栈帧：局部变量表、操作数栈、动态链接、返回地址 | `StackOverflowError` | 否，由 `-Xss` 控制 |
| 本地方法栈（Native Method Stack） | 线程私有 | native 方法的调用信息 | 同上 | 否 |
| 堆（Heap） | 共享 | 对象实例、数组、字符串常量池 | `OutOfMemoryError: Java heap space` | 是，`-Xms/-Xmx` |
| 方法区（Method Area）/ 元空间（Metaspace） | 共享 | 类元信息、字段与方法描述、运行时常量池引用、JIT 编译结果 | `OutOfMemoryError: Metaspace` | 否，由 `-XX:MaxMetaspaceSize` 控制 |
| 直接内存（Direct Memory） | 进程共享 | `DirectByteBuffer` 申请的本机内存 | `OutOfMemoryError: Direct buffer memory` | 否，由 `-XX:MaxDirectMemorySize` 控制 |

三条容易记错的细节：

其一，方法区是规范里的概念，JDK 8 之前用「永久代」（Permanent Generation）实现，之后换成元空间（Metaspace），改用本地内存，默认不设上限，只受机器物理内存约束。所以从 JDK 8 升级上来、动态生成大量类的应用（CGLIB 代理、Groovy 脚本、热部署），第一次会见到 `Metaspace` 溢出，而永久代的老参数 `-XX:MaxPermSize` 在新版本上直接无效。

其二，字符串常量池与静态变量在 JDK 7 就已从永久代移入堆，所以 `String.intern()` 撑爆的是堆，不是元空间。

其三，`-Xss` 决定的单线程栈大小与「能开多少线程」直接相关：每个线程都要预留一块栈，栈越大，同样内存下可创建的线程越少，溢出报错也从 `StackOverflowError` 变成 `OutOfMemoryError: unable to create new native thread`。

```java
// 语言：Java。演示两种溢出，需分别用不同参数启动
public class OverflowDemo {
    private int depth = 0;

    // 递归没有终止条件：栈帧只进不出，抛 StackOverflowError
    void deepRecursion() {
        depth++;
        deepRecursion();
    }

    // 静态集合持有引用：GC Roots 可达，永远回收不掉，抛 OutOfMemoryError
    static final java.util.List<byte[]> LEAK = new java.util.ArrayList<>();

    public static void main(String[] args) {
        OverflowDemo demo = new OverflowDemo();
        try {
            demo.deepRecursion();
        } catch (StackOverflowError e) {
            System.out.println("栈深度=" + demo.depth);   // 通常几千到一万多，取决于 -Xss
        }
        while (true) {
            LEAK.add(new byte[1024 * 1024]);             // 每次都进老年代，直到堆耗尽
        }
    }
}
```

启动参数：`java -Xss512k -Xms64m -Xmx64m -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=./dump OverflowDemo`。把栈压到 512KB 时递归深度明显变小，这就是「栈溢出与堆无关」的直观证据。

`HeapDumpOnOutOfMemoryError` 是生产环境必加的参数：事故现场只出现一次，没有堆转储，事后只能靠猜。

常见的溢出信息与首查方向：

| 报错信息 | 典型原因 | 首查手段 |
| --- | --- | --- |
| `Java heap space` | 大对象、集合无上限增长、一次查询捞全表 | 堆转储 + MAT 支配树 |
| `GC overhead limit exceeded` | 堆快满，GC 跑了 98% 时间却只回收不到 2% | 同上，重点看是否内存泄漏而非单纯不够 |
| `Metaspace` | 动态生成类过多、热部署反复加载、`MaxMetaspaceSize` 设得过小 | `jcmd <pid> VM.metaspace`、检查反射/字节码框架 |
| `unable to create new native thread` | 线程数触顶、`-Xss` 过大、系统 `ulimit -u` 限制 | `jstack`、`cat /proc/<pid>/status` 的 Threads |
| `Direct buffer memory` | NIO 直接内存未释放，堆外撑爆 | 检查 `MaxDirectMemorySize` 与 `ByteBuffer.allocateDirect` |

### 8.2 对象创建与内存布局

`new` 一个对象，虚拟机做五件事：检查类是否已加载（未加载就先走类加载）、在堆上分配内存、把分配到的空间清零、设置对象头、执行构造器。分配方式有两种：堆内存规整时用「指针碰撞」（一块空闲指针往后挪），碎片化时用「空闲列表」（从一张表里挑够大的块）。并发分配靠两种手段保证安全，一是 CAS 加失败重试，二是 TLAB（Thread Local Allocation Buffer，线程本地分配缓冲区），每个线程先在 Eden 里预占一小块，自己用，用完再换，默认开启。

对象在内存中分三段：

| 部分 | 内容 | 64 位开启指针压缩时的大小 |
| --- | --- | --- |
| 对象头 Mark Word | 哈希码、GC 分代年龄、锁状态标志、偏向线程 ID、偏向时间戳 | 8 字节 |
| 对象头类型指针（Klass Pointer） | 指向方法区里的类元数据 | 4 字节 |
| 实例数据 | 父类与自身声明的字段 | 按字段类型累加 |
| 对齐填充 | 补齐到 8 字节整数倍 | 0 到 7 字节 |

Mark Word 是一块会「变身」的复用空间：无锁时存哈希码与分代年龄，偏向锁时存偏向线程 ID，轻量级锁时存指向栈中锁记录的指针，重量级锁时存指向 Monitor 的指针。分代年龄只占 4 位，最大值 15，这正是 `-XX:MaxTenuringThreshold` 不能超过 15 的原因。

字段不是按声明顺序排的，虚拟机会按「long/double → int/float → short/char → byte/boolean → 引用」重排，把同类宽度的字段挤在一起减少填充。所以「调换字段声明顺序能否省内存」的答案是：可能省，但省的是填充字节，靠它优化通常不值得，理解重排能解释「为什么对象大小不等于字段之和」即可。

```java
// 语言：Java，依赖 org.openjdk.jol:jol-core:0.17
import org.openjdk.jol.info.ClassLayout;

public class LayoutDemo {
    static class Point {          // 两个 int：8 字节实例数据
        int x;
        int y;
    }
    static class Mixed {          // long + int + boolean：13 字节，需填充
        boolean flag;             // 重排后会被挪到末尾
        long id;
        int count;
    }

    public static void main(String[] args) {
        System.out.println(ClassLayout.parseInstance(new Object()).toPrintable());
        System.out.println(ClassLayout.parseInstance(new Point()).toPrintable());
        System.out.println(ClassLayout.parseInstance(new Mixed()).toPrintable());
    }
}
```

在 64 位、默认开启压缩指针的 HotSpot 上，`new Object()` 占 16 字节：12 字节对象头加 4 字节对齐填充，没有实例数据。`Point` 的两个 int 共 8 字节，12 + 8 = 20，不是 8 的倍数，补 4 字节后实际占 24 字节。`Mixed` 的实例数据是 8 + 4 + 1 = 13 字节，12 + 13 = 25，补到 32 字节，其中 7 字节是纯浪费；字段重排的作用就是把 1 字节的 boolean 挪到末尾，让它正好占住填充位，从而少浪费 4 个字节。压缩指针（`-XX:+UseCompressedOops`）在堆小于 32GB 时默认开启，把对象引用压到 4 字节；一旦堆开到 32GB 以上，压缩指针失效，引用翻倍，实际可用堆反而可能少于 31GB 的配置——「堆越大越好」在这里是个具体的陷阱。

```java
// 语言：Java。用 Unsafe 验证「堆开到 32GB 以上引用变宽」的代价（仅演示，生产别直接调 Unsafe）
import sun.misc.Unsafe;
import java.lang.reflect.Field;

public class OopSizeDemo {
    public static void main(String[] args) throws Exception {
        Field f = Unsafe.class.getDeclaredField("theUnsafe");
        f.setAccessible(true);
        Unsafe unsafe = (Unsafe) f.get(null);
        // 关闭压缩指针时为 8，开启时为 4，可与 -XX:-UseCompressedOops 对照运行
        System.out.println("对象引用宽度=" + unsafe.arrayIndexScale(Object[].class));
    }
}
```

对象不一定都在堆上。JIT 编译器的逃逸分析（Escape Analysis）会判断对象是否逃出方法，没逃出就可能被「标量替换」拆成几个局部变量，连分配都省掉。这解释了一个常被误用的结论：循环里频繁 `new` 小对象未必造成压力，但能不能被优化掉取决于 JIT 是否编译了这段代码、对象是否真的不逃逸，不能作为写烂代码的理由。

### 8.3 类加载机制与双亲委派

类从 `.class` 文件到能用，走完「加载 → 链接 → 初始化」三阶段，链接内部再分验证、准备、解析。准备阶段给静态变量赋零值（`static int count = 10` 此时 count 是 0，赋值要等到初始化阶段），但 `static final` 的编译期常量在这一步就写入了真实值。初始化阶段才真正执行 `<clinit>`，也就是静态字段赋值与静态代码块，虚拟机会保证它只执行一次且线程安全——这是「静态内部类实现单例」能成立的原因。

触发初始化的六个时机：`new`、读写静态字段、调用静态方法、反射调用、初始化子类时父类先初始化、作为启动类的 main 所在类。注意「引用父类的静态字段只初始化父类」「通过数组定义引用类不会初始化它」这两条例外。

类加载器分四层：

| 加载器 | 加载路径 | 能否被 Java 代码直接引用 |
| --- | --- | --- |
| 启动类加载器（Bootstrap ClassLoader） | `JAVA_HOME/lib` 下的核心库，如 `java.lang.*` | 否，打印出来是 null |
| 扩展/平台类加载器（Extension / Platform） | JDK 8 为 `lib/ext`，JDK 9 起为平台模块 | 是 |
| 应用类加载器（Application ClassLoader） | 类路径 `-cp`、`CLASSPATH` | 是 |
| 自定义类加载器 | 自己指定的任意位置，如插件目录、加密的 jar | 是 |

双亲委派（Parent Delegation，收到加载请求先交给父加载器、父加载器加载不了才自己加载）的作用是保证同一个类不会被重复加载，`java.lang.String` 永远由启动类加载器提供，用户自己写一个同名类也不会被加载。实现方式只有十几行：

```java
// 语言：Java。自定义类加载器：只重写 findClass，保留双亲委派
public class DirClassLoader extends ClassLoader {
    private final java.nio.file.Path dir;

    public DirClassLoader(java.nio.file.Path dir) {
        super(DirClassLoader.class.getClassLoader());   // 指定父加载器
        this.dir = dir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        java.nio.file.Path file = dir.resolve(name.replace('.', '/') + ".class");
        try {
            byte[] bytes = java.nio.file.Files.readAllBytes(file);
            // defineClass 把字节数组变成 Class 对象，这是与 JVM 交互的关键一步
            return defineClass(name, bytes, 0, bytes.length);
        } catch (java.io.IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
}
```

双亲委派会被主动打破，三种真实场景：SPI 机制（如 JDBC 的 `DriverManager` 由启动类加载器加载，却要加载厂商放在应用类路径下的驱动实现，于是用线程上下文类加载器「反向委派」）；Tomcat 的 Web 应用类加载器（每个应用一个，先自己加载再委派，实现应用间类隔离）；OSGi 与热部署框架（按模块图加载）。理解了这一点，才能解释「为什么两个模块明明都依赖了同一个 jar，却报 `ClassCastException: com.foo.Bar cannot be cast to com.foo.Bar`」——类在 JVM 里的唯一标识是「加载器 + 全限定名」，两个加载器各加载一份就是两个不同的类。

排错命令：

```bash
# 语言：Shell。看类从哪个 jar 加载、被加载了几次
java -verbose:class -cp app.jar com.example.Main | grep com.foo.Bar
# JDK 9+ 统一日志方式（推荐）
java -Xlog:class+load=info -cp app.jar com.example.Main | grep com.foo.Bar
# 运行中查看类加载器统计
jcmd <pid> VM.classloader_stats
# 定位依赖冲突：同一个类出现在多个 jar
mvn dependency:tree -Dincludes=com.foo:bar
```

异常语义也要分清：`ClassNotFoundException` 是「用字符串找类没找到」，发生在显式加载（`Class.forName`）时；`NoClassDefFoundError` 是「编译期有、运行期找不到」，通常是打包漏了依赖或静态初始化抛异常。`LinkageError` 系列里的 `NoSuchMethodError` 几乎都是版本冲突：编译用了 2.0 的 jar，运行时类路径上却是 1.0。

### 8.4 垃圾回收

垃圾回收（Garbage Collection，GC，自动找出不再被使用的对象并回收其内存的机制）解决的第一个问题是「怎么算不再使用」。JVM 用的是可达性分析（Reachability Analysis）：从一组根对象出发遍历引用链，走不到的对象就是垃圾。根对象（GC Roots）包括虚拟机栈中局部变量表引用的对象、方法区静态字段与常量引用的对象、JNI 引用的对象、活跃线程、被同步锁持有的对象。

与之相对，引用计数无法处理循环引用，Java 没有采用。但 Java 定义了四种引用强度，用来表达「可回收程度」：

| 引用类型 | 回收时机 | 典型用途 |
| --- | --- | --- |
| 强引用 StrongReference | 永不主动回收，宁可 OOM | 普通 `new` 出来的对象 |
| 软引用 SoftReference | 内存不足时回收 | 内存敏感缓存 |
| 弱引用 WeakReference | 下次 GC 必回收 | `ThreadLocalMap` 的 key、规范化映射表 |
| 虚引用 PhantomReference | 回收前收到通知，无法通过它取对象 | 堆外内存清理（`DirectByteBuffer` 的 Cleaner） |

分代收集建立在两条经验假说上：绝大多数对象朝生夕死，熬过越多次回收的对象越难死。于是堆分新生代（Young）与老年代（Old），新生代再分一块 Eden 与两块 Survivor，默认比例 8:1:1。新对象在 Eden 分配，Eden 满触发 Minor GC，存活对象复制到 Survivor 并年龄加一，年龄到阈值（默认 15，动态年龄判定会提前）晋升老年代；大对象（超过 `-XX:PretenureSizeThreshold`）直接进老年代，避免在 Survivor 之间来回复制。老年代满了触发 Major/Full GC。

三种回收算法的取舍：

| 算法 | 做法 | 优点 | 缺点 | 主力场景 |
| --- | --- | --- | --- | --- |
| 标记-清除 | 标记存活对象，清除其余 | 实现简单，不移动对象 | 产生内存碎片 | CMS 的并发清理 |
| 标记-复制 | 存活对象复制到另一半空间 | 无碎片，分配快 | 浪费一半空间 | 新生代 |
| 标记-整理 | 标记后把存活对象向一端移动 | 无碎片，不浪费空间 | 移动成本高，需停顿 | 老年代 |

主流收集器对比：

| 收集器 | 分代 | 算法 | 特点与默认版本 | 关键参数 |
| --- | --- | --- | --- | --- |
| Serial / Serial Old | 新生代 / 老年代 | 复制 / 整理 | 单线程，客户端模式默认 | `-XX:+UseSerialGC` |
| Parallel Scavenge / Parallel Old | 新生代 / 老年代 | 复制 / 整理 | 吞吐量优先，JDK 8 服务端默认 | `-XX:+UseParallelGC`、`-XX:GCTimeRatio` |
| CMS | 老年代 | 标记-清除，并发 | 低停顿，JDK 9 废弃、14 移除 | `-XX:+UseConcMarkSweepGC` |
| G1 | 不分代（逻辑分代） | 复制 + 整理，Region 化 | JDK 9 起服务端默认，可设定停顿目标 | `-XX:+UseG1GC`、`-XX:MaxGCPauseMillis` |
| ZGC | 不分代 / JDK 21 起分代 | 染色指针 + 读屏障，并发 | 停顿常在 1ms 内，适合大堆 | `-XX:+UseZGC` |

G1 把堆切成大小相等的 Region（1 到 32MB，2 的幂，默认堆大小除以 2048），按「回收价值」排序，优先回收垃圾最多的 Region，因此叫 Garbage First。超过 Region 一半大小的对象叫 Humongous 对象，单独占连续的 Region，短命的大对象会造成额外压力。`-XX:MaxGCPauseMillis` 是软目标，设得太激进（比如 20ms）会让 G1 缩小回收范围、频繁触发，反而降低吞吐。

```bash
# 语言：Shell。JDK 9+ 统一 GC 日志（-Xlog），替代 JDK 8 的 -XX:+PrintGCDetails
java -Xms2g -Xmx2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 \
     -Xlog:gc*,gc+heap=debug:file=gc.log:time,level,tags:filecount=5,filesize=50m \
     -jar app.jar
```

一次典型的 GC 健康度检查：

```bash
# 语言：Shell。每 1 秒采样一次，共 20 次，观察各代占用与 GC 次数
jstat -gcutil <pid> 1000 20
# 输出列含义：S0 S1 两个 Survivor 使用率，E Eden，O 老年代，M 元空间，YGC/YGCT 年轻代次数与耗时，FGC/FGCT 全量次数与耗时
```

判读规则很直接：`O` 列在一次 Full GC 后仍接近 100% 且持续上升，说明有对象一直被引用，是泄漏而非「堆不够」；`FGC` 次数在几分钟内涨到几十次，先别急着加 `-Xmx`，加堆往往只是把事故延后。`YGCT` 单项超过几十毫秒也要留意，它通常意味着存活对象太多、Survivor 装不下，对象被过早晋升。

### 8.5 JMM 与 happens-before

JMM（Java Memory Model，Java 内存模型，规定多线程读写共享变量时可见性与有序性的规则）是一份规范，不是内存布局。它把每个线程的本地缓存抽象成「工作内存」，把真实内存抽象成「主内存」，线程修改变量要先写回主内存，读变量要从主内存刷新。这套抽象让「一个线程写的值另一个线程什么时候能看到」有了可判定的答案。

并发问题由此分成三类：原子性（`i++` 是读、加、写三步，不加锁会丢更新）、可见性（一个线程改了，另一个线程读到旧值）、有序性（编译器与 CPU 会重排指令）。

JMM 用 happens-before（先行发生，若 A happens-before B，则 A 的结果对 B 可见且 A 排在 B 之前）来定义可见性，八条规则中最常用的五条：

- 程序次序规则：同一线程内，前面的操作 happens-before 后面的操作。
- 监视器锁规则：对同一个锁的解锁 happens-before 后续对这个锁的加锁。
- volatile 规则：对 volatile 变量的写 happens-before 后续对这个变量的读。
- 线程启动规则：`Thread.start()` happens-before 新线程中的任何操作。
- 线程终止规则：线程中的所有操作 happens-before 其他线程从 `join()` 成功返回。

另有中断规则、对象终结规则与传递性。注意 happens-before 讲的是「可见且有序」，不承诺「A 一定先被执行完」，两者不要混为一谈。

volatile（易失变量，保证可见性并禁止指令重排、但不保证复合操作原子性的修饰符）是最容易用错的一个。它靠内存屏障实现：写之前插入 StoreStore 屏障、写之后插入 StoreLoad 屏障，读之后插入 LoadLoad 与 LoadStore 屏障，从而禁止与相邻指令重排。

```java
// 语言：Java。volatile 保证可见性，但不保证 i++ 的原子性
public class VolatilePitfall {
    private static volatile int count = 0;      // 加 volatile 仍然会丢更新

    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            for (int i = 0; i < 100_000; i++) count++;   // 读-改-写三步，中间可被抢占
        };
        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        t1.start(); t2.start(); t1.join(); t2.join();
        System.out.println(count);              // 通常是 12 万到 19 万，几乎不可能是 200000
    }
}
```

要原子就换成 `AtomicInteger.incrementAndGet()`（CAS 实现）或 `synchronized`。volatile 的正确用途是「一个线程写、多个线程读的状态标志」，以及双重检查锁定里的单例引用：

```java
// 语言：Java。DCL 单例：漏掉 volatile 会在极低概率下拿到半初始化对象
public final class Config {
    private static volatile Config instance;    // volatile 禁止「分配-赋值-初始化」重排

    private final java.util.Map<String, String> data;

    private Config() {
        data = new java.util.HashMap<>();
        data.put("env", "prod");                // 构造器里做耗时初始化，放大了重排的窗口
    }

    public static Config getInstance() {
        if (instance == null) {                 // 第一次检查：避免每次都加锁
            synchronized (Config.class) {
                if (instance == null) {         // 第二次检查：防止重复创建
                    instance = new Config();
                }
            }
        }
        return instance;
    }
}
```

没有 volatile 时，`instance = new Config()` 的三步（分配内存、初始化对象、把引用赋给 instance）可能被重排成「分配 → 赋值 → 初始化」，另一个线程在第一次检查时看到非 null，拿到的却是一个 `data` 还是 null 的对象，随后抛空指针。这个 bug 单机压测几乎撞不到，在多核高并发下才偶发，是「JMM 不学也会写代码、但会写出偶发 bug」的典型样本。

### 8.6 诊断工具与一次内存泄漏排查

命令行工具按用途分四组，JDK 自带，不装额外软件就能定位大部分问题：

| 工具 | 用途 | 常用命令 |
| --- | --- | --- |
| `jps` | 列出本机 Java 进程与主类 | `jps -lvm` |
| `jstat` | 观察 GC 与内存各区统计 | `jstat -gcutil <pid> 1000` |
| `jstack` | 线程栈快照，查死锁、卡死、线程暴涨 | `jstack -l <pid> > stack.txt` |
| `jmap` | 堆信息与堆转储 | `jmap -histo:live <pid> \| head -20`、`jmap -dump:live,format=b,file=heap.hprof <pid>` |
| `jcmd` | 统一入口，覆盖上述功能 | `jcmd <pid> GC.heap_info`、`jcmd <pid> VM.flags` |
| MAT / VisualVM / Arthas | 转储分析、在线诊断 | 可视化看支配树、热点方法 |

下面是一次完整的排查过程，场景是一个订单查询服务，上线三天后每天凌晨 OOM 重启一次。

第一步，确认进程与 GC 趋势。`jps -l` 拿到 pid，`jstat -gcutil 1000 30` 显示老年代使用率从 40% 稳步涨到 98%，Full GC 后只回落到 92%，随后继续上涨。这是内存泄漏的典型曲线：如果只是堆不够，Full GC 后应该能回落到 30% 以下。

第二步，拿到现场。服务已配 `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/dump`，取到 `java_pid1234.hprof`；如果没有配，用 `jmap -dump:live,format=b,file=heap.hprof <pid>` 主动抓一份，注意 `live` 会触发一次 Full GC，谨慎在生产使用。

第三步，分析支配树。MAT 打开转储，Leak Suspects 报告指向一个 `java.lang.ThreadLocal$ThreadLocalMap` 持有约 900MB，往下展开是一批 `OrderQueryContext` 对象。到这里位置已经清楚了：线程池里的线程被复用，每个请求往 `ThreadLocal` 里塞了一个上下文，用完没清。

第四步，回到代码看根因。`ThreadLocalMap` 的 Entry 用弱引用包住 key，value 却是强引用。线程结束后如果线程被销毁，整个 Map 随之消失；但线程池里的线程是长期存活的，key 被 GC 后变成 null，value 仍被线程对象间接引用，直到下一次 `get`/`set` 才有机会清理。这就是「key 被回收、value 泄漏」。

```java
// 语言：Java。错误写法：只 set 不 remove，线程复用时上下文一直累积
public class OrderQueryContext {
    private static final ThreadLocal<OrderQueryContext> HOLDER = new ThreadLocal<>();
    private final java.util.Map<String, Object> cache = new java.util.HashMap<>();

    public static void set(OrderQueryContext ctx) { HOLDER.set(ctx); }
    public static OrderQueryContext get() { return HOLDER.get(); }
    // 缺少 remove()：线程池线程不会死，value 永远可达

    // 正确写法：用完必须清理，放在 finally 里保证异常路径也执行
    public static void run(OrderQueryContext ctx, Runnable task) {
        HOLDER.set(ctx);
        try {
            task.run();
        } finally {
            HOLDER.remove();          // 关键一行，配合线程池使用是硬性要求
        }
    }
}
```

第五步，修复与验证。补上 `finally` 里的 `remove()`；同时在启动参数加 `-XX:+UseG1GC -Xms4g -Xmx4g`（原本 `-Xms` 只给了 512m，与 `-Xmx` 差距大，堆反复伸缩也放大了抖动）。修复后连续观察一周，`jstat` 里老年代稳定在 35% 左右，FGC 归零。

顺带记一个用堆直方图快速缩小范围的技巧：`jmap -histo:live <pid> | head -20` 会按实例数与占用字节排序，泄漏对象的类往往一眼可见。另外 `jstack` 看线程数也很关键——线程泄漏的表现是线程数只增不减，而不是堆涨。

线程数异常时先看总数与状态分布：`jstack <pid> | grep -c "java.lang.Thread.State: WAITING"`；死锁用 `jstack -l` 会自动打印 `Found one Java-level deadlock` 段落，不用逐个看。

### 8.7 常见坑

- 只盯 `-Xmx` 而不管 `-Xms` 与 `-Xss`：`-Xms` 远小于 `-Xmx` 会让堆反复伸缩、加剧 GC 抖动，两者应设成相同值一次性预留；`-Xss` 调大则单线程预留更多内存、可创建线程更少，可能把 `StackOverflowError` 换成 `unable to create new native thread`，栈大小通常保持默认、递归改迭代。
- 把 `OutOfMemoryError` 当成「堆不够」就调大堆：泄漏类问题加堆只是把重启周期从 3 天拉长到 10 天。先看 Full GC 后老年代能否回落，再决定加不加。
- 在 `ThreadLocal` 上只 `set` 不 `remove`：线程池下必然泄漏，且比普通集合泄漏更难发现，因为泄漏点在「每个线程」而非「全局集合」。
- 认为 `volatile` 能替代锁：它只解决可见性与有序性，`i++`、先检查后执行这类复合操作照样出错，必须配 CAS 或 `synchronized`。

## 9. Java 8+ 新特性

Java 8 是这门语言的分水岭：它把「把函数当参数传」和「声明式地处理集合」带进了主流写法，之后 Java 9 到 Java 21 又陆续补上了语法糖与新的并发模型。这些特性单独看都不难，难的是知道每个特性的边界——Lambda 为什么不能改外部变量、parallelStream 在什么情况下反而更慢、Optional 什么时候是负担、虚拟线程为什么不能塞进线程池。下面按「是什么、解决什么问题、怎么写、哪里会踩坑」逐条拆解。

### 9.1 Lambda 与函数式接口

Lambda 表达式（Lambda Expression，用 `(参数) -> 表达式或代码块` 就地定义匿名函数的语法）本身不是一个类型，必须赋值给一个函数式接口才能使用。函数式接口（Functional Interface，有且只有一个抽象方法的接口）是 Lambda 的落地类型；`@FunctionalInterface` 注解让编译器守住「只有一个抽象方法」这条规则，default 方法和 static 方法不算抽象方法，因此可以随意添加。

| 内置接口 | 抽象方法 | 形状 | 典型用途 |
| --- | --- | --- | --- |
| `Function<T,R>` | `R apply(T t)` | 有入有出 | 类型转换、取值 |
| `Consumer<T>` | `void accept(T t)` | 只进不出 | 打印、入库、发消息 |
| `Supplier<T>` | `T get()` | 只出不进 | 延迟创建、工厂 |
| `Predicate<T>` | `boolean test(T t)` | 入参出布尔 | 过滤、校验 |
| `UnaryOperator<T>` | `T apply(T t)` | 入出同类型 | 原地变形 |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | 两个入参 | 合并、比较 |

```java
@FunctionalInterface
interface PriceRule {
    BigDecimal apply(BigDecimal origin);            // 唯一的抽象方法

    default PriceRule andThen(PriceRule next) {     // default 方法不影响函数式接口
        return v -> next.apply(this.apply(v));
    }
}

PriceRule discount = p -> p.multiply(new BigDecimal("0.8"));
PriceRule plusShip = p -> p.add(new BigDecimal("10"));
BigDecimal finalPrice = discount.andThen(plusShip).apply(new BigDecimal("100")); // 90
```

方法引用（Method Reference，`类名::方法名` 的简写形式）是 Lambda 的语法糖，共四种形态：静态方法 `Integer::parseInt`、特定对象的实例方法 `System.out::println`、任意对象的实例方法 `String::length`（形如 `s -> s.length()`）、构造器 `ArrayList::new`。

Lambda 与匿名内部类看起来相近，语义上却有两处关键差别，写框架代码时经常因此出错：

| 对比项 | 匿名内部类 | Lambda |
| --- | --- | --- |
| 编译产物 | 生成 `Outer$1.class` 类文件 | 用 `invokedynamic` 在运行期生成，不产生类文件 |
| `this` 的含义 | 指向匿名类自身实例 | 指向**外围**对象，无法在内部引用自己 |
| 可捕获的变量 | 只能捕获 final 局部变量 | 只能捕获 effectively final 局部变量 |
| 抽象方法数量 | 可覆盖多个方法 | 只能对应一个抽象方法 |

因为 `this` 指向外围对象，在 Lambda 里调用 `this.toString()` 得到的是外层类的输出，想打印 Lambda 自身只能靠 `getClass()`。这也是 Lambda 不能用来实现多方法接口的根本原因——它没有自己的类型。

Lambda 有一条硬限制：它只能捕获「effectively final」的局部变量，也就是声明后不再被重新赋值的变量。原因是 Lambda 可能在方法返回之后、甚至在另一个线程里执行，此时局部变量早已出栈，捕获的只能是创建时的副本；如果允许修改，副本与原件就会不一致。把结果作为返回值传出去，而不是去改外面的变量，才是正确写法。

### 9.2 Stream 的中间操作、终端操作与惰性求值

Stream（流，对数据源做声明式、可链式处理且自身不存储元素的一套 API）由三段构成：数据源、中间操作（Intermediate Operation，返回新的 Stream 但暂不执行）、终端操作（Terminal Operation，触发整条管道执行并产生结果）。惰性求值是它最重要的设计，也是理解性能的前提。

```java
record Order(String user, String sku, int qty, BigDecimal price) {}

List<Order> orders = List.of(
        new Order("u1", "A", 2, new BigDecimal("30")),
        new Order("u1", "B", 1, new BigDecimal("50")),
        new Order("u2", "A", 3, new BigDecimal("30")));

// 按用户汇总金额：filter/map 是中间操作，collect 是终端操作
Map<String, BigDecimal> amountByUser = orders.stream()
        .filter(o -> o.qty() > 0)
        .collect(Collectors.groupingBy(Order::user,
                Collectors.reducing(BigDecimal.ZERO,
                        o -> o.price().multiply(BigDecimal.valueOf(o.qty())),
                        BigDecimal::add)));
// 结果：{u1=110, u2=90}
```

常用操作按类别记更省事。中间操作全部返回 Stream，可以继续链下去：

| 类别 | 操作 | 说明 |
| --- | --- | --- |
| 筛选 | `filter`、`distinct`、`limit`、`skip` | `filter` 无状态；`distinct`、`limit`、`skip` 有状态，并行时要缓冲 |
| 映射 | `map`、`mapToInt`、`flatMap` | `flatMap` 把每个元素展开成流再拼平，用于一对多 |
| 排序 | `sorted`、`sorted(comparator)` | 有状态，必须等全部元素到齐才能出第一个结果 |
| 窥视 | `peek` | 仅供调试，不要用它做业务副作用 |

终端操作负责触发执行并产出结果：`collect`（收集成集合、Map、字符串）、`reduce`（归约成单值）、`count`、`min`/`max`、`anyMatch`/`allMatch`/`noneMatch`（短路）、`findFirst`/`findAny`、`forEach`。区分 `map` 与 `flatMap` 一个例子就够：把 `List<List<Integer>>` 拍平需要用 `flatMap(List::stream)`，用 `map` 会得到 `Stream<List<Integer>>`，类型对不上。

惰性可以直接验证：只写中间操作、不接终端操作时，管道根本不会启动。

```java
Stream.of("a", "b", "c")
      .map(s -> { System.out.println("map " + s); return s.toUpperCase(); });
// 一行都不打印：没有终端操作，map 没有被调用

long n = Stream.iterate(1, i -> i + 1)     // 无限流
        .filter(i -> i % 7 == 0)
        .findFirst()                       // 短路终端操作，找到就停
        .orElse(-1);                       // n = 7
```

三条必须记住的规则：流只能被消费一次，重复使用会抛 `IllegalStateException: stream has already been operated upon or closed`；短路操作（`findFirst`、`anyMatch`、`limit`）让无限流可用；不要在流里修改外部集合，那既破坏惰性求值的语义，在并行时还会丢数据。

`Collectors.toMap` 有一个高频故障：key 重复时直接抛 `IllegalStateException: Duplicate key`，而且异常信息只给出冲突的 key，不指出是哪两行数据。必须显式给合并策略，同时注意 `toMap` 不接受 null 值，遇到 null 会抛 NPE。

```java
// 三个参数：key 映射、value 映射、冲突时的合并函数
Map<String, Integer> qtyBySku = orders.stream()
        .collect(Collectors.toMap(Order::sku, Order::qty, Integer::sum));
```

### 9.3 parallelStream 的坑与取舍

`parallelStream()` 把管道拆成多段交给 ForkJoinPool 的公共池（commonPool，默认并行度为 CPU 核数减一）执行。它并非「免费的加速开关」，四类场景下会直接出问题。

最常见的是共享可变状态。`ArrayList` 不是线程安全的，多线程同时 `add` 会丢元素，甚至抛 `ArrayIndexOutOfBoundsException`，因为扩容时数组正在被复制。

```java
// 错误：普通 ArrayList 并发 add，元素会丢
List<Integer> bad = new ArrayList<>();
IntStream.range(0, 10_000).parallel().forEach(bad::add);   // size 常常小于 10000

// 正确：用 collect，每个线程写自己的容器，最后合并
List<Integer> good = IntStream.range(0, 10_000).parallel()
        .boxed()
        .collect(Collectors.toList());
```

另外三类：公共池是全 JVM 共享的，一旦有阻塞 I/O 或慢任务占满它，所有用到并行流和 `CompletableFuture` 默认池的代码都会排队；有状态或依赖顺序的操作（`limit`、`findFirst`、`forEachOrdered`）在并行下要么结果不确定，要么为保序付出额外合并成本；数据量小的时候，拆分与合并的开销远大于计算本身。

经验阈值：单机数据量在一万条以内、单元素处理是微秒级，就老老实实用串行流；确实是计算密集型且数据量大，再考虑并行，并用自定义 ForkJoinPool 隔离；是 I/O 等待，就该用异步或虚拟线程，而不是靠并行流堆线程。

确需并行时，把任务提交到自己的池子里，可以避免拖累全局：

```java
// 独立池：并行度 4，避免与其它并行流、CompletableFuture 抢 commonPool
ForkJoinPool pool = new ForkJoinPool(4);
try {
    long count = pool.submit(() ->
            IntStream.range(0, 1_000_000).parallel().filter(i -> i % 3 == 0).count()
    ).get();
    System.out.println(count);
} finally {
    pool.shutdown();
}
```

注意并行流会把任务交给「执行终端操作的那个线程」所属的池，所以必须把整条管道放进 `submit` 的 Lambda 内部；先在主线程建好流再提交，用的仍是公共池。

### 9.4 Optional 的正确用法

Optional（容器对象，用来在类型上显式表达「此处可能没有值」）的设计意图很单一：作为方法返回值，提醒调用方必须处理缺失。它不是用来替代所有 null 判断的通用工具。

| 方法 | 行为 | 注意点 |
| --- | --- | --- |
| `Optional.of(v)` | 包装非空值 | 传 null 立刻抛 NPE |
| `Optional.ofNullable(v)` | 允许 null | 最常用 |
| `orElse(default)` | 缺失时返回默认值 | 参数**总是**被求值 |
| `orElseGet(supplier)` | 缺失时才调用 | 默认值昂贵时必用 |
| `orElseThrow()` | 缺失时抛异常 | 替代 `get()` |
| `map` / `flatMap` | 有值才映射 | 链式消除多层判空 |
| `ifPresent` / `ifPresentOrElse` | 有值才消费 | 替代 `isPresent` + `get` |

```java
// 反例：orElse 的参数无论有没有值都会先算一遍
User u1 = repo.findById(id).orElse(loadFromRemote(id));   // 每次都发一次远程查询

// 正例：只在缺失时才执行
User u2 = repo.findById(id).orElseGet(() -> loadFromRemote(id));

// 反例：isPresent + get 等价于 null 判断，白用 Optional
if (opt.isPresent()) { return opt.get().getName(); }
// 正例
return opt.map(User::getName).orElse("unknown");
```

链式取值最能体现价值。假设订单要取「用户的城市」，用户和地址都可能缺失，传统写法是两层 `if` 加 `null` 判断，Optional 写法一行搞定：

```java
// 传统写法：每一层都要判空，漏一层就是 NPE
String city = "unknown";
Order o = orderRepo.findById(id);
if (o != null && o.user() != null && o.user().address() != null) {
    city = o.user().address().city();
}

// Optional 写法：任一层为空，整条链就短路返回默认值
String city2 = Optional.ofNullable(orderRepo.findById(id))
        .map(Order::user)
        .map(User::address)
        .map(Address::city)
        .orElse("unknown");
```

这里 `map` 与 `flatMap` 的区别必须分清：`map` 的映射函数返回普通对象，若返回值已经是 Optional 就会套成 `Optional<Optional<T>>`；`flatMap` 的映射函数直接返回 Optional，由它负责拆一层。从 `Optional<Order>` 取用户的 Optional 时用 `flatMap(Order::findUser)`，取普通字段时用 `map(Order::user)`。

边界同样要守住：不要用 Optional 做字段、方法参数或集合元素，它不可序列化，还会把判空负担推给每个调用方；返回集合时直接返回空集合，而不是 `Optional<List<T>>`；`get()` 只在逻辑上已确认有值时使用，其余一律 `orElseThrow`。

### 9.5 新日期时间 API

`java.time`（JSR-310）替换的是 `java.util.Date` 与 `Calendar` 这两套设计糟糕的老 API。核心类分工明确：`LocalDate` 只有日期，`LocalTime` 只有时间，`LocalDateTime` 是没有时区的日期时间，`Instant` 是 UTC 时间戳，`ZonedDateTime` 带时区，`Duration` 表示时间量，`Period` 表示日期间隔，`DateTimeFormatter` 负责格式化。

| 需求 | 旧 API（问题） | 新 API |
| --- | --- | --- |
| 表示日期 | `Date` 可变、月份从 0 开始 | `LocalDate` 不可变、月份 1 到 12 |
| 格式化 | `SimpleDateFormat` 非线程安全 | `DateTimeFormatter` 线程安全 |
| 时间戳 | `System.currentTimeMillis()` | `Instant.now()` |
| 日期计算 | `Calendar` 可变、API 繁琐 | `plusDays`、`Duration.between` |

```java
LocalDate today = LocalDate.of(2026, 3, 1);
LocalDate next = today.plusDays(30);                  // 返回新对象，today 不变
long days = ChronoUnit.DAYS.between(today, next);     // 30

Instant start = Instant.now();
// ... 业务逻辑
Duration cost = Duration.between(start, Instant.now());   // 计时
System.out.println(cost.toMillis() + "ms");

DateTimeFormatter fmt = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String s = LocalDateTime.now().format(fmt);
LocalDateTime parsed = LocalDateTime.parse("2026-03-01 08:30:00", fmt);

Instant instant = parsed.atZone(ZoneId.of("Asia/Shanghai")).toInstant();  // 跨时区统一存 Instant
```

与老代码互转用 `Date.from(instant)` 和 `date.toInstant()`。格式化字符串里最容易写错的是大小写：`yyyy` 才是年份，大写的 `YYYY` 是「该周所属的年份」，在跨年前后那几天会把 2025-12-31 格式化成 2026，排查起来非常隐蔽；同理月份必须是 `MM` 而不是 `mm`（分钟）。

时区错位是新 API 上线后最常见的一类线上问题，典型现象是「本地测试正常，服务器上时间差 8 小时」。可以按四步定位：

1. 先确认存进数据库的是什么类型。用 `LocalDateTime` 存「墙上时间」，再用 UTC 时区的服务器去读，就会整体偏移 8 小时；跨时区系统应统一用 `Instant` 或 `TIMESTAMP`。
2. 打印 `ZoneId.systemDefault()` 与 `TimeZone.getDefault().getID()`，确认容器里的时区是不是 `UTC`。JDK 容器的默认时区常常是 UTC，而开发机是 `Asia/Shanghai`。
3. 检查格式化时是否用了 `withZone`。`DateTimeFormatter` 本身不带时区，格式化 `Instant` 前必须先 `atZone(zone)`，否则会按 UTC 输出。
4. 最后才怀疑夏令时。用 `ZonedDateTime` 而不是手工加减小时，`America/New_York` 这类时区在切换日并不是固定偏移。

### 9.6 var 与 switch 表达式

`var`（局部变量类型推断，Java 10）让编译器根据右侧表达式推断类型。它是编译期语法糖，字节码里的类型依然明确，Java 仍是静态强类型语言，只是少写了一遍类型名。约束有四条：只能用于局部变量，字段、方法参数、返回值都不行；声明时必须初始化，`var x;` 不合法；不能直接赋 null，编译器无从推断；数组初始化不能省略类型，`var a = {1, 2}` 不合法。

```java
var list = new ArrayList<String>();   // 推断为 ArrayList<String>，不是 List
var map = new HashMap<String, Integer>();
for (var entry : map.entrySet()) { }  // 循环变量也可以用
```

三个容易被误解的点：`var` 不是 `val`，变量本身仍可重新赋值，不可变要靠 `final var`；`var` 声明的变量可以指向匿名类并调用它自己的方法，因为推断出的是匿名类型本身；`var x = 1, y = 2;` 不合法，一次只能声明一个变量，因为推断需要每个变量各自对应一个初始化表达式。

switch 表达式（Java 14）解决了老 switch 的两个老毛病：漏写 `break` 导致穿透，以及只能当语句用、无法直接产出值。箭头写法不穿透，整体可以赋值，代码块里用 `yield` 返回值。

```java
String level = switch (score / 10) {
    case 10, 9 -> "A";                       // 多个标签用逗号
    case 8 -> "B";
    case 7 -> "C";
    default -> {
        String label = "D";                  // 代码块需要 yield 返回值
        yield score < 0 ? "invalid" : label;
    }
};
```

对枚举做 switch 表达式时，覆盖全部常量就无需 `default`，编译器会检查穷尽性；将来新增枚举值会直接编译失败，这正是我们想要的效果——把遗漏暴露在编译期，而不是运行期的错误分支。

### 9.7 record、sealed 与文本块

record（记录类，Java 16 正式版）是不可变的数据载体。声明一行组件列表，编译器自动生成全参构造器、同名访问器、`equals`、`hashCode`、`toString`。

```java
public record Point(int x, int y) {
    // 紧凑构造器：只写校验逻辑，参数列表与赋值由编译器补全
    public Point {
        if (x < 0 || y < 0) throw new IllegalArgumentException("坐标不能为负");
    }
    public int manhattan() { return x + y; }   // 可以加方法
}
// 访问器是 x()、y()，没有 getX()
```

record 的限制来自它的定位：隐式 final，不能继承其他类（但可以实现接口），不能声明额外的实例字段，组件本身就是字段。它适合 DTO、消息体、坐标、多返回值；不适合需要可变状态的类，也不能直接当 JPA 实体——持久化框架依赖无参构造器和 setter，与 record 的语义冲突。

sealed（密封类型，Java 17 正式版）用 `permits` 限定谁能继承，把「这个接口只有这几种实现」写进类型系统。

```java
public sealed interface Shape permits Circle, Square, Triangle {}
record Circle(double r) implements Shape {}
record Square(double side) implements Shape {}
final class Triangle implements Shape { /* ... */ }
```

被许可的子类必须显式声明为 `final`、`sealed` 或 `non-sealed` 之一，编译器据此知道继承链到此为止还是继续收窄。配合 switch 模式匹配，可以做到新增实现就编译报错的穷尽检查。

模式匹配（Pattern Matching）让类型判断与类型转换合二为一。`instanceof` 模式在 Java 16 转正，判断成功后直接绑定变量，不再需要强转；record 解构模式在 Java 21 转正，可以直接把组件取出来。

```java
// instanceof 模式：判断即绑定，作用域限定在判断为真的分支里
if (obj instanceof Circle c) {
    System.out.println(c.r());          // 不再需要 (Circle) obj 强转
}

// switch 模式匹配 + record 解构：直接按形状分发
double area = switch (shape) {
    case Circle(double r) -> Math.PI * r * r;          // 解构出 r
    case Square(double side) -> side * side;
    case Triangle t -> t.base() * t.height() / 2;      // 不关心内部结构时用类型模式
};
```

sealed 加上 record 解构，是这套语法真正的价值所在：编译器知道所有可能的分支，少写任何一个都会报错，加了新实现也会立刻暴露漏改的 `switch`。这正是老式 `instanceof` 一串 `if` 永远做不到的检查。

文本块（Text Block，Java 15 正式版）用三个双引号写多行字符串，专治 SQL、JSON、HTML 里的转义地狱。

```java
String sql = """
        SELECT id, name
        FROM users
        WHERE status = 'ACTIVE'
        """;
```

三条规则要记牢：开头 `"""` 之后必须换行；所有行共同的最小缩进会被去掉，结尾 `"""` 的位置也参与计算；结尾 `"""` 单独占一行时字符串以换行结尾。此外 `\` 表示行连接（把下一行接上来），`\s` 表示一个不会被裁掉的空格。

### 9.8 虚拟线程与适用场景

虚拟线程（Virtual Thread，Java 21 正式定稿）是由 JVM 调度、挂载在少量平台线程（载体线程）上执行的轻量线程。它在阻塞时会把自身从载体线程上卸载，载体线程转去执行别的虚拟线程，因此「一个请求一个线程」这种最直观的模型又能支撑十万级并发，不必再为了省线程把代码写成回调地狱。

| 维度 | 平台线程 | 虚拟线程 |
| --- | --- | --- |
| 与 OS 线程关系 | 一对一 | 多对多，由 JVM 调度 |
| 栈内存 | 默认约 1MB，数量受内存限制 | 几百字节起，按需增长，分配在堆上 |
| 创建成本 | 毫秒级，必须池化 | 微秒级，不需要池化 |
| 适用负载 | CPU 密集型 | 阻塞 I/O 密集型 |

```java
// 方式一：直接启动
Thread t = Thread.ofVirtual().name("vt-", 0).start(() -> handle());
t.join();

// 方式二：每个任务一个虚拟线程的执行器（推荐）
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> futures = urls.stream()
            .map(u -> executor.submit(() -> httpGet(u)))   // 每个请求一个虚拟线程
            .toList();
    for (Future<String> f : futures) System.out.println(f.get());
}   // close() 会等待所有任务结束
```

五条实践要点。第一，不要池化虚拟线程：它便宜到可以直接创建，池化只会引入排队和 ThreadLocal 复用问题，限流应该放在连接池、信号量这类真正的稀缺资源上。第二，只对「阻塞等待」有益：CPU 密集型任务用虚拟线程不会更快，载体线程数量仍受核数限制。第三，注意固定（pinning）：在 JDK 24 之前，虚拟线程在 `synchronized` 块内阻塞时会被钉在载体线程上，退化成平台线程，改用 `ReentrantLock` 可规避；调用 native 方法或持有 JNI 锁同样会固定。第四，慎用 ThreadLocal：线程数量巨大时每个线程一份副本会放大内存占用，优先用方法参数或 ScopedValue 传递上下文。第五，下游资源仍是瓶颈：数据库连接池只有 20 个连接，开一万个虚拟线程也只会有 9980 个在等连接。

排查虚拟线程问题有两个入手点：确认有多少个虚拟线程卡在同一次阻塞调用上，以及载体线程是否被固定。JFR 事件 `jdk.VirtualThreadStart`、`jdk.VirtualThreadPinned` 会把这两类信息都记下来，`jcmd <pid> Thread.dump_to_file -format=json` 则能导出包含虚拟线程的线程快照，比传统的 `jstack` 更适合分析上万线程的场景。

从平台线程迁移过来不必推翻重写：把 `Executors.newFixedThreadPool(n)` 换成 `newVirtualThreadPerTaskExecutor()`，把 `synchronized` 换成 `ReentrantLock`，再评估一遍每个请求里是否持有 ThreadLocal 副本，通常就完成了大部分改造。

### 9.9 常见坑速查

1. 在 Lambda 里修改外部局部变量，编译报 `local variables referenced from a lambda expression must be final or effectively final`。用单元素数组或 `AtomicInteger` 属于绕过而非解决，正确做法是把结果作为返回值返回。
2. `orElse(compute())` 在值存在时也会执行一次昂贵计算，导致无谓的远程调用或对象创建；默认值需要计算时一律改用 `orElseGet(() -> compute())`。
3. 日期格式化把年份写成大写的 `YYYY`、或把月份写成小写的 `mm`，跨年与分钟会静默错位；年份用 `yyyy`，月份用 `MM`，分钟才是 `mm`。
4. `parallelStream` 里往普通 `ArrayList` 或共享的非线程安全对象写入，结果是丢数据或随机异常；需要并行收集就用 `collect`，需要并行 I/O 就用虚拟线程或异步。

## 10. 主流框架与工程

### 10.1 控制反转与依赖注入

Spring 是 Java 后端的事实标准框架（Framework，提供通用能力的半成品程序骨架），它的内核只有两件事：把对象的创建交给容器，把对象之间的依赖也交给容器自动装配。控制反转（Inversion of Control，把对象创建与依赖装配的控制权从业务代码交还给框架容器的设计思想）解决的是一个很具体的问题：手写 `new` 会让上层代码锁死在下层实现上。

```java
// 反面写法：UserService 自己造依赖
public class UserService {
    private final UserRepository repo = new UserRepository(); // 直接绑定具体实现
}
```

这里的问题不在「多写了一行」，而在三处：单元测试时无法替换成内存实现，必须连真实数据库；换一个实现要改源码重新编译；每个调用方各自 `new` 一份，配置（连接池、超时）无法统一。依赖注入（Dependency Injection，容器在创建对象时把所需依赖从外部传入的做法）把这三件事都解掉了：类只声明「我需要什么」，容器负责「给你什么」。

```java
@Repository
public class UserRepository { /* 数据访问 */ }

@Service
public class UserService {
    private final UserRepository repo;

    // 只有一个构造器时 @Autowired 可省略，Spring 自动按类型注入
    public UserService(UserRepository repo) { this.repo = repo; }
}
```

| 注入方式 | 写法 | 优点 | 风险 |
| --- | --- | --- | --- |
| 构造器注入 | 构造器参数 | 依赖可 `final`、缺失即启动失败、脱离容器也能 new 出来测 | 参数过多说明职责过重 |
| Setter 注入 | `@Autowired` 标注 setter | 可做可选依赖、允许运行时替换 | 依赖可能为 null，空指针推迟到运行期 |
| 字段注入 | `@Autowired` 标在字段上 | 代码最短 | 不能 `final`、无法直接单测、隐藏循环依赖 |

Spring 官方推荐构造器注入，理由就是那张表里的「缺失即启动失败」——依赖问题在启动时暴露，比在生产环境跑出一个 `NullPointerException` 便宜得多。

Bean 的注册靠注解加包扫描：`@Component` 是通用组件，`@Service` 标业务层，`@Repository` 标持久层（额外把数据库异常翻译成 Spring 的统一异常），`@Controller` 标 Web 层，`@Configuration` 配合 `@Bean` 用来把第三方类（如数据源、线程池）注册进容器。`@ComponentScan` 默认只扫启动类所在包及其子包，所以启动类必须放在最外层包，否则写好的 Bean 一个都不会被创建——这是新手第一个高频事故。

容器里的 Bean 默认是单例（Singleton，整个容器内只存在一个实例）。单例意味着状态共享：在 Service 里写 `private int count;` 这类可变成员变量，并发请求会互相覆盖，必须改成方法内局部变量或线程安全结构。

作用域（Scope，Bean 实例的创建与共享范围）不止单例一种：

| 作用域 | 实例数量 | 典型用途 | 注意 |
| --- | --- | --- | --- |
| `singleton` | 容器内一个 | 无状态 Service、Mapper、配置 | 默认值，禁止存请求级状态 |
| `prototype` | 每次获取都新建 | 有状态工具对象 | 容器不管理销毁，`@PreDestroy` 不执行 |
| `request` | 每个 HTTP 请求一个 | 请求上下文、当前用户 | 仅在 Web 环境有效 |
| `session` | 每个会话一个 | 登录态（更推荐存 Redis） | 集群下需会话共享 |

Bean 的生命周期大致是：实例化 → 属性填充（依赖注入）→ 各类 `Aware` 回调 → `BeanPostProcessor` 前置处理 → `@PostConstruct` / `InitializingBean` → 使用 → `@PreDestroy` / `DisposableBean`。工程上用得最多的是 `@PostConstruct`（初始化缓存、校验配置）与 `@PreDestroy`（关闭线程池、释放连接）。

循环依赖指 A 依赖 B、B 又依赖 A。构造器注入会在启动时直接失败并报 `Requested bean is currently in creation`，这是好事——它把设计问题暴露在启动阶段；字段注入则被 Spring 的三级缓存（`singletonObjects` 成品池、`earlySingletonObjects` 早期引用池、`singletonFactories` 对象工厂池）悄悄兜住，运行起来不报错，但对象可能拿到尚未初始化完成的半成品。遇到循环依赖优先重构拆分职责，而不是靠改成字段注入绕过去。

### 10.2 AOP 与代理

横切逻辑（Cross-Cutting Concern，日志、事务、鉴权、监控这类散落在众多方法里、与业务无关却又必须执行的代码）如果每个方法手写一遍，重复且容易漏。AOP（Aspect-Oriented Programming，面向切面编程，把这些逻辑集中定义、再统一织入到目标方法执行前后的技术）就是为此而生。

术语对应关系：切面（Aspect）是逻辑的集合类；切点（Pointcut）是「在哪些方法上织入」的表达式；通知（Advice）是「在什么时机执行」，分 `@Before`、`@After`、`@AfterReturning`、`@AfterThrowing`、`@Around`；连接点（JoinPoint）是运行时被拦截到的那个方法执行点。

```java
@Aspect
@Component
public class TimingAspect {
    private static final Logger log = LoggerFactory.getLogger(TimingAspect.class);

    // 切点：com.example.demo.service 包及子包下任意类的任意方法
    @Around("execution(* com.example.demo.service..*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();          // 必须调用，否则目标方法不会执行
        } finally {
            long cost = (System.nanoTime() - start) / 1_000_000;
            log.info("{} cost={}ms", pjp.getSignature().toShortString(), cost);
        }
    }
}
```

切点表达式 `execution(修饰符? 返回类型 包.类.方法(参数))` 里两个通配符最常用：`..` 表示任意层级的包或任意参数，`*` 表示任意字符。写成 `execution(* com.example..*Service.*(..))` 就只拦所有 `Service` 结尾的类。

同一个切点上有多个通知时，执行顺序为 `@Around` 前半 → `@Before` → 目标方法 → `@AfterReturning` 或 `@AfterThrowing` → `@After` → `@Around` 后半，可用 `@Order(1)` 控制多个切面之间的先后。真实项目里 AOP 主要落在四类场景：声明式事务、方法级缓存（`@Cacheable`）、接口耗时与调用链埋点、权限与幂等校验。

Spring AOP 的底层是动态代理，两种实现差异明显：

| 维度 | JDK 动态代理 | CGLIB |
| --- | --- | --- |
| 前提 | 目标类实现至少一个接口 | 目标类可被继承（非 final） |
| 生成物 | 实现同一接口的代理类 | 目标类的子类 |
| 覆盖范围 | 只有接口里声明的方法 | 所有非 final、非 private 的 public 方法 |
| 依赖 | JDK 自带 | 第三方字节码库（Spring Boot 已内置） |
| 生效开关 | `spring.aop.proxy-target-class=false` 且目标有接口 | Spring Boot 2.x 起默认为 true，统一走 CGLIB |

代理机制直接决定了 AOP 的失效场景，排查时按顺序看四件事：一是自调用，`this.methodB()` 用的是原始对象而不是代理，切面自然不执行；二是方法为 `private`、`final`、`static`，代理无法覆盖；三是对象是自己 `new` 出来的，不在容器里就没有代理；四是目标方法不是 `public`，JDK 代理只拦接口方法、CGLIB 只拦可继承的 public 方法。修法是把方法拆到另一个 Bean，或注入自身代理（`@Lazy private UserService self;`），或用 `AopContext.currentProxy()`（需 `@EnableAspectJAutoProxy(exposeProxy = true)`）。`@Transactional` 在自调用下失效，根因就是第一条。

### 10.3 Spring Boot：起步依赖与自动配置

起步依赖（Starter，把某一类功能所需的依赖打包成一个坐标的依赖集合）让「加一行依赖」代替「加十几个依赖并逐个对齐版本」。`spring-boot-starter-web` 会带进 Spring MVC、内嵌 Tomcat、Jackson 与校验器；版本由父 POM 里的 `spring-boot-dependencies`（一种 BOM，Bill of Materials，只声明版本号的依赖清单）统一仲裁，因此子依赖一般不写 `<version>`，写错版本反而会破坏兼容性。

`@SpringBootApplication` 是三个注解的复合：`@SpringBootConfiguration` 标明配置类、`@EnableAutoConfiguration` 开启自动配置、`@ComponentScan` 开启包扫描。

版本上有一个必须记住的分界线：Spring Boot 2.x 基线是 JDK 8，包名沿用 `javax.*`；Spring Boot 3.x 基线抬到 JDK 17，并全面迁移到 Jakarta EE 9 的 `jakarta.*` 命名空间。这意味着把 2.x 升到 3.x 时，任何依赖 `javax.servlet`、`javax.persistence` 的第三方库都必须同步换版本，否则编译期就会出现「找不到符号」。新项目直接用 3.x 加 JDK 17 或 21；已有 2.x 项目不必为了升级而升级，先确认关键中间件客户端有兼容版本。

自动配置（Auto-configuration，根据 classpath 上是否存在某个类、容器里是否已有某个 Bean，条件性地注册配置类的机制）的入口是 jar 包内 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件（Spring Boot 2.7 之前是 `spring.factories` 里的 `EnableAutoConfiguration` 键），里面列出一批配置类，每个类上挂条件注解：

| 条件注解 | 生效条件 |
| --- | --- |
| `@ConditionalOnClass` | classpath 上存在指定类，比如有 `DataSource` 才配数据源 |
| `@ConditionalOnMissingBean` | 容器中没有该类型 Bean，即用户自定义优先 |
| `@ConditionalOnProperty` | 配置项存在且等于期望值，如 `spring.redis.enabled=true` |
| `@ConditionalOnWebApplication` | 当前是 Web 应用环境 |

由此推出覆盖自动配置的三条正规路径：自己定义同类型 Bean（靠 `@ConditionalOnMissingBean` 让默认配置退让）、在配置文件里改属性、用 `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)` 或 `spring.autoconfigure.exclude` 直接排除。

配置的优先级从高到低大致是：命令行参数 → Java 系统属性 → 操作系统环境变量 → jar 包外的 `application-{profile}.yml` → jar 包内的 `application-{profile}.yml` → jar 包内的 `application.yml` → `@PropertySource` 标注的文件 → 代码里的默认值。排查「我明明改了配置却不生效」时，先看是不是被更高优先级的环境变量或命令行参数盖住了。

自动配置不生效时不用猜，用 `java -jar app.jar --debug` 启动会打印条件评估报告（Condition Evaluation Report），分 `Positive matches` 与 `Negative matches` 两段，能直接看到某个配置类是被 `@ConditionalOnMissingBean` 让位给了自定义 Bean，还是因为缺少某个类被 `@ConditionalOnClass` 跳过。定位「为什么数据源没初始化」「为什么拦截器没注册」这类问题，比翻源码快得多。

### 10.4 MVC 分层与 RESTful 接口

分层不是为了好看，而是为了让每层只承担一种变化原因：

| 层 | 典型类 | 职责 | 不该做的事 |
| --- | --- | --- | --- |
| 接口层 | `XxxController` | 参数校验、协议转换、状态码 | 写业务规则、直接拼 SQL |
| 业务层 | `XxxService` | 事务边界、业务编排 | 处理 `HttpServletRequest` |
| 持久层 | `XxxMapper` / `XxxRepository` | 数据访问 | 包含业务判断 |
| 传输对象 | `XxxDTO` / `XxxVO` | 对外契约 | 直接暴露数据库实体 |

入参校验用 JSR-380 注解表达约束，由 `@Valid` 触发，校验失败抛 `MethodArgumentNotValidException`，交给统一异常处理器转成 400：

| 注解 | 约束 | 常见误用 |
| --- | --- | --- |
| `@NotNull` | 不能为 null | 加在 `String` 上却期望拦住空串 |
| `@NotBlank` | 字符串去空格后非空 | 用在数值字段上（会报错） |
| `@Size(min, max)` | 长度或集合大小范围 | 与数据库列宽不一致 |
| `@Min` / `@Max` | 数值边界 | 金额用 `double` 校验 |
| `@Pattern` | 正则匹配 | 正则未做锚定导致部分匹配 |

RESTful（Representational State Transfer，用 URL 表示资源、用 HTTP 动词表示操作、用状态码表示结果的接口风格）的关键是「名词进 URL，动词进方法」：`GET /api/users/1` 而不是 `GET /api/getUser?id=1`。常用状态码：200 成功、201 已创建、204 无内容、400 参数错误、401 未认证、403 无权限、404 资源不存在、409 冲突、500 服务端异常。

对外接口建议统一响应外壳，例如 `{"code":0,"message":"ok","data":{...}}`，好处是前端只写一处解析逻辑，错误码也能承载业务语义；但 HTTP 状态码不要一律返回 200，否则网关、监控、重试策略全部失效。分页接口固定 `page`（从 0 或 1 开始要写进文档）与 `size` 两个参数，并对 `size` 设上限，防止有人传 `size=100000` 把数据库拖垮。

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) { this.userService = userService; }

    @GetMapping("/{id}")
    public UserVO get(@PathVariable Long id) { return userService.get(id); }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Long create(@RequestBody @Valid CreateUserRequest req) { return userService.create(req); }

    @GetMapping
    public List<UserVO> list(@RequestParam(defaultValue = "0") int page,
                             @RequestParam(defaultValue = "20") int size) {
        return userService.list(page, size);
    }
}
```

统一异常处理用一个 `@RestControllerAdvice` 收敛，避免每个方法写 try-catch，也避免把堆栈直接返回给前端：

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, Object> onInvalid(MethodArgumentNotValidException e) {
        String msg = e.getBindingResult().getFieldErrors().stream()
                .map(f -> f.getField() + ": " + f.getDefaultMessage())
                .collect(Collectors.joining("; "));
        return Map.of("code", 40001, "message", msg);   // 对外只给可读信息
    }
}
```

### 10.5 持久层：MyBatis 与 JPA 的取舍

持久层框架都是对象关系映射（Object-Relational Mapping，ORM，把 Java 对象与数据库表行互相转换的技术）的实现，区别在于「映射规则谁来写、SQL 谁来决定」：MyBatis 让你写 SQL、它负责参数绑定与结果映射；JPA 让你写实体与注解，SQL 由框架生成。

| 维度 | MyBatis | JPA / Hibernate |
| --- | --- | --- |
| 定位 | 半自动 ORM，SQL 自己写 | 全自动 ORM，对象到表自动映射 |
| SQL 控制力 | 完全可控，便于手工优化与改写 | 由框架生成，调优要懂脏检查与抓取策略 |
| 上手成本 | 低，会 SQL 就能干活 | 高，缓存、级联、延迟加载都需要理解 |
| 复杂查询 | 灵活，多表报表友好 | 复杂关联与动态条件较别扭 |
| 数据库迁移 | SQL 方言要改 | 换库基本无感 |
| 常见事故 | `${}` 拼接导致注入 | N+1 查询、延迟加载在序列化时炸 |

MyBatis 的映射文件里，`#{}` 是预编译占位符，`${}` 是字符串直接拼接。前者安全，后者只在动态表名、动态排序字段这类无法预编译的场景使用，且必须先做白名单校验，否则 `ORDER BY ${sort}` 就是一条现成的注入通道。

```xml
<select id="findActiveByName" resultType="com.example.demo.domain.User">
    SELECT id, name, created_at
    FROM t_user
    WHERE name = #{name}      <!-- 预编译参数，防注入 -->
      AND status = 1
</select>
```

JPA 的 N+1 问题指：查出一批主对象后，遍历访问其关联集合，框架为每一条记录再发一条 SQL，1 条变 N+1 条。典型症状是接口 QPS 上不去、数据库慢查询里全是同构的小 SQL，解法是用 `@EntityGraph` 或 `join fetch` 一次性抓取，或改成按 ID 批量查询。

选型上的结论很朴素：业务以增删改查为主、SQL 手写量不大、团队更熟悉对象模型，用 JPA；报表、多表关联、已有大量优化过的 SQL、需要精确控制执行计划，用 MyBatis。两者混用也常见：核心写路径用 JPA 保证一致性，复杂查询走 MyBatis 手写 SQL。MyBatis 的分页不要自己拼 `LIMIT`，用分页插件（PageHelper）统一拦截，否则不同数据库方言要各写一套；JPA 的并发更新用 `@Version` 乐观锁字段，而不是靠「先查再改」的读改写，后者在并发下必然丢失更新。

### 10.6 事务与隔离级别

声明式事务靠的也是 AOP 代理：`@Transactional` 在方法执行前从连接池取出连接、关闭自动提交，方法正常返回就提交，抛出未捕获的运行时异常就回滚。事务状态绑定在 `ThreadLocal` 上，所以它只在同一个线程、同一个连接里成立。

失效场景要背下来：自调用绕过代理；`private` 或 `final` 方法不能代理；异常在方法内被 `catch` 掉且没有重新抛出；抛出的是受检异常（默认只对 `RuntimeException` 与 `Error` 回滚）；业务逻辑被丢进 `CompletableFuture` 在新线程执行；数据库表引擎是 MyISAM 这类不支持事务的引擎。

```java
@Service
public class OrderService {
    @Transactional(rollbackFor = Exception.class, timeout = 5)
    public void createOrder(CreateOrderRequest req) {
        orderMapper.insert(req.toOrder());
        // 远程调用、消息发送不应放在事务内，否则长耗时占着连接
        throw new IllegalStateException("库存不足");   // 运行时异常 → 回滚
    }
}
```

传播行为（Propagation，一个事务方法被另一个事务方法调用时，事务如何传递）以 `REQUIRED`（默认，有事务就加入，没有就新建）和 `REQUIRES_NEW`（挂起当前事务，自己开一个新事务）最常用；需要「主流程失败但日志必须落库」时用后者。

事务还有一个必须配对的工程习惯：把事务范围压到最短。事务里不要做 HTTP 远程调用、文件上传、消息发送，这些操作耗时不可控，会把数据库连接一直占住，并发一高连接池就被打满，现象是接口全部变慢但数据库 CPU 并不高。排查长事务可以直接查 InnoDB 的运行事务表：

```sql
-- MySQL 8：按已运行秒数找出最老的事务，看它在等什么、锁了谁
SELECT trx_id, trx_state, trx_started,
       TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS running_sec,
       trx_mysql_thread_id AS conn_id, trx_query
FROM information_schema.innodb_trx
ORDER BY trx_started ASC;
```

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 常见默认 |
| --- | --- | --- | --- | --- |
| READ_UNCOMMITTED | 可能 | 可能 | 可能 | 极少使用 |
| READ_COMMITTED | 不会 | 可能 | 可能 | Oracle、PostgreSQL 默认 |
| REPEATABLE_READ | 不会 | 不会 | 可能（MySQL 用间隙锁基本规避） | MySQL InnoDB 默认 |
| SERIALIZABLE | 不会 | 不会 | 不会 | 并发极差 |

脏读是读到别人未提交的数据；不可重复读是同一事务内两次读同一行结果不同；幻读是同一范围查询两次行数不同。隔离级别越高一致性越强、并发越差，工程上按业务容忍度选，而不是一律拉到最高。

单库单表靠本地事务就够，跨服务、跨库的一致性不要指望 `@Transactional`，它管不到另一个数据库连接。这类场景的通行做法是本地消息表或事务消息加幂等消费，用「最终一致」换可用性；而幂等的兜底往往落在数据库唯一索引上——先加唯一约束，再处理重复插入异常，比在代码里先查后插可靠得多。

### 10.7 配置、Profile 与日志

多环境配置靠 Profile（一组按环境激活的配置集合），把公共配置放 `application.yml`，环境差异放 `application-dev.yml`、`application-prod.yml`，用 `spring.profiles.active` 或启动参数 `--spring.profiles.active=prod` 激活。

```yaml
server:
  port: 8080
spring:
  profiles:
    active: dev
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/demo?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
    username: ${DB_USER:root}          # 冒号后是默认值，优先取环境变量
    password: ${DB_PASSWORD:}
logging:
  level:
    root: INFO
    com.example.demo: DEBUG
  pattern:
    console: "%d{HH:mm:ss.SSS} %-5level [%X{traceId}] %logger{36} - %msg%n"
```

密码、密钥这类敏感项不要写在仓库里，用环境变量占位符 `${DB_PASSWORD}`，并提供默认值兜底。成组的配置项用 `@ConfigurationProperties(prefix = "app.mail")` 绑定到一个类上，比散落的 `@Value` 好维护，还能追加 `@Validated` 做启动时校验。

日志规范建议：用 SLF4J（Simple Logging Facade for Java，日志门面，只提供 API、由 Logback 等实现真正输出）打日志，代码里不要直接依赖具体实现，方便换实现。占位符 `{}` 而不是字符串拼接，避免低级别日志被关闭时仍然白做字符串运算；异常对象作为最后一个参数、且不带占位符，否则堆栈不会打印；用 MDC（Mapped Diagnostic Context，按线程绑定的上下文映射）塞入 `traceId`，让一次请求的日志能被串起来。

```java
log.info("create order userId={}, amount={}", userId, amount);  // 正确：占位符延迟求值
log.info("create order userId=" + userId);                      // 错误：无条件拼接字符串
log.error("pay failed, orderNo={}", orderNo, e);                 // 正确：e 作为最后参数，打印堆栈
```

线上日志必须配滚动策略，否则一个磁盘写满就把服务拖死。`application.yml` 里的 `logging.file.name` 只够本地调试，生产一般放一个 `logback-spring.xml`：

```xml
<configuration>
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>logs/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>   <!-- 单文件上限 -->
            <maxHistory>15</maxHistory>        <!-- 只保留 15 天 -->
            <totalSizeCap>5GB</totalSizeCap>   <!-- 总量上限，防止磁盘被吃满 -->
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %-5level [%thread] [%X{traceId}] %logger{40} - %msg%n</pattern>
        </encoder>
    </appender>
    <root level="INFO"><appender-ref ref="FILE"/></root>
</configuration>
```

### 10.8 最小可运行 Web 服务与常见坑

```xml
<!-- pom.xml 关键片段 -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version>
</parent>
<properties><java.version>17</java.version></properties>
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
<build><plugins><plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin></plugins></build>
```

```java
@SpringBootApplication   // 放在最外层包，如 com.example.demo
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

启动与验证：

```bash
mvn -q spring-boot:run                       # 开发期直接跑
mvn -q clean package                          # 打出可执行 jar
java -jar target/demo-0.0.1-SNAPSHOT.jar      # 生产启动方式
curl -s http://127.0.0.1:8080/api/users/1     # 冒烟验证
```

常见坑：

1. `Web server failed to start. Port 8080 was already in use`：端口被占。用 `netstat -ano | findstr :8080` 找到 PID 后处理，或改 `server.port`，不要在不明进程的情况下直接杀。
2. `Field userMapper in ... required a bean of type 'UserMapper' that could not be found`：一半是启动类不在最外层包导致扫描不到，另一半是 MyBatis 的 Mapper 接口忘了加 `@Mapper` 或启动类忘了 `@MapperScan`。
3. `@Transactional` 写了却没回滚：先确认是不是自调用，再确认异常类型（受检异常要显式写 `rollbackFor = Exception.class`），最后确认异常没有被上层 catch 吞掉。
4. 配置改了不生效或启动报占位符解析失败：检查 Profile 是否真的激活（启动日志里有一行 `The following 1 profile is active`），以及 `${XXX}` 在没有默认值时是否缺少对应环境变量。

## 11. 工程实践与常见坑

工程实践（Engineering Practice，把「本机能跑」的代码变成「别人敢改、线上敢放、出问题能查」的一整套习惯）不是编码风格的洁癖，而是把一类特殊缺陷挡在发布之前。这类缺陷的特点是：编译器不报错，单元测试不写就发现不了，上了线只在特定数据量、特定并发、特定金额下爆发。下面按「语义约定 → 数值精度 → 字符串与包装类 → 集合 → 资源 → 测试 → 静态检查 → 线上排查 → 学习路径」的顺序，把 Java 工程里复现率最高的坑逐个拆开。

### 11.1 先分清哪些错编译器能挡、哪些挡不住

Java 的类型系统相当严格，但它只保证「类型对得上」，不保证「语义对」。把这两类问题分开，排查效率会立刻不同：

| 缺陷类型 | 编译器是否报错 | 典型例子 | 靠什么发现 |
| --- | --- | --- | --- |
| 类型不匹配 | 报错 | 把 `String` 赋给 `int` | 编译 |
| 未处理受检异常 | 报错 | 忘了 `catch IOException` | 编译 |
| 语义约定违反 | 不报 | 重写 `equals` 不重写 `hashCode` | 代码评审、单测 |
| 数值精度丢失 | 不报 | 用 `double` 存金额 | 金额对账、断言 |
| 资源与状态泄漏 | 不报 | `ThreadLocal` 用完不 `remove` | 压测、堆转储 |
| 并发与顺序问题 | 不报 | `HashMap` 多线程写入 | 压测、静态检查 |

四条可以直接当评审清单用的底线：一是语义正确优先于写法巧妙，能一眼看懂的 `for` 循环胜过炫技的流式嵌套；二是失败路径必须显式，每个 `catch` 要么处理要么带上上下文抛出；三是可测试，依赖通过构造器注入而不是在方法里 `new`；四是可观测，关键分支打日志并带上业务主键，没有主键的日志在排障时等于没有。

### 11.2 equals 与 hashCode：一对必须同时重写的契约

`Object.equals` 的默认实现是引用比较，`Object.hashCode` 的默认实现是对象地址派生值。只要重写了其中一个，就必须重写另一个，因为散列表（`HashMap`、`HashSet`、`ConcurrentHashMap`）的查找分两步：先用 `hashCode` 定位桶，再用 `equals` 在桶内比对。只重写 `equals` 时，两个「业务上相等」的对象算出的哈希不同，会被放进不同的桶，于是 `map.get(key)` 返回 `null`、`set.contains(x)` 返回 `false`、`distinct()` 去重失效。

契约有五条：自反（`x.equals(x)` 为真）、对称（`a.equals(b)` 与 `b.equals(a)` 同真同假）、传递、一致（对象不变则结果不变）、与 `null` 比较恒为假；外加一条硬约束——`equals` 相等则 `hashCode` 必须相等，反之不要求。

```java
public final class Order {
    private final String orderNo;   // 业务主键，天然唯一
    private final BigDecimal amount;

    public Order(String orderNo, BigDecimal amount) {
        this.orderNo = orderNo;
        this.amount = amount;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                       // 同一引用，直接短路
        if (o == null || getClass() != o.getClass()) return false; // 用 getClass 避免子类破坏对称性
        Order other = (Order) o;
        return Objects.equals(orderNo, other.orderNo);    // 只比业务主键，不比可变字段
    }

    @Override
    public int hashCode() {
        return Objects.hash(orderNo);                      // 与 equals 使用同一组字段
    }
}
```

两个高频细节。第一，参与 `hashCode` 的字段必须是不可变的：如果 `orderNo` 有 setter，对象放进 `HashSet` 后再改字段，哈希值变了，桶位置没变，这个对象就永远找不回来了，`set.size()` 还会骗人。第二，用 Lombok 的 `@Data` 或 `@EqualsAndHashCode` 时默认把全部字段纳入比较，既有可变字段的风险，也有性能开销（每次比较遍历十几个字段）；正确做法是 `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` 并只标注业务主键，继承场景再加 `callSuper = true`。

另一种省事的写法是用记录类（`record`，Java 16 正式引入的不可变数据载体）：`record Order(String orderNo, BigDecimal amount) {}` 会自动生成基于全部组件的 `equals`、`hashCode` 与 `toString`。它适合 DTO 与值对象，因为组件天然是 `final`，不存在「放进去又被改」的风险；但它的相等性是「所有字段都相等」，与「业务主键相同即相等」不是一回事——两个订单号相同、金额被修正过的 `record` 会被判为不等。需要按主键判等时，仍然要手写这两个方法。

### 11.3 BigDecimal：金额、税率、折扣的唯一正确选择

`float` 与 `double` 用二进制表示小数，`0.1` 无法精确表示，所以 `0.1 + 0.2 == 0.30000000000000004`。金额、单价、汇率、积分这类「必须精确到分」的量只能用 `BigDecimal`（任意精度十进制数，内部由未缩放整数与 scale 两部分组成）。

三个必踩的点，都能用下面这段代码看明白：

```java
// 1. 构造器：double 入参会把二进制误差原样放大
BigDecimal bad  = new BigDecimal(0.1);      // 0.1000000000000000055511151231257827...
BigDecimal good = new BigDecimal("0.1");    // 精确的 0.1，推荐
BigDecimal ok   = BigDecimal.valueOf(0.1);  // 内部走 Double.toString，效果等同字符串构造

// 2. 除法：除不尽且不指定精度会抛 ArithmeticException
BigDecimal total = new BigDecimal("19.99").multiply(new BigDecimal("3")); // 59.97
BigDecimal avg = total.divide(new BigDecimal("3"), 2, RoundingMode.HALF_UP); // 19.99
// BigDecimal wrong = total.divide(new BigDecimal("3")); // 抛异常：Non-terminating decimal expansion

// 3. 比较：equals 连 scale 一起比，业务判断要用 compareTo
new BigDecimal("1.0").equals(new BigDecimal("1.00"));     // false，scale 不同
new BigDecimal("1.0").compareTo(new BigDecimal("1.00"));  // 0，数值相等

// 4. 不可变：所有运算返回新对象，丢弃返回值等于没算
total.setScale(1, RoundingMode.DOWN);                     // 返回值被丢弃，total 仍是 59.97
BigDecimal rounded = total.setScale(1, RoundingMode.DOWN); // 59.9
```

舍入模式要按业务选：`HALF_UP` 是「四舍五入」，也是国内计价最常用的；`HALF_EVEN` 是「银行家舍入」，遇到 .5 时向偶数靠拢，多次累加时偏差更小，适合利息与统计。工程上还要把精度约束写进整条链路：数据库列用 `DECIMAL(18,2)` 而不是 `DOUBLE`；JPA 实体上写 `@Column(precision = 18, scale = 2)`；Jackson 序列化加上 `JsonWriteFeature.WRITE_BIGDECIMAL_AS_PLAIN`，否则 `100` 可能被输出成 `1E+2`，前端解析成字符串后再做运算就出错；对外接口的 JSON 里金额用字符串而不是数字，避免 JavaScript 的 `Number` 精度只有 53 位。

### 11.4 字符串拼接与包装类：两个最安静的坑

`String` 是不可变对象（Immutable，创建后内容不可修改），每次 `+` 都会产生新对象。编译器会把**单个表达式**里的拼接改写成 `StringBuilder`，但在循环体内，每次迭代都是一次独立的拼接表达式，等于每轮都 `new StringBuilder()` 再 `toString()`，时间复杂度退化到 O(n²)，十万级数据就能吃掉几百 MB 临时内存。

```java
String s = "";
for (int i = 0; i < 10000; i++) s += i;   // 每轮都新建 StringBuilder + String，产生 2 万个临时对象

StringBuilder sb = new StringBuilder(64 * 1024); // 预估容量，减少扩容时的数组复制
for (int i = 0; i < 10000; i++) sb.append(i);
String result = sb.toString();

// 日志场景：占位符延迟求值，level 不匹配时连参数转字符串都不做
log.debug("order={} amount={}", orderNo, amount);
```

包装类（Wrapper Class，为每个基本类型提供的对象形式）的坑集中在自动装箱与拆箱：`Integer` 会缓存 `-128` 到 `127` 之间的实例，因此小整数用 `==` 比较偶尔「看起来是对的」，一旦超出缓存区间就变成引用比较而返回 `false`。这条规则的隐蔽之处在于测试数据常常在缓存区间内，线上数据不在。

```java
Integer a = 127, b = 127;
System.out.println(a == b);        // true：命中 IntegerCache
Integer c = 128, d = 128;
System.out.println(c == d);        // false：各自 new 了对象
System.out.println(c.equals(d));   // true：正确写法

Map<String, Integer> counter = new HashMap<>();
Integer v = counter.get("missing"); // null
// int n = v;                       // NullPointerException：拆箱发生在赋值瞬间
int n = v == null ? 0 : v;          // 显式兜底
```

同一类问题还出现在三元表达式里：`flag ? 1 : integerValue` 中若一边是基本类型，编译器会对另一边强制拆箱，`integerValue` 为 `null` 时抛 NPE，而这段代码看上去毫无风险。

字符串本身还有一个更高频的错误：用 `==` 比较内容。字符串字面量会进入字符串常量池（String Pool，JVM 维护的字面量复用表），两个内容相同的字面量指向同一个实例，所以 `"a" == "a"` 为真；而 `new String("a")` 或从网络、文件、数据库读进来的字符串不在池里，`==` 立刻为假。`intern()` 能把手动创建的字符串登记进池并返回池内实例，但它占用元空间、且池的访问在高并发下有争用，只适合少量固定的枚举式取值，不要拿去「优化」动态拼接的结果。`switch` 支持字符串也只是语法糖：编译后先按 `hashCode` 分派、再用 `equals` 兜底，语义上仍然是内容比较。

### 11.5 集合的四个高频坑

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));

// 坑 1：for-each 中删除元素，抛 ConcurrentModificationException
for (String s : list) {
    if ("b".equals(s)) list.remove(s);
}
list.removeIf(s -> "b".equals(s));      // 正确：内部用迭代器 remove

// 坑 2：Arrays.asList 返回定长视图，底层就是原数组
List<String> fixed = Arrays.asList("a", "b");
fixed.add("c");                          // UnsupportedOperationException
List<String> ok = new ArrayList<>(Arrays.asList("a", "b"));
int[] arr = {1, 2, 3};
List<int[]> one = Arrays.asList(arr);    // 基本类型数组会被当成一个元素，size 是 1

// 坑 3：subList 是视图不是拷贝，原表结构一变它就失效
List<String> view = list.subList(0, 2);
list.add("d");
view.get(0);                             // ConcurrentModificationException
List<String> copy = new ArrayList<>(list.subList(0, 2)); // 正确：拷贝出来
```

第四个坑是 `HashMap` 的初始容量。`new HashMap<>(16)` 的 16 不是「能装 16 个元素」，而是桶数组长度，扩容阈值是容量乘负载因子 0.75，所以 16 个桶只能无扩容装 12 个元素。已知元素数量 n 时，正确写法是 `new HashMap<>((int) (n / 0.75f) + 1)`，可以省掉多次 rehash 与数组复制。

前三类错误报出的 `ConcurrentModificationException` 值得单独说一句，因为很多人以为它只在多线程下出现。它的判定依据是集合内部的修改计数 `modCount`：迭代器创建时记下快照，每次取元素前比对一次，一旦发现集合被别的路径改过就抛出。单线程在 for-each 里删除元素同样会触发，这不是并发问题，而是「用错了删除入口」。

| 坑 | 现象 | 根因 | 正确做法 |
| --- | --- | --- | --- |
| for-each 删除 | `ConcurrentModificationException` | 迭代器 modCount 与集合不一致 | `removeIf` 或显式 `Iterator.remove` |
| `Arrays.asList` | 加元素抛不支持异常 | 返回 `Arrays$ArrayList` 定长视图 | `new ArrayList<>(...)` |
| `List.of` | 加元素、存 `null` 都抛异常 | 真正的不可变集合 | 需要可变就显式拷贝 |
| `subList` | 原表修改后访问视图报错 | 视图持有原表引用 | 需要独立副本就拷贝 |
| `HashMap` 容量 | 频繁扩容、`put` 变慢 | 阈值是容量 × 0.75 | 按预估值设初始容量 |

### 11.6 空指针与资源泄漏：两类最贵的运行时问题

NPE 的来源就那么几种：方法返回 `null`、自动拆箱、数组越界其实不是 NPE 但常一起出现、集合里的元素为 `null`、链式调用中途为 `null`、并发下对象被另一个线程置空。防御手段按成本从低到高是：`Objects.requireNonNull` 在入口快速失败并给出参数名；返回空集合而不是 `null`；用 `Optional` 表达「可能没有」但只用在返回值上，不要当字段或方法参数。

定位 NPE 不必靠猜。JDK 14 起默认开启详细异常信息（`-XX:+ShowCodeDetailsInExceptionMessages`），消息会写成 `Cannot invoke "String.length()" because "user.name" is null`，直接把为 `null` 的表达式链点出来；编译时保留调试信息（`javac -g`，Maven 默认开启）才能让栈里的行号对上源码。此外，在方法入口用 `Objects.requireNonNull(orderId, "orderId")` 快速失败，报错会带参数名并停在调用边界，比等到第八十行才抛出的 NPE 省下的排查时间以小时计。

`Optional`（Java 8 引入的容器对象，用类型显式表达「值可能不存在」）被误用最典型的是 `optional.get()` 前不做判断，那只是把 NPE 换成了 `NoSuchElementException`。`orElse` 与 `orElseGet` 的区别也值得记：前者无论是否需要都会执行参数表达式，参数里有数据库查询或远程调用时必须用后者。

资源释放要交给 try-with-resources（自动关闭资源的语法，编译后生成 `finally` 块）：关闭顺序是声明的逆序，且关闭时的异常会被 `addSuppressed` 挂到主异常上，不会吞掉原始错误。

```java
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(SQL)) {
    ps.setLong(1, orderId);
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) { /* 处理结果 */ }
    }
} catch (SQLException e) {
    throw new BizException("QUERY_FAILED", e);   // 带上下文抛出，别只 e.printStackTrace()
}
```

比连接泄漏更隐蔽的是 `ThreadLocal`（线程本地变量，每个线程持有独立副本）没清理。线程池里的线程会长期存活，`ThreadLocalMap` 的 key 是弱引用、value 是强引用，线程对象活着，value 就一直被持有：

```java
private static final ThreadLocal<UserContext> CTX = new ThreadLocal<>();

public void handle(Request req) {
    CTX.set(buildContext(req));
    try {
        doBusiness();
    } finally {
        CTX.remove();   // 必须 remove：否则内存泄漏，且下一个请求会读到上一个用户的身份
    }
}
```

### 11.7 JUnit 5 与 Mockito：把坑变成断言

JUnit 5（Java 主流测试框架的第五代，由 Jupiter 引擎提供注解与断言）与 Mockito（模拟对象框架，生成依赖的假实现以隔离外部系统）配合，是单测的标准组合。关键是测「自己的逻辑」，不是测框架与数据库。

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock UserRepository repo;          // 假实现，不连数据库
    @InjectMocks UserService service;   // 把 mock 注入到被测对象

    @Test
    void 用户不存在时抛出业务异常() {
        when(repo.findById(1L)).thenReturn(Optional.empty());

        BizException ex = assertThrows(BizException.class, () -> service.get(1L));

        assertEquals("USER_NOT_FOUND", ex.getCode());
        verify(repo, times(1)).findById(1L);   // 校验只查了一次，防止重复查询
    }
}
```

边界值用参数化测试写，比复制十个方法更易维护：

```java
@ParameterizedTest(name = "{0} + {1} = {2}")
@CsvSource({"0.1,0.2,0.30", "1.005,0,1.01", "0,0,0.00"})
void 金额相加保留两位(String a, String b, String expect) {
    BigDecimal sum = new BigDecimal(a).add(new BigDecimal(b)).setScale(2, RoundingMode.HALF_UP);
    assertEquals(new BigDecimal(expect), sum);   // scale 一致，equals 才成立
}
```

测试的价值分布并不均匀，最值得写的三类是：服务层的分支逻辑、金额与时间的换算、边界的空值与越界。Controller 上的注解解析、纯 getter/setter、框架自身的配置不值得写。mock 也要克制，一个方法里 mock 了五个依赖、`when` 写了二十行，通常说明被测类职责过重，改结构比补测试有效；协作者是纯计算工具类时直接用真实对象，反而更接近线上行为。

还有几条实测经验：不要 mock 值对象和 DTO，直接 `new` 出来即可；`mock` 静态方法与 `final` 类需要 Mockito 5 默认的 inline mock maker；异步逻辑用 `Awaitility` 轮询或 `CountDownLatch` 同步，绝不用 `Thread.sleep(1000)` 赌时间；一个测试只验证一个行为主题，失败时才能从方法名读出原因；覆盖率是观察指标而不是目标，分支覆盖更能反映边界是否被测到。

### 11.8 代码规范与静态检查：把评审意见写成规则

规范（Coding Standard，团队对命名、注释、异常、日志等的统一约定）的价值在于降低阅读成本，国内团队普遍参考《阿里巴巴 Java 开发手册》，Google Java Style 则更偏格式。规范里最该被执行的是几条硬规则：常量替代魔法值、方法不超过一屏、异常不吞、日志带主键、`switch` 必须有 `default`。

光靠人评审不可持续，要用工具把规则变成 CI 门禁。四类工具分工不同：Checkstyle 看源码格式与命名，SpotBugs 看字节码层面的缺陷模式（例如用 `==` 比较字符串、忽略返回值、`SimpleDateFormat` 作为静态字段），PMD 看重复代码与坏味道，SonarQube 汇总技术债并挂在 MR 上做增量门禁。

```xml
<plugin>
  <groupId>com.github.spotbugs</groupId>
  <artifactId>spotbugs-maven-plugin</artifactId>
  <version>4.8.6.4</version>
  <configuration>
    <effort>Max</effort>          <!-- 分析深度：Max 最慢但发现率最高 -->
    <threshold>Medium</threshold> <!-- 只报告 Medium 及以上 -->
    <failOnError>true</failOnError><!-- 发现问题就让构建失败 -->
  </configuration>
</plugin>
```

接入节奏建议分三步：先只做报警不阻断，跑两周统计噪声；再把 `Blocker`、`Critical` 设为阻断，其余降级为报告；最后加上「新增代码覆盖率不低于 60%」的增量门禁。一次性把历史代码全量纳入阻断，团队只会选择加 `@SuppressWarnings` 绕过。

日志同样属于规范范畴，而且是最便宜的观测手段。三条惯例：用占位符而不是字符串拼接，级别判定在前、参数求值在后；异常必须把 `cause` 一起传出，只打 `e.getMessage()` 会丢掉栈；一次请求的上下文用 MDC 塞 `traceId`，散落在多个类里的日志才能串成一条链路。反向的规则是别把带手机号、身份证、银行卡的 `toString` 直接打进日志，脱敏要收口在日志适配层，不能指望每个开发者都记得。

### 11.9 两个真实排查案例

**案例一：老年代持续上涨，Full GC 后不下降。** 这类内存泄漏（Memory Leak，对象已不再被业务使用，却仍被 GC Root 可达引用链持有）的排查是固定动作：

```bash
jcmd 12345 GC.heap_info                  # 看各代占用与 GC 次数，确认是堆内问题
jmap -histo:live 12345 | head -20        # 触发一次 Full GC 后按存活对象数量排名
jmap -dump:format=b,file=/tmp/heap.hprof 12345   # 导出堆转储，用 MAT 分析
```

三份数据能快速定位：`jmap -histo` 里某个业务类实例数异常大，说明缓存无界；MAT 的 Dominator Tree 看谁占内存最多，Path to GC Roots 看引用链的起点是 `static` 字段还是线程。常见元凶是本地 `static Map` 做缓存、监听器注册后没注销、`ThreadLocal` 不 `remove`。修复方式不是「定期 clear」，而是给缓存加上界：

```java
Cache<Long, Report> cache = Caffeine.newBuilder()
        .maximumSize(10_000)                       // 硬上限，杜绝无界增长
        .expireAfterWrite(Duration.ofMinutes(10))  // 写入后过期，保证数据不太旧
        .recordStats()                             // 打开命中率统计，便于调参
        .build();
```

**案例二：某个实例 CPU 打到 100%。** 先把进程内的线程定位出来，再对号入座看栈：

```bash
top -Hp 12345              # 找到最耗 CPU 的线程号，例如 12388
printf '%x\n' 12388        # 转成十六进制 3064，jstack 里的 nid 就是这个值
jstack 12345 | grep -A 20 'nid=0x3064'
```

栈顶若落在业务代码的死循环或正则匹配上，基本就是它；若全部是 `GC task thread`，问题不在代码而在堆，回到案例一。正则导致 CPU 打满是经典陷阱，嵌套量词会触发指数级回溯：

```java
// 输入 30 个 a 后接一个 !，(a+)+$ 会尝试所有切分方式，耗时呈指数增长
boolean hit = Pattern.compile("(a+)+$").matcher("aaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!").matches();
```

线上不方便重启时用 Arthas 的 `thread -n 3` 看最忙的三个线程，`trace 类名 方法名` 看单次调用耗时分布，`watch` 观察入参出参，都不需要改代码。

### 11.10 学习路径：三个阶段与验收标准

Java 的知识面很宽，但学习顺序有强依赖。下面这张表把每个阶段的目标写成可验收的动作，而不是「熟悉」「了解」这类无法判断的词。

| 阶段 | 重点内容 | 验收标准 | 常见走弯路 |
| --- | --- | --- | --- |
| 入门 1–2 个月 | 语法、面向对象、集合、异常、IO | 能不看资料写出带泛型的工具类与自定义异常 | 一上手就学框架，只会复制配置 |
| 进阶 3–6 个月 | 并发、JVM 内存与 GC、反射与动态代理 | 能解释线程池七参数并手写一个可复现的死锁 | 背八股不理解，遇到栈就慌 |
| 工程 6–12 个月 | Spring 生态、数据库与事务、测试与 CI | 能独立把一个服务从开发推到上线并配好监控 | 只写业务代码不写测试与日志 |

配套的练习方式比看视频有效得多：读 JDK 源码时从 `java.util.ArrayList`、`java.util.HashMap` 这类两千行以内的类开始，边读边写注释；每学一个坑就写一个能复现它的单元测试，把「知道」变成「验证过」；每月挑一次线上问题做复盘，按「现象—数据—假设—验证—根因—改进」六段写下来，这份积累在面试和晋升时比任何证书都硬。

### 11.11 面试高频速查表

面试的差距很少体现在结论上，而体现在被追问「为什么」时能不能接着往下说。下面这张表把最容易被追问的问题压成速查格式：左列是主题，中列是必须能一口气说清的结论，右列是追问一般会往哪个方向挖。

| 主题 | 一句话答案 | 常见追问 |
| --- | --- | --- |
| equals/hashCode | 相等则哈希必相等，重写必须成对且字段不可变 | 为什么 `getClass` 优于 `instanceof` |
| BigDecimal | 用字符串构造，除法必带精度与舍入模式 | `equals` 与 `compareTo` 差别 |
| 字符串拼接 | 循环内用 `StringBuilder` 并预估容量 | 编译期做了什么优化 |
| `Integer` 比较 | `-128~127` 有缓存，比较值必须 `equals` | 为什么设计缓存 |
| 集合删除 | 用 `removeIf` 或迭代器，不要 for-each 直接删 | fail-fast 的原理 |
| `HashMap` 容量 | 阈值是容量 × 0.75，按元素数反推容量 | 扩容时链表怎么拆 |
| 内存泄漏 | 长生命周期对象持有短生命周期对象 | 怎么用 MAT 定位 |
| 线程池 | 核心数、队列、最大数、拒绝策略要一起看 | 为什么不用 `Executors` |
| `Optional` 用法 | 只用于返回值，取值前先判断 | `orElse` 与 `orElseGet` 什么时候有区别 |
| 单测 | 测行为不测实现，外部依赖用 mock | 怎么测异步与时间 |
| 排障 | 先看数据再看代码，逐步缩小范围 | 堆转储里怎么找引用链 |
