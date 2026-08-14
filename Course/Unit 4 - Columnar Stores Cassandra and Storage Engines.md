# Unit IV — Wide-Column Architecture, Cassandra Data Modeling & Storage Engines
### 25CSA642A — NoSQL Databases

This unit explores wide-column stores through **Apache Cassandra** and underlying storage engine mechanics. It details Log-Structured Merge-Trees (LSM-Trees) vs. B-Trees, Bloom filter mathematics, compaction strategies, the Chebotko query-driven data modeling methodology, partition sizing mathematics, collections, User-Defined Types (UDTs), Lightweight Transactions (Paxos LWT), and Python application development with the DataStax Cassandra driver.

**Roadmap of this file:**
1. Wide-Column Store Architecture & Internal Mechanics
2. Storage Engine Internals — LSM-Trees vs. B-Trees, Memtables, SSTables & Tombstones
3. Mathematical Foundations — Bloom Filters & Compaction Strategies
4. Query-Driven Data Modeling — Chebotko Methodology & Partition Sizing
5. Advanced CQL Data Types — Collections, UDTs, TTL & Counters
6. Consistency Tuning & Lightweight Transactions (Paxos Consensus)
7. Cassandra Application Development with Python (`cassandra-driver`)

---

## 1. Wide-Column Store Architecture & Internal Mechanics

### 1.1 Core Concepts (Simplified)
- **Masterless Peer-to-Peer Ring:** Every node in a Cassandra cluster plays an identical role (no master node, eliminating single points of failure).
- **Consistent Hashing:** Partition keys are hashed using the Murmur3 partitioner into a token range between $-2^{63}$ and $2^{63}-1$ to determine node placement along the cluster ring.
- **Virtual Nodes (vnodes):** Distributes physical node storage across hundreds of token ranges, smoothing cluster rebalancing.
- **Gossip Protocol:** Nodes exchange cluster topology and node health metadata every second via a decentralized P2P epidemic protocol.
- **Snitches:** Map network topology (racks and datacenters) to ensure replicas are placed across independent power supplies and physical racks.

---

## 2. Storage Engine Internals — LSM-Trees vs. B-Trees

### 2.1 LSM-Trees vs. B-Trees Comparison

| Property | B-Tree (RDBMS / MongoDB WiredTiger) | LSM-Tree (Cassandra / Bigtable / RocksDB) |
|---|---|---|
| **Write Mechanism** | In-place page updates; random disk I/O | Append-only sequential writes to disk log & SSTables |
| **Write Latency** | Higher (requires disk seeks or journal sync) | Sub-millisecond (sequential append) |
| **Read Path** | $O(\log n)$ page search; fast point reads | Must check Memtable + Bloom Filter + multiple SSTables |
| **Read Amplification** | Low ($1$ to $3$ disk page seeks) | Moderate to High (mitigated by Bloom filters & compaction) |
| **Write Amplification** | Moderate | High during background compaction merges |
| **Deletes** | In-place removal / page rebalancing | Writes a **Tombstone** marker; purged during compaction |

### 2.2 The Cassandra Write & Read Paths

#### The Write Path (Append-Only):
```
Client Write ──► Coordinator Node ──► CommitLog (Disk Sequential) + Memtable (RAM) ──► ACK to Client
                                                              │
                                            (When full)       ▼
                                                       Flush to SSTable (Disk)
```
1. **CommitLog:** Appended sequentially to disk for crash-recovery durability.
2. **Memtable:** In-memory sorted structure (SkipList) accepting mutations concurrently.
3. **SSTable (Sorted String Table):** Immutable on-disk file containing sorted rows, index summaries, and Bloom filters.

#### The Read Path:
1. Check **Memtable** for active updates.
2. Query **Bloom Filters** to rule out SSTables that definitely do *not* contain the partition key.
3. Check the **Key Cache** and **Partition Summary** to locate the exact SSTable byte offset.
4. Merge matching row fragments across surviving SSTables and Memtables, reconciling timestamps via **Last-Write-Wins (LWW)** and filtering out expired **Tombstones**.

---

## 3. Mathematical Foundations — Bloom Filters & Compaction Strategies

### 3.1 Bloom Filter Mathematics
A Bloom filter is a space-efficient probabilistic data structure that tests whether an element is a member of a set:
- **True Negatives:** 100% guaranteed (if it says the key is not in the SSTable, it is definitely not there; disk seek avoided).
- **False Positives:** Possible with probability $p$.

Given $m$ filter bits, $n$ inserted keys, and $k$ independent hash functions:
$$p \approx \left(1 - e^{-kn/m}\right)^k$$

