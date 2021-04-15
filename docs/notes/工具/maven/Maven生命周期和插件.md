# Maven生命周期和插件

## maven的生命周期

### 生命周期介绍

- maven的生命周期就是为了对所有的构建过程进行抽象和统一。
- 包含了项目的`清理`、`初始化`、`编译`、`测试`、`打包`、`集成测试`、`验证`、`部署`和`站点生成`等几乎所有构建步骤。
- 几乎所有的项目构建，都能映射到这样一个生命周期上。

### 生命周期详解

1. 三套生命周期

   - maven包含三套**`相互独立`**的生命周期。
   - `clean`生命周期：用于清理项目。
   - `default`生命周期：用于构建项目。
   - `site`生命周期：用于建立项目站点。

2. 生命周期内部的阶段

   - 每个生命周期包含`多个阶段`(phase)。
   - 这些阶段是`有序`的。
   - `后面`的阶段`依赖`于`前面`的阶段。当执行某个阶段的时候，会先执行它前面的阶段。
   - 用户和maven最直接的交互就是调用这些生命周期阶段。

3. **clean生命周期**
   clean生命周期的目的是清理项目，它包含3个阶段：

   | clean生命周期阶段 | 说明                          |
   | ----------------- | ----------------------------- |
   | pre-clean         | 执行一些clean前需要完成的工作 |
   | `clean`           | 清理上一次构建生成的文件      |
   | post-clean        | 执行一些clean后需要完成的工作 |

4. **default生命周期**
   default生命周期的目的是构建项目，它定义了真正构建时所需要完成的所有步骤，是所有生命周期中最核心的部分。
   包含23个阶段：[详细介绍](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)

   | default生命周期阶段     | 说明                                                       |
   | ----------------------- | ---------------------------------------------------------- |
   | validate                | 验证项目是否正确并且所有必要信息都可用                     |
   | initialize              | 初始化构建状态，比如设置属性值、创建目录                   |
   | generate-sources        | 生成包含在编译阶段中的任何源代码                           |
   | process-sources         | 处理源代码，比如说，过滤任意值                             |
   | generate-resources      | 生成将会包含在项目包中的资源文件                           |
   | process-resources       | 复制和处理资源到目标目录，为打包阶段最好准备               |
   | `compile`               | 编译项目的源代码                                           |
   | process-classes         | 处理编译生成的文件，比如说对Java class文件做字节码改善优化 |
   | generate-test-sources   | 生成包含在编译阶段中的任何测试源代码                       |
   | process-test-sources    | 处理测试源代码，比如说，过滤任意值                         |
   | generate-test-resources | 为测试创建资源文件                                         |
   | process-test-resources  | 复制和处理测试资源到目标目录                               |
   | test-compile            | 编译测试源代码到测试目标目录                               |
   | process-test-classes    | 处理测试源码编译生成的文件                                 |
   | `test`                  | 使用合适的单元测试框架运行测试 , 测试代码不会被打包或部署  |
   | prepare-package         | 在实际打包之前，执行任何的必要的操作为打包做准备           |
   | `package`               | 将编译后的代码打包成可分发的格式，比如JAR                  |
   | pre-integration-test    | 在执行集成测试前进行必要的动作。比如说，搭建需要的环境     |
   | integration-test        | 如有必要，将程序包处理并部署到可以运行集成测试的环境中     |
   | post-integration-test   | 执行集成测试完成后进行必要的动作。比如说，清理集成测试环境 |
   | verify                  | 运行任何检查以验证包是否有效并符合质量标准                 |
   | `install`               | 安装项目包到maven本地仓库，供本地其他maven项目使用         |
   | `deploy`                | 将最终包复制到远程仓库，供其他开发人员和maven项目使用      |

5. **site生命周期**
   site生命周期的目的是建立和发布项目站点。
   Maven能够基于pom.xml所包含的信息，自动生成一个友好的站点，方便团队交流和发布项目信息。
   包含以下4个阶段：

   | site生命周期阶段 | 说明                                     |
   | ---------------- | ---------------------------------------- |
   | pre-site         | 执行一些在生成项目站点之前需要完成的工作 |
   | site             | 生成项目站点文档                         |
   | post-site        | 执行一些在生成项目站点之后需要完成的工作 |
   | site-deploy      | 将生成的项目站点发布到服务器上           |

