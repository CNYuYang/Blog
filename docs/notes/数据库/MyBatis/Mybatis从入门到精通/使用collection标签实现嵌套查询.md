# 使用collection标签实现嵌套查询

本篇博客主要讲解使用collection标签实现嵌套查询的方法。

## 需求升级

在上篇博客中，我们实现了需求：根据用户id查询用户信息的同时获取用户拥有的角色。

因为角色可以拥有多个权限，所以本篇博客我们升级需求为：根据用户id查询用户信息的同时获取用户拥有的角色以及角色包含的权限。

## 实现方式

因为我们需要使用到权限表的映射，所以我们需要先在SysPrivilegeMapper.xml中添加如下映射：

```xml
<resultMap id="sysPrivilegeMap" type="run.yuyang.mybatis.model.SysPrivilege">
    <id property="id" column="id"/>
    <result property="privilegeName" column="privilege_name"/>
    <result property="privilegeUrl" column="privilege_url"/>
</resultMap>
```

一般情况下不建议修改数据库表对应的实体类，所以这里我们新建类SysRoleExtend，让它继承SysRole类，并添加如下字段：

```java
package run.yuyang.mybatis.model;

import java.util.List;

public class SysRoleExtend extends SysRole {
    /**
     * 角色包含的权限列表
     */
    private List<SysPrivilege> sysPrivilegeList;

    public List<SysPrivilege> getSysPrivilegeList() {
        return sysPrivilegeList;
    }

    public void setSysPrivilegeList(List<SysPrivilege> sysPrivilegeList) {
        this.sysPrivilegeList = sysPrivilegeList;
    }
}
```

然后在SysRoleMapper.xml中新建如下映射：

```xml
<resultMap id="rolePrivilegeListMap" extends="roleMap"
           type="run.yuyang.mybatis.model.SysRoleExtend">
    <collection property="sysPrivilegeList" columnPrefix="privilege_"
                resultMap="run.yuyang.mybatis.mapper.SysPrivilegeMapper.sysPrivilegeMap"/>
</resultMap>
```

这里的roleMap我们在之前的博客中已经定义过，代码如下：

```xml
<resultMap id="roleMap" type="run.yuyang.mybatis.model.SysRole">
    <id property="id" column="id"/>
    <result property="roleName" column="role_name"/>
    <result property="enabled" column="enabled"/>
    <result property="createBy" column="create_by"/>
    <result property="createTime" column="create_time" jdbcType="TIMESTAMP"/>
</resultMap>
```

`run.yuyang.mybatis.mapper.SysPrivilegeMapper.sysPrivilegeMap`就是我们刚刚在SysPrivilegeMapper.xml中新建的映射sysPrivilegeMap。

然后，需要将上篇博客中的userRoleListMap修改为：

```xml
<resultMap id="userRoleListMap" type="run.yuyang.mybatis.model.SysUserExtend" extends="sysUserMap">
    <collection property="sysRoleList" columnPrefix="role_"
                resultMap="run.yuyang.mybatis.mapper.SysRoleMapper.rolePrivilegeListMap">
    </collection>
</resultMap>
```

并且要修改上篇博客中id为selectAllUserAndRoles的select标签代码，因为要关联角色权限关系表和权限表：

```xml
<select id="selectAllUserAndRoles" resultMap="userRoleListMap">
    SELECT  u.id,
            u.user_name,
            u.user_password,
            u.user_email,
            u.create_time,
            r.id role_id,
            r.role_name role_role_name,
            r.enabled role_enabled,
            r.create_by role_create_by,
            r.create_time role_create_time,
            p.id role_privilege_id,
            p.privilege_name role_privilege_privilege_name,
            p.privilege_url role_privilege_privilege_url
    FROM sys_user u
    INNER JOIN sys_user_role ur ON u.id = ur.user_id
    INNER JOIN sys_role r ON ur.role_id = r.id
    INNER JOIN sys_role_privilege rp ON rp.role_id = r.id
    INNER JOIN sys_privilege p ON p.id = rp.privilege_id
</select>
```

**注意事项：**

