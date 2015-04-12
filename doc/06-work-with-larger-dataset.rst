====
Work with increased amount of data
====


Add more data into the Cluster
----

This section is a walkthrough to add more data in a single table, gather more information from the Cluster, and perform some table maintenance.

Commands::
  
  mysql> use mydb
  mysql> DROP TABLE IF EXISTS table1;
  mysql> CREATE TABLE `table1` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `v` VARCHAR(32) NOT NULL,
     PRIMARY KEY (`id`)
  ) ENGINE=ndbcluster;
  
  INSERT INTO table1 VALUES(NULL,MD5(RAND()));
  INSERT INTO table1 SELECT NULL,MD5(RAND()) FROM table1;
  SELECT * FROM table1;
  
  shell> for i in `seq 1 20` ; do echo "INSERT INTO table1 SELECT NULL,MD5(RAND()) FROM table1 LIMIT 8192;" ; done | mysql mydb
  shell> for i in `seq 1 20` ; do echo "INSERT INTO table1 SELECT NULL,MD5(RAND()) FROM table1 LIMIT 8192;" ; done | mysql mydb
  
  mysql> SELECT COUNT(*) FROM table1;
  +----------+
  | COUNT(*) |
  +----------+
  |   237568 |
  +----------+
  1 row in set (0.01 sec)
  
  mysql> SHOW TABLE STATUS LIKE 'table1'\G
  *************************** 1. row ***************************
             Name: table1
           Engine: ndbcluster
          Version: 10
       Row_format: Dynamic
             Rows: 237568
   Avg_row_length: 28
      Data_length: 17498112
  Max_data_length: 0
     Index_length: 0
        Data_free: 0
   Auto_increment: 237569
      Create_time: NULL
      Update_time: NULL
       Check_time: NULL
        Collation: latin1_swedish_ci
         Checksum: NULL
   Create_options: 
          Comment: 
  1 row in set (0.01 sec)
  

Check memory usage from ndb_mgm ::  

  ndb_mgm> ALL REPORT MEMORY
  Node 2: Data usage is 28%(733 32K pages of total 2560)
  Node 2: Index usage is 20%(485 8K pages of total 2336)
  Node 3: Data usage is 28%(733 32K pages of total 2560)
  Node 3: Index usage is 20%(485 8K pages of total 2336)
  
    
Use `ndb_desc <http://dev.mysql.com/doc/refman/5.5/en/mysql-cluster-programs-ndb-desc.html>`_ to gather information about the table::
  
  root@node1:~# ndb_desc -p -d mydb table1
  -- table1 --
  Version: 6
  Fragment type: 9
  K Value: 6
  Min load factor: 78
  Max load factor: 80
  Temporary table: no
  Number of attributes: 2
  Number of primary keys: 1
  Length of frm data: 248
  Row Checksum: 1
  Row GCI: 1
  SingleUserMode: 0
  ForceVarPart: 1
  FragmentCount: 2
  ExtraRowGciBits: 0
  ExtraRowAuthorBits: 0
  TableStatus: Retrieved
  -- Attributes -- 
  id Int PRIMARY KEY DISTRIBUTION KEY AT=FIXED ST=MEMORY AUTO_INCR
  v Varchar(32;latin1_swedish_ci) NOT NULL AT=SHORT_VAR ST=MEMORY
  
  -- Indexes -- 
  PRIMARY KEY(id) - UniqueHashIndex
  PRIMARY(id) - OrderedIndex
  
  -- Per partition info -- 
  Partition       Row count       Commit count    Frag fixed memory       Frag varsized memory    Extent_space    Free extent_space
  0               118579          118579          3342336                 5373952                 0               0                 
  1               118989          118989          3375104                 5406720                 0               0                 
  
  
  NDBT_ProgramExit: 0 - OK




Handling large transaction
----

The follows creates a transaction larger than MySQL Cluster can handle::
  
  mysql> START TRANSACTION;
  Query OK, 0 rows affected (0.00 sec)
  
  mysql> SELECT COUNT(*) FROM table1;
  +----------+
  | COUNT(*) |
  +----------+
  |   237568 |
  +----------+
  1 row in set (0.02 sec)
  
  mysql> INSERT INTO table1 SELECT NULL,MD5(RAND()) FROM table1 LIMIT 10000;
  Query OK, 10000 rows affected (0.77 sec)
  Records: 10000  Duplicates: 0  Warnings: 0
  
  mysql> INSERT INTO table1 SELECT NULL,MD5(RAND()) FROM table1 LIMIT 10000;
  Query OK, 10000 rows affected (0.77 sec)
  Records: 10000  Duplicates: 0  Warnings: 0
  
  mysql> INSERT INTO table1 SELECT NULL,MD5(RAND()) FROM table1 LIMIT 10000;
  Query OK, 10000 rows affected (0.72 sec)
  Records: 10000  Duplicates: 0  Warnings: 0
  
  mysql> SELECT COUNT(*) FROM table1;
  +----------+
  | COUNT(*) |
  +----------+
  |   267568 |
  +----------+
  1 row in set (0.00 sec)
  
  mysql> INSERT INTO table1 SELECT NULL,MD5(RAND()) FROM table1 LIMIT 10000;
  ERROR 1297 (HY000): Got temporary error 233 'Out of operation records in transaction coordinator (increase MaxNoOfConcurrentOperations)' from NDBCLUSTER
  mysql> SELECT COUNT(*) FROM table1;
  +----------+
  | COUNT(*) |
  +----------+
  |   237568 |
  +----------+
  1 row in set (0.01 sec)


Notes:

* A transaction can handle a finite amount of records ( reads , updates and deletes ) , defined by `MaxNoOfConcurrentOperations <http://dev.mysql.com/doc/refman/5.5/en/mysql-cluster-ndbd-definition.html#ndbparam-ndbd-maxnoofconcurrentoperations>`_ ;

* The Transaction Coordinator (TC) tracks all the records involved in a transaction, and each data node tracks all the local records ;

* For very large transaction, both `MaxNoOfConcurrentOperations <http://dev.mysql.com/doc/refman/5.5/en/mysql-cluster-ndbd-definition.html#ndbparam-ndbd-maxnoofconcurrentoperations>`_ and `MaxNoOfLocalOperations <http://dev.mysql.com/doc/refman/5.5/en/mysql-cluster-ndbd-definition.html#ndbparam-ndbd-maxnooflocaloperations>`_ ( defaults to *MaxNoOfConcurrentOperations X 1.1* ) must be configured;

* in case of failure, the whole transaction is rolled back, not just the last statement;

* the data nodes allocates ~1KB for each MaxNoOfConcurrentOperations ;

What about DELETE?::
  
  mysql> DELETE FROM table1 LIMIT 50000;
  ERROR 1297 (HY000): Got temporary error 233 'Out of operation records in transaction coordinator (increase MaxNoOfConcurrentOperations)' from NDBCLUSTER
  mysql> SELECT COUNT(*) FROM table1;
  +----------+
  | COUNT(*) |
  +----------+
  |   237568 |
  +----------+
  1 row in set (0.00 sec)


