MySQL为什么varchar字段用数字查无法命中索引，而int字段用字符串查却能命中?
---


字符串字段误使用数字进行查询，会导致隐式类型转换，无法命中索引的坑我相信大多数小伙伴都踩过。
特别是当字段中存的大多数数据都是数字时，很容易先入为主地认为字段是 `int` 类型，错误地使用类似 `where file_id=123456789` 执行了查询。好一点的可能事先通过 `Explain` 命令查看语句的执行计划，发现竟然没用命中索引，从而纠正错误；杯具一点的代码发布上线后出现大量慢查询，数据库服务器的CPU使用率和磁盘IO飙升，酿成生产事故。

而细心的小伙伴一定会发现，虽然 `varchar` 字段用数字查无法命中索引，而 `int` 字段用字符串查却通常能很快查出结果。这是为什么呢？

下面我们通过实际测试来说明出现这种现象的原因。
测试用 MySQL版本为 5.7.18，数据表 `file` 结构如下，存储引擎为 **InnoDB**，表数据条数为 5 百万+。

```sql
mysql> SELECT VERSION();
+---------------------+
| VERSION()           |
+---------------------+
| 5.7.18-20170830-log |
+---------------------+

mysql> DESC `file`;
+----------+---------------------+------+-----+---------+----------------+
| Field    | Type                | Null | Key | Default | Extra          |
+----------+---------------------+------+-----+---------+----------------+
| id       | int(11)             | NO   | PRI | NULL    | auto_increment |
| fs_id    | varchar(20)         | NO   | MUL | NULL    |                |
| filename | varchar(255)        | NO   |     | NULL    |                |
| shareid  | bigint(20) unsigned | NO   | MUL | NULL    |                |
| uk       | bigint(20) unsigned | NO   |     | NULL    |                |
| pid      | varchar(32)         | NO   |     | NULL    |                |
+----------+---------------------+------+-----+---------+----------------+

mysql> SELECT COUNT(*) FROM `file`;
+----------+
| COUNT(*) |
+----------+
|  5416697 |
+----------+
```

#### **varchar 字段用数字进行查询**

数据表 `file` 中的 `fs_id` 字段是 `varchar` 类型，并且建立了普通索引 `idx_fs_id`。
当使用字符串进行查询时，耗时 **0.07** 秒。
通过 **EXPLAIN** 命令查看执行计划，结果表明查询时使用了 `fs_id` 字段的索引。

```sql
mysql> SELECT * FROM `file` WHERE `fs_id`='635341798980956';
+---------+-----------------+-------------+------------+------------+---------+
| id      | fs_id           | filename    | shareid    | uk         | pid     |
+---------+-----------------+-------------+------------+------------+---------+
| 1043170 | 635341798980956 | ⑮MySQL高级 | 3181065465 | 3959617630 | o6RlSp0 |
+---------+-----------------+-------------+------------+------------+---------+
1 row in set (0.07 sec)

mysql> EXPLAIN SELECT * FROM `file` WHERE `fs_id`='635341798980956';
+----+-------------+-------+------------+------+---------------+-----------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key       | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+-----------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | file  | NULL       | ref  | idx_fs_id     | idx_fs_id | 62      | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+-----------+---------+-------+------+----------+-------+
```

然而，当使用数字进行查询时，耗时 **7.04** 秒。
通过 **EXPLAIN** 命令查看执行计划，发现查询时进行了全表扫描，并未使用到索引。

```sql
mysql> SELECT * FROM `file` WHERE `fs_id`=635341798980956;
+---------+-----------------+-------------+------------+------------+---------+
| id      | fs_id           | filename    | shareid    | uk         | pid     |
+---------+-----------------+-------------+------------+------------+---------+
| 1043170 | 635341798980956 | ⑮MySQL高级 | 3181065465 | 3959617630 | o6RlSp0 |
+---------+-----------------+-------------+------------+------------+---------+
1 row in set (7.04 sec)

mysql> EXPLAIN SELECT * FROM `file` WHERE `fs_id`=635341798980956;
+----+-------------+-------+------------+------+---------------+------+---------+------+---------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows    | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+---------+----------+-------------+
|  1 | SIMPLE      | file  | NULL       | ALL  | idx_fs_id     | NULL | NULL    | NULL | 4878670 |    10.00 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+---------+----------+-------------+
```

