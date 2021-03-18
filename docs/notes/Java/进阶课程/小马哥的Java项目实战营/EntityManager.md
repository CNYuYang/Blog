# EntityManager

## 简介

`EntityManager`是`Java Persistence API`的一部分。首先，它实现了JPA 2.0规范定义的编程接口和生命周期规则。

此外，我们可以使用EntityManager中的API来访问持久性上下文。

在本中，我们将研究EntityManager的配置，类型和各种API 。

## Maven依赖

首先我们需要引入Hibernate的依赖：

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>5.4.0.Final</version>
</dependency>
```

然后，我们也要引入数据库驱动相关的依赖：

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.13</version>
</dependency>
```

## 配置

现在开始演示`EntityManager`的使用，将`Movie`实体类与数据库中的`MOVIE`表对应起来。

### 定义一个实体类

通过`@Entiy`注解创建一个数据库对应的表：

```java
@Entity
@Table(name = "MOVIE")
public class Movie {
    
    @Id
    private Long id;

    private String movieName;

    private Integer releaseYear;

    private String language;

    // standard constructor, getters, setters
}
```

###  创建*persistence.xml*文件 

当`EntityManagerFactory`被创建的时候，JPA的实现类就会加载读取`META-INF/persistence.xml`文件。

文件里面的内容，便是`EntityManager`的配置。

```xml
<persistence-unit name="com.baeldung.movie_catalog">
    <description>Hibernate EntityManager Demo</description>
    <class>com.baeldung.hibernate.pojo.Movie</class> 
    <exclude-unlisted-classes>true</exclude-unlisted-classes>
    <properties>
        <property name="hibernate.dialect" value="org.hibernate.dialect.MySQL5Dialect"/>
        <property name="hibernate.hbm2ddl.auto" value="update"/>
        <property name="javax.persistence.jdbc.driver" value="com.mysql.jdbc.Driver"/>
        <property name="javax.persistence.jdbc.url" value="jdbc:mysql://127.0.0.1:3306/moviecatalog"/>
        <property name="javax.persistence.jdbc.user" value="root"/>
        <property name="javax.persistence.jdbc.password" value="root"/>
    </properties>
</persistence-unit>
```

我们定义了`persistence-unit`，该`persistence-unit`指定了由`EntityManager`管理的实体类。

此外，我们定义了数据库的方言和其他`JDBC`属性，这些属性将用于`Hibernate`与基础数据库连接。

## 创建EntityManager

创建`EntityManager`有两种方式，容器注入与工厂方式。

### 容器注入

入下面代码所示，容器将`EntityManager`注入到组件中。

但是容器注入的方式，其实也是框架使用工厂方式创建的对象。

```java
@PersistenceContext
EntityManager entityManager;
```

同时，当你使用容器注入的方式创建`EntityManager`，那也意味着框架帮你管理事务的创建、提交、回滚等。

同样，上述容器也负责关闭`EntityManager`， 因此 无需手动清理就可以安全使用。即使我们尝试关闭容器管理的`EntityManager`，它也应该抛出`IllegalStateException`。

### 工厂方式创建

相反，`EntityManager`的生命周期由此处的应用程序管理。

实际上，我们将手动创建`EntityManager`。此外，我们还将管理已创建的`EntityManager`的生命周期 。

首先，让我们创建`EntityManagerFactory`：

```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("com.baeldung.movie_catalog");
```

为了创建`EntityManager`，我们必须在`EntityManagerFactory`中显式调用*createEntityManager()*：

```java
public static EntityManager getEntityManager() {
    return emf.createEntityManager();
}
```

由于我们负责创建`EntityManager`实例，因此关闭它们也是我们的责任。因此，在使用完每个`EntityManager`后，我们应该关闭它们。

### 线程安全

`EntityManagerFactory`是线程安全的，所以可以这样写：

```java
EntityManagerFactory emf = // fetched from somewhere
EntityManager em = emf.createEntityManager();
```

另一方面，`EntityManager`是线程不安全的，所以每个线程应该获取自己的`EntityManager`，然后使用它，关闭它。

```java
EntityManagerFactory emf = // fetched from somewhere 
EntityManager em = emf.createEntityManager();
// use it in the current thread
```

但是，使用容器注入方式的时候，情况又恰好相反：

```java
@Service
public class MovieService {

    @PersistenceContext // or even @Autowired
    private EntityManager entityManager;
    
    // omitted
}
```

无论哪种方式，容器都确保每个线程都有自己的`EntityManager`。

## Hibernate相关操作

该`EntityManager`的API提供的方法的集合。通过使用这些方法，我们可以与数据库进行交互。

### Persisting Entities

为了把对象联系起来，我们使用*persist()*方法

```java
public void saveMovie() {
    EntityManager em = getEntityManager();
    
    em.getTransaction().begin();
    
    Movie movie = new Movie();
    movie.setId(1L);
    movie.setMovieName("The Godfather");
    movie.setReleaseYear(1972);
    movie.setLanguage("English");

    em.persist(movie);
    em.getTransaction().commit();
}
```

一旦对象保存在数据库中，它就处于*persistent*状态。

### Loading Entities

为了从数据库中检索对象，我们可以使用*find()*方法。

在此，该方法通过主键进行搜索。实际上，该方法需要实体类类型和主键：

```java
public Movie getMovie(Long movieId) {
    EntityManager em = getEntityManager();
    Movie movie = em.find(Movie.class, new Long(movieId));
    em.detach(movie);
    return movie;
}
```

但是，如果只需要对实体的引用，则可以改用*getReference()*方法。实际上，它将代理返回给实体：

```java
Movie movieRef = em.getReference(Movie.class, new Long(movieId));
```

### Detaching Entities

如果需要将实体与持久性上下文分离，可以使用*detach()*方法。我们将要分离的对象作为参数传递给方法：

```java
em.detach(movie);
```

一旦实体与持久性上下文分离，它将处于分离状态。

### Merging Entities

合并方法有助于在托管实体中引入对分离实体所做的修改

```java
public void mergeMovie() {
    EntityManager em = getEntityManager();
    Movie movie = getMovie(1L);
    em.detach(movie);
    movie.setLanguage("Italian");
    em.getTransaction().begin();
    em.merge(movie);
    em.getTransaction().commit();
}
```

### Querying for Entities

此外，我们可以利用JPQL来查询实体。我们将调用*getResultList()*来执行它们。

当然，如果查询仅返回单个对象，我们可以使用*getSingleResult()*：

```java
public List<?> queryForMovies() {
    EntityManager em = getEntityManager();
    List<?> movies = em.createQuery("SELECT movie from Movie movie where movie.language = ?1")
      .setParameter(1, "English")
      .getResultList();
    return movies;
}
```

### Removing Entities

另外，我们可以使用*remove()*方法从数据库中删除实体。重要的是要注意，对象不是分离的而是被移除的。

在这里，实体的状态从持久变为新：

```java
public void removeMovie() {
    EntityManager em = HibernateOperations.getEntityManager();
    em.getTransaction().begin();
    Movie movie = em.find(Movie.class, new Long(1L));
    em.remove(movie);
    em.getTransaction().commit();
}
```

