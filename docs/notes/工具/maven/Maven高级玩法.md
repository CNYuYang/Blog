# Maven 高级玩法

## 实用技巧

### Maven 提速

#### 多线程

```bash
# 用 4 个线程构建，以及根据 CPU 核数每个核分配 1 个线程进行构建
$ mvn -T 4 clean install
$ mvn -T 1C clean install
```

#### 跳过测试

```bash
-DskipTests               # 不执行测试用例，但编译测试用例类生成相应的 class 文件至 target/test-classes 下
-Dmaven.test.skip=true    # 不执行测试用例，也不编译测试用例类

# 结合上文的`并行执行`
$ mvn -T 1C clean install -Dmaven.test.skip=true

# 如果还是阻塞: 资源管理器 - shutdown all java app

# 如果 jar 包过大，可以下载按照路径放在 repository 中，之后可能还需要 mvn clean 来下载 groovy-all-2.3.11.pom 文件 (mvnrepository.com)
D:\apps\maven\repository\org\codehaus\groovy\groovy-all\2.3.11\groovy-all-2.3.11.jar
```

#### 编译失败后，接着编译

```bash
# 如果是 Apache Eagle 之类带有几十个子项目的工程，如果从头编译所有的模块，会很耗功夫
# 通过指定之前失败的模块名，可以继续之前的编译
$ mvn -rf :moduleName clean install
```

#### 跳过失败的模块，编译到最后再报错

```bash
$ mvn clean install --fail-at-end
```

#### 使用 Aliyun 国内镜像

```xml
<mirror>
  <id>alimaven</id>
  <name>aliyun maven</name>
  <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
  <mirrorOf>central</mirrorOf>
</mirror>
```



### 指定 Repository 目录

```xml
<!-- Default: ~/.m2/repository  -->
<localRepository>D:\apps\maven\repository</localRepository>
```

### 如何使用 Maven 编译指定 module

```bash
$ mvn install -pl <module_name> -am

  -pl, --projects      (Build specified reactor projects instead of all project)
  -am, --also-make     (If project list is specified, also build projects required by the list)
  -amd, --also-make-dependents    (If project list is specified, also build projects that depend on)
```

### 如何使用 Maven 编译跳过 module

```bash
$ mvn install -pl <!module_name>
```

### Maven 标准目录结构

```
src/main/java             Application/Library sources
src/main/resources        Application/Library resources
src/main/filters          Resource filter files
src/main/webapp           Web application sources
src/test/java             Test sources
src/test/resources        Test resources
src/test/filters          Test resource filter files
src/it                    Integration Tests (primarily for plugins)
src/assembly              Assembly descriptors
src/site                  Site
LICENSE.txt               Project's license
NOTICE.txt                Notices and attributions required by libraries that the project depends on
README.txt                Project's readme
```

### 如何在 Maven 中使用多个 source

```xml
<plugin>
  <groupId>org.codehaus.mojo</groupId>
  <artifactId>build-helper-maven-plugin</artifactId>
  <executions>
    <execution>
      <phase>generate-sources</phase>
      <goals>
        <goal>add-source</goal>
      </goals>
      <configuration>
        <sources>
          <source>src/main/scala</source>
        </sources>
      </configuration>
    </execution>
  </executions>
</plugin>
```

### Using Scala UnitTest by Maven

#### 安装 [Scala](https://yuzhouwan.com/posts/18651/)

#### Maven 依赖

```xml
<dependency>
  <groupId>org.scalatest</groupId>
  <artifactId>scalatest_2.10</artifactId>
  <version>2.2.1</version>
</dependency>
```

#### 创建 unittest 需要的 trait

```
import org.scalatest._
abstract class UnitTestStyle extends FlatSpec
with Matchers with OptionValues with Inside with Inspectors
```

#### 编写测试

```
import spark.streaming.detect.SendNetflow
class SendNetflowTest extends UnitTestStyle {

  "Clean method" should "output a string" in {

    val s = "0,tcp,http,SF,229,9385,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,9,9,0.00,0.00,0.00,0.00,1.00,0.00,0.00,9,90,1.00,0.00,0.11,0.04,0.00,0.00,0.00,0.00,normal."
    SendNetflow.clean(s) should be("0,229,9385,0,0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,9,9,0.00,0.00,0.00,0.00,1.00,0.00,0.00,9,90,1.00,0.00,0.11,0.04,0.00,0.00,0.00,0.00\tnormal.")
  }

  it should "throw Exception if an empty string is inputted" in {
    val emptyS = ""
    a[RuntimeException] should be thrownBy {
    SendNetflow.clean(emptyS)
  }
}
```

