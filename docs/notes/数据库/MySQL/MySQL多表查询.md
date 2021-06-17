# MySQL多表查询

## 使用SELECT子句进行多表查询

```sql
SELECT 字段名 FROM 表1，表2 … WHERE 表1.字段 = 表2.字段 AND 其它查询条件

SELECT a.id,a.name,a.address,a.date,b.math,b.english,b.chinese FROM tb_demo065_tel AS b,tb_demo065 AS a WHERE a.id=b.id

```

注:在上面的的代码中，以两张表的id字段信息相同作为条件建立两表关联，但在实际开发中不应该这样使用，最好用主外键约束来实现

## 使用表的别名进行多表查询

如:

```sql
SELECT a.id,a.name,a.address,b.math,b.english,b.chinese FROM tb_demo065  a,tb_demo065_tel  b WHERE a.id=b.id AND b.id='$_POST[textid]'

```

SQL语言中，可以通过两种方式为表指定别名 

第一种是通过关键字AS指定,如

```sql
SELECT a.id,a.name,a.address,b.math,b.english,b.chinese FROM tb_demo065 AS a,tb_demo065_tel AS b WHERE a.id=b.id

```

第二种是在表名后直接加表的别名实现

```sql
SELECT a.id,a.name,a.address,b.math,b.english,b.chinese FROM tb_demo065  a,tb_demo065_tel  b WHERE a.id=b.id 

```

使用表的别名应注意几下几点

1. 别名通常是一个缩短了的表名，用于在连接中引用表中的特定列，如果连接中的多个表中有相同的名称列存在，必须用表名或表的别名限定列名
2. 如果定义了表的别名就不能再使用表名

## 合并多个结果集

SQL语言中，可以通过UNION 或 ALL将多个SELECT语句的查询结果合并输出，这两个关键字的使用说明如下：

UNION:利用该关键字可以将多个SELECT 语句的查询结果合并输出，并删除重复行

ALL:利用该关键字可以将多个SELECT 语句的查询结果合并输出，但不会删除重复行

在使用UNION或ALL关键字将多个表合并输出时，查询结果必须具有相同的结构并且数据类型必须兼容,另外使用UNION时两张表的字段数量也必须相同，否则会提示SQL语句有错误。

e.x:SELECT id,name,pwd FROM tb_demo067 UNION SELECT uid,price,date FROM tb_demo067_tel

## 简单嵌套查询

子查询:子查询是一个SELECT查询，返回单个值且嵌套在SELECT、INSERT、UPDATE和DELETE语句或其它查询语句中，任何可以使用表达式的地方都可以使用子查询.

```sql
SELECT id,name,sex,date FROM tb_demo068 WHERE id in(SELECT id FROM tb_demo068 WHERE id='$_POST[test]')

```

内连接：把查询结果作为WHERE子句的查询条件即称为内连接

## 复杂的嵌套查询

多表之间的嵌套查询可以通过谓词IN实现，语法格式如下:

```sql
test_expression[NOT] IN{

 subquery

}

```

参数说明：test_expression指SQL表达式，subquery包含某结果集的子查询

多表嵌套查询的原理:无论是多少张表进行嵌套，表与表之间一定存在某种关联，通过WHERE子句建立此种关联实现查询

## 嵌套查询在查询统计中的应用

实现多表查询时，可以同时使用谓词ANY、SOME、ALL,这些谓词被称为定量比较谓词，可以和比较运算符联合使用，判断是否全部返回值都满足搜索条件.SOME和ANY谓词是存在量的，只注重是否有返回值满足搜索条件，这两个谓词的含义相同，可以替换使用;ALL谓词称为通用谓词，它只关心是否有谓词满足搜索要求.

```sql
SELECT * FROM tb_demo069_people WHERE uid IN(SELECT deptID FROM tb_demo069_dept WHERE deptName='$_POST[select]')

SELECT a.id,a.name FROM tb_demo067 AS a WHERE id<3)

```

> ANY 大于子查询中的某个值 

> =ANY 大于等于子查询中的某个值 

<=ANY 小于等于子查询中的某个值 

=ANY 等于子查询中的某个值 

!=ANY或<>ANY 不等于子查询中的某个值 

> ALL 大于子查询中的所有值 

> =ALL 大于等于子查询中的所有值 

<=ALL 小于等于子查询中的所有值 

=ALL 等于子查询中的所有值 

!=ALL或<>ALL 不等于子查询中的所有值

## 使用子查询作派生的表

在实际项目开发过程中经常用到从一个信息较为完善的表中派生出一个只含有几个关键字段的信息表，通过子查询就可以来实现这一目标,如

```sql
SELECT people.name,people.chinese,people.math,people.english FROM (SELECT name,chinese,math,english FROM tb_demo071) AS people

```

注:子查询应遵循以下规则:

(1)由比较运算符引入的内层子查询只包含一个表达式或列名，在外层语句中的WHERE子句内命名的列必须与内层子查询命名的列兼容

(2)由不可更改的比较运算符引入的子查询(比较运算符后面不跟关键字ANY或ALL)不包括GROUP BY 或 HAVING子句，除非预先确定了成组或单个的值

(3)用EXISTS引入的SELECT列表一般都由*组成，不必指定列名

(4)子查询不能在内部处理其结果

## 使用子查询作表达式

