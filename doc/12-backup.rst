====
Backup
====

Backup with mysqldump and restore
====

Perform a backup with mysqldump::
  
  root@node2:~# mysqldump world > world.sql

Check the content of the dump file::
  
  root@node2:~# less world.sql

From another node, drop the database::
    
  root@node3:~# mysql -e "SHOW DATABASES"     
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mydb               |
  | mysql              |
  | ndbinfo            |
  | performance_schema |
  | test               |
  | world              |
  +--------------------+
  root@node3:~# mysql -e "DROP DATABASE world"
  root@node3:~# mysql -e "SHOW DATABASES"     
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mydb               |
  | mysql              |
  | ndbinfo            |
  | performance_schema |
  | test               |
  +--------------------+

Verify that the world schema is not available on other SQL Nodes too. For example::
  
  root@node2:~# mysql -e "SHOW DATABASES"
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mydb               |
  | mysql              |
  | ndbinfo            |
  | performance_schema |
  | test               |
  +--------------------+

Restore the backup::
  
  root@node2:~# mysqladmin create world
  root@node2:~# mysql world < world.sql 

Verify that the backup is now restored on all SQL Nodes. For example::
  
  root@node1:~# mysql -e "SHOW DATABASES"
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mydb               |
  | mysql              |
  | ndbinfo            |
  | performance_schema |
  | test               |
  | world              |
  +--------------------+


Point in time recovery
----

Point in time recovery is not that simple as restoring a full backup, and is not a topic discussed in this tutorial.


Native NDB Online Backup
====


Backups are started from the Management Node::
  
  ndb_mgm> START BACKUP
  Waiting for completed, this may take several minutes
  Node 2: Backup 1 started from node 1
  Node 2: Backup 1 started from node 1 completed
   StartGCP: 61949 StopGCP: 61952
   #Records: 302613 #LogRecords: 0
   Data: 15297740 bytes Log: 0 bytes

During this first backup, the Cluster was not being written as there were no clients connected.

Starting a script that continuously writes data in the cluster::
  
  root@node3:~# for i in `seq 1 600` ; do echo "INSERT INTO table1 (id,v) SELECT NULL,MD5(RAND()) FROM table1 LIMIT 1024;" | mysql mydb ; sleep 0.1 ; done

And starting a backup after few seconds::
  
  ndb_mgm> START BACKUP
  Waiting for completed, this may take several minutes
  Node 2: Backup 2 started from node 1
  Node 2: Backup 2 started from node 1 completed
   StartGCP: 62136 StopGCP: 62139
   #Records: 313877 #LogRecords: 2112
   Data: 15883468 bytes Log: 157960 bytes

Backups are automatically stored in a directory named "BACKUP" in the path specified by the variable "BackupDataDir" ( that defaults to FileSystemPath )::
  
  root@node2:/mysqlcluster/BACKUP# ls -l
  total 8
  drwxr-x--- 2 root root 4096 2013-04-20 00:02 BACKUP-1
  drwxr-x--- 2 root root 4096 2013-04-20 00:09 BACKUP-2
  
  root@node3:~# cd /mysqlcluster/BACKUP/
  root@node3:/mysqlcluster/BACKUP# ls -l
  total 8
  drwxr-x--- 2 root root 4096 2013-04-20 00:02 BACKUP-1
  drwxr-x--- 2 root root 4096 2013-04-20 00:09 BACKUP-2

Each backup creates a new directory, and the content is the follow::
  
  root@node2:/mysqlcluster/BACKUP# ls -l BACKUP-2
  total 7828
  -rw-r--r-- 1 root root 7889508 2013-04-20 00:09 BACKUP-2-0.2.Data
  -rw-r--r-- 1 root root   37516 2013-04-20 00:09 BACKUP-2.2.ctl
  -rw-r--r-- 1 root root   78436 2013-04-20 00:09 BACKUP-2.2.log
  
  root@node3:/mysqlcluster/BACKUP# ls -l BACKUP-2
  total 7928
  -rw-r--r-- 1 root root 7994840 2013-04-20 00:09 BACKUP-2-0.3.Data
  -rw-r--r-- 1 root root   37516 2013-04-20 00:09 BACKUP-2.3.ctl
  -rw-r--r-- 1 root root   79628 2013-04-20 00:09 BACKUP-2.3.log


Restore from NDB Backup using ndb_restore
====