这里sys_privilege表的列名的别名前缀为`role_privilege_`，这是因为userRoleListMap中collection的columnPrefix属性为`role_`，并且指定的`run.yuyang.mybatis.mapper.SysRoleMapper.rolePrivilegeListMap`中collection的columnPrefix属性为`privilege_`，所以这里的前缀需要叠加，就变成了`role_privilege_`。

## 单元测试

修改上篇博客中建的测试方法testSelectAllUserAndRoles()代码为：

```java
@Test
public void testSelectAllUserAndRoles() {
    SqlSession sqlSession = getSqlSession();

    try {
        SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);

        List<SysUserExtend> sysUserList = sysUserMapper.selectAllUserAndRoles();
        System.out.println("用户数：" + sysUserList.size());
        for (SysUserExtend sysUser : sysUserList) {
            System.out.println("用户名：" + sysUser.getUserName());
            for (SysRoleExtend sysRoleExtend : sysUser.getSysRoleList()) {
                System.out.println("角色名：" + sysRoleExtend.getRoleName());
                for (SysPrivilege sysPrivilege : sysRoleExtend.getSysPrivilegeList()) {
                    System.out.println("权限名：" + sysPrivilege.getPrivilegeName());
                }
            }
        }
    } finally {
        sqlSession.close();
    }
}

```

运行测试代码，测试通过，输出日志如下：

> DEBUG [main] - ==>  Preparing: SELECT u.id, u.user_name, u.user_password, u.user_email, u.create_time, r.id role_id, r.role_name role_role_name, r.enabled role_enabled, r.create_by role_create_by, r.create_time role_create_time, p.id role_privilege_id, p.privilege_name role_privilege_privilege_name, p.privilege_url role_privilege_privilege_url FROM sys_user u INNER JOIN sys_user_role ur ON u.id = ur.user_id INNER JOIN sys_role r ON ur.role_id = r.id INNER JOIN sys_role_privilege rp ON rp.role_id = r.id INNER JOIN sys_privilege p ON p.id = rp.privilege_id
>
> DEBUG [main] - ==> Parameters:
>
> TRACE [main] - <==    Columns: id, user_name, user_password, user_email, create_time, role_id, role_role_name, role_enabled, role_create_by, role_create_time, role_privilege_id, role_privilege_privilege_name, role_privilege_privilege_url
>
> TRACE [main] - <==        Row: 1, admin, 123456, admin@mybatis.tk, 2019-06-27 18:21:07.0, 1, 管理员, 1, 1, 2019-06-27 18:21:12.0, 1, 用户管理, /users
>
> TRACE [main] - <==        Row: 1, admin, 123456, admin@mybatis.tk, 2019-06-27 18:21:07.0, 1, 管理员, 1, 1, 2019-06-27 18:21:12.0, 2, 角色管理, /roles
>
> TRACE [main] - <==        Row: 1, admin, 123456, admin@mybatis.tk, 2019-06-27 18:21:07.0, 1, 管理员, 1, 1, 2019-06-27 18:21:12.0, 3, 系统日志, /logs
>
> TRACE [main] - <==        Row: 1, admin, 123456, admin@mybatis.tk, 2019-06-27 18:21:07.0, 2, 普通用户, 1, 1, 2019-06-27 18:21:12.0, 4, 人员维护, /persons
>
> TRACE [main] - <==        Row: 1, admin, 123456, admin@mybatis.tk, 2019-06-27 18:21:07.0, 2, 普通用户, 1, 1, 2019-06-27 18:21:12.0, 5, 单位维护, /companies
>
> TRACE [main] - <==        Row: 1001, test, 123456, test@mybatis.tk, 2019-06-27 18:21:07.0, 2, 普通用户, 1, 1, 2019-06-27 18:21:12.0, 4, 人员维护, /persons
>
> TRACE [main] - <==        Row: 1001, test, 123456, test@mybatis.tk, 2019-06-27 18:21:07.0, 2, 普通用户, 1, 1, 2019-06-27 18:21:12.0, 5, 单位维护, /companies
>
> DEBUG [main] - <==      Total: 7
>
> 用户数：2
>
> 用户名：admin
>
> 角色名：管理员
>
> 权限名：用户管理
>
> 权限名：角色管理
>
> 权限名：系统日志
>
> 角色名：普通用户
>
> 权限名：人员维护
>
> 权限名：单位维护
>
> 用户名：test
>
> 角色名：普通用户
>
> 权限名：人员维护
>
> 权限名：单位维护

