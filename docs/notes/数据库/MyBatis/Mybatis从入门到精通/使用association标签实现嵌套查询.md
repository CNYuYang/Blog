# 使用association标签实现嵌套查询

本篇博客主要讲解使用association标签实现嵌套查询的方法。

## 明确需求

仍然延用上篇博客中的需求：根据用户id查询用户信息的同时获取该用户的角色信息(假设一个员工只能拥有一个角色)。

在上篇博客中，我们分别使用了3种方式来实现这个需求，但这3个需求都有一个共同点，就是我们使用了多表查询，即查询一次数据库就获取到我们想要的所有数据。

有的同学就说了，我不喜欢多表查询的方式，数据量大的时候会影响性能，这么简单的需求，我可以拆分成两步啊，第一步，先根据用户id查询出用户的信息和用户的角色id（仍然要关联表，只是由3张表关联减为了2张表关联），第二步，根据第一步查询出的角色id再去查询角色信息。

这种方式当然可以，而且使用业务代码就能实现这个逻辑，不过本篇博客我们不讲这种方式，而是通过association标签来实现。

## 实现方式

因为我们需要根据角色id查询角色的信息，所以我们需要先在SysRoleMapper.xml中添加如下查询：

```xml
<select id="selectRoleById" resultMap="roleMap">
    SELECT * FROM sys_role WHERE id = #{id}
</select>
```

这里的roleMap就是我们在上篇博客中定义的，代码如下：

```xml
<resultMap id="roleMap" type="run.yuyang.mybatis.model.SysRole">
    <id property="id" column="id"/>
    <result property="roleName" column="role_name"/>
    <result property="enabled" column="enabled"/>
    <result property="createBy" column="create_by"/>
    <result property="createTime" column="create_time" jdbcType="TIMESTAMP"/>
</resultMap>
```

然后，在接口SysUserMapper中添加如下方法：

```java
/**
 * 根据用户id获取用户信息和用户的角色信息，嵌套查询方式
 *
 * @param id
 * @return
 */
SysUserExtend selectUserAndRoleByIdSelect(Long id);
```

接着，在对应的SysUserMapper.xml中添加如下代码：

```xml
<resultMap id="userRoleMapSelect" type="run.yuyang.mybatis.model.SysUserExtend" extends="sysUserMap">
    <association property="sysRole"
                 select="run.yuyang.mybatis.mapper.SysRoleMapper.selectRoleById"
                 column="{id=role_id}"/>
</resultMap>

<select id="selectUserAndRoleByIdSelect" resultMap="userRoleMapSelect">
    SELECT  u.id,
            u.user_name,
            u.user_password,
            u.user_email,
            u.user_info,
            u.head_img,
            u.create_time,
            ur.role_id
    FROM sys_user u
    INNER JOIN sys_user_role ur ON u.id = ur.user_id
    WHERE u.id = #{id}
</select>
```

可以发现，我们给association标签添加了`select="run.yuyang.mybatis.mapper.SysRoleMapper.selectRoleById"`，引用的就是我们在SysRoleMapper.xml中定义的查询。

还添加了`column="{id=role_id}"`，这里的id就是run.yuyang.mybatis.mapper.SysRoleMapper.selectRoleById需要的参数名称，role_id是参数值，名称要和上面的查询中的最后一列保持一致。

**注意事项：如果是多个参数的话，可以使用`column="{id=role_id,name=role_name}"`这样的格式。**

## 单元测试

在SysUserMapperTest类中添加测试方法如下：

```java
@Test
public void testSelectUserAndRoleByIdSelect() {
    SqlSession sqlSession = getSqlSession();

    try {
        SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);

        SysUserExtend sysUserExtend = sysUserMapper.selectUserAndRoleByIdSelect(1001L);
        Assert.assertNotNull(sysUserExtend);

        Assert.assertNotNull(sysUserExtend.getSysRole());
    } finally {
        sqlSession.close();
    }
}
```

从日志可以看出，分别执行了2次查询，查询了两次数据库。

## 延迟加载

有的同学可能会说，返回的角色信息我不一定用啊，每次都查询一次数据库，好浪费性能啊，能不能在我使用到角色信息即获取sysRole属性时再去查询数据呢？答案当然是能，那么如何实现呢？

实现延迟加载需要使用association标签的fetchType属性，该属性有lazy和eager两个值，分别代表延迟加载和积极加载。

所以上面的配置就要修改成：

