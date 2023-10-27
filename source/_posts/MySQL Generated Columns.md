---
title: MySQL Generated Columns
date: 2023/10/26 11:14:50
categories:
- [数据库, MySQL]
tags:
- 数据库
- MySQL
- 数据类型
---

### 序言

`MySQL 5.7` 引入了生成列（`Generated Columns` 还有一种虚拟列的叫法）的支持。生成列中的值由列定义的表达式计算得出。

### 创建生成列

创建生成的列语法定义如下：

```sql
col_name data_type [GENERATED ALWAYS] AS (expr)
  [VIRTUAL | STORED] [NOT NULL | NULL]
  [UNIQUE [KEY]] [[PRIMARY] KEY]
  [COMMENT 'string']
```

`AS (expr)` 表示生成列的同时定义用于计算生成列值的表达式。

`AS` 前面可以选择性的加上 `GENERATED ALWAYS`，使列的生成性质更加明确。

关键字 `VIRTUAL` 或 `STORED` 表示列值的存储方式，它对列的使用有影响：

-  `VIRTUAL` : 列值不会被存储，不占用任何存储空间。而是在读取行时计算，同时在执行触发器（特别是 BEFORE 触发器，它们在插入、更新或删除操作之前执行）之后，虚拟列的值会被计算。这是因为虚拟列的值通常**依赖于其他列的值或特定的表达式**，而这些值可能会在触发器中进行修改。因此，虚拟列的值需要在触发器执行之后计算，以确保它们反映最新的数据。`InnoDB` 支持虚拟列上的二级索引。
- `STORED` : 插入或更新行时，会计算并存储列值。存储列需要存储空间，并可编制索引。

如果以上两个关键字都未指定，则默认为 `VIRTUAL`。

允许在表中混合使用 `VIRTUAL` 列和 `STORED` 列。

还可以给出其他属性，例如生成列是否需要建立索引或是否可以为 `NULL`，以及提供列注释等。

对于 `CREATE TABLE ...LIKE`，目标表会**保留**原始表中生成的列信息。

对于 `CREATE TABLE ...SELECT` ，目标表**不保留**关于选自表中的列是否为生成列的信息。语句的 SELECT 部分不能为目标表中的生成列赋值。

下面这个简单的例子创建了一个表，该表存储了 `sidea` 和 `sidb` 列中直角三角形边的长度，并计算`sidec` 中斜边的长度（其他两边的平方和的平方根）：

```sql
mysql> CREATE TABLE triangle (
    ->   sidea DOUBLE,
    ->   sideb DOUBLE,
    ->   sidec DOUBLE AS (SQRT(sidea * sidea + sideb * sideb))
    -> );
Query OK, 0 rows affected (0.12 sec)
```

插入数据：

```sql
INSERT INTO triangle (sidea, sideb) VALUES(1,1),(3,4),(6,8);
```

从该表中查询会得到以下结果：

```sql
mysql> SELECT * FROM triangle;
+-------+-------+--------------------+
| sidea | sideb | sidec              |
+-------+-------+--------------------+
|     1 |     1 | 1.4142135623730951 |
|     3 |     4 |                  5 |
|     6 |     8 |                 10 |
+-------+-------+--------------------+
```

任何使用 `triangle` 表的应用程序都可以直接访问斜边值，而无需指定计算斜边值的表达式。

### 生成列表达式规则

如果表达式中包含不被允许的结构，则会发生错误，所以生成的列表达式必须遵守以下规则：

- 允许使用字面量、确定性内置函数和运算符。**如果在表格数据相同的情况下，多次调用只会产生相同的结果，而与所连接的用户无关，那么这个函数就是确定的**。
- 不允许使用存储函数和可加载函数。
- 不允许使用存储过程和函数参数。
- 不允许使用变量（包括系统变量、用户定义变量和存储的程序局部变量）。
- 不允许使用子查询。
- 生成列的定义**可以引用其他生成列，但只能引用表定义中较早出现的生成列**。生成的列定义可以引用表中的**任何基础（非生成）列**，无论其定义出现的时间是早还是晚。
- 生成的列定义中不能使用 `AUTO_INCREMENT` 属性。
- 在生成列定义中，`AUTO_INCREMENT` 列不能被作为基列。
- 如果表达式求值导致截断或为函数提供了不正确的输入，则 `CREATE TABLE` 语句将以错误结束，`DDL` 操作将被拒绝。

> 注意：如果表达式求值的数据类型与声明的列类型不同，则会根据 `MySQL` 默认的隐式类型转换规则强制转换为声明的类型。
>
> 表达式计算使用计算时有效的 `SQL` 模式。如果表达式定义的任何部分依赖于 `SQL` 模式，那么除非在所有使用过程中 `SQL` 模式都相同，否则不同的表格使用过程中可能会出现不同的结果。



