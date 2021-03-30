# JNDI 依赖注入

## web.xml 配置

可在 Web 应用的部署描述符文件（`/WEB-INF/web.xml`）中使用下列元素来定义资源：

- `<env-entry>` 应用的环境项。一个可用于配置应用运行方式的单值参数。
- `<resource-ref>` 资源引用，通常是引用保存某种资源的对象工厂，比如 JDBC `DataSource` 或 JavaMail `Session` 这样的资源；或者引用配置在 Tomcat 中的自定义对象工厂中的资源。
- `<resource-env-ref>` 资源环境引用。Servlet 2.4 所添加的一种新 `resource-ref`，它简化了不需要认证消息的资源的配置。

有了这些，Tomcat 就能利用适宜的资源工厂来创建资源，再也不需要其他配置信息了。Tomcat 将使用 `/WEB-INF/web.xml` 中的信息来创建资源。

另外，Tomcat 还提供了一些用于 JNDI 的特殊选项，它们没有指定在 web.xml 中。比如，其中包括的 `closeMethod` 能在 Web 应用停止时，迅速清除 JNDI 资源；`singleton` 控制是否会在每次 JNDI 查找时创建资源的新实例。要想使用这些配置选项，资源必须指定在 Web 应用的 [`<Context>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html) 元素内，或者位于 `$CATALINA_BASE/conf/server.xml` 的 [`<GlobalNamingResources>`](http://tomcat.apache.org/tomcat-8.0-doc/config/globalresources.html) 元素中。

## context.xml 配置

如果 Tomcat 无法确定合适的资源工厂，并且/或者需要额外的配置信息，就必须在 Tomcat 创建资源之前指定好额外的具体配置。Tomcat 特定资源配置应位于 `<Context>` 元素内，它可以指定在 `$CATALINA_BASE/conf/server.xml`，或者，最好放在每个 Web 应用的上下文 XML 文件中（`META-INF/context.xml`）。

要想完成 Tomcat 的特定资源配置，需要使用 `<Context>` 元素中的下列元素：

- [`<Environment>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html#Environment_Entries) 对将通过 JNDI 的 `InitialContext` 方法暴露给 Web 应用的环境项的名称与数值加以配置（等同于 Web 应用部署描述符文件中包含了一个 `<env-entry>` 元素）。

- [`<Resource>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html#Resource_Definitions) 定义应用所能用到的资源名称和数据类型（等同于 Web 应用部署描述符文件中包含了一个 `<resource-ref>` 元素）。

- [`<ResourceLink>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html#Resource_Links)加一个链接，使其指向全局 JNDI 上下文中定义的资源。使用资源链接可以使 Web 应用访问定义在 `<Server>` 元素中子元素 `<GlobalNamingResources>` 中的资源。

- [`<Transaction>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html#Trans) 添加一个资源工厂，用于对从 `java:comp/UserTransaction` 获得的 UserTransaction 接口进行实例化。
以上这些元素内嵌于 `<Context>` 元素中，而且是与特定应用相关联的。

如果资源已经定义在 `<Context>` 元素中，那就不必再在部署描述符文件中定义它了。但建议在部署描述符文件中保留相关项，以便记录应用资源需求。

加入同样一个资源名称既被定义在 Web 应用部署描述符文件的 `<env-entry>` 元素中，又被定义在 Web 应用的 `<Context>` 元素的 `<Environment>` 元素内，那么只有当相应的 `<Environment>` 元素允许时（将其中的 `override` 属性设为 `true`），部署描述符文件中的值才会优先对待。

## 全局配置

Tomcat 为整个服务器维护着一个全局资源的独立命名空间。这些全局资源配置在 `$CATALINA_BASE/conf/server.xml` 的 [`<GlobalNamingResources>`](http://tomcat.apache.org/tomcat-8.0-doc/config/globalresources.html) 元素内。可以使用 [`<ResourceLink>`](http://tomcat.apache.org/tomcat-8.0-doc/config/context.html#Resource_Links)将这些资源暴露给 Web 应用，以便在每一应用上下文中将其包含进来。

如果资源已经定义在 `<Context>` 元素中，那就不必再在部署描述符文件中定义它了。但建议在部署描述符文件中保留相关项，以便记录应用资源需求。

## 使用资源

当 Web 应用最初部署时，就配置 `InitialContext`，使其可被 Web 应用的各组件所使用（只读访问）。JNDI 命名空间的 `java:comp/env` 部分中包含着所有的配置项与资源，所以访问资源（在下例中，就是一个 JDBC 数据源）应按如下形式进行：

```java
// 获取环境命名上下文
Context initCtx = new InitialContext();
Context envCtx = (Context) initCtx.lookup("java:comp/env");

// 查找数据源
DataSource ds = (DataSource)
  envCtx.lookup("jdbc/EmployeeDB");

// 分配并使用池中的连接
Connection conn = ds.getConnection();
... use this connection to access the database ...
conn.close();
```