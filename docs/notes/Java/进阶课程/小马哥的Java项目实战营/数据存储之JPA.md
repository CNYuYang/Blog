# 数据存储之JPA 

## Entity

> @Entity说明这个class是实体类，并且使用默认的orm规则，即class名对应数据库表中表名，class字段名即表中的字段名。
> 
>（如果想改变这种默认的orm规则，就要使用@Table来改变class名与数据库中表名的映射规则，@Column来改变class中字段名与db中表的字段名的映射规则）

元数据属性说明：

- name: 表名

下面的代码说明,Customer类对应数据库中的Customer表，其中name为可选，缺省类名即表名！

```java
@Entity(name=”Customer”)
public class Customer { ... }
```

## Table

> Table用来定义entity主表的name，catalog，schema等属性。

元数据属性说明：
- name: 表名
- catalog: 对应关系数据库中的catalog
- schema：对应关系数据库中的schema
- UniqueConstraints:定义一个UniqueConstraint数组，指定需要建唯一约束的列

```java
@Entity
@Table(name="CUST")
public class Customer { ... }
```

## SecondaryTable

> 一个entity class可以映射到多表，SecondaryTable用来定义单个从表的名字，主键名字等属性。

元数据属性说明：
- name: 表名
- catalog: 对应关系数据库中的catalog
- schema：对应关系数据库中的schema
- pkJoin: 定义一个PrimaryKeyJoinColumn数组，指定从表的主键列
- UniqueConstraints: 定义一个UniqueConstraint数组，指定需要建唯一约束的列

下面的代码说明Customer类映射到两个表，主表名是CUSTOMER，从表名是CUST_DETAIL，从表的主键列和主表的主键列类型相同，列名为CUST_ID。


```java
@Entity
@Table(name="CUSTOMER")
@SecondaryTable(name="CUST_DETAIL",pkJoin=@PrimaryKeyJoinColumn(name="CUST_ID"))
public class Customer { ... }
```

## SecondaryTables

> 当一个entity class映射到一个主表和多个从表时，用SecondaryTables来定义各个从表的属性。

元数据属性说明：
- value： 定义一个SecondaryTable数组，指定每个从表的属性。

```java
@Table(name = "CUSTOMER")
@SecondaryTables( value = {
@SecondaryTable(name = "CUST_NAME", pkJoin = { @PrimaryKeyJoinColumn(name = "STMO_ID", referencedColumnName = "id") }),
@SecondaryTable(name = "CUST_ADDRESS", pkJoin = { @PrimaryKeyJoinColumn(name = "STMO_ID", referencedColumnName = "id") }) })
public class Customer {}
```

## UniqueConstraint

> UniqueConstraint定义在Table或SecondaryTable元数据里，用来指定建表时需要建唯一约束的列。

元数据属性说明：
- columnNames:定义一个字符串数组，指定要建唯一约束的列名。

```java
@Entity
@Table(name="EMPLOYEE",
	uniqueConstraints={@UniqueConstraint(columnNames={"EMP_ID", "EMP_NAME"})}
)
public class Employee { ... }
```

## Column

> Column元数据定义了映射到数据库的列的所有属性：列名，是否唯一，是否允许为空，是否允许更新等。

元数据属性说明：
- name:列名。
- unique: 是否唯一
- nullable: 是否允许为空
- insertable: 是否允许插入
- updatable: 是否允许更新
- columnDefinition: 定义建表时创建此列的DDL
- secondaryTable: 从表名。如果此列不建在主表上（默认建在主表），该属性定义该列所在从表的名字。

```java
public class Person {
@Column(name = "PERSONNAME", unique = true, nullable = false, updatable = true)
private String name;

@Column(name = "PHOTO", columnDefinition = "BLOB NOT NULL", secondaryTable="PER_PHOTO")
private byte[] picture;
```

## OneToOne

`@OneToOne`, 表示一对一的映射关系，比如一个账号对应一个用户，一个实体用来描述账号的信息（账号，密码，账号是否可用，账号对应的角色等），另外一个实体用来描述用户的信息（昵称，性别，国籍等）。

该注解有六个属性：

```java
public @interface OneToOne {
    java.lang.Class targetEntity() default void.class;

    javax.persistence.CascadeType[] cascade() default {};

    javax.persistence.FetchType fetch() default javax.persistence.FetchType.EAGER;

    boolean optional() default true;

    java.lang.String mappedBy() default "";

    boolean orphanRemoval() default false;
}

```

