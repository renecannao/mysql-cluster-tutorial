====
engine_condition_pushdown and ndb_join_pushdown
====


Indexes statistics
====

Indexes statistics can be incorrect in NDBCLUSTER if ANALYZE TABLE is not executed.


Add a new index on test table::
  
  mysql> ALTER TABLE table1 MODIFY v2 int(11) STORAGE MEMORY DEFAULT NULL;
  Query OK, 218216 rows affected (3.37 sec)
  Records: 218216  Duplicates: 0  Warnings: 0
  
  mysql> ALTER TABLE table1 ADD INDEX idx_v2(v2);
  Query OK, 0 rows affected (1.26 sec)
  Records: 0  Duplicates: 0  Warnings: 0

Updated some rows::
  
  mysql> UPDATE table1 SET v2=CEIL(RAND()*100) WHERE v2 IS NULL LIMIT 20000;
  Query OK, 20000 rows affected (20.64 sec)
  Rows matched: 20000  Changed: 20000  Warnings: 0
  
  mysql> UPDATE table1 SET v2=CEIL(RAND()*100) WHERE v2 IS NULL LIMIT 20000;
  Query OK, 20000 rows affected (19.14 sec)
  Rows matched: 20000  Changed: 20000  Warnings: 0

Run EXPLAIN on a index scan range, and verify the matching rows::
  
  mysql> EXPLAIN SELECT COUNT(*) FROM table1 WHERE v2 BETWEEN 1 AND 10;
  +----+-------------+--------+-------+---------------+--------+---------+------+-------+----------------------------------------------+
  | id | select_type | table  | type  | possible_keys | key    | key_len | ref  | rows  | Extra                                        |
  +----+-------------+--------+-------+---------------+--------+---------+------+-------+----------------------------------------------+
  |  1 | SIMPLE      | table1 | range | idx_v2        | idx_v2 | 5       | NULL | 10910 | Using where with pushed condition; Using MRR |
  +----+-------------+--------+-------+---------------+--------+---------+------+-------+----------------------------------------------+
  |  1 | SIMPLE      | table1 | range | idx_v2        | idx_v2 | 5       | NULL | 11782 | Using where with pushed condition |
  +----+-------------+--------+-------+---------------+--------+---------+------+-------+-----------------------------------+
  1 row in set (0.00 sec)
  
  mysql> SELECT COUNT(*) FROM table1 WHERE v2 BETWEEN 1 AND 10;
  +----------+
  | COUNT(*) |
  +----------+
  |     3953 |
  +----------+
  1 row in set (0.03 sec)

Run ANALYZE TABLE to update indexes statistics::
  
  mysql> ANALYZE TABLE table1;
  +-------------+---------+----------+----------+
  | Table       | Op      | Msg_type | Msg_text |
  +-------------+---------+----------+----------+
  | mydb.table1 | analyze | status   | OK       |
  +-------------+---------+----------+----------+
  1 row in set (12.80 sec)
  
  mysql> EXPLAIN SELECT COUNT(*) FROM table1 WHERE v2 BETWEEN 1 AND 10;
  +----+-------------+--------+-------+---------------+--------+---------+------+------+-----------------------------------+
  | id | select_type | table  | type  | possible_keys | key    | key_len | ref  | rows | Extra                             |
  +----+-------------+--------+-------+---------------+--------+---------+------+------+-----------------------------------+
  |  1 | SIMPLE      | table1 | range | idx_v2        | idx_v2 | 5       | NULL | 3970 | Using where with pushed condition |
  +----+-------------+--------+-------+---------------+--------+---------+------+------+-----------------------------------+
  1 row in set (0.03 sec)
  
  mysql> SELECT COUNT(*) FROM table1 WHERE v2 BETWEEN 1 AND 10;
  +----------+
  | COUNT(*) |
  +----------+
  |     3953 |
  +----------+
  1 row in set (0.04 sec)



Engine condition pushdown
====


