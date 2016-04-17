====
MySQL Cluster Installation and Configuration
====

Installation
----

Binaries can be downloaded from `MySQL website <http://dev.mysql.com/downloads/cluster/>`_ or `MySQL Archives <http://downloads.mysql.com/archives.php>`_ .

For convenience, binaries are already present on all the EC2 instances.

Unpack the tarball into each guest node and prepare the datadir::

  guest> cd /usr/local
  guest> sudo tar -zxf ~/mysql-cluster-gpl-7.4.9-linux-glibc2.5-x86_64.tar.gz
  guest> sudo su -
  guest> cd /usr/local
  guest> ln -s mysql-cluster-gpl-7.4.9-linux-glibc2.5-x86_64 mysql
  guest> export PATH=/usr/local/mysql/bin:$PATH
  guest> mkdir /mysqlcluster

For semplicity, add to /root/.bashrc ::

  export PATH=/usr/local/mysql/bin:$PATH

Create mysql user/group::

  guest> useradd mysql


Configuration file
----

The *global configuration file* for MySQL Cluster is an INI format file named *config.ini* divided in sections.

Sections in config.ini are:

* **[computer]** : defines cluster hosts;

* **[ndbd]** : defines data node processes;

* **[mysqld]** or **[api]** : defines API nodes;

* **[mgm]** or **[ndb_mgmd]** : defines management nodes;

* **[tcp]** : defines TCP/IP connections between nodes;

It is possible (and recommended) to define default values for each section in the form of **[** *section* **default]** . Examples:

* [tcp default]

* [ndb_mgmd default]

* [ndbd default]



Basic example of global configuration file
~~~~
Below is listed a very simple config.ini file::

	[ndbd default]
	NoOfReplicas=2
	DataDir=/var/lib/mysql-cluster

	[ndb_mgmd]
	Hostname=mgmd1.example.com
	DataDir=/var/lib/mysql-cluster

	[ndbd]
	HostName=ndbd1.example.com

	[ndbd]
	HostName=ndbd2.example.com

	[mysqld]
	[mysqld]
	[mysqld]



Second example of global configuration file
~~~~
Below is listed a bit more complex config.ini file::

	[TCP DEFAULT]
	SendBufferMemory=2M
	ReceiveBufferMemory=2M

	[NDB_MGMD DEFAULT]
	PortNumber=1186
	Datadir=/mysqlcluster/

	[NDB_MGMD]
	NodeId=1
	Hostname=192.168.1.201
	LogDestination=FILE:filename=ndb_1_cluster.log,maxsize=10000000,maxfiles=6

	[NDBD DEFAULT]
	NoOfReplicas=2
	DataMemory=80M
	IndexMemory=15M
	LockPagesInMainMemory=1
	Datadir=/mysqlcluster/

	MaxNoOfConcurrentOperations=1000
	MaxNoOfConcurrentTransactions=1024

	StringMemory=25
	MaxNoOfTables=1024
	MaxNoOfOrderedIndexes=256
	MaxNoOfUniqueHashIndexes=64
	MaxNoOfAttributes=2560
	MaxNoOfTriggers=2560

	FragmentLogFileSize=16M
	InitFragmentLogFiles=FULL
	NoOfFragmentLogFiles=5
	RedoBuffer=8M

	TimeBetweenGlobalCheckpoints=1000
	DiskCheckpointSpeedInRestart=20M
	DiskCheckpointSpeed=4M
	TimeBetweenLocalCheckpoints=20
	CompressedLCP=1

	HeartbeatIntervalDbDb=15000
	HeartbeatIntervalDbApi=15000

	BackupMaxWriteSize=1M
	BackupDataBufferSize=12M
	BackupLogBufferSize=8M
	BackupMemory=20M
	CompressedBackup=1

	SharedGlobalMemory=10M
	DiskPageBufferMemory=32M

	[NDBD]
	NodeId=3
	Hostname=192.168.1.203

	[NDBD]
	NodeId=4
	Hostname=192.168.1.204

	[MYSQLD]
	[MYSQLD]
	[MYSQLD]
	[MYSQLD]
	[API]
	[API]
	[API]
	[API]
	[API]
	[API]


Configure and start your first MySQL Cluster
----

Copy the file `config.ini <https://github.com/renecannao/mysql-cluster-tutorial/blob/master/configfiles/config.ini.ec2>`_ into node1 **only**::
  
  node1> vi /mysqlcluster/config.ini

