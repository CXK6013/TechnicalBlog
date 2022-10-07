# LeetCode刷题四部曲之SQL篇(一)

> 这个系列也其实想开很久了, 但是每次想要开的时候，总有别的文章挡在前面。这个系列大致会分成四个系列:
>
> - 算法
> - 数据库
> - shell
> - 并发

## 前言

这周先开个头，看看能不能做到每日一题，这个系列会放在GitHub上。前文我们已经重新梳理了对SQL模型的理解, 这里我们刷题，增进一下对SQL的理解。在实践中丰富我们的SQL模型，重在体会思想。尽量直接在LeetCode提交SQL, 盲写。

## 今天的题目

### 176. Second Highest Salary  第二高的薪水

编写一个 SQL 查询，获取 Employee 表中第二高的薪水（Salary） 。

| Id   | Salary |
| ---- | ------ |
| 1    | 100    |
| 2    | 200    |
| 3    | 300    |


例如上述 Employee 表，SQL查询应该返回 200 作为第二高的薪水。如果不存在第二高的薪水，那么查询应返回 null。

| SecondHighestSalary |
| ------------------- |
| 200                 |



### 第一种解法解读

解法思考，这道题目用程序做的话，无法是排个序然后去数组下标为1的元素即可。这是程序的思路，那SQL呢，但是SQL中没有下标，但是我们有分页函数。

所以我的第一版提交长这个样子:

```mysql
SELECT DISTINCT Salary  AS SecondHighestSalary FROM Employee ORDER BY Salary  DESC LIMIT 1, OFFSET 1
```

但是答案是不对的：

![](https://imglf4.lf127.net/img/SVJIRzhzT1h0YnIxRDk0RDhsRXBocHZ1dEV1WUR0UnFsZXlhczc2RUxHbElPY0hOcUxWL1dnPT0.png)

我输出的是个空？不是NULL，这看来又增进了我对MySQL的理解啊, 只有一条的话，跳过这条，取后面的就是空而不是NULL。那这个是为什么呢。我们来用命令行来看一下这个结果, 我装MySQL的时候忘记配置环境变量，但是现在还不想配置，但是可以通过图形化工具将命令行带出来:

1.![](https://imglf6.lf127.net/img/MkdLU1BCdEFJMHoxTkJ1clNWbStDVG5hWjFqakprQXlVUFQ0OGZORWpWUGpxdElGWE5ENklBPT0.png)

2. ![](https://imglf5.lf127.net/img/VWNSY3kwMmsrN3EzYkZFa2NOc1FtWDkxVnZpdXNNeUxxWFV4MzgyaVZyR3VpaTRwVnd3TGtBPT0.png)

   3 . 然后我们查询student表中不存在的记录，看看返回什么:

![](https://imglf5.lf127.net/img/RERLZG1YTWpvc1BkbkRQenh1c0drRm9NZndXNm8yWDZSd0tPS29DT3U5bWt0Nk5vVWhSM3RBPT0.png)

我们在SQL查询模型和子查询再学习中将SQL的每一个执行过程会输出一个虚拟表，最终得到的也是一个虚拟表, 这里我们在复习一下:

![img](https://p.pstatp.com/origin/pgc-image/08c3c74b45724fcfb6b23788b5ec3a1d)

所以我们在student查询一个不存在的记录，返回为空集，这个是在预期的，符合我们的SQL查询模型的。那如果我们将这个查询当作一个表达式来使用呢，也就是像下面这样:

```mysql
SELECT (SELECT NAME FROM Student where id = 3) name;
```

输出了为NULL:

![](https://imglf3.lf127.net/img/bFdhc3NZWFdjUGZYclBtQjRISm9MVkQ2NXhmOTV1aWIwNTlhcVpEYXJId21EbzFkS1R2SDFRPT0.png)

在oracle下其实要这么写，才能过关:

```sql
SELECT (SELECT NAME FROM Student where id = 3) name FROM Dual
```

那为什么这个会输出NULL呢，原因在于此时SELECT (SELECT NAME FROM Student where id = 3) 被当作表达式解析求值，对于表达式求值来说, 要么有值，要么为NULL。 上面的那种写法被称为标量子查询，

明白了这个，我们就可以将SQL改写为:

```sql
SELECT
(SELECT distinct  salary  AS SecondHighestSalary   FROM  Employee  ORDER BY  salary DESC  LIMIT 1 OFFSET 1) AS
SecondHighestSalary
```

那这个distinct怎么理解, 不加distinct又报错, 为什么加上了distinct就可以,  出现这种问题来源于我们错误的SQL查询模型，在上面的SQL查询模型中，distinct在SELECT之后执行, 事实上他应该在SELECT之前执行, distinct对结果集去重, 然后SELECT取出结果集。所以上面我们的SQL查询模型可以修正为下面这样:

![](https://imglf4.lf127.net/img/WlJPRVhmaUlHTzlRNDVTK0tnTEtvSGFONXRLSnNseXRnVFduYXV3ZkxERkJxSndaWkdaWGRBPT0.png)

有了这个新的模型，我们就不难理解这个了这个SQL查询模型了，先按照salary倒序排, 然后再去重(防止表里只出现两条相同的最大的), 取第二条。 这里我们在给出等价的oracle写法:

```sql
SELECT 
(SELECT DISTINCT(salary) AS SecondHighestSalary FROM 
(SELECT  salary,ROWNUM AS rowno FROM Employee WHERE ROWNUM <= 2) table_alias
WHERE table_alias.rowno > 1) AS SecondHighestSalary FROM DUAL
```

### 第二种解法解读

第一种解法相对来说更贴进去程序语言像Java之类的, 我们可以其实换一种思路来解决这个问题, 借助于SQL的强大统计特性, 也就是先找出最大的salary, 然后再从表里面小于这个最大的salary的，这也就是第二大的salary。这种思路更贴向SQL, 而且跨数据库, 我们来直接写一下: 

```sql
SELECT max(salary) AS SecondHighestSalary    FROM Employee WHERE salary <  (SELECT max(salary) FROM Employee)
```

但这种解法并没有第一种快。

### 标量子查询简介[1]

这里关于标量子查询全部引自亚马逊SQL参考, 文末放有链接。

标量子查询是圆括号中的常规 SELECT 查询，仅返回一个值：带有一个列的一行。将执行此查询，返回值将在外部查询中使用。如果子查询返回零行，则子查询表达式的值为 null。如果它返回多行，则 Amazon Redshift 将返回错误。子查询可引用父查询中的变量，这将在子查询的任何一次调用中充当常量。

标量子查询在下列情况下是无效表达式：

- 作为表达式的默认值
- 在 GROUP BY 和 HAVING 子句中

以下子查询计算 2008 年全年的每笔销售支付的平均价格，然后外部查询使用输出中的值来比较每个季度每笔销售的平均价格：

```
select qtr, avg(pricepaid) as avg_saleprice_per_qtr,
```

```sql
select qtr, avg(pricepaid) as avg_saleprice_per_qtr,
(select avg(pricepaid)
from sales join date on sales.dateid=date.dateid
where year = 2008) as avg_saleprice_yearly
from sales join date on sales.dateid=date.dateid
where year = 2008
group by qtr
order by qtr;
qtr  | avg_saleprice_per_qtr | avg_saleprice_yearly
-------+-----------------------+----------------------
1     |                647.64 |               642.28
2     |                646.86 |               642.28
3     |                636.79 |               642.28
4     |                638.26 |               642.28
(4 rows)
```

## 参考资料

[1] 标量子查询   https://docs.aws.amazon.com/zh_cn/redshift/latest/dg/r_scalar_subqueries.html





