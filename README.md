# Cassandra Study

**Cassandra Latest Version - 3.11**

- [Cassandra Basics](#cassandra-basics)


## Cassandra Basics

* **Composite key** or primary key consists of
  * Partition key - determines the nodes on which the rows are stored. It can be made of multiple columns
  * Clustering columns (optional) - determine how data is sorted for storage within a partition
* **Clustering columns** 
  * guarantee uniqueness
  * support desired sort ordering
  * support range queries
* Each row has a unique value for **Composite key** (Partition key + Clustering columns)
* Each partition has a unique value for **Partition key**
* **Static column** - NOT part of the primary key but is shared by every row in the partition
* **Column** - name/value pair
* **Row** - container of columns referenced by a primary key
* **Table** - container of rows
* **Keyspace** - container for tables
* **Cluster** - container for keyspaces that spans one or more nodes
* Each write operation generates a timestamp for each column updated. Internally the timesstamp is used for conflict resolution - the last timestamp wins
* Each column value has a TTL which is null by default (indicating the value will not expire) until set explicitly
* **Secondary index** 
  * allows querying data based on columns that are not present in the primary key
  * for optimal read performance denormalized table designs or materialized views are preferred over secondary indexes
  * each node needs to maintain its own copy of secondary index based on the data it contains
* Joins and referential integrity are not allowed
* Each table is stored in a separate file on disk and hence it is important to keep the related columns in the same table
* A query that searches a single partition will give the best performance
* Sorting is possible only on the clustering columns
* Table naming convention - hotels_by_poi
* Hard limit - 2 billion cells per partition but there will be performance issues before reaching that limit
* To split a large partition, add an additional column to the partition key. E.g. if the partition key is "year", all data of a given year will end up in a single partitions. If "month" column is also added in the partition key, each partition will contain only a month's worth of data
* Cassandra treats each new row as an upsert: if the new row has the same primary key as that of an existing row, Cassandra processes it as an update to the existing row
* Cassandra is optimized for high write throughput, and almost all writes are equally efficient. If you can perform extra writes to improve the efficiency of your read queries, it's almost always a good tradeoff. Reads tend to be more expensive and are much more difficult to tune

## Architecture

### Storage

* When a write occurs, Cassandra first appends the write to a commitlog segment on the disk for durability and then writes the data to a memory structure called memtable
* Commitlogs have size-bound segments. Once the size limit is reached a new segment is created
* Commitlog segments are truncated when Cassandra has flushed data - older than a certain point - from the memtables to the SSTables on the disk
* If a node stops working, commit log is replayed to restore the last state of the memtables
* To reduce the commit log replay time, the recommended best practice is to flush the memtable before restarting the nodes
* `commitlog_sync` setting determines when the data on the commitlog will be `fsync`ed to the disk. Possible values:
  * `batch` (default) - Cassandra won’t ack writes to the clients until the commit log has been `fsync`ed to the disk. It will wait `commitlog_sync_batch_window_in_ms` milliseconds between `fsync`s. Default value is 2 milliseconds
  * `periodic` - Writes are immediately ack’ed to the clients, and the Commitlog is simply synced every `commitlog_sync_period_in_ms` milliseconds. Default value is 10 seconds
* Commitlogs should go in a separate device in `batch` mode
* Memtables are automatically flushed to SSTables in two scenarios
  * The memory usage exceeds a configurable limit. The memtable is written out to a new SSTable
  * The commitlog size on disk exceeds a certain threshold. All dirty column families present in the oldest segment will be flushed from memtable to SSTable. The segment will then be removed
* Memtables and SSTables are maintained per table
* The commit log is shared among tables
* SSTables are immutable and cannot be written to after the memtable is flushed
* Each SSTable is comprised of:
  * Data (data.db) - The actual data i.e. the content of rows
  * Primary Index (index.db) - An index from partition keys to positions in the Data.db file. For wide partitions, this may also include an index to rows within a partition
  * Bloom Filter (Filter.db) - A Bloom Filter of the partition keys in the SSTable
  * Compression Information (Compressioninfo.db) - Metadata about the offsets and lengths of compression chunks in the Data.db file
  * Statistics (Statistics.db) - Stores metadata about the SSTable, including information about timestamps, tombstones, clustering keys, compaction, repair, compression, TTLs, and more
  * Digest (Digest.crc32, Digest.adler32, Digest.sha1) - A CRC-32 digest of the Data.db file
  * CRC (CRC.db)
  * SSTable Index Summary (SUMMARY.db) - A sampling of (by default) every 128th entry in the Index.db file
  * SSTable Table of Contents (TOC.txt) - A plain text list of the component files for the SSTable
  * Secondary Index (SI_.*.db)
* Within the Data.db file, rows are organized by partition. These partitions are sorted in token order (i.e. by a hash of the partition key when the default partitioner, Murmur3Partition, is used). Within a partition, rows are stored in the order of their clustering keys

To keep the database healthy, Cassandra periodically merges SSTables and discards old data. This process is called compaction.
Compaction works on a collection of SSTables. From these SSTables, compaction collects all versions of each unique row and assembles one complete row, using the most up-to-date version (by timestamp) of each of the row's columns. The merge process is performant, because rows are sorted by partition key within each SSTable, and the merge process does not use random I/O. The new versions of each row is written to a new SSTable. The old versions, along with any rows that are ready for deletion, are left in the old SSTables, and are deleted as soon as pending reads are completed.

### Tomb Stone

* On deletion, Cassandra marks the data with a tombstone
* Garbage Collection Grace Seconds (default values - 10 days) must expire before the compaction process can garbage collect the tobstone and the associated data
* The Garbage Collection Grace Seconds setting allows the nodes that failed before receiving the delete operation to recover. Otherwise, the deleted data may resurrect in the node
* Any node that remains down for Garbage Collection Grace Seconds is considered as failed and hence replaced
* Each table can have its own value for Garbage Collection Grace Seconds

### Compaction

* Compaction is the process of merging SSTables for a given table and discard old and obsolete data
* Once the merge is complete, old SSTables are dropped
* **Compaction Strategies** -
  * **SizeTieredCompactionStrategy (STCS)** - Recommended for write-intensive workloads (default)
  * **LeveledCompactionStrategy (LCS)** - Recommended for read-intensive workloads
  * **TimeWindowCompactionStrategy (TWCS)** - Recommended for time series and expiring TTL workloads

### Snitch

* Snitch provides information about the network topology:
  * To route requests efficiently
  * To spread replicas around across “datacenters” and “racks” for better fault tolerance
* The replication strategy places the replicas based on the information provided by the snitch
* **Dynamic Snitch** - 
  * Enabled by default
  * All snitches use a dynamic snitch layer
  * Monitors the latency and other factors like whether compactions is currently going on in a node to route away the read requests from poorly performig node
  * All performance scores determined by the dynamic snitch are reset periodically to give a fresh opportunity to the poorly performing nodes

### Replication

* Replication factor determines the number of copies of data to be stored in the cluster for fault tolerance and availability
* Replication factor is a setting at the keyspace level
* All replicas are equally important; there is no primary or master replica. As a general rule, the replication factor should not exceed the number of nodes in the cluster
* **Replication Strategy**
  * **SimpleStrategy** - Places replicas at consecutive nodes around the ring, starting with the node indicated by the partitioner
  * **NetworkTopologyStrategy** - Allows specifying a different replication factor for each data center. Within a data center, it allocates replicas to different racks to maximize the availability

### Partitioner

* The Murmur3Partitioner is the default partitioning strategy

### Consistency Level

* It determines how many nodes must respond to a read or write operation in order to consider it as successful
* The consistency level is specified per query by the client
* The consistency level defaults to **ONE** for all write and read operations
* R + W > N = strong consistency , where R = read replica count, W = write replica count, and N = replication factor. All client reads will see the most recent writes
* **Write Consistency Levels** -
  * **ALL** -  A write must be written to the commit log and memtable on all replica nodes in the cluster for that partition
  * **EACH_QUORUM** - A write must be written to the commit log and memtable on a quorum of replica nodes in each datacenter
  * **QUORUM** -  A write must be written to the commit log and memtable on a quorum of replica nodes across all datacenters.
  * **LOCAL_QUORUM** - A write must be written to the commit log and memtable on a quorum of replica nodes in the same datacenter as the coordinator
  * **ONE** - A write must be written to the commit log and memtable of at least one replica node
  * **TWO** - A write must be written to the commit log and memtable of at least two replica nodes
  * **THREE** - A write must be written to the commit log and memtable of at least three replica nodes
  * **LOCAL_ONE** - A write must be sent to, and successfully acknowledged by, at least one replica node in the local datacenter
  * **ANY** - A write must be written to at least one node. If all replica nodes for the given partition key are down, the write can still succeed after a hinted handoff has been written. If all replica nodes are down at write time, an ANY write is not readable until the replica nodes for that partition have recovered
* **Read Consistency Levels**
  * **ALL** - Returns the record after all replicas have responded. The read operation will fail if a replica does not respond
  * **QUORUM** - Returns the record after a quorum of replicas from all datacenters has responded
  * **LOCAL_QUORUM** - Returns the record after a quorum of replicas in the current datacenter as the coordinator has reported
  * **ONE** - Returns a response from the closest replica, as determined by the snitch. By default, a read repair runs in the background to make the other replicas consistent
  * **TWO** - Returns the most recent data from two of the closest replicas
  * **THREE** - Returns the most recent data from three of the closest replicas
  * **LOCAL_ONE** - Returns a response from the closest replica in the local datacenter
  * **SERIAL** - Allows reading the current (and possibly uncommitted) state of data without proposing a new addition or update. If a SERIAL read finds an uncommitted transaction in progress, it will commit the transaction as part of the read. Similar to QUORUM
  * **LOCAL_SERIAL** - Same as SERIAL, but confined to the datacenter. Similar to LOCAL_QUORUM
* A client may connect to any node in the cluster to initiate a read or write query. This node is called a coordinator node
* If a replica is not available at the time of write, the coordinator node can store a hint such that when the replica node becomes available, and the coordinator node comes to know about it through gossip, it will send the write to the replica. This is called **hinted-handoff**
* Hints do not count as writes for the purpose of consistency level except the consistency level of ANY. This behaviour of ANY is also known as **Sloppy Quorum**
* The coordinator node contacted by the client application forwards the write request to one replica in each of the other datacenters, with a special tag to forward the write to the other local replicas
* If the coordinator cannot write to enough replicas to meet the requested consistency level, it throws an Unavailable Exception and does not perform any writes
* **SERIAL** & **LOCAL_SERIAL** (equivalent to QUORUM & LOCAL_QUORUM respectively) consistency modes are for use with lightweight transaction
* If there are enough replicas available but the required writes don't finish within the timeout window, the coordinator throws a Timeout Exception
* **Limitations of hinted-handoff** - If a node remains down for quite some time, hints can build up and then overwhelming the node when it is back online. Hence hints are stored for a certain time window only and it can be disabled

### Read Path



### Write Path

* The client connects to any node which becomes the co-ordinator node for the write operation
* The co-ordinator node contacts all replicas based on replication factor and consistency level
* 

### Failure Detection

* Cassandra uses Gossip protocol for failure detection. The nodes exchange information about themselves and about the other nodes that they have gossiped about, so all nodes quickly learn about all other nodes in the cluster
  * Once per second, the gossipier will choose a random node in the cluster and initialize a gozzip session with it
  * Each round of gossip requires 3 messages
  * The gossip initiator sends its chosen friend a GossipDigestSynMessage
  * When the friend receives this message, it returns a GossipDigestAckMessage
  * When the initiator receives the ack message from the friend, it sends the friend a GossipDigestAck2Message to complete the round of gossip
  * When the gossipier determines that another endpoint is dead, it "convicts" that endpoint by marking it as dead in its local list and logging the fact
  * A gossip message has a version associated with it, so that during a gossip exchange, older information is overwritten with the most current state for a particular node
* Each node keeps track of the state of other nodes in the cluster by means of an accrual failure detector (or phi failure detector). This detector evaluates the health of other nodes based on a sliding window of gossip
* Making every node a seed node is not recommended because of increased maintenance and reduced gossip performance. Gossip optimization is not critical, but it is recommended to use a small seed list (approximately three nodes per datacenter)

## Operations

Read path
Check the memtable
Check row cache, if enabled
Checks Bloom filter
Checks partition key cache, if enabled
Goes directly to the compression offset map if a partition key is found in the partition key cache, or checks the partition summary if not
If the partition summary is checked, then the partition index is accessed
Locates the data on disk using the compression offset map
Fetches the data from the SSTable on disk

If the memtable has the desired partition data, then the data is read and then merged with the data from the SSTables
In Cassandra 2.2 and later, row cache is stored in fully off-heap memory using a new implementation that relieves garbage collection pressure in the JVM
The operating system page cache is best at improving performance, although the row cache can provide some improvement for very read-intensive operations, where read operations are 95% of the load
The row cache is not write-through. If a write comes in for the row, the cache for that row is invalidated and is not cached again until the row is read
The row cache uses LRU (least-recently-used) eviction to reclaim memory when the cache has filled up
The Bloom filter is stored in off-heap memory
Each SSTable has a Bloom filter associated with it. A Bloom filter can establish that a SSTable does not contain certain partition data
If the Bloom filter does not rule out an SSTable, Cassandra checks the partition key cache
The Bloom filter grows to approximately 1-2 GB per billion partitions
The partition key cache, if enabled, stores a cache of the partition index in off-heap memory
If a partition key is found in the key cache can go directly to the compression offset map to find the compressed block on disk that has the data
If a partition key is not found in the key cache, then the partition summary is searched
The partition summary is an off-heap in-memory structure that stores a sampling of the partition index
A partition index contains all partition keys, whereas a partition summary samples every X keys, and maps the location of every Xth key's location in the index file
After finding the range of possible partition key values, the partition index is searched
The partition index resides on disk and stores an index of all partition keys mapped to their offset
Using the information found, the partition index now goes to the compression offset map to find the compressed block on disk that has the data. If the partition index must be searched, two seeks to disk will be required to find the desired data
The compression offset map stores pointers to the exact location on disk that the desired partition data will be found. It is stored in off-heap memory and is accessed by either the partition key cache or the partition index. The desired compressed partition data is fetched from the correct SSTable(s) once the compression offset map identifies the disk location.
Compression is enabled by default even though going through the compression offset map consumes CPU resources. Having compression enabled makes the page cache more effective, and typically, almost always pays off
Cassandra extends the concept of eventual consistency by offering tunable consistency
Write operations will use hinted handoffs to ensure the writes are completed when replicas are down or otherwise not responsive to the write request
Typically, a client specifies a consistency level that is less than the replication factor specified by the keyspace
Strong consistency can be guaranteed when the following condition is true:R + W > N where
- R is the consistency level of read operations
- W is the consistency level of write operations
- N is the number of replicas
In Cassandra, a write operation is atomic at the partition level
Cassandra uses client-side timestamps to determine the most recent update to a column
The latest timestamp always wins when requesting data, so if multiple client sessions update the same columns in a row concurrently, the most recent update is the one seen by readers.
Cassandra write and delete operations are performed with full row-level isolation. This means that a write to a row within a single partition on a single node is only visible to the client performing the operation – the operation is restricted to this scope until it is complete. All updates in a batch operation belonging to a given partition key have the same restriction
Consistent hashing allows distribution of data across a cluster to minimize reorganization when nodes are added or removed


3 built-in repair mechanisms:
- hinted-handoff
- read-repair
- anti-entropy node repair

For read requests, the replicas as required by the consistency level must respond. Other replicas will be checked for consistency in the background. Read repair will be done in the background if required

If the table has been configured with the speculative_retry property, the coordinator node for the read request will retry the request with another replica node if the original replica node exceeds a configurable timeout value to complete the read request

There are three types of read requests that a coordinator can send to a replica:

A direct read request

A digest request

A background read repair request


