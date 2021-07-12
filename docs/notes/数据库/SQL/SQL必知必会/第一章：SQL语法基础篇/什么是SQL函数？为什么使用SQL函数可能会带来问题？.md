# 什么是SQL函数？为什么使用SQL函数可能会带来问题？

函数在计算机语言的使用中贯穿始终，在 SQL 中我们也可以使用函数对检索出来的数据进行函数操作，比如求某列数据的平均值，或者求字符串的长度等。从函数定义的角度出发，我们可以将函数分成内置函数和自定义函数。在 SQL 语言中，同样也包括了内置函数和自定义函数。内置函数是系统内置的通用函数，而自定义函数是我们根据自己的需要编写的，下面讲解的是 SQL 的内置函数。

你需要从以下几个方面掌握 SQL 函数：

1. 什么是 SQL 函数？
2. 内置的 SQL 函数都包括哪些？
3. 如何使用 SQL 函数对一个数据表进行操作，比如针对一个王者荣耀的英雄数据库，我们可以使用这些函数完成哪些操作？
4. 什么情况下使用 SQL 函数？为什么使用 SQL 函数有时候会带来问题？

## 什么是 SQL 函数

当我们学习编程语言的时候，也会遇到函数。函数的作用是什么呢？它可以把我们经常使用的代码封装起来，需要的时候直接调用即可。这样既提高了代码效率，又提高了可维护性。

SQL 中的函数一般是在数据上执行的，可以很方便地转换和处理数据。一般来说，当我们从数据表中检索出数据之后，就可以进一步对这些数据进行操作，得到更有意义的结果，比如返回指定条件的函数，或者求某个字段的平均值等。

## 常用的 SQL 函数有哪些

SQL 提供了一些常用的内置函数，当然你也可以自己定义 SQL 函数。SQL 的内置函数对于不同的数据库软件来说具有一定的通用性，我们可以把内置函数分成四类：

1. 算术函数
2. 字符串函数
3. 日期函数
4. 转换函数

这 4 类函数分别代表了算术处理、字符串处理、日期处理、数据类型转换，它们是 SQL 函数常用的划分形式，你可以思考下，为什么是这 4 个维度？

函数是对提取出来的数据进行操作，那么数据表中字段类型的定义有哪几种呢？

我们经常会保存一些数值，不论是整数类型，还是浮点类型，实际上对应的就是数值类型。同样我们也会保存一些文本内容，可能是人名，也可能是某个说明，对应的就是字符串类型。此外我们还需要保存时间，也就是日期类型。那么针对数值、字符串和日期类型的数据，我们可以对它们分别进行算术函数、字符串函数以及日期函数的操作。如果想要完成不同类型数据之间的转换，就可以使用转换函数。

算术函数，顾名思义就是对数值类型的字段进行算术运算。常用的算术函数及含义如下表所示：

![](./images/07-01.png)

这里我举一些简单的例子，你来体会下：

<code>SELECT ABS(-2)</code>，运行结果为 2。

<code>SELECT MOD(101,3)</code>，运行结果 2。

<code>SELECT ROUND(37.25,1)</code>，运行结果 37.3。

常用的字符串函数操作包括了字符串拼接，大小写转换，求长度以及字符串替换和截取等。具体的函数名称及含义如下表所示：

![](./images/07-02.png)

<code>SELECT CONCAT('abc', 123)</code>，运行结果为 abc123。

<code>SELECT LENGTH('你好')</code>，运行结果为 6。

<code>SELECT CHAR_LENGTH('你好')</code>，运行结果为 2。

<code>SELECT LOWER('ABC')</code>，运行结果为 abc。

<code>SELECT UPPER('abc')</code>，运行结果 ABC。

<code>SELECT REPLACE('fabcd', 'abc', 123)</code>，运行结果为 f123d。

<code>SELECT SUBSTRING('fabcd', 1,3)</code>，运行结果为 fab。

日期函数是对数据表中的日期进行处理，常用的函数包括：

![](./images/07-03.png)

下面是一些简单的例子，你可自己运行下：

<code>SELECT CURRENT_DATE()</code>，运行结果为 2019-04-03。

<code>SELECT CURRENT_TIME()</code>，运行结果为 21:26:34。

<code>SELECT CURRENT_TIMESTAMP()</code>，运行结果为 2019-04-03 21:26:34。

