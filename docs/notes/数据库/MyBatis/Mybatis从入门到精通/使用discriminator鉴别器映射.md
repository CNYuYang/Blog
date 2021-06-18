# 使用discriminator鉴别器映射

本篇博客主要讲解鉴别器映射discriminator标签的简单用法。

## 明确需求

在设计之初，sys_role表的enabled字段有2个可选值，其中1代表启用，0 代表禁用，当状态启用时就有对应的权限信息，当状态禁用时就没有对应的权限信息，只需查询出角色信息即可。

所以我们的需求为：根据用户id查询用户拥有的角色列表，如果角色是启用的，就继续查询出角色对应的权限列表，如果角色是禁用的，就不需要查询对应的权限列表。

## 实现方式

首先，我们需要在SysRoleMapper.xml中新建角色表的映射roleMapExtend，注意这里我们使用的是之前新建的扩展类SysRoleExtend：

```xml
<resultMap id="roleMapExtend" type="run.yuyang.mybatis.model.SysRoleExtend">
    <id property="id" column="id"/>
    <result property="roleName" column="role_name"/>
    <result property="enabled" column="enabled"/>
    <result property="createBy" column="create_by"/>
    <result property="createTime" column="create_time" jdbcType="TIMESTAMP"/>
</resultMap>
```

然后回顾下上篇博客中使用到的rolePrivilegeListMapSelect映射，因为接下来会用到：

```xml
<resultMap id="rolePrivilegeListMapSelect" extends="roleMap"
           type="run.yuyang.mybatis.model.SysRoleExtend">
    <collection property="sysPrivilegeList" fetchType="lazy"
                column="{roleId=id}"
                select="run.yuyang.mybatis.mapper.SysPrivilegeMapper.selectPrivilegeByRoleId"/>
</resultMap>
```

接下来新建本篇博客的主人公映射rolePrivilegeListMapChoose和对应的查询语句：

```xml
<resultMap id="rolePrivilegeListMapChoose"
           type="run.yuyang.mybatis.model.SysRoleExtend">
    <discriminator column="enabled" javaType="int">
        <case value="1" resultMap="rolePrivilegeListMapSelect"/>
        <case value="0" resultMap="roleMapExtend"/>
    </discriminator>
</resultMap>

<select id="selectRoleByUserIdChoose" resultMap="rolePrivilegeListMapChoose">
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

discriminator标签常用的2个属性讲解：

- column：设置要进行鉴别比较值的列名。
- javaType：指定列的类型，保证使用相同的Java类型来比较值。

discriminator标签可以有1个或多个case标签，case标签包含以下3个属性：

- value：该值为discriminator标签column属性用来匹配的值。
- resultMap：当column的值和value的值匹配时，可以配置使用resultMap指定的映射，resultMap优先级高于resultType。
- resultType：当column的值和value的值匹配时，用于配置使用resultType指定的映射。

case标签下面可以包含的标签和resultMap一样，用法也一样。

然后在SysRoleMapper接口中添加如下方法：

```java
/**
 * 根据用户id获取用户的角色信息
 *
 * @param userId
 * @return
 */
List<SysRoleExtend> selectRoleByUserIdChoose(Long userId);
```

## 单元测试

在SysRoleMapperTest测试类中添加如下测试方法：

```java
@Test
public void testSelectRoleByUserIdChoose() {
    SqlSession sqlSession = getSqlSession();

    try {
        SysRoleMapper sysRoleMapper = sqlSession.getMapper(SysRoleMapper.class);

        // 将id=2的角色的enabled赋值为0
        SysRole sysRole = sysRoleMapper.selectById(2L);
        sysRole.setEnabled(0);
        sysRoleMapper.updateById(sysRole);

        // 获取用户id为1的角色
        List<SysRoleExtend> sysRoleExtendList = sysRoleMapper.selectRoleByUserIdChoose(1L);
        for (SysRoleExtend item : sysRoleExtendList) {
            System.out.println("角色名：" + item.getRoleName());
            if (item.getId().equals(1L)) {
                // 第一个角色存在权限信息
                Assert.assertNotNull(item.getSysPrivilegeList());
            } else if (item.getId().equals(2L)) {
                // 第二个角色的权限为null
                Assert.assertNull(item.getSysPrivilegeList());
                continue;
            }
            for (SysPrivilege sysPrivilege : item.getSysPrivilegeList()) {
                System.out.println("权限名：" + sysPrivilege.getPrivilegeName());
            }
        }
    } finally {
        sqlSession.close();
    }
}

```

运行测试代码，测试通过，输出日志如下：

> DEBUG [main] - ==>  Preparing: SELECT id,role_name,enabled,create_by,create_time FROM sys_role WHERE id = ?
>
> DEBUG [main] - ==> Parameters: 2(Long)
>
> TRACE [main] - <==    Columns: id, role_name, enabled, create_by, create_time
>
> TRACE [main] - <==        Row: 2, 普通用户, 1, 1, 2019-06-27 18:21:12.0
>
> DEBUG [main] - <==      Total: 1
>
> DEBUG [main] - ==>  Preparing: UPDATE sys_role SET role_name = ?,enabled = ?,create_by=?, create_time=? WHERE id=?
>
> DEBUG [main] - ==> Parameters: 普通用户(String), 0(Integer), 1(Long), 2019-06-27 18:21:12.0(Timestamp), 2(Long)
>
> DEBUG [main] - <==    Updates: 1
>
> DEBUG [main] - ==>  Preparing: SELECT r.id, r.role_name, r.enabled, r.create_by, r.create_time FROM sys_role r INNER JOIN sys_user_role ur ON ur.role_id = r.id WHERE ur.user_id = ?
>
> DEBUG [main] - ==> Parameters: 1(Long)
>
> TRACE [main] - <==    Columns: id, role_name, enabled, create_by, create_time
>
> TRACE [main] - <==        Row: 1, 管理员, 1, 1, 2019-06-27 18:21:12.0
>
> TRACE [main] - <==        Row: 2, 普通用户, 0, 1, 2019-06-27 18:21:12.0
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

从日志可以看出，角色1是启用的，所以又执行了一次查询获取角色的权限列表，角色2是禁用的，所以没有执行。
