# Maven 常用插件

## maven-compiler-plugin

设置maven编译的jdk版本，maven3默认用jdk1.5，maven2默认用jdk1.3

```xml
<plugin>                                                                                                                                                                                                  
    <groupId>org.apache.maven.plugins</groupId>                                                                                               
    <artifactId>maven-compiler-plugin</artifactId>                                                                                            
    <version>3.8.0</version>                                                                                                                   
    <configuration>                                                                                                                                          
        <source>1.8</source> <!-- 源代码使用的JDK版本 -->                                                                                             
        <target>1.8</target> <!-- 需要生成的目标class文件的编译版本 -->                                                                                     
        <encoding>UTF-8</encoding><!-- 字符集编码 -->
        <skipTests>true</skipTests><!-- 跳过测试 -->
        <testExcludes>
            <exclude>com/example/demo/**Test*</exclude>
            <exclude>com/example/demo/**/**Test*</exclude>
        </testExcludes>        
    </configuration>                                                                                                                          
</plugin>
```


## maven-jar-plugin

打包jar文件时，配置manifest文件，加入lib包的jar依赖

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <configuration>
        <classesDirectory>target/classes/</classesDirectory>
        <archive>
            <manifest>
                <mainClass>com.alibaba.dubbo.container.Main</mainClass>
                <!-- 打包时 MANIFEST.MF文件不记录的时间戳版本 -->
                <!--自动加载META-INF/spring目录下的所有Spring配置-->
                <useUniqueVersions>false</useUniqueVersions>
                <addClasspath>true</addClasspath>
                <classpathPrefix>lib/</classpathPrefix>
            </manifest>
            <manifestEntries>
                <Class-Path>.</Class-Path>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```

## maven-war-plugin 

打包war项目的时候排除某些web资源文件，这时就应该配置maven-war-plugin如下：

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-war-plugin</artifactId>
  <version>2.1.1</version>
  <configuration>
    <webResources>
      <resource>
        <directory>src/main/webapp</directory>
        <excludes>
          <exclude>**/*.jpg</exclude>
        </excludes>
      </resource>
    </webResources>
  </configuration>
</plugin>
```

## maven-source-plugin

打包源码, 注意：在多项目构建中，将source-plugin置于顶层或parent的pom中并不会发挥作用，必须置于具体项目的pom中。

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-source-plugin</artifactId>
    <configuration>  
        <!-- <finalName>${project.build.name}</finalName> -->  
        <attach>true</attach>  
        <encoding>${project.build.sourceEncoding}</encoding>  
    </configuration>
    <executions>
        <execution>
            <id>attach-sources</id>
            <phase>compile</phase>
            <goals>
                <goal>jar</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

## maven-javadoc-plugin 

生成javadoc包

```xml
<plugin>          
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-javadoc-plugin</artifactId>
  <version>2.7</version>
  <executions>
    <execution>
      <id>attach-javadocs</id>
        <goals>
          <goal>jar</goal>
        </goals>
    </execution>
  </executions>
</plugin> 
```

## maven-surefire-plugin 

打包时跳过单元测试

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>2.16</version>
    <configuration>
        <!-- <skip>true</skip> -->
        <skipTests>true</skipTests>
        <forkMode>once</forkMode>
        <argLine>-Dfile.encoding=UTF-8</argLine>
        <testFailureIgnore>true</testFailureIgnore>
</plugin>
```

mvn package -Dmaven.test.skip=true 


## maven-resource-plugin

说明：该插件处理项目的资源文件拷贝到输出目录。可以分别处理main resources 和 test resources。

```xml
<plugin>
   <groupId>org.apache.maven.plugins</groupId>
   <artifactId>maven-resources-plugin</artifactId>
   <version>3.0.1</version>
   <configuration>
         <encoding>UTF-8</encoding>
    </configuration>
   <executions>  
        <execution>  
            <phase>compile</phase>  
        </execution>  
    </executions>  
    <configuration>  
        <encoding>${project.build.sourceEncoding}</encoding>  
    </configuration> 
