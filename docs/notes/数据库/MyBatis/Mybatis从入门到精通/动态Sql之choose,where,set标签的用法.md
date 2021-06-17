# 动态Sql之choose,where,set标签的用法

本篇博客主要讲解如何使用choose,where,set标签生成动态的Sql。

## choose 用法

假设有这样1个需求：当参数id有值时优先使用id查询，当id没有值时就去判断用户名是否有值，如果有值就用用户名查询，如果没值，就使查询无结果。

首先，我们在接口SysUserMapper中添加如下方法：

```java
/**
 * 根据用户id或用户名查询
 *
 * @param sysUser
 * @return
 */
SysUser selectByIdOrUserName(SysUser sysUser);
```

然后在对应的SysUserMapper.xml中添加如下代码：

```xml
<select id="selectByIdOrUserName" resultType="run.yuyang.mybatis.model.SysUser">
    SELECT  id,
            user_name,
            user_password,
            user_email,
            create_time
    FROM sys_user
    WHERE 1 = 1
    <choose>
        <when test="id != null">
            AND id = #{id}
        </when>
        <when test="userName != null and userName != ''">
            AND user_name = #{userName}
        </when>
        <otherwise>
            AND 1 = 2
        </otherwise>
    </choose>
</select>
```

**注意事项：**

在以上的代码中，如果没有otherwise这个限制条件，当id和userName都为null时，所有的用户都会被查询出来，但我们接口的返回值是SysUser，当查询结果是多个时就会报错。添加otherwise条件后，由于where条件不满足，因此在这种情况下就查询不到结果。

最后，在SysUserMapperTest测试类中添加如下测试方法：

```java
@Test
public void testSelectByIdOrUserName() {
    SqlSession sqlSession = getSqlSession();

    try {
        SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);

        // 按id查询
        SysUser query = new SysUser();
        query.setId(1L);
        query.setUserName("admin");
        SysUser sysUser = sysUserMapper.selectByIdOrUserName(query);
        Assert.assertNotNull(sysUser);

        // 只按userName查询
        query.setId(null);
        sysUser = sysUserMapper.selectByIdOrUserName(query);
        Assert.assertNotNull(sysUser);

        // id 和 userName 都为空
        query.setUserName(null);
        sysUser = sysUserMapper.selectByIdOrUserName(query);
        Assert.assertNull(sysUser);
    } finally {
        sqlSession.close();
    }
}
```

运行测试代码，测试通过，输出日志如下：

> DEBUG [main] - ==>  Preparing: SELECT id, user_name, user_password, user_email, create_time FROM sys_user WHERE 1 = 1 AND id = ?
>
> DEBUG [main] - ==> Parameters: 1(Long)
>
> TRACE [main] - <==    Columns: id, user_name, user_password, user_email, create_time
>
> TRACE [main] - <==        Row: 1, admin, 123456, admin@mybatis.tk, 2019-06-27 18:21:07.0
>
> DEBUG [main] - <==      Total: 1
>
> DEBUG [main] - ==>  Preparing: SELECT id, user_name, user_password, user_email, create_time FROM sys_user WHERE 1 = 1 AND user_name = ?
>
> DEBUG [main] - ==> Parameters: admin(String)
>
> TRACE [main] - <==    Columns: id, user_name, user_password, user_email, create_time
>
> TRACE [main] - <==        Row: 1, admin, 123456, admin@mybatis.tk, 2019-06-27 18:21:07.0
>
> DEBUG [main] - <==      Total: 1
>
> DEBUG [main] - ==>  Preparing: SELECT id, user_name, user_password, user_email, create_time FROM sys_user WHERE 1 = 1 AND 1 = 2
>
> DEBUG [main] - ==> Parameters:
>
> DEBUG [main] - <==      Total: 0

## where 用法

where标签的作用：如果该标签包含的元素中有返回值，就插入一个where，如果where后面的字符串是以AND或者OR开头的，就将它们剔除。

假设有这样1个需求：根据用户的输入条件来查询用户列表，如果输入了用户名，就根据用户名模糊查询，如果输入了邮箱，就根据邮箱精确查询，如果同时输入了用户名和邮箱，就用这两个条件去匹配用户。

首先，我们在接口SysUserMapper中添加如下方法：

```java
/**
 * 根据动态条件查询用户信息(使用Where标签)
 *
 * @param sysUser
 * @return
 */
List<SysUser> selectByUserWhere(SysUser sysUser);
```

然后在对应的SysUserMapper.xml中添加如下代码：

```xml
<select id="selectByUserWhere" resultType="run.yuyang.mybatis.model.SysUser">
    SELECT id,
    user_name,
    user_password,
    user_email,
    create_time
    FROM sys_user
    <where>
        <if test="userName != null and userName != ''">
            AND user_name LIKE CONCAT('%',#{userName},'%')
        </if>
        <if test="userEmail != null and userEmail != ''">
            AND user_email = #{userEmail}
        </if>
    </where>
</select>
```

当if条件都不满足的时候，where元素中没有内容，此时的Sql语句不会有where，语法正确。

如果if条件满足，where元素的内容就是以AND开头的条件，where会自动去掉开头的and，保证Sql语句的正确。

最后，在SysUserMapperTest测试类中添加如下测试方法：