<code>SELECT EXTRACT(YEAR FROM '2019-04-03')</code>，运行结果为 2019。

<code>SELECT DATE('2019-04-01 12:00:05')</code>，运行结果为 2019-04-01。

这里需要注意的是，DATE 日期格式必须是 yyyy-mm-dd 的形式。如果要进行日期比较，就要使用 DATE 函数，不要直接使用日期与字符串进行比较，我会在后面的例子中讲具体的原因。

转换函数可以转换数据之间的类型，常用的函数如下表所示：

![](./images/07-04.png)

<code>SELECT CAST(123.123 AS INT)</code>，运行结果会报错。

<code>SELECT CAST(123.123 AS DECIMAL(8,2))</code>，运行结果为 123.12。

<code>SELECT COALESCE(null,1,2)</code>，运行结果为 1。

CAST 函数在转换数据类型的时候，不会四舍五入，如果原数值有小数，那么转换为整数类型的时候就会报错。不过你可以指定转化的小数类型，在 MySQL 和 SQL Server 中，你可以用<code>DECIMAL(a,b)</code>来指定，其中 a 代表整数部分和小数部分加起来最大的位数，b 代表小数位数，比如<code>DECIMAL(8,2)</code>代表的是精度为 8 位（整数加小数位数最多为 8 位），小数位数为 2 位的数据类型。所以<code>SELECT CAST(123.123 AS DECIMAL(8,2))</code>的转换结果为 123.12。

## 用 SQL 函数对王者荣耀英雄数据做处理

我创建了一个王者荣耀英雄数据库，一共有 69 个英雄，23 个属性值。SQL 文件见 Github 地址：<a href="https://github.com/cystanford/sql_heros_data">https://github.com/cystanford/sql_heros_data</a>。

![](./images/07-05.png)

首先显示英雄以及他的物攻成长，对应字段为<code>attack_growth</code>。我们让这个字段精确到小数点后一位，需要使用的是算术函数里的 ROUND 函数。

```
SQL：SELECT name, ROUND(attack_growth,1) FROM heros

```

代码中，<code>ROUND(attack_growth,1)</code>中的<code>attack_growth</code>代表想要处理的数据，“1”代表四舍五入的位数，也就是我们这里需要精确到的位数。

运行结果为：

![](./images/07-06.png)

```
SQL：SELECT MAX(hp_max) FROM heros

```

运行结果为 9328。

假如我们想要知道最大生命值最大的是哪个英雄，以及对应的数值，就需要分成两个步骤来处理：首先找到英雄的最大生命值的最大值，即<code>SELECT MAX(hp_max) FROM heros</code>，然后再筛选最大生命值等于这个最大值的英雄，如下所示。

```
SQL：SELECT name, hp_max FROM heros WHERE hp_max = (SELECT MAX(hp_max) FROM heros)

```

运行结果：

![](./images/07-07.png)

假如我们想显示英雄的名字，以及他们的名字字数，需要用到<code>CHAR_LENGTH</code>函数。

```
SQL：SELECT CHAR_LENGTH(name), name FROM heros

```

运行结果为：

![](./images/07-08.png)

假如想要提取英雄上线日期（对应字段 birthdate）的年份，只显示有上线日期的英雄即可（有些英雄没有上线日期的数据，不需要显示），这里我们需要使用 EXTRACT 函数，提取某一个时间元素。所以我们需要筛选上线日期不为空的英雄，即<code>WHERE birthdate is not null</code>，然后再显示他们的名字和上线日期的年份，即：

```
SQL： SELECT name, EXTRACT(YEAR FROM birthdate) AS birthdate FROM heros WHERE birthdate is NOT NULL

```

或者使用如下形式：

```
SQL: SELECT name, YEAR(birthdate) AS birthdate FROM heros WHERE birthdate is NOT NULL

```

运行结果为：

![](./images/07-09.png)

假设我们需要找出在 2016 年 10 月 1 日之后上线的所有英雄。这里我们可以采用 DATE 函数来判断 birthdate 的日期是否大于 2016-10-01，即<code>WHERE DATE(birthdate)&gt;'2016-10-01'</code>，然后再显示符合要求的全部字段信息，即：

```
SQL： SELECT * FROM heros WHERE DATE(birthdate)&gt;'2016-10-01'

```

需要注意的是下面这种写法是不安全的：

