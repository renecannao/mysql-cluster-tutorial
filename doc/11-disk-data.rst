====
Disk Data
====


Create Undo Log Group
----

On one of the SQL node, run the CREATE LOGFILE GROUP command::
  
  mysql> CREATE LOGFILE GROUP lg_1 ADD UNDOFILE 'undo1.dat' INITIAL_SIZE=64M ENGINE=ndb;
  Query OK, 0 rows affected (8.37 sec)

Verify that the undo file is created on every data node. For example::
  
  root@node2:~# cd /mysqlcluster/ndb_2_fs/
  root@node2:/mysqlcluster/ndb_2_fs# ls -lh
  total 65M
  drwxr-x--- 4 root root 4.0K 2013-04-15 12:50 D1
  drwxr-x--- 3 root root 4.0K 2013-04-15 12:49 D10
  drwxr-x--- 3 root root 4.0K 2013-04-15 12:49 D11
  drwxr-x--- 4 root root 4.0K 2013-04-15 12:50 D2
  drwxr-x--- 3 root root 4.0K 2013-04-15 12:49 D8
  drwxr-x--- 3 root root 4.0K 2013-04-15 12:49 D9
  drwxr-x--- 4 root root 4.0K 2013-04-15 13:59 LCP
  -rw-r--r-- 1 root root  64M 2013-04-15 14:05 undo1.dat

Modify the Undo Log Group adding a new undo file::
  
  mysql> ALTER LOGFILE GROUP lg_1 ADD UNDOFILE 'undo2.dat' INITIAL_SIZE=32M ENGINE=ndb;
  Query OK, 0 rows affected (0.75 sec)

Verify that the new undo file is created on every data node. For example::
  
  root@node2:/mysqlcluster/ndb_2_fs# ls -lh
  total 97M
  drwxr-x--- 4 root root 4.0K 2013-04-15 12:50 D1
  drwxr-x--- 3 root root 4.0K 2013-04-15 12:49 D10
  drwxr-x--- 3 root root 4.0K 2013-04-15 12:49 D11
  drwxr-x--- 4 root root 4.0K 2013-04-15 12:50 D2
  drwxr-x--- 3 root root 4.0K 2013-04-15 12:49 D8
  drwxr-x--- 3 root root 4.0K 2013-04-15 12:49 D9
  drwxr-x--- 4 root root 4.0K 2013-04-15 13:59 LCP
  -rw-r--r-- 1 root root  64M 2013-04-15 14:05 undo1.dat
  -rw-r--r-- 1 root root  32M 2013-04-15 14:11 undo2.dat

Create tablespaces
----

Create the first tablespace with a first datafile::
  
  mysql> CREATE TABLESPACE ts1 ADD DATAFILE 'datafile1.dat'         
         USE LOGFILE GROUP lg_1
         INITIAL_SIZE=64M
         ENGINE=ndb;
  Query OK, 0 rows affected (6.36 sec)

Add a new datafile::
  
  mysql> ALTER TABLESPACE ts1 ADD DATAFILE 'datafile2.dat' 
         INITIAL_SIZE=64M
         ENGINE=ndb;
  Query OK, 0 rows affected (2.04 sec)

Create a new tablespace::
  
  mysql> CREATE TABLESPACE ts2 ADD DATAFILE 'ts2datafile1.dat' USE LOGFILE GROUP lg_1 INITIAL_SIZE=32M ENGINE=ndb;
  Query OK, 0 rows affected (0.34 sec)


Verify that the datafile were created on every data node. For example::
  
  root@node2:/mysqlcluster/ndb_2_fs# ls -lh
  total 257M
  drwxr-x--- 4 root root 4.0K 2013-04-15 12:50 D1
  drwxr-x--- 3 root root 4.0K 2013-04-15 12:49 D10
  drwxr-x--- 3 root root 4.0K 2013-04-15 12:49 D11
  drwxr-x--- 4 root root 4.0K 2013-04-15 12:50 D2
  drwxr-x--- 3 root root 4.0K 2013-04-15 12:49 D8
  drwxr-x--- 3 root root 4.0K 2013-04-15 12:49 D9
  -rw-r--r-- 1 root root  65M 2013-04-15 14:23 datafile1.dat
  -rw-r--r-- 1 root root  65M 2013-04-15 14:24 datafile2.dat
  -rw-r--r-- 1 root root  33M 2013-04-15 14:26 ts2datafile1.dat
  drwxr-x--- 4 root root 4.0K 2013-04-15 13:59 LCP
  -rw-r--r-- 1 root root  64M 2013-04-15 14:05 undo1.dat
  -rw-r--r-- 1 root root  32M 2013-04-15 14:11 undo2.dat



