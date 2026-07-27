# Cassandra

- Peer-to-peer distributed system wherein data is replicated among multiple nodes
- Each node is equal and where specific data is stored.
- Nodes are logically arranged in continuous loop called a ring
- Data is spread across the ring using consistent hashing
- This ring of nodes make up the datacenters, which inturn makes up a cluster
- to track health of cluster, nodes communicate with each other every second using peer-to-peer communication protocol called '*Gossip*'
- *Snitches* determine the physical topology of network and thus routing requests and safely distribute data replica across different physical racks to withstand power failures.

Usecases:

- Event logging
- Content management
- Counters - using CounterColumnType, to count and categorize visitors
- Expiring usage - Use expiring columns with TTL

## Logical Hierarchy

Cassandra Cluster -[1..n]-> Keyspace -[1..n]-> Table (Column Family) -[1..n]-> Row -[1..n]-> Column

- A cluster is the outermost container that spans one or more data centers
- A keyspace is analogous to a database in RDBMS; it defines replication strategy and factor
- A table (column family) stores rows, each with a primary key (mandatory)
- A primary key in Cassandra consists of one or more partition keys and zero of more clustering key components.
- The primary goal of partition key is to distribute data evenly across a cluster and query the data efficiently
- Rows contain columns; columns are the smallest unit (name-value pair with a timestamp)
- Columns can be static, clustering, or regular

## Cassandra Architecture

- **Node**: A single machine running Cassandra
- **Virtual Nodes (vnodes)**: Each node is divided into multiple virtual nodes for even distribution
- **Partitioner**: Determines how data is distributed across the ring (Murmur3Partitioner by default)
- **Replication Factor**: Number of copies of data across the cluster, minimum 3
- **Consistency Level**: Configurable per query (ONE, QUORUM, ALL, etc.)

### Replication Strategies

- **SimpleStrategy**: For single data center; places replicas on consecutive nodes clockwise
- **NetworkTopologyStrategy**: For multiple data centers; specifies replication factor per data center

### Consistency Levels

Consistency level is set **per query**, each read and write can choose its own.

| Level | Description |
|-------|-------------|
| `ONE` | Fastest, weakest, responds from closest replica. Stale reads possible. |
| `TWO` | Waits for 2 replicas. |
| `THREE` | Waits for 3 replicas. |
| `QUORUM` | Majority of replicas (`RF/2 + 1`). Balance of consistency and speed. |
| `ALL` | All replicas must respond. Strongest consistency, but unavailable if any replica is down. |
| `LOCAL_ONE` | Responds from the closest replica in the **local data center** only. |
| `LOCAL_QUORUM` | Quorum within the local data center only. |
| `EACH_QUORUM` | Quorum in **every** data center. Used for multi-DC writes. |
| `SERIAL` | Linearizable consistency for reads/writes, checks for pending transactions before responding. Used with lightweight transactions (`IF NOT EXISTS` / `IF`). |
| `LOCAL_SERIAL` | Same as `SERIAL` but restricted to the local data center. |

**Write consistency** example:

```sql
-- Insert with LOCAL_QUORUM consistency
INSERT INTO users (id, name) VALUES (uuid(), 'Alice')
USING CONSISTENCY LOCAL_QUORUM;
```

> Rule of thumb: `W + R > RF` ensures strong consistency for reads-after-writes. E.g., `RF=3, W=QUORUM(2), R=QUORUM(2)` guarantees at least one overlapping replica.

## CQL (Cassandra Query Language)

### Creating a Keyspace

- `'class'` selects the replication strategy
- `'datacenter1'` is the name of a data center in the cluster; replace with actual data center names (e.g., `'us-east'`, `'us-west'`). The value is the replication factor for that data center.
- For `SimpleStrategy`, use `'replication_factor'` instead of data center names:

```sql
CREATE KEYSPACE keyspace_name
WITH replication = {
  'class': 'SimpleStrategy',
  'replication_factor': 3
};
```

