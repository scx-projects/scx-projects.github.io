# SCX Parent

SCX Parent 是 SCX 项目的 Maven 父工程。

它不是运行时库，也不是业务框架，而是一个用于统一 Maven 构建配置的 parent pom。

它主要负责统一管理：

```text
Java 版本
源码编码
Maven 插件版本
编译配置
测试配置
Jar 打包配置
源码包生成
Javadoc 包生成
GPG 签名
Central 发布配置
依赖复制目录
```

一般情况下，SCX 系列模块都会继承 `scx-parent`，从而获得一致的构建行为。

[GitHub](https://github.com/scx-projects/scx-parent)

## Maven

### 作为父工程使用

在项目的 `pom.xml` 中添加：

```xml
<parent>
    <groupId>dev.scx</groupId>
    <artifactId>scx-parent</artifactId>
    <version>1</version>
    <relativePath/>
</parent>
```

完整示例：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>dev.scx</groupId>
        <artifactId>scx-parent</artifactId>
        <version>1</version>
        <relativePath/>
    </parent>

    <artifactId>your-project</artifactId>
    <version>0.0.1</version>

</project>
```

`relativePath` 推荐保持为空：

```xml
<relativePath/>
```

这表示 Maven 不从本地相对路径查找父工程，而是从本地仓库或远程仓库解析 `scx-parent`。

## 基本概念

SCX Parent 中最核心的概念包括：

```text
parent pom           Maven 父工程
pluginManagement     Maven 插件版本和默认配置
properties           统一属性
scx.mainClass        应用入口类
scx.libraryDirectory 依赖库复制目录
```

它们之间的关系可以简单理解为：

```text
子项目 pom.xml
    ↓
继承 scx-parent
    ↓
获得统一 Maven 构建配置
    ↓
使用 Maven 构建项目
```

也就是说：

```text
scx-parent 负责构建约定
子项目负责声明自己的 artifactId、version、dependencies 和业务代码
```

## 它不是普通依赖

`scx-parent` 的 packaging 是：

```xml
<packaging>pom</packaging>
```

因此它不是普通 jar 依赖。

不要这样使用：

```xml
<dependency>
    <groupId>dev.scx</groupId>
    <artifactId>scx-parent</artifactId>
    <version>1</version>
</dependency>
```

正确用法是放在：

```xml
<parent>
    ...
</parent>
```

也就是说：

```text
scx-parent 是父 POM
不是运行时依赖
```

## 快速开始

### 1. 继承 scx-parent

```xml
<parent>
    <groupId>dev.scx</groupId>
    <artifactId>scx-parent</artifactId>
    <version>1</version>
    <relativePath/>
</parent>
```

### 2. 声明项目基本信息

```xml
<artifactId>my-app</artifactId>
<version>0.0.1</version>
```

### 3. 如果是可运行项目，设置主类

```xml
<properties>
    <scx.mainClass>com.example.Main</scx.mainClass>
</properties>
```

### 4. 正常使用 Maven 构建

```bash
mvn clean package
```


## 应用项目示例

如果你的项目是一个可以运行的应用，可以这样写：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>dev.scx</groupId>
        <artifactId>scx-parent</artifactId>
        <version>1</version>
        <relativePath/>
    </parent>

    <artifactId>my-app</artifactId>
    <version>0.0.1</version>

    <properties>
        <scx.mainClass>com.example.Main</scx.mainClass>
    </properties>

    <dependencies>
        <!-- your dependencies -->
    </dependencies>

</project>
```

主类示例：

```java
package com.example;

public class Main {

    public static void main(String[] args) {
        System.out.println("Hello SCX Parent");
    }

}
```

构建：

```bash
mvn clean package
```

运行：

```bash
java -jar target/my-app-0.0.1.jar
```

## 库项目示例

如果你的项目只是一个库模块，不需要直接运行，可以不配置 `scx.mainClass`。

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>dev.scx</groupId>
        <artifactId>scx-parent</artifactId>
        <version>1</version>
        <relativePath/>
    </parent>

    <artifactId>my-lib</artifactId>
    <version>0.0.1</version>

    <dependencies>
        <!-- your dependencies -->
    </dependencies>

</project>
```

库项目通常只需要：

```bash
mvn clean package
```

或者：

```bash
mvn clean install
```

## 默认属性

SCX Parent 统一设置了一些常用属性。

可以理解为：

```text
scx.javaVersion          25
scx.encoding             UTF-8
scx.libraryDirectory     lib
```

同时还会把 Maven 常用编码属性统一到 `scx.encoding`：

```text
project.build.sourceEncoding
project.reporting.outputEncoding
```

并把编译 release 绑定到：

```text
scx.javaVersion
```

因此默认情况下，继承 `scx-parent` 的项目会使用：

```text
Java 25
UTF-8
```

如果子项目需要覆盖，可以在自己的 `pom.xml` 中重新声明属性。

示例：

```xml
<properties>
    <scx.javaVersion>25</scx.javaVersion>
    <scx.encoding>UTF-8</scx.encoding>
</properties>
```

一般不建议随意覆盖这些属性，除非项目确实有特殊需求。

## scx.mainClass

`scx.mainClass` 用于指定可执行 jar 的入口类。

示例：

```xml
<properties>
    <scx.mainClass>com.example.Main</scx.mainClass>
</properties>
```

它会被 jar 插件用于生成 manifest 中的 `Main-Class`。

也会被 `exec-maven-plugin` 用于运行项目。

因此，如果项目需要通过下面方式运行：

```bash
mvn compile exec:java
```

或者需要生成可直接执行的 jar，就应该配置：

```xml
<scx.mainClass>...</scx.mainClass>
```

如果是普通库模块，可以不配置。

## scx.libraryDirectory

`scx.libraryDirectory` 用于指定依赖复制目录。

默认值是：

```text
lib
```

在 Maven dependency plugin 中，它用于把项目依赖复制到：

```text
target/lib
```

该目录也会作为可执行 jar manifest 中的 classpath 前缀。

执行：

```bash
mvn dependency:copy-dependencies
```

后，项目可以形成传统的 jar + lib 目录结构：

```text
target/
    my-app-0.0.1.jar
    lib/
        dependency-1.jar
        dependency-2.jar
```

## Maven 插件管理

SCX Parent 统一声明和管理多个 Maven 插件。

主要包括：

```text
maven-resources-plugin
maven-compiler-plugin
maven-surefire-plugin
maven-jar-plugin
maven-source-plugin
maven-javadoc-plugin
maven-gpg-plugin
maven-install-plugin
maven-deploy-plugin
maven-clean-plugin
maven-dependency-plugin
maven-project-info-reports-plugin
maven-site-plugin
exec-maven-plugin
central-publishing-maven-plugin
```

这些插件共同覆盖：

```text
资源处理
编译
测试
Jar 打包
源码包
Javadoc 包
GPG 签名
安装到本地仓库
部署到远程仓库
清理
依赖复制
项目报告
站点生成
运行 main class
发布到 Maven Central
```

子项目继承后，可以直接使用这些插件的默认版本和部分默认配置。

## Java 编译配置

SCX Parent 默认使用 Java 25。

它会把编译 release 设置为：

```text
${scx.javaVersion}
```

也就是：

```text
25
```

这意味着子项目编译时默认目标就是 Java 25。

如果你的项目运行环境不是 Java 25，需要在子项目中覆盖：

```xml
<properties>
    <scx.javaVersion>21</scx.javaVersion>
</properties>
```

但对于 SCX 系列项目，通常应保持和父工程一致。

## 编码配置

SCX Parent 默认编码是：

```text
UTF-8
```

这会影响：

```text
源码读取
资源处理
报告输出
Javadoc 输出
```

推荐所有子项目都使用 UTF-8，避免跨平台构建时出现中文乱码或资源乱码问题。

## Jar 打包配置

SCX Parent 配置了 `maven-jar-plugin`。

它主要处理：

```text
生成 jar
生成 manifest
设置 Main-Class
设置 Class-Path
```

当设置了：

```xml
<scx.mainClass>com.example.Main</scx.mainClass>
```

构建出的 jar manifest 中可以包含主类信息。

同时依赖库目录默认是：

```text
lib
```

因此可执行应用可以配合 `dependency:copy-dependencies` 形成这样的结构：

```text
target/
    my-app-0.0.1.jar
    lib/
        xxx.jar
        yyy.jar
```

## Source Jar

SCX Parent 配置了 `maven-source-plugin`。

它会生成源码包。

源码包通常用于发布到 Maven Central。

生成结果类似：

```text
target/my-lib-0.0.1-sources.jar
```

对于公开发布的库，源码包是非常重要的。

## Javadoc Jar

SCX Parent 配置了 `maven-javadoc-plugin`。

它会在构建过程中生成 Javadoc 包。

生成结果类似：

```text
target/my-lib-0.0.1-javadoc.jar
```

Javadoc 包通常也是发布到 Maven Central 所需要的内容之一。

## GPG 签名

SCX Parent 配置了 `maven-gpg-plugin`。

它用于发布时对构建产物签名。

典型产物包括：

```text
jar
pom
sources.jar
javadoc.jar
```

发布到 Maven Central 时通常需要签名文件。

需要注意，GPG 签名依赖本地或 CI 环境中的 GPG 配置。

父工程默认通过 `gpg.skip=true` 跳过签名；发布时需要显式启用签名，并准备本地或 CI 环境中的 GPG 配置。

## Central Publishing

SCX Parent 配置了 `central-publishing-maven-plugin`。

它用于把构建产物发布到 Maven Central。

它属于发布流程的一部分，不是普通本地开发必须关注的内容。

普通开发通常只需要：

```bash
mvn clean package
```

或者：

```bash
mvn clean install
```

只有需要发布到中央仓库时，才需要关注：

```text
Central Portal 账号
GPG 签名
Maven settings.xml
CI secrets
发布 profile 或发布命令
```

## exec-maven-plugin

SCX Parent 配置了 `exec-maven-plugin`。

它可以配合：

```xml
<scx.mainClass>...</scx.mainClass>
```

运行项目。

例如：

```bash
mvn compile exec:java
```

如果没有设置 `scx.mainClass`，运行项目时可能会失败。

## dependency plugin

SCX Parent 配置了 `maven-dependency-plugin`。

它主要用于复制依赖到指定目录。

默认目录和 `scx.libraryDirectory` 有关。

可以理解为：

```text
target/lib
```

这适合传统 jar + lib 目录的发布形式。

例如：

```bash
mvn dependency:copy-dependencies
```

该目标没有绑定到默认 Maven 生命周期，需要时应显式执行。

## CI 配置

项目可以配合 GitHub Actions 做 CI。

CI 的主要流程是：

```text
push
pull_request
workflow_dispatch
    ↓
checkout
    ↓
setup-java
    ↓
java-version: 25
distribution: temurin
cache: maven
    ↓
mvn package
    ↓
upload target/
```

这说明 `scx-parent` 自身也使用 Java 25 进行 CI 构建。

## Dependabot

仓库配置了 Dependabot。

更新范围包括：

```text
maven
github-actions
```

更新周期是：

```text
weekly
```

这用于自动检查 Maven 插件、依赖和 GitHub Actions 版本更新。

## 典型项目结构

继承 `scx-parent` 后，一个普通应用项目可以是：

```text
my-app
├── pom.xml
└── src
    └── main
        ├── java
        │   └── com
        │       └── example
        │           └── Main.java
        └── resources
            └── application.conf
```

其中：

```text
pom.xml             继承 scx-parent
src/main/java       Java 源码
src/main/resources  资源文件
```

## 完整应用 pom 示例

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>dev.scx</groupId>
        <artifactId>scx-parent</artifactId>
        <version>1</version>
        <relativePath/>
    </parent>

    <artifactId>demo-app</artifactId>
    <version>0.0.1</version>

    <properties>
        <scx.mainClass>com.example.demo.Main</scx.mainClass>
    </properties>

    <dependencies>
        <dependency>
            <groupId>dev.scx</groupId>
            <artifactId>scx-tcp</artifactId>
            <version>0.10.0</version>
        </dependency>
    </dependencies>

</project>
```

主类：

```java
package com.example.demo;

public class Main {

    public static void main(String[] args) {
        System.out.println("demo app started");
    }

}
```

运行：

```bash
mvn compile exec:java
```

构建：

```bash
mvn clean package
```


## 完整库 pom 示例

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>dev.scx</groupId>
        <artifactId>scx-parent</artifactId>
        <version>1</version>
        <relativePath/>
    </parent>

    <artifactId>demo-lib</artifactId>
    <version>0.0.1</version>

    <dependencies>
        <dependency>
            <groupId>dev.scx</groupId>
            <artifactId>scx-node</artifactId>
            <version>0.10.0</version>
        </dependency>
    </dependencies>

</project>
```

库项目一般不需要：

```xml
<scx.mainClass>...</scx.mainClass>
```

## 本地开发常用命令

### 编译

```bash
mvn compile
```

### 测试

```bash
mvn test
```

### 打包

```bash
mvn package
```

### 清理并打包

```bash
mvn clean package
```

### 安装到本地仓库

```bash
mvn clean install
```

### 运行应用

```bash
mvn compile exec:java
```

### 复制依赖

```bash
mvn dependency:copy-dependencies
```

## 发布相关命令

如果项目需要发布，一般会使用 Maven deploy 相关流程。

```bash
mvn clean deploy
```

发布流程可能涉及：

```text
source jar
javadoc jar
gpg sign
central publishing
settings.xml
credentials
CI secrets
```

这些配置通常不适合普通本地开发直接执行。

如果本地没有 GPG 或 Central Portal 配置，发布相关命令可能失败。

## 设计边界

SCX Parent 的边界非常明确。

它负责：

```text
统一 Maven 构建配置
统一插件版本
统一 Java 版本
统一编码
统一打包约定
```

它不负责：

```text
运行时代码
业务框架
依赖版本 BOM
自动生成项目结构
应用配置管理
测试框架封装
发布账号配置
CI secret 管理
```

这些应由子项目或其它工具负责。

## 与普通 parent pom 的关系

SCX Parent 本质上就是一个 Maven parent pom。

它和普通 parent pom 一样，可以让子项目继承通用配置。

不同之处在于，它包含了 SCX 项目自己的构建约定，例如：

```text
Java 25
UTF-8
lib 依赖目录
scx.mainClass
Central publishing 相关插件
```

因此它适合 SCX 系列项目使用。

如果外部项目使用，也应该理解这些默认约定。

## 与 BOM 的区别

SCX Parent 不是 BOM。

BOM 通常用于统一依赖版本：

```xml
<dependencyManagement>
    ...
</dependencyManagement>
```

而 SCX Parent 的主要职责是统一构建配置。

也就是说：

```text
BOM 管依赖版本
Parent 管构建约定
```

SCX Parent 更偏向：

```text
build parent
```

而不是：

```text
dependency BOM
```

## 多模块项目

如果你的项目是多模块 Maven 项目，可以让根 pom 继承 `scx-parent`。

示例：

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>dev.scx</groupId>
        <artifactId>scx-parent</artifactId>
        <version>1</version>
        <relativePath/>
    </parent>

    <artifactId>demo-parent</artifactId>
    <version>0.0.1</version>
    <packaging>pom</packaging>

    <modules>
        <module>demo-core</module>
        <module>demo-app</module>
    </modules>

</project>
```

子模块可以继承这个项目自己的根 pom。

```xml
<parent>
    <groupId>com.example</groupId>
    <artifactId>demo-parent</artifactId>
    <version>0.0.1</version>
</parent>
```

这种结构可以让整个项目间接继承 `scx-parent` 的构建配置。

## 设计说明

### 1. SCX Parent 是构建约定

它的核心价值不是提供 Java API。

它的核心价值是让多个项目使用同一套 Maven 构建规则。

例如：

```text
同一个 Java 版本
同一个编码
同一套插件版本
同一种 source/javadoc/sign/publish 配置
```

### 2. packaging 是 pom

因为它是父工程，所以 packaging 是：

```text
pom
```

它不会产出普通运行时 jar。

### 3. 子项目通过 parent 继承配置

子项目只需要声明：

```xml
<parent>
    ...
</parent>
```

就可以继承大部分构建配置。

这减少了每个子项目重复写插件配置的成本。

### 4. 应用主类由 scx.mainClass 指定

可运行项目需要设置：

```xml
<scx.mainClass>...</scx.mainClass>
```

这样 Maven 插件才能知道入口类。

### 5. lib 目录是传统应用打包约定

`scx.libraryDirectory` 默认是：

```text
lib
```

它适合 jar + lib 的应用分发方式。

这种方式简单、直观，也容易手动部署。

### 6. 发布配置不等于自动发布

SCX Parent 配置了发布相关插件。

但是否能发布，仍然取决于：

```text
GPG 配置
Maven settings.xml
Central Portal 凭据
CI secret
发布权限
```

父工程只是提供插件配置基础。

## 常见问题

### SCX Parent 是运行时库吗？

不是。

它是 Maven 父工程。

### 应该放在 dependencies 里吗？

不应该。

它应该放在：

```xml
<parent>
    ...
</parent>
```

### packaging 是什么？

```text
pom
```

### 默认 Java 版本是多少？

默认 Java 版本是：

```text
25
```

### 默认编码是什么？

默认编码是：

```text
UTF-8
```

### scx.mainClass 是干什么的？

用于指定应用入口类。

例如：

```xml
<scx.mainClass>com.example.Main</scx.mainClass>
```

它会影响 `exec-maven-plugin` 运行和 jar manifest 生成。

### 库项目需要 scx.mainClass 吗？

通常不需要。

只有可运行应用才需要。

### scx.libraryDirectory 默认是什么？

默认是：

```text
lib
```

用于复制依赖和生成 jar manifest classpath。

### 如何运行项目？

如果配置了 `scx.mainClass`，可以使用：

```bash
mvn compile exec:java
```

### 如何构建项目？

使用 Maven：

```bash
mvn clean package
```

### 如何复制依赖？

使用 Maven dependency plugin：

```bash
mvn dependency:copy-dependencies
```

### SCX Parent 会管理依赖版本吗？

它主要管理 Maven 构建插件和构建属性。

它不是依赖 BOM。

### 它会自动发布到 Maven Central 吗？

不会。

它只是配置了发布相关插件。

真正发布还需要凭据、签名、权限和发布命令。

### 本地没有 GPG 会影响普通构建吗？

默认配置会通过 `gpg.skip=true` 跳过签名。

发布时如果启用 GPG 签名，则需要准备对应的本地或 CI 配置。

### GitHub Actions 使用什么 Java 版本？

CI 中使用 Java 25。

### Dependabot 会更新什么？

会检查：

```text
Maven
GitHub Actions
```

更新周期是 weekly。

### 外部项目可以继承 SCX Parent 吗？

可以。

但需要接受它的构建约定，例如 Java 25、UTF-8、插件配置和 `scx.*` 属性约定。

### 如果我只想复用依赖版本，应该用它吗？

不应该。

SCX Parent 不是 BOM。

如果只想复用依赖版本，应使用 dependencyManagement/BOM 形式的模块。

### 为什么要单独做一个 parent 项目？

因为多个 SCX 子项目都需要类似的 Maven 构建配置。

把这些配置放进 parent，可以减少重复配置，并保证构建行为一致。
