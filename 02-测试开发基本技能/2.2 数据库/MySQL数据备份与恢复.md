---
title: "MySQL 数据备份与恢复"
tags:
  - 实战
  - MySQL
  - 数据库
  - 备份恢复
created: "2026-07-06"
updated: "2026-07-06"
---

# MySQL 数据备份与恢复

> 整理 MySQL 环境下快速复制表结构/数据、导出导入单表以及整库备份恢复的常用命令，特别说明开启 GTID 时的注意事项。

## 简介

在测试开发工作中，经常需要在测试前备份基线数据、测试后恢复环境，或者快速复制生产表到测试环境。MySQL 提供了 `CREATE TABLE ... LIKE`、`INSERT INTO ... SELECT`、`mysqldump` 等多种方式完成这些操作。当数据库实例开启 GTID（Global Transaction Identifier）时，部分语句会受到限制，需要采用替代方案。

## 核心功能 / 机制

### 1. 快速复制表结构

`CREATE TABLE ... LIKE` 可以完整复制原表的表结构，包括字段、索引、主键、字符集等，但不复制数据。

```sql
mysql> CREATE TABLE user_backup LIKE user;
Query OK, 0 rows affected (0.03 sec)
```

### 2. 快速复制表数据

在两张表结构完全一致的情况下，使用 `INSERT INTO ... SELECT` 将数据从旧表复制到新表。

```sql
mysql> INSERT INTO user_backup SELECT * FROM user;
Query OK, 4 rows affected (0.01 sec)
Records: 4  Duplicates: 0  Warnings: 0
```

### 3. 整表导出 / 导入

使用 `mysqldump` 导出表结构和数据，使用 `mysql` 命令或 `source` 命令导入。

### 4. 整库导出 / 导入

`mysqldump` 同样可以导出整个数据库，再导入到目标库中。


## 常用操作

### 快速复制表的完整结构和数据

#### 为什么不直接用 `CREATE TABLE ... AS SELECT`

在 MySQL 5.7.x 中，如果实例开启了 GTID，执行以下语句会报错：

```sql
mysql> CREATE TABLE user_backup AS SELECT * FROM user;
ERROR 1786 (HY000): Statement violates GTID consistency: CREATE TABLE ... SELECT.
```