```java
@Test
public void testSelectByUserWhere() {
    SqlSession sqlSession = getSqlSession();

    try {
        SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);

        // 只按用户名查询
        SysUser query = new SysUser();
        query.setUserName("ad");
        List<SysUser> sysUserList = sysUserMapper.selectByUserWhere(query);
        Assert.assertTrue(sysUserList.size() > 0);

        // 只按邮箱查询
        query = new SysUser();
        query.setUserEmail("test@mybatis.tk");
        sysUserList = sysUserMapper.selectByUserWhere(query);
        Assert.assertTrue(sysUserList.size() > 0);

        // 同时按用户民和邮箱查询
        query = new SysUser();
        query.setUserName("ad");
        query.setUserEmail("test@mybatis.tk");
        sysUserList = sysUserMapper.selectByUserWhere(query);
        // 由于没有同时符合这两个条件的用户，因此查询结果数为0
        Assert.assertTrue(sysUserList.size() == 0);
    } finally {
        sqlSession.close();
    }
}

```

运行测试代码，测试通过，输出日志如下：

> DEBUG [main] - ==>  Preparing: SELECT id, user_name, user_password, user_email, create_time FROM sys_user WHERE user_name LIKE CONCAT('%',?,'%')
>
> DEBUG [main] - ==> Parameters: ad(String)
>
> TRACE [main] - <==    Columns: id, user_name, user_password, user_email, create_time
>
> TRACE [main] - <==        Row: 1, admin, 123456, admin@mybatis.tk, 2019-06-27 18:21:07.0
>
> DEBUG [main] - <==      Total: 1
>
> DEBUG [main] - ==>  Preparing: SELECT id, user_name, user_password, user_email, create_time FROM sys_user WHERE user_email = ?
>
> DEBUG [main] - ==> Parameters: test@mybatis.tk(String)
>
> TRACE [main] - <==    Columns: id, user_name, user_password, user_email, create_time
>
> TRACE [main] - <==        Row: 1001, test, 123456, test@mybatis.tk, 2019-06-27 18:21:07.0
>
> DEBUG [main] - <==      Total: 1
>
> DEBUG [main] - ==>  Preparing: SELECT id, user_name, user_password, user_email, create_time FROM sys_user WHERE user_name LIKE CONCAT('%',?,'%') AND user_email = ?
>
> DEBUG [main] - ==> Parameters: ad(String), test@mybatis.tk(String)
>
> DEBUG [main] - <==      Total: 0

## set 用法

set标签的作用：如果该标签包含的元素中有返回值，就插入一个set，如果set后面的字符串是以,结尾的，就将这个逗号剔除。

假设有这样1个需求：更新用户信息的时候不能将原来有值但没有发生变化的字段更新为空或null，即只更新有值的字段。

首先，我们在接口SysUserMapper中添加如下方法：

```java
/**
 * 根据主键选择性更新用户信息(使用Set标签)
 *
 * @param sysUser
 * @return
 */
int updateByIdSelectiveSet(SysUser sysUser);
```

然后在对应的SysUserMapper.xml中添加如下代码：

```xml
<update id="updateByIdSelectiveSet">
    UPDATE sys_user
    <set>
        <if test="userName != null and userName != ''">
            user_name = #{userName},
        </if>
        <if test="userPassword != null and userPassword != ''">
            user_password = #{userPassword},
        </if>
        <if test="userEmail != null and userEmail != ''">
            user_email = #{userEmail},
        </if>
        <if test="userInfo != null and userInfo != ''">
            user_info = #{userInfo},
        </if>
        <if test="headImg != null">
            head_img = #{headImg,jdbcType=BLOB},
        </if>
        <if test="createTime != null">
            create_time = #{createTime,jdbcType=TIMESTAMP},
        </if>
        id = #{id},
    </set>
    WHERE id = #{id}
</update>

```

**注意事项：**为了避免所有的条件都不满足，生成的Sql语句没有set标签，因此在最后加上了`id = #{id},`这样必然存在的赋值。

最后，在SysUserMapperTest测试类中添加如下测试方法：

```java
@Test
public void testUpdateByIdSelectiveSet() {
    SqlSession sqlSession = getSqlSession();

    try {
        SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);

        SysUser sysUser = new SysUser();
        // 更新id=1的用户
        sysUser.setId(1L);
        // 修改邮箱
        sysUser.setUserEmail("test@mybatis.tk");

        int result = sysUserMapper.updateByIdSelectiveSet(sysUser);
        Assert.assertEquals(1, result);

        // 查询id=1的用户
        sysUser = sysUserMapper.selectById(1L);
        // 修改后的名字保持不变，但是邮箱变成了新的
        Assert.assertEquals("admin", sysUser.getUserName());
        Assert.assertEquals("test@mybatis.tk", sysUser.getUserEmail());
    } finally {
        sqlSession.close();
    }
}

```

运行测试代码，测试通过，输出日志如下：

> DEBUG [main] - ==>  Preparing: UPDATE sys_user SET user_email = ?, id = ? WHERE id = ?
>
> DEBUG [main] - ==> Parameters: test@mybatis.tk(String), 1(Long), 1(Long)
>
> DEBUG [main] - <==    Updates: 1
>
> DEBUG [main] - ==>  Preparing: SELECT id, user_name, user_password, user_email, create_time FROM sys_user WHERE id = ?
>
> DEBUG [main] - ==> Parameters: 1(Long)
>
> TRACE [main] - <==    Columns: id, user_name, user_password, user_email, create_time
>
> TRACE [main] - <==        Row: 1, admin, 123456, test@mybatis.tk, 2019-06-27 18:21:07.0
>
> DEBUG [main] - <==      Total: 1

