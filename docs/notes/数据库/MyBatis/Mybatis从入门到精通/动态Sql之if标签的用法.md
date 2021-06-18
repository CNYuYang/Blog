# 动态Sql之if标签的用法

本篇博客主要讲解如何使用if标签生成动态的Sql，主要包含以下3个场景：

1. 根据查询条件实现动态查询
2. 根据参数值实现动态更新某些列
3. 根据参数值实现动态插入某些列

## 使用if标签实现动态查询

假设有这样1个需求：根据用户的输入条件来查询用户列表，如果输入了用户名，就根据用户名模糊查询，如果输入了邮箱，就根据邮箱精确查询，如果同时输入了用户名和邮箱，就用这两个条件去匹配用户。

首先，我们在接口SysUserMapper中添加如下方法：

```java
/**
 * 根据动态条件查询用户信息
 *
 * @param sysUser
 * @return
 */
List<SysUser> selectByUser(SysUser sysUser);
```

然后在对应的SysUserMapper.xml中添加如下代码：

```xml
<select id="selectByUser" resultType="run.yuyang.mybatis.model.SysUser">
    SELECT  id,
            user_name,
            user_password,
            user_email,
            create_time
    FROM sys_user
    WHERE 1 = 1
    <if test="userName != null and userName != ''">
        AND user_name LIKE CONCAT('%',#{userName},'%')
    </if>
    <if test="userEmail != null and userEmail != ''">
        AND user_email = #{userEmail}
    </if>
</select>
```

代码简单讲解：

1)if标签的test属性必填，该属性值是一个符合OGNL要求的判断表达式，一般只用true或false作为结果。

2)判断条件property != null 或 property == null，适用于任何类型的字段，用于判断属性值是否为空。

3)判断条件property != '' 或 property ==  ''，仅适用于String类型的字段，用于判断是否为空字符串。

4)当有多个判断条件时，使用and或or进行连接，嵌套的判断可以使用小括号分组，and相当于Java中的与(&&)，or相关于Java中的或(||)。

所以上面代码的意思就是先判断字段是否为null，然后再判断字段是否为空字符串。

最后，在SysUserMapperTest测试类中添加如下测试方法：

```java
@Test
public void testSelectByUser() {
    SqlSession sqlSession = getSqlSession();

    try {
        SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);

        // 只按用户名查询
        SysUser query = new SysUser();
        query.setUserName("ad");
        List<SysUser> sysUserList = sysUserMapper.selectByUser(query);
        Assert.assertTrue(sysUserList.size() > 0);

        // 只按邮箱查询
        query = new SysUser();
        query.setUserEmail("test@mybatis.tk");
        sysUserList = sysUserMapper.selectByUser(query);
        Assert.assertTrue(sysUserList.size() > 0);

        // 同时按用户民和邮箱查询
        query = new SysUser();
        query.setUserName("ad");
        query.setUserEmail("test@mybatis.tk");
        sysUserList = sysUserMapper.selectByUser(query);
        // 由于没有同时符合这两个条件的用户，因此查询结果数为0
        Assert.assertTrue(sysUserList.size() == 0);
    } finally {
        sqlSession.close();
    }
}
```

## 使用if标签实现动态更新

假设有这样1个需求：更新用户信息的时候不能将原来有值但没有发生变化的字段更新为空或null，即只更新有值的字段。

首先，我们在接口SysUserMapper中添加如下方法：

```java
/**
 * 根据主键选择性更新用户信息
 *
 * @param sysUser
 * @return
 */
int updateByIdSelective(SysUser sysUser);
```

然后在对应的SysUserMapper.xml中添加如下代码：

```xml
<update id="updateByIdSelective">
    UPDATE sys_user
    SET
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
    id = #{id}
    WHERE id = #{id}
</update>
```

最后，在SysUserMapperTest测试类中添加如下测试方法：

```java
@Test
public void testUpdateByIdSelective() {
    SqlSession sqlSession = getSqlSession();

    try {
        SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);

        SysUser sysUser = new SysUser();
        // 更新id=1的用户
        sysUser.setId(1L);
        // 修改邮箱
        sysUser.setUserEmail("test@mybatis.tk");

        int result = sysUserMapper.updateByIdSelective(sysUser);
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

## 使用if标签实现动态插入

假设有这样1个需求：往数据库表中插入数据的时候，如果某一列的参数值不为空，就使用传入的值，如果传入的参数值为空，就使用数据库中的默认值(通常是空)，而不使用传入的空值。

为了更好的理解该示例，我们先给sys_user表的user_email字段设置默认值：test@mybatis.tk，Sql语句如下：

```sql
ALTER TABLE sys_user
MODIFY COLUMN user_email VARCHAR(50) NULL DEFAULT  'test@mybatis.tk'
COMMENT '邮箱'
AFTER user_password;
```

首先，我们在接口SysUserMapper中添加如下方法：

```java
/**
 * 根据传入的参数值动态插入列
 *
 * @param sysUser
 * @return
 */
int insertSelective(SysUser sysUser);
```

然后在对应的SysUserMapper.xml中添加如下代码：

```xml
<insert id="insertSelective" useGeneratedKeys="true" keyProperty="id">
    INSERT INTO sys_user(user_name, user_password,
    <if test="userEmail != null and userEmail != ''">
        user_email,
    </if>
    user_info, head_img, create_time)
    VALUES (#{userName},#{userPassword},
    <if test="userEmail != null and userEmail != ''">
        #{userEmail},
    </if>
    #{userInfo},#{headImg,jdbcType=BLOB},#{createTime,jdbcType=TIMESTAMP})
</insert>
```

最后，在SysUserMapperTest测试类中添加如下测试方法：

```java
@Test
public void testInsertSelective() {
    SqlSession sqlSession = getSqlSession();

    try {
        SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);

        SysUser sysUser = new SysUser();
        sysUser.setUserName("test-selective");
        sysUser.setUserPassword("123456");
        sysUser.setUserInfo("test info");
        sysUser.setCreateTime(new Date());

        sysUserMapper.insertSelective(sysUser);

        // 获取刚刚插入的数据
        sysUser = sysUserMapper.selectById(sysUser.getId());
        // 因为没有指定userEmail,所以用的是数据库的默认值
        Assert.assertEquals("test@mybatis.tk", sysUser.getUserEmail());
    } finally {
        sqlSession.close();
    }
}
```