Tips: Full code is [here](https://github.com/asdf2014/yuzhouwan/tree/master/yuzhouwan-bigdata/yuzhouwan-bigdata-spark/src/test/scala/com/yuzhouwan/bigdata/spark/style).

### Using slf4j by Maven

#### Maven 配置

```xml
<slf4j.version>1.7.12</slf4j.version>
<logback.version>1.1.3</logback.version>

<dependency>
  <groupId>ch.qos.logback</groupId>
  <artifactId>logback-core</artifactId>
  <version>${logback.version}</version>
</dependency>

<dependency>
  <groupId>ch.qos.logback</groupId>
  <artifactId>logback-classic</artifactId>
  <version>${logback.version}</version>
  <exclusions>
    <exclusion>
      <groupId>ch.qos.logback</groupId>
      <artifactId>logback-core</artifactId>
    </exclusion>
    <exclusion>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
    </exclusion>
  </exclusions>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-api</artifactId>
  <version>${slf4j.version}</version>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>jcl-over-slf4j</artifactId>
  <version>${slf4j.version}</version>
  <exclusions>
    <exclusion>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
    </exclusion>
  </exclusions>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>log4j-over-slf4j</artifactId>
  <version>${slf4j.version}</version>
  <exclusions>
    <exclusion>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
    </exclusion>
  </exclusions>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>jul-to-slf4j</artifactId>
  <version>${slf4j.version}</version>
  <exclusions>
    <exclusion>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-api</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

#### logback.xml in resources directory

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <!-- %p:Level %m:Message %c.%M:Package+Method %F:%L:File+Line -->
  <property name="pattern" value="%d{yyyy-MM-dd HH:mm:ss.SSS} | %p | %m | %c.%M | %F:%L %n"/>

  <!-- Print in Console -->
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder charset="UTF-8">
    <pattern>${pattern}</pattern>
    </encoder>
  </appender>

  <root level="ALL">
    <appender-ref ref="STDOUT"/>
  </root>
</configuration>
```

#### 编码

##### 示例

```java
private static final Logger _log = LoggerFactory.getLogger(ZKEventWatch.class);
_log.info(state);
```

##### 遇到的坑

###### SLF4J multi bindings

- 描述

```
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/D:/apps/maven/repository/ch/qos/logback/logback-classic/1.1.3/logback-classic-1.1.3.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/D:/apps/maven/repository/org/slf4j/slf4j-log4j12/1.6.1/slf4j-log4j12-1.6.1.jar!/org/slf4j/impl/StaticLoggerBinder.class]
```

- 解决

```xml
<!-- 使用 Apache Curator 需要 exclusion slf4j-log4j12，否则会出现问题 -->
<dependency>
  <groupId>org.apache.curator</groupId>
  <artifactId>curator-recipes</artifactId>
  <version>${apache.curator.version}</version>
  <exclusions>
    <exclusion>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-log4j12</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

Tips: Full code is [here](https://github.com/asdf2014/yuzhouwan/blob/master/yuzhouwan-common/pom.xml).

### 导出依赖 Jar

　如何利用 Maven 将依赖的 jar 都导入到一个文件夹下

```xml
<build>
  <plugins>
    <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <version>2.10</version>
    <configuration>
      <outputDirectory>${project.build.directory}/lib</outputDirectory>
      <excludeTransitive>false</excludeTransitive>
      <stripVersion>false</stripVersion>
    </configuration>
    </plugin>
  </plugins>
</build>
```



```bash
$ mvn clean dependency:copy-dependencies

# 将 yuzhouwan-flume 依赖的 jars 都 导出到一个文件夹
# 用相对路径，不能是 绝对路径，否则 maven 会报错：Unknown lifecycle phase "Java"
$ mvn dependency:copy-dependencies -f yuzhouwan-flume/pom.xml -DoutputDirectory=yuzhouwan-flume/target/lib
$ mvn dependency:copy-dependencies -f yuzhouwan-api/yuzhouwan-admin-api/pom.xml -DoutputDirectory=target/lib
```



### Profiles

#### [根据操作系统自动选择 Profile](https://maven.apache.org/enforcer/enforcer-rules/requireOS.html)

```xml
<profiles>
    <profile>
        <id>linux</id>
        <activation>
            <activeByDefault>true</activeByDefault>
            <os>
                <family>unix</family>
                <name>Linux</name>
            </os>
        </activation>
        <properties>
            <blog>https://yuzhouwan.com/posts/15691/</blog>
        </properties>
    </profile>
    <profile>
        <id>mac</id>
        <activation>
            <os>
                <family>mac</family>
            </os>
        </activation>
        <properties>
            <blog>https://yuzhouwan.com/posts/190101/</blog>
        </properties>
    </profile>
</profiles>
```

#### 指定 Profile

```
# 激活指定的 profile
$ mvn clean install -P <profile_A>

# 不激活指定的 profile
$ mvn clean install -P '!<profile_B>'

# 也可以结合使用
$ mvn clean install -P <profile_A> -P '!<profile_B>'
```

Tips: Full code is [here](https://github.com/asdf2014/yuzhouwan/blob/master/yuzhouwan-common/pom.xml).

### 日志级别

　-Dorg.slf4j.simpleLogger.defaultLogLevel=[error](https://maven.apache.org/maven-logging.html)

### Generate Code for Antlr

```
$ mvn clean install -T 1C -DskipTests=true -Dorg.slf4j.simpleLogger.defaultLogLevel=error -B
```

### Maven Properties

```
$ mvn clean install -Dmy.property=propertyValue
```

### Maven Check Style

```
$ mvn clean install -Dcheckstyle.skip
```

### Maven Update

```
$ mvn clean install -U
# -U means force update of dependencies.
```

### Maven Dependency

#### Pre-download

```
$ mvn dependency:go-offline
```

#### Version-compare

```
# 在开发一些 Coprocessor 的时候，需要保证和 HBase 集群的依赖 jar 版本一致，可以使用该方法
$ mvn versions:compare-dependencies -DremotePom=org.apache.hbase:hbase:0.98.8-hadoop2 -DreportOutputFile=depDiffs.txt
```

### Maven Proxy

#### 命令行设置

```
# Linux (bash)
$ export MAVEN_OPTS="-DsocksProxyHost=10.10.10.10 -DsocksProxyPort=8080"
# Windows
$ set MAVEN_OPTS="-DsocksProxyHost=10.10.10.10 -DsocksProxyPort=8080"
```

#### 配置文件

```
$ vim $MAVEN_HOME/conf/settings.xml

  <proxies>
    <!--<proxy>
      <id>http-proxy</id>
      <active>true</active>
      <protocol>http</protocol>
      <host>10.10.10.10</host>
      <port>8888</port>
      <nonProxyHosts>*.yuzhouwan.com</nonProxyHosts>
    </proxy>-->

    <proxy>
      <id>socks-proxy</id>
      <active>true</active>
      <protocol>socks5</protocol>
      <host>127.0.0.1</host>
      <port>1080</port>
      <nonProxyHosts>*.yuzhouwan.com</nonProxyHosts>
    </proxy>
  </proxies>
```

### Avro 插件

#### 创建 avsc 文件

```
{
  "namespace": "com.yuzhouwan.hacker.avro",
  "type": "record",
  "name": "User",
  "fields": [
    {
      "name": "name",
      "type": "string"
    },
    {
      "name": "favorite_number",
      "type": [
        "int",
        "null"
      ]
    },
    {
      "name": "favorite_color",
      "type": [
        "string",
        "null"
      ]
    }
  ]
}
```

#### 使用 avro-tools 工具生成 Avro 类

```
# 下载 avro-tools
# https://mvnrepository.com/artifact/org.apache.avro/avro-tools/1.8.1
# http://archive.apache.org/dist/avro/avro-1.8.1/java/                    (better)
$ wget http://archive.apache.org/dist/avro/avro-1.8.1/java/avro-tools-1.8.1.jar -c avro-tools-1.8.1.jar

# 利用 avsc 文件中的 Schema 生成 Avro 类
$ java -jar avro-tools-1.8.1.jar compile schema user.avsc .
```

#### 使用 avro-plugin 插件生成 Avro 类

```
<properties>
  <apache.avro.version>1.7.7</apache.avro.version>
</properties>

<dependencies>
  <!-- apache avro -->
  <dependency>
    <groupId>org.apache.avro</groupId>
    <artifactId>avro</artifactId>
    <version>${apache.avro.version}</version>
  </dependency>
</dependencies>

<build>
  <plugins>
    <plugin>
      <groupId>org.apache.avro</groupId>
      <artifactId>avro-maven-plugin</artifactId>
      <version>${apache.avro.version}</version>
      <executions>
        <execution>
          <phase>generate-sources</phase>
          <goals>
            <goal>schema</goal>
          </goals>
          <configuration>
            <sourceDirectory>${project.basedir}/src/main/resources/avsc/</sourceDirectory>
            <outputDirectory>${project.basedir}/src/main/java/</outputDirectory>
          </configuration>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

Tips: Full code is [here](https://github.com/asdf2014/yuzhouwan/tree/master/yuzhouwan-hacker/src/main/java/com/yuzhouwan/hacker/avro).

### [Shade](http://maven.apache.org/plugins/maven-shade-plugin/) 插件解决 Jar 多版本共存

#### 适用场景

　在整合大型项目时，可能都依赖 commons-collections 此类的工具包，但是也可能 commons-collections 的版本却因为不一致，导致冲突。可以通过 Shade 插件的 `relocation` 方法进行重定向，此时冲突的依赖 package 名，会被重命名，并且依赖该 jar 的程序，都会被自动替换为新的 package 名

#### 基本用法

##### 指定 Shade 插件的版本

```
<properties>
    <maven.shade.plugin.version>3.2.4</maven.shade.plugin.version>
</properties>
```

##### 实例一

　除了 org.mapdb.* 路径之外的 `org.apache.commons.collections` 包路径，将被重命名为 `com.yuzhouwan.org.apache.commons.collections`

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>${maven.shade.plugin.version}</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <shadedArtifactAttached>false</shadedArtifactAttached>
                <createSourcesJar>true</createSourcesJar>
                <artifactSet>
                    <excludes>
                        <exclude>org.mapdb.*</exclude>
                    </excludes>
                </artifactSet>
                <relocations>
                    <relocation>
                        <pattern>org.apache.commons.collections</pattern>
                        <shadedPattern>com.yuzhouwan.org.apache.commons.collections</shadedPattern>
                    </relocation>
                </relocations>
            </configuration>
        </execution>
    </executions>
</plugin>
```

##### 实例二

　除了 org.glassfish.* 路径之外的 `javax.ws` 包路径，将被重命名为 `shade.javax.ws`。同时，`org.apache.curator` 包路径，将被重命名为 `shade.org.apache.curator`。另外，将 `shadedArtifactAttached` 参数指定为 true 之后，可以避免子模块中出现找不到 shade 后的新包路径的问题

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>${maven.shade.plugin.version}</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <shadedArtifactAttached>true</shadedArtifactAttached>
                <shadedClassifierName>all</shadedClassifierName>
                <createSourcesJar>true</createSourcesJar>
                <artifactSet>
                    <!-- 因为 org.glassfish 里面存在类似 public static class Builder implements javax.ws.rs.client.Invocation.Builder 全限定名的写法 -->
                    <!-- 所以只能用 shade 去隐藏 com.sun.jersey -->
                    <excludes>
                        <exclude>org.glassfish.*</exclude>
                    </excludes>
                </artifactSet>
                <relocations>
                    <relocation>
                        <pattern>javax.ws</pattern>
                        <shadedPattern>shade.javax.ws</shadedPattern>
                    </relocation>
                    <relocation>
                        <pattern>org.apache.curator</pattern>
                        <shadedPattern>shade.org.apache.curator</shadedPattern>
                    </relocation>
                    <!--<relocation>
                        <pattern>org.glassfish</pattern>
                        <shadedPattern>shade.org.glassfish</shadedPattern>
                    </relocation>
                    <relocation>
                        <pattern>com.sun.jersey</pattern>
                        <shadedPattern>shade.com.sun.jersey</shadedPattern>
                    </relocation>-->
                </relocations>
            </configuration>
        </execution>
    </executions>
</plugin>
```

#### 踩过的坑

##### Error creating shaded jar: Error in ASM processing class

###### 描述

```
Failed to execute goal org.apache.maven.plugins:maven-shade-plugin:2.1:shade (default) on project base-k2d-flow: Error creating shaded jar: Error in ASM processing class com/yuzhouwan/bigdata/k2d/K2DPartitioner.class: 52264 -> [Help 1]
```

###### 解决

　升级 `maven-shade-plugin` 版本到 `2.4.x`

##### 通过 Shade 将问题解决后，部署上线后，依赖冲突复现

###### 解决

　因为官方明确指出 `java.io.File.listFiles()` 方法返回的文件序列顺序，是不做保证的。所以，上线后运行环境发生变化，加载 `lib` 目录下的 `jar 包`顺序，可能会发生变化。这时候，就需要利用不同 `lib` 目录，并修改 classpath 的方式，将 `jar 包`加载顺序确定下来

　例如，在整合 [Druid](https://yuzhouwan.com/posts/5845/) 这类依赖树比较庞大的工程，就遇到了这么一种情况。Druid 中 com.sun.jersey (1.19) 和 Dataflow 中 org.glassfish.jersey (2.25.1) 发生冲突。增加了上述实例二中的 Shade 操作仍然会在线上环境，复现依赖冲突。这时，我们可以通过增加一个 `lib_jersey` 目录，存放 `javax.ws.rs-api-2.1.jar`，并修改 classpath 为 `lib_jersey/*:lib/*`。以此，保证了 `lib_jersey` 中的依赖得以优先加载，从而解决冲突

##### 报错 Invalid signature file digest for Manifest main attributes

###### 描述

```
java.lang.SecurityException: Invalid signature file digest for Manifest main attributes
```

###### 解决

```
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>${maven.shade.plugin.version}</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <shadedArtifactAttached>true</shadedArtifactAttached>
                        <createSourcesJar>true</createSourcesJar>
                        <relocations>
                            <relocation>
                                <pattern>com.google.common</pattern>
                                <shadedPattern>com.yuzhouwan.google.common</shadedPattern>
                            </relocation>
                        </relocations>
                        <artifactSet>
                            <includes>
                                <include>org.apache.hadoop:hadoop-common</include>
                            </includes>
                        </artifactSet>
                        <filters>
                            <!-- 加上如下的 filter 过滤，可以忽略有问题的 META-INF 目录 -->
                            <filter>
                                <artifact>*:*</artifact>
                                <excludes>
                                    <exclude>META-INF/*.SF</exclude>
                                    <exclude>META-INF/*.DSA</exclude>
                                    <exclude>META-INF/*.RSA</exclude>
                                </excludes>
                            </filter>
                        </filters>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
# 之后，也可以通过 zip 命令，将 META-INF 目录下的相关文件，从 jar 包中删除
$ zip -d yuzhouwan.jar 'META-INF/*.SF' 'META-INF/*.DSA' 'META-INF/*.RSA'
```

如果是某一个第三方依赖导致的版本冲突，则只需要针对这个第三方依赖进行 shade 操作即可。因此，这里的 artifactSet 使用了 includes，而不是 excludes

### Assembly 插件

#### 重命名 jar 包

```
<files>
    <file>
        <source>${project.basedir}/../yuzhouwan-common/target/yuzhouwan-common-${project.version}.jar</source>
        <outputDirectory>lib</outputDirectory>
        <destName>ayuzhouwan-common-${project.version}.jar</destName>
    </file>
</files>
```

更名为 a 开头的 jar 包之后，可以确保在 classpath 中能被优先加载起来。如此，项目中对依赖里面类的修改，才会生效

#### 不同模块的依赖，打包到不同的目录下

##### 具体步骤

　第一步，先在负责打包分发的 distribution 模块中，设置 pom.xml 文件

```
<properties>
    <maven.assembly.plugin.version>2.3</maven.assembly.plugin.version>
</properties>

<packaging>pom</packaging>

<dependencies>
    <dependency>
        <groupId>com.yuzhouwan</groupId>
        <artifactId>core</artifactId>
    </dependency>
    <dependency>
        <groupId>com.yuzhouwan</groupId>
        <artifactId>ai</artifactId>
    </dependency>
    <dependency>
        <groupId>com.yuzhouwan</groupId>
        <artifactId>bigdata</artifactId>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <artifactId>maven-assembly-plugin</artifactId>
            <version>${maven.assembly.plugin.version}</version>
            <executions>
                <execution>
                    <id>assemble</id>
                    <phase>package</phase>
                    <goals>
                        <goal>single</goal>
                    </goals>
                    <configuration>
                        <descriptors>
                            <descriptor>src/assembly/bin.xml</descriptor>
                            <descriptor>src/assembly/src.xml</descriptor>
                        </descriptors>
                        <tarLongFileMode>gnu</tarLongFileMode>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

　第二步，创建好对应的 bin.xml 和 src.xml

```
<assembly xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2"
          xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2 http://maven.apache.org/xsd/assembly-1.1.2.xsd">

    <id>bin</id>

    <formats>
        <format>dir</format>
        <format>tar.gz</format>
    </formats>

    <baseDirectory>yuzhouwan-${project.version}-bin</baseDirectory>

    <fileSets>
        <fileSet>
            <directory>../</directory>

            <excludes>
                <exclude>**/target/**</exclude>
                <exclude>**/.classpath</exclude>
                <exclude>**/.project</exclude>
                <exclude>**/.settings/**</exclude>
                <exclude>lib/**</exclude>
                <exclude>conf/**</exclude>
            </excludes>

            <includes>
                <include>README</include>
                <include>LICENSE</include>
                <include>NOTICE</include>
                <include>CHANGELOG</include>
                <include>RELEASE-NOTES</include>
                <include>conf/</include>
            </includes>
        </fileSet>

        <fileSet>
            <directory>../</directory>
            <includes>
                <include>bin/yuzhouwan-cli.sh</include>
            </includes>
            <fileMode>0777</fileMode>
            <directoryMode>0755</directoryMode>
        </fileSet>
    </fileSets>

    <moduleSets>
        <moduleSet>
            <includeSubModules>false</includeSubModules>
            <useAllReactorProjects>true</useAllReactorProjects>
            <includes>
                <include>com.yuzhouwan:yuzhouwan-core</include>
            </includes>
            <excludes>
                <exclude>com.yuzhouwan:ai</exclude>
                <exclude>com.yuzhouwan:bigdata</exclude>
            </excludes>
            <binaries>
                <outputDirectory>lib/core</outputDirectory>
                <unpack>false</unpack>
                <dependencySets>
                    <dependencySet>
                        <useProjectArtifact>false</useProjectArtifact>
                        <useTransitiveDependencies>true</useTransitiveDependencies>
                        <unpack>false</unpack>
                        <excludes>
                            <exclude>com.yuzhouwan:ai</exclude>
                            <exclude>com.yuzhouwan:bigdata</exclude>
                        </excludes>
                    </dependencySet>
                </dependencySets>
            </binaries>
        </moduleSet>

        <moduleSet>
            <includeSubModules>false</includeSubModules>
            <useAllReactorProjects>true</useAllReactorProjects>
            <includes>
                <include>com.yuzhouwan:ai</include>
            </includes>
            <excludes>
                <exclude>com.yuzhouwan:bigdata</exclude>
            </excludes>
            <binaries>
                <outputDirectory>lib/plugins/ai</outputDirectory>
                <unpack>false</unpack>
                <dependencySets>
                    <dependencySet>
                        <useProjectArtifact>false</useProjectArtifact>
                        <useTransitiveDependencies>true</useTransitiveDependencies>
                        <unpack>false</unpack>
                        <excludes>
                            <exclude>com.yuzhouwan:bigdata</exclude>
                        </excludes>
                    </dependencySet>
                </dependencySets>
            </binaries>
        </moduleSet>

        <moduleSet>
            <includeSubModules>false</includeSubModules>
            <useAllReactorProjects>true</useAllReactorProjects>
            <includes>
                <include>com.yuzhouwan:bigdata</include>
            </includes>
            <excludes>
                <exclude>com.yuzhouwan:ai</exclude>
            </excludes>
            <binaries>
                <outputDirectory>lib/plugins/bigdata</outputDirectory>
                <unpack>false</unpack>
                <dependencySets>
                    <dependencySet>
                        <useProjectArtifact>false</useProjectArtifact>
                        <useTransitiveDependencies>true</useTransitiveDependencies>
                        <unpack>false</unpack>
                        <excludes>
                            <exclude>com.yuzhouwan:ai</exclude>
                        </excludes>
                    </dependencySet>
                </dependencySets>
            </binaries>
        </moduleSet>
    </moduleSets>
</assembly>
<assembly xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2"
          xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.2 http://maven.apache.org/xsd/assembly-1.1.2.xsd">

    <id>src</id>

    <formats>
        <format>dir</format>
        <format>tar.gz</format>
    </formats>

    <baseDirectory>yuzhouwan-${project.version}-src</baseDirectory>

    <fileSets>
        <fileSet>
            <directory>../</directory>

            <excludes>
                <exclude>**/target/**</exclude>
                <exclude>**/.classpath</exclude>
                <exclude>**/.project</exclude>
                <exclude>**/*.iml</exclude>
                <exclude>**/.settings/**</exclude>
                <exclude>lib/**</exclude>
            </excludes>

            <includes>
                <include>.ci</include>
                <include>.gitignore</include>
                <include>DEVNOTES</include>
                <include>README</include>
                <include>LICENSE</include>
                <include>NOTICE</include>
                <include>CHANGELOG</include>
                <include>RELEASE-NOTES</include>
                <include>bin/**</include>
                <include>conf/**</include>
                <include>native/**</include>
                <include>pom.xml</include>
                <include>dev-conf/**</include>
                <include>yuzhouwan-core/**</include>
                <include>yuzhouwan-ai/**</include>
                <include>yuzhouwan-bigdata/**</include>
            </includes>
        </fileSet>
    </fileSets>
</assembly>
```

　构建完成之后，即可看到 `yuzhouwan-ai` 和 `yuzhouwan-bigdata` 模块的依赖，分别被打包到了 `plugins/ai` 和 `plugins/bigdata` 目录下

```
# 需要预先安装 tree 命令（yum install tree -y）
$ tree
  .
  ├── bin
  │   └── yuzhouwan-cli.sh
  └── lib
      ├── core
      └── plugins
          ├── ai
          └── bigdata
```

##### 踩过的坑

###### 模块间存在版本冲突的 jar 包

- 问题描述

  ```
  Currently, inclusion of module dependencies may produce unpredictable results if a version conflict occurs
  ```

- 解决

  如果上述 `yuzhouwan-ai` 和 `yuzhouwan-bigdata` 模块，与 `yuzhouwan-common` 模块存在版本冲突的 jar 包，则需要将 `bin.xml` 拆解成 `bin-common.xml`、`bin-ai.xml` 和 `bin-bigdata.xml` 三个 assembly 配置文件，依次进行构建（此时需要将 id 设置成一样的，不然会被构建到不同的目录下）。否则，版本冲突的 jar 包，将只能保留其中一个版本，且具体保留哪个版本是不确定的

## 常见问题

### 如何使用其他语言编译出来的 Jar

#### Clojar

　如果需要在 Maven 中，添加新的 Repository，这里以 Clojar 为例，配置如下：

```
<repositories>
  <repository>
    <id>clojars.org</id>
    <url>http://clojars.org/repo</url>
  </repository>
</repositories>
```



### 本地缓存

　Maven 中出现 Repository 被缓存在本地，则可以进一步添加下面属性，以配置更新策略：

```
<repository>
  <id>clojars.org</id>
  <url>http://clojars.org/repo</url>
  <releases>
    <enabled>true</enabled>
    <updatePolicy>always</updatePolicy>
  </releases>
  <snapshots>
    <enabled>true</enabled>
    <updatePolicy>always</updatePolicy>
  </snapshots>
</repository>
```

### 下载问题

#### 已 download，未 import

　如果发现 `pom.xml` 中的 依赖虽然在 `.m2` 中被下载了，但是没有被 `import` 到项目中
　尝试在 Intellij Idea 的 setting 中 找到 `Maven -> Ignored Files`，看看对应的 `pom.xml` 有没有被勾选

如果仍然无法解决，也有可能是 Intellij Idea 自身的问题，可尝试通过 Just Restart 来解决

### 编译问题

#### JDK 版本不一致

##### 描述

```
Error:java: Compilation failed: internal java compiler error
```

##### 解决

　调整 compiler 的级别，并在 `pom.xml` 中添加 build 标签，规定 compiler 的级别

```
<build>
  <finalName>elastic-netflow-v5</finalName>

  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>2.3.2</version>
      <configuration>
        <source>1.7</source>
        <target>1.7</target>
      </configuration>
    </plugin>

    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-release-plugin</artifactId>
      <version>2.4.1</version>
    </plugin>
  </plugins>
</build>
```

#### Scala 版本不一致

##### 描述

```
[ERROR] error: error while loading <root>, Error accessing D:\.m2\repository\org\apache\curator\curator-client\2.4.0\curator-client-2.4.0.jar
[ERROR] error: scala.reflect.internal.MissingRequirementError: object java.lang.Object in compiler mirror not found.
```

##### 解决

　查看 pom 文件中指定的 Scala 版本，切换到对应的版本，进行编译即可

#### 存在没有下载好的依赖

##### 描述

```
Error creating properties files for forking; nested exception is java.io.IOException: No such file or directory
```

##### 解决

　安装依赖

```
$ mvn dependency::tree
```

　离线 build (offline)

```
$ mvn install -o
```



#### 本地 mvn clean install 到 .m2/repository 中的 jar 和 仓库中的不一致时

##### 描述

```
"Class not found 'xxxx' Empty test suite" in java unittest, after change the modules' names
```

##### 解决

```
Maven           ->        Reimport            ->        Generte Source and Update Folder
Ctrl + F9       ->        Make Project
```

#### 打包存在脏程序

　`war:war` 之前需要先执行 `mvn clean install`。否则，会因为 `profiles` 的缘故，导致 `war` 包中 缺少文件

### 语法问题

#### Maven 中无法识别 ${project.version}

　需要将所有（除了 root / parent 模块）的 `<version>1.0</version>` 中的具体版本号（如 `1.0.0`）全部替换成 `"${project.version}"`

### 插件问题

#### xxx module must not contain source root. the root already belongs to module xxx

##### 解决

```
Artifacts Setting  ->  Modules -> Sourc tab
delete the fold by clicking on the X icon to the right of it
```

#### ExecutionException The forked VM terminated without properly saying goodbye

##### 描述

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-surefire-plugin:2.19.1:test (default-test) on project eagle-app-streamproxy: ExecutionException The forked VM terminated without properly saying goodbye. VM crash or System.exit called?
```

##### 解决

```
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <version>${maven-surefire.version}</version>
  <configuration>
    <!-- 增加 argLine 参数以调整 JVM -->
    <argLine>-Xmx2048m -Xms1024m -XX:MaxPermSize=512m -XX:-UseGCOverheadLimit</argLine>
    <forkMode>always</forkMode>
  </configuration>
</plugin>
```

详见：[Eagle-RP-897](https://github.com/apache/eagle/pull/897)

#### Fatal error compiling: 无效的目标发行版: 1.8

　需要在 IDE 的 `Project Structure(Ctrl+Alt+Shift+S)` 里设置 `Project JDK` 为 `jdk1.8`，并在 `Settings(Ctrl+Alt+S)` 里设置 `Java Compiler` 的 `bytecode version` 为 `1.8`

#### Assembly 插件报错 java.lang.StackOverflowError

##### 解决

```
# 调大默认的堆栈大小
export MAVEN_OPTS=-Xss2m
```

##### 补充

```
-Xms：jvm 进程启动时分配的内存
-Xmx：jvm 进程运行过程中最大能分配到的内存
-Xss：jvm 进程中启动线程，为每个线程分配的内存大小
```

#### protobuf-maven-plugin 找不到 protoc 运行程序

##### 解决

　本地安装 Protobuf 即可

```
$ gcc --version
  Configured with: --prefix=/Library/Developer/CommandLineTools/usr --with-gxx-include-dir=/Library/Developer/CommandLineTools/SDKs/MacOSX10.14.sdk/usr/include/c++/4.2.1
  Apple LLVM version 10.0.1 (clang-1001.0.46.4)
  Target: x86_64-apple-darwin18.7.0
  Thread model: posix
  InstalledDir: /Library/Developer/CommandLineTools/usr/bin

$ brew install autoconf automake libtool curl
$ wget https://github.com/google/protobuf/releases/download/v2.5.0/protobuf-2.5.0.tar.gz
$ tar zxvf protobuf-2.5.0.tar.gz
$ cd protobuf-2.5.0
$ ./autogen.sh
$ ./configure
$ make
$ make check
$ make install
$ protoc --version
  libprotoc 2.5.0
```

#### 如何跳过 gpg 插件

　增加 `<skip>true</skip>` 配置即可

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-gpg-plugin</artifactId>
    <executions>
        <execution>
            <id>sign-artifacts</id>
            <phase>verify</phase>
            <goals>
                <goal>sign</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <skip>true</skip>
    </configuration>
</plugin>
```

#### Permission denied of shell script

##### 解决

```
# 修改 exec-maven-plugin 中需要用到脚本即可
$ chmod +x yuzhouwan.sh

# 还可以通过以下命令，修改已经上传的文件权限
$ git update-index --chmod=+x yuzhouwan.sh
```

### 依赖问题

#### Maven 中央仓库找不到对应版本的依赖

##### 如何将其他项目的 jar 安装到 maven 仓库中

###### 利用命令安装

```
$ mvn install:install-file -Dfile=<jar 包的位置> -DgroupId=<上面的 groupId> -DartifactId=<上面的 artifactId> -Dversion=<上面的 version> -Dpackaging=jar -DgeneratePom=true -DpomFile=<指定一个 pom.xml 添加进去>

$ mvn install:install-file -Dfile=G:\ES\yuzhouwan_netflow-rest\lib\ite-esmanage-0.7.2-SNAPSHOT.jar -DgroupId=ite -DartifactId=esmanage -Dversion=0.7.2-SNAPSHOT -Dpackaging=jar -DgeneratePom=true -DpomFile=C:\yuzhouwan\Maven\pom.xml

# -DgeneratePom 没有成功
# 原因是 这个第三方的 jar 中，pom.xml 路径不对，必须手动 copy 出来，并用 -DpomFile 的方式导入
```

###### 利用插件安装

```
<!-- 注意：${project.basedir} 是当前模块的根目录，需要把依赖包，放对位置 -->
<build>
  <pluginManagement>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-install-plugin</artifactId>
        <version>2.5.2</version>
        <executions>
          <execution>
            <id>install-external</id>
            <phase>clean</phase>
            <configuration>
              <file>${project.basedir}/lib/flume-hdfs-sink-${flume.version}.jar</file>
              <repositoryLayout>default</repositoryLayout>
              <groupId>org.apache.flume.flume-ng-sinks</groupId>
              <artifactId>flume-hdfs-sink</artifactId>
              <version>${flume.version}</version>
              <packaging>jar</packaging>
              <generatePom>true</generatePom>
            </configuration>
            <goals>
              <goal>install-file</goal>
            </goals>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </pluginManagement>
</build>
```

#### 控制 Scope 范围

```
# compile（编译范围）
　compile 范围是默认的。编译范围依赖在所有的 classpath 中可用，同时它们也会被打包

# provided（已提供范围）
　provided 范围只有在当 JDK 或者一个容器已提供该依赖之后才使用。它们不是传递性的，也不会被打包。适用于在实现了公共 common 模块，其中有很多依赖，但是只有在其他依赖了 common 模块的子模块中，重新声明依赖，才会真正被打包到子模块中。这样，既可以保证 common 模块的正常编译，又可以减少子模块中的依赖包大小

# runtime（运行时范围）
　runtime 范围在运行和测试系统的时候需要，但在编译的时候不需要。例如，在编译的时候，如果只需要 JDBC API Jar，但只有在运行的时候才需要 JDBC 驱动的具体实现

# test（测试范围）
　test 范围在一般的编译和运行时都不需要，它们只有在测试编译和测试运行阶段可用。常用于 JUnit 的依赖

# system（系统范围）
　system 范围与 provided 类似，但是必须显式的提供一个对于本地系统中 Jar 文件的路径。这么做是为了允许基于本地对象编译（systemPath），而这些对象是系统类库的一部分。这样的构件应该是一直可用的，Maven 也不会在仓库中去寻找它（不推荐使用）
```

#### 引入某一个本身依赖树很庞大的 Dependency

```
# 依赖列表
$ mvn dependency:list

# 依赖树（这里推荐直接用 Intellij Idea 中自带的 "Show Dependencies" 功能，可视化地展示依赖树 和 对应依赖冲突，并能够直接对 依赖树进行修剪）
$ mvn dependency:tree

# 依赖分析
$ mvn dependency:analyze
  Unused declared dependencies  表示项目中未使用的，但显示声明的依赖
  Used undeclared dependencies  表示项目中使用到的，但是没有显示声明的依赖
```

#### Detected both log4j-over-slf4j.jar AND slf4j-log4j12.jar on the class path, preempting StackOverflowError.

##### 解决

```
<spark.scala.version>2.11</spark.scala.version>
<apache.kafka.version>0.9.0.1</apache.kafka.version>

<dependency>
  <groupId>org.apache.kafka</groupId>
  <artifactId>kafka_${spark.scala.version}</artifactId>
  <version>${apache.kafka.version}</version>
  <exclusions>
    <exclusion>
      <groupId>org.slf4j</groupId>
      <artifactId>slf4j-log4j12</artifactId>
    </exclusion>
    <exclusion>
      <groupId>log4j</groupId>
      <artifactId>log4j</artifactId>
    </exclusion>
  </exclusions>
</dependency>
```

#### Could not find artifact jdk.tools:jdk.tools:jar:1.7 at specified path /Library/Java/JavaVirtualMachines/jdk-12.0.2.jdk/Contents/Home/../lib/tools.jar

##### 描述

　Hive 中存在 jdk.tools 依赖，需要特定的 JDK 版本

```
<dependencies>
  <dependency>
    <groupId>org.apache.hive</groupId>
    <artifactId>hive-jdbc</artifactId>
    <version>2.1.0</version>
  </dependency>
</dependencies>

<build>
  <plugins>
    <plugin>
      <artifactId>maven-compiler-plugin</artifactId>
      <configuration>
        <source>1.8</source>
        <target>1.8</target>
        <encoding>UTF-8</encoding>
      </configuration>
    </plugin>
  </plugins>
</build>
```

##### 解决

　通过 exclusion 标签将 jdk.tools 去除掉即可。如果仍然需要 jdk.tools 依赖，则可以另外单独增加 jdk.tools 的 dependency，并指定好正确的 JDK 版本

```
<dependency>
  <groupId>org.apache.hive</groupId>
  <artifactId>hive-jdbc</artifactId>
  <version>2.1.0</version>
  <exclusions>
    <exclusion>
      <artifactId>jdk.tools</artifactId>
      <groupId>jdk.tools</groupId>
    </exclusion>
  </exclusions>
</dependency>

<dependency>
    <groupId>jdk.tools</groupId>
    <artifactId>jdk.tools</artifactId>
    <version>1.8</version>
</dependency>
```