For `NetworkTopologyStrategy`, specify replication per data center by name:

```sql
CREATE KEYSPACE keyspace_name
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'datacenter1': 3,
  'datacenter2': 2
};
```

### durable_writes

- `durable_writes` is a keyspace-level option available only with **NetworkTopologyStrategy**
- When set to `true` (default), writes are persisted to the commit log before applying to memtables, ensuring data durability
- When set to `false`, writes bypass the commit log, improving write performance but risking data loss on node failure
- Useful for non-critical or ephemeral keyspaces where some data loss is acceptable

```sql
CREATE KEYSPACE keyspace_name
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'datacenter1': 3
}
AND durable_writes = false;
```

### Use keyspace

```sql
USE keyspace_name;
```

### List all keyspaces

```sql
DESCRIBE keyspaces;
```

### Drop keyspace

```sql
DROP KEYSPACE keyspace_name;
```

### Creating a Table

```sql
CREATE TABLE table_name (
  id UUID PRIMARY KEY,
  name text,
  age int,
  email text
);
```

### With composite primary key

```sql
CREATE TABLE table_name (
  partition_key text,
  clustering_key int,
  data text,
  PRIMARY KEY (partition_key, clustering_key)
);
```

### Composite partition key + clustering keys

The first element in `PRIMARY KEY` is the partition key. Use `(col1, col2)` to create a composite partition key; remaining columns are clustering keys.

```sql
CREATE TABLE sensor_data (
  year int,
  month int,
  day int,
  sensor_id text,
  reading float,
  recorded_at timestamp,
  PRIMARY KEY ((year, month, day), sensor_id, recorded_at)
);
```

- `(year, month, day)`, composite partition key: all rows sharing these three values land on the same partition
- `sensor_id`, first clustering key
- `recorded_at`, second clustering key (further sorting within same sensor_id)

Query must include all partition key columns:

```sql
SELECT * FROM sensor_data
WHERE year = 2026 AND month = 7 AND day = 27;
```

### Controlling clustering order

Define the sort order of clustering columns at table creation using `WITH CLUSTERING ORDER BY`. Default is ASC.

```sql
CREATE TABLE events (
  category text,
  created_at timestamp,
  event_data text,
  PRIMARY KEY (category, created_at)
) WITH CLUSTERING ORDER BY (created_at DESC);
```

This ensures rows within a partition are stored in descending order by `created_at`, avoiding runtime sorting.

### Drop table

```sql
DROP TABLE table_name;
```

### Alter table

```sql
-- Add a column
ALTER TABLE table_name ADD email text;

-- Drop a column (removes data immediately, cannot be re-added with same name)
ALTER TABLE table_name DROP email;

-- Rename a column (only non-PK columns)
ALTER TABLE table_name RENAME age TO years_old;
```

> Dropped columns are tombstoned. Re-adding a column with the same name after dropping is **not allowed**.

### List all tables in keyspace

```sql
DESCRIBE TABLES;
```

### Indexes

Create a secondary index on a non-PK column to query without the partition key.

```sql
-- Create index
CREATE INDEX ON table_name (email);

-- Create index with a custom name
CREATE INDEX email_idx ON table_name (email);
```

Query using the indexed column (no `ALLOW FILTERING` needed):

```sql
SELECT * FROM users WHERE email = 'alice@example.com';
```

#### Drop index

```sql
DROP INDEX email_idx;

DROP INDEX IF EXISTS email_idx;
```

> `IF EXISTS` prevents an error if the index doesn't exist, useful in scripts and migrations.

**Caveats:**
- Secondary indexes work best on **low-cardinality columns** (e.g., status flags)
- High-cardinality indexes (e.g., email) query every node, poor for large clusters
- Index on a clustering column is often more efficient than on a regular column
- Avoid indexes on frequently updated or deleted columns (tombstone amplification)

### Inserting data

