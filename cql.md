##### Primary key may be
* Simple without clustering column
  * PRIMARY KEY (pk_col)
* Composite without clustering column
  * PRIMARY KEY ((pk_col_1, .., pk_col_x))
  * Useful for creating buckets
* Simple with clustering column (compound)
  * PRIMARY KEY (pk_col, c_col_1, .., c_col_y)
* Composite with clustering column
  * PRIMARY KEY ((pk_col_1, .., pk_col_x), c_col_1, .., c_col_y)

##### Clustering columns can be used to filter data
* Equality operators allowed upto the column having range search
* WHERE pk_col=? AND c_col_1=? AND ... AND c_col_n < ?

##### Normal INSERT is actually “insert or replace”
* At the very minimum, you need to insert Partition Key +
  * Static columns, or
  * Clustering columns
* Non-key values are overwritten
* Use IF NOT EXISTS if this is not desired. This is expensive.

##### UPDATE is also “insert or replace” for the values being updated
* WHERE clause for an UPDATE can and should have only
  * PRIMARY KEY columns when updating any non-special column
  * Partition Key when updating only Static column
* Obviously, PRIMARY KEY columns cannot be updated
* Conditional UPDATE via IF col_1 rel_op ? AND … AND col_n rel_op ?
  * rel_op is one of: =, <, <=, >, >=
  * This is expensive
  * Note that OR is not supported.
  * There is no “IS NULL” or “= null” operator. Use INSERT..IF NOT EXISTS.

##### Writing data as “null” actually deletes the column
* Value may be deleted even before they are ever inserted!

##### Any INSERT or UPDATE may be “partial”
* Value of a column is determined by the latest “written” data
  * Make sure clients and servers have their clocks sync’d
* Compaction brings together latest values of columns spread across SSTables to one SSTable. This is required because Cassandra does not read before write before INSERT or UPDATE.

##### The WHERE condition for a DELETE can only have ‘=’ operator on the PRIMARY KEY columns
* At the very minimum, you need to specify the partition key; you may skip the right-most clustering columns

##### Each value can be set to expire via TTL
* Tombstones generated after TTL expires

##### Static column
* One value per partition key
* Can be used in Light Weight Transactions to solve race condition on a queue

##### Secondary Indexes are stored locally in each data node
* Best for low cardinality data for values that do not get updated frequently
* All nodes need to be touched to get the required data, but avoids load on server having popular partition keys
* Only applicable for a single column

##### Global Indexes coming in 3.0
* Use tables and insert in BATCH if required for now

##### BATCH
* Use to atomically insert (materialized view) data across multiple tables atomically
* Cassandra offers row level atomicity
* 1st row inserted will be visible before the last row inserted is inserted
* Isolation is limited to rows in a table having the same partition key

##### Range search for partition key allowed only if using ByteOrderPartitioner
* Default is Murmur3Partitioner

##### IN predicate
* Use sparingly - multiple nodes may be required to satisfy the request. Client drivers send request asynchronously, performance may be better if using single prepared queries.
* Allowed for all clustering columns and the last column in a composite key

##### ORDER BY clause
* Allowed for clustering columns
* Make sure table definition has the desired order in CLUSTERING ORDER BY clause

##### Collections can be used to store small sets of, possibly unstructured, data
* MAP: key-value pairs, unique keys, ordered by keys
* SET: unique values, ordered by values
* LIST: sequence of (non-unique) values, ordered by position
* Easily updated using + and - operators

##### UUID and TimeUUID generate unique values
* TimeUUID encodes the time, used for comparison on range and order clauses. Can be used if incremental unique value is required.

##### TIMESTAMP stores epoch time in ms

##### COUNTER
* Can only be used in a table in which non-counter columns are part of the PRIMARY KEY
* Requires read before write internally to make sure value is correct when nodes go down
* Not an idempotent operation. Failures, esp. timeout, may have actually updated the value. Hence, use only if approximate values are good enough.
