====
Initial node start
====

Walk through on DataNode Filesystem
====

Verify the content of the DataNode fileystem::
  
  root@node2:~# cd /mysqlcluster/   
  root@node2:/mysqlcluster# ls -l

The files in the DataDir are:

* error log

* output file

* pid file

* trace files

* directory named ndb_*nodeid*_fs

Content of ndb filesystem::
  
  root@node2:/mysqlcluster# cd ndb_2_fs/
  root@node2:/mysqlcluster/ndb_2_fs# ls -l
  total 28
  drwxr-x--- 4 root root 4096 2013-04-13 20:32 D1
  drwxr-x--- 3 root root 4096 2013-04-13 20:32 D10
  drwxr-x--- 3 root root 4096 2013-04-13 20:32 D11
  drwxr-x--- 4 root root 4096 2013-04-13 20:32 D2
  drwxr-x--- 3 root root 4096 2013-04-13 20:32 D8
  drwxr-x--- 3 root root 4096 2013-04-13 20:32 D9
  drwxr-x--- 4 root root 4096 2013-04-13 21:29 LCP

Contents of the above directories:

* D1 and D2 : store information about data dictionary ( DBDICT ) and partitions ( DBDIH );

* D8 , D9 , D10 and D11 : RedoLog files . 4 sets. Each directory has NoOfFragmentLogFiles files of size FragmentLogFileSize 

* LCP : 2 local checkpoints

Simulate a severe failure
====

Run a script to writes data on MySQL Cluster::
  
  node3> for i in `seq 1 600` ; do date ; echo "INSERT INTO table1 (id,v) VALUES (NULL,MD5(RAND()));" | mysql mydb ; sleep 1 ; done

Find the files that are recently modified int the Data Node filesystem, and delete them::
  
  root@node2:/mysqlcluster/ndb_2_fs# find . -mmin -1
  root@node2:/mysqlcluster/ndb_2_fs# find . -mmin -1 -exec rm {} \;
  root@node2:/mysqlcluster/ndb_2_fs# find . -mmin -1

Did the Datanode crashed?::
  
  ndb_mgm> SHOW
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2    @192.168.123.102  (mysql-5.5.29 ndb-7.2.10, Nodegroup: 0, Master)
  id=3    @192.168.123.103  (mysql-5.5.29 ndb-7.2.10, Nodegroup: 0)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.5.29 ndb-7.2.10)
  
  [mysqld(API)]   4 node(s)
  id=11   @192.168.123.101  (mysql-5.5.29 ndb-7.2.10)
  id=12   @192.168.123.102  (mysql-5.5.29 ndb-7.2.10)
  id=13   @192.168.123.103  (mysql-5.5.29 ndb-7.2.10)
  id=14 (not connected, accepting connect from any host)

No! Why? ::
  
  root@node2:/mysqlcluster/ndb_2_fs# lsof -n | grep ndbd | grep deleted

The files are not really removed because they are still open. Kill ndbd to force the remove::

  root@node2:/mysqlcluster/ndb_2_fs# killall ndbd

Check the status of the Cluster::
  
  ndb_mgm> Node 2: Node shutdown completed. Initiated by signal 15.
  
  ndb_mgm> SHOW
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2 (not connected, accepting connect from 192.168.123.102)
  id=3    @192.168.123.103  (mysql-5.5.29 ndb-7.2.10, Nodegroup: 0, Master)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.5.29 ndb-7.2.10)
  
  [mysqld(API)]   4 node(s)
  id=11   @192.168.123.101  (mysql-5.5.29 ndb-7.2.10)
  id=12   @192.168.123.102  (mysql-5.5.29 ndb-7.2.10)
  id=13   @192.168.123.103  (mysql-5.5.29 ndb-7.2.10)
  id=14 (not connected, accepting connect from any host)

Restart the node::
  
  root@node2:~# ndbd

Does the data node comes back online? Yes::
  
  ndb_mgm> Node 2: Started (version 7.2.10)
  
  ndb_mgm> SHOW
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2    @192.168.123.102  (mysql-5.5.29 ndb-7.2.10, Nodegroup: 0)
  id=3    @192.168.123.103  (mysql-5.5.29 ndb-7.2.10, Nodegroup: 0, Master)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.5.29 ndb-7.2.10)
  
  [mysqld(API)]   4 node(s)
  id=11   @192.168.123.101  (mysql-5.5.29 ndb-7.2.10)
  id=12   @192.168.123.102  (mysql-5.5.29 ndb-7.2.10)
  id=13   @192.168.123.103  (mysql-5.5.29 ndb-7.2.10)
  id=14 (not connected, accepting connect from any host)

Let's try to corrupt the filesystem even more::
  
  root@node2:/mysqlcluster/ndb_2_fs# rm -rf LCP/* && killall ndbd

Restart ndbd::
  
  root@node2:~# ndbd

From ndb_mgm::
  
  ndb_mgm> Node 2: Forced node shutdown completed. Occured during startphase 5. Caused by error 2341: 'Internal program error (failed ndbrequire)(Internal error, programming error or missing error message, please report a bug). Temporary error, restart node'.

Node2 is now failing to start. More information in these files::
  
  root@node2:~# cat /mysqlcluster/ndb_2_error.log
  root@node2:~# cat /mysqlcluster/ndb_2_out.log
  root@node2:~# cat /mysqlcluster/ndb_2_trace.log.2


Perform an Initial Node Restart
====

Start the data node process with --initial ::
  
  root@node2:~# ndbd --initial

Did the data node start successfully now?::
  
  ndb_mgm> Node 2: Started (version 7.2.10)
  
  ndb_mgm> SHOW
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2    @192.168.123.102  (mysql-5.5.29 ndb-7.2.10, Nodegroup: 0)
  id=3    @192.168.123.103  (mysql-5.5.29 ndb-7.2.10, Nodegroup: 0, Master)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.5.29 ndb-7.2.10)
  
  [mysqld(API)]   4 node(s)
  id=11   @192.168.123.101  (mysql-5.5.29 ndb-7.2.10)
  id=12   @192.168.123.102  (mysql-5.5.29 ndb-7.2.10)
  id=13   @192.168.123.103  (mysql-5.5.29 ndb-7.2.10)
  id=14 (not connected, accepting connect from any host)

