# XML方式的基本用法之多表查询

## 多表查询

上篇博客中，我们示例的2个查询都是单表查询，但实际的业务场景肯定是需要多表查询的，比如现在有个需求：

查询某个用户拥有的所有角色。这个需求要涉及到sys_user，sys_user_role，sys_role三张表，如何实现呢？

首先，在SysUserMapper接口中定义如下方法。

```java
/**
 * 根据用户id获取角色信息
 *
 * @param userId
 * @return
 */
List<SysRole> selectRolesByUserId(Long userId);
```

然后打开对应的SysUserMapper.xml文件，添加如下select语句：

```xml
<select id="selectRolesByUserId" resultType="com.zwwhnly.mybatisaction.model.SysRole">
    SELECT r.id,
           r.role_name   roleName,
           r.enabled,
           r.create_by   createBy,
           r.create_time createTime
    FROM sys_user u
    INNER JOIN sys_user_role ur ON u.id = ur.user_id
    INNER JOIN sys_role r ON ur.role_id = r.id
    WHERE u.id = #{userId}
</select>
```

细心的读者可能会发现，我们虽然使用到了多表查询，但是resultType设置的仍然是单表，即只包含角色表的信息。

如果我希望这个查询语句同时返回SysUser表的user_name字段呢，该如何设置resultType？

**方法1：** 直接在SysRole实体类中添加userName字段。

```java
private String userName;

public String getUserName() {
    return userName;
}

public void setUserName(String userName) {
    this.userName = userName;
}
```

此时resultType不用修改。

**方法2：** 新建扩展类，在扩展类中添加userName字段。

```java
package run.yuyang.mybatis.model;

public class SysRoleExtend extends SysRole {
    private String userName;

    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }
}
```

此时需要将resultType修改为：run.yuyang.mybatis.model.SysRoleExtend。

这种方式比较适合需要少量额外字段的场景。如果需要其他表的大量字段，可以使用方式3或者方式4，个人推荐使用方式4。

**方法3：** 在SysRole实体类中添加SysUser类型的字段。

```java
private SysUser sysUser;

public SysUser getSysUser() {
   return sysUser;
}

public void setSysUser(SysUser sysUser) {
    this.sysUser = sysUser;
}
```

此时resultType不用修改。

**方法4(推荐使用)：** 新建扩展类，在扩展类中添加SysUser类型的字段。

书中推荐的是方式3，方式4是我个人认为更好的方式，因为实体类一般由工具自动生成，增加了字段后，后续容易忘记导致被覆盖掉。

```java
package run.yuyang.mybatis.model;

public class SysRoleExtend extends SysRole {
    private SysUser sysUser;

    public SysUser getSysUser() {
        return sysUser;
    }

    public void setSysUser(SysUser sysUser) {
        this.sysUser = sysUser;
    }
}
```

此时需要将resultType修改为：run.yuyang.mybatis.model.SysRoleExtend。

此时xml中的查询语句如下。

```xml
<select id="selectRolesByUserId" resultType="run.yuyang.mybatis.model.SysRoleExtend">
    SELECT r.id,
           r.role_name   roleName,
           r.enabled,
           r.create_by   createBy,
           r.create_time createTime,
           u.user_name   "sysUser.userName",
           u.user_email   "sysUser.userEmail"
    FROM sys_user u
    INNER JOIN sys_user_role ur ON u.id = ur.user_id
    INNER JOIN sys_role r ON ur.role_id = r.id
    WHERE u.id = #{userId}
</select>
```

### 测试

编写测试类:

```java
public class TestChapter3 {

    private static SqlSessionFactory sqlSessionFactory;

    public static void main(String[] args) {

        try  {
            @Cleanup Reader reader = Resources.getResourceAsReader("mybatis-config.xml");
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
            @Cleanup SqlSession sqlSession = sqlSessionFactory.openSession();
            SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);
            List<SysRole> sysRoleList = sysUserMapper.selectRolesByUserId(1L);
            for (SysRole role : sysRoleList) {
                System.out.println(role.toString());
            }

        } catch (Exception e) {
            e.printStackTrace();
        }

    }

}
```

执行结果：

```
SysRole(id=1, roleName=管理员, enabled=1, createBy=1, createTime=Wed Jun 16 11:40:11 CST 2021)
SysRole(id=2, roleName=普通用户, enabled=1, createBy=1, createTime=Wed Jun 16 11:40:11 CST 2021)
```

## 多个接口参数的用法

### 参数类型是基本类型

截止目前，我们定义的方法都只有1个参数，要么是只有1个基本类型的参数，比如selectById(Long id);。

要么是只有1个对象作为参数，即将多个参数合并成了1个对象。

但有些场景下，比如只有2个参数，没有必要为这2个参数再新建一个对象，比如我们现在需要根据用户的id和角色的状态来获取用户的所有角色，那么该如何使用呢？

首先，在接口SysUserMapper中添加如下方法。

```java
/**
 * 根据用户id和角色的enabled状态获取用户的角色
 *
 * @param userId
 * @param enabled
 * @return
 */
List<SysRole> selectRolesByUserIdAndRoleEnabled(Long userId,Integer enabled);
```

然后，打开对应的SysUserMapper.xml文件，添加如下代码。

```xml
<select id="selectRolesByUserIdAndRoleEnabled" resultType="run.yuyang.mybatis.model.SysRole">
    SELECT r.id,
           r.role_name   roleName,
           r.enabled,
           r.create_by   createBy,
           r.create_time createTime
    FROM sys_user u
    INNER JOIN sys_user_role ur ON u.id = ur.user_id
    INNER JOIN sys_role r ON ur.role_id = r.id
    WHERE u.id = #{userId} AND r.enabled = #{enabled}
</select>
```