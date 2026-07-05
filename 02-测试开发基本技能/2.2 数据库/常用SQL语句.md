---
title: "常用 SQL 语句"
tags:
  - 速查
  - SQL
  - 数据库
created: "2026-07-06"
updated: "2026-07-06"
---

# 常用 SQL 语句

> 以 `student`（学生信息表）和 `score`（学生成绩表）为示例场景，整理测试开发中最常用的 DQL 查询操作，包括条件查询、排序、分页、聚合和连接查询。

## 示例表结构

```sql
-- 学生信息表
CREATE TABLE student (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL COMMENT '姓名',
    number VARCHAR(20) NOT NULL COMMENT '学号',
    sex VARCHAR(10) COMMENT '性别',
    class VARCHAR(50) COMMENT '班级'
);

-- 学生成绩表
CREATE TABLE score (
    stu_id INT PRIMARY KEY COMMENT '学生ID',
    Math INT COMMENT '数学成绩',
    English INT COMMENT '英语成绩',
    Language INT COMMENT '语文成绩',
    create_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间'
);
```

### student 表数据

```
+----+-----------+--------------+-----+------------------+
| id | name      | number       | sex | class            |
+----+-----------+--------------+-----+------------------+
|  1 | 张三      | 200920205104 | 男  | 软件1班          |
|  2 | 王五      | 200920205103 | 男  | 软件1班          |
|  3 | 李四      | 200920205202 | 男  | 软件2班          |
|  4 | 陈浩      | 200920205201 | 男  | 软件2班          |
|  5 | 姗姗      | 201303051487 | 女  | 酒店管理1班      |
+----+-----------+--------------+-----+------------------+
5 rows in set (0.00 sec)
```

### score 表数据

```
+--------+------+---------+----------+---------------------+
| stu_id | Math | English | Language | create_time         |
+--------+------+---------+----------+---------------------+
|      1 |   95 |      90 |       80 | 2020-08-22 12:20:45 |
|      2 |   92 |      77 |       79 | 2020-08-22 12:20:45 |
|      3 |   59 |      70 |       88 | 2020-08-22 12:20:45 |
|      4 |  100 |      96 |       99 | 2020-08-22 12:20:45 |
|      5 |   76 |      80 |       66 | 2020-08-22 12:20:45 |
+--------+------+---------+----------+---------------------+
5 rows in set (0.00 sec)
```

## 一、条件查询（WHERE、LIKE）

### 1.1 WHERE 条件查询

```sql
-- 统计数学成绩及格的学生ID
mysql> SELECT stu_id, Math FROM score WHERE Math >= 60;

-- 使用 BETWEEN 查询数学成绩在 60~100 之间的学生
mysql> SELECT stu_id, Math FROM score WHERE Math BETWEEN 60 AND 100;
```

### 1.2 AND 多条件查询

```sql
-- 统计语文和数学成绩都大于 80 分的学生ID
mysql> SELECT stu_id, Language, Math FROM score WHERE Math > 80 AND Language > 80;
```

### 1.3 LIKE 模糊查询

```sql
-- 查询不是 2009 年入学的学生姓名和学号
mysql> SELECT name, number FROM student WHERE number NOT LIKE '2009%';

-- 查询就读软件专业的学生名字和班级
mysql> SELECT name, class FROM student WHERE class LIKE '%软件%';
```

## 二、排序查询（ORDER BY）

```sql
-- 按学号降序查询学生的姓名和学号
mysql> SELECT name, number FROM student ORDER BY number DESC;

+-----------+--------------+
| name      | number       |
+-----------+--------------+
| 姗姗      | 201303051487 |
| 李四      | 200920205202 |
| 陈浩      | 200920205201 |
| 张三      | 200920205104 |
| 王五      | 200920205103 |
+-----------+--------------+
5 rows in set (0.00 sec)
```

```sql
-- 按学号降序查询男学生的姓名和学号
mysql> SELECT name, number FROM student WHERE sex='男' ORDER BY number DESC;

+-----------+--------------+
| name      | number       |
+-----------+--------------+
| 李四      | 200920205202 |
| 陈浩      | 200920205201 |
| 张三      | 200920205104 |
| 王五      | 200920205103 |
+-----------+--------------+
4 rows in set (0.00 sec)
```

## 三、分页查询（LIMIT、OFFSET）

```sql
-- 从结果集中只显示前 2 条数据
mysql> SELECT name, number FROM student ORDER BY number DESC LIMIT 2;

+-----------+--------------+
| name      | number       |
+-----------+--------------+
| 姗姗      | 201303051487 |
| 李四      | 200920205202 |
+-----------+--------------+
2 rows in set (0.00 sec)
```

```sql
-- 查询第二页的数据（每页 2 条）
mysql> SELECT name, number FROM student ORDER BY number DESC LIMIT 2 OFFSET 2;

+-----------+--------------+
| name      | number       |
+-----------+--------------+
| 陈浩      | 200920205201 |
| 张三      | 200920205104 |
+-----------+--------------+
2 rows in set (0.01 sec)
```

## 四、聚合查询（AVG、SUM、MIN、MAX）

通常，使用聚合查询时，我们应该给列名设置一个别名，便于处理结果。

### 4.1 COUNT 统计总数

