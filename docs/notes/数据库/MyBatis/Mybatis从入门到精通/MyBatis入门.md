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

## 环境准备

执行下列sql语句，创建对应数据库及插入数据:

```sql
CREATE TABLE country
(
  id          INT          NOT NULL AUTO_INCREMENT,
  countryname VARCHAR(255) NULL,
  countrycode VARCHAR(255) NULL,
  PRIMARY KEY (id)
);

INSERT country(countryname, countrycode)
VALUES ('中国', 'CN'),
       ('美国', 'US'),
       ('俄罗斯', 'RU'),
       ('英国', 'GB'),
       ('法国', 'FR');
```

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
        <package name="run.yuyang.mybatis.model"/>
    </typeAliases>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC">
                <property name="" value=""/>
            </transactionManager>
            <dataSource type="UNPOOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/mybatis"/>
                <property name="username" value="root"/>
                <property name="password" value="2020possible"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="mapper/CountryMapper.xml"/>
    </mappers>
</configuration>
```

配置简单讲解：

- 元素下面配置了一个包名，通常确定一个类的时候需要使用类的全限定名称，例如run.yuyang.mybatis.model.Country。在MyBatis中需要频繁用到类的全限定名称，为了方便使用，我们配置了run.yuyang.mybatis.model包，这样配置后，在使用类的时候不需要写包名的部分，只使用Country即可。
  - 环境配置中主要配置了数据库连接，如这里我们使用的是本地的mybatis数据库，用户名为root，密码为：2020possible（大家可根据自己的实际情况修改数据库及用户名和密码）。
- 中配置了一个包含完整类路径的CountryMapper.xml，这是一个MyBatis的Sql语句和映射配置文件。

### POJO

在src/main/java下新建包：run.yuyang.mybatis，然后在这个包下再创建包：model。

在model包下创建数据库表country表对应的实体类Country：

```java
package run.yuyang.mybatis.model;


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

进行测试

1. 读取配置文件
2. 创建 SqlSessionFactory工厂 
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
> 
> 2     美国  US
> 
> 3    俄罗斯  RU
> 
> 4     英国  GB
> 
> 5     法国  FR