1. targetEntity 关联目标实体类，指定类型后该属性可省略；
2. cascade表示关联关系中的级联操作权限，有五种权限：

    - CascadeType.PERSIST：级联新增（又称级联保存）；
    - CascadeType.MERGE：级联合并，更新该实体时，与其有映射关系的实体也跟随更新；
    - CascadeType.REMOVE：级联删除，删除该实体时，与其有映射关系的实体也跟随删除；
    - CascadeType.REFRESH：级联刷新，该实体被操作前都会刷新，保证数据合法性；
    - CascadeType.ALL：包含以上四种级联操作；

3. fetch数据加载策略，默认值为FetchType.EAGER：

   - FetchType.LAZY 表示数据获取方式为懒加载；
   - FetchType.EAGER 表示数据获取方式为急加载；

4. optional 表示关联关系是否必须，当该值为true时，one的一方可以为null；
5. mappedBy 指定映射关系由哪一方维护，一般使用在双向映射场景；
6. orphanRemoval 孤值删除，将会删除孤立数据，外键为null的数据将被删除；

我们在使用的时候，通常为了保证表的简洁性，将主键共享，意思是用户的id和账号的id是一样的，不在表中单独存在一个字段用来描述关联关系；比如下面的例子：
首先创建一个账号实体

```java
import org.hibernate.annotations.GenericGenerator;
import org.hibernate.annotations.Parameter;
import javax.persistence.*;

@Table(name = "base_account")
@Entity
@org.hibernate.annotations.Table(appliesTo = "base_account", comment = "账号信息表")
public class AccountDO {

    @Id
    @GenericGenerator(name="idGenerator", strategy = "uuid")
    @GeneratedValue(generator = "idGenerator")
    @Column(name = "ACCOUNT_ID", length = 32)
    private String accountId;

    @Column(name = "USERNAME", columnDefinition = "VARCHAR(32) NOT NULL COMMENT '账号'")
    private String username;

    @Column(name = "PASSWORD", columnDefinition = "VARCHAR(128) NOT NULL COMMENT '密码'")
    private String password;

    @OneToOne(cascade = {CascadeType.PERSIST, CascadeType.REMOVE, CascadeType.REFRESH})
    @PrimaryKeyJoinColumn
    private UserDO userDO;
    
    // 省略构造函数，get/set方法，toString方法等
```

创建一个用户信息实体

```java
import org.hibernate.annotations.GenericGenerator;
import org.hibernate.annotations.Parameter;
import javax.persistence.*;

@Table(name = "base_user")
@Entity
@org.hibernate.annotations.Table(appliesTo = "base_user", comment = "用户信息表")
public class UserDO {

    @Id
    @GenericGenerator(name = "idGenerator", strategy = "foreign", parameters = @Parameter(name = "property", value = "accountDO"))
    @GeneratedValue(generator = "idGenerator")
    @Column(name = "USER_ID", length = 32)
    private String userId;

    @Column(name = "NICKNAME", columnDefinition = "VARCHAR(32) NOT NULL COMMENT '昵称'")
    private String nickname;

    @Column(name = "SEX", columnDefinition = "CHAR(2) DEFAULT NULL COMMENT '性别'")
    private String sex;

    @OneToOne(mappedBy = "userDO")
    private AccountDO accountDO;
    
        // 省略构造函数，get/set方法，toString方法等
```

用户实体的主键和账号实体的主键都使用一个生成策略，生成的`id`也一样，且在账号实体中使用`@PrimaryKeyJoinColumn`来声明在表中不建立对应的映射字段。

## ManyToOne

> @ManyToOne表示一个多对一的映射,该注解标注的属性通常是数据库表的外键.

元数据属性说明：
- optional：是否允许该字段为null，该属性应该根据数据库表的外键约束来确定，默认为true
- fetch：表示抓取策略，默认为FetchType.EAGER
- cascade：表示默认的级联操作策略，可以指定为ALL，PERSIST，MERGE，REFRESH和REMOVE中的若干组合，默认为无级联操作
- targetEntity：表示该属性关联的实体类型。该属性通常不必指定，ORM框架根据属性类型自动判断targetEntity。

```java
@ManyToOne(fetch=FetchType,cascade=CascadeType)
```

## OneToMany

> @OneToMany描述一个 一对多的关联,该属性应该为集体类型,在数据库中并没有实际字段。

元数据属性说明：
- fetch：表示抓取策略,默认为FetchType.LAZY,因为关联的多个对象通常不必从数据库预先读取到内存
- cascade：表示级联操作策略,对于OneToMany类型的关联非常重要,通常该实体更新或删除时,其关联的实体也应当被更新或删除

