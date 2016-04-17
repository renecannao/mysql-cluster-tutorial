====
MySQL Cluster Version Upgrade
====

Upgrade in MySQL Cluster is often possible with zero downtime.

Upgraded (and downgrades) are performed through rolling restarts.

Review sections `Upgrading and Downgrading MySQL Cluster NDB 7.2 <http://dev.mysql.com/doc/refman/5.6/en/mysql-cluster-upgrade-downgrade.html>`_ and `Upgrading and Downgrading MySQL Cluster <http://dev.mysql.com/doc/refman/5.6/en/mysql-cluster-upgrade-downgrade.html>`_ for a list of compatibility notes and issues when upgrading and/or upgrading MySQL Cluster.


Binaries can be downloaded from `MySQL website <http://dev.mysql.com/downloads/cluster/>`_ or `MySQL Archives <http://downloads.mysql.com/archives.php>`_ .

For convenience, binaries are present in the home directory and are the generic Linux version for 64 bits system.

Unpack the tarball into each guest node and prepare the datadir::

  guest> cd /usr/local
  guest> sudo tar -zxf ~/mysql-cluster-gpl-7.4.10-linux-glibc2.5-x86_64.tar.gz
  guest> sudo su -
  guest> cd /usr/local
  guest> rm mysql
  guest> ln -s mysql-cluster-gpl-7.4.10-linux-glibc2.5-x86_64 mysql

Verify that Cluster is up and running. Verify also the version::
  
  ndb_mgm> SHOW
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2    @192.168.123.102  (mysql-5.6.28 ndb-7.4.9, Nodegroup: 0, Master)
  id=3    @192.168.123.103  (mysql-5.6.28 ndb-7.4.9, Nodegroup: 0)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.6.28 ndb-7.4.9)
  
  [mysqld(API)]   4 node(s)
  id=11   @192.168.123.103  (mysql-5.6.28 ndb-7.4.9)
  id=12   @192.168.123.102  (mysql-5.6.28 ndb-7.4.9)
  id=13   @192.168.123.101  (mysql-5.6.28 ndb-7.4.9)
  id=14 (not connected, accepting connect from any host)


Restart the Management Node
~~~~

Kill the management node ( ndb_mgmd ) and restart it::
  
  node1> killall ndb_mgmd
  node1> ndb_mgmd --config-dir=/mysqlcluster/ --config-file=/mysqlcluster/config.ini

Check the status from ndb_mgm::
  
  ndb_mgm> SHOW
  Connected to Management Server at: 192.168.123.101:1186
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2    @192.168.123.102  (mysql-5.6.28 ndb-7.4.9, Nodegroup: 0, Master)
  id=3    @192.168.123.103  (mysql-5.6.28 ndb-7.4.9, Nodegroup: 0)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.6.28 ndb-7.4.10)
  
  [mysqld(API)]   4 node(s)
  id=11   @192.168.123.103  (mysql-5.6.28 ndb-7.4.9)
  id=12   @192.168.123.102  (mysql-5.6.28 ndb-7.4.9)
  id=13   @192.168.123.101  (mysql-5.6.28 ndb-7.4.9)
  id=14 (not connected, accepting connect from any host)


Restart the Data Nodes
~~~~

Kill the data node2::
  
  node2> killall ndbd


Verify the status::
  
  ndb_mgm> Node 2: Node shutdown completed. Initiated by signal 15.
   
  ndb_mgm> SHOW
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2 (not connected, accepting connect from 192.168.123.102)
  id=3    @192.168.123.103  (mysql-5.6.28 ndb-7.4.9, Nodegroup: 0, Master)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.6.28 ndb-7.4.10)
  
  [mysqld(API)]   4 node(s)
  id=11   @192.168.123.103  (mysql-5.6.28 ndb-7.4.9)
  id=12   @192.168.123.102  (mysql-5.6.28 ndb-7.4.9)
  id=13   @192.168.123.101  (mysql-5.6.28 ndb-7.4.9)
  id=14 (not connected, accepting connect from any host)


Start the data node on node2::
  
  node2> ndbd

Wait and verify that Data Node was started successfully::
  
  ndb_mgm> Node 2: Started (version 7.2.10)
  
  ndb_mgm> SHOW
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2    @192.168.123.102  (mysql-5.6.28 ndb-7.4.10, Nodegroup: 0)
  id=3    @192.168.123.103  (mysql-5.6.28 ndb-7.4.9, Nodegroup: 0, Master)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.6.28 ndb-7.4.10)
  
  [mysqld(API)]   4 node(s)
  id=11   @192.168.123.103  (mysql-5.6.28 ndb-7.4.9)
  id=12   @192.168.123.102  (mysql-5.6.28 ndb-7.4.9)
  id=13   @192.168.123.101  (mysql-5.6.28 ndb-7.4.9)
  id=14 (not connected, accepting connect from any host)


Repeat the same for node3.

Verify the status of the Cluster::
  
  ndb_mgm> SHOW
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2    @192.168.123.102  (mysql-5.6.28 ndb-7.4.10, Nodegroup: 0, Master)
  id=3    @192.168.123.103  (mysql-5.6.28 ndb-7.4.10, Nodegroup: 0)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.6.28 ndb-7.4.10)
  
  [mysqld(API)]   4 node(s)
  id=11   @192.168.123.103  (mysql-5.6.28 ndb-7.4.9)
  id=12   @192.168.123.102  (mysql-5.6.28 ndb-7.4.9)
  id=13   @192.168.123.101  (mysql-5.6.28 ndb-7.4.9)
  id=14 (not connected, accepting connect from any host)


Management Node and Data Nodes are now restarted. Now is the turn to restart the SQL Nodes.

Restart mysqld on Node2::
  
  root@node2:~# service mysql restart  
  Shutting down MySQL
  ... * 
  Starting MySQL
  .. * 
  root@node2:~# 

... and on Node3::
  
  root@node3:~# service mysql restart
  Shutting down MySQL
  . * 
  Starting MySQL
  . * 

... and on Node1::
  
  root@node3:~# service mysql restart
  Shutting down MySQL
  . * 
  Starting MySQL
  .. * 

Don't forget to run mysql_upgrade on all SQL nodes::
  
  root@node2:~# mysql_upgrade 
  Looking for 'mysql' as: mysql
  Looking for 'mysqlcheck' as: mysqlcheck
  ...
  Running 'mysql_fix_privilege_tables'...
  OK

Verify the status of the Cluster::
  
  ndb_mgm> SHOW
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2    @192.168.123.102  (mysql-5.6.28 ndb-7.4.10, Nodegroup: 0, Master)
  id=3    @192.168.123.103  (mysql-5.6.28 ndb-7.4.10, Nodegroup: 0)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.6.28 ndb-7.4.10)
  
  [mysqld(API)]   4 node(s)
  id=11   @192.168.123.101  (mysql-5.6.28 ndb-7.4.10)
  id=12   @192.168.123.103  (mysql-5.6.28 ndb-7.4.10)
  id=13 (not connected, accepting connect from any host)
  id=14   @192.168.123.102  (mysql-5.6.28 ndb-7.4.10)

The whole Cluster is now upgraded from 5.6.28 ndb-7.4.9 to 5.6.28 ndb-7.4.10