</plugin>
```

## maven-dependency-plugin

自动拷贝jar包到target目录

```xml
<plugin>  
    <groupId>org.apache.maven.plugins</groupId>  
    <artifactId>maven-dependency-plugin</artifactId>  
    <version>2.6</version>  
    <executions>  
        <execution>  
            <id>copy-dependencies</id>  
            <phase>compile</phase>  
            <goals>  
                <goal>copy-dependencies</goal>  
            </goals>  
            <configuration>  
                <!-- ${project.build.directory}为Maven内置变量，缺省为target -->  
                <outputDirectory>${project.build.directory}/lib</outputDirectory>  
                <!-- 表示是否不包含间接依赖的包 -->  
                <excludeTransitive>false</excludeTransitive>  
                <!-- 表示复制的jar文件去掉版本信息 -->  
                <stripVersion>true</stripVersion>  
            </configuration>  
        </execution>  
    </executions>  
</plugin>  
```

## maven-jetty-plugin 

```xml
<plugin>
    <groupId>org.mortbay.jetty</groupId>
    <artifactId>maven-jetty-plugin</artifactId>
    <version>6.1.5</version>
    <configuration>
        <webAppSourceDirectory>src/main/webapp</webAppSourceDirectory>
        <scanIntervalSeconds>3</scanIntervalSeconds>
        <contextPath>/</contextPath>
        <connectors>
            <connector implementation="org.mortbay.jetty.nio.SelectChannelConnector">
                <port>8088</port>
            </connector>
        </connectors>
    </configuration>
</plugin> 
```

## maven-assembly-plugin

该插件允许用户整合项目的输出,包括依赖,模块,网站文档和其他文档到一个单独的文档，即可用定制化打包。
创建的文档格式包括:zip, tar, tar.gz(tgz), gar.bz2(tbgz2), jar, dir,war 等等。四种预定义的描述器可用：bin, jar-with-dependencies, src, project.

```xml
<plugin>
  <artifactId>maven-assembly-plugin</artifactId>
  <version>3.0.0</version>
  <configuration>
    <descriptorRefs>
      <descriptorRef>jar-with-dependencies</descriptorRef>
    </descriptorRefs>
  </configuration>
  <executions>
    <execution>
      <id>make-assembly</id> <!-- this is used for inheritance merges -->
      <phase>package</phase> <!-- bind to the packaging phase -->
      <goals>
        <goal>single</goal>
      </goals>
    </execution>
  </executions>
</plugin>
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.0.0 http://maven.apache.org/xsd/assembly-2.0.0.xsd">
  <id>bin</id>
  <formats>
    <format>tar.gz</format>
    <format>tar.bz2</format>
    <format>zip</format>
  </formats>
  <fileSets>
    <fileSet>
      <directory>${project.basedir}</directory>
      <outputDirectory>/</outputDirectory>
      <includes>
        <include>README*</include>
        <include>LICENSE*</include>
        <include>NOTICE*</include>
      </includes>
    </fileSet>
    <fileSet>
      <directory>${project.build.directory}</directory>
      <outputDirectory>/</outputDirectory>
      <includes>
        <include>*.jar</include>
      </includes>
    </fileSet>
    <fileSet>
      <directory>${project.build.directory}/site</directory>
      <outputDirectory>docs</outputDirectory>
    </fileSet>
  </fileSets>
</assembly>
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.0.0 http://maven.apache.org/xsd/assembly-2.0.0.xsd">
  <!-- TODO: a jarjar format would be better -->
  <id>jar-with-dependencies</id>
  <formats>
    <format>jar</format>
  </formats>
  <includeBaseDirectory>false</includeBaseDirectory>
  <dependencySets>
    <dependencySet>
      <outputDirectory>/</outputDirectory>
      <useProjectArtifact>true</useProjectArtifact>
      <unpack>true</unpack>
      <scope>runtime</scope>
    </dependencySet>
  </dependencySets>
</assembly>
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/ASSEMBLY/2.0.0 http://maven.apache.org/xsd/assembly-2.0.0.xsd">
  <id>src</id>
  <formats>
    <format>tar.gz</format>
    <format>tar.bz2</format>
    <format>zip</format>
  </formats>
  <fileSets>
    <fileSet>
      <directory>${project.basedir}</directory>
      <includes>
        <include>README*</include>
        <include>LICENSE*</include>
        <include>NOTICE*</include>
        <include>pom.xml</include>
      </includes>
      <useDefaultExcludes>true</useDefaultExcludes>
    </fileSet>
    <fileSet>
      <directory>${project.basedir}/src</directory>
      <useDefaultExcludes>true</useDefaultExcludes>
    </fileSet>
  </fileSets>
</assembly>
```

 