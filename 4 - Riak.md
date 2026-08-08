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

### Tunable Consistency — Deep Dive

Riak's consistency model is entirely configurable per-request or per-bucket. Every parameter controls **how many vnodes must acknowledge** an operation before Riak considers it successful.

---

#### `n_val` — Replication Factor

```
n_val = 3  (default)
```

- The number of **physical copies** of every key stored in the cluster
- Riak picks `n_val` vnodes along the ring starting from the key's hash position
- These vnodes are ideally on **different physical nodes** (Riak tries to spread them)
- **Changing `n_val` on an existing bucket is dangerous**: existing data keeps the old replication count until it is read-repaired or rewritten
- All other quorum values (R, W, PR, PW, DW) are bounded by `n_val`

```
n_val=3 means: key "user:42" → replicated to vnodes on Node A, Node B, Node C
```

---

#### Primary vs Fallback Vnodes

This distinction is critical for understanding PR and PW:

- **Primary vnodes**: the `n_val` vnodes that are **normally responsible** for a key (determined by consistent hashing)
- **Fallback vnodes**: temporary replacement vnodes on other nodes that hold data during a failure (via **Hinted Handoff**)

When a primary node goes down:
1. Writes are redirected to a fallback vnode on another node
2. The fallback holds the data temporarily
3. When the primary recovers, hinted handoff transfers the data back

---

#### `R` — Read Quorum

```
R = 2  (default, also expressed as "quorum")
```

- Minimum number of **any** vnodes (primary or fallback) that must return a successful response for a read
- Lower R = faster reads, but higher chance of reading stale data
- Riak sends read requests to all `n_val` vnodes simultaneously and returns as soon as `R` respond

| R value | Behavior |
|---------|----------|
| `R=1` | Return immediately from the fastest vnode. Fastest, most stale |
| `R=2` (quorum) | Wait for majority. Balanced consistency |
| `R=n_val` (all) | Wait for all replicas. Slowest, most consistent |

---

#### `W` — Write Quorum

```
W = 2  (default, also expressed as "quorum")
```

- Minimum number of **any** vnodes that must acknowledge a write before returning success
- A `W` acknowledgement means the vnode received and stored the write in memory (not necessarily flushed to disk)
- Lower W = faster writes, higher risk of data loss on immediate crash

| W value | Behavior |
|---------|----------|
| `W=1` | Return after 1 vnode acks. Fastest writes, risk of loss |
| `W=2` (quorum) | Return after majority. Balanced |
| `W=n_val` (all) | Wait for all replicas. Slowest, safest |

---

#### `DW` — Durable Write Quorum

```
DW = quorum  (default)
```

- Like `W`, but the acknowledgement is only sent after the vnode has **flushed the write to disk**
- A subset of `W` acks: if `DW=2`, at least 2 vnodes must confirm the data is on disk
- `DW` is always `<= W` in practice (you need W acks, and DW of those must be durable)
- Use DW when you cannot tolerate data loss even on power failure

```
W=3, DW=2 → 3 vnodes receive the write, at least 2 must confirm disk flush
```

> **DW=0**: Disable durability requirement entirely — Riak returns as soon as the write hits any in-memory store. Maximum performance, minimum safety.

---

#### `PW` — Primary Write Quorum

```
PW = 0  (default, disabled)
```

- Minimum number of **primary** vnodes (not fallbacks) that must acknowledge a write
- If fewer than `PW` primary vnodes are reachable, the write fails — **even if fallback vnodes are available**
- This is stricter than `W` because it rejects writes that would only land on fallbacks

```
n_val=3, PW=2:
  → If 2 primary nodes are up → write succeeds
  → If only 1 primary + 1 fallback are up → write FAILS (only 1 primary ack)
```

Use PW when you need to ensure data lands on the authoritative nodes, not temporary stand-ins.

---

#### `PR` — Primary Read Quorum

```
PR = 0  (default, disabled)
```

- Minimum number of **primary** vnodes that must respond to a read
- Guarantees you're reading from the authoritative nodes, not potentially-stale fallbacks
- If fewer than `PR` primary vnodes are reachable, the read fails