Alter table to use Data Disk
----

We will use table "table1" to test Data Disk. Let's check the table first::
  
  mysql> use mydb
  Database changed
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
  ) ENGINE=ndbcluster AUTO_INCREMENT=812523 DEFAULT CHARSET=latin1
  1 row in set (0.00 sec)
  
  mysql> SHOW TABLE STATUS LIKE 'table1'\G
  *************************** 1. row ***************************
             Name: table1
           Engine: ndbcluster
          Version: 10
       Row_format: Dynamic
             Rows: 235655
   Avg_row_length: 28
      Data_length: 17399808
  Max_data_length: 0
     Index_length: 0
        Data_free: 0
   Auto_increment: 812523
      Create_time: NULL
      Update_time: NULL
       Check_time: NULL
        Collation: latin1_swedish_ci
         Checksum: NULL
   Create_options: 
          Comment: 
  1 row in set (0.10 sec)
  
  
  
  
  ndb_mgm> ALL REPORT MEMORY
  Node 2: Data usage is 39%(1021 32K pages of total 2560)
  Node 2: Index usage is 23%(544 8K pages of total 2336)
  Node 3: Data usage is 39%(1021 32K pages of total 2560)
  Node 3: Index usage is 23%(544 8K pages of total 2336)
  
  
  
  root@node1:~# ndb_desc -p -d mydb table1
  -- table1 --
  Version: 83886090
  Fragment type: HashMapPartition
  K Value: 6
  Min load factor: 78
  Max load factor: 80
  Temporary table: no
  Number of attributes: 3
  Number of primary keys: 1
  Length of frm data: 297
  Row Checksum: 1
  Row GCI: 1
  SingleUserMode: 0
  ForceVarPart: 1
  FragmentCount: 2
  ExtraRowGciBits: 0
  ExtraRowAuthorBits: 0
  TableStatus: Retrieved
  HashMap: DEFAULT-HASHMAP-3840-2
  -- Attributes --
  id Int PRIMARY KEY DISTRIBUTION KEY AT=FIXED ST=MEMORY AUTO_INCR
  v Varchar(32;latin1_swedish_ci) NOT NULL AT=SHORT_VAR ST=MEMORY
  v2 Int NULL AT=FIXED ST=MEMORY DYNAMIC
  -- Indexes -- 
  PRIMARY KEY(id) - UniqueHashIndex
  PRIMARY(id) - OrderedIndex
  idx_v(v) - OrderedIndex
  idx_v2(v2) - OrderedIndex
  -- Per partition info -- 
  Partition       Row count       Commit count    Frag fixed memory       Frag varsized memory    Extent_space    Free extent_space
  0               116728          116728          3309568                 5308416                 0               0                 
  1               118927          118927          3375104                 5406720                 0               0                 
  
  
  NDBT_ProgramExit: 0 - OK
  
  
At this point it is not possible to convert the table to use Disk Data because all the columns are indexed: only not indexed column can be moved to Disk Data.

We can proceed removing an index, and run OPTIMIZE TABLE::
  
  mysql> ALTER TABLE table1 DROP INDEX idx_v2; OPTIMIZE TABLE table1;
  Query OK, 0 rows affected (0.32 sec)
  Records: 0  Duplicates: 0  Warnings: 0
  
  +-------------+----------+----------+----------+
  | Table       | Op       | Msg_type | Msg_text |
  +-------------+----------+----------+----------+
  | mydb.table1 | optimize | status   | OK       |
  +-------------+----------+----------+----------+
  1 row in set (39.67 sec)
  
  
  ndb_mgm> ALL REPORT MEMORY
  Node 2: Data usage is 34%(890 32K pages of total 2560)
  Node 2: Index usage is 23%(544 8K pages of total 2336)
  Node 3: Data usage is 34%(890 32K pages of total 2560)
  Node 3: Index usage is 23%(544 8K pages of total 2336)
  