`fs_id` 字段明明建立了索引，但是使用数字进行查询时，还是需要全表扫描。
之所以出现这种情况，相信大多数小伙伴都知道，是因为 `fs_id` 字段是字符串类型，而输入参数却是整数类型，所以触发了**隐式类型转换**。

#### **int 字段用字符串查进行查询**

数据表 `file` 中的 `shareid` 字段是 `bigint` 类型，并且建立了普通索引 `idx_shareid`。
当使用数字和字符串进行查询时，耗时都是 **0.04** 秒。
通过 **EXPLAIN** 命令查看执行计划，结果表明无论数字两边加不加引号，查询时使用了 `idx_shareid` 字段的索引。

```
mysql> SELECT * FROM `file` WHERE `shareid`=3181065465;
+---------+-----------------+-------------+------------+------------+---------+
| id      | fs_id           | filename    | shareid    | uk         | pid     |
+---------+-----------------+-------------+------------+------------+---------+
| 1043170 | 635341798980956 | ⑮MySQL高级 | 3181065465 | 3959617630 | o6RlSp0 |
+---------+-----------------+-------------+------------+------------+---------+
1 row in set (0.04 sec)

mysql> SELECT * FROM `file` WHERE `shareid`='3181065465';
+---------+-----------------+-------------+------------+------------+---------+
| id      | fs_id           | filename    | shareid    | uk         | pid     |
+---------+-----------------+-------------+------------+------------+---------+
| 1043170 | 635341798980956 | ⑮MySQL高级 | 3181065465 | 3959617630 | o6RlSp0 |
+---------+-----------------+-------------+------------+------------+---------+
1 row in set (0.04 sec)

mysql> EXPLAIN SELECT * FROM `file` WHERE `shareid`=3181065465;
+----+-------------+-------+------------+------+---------------+-------------+---------+-------+------+----------+-------+
| id | select_type | table | partitions | type | possible_keys | key         | key_len | ref   | rows | filtered | Extra |
+----+-------------+-------+------------+------+---------------+-------------+---------+-------+------+----------+-------+
|  1 | SIMPLE      | file  | NULL       | ref  | idx_shareid   | idx_shareid | 8       | const |    1 |   100.00 | NULL  |
+----+-------------+-------+------------+------+---------------+-------------+---------+-------+------+----------+-------+

mysql> EXPLAIN SELECT * FROM `file` WHERE `shareid`='3181065465';
+----+-------------+-------+------------+------+---------------+-------------+---------+-------+------+----------+-----------------------+
| id | select_type | table | partitions | type | possible_keys | key         | key_len | ref   | rows | filtered | Extra                 |
+----+-------------+-------+------------+------+---------------+-------------+---------+-------+------+----------+-----------------------+
|  1 | SIMPLE      | file  | NULL       | ref  | idx_shareid   | idx_shareid | 8       | const |    1 |   100.00 | Using index condition |
+----+-------------+-------+------------+------+---------------+-------------+---------+-------+------+----------+-----------------------+
```

对于这个结果，现在有三个疑问：
1. 隐式类型转换的规则是什么？
2. 为什么触发隐式类型转换，查询数据时就需要全表扫描？
3. 为什么 int 字段用字符串查就能命中索引？

#### **1. 隐式类型转换的规则是什么？**

有一个非常简单的方法，可以验证隐式类型转换的规则，就是看 `SELECT '10' > 9` 和  `SELECT 9 > '10'`  的结果：

如果规则是 **“将字符串转成数字”**，那么就是做数字比较，`SELECT '10' > 9` 的结果应该是 1，`SELECT 9 > '10'` 的结果应该是 0；
如果规则是 **“将数字转成字符串”**，那么就是做字符串比较，`SELECT '10' > 9` 的结果应该是 0，`SELECT 9 > '10'` 的结果应该是 1。

