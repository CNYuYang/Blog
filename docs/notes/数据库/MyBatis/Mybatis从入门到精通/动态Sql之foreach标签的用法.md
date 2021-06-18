# 动态Sql之foreach标签的用法

本篇博客主要讲解如何使用foreach标签生成动态的Sql，主要包含以下3个场景：

1. foreach 实现in集合
2. foreach 实现批量插入
3. foreach 实现动态update

## foreach 实现in集合

假设有这样1个需求：根据传入的用户id集合查询出所有符合条件的用户，此时我们需要使用到Sql中的IN，如 id in (1,1001)。

首先，我们在接口SysUserMapper中添加如下方法：

```java
/**
 * 根据用户id集合查询用户
 *
 * @param idList
 * @return
 */
List<SysUser> selectByIdList(List<Long> idList);
```

然后在对应的SysUserMapper.xml中添加如下代码：

```sql
<select id="selectByIdList" resultType="run.yuyang.mybatis.model.SysUser">
    SELECT id,
    user_name,
    user_password,
    user_email,
    create_time
    FROM sys_user
    WHERE id IN
    <foreach collection="list" open="(" close=")" separator=","
             item="id" index="i">
        #{id}
    </foreach>
</select>
```

最后，在SysUserMapperTest测试类中添加如下测试方法：

```java
@Test
public void testSelectByIdList() {
    SqlSession sqlSession = getSqlSession();

    try {
        SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);

        List<Long> idList = new ArrayList<Long>();
        idList.add(1L);
        idList.add(1001L);

        List<SysUser> userList = sysUserMapper.selectByIdList(idList);
        Assert.assertEquals(2, userList.size());
    } finally {
        sqlSession.close();
    }
}
```

通过日志会发现，foreach元素中的内容最终生成的Sql语句为(1,1001)。

foreach包含属性讲解：

- open：整个循环内容开头的字符串。
- close：整个循环内容结尾的字符串。
- separator：每次循环的分隔符。
- item：从迭代对象中取出的每一个值。
- index：如果参数为集合或者数组，该值为当前索引值，如果参数为Map类型时，该值为Map的key。
- collection：要迭代循环的属性名。

也许有人会好奇，为什么collection的值是list？该值该如何设置呢？

上面的例子中只有一个集合参数，我们把collection属性的值设置为了list，其实也可以写成collection，为什么呢？让我们看下DefaultSqlSession中的默认处理逻辑：

```java
private Object wrapCollection(Object object) {
    DefaultSqlSession.StrictMap map;
    if (object instanceof Collection) {
        map = new DefaultSqlSession.StrictMap();
        map.put("collection", object);
        if (object instanceof List) {
            map.put("list", object);
        }

        return map;
    } else if (object != null && object.getClass().isArray()) {
        map = new DefaultSqlSession.StrictMap();
        map.put("array", object);
        return map;
    } else {
        return object;
    }
}
```

虽然使用默认值，代码也可以正常运行，但还是推荐使用@Param来指定参数的名字，如下所示：

```java
List<SysUser> selectByIdList(@Param("idList") List<Long> idList);

<foreach collection="idList" open="(" close=")" separator=","
         item="id" index="i">
    #{id}
</foreach>
```

如果参数是一个数组参数，collection可以设置为默认值array，看了上面的源码，应该不难理解。

```java
/**
 * 根据用户id数组查询用户
 *
 * @param idArray
 * @return
 */
List<SysUser> selectByIdArray(Long[] idArray);
```

```xml
<select id="selectByIdArray" resultType="run.yuyang.mybatis.model.SysUser">
    SELECT id,
    user_name,
    user_password,
    user_email,
    create_time
    FROM sys_user
    WHERE id IN
    <foreach collection="array" open="(" close=")" separator=","
             item="id" index="i">
        #{id}
    </foreach>
</select>
```

虽然使用默认值，代码也可以正常运行，但还是推荐使用@Param来指定参数的名字，如下所示：

```java
List<SysUser> selectByIdArray(@Param("idArray")Long[] idArray);
```

```xml
<foreach collection="idArray" open="(" close=")" separator=","
         item="id" index="i">
    #{id}
</foreach>
```

## foreach 实现批量插入

假设有这样1个需求：将传入的用户集合批量写入数据库。

首先，我们在接口SysUserMapper中添加如下方法：

```java
/**
 * 批量插入用户信息
 *
 * @param userList
 * @return
 */
int insertList(List<SysUser> userList);
```

然后在对应的SysUserMapper.xml中添加如下代码：

```xml
<insert id="insertList" useGeneratedKeys="true" keyProperty="id">
    INSERT INTO sys_user(user_name, user_password, user_email, user_info, head_img, create_time)
    VALUES
    <foreach collection="list" item="user" separator=",">
        (#{user.userName},#{user.userPassword},#{user.userEmail},#{user.userInfo},#{user.headImg,jdbcType=BLOB},#{user.createTime,jdbcType=TIMESTAMP})
    </foreach>
</insert>
```

最后，在SysUserMapperTest测试类中添加如下测试方法：

```java
@Test
public void testInsertList() {
    SqlSession sqlSession = getSqlSession();

    try {
        SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);

        List<SysUser> sysUserList = new ArrayList<SysUser>();
        for (int i = 0; i < 2; i++) {
            SysUser sysUser = new SysUser();
            sysUser.setUserName("test" + i);
            sysUser.setUserPassword("123456");
            sysUser.setUserEmail("test@mybatis.tk");

            sysUserList.add(sysUser);
        }

        int result = sysUserMapper.insertList(sysUserList);

        for (SysUser sysUser : sysUserList) {
            System.out.println(sysUser.getId());
        }

        Assert.assertEquals(2, result);
    } finally {
        sqlSession.close();
    }
}

```

## foreach 实现动态update

假设有这样1个需求：根据传入的Map参数更新用户信息。

首先，我们在接口SysUserMapper中添加如下方法：

```java
/**
 * 通过Map更新列
 *
 * @param map
 * @return
 */
int updateByMap(Map<String, Object> map);
```

然后在对应的SysUserMapper.xml中添加如下代码：

```xml
<update id="updateByMap">
    UPDATE sys_user
    SET
    <foreach collection="_parameter" item="val" index="key" separator=",">
        ${key} = #{val}
    </foreach>
    WHERE id = #{id}
</update>
```

最后，在SysUserMapperTest测试类中添加如下测试方法：

```java
@Test
public void testUpdateByMap() {
    SqlSession sqlSession = getSqlSession();

    try {
        SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);

        Map<String, Object> map = new HashMap<String, Object>();
        map.put("id", 1L);
        map.put("user_email", "test@mybatis.tk");
        map.put("user_password", "12345678");

        Assert.assertEquals(1, sysUserMapper.updateByMap(map));

        SysUser sysUser = sysUserMapper.selectById(1L);
        Assert.assertEquals("test@mybatis.tk", sysUser.getUserEmail());
        Assert.assertEquals("12345678", sysUser.getUserPassword());
    } finally {
        sqlSession.close();
    }
}
```

上面示例中，collection使用的是默认值_parameter，也可以使用@Param指定参数名字，如下所示：

```java
int updateByMap(@Param("userMap") Map<String, Object> map);
```

```xml
<update id="updateByMap">
    UPDATE sys_user
    SET
    <foreach collection="userMap" item="val" index="key" separator=",">
        ${key} = #{val}
    </foreach>
    WHERE id = #{userMap.id}
</update>
```
