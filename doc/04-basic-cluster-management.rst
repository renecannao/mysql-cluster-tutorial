====
Basic MySQL Cluster management
====

Basic MySQL Cluster start
----

We previously show how to start the Cluster. In short, the process is:

* Start the management node ( ndb_mgmd )

* Start the Data Nodes ( ndbd )

* Start the SQL nodes ( mysqld )



Restart a data node process
----

From the data node host::
  
  node2> killall ndbd

or, from the management node::
  
  ndb_mgm> 2 STOP


Start a data node process from the data node host::
  
  node2> ndbd

or, start the data node process with *--nostart* ( or -n ) , and then start it from the management node::
  
  node2> ndbd --nostart
  
  ndb_mgm> 2 START

Restart a data node process directly from the management node::
  
  ndb_mgm> 2 RESTART

**Lab** : restart both data nodes. Issues?


Shutdown the Cluster
----

There are 3 possible ways to shutdown MySQL Cluster:

* kill all the processes ;

* stop all the processes from the management node ;

* issue the command SHUTDOWN in the management node::
    
    ndb_mgm> SHUTDOWN
    Connected to Management Server at: 192.168.123.101:1186
    Node 2: Cluster shutdown initiated
    Node 3: Cluster shutdown initiated
    Node 2: Node shutdown completed.
    Node 3: Node shutdown completed.
    3 NDB Cluster node(s) have shutdown.
    Disconnecting to allow management server to shutdown.
    ndb_mgm>


Options for MySQL Cluster programs
----

Common options
~~~~

* --ndb-connectstring , --connect-string , -c

  Specifies the address(es) of the management node(s), and optionally their port. Ex::
  
    ndb_mgm --ndb-connectstring=server-1:1186

* --ndb-nodeid

  Set the NodeId for this specific process. Ex::
    
    ndbd --ndb-nodeid=2


ndbd options
~~~~

The follow is a list of the most important options for ndbd ( and ndbmtd ) :

* --nostart , -n
  
  ndbd process doesn't start fully, but waits the START commands from the management node

* --initial

  ndbd performs an initial starts. With this option, the filesystem of the ndbd process is cleaned and all the data delete.

* --initial-start

  ndbd performs an partial start, without waiting all the nodes to become available . Requires --nowait-nodes

* --nowait-nodes

  specifies a list of nodes to not wait before performing a partial start


Partial start
~~~~

A patial start of MySQL Cluster occurs when a the Cluster starts with a limited number of data nodes.

MySQL Cluster tries to avoid to start if not all the nodes are available.

If not all data nodes are available within *StartPartialTimeout* milliseconds ( 30000 by default ), MySQL Cluster perform a partial start.

*--initial-start* changes this behaviour, and MySQL Cluster perform a partial start without waiting all data nodes. 