```
mysql> SELECT '10' > 9;
+----------+
| '10' > 9 |
+----------+
|        1 |
+----------+
```

```
mysql> SELECT 9 > '10';
+----------+
| 9 > '10' |
+----------+
|        0 |
+----------+
```

可见，`SELECT '10' > 9` 的结果是 1，`SELECT 9 > '10'` 的结果是 0，所以可以确认 MySQL 类型转换规则是：**当字符串和数字做比较时，会将字符串转换成数字**。

因此，当我们使用下面语句进行查询时：

```sql
SELECT * FROM `file` WHERE `fs_id`=635341798980956;
```

对于MySQL优化器来说，这个语句相当于将 `fs_id` 转换成 **int** 类型再与输入的值进行比较：
```sql
SELECT * FROM `file` WHERE CAST(`fs_id` AS signed INT)=635341798980956;
```

众所周知，**如果查询时对索引字段进行函数操作，查询过程将无法使用索引**。

#### **2. 为什么触发隐式类型转换，查询数据时就需要全表扫描？**

对于  InnoDB 的 **B+树索引结构**，相信大多数小伙伴都有一定的了解。

示例有下面一组数据：
```
1, 2, 3, 4, 6, 6, 7, 11, 13, 21, 23, 39, 42, 61, 71, 
101, 201, 220, 303, 345, 411, 601, 620, 701, 1402, 3333
```

当作为数值类型建立索引时，B+树索引结构如下：

![数值类型B+树索引](https://md.s1031.cn/xsj/2021_7_13_数值类型B+树索引.png)

当作为字符串类型建立索引时，数据顺序和B+树索引结构如下：
```
1, 101, 11, 13, 1402, 2, 201, 21, 220, 23, 3, 303, 3333, 
345, 39, 4, 411, 42, 6, 6, 601, 61, 620, 7, 701, 71
```

![字符串类型B+树索引](https://md.s1031.cn/xsj/2021_7_13_字符串类型B+树索引.png)

实际上，**B+ 树索引的快速定位能力，来源于同一层兄弟节点的有序性**。**对索引字段做函数操作，可能会破坏索引值的有序性**。
而**当字符串和数字做比较时，会将字符串转换成数字**。

因此，当字符串类型的字段 `fs_id` 接收到数值类型的输入参数时，`fs_id` 会被转换成数值类型，字符串类型建立的索引对于数值类型来说是乱序的，因此无法使用 `fs_id` 字段的索引，只能通过全表扫描进行查找。


#### **3. 为什么 int 字段用字符串查就能命中索引？**

是因为 **数值类型字段用字符串查询** 不会触发隐式类型转换吗？并不是。

通过上面验证，我们已知：**当字符串和数字做比较时，会将字符串转换成数字**。

因此，当我们使用字符串类型作为输入参数对数值型字段进行查询时：

```sql
SELECT * FROM `file` WHERE `shareid`='3181065465'
```

对于MySQL优化器来说，这个语句相当于将输入参数 `'3181065465'` 转换成 **int** 类型再进行查询：
```sql
SELECT * FROM `file` WHERE `shareid`=CAST('3181065465' AS signed INT);
```

而对等号后面的输入参数进行函数操作，是不影响字段 `shareid` 的索引使用的。因此，虽然数值类型字段用字符串查询也会发生隐式类型转换，但是并不影响字段索引的使用。


### 总结


**MySQL查询中，当字符串和数字做比较时，会触发隐式类型转换，转换规则是将字符串转换成数字。**

**当索引字段是字符串类型，输入参数是数值类型时，会将字段转换成数值类型再进行查找，也就是对索引字段做了函数操作，破坏了索引的有序性，因此无法使用索引。**
**当索引字段是数值类型，输参数是字符串类型时，会将输入参数转换成数值类型再进行查找，对等号后面的输入参数进行函数操作，并不影响索引字段的有序性，因此可以使用索引。**