```sql
INSERT INTO table_name (id, name, age, email)
VALUES (uuid(), 'John', 30, 'john@example.com');
```

#### Null values and unequal columns

Cassandra is **schema-flexible** for non-PK columns, columns can be omitted or set to null:

```sql
-- Insert with fewer columns; missing columns are treated as absent (no tombstone)
INSERT INTO table_name (id, name) VALUES (uuid(), 'Alice');

-- Explicit null; Cassandra writes a tombstone for that column
INSERT INTO table_name (id, name, age, email)
VALUES (uuid(), 'Bob', null, 'bob@example.com');
```

> **Key difference**: Omitting a column leaves it absent with no storage cost.  
> - Setting `null` explicitly writes a **tombstone**, consuming space and potentially counting toward the `gc_grace_seconds` deadline.  
> - Prefer omission over explicit null.

### Selecting data

```sql
-- Select all
SELECT * FROM table_name;

-- Select with condition (must include partition key)
SELECT * FROM table_name WHERE partition_key = 'value';

-- Select with clustering order
SELECT * FROM table_name
WHERE partition_key = 'value'
ORDER BY clustering_key DESC;

-- Limit results
SELECT * FROM table_name LIMIT 10;
```

### Updating data

```sql
UPDATE table_name
SET age = 31
WHERE id = some_uuid;
```

> Only **non-primary-key columns** can be set. Primary key columns (partition + clustering) are immutable after insert.  
> The WHERE clause must include the **full primary key** (all partition + all clustering columns) to target a single row.

### Deleting data

Only **non-primary-key columns** can be deleted individually. Primary key columns (partition + clustering) cannot be deleted.

```sql
-- Delete a non-PK column from a specific row
DELETE age FROM table_name WHERE id = some_uuid;

-- Delete entire row (including PK columns)
DELETE FROM table_name WHERE id = some_uuid;

-- Delete a subset of rows within a partition (via clustering key filter)
DELETE FROM sensor_data WHERE year = 2026 AND month = 7 AND day = 27 AND sensor_id = 's1';
```

#### What deletion entails (tombstones)

Cassandra never overwrites or erases data in-place. Every `DELETE` writes a **tombstone**, a deletion marker with a timestamp.

- Tombstones mark the affected columns/rows as deleted; data is hidden from reads but still occupies disk space
- During **compaction**, Cassandra reconciles tombstones with live data. If a tombstone is older than `gc_grace_seconds` (default 10 days), the underlying data is permanently removed
- Deleting a **non-PK column** writes a tombstone for just that column's cell within the row
- Deleting an **entire row** writes a tombstone for the row (range tombstone across all its columns)
- Deleting a **subset of rows** (via clustering filter) writes a range tombstone for those clustering keys
- Explicit `null` in `INSERT` also creates a tombstone (same as DELETE)

**Caveats:**
- Excessive tombstones slow down reads, Cassandra must scan past them
- Queries that hit too many tombstones fail with `TombstoneOverwhelmingException` (threshold: 100K by default)
- Avoid frequent DELETE-heavy patterns; prefer TTL for automatic expiry

### TTL (Time-To-Live)

```sql
-- Row expires after 86400 seconds (1 day)
INSERT INTO table_name (id, data)
VALUES (uuid(), 'temp data')
USING TTL 86400;
```

### Counter column type

Used for increment/decrement operations (e.g., page views, likes). Counter columns cannot be mixed with non-counter columns in the same table.

```sql
CREATE TABLE page_views (
  page_id UUID PRIMARY KEY,
  view_count counter
);
```

```sql
-- Increment
UPDATE page_views SET view_count = view_count + 1 WHERE page_id = some_uuid;

-- Decrement
UPDATE page_views SET view_count = view_count - 1 WHERE page_id = some_uuid;
```

```sql
-- Read counter value
SELECT view_count FROM page_views WHERE page_id = some_uuid;
```