例如：

实体User和Order是OneToMany的关系，则实体User被删除时，其关联的实体Order也应该被全部删除

```java
@OneToMany(fetch=FetchType,cascade=CascadeType)
```

## ManyToMany

> @ManyToMany 描述一个多对多的关联.多对多关联上是两个一对多关联,但是在ManyToMany描述中,中间表是由ORM框架自动处理

元数据属性说明：
- targetEntity:表示多对多关联的另一个实体类的全名,例如:package.Book.class
- mappedBy:表示多对多关联的另一个实体类的对应集合属性名称

两个实体间相互关联的属性必须标记为@ManyToMany,并相互指定targetEntity属性,需要注意的是,有且只有一个实体的@ManyToMany注解需要指定mappedBy属性,指向targetEntity的集合属性名称

利用ORM工具自动生成的表除了User和Book表外,还自动生成了一个User_Book表,用于实现多对多关联

## JoinColumn

> 如果在entity class的field上定义了关系（one2one或one2many等），我们通过JoinColumn来定义关系的属性。JoinColumn的大部分属性和Column类似。

元数据属性说明：
- name:列名。
- referencedColumnName:该列指向列的列名（建表时该列作为外键列指向关系另一端的指定列）
- unique: 是否唯一
- nullable: 是否允许为空
- insertable: 是否允许插入
- updatable: 是否允许更新
- columnDefinition: 定义建表时创建此列的DDL
- secondaryTable: 从表名。如果此列不建在主表上（默认建在主表），该属性定义该列所在从表的名字。

下面的代码说明Custom和Order是一对一关系。在Order对应的映射表建一个名为CUST_ID的列，该列作为外键指向Custom对应表中名为ID的列。

```java
public class Custom {

@OneToOne
@JoinColumn(
name="CUST_ID", referencedColumnName="ID", unique=true, nullable=true, updatable=true)
public order getOrder() {
	return order;
}
```

## JoinColumns

> 如果在entity class的field上定义了关系（one2one或one2many等），并且关系存在多个JoinColumn，用JoinColumns定义多个JoinColumn的属性。

元数据属性说明：
- value: 定义JoinColumn数组，指定每个JoinColumn的属性。

下面的代码说明Custom和Order是一对一关系。在Order对应的映射表建两列，一列名为CUST_ID，该列作为外键指向Custom对应表中名为ID的列,另一列名为CUST_NAME，该列作为外键指向Custom对应表中名为NAME的列。

```java
public class Custom {
@OneToOne
@JoinColumns({
@JoinColumn(name="CUST_ID", referencedColumnName="ID"),
@JoinColumn(name="CUST_NAME", referencedColumnName="NAME")
})
public order getOrder() {
	return order;
}
```

## Id

> 声明当前field为映射表中的主键列。
> id值的获取方式有五种：TABLE, SEQUENCE, IDENTITY, AUTO, NONE。
> Oracle和DB2支持SEQUENCE，SQL Server和Sybase支持IDENTITY,MySQL支持AUTO。
> 所有的数据库都可以指定为AUTO，我们会根据不同数据库做转换。
> NONE (默认)需要用户自己指定Id的值。

元数据属性说明：
- generate():主键值的获取类型
- generator():TableGenerator的名字（当generate=GeneratorType.TABLE才需要指定该属性）

下面的代码声明Task的主键列id是自动增长的。(Oracle和DB2从默认的SEQUENCE取值，SQL Server和Sybase该列建成IDENTITY，mysql该列建成auto increment。)

```java
@Entity
@Table(name = "OTASK")
public class Task {
  @Id(generate = GeneratorType.AUTO)
  public Integer getId() {
	  return id;
  }
}
```

## IdClass

> 当entity class使用复合主键时，需要定义一个类作为id class。
> id class必须符合以下要求:
> 类必须声明为public，并提供一个声明为public的空构造函数。
> 必须实现Serializable接口，覆写 equals() 和 hashCode()方法。
> entity class的所有id field在id class都要定义，且类型一样。

元数据属性说明：
- value: id class的类名

```java
public class EmployeePK implements java.io.Serializable{
   String empName;
   Integer empAge;

   public EmployeePK(){}

   public boolean equals(Object obj){ ......}
   public int hashCode(){......}
}


@IdClass(value=com.acme.EmployeePK.class)
@Entity(access=FIELD)
public class Employee {
    @Id String empName;
    @Id Integer empAge;
}
```

[https://blog.csdn.net/yswKnight/article/details/79257372](https://blog.csdn.net/yswKnight/article/details/79257372)