原因：`CREATE TABLE ... SELECT` 在一个事务中同时包含 DDL 和 DML，GTID 一致性检查不允许这种混合语句。参考：[https://www.unixso.com/MySQL/gtid_mode-enforce_gtid_consistency.html](https://www.unixso.com/MySQL/gtid_mode-enforce_gtid_consistency.html)

#### 方法一：`CREATE TABLE ... LIKE` + `INSERT INTO ... SELECT`

```sql
-- 第一步：只复制表结构，命名新表为 user_backup
mysql> CREATE TABLE user_backup LIKE user;
Query OK, 0 rows affected (0.03 sec)

-- 第二步：只复制表数据，且两张表结构一模一样
mysql> INSERT INTO user_backup SELECT * FROM user;
Query OK, 4 rows affected (0.01 sec)
Records: 4  Duplicates: 0  Warnings: 0
```

#### 方法二：`SHOW CREATE TABLE` + 手动建表 + `INSERT INTO ... SELECT`

```sql
-- 第一步：获取数据表的完整结构
mysql> SHOW CREATE TABLE user \G;

*************************** 1. row ***************************
       Table: user
Create Table: CREATE TABLE `user` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `created_at` datetime DEFAULT NULL,
  `disable` bit(1) NOT NULL,
  `email` varchar(255) DEFAULT NULL,
  `is_admin` bit(1) NOT NULL,
  `ldap_source` bigint(20) DEFAULT NULL,
  `name` varchar(255) NOT NULL,
  `nickname` varchar(255) DEFAULT NULL,
  `updated_at` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `UK_gj2fy3dcix7ph7k8684gka40c` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=43 DEFAULT CHARSET=utf8
1 row in set (0.00 sec)

ERROR:
No query specified
```

```sql
-- 第二步：修改 SQL 语句的表名，并执行 SQL 语句
mysql> CREATE TABLE `user_backup1` (
    ->   `id` bigint(20) NOT NULL AUTO_INCREMENT,
    ->   `created_at` datetime DEFAULT NULL,
    ->   `disable` bit(1) NOT NULL,
    ->   `email` varchar(255) DEFAULT NULL,
    ->   `is_admin` bit(1) NOT NULL,
    ->   `ldap_source` bigint(20) DEFAULT NULL,
    ->   `name` varchar(255) NOT NULL,
    ->   `nickname` varchar(255) DEFAULT NULL,
    ->   `updated_at` datetime DEFAULT NULL,
    ->   PRIMARY KEY (`id`),
    ->   UNIQUE KEY `UK_gj2fy3dcix7ph7k8684gka40c` (`name`)
    -> ) ENGINE=InnoDB AUTO_INCREMENT=43 DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.03 sec)

-- 第三步：INSERT INTO ... SELECT 语句来实现拷贝数据
mysql> INSERT INTO user_backup1 SELECT * FROM user;
Query OK, 4 rows affected (0.01 sec)
Records: 4  Duplicates: 0  Warnings: 0
```

### 导出单表

```bash
root@Master:/# mysqldump -uwise2c -p bldr user > user_bak.sql
Enter password:
Warning: A partial dump from a server that has GTIDs will by default include the GTIDs of all transactions, even those that changed suppressed parts of the database. If you don't want to restore GTIDs, pass --set-gtid-purged=OFF. To make a complete dump, pass --all-databases --triggers --routines --events.
```

当前数据库实例中开启了 GTID 功能，在开启有 GTID 功能的数据库实例中，导出其中任何一个库，如果没有显式地指定 `--set-gtid-purged` 参数，都会提示这一行信息。意思是默认情况下，导出的库中含有 GTID 信息，如果不想导出包含有 GTID 信息的数据库，需要显式地添加 `--set-gtid-purged=OFF` 参数。

```bash
# 加上 --set-gtid-purged=OFF 参数
root@Master:/# mysqldump -uwise2c -p --set-gtid-purged=OFF bldr user > user_bak.sql
Enter password:

root@Master:/# ls user*
user_bak.sql
```

### 导入单表

#### 方法一：mysql 命令导入

```bash
root@Master:/# mysql -uwise2c -p bldr < user_bak.sql
Enter password:
```

#### 方法二：source 命令导入

```bash
root@Master:/# mysql -uwise2c -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.

mysql> use bldr;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed

mysql> source user_bak.sql
Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)
```

### 导出 / 导入整个数据库

```bash
# 导出数据库
root@Master:/# mysqldump -uwise2c -p --set-gtid-purged=OFF builderdb > bak.sql
Enter password:

# 导入数据库
root@Master:/# mysql -uwise2c -p builderdb < bak.sql
Enter password:
```

## 配置说明

| 参数                               | 作用                         | 示例                                                        |
| -------------------------------- | -------------------------- | --------------------------------------------------------- |
| `--set-gtid-purged=OFF`          | 导出时不包含 GTID 信息，避免导入到新实例时冲突 | `mysqldump -u root -p --set-gtid-purged=OFF db > bak.sql` |
| `--all-databases`                | 导出所有数据库                    | `mysqldump -u root -p --all-databases > all.sql`          |
| `--triggers --routines --events` | 同时导出触发器、存储过程、事件            | 用于完整数据库迁移                                                 |
| `-p`                             | 提示输入密码                     | `mysqldump -u root -p db > bak.sql`                       |

## 注意事项 / 常见误区

- ❌ 不要直接在生产环境执行 `CREATE TABLE ... AS SELECT`，GTID 开启时会直接报错；即使不开启 GTID，它也不会复制索引和约束。
- ✅ 复制表结构优先使用 `CREATE TABLE ... LIKE`，可以完整保留主键、索引、字符集等。
- ✅ 跨实例导入导出时建议加 `--set-gtid-purged=OFF`，否则容易因 GTID 不一致导致导入失败。
- ✅ 执行 `DROP TABLE` 或 `TRUNCATE` 前务必确认环境，测试环境也要养成先备份的习惯。
- ✅ 敏感数据备份应加密存储或脱敏处理，避免泄露。

## 参考来源

- [MySQL GTID consistency 错误解决](https://www.unixso.com/MySQL/gtid_mode-enforce_gtid_consistency.html)
- [[数据库基础知识]]
- [[常用 SQL 语句]]