从日志可以看出，不仅查询出了用户拥有的角色信息，也查询出了角色包含的权限信息。

## 延迟加载

有的同学可能会说，返回的角色信息和权限信息我不一定用啊，每次关联这么多表查询一次数据库，好影响性能啊，能不能在我使用到角色信息即获取sysRoleList属性时再去数据库查询呢？答案当然是能，那么如何实现呢？

实现延迟加载需要使用collection标签的fetchType属性，该属性有lazy和eager两个值，分别代表延迟加载和积极加载。

由于需要根据角色Id获取该角色对应的所有权限信息，所以我们要先在SysPrivilegeMapper.xml中定义如下查询：

```xml
<select id="selectPrivilegeByRoleId" resultMap="sysPrivilegeMap">
    SELECT p.*
    FROM sys_privilege p
    INNER JOIN sys_role_privilege rp ON rp.privilege_id = p.id
    WHERE rp.role_id = #{roleId}
</select>
```

然后在SysRoleMapper.xml中添加如下查询：

```xml
<resultMap id="rolePrivilegeListMapSelect" extends="roleMap"
           type="run.yuyang.mybatis.model.SysRoleExtend">
    <collection property="sysPrivilegeList" fetchType="lazy"
                column="{roleId=id}"
                select="run.yuyang.mybatis.mapper.SysPrivilegeMapper.selectPrivilegeByRoleId"/>
</resultMap>

<select id="selectRoleByUserId" resultMap="rolePrivilegeListMapSelect">
    SELECT
          r.id,
          r.role_name,
          r.enabled,
          r.create_by,
          r.create_time
    FROM sys_role r
    INNER JOIN sys_user_role ur ON ur.role_id = r.id
    WHERE ur.user_id = #{userId}
</select>
```

上面的`column="{roleId=id}"`中，roleId指的是select指定的方法selectPrivilegeByRoleId的参数，id指的是查询selectRoleByUserId中查询出的角色id。

然后在SysUserMapper.xml中添加如下查询：

```xml
<resultMap id="userRoleListMapSelect" extends="sysUserMap"
           type="run.yuyang.mybatis.model.SysUserExtend">
    <collection property="sysRoleList" fetchType="lazy"
                select="run.yuyang.mybatis.mapper.SysRoleMapper.selectRoleByUserId"
                column="{userId=id}"/>
</resultMap>

<select id="selectAllUserAndRolesSelect" resultMap="userRoleListMapSelect">
    SELECT
          u.id,
          u.user_name,
          u.user_password,
          u.user_email,
          u.create_time
    FROM sys_user u
    WHERE u.id = #{id}
</select>
```

上面的`column="{userId=id}"`中，userId指的是select指定的方法selectRoleByUserId的参数，id指的是查询selectAllUserAndRolesSelect中查询出的用户id。

然后，在SysUserMapper接口中，添加如下方法：

```java
/**
 * 通过嵌套查询获取指定用户的信息以及用户的角色和权限信息
 *
 * @param id
 * @return
 */
SysUserExtend selectAllUserAndRolesSelect(Long id);
```

最后，在SysUserMapperTest类中添加如下测试方法：

```java
@Test
public void testSelectAllUserAndRolesSelect() {
    SqlSession sqlSession = getSqlSession();

    try {
        SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);

        SysUserExtend sysUserExtend = sysUserMapper.selectAllUserAndRolesSelect(1L);
        System.out.println("用户名：" + sysUserExtend.getUserName());
        for (SysRoleExtend sysRoleExtend : sysUserExtend.getSysRoleList()) {
            System.out.println("角色名：" + sysRoleExtend.getRoleName());
            for (SysPrivilege sysPrivilege : sysRoleExtend.getSysPrivilegeList()) {
                System.out.println("权限名：" + sysPrivilege.getPrivilegeName());
            }
        }
    } finally {
        sqlSession.close();
    }
}
```

运行测试方法，输出日志如下：