What is "with pushed condition" ?::
  
  root@node2:~# mysql mydb
  Reading table information for completion of table and column names
  You can turn off this feature to get a quicker startup with -A
  
  Welcome to the MySQL monitor.  Commands end with ; or \g.
  Your MySQL connection id is 1210
  Server version: 5.5.29-ndb-7.2.10-cluster-gpl MySQL Cluster Community Server (GPL)
  
  Copyright (c) 2000, 2012, Oracle and/or its affiliates. All rights reserved.
  
  Oracle is a registered trademark of Oracle Corporation and/or its
  affiliates. Other names may be trademarks of their respective
  owners.
  
  Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
  
  mysql> SHOW VARIABLES LIKE 'optimizer_switch';
  +------------------+------------------------------------------------------------------------------------------------------------------------+
  | Variable_name    | Value                                                                                                                  |
  +------------------+------------------------------------------------------------------------------------------------------------------------+
  | optimizer_switch | index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=on |
  +------------------+------------------------------------------------------------------------------------------------------------------------+
  1 row in set (0.00 sec)

Run a new query with index range scan, and verify few status variables::
  
  mysql> EXPLAIN SELECT COUNT(*) FROM table1 WHERE v2 BETWEEN 1 AND 10 AND v>'ee';
  +----+-------------+--------+-------+---------------+--------+---------+------+------+-----------------------------------+
  | id | select_type | table  | type  | possible_keys | key    | key_len | ref  | rows | Extra                             |
  +----+-------------+--------+-------+---------------+--------+---------+------+------+-----------------------------------+
  |  1 | SIMPLE      | table1 | range | idx_v2        | idx_v2 | 5       | NULL | 3970 | Using where with pushed condition |
  +----+-------------+--------+-------+---------------+--------+---------+------+------+-----------------------------------+
  1 row in set (0.01 sec)
  
  mysql> SHOW STATUS LIKE 'Ndb_api_read_row_count_session';
  +--------------------------------+-------+
  | Variable_name                  | Value |
  +--------------------------------+-------+
  | Ndb_api_read_row_count_session | 1     |
  +--------------------------------+-------+
  1 row in set (0.00 sec)
  
  mysql> SELECT COUNT(*) FROM table1 WHERE v2 BETWEEN 1 AND 10 AND v>'ee';
  +----------+
  | COUNT(*) |
  +----------+
  |      259 |
  +----------+
  1 row in set (0.04 sec)

Only the matching rows are sent to the API node::
  
  mysql> SHOW STATUS LIKE 'Ndb_api_read_row_count_session';
  +--------------------------------+-------+
  | Variable_name                  | Value |
  +--------------------------------+-------+
  | Ndb_api_read_row_count_session | 260   |
  +--------------------------------+-------+
  1 row in set (0.00 sec)
  
  mysql> quit
  Bye

Running the same without engine_condition_pushdown::

  root@node2:~# mysql mydb
  Reading table information for completion of table and column names
  You can turn off this feature to get a quicker startup with -A
  
  Welcome to the MySQL monitor.  Commands end with ; or \g.
  Your MySQL connection id is 1211
  Server version: 5.5.29-ndb-7.2.10-cluster-gpl MySQL Cluster Community Server (GPL)
  
  Copyright (c) 2000, 2012, Oracle and/or its affiliates. All rights reserved.
  
  Oracle is a registered trademark of Oracle Corporation and/or its
  affiliates. Other names may be trademarks of their respective
  owners.
  
  Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
  
  mysql> SHOW VARIABLES LIKE 'optimizer_switch';
  +------------------+------------------------------------------------------------------------------------------------------------------------+
  | Variable_name    | Value                                                                                                                  |
  +------------------+------------------------------------------------------------------------------------------------------------------------+
  | optimizer_switch | index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=on |
  +------------------+------------------------------------------------------------------------------------------------------------------------+
  1 row in set (0.00 sec)
  
  mysql> SET optimizer_switch='index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=off';
  Query OK, 0 rows affected (0.00 sec)
  
  mysql> EXPLAIN SELECT COUNT(*) FROM table1 WHERE v2 BETWEEN 1 AND 10 AND v>'ee';
  +----+-------------+--------+-------+---------------+--------+---------+------+------+-------------+
  | id | select_type | table  | type  | possible_keys | key    | key_len | ref  | rows | Extra       |
  +----+-------------+--------+-------+---------------+--------+---------+------+------+-------------+
  |  1 | SIMPLE      | table1 | range | idx_v2        | idx_v2 | 5       | NULL | 3970 | Using where |
  +----+-------------+--------+-------+---------------+--------+---------+------+------+-------------+
  1 row in set (0.00 sec)
  
  mysql> SELECT COUNT(*) FROM table1 WHERE v2 BETWEEN 1 AND 10 AND v>'ee';
  +----------+
  | COUNT(*) |
  +----------+
  |      259 |
  +----------+
  1 row in set (0.04 sec)

