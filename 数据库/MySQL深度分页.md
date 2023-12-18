#### MySQL大数据量深度分页

##### 子查询优化

```
mysql> SELECT MCS_PROD_ID,MCS_CODE,MCS_NAME FROM MCS_PROD WHERE MCS_PROD_ID >= ( SELECT m1.MCS_PROD_ID FROM MCS_PROD m1 WHERE m1.UPDT_TIME >= '1970-01-01 00:00:00.0' ORDER BY m1.UPDT_TIME LIMIT 3000000, 1) LIMIT 1;
+-------------+-------------------------+------------------------+
| MCS_PROD_ID | MCS_CODE                | MCS_NAME               |
+-------------+-------------------------+------------------------+
|     3021401 | XA892010009391491861476 | 金属解剖型接骨板T型接骨板A |
+-------------+-------------------------+------------------------+
1 row in set (0.76 sec)

mysql> EXPLAIN SELECT MCS_PROD_ID,MCS_CODE,MCS_NAME FROM MCS_PROD WHERE MCS_PROD_ID >= ( SELECT m1.MCS_PROD_ID FROM MCS_PROD m1 WHERE m1.UPDT_TIME >= '1970-01-01 00:00:00.0' ORDER BY m1.UPDT_TIME LIMIT 3000000, 1) LIMIT 1;
+----+-------------+----------+------------+-------+---------------+------------+---------+------+---------+----------+--------------------------+
| id | select_type | table    | partitions | type  | possible_keys | key        | key_len | ref  | rows    | filtered | Extra                    |
+----+-------------+----------+------------+-------+---------------+------------+---------+------+---------+----------+--------------------------+
|  1 | PRIMARY     | MCS_PROD | NULL       | range | PRIMARY       | PRIMARY    | 4       | NULL | 2296653 |   100.00 | Using where              |
|  2 | SUBQUERY    | m1       | NULL       | range | MCS_PROD_1    | MCS_PROD_1 | 5       | NULL | 2296653 |   100.00 | Using where; Using index |
+----+-------------+----------+------------+-------+---------------+------------+---------+------+---------+----------+--------------------------+
2 rows in set, 1 warning (0.77 sec)
```

使用子查询查询出主键，使用聚簇索引，省去回表操作

##### 延迟关联

```
mysql> SELECT MCS_PROD_ID,MCS_CODE,MCS_NAME FROM MCS_PROD INNER JOIN (SELECT m1.MCS_PROD_ID FROM MCS_PROD m1 WHERE m1.UPDT_TIME >= '1970-01-01 00:00:00.0' ORDER BY m1.UPDT_TIME LIMIT 3000000, 1) AS  MCS_PROD2 USING(MCS_PROD_ID);
+-------------+-------------------------+------------------------+
| MCS_PROD_ID | MCS_CODE                | MCS_NAME               |
+-------------+-------------------------+------------------------+
|     3021401 | XA892010009391491861476 | 金属解剖型接骨板T型接骨板A |
+-------------+-------------------------+------------------------+
1 row in set (0.75 sec)

mysql> EXPLAIN SELECT MCS_PROD_ID,MCS_CODE,MCS_NAME FROM MCS_PROD INNER JOIN (SELECT m1.MCS_PROD_ID FROM MCS_PROD m1 WHERE m1.UPDT_TIME >= '1970-01-01 00:00:00.0' ORDER BY m1.UPDT_TIME LIMIT 3000000, 1) AS  MCS_PROD2 USING(MCS_PROD_ID);
+----+-------------+------------+------------+--------+---------------+------------+---------+-----------------------+---------+----------+--------------------------+
| id | select_type | table      | partitions | type   | possible_keys | key        | key_len | ref                   | rows    | filtered | Extra                    |
+----+-------------+------------+------------+--------+---------------+------------+---------+-----------------------+---------+----------+--------------------------+
|  1 | PRIMARY     | <derived2> | NULL       | ALL    | NULL          | NULL       | NULL    | NULL                  | 2296653 |   100.00 | NULL                     |
|  1 | PRIMARY     | MCS_PROD   | NULL       | eq_ref | PRIMARY       | PRIMARY    | 4       | MCS_PROD2.MCS_PROD_ID |       1 |   100.00 | NULL                     |
|  2 | DERIVED     | m1         | NULL       | range  | MCS_PROD_1    | MCS_PROD_1 | 5       | NULL                  | 2296653 |   100.00 | Using where; Using index |
+----+-------------+------------+------------+--------+---------------+------------+---------+-----------------------+---------+----------+--------------------------+
3 rows in set, 1 warning (0.00 sec)
```

思路以及性能与子查询优化一致，只不过采用了 JOIN 的形式执行

##### 书签记录

关于 LIMIT 深分页问题，核心在于 OFFSET 值，它会 **导致 MySQL 扫描大量不需要的记录行然后抛弃掉**

我们可以先使用书签 **记录获取上次取数据的位置**，下次就可以直接从该位置开始扫描，这样可以 **避免使用 OFFEST**

假设需要查询 3000000 行数据后的第 1 条记录，查询可以这么写

```
mysql> SELECT MCS_PROD_ID,MCS_CODE,MCS_NAME FROM MCS_PROD WHERE MCS_PROD_ID < 3000000 ORDER BY UPDT_TIME LIMIT 1;
+-------------+-------------------------+---------------------------------+
| MCS_PROD_ID | MCS_CODE                | MCS_NAME                        |
+-------------+-------------------------+---------------------------------+
|         127 | XA683240878449276581799 | 股骨近端-1螺纹孔锁定板（纯钛）YJBL01 |
+-------------+-------------------------+---------------------------------+
1 row in set (0.00 sec)

mysql> EXPLAIN SELECT MCS_PROD_ID,MCS_CODE,MCS_NAME FROM MCS_PROD WHERE MCS_PROD_ID < 3000000 ORDER BY UPDT_TIME LIMIT 1;
+----+-------------+----------+------------+-------+---------------+------------+---------+------+------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys | key        | key_len | ref  | rows | filtered | Extra       |
+----+-------------+----------+------------+-------+---------------+------------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | MCS_PROD | NULL       | index | PRIMARY       | MCS_PROD_1 | 5       | NULL |    2 |    50.00 | Using where |
+----+-------------+----------+------------+-------+---------------+------------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)
```

好处是很明显的，查询速度超级快，**性能都会稳定在毫秒级**，从性能上考虑碾压其它方式

不过这种方式局限性也比较大，需要一种类似连续自增的字段，以及业务所能包容的连续概念，视情况而定