Copy `my.cnf <https://github.com/renecannao/mysql-cluster-tutorial/blob/master/configfiles/my.cnf.ec2>`_ into **all** nodes::

  guest> vi /etc/my.cnf


Start the management node
~~~~

Pre-requirement. Update /etc/hosts ::

  guest> sudo sed -i -e 's/^127.0.0.1.*$/127.0.0.1 localhost/' /etc/hosts

Start the management node::

  node1> ndb_mgmd --config-dir=/mysqlcluster/ --config-file=/mysqlcluster/config.ini 
  MySQL Cluster Management Server mysql-5.6.28 ndb-7.4.9
 
Don't trust the output of ndb_mgmd . Verify that the process is running, and verify the cluster log::
  
  node1> ps aux | grep ndb_mgmd
  node1> cat /mysqlcluster/ndb_1_cluster.log
  node1> tail -f /mysqlcluster/ndb_1_cluster.log

Verify the status of the cluster with ndb_mgm::
 
  node1> ndb_mgm

  ndb_mgm> SHOW
  Connected to Management Server at: 192.168.123.101:1186
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2 (not connected, accepting connect from 192.168.123.102)
  id=3 (not connected, accepting connect from 192.168.123.103)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.5.27 ndb-7.2.8)
  
  [mysqld(API)]   4 node(s)
  id=11 (not connected, accepting connect from any host)
  id=12 (not connected, accepting connect from any host)
  id=13 (not connected, accepting connect from any host)
  id=14 (not connected, accepting connect from any host)


Start the data nodes
~~~~

Start the data nodes on node2 and node3::
  
  node2> ndbd
  2013-03-23 20:32:17 [ndbd] INFO     -- Angel connected to '192.168.123.101:1186'
  2013-03-23 20:32:17 [ndbd] INFO     -- Angel allocated nodeid: 2

  node3> ndbd
  2013-03-23 20:32:25 [ndbd] INFO     -- Angel connected to '192.168.123.101:1186'
  2013-03-23 20:32:25 [ndbd] INFO     -- Angel allocated nodeid: 3


Verify that the cluster is up::

  ndb_mgm> SHOW
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2    @192.168.123.102  (mysql-5.5.27 ndb-7.2.8, Nodegroup: 0, Master)
  id=3    @192.168.123.103  (mysql-5.5.27 ndb-7.2.8, Nodegroup: 0)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.5.27 ndb-7.2.8)
  
  [mysqld(API)]   4 node(s)
  id=11 (not connected, accepting connect from any host)
  id=12 (not connected, accepting connect from any host)
  id=13 (not connected, accepting connect from any host)
  id=14 (not connected, accepting connect from any host)

Start mysqld processes
~~~~

Install mysql system tables on each guest host::

  guest> cd /usr/local/mysql
  guest> sudo ./scripts/mysql_install_db
  ...
  guest> sudo cp support-files/mysql.server /etc/init.d/mysql


Start mysqld on each guest host::
  
  guest> sudo service mysql start
  
Verify supported engines in mysql::
  
  mysql> SHOW ENGINES;
  +--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
  | Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
  +--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
  | ndbcluster         | YES     | Clustered, fault-tolerant tables                               | YES          | NO   | NO         |
  | CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
  | MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
  | ndbinfo            | YES     | MySQL Cluster system information storage engine                | NO           | NO   | NO         |
  | BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
  | MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
  | ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
  | InnoDB             | NO      | Supports transactions, row-level locking, and foreign keys     | NULL         | NULL | NULL       |
  | PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
  | FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
  | MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
  +--------------------+---------+----------------------------------------------------------------+--------------+------+------------+

Verify the output in cluster log::
  
  node1> tail -n 30 /mysqlcluster/ndb_1_cluster.log

Verify status of the cluster::
  
  ndb_mgm> SHOW
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2    @192.168.123.102  (mysql-5.5.27 ndb-7.2.8, Nodegroup: 0, Master)
  id=3    @192.168.123.103  (mysql-5.5.27 ndb-7.2.8, Nodegroup: 0)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.5.27 ndb-7.2.8)
  
  [mysqld(API)]   4 node(s)
  id=11   @192.168.123.102  (mysql-5.5.27 ndb-7.2.8)
  id=12   @192.168.123.101  (mysql-5.5.27 ndb-7.2.8)
  id=13   @192.168.123.103  (mysql-5.5.27 ndb-7.2.8)
  id=14 (not connected, accepting connect from any host)