6. mvn命令和生命周期
   从命令行执行maven任务的最主要方式就是调用maven的生命周期阶段。
   需要注意的是，每套生命周期是相互独立的，但是每套生命周期的阶段是有前后依赖关系的。

   - 格式： `mvn 阶段 [阶段2] ...[阶段n]`
   - `mvn clean`：该命令调用clean生命周期的clean阶段。
     - 实际执行的阶段为clean生命周期中的pre-clean和clean阶段。
   - `mvn test`：该命令调用default生命周期的test阶段。
     - 实际执行的阶段为default生命周期的从validate到test的所有阶段。
     - 这也就解释了为什么执行测试的时候，项目的代码能够自动编译。
   - `mvn clean install`：该命令调用clean生命周期的clean阶段和default生命周期的install阶段。
     - 实际执行的阶段为clean生命周期的pre-clean、clean阶段，以及default生命周期的从validate到install的所有阶段。
   - `mvn clean deploy`：该命令调用clean生命周期的clean阶段和default生命周期的deploy阶段。
     - 实际执行的阶段为clean生命周期的pre-clean、clean阶段，以及default生命周期的所有阶段。
     - 包含了清理上次构建的结果、编译代码、运行单元测试、打包、将打好的包安装到本地仓库、将打好的包发布到远程仓库等所有操作。

## maven插件

### 插件目标

- maven的核心仅仅定义了抽象的生命周期，具体的任务交给插件完成。
  - 每套生命周期包含多个阶段，每个阶段执行什么操作，都由插件完成。
- 插件以独立的构件形式存在。
  - 为了能够复用代码，每个插件包含多个功能。
  - **插件中的每个功能就叫做插件的目标（Plugin Goal），每个插件中可能包含一个或者多个插件目标（Plugin Goal）**。

### 插件绑定

- maven**生命周期的阶段与插件目标绑定**，以完成某个具体的构件任务。
  - 比如项目编译这个任务，对应了`default生命周期`阶段的`compile阶段`，而`maven-compiler-plugin`插件的`compile目标`能够完成该任务，因此将他们进行绑定，实现项目编译任务。
- 生命周期阶段与插件进行绑定后，可以通过`mvn 阶段`来执行和这个阶段绑定的插件目标。

### 内置绑定

1. 说明
   为了让用户几乎不用任何配置就能构建maven项目，maven为一些主要的生命周期阶段绑定好了插件目标，当我们通过命令调用生命周期阶段时，绑定的插件目标就会执行对应的任务

2. **clean生命周期阶段**与插件目标的绑定关系

   | 生命周期阶段 | 插件目标                 | 作用                   |
   | ------------ | ------------------------ | ---------------------- |
   | pre-clean    |                          |                        |
   | clean        | maven-clean-plugin:clean | **删除项目的输出目录** |
   | post-clean   |                          |                        |

