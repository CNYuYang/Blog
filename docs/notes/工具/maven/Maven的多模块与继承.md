# Maven的多模块与继承

通常来说，在Maven的多模块工程中，都存在一个pom类型的工程作为根模块，该工程只包含一个pom.xml文件，在该文件中以模块（module）的形式声明它所包含的子模块，即多模块工程。在子模块的pom.xml文件中，又以parent的形式声明其所属的父模块，即继承。然而，这两种声明并不必同时存在，我们将在下文中讲到这其中的区别。

 

## **创建Maven多模块工程**

 

多模块的好处是你只需在根模块中执行Maven命令，Maven会分别在各个子模块中执行该命令，执行顺序通过Maven的Reactor机制决定。先来看创建Maven多模块工程的常规方法。在我们的示例工程中，存在一个父工程，它包含了两个子工程（模块），一个core模块，一个webapp模块，webapp模块依赖于core模块。这是一种很常见的工程划分方式，即core模块中包含了某个领域的核心业务逻辑，webapp模块通过调用core模块中服务类来创建前端网站。这样将核心业务逻辑和前端展现分离开来，如果之后决定开发另一套桌面应用程序，那么core模块是可以重用在桌面程序中。

首先通过Maven的Archetype插件创建一个父工程，即一个pom类型的Maven工程，其中只包含一个pom.xml文件：

```bash
mvn archetype:generate -DgroupId=me.davenkin -DartifactId=maven-multi-module -DarchetypeGroupId=org.codehaus.mojo.archetypes -DarchetypeArtifactId=pom-root -DinteractiveMode=false
```

以上命令在当前目录下创建了一个名为maven-multi-module的目录，该目录便表示这个pom类型的父工程，在该目录只有一个pom.xml文件：


```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"

 xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

 <modelVersion>4.0.0</modelVersion>

 <groupId>me.davenkin</groupId>

 <artifactId>maven-multi-module</artifactId>

 <version>1.0-SNAPSHOT</version>

 <packaging>pom</packaging>

 <name>maven-multi-module</name>

</project>
```
 

这个pom.xml非常简单，最值得一看的是其中的“<packaging>pom</packaging>”，表示该工程为pom类型。其他的Maven工程类型还有jar、war、ear等。

 

此时，父工程便创建好了，接下来我们创建core模块，由于core模块属于maven-multi-module模块，我们将工作目录切换到maven-multi-module目录下，创建core模块命令如下：

 

```bash
mvn archetype:generate -DgroupId=me.davenkin -DartifactId=core  -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false
```

 

这里我们使用了Archetype插件的maven-archetype-quickstart，它创建一个jar类型的模块。此时，如果我们在打开maven-multi-module模块的pom.xml会发现，其中多了以下内容：

 

```xml
 <modules>

   <module>core</module>

 </modules>
```

 

 

这里的core即是我们刚才创建的core模块，再看看core模块中的pom.xml文件：

 



```xml
<?xml version="1.0"?>

<project xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd" xmlns="http://maven.apache.org/POM/4.0.0"

   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

 <modelVersion>4.0.0</modelVersion>

 <parent>

   <artifactId>maven-multi-module</artifactId>

   <groupId>me.davenkin</groupId>

   <version>1.0-SNAPSHOT</version>

 </parent>

 <groupId>me.davenkin</groupId>

 <artifactId>core</artifactId>

 <version>1.0-SNAPSHOT</version>

 <name>core</name>

 <url>http://maven.apache.org</url>

 <dependencies>

   <dependency>

     <groupId>junit</groupId>

     <artifactId>junit</artifactId>

     <version>3.8.1</version>

     <scope>test</scope>

   </dependency>

 </dependencies>

</project>
```



 

 