```xml
<association property="sysRole"
             fetchType="lazy"
             select="run.yuyang.mybatis.mapper.SysRoleMapper.selectRoleById"
             column="{id=role_id}"/>
```

为了能看到效果，我们在测试方法中添加一行输出语句：

```java
System.out.println("调用sysUserExtend.getSysRole()");
Assert.assertNotNull(sysUserExtend.getSysRole());
```

再次运行测试方法，发现输出日志和预期的不一样，在获取sysRole属性前还是查询了2次数据库，这是为什么呢？

这是因为MyBatis的全局配置中，有一个aggressiveLazyLoading参数，如果这个参数的值为ture，会使带有延迟加载属性的对象完整加载，如果为false，则会按需加载，这个参数默认值为ture（3.4.5版本开始默认值改为false）,而截止目前，我们使用的版本为3.3.1。

```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.3.1</version>
</dependency>
```

所以我们要在mybatis-config.xml中添加如下配置：

```xml
<settings>
    <!--其他配置-->
    <setting name="aggressiveLazyLoading" value="false"/>
</settings>
```

再次运行测试方法，发现输出日志和预期的一样了：

> DEBUG [main] - ==>  Preparing: SELECT u.id, u.user_name, u.user_password, u.user_email, u.create_time, ur.role_id FROM sys_user u INNER JOIN sys_user_role ur ON u.id = ur.user_id WHERE u.id = ?
>
> DEBUG [main] - ==> Parameters: 1001(Long)
>
> TRACE [main] - <==    Columns: id, user_name, user_password, user_email, create_time, role_id
>
> TRACE [main] - <==        Row: 1001, test, 123456, test@mybatis.tk, 2019-06-27 18:21:07.0, 2
>
> DEBUG [main] - <==      Total: 1
>
> **调用sysUserExtend.getSysRole()**
>
> DEBUG [main] - ==>  Preparing: SELECT * FROM sys_role WHERE id = ?
>
> DEBUG [main] - ==> Parameters: 2(Long)
>
> TRACE [main] - <==    Columns: id, role_name, enabled, create_by, create_time
>
> TRACE [main] - <==        Row: 2, 普通用户, 1, 1, 2019-06-27 18:21:12.0
>
> DEBUG [main] - <==      Total: 1

有的同学可能又会说，你现在把全局的aggressiveLazyLoading改为了false，我能不能在触发某个方法时将所有的数据都加载进来呢？答案当然是能(不然怎么往下写，哈哈)，那么如何实现呢？

**MyBatis提供了参数lazyLoadTriggerMethods，这个参数的含义是，当调用配置中的方法时，加载全部的延迟加载数据，默认值为“equals,clone,hashCode,toString”。**

简单修改下测试方法的代码：

```java
System.out.println("调用sysUserExtend.equals(null)");
sysUserExtend.equals(null);

System.out.println("调用sysUserExtend.getSysRole()");
Assert.assertNotNull(sysUserExtend.getSysRole());
```

再次运行测试方法，输出的部分日志如下：

> 调用sysUserExtend.equals(null)
>
> DEBUG [main] - ==>  Preparing: SELECT * FROM sys_role WHERE id = ?
>
> DEBUG [main] - ==> Parameters: 2(Long)
>
> TRACE [main] - <==    Columns: id, role_name, enabled, create_by, create_time
>
> TRACE [main] - <==        Row: 2, 普通用户, 1, 1, 2019-06-27 18:21:12.0
>
> DEBUG [main] - <==      Total: 1
>
> 调用sysUserExtend.getSysRole()

从日志可以看出，调用equals方法后，就触发了延迟加载属性的查询。

## 总结

使用association标签实现嵌套查询，用到的属性总结如下：

1)select：另一个映射查询的id，MyBatis会额外执行这个查询获取嵌套对象的结果。

2)column：将主查询中列的结果作为嵌套查询的参数，配置方式如column="{prop1=col1,prop2=col2}",prop1和prop2将作为嵌套查询的参数。

3)fetchType：数据加载方式，可选值为lazy和eager，分别为延迟加载和积极加载。

4)如果要使用延迟加载，除了将fetchType设置为lazy，还需要注意全局配置aggressiveLazyLoading的值应该为false。这个参数在3.4.5版本之前默认值为ture，从3.4.5版本开始默认值改为false。

5)MyBatis提供的lazyLoadTriggerMethods参数，支持在触发某方法时直接触发延迟加载属性的查询，如equals()方法。
