# What’s happening inside Cassandra?

## On write
* Request goes to coordinator node. Cassandra may have multiple coordinators.
  * Hash value of the [(Composite) Partition Key](http://www.datastax.com/documentation/cql/3.0/cql/cql_reference/create_table_r.html?scroll=reference_ds_v3f_vfk_xj__defining-a-primary-key-column) is calculated based on the chosen [Partitioner](http://www.datastax.com/documentation/cassandra/2.1/cassandra/architecture/architecturePartitionerAbout_c.html).
  * Nodes to which the data needs to be written are determined based on the [replication](http://www.datastax.com/documentation/cassandra/2.1/cassandra/architecture/architectureDataDistributeReplication_c.html), [snitch](http://www.datastax.com/documentation/cassandra/2.1/cassandra/architecture/architectureSnitchesAbout_c.html) and the range of data the node is responsible for (pre-determined based on num_tokens or vnodes).
  * Coordinator waits for one or more nodes to respond (based on the consistency level) before responding back to the client.
  * If one or more of the data nodes are down, determined by [gossip](http://www.datastax.com/documentation/cassandra/2.1/cassandra/architecture/architectureGossipAbout_c.html), or if the node fails to respond, the coordinator may [write a hint](http://www.datastax.com/dev/blog/modern-hinted-handoff) and will retry when the node comes back. Write is deemed successful if the required consistency level is met.
  * For [BATCH](http://www.datastax.com/documentation/cql/3.1/cql/cql_reference/batch_r.html) requests, request is stored as a blob in the batchlog before being processed. This is avoided when using UNLOGGED.

* Inside the data node:
  * Data is written to commit log if [DURABLE_WRITES](http://www.datastax.com/documentation/cql/3.1/cql/cql_reference/create_keyspace_r.html?scroll=reference_ds_ask_vyj_xj__durable-writes) is true.
  * A copy is written to the Memtable
  * Background activities
     * Commit log sync to disk based on [commitlog_sync](http://www.datastax.com/documentation/cassandra/2.1/cassandra/configuration/configCassandra_yaml_r.html?scroll=reference_ds_qfg_n1r_1k__commitlog_sync)
     * Memtable flushed to disk as SSTable based on [commitlog_total_space_in_mb](http://www.datastax.com/documentation/cassandra/2.1/cassandra/configuration/configCassandra_yaml_r.html?scroll=reference_ds_qfg_n1r_1k__commitlog_total_space_in_mb)
     * Compaction brings together latest values of columns spread across SSTables to one SSTable. This is required because Cassandra does not read before write before INSERT or UPDATE.
  * Write may fail if Memtable flush queue is having memtable_flush_queue_size Memtables.
  * Tombstones mark data as deleted
     * Marker on a row indicating if a column is deleted
     * Actual delete happens during compaction
     * If you have a delete heavy workload, they may cause performance degradation if using range reads and compaction is infrequent

## On read
* Request goes to coordinator node.
  * Coordinator determines the required nodes based on the snitch, hash of the Partition Key and required consistency level.
  * Conflict resolution based on latest timestamp of the given value.
* Inside the data node:
  * Consult bloom filters for the probability of finding data in a table
  * Check if key offset is cached, if not, determine key from SSTable index
  * Fetch latest version of the data (updates would have caused multiple versions spread across SSTables), uncompress if required and return results; ignore values which have tombstones set.

## Background activities
* Gossip
  * Determines which nodes are up or down
  * Determines health and load of the nodes
* Transfer request data between coordinator and data nodes
* Transfer data between nodes when new nodes are added or nodes are being decommissioned
* Hinted handoff
* nodetool repair
  * Required if node is down for more than [max_hint_window_in_ms](http://www.datastax.com/documentation/cassandra/2.1/cassandra/configuration/configCassandra_yaml_r.html?scroll=reference_ds_qfg_n1r_1k__max_hint_window_in_ms)

## Physical storage
* Commit log witten to [commitlog_directory](http://www.datastax.com/documentation/cassandra/2.1/cassandra/configuration/configCassandra_yaml_r.html?scroll=reference_ds_qfg_n1r_1k__commitlog_directory)
* SSTables written to [data_file_directories](http://www.datastax.com/documentation/cassandra/2.1/cassandra/configuration/configCassandra_yaml_r.html?scroll=reference_ds_qfg_n1r_1k__data_file_directories)

## Logical storage
* ColumnFamily data model similar to Google’s Bigtable
* Think of it as a MySQL/SQLite index or a map of a map: an outer map keyed by a partition key, and an inner map keyed by a column key. Both maps are sorted.
  * SortedMap&lt;PrimaryKey, SortedMap&lt;ColumnKey, ColumnValue&gt;&gt;
* Supports upto 2 Billion columns
  * Millions recommended
  * Split rows if necessary, via buckets or other natural keys like “year,month,date”
* SSTables are immutable
* Keyspace
  * contains Tables/ColumnFamily and defines the schema
  * Determines replication
* Tables store column names and values
  * Partition key determines unique physical row
  * Primary key determines unique logical CQL row
     * Primary key has partition key and optional clustering columns
  * Clustering columns causes the physical row to expand, encompasses multiple logical CQL rows
  * Static column stores a unique value per physical row