```
SELECT * FROM heros WHERE birthdate&gt;'2016-10-01'

```

因为很多时候你无法确认 birthdate 的数据类型是字符串，还是 datetime 类型，如果你想对日期部分进行比较，那么使用<code>DATE(birthdate)</code>来进行比较是更安全的。

运行结果为：

![](./images/07-10.png)

假设我们需要知道在 2016 年 10 月 1 日之后上线英雄的平均最大生命值、平均最大法力和最高物攻最大值。同样我们需要先筛选日期条件，即<code>WHERE DATE(birthdate)&gt;'2016-10-01'</code>，然后再选择<code>AVG(hp_max), AVG(mp_max), MAX(attack_max)</code>字段进行显示。

```
SQL： SELECT AVG(hp_max), AVG(mp_max), MAX(attack_max) FROM heros WHERE DATE(birthdate)&gt;'2016-10-01'

```

运行结果为：

![](./images/07-11.png)

## 为什么使用 SQL 函数会带来问题

尽管 SQL 函数使用起来会很方便，但我们使用的时候还是要谨慎，因为你使用的函数很可能在运行环境中无法工作，这是为什么呢？

如果你学习过编程语言，就会知道语言是有不同版本的，比如 Python 会有 2.7 版本和 3.x 版本，不过它们之间的函数差异不大，也就在 10% 左右。但我们在使用 SQL 语言的时候，不是直接和这门语言打交道，而是通过它使用不同的数据库软件，即 DBMS。DBMS 之间的差异性很大，远大于同一个语言不同版本之间的差异。实际上，只有很少的函数是被 DBMS 同时支持的。比如，大多数 DBMS 使用（||）或者（+）来做拼接符，而在 MySQL 中的字符串拼接函数为<code>Concat()</code>。大部分 DBMS 会有自己特定的函数，这就意味着采用 SQL 函数的代码可移植性是很差的，因此在使用函数的时候需要特别注意。

## 关于大小写的规范

细心的人可能会发现，我在写 SELECT 语句的时候用的是大写，而你在网上很多地方，包括你自己写的时候可能用的是小写。实际上在 SQL 中，关键字和函数名是不用区分字母大小写的，比如 SELECT、WHERE、ORDER、GROUP BY 等关键字，以及 ABS、MOD、ROUND、MAX 等函数名。

不过在 SQL 中，你还是要确定大小写的规范，因为在 Linux 和 Windows 环境下，你可能会遇到不同的大小写问题。

比如 MySQL 在 Linux 的环境下，数据库名、表名、变量名是严格区分大小写的，而字段名是忽略大小写的。

而 MySQL 在 Windows 的环境下全部不区分大小写。

这就意味着如果你的变量名命名规范没有统一，就可能产生错误。这里有一个有关命名规范的建议：

1. 关键字和函数名称全部大写；
2. 数据库名、表名、字段名称全部小写；
3. SQL 语句必须以分号结尾。

虽然关键字和函数名称在 SQL 中不区分大小写，也就是如果小写的话同样可以执行，但是数据库名、表名和字段名在 Linux MySQL 环境下是区分大小写的，因此建议你统一这些字段的命名规则，比如全部采用小写的方式。同时将关键词和函数名称全部大写，以便于区分数据库名、表名、字段名。

## 总结

函数对于一门语言的重要性毋庸置疑，我们在写 Python 代码的时候，会自己编写函数，也会使用 Python 内置的函数。在 SQL 中，使用函数的时候需要格外留意。不过如果工程量不大，使用的是同一个 DBMS 的话，还是可以使用函数简化操作的，这样也能提高代码效率。只是在系统集成，或者在多个 DBMS 同时存在的情况下，使用函数的时候就需要慎重一些。

比如<code>CONCAT()</code>是字符串拼接函数，在 MySQL 和 Oracle 中都有这个函数，但是在这两个 DBMS 中作用却不一样，<code>CONCAT</code>函数在 MySQL 中可以连接多个字符串，而在 Oracle 中<code>CONCAT</code>函数只能连接两个字符串，如果要连接多个字符串就需要用（||）连字符来解决。

![](./images/07-12.png)

欢迎你在评论区与我分享你的答案，也欢迎点击”请朋友读“，把这篇文章分享给你的朋友或者同事。

![](./images/07-13.png)