Drop a database with NDB Cluster tables::
  
  root@node1:~# mysql -e "SHOW SCHEMAS"
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mydb               |
  | mysql              |
  | ndbinfo            |
  | performance_schema |
  | test               |
  | world              |
  +--------------------+
  root@node1:~# mysql -e "DROP DATABASE mydb"
  root@node1:~# mysql -e "SHOW SCHEMAS"
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mysql              |
  | ndbinfo            |
  | performance_schema |
  | test               |
  | world              |
  +--------------------+
  
Verify with ndb_desc that the table "table1" doesn't exist anymore::
  
  root@node1:~# ndb_desc -p -d mydb table1
  No such object: table1
  
  
  NDBT_ProgramExit: 0 - OK
  
  
  
Use ndb_restore to reimport the metadata::  
  
  root@node2:/mysqlcluster/BACKUP# cd BACKUP-2
  root@node2:/mysqlcluster/BACKUP/BACKUP-2# ndb_restore -m -n 2 -b 2 --include-databases=mydb
  Nodeid = 2
  Backup Id = 2
  backup path = ./
  Including Databases: mydb 
  Opening file './BACKUP-2.2.ctl'
  File size 37516 bytes
  Backup version in files: ndb-6.3.11 ndb version: mysql-5.5.29 ndb-7.2.10
  Stop GCP of Backup: 62138
  Connected to ndb!!
  Creating logfile group: lg_1...FAILED
  Create logfile group failed: lg_1: 721: Schema object with given name already exists
  Restore: Failed to restore table: mysql/def/NDB$BLOB_7_3 ... Exiting 
  
  NDBT_ProgramExit: 1 - Failed
  
  
By default, ndb_restore tries also to rebuild Disk Data objects (undo log group and tablespaces). If these objects already exists, do not try to restore them::
  
  root@node2:/mysqlcluster/BACKUP/BACKUP-2# ndb_restore --restore-meta --nodeid 2 --backupid 2 --include-databases=mydb --no-restore-disk-objects
  Nodeid = 2
  Backup Id = 2
  backup path = ./
  Including Databases: mydb 
  Opening file './BACKUP-2.2.ctl'
  File size 37516 bytes
  Backup version in files: ndb-6.3.11 ndb version: mysql-5.5.29 ndb-7.2.10
  Stop GCP of Backup: 62138
  Connected to ndb!!
  Successfully restored table `mydb/def/table1`
  Successfully restored table event REPL$mydb/table1
  Successfully restored table `mydb/def/table2`
  Successfully restored table event REPL$mydb/table2
  Successfully created index `idx_v2` on `table1`
  Successfully created index `PRIMARY` on `table1`
  Successfully created index `PRIMARY` on `table2`
  
  NDBT_ProgramExit: 0 - OK
  
At this state the tables are created, but empty::
  
  root@node2:/mysqlcluster/BACKUP/BACKUP-2# ndb_desc -p -d mydb table1
  -- table1 --
  Version: 13
  Fragment type: HashMapPartition
  K Value: 6
  Min load factor: 78
  Max load factor: 80
  Temporary table: no
  Number of attributes: 3
  Number of primary keys: 1
  Length of frm data: 293
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
  v2 Int NULL AT=FIXED ST=MEMORY
  -- Indexes -- 
  PRIMARY KEY(id) - UniqueHashIndex
  PRIMARY(id) - OrderedIndex
  idx_v2(v2) - OrderedIndex
  -- Per partition info -- 
  Partition       Row count       Commit count    Frag fixed memory       Frag varsized memory    Extent_space    Free extent_space
  
  
  NDBT_ProgramExit: 0 - OK
  
  
The API nodes are not aware of the new database::
  
  root@node1:~# mysql -e "SHOW DATABASES"
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mysql              |
  | ndbinfo            |
  | performance_schema |
  | test               |
  | world              |
  +--------------------+
  
When a new database is imported via ndb_restore , we still need to issue CREATE DATABASE in one (any) of the API node, and the tables will automatically appears in all the API nodes::
  
  root@node2:/mysqlcluster/BACKUP/BACKUP-2# mysqladmin create mydb
  
  root@node1:~# mysql -e "SHOW DATABASES"
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mydb               |
  | mysql              |
  | ndbinfo            |
  | performance_schema |
  | test               |
  | world              |
  +--------------------+
  root@node1:~# mysql -e "SHOW TABLES" mydb
  +----------------+
  | Tables_in_mydb |
  +----------------+
  | table1         |
  | table3         |
  | table2         |
  | tbl1           |
  +----------------+
  