```sql
-- 统计全年级的学生总数
mysql> SELECT COUNT(*) "学生数量" FROM student;

+--------------+
| 学生数量     |
+--------------+
|            5 |
+--------------+
1 row in set (0.00 sec)
```

### 4.2 AVG 平均值

```sql
-- 统计全年级数学的平均分
mysql> SELECT AVG(Math) AS "数学平均分" FROM score;

+-----------------+
| 数学平均分      |
+-----------------+
|         84.4000 |
+-----------------+
1 row in set (0.00 sec)
```

### 4.3 子查询比较

```sql
-- 查找低于年级数学平均分的学生ID和数学成绩
mysql> SELECT stu_id, Math FROM score WHERE Math < (
    SELECT AVG(Math) AS "数学平均分" FROM score
);

+--------+------+
| stu_id | Math |
+--------+------+
|      3 |   59 |
|      5 |   76 |
+--------+------+
2 rows in set (0.00 sec)
```

### 4.4 SUM 求和

```sql
-- 统计学生ID=1的总成绩
mysql> SELECT SUM(Math + English + Language) FROM score WHERE stu_id = 1;

+----------------------------+
| sum(Math+English+Language) |
+----------------------------+
|                        265 |
+----------------------------+
1 row in set (0.00 sec)
```

### 4.5 GROUP BY 分组聚合

```sql
-- 统计每个学生的总成绩和平均分
mysql> SELECT
    stu_id,
    SUM(Math + English + Language) AS "总分",
    SUM(Math + English + Language) / 3 AS "平均分"
FROM score
GROUP BY stu_id;

+--------+--------+-----------+
| stu_id | 总分   | 平均分    |
+--------+--------+-----------+
|      1 |    265 |   88.3333 |
|      2 |    248 |   82.6667 |
|      3 |    217 |   72.3333 |
|      4 |    295 |   98.3333 |
|      5 |    222 |   74.0000 |
+--------+--------+-----------+
5 rows in set (0.01 sec)
```

### 4.6 HAVING 分组过滤

```sql
-- 查询总成绩大于 260 分的学生ID
mysql> SELECT
    stu_id,
    SUM(Math + English + Language) AS "总分"
FROM score
GROUP BY stu_id
HAVING SUM(Math + English + Language) > 260;

+--------+--------+
| stu_id | 总分   |
+--------+--------+
|      1 |    265 |
|      4 |    295 |
+--------+--------+
2 rows in set (0.00 sec)
```

## 五、连接查询

连接查询是另一种类型的多表查询。连接查询对多个表进行 JOIN 运算，简单地说，就是先确定一个主表作为结果集，然后，把其他表的行有选择性地“连接”在主表结果集上。

注意 `INNER JOIN` 查询的写法是：

1. 先确定主表，仍然使用 `FROM <表1>` 的语法；
2. 再确定需要连接的表，使用 `INNER JOIN <表2>` 的语法；
3. 然后确定连接条件，使用 `ON <条件...>`，这里的条件是 `score.stu_id = student.id`，表示 `score` 表的 `stu_id` 列与 `student` 表的 `id` 列相同的行需要连接；
4. 可选：加上 `WHERE` 子句、`ORDER BY` 等子句。

### 5.1 INNER JOIN 内连接

```sql
-- 列出数学成绩及格的学生姓名、学号和数学成绩
mysql> SELECT name, number, Math
FROM score
INNER JOIN student ON score.stu_id = student.id
WHERE Math >= 60;

+-----------+--------------+------+
| name      | number       | Math |
+-----------+--------------+------+
| 张三      | 200920205104 |   95 |
| 王五      | 200920205103 |   92 |
| 陈浩      | 200920205201 |  100 |
| 姗姗      | 201303051487 |   76 |
+-----------+--------------+------+
4 rows in set (0.00 sec)
```

### 5.2 子查询 + IN

```sql
-- 查询总成绩大于 260 分的学生ID
mysql> SELECT
    stu_id,
    SUM(Math + English + Language) AS "总分"
FROM score
GROUP BY stu_id
HAVING SUM(Math + English + Language) > 260;

+--------+--------+
| stu_id | 总分   |
+--------+--------+
|      1 |    265 |
|      4 |    295 |
+--------+--------+
2 rows in set (0.00 sec)

-- 查询出学生成绩总分大于 260 分的名字
mysql> SELECT id, name FROM student
WHERE id IN (
    SELECT stu_id FROM score
    GROUP BY stu_id
    HAVING SUM(Math + English + Language) > 260
);

+----+-----------+
| id | name      |
+----+-----------+
|  1 | 张三      |
|  4 | 陈浩      |
+----+-----------+
2 rows in set (0.00 sec)
```

| 场景   | 示例 SQL                                             | 用途          |
| ---- | -------------------------------------------------- | ----------- |
| 数据断言 | `SELECT Math FROM score WHERE stu_id = 1`          | 验证测试后数据库状态  |
| 范围校验 | `SELECT * FROM score WHERE Math BETWEEN 0 AND 100` | 检查成绩是否在合理区间 |
| 异常排查 | `SELECT * FROM student WHERE class IS NULL`        | 发现缺失或异常数据   |
| 聚合验证 | `SELECT AVG(Math) FROM score`                      | 验证统计接口计算结果  |
| 多表关联 | `INNER JOIN student ...`                           | 验证关联数据一致性   |

## 关联文档

- [[数据库基础知识]]