请注意里面的  “<parent>...  </parent>”，它将maven-multi-module模块做为了自己的父模块。这里我们看出，当创建core模块时，Maven将自动识别出已经存在的maven-multi-module父模块，然后分别创建两个方向的指引关系，即在maven-multi-module模块中将core作为自己的子模块，在core模块中将maven-multi-module作为自己的父模块。要使Maven有这样的自动识别功能，我们需要在maven-multi-module目录下创建core模块（请参考前文），不然，core模块将是一个独立的模块，但是我们可以通过手动修改两个模块中的pom.xml文件来创建他们之间的父子关系，从而达到同样的目的。

 

依然在maven-multi-module目录下，通过与core相同的方法创建webapp模块：

 

```bash
mvn archetype:generate -DgroupId=me.davenkin -DartifactId=webapp -DarchetypeArtifactId=maven-archetype-webapp -DinteractiveMode=false
```

 

 

这里的maven-archetype-webapp表明Maven创建的是一个war类型的工程模块。此时再看maven-multi-module模块的pom.xml文件，其中的<modules>中多了一个webapp模块：

 



```xml
<modules>

   <module>core</module>

   <module>webapp</module>

 </modules>
```



 

 

而在webapp模块的pom.xml文件中，也以 “<parent>...  </parent>”的方式将maven-multi-module模块声明为自己的父模块，这些同样是得益于Maven自动识别的结果。

 

此时，在maven-multi-module目录下，我们执行以下命令完成整个工程的编译、打包和安装到本地Maven Repository的过程：

 

```bash
mvn clean install
```

 

 

## **手动添加子模块之间的依赖关系**

 

此时我们虽然创建了一个多模块的Maven工程，但是有两个问题我们依然没有解决：

 

（1）没有发挥Maven父模块的真正作用（配置共享）

（2）webapp模块对core模块的依赖关系尚未建立


针对（1），Maven父模块的作用本来是使子模块可以继承并覆盖父模块中的配置，比如dependency等，但是如果我们看看webapp和core模块中pom.xml文件，他们都声明了对Junit的依赖，而如果多个子模块都依赖于相同的类库，我们应该将这些依赖配置在父模块中，继承自父模块的子模块将自动获得这些依赖。所以接下来我们要做的便是：将webapp和core模块对junit的依赖删除，并将其迁移到父模块中。

 

对于（2），Maven在创建webapp模块时并不知道webapp依赖于core，所以这种依赖关系需要我们手动加入，在webapp模块的pom.xml中加入对core模块的依赖：

 



```xml
       <dependency>

           <groupId>me.davenkin</groupId>

           <artifactId>core</artifactId>

           <version>1.0-SNAPSHOT</version>

       </dependency>
```


此时再在maven-multi-module目录下执行 “mvn clean install”，Maven将根据自己的Reactor机制决定哪个模块应该先执行，哪个模块应该后执行。比如，这里的webapp模块依赖于core模块，那么Maven会先在core模块上执行“mvn clean install”，再在webapp模块上执行相同的命令。在webapp上执行“mvn clean install”时，由于core模块已经被安装到了本地的Repository中，webapp便可以顺利地找到所依赖的core模块。

 

总的来看，此时命令的执行顺序为maven-multi-module -> core -> webapp，先在maven-multi-module上执行是因为其他两个模块都将它作为父模块，即对它存在依赖关系，又由于core被webapp依赖，所以接下来在core上执行命令，最后在webapp上执行。

 

这里又有一个问题：为什么非得在maven-multi-module目录下执行 “mvn clean install”？答案是，并不是非得如此，只是你需要搞清楚Maven的工作机制。在maven-multi-module目录下执行，即是在父工程中执行，此时Maven知道父模块所包含的所有子模块，并会自动按照模块依赖关系处理执行顺序。如果只在子模块中执行，那么Maven并不知道它对其他模块的依赖关系。举个例子，当在webapp中执行 “mvn clean install”，Maven发现webapp自己依赖于core，此时Maven会在本地的Repository中去找core，如果存在，那么你很幸运，如果不存在，那么对不起，运行失败，说找不到core，因为Maven并不会先将core模块安装到本地Repository。此时你需要做的是，切换到core目录，执行“mvn clean install”将core模块安装到本地Repository，再切换回webapp目录，执行“mvn clean install”，万事才大吉。

 