We can now alter the table to use Disk Data::
  
  mysql> ALTER TABLE table1 TABLESPACE ts1 STORAGE DISK;
  Query OK, 0 rows affected (0.18 sec)
  Records: 0  Duplicates: 0  Warnings: 0
  
  mysql> SHOW TABLE STATUS LIKE 'table1'\G                                                                                                                                                   *************************** 1. row ***************************
             Name: table1
           Engine: ndbcluster
          Version: 10
       Row_format: Dynamic
             Rows: 211048
   Avg_row_length: 28
      Data_length: 16318464
  Max_data_length: 0
     Index_length: 0
        Data_free: 0
   Auto_increment: 272492
      Create_time: NULL
      Update_time: NULL
       Check_time: NULL
        Collation: latin1_swedish_ci
         Checksum: NULL
   Create_options:
          Comment:
  1 row in set (0.00 sec)
  
  mysql> SHOW CREATE TABLE table1\G
  *************************** 1. row ***************************
         Table: table1
  Create Table: CREATE TABLE `table1` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `v` varchar(32) NOT NULL,
    `v2` int(11) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `idx_v` (`v`)
  ) /*!50100 TABLESPACE ts1 STORAGE DISK */ ENGINE=ndbcluster AUTO_INCREMENT=272492 DEFAULT CHARSET=latin1
  1 row in set (0.00 sec)

  ndb_mgm> all report memory
  Node 2: Data usage is 32%(831 32K pages of total 2560)
  Node 2: Index usage is 19%(457 8K pages of total 2336)
  Node 3: Data usage is 32%(831 32K pages of total 2560)
  Node 3: Index usage is 19%(458 8K pages of total 2336)

  shell> ndb_desc -p -d mydb table1
  ...
  

Memory usage didn't change, and the table is still in memory, completely.
Let try a different ALTER TABLE::

  mysql> ALTER TABLE table1 MODIFY v2 int(11) DEFAULT NULL STORAGE DISK;
  Query OK, 211048 rows affected (5.28 sec)
  Records: 211048  Duplicates: 0  Warnings: 0

Now let's check ndb_desc::

  ndb_desc -p -d mydb table1
  -- table1 --
  Version: 16777218
  Fragment type: HashMapPartition
  K Value: 6
  Min load factor: 78
  Max load factor: 80
  Temporary table: no
  Number of attributes: 3
  Number of primary keys: 1
  Length of frm data: 317
  Row Checksum: 1
  Row GCI: 1
  SingleUserMode: 0
  ForceVarPart: 1
  FragmentCount: 2
  ExtraRowGciBits: 0
  ExtraRowAuthorBits: 0
  TableStatus: Retrieved
  HashMap: DEFAULT-HASHMAP-3840-2
  -- Attributes --
  id Int PRIMARY KEY DISTRIBUTION KEY AT=FIXED ST=MEMORY AUTO_INCR
  v Varchar(32;latin1_swedish_ci) NOT NULL AT=SHORT_VAR ST=MEMORY
  v2 Int NULL AT=FIXED ST=DISK
  -- Indexes --
  PRIMARY KEY(id) - UniqueHashIndex
  idx_v(v) - OrderedIndex
  PRIMARY(id) - OrderedIndex
  -- Per partition info --
  Partition       Row count       Commit count    Frag fixed memory       Frag varsized memory    Extent_space    Free extent_space
  0               105063          105063          3801088                 4784128                 3145728         1032180
  1               105985          105985          3833856                 4816896                 3145728         1013740
  
  
  NDBT_ProgramExit: 0 - OK

Memory usage::

  ndb_mgm> ALL REPORT MEMORY
  Node 2: Data usage is 33%(851 32K pages of total 2560)
  Node 2: Index usage is 20%(489 8K pages of total 2336)
  Node 3: Data usage is 33%(851 32K pages of total 2560)
  Node 3: Index usage is 20%(489 8K pages of total 2336) 


Why Data usage ( in memory ) increased? We can try to run OPTIMIZE after ALTER TABLE::
  
  mysql> OPTIMIZE TABLE table1;
  +-------------+----------+----------+----------+
  | Table       | Op       | Msg_type | Msg_text |
  +-------------+----------+----------+----------+
  | mydb.table1 | optimize | status   | OK       |
  +-------------+----------+----------+----------+
  1 row in set (39.64 sec)
  
  ndb_mgm> ALL REPORT MEMORY
  Node 2: Data usage is 33%(851 32K pages of total 2560)
  Node 2: Index usage is 20%(489 8K pages of total 2336)
  Node 3: Data usage is 33%(851 32K pages of total 2560)
  Node 3: Index usage is 20%(489 8K pages of total 2336) 


When Disk Data is used, each record uses an in-memory 8-byte pointer to the portion of the row stored in Data Disk. This means that Disk Data is convenient only if a lot of columns can be moved away from memory.
  
We can now try to move "v" in Disk Data::
  
  mysql> ALTER TABLE table1 DROP INDEX idx_v; OPTIMIZE TABLE table1;
  Query OK, 0 rows affected (0.17 sec)
  Records: 0  Duplicates: 0  Warnings: 0
  
  +-------------+----------+----------+----------+
  | Table       | Op       | Msg_type | Msg_text |
  +-------------+----------+----------+----------+
  | mydb.table1 | optimize | status   | OK       |
  +-------------+----------+----------+----------+
  1 row in set (12.18 sec)

