====
ALTER TABLE Online Operations
====

In the previous section we saw that OPTIMIZE TABLE is an online operation in MySQL Cluster.

Additionally, MySQL Cluster supports Online Operations for ADD COLUMN, ADD INDEX and DROP INDEX.

Limitations:

* DROP COLUMN is not an online operation

* multiple ADD COLUMN / ADD INDEX / DROP INDEX cannot be combined.

* Online Operations are possible only for columns that are stored in memory: columns cannot be added online in Data Disk, neither indexes.

Online add column
----

MySQL Cluster supports online ADD COLUMN.

Verify the table structure and status::
  
  mysql> SHOW CREATE TABLE table1\G
  *************************** 1. row ***************************
         Table: table1
  Create Table: CREATE TABLE `table1` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `v` varchar(32) NOT NULL,
    PRIMARY KEY (`id`)
  ) ENGINE=ndbcluster AUTO_INCREMENT=719163 DEFAULT CHARSET=latin1
  1 row in set (0.00 sec)
  
  mysql> SHOW TABLE STATUS LIKE 'table1'\G
  *************************** 1. row ***************************
             Name: table1
           Engine: ndbcluster
          Version: 10
       Row_format: Dynamic
             Rows: 142295
   Avg_row_length: 28
      Data_length: 10518528
  Max_data_length: 0
     Index_length: 0
        Data_free: 0
   Auto_increment: 719163
      Create_time: NULL
      Update_time: NULL
       Check_time: NULL
        Collation: latin1_swedish_ci
         Checksum: NULL
   Create_options: 
          Comment: 
  1 row in set (0.02 sec)

From node3, start a series of INSERTs::
  
  node3> time for i in `seq 1 30` ; do date ; echo "INSERT INTO table1 (id,v) SELECT NULL,MD5(RAND()) FROM table1 LIMIT 1024;" | mysql mydb ; sleep 1 ; done


From node2 , run the DDL to add a column::
  
  mysql> ALTER TABLE table1 ADD COLUMN v2 int;
  Query OK, 0 rows affected, 1 warning (0.36 sec)
  Records: 0  Duplicates: 0  Warnings: 1
  
  mysql> SHOW WARNINGS;
  +---------+------+---------------------------------------------------------------+
  | Level   | Code | Message                                                       |
  +---------+------+---------------------------------------------------------------+
  | Warning | 1478 | Converted FIXED field to DYNAMIC to enable on-line ADD COLUMN |
  +---------+------+---------------------------------------------------------------+
  1 row in set (0.00 sec)
  
  mysql> SHOW TABLE STATUS LIKE 'table1'\G
  *************************** 1. row ***************************
             Name: table1
           Engine: ndbcluster
          Version: 10
       Row_format: Dynamic
             Rows: 173015
   Avg_row_length: 28
      Data_length: 12779520
  Max_data_length: 0
     Index_length: 0
        Data_free: 0
   Auto_increment: 749883
      Create_time: NULL
      Update_time: NULL
       Check_time: NULL
        Collation: latin1_swedish_ci
         Checksum: NULL
   Create_options: 
          Comment: 
  1 row in set (0.03 sec)


NDB Cluster supports online operations only when the row format is Dymanic. Row format is automatically converted to Dynamic if it was Fixed.


Add and drop index
----


From node3, start a series of INSERTs::
  
  node3> time for i in `seq 1 60` ; do date ; echo "INSERT INTO table1 (id,v) SELECT NULL,MD5(RAND()) FROM table1 LIMIT 1024;" | mysql mydb ; sleep 1 ; done

From node2 , run the DML to add and than drop indexes::
  
  mysql> ALTER TABLE table1 ADD INDEX idx_v (v);
  Query OK, 0 rows affected (2.13 sec)
  Records: 0  Duplicates: 0  Warnings: 0
  
  mysql> ALTER TABLE table1 ADD INDEX idx_v2 (v2);
  Query OK, 0 rows affected (1.17 sec)
  Records: 0  Duplicates: 0  Warnings: 0
  
  mysql> ALTER TABLE table1 DROP INDEX idx_v;
  Query OK, 0 rows affected (0.28 sec)
  Records: 0  Duplicates: 0  Warnings: 0
  
  mysql> ALTER TABLE table1 ADD INDEX idx_v (v);
  Query OK, 0 rows affected (2.65 sec)
  Records: 0  Duplicates: 0  Warnings: 0

ALTER TABLE is online, and not blocking.

Verify table status::
  
  mysql> SHOW CREATE TABLE table1\G
  *************************** 1. row ***************************
         Table: table1
  Create Table: CREATE TABLE `table1` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `v` varchar(32) NOT NULL,
    `v2` int(11) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `idx_v2` (`v2`),
    KEY `idx_v` (`v`)
  ) ENGINE=ndbcluster AUTO_INCREMENT=811323 DEFAULT CHARSET=latin1
  1 row in set (0.02 sec)
  
  mysql> SHOW TABLE STATUS LIKE 'table1'\G
  *************************** 1. row ***************************
             Name: table1
           Engine: ndbcluster
          Version: 10
       Row_format: Dynamic
             Rows: 234455
   Avg_row_length: 28
      Data_length: 17268736
  Max_data_length: 0
     Index_length: 0
        Data_free: 0
   Auto_increment: 811323
      Create_time: NULL
      Update_time: NULL
       Check_time: NULL
        Collation: latin1_swedish_ci
         Checksum: NULL
   Create_options: 
          Comment: 
  1 row in set (0.03 sec)