The optimal number of hash functions $k$ minimizing false positive rate $p$ is:
$$k = \frac{m}{n} \ln 2 \approx 0.693 \frac{m}{n}$$

*Takeaway:* Bloom filters save millions of disk reads by eliminating unnecessary SSTable lookups for non-existent partition keys in $O(1)$ memory operations.

### 3.2 Compaction Strategies

```
Small SSTables ──► [Compaction Engine] ──► Consolidated SSTable (Tombstones Purged)
```

1. **Size-Tiered Compaction Strategy (STCS):**
   - Merges SSTables of similar sizes.
   - *Best for:* Write-heavy workloads (e.g., event streaming, logging).
2. **Leveled Compaction Strategy (LCS):**
   - Divides SSTables into exponentially sized levels ($L_0, L_1, L_2, \dots$) where each level is $10\times$ larger than the previous.
   - Within each level above $L_0$, keys never overlap across SSTables.
   - *Best for:* Read-heavy workloads ($90\%$ reads), guaranteeing $90\%$ of reads hit at most 1 SSTable.
3. **Time-Window Compaction Strategy (TWCS):**
   - Compacts SSTables within discrete temporal windows (e.g., 1 day).
   - *Best for:* Time-series / IoT data with fixed TTLs.

---

## 4. Query-Driven Data Modeling — Chebotko Methodology & Partition Sizing

### 4.1 Chebotko Design Methodology
In Cassandra, you **do not design around entities; you design around queries**:
1. **Rule 1 — One Table per Query:** If your application has 5 distinct query patterns, design 5 specialized tables.
2. **Rule 2 — Minimize Partition Lookups:** Every query should ideally read data from exactly **1 partition**.
3. **Rule 3 — Equality on Partition Key, Ranges on Clustering Key:** Partition keys route to nodes; clustering keys sort rows sequentially on disk.

```
Primary Key = ((Partition_Key_Cols), Clustering_Key_Cols)
                 ▲                          ▲
                 │                          │
           Determines Node            Determines On-Disk
           Hash Ring Location         Sorting & Range Query
```

### 4.2 Partition Sizing Formula & Rules
A partition that is too large causes JVM Garbage Collection pauses and node crashes.

**Partition Sizing Formula:**
$$\text{Size}_{\text{partition}} = \sum (\text{Static Columns}) + N_{\text{rows}} \times \left( \sum (\text{Clustering Columns}) + \sum (\text{Regular Columns}) + \text{Overhead} \right)$$

**Safety Thresholds:**
- **Maximum Partition Disk Size:** $\le 100\text{ MB}$ (ideal: $< 20\text{ MB}$).
- **Maximum Number of Cells per Partition:** $\le 100,000$ cells.

---

## 5. Advanced CQL Data Types — Collections, UDTs, TTL & Counters

### 5.1 Collections & User-Defined Types (UDT)

```sql
-- 1. Create a User-Defined Type (UDT)
CREATE TYPE address_type (
  street text,
  city text,
  zip_code int
);

-- 2. Create Table with Collections (Set, List, Map) and UDT
CREATE TABLE user_profiles (
  user_id UUID,
  account_tier text,
  emails set<text>,                 -- Unique elements
  recent_logins list<timestamp>,     -- Ordered sequence
  metadata map<text, text>,          -- Key-Value attributes
  home_address frozen<address_type>, -- Immutable compound struct
  PRIMARY KEY (user_id)
);

-- 3. Updating Collections in CQL
UPDATE user_profiles
SET emails = emails + {'alice@dev.com'},
    metadata['device'] = 'iPhone 15',
    recent_logins = [toTimestamp(now())] + recent_logins
WHERE user_id = 7f8c5b38-1234-4567-89ab-cdef01234567;
```

### 5.2 Expiring Columns (TTL) & Counter Tables
```sql
-- Expiring column: Automatically purges after 24 hours (86400 seconds)
INSERT INTO active_sessions (session_id, user_id)
VALUES (uuid(), 'user_101')
USING TTL 86400;

-- Counter Table: Dedicated table for atomic increments
CREATE TABLE article_metrics (
  article_id UUID PRIMARY KEY,
  view_count counter,
  share_count counter
);

UPDATE article_metrics
SET view_count = view_count + 1
WHERE article_id = 7f8c5b38-1234-4567-89ab-cdef01234567;
```

---

## 6. Consistency Tuning & Lightweight Transactions (Paxos Consensus)

### 6.1 Tunable Consistency Quorum Table