Question: was column "v" already moved on Data Disk?

* SHOW CREATE TABLE doesn't return enough information;

* DROP INDEX returned 0 rows affected: this means it was an online operation, therefore performed in memory;

* DataMemory usage dropped by only 4% ::

  ndb_mgm> all report memory
  Node 2: Data usage is 29%(761 32K pages of total 2560)
  Node 2: Index usage is 20%(489 8K pages of total 2336)
  Node 3: Data usage is 29%(761 32K pages of total 2560)
  Node 3: Index usage is 20%(489 8K pages of total 2336)

Checking with ndb_desc::
  
  # ndb_desc -p -d mydb table1
  -- table1 --
  Version: 33554434
  Fragment type: HashMapPartition
  K Value: 6
  Min load factor: 78
  Max load factor: 80
  Temporary table: no
  Number of attributes: 3
  Number of primary keys: 1
  Length of frm data: 300
  Row Checksum: 1
  Row GCI: 1
  SingleUserMode: 0
  ForceVarPart: 1
  FragmentCount: 2
  ExtraRowGciBits: 0
  ExtraRowAuthorBits: 0
  TableStatus: Retrieved
  HashMap: DEFAULT-HASHMAP-3840-2
  -- Attributes --
  id Int PRIMARY KEY DISTRIBUTION KEY AT=FIXED ST=MEMORY AUTO_INCR
  v Varchar(32;latin1_swedish_ci) NOT NULL AT=SHORT_VAR ST=MEMORY
  v2 Int NULL AT=FIXED ST=DISK
  -- Indexes --
  PRIMARY KEY(id) - UniqueHashIndex
  PRIMARY(id) - OrderedIndex
  -- Per partition info --
  Partition       Row count       Commit count    Frag fixed memory       Frag varsized memory    Extent_space    Free extent_space
  0               105063          315189          3801088                 4784128                 3145728         1032180
  1               105985          317955          3833856                 4816896                 3145728         1013740
  
  
  NDBT_ProgramExit: 0 - OK

Only column "v2" seems to be on disk ( ST=DISK ), while column "v" is in memory ( ST=MEMORY ).

Moving column "v2" to disk::
  
  mysql> ALTER TABLE table1 MODIFY v varchar(32) NOT NULL STORAGE DISK;
  Query OK, 211048 rows affected (3.30 sec)
  Records: 211048  Duplicates: 0  Warnings: 0
  
  mysql> SHOW CREATE TABLE table1\G                                                                                                                                                          *************************** 1. row ***************************
         Table: table1
  Create Table: CREATE TABLE `table1` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `v` varchar(32) NOT NULL /*!50606 STORAGE DISK */,
    `v2` int(11) /*!50606 STORAGE DISK */ DEFAULT NULL,
    PRIMARY KEY (`id`)
  ) /*!50100 TABLESPACE ts1 STORAGE DISK */ ENGINE=ndbcluster AUTO_INCREMENT=395994 DEFAULT CHARSET=latin1
  1 row in set (0.00 sec)  

New memory usage::
  
  ndb_mgm> ALL REPORT MEMORY
  Node 2: Data usage is 18%(464 32K pages of total 2560)
  Node 2: Index usage is 20%(489 8K pages of total 2336)
  Node 3: Data usage is 18%(464 32K pages of total 2560)
  Node 3: Index usage is 20%(489 8K pages of total 2336)

New output of ndb_desc::
  
  # ndb_desc -p -d mydb table1
  -- table1 --
  Version: 16777219
  Fragment type: HashMapPartition
  K Value: 6
  Min load factor: 78
  Max load factor: 80
  Temporary table: no
  Number of attributes: 3
  Number of primary keys: 1
  Length of frm data: 296
  Row Checksum: 1
  Row GCI: 1
  SingleUserMode: 0
  ForceVarPart: 1
  FragmentCount: 2
  ExtraRowGciBits: 0
  ExtraRowAuthorBits: 0
  TableStatus: Retrieved
  HashMap: DEFAULT-HASHMAP-3840-2
  -- Attributes --
  id Int PRIMARY KEY DISTRIBUTION KEY AT=FIXED ST=MEMORY AUTO_INCR
  v Varchar(32;latin1_swedish_ci) NOT NULL AT=SHORT_VAR ST=DISK
  v2 Int NULL AT=FIXED ST=DISK
  -- Indexes --
  PRIMARY KEY(id) - UniqueHashIndex
  PRIMARY(id) - OrderedIndex
  -- Per partition info --
  Partition       Row count       Commit count    Frag fixed memory       Frag varsized memory    Extent_space    Free extent_space
  0               105063          105063          3801088                 0                       6291456         374136
  1               105985          105985          3833856                 0                       6291456         322504
  
  
  NDBT_ProgramExit: 0 - OK
  

