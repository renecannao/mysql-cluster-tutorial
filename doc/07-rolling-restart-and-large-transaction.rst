====
Rolling restart to allow large transactions
====


How to perform a rolling restart
====

In the previous section we could not run a large transaction.

To successfully run a large transaction, we need to reconfigure `MaxNoOfConcurrentOperations <http://dev.mysql.com/doc/refman/5.5/en/mysql-cluster-ndbd-definition.html#ndbparam-ndbd-maxnoofconcurrentoperations>`_ .

Steps:

* Reconfigure config.ini

* restart the Management Node with the new config.ini

* restart each Data Node , one at the time


Reconfigure config.ini
~~~~

Settings need t be configured in the Management Node.

In node1:/mysqlcluster/config.ini add the follow in the [NDBD DEFAULT] section::
  
  MaxNoOfConcurrentOperations=65536


Restart the Management Node
~~~~

Kill the management node ( ndb_mgmd ) and restart it::
  
  node1> killall ndb_mgmd
  node1> ndb_mgmd --config-dir=/mysqlcluster/ --config-file=/mysqlcluster/config.ini --initial

Notes:

* *--initial* forces the management node to re-read the config file and discard any configuration cached
* *--reload* is similar to *--initial* , but the configuration cache is recreated only if differs from config file, and it doesn't delete previous configuration caches


Check the status from ndb_mgm::
  
  ndb_mgm> SHOW
  Connected to Management Server at: 192.168.123.101:1186
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2    @192.168.123.102  (mysql-5.5.27 ndb-7.2.8, Nodegroup: 0, Master)
  id=3    @192.168.123.103  (mysql-5.5.27 ndb-7.2.8, Nodegroup: 0)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.5.27 ndb-7.2.8)
  
  [mysqld(API)]   4 node(s)
  id=11   @192.168.123.103  (mysql-5.5.27 ndb-7.2.8)
  id=12   @192.168.123.102  (mysql-5.5.27 ndb-7.2.8)
  id=13   @192.168.123.101  (mysql-5.5.27 ndb-7.2.8)
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
  id=3    @192.168.123.103  (mysql-5.5.27 ndb-7.2.8, Nodegroup: 0, Master)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.5.27 ndb-7.2.8)
  
  [mysqld(API)]   4 node(s)
  id=11   @192.168.123.103  (mysql-5.5.27 ndb-7.2.8)
  id=12   @192.168.123.102  (mysql-5.5.27 ndb-7.2.8)
  id=13   @192.168.123.101  (mysql-5.5.27 ndb-7.2.8)
  id=14 (not connected, accepting connect from any host)

Start the data node on node2::
  
  node2> ndbd

Wait and verify that Data Node was started successfully::
  
  ndb_mgm> Node 2: Started (version 7.2.8)

  ndb_mgm> SHOW
  Cluster Configuration
  ---------------------
  [ndbd(NDB)]     2 node(s)
  id=2    @192.168.123.102  (mysql-5.5.27 ndb-7.2.8, Nodegroup: 0)
  id=3    @192.168.123.103  (mysql-5.5.27 ndb-7.2.8, Nodegroup: 0, Master)
  
  [ndb_mgmd(MGM)] 1 node(s)
  id=1    @192.168.123.101  (mysql-5.5.27 ndb-7.2.8)
  
  [mysqld(API)]   4 node(s)
  id=11   @192.168.123.103  (mysql-5.5.27 ndb-7.2.8)
  id=12   @192.168.123.102  (mysql-5.5.27 ndb-7.2.8)
  id=13   @192.168.123.101  (mysql-5.5.27 ndb-7.2.8)
  id=14 (not connected, accepting connect from any host)


Repeat the same for node3.



Run a large transaction
=====

