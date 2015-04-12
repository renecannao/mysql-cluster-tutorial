====
Load data into MySQL Cluster
====

To populate MySQL Cluster with data we are going to use the *world* database available `here <http://dev.mysql.com/doc/index-other.html>`_ or in the USB drive::
  
  mysql1> CREATE SCHEMA world;
  node1> zcat /vagrant/world.sql.gz | mysql world
  mysql1> use world
  mysql1> SHOW TABLES;
  mysql1> SELECT * FROM City LIMIT 10;

**Lab** :

* is the schema created on all nodes?

* are the tables created on all nodes?