多么繁琐的步骤，此时你应该能体会到在maven-multi-module下执行Maven命令的好处了吧。总结一下：在maven-multi-module下执行“mvn clean install”， Maven会在每个模块上执行该命令，然后又发现webapp依赖于core，此时他们之间有一个协调者（即父工程），它知道将core作为webapp的依赖，于是会先在core模块上执行“mvn clean install”，当在webapp上执行命令时，无论先前的core模块是否存在于本地Repository中，父工程都能够获取到core模块（如果不存在于本地Repository，它将现场编译core模块，再将其做为webapp的依赖，比如此时使用“mvn clean package”也是能够构建成功的），所以一切成功。

 

这里又牵扯到Maven如何查找依赖的问题，简单来说，Maven会先在本地Repository中查找依赖，如果依赖存在，则使用该依赖，如果不存在，则通过pom.xml中的Repository配置从远程下载依赖到本地Repository中。默认情况下，Maven将使用[Maven Central Repository](http://search.maven.org/)作为远端Repository。于是你又有问题了：“在pom.xml中为什么没有看到这样的配置信息啊？”原因在于，任何一个Maven工程都默认地继承自一个[Super POM](http://books.sonatype.com/mvnref-book/reference/pom-relationships-sect-pom.html#pom-relationships-sect-super-pom)，Repository的配置信息便包含在其中。

 

## **多模块 vs 继承**

 

在文章一开始我们便提到，在Maven中，由多模块（Project Aggregation）和继承（Project Inheritance）关系并不必同时存在。

 

（1）如果保留webapp和core中对maven-multi-module的父关系声明，即保留 “<parent>...  </parent>”，而删除maven-multi-module中的子模块声明，即“<modules>...<modules>”，会发生什么情况？此时，整个工程已经不是一个多模块工程，而只是具有父子关系的多个工程集合。如果我们在maven-multi-module目录下执行“mvn clean install”，Maven只会在maven-multi-module本身上执行该命令，继而只会将maven-multi-module安装到本地Repository中，而不会在webapp和core模块上执行该命令，因为Maven根本就不知道这两个子模块的存在。另外，如果我们在webapp目录下执行相同的命令，由于由子到父的关系还存在，Maven会在本地的Repository中找到maven-multi-module的pom.xml文件和对core的依赖（当然前提是他们存在于本地的Repository中），然后顺利执行该命令。

 

这时，如果我们要发布webapp，那么我们需要先在maven-multi-module目录下执行“mvn clean install”将最新的父pom安装在本地Repository中，再在core目录下执行相同的命令将最新的core模块安装在本地Repository中，最后在webapp目录下执行相同的命令完成最终war包的安装。麻烦。

 

（2）如果保留maven-multi-module中的子模块声明，而删除webapp和core中对maven-multi-module的父关系声明，又会出现什么情况呢？此时整个工程只是一个多模块工程，而没有父子关系。Maven会正确处理模块之间的依赖关系，即在webapp模块上执行Maven命令之前，会先在core模块上执行该命令，但是由于core和webapp模块不再继承自maven-multi-module，对于每一个依赖，他们都需要自己声明，比如我们需要分别在webapp和core的pom.xml文件中声明对Junit依赖。

 

综上，多模块和父子关系是不同的。如果core和webapp只是在逻辑上属于同一个总工程，那么我们完全可以只声明模块关系，而不用声明父子关系。如果core和webapp分别处理两个不同的领域，但是它们又共享了很多，比如依赖等，那么我们可以将core和webapp分别继承自同一个父pom工程，而不必属于同一个工程下的子模块。更多解析请参考[这里](http://maven.apache.org/guides/introduction/introduction-to-the-pom.html)。