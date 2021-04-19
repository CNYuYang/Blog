# SecurityManager

## 前言

之前在看一些框架源码的时候，总是会遇到类似下面这样的代码。每次遇到都不是很清楚`doPrivileged`方法的作用是什么。为了解答自己的疑惑，就去看了有关Java安全相关的知识点，于是就有了这篇文章。

文本将从介绍Java安全模型开始，通过对Java安全模型的简单介绍，揭开`doPrivileged()`的面纱。

```java
AccessController.doPrivileged(
    new PrivilegedAction<String>() {
        public String run() {
            return java.security.Security.getProperty(
                "package.access");
        }
    }
);
```

## Java安全模型

在开始介绍`doPrivileged()`的作用之前，我们有必要先了解下Java的安全模型。因为`doPrivileged()`方法是在Java的安全框架之下发挥作用的。

Java语言随着互联网的兴起而逐步走向成熟，Java在设计之初就做了安全上的考虑。Java安全模型的设计需要考虑如何保证Java程序可以安全地访问资源，并进行可控的授权。下面，我们开始介绍Java的安全模型。

### 沙箱模型

Java安全模型的最初版本（Java 1.0版本）是一个沙箱模型。沙箱模型提供了一个非常受限的运行环境供应用程序运行。运行在沙箱中的程序没有访问操作系统资源的权限，比如访问文件、网络等等。这个版本的安全模型有效地将远程代码和本地代码隔离开，防止恶意的远程代码对本地计算机系统的破坏。

![JDK 1.0 沙箱模型](./img/01-01.png)

虽然严格的限制可以防止系统被破话，但是这也给程序的功能扩展带来了障碍。当一些没有权限的代码希望访问系统资源的时候，沙箱模型的限制将阻碍这些功能的实现。所以，在后续的Java 1.1版本中，为了可以对代码的资源访问进行授权，引入了安全策略。

![JDK 1.1 沙箱模型](./img/01-02.png)

在Java 1.2中，又再次对Java的沙箱模型进行了改进，增加了代码签名机制。不管是远程代码还是本地代码，都需要按照安全策略的配置，由类加载器加载到JVM中权限不同的运行空间，进行权限的差异化控制。

![JDK 1.2 沙箱模型](./img/01-03.png)

### 域模型

在当前最新的Java安全模型中，引入了 **域（Domain）** 的概念。虚拟机会将所有代码加载到不同的域中。其中系统域负责和操作系统的资源进行交互，而各个应用域对系统资源的访问需要通过系统域的代理来实现受限访问。JVM中的不同域关联了不同的权限，处于域中的类将拥有这个域所包含的所有权限。

![域模型](./img/01-04.png)

由于当前Java的安全模型是使用域模型来实现的，所以，下面我们将介绍基于域模型的Java安全模型是如何进行权限访问控制的。

## SecurityManager

`java.security.SecurityManager`是Java安全模型中很重要的一个API。正如其名，SecurityManager是一个管理器，它提供了一系列的API来实现权限的管理和控制。

### 启动

默认情况下，Java的安全模型是不启用的。为了使用Java的安全模型，需要通过初始化安全管理器（SecurityManager）来启用Java安全模型。

```java
// 获取安全管理器，如果安全管理器未安装，则返回null
SecurityManager manager = System.getSecurityManager();
if (manager == null) {
  // 安装安全管理器
  System.setSecurityManager(new SecurityManager());
}
```

没有安装安全管理器的情况下，调用`System.getSecurityManager()`返回的是`null`。通过`System.setSecurityManager(new SecurityManager())`可以安装和初始化一个安全管理器。

### 使用

**SecurityManager** 提供了一系列供用户调用的API来做权限检查。SecurityManager的典型用法如下：

```java
SecurityManager security = System.getSecurityManager();
if (security != null) {
    security.checkXXX(argument,  . . . );
}
```

