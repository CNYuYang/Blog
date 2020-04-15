
# MyBatis 入门

## MyBatis简介

MyBatis 本是Apache的一个开源项目iBatis, 2010年这个项目由Apache Software Foundation 迁移到了Google Code，并且改名为MyBatis，实质上Mybatis对ibatis进行一些改进。

MyBatis是一个优秀的持久层框架，它对JDBC的操作数据库的过程进行封装，使开发者只需要关注 SQL 本身，而不需要花费精力去处理例如注册驱动、创建Connection、创建Statement、手动设置参数、结果集检索等JDBC繁杂的过程代码。

MyBatis通过xml或注解的方式将要执行的各种Statement（Statement、PreparedStatement、CallableStatement）配置起来，并通过Java对象和Statement中的SQL进行映射生成最终执行的SQL语句，最后由MyBatis框架执行SQL并将结果映射成Java对象并返回。

## 分析原生态JDBC程序中存在的问题

### 原生态JDBC程序代码

```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
public class Test
{
    public static void main(String[] args)
    {
        Connection connection = null;
        PreparedStatement preparedStatement = null;
        ResultSet resultSet = null;

        try
        {
            //1、加载数据库驱动
            Class.forName("com.mysql.jdbc.Driver");

            //2、通过驱动管理类获取数据库链接
            connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/mybatis?characterEncoding=utf-8",
                    "root", "root");

            //3、定义sql语句 ?表示占位符
            String sql = "select * from user where username = ?";

            //4、获取预处理statement
            preparedStatement = connection.prepareStatement(sql);

            //5、设置参数，第一个参数为sql语句中参数的序号（从1开始），第二个参数为设置的参数值
            preparedStatement.setString(1, "admin");

            //6、向数据库发出sql执行查询，查询出结果集
            resultSet = preparedStatement.executeQuery();

            //7、遍历查询结果集
            while (resultSet.next())
            {
                System.out.println(resultSet.getString("id") + "  " + resultSet.getString("username"));
            }
        }
        catch (Exception e)
        {
            e.printStackTrace();
        }
        finally
        {
            //8、释放资源
            if (resultSet != null)
            {
                try
                {
                    resultSet.close();
                }
                catch (SQLException e)
                {

                    e.printStackTrace();
                }
            }
            if (preparedStatement != null)
            {
                try
                {
                    preparedStatement.close();
                }
                catch (SQLException e)
                {
                    e.printStackTrace();
                }
            }
            if (connection != null)
            {
                try
                {
                    connection.close();
                }
                catch (SQLException e)
                {
                    e.printStackTrace();
                }
            }

        }

    }

}
```

### JDBC问题总结

- 数据库连接频繁开启和关闭，会严重影响数据库的性能。

- 代码中存在硬编码，分别是数据库部分的硬编码和SQL执行部分的硬编码。

## 利用Meavn创建工程

### 添加pom依赖如下

```xml
<dependencies>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.8</version>
    </dependency>
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>3.5.2</version>
    </dependency>
    <!-- MyBatis-->
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>5.1.38</version>
    </dependency>
</dependencies>
```

### mybatis-config

首先在src/main/resources目录下创建mybatis-config.xml配置文件:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

    <typeAliases>
        <package name="cn.reyunn.mybatis.model"/>
    </typeAliases>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC">
                <property name="" value=""/>
            </transactionManager>
            <dataSource type="UNPOOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://cdb-bgss2kq6.bj.tencentcdb.com:10004/mybatis_test"/>
                <property name="username" value="root"/>
                <property name="password" value="whut2017"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="mapper/CountryMapper.xml"/>
    </mappers>
</configuration>

```

配置简单讲解：

- 元素下面配置了一个包名，通常确定一个类的时候需要使用类的全限定名称，例如cn.reyunn.mybatis.model.Country。在MyBatis中需要频繁用到类的全限定名称，为了方便使用，我们配置了cn.reyunn.mybatis.model.model包，这样配置后，在使用类的时候不需要写包名的部分，只使用Country即可。
- 环境配置中主要配置了数据库连接，如这里我们使用的是腾讯云的mybatis_test数据库，用户名为root，密码为：whut2017（大家可根据自己的实际情况修改数据库及用户名和密码）。
- 中配置了一个包含完整类路径的CountryMapper.xml，这是一个MyBatis的Sql语句和映射配置文件。

### POJO

在src/main/java下新建包：cn.reyunn.mybatis，然后在这个包下再创建包：model。

在model包下创建数据库表country表对应的实体类Country：

```java
package cn.reyunn.mybatis.model;


import lombok.Data;

@Data
public class Country {

    private Integer id;

    private String countryname;

    private String countrycode;

}
```

### CountryMapper

在src/main/resources下创建目录mapper目录，然后在该目录下创建CountryMapper.xml文件。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="CountryMapper">
    <select id="selectAll" resultType="Country">
        SELECT id,countryname,countrycode from country
    </select>
</mapper>
```

## 使用

对刚才的付出进行检测😁

1. 读取配置文件
2.  创建 SqlSessionFactory工厂 
3. 使用工厂生产SqlSession对象
4. 使用SqlSession创建Dao接口的代理对象
5. 使用代理对象执行方法
6. 释放资源

```java
public class Main {

    private static SqlSessionFactory sqlSessionFactory;
    
    public static void main(String[] args) {

        try  {
            @Cleanup Reader reader = Resources.getResourceAsReader("mybatis-config.xml");
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
            @Cleanup SqlSession sqlSession = sqlSessionFactory.openSession();
            List<Country> countryList = sqlSession.selectList("selectAll");
            printCountryList(countryList);
        } catch (Exception e) {
            e.printStackTrace();
        }

    }

    private static void printCountryList(List<Country> countryList) {
        for (Country country : countryList) {
            System.out.printf("%-4d%4s%4s\n", country.getId(), country.getCountryname(), country.getCountrycode());
        }
    }

}
```

运行代码，输出如下：

> 1     中国  CN
> 2     美国  US
> 3    俄罗斯  RU
> 4     英国  GB
> 5     法国  FR

# XML方式的基本用法之Select

## 明确需求

书中提到的需求是一个基于角色的权限控制需求（RBAC，即Role-Based Access Control），提到权限管理，相信大家都不陌生，因为大部分的系统都是需要权限管理的，大致描述如下：

- 权限点用来管理要控制权限的资源，比如某个页面，某个按钮。

- 创建一个角色，给这个角色分配某些权限点，比如商品模块的所有页面的权限。

- 新建一个用户，给这个用户分配某些角色。

##  数据准备

首先执行如下脚本创建上图中的5张表：用户表，角色表，权限表，用户角色关联表，角色权限关联表。

```mysql
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

```mysql
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

在包cn.reyunn.mybatis.model下依次创建这5张表对应的实体类：

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

> 因为Java中的基本类型会有默认值，例如当某个类中存在private int age;字段时，age的默认值为0，所以无法满足age为null的情况，如果使用age !=null，结果总为ture，会导致一些隐藏的bug。

## 创建Mapper.xml文件

在src/main/resources下的mapper目录下依次创建5张表对应的Mapper.xml文件，在这里就不再赘述了。

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

报错的原因是上节中，我们并没有为CountryMapper.xml文件创建对应的接口，使用包名配置方式后，就需要创建，所以解决方案就是在src/main/java下新建包cn.reyunn.mybatis.mapper下，然后在该包下新建接口CountryMapper，然后在接口中添加方法selectAll()。