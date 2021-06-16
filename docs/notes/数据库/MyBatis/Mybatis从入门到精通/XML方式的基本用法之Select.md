# XML方式的基本用法之Select

## 明确需求

书中提到的需求是一个基于角色的权限控制需求（RBAC，即Role-Based Access Control），提到权限管理，相信大家都不陌生，因为大部分的系统都是需要权限管理的，大致描述如下：

- 权限点用来管理要控制权限的资源，比如某个页面，某个按钮。

- 创建一个角色，给这个角色分配某些权限点，比如商品模块的所有页面的权限。

- 新建一个用户，给这个用户分配某些角色。

##  数据准备

首先执行如下脚本创建上图中的5张表：用户表，角色表，权限表，用户角色关联表，角色权限关联表。

```sql
CREATE TABLE sys_user
(
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '用户ID',
  user_name VARCHAR(50) COMMENT '用户名',
  user_password VARCHAR(50) COMMENT '密码',
  user_email VARCHAR(50) COMMENT '邮箱',
  user_info TEXT COMMENT '简介',
  head_img BLOB COMMENT '头像',
  create_time DATETIME COMMENT '创建时间',
  PRIMARY KEY (id)
);
ALTER TABLE sys_user COMMENT '用户表';

CREATE TABLE sys_role
(
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '角色ID',
  role_name VARCHAR(50) COMMENT '角色名',
  enabled INT COMMENT '有效标志',
  create_by BIGINT COMMENT '创建人',
  create_time DATETIME COMMENT '创建时间',
  PRIMARY KEY (id)
);
ALTER TABLE sys_role COMMENT '角色表';

CREATE TABLE sys_privilege
(
  id BIGINT NOT NULL AUTO_INCREMENT COMMENT '权限ID',
  privilege_name VARCHAR(50) COMMENT '权限名称',
  privilege_url VARCHAR(200) COMMENT '权限URL',
  PRIMARY KEY (id)
);
ALTER TABLE sys_privilege COMMENT '权限表';

CREATE TABLE sys_user_role
(
  user_id BIGINT COMMENT '用户ID',
  role_id BIGINT COMMENT '角色ID'
);
ALTER TABLE sys_user_role COMMENT '用户角色关联表';

CREATE TABLE sys_role_privilege
(
  role_id BIGINT COMMENT '角色ID',
  privilege_id BIGINT COMMENT '权限ID'
);
ALTER TABLE sys_role_privilege COMMENT '角色权限关联表';
```

然后执行如下脚本添加测试数据：

```sql
INSERT INTO sys_user VALUES (1,'admin','123456','admin@mybatis.tk','管理员',NULL,current_timestamp);
INSERT INTO sys_user VALUES (1001,'test','123456','test@mybatis.tk','测试用户',NULL,current_timestamp);

INSERT INTO sys_role VALUES (1,'管理员',1,1,current_timestamp);
INSERT INTO sys_role VALUES (2,'普通用户',1,1,current_timestamp);

INSERT INTO sys_user_role VALUES (1,1);
INSERT INTO sys_user_role VALUES (1,2);
INSERT INTO sys_user_role VALUES (1001,2);

INSERT INTO sys_privilege VALUES (1,'用户管理','/users');
INSERT INTO sys_privilege VALUES (2,'角色管理','/roles');
INSERT INTO sys_privilege VALUES (3,'系统日志','/logs');
INSERT INTO sys_privilege VALUES (4,'人员维护','/persons');
INSERT INTO sys_privilege VALUES (5,'单位维护','/companies');

INSERT INTO sys_role_privilege VALUES (1,1);
INSERT INTO sys_role_privilege VALUES (1,2);
INSERT INTO sys_role_privilege VALUES (1,3);
INSERT INTO sys_role_privilege VALUES (2,4);
INSERT INTO sys_role_privilege VALUES (2,5);
```

##  创建实体类

在包run.yuyang.mybatis.model下依次创建这5张表对应的实体类：

### 用户表

```java
/*
 * 用户表
 */

@Data
public class SysUser {
    /**
     * 用户ID
     */
    private Long id;

    /**
     * 用户名
     */
    private String userName;

    /**
     * 密码
     */
    private String userPassword;

    /**
     * 邮箱
     */
    private String userEmail;

    /**
     * 简介
     */
    private String userInfo;

    /**
     * 头像
     */
    private byte[] headImg;

    /**
     * 创建时间
     */
    private Date createTime;

}
```

### 角色表

