# Cassandra Study

**Cassandra Latest Version - 3.11**

- [Cassandra Basics](#cassandra-basics)


## Cassandra Basics

* **Composite key** or primary key consists of
  * a partition key - used to determine the nodes on which the rows are stored and can itseld consist of multiple columns
  * optional set of clustering keys - determines how data is sorted for storage within a partition
* **Clustering columns** 
  * guarantee uniqueness
  * support desired sort ordering
  * support range queries
* **Static column** - not part of the primary key but is shared by every row in the partition
* **Column** - name/value pair
* **Row** - container of columns referenced by a primary key
* **Table** - container of rows
* **Keyspace** - container for tables
* **Cluster** - container for keyspaces that spans one or more nodes
* Each write operation generates a timestamp for each column updated. Internally the timesstamp is used for conflict resolution - the last timestamp wins
* Each column value has a TTL which is null by default (indicating the value will not expire) until set explicitly
* **Secondary indexe** 
  * allows querying data based on columns that are not present in the primary key
  * for optimal read performance denormalized table designs or materialized views are preferred over secondary indexes
  * each node needs to maintain its own copy of secondary index based on the data it contains
* Joins and referential integrity are not allowed
* Each table is stored in a separate file on disk and hence it is important to keep the related columns in the same table
* A query that searches a single partition will give the best perdormance
* Sorting is possible only on the clustering columns
* Table naming convention - hotels_by_poi
* Hard limit - 2 billion cells per partition but there will be performance issues before reaching that limit
* To split a large partition, add an additional column to the partition key
* To create a separate partition for each day's data, include the date attribute in the partition key. If this results in very small partitions add only the month

## Architecture

* Cassandra tries to distribute copies of data across different data centers and racks to maximize availability and partition tolerance while preferring to route queries to nodes in the local data center to maximize performance
* Cassandra uses Gossip protocol for failure detection. The nodes exchange information about themselves and about the other nodes that they have gossiped about, so all nodes quickly learn about all other nodes in the cluster
  * Once per second, the gossipier will choose a random nodein the cluster and initialize a gozzip session with it
  * Each round of gossip requires 3 messages
  * The gossip initiator sends its chosen friend a GossipDigestSynMessage
  * When the friend receives this message , it returns a GossipDigestAckMessage
  * When the initiator receives the ack message from the friend, it sends the friend a GossipDigestAck2Message to complete the round of gossip
  * When the gossipier determines that another endpoint is dead, it "convicts" that endpoint by marking it as dead in its local list and logging the fact
  * A gossip message has a version associated with it, so that during a gossip exchange, older information is overwritten with the most current state for a particular node
* Each node keeps track of the state of other nodes in the cluster by means of an accrual failure detector (or phi failure detector). This detector evaluates the health of other nodes based on a sliding window of gossip
* Making every node a seed node is not recommended because of increased maintenance and reduced gossip performance. Gossip optimization is not critical, but it is recommended to use a small seed list (approximately three nodes per datacenter)


