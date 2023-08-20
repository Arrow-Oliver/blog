---
title: "日常练习SQL"
date: 2023-08-20
draft: false
categories: [数据库]
tags: [MySQL]
card: false
weight: 0
---

# SQL执行顺序：

1. from 
2. on 
3. join 
4. where 
5. group by(开始使用select中的别名，后面的语句中都可以使用)
6. avg,sum.... 
7. having 
8. select 
9. distinct 
10. order by
11. limit 

# 1. 条件分支

**case when**

语法：

~~~mysql
CASE WHEN (条件1) THEN 结果1
	   WHEN (条件2) THEN 结果2
	   ...
	   ELSE 其他结果 END
~~~

举例：

​	假设有一个学生表 `student`，包含以下字段：`name`（姓名）、`age`（年龄）。请你编写一个 SQL 查询，将学生按照年龄划分为三个年龄等级（age_level）：60 岁以上为 "老同学"，20 岁以上（不包括 60 岁以上）为 "年轻"，20 岁及以下、以及没有年龄信息为 "小同学"。

​	返回结果应包含学生的姓名（name）和年龄等级（age_level），并按姓名升序排序。

结果：

~~~mysql
SELECT
  name,
  CASE
    WHEN (age > 60) THEN '老同学'
    WHEN (age > 20) THEN '年轻'
    ELSE '小同学'
  END AS age_level
FROM
  student
ORDER BY
  name asc;
~~~

# 2. 时间函数

使用时间函数获取当前日期、当前日期时间和当前时间：

~~~mysql
-- 获取当前日期
SELECT DATE() AS current_date;

-- 获取当前日期时间
SELECT DATETIME() AS current_datetime;

-- 获取当前时间
SELECT TIME() AS current_time;
~~~

查询结果：

| current_date | current_datetime    | current_time |
| ------------ | ------------------- | ------------ |
| 2023-08-01   | 2023-08-01 14:30:00 | 14:30:00     |

# 3. 开窗函数

## sum over

该函数用法为：

```sql
SUM(计算字段名) OVER (PARTITION BY 分组字段名)
```

举例：

​		假设有一个学生表 `student`，包含以下字段：`id`（学号）、`name`（姓名）、`age`（年龄）、`score`（分数）、`class_id`（班级编号）。

​		请你编写一个 SQL 查询，返回每个学生的详细信息（字段顺序和原始表的字段顺序一致），并计算每个班级的学生平均分（
（class_avg_score）。

答案：

~~~mysql
SELECT
  id,
  name,
  age,
  score,
  class_id,
  AVG(score) OVER (
    PARTITION BY
      class_id
  ) AS class_avg_score
FROM
  student;
~~~

## sum over order by

示例用法如下：

```sql
SUM(计算字段名) OVER (PARTITION BY 分组字段名 ORDER BY 排序字段 排序规则)
```

举例：

​		假设有一个学生表 `student`，包含以下字段：`id`（学号）、`name`（姓名）、`age`（年龄）、`score`（分数）、`class_id`（班级编号）。

​		请你编写一个 SQL 查询，返回每个学生的详细信息（字段顺序和原始表的字段顺序一致），并且按照分数升序的方式累加计算每个班级的学生总分（class_sum_score）。

答案：

~~~sql
SELECT
  id,
  name,
  age,
  score,
  class_id,
  SUM(score) OVER (
    PARTITION BY
      class_id
    ORDER BY
      score ASC
  ) AS class_sum_score
FROM
  student;
~~~

运行结果：