SELECT (SELECT AVG(chinese)FROM tb_demo071),(SELECT AVG(english)FROM tb_demo071),(SELECT AVG(math)FROM tb_demo071) FROM tb_demo071

注：在使用子查询时最好为列表项取个别名，这样可以方便用户在使用mysql_fetch_array()函数时为表项赋值,如

```sql
SELECT (SELECT AVG(chinese) FROM tb_demo071) AS yuwen ,(SELECT AVG(english) FROM tb_demo071) AS yingyu,(SELECT AVG(math) FROM tb_demo071) AS shuxue FROM tb_demo071

```

## 使用子查询关联数据

SELECT * FROM tb_demo072_student WHERE id=(SELECT id FROM tb_demo072_class WHERE className = '$_POST[text]')

## 多表联合查询

利用SQL语句中的UNION，可以将不同表中符合条件的数据信息显示在同一列中。

```sql
e.x:SELECT * FROM tb_demo074_student UNION SELECT * FROM tb_demo074_fasten

```

注:使用UNION时应注意以下两点：

(1)在使用UNION运算符组合的语句中，所有选择列表的表达式数目必须相同，如列名、算术表达式及聚合函数等

(2)在每个查询表中，对应列的数据结构必须一样。

## 对联合后的结果进行排序

为了UNION的运算兼容，要求所有SELECT语句都不能有ORDER BY语句，但有一种情况例外，那就是在最后一个SELECT语句中放置ORDER BY 子句实现结果的最终排序输出。

```sql
e.x:SELECT * FROM tb_demo074_student UNION SELECT * FROM tb_demo074_fasten ORDER BY id

```

使用UNION条件上相对比较苛刻，所以使用此语句时一定要注意两个表项数目和字段类型是否相同

## 条件联合语句

```sql
SELECT * FROM tb_demo076_BEIJING GROUP BY name HAVING name='人民邮电出版社' OR name='机械工业出版社' UNION SELECT * FROM tb_demo076_BEIJING GROUP BY name HAVING name <>'人民邮电出版社' AND name <>'机械工业再版社' ORDER BY id

```

上面语句应用了GROUP BY分组语句和HAVING语句实现条件联合查询。其实现目的是先保证将'人民邮电出版社'和'机械工业出版社'始终位于名单最前列，然后再输出其它的出版社

## 简单内连接查询

```sql
SELECT filedlist FROM table1 [INNER] JOIN table2 ON table1.column1 = table2.column1

```

其中，filedlist是要显示的字段,INNER表示表之间的连接方式为内连接，table1.column1=table2.column1用于指明两表间的连接条件，如:

```sql
SELECT a.name,a.address,a.date,b.chinese,b.math,b.english FROM tb_demo065 AS a INNER JOIN tb_demo065_tel AS b on a.id=b.id

```

## 复杂内连接查询

复杂的内连接查询是在基本的内连接查询的基础上再附加一些查询条件，如:

```sql
SELECT a.name,a.address,a.date,b.chinese,b.math,b.english FROM tb_demo065 AS a INNER JOIN tb_demo065_tel AS b on a.id=b.id WHERE b.id=(SELECT id FROM  tb_demo065 WHERE tb_demo065.name='$_POST[text]')

```

总之，实现表与表之间的关联的本质是两表之间存在共同的数据项或者相同的数据项，通过WHERE 子句或内连接INNER JOIN … ON 语句将两表连接起来，实现查询

## 使用外连接实现多表联合查询

(1)LEFT OUTER JOIN表示表之间通过左连接方式相互连接，也可简写成LEFT JOIN,它是以左侧的表为基准故称左连接，左侧表中所有信息将被全部输出，而右侧表信息则只会输出符合条件的信息，对不符合条件的信息则返回NULL

```sql
e.x:SELECT a.name,a.address,b.math,b.english FROM tb_demo065 AS A LEFT OUTER JOIN tb_demo065_tel AS b ON a.id=b.id

```

(2)RIGHT OUTER JOIN表示表之间通过右连接方式相互连接，也可简写成RIGHT JOIN,它是以右侧的表为基准故称右连接，右侧表中所有信息将被全部输出，而左侧表信息则只会输出符合条件的信息，对不符合条件的信息则返回NULL

```sql
E.X:SELECT a.name,a.address,b.math,b.english FROM tb_demo065 AS A RIGHT OUTER JOIN tb_demo065_tel AS b ON a.id=b.id

```

## 利用IN或NOTIN关键字限定范围

```sql
e.x:SELECT * FROM tb_demo083 WHERE code IN(SELECT code FROM tb_demo083 WHERE code BETWEEN '$_POST[text1]' AND '$_POST[text2]')

```

利用IN可指定在范围内查询，若要求在某范围外查询可以用NOT IN代替它

## 由IN引入的关联子查询

```sql
e.x:SELECT * FROM tb_demo083 WHERE code IN(SELECT code FROM tb_demo083 WHERE code = '$_POST[text]')

```

## 利用HAVING语句过滤分组数据

HAVING子句用于指定组或聚合的搜索条件，HAVING通常与GROUP BY 语句一起使用，如果SQL语句中不含GROUP BY子句，则HAVING的行为与WHERE子句一样.

```sql
e.x:SELECT name,math FROM tb_demo083 GROUP BY id HAVING math > '95'
```
