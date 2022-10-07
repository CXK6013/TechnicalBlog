# SQL查询模型和查疑补漏

> 上一次系统的学习SQL是多久以前了, 貌似时间有点久了。工作中写SQL又对SQL有了新的理解,再加上又最近又打算写SQL优化相关的文章，这里就又系统的梳理一下对SQL的理解, 也算是MySQL优化系列文章的铺垫了.

## SQL 执行顺序

我们平时写的SQL大的基本结构如下: 

```mysql
SELECT DISTINCT
    < select_list >
FROM
    < left_table > < join_type >
JOIN < right_table > ON < join_condition >
WHERE
    < where_condition >
GROUP BY
    < group_by_list >
HAVING
    < having_condition >
ORDER BY
    < order_by_condition >
```

虽然我们写SQL的时候是先SELECT, 但是SELECT其实是最后执行的, 他的执行顺序是下面这样: 

```mysql
FROM <left_table>
ON <join_condition>
<join_type> JOIN <right_table>
WHERE <where_condition>
GROUP BY <group_by_list>
HAVING <having_condition>
ORDER BY <order_by_condition>(有的时候order by也在SELECT之后执行,如果使用了select出来的别名)
SELECT
DISTINCT <select_list>
LIMIT <limit_number>
```

我有段时间尝试将SQL的执行过程当作for循环来看, FROM后面是遍历的范围, SELECT是输出语句，但这似乎无法解释SELECT里面可以套SELECT, 所以我就为这个理解打了一个补丁, 一个SELECT是一个输出语句, 分列两行。但用程序语言的for循环去理解SQL, 总会有说不通的地方,所以这次还是将SQL回归SQL。

-  先是连接