> Counters only support increment/decrement via `UPDATE`, `INSERT` is not allowed.

### Batches

Group multiple mutations (INSERT, UPDATE, DELETE) into a single atomic operation. Batches are **not** a performance optimisation, they exist for atomicity.

```sql
BEGIN BATCH
  INSERT INTO users (id, name) VALUES (uuid(), 'Alice');
  INSERT INTO users (id, name) VALUES (uuid(), 'Bob');
  UPDATE accounts SET balance = balance - 100 WHERE user_id = 'abc';
APPLY BATCH;
```

#### Batch types

| Type | Behavior |
|------|----------|
| **LOGGED** (default) | Uses a batch log to ensure atomicity across partitions. Slower but guarantees all-or-nothing. |
| **UNLOGGED** | No batch log. Faster but only atomic within a single partition. Use only when all mutations share the same partition key. |

```sql
-- Unlogged batch (all ops share partition key)
BEGIN UNLOGGED BATCH
  UPDATE orders SET status = 'shipped' WHERE user_id = 'abc' AND order_id = 1;
  UPDATE orders SET status = 'shipped' WHERE user_id = 'abc' AND order_id = 2;
APPLY BATCH;
```

#### Rollback support

Cassandra **does not support rollback**. There is no `ROLLBACK` command.

- LOGGED batches are **atomic but not transactional**, if the coordinator crashes mid-batch, the batch log ensures the batch is replayed on recovery, not rolled back
- UNLOGGED batches have no rollback mechanism; partial writes are possible if the coordinator fails
- There is no `BEGIN TRANSACTION` / `COMMIT` / `ROLLBACK`, Cassandra is an AP system that prioritises availability over transactional guarantees

**Caveats:**
- Batches are **anti-pattern for performance**, they increase coordinator load and latency
- Never use batches to batch unrelated writes; they were designed for atomic updates within a single partition
- Large batches (> 5-10 KB) cause pressure on the coordinator; keep them small

### SSTables and Storage

Cassandra's storage engine is **LSM-tree** (Log-Structured Merge-Tree).

#### Write path

```
Write → Commit Log (disk) → Memtable (memory) → Flush → SSTable (disk)
```

1. **Commit Log**: Every write is appended to the commit log for durability (crash recovery)
2. **Memtable**: Data is written to an in-memory structure (sorted by partition key + clustering key)
3. **Flush**: When a memtable is full or reaches a threshold, it is flushed to disk as an **SSTable**
4. **SSTable**: Immutable, sorted data file on disk. Once written, never modified.

#### SSTable structure

- SSTables are **immutable**, updates and deletes write new SSTables; old data is compacted away later
- Each SSTable contains: data partitions, indexes (partition key → offset), bloom filters (fast partition lookup)
- Bloom filters allow Cassandra to skip SSTables that definitely don't contain the requested partition key

#### Compaction

Cassandra periodically merges multiple SSTables into one via **compaction**:

- **SizeTieredCompactionStrategy (STCS)**: Default. Merges SSTables of similar size. Good for write-heavy workloads.
- **LeveledCompactionStrategy (LCS)**: Organises SSTables into levels (L0, L1, ...). Better read performance, more write overhead.
- **TimeWindowCompactionStrategy (TWCS)**: For time-series data. Compacts SSTables within a time window.

**Benefits of compaction:**
- Removes tombstones older than `gc_grace_seconds`
- Merges updates into a single row version
- Reclaims disk space from deleted/expired data

```sql
-- Configure compaction strategy per table
CREATE TABLE events (
  category text,
  created_at timestamp,
  data text,
  PRIMARY KEY (category, created_at)
) WITH compaction = {
  'class': 'TimeWindowCompactionStrategy',
  'compaction_window_unit': 'DAYS',
  'compaction_window_size': 1
};
```

#### Tombstones revisited

