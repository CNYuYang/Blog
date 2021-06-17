# XML方式的基本用法之增删改

## insert用法

### 简单的insert方法

假如现在我们想新增一个用户，该如何操作呢？

首先，在接口SysUserMapper中添加如下方法。

```java
/**
 * 新增用户
 *
 * @param sysUser
 * @return
 */
int insert(SysUser sysUser);
```

然后打开对应的SysUserMapper.xml文件，添加如下语句。

```xml
<insert id="insert">
    INSERT INTO sys_user(id, user_name, user_password, user_email, user_info, head_img, create_time)
    VALUES (#{id},#{userName},#{userPassword},#{userEmail},#{userInfo},#{headImg,jdbcType=BLOB},#{createTime,jdbcType=TIMESTAMP})
</insert>
```

**特别说明：**

1)为了防止类型错误，对于一些特殊的数据类型，建议指定具体的jdbcType值。例如headImg指定BLOB类型，createTime指定TIMESTAMP类型。

2)BLOB对应的类型是ByteArrayInputStream，就是二进制数据流。

3)由于数据库区分date、time、datetime类型，但是在Java中一般都使用java.util.Date类型。因此为了保证数据类型的正确，需要手动指定日期类型。date、time、datetime对应的JDBC类型分别为DATE、TIME、TIMESTAMP。

新建测试类，用于测试insert()方法。

```java
public class TestChapter4 {

    private static SqlSessionFactory sqlSessionFactory;

    public static void main(String[] args) {

        try  {
            @Cleanup Reader reader = Resources.getResourceAsReader("mybatis-config.xml");
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
            @Cleanup SqlSession sqlSession = sqlSessionFactory.openSession();
            SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);
            SysUser sysUser = new SysUser();
            sysUser.setUserName("test1");
            sysUser.setUserPassword("123456");
            sysUser.setUserEmail("test@mybatis.tk");
            sysUser.setUserInfo("test info");
            // 正常情况下应该读入一张图片保存到byte数组中
            sysUser.setHeadImg(new byte[]{1, 2, 3});
            sysUser.setCreateTime(new Date());

            // 这里的返回值result是执行的SQL影响的行数
            int result = sysUserMapper.insert(sysUser);
            System.out.println(result);

        } catch (Exception e) {
            e.printStackTrace();
        }

    }

}
```

程序输出`1`说明插入成功。

### 返回主键值(JDBC方式)

在上面的例子中，新增完数据，我们并没有拿到数据库中自增的id值，但有些场景中，我们需要先拿到数据库中自增的值，然后再处理其余的逻辑，那么如何拿到数据库中的自增的id值呢？

首先，在接口SysUserMapper中添加方法如下。

```java
/**
 * 新增用户-使用useGeneratedKeys方式
 *
 * @param sysUser
 * @return
 */
int insertUseGeneratedKeys(SysUser sysUser);
```
然后打开对应的SysUserMapper.xml，添加如下代码。

```xml
<insert id="insertUseGeneratedKeys" useGeneratedKeys="true" keyProperty="id">
    INSERT INTO sys_user(user_name, user_password, user_email, user_info, head_img, create_time)
    VALUES (#{userName},#{userPassword},#{userEmail},#{userInfo},#{headImg,jdbcType=BLOB},#{createTime,jdbcType=TIMESTAMP})
</insert>
```

useGeneratedKeys设置为ture后，MyBatis会使用JDBC的getGeneratedKeys()方法来取出由数据库内部生成的主键。获取到主键后将其赋值给keyProperty配置的id属性。

测试新增的insertUseGeneratedKeys()方法。