```java
/**
 * 角色表
 */
@Data
public class SysRole {
    /**
     * 角色ID
     */
    private Long id;

    /**
     * 角色名
     */
    private String roleName;

    /**
     * 有效标志
     */
    private Integer enabled;

    /**
     * 创建人
     */
    private Long createBy;

    /**
     * 创建时间
     */
    private Date createTime;

}
```

### 权限表

```java
/**
 * 权限表
 */
@Data
public class SysPrivilege {
    /**
     * 权限ID
     */
    private Long id;

    /**
     * 权限名称
     */
    private String privilegeName;

    /**
     * 权限URL
     */
    private String privilegeUrl;

}
```

### 用户角色关联表

```java
/**
 * 用户角色关联表
 */
@Data
public class SysUserRole {

    /**
     * 用户ID
     */
    private Long userId;

    /**
     * 角色ID
     */
    private Long roleId;

}
```

### 角色权限关联表

```java
/**
 * 角色权限关联表
 */
@Data
public class SysRolePrivilege {
    /**
     * 角色ID
     */
    private Long roleId;

    /**
     * 权限ID
     */
    private Long privilegeId;
}
```

### 注意事项：

**MyBatis默认遵循“下划线转驼峰”命名方式。**

> 如sys_user表对应的实体类名是SysUser，数据库字段user_name对应的实体类字段是userName。

**在实体类中不要使用Java的基本类型，基本类型包括byte、int、short、long、float、doubule、char、boolean。**

> 因为Java中的基本类型会有默认值，例如当某个类中存在private int age;字段时，age的默认值为0，所以无法满足age为null的情况，如果使用age != null，结果总为ture，会导致一些隐藏的bug。

## 创建Mapper.xml文件

在src/main/resources下的mapper目录下依次创建5张表对应的Mapper.xml文件。

为了后续更快速的创建Mapper.xml文件，我们可以添加模版：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper>
</mapper>
```

创建完成后，打开我们在上篇博客中创建的mybatis-config.xml文件，修改节点的内容为：

```xml
<mappers>
    <mapper resource="mapper/CountryMapper.xml"/>
    <mapper resource="mapper/SysUserMapper.xml"/>
    <mapper resource="mapper/SysRoleMapper.xml"/>
    <mapper resource="mapper/SysPrivilegeMapper.xml"/>
    <mapper resource="mapper/SysUserRoleMapper.xml"/>
    <mapper resource="mapper/SysRolePrivilegeMapper.xml"/>
</mappers>
```

使用这种方式，最明显的缺点就是，我们后续如果新增了Mapper.xml文件，仍然需要来修改文件，非常不好维护，因此我们修改成如下配置方式，配置一个包名：

```xml
<mappers>
        <package name="mapper"/>
</mappers>
```

修改完成后，运行上节的代码发现执行报如下错误：

> Caused by: java.lang.IllegalArgumentException: Mapped Statements collection does not contain value for selectAll

报错的原因是上节中，我们并没有为CountryMapper.xml文件创建对应的接口，**使用包名配置方式后，就需要创建**，所以解决方案就是在src/main/java下新建包mapper下，然后在该包下新建接口CountryMapper，然后在接口中添加方法selectAll()。

```java
import java.util.List;

public interface CountryMapper {
    /**
     * 查询全部国家
     *
     * @return
     */
    List<Country> selectAll();
}
```

## 创建Mapper接口

找到src/main/java目录下的包mapper，在该包下创建XML文件对应的接口类，分别为SysUserMapper.java，SysRoleMapper.java，SysPrivilegeMapper.java，SysUserRoleMapper.java，SysRolePrivilegeMapper.java。

这里只展示下SysUserMapper.java的代码：

```java
package mapper;

/**
 * @auther YuYang
 */
public interface SysUserMapper {
}
```

**注意事项：当Mapper接口和XML文件关联的时候，命名空间namespace的值需要配置成接口的全限定名称，MyBatis内部就是通过这个值将接口和XML关联起来的。**

> 例如SysUserMapper.xml中配置的namespace就修改为mapper.SysUserMapper

## select用法

### 查询单条数据

假设我们需要通过id查询用户的信息，首先，我们需要打开SysUserMapper.java接口定义方法：

```java
/**
 * 通过id查询用户
 *
 * @param id
 * @return
 */