| Consistency Level | Quorum Required | Failure Tolerance (RF=3) |
|---|---|---|
| `ONE` | 1 replica | Tolerates 2 node failures (fastest, stale reads possible) |
| `QUORUM` | $\lfloor RF / 2 \rfloor + 1 = 2$ replicas | Tolerates 1 node failure (balanced) |
| `LOCAL_QUORUM` | Majority in local DC only | Tolerates 1 node failure in local DC (no cross-DC WAN latency) |
| `ALL` | All 3 replicas | Tolerates 0 node failures (strongest, fragile availability) |

### 6.2 Lightweight Transactions (LWT)
Cassandra provides linearizable Compare-and-Set (CAS) semantics via a 4-phase Paxos consensus protocol (`SERIAL` consistency):

```sql
-- Unique registration without race conditions
INSERT INTO users (username, email, password_hash)
VALUES ('anandu', 'anandu@example.com', 'hash_secret')
IF NOT EXISTS;

-- Conditional balance transfer
UPDATE bank_accounts
SET balance = balance - 250
WHERE account_id = 'ACC-99'
IF balance >= 250;
```

---

## 7. Cassandra Application Development with Python (`cassandra-driver`)

The DataStax Python driver provides connection pooling, token-aware routing, and asynchronous query execution:

```python
from cassandra.cluster import Cluster, ExecutionProfile, EXEC_PROFILE_DEFAULT
from cassandra.auth import PlainTextAuthProvider
from cassandra import ConsistencyLevel
from cassandra.query import SimpleStatement, PreparedStatement
import uuid

# 1. Configure Execution Profile with LOCAL_QUORUM consistency
profile = ExecutionProfile(
    consistency_level=ConsistencyLevel.LOCAL_QUORUM,
    request_timeout=10.0
)

# 2. Initialize Cluster & Session
cluster = Cluster(['127.0.0.1'], execution_profiles={EXEC_PROFILE_DEFAULT: profile})
session = cluster.connect("ecommerce_keyspace")

# 3. Prepare Statements (Parsed and cached on nodes for optimal performance)
insert_order_stmt = session.prepare("""
    INSERT INTO customer_orders (customer_id, order_id, total_amount, status)
    VALUES (?, ?, ?, ?)
""")

# 4. Synchronous Execution
order_id = uuid.uuid4()
session.execute(insert_order_stmt, ("CUST-101", order_id, 149.99, "completed"))

# 5. Asynchronous Non-Blocking Execution with Callbacks
def handle_success(rows):
    for row in rows:
        print(f"Order: {row.order_id}, Amount: {row.total_amount}")

def handle_error(exception):
    print(f"Query failed: {exception}")

future = session.execute_async("SELECT order_id, total_amount FROM customer_orders WHERE customer_id = 'CUST-101'")
future.add_callbacks(handle_success, handle_error)

# Clean up connections on app shutdown
# cluster.shutdown()
```

---

## Practice Problems & Solutions

1. **Partition Key Design:** A streaming platform stores billions of video viewing events. The query pattern is: *"Get all viewing events for user X during date range Y, sorted newest first."* Design the primary key.
   * *Solution:*
     ```sql
     CREATE TABLE user_viewing_history (
       user_id UUID,
       view_date date,
       event_timestamp timestamp,
       video_id UUID,
       watch_duration_seconds int,
       PRIMARY KEY ((user_id, view_date), event_timestamp)
     ) WITH CLUSTERING ORDER BY (event_timestamp DESC);
     ```
     *Rationale:* `(user_id, view_date)` forms a composite partition key to prevent unbounded partition growth over multiple years, and `event_timestamp` acts as the clustering key to sort records on disk in descending order.
2. **Compaction Strategy Selection:** A smart-meter telemetry application receives 500,000 temperature readings per second, with data configured to expire after 30 days via TTL. Which compaction strategy should be chosen?
   * *Solution:* **Time-Window Compaction Strategy (TWCS)**. It groups SSTables by timestamp windows and drops entire SSTables in a single disk operation once the window passes expiry, eliminating compaction overhead.

---

## Unit IV Summary Cheat-Sheet

| Concept | Key Takeaway |
|---|---|
| **LSM-Tree Storage** | Sequential commit log + in-memory memtable + immutable sorted SSTables. |
| **Bloom Filters** | Probabilistic bit-arrays that eliminate disk seeks for non-existent partition keys. |
| **Chebotko Rule** | Design tables around specific query patterns; filter by partition key for single-node lookup. |
| **Partition Sizing** | Keep partition disk footprint $< 100\text{ MB}$ and cell counts $< 100,000$. |
| **LWT (Paxos)** | Conditional mutations (`IF NOT EXISTS`, `IF col = val`) with linearizable `SERIAL` consistency. |
| **Python Driver** | Use `PreparedStatement` to cache query syntax and `LOCAL_QUORUM` for DC-isolated consistency. |