3. **default生命周期阶段**与插件目标的绑定关系(打包类型:`jar`)

   | 生命周期阶段           | 插件目标                             | 作用                              |
   | ---------------------- | ------------------------------------ | --------------------------------- |
   | process-resources      | maven-resources-plugin:resources     | 复制主资源文件到主输出目录        |
   | compile                | maven-compiler-plugin:compile        | 编译主代码到主输出目录            |
   | process-test-resources | maven-resources-plugin:testResources | 复制测试资源文件到测试输出目录    |
   | test-compile           | maven-compiler-plugin:testCompile    | 编译测试代码到测试输出目录        |
   | test                   | maven-surefire-plugin:test           | 执行测试用例                      |
   | package                | maven-jar-plugin:jar                 | 创建项目jar包                     |
   | install                | maven-install-plugin:install         | 将项目输出构件安装到本地maven仓库 |
   | deploy                 | maven-deploy-plugin:deploy           | 将项目输出构件部署到远程仓库      |

   - 关于输出目录，参考[Maven学习2: 依赖管理 - 2.约定优于配置](https://segmentfault.com/a/1190000021287778#item-2)

4. **site生命周期阶段**与插件目标的绑定关系

   | 生命周期阶段 | 插件目标                 | 作用                       |
   | ------------ | ------------------------ | -------------------------- |
   | pre-site     |                          |                            |
   | site         | maven-site-plugin:site   | 生成项目站点               |
   | post-site    |                          |                            |
   | site-deploy  | maven-site-plugin:deploy | 将项目站点部署到远程服务器 |

5. 构建过程验证

   - 与1.2生命周期详解中6.mvn命令和生命周期示例相同，项目依然使用demo1作为示例。

   - `mvn clean`：

     - `maven-clean-plugin:2.5:clean (default-clean)`: 插件artifactId:version:goal(goal-id)， 删除了项目输出目录。

     ```
     [INFO] Scanning for projects...
     [INFO] 
     [INFO] ---------------------------< com.john:demo1 >---------------------------
     [INFO] Building demo1 1.0-SNAPSHOT
     [INFO] --------------------------------[ jar ]---------------------------------
     [INFO] 
     [INFO] --- maven-clean-plugin:2.5:clean (default-clean) @ demo1 ---
     [INFO] Deleting /Users/john/Desktop/demo1/target
     [INFO] ------------------------------------------------------------------------
     [INFO] BUILD SUCCESS
     [INFO] ------------------------------------------------------------------------
     [INFO] Total time: 0.458 s
     [INFO] Finished at: 2019-12-22T16:28:50+08:00
     [INFO] ------------------------------------------------------------------------
     ```

   - `mvn test`：

     - 调用default生命周期的test阶段，实际执行的时候 default生命周期的validate阶段直到test阶段都要执行，每个阶段绑定的插件目标也会被执行。可以看到处理资源、编译源码、处理测试资源、编译测试源码、测试插件目标都得到了执行。

     ```
     [INFO] Scanning for projects...
     [INFO] 
     [INFO] ---------------------------< com.john:demo1 >---------------------------
     [INFO] Building demo1 1.0-SNAPSHOT
     [INFO] --------------------------------[ jar ]---------------------------------
     [INFO] 
     [INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ demo1 ---
     [INFO] Using 'UTF-8' encoding to copy filtered resources.
     [INFO] Copying 1 resource
     [INFO] 
     [INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ demo1 ---
     [INFO] Changes detected - recompiling the module!
     [INFO] Compiling 1 source file to /Users/john/Desktop/demo1/target/classes
     [INFO] 
     [INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ demo1 ---
     [INFO] Using 'UTF-8' encoding to copy filtered resources.
     [INFO] Copying 1 resource
     [INFO] 
     [INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ demo1 ---
     [INFO] Changes detected - recompiling the module!
     [INFO] Compiling 1 source file to /Users/john/Desktop/demo1/target/test-classes
     [INFO] 
     [INFO] --- maven-surefire-plugin:2.12.4:test (default-test) @ demo1 ---
     [INFO] Surefire report directory: /Users/john/Desktop/demo1/target/surefire-reports
     
     -------------------------------------------------------
      T E S T S
     -------------------------------------------------------
     Running com.john.AppTest
     this is a test case
     arg is john
     Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 0.061 sec
     
     Results :
     
     Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
     
     [INFO] ------------------------------------------------------------------------
     [INFO] BUILD SUCCESS
     [INFO] ------------------------------------------------------------------------
     [INFO] Total time: 2.394 s
     [INFO] Finished at: 2019-12-22T16:32:23+08:00
     [INFO] ------------------------------------------------------------------------
     ```

   - `mvn clean install`：

     - 该命令调用clean生命周期的clean阶段和default生命周期的install阶段。
     - （构建过程内容省略）

   - `mvn clean deploy`：

     - 该命令调用clean生命周期的clean阶段和default生命周期的deploy阶段。
     - （构建过程内容省略）

### 自定义绑定

1. 说明
   自己选择将某个插件目标绑定到生命周期的某个阶段上，使得maven项目在构建过程中执行更多更富特色的任务。

2. 举例：创建项目的源码jar包

   - 使用插件：`org.apache.maven.plugins:maven-source-plugin:3.2.1`

   - 插件目标：`jar-no-fork`

     - 这里有个疑问：maven-source-plugin提供的`jar`、`jar-no-fork`两个目标都可以进行项目源码的打包，为什么选择jar-no-fork？
     - 解释：
     - `jar`： Invokes the execution of the lifecycle phase: `generate-sources` prior to executing itself. jar这个插件目标又重新执行了在它前面的阶段`generate-sources`。
     - `jar-no-fork`；**不会重新执行**在它前面的阶段`generate-sources`。

   - 绑定阶段：`package`（默认），我们这里使用`verify`（在测试完成之后并将构件安装到本地仓库之前执行的阶段。）

   - 具体配置(pom.xml)

     ```
     <build>
         <plugins>
             <plugin>
                 <groupId>org.apache.maven.plugins</groupId>
                 <artifactId>maven-source-plugin</artifactId>
                 <version>3.2.1</version>
                 <executions>
                     <execution>
                         <id>attach-sources</id>
                         <phase>verify</phase>
                         <goals>
                             <goal>
                                 jar-no-fork
                             </goal>
                         </goals>
                     </execution>
                 </executions>
     
             </plugin>
         </plugins>
     </build>
     ```

   - 执行`mvn verify`：

     ```
     [INFO] Scanning for projects...
     [INFO] 
     [INFO] ---------------------------< com.john:demo1 >---------------------------
     [INFO] Building demo1 1.0-SNAPSHOT
     [INFO] --------------------------------[ jar ]---------------------------------
     [INFO] 
     [INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ demo1 ---
     [INFO] Using 'UTF-8' encoding to copy filtered resources.
     [INFO] Copying 1 resource
     [INFO] 
     [INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ demo1 ---
     [INFO] Nothing to compile - all classes are up to date
     [INFO] 
     [INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ demo1 ---
     [INFO] Not copying test resources
     [INFO] 
     [INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ demo1 ---
     [INFO] Not compiling test sources
     [INFO] 
     [INFO] --- maven-surefire-plugin:2.12.4:test (default-test) @ demo1 ---
     [INFO] Tests are skipped.
     [INFO] 
     [INFO] --- maven-jar-plugin:2.4:jar (default-jar) @ demo1 ---
     [INFO] Building jar: /Users/john/Desktop/demo1/target/demo1-1.0-SNAPSHOT.jar
     [INFO] 
     [INFO] --- maven-source-plugin:3.2.1:jar-no-fork (attach-sources) @ demo1 ---
     [INFO] Building jar: /Users/john/Desktop/demo1/target/demo1-1.0-SNAPSHOT-sources.jar
     [INFO] ------------------------------------------------------------------------
     [INFO] BUILD SUCCESS
     [INFO] ------------------------------------------------------------------------
     [INFO] Total time: 2.101 s
     [INFO] Finished at: 2019-12-22T17:00:39+08:00
     [INFO] ------------------------------------------------------------------------
     ```

### 插件配置

1. 命令行插件配置

   - 在maven命令中使用`-D`参数，并伴随一个参数键=参数值的形式，来配置插件目标的参数。
   - 参数`-D`是java自带的，其功能是通过命令行设置一个java系统属性。
   - 例如，跳过执行测试目标，`mvn clean pakcage -Dmaven.test.skip=true`

2. pom文件中插件全局配置

   - 用户在pom文件声明插件时，可以对插件进行全局的配置。

   - 例如，配置maven-compiler-plugin编译Java 1.8版本的源文件，生成与JVM 1.8兼容的字节码文件。

     ```
     <build>
         <plugins>
             <plugin>
                 <groupId>org.apache.maven.plugins</groupId>
                 <artifactId>maven-compiler-plugin</artifactId>
                 <version>3.8.1</version>
                 <configuration>
                     <!-- 编译器版本 -->
                     <compilerVersion>1.8</compilerVersion>
                     <!-- 源码版本 -->
                     <source>1.8</source>
                     <!-- 目标代码版本 -->
                     <target>1.8</target>
                 </configuration>
             </plugin>
         </plugins>
     </build>
     ```

### 获取插件信息

1. 在线插件信息

   - Apache：[Maven Plugins](https://maven.apache.org/plugins/index.html)
   - 可以查看插件介绍、插件目标、目标参数等等。

2. 使用 `maven-help-plugin` 描述插件

   - 使用 `maven-help-plugin` 获取插件信息。

   - 方式1：使用插件坐标：`mvn help:describe -Dplugin=groupId:artifactId:version`

   - 方式2：使用插件目标前缀：`mvn help:describe -Dplugin=Goal Prefix`，目标前缀的作用是方便在命令行直接运行插件。

   - 如果要获取插件的详细描述，可以在命令后加上`-Ddetail`参数。

   - 举例：两种方式获取`maven-compiler-plugin:3.8.1`的插件描述信息

     - `mvn help:describe -Dplugin=org.apache.maven.plugins:maven-compiler-plugin:3.8.1`

     - `mvn help:describe -Dplugin=compiler`

     - 过程输出

       ```
       [INFO] Scanning for projects...
       [INFO] 
       [INFO] ---------------------------< com.john:demo1 >---------------------------
       [INFO] Building demo1 1.0-SNAPSHOT
       [INFO] --------------------------------[ jar ]---------------------------------
       [INFO] 
       [INFO] --- maven-help-plugin:3.2.0:describe (default-cli) @ demo1 ---
       [INFO] org.apache.maven.plugins:maven-compiler-plugin:3.8.1
       
       Name: Apache Maven Compiler Plugin
       Description: The Compiler Plugin is used to compile the sources of your
         project.
       Group Id: org.apache.maven.plugins
       Artifact Id: maven-compiler-plugin
       Version: 3.8.1
       Goal Prefix: compiler
       
       This plugin has 3 goals:
       
       compiler:compile
         Description: Compiles application sources
       
       compiler:help
         Description: Display help information on maven-compiler-plugin.
           Call mvn compiler:help -Ddetail=true -Dgoal=<goal-name> to display
           parameter details.
       
       compiler:testCompile
         Description: Compiles application test sources.
       
       For more information, run 'mvn help:describe [...] -Ddetail'
       
       [INFO] ------------------------------------------------------------------------
       [INFO] BUILD SUCCESS
       [INFO] ------------------------------------------------------------------------
       [INFO] Total time: 2.162 s
       [INFO] Finished at: 2019-12-22T17:42:55+08:00
       [INFO] ------------------------------------------------------------------------
       ```

3. 从命令行调用插件

   - maven命令帮助：`mvn -h`
   - 输出：`usage: mvn [options] [<goal(s)>] [<phase(s)>]`
   - 获取项目依赖树：`mvn dependency:tree`
   - 分析当前项目依赖：`mvn dependency:analyze`
   - 查看当前项目的已解析依赖：`mvn dependency:list`
   - 查看项目最终pom.xml文件：`mvn help:effective-pom`

### 插件解析

maven简化了插件的使用和配置，但是具体maven是怎么解析插件的呢？

1. 插件仓库

   - 与依赖构件一样，插件构件同样基于坐标存储在maven仓库中。在需要的时候，maven会从本地仓库查找插件，如果不存在，则从远程仓库查找。找到后下载到本地仓库使用。

   - 配置插件仓库，插件仓库与依赖构件仓库是分开配置的。

   - 在项目pom文件`pluginRepositories -> pluginRepository`元素中配置插件仓库。

   - 具体配置（maven 内置的插件远程仓库）：

     ```
     <pluginRepositories>
       <pluginRepository>
         <id>central</id>
         <name>Central Repository</name>
         <url>https://repo.maven.apache.org/maven2</url>
         <layout>default</layout>
         <snapshots>
           <enabled>false</enabled>
         </snapshots>
         <releases>
           <updatePolicy>never</updatePolicy>
         </releases>
       </pluginRepository>
     </pluginRepositories>
     ```

2. 插件的默认groupId

   - `org.apache.maven.plugins`，配置时不建议省略。

3. 解析插件版本

   - maven在超级pom中为所有核心插件定义了版本。
   - 使用插件时候，应该一直显式设定版本。

4. 解析插件前缀

   - `插件前缀(Goal Prefix)与groupId:artifactId是一一对应`的，这种对应关系存储在仓库元数据中。

     - 示例：[匹配关系](http://repo1.maven.org/maven2/org/apache/maven/plugins/maven-metadata.xml)

     - 本地仓库插件元数据文件位置：`~/.m2/repository/org/apache/maven/plugins/maven-metadata-central.xml`

       ```
       <metadata>
           <plugins>
               <plugin>
                   <name>Apache Maven Clean Plugin</name>
                   <prefix>clean</prefix>
                   <artifactId>maven-clean-plugin</artifactId>
               </plugin>
               <plugin>
                   <name>Apache Maven Compiler Plugin</name>
                   <prefix>compiler</prefix>
                   <artifactId>maven-compiler-plugin</artifactId>
               </plugin>
           </plugins>
       </metadata>
       ```

   - 默认的插件仓库groupId：`org.apache.maven.plugins`

   - 配置自己的插件仓库元数据：

     - 用户的maven配置文件：`~/.m2/settings.xml`

     - `settings -> pluginGroups -> pluginGroup`元素中配置。

       ```
       <settings>
         <pluginGroups>
           <pluginGroup>com.your.plugins</pluginGroup>
         </pluginGroups>
       </settings>
       ```