> DEBUG [main] - ==>  Preparing: SELECT u.id, u.user_name, u.user_password, u.user_email, u.create_time FROM sys_user u WHERE u.id = ?
>
> DEBUG [main] - ==> Parameters: 1(Long)
>
> TRACE [main] - <==    Columns: id, user_name, user_password, user_email, create_time
>
> TRACE [main] - <==        Row: 1, admin, 123456, admin@mybatis.tk, 2019-06-27 18:21:07.0
>
> DEBUG [main] - <==      Total: 1
>
> 用户名：admin
>
> DEBUG [main] - ==>  Preparing: SELECT r.id, r.role_name, r.enabled, r.create_by, r.create_time FROM sys_role r INNER JOIN sys_user_role ur ON ur.role_id = r.id WHERE ur.user_id = ?
>
> DEBUG [main] - ==> Parameters: 1(Long)
>
> TRACE [main] - <==    Columns: id, role_name, enabled, create_by, create_time
>
> TRACE [main] - <==        Row: 1, 管理员, 1, 1, 2019-06-27 18:21:12.0
>
> TRACE [main] - <==        Row: 2, 普通用户, 1, 1, 2019-06-27 18:21:12.0
>
> DEBUG [main] - <==      Total: 2
>
> 角色名：管理员
>
> DEBUG [main] - ==>  Preparing: SELECT p.* FROM sys_privilege p INNER JOIN sys_role_privilege rp ON rp.privilege_id = p.id WHERE rp.role_id = ?
>
> DEBUG [main] - ==> Parameters: 1(Long)
>
> TRACE [main] - <==    Columns: id, privilege_name, privilege_url
>
> TRACE [main] - <==        Row: 1, 用户管理, /users
>
> TRACE [main] - <==        Row: 2, 角色管理, /roles
>
> TRACE [main] - <==        Row: 3, 系统日志, /logs
>
> DEBUG [main] - <==      Total: 3
>
> 权限名：用户管理
>
> 权限名：角色管理
>
> 权限名：系统日志
>
> 角色名：普通用户
>
> DEBUG [main] - ==>  Preparing: SELECT p.* FROM sys_privilege p INNER JOIN sys_role_privilege rp ON rp.privilege_id = p.id WHERE rp.role_id = ?
>
> DEBUG [main] - ==> Parameters: 2(Long)
>
> TRACE [main] - <==    Columns: id, privilege_name, privilege_url
>
> TRACE [main] - <==        Row: 4, 人员维护, /persons
>
> TRACE [main] - <==        Row: 5, 单位维护, /companies
>
> DEBUG [main] - <==      Total: 2
>
> 权限名：人员维护
>
> 权限名：单位维护

仔细分析上面的日志，会发现只有在使用到了角色信息和权限信息时，才执行了对应的数据库查询。

需要注意的是，延迟加载依赖于MyBatis全局配置中的aggressiveLazyLoading，在之前的博客讲解association标签时，我们已经将其配置为了false，所以这里的执行结果符合我们的预期：

```xml
<settings>
    <!--其他配置-->
    <setting name="aggressiveLazyLoading" value="false"/>
</settings>
```

关于该参数的详细讲解，请查看[MyBatis从入门到精通(十)：使用association标签实现嵌套查询](https://www.cnblogs.com/zwwhnly/p/11175503.html)。

## 总结

使用collection标签实现嵌套查询，用到的属性总结如下：

1)select：另一个映射查询的id，MyBatis会额外执行这个查询获取嵌套对象的结果。

2)column：将主查询中列的结果作为嵌套查询的参数，配置方式如column="{prop1=col1,prop2=col2}",prop1和prop2将作为嵌套查询的参数。

3)fetchType：数据加载方式，可选值为lazy和eager，分别为延迟加载和积极加载。

4)如果要使用延迟加载，除了将fetchType设置为lazy，还需要注意全局配置aggressiveLazyLoading的值应该为false。这个参数在3.4.5版本之前默认值为ture，从3.4.5版本开始默认值改为false。

5)MyBatis提供的lazyLoadTriggerMethods参数，支持在触发某方法时直接触发延迟加载属性的查询，如equals()方法。