鉴于篇幅原因，具体的API可以参考[JDK7](https://docs.oracle.com/javase/7/docs/api/java/lang/SecurityManager.html)文档，这里只是简单介绍下如何使用SecurityManager来对文件访问进行权限检查。

```java
/**
 * Throws a <code>SecurityException</code> if the
 * calling thread is not allowed to read the file specified by the
 * string argument.
 * <p>
 * This method calls <code>checkPermission</code> with the
 * <code>FilePermission(file,"read")</code> permission.
 * <p>
 * If you override this method, then you should make a call to
 * <code>super.checkRead</code>
 * at the point the overridden method would normally throw an
 * exception.
 *
 * @param      file   the system-dependent file name.
 * @exception  SecurityException if the calling thread does not have
 *             permission to access the specified file.
 * @exception  NullPointerException if the <code>file</code> argument is
 *             <code>null</code>.
 * @see        #checkPermission(java.security.Permission) checkPermission
 */
public void checkRead(String file);
```

一个文件读权限检查的例子

```java
public class Test {
    public static void main(String ...args) throws Exception {
        if (System.getSecurityManager() == null) {
            System.setSecurityManager(new SecurityManager());
        }
        System.getSecurityManager().checkRead("foo.txt");
    }
}
```

输出结果

```java
Exception in thread "main" java.security.AccessControlException: access denied ("java.io.FilePermission" "foo.txt" "read")
	at java.security.AccessControlContext.checkPermission(AccessControlContext.java:472)
	at java.security.AccessController.checkPermission(AccessController.java:884)
	at java.lang.SecurityManager.checkPermission(SecurityManager.java:549)
	at java.lang.SecurityManager.checkRead(SecurityManager.java:888)
	at java.io.FileInputStream.<init>(FileInputStream.java:127)
	at java.io.FileInputStream.<init>(FileInputStream.java:93)
	at org.learn.agent.security.ReadFile.main(ReadFile.java:15)
```

如果我们去看`FileInputStream`的构造方法，也可以看到类似上面的代码。

```java
public FileInputStream(File file) throws FileNotFoundException {
    String name = (file != null ? file.getPath() : null);
    SecurityManager security = System.getSecurityManager();
    if (security != null) {
        security.checkRead(name);
    }
    ....
}
```

## AccessController

SecurityManager提供了权限检查的API，但是如果你去看SecurityManager的源码，你会发现，实际的权限检查是委托给了AccessController这个类去做的。AccessController本身不能被实例化，不过AccessController提供了一系列静态方法来进行权限检查、特权标记、获取权限访问控制上下文等操作。

```java
public void checkRead(String file) {
    checkPermission(new FilePermission(file,
        SecurityConstants.FILE_READ_ACTION));
}

public void checkPermission(Permission perm) {
    java.security.AccessController.checkPermission(perm);
}
```

可以看到，`checkRead`方法最终是调用了AccessController的`checkPermission`方法来进行文件权限检查，checkPermission的参数是一个`java.security.Permission`对象。

### 实现访问控制的基础

前面提到，Java当前采用的Java安全模型是域模型。在域模型中，为了实现权限控制，涉及到一些概念，包括：

1. 代码源(code source)：加载Java类的地址
2. 权限（permission）：封装特定操作的请求
3. 策略（policy）：授权的规则
4. 保护域（protection domain）：封装代码和代码所拥有的权限

![访问控制元素](./img/01-05.png)

#### 代码源

代码源表示类的来源URL地址，代码源由类加载器负责创建和管理。

```java
/**
 * Constructs a CodeSource and associates it with the specified
 * location and set of certificates.
 *
 * @param url the location (URL).
 *
 * @param certs the certificate(s). It may be null. The contents of the
 * array are copied to protect against subsequent modification.
 */
public CodeSource(URL url, java.security.cert.Certificate certs[]) {
    this.location = url;

    // Copy the supplied certs
    if (certs != null) {
        this.certs = certs.clone();
    }
}
```

#### 权限

权限使用Permission表示，AccessController通过对Permission的判断来进行权限控制。Permission是一个抽象类，不同的权限有具体的Permission实现。关于permission对象的详细信息可以参看文档 [Permission](https://docs.oracle.com/javase/6/docs/technotes/guides/security/permissions.html)。

#### 策略

AccessController使用安全策略（Policy）表示一个代码源的具体访问规则。JVM默认的策略文件位于`${JAVA_HOME}/jre/lib/security/java.policy`。当然，也可以在启动JVM的时候通过`-Djava.security.policy=policy_filename`来指定安全策略文件。安全策略文件的格式一般如下：

```java
grant codebase {
  permission PermissionClass target, action
};
```

其中`codebase`是代码加载的URL路径，如果是本地文件，可以用`file:`来表示，支持相对路径和绝对路径。相对路径表示相对于当前JVM运行的工作目录。`codebase`如果以目录结尾，则表示这个目录下的所有类文件；如果以文件结尾，则表示这是一个jar包。

1. `file:path/to/classes/dir/` 表示当前工作目录下的class目录
2. `file:/path/to/classes/foo.jar` 表示根目录（unix环境）下的jar包
3. `file://path/to/classes/dir/` 和1一样
4. `file:///path/to/classes/foo.jar` 和2一样

codebase可以使用变量，比如 `file:${ {java.ext.dirs} }/*`。

Java的通过Policy来封装策略，在一个JVM实例中，只能存在一个Policy实例。通过`Policy.getPolicy()`可以获取策略对象，也可以通过`Policy.setPolicy(newPolicy)`来设置新的Policy实例。

#### 保护域

保护域（ProtectionDomain）可以理解为是代码源和权限映射关系的集合。一个类如果属于一个保护域，那么这个类将拥有这个域中的所有权限。

有了上面提到的四个基础，具体的访问控制逻辑就可以由AccessController来实现了。下面我们来看看AccessController是如何做访问控制的。

### 访问控制的机制

下面的这段代码，AccessController调用了`checkPermission`对`FilePermission`进行了权限判断。

```java
FilePermission perm = new FilePermission("path/file", "read");
AccessController.checkPermission(perm);
```

那么，AccessController是如何判断是否有读权限的呢？

首先，AccessController判断权限的主体是调用者（Caller）。基于访问控制的规则，AccessController在进行权限判断的时候，它不仅仅检查当前Caller是否拥有权限，而是对整个调用链上的所有Caller进行权限检查。对调用链上的每个Caller，都会基于它们各自所属的ProtectionDomain中的权限集合进行检查。当满足下面的二个条件，则表示访问被授权：

1. 在当前调用链上，从当前的Caller到初始Caller之间的所有Caller都能被各自所属的ProtectionDomain中的权限授权。
2. 在当前调用链上，其中有一个Caller被标记为privilege并且被授权，同时接下来调用的过程中所有调用都能被各自的域授权。

如果上述的二点有任何一点不满足，则`AccessController.checkPermission()`会抛出`AccessControlException`。

![访问控制元素](./img/01-06.png)

在第一点中可以看到，这个判断过程需要对调用链上所有已经经过的Caller都进行判断。这个过程一般可以分为两种执行策略。一种是每次调用的时候都进行权限判断，如上图左边部分所示。还有一种，是只有当遇到调用`AccessController.checkPermission()`进行权限判断的时候，从当前Caller开始，顺着调用链向上回溯，过程可以参考上图右边部分。试想，当整个调用链中没有遇到权限检查的时候，第一种方案仍然需要进行权限检查，而后一种方案则更加高效。当前AccessController的权限检查策略，采用的就是后一种方案。具体的逻辑可以用伪代码表示：

```java
i = m;
while (i > 0) {
    if (caller i in its domain does not have the permission)
        throw AccessControlException
    else if (caller i is marked as privileged) 
        return;
    i = i - 1;
};
```

在第二点中提到，一个Caller可以被标记为privilege。这里就有一个问题，一个Caller怎么才算被标记为privilege呢？到这里，我们终于可以开始介绍本文开头疑惑的问题了。这个privilege标记和doPrivileged()之间是什么关系呢？下面我们就来介绍doPrivileged()的作用原理。

## doPrivileged的作用

AccessController引入了一个`doPrivileged()`静态方法，只要Caller执行了doPrivileged()方法，那么这个Caller就会被标记为privilege，Java安全模型就不会去检查这个Caller的权限。也就是说，调用`doPrivileged()`的Caller被授予了特权，这个Caller可以免去权限检查。在进行权限检查的时候，回溯调用链的过程中，一旦遇到被标记为privilege的Caller，那么AccessController将停止向上回溯，权限检查通过。

![访问控制元素](./img/01-07.png)

```java
i = m;
while (i > 0) {
    if (caller i in its domain does not have the permission)
        throw AccessControlException
    else if (caller i is marked as privileged) 
    // 停止检查，直接返回
        return;
    i = i - 1;
};
```

doPrivileged方法一般使用方式如下：

```java
somemethod() {
     ...
     AccessController.doPrivileged(new PrivilegedAction<Void>() {
         public Void run() {
             System.loadLibrary("awt");
             return null;
         }
     });
     ...
}
```

PrivilegedAction是一个接口，只有一个`run`方法。`doPrivileged(PrivilegedAction<T> action)`支持一个`PrivilegedAction`类型的参数，当Caller被标记特权后会执行PrivilegedAction的`run`方法，返回`run()`方法的返回值。

```java
public interface PrivilegedAction<T> {
    /**
     * Performs the computation.  This method will be called by
     * {@code AccessController.doPrivileged} after enabling privileges.
     *
     * @return a class-dependent value that may represent the results of the
     *         computation. Each class that implements
     *         {@code PrivilegedAction}
     *         should document what (if anything) this value represents.
     * @see AccessController#doPrivileged(PrivilegedAction)
     * @see AccessController#doPrivileged(PrivilegedAction,
     *                                     AccessControlContext)
     */
    T run();
}
```

如果PrivilegedAction中的`run`方法执行过程中会抛出检查异常，则可以用`PrivilegedExceptionAction`代替。

```java
somemethod() throws FileNotFoundException {
     ...
   try {
     FileInputStream fis = (FileInputStream)
      AccessController.doPrivileged(
       new PrivilegedExceptionAction<InputStream>() {
         public InputStream run() throws FileNotFoundException {
             return new FileInputStream("someFile");
         }
       }
     );
   } catch (PrivilegedActionException e) {
     // e.getException() should be an instance of
     // FileNotFoundException,
     // as only "checked" exceptions will be "wrapped" in a
     // <code>PrivilegedActionException</code>.
     throw (FileNotFoundException) e.getException();
   }
     ...
}
```

PrivilegedExceptionAction定义如下：

```java
public interface PrivilegedExceptionAction<T> {
    /**
     * Performs the computation.  This method will be called by
     * {@code AccessController.doPrivileged} after enabling privileges.
     *
     * @return a class-dependent value that may represent the results of the
     *         computation.  Each class that implements
     *         {@code PrivilegedExceptionAction} should document what
     *         (if anything) this value represents.
     * @throws Exception an exceptional condition has occurred.  Each class
     *         that implements {@code PrivilegedExceptionAction} should
     *         document the exceptions that its run method can throw.
     * @see AccessController#doPrivileged(PrivilegedExceptionAction)
     * @see AccessController#doPrivileged(PrivilegedExceptionAction,AccessControlContext)
     */

    T run() throws Exception;
}
```

下面，我们通过一个例子来看看`doPrivileged()`方法是如何使用的。

## 例子

下面我们通过FilePermission的例子来演示。首先我们创建两个类：Main.java和FileReader.java

```java
// Main
import java.security.AccessController;
import java.security.PrivilegedExceptionAction;

public class Main {
    public static void main(String ...args) throws Exception {
        if (System.getSecurityManager() == null) {
            System.setSecurityManager(new SecurityManager());
        }
        FileReader reader = new FileReader();
        reader.read("test/foo.txt");
    }
}

// FileReader
import java.io.FileInputStream;
import java.io.InputStream;

public class FileReader {
    public void read(String fileName) throws Exception {
        System.out.println(fileName);
        InputStream in = new FileInputStream(fileName);
    }
}
```

项目编译以后的目录结构如下：

```
$ javac -sourcepath src src/Main.java -d output
$ tree
├── output
│   ├── FileReader.class
│   └── Main.class
├── src
│   ├── FileReader.java
│   └── Main.java
└── test
    └── foo.txt
```

现在，我们在没有使用任何策略文件的情况下，执行Main的`main()`方法，会发现抛出了`java.securit.AccessControlException`异常，提示没有对`test/foo.txt`的读权限。

```java
$ java -cp output Main
Exception in thread "main" java.security.AccessControlException: access denied ("java.io.FilePermission" "test/foo.txt" "read")
	at java.security.AccessControlContext.checkPermission(AccessControlContext.java:472)
	at java.security.AccessController.checkPermission(AccessController.java:884)
	at java.lang.SecurityManager.checkPermission(SecurityManager.java:549)
	at java.lang.SecurityManager.checkRead(SecurityManager.java:888)
	at java.io.FileInputStream.<init>(FileInputStream.java:127)
	at java.io.FileInputStream.<init>(FileInputStream.java:93)
	at FileReader.read(FileReader.java:7)
	at Main.main(Main.java:10)
```

现在，我们创建一个policy文件`my.policy`，为了让我们的类有读的权限，我们在这个policy文件中定义如下策略：

```java
grant codebase "file:./output/" {
	permission java.io.FilePermission "./test/*", "read";
};
```

这个策略文件表示所有从目录`output`中加载的类都有对`test`目录下文件的读权限。现在，我们加上JVM参数 `-Djava.security.policy=my.policy`再运行程序，可以发现代码可以正常运行了。

```
$ tree
├── my.policy
├── output
│   ├── FileReader.class
│   └── Main.class
├── src
│   ├── FileReader.java
│   └── Main.java
└── test
    └── foo.txt
$ java -cp output -Djava.security.policy=my.policy Main
test/foo.txt
```

下面，我再调整下代码来演示doPrivileged的作用。我们先把刚才的两个类放到不同的目录下，然后对其中一个类进行授权，另一个类不授权，观察doPrivileged是否生效。

```java
import java.security.AccessController;
import java.security.PrivilegedExceptionAction;

public class Main {
    public static void main(String ...args) throws Exception {
        if (System.getSecurityManager() == null) {
            System.setSecurityManager(new SecurityManager());
        }
        AccessController.doPrivileged(new PrivilegedExceptionAction<Void>() {
            @Override
            public Void run() throws Exception {
	    	        FileReader reader = new FileReader();
	    	        reader.read("test/foo.txt");
            }
        });
    }
}
```

授权文件`my.policy`如下：

```java
grant codebase "file:./output/" {
	permission java.io.FilePermission "./test/*", "read";
};
```

为了让Main类不授权，将Main类从output移到外面。

```
$ tree
.
├── Main$1.class
├── Main.class
├── my.policy
├── output
│   └── FileReader.class
├── src
│   ├── FileReader.java
│   └── Main.java
└── test
    └── foo.txt
```

首先，我们看下两个类都不授权的场景：

```java
$ java -cp .:output Main
test/foo.txt
Exception in thread "main" java.security.AccessControlException: access denied ("java.io.FilePermission" "test/foo.txt" "read")
	at java.security.AccessControlContext.checkPermission(AccessControlContext.java:472)
	at java.security.AccessController.checkPermission(AccessController.java:884)
	at java.lang.SecurityManager.checkPermission(SecurityManager.java:549)
	at java.lang.SecurityManager.checkRead(SecurityManager.java:888)
	at java.io.FileInputStream.<init>(FileInputStream.java:127)
	at java.io.FileInputStream.<init>(FileInputStream.java:93)
	at FileReader.read(FileReader.java:7)
	at Main$1.run(Main.java:13)
	at Main$1.run(Main.java:9)
	at java.security.AccessController.doPrivileged(Native Method)
	at Main.main(Main.java:9)
```

现在，我们对FileReader类授权：

```
$ java -cp .:output -Djava.security.policy=my.policy Main
test/foo.txt
```

可以看到，虽然我们只是对FileReader进行了授权，但是由于`doPrivileged()`的作用，我们仍然可以安全得调用FileReader的`read`方法。

## 总结

`AccessController.doPrivileged`机制的存在，可以允许我们在自己的代码没有授权，而调用模块代码被授权的情况下进行受限资源的访问。

试想，如果没有这种特权机制，而我们又需要调用一个访问受限资源的模块。为了实现这个功能，我们就需要对自己的代码也授予相同的权限（或者说，需要整个调用链路上的所有调用者都进行授权）才能成功地调用该模块的代码。而通过`AccessController.doPrivileged`机制，可以简化这个流程。这在一定程度上也是对权限粒度的控制，不至于权限放的太开。

通过对Java安全模型的介绍，我们已经基本解释了`AccessController.doPrivileged()`的作用。后面在阅读类似源码的时候就可以做到心中有数。