| id   | name   | age  | score | class_id | class_sum_score |
| :--- | :----- | :--- | :---- | :------- | :-------------- |
| 1    | 鸡哥   | 25   | 2.5   | 1        | 2.5             |
| 2    | 绿箭   | 18   | 400   | 1        | 402.5           |
| 4    | 摸FISH |      | 360   | 2        | 360             |
| 3    | 热dog  | 40   | 600   | 2        | 960             |
| 5    | 李阿巴 | 19   | 120   | 3        | 120             |
| 6    | 老李   | 56   | 500   | 3        | 620             |
| 8    | 王加瓦 | 23   | 0     | 4        | 0               |
| 7    | 李变量 | 24   | 390   | 4        | 390             |
| 9    | 赵派森 | 80   | 600   | 4        | 990             |
| 10   | 孙加加 | 60   | 100.5 | 5        | 100.5           |

与**sum over** 函数比较

结果为：

~~~mysql
SELECT
  id,
  name,
  age,
  score,
  class_id,
  SUM(score) OVER (
    PARTITION BY
      class_id
  ) AS class_sum_score
FROM
  student;
~~~

| id   | name   | age  | score | class_id | class_sum_score |
| :--- | :----- | :--- | :---- | :------- | :-------------- |
| 1    | 鸡哥   | 25   | 2.5   | 1        | 402.5           |
| 2    | 绿箭   | 18   | 400   | 1        | 402.5           |
| 3    | 热dog  | 40   | 600   | 2        | 960             |
| 4    | 摸FISH |      | 360   | 2        | 960             |
| 5    | 李阿巴 | 19   | 120   | 3        | 620             |
| 6    | 老李   | 56   | 500   | 3        | 620             |
| 7    | 李变量 | 24   | 390   | 4        | 990             |
| 8    | 王加瓦 | 23   | 0     | 4        | 990             |
| 9    | 赵派森 | 80   | 600   | 4        | 990             |
| 10   | 孙加加 | 60   | 100.5 | 5        | 100.5           |

> 不使用order by 将计算的最终结果放入每个数据中；而使用order by 后，可以实现累加的效果。

## rank

Rank 开窗函数的语法如下：

```sql
RANK() OVER (
  PARTITION BY 列名1, 列名2, ... -- 可选，用于指定分组列
  ORDER BY 列名3 [ASC|DESC], 列名4 [ASC|DESC], ... -- 用于指定排序列及排序方式
) AS rank_column
```

其中，`PARTITION BY` 子句可选，用于指定分组列，将结果集按照指定列进行分组；`ORDER BY` 子句用于指定排序列及排序方式，决定了计算 Rank 时的排序规则。`AS rank_column` 用于指定生成的 Rank 排名列的别名。

举例：

​		假设有一个学生表 `student`，包含以下字段：`id`（学号）、`name`（姓名）、`age`（年龄）、`score`（分数）、`class_id`（班级编号）。

​		请你编写一个 SQL 查询，返回每个学生的详细信息（字段顺序和原始表的字段顺序一致），并且按照分数降序的方式计算每个班级内的学生的分数排名（ranking）。

结果：

~~~mysql
SELECT
  id,
  name,
  age,
  score,
  class_id,
  RANK() OVER (
    PARTITION BY
      class_id
    ORDER BY
      score DESC
  ) AS ranking
FROM
  student;
~~~

| id   | name   | age  | score | class_id | ranking |
| :--- | :----- | :--- | :---- | :------- | :------ |
| 2    | 鱼皮   | 18   | 400   | 1        | 1       |
| 1    | 绿箭   | 25   | 2.5   | 1        | 2       |
| 3    | 热dog  | 40   | 600   | 2        | 1       |
| 4    | 摸FISH |      | 360   | 2        | 2       |
| 6    | 老李   | 56   | 500   | 3        | 1       |
| 5    | 李阿巴 | 19   | 120   | 3        | 2       |
| 9    | 赵派森 | 80   | 600   | 4        | 1       |
| 7    | 李变量 | 24   | 390   | 4        | 2       |
| 8    | 王加瓦 | 23   | 0     | 4        | 3       |
| 10   | 孙加加 | 60   | 100.5 | 5        | 1       |

## row_number

Row_Number 开窗函数的语法如下（几乎和 Rank 函数一模一样）：