```
n_val=3, PR=1:
  → Always returns the latest value from at least one primary node
  → Rejects reads if all 3 primary nodes are down (even if fallbacks exist)
```

| Setting | Risk Without It |
|---------|----------------|
| No PR (R only) | You might read from a fallback vnode that has stale data from before a failure |
| PR=1 | Guarantees at least one primary node is involved — more up-to-date |
| PR=quorum | Very strong freshness guarantee, but less available during failures |

---

#### `RW` — Delete Quorum (Legacy)

```
RW = quorum  (default, deprecated in Riak 2.0+)
```

- Legacy parameter used in older Riak versions to control the quorum for **delete operations**
- Controls how many vnodes must ack a delete before returning success
- In Riak 2.0+, deletes use `R` and `W` separately (read to get vclock, then write tombstone)
- Still accepted for backward compatibility but `R + W` is now the preferred delete path

---

#### Quorum Shorthand Values

Instead of integers, Riak accepts symbolic names:

| Shorthand | Meaning |
|-----------|---------|
| `all` | Must equal `n_val` — every replica must respond |
| `quorum` | `floor(n_val / 2) + 1` — majority (default for R, W, DW) |
| `one` | Equivalent to `1` — any single vnode |

With `n_val=3`:
- `quorum` = `floor(3/2) + 1` = **2**
- `all` = **3**
- `one` = **1**

---

#### How Parameters Interact — The Full Picture

```
A write request with n_val=3, W=2, DW=1, PW=1:

  Coordinator node
       │
       ├──► Primary vnode A  ─ acks W ─ acks DW (flushed to disk) ─ acks PW ✓
       ├──► Primary vnode B  ─ acks W ✓
       └──► Primary vnode C  ─ (slow, hasn't responded yet)

  Result: W=2 ✓ (A+B), DW=1 ✓ (A flushed), PW=1 ✓ (A is primary)
  → Write SUCCEEDS, C still writes asynchronously
```

```
A read request with n_val=3, R=2, PR=1:

  Coordinator node
       │
       ├──► Primary vnode A   → returns value v2 (latest vclock)
       ├──► Primary vnode B   → returns value v1 (stale, hasn't received Read Repair yet)
       └──► Fallback vnode F  → returns value v1

  R=2 satisfied (A+B responded), PR=1 satisfied (A is primary)
  → Read SUCCEEDS with v2. Read Repair pushes v2 to B and F in background.
```

---

#### Consistency vs Availability Trade-off

```
More Available (AP)                    More Consistent (CP)
       │                                        │
  R=1, W=1                              R=n, W=n, PR=n, PW=n
  PR=0, PW=0                            DW=n
  Tolerates node failures                Fails if any replica is down
  Reads may be stale                     Always reads the latest data
```

| Config | Consistency | Availability | Use Case |
|--------|-------------|--------------|----------|
| `R=1, W=1` | Eventual | Highest | Session caches, ephemeral state |
| `R=quorum, W=quorum` | Strong-ish | Good | User profiles, shopping carts |
| `R=all, W=all` | Linearizable-ish | Low | Critical counters (prefer CRDTs instead) |
| `PR=1, PW=1` | Primary-guaranteed | Medium | Avoid stale fallback reads/writes |
| `W=quorum, DW=quorum` | Durable | Medium | Financial records, audit logs |

---

#### Setting Quorum Per-Request (HTTP)

Override bucket defaults on individual requests via query parameters:

```http
GET /buckets/my_bucket/keys/my_key?r=1&pr=1
```

```http
PUT /buckets/my_bucket/keys/my_key?w=3&dw=2&pw=2
Content-Type: application/json

{ "balance": 9999 }
```

```http
DELETE /buckets/my_bucket/keys/my_key?r=quorum&w=quorum
```

---

#### Setting Quorum Per-Bucket (Bucket Properties)