#### 约束

- 存储生成列上的外键约束不能使用 `CASCADE`、`SET NULL` 或 `SET DEFAULT` 作为 `ON UPDATE` 引用操作，也不能使用 `SET NULL` 或 `SET DEFAULT` 作为 `ON DELETE` 引用操作。
- 存储生成列的基列上的外键约束不能使用 `CASCADE`、`SET NULL` 或 `SET DEFAULT` 作为 `ON UPDATE` 或 `ON DELETE` 引用操作。
- 外键约束不能引用虚拟生成的列。
- 对于 `INSERT`、`REPLACE` 和 `UPDATE`，如果生成的列被明确插入、替换或更新，则唯一允许的值是 `DEFAULT`。

视图中的生成列被认为是可更新的，因为可以对其进行赋值。但是，如果要显式更新该列，则唯一允许的值是 `DEFAULT`。

### 生成列使用场景

生成列有几种使用情况，比如下面这些：

- 虚拟生成列可以用来简化和统一查询。一个复杂的条件可以定义为一个生成列，并在表的多个查询中引用，以确保所有查询都使用完全相同的条件。
- 存储生成的列可用作物化缓存，用于处理计算成本较高的复杂条件。
- 生成列可以模拟函数式索引：使用生成的列来定义函数表达式并为其建立索引。这对于处理无法直接建立索引的列类型（如 `JSON` 列）非常有用。对于存储生成列，这种方法的缺点是要存储两次值，一次是生成列的值，另一次是索引中的值。
- 如果生成的列有索引，优化器会识别与列定义相匹配的查询表达式，即使查询没有直接引用列名，查询执行过程中也会酌情使用列索引。

举例：

假设表 `t1` 包含名和姓两列，应用程序经常使用这样的表达式来构造全名：

```sql
SELECT CONCAT(first_name,' ',last_name) AS full_name FROM t1;
```

避免写出表达式的一种方法是在 `t1` 上创建视图 `v1`，这样就可以直接选择 `full_name`，而无需使用表达式，从而简化了应用程序：

```sql
CREATE VIEW v1 AS
SELECT *, CONCAT(first_name,' ',last_name) AS full_name FROM t1;

SELECT full_name FROM v1;
```

生成列还能让应用程序直接选择 `full_name`，而无需定义视图：

```sql
CREATE TABLE t1 (
  first_name VARCHAR(10),
  last_name VARCHAR(10),
  full_name VARCHAR(255) AS (CONCAT(first_name,' ',last_name))
);

SELECT full_name FROM t1;
```



### 生成列使用索引

上一篇文章所提到，`JSON` 列无法直接建立索引。要创建间接引用此类列的索引，可以定义一个生成列来提取应被索引的信息，然后在生成列上创建索引。

下面的例子我们将在生成列上创建间接引用 `JSON` 列的索引：

首先我们创建一个班级表，`student` 保存学生信息，`g` 表示引用 `student` 中的 `id` 生成的虚拟列：

```sql
mysql> CREATE TABLE CLASS (
    ->          student JSON,    
    ->          g INT GENERATED ALWAYS AS (student -> "$.id"),
    ->          INDEX i (g)
    -> );
Query OK, 0 rows affected (0.33 sec)
```

往表中插入数据：

```sql
mysql> INSERT INTO CLASS(student) VALUES 
    ->          ('{"id": "1", "name": "小王"}'), ('{"id": "2", "name": "小李"}'), 
    ->          ('{"id": "3", "name": "老张"}'), ('{"id": "4", "name": "老赵"}'); 
Query OK, 4 rows affected (0.03 sec)
```

查询：

```sql
mysql> SELECT student->>"$.name" AS NAME  
    ->          FROM CLASS WHERE g > 2;   
+------+
| NAME |
+------+
| 老张 |
| 老赵 |
+------+
```

然后来看一下这条 `SQL` 语句的执行计划：

```sql
mysql> EXPLAIN SELECT student->"$.name" AS NAME 
    ->          FROM CLASS WHERE g > 2;
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type  | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | CLASS | NULL       | range | i             | i    | 5       | NULL |    2 |   100.00 | Using where |
+----+-------------+-------+------------+-------+---------------+------+---------+------+------+----------+-------------+
```

可以看到，这条查询语句使用到了索引 `i` , 也就是说我们可以通过对生成列中引用 `JSON` 中的属性并建立索引间接达到对 `JSON` 中的属性建立索引的效果。