After we changed MaxNoOfConcurrentOperations we should be able to perform a larger transaction::
  
  mysql> START TRANSACTION;
  Query OK, 0 rows affected (0.00 sec)
  
  mysql> SELECT COUNT(*) FROM table1;
  +----------+
  | COUNT(*) |
  +----------+
  |   237568 |
  +----------+
  1 row in set (0.00 sec)
  
  mysql> DELETE FROM table1 LIMIT 50000;
  Query OK, 50000 rows affected (1.20 sec)
  
  mysql> SELECT COUNT(*) FROM table1;
  +----------+
  | COUNT(*) |
  +----------+
  |   187568 |
  +----------+
  1 row in set (0.01 sec)


Check memory usage from ndb_mgm::
  
  ndb_mgm> ALL REPORT MEMORY
  Node 2: Data usage is 36%(928 32K pages of total 2560)
  Node 2: Index usage is 20%(479 8K pages of total 2336)
  Node 3: Data usage is 36%(928 32K pages of total 2560)
  Node 3: Index usage is 20%(479 8K pages of total 2336)

Why memory usage has increased after a DELETE statement?

Commit::
  
  mysql> COMMIT;
  Query OK, 0 rows affected (0.23 sec)
  
  mysql> SELECT COUNT(*) FROM table1;
  +----------+
  | COUNT(*) |
  +----------+
  |   187568 |
  +----------+
  1 row in set (0.00 sec)

Check memory usage from ndb_mgm::
  
  ndb_mgm> ALL REPORT MEMORY
  Node 2: Data usage is 29%(763 32K pages of total 2560)
  Node 2: Index usage is 18%(431 8K pages of total 2336)
  Node 3: Data usage is 29%(763 32K pages of total 2560)
  Node 3: Index usage is 18%(424 8K pages of total 2336)

Memory usage went back to normal, and didn't decrease from previous value.

Delete more rows::
  
  mysql> DELETE FROM table1 LIMIT 50000;
  Query OK, 50000 rows affected (1.41 sec)
  
  mysql> DELETE FROM table1 LIMIT 50000;
  Query OK, 50000 rows affected (1.41 sec)
  
  mysql> SELECT COUNT(*) FROM table1;
  +----------+
  | COUNT(*) |
  +----------+
  |    87568 |
  +----------+
  1 row in set (0.00 sec)

Check memory usage from ndb_mgm::
  
  ndb_mgm> ALL REPORT MEMORY
  Node 2: Data usage is 29%(763 32K pages of total 2560)
  Node 2: Index usage is 18%(431 8K pages of total 2336)
  Node 3: Data usage is 29%(763 32K pages of total 2560)
  Node 3: Index usage is 18%(424 8K pages of total 2336)

Memory usage doesn't change deleting rows!

OPTIMIZE TABLE needs to be executed to free space::
  
  mysql> OPTIMIZE TABLE table1;
  +-------------+----------+----------+----------+
  | Table       | Op       | Msg_type | Msg_text |
  +-------------+----------+----------+----------+
  | mydb.table1 | optimize | status   | OK       |
  +-------------+----------+----------+----------+
  1 row in set (11.54 sec)
  
  
  ndb_mgm> ALL REPORT MEMORY
  Node 2: Data usage is 21%(561 32K pages of total 2560)
  Node 2: Index usage is 11%(274 8K pages of total 2336)
  Node 3: Data usage is 21%(561 32K pages of total 2560)
  Node 3: Index usage is 11%(273 8K pages of total 2336)




Notes on OPTIMIZE TABLE:

* OPTIMIZE TABLE processes the table in batches, and each batch is executed after *ndb_optimization_delay* milliseconds ;

* OPTIMIZE TABLE can be killed : this interrupts the optimizations ;

* OPTIMIZE TABLE blocks queries against the same table if executed on the same SQL node, but not if executed on other nodes : it is an online process.


Testing "online" OPTIMIZE TABLE::
  
  node3> for i in `seq 1 30` ; do date ; echo "INSERT INTO table1 SELECT NULL,MD5(RAND()) FROM table1 LIMIT 1024;" | mysql mydb ; sleep 1 ; done
  
  mysql2> OPTIMIZE TABLE table1;


Notes:

* Another method to free unused memory is to perform a new rolling restart. During startup the Data Node allocates to a table only the number of pages required to store the data.
