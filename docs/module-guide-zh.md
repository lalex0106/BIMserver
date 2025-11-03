# BIMserver 多模块阅读与使用指南

本指南针对仓库中的每个 Maven 子模块提供中文概览、源码入口指引与常用操作说明，方便二次开发时快速定位功能与构建方式。

## 总体结构与构建方式

- 项目采用 Maven 聚合工程结构，根 `pom.xml` 声明了 `Bdb`、`BimServer`、`BimServerClientLib`、`PluginBase`、`Shared`、`BimServerWar` 与 `BimServerJar` 等核心子模块，并提供统一的编译器版本与插件管理配置。【F:pom.xml†L1-L155】
- 仓库顶层同时包含 `Documentation`、`docs`、`Tests` 等辅助目录；其中 `docs` 目录用于维护额外文档资源，是本文档的存放位置。【68cf58†L1-L4】
- 统一构建命令：在仓库根目录执行 `mvn clean install` 即可依次编译所有子模块；如需运行集成测试，可附加 `-Ptest` 以启用 `Tests` 模块的 profile。【F:pom.xml†L44-L154】

> **提示**：各子模块共享父 POM 中的 Java 17 目标平台与日志、Jetty、CXF 等版本约束，二次开发时若需调整版本，建议在父模块统一修改后再对子模块进行最小化覆盖。

## Bdb 模块

### 模块定位

`Bdb` 提供对 Berkeley DB Java Edition (`com.sleepycat:je`) 的最小封装，使核心服务能够以独立的 JAR 形式复用该嵌入式键值数据库。【F:Bdb/pom.xml†L1-L33】

### 目录结构与入口

- 模块以 Maven `jar` 打包，不包含额外源码目录，主要负责向聚合工程暴露 Berkeley DB 依赖，并注册 Oracle 官方仓库地址以解析依赖。【F:Bdb/pom.xml†L9-L25】

### 使用建议

- 如需扩展自定义的存储适配器，可在 `BimServer` 模块内扩写数据库集成逻辑，而 `Bdb` 保持轻量依赖模块的定位，仅在必要时添加新的数据库驱动坐标。

## Shared 模块

### 模块定位

`Shared` 是 `BimServer` 与 `BimServerClientLib` 之间的公共代码基，包括 Servlet 接口、命令行工具、EMF/Protobuf 模型等通用组件。【F:Shared/pom.xml†L2-L55】【F:Shared/README.md†L1-L1】

### 目录结构

- 代码位于 `Shared/src/org/bimserver` 下，按功能拆分为 `database`、`ifc`、`plugins`、`reflector`、`shared`、`utils` 等子包，覆盖模型定义与通用工具。【9675df†L1-L9】

### 使用建议

- 开发面向服务器与客户端共享的基础能力（如数据模型、传输协议、缓存工具）时，优先在 `Shared` 模块实现，避免重复实现。
- `Shared` 依赖 `pluginbase`、`protobuf-java`、`fastutil` 等库，开发新功能时应确保版本兼容并遵循父模块指定的依赖范围。【F:Shared/pom.xml†L15-L55】

## PluginBase 模块

### 模块定位

`PluginBase` 为插件开发提供编译期 API，涵盖插件生命周期管理、几何处理、EMF 访问等基础能力，便于在外部仓库创建自定义插件。【F:PluginBase/pom.xml†L2-L191】【F:PluginBase/README.md†L1-L1】

### 目录结构

- 源码划分在 `client`、`bimbots`、`database`、`emf`、`geometry`、`plugins`、`shared`、`utils` 等子目录，覆盖从客户端通信到几何转换的常用工具类。【95eb0e†L1-L8】
- `generated` 目录可通过构建阶段的 `build-helper-maven-plugin` 自动加入源码路径，适用于代码生成场景。【F:PluginBase/pom.xml†L12-L43】

### 使用建议

- 编写插件时仅需在外部项目依赖 `pluginbase`（必要时叠加 `shared`），即可获得 HTTP 客户端、Maven 解析器、Jackson/Guava 等工具链支持。【F:PluginBase/pom.xml†L44-L191】
- 若插件需要额外的第三方库，建议在插件项目自身声明，避免修改 `PluginBase` 的公共依赖集合。

## BimServer 模块

### 模块定位

`BimServer` 是核心服务模块，提供 EMF 模型存储、数据库迁移、SOAP/JSON/Protocol Buffers API、通知系统、缓存与插件管理等完整服务器功能。【F:BimServer/README.md†L1-L8】

### 目录结构

- 模块包含 `deploy`（服务器默认配置）、`emailtemplates`、`generated`（代码生成输出）、`models`（IFC 模式定义）、`templates` 与 Web 前端资源 `www` 等目录，支撑服务端运行时的多种资产。【e865f9†L1-L3】
- 核心源码位于 `src/org/bimserver`，`BimServer` 类作为启动入口，负责初始化数据库、插件、缓存、通知、嵌入式 Web 服务器等子系统。【F:BimServer/src/org/bimserver/BimServer.java†L162-L313】

### 运行与扩展

- 通过构造 `BimServerConfig` 可指定 `home` 目录、端口、资源加载方式、是否启动内嵌 Web 服务器等运行参数；核心类会在启动时创建临时目录、注册插件仓库并初始化服务工厂。【F:BimServer/src/org/bimserver/BimServer.java†L216-L296】
- 若需增加新的通知、渲染或合并策略，可扩展相应的 `PluginManager`、`RenderEnginePools` 等组件，并在配置中注册插件包。
- Web 层默认由内嵌 Jetty 托管，Servlet 入口位于 `RootServlet` 等类中，可结合 `BimServerWar` 或 `BimServerJar` 的部署方案进行调试。

