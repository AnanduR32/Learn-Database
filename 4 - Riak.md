# Riak

- Distributed key-value NoSQL database inspired by Amazon's Dynamo paper
- Written in Erlang, designed for fault-tolerance and high availability
- Data replicated across multiple nodes with tunable consistency

Usecases:

- Session storage (Fast read/write with configurable replication)
- User profiles (Key-value lookup with metadata)
- Shopping cart data (CRDTs handle concurrent edits without conflicts)
- Log aggregation (Time-series data with key-based sharding)

## Logical Hierarchy

Riak Cluster -[1..n]-> Bucket Type -[1..n]-> Bucket -[1..n]-> Key -> Value (with metadata)

- A cluster consists of multiple physical nodes forming a ring
- A bucket type groups buckets with the same configuration (replication, data type)
- A bucket is a logical namespace for keys (like a table)
- Each key maps to a value stored along with metadata (links, tags, vclock)

## Riak Architecture

- **Ring**: Nodes are arranged in a 160-bit hash ring using consistent hashing
- **Virtual Nodes (vnodes)**: Each physical node hosts multiple vnodes for even distribution
- **N_val**: Replication factor, number of nodes a key is replicated to (default 3)
- **Hinted Handoff**: If a replica node is down, another node temporarily accepts writes
- **Read Repair**: During reads, stale replicas are updated in the background
- **Gossip Protocol**: Nodes share ring state and health information

### Tunable Consistency (R / W / DW / PW / PR)

- **N_val**: Number of replicas per key
- **R**: Number of successful read responses required (default 2)
- **W**: Number of successful write responses required (default 2)
- **DW**: Number of successful durable write responses (written to disk)
- **PW**: Number of successful primary replica writes
- **PR**: Number of successful primary replica reads

Common configurations:

- `R=1, W=1` : Fastest, lowest consistency
- `R=N, W=N` : Strongest consistency, slowest
- `R=QUORUM, W=QUORUM` : Quorum-based (N/2 + 1)

### Conflict Resolution

- **Vector Clocks**: Track causal history; siblings resolved on read by the application
- **CRDTs** (Riak 2.0+): Convergent data types (maps, sets, counters, flags, registers) that resolve automatically
- Last-Write-Wins (LWW) is the fallback if vector clocks are not used

## Riak HTTP API

Riak exposes a RESTful HTTP API by default on port 8098.

### Store a value (PUT)

```http
PUT /buckets/my_bucket/keys/my_key
Content-Type: application/json

{ "name": "Alice", "age": 30 }
```

### Retrieve a value (GET)

```http
GET /buckets/my_bucket/keys/my_key
```

### Delete a value (DELETE)

```http
DELETE /buckets/my_bucket/keys/my_key
```

### List all keys in a bucket

```http
GET /buckets/my_bucket/keys?keys=true
```

### List all buckets

```http
GET /buckets?buckets=true
```

### Fetch bucket properties

```http
GET /buckets/my_bucket/props
```

### Set bucket properties (N_val, allow_mult, etc.)

```http
PUT /buckets/my_bucket/props
Content-Type: application/json

{
  "props": {
    "n_val": 3,
    "allow_mult": true
  }
}
```

### Secondary Indexes (2i)

```http
PUT /buckets/my_bucket/keys/my_key
Content-Type: application/json
x-riak-index-age-bin: 30
x-riak-index-email: alice@example.com

{ "name": "Alice", "age": 30 }
```

Query by secondary index:

```http
GET /buckets/my_bucket/index/age-bin/30

GET /buckets/my_bucket/index/email_bin/alice@example.com

GET /buckets/my_bucket/index/age-bin/20/to/40
```

## Riak Data Types (CRDTs)

Riak supports convergent data types that resolve conflicts automatically:

- **Counter**: Increment/decrement operations, resolves by merging counts
- **Set**: Add/remove elements, resolves by union of adds
- **Map**: Nested structure combining counters, sets, registers, flags, and maps
- **Register**: Stores a binary value (last-write-wins)
- **Flag**: Boolean value (enable/disable)

### CRDT operations via HTTP

```http
POST /buckets/my_bucket/datatypes/counter_key
Content-Type: application/json

{
  "increment": 5
}
```

```http
POST /buckets/my_bucket/datatypes/set_key
Content-Type: application/json

{
  "add_all": ["apple", "banana"]
}
```

## Riak Search (Yokozuna)

- Built on Apache Solr
- Full-text search across Riak values
- Indexes are automatically updated on writes

```http
GET /search/query/my_index?q=name:Alice
```

### Strong Consistency (Riak 2.0+)

Riak also supports **strong consistency** per-key using a consensus subsystem (Paxos-based):

```http
# Create a bucket type with strong consistency
PUT /riak-types/strong_consistency_type/props
Content-Type: application/json

{
  "props": {
    "consistent": true
  }
}
```