Now both columns "v" and "v2" are on Data Disk.


Verify status of Disk Data files::
   
  mysql> USE INFORMATION_SCHEMA
  Reading table information for completion of table and column names
  You can turn off this feature to get a quicker startup with -A
  
  Database changed
  
  
  mysql> SHOW CREATE TABLE FILES\G
  ...
  
  mysql> SELECT FILE_NAME, FILE_TYPE, TABLESPACE_NAME, LOGFILE_GROUP_NAME, FREE_EXTENTS, TOTAL_EXTENTS, EXTENT_SIZE, INITIAL_SIZE FROM FILES WHERE ENGINE='NDBCLUSTER';
  +------------------+------------+-----------------+--------------------+--------------+---------------+-------------+--------------+
  | FILE_NAME        | FILE_TYPE  | TABLESPACE_NAME | LOGFILE_GROUP_NAME | FREE_EXTENTS | TOTAL_EXTENTS | EXTENT_SIZE | INITIAL_SIZE |
  +------------------+------------+-----------------+--------------------+--------------+---------------+-------------+--------------+
  | datafile1.dat    | DATAFILE   | ts1             | lg_1               |           64 |            64 |     1048576 |     67108864 |
  | datafile1.dat    | DATAFILE   | ts1             | lg_1               |           64 |            64 |     1048576 |     67108864 |
  | ts2datafile1.dat | DATAFILE   | ts2             | lg_1               |           32 |            32 |     1048576 |     33554432 |
  | ts2datafile1.dat | DATAFILE   | ts2             | lg_1               |           32 |            32 |     1048576 |     33554432 |
  | datafile2.dat    | DATAFILE   | ts1             | lg_1               |           52 |            64 |     1048576 |     67108864 |
  | datafile2.dat    | DATAFILE   | ts1             | lg_1               |           52 |            64 |     1048576 |     67108864 |
  | NULL             | TABLESPACE | ts1             | lg_1               |         NULL |          NULL |     1048576 |         NULL |
  | NULL             | TABLESPACE | ts2             | lg_1               |         NULL |          NULL |     1048576 |         NULL |
  | undo1.dat        | UNDO LOG   | NULL            | lg_1               |         NULL |      16777216 |           4 |     67108864 |
  | undo1.dat        | UNDO LOG   | NULL            | lg_1               |         NULL |      16777216 |           4 |     67108864 |
  | undo2.dat        | UNDO LOG   | NULL            | lg_1               |         NULL |       8388608 |           4 |     33554432 |
  | undo2.dat        | UNDO LOG   | NULL            | lg_1               |         NULL |       8388608 |           4 |     33554432 |
  | NULL             | UNDO LOG   | NULL            | lg_1               |     24432992 |          NULL |           4 |         NULL |
  +------------------+------------+-----------------+--------------------+--------------+---------------+-------------+--------------+
  13 rows in set (0.04 sec)
  
  mysql> SELECT TABLESPACE_NAME `TABLESPACE`, SUM(FREE_EXTENTS*EXTENT_SIZE)/(1024*1024) `Free MB`, SUM(TOTAL_EXTENTS*EXTENT_SIZE)/(1024*1024) `Total MB` FROM FILES WHERE ENGINE='NDBCLUSTER' AND FILE_TYPE='DATAFILE' GROUP BY TABLESPACE_NAME;
  +------------+----------+----------+
  | TABLESPACE | Free MB  | Total MB |
  +------------+----------+----------+
  | ts1        | 232.0000 | 256.0000 |
  | ts2        |  64.0000 |  64.0000 |
  +------------+----------+----------+
  2 rows in set (0.05 sec)
  
  
  
  
  
  mysql> use ndbinfo;
  Reading table information for completion of table and column names
  You can turn off this feature to get a quicker startup with -A
  
  Database changed
  mysql> SELECT * FROM logspaces WHERE log_type='DD-UNDO';
  +---------+----------+--------+----------+-----------+--------+
  | node_id | log_type | log_id | log_part | total     | used   |
  +---------+----------+--------+----------+-----------+--------+
  |       2 | DD-UNDO  |     25 |        0 | 100663296 | 212912 |
  |       3 | DD-UNDO  |     25 |        0 | 100663296 | 212912 |
  +---------+----------+--------+----------+-----------+--------+
  2 rows in set (0.01 sec)
  