```sql
ROW_NUMBER() OVER (
  PARTITION BY column1, column2, ... -- 可选，用于指定分组列
  ORDER BY column3 [ASC|DESC], column4 [ASC|DESC], ... -- 用于指定排序列及排序方式
) AS row_number_column
```

其中，`PARTITION BY`子句可选，用于指定分组列，将结果集按照指定列进行分组。`ORDER BY` 子句用于指定排序列及排序方式，决定了计算 Row_Number 时的排序规则。`AS row_number_column` 用于指定生成的行号列的别名。

> 它与之前讲到的 Rank 函数，Row_Number 函数为每一行都分配一个唯一的整数值，不管是否存在并列（相同排序值）的情况。每一行都有一个唯一的行号，从 1 开始连续递增。

## lag / lead

1）Lag 函数

Lag 函数用于获取 **当前行之前** 的某一列的值。它可以帮助我们查看上一行的数据。

Lag 函数的语法如下：

```sql
LAG(column_name, offset, default_value) OVER (PARTITION BY partition_column ORDER BY sort_column)
```

参数解释：

- `column_name`：要获取值的列名。
- `offset`：表示要向上偏移的行数。例如，offset为1表示获取上一行的值，offset为2表示获取上两行的值，以此类推。
- `default_value`：可选参数，用于指定当没有前一行时的默认值。
- `PARTITION BY`和`ORDER BY`子句可选，用于分组和排序数据。

2）Lead 函数

Lead 函数用于获取 **当前行之后** 的某一列的值。它可以帮助我们查看下一行的数据。

Lead 函数的语法如下：

```sql
LEAD(column_name, offset, default_value) OVER (PARTITION BY partition_column ORDER BY sort_column)
```

参数解释：

- `column_name`：要获取值的列名。
- `offset`：表示要向下偏移的行数。例如，offset为1表示获取下一行的值，offset为2表示获取下两行的值，以此类推。
- `default_value`：可选参数，用于指定当没有后一行时的默认值。
- `PARTITION BY`和`ORDER BY`子句可选，用于分组和排序数据。

举例：

​		假设有一个学生表 `student`，包含以下字段：`id`（学号）、`name`（姓名）、`age`（年龄）、`score`（分数）、`class_id`（班级编号）。

​		请你编写一个 SQL 查询，返回每个学生的详细信息（字段顺序和原始表的字段顺序一致），并且按照分数降序的方式获取每个班级内的学生的前一名学生姓名（prev_name）、后一名学生姓名（next_name）。

结果：

~~~mysql
SELECT
  id,
  name,
  age,
  score,
  class_id,
  LAG(name) over (
    PARTITION BY
      class_id
    ORDER BY
      score DESC
  ) as prev_name,
  LEAD(name) OVER (
    PARTITION BY
      class_id
    ORDER BY
      score DESC
  ) AS next_name
FROM
  student;
~~~

| id   | name   | age  | score | class_id | prev_name | next_name |
| :--- | :----- | :--- | :---- | :------- | :-------- | :-------- |
| 2    | 绿箭   | 18   | 400   | 1        |           | 鸡哥      |
| 1    | 鸡哥   | 25   | 2.5   | 1        | 鱼皮      |           |
| 3    | 热dog  | 40   | 600   | 2        |           | 摸FISH    |
| 4    | 摸FISH |      | 360   | 2        | 热dog     |           |
| 6    | 老李   | 56   | 500   | 3        |           | 李阿巴    |
| 5    | 李阿巴 | 19   | 120   | 3        | 老李      |           |
| 9    | 赵派森 | 80   | 600   | 4        |           | 李变量    |
| 7    | 李变量 | 24   | 390   | 4        | 赵派森    | 王加瓦    |
| 8    | 王加瓦 | 23   | 0     | 4        | 李变量    |           |
| 10   | 孙加加 | 60   | 100.5 | 5        |           |           |