- A tombstone is an SSTable entry marking a deletion
- During compaction, if the tombstone is older than `gc_grace_seconds` (default 10 days, configurable per table), the underlying data is purged
- If a node is down longer than `gc_grace_seconds`, the tombstone may be compacted away before the node repairs, resurrecting deleted data (a known Cassandra edge case)

```sql
-- Configure gc_grace_seconds per table
CREATE TABLE users ( ... )
WITH gc_grace_seconds = 86400;  -- 1 day instead of 10
```

> **Key insight**: Because SSTables are immutable, a single row's latest state may be spread across multiple SSTables. Reads must merge all SSTables + memtable to reconstruct the current value. This is called **read amplification**, mitigated by compaction.

### Supported datatypes

- text / varchar
- int / bigint / smallint / tinyint
- float / double / decimal
- boolean
- uuid / timeuuid
- timestamp / date / time
- blob
- inet (IP address)
- counter
- frozen (for nested collections)
- tuple
- list / set / map

### Filtering conditions

- **WHERE on partition key** (required): must include the full partition key

    ```sql
    SELECT * FROM users WHERE user_id = 'abc';
    ```

    If the partition key is composite, **all columns** of the composite key must be provided:

    ```sql
    SELECT * FROM sensor_data
    WHERE year = 2026 AND month = 7 AND day = 27;
    ```

    Omitting any partition key column will cause Cassandra to reject the query (or require ALLOW FILTERING).

- **WHERE on clustering columns**: allowed after partition key filter

    ```sql
    SELECT * FROM users WHERE user_id = 'abc' AND joined_date > '2024-01-01';
    ```

- **ALLOW FILTERING**: permits filtering on non-indexed columns

    ```sql
    SELECT * FROM users WHERE email = 'a@b.com' ALLOW FILTERING;
    ```

    **Penalties & Caveats:**
    - Cassandra must scan **all rows in all partitions** of the table, then filter client-side, O(n) across the entire dataset
    - Can cause **timeouts**, high latency, and increased GC pressure on large tables
    - Will **fail** on large datasets if the scan exceeds the coordinator node's timeout
    - Never use on production queries hit frequently; only for **ad-hoc analytics** or small tables
    - Prefer a **secondary index** or denormalized table designed for the query pattern instead

- **IN**: check multiple partition key values

    ```sql
    SELECT * FROM users WHERE user_id IN ('abc', 'def');
    ```

- **CONTAINS**: check for value in collection column

    ```sql
    SELECT * FROM users WHERE tags CONTAINS 'admin' ALLOW FILTERING;
    ```

- **ORDER BY**: only on clustering columns in same order as defined

    ```sql
    SELECT * FROM events
    WHERE category = 'click'
    ORDER BY created_at DESC;
    ```

> Queries in Cassandra are designed to be **partition-key-first**, always filter by partition key for efficiency. Full table scans are anti-patterns.

### CQL limitations

CQL intentionally omits many SQL features to enforce Cassandra's distributed design:

| Feature | Supported? | Notes |
|---------|-----------|-------|
| `AND` | Yes | Only supported logical operator in WHERE |
| `OR` | No | Use `IN` on partition key or multiple queries |
| `JOIN` | No | Denormalize data into a single table |
| Subqueries | No | Not supported at all |
| `GROUP BY` | Limited | Only on partition key (Cassandra 3.10+) |
| `HAVING` | No | Filter in application layer |
| `UNION` / `INTERSECT` | No | Not supported |
| Aggregates (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`) | Limited | Only within a single partition; cross-partition requires `ALLOW FILTERING` with severe penalty |
| `DISTINCT` | Yes | Only on partition key columns |
| `LIKE` | No | Use secondary index with `CONTAINS` or SASI index |
| `BETWEEN` | No | Use range queries with `>=` and `<=` on clustering columns |
| `NOT` / `!=` | No | Not supported in WHERE |
| `IS NULL` | No | Cassandra has no null concept; omitted columns are simply absent |