All the rows matching the index range scan are sent to the API node, that then performs the filtering::
  
  mysql> SHOW STATUS LIKE 'Ndb_api_read_row_count_session';
  +--------------------------------+-------+
  | Variable_name                  | Value |
  +--------------------------------+-------+
  | Ndb_api_read_row_count_session | 3954  |
  +--------------------------------+-------+
  1 row in set (0.00 sec)
 



ndb_join_pushdown
====

The follow is an example of how joins are executed in NDB::
  
  root@node2:~# mysql mydb
  Reading table information for completion of table and column names
  You can turn off this feature to get a quicker startup with -A
  
  Welcome to the MySQL monitor.  Commands end with ; or \g.
  Your MySQL connection id is 1215
  Server version: 5.5.29-ndb-7.2.10-cluster-gpl MySQL Cluster Community Server (GPL)
  
  Copyright (c) 2000, 2012, Oracle and/or its affiliates. All rights reserved.
  
  Oracle is a registered trademark of Oracle Corporation and/or its
  affiliates. Other names may be trademarks of their respective
  owners.
  
  Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
  
  mysql> EXPLAIN SELECT COUNT(*) FROM table1 t1 JOIN table1 t2 ON t1.v2=t2.v2 WHERE t1.v2 BETWEEN 1 AND 5 AND t2.v > 'ee';
  +----+-------------+-------+-------+---------------+--------+---------+------------+------+-------------------------------------------------------------------+
  | id | select_type | table | type  | possible_keys | key    | key_len | ref        | rows | Extra                                                             |
  +----+-------------+-------+-------+---------------+--------+---------+------------+------+-------------------------------------------------------------------+
  |  1 | SIMPLE      | t1    | range | idx_v2        | idx_v2 | 5       | NULL       | 2042 | Parent of 2 pushed join@1; Using where with pushed condition      |
  |  1 | SIMPLE      | t2    | ref   | idx_v2        | idx_v2 | 5       | mydb.t1.v2 | 2353 | Child of 't1' in pushed join@1; Using where with pushed condition |
  +----+-------------+-------+-------+---------------+--------+---------+------------+------+-------------------------------------------------------------------+
  2 rows in set (0.01 sec)
  
  mysql> SHOW VARIABLES LIKE 'optimizer_switch';
  +------------------+------------------------------------------------------------------------------------------------------------------------+
  | Variable_name    | Value                                                                                                                  |
  +------------------+------------------------------------------------------------------------------------------------------------------------+
  | optimizer_switch | index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=on |
  +------------------+------------------------------------------------------------------------------------------------------------------------+
  1 row in set (0.00 sec)
  
  mysql> SHOW VARIABLES LIKE 'ndb_join_pushdown';
  +-------------------+-------+
  | Variable_name     | Value |
  +-------------------+-------+
  | ndb_join_pushdown | ON    |
  +-------------------+-------+
  1 row in set (0.00 sec)

To fully understand the effect of engine_condition_pushdown and ndb_join_pushdown, we now disable them::
 
  mysql> SET optimizer_switch='index_merge=on,index_merge_union=on,index_merge_sort_union=on,index_merge_intersection=on,engine_condition_pushdown=off';
  Query OK, 0 rows affected (0.00 sec)
  
  mysql> SET ndb_join_pushdown=OFF;
  Query OK, 0 rows affected (0.00 sec)
  
  mysql> EXPLAIN SELECT COUNT(*) FROM table1 t1 JOIN table1 t2 ON t1.v2=t2.v2 WHERE t1.v2 BETWEEN 1 AND 5 AND t2.v > 'ee';
  +----+-------------+-------+-------+---------------+--------+---------+------------+------+-------------+
  | id | select_type | table | type  | possible_keys | key    | key_len | ref        | rows | Extra       |
  +----+-------------+-------+-------+---------------+--------+---------+------------+------+-------------+
  |  1 | SIMPLE      | t1    | range | idx_v2        | idx_v2 | 5       | NULL       | 2042 | Using where |
  |  1 | SIMPLE      | t2    | ref   | idx_v2        | idx_v2 | 5       | mydb.t1.v2 | 2353 | Using where |
  +----+-------------+-------+-------+---------------+--------+---------+------------+------+-------------+
  2 rows in set (0.00 sec)
  
  mysql> SHOW STATUS LIKE 'Ndb_api_read_row_count_session';
  +--------------------------------+-------+
  | Variable_name                  | Value |
  +--------------------------------+-------+
  | Ndb_api_read_row_count_session | 1     |
  +--------------------------------+-------+
  1 row in set (0.00 sec)
  
  mysql> SELECT COUNT(*) FROM table1 t1 JOIN table1 t2 ON t1.v2=t2.v2 WHERE t1.v2 BETWEEN 1 AND 5 AND t2.v > 'ee';
  +----------+
  | COUNT(*) |
  +----------+
  |    50783 |
  +----------+
  1 row in set (9.03 sec)
    
  mysql> SHOW STATUS LIKE 'Ndb_api_read_row_count_session';
  +--------------------------------+--------+
  | Variable_name                  | Value  |
  +--------------------------------+--------+
  | Ndb_api_read_row_count_session | 810981 |
  +--------------------------------+--------+
  1 row in set (0.00 sec)
   
  mysql> quit
  Bye

