#Performance
## Performance considerations
* [Right compaction strategy?](http://docs.datastax.com/en/cql/3.1/cql/cql_reference/tabProp.html?scroll=tabProp__moreCompaction)
  * SizeTieredCompactionStrategy
    * Compaction happens when SSTable reaches a certain size
    * Not good for frequently updated cell in a (wide) row - multiple seeks to get entire row
    * Good for write-once data
    * Requires at least as much free disk space for compaction as the size of the largest column family
  * LeveledCompactionStrategy
    * Predictable read latency for rows which are frequently updated and deleted
    * SSTables of a fixed size grouped into levels
    * Each level 10x size of previous level (eg. L0 is ~5 MB, L1 is ~50 MB)
    * Within each level, SSTables are guaranteed to be non-overlapping
  * DateTieredCompactionStrategy
    * All rows in a certain time range stored in a single SSTable
    * All values set to expire at a certain time stored in a single SSTable
* [Right compression mode?](http://docs.datastax.com/en/cql/3.1/cql/cql_reference/tabProp.html?scroll=tabProp__moreCompression)
  * No compression may mean less CPU usage, but more storage, more network if using network store like EBS
* [Right consistency level?](http://www.datastax.com/documentation/cassandra/2.1/cassandra/dml/dml_config_consistency_c.html)
  * You can give up consistency for faster response and availability
* Right data model?
  * Wrong use of secondary index?
  * Using ALLOW FILTERING?
* Reading huge volumes of data in a single query?
  * Buffers may take up memory in Cassandra and in the clients
  * Network timeout may trigger a retry and magnify the issue
* Right data access pattern?
  * Using Cassandra as queue is slow because "Delete" just creates a tombstone. Range queries read them and skip the values.
  * Read before write
* Commit log and data are in same disk?
* nodetool repair being run regularly?
* Using vnodes?
  * Eases load when nodes are added and removed

## Performance tuning and monitoring
* [Reference](http://jonathanhui.com/cassandra-performance-tuning-and-monitoring)
* Enable tracing
* nodetool
  * cfhistograms
  * cfstats
  * tpstats
* OS
  * top
  * iotop
  * iftop
  * vmstat
  * iostat
* cassandra-stress
  * Remember to run for at least an hour to ensure effects of compaction are taken into account.
* Write Survey Mode