```http
# Use the strong consistency bucket type
PUT /buckets/strong_consistency_type:my_bucket/keys/my_key
Content-Type: application/json

{ "counter": 42 }
```

> Strong consistency in Riak has a performance cost, it sacrifices latency and availability during partitions. Use only when strictly needed (e.g., atomic counters, locks).

### Batch Operations

Riak supports batching multiple operations (not atomic, no rollback):

```http
POST /riak/bucket/keys/bulk
Content-Type: multipart/mixed

--batch
Content-Type: multipart/alternative

PUT /buckets/users/keys/alice
{ "name": "Alice" }

--batch
PUT /buckets/users/keys/bob
{ "name": "Bob" }

--batch--
```

- Batches reduce network round-trips but are **not atomic**, individual operations may succeed or fail independently
- Riak has **no rollback** mechanism. There is no `ROLLBACK` or transaction support
- For atomic operations within a key, use CRDTs or conditional puts (If-Match / If-None-Match headers)

### Tombstones and Siblings

**Tombstones:**
- When a key is deleted, Riak writes a **tombstone** (a deletion marker with a timestamp)
- Tombstones are eventually reaped via **active anti-entropy** or `delete_mode` setting
- Default `delete_mode = 1` (immediate tombstone); set to `0` to disable tombstones entirely

**Siblings:**
- When `allow_mult = true`, concurrent writes to the same key produce **siblings** (multiple values)
- Siblings are stored and returned to the client, which resolves them
- Vector clocks track causality; **sibling explosion** occurs if too many clients write concurrently
- **Vector clock pruning**: Riak prunes old vector clocks (default: max 5000, prune 1000) to prevent unbounded growth

```http
# Enable siblings on a bucket
PUT /buckets/my_bucket/props
Content-Type: application/json

{
  "props": {
    "allow_mult": true,
    "last_write_wins": false
  }
}
```

### Storage Backend

Riak supports pluggable storage backends:

| Backend | Description |
|---------|-------------|
| **Bitcask** | Default. In-memory hash table with append-only writes. All keys fit in RAM. Fast writes, high read performance for hot keys. |
| **LevelDB** | LSM-tree on disk. Handles datasets larger than RAM. Good for write-heavy workloads with many unique keys. |
| **Memory** | Pure in-memory. Data lost on restart. Use for caching or ephemeral data. |

#### Write path (Bitcask)

```
Write → Data File (append-only on disk) → Hint File (key → offset mapping) → In-memory Keydir (hash table)
```

- Every write appends to the current **data file** on disk
- A **hint file** stores key-to-offset mappings for fast startup
- The **keydir** (in-memory hash table) maps each key to its latest file + offset for O(1) reads
- Old data files are merged during **compaction** (merging multiple files, discarding overwritten/deleted keys)

#### Write path (LevelDB)

```
Write → Log (disk) → Memtable (sorted in memory) → SSTable (immutable, sorted on disk) → Compaction
```

- Same LSM-tree approach as Cassandra
- Multiple SSTables per key; read requires merging
- Bloom filters for quick negative lookups

### MapReduce

Riak supports distributed MapReduce for processing data across the cluster:

```http
POST /mapred
Content-Type: application/json

{
  "inputs": "my_bucket",
  "query": [
    {
      "map": {
        "language": "erlang",
        "source": "fun(Obj, _) -> [Obj.key] end"
      }
    },
    {
      "reduce": {
        "language": "erlang",
        "source": "fun(List, _) -> [length(List)] end"
      }
    }
  ]
}
```

> MapReduce in Riak is useful for ad-hoc analytics, but not designed for real-time queries. It streams data from all vnodes, applying map functions in parallel and reducing results on the coordinating node.

## Consistency vs Availability Settings

| Quorum  | Behavior |
|---------|----------|
| R=1, W=1 | Fast, eventual consistency, stale reads possible |
| R=N, W=N | Strong consistency, but unavailable if any replica fails |
| R=QUORUM, W=QUORUM | Balance of consistency and availability |

> Riak prioritizes **availability and partition tolerance** (AP in CAP theorem). Tune R/W/N values to trade consistency for latency and fault-tolerance.

## Riak Core, KV, Search Summary

**Riak Core** is the distributed systems foundation -- consistent hashing ring, vnodes, gossip protocol, hinted handoff. It provides the scalability and fault-tolerance primitives that everything else builds on.

**Riak KV** is the main key-value store built on Core. It offers tunable consistency (R/W/N values), CRDTs, vector clocks, and pluggable backends (Bitcask, LevelDB). Use for session storage, user profiles, shopping carts, log aggregation.

**Riak Search** (Yokozuna) is a full-text search layer on top of KV, powered by Apache Solr. It auto-indexes values on write and enables Solr queries. Use when you need to search within KV data by content rather than by key.
