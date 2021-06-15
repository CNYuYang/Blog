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