As already seen in ndb_desc , the table is empty. This is expected, as so far we only imported the metadata::
  
  root@node1:~# mysql -e "SELECT COUNT(*) FROM table1" mydb
  +----------+
  | COUNT(*) |
  +----------+
  |        0 |
  +----------+

Check the memory usage before importing the data::
  
  ndb_mgm> ALL REPORT MEMORY
  Node 2: Data usage is 2%(60 32K pages of total 2560)
  Node 2: Index usage is 2%(53 8K pages of total 2336)
  Node 3: Data usage is 2%(60 32K pages of total 2560)
  Node 3: Index usage is 2%(53 8K pages of total 2336)
  
Restore the data from node2::
  
  root@node2:/mysqlcluster/BACKUP/BACKUP-2# ndb_restore --restore-data --nodeid 2 --backupid 2 --include-databases=mydb
  Nodeid = 2
  Backup Id = 2
  backup path = ./
  Including Databases: mydb
  Opening file './BACKUP-2.2.ctl'
  File size 37516 bytes
  Backup version in files: ndb-6.3.11 ndb version: mysql-5.5.29 ndb-7.2.10
  Stop GCP of Backup: 62138
  Connected to ndb!!
  Opening file './BACKUP-2-0.2.Data'
  File size 7889508 bytes
  _____________________________________________________
  Processing data in table: mysql/def/NDB$BLOB_7_3(8) fragment 0
  _____________________________________________________
  Processing data in table: mysql/def/ndb_index_stat_sample(5) fragment 0
  _____________________________________________________
  Processing data in table: world/def/CountryLanguage(15) fragment 0
  _____________________________________________________
  Processing data in table: sys/def/NDB$EVENTS_0(3) fragment 0
  _____________________________________________________
  Processing data in table: mysql/def/ndb_apply_status(9) fragment 0
  _____________________________________________________
  Processing data in table: mysql/def/ndb_index_stat_head(4) fragment 0
  _____________________________________________________
  Processing data in table: mydb/def/table1(23) fragment 0
  _____________________________________________________
  Processing data in table: mydb/def/table2(18) fragment 0
  _____________________________________________________
  Processing data in table: mydb/def/table3(21) fragment 0
  _____________________________________________________
  Processing data in table: mydb/def/tbl1(20) fragment 0
  _____________________________________________________
  Processing data in table: world/def/City(11) fragment 0
  _____________________________________________________
  Processing data in table: sys/def/SYSTAB_0(2) fragment 0
  _____________________________________________________
  Processing data in table: mysql/def/ndb_schema(7) fragment 0
  _____________________________________________________
  Processing data in table: world/def/Country(13) fragment 0
  Opening file './BACKUP-2.2.log'
  File size 78436 bytes
  Restored 122310 tuples and 1001 log entries
  
  NDBT_ProgramExit: 0 - OK
  
  
Verify memory usage::
  
  ndb_mgm> ALL REPORT MEMORY
  Node 2: Data usage is 12%(319 32K pages of total 2560)
  Node 2: Index usage is 13%(311 8K pages of total 2336)
  Node 3: Data usage is 13%(333 32K pages of total 2560)
  Node 3: Index usage is 13%(311 8K pages of total 2336)
  
  
  
The table is now half full::
  
  mysql> USE mydb
  Database changed
  mysql> SELECT COUNT(*) FROM table1;
  +----------+
  | COUNT(*) |
  +----------+
  |   122309 |
  +----------+
  1 row in set (0.02 sec)
  
  
All the data is in partition 0::
  
  root@node1:~# ndb_desc -p -d mydb table1
  -- table1 --
  Version: 15
  Fragment type: HashMapPartition
  K Value: 6
  Min load factor: 78
  Max load factor: 80
  Temporary table: no
  Number of attributes: 3
  Number of primary keys: 1
  Length of frm data: 293
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
  v2 Int NULL AT=FIXED ST=MEMORY
  -- Indexes -- 
  PRIMARY KEY(id) - UniqueHashIndex
  PRIMARY(id) - OrderedIndex
  idx_v2(v2) - OrderedIndex
  -- Per partition info -- 
  Partition       Row count       Commit count    Frag fixed memory       Frag varsized memory    Extent_space    Free extent_space
  0               122309          122309          5439488                 0                       6291456         396048           
  
  
  NDBT_ProgramExit: 0 - OK
  
  
  