```http
PUT /buckets/my_bucket/props
Content-Type: application/json

{
  "props": {
    "n_val": 3,
    "r": "quorum",
    "w": "quorum",
    "dw": "quorum",
    "pr": 0,
    "pw": 0,
    "rw": "quorum",
    "allow_mult": true
  }
}
```

Common configurations:

- `R=1, W=1` : Fastest, lowest consistency
- `R=N, W=N` : Strongest consistency, slowest
- `R=QUORUM, W=QUORUM` : Quorum-based (N/2 + 1), the sweet spot

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

Index names must end with `_bin` (string) or `_int` (integer). Pass them as HTTP headers on write:

```http
PUT /buckets/my_bucket/keys/my_key
Content-Type: application/json
x-riak-index-age_int: 30
x-riak-index-email_bin: alice@example.com

{ "name": "Alice", "age": 30 }
```

Query by secondary index:

```http
GET /buckets/my_bucket/index/age_int/30

GET /buckets/my_bucket/index/email_bin/alice@example.com

GET /buckets/my_bucket/index/age_int/20/40
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

Riak KV has **no native HTTP bulk/batch endpoint**. The recommended approaches are:

- **Parallel individual requests**: Issue multiple `PUT`/`GET`/`DELETE` requests concurrently from your application
- **MapReduce** (`POST /mapred`): For batch processing or aggregation across many keys (see MapReduce section below)
- **Protocol Buffers API** (port 8087): More efficient than HTTP for high-throughput bulk writes

> Batches in Riak have **no rollback** mechanism. There is no transaction support — individual operations may succeed or fail independently.

### Tombstones and Siblings

**Tombstones:**

- When a key is deleted, Riak writes a **tombstone** (a deletion marker with a timestamp)
- Tombstones are eventually reaped via **active anti-entropy** or based on the `delete_mode` setting
- `delete_mode` options:
  - `keep` — tombstones are never removed (safest, prevents object resurrection)
  - `immediate` — tombstone is removed instantly (risk: deleted objects can resurrect on node re-join)
  - integer (milliseconds) — tombstone is reaped after the given duration; **default is `3000` (3 seconds)**

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
| --- | --- |
| **Bitcask** | Default. In-memory hash table with append-only writes. All keys fit in RAM. Fast writes, high read performance for hot keys. |
| **LevelDB** | LSM-tree on disk. Handles datasets larger than RAM. Good for write-heavy workloads with many unique keys. |
| **Memory** | Pure in-memory. Data lost on restart. Use for caching or ephemeral data. |

#### Write path (Bitcask)

```txt
Write → Data File (append-only on disk) → Hint File (key → offset mapping) → In-memory Keydir (hash table)
```

- Every write appends to the current **data file** on disk
- A **hint file** stores key-to-offset mappings for fast startup
- The **keydir** (in-memory hash table) maps each key to its latest file + offset for O(1) reads
- Old data files are merged during **compaction** (merging multiple files, discarding overwritten/deleted keys)

#### Write path (LevelDB)

```txt
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

| Quorum | Behavior |
| --- | --- |
| R=1, W=1 | Fast, eventual consistency, stale reads possible |
| R=N, W=N | Strong consistency, but unavailable if any replica fails |
| R=QUORUM, W=QUORUM | Balance of consistency and availability |

> Riak prioritizes **availability and partition tolerance** (AP in CAP theorem). Tune R/W/N values to trade consistency for latency and fault-tolerance.

## Riak Core, KV, Search Summary

**Riak Core** is the distributed systems foundation -- consistent hashing ring, vnodes, gossip protocol, hinted handoff. It provides the scalability and fault-tolerance primitives that everything else builds on.

**Riak KV** is the main key-value store built on Core. It offers tunable consistency (R/W/N values), CRDTs, vector clocks, and pluggable backends (Bitcask, LevelDB). Use for session storage, user profiles, shopping carts, log aggregation.

**Riak Search** (Yokozuna) is a full-text search layer on top of KV, powered by Apache Solr. It auto-indexes values on write and enables Solr queries. Use when you need to search within KV data by content rather than by key.