SysUser selectById(Long id);
```

然后打开对应的SysUserMapper.xml文件添加如下内容：

```xml
<resultMap id="sysUserMap" type="run.yuyang.mybatis.model.SysUser">
    <id property="id" column="id"/>
    <result property="userName" column="user_name"/>
    <result property="userPassword" column="user_password"/>
    <result property="userEmail" column="user_email"/>
    <result property="userInfo" column="user_info"/>
    <result property="headImg" column="head_img" jdbcType="BLOB"/>
    <result property="createTime" column="create_time" jdbcType="TIMESTAMP"/>
</resultMap>

<select id="selectById" resultMap="sysUserMap">
    SELECT * FROM sys_user WHERE id = #{id}
</select>
```

**说明：**

1. MyBatis通过select标签的id属性值和接口的名称进行关联。
2. 标签的id属性值不能出现英文句号"."。
3. 标签的id属性值在同一个命名空间下不能重复。
4. 因为接口方法是可以重载的，所以接口中可以出现多个同名但参数不同的方法，但是XML中id的值不能重复，因此接口中的所有同名方法会对应着XML中的同一个id的方法。

XML 代码讲解：

- select：映射查询语句使用的标签。
- id：查询语句的唯一标识符，可用来代表这条语句。
- resultMap：用于设置数据库返回列和Java对象的映射关系。
- SELECT * FROM sys_user WHERE id = #{id}是查询语句。
- {id}：MyBatis SQL中使用预编译参数的一种方式，大括号中的id代表传入的参数名。

resultMap标签用于配置Java对象的属性和查询结果列的对应关系，通过resultMap中配置的column和property可以将查询列的值映射到type对象的属性上。

上面查询语句用到的resultMap标签讲解：

- id：必填且唯一。select标签resultMap属性的值为此处id设置的值。
- type：必填。用于配置查询列所映射到的Java对象模型。
- column：从数据库中得到的列名或者列的别名。
- property：要映射到的列结果的属性，即Java对象模型的属性。
- jdbcType：列对应的数据库类型。

### 查询多条数据

假设我们需要查询所有用户的信息，首先，我们需要打开SysUserMapper.java接口定义方法：

```java
/**
 * 查询全部用户
 *
 * @return
 */
List<SysUser> selectAll();
```

然后打开对应的SysUserMapper.xml文件添加如下内容：

```xml
<select id="selectAll" resultType="run.yuyang.mybatis.model.SysUser">
    SELECT id,
           user_name     userName,
           user_password userPassword,
           user_email    userEmail,
           user_info     userInfo,
           head_img      headImg,
           create_time   createTime
    FROM sys_user
</select>
```

> 注意事项：这里我们并没有使用resultMap属性来设置要返回结果的类型，而是通过resultType属性直接指定
>
> 要返回结果的类型，使用此种方式需要设置查询列的别名，别名要和resultType指定对象的属性名保持一致，
>
> 进而实现自动映射。

MyBatis提供了一个全局属性mapUnderscoreToCamelCase，将这个属性的值设置为ture可以自动将以下划线命名的数据库列映射到Java对象的驼峰式命名属性中。
那么如何打开呢？
方法是打开我们在上篇博客中新建的mybatis-config文件，在settings节点添加如下配置：

```xml
<settings>
    <!--其他配置-->
    <setting name="mapUnderscoreToCamelCase" value="true"/>
</settings>
```

此时，前面的selectAll语句可以简化为如下。

```xml
<select id="selectAll" resultType="run.yuyang.mybatis.model.SysUser">
    SELECT id,
           user_name,
           user_password,
           user_email,
           user_info,
           head_img,
           create_time
    FROM sys_user
</select>
```

## 测试

将上篇博客中该段代码：

```java
List<Country> countryList = sqlSession.selectList("selectAll");
```

修改为如下：

```java
List<Country> countryList = sqlSession.selectList("mapper.CountryMapper.selectAll");
```

因为在SysUserMapper中也添加了一个selectAll()方法，selectAll不再唯一，因此调用时必须带上namespace。

新建SysUser的测试类，代码如下。

```java
public class TestChapter2 {

    private static SqlSessionFactory sqlSessionFactory;

    public static void main(String[] args) {

        try  {
            @Cleanup Reader reader = Resources.getResourceAsReader("mybatis-config.xml");
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
            @Cleanup SqlSession sqlSession = sqlSessionFactory.openSession();
            SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);
            SysUser sysUser = sysUserMapper.selectById(1L);
            System.out.println(null != sysUser);
            System.out.println(sysUser.getUserName());

            List<SysUser> sysUserList = sysUserMapper.selectAll();
            System.out.println(sysUserList.size() > 0);

        } catch (Exception e) {
            e.printStackTrace();
        }

    }

}
```

输出为:

```
true
admin
true
```