====
Create a table in MySQL Cluster
====

Create a new schema in MySQL Cluster
----

Check what schemas exists on each SQL nodes::
  
  mysql> SHOW SCHEMAS;
  +--------------------+
  | Database           |
  +--------------------+
  | information_schema |
  | mysql              |
  | ndbinfo            |
  | performance_schema |
  | test               |
  +--------------------+
  5 rows in set (0.00 sec)

Create a new schema in one of the SQL nodes::
  
  mysql1> CREATE DATABASE mydb;
  Query OK, 1 row affected (0.06 sec)
  
  mysql1> SHOW SCHEMAS;
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
  6 rows in set (0.00 sec)
  
Is the new schema replicated on each SQL nodes? Yes, MySQL Cluster support schema autodiscovery::
  
  mysql> SHOW SCHEMAS;


Create a new table in MySQL Cluster
~~~~

Create a new table in one of the SQL nodes::
  
  mysql1> use mydb;
  Database changed
  mysql1> CREATE TABLE table1 (id INT NOT NULL AUTO_INCREMENT PRIMARY KEY) ENGINE=ndbcluster;
  Query OK, 0 rows affected (0.22 sec)
  
  mysql1> INSERT INTO table1 VALUES (1),(2),(3),(4),(5),(6),(7);
  Query OK, 7 rows affected (0.04 sec)
  Records: 7  Duplicates: 0  Warnings: 0

**Lab** : Verify that the table and the data is available in all SQL nodes::
  
  mysql> use mydb;
  mysql> SHOW CREATE TABLE table1\G
  mysql> SELECT * FROM table1;

**Lab** : In which order are the rows returned?