> 连接如果不加on筛选, 那么就是将各个表中的记录依次都取出来依次匹配组合加入结果集并返回给用户。下面是连接过程的示例:

 ![](https://p.pstatp.com/origin/pgc-image/1a36de22b82d4d109476619a80f67320)



这个过程看起来是把t1表的记录和t2表的记录连接起来组成新的更大记录, 这也就是我们常用的连接查询，语法上很简单: 

```mysql
SELECT * FROM t1,t2 
```

上面这种连接方式我们一般称之为内连接，连接查询的结果集中包含一个表中的每一条记录与另一张表的每一条记录的组合,像这样的结果集可以称之为笛卡尔积。表t1有3条记录,表t2有3条记录，最终形成的结果集如上图所示就是9条记录。如果我们不加任何限制几张小表连接, 形成的笛卡尔积就可能而非常巨大。比如3个100行的记录的表连接起来产生的笛卡尔积就非常巨大, 就有100×100×100=一百万行数据，所以再连接的时候加上过滤条件是特别有必要的。如果内连接后面跟有    where，那么where也会参与到内连接的过程中。

有内连接就会有外连接,在开发中一般比较常用是外连接, 也就是left join、right join 、inner join。这里我们再复习以下这些外连接的语义:	

> LEFT JOIN的标准写法:  SELECT * FROM t1 LEFT [OUTER] JOIN t2 ON 连接条件 where [普通过滤条件]  , 一般我们都省略OUTER,写不写都无所谓，语义为取出t1的所有记录和t2符合连接条件的记录,t1中和t2无法匹配的记录, 填充NULL。为了行文方便, 这里我们准备两张表: 

```mysql
CREATE TABLE `score`  (
  `id` int(11) NOT NULL COMMENT '唯一标识',
  `coursename` varchar(255)  COMMENT '课程名字',
  `score` int(255) NULL DEFAULT NULL COMMENT '成绩',
  `number` varchar(255) DEFAULT NULL COMMENT '学号',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;

CREATE TABLE `student`  (
  `id` int(11) NOT NULL COMMENT '唯一标识',
  `name` varchar(255)  COMMENT '姓名',
  `number` varchar(255)  COMMENT '学号',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;
```

> SELECT *  FROM student s1 left join  score s2 on s1.number = s2.number 

最后的结果: 

![](https://p.pstatp.com/origin/pgc-image/eab13dae8ef74366a789fd3c270dfac9)

由于王五同学在成绩表里没有匹配的成绩,但是还是出现在了结果集中，补上了NULL。

> 右连接的标准写法: SELECT * FROM t1 RIGHT [OUTER] JOIN t2 ON 连接条件 where [普通过滤条件] ，语义为取出t2的所有记录和t1符合连接条件的记录,t2中和t1无法匹配的记录, 填充NULL。为了行文方便.
>
> 示例: 

```mysql
SELECT s1.name,s2.coursename ,s2.score,s2.number FROM student s1 right  join  score s2 on s1.number = s2.number 
```

 ![](https://p.pstatp.com/origin/pgc-image/d1d7efbfa5af4fcc9ee74fed6dbcd12e)

   如果我们只是想看两表相互匹配的记录呢, 由此我们就引出了INNER JOIN, 这其实是上面介绍的内连接的另一种写法, 两表在连接过程中不符合on中的条件的不会出现在结果集中, 写法如下: 

```mysql
SELECT * FROM t1 [INNER | CROSS] JOIN t2 [ON 连接条件] [WHERE 普通过滤条件];
```

下面三种写法是互相等价的: 

```mysql
SELECT * FROM t1 join t2;
SELECT * FROM t1 inner join t2 (这是最常见的形式)
SELECT * FROM t1 CROSS JOIN t2;
SELECT * FROM t1,t2
```

粗略的说, 我们可以将“连接”理解为胶水,因为它可以将两张表并成一张表。

- 再是group by

>经过上面的连接我们已经得到了一张新的结果集, 或者说是一张新的表格。现在如果我们希望求出每个人的总成绩呢，由此我们就引出了group by。

```mysql
SELECT number,sum(score) FROM score GROUP BY number 
```

score就会切割成下面这样: 

![](https://p.pstatp.com/origin/pgc-image/2d87bd5c1cf84764800d01bac679865a)

有的时候一个列涵盖的太大，我们希望按这一列分组之后, 每一组再进行分组，就比如说上面, 假设我们为成绩表添加一个字段attribute, 标识这门课程属于理科是文科，那么上面的001组也就可以被拆为两组，如下图所示: 

![](https://p.pstatp.com/origin/pgc-image/3d3162960b3a421a89d00a4d1c2fe375)

 我们的SQL改写成下面这样就可以实现再分组的效果: 

```
SELECT number,sum(score) FROM score GROUP BY number,attribute
```

一般来说主流的数据库是要求在使用group by 之后,只允许出现分组列的, 原因也很简单, 非分组列有好几个，显示出来的话显示哪个呢？那么有人就会问了,假如我select的非分组列和分组列一样，刚好都是相同的呢，这样你是不是没有选择困难症了，对此MYSQL的设计人员认为你说的有道理，他们推出了ONLY_FULL_GROUP_BY模式，在5.7.x版本以上默认开启,允许SELECT后出现非分组列。

- 再是having

>having的作用点一是分组列，二是作用于分组的聚集函数。
>
>where 先于having执行。

- 再是order by

> 上面的每一步都会得到虚拟表,在这一步对上面的虚拟表进行排序。

- 再是select 和 distinct

> SELECT 可以构造出新的列, 所以order by用的不是虚拟表的列, 用的是select 构造出来的列进行排序,那么有的时候order by在SELECT后面执行。

综上所说一个SQL的执行顺序大致是下面这样的: 

![](https://p.pstatp.com/origin/pgc-image/08c3c74b45724fcfb6b23788b5ec3a1d)

### 子查询再学习

 这里将子查询的几种形式汇总以下。

- 双列子查询

>  与之相对的是单列子查询，我们通常会写SELECT * FROM Student where name in (’张三‘, 李四)。
>
> 但如果我们有两列呢,  找出学号是001，成绩是80的记录，我们可以这样写: 
>
> SELECT * FROM SCORE WHERE  (number,score) in ( SELECT '001', '80') 

- EXISTS 和 NOT EXISTS

> 有的时候外层并不关心子查询中的结果是什么, 而只关心子查询的结果集是不是空集。这时我们就用到了EXISTS 和 NOT EXISTS
>
> EXISTS(SELECT ... ) 当子查询结果集不是空集时表达式为真
>
>  NOT EXISTS 当子查询结果集是空集时结果为真

- ANY/SOME

> ANY 是任意一个, 语法形式:   列   comparison_operator   ANY/SOME(子查询)  comparison_operator 是操作符，比如大于、等于。
>
> 只有列和子查询做comparison_operator匹配，那么整个表达式都成立。

- ALL

> ALL  于ANY相反, 要求全部匹配。

## 语句书写标准顺序

```mysql
SELECT DISTINCT 

FROM 表名

WHERE

GROUP BY

HAVING 

ORDER BY 
```

这是SQL语法的标准顺序, 书写SQL必须严格按照此顺序写, 不然就会报SQL语法错误。

## 参考资料

- 步步深入：MySQL架构总览->查询执行流程->SQL解析顺序     https://www.cnblogs.com/annsshadow/p/5037667.html

- MySQL 是怎样使用的: 从零蛋开始学习MySQL 掘金小册