Restore the data from node3::
  
  root@node3:/mysqlcluster/BACKUP# cd BACKUP-2/
  root@node3:/mysqlcluster/BACKUP/BACKUP-2# ls -l
  total 7928
  -rw-r--r-- 1 root root 7994840 2013-04-20 00:09 BACKUP-2-0.3.Data
  -rw-r--r-- 1 root root   37516 2013-04-20 00:09 BACKUP-2.3.ctl
  -rw-r--r-- 1 root root   79628 2013-04-20 00:09 BACKUP-2.3.log
  root@node3:/mysqlcluster/BACKUP/BACKUP-2# ndb_restore --restore-data --nodeid 3 --backupid 2 --include-databases=mydb
  Nodeid = 3
  Backup Id = 2
  backup path = ./
  Including Databases: mydb 
  Opening file './BACKUP-2.3.ctl'
  File size 37516 bytes
  Backup version in files: ndb-6.3.11 ndb version: mysql-5.5.29 ndb-7.2.10
  Stop GCP of Backup: 62138
  Connected to ndb!!
  Opening file './BACKUP-2-0.3.Data'
  File size 7994840 bytes
  _____________________________________________________
  Processing data in table: mysql/def/NDB$BLOB_7_3(8) fragment 1
  _____________________________________________________
  Processing data in table: mysql/def/ndb_index_stat_sample(5) fragment 1
  _____________________________________________________
  Processing data in table: world/def/CountryLanguage(15) fragment 1
  _____________________________________________________
  Processing data in table: sys/def/NDB$EVENTS_0(3) fragment 1
  _____________________________________________________
  Processing data in table: mysql/def/ndb_apply_status(9) fragment 1
  _____________________________________________________
  Processing data in table: mysql/def/ndb_index_stat_head(4) fragment 1
  _____________________________________________________
  Processing data in table: mydb/def/table1(23) fragment 1
  _____________________________________________________
  Processing data in table: mydb/def/table2(18) fragment 1
  _____________________________________________________
  Processing data in table: mydb/def/table3(21) fragment 1
  _____________________________________________________
  Processing data in table: mydb/def/tbl1(20) fragment 1
  _____________________________________________________
  Processing data in table: world/def/City(11) fragment 1
  _____________________________________________________
  Processing data in table: sys/def/SYSTAB_0(2) fragment 1
  _____________________________________________________
  Processing data in table: mysql/def/ndb_schema(7) fragment 1
  _____________________________________________________
  Processing data in table: world/def/Country(13) fragment 1
  Opening file './BACKUP-2.3.log'
  File size 79628 bytes
  Restored 124612 tuples and 1047 log entries
  
  NDBT_ProgramExit: 0 - OK
  
Verify memory usage::
  
  ndb_mgm> ALL REPORT MEMORY
  Node 2: Data usage is 23%(590 32K pages of total 2560)
  Node 2: Index usage is 24%(578 8K pages of total 2336)
  Node 3: Data usage is 23%(604 32K pages of total 2560)
  Node 3: Index usage is 24%(577 8K pages of total 2336)
  
The table is now full::
  
  mysql> SELECT COUNT(*) FROM table1;
  +----------+
  | COUNT(*) |
  +----------+
  |   246919 |
  +----------+
  1 row in set (0.01 sec)

Verify with ndb_desc::
  
  root@node1:~# ndb_desc -p -d mydb table1
  -- table1 --
  Version: 15
  Fragment type: HashMapPartition
  K Value: 6
  Min load factor: 78
  Max load factor: 80
  Temporary table: no
  Number of attributes: 3
  Number of primary keys: 1
  Length of frm data: 293
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
  v2 Int NULL AT=FIXED ST=MEMORY
  -- Indexes -- 
  PRIMARY KEY(id) - UniqueHashIndex
  PRIMARY(id) - OrderedIndex
  idx_v2(v2) - OrderedIndex
  -- Per partition info -- 
  Partition       Row count       Commit count    Frag fixed memory       Frag varsized memory    Extent_space    Free extent_space
  0               122309          122309          5439488                 0                       6291456         396048           
  1               124610          124610          5537792                 0                       6291456         285600           
  
  
  NDBT_ProgramExit: 0 - OK  
