====
Testing High Availability
====

We already discussed how MySQL Cluster provides high availability, so now we are going to test few failure scenarios to verify how the Cluster reacts to them.


Verify that the data in the MySQL Cluster is still available after a complete Cluster restart::
  
  mysql> use mydb
  mysql> SHOW TABLES;
  mysql> SHOW CREATE TABLE table1\G
  mysql> SELECT * FROM table1;
  mysql> use world
  mysql> SHOW TABLES;
  mysql> SELECT * FROM City LIMIT 10;

Is the data available from all SQL Nodes?

Stop a data node::
  
  ndb_mgm> 3 STOP
  ndb_mgm> SHOW
  ndb_mgm> ALL STATUS

It the data still available from all SQL Nodes?

Create a new table::
  
  mysql> use mydb
  mysql> CREATE TABLE table2 (id INT NOT NULL PRIMARY KEY) ENGINE=ndbcluster;
  mysql> INSERT INTO table2 VALUES (1), (2), (3);

Is the data still available from all SQL Nodes?

Shutdown the management node::
  
  ndb_mgm> 1 STOP
  Node 1 has shutdown.
  Disconnecting to allow Management Server to shutdown

Is the data still available from all SQL Nodes?

Stop mysqld in one node::
  
  node1> service mysql stop

Is the data still available from all SQL Nodes?

Stop mysqld in another node::
  
  node2> service mysql stop

Is the data still available from node3?

Bring online the whole Cluster::
  
  node1> ndb_mgmd --config-dir=/mysqlcluster/ --config-file=/mysqlcluster/config.ini
  node1> service mysql start

  node2> service mysql start
  
  node3> ndbd

Verify the status of the Cluster::
  
  ndb_mgm> SHOW

Stop all network communication on node3 to simulate a HW failure::
  
  node3> iptables -I INPUT -p tcp --dport 22 -j ACCEPT ; iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT
  node3> iptables -P INPUT DROP ; iptables -P OUTPUT DROP

Verify the status of the Cluster::
  
  ndb_mgm> SHOW
  node1> tail -f /mysqlcluster/ndb_1_cluster.log

Try to access MySQL Cluster from SQL Node on node3::
  
  mysql3> use mydb
  Database changed
  mysql3> SHOW TABLES;
  +----------------+
  | Tables_in_mydb |
  +----------------+
  | table1         |
  | table2         |
  +----------------+
  2 rows in set, 1 warning (0.00 sec)
  
  mysql3> SHOW WARNINGS;
  +---------+------+---------------------------------------------------------------------------------+
  | Level   | Code | Message                                                                         |
  +---------+------+---------------------------------------------------------------------------------+
  | Warning | 1296 | Got error 4009 'Cluster Failure' from NDB. Could not acquire global schema lock |
  +---------+------+---------------------------------------------------------------------------------+
  1 row in set (0.00 sec)
  
  mysql3> SELECT * FROM table2;
  ERROR 1296 (HY000): Got error 157 'Unknown error code' from NDBCLUSTER
  mysql3> CREATE TABLE table3 (id INT) ENGINE=ndbcluster;
  ERROR 157 (HY000): Could not connect to storage engine

Is ndbd still running on node3?::
  
  node3> ps aux | grep ndbd

Why not? Check error log::
  
  node3> cat /mysqlcluster/ndb_3_error.log

Try to access MySQL Cluster from SQL Node on node2::
  
  mysql2> use mydb
  Database changed
  mysql2> SHOW TABLES;
  +----------------+
  | Tables_in_mydb |
  +----------------+
  | table1         |
  | table2         |
  +----------------+
  2 rows in set (0.00 sec)
  
  mysql2> SELECT * FROM table2;

Stop all network communication on node1 to simulate a HW failure::
  
  node1> iptables -I INPUT -p tcp --dport 22 -j ACCEPT ; iptables -I OUTPUT -p tcp --sport 22 -j ACCEPT
  node1> iptables -I INPUT -p tcp --sport 22 -j ACCEPT ; iptables -I OUTPUT -p tcp --dport 22 -j ACCEPT
  node1> iptables -P INPUT DROP ; iptables -P OUTPUT DROP

Is the MySQL Cluster still alive?

Restore the Cluster::
  
  node1> iptables -P INPUT ACCEPT ; iptables -P OUTPUT ACCEPT

  node3> iptables -P INPUT ACCEPT ; iptables -P OUTPUT ACCEPT
  node3> ndbd

  ndb_mgm> SHOW

Simulate a split brain scenario::
  
  node1> iptables -P INPUT DROP ; iptables -P OUTPUT DROP

  node3> iptables -P INPUT DROP ; iptables -P OUTPUT DROP

**Question:** Is MySQL Cluster still alive?



Restore the Cluster::

  node1> iptables -P INPUT ACCEPT ; iptables -P OUTPUT ACCEPT

  node3> iptables -P INPUT ACCEPT ; iptables -P OUTPUT ACCEPT
  
  node2> ndbd
  
  node3> ndbd

  ndb_mgm> SHOW  