## BimServerClientLib 模块

### 模块定位

`BimServerClientLib` 提供 Java 客户端 API，支持 JSON、SOAP 与 Protocol Buffers 三种通信方式，可选择直接内嵌模式或远程调用。【F:BimServerClientLib/README.md†L1-L1】【F:BimServerClientLib/pom.xml†L2-L61】

### 目录结构

- 源码集中在 `src/org/bimserver/client` 下，按协议划分为 `json`、`soap`、`protocolbuffers` 以及 `notifications` 等子包，方便根据通信需求选择实现。【a74b54†L1-L5】
- 测试代码位于 `test` 目录，可用于验证不同协议的互操作性。【F:BimServerClientLib/pom.xml†L11-L21】

### 使用建议

- 在外部应用中引入该库后，可通过 `DirectBimServerClientFactory` 建立与服务器的同 JVM 连接，或改用 CXF/Jetty WebSocket 依赖完成远程访问。【F:BimServerClientLib/pom.xml†L23-L60】
- 二次开发常见场景包括：封装业务 API、集成通知推送、批量上传 IFC 模型等，建议结合 `Shared` 模块提供的模型类进行序列化与验证。

## BimServerJar 模块

### 模块定位

`BimServerJar` 打包可执行的独立 JAR，便于快速评估或单机部署，无需外置应用服务器。【F:BimServerJar/README.md†L1-L1】【F:BimServerJar/pom.xml†L2-L121】

### 目录结构与关键类

- `src/org/bimserver/JarBimServer.java` 提供命令行入口，解析 `address`、`port`、`homedir` 参数后创建 `BimServerConfig`、启动嵌入式 Jetty 并挂载静态资源；同时自动配置日志输出目录。【F:BimServerJar/src/org/bimserver/JarBimServer.java†L54-L168】
- `EmbeddedWebServer` 基于 Jetty 配置 Servlet、WebSocket 及 Jolokia 监控端点，并将 `RootServlet` 映射到根路径，实现内嵌 Web UI。【F:BimServerJar/src/org/bimserver/EmbeddedWebServer.java†L25-L101】
- 模块包含 `settings.properties` 以记录默认主机、端口、堆大小等运行参数，可在打包后按需调整。【F:BimServerJar/settings.properties†L1-L12】

### 使用建议

- 打包命令：`mvn -pl BimServerJar -am package` 将生成 `starter.jar`（Assembly 插件指定名称），适合快速启动本地服务。【F:BimServerJar/pom.xml†L35-L63】
- 调试插件或前端资源时，可直接修改 `BimServer/www` 目录并通过独立 JAR 运行验证。

## BimServerWar 模块

### 模块定位

`BimServerWar` 负责生成部署到外部 Servlet 容器（如 Tomcat、Jetty）的 WAR 包，适合与企业现有基础设施集成。【F:BimServerWar/README.md†L1-L1】【F:BimServerWar/pom.xml†L2-L105】

### 目录结构与资源打包

- `deploy/logback.xml`、`extra/*.html`、`license.txt` 等资源会在打包过程中被复制到相应的 WAR 目录；来自 `BimServer` 模块的邮件模板、Web 静态文件和部署配置也会被合并进 `WEB-INF` 或根目录。【F:BimServerWar/pom.xml†L12-L80】【6dfe87†L1-L2】
- `web.xml` 定义了 WAR 的 Servlet 配置，可在模块中自定义过滤器或监听器以满足部署需求。【F:BimServerWar/pom.xml†L12-L82】

### 使用建议

- 执行 `mvn -pl BimServerWar package` 生成 WAR 包后，可将输出部署至任何兼容 Servlet 3.1 的容器；如需自定义日志级别或静态页面，可修改 `deploy` 与 `extra` 目录下的资源文件。
- 若部署环境对插件目录有特殊要求，可借助父模块配置的 `plugins` 目录路径在运行时挂载自定义插件包。

## Tests 模块

### 模块定位

`Tests` 提供端到端测试套件，通过 JUnit Platform 启动完整的 BIMserver 实例、安装所需插件，并验证主要功能流程。【F:Tests/pom.xml†L1-L80】【F:Tests/test/org/bimserver/tests/AllTests.java†L45-L160】

### 运行流程

- `AllTests` 使用 `@Suite` 批注扫描 `org.bimserver.tests` 包中的测试用例；在 `beforeSuite` 阶段生成临时 `home` 目录、初始化 `BimServer`、启动嵌入式 Web 服务器并通过管理接口完成初始配置与插件安装。【F:Tests/test/org/bimserver/tests/AllTests.java†L45-L123】
- 测试结束时调用 `resetBimServer` 停止服务器并清理资源，确保多轮测试不会残留状态。【F:Tests/test/org/bimserver/tests/AllTests.java†L129-L160】

### 使用建议

- 默认 Surefire 目标为 `AllTests`，执行 `mvn -Ptest test` 即可触发整套测试；如需运行自定义用例，可在命令中覆盖 `-Dtest=YourTest` 或调整 `pom.xml` 中的 `test` 配置。【F:Tests/pom.xml†L21-L37】
- 二次开发时建议在该模块新增集成测试，复用已有的服务器启动流程，确保新功能与插件兼容性。

---

通过以上模块化说明，您可以根据业务需求快速定位到对应源码与配置位置，结合父 POM 统一的构建约束，完成 BIMserver 的功能扩展与部署定制。