```java
SysUser sysUser = new SysUser();
sysUser.setUserName("test2");
sysUser.setUserPassword("123456");
sysUser.setUserEmail("test@mybatis.tk");
sysUser.setUserInfo("test info");
// 正常情况下应该读入一张图片保存到byte数组中
sysUser.setHeadImg(new byte[]{1, 2, 3});
sysUser.setCreateTime(new Date());

// 这里的返回值result是执行的SQL影响的行数
int result = sysUserMapper.insertUseGeneratedKeys(sysUser);
System.out.println(result);
System.out.println(sysUser.getId());
```

运行该测试方法，测试通过，输出日志如下。

```
1
1003
```

### 返回主键值(selectKey方式)

上例中回写主键的方法只适用于支持主键自增的数据库。

但有些数据库（比如Oracle）不提供主键自增的功能，而是使用序列得到一个值，然后将这个值赋给id，再将数据插入到数据库。

对于这种情况，就可以采用selectKey方式，因为selectKey方式不仅适用于不提供主键自增功能的数据库，也适用于提供主键自增功能的数据库。

我们先来看下MySql的例子。

首先，在接口SysUserMapper中添加如下方法。

```java
/**
 * 新增用户-使用selectKey方式
 *
 * @param sysUser
 * @return
 */
int insertUseSelectKey(SysUser sysUser);
```

然后打开对应的SysUserMapper.xml文件，添加如下代码。

```xml
<insert id="insertUseSelectKey">
    INSERT INTO sys_user(user_name, user_password, user_email, user_info, head_img, create_time)
    VALUES (#{userName},#{userPassword},#{userEmail},#{userInfo},#{headImg,jdbcType=BLOB},#{createTime,jdbcType=TIMESTAMP})
    <selectKey keyColumn="id" resultType="long" keyProperty="id" order="AFTER">
        SELECT 1491008
    </selectKey>
</insert>
```

和上面的例子相比，这里的语句多了selectKey标签，其中：

- keyColumn：主键的数据库列名。
- resultType：返回值类型。
- keyProperty：主键对应的属性名。
- **order：该属性的设置和使用的数据库有关，如果使用的是MySql数据库，设置的值是AFTER，因为当前记录的主键值在insert语句执行成功后才能获取到。如果使用的是Oracle数据库，设置的值是BEFORE，因为Oracle中需要先从序列获取值，然后将值作为主键插入到数据库中。**

## update用法

假如我们现在希望通过主键id来更新用户信息，该如何操作呢？

首先，在接口SysUserMapper中添加如下方法。

```java
/**
 * 根据主键更新
 *
 * @param sysUser
 * @return
 */
int updateById(SysUser sysUser);
```

然后，打开对应的SysUserMapper.xml文件，添加如下代码。

```xml
<update id="updateById">
    UPDATE sys_user
    SET user_name = #{userName},
        user_password = #{userPassword},
        user_email = #{userEmail},
        user_info = #{userInfo},
        head_img = #{headImg,jdbcType=BLOB},
        create_time = #{createTime,jdbcType=TIMESTAMP}
    WHERE id = #{id}
</update>
```

在测试类中，添加如下测试方法。

```java
SysUserMapper sysUserMapper = sqlSession.getMapper(SysUserMapper.class);
SysUser sysUser = sysUserMapper.selectById(1L);
sysUser.setUserName("admin_test");

// 这里的返回值result是执行的SQL影响的行数
int result = sysUserMapper.updateById(sysUser);
System.out.println(result);
```

## delete用法

假如我们现在希望通过主键id来删除用户信息，该如何操作呢？

首先，在接口SysUserMapper中添加如下方法。

```java
/**
* 根据主键删除
*
* @param id
* @return
*/
int deleteById(Long id);

/**
* 根据对象的主键删除
*
* @param sysUser
* @return
*/
int deleteBySysUser(SysUser sysUser);
```

然后，打开对应的SysUserMapper.xml文件，添加如下代码。

```xml
<delete id="deleteById">
    DELETE FROM sys_user WHERE id = #{id}
</delete>
<delete id="deleteBySysUser">
    DELETE FROM sys_user WHERE id = #{id}
</delete>
```