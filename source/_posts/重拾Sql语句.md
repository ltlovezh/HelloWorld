---
title: 重拾Sql语句
date: 2017-09-16 16:19:01
tags:
	- sql
categories:
 	- 数据库
---
## 基础的SQL
基本的SQL语句可以概括为以下伪代码：
``` sql
SELECT 
DISTINCT <select_list> 
FROM <left_table> 
<join_type> JOIN <right_table> 
ON <join_condition> 
WHERE <where_condition> 
GROUP BY <group_by_list> 
HAVING <having_condition> 
ORDER BY <order_by_condition> 
LIMIT <limit_number>
```
搞清楚上面各个伪代码的执行顺序，对我们写SQL有很大的帮助。
<!-- more -->

1. 首先是处理`FROM <left_table> <join_type> JOIN <right_table> ON <join_condition> `，对两张表根据`ON`操作符指定的关联条件，合并成一个临时表A。
>其中，`<join_type>`包括：`INNER JOIN`、`Full outer join`、`LEFT OUTER JOIN`、`RIGHT OUTER JOIN`、`CROSS JOIN`。
>1. `INNER JOIN`表示合并的临时表A中，仅包含符合`ON`条件的记录。
>2. `LEFT OUTER JOIN`表示合并的临时表A中，不仅包含符合`ON`条件的记录，还会把`left_table`中剩余的记录也保存在临时表，同时将对应的`right_table`中的所有列字段赋值为NULL。
>3. `RIGHT OUTER JOIN`表示合并的临时表A中，不仅包含符合`ON`条件的记录，还会把`right_table`中剩余的记录也保存在临时表A，同时将对应的`left_table`中的所有列字段赋值为NULL。
>4. `Full outer join`表示合并的临时表A中，不仅包含符合`ON`条件的记录，还会把`left_table`和`right_table`中剩余的记录也保存在临时表A，同时将对应的另外一张表的列字段赋值为NULL。其实就是两张表的交集。
>5. `CROSS JOIN`表示两张表的笛卡尔积，一般很少使用。
> 关于`JOIN`的各种使用，可以参考[图解SQL的JOIN](https://coolshell.cn/articles/3463.html)
2. 第一步把两张表合并到了一张临时表A，接下来是对临时表A处理`WHERE <where_condition>`指定的过滤条件，删除一些不符合条件的记录，得到临时表B。
3. 上一步得到了筛选记录后的临时表B，接下来针对临时表B，根据`GROUP BY <group_by_list>`指定的列字段，进行分组操作。然后根据`HAVING <having_condition>`指定的条件对分组进行过滤，得到临时表C。这里HAVING指定的条件只能包含分组字段，或者其他列字段的聚合函数。
4. 上一步得到了分组后的临时表C，接下来就是根据`SELECT 
DISTINCT <select_list>`规定的字段，选出仅包含指定列字段的临时表D，如果指定了`DISTINCT`，那么还要把重复的行记录过滤掉。
5. 上一步得到了筛选列字段后的临时表D，然后就是根据`ORDER BY <order_by_condition>`指定的列字段，对临时表D的所有行记录进行排序，得到临时表E。
6. 最后，根据`LIMIT <limit_number>`指定的条件，从临时表E中，摘录出指定数量的行记录，生成最终的结果表。
>limit的用法是：limit n,m，表示从第n条记录开始选择m条记录。一般可用于列表分页，对于小数据，使用limit没有任何问题。但是当数据量非常大的时候，使用limit是非常低效的。因为limit的机制是每次都从头开始扫描，如果需要从第50万行开始，读取10条数据，那么就需要先扫描定位到第50万行，然后再读取10条记录，而扫描是一个非常低效的过程。

一般情况下，基本的SQL语句都可以按照上面6个步骤进行分析。

## SQL函数
上面介绍了基本SQL语句的内部执行顺序，下面我们看一些常用的SQL函数，这些函数的使用能帮助我们解决一些复杂的SQL问题。

### `Case When Then`
Case函数很像`if else`语句，可以进行多条件判断。Case具有两种格式：简单Case函数和Case搜索函数。

#### 简单Case函数
``` sql
CASE sex
    WHEN '1' THEN '男'
    WHEN '2' THEN '女'
ELSE '其他' END
```
#### Case搜索函数
``` sql
CASE WHEN sex = '1' THEN '男'
     WHEN sex = '2' THEN '女'
ELSE '其他' END

SELECT name, score, 
(CASE WHEN score < 60 THEN '不及格' 
WHEN score BETWEEN 60 AND 90 THEN '良好' 
WHEN score > 90 THEN '优秀' END) as level
FROM student
```

上面两种方式，可以实现相同的功能。简单Case函数的写法相对比较简洁，但是和Case搜索函数相比，功能方面会有些限制，比如写判断式。

#### 具体案例
有如下一个数据库表，标示了各个国家的人口，要求求出亚洲和美洲的人口总数：

| country   | people  |   
| --------  | :-----: | 
| brazil    | 100     |  
| china     | 100     |  
| india     | 100     |   
| mexico    | 100     |   
| usa       | 100     |  
| england   | 100     |

我们只要把属于亚洲和美洲的国家的人口累加，就可以了。所以sql语句如下所示：
``` sql
SELECT 
(case country when 'china' then "asia"
			 when 'india' then "asia"
			 when 'mexico' then "america"
			 when 'usa' then "america"
			 when 'brazil' then "america"
			 else
				"other"
			 end) as continent , sum(people) as num FROM leon.TableA
group by
(case country when 'china' then "asia"
			 when 'india' then "asia"
			 when 'mexico' then "america"
			 when 'usa' then "america"
			 when 'brazil' then "america"
			 else
				"other"
			 end);
```

上述SQL语句首先把country分为亚洲和美洲两个维度，然后根据这个新维度进行分组，并使用聚合函数，求出各个大洲的总人口。
最后得出的结果表如下所示：

| continent | num       |   
| --------  | :-----:   | 
| america   | 300       |  
| asia      | 200       |  
| other     | 100       |


---

后续使用到继续补充...

## 参考文档

* [图解SQL的JOIN](https://coolshell.cn/articles/3463.html)
* [SQL逻辑查询语句执行顺序](http://www.jellythink.com/archives/924)