Conclusion: the join matches ~51k rows, but ~811k rows are sent to the API node, where the filtering is performed.


Performing the same join, but with engine_condition_pushdown and ndb_join_pushdown enabled::
  
  root@node2:~# mysql mydb
  Reading table information for completion of table and column names
  You can turn off this feature to get a quicker startup with -A
  
  Welcome to the MySQL monitor.  Commands end with ; or \g.
  Your MySQL connection id is 1217
  Server version: 5.5.29-ndb-7.2.10-cluster-gpl MySQL Cluster Community Server (GPL)
  
  Copyright (c) 2000, 2012, Oracle and/or its affiliates. All rights reserved.
  
  Oracle is a registered trademark of Oracle Corporation and/or its
  affiliates. Other names may be trademarks of their respective
  owners.
    
  Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
  
  mysql> EXPLAIN SELECT COUNT(*) FROM table1 t1 JOIN table1 t2 ON t1.v2=t2.v2 WHERE t1.v2 BETWEEN 1 AND 5 AND t2.v > 'ee';
  +----+-------------+-------+-------+---------------+--------+---------+------------+------+-------------------------------------------------------------------+
  | id | select_type | table | type  | possible_keys | key    | key_len | ref        | rows | Extra                                                             |
  +----+-------------+-------+-------+---------------+--------+---------+------------+------+-------------------------------------------------------------------+
  |  1 | SIMPLE      | t1    | range | idx_v2        | idx_v2 | 5       | NULL       | 2042 | Parent of 2 pushed join@1; Using where with pushed condition      |
  |  1 | SIMPLE      | t2    | ref   | idx_v2        | idx_v2 | 5       | mydb.t1.v2 | 2353 | Child of 't1' in pushed join@1; Using where with pushed condition |
  +----+-------------+-------+-------+---------------+--------+---------+------------+------+-------------------------------------------------------------------+
  2 rows in set (0.00 sec)
  
  mysql> SHOW STATUS LIKE 'Ndb_api_read_row_count_session';
  +--------------------------------+-------+
  | Variable_name                  | Value |
  +--------------------------------+-------+
  | Ndb_api_read_row_count_session | 1     |
  +--------------------------------+-------+
  1 row in set (0.00 sec)
  
  mysql> SELECT COUNT(*) FROM table1 t1 JOIN table1 t2 ON t1.v2=t2.v2 WHERE t1.v2 BETWEEN 1 AND 5 AND t2.v > 'ee';
  +----------+
  | COUNT(*) |
  +----------+
  |    50783 |
  +----------+
  1 row in set (5.51 sec)
  
  mysql> SHOW STATUS LIKE 'Ndb_api_read_row_count_session';
  +--------------------------------+-------+
  | Variable_name                  | Value |
  +--------------------------------+-------+
  | Ndb_api_read_row_count_session | 52793 |
  +--------------------------------+-------+
  1 row in set (0.00 sec)

Conclusion: with engine_condition_pushdown and ndb_join_pushdown enabled, filtering and join is performed on the Data Node and not on the API Node.

Note: a lot of limitation applies to both engine_condition_pushdown and ndb_join_pushdown.

