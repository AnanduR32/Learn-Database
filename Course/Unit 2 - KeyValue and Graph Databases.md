# Unit II — Key-Value Databases (Riak & Redis) & Graph Databases (Neo4j)
### 25CSA642A — NoSQL Databases

This unit explores Key-Value databases using **Riak** (focusing on consistency, vector clocks, CRDTs, virtual nodes, and transaction limits) and **Redis** (focusing on in-memory data structures, caching design patterns, eviction policies, and distributed locks) alongside Graph databases using **Neo4j** (focusing on nodes, relationships, index-free adjacency, and causal clustering).

**Roadmap of this file:**
1. Key-Value Store Architecture — Riak Consistency, Vector Clocks, CRDTs, Structure of Data
2. In-Memory Key-Value Stores — Redis Data Structures, Caching Patterns & Distributed Locks
3. Scaling and Suitable Use Cases (Session Info, User Profiles, Shopping Cart Data, Leaderboards)
4. When Not to Use a Key-Value Store — Relationships among Data & Multi-Operation Transactions
5. Graph Databases using Neo4j — Property Graph Model, Cypher Query Language, Index-Free Adjacency
6. Neo4j Architecture, Causal Clustering & Scaling Limits
7. Suitable Use Cases — Connected Data, Routing/Dispatch, Recommendation Engines & Fraud Detection

---

## 1. Key-Value Store Architecture — Riak Consistency, Vector Clocks, CRDTs

### 1.1 Core Concepts (Simplified)
- **Riak KV:** A masterless, highly available Key-Value store based on Amazon's Dynamo architecture.
- **Data Access:** Data is stored as an opaque byte blob under a specific **Key** inside a **Bucket** (logical namespace).
- **Consistency Mechanics:** Tuned using the quorum formula $R + W > N$ (Read Quorum $R$, Write Quorum $W$, Replication Factor $N$).
- **Vector Clocks:** Every object maintains a causal history map `{NodeID -> Counter}`:
  - If Clock 1 dominates Clock 2 (all counters $\ge$ and at least one $>$), Clock 1 is strictly newer (**no conflict**).
  - If neither clock dominates the other, a **sibling conflict** exists. Riak presents both versions to the client or uses CRDT merge rules to resolve them.
- **CRDTs (Conflict-free Replicated Data Types):** Data structures (Counters, Sets, Maps, Registers, Flags) built into Riak 2.0+ that mathematically resolve sibling conflicts deterministically without custom application resolution.

### 1.2 Vector Clock Logic & Proof Sketch
- **Partial Ordering:** The vector clock comparison relation $\preceq$ forms a mathematical partial order (reflexive, antisymmetric, transitive).
- **Conflict Detection:** If Node 1 and Node 2 accept concurrent updates during a network partition:
  - Node 1 increments its counter: $\{n_1: 3, n_2: 1\}$
  - Node 2 increments its counter: $\{n_1: 2, n_2: 2\}$
- Comparing $\{3,1\}$ and $\{2,2\}$ reveals that neither vector dominates the other ($3 > 2$ for $n_1$, but $1 < 2$ for $n_2$). This signals a **concurrent write conflict** requiring sibling generation or CRDT merge.

### 1.3 Worked Example
1. Object created on Node 1: Clock = `{n1: 1}`.
2. Node 1 updates object: Clock = `{n1: 2}`.
3. Network partition isolates Node 1 and Node 2:
   - Node 1 accepts write: `{n1: 3, n2: 0}`
   - Node 2 accepts write: `{n1: 2, n2: 1}`
4. When the partition heals, Riak compares `{3,0}` and `{2,1}`. Neither dominates; Riak preserves both values as **siblings** until resolved.

---

## 2. In-Memory Key-Value Stores — Redis Data Structures, Caching Patterns & Distributed Locks

### 2.1 Core Concepts (Simplified)
- **Redis (Remote Dictionary Server):** In-memory, single-threaded event loop data structure store delivering sub-millisecond read/write latencies.
- **Rich Data Structures:** Unlike Riak's opaque blobs, Redis understands internal data structures:
  - **Strings:** Binary-safe text/numbers (atomic `INCR`, `DECR`, `MSET`, `MGET`).
  - **Hashes:** Field-value maps (`HSET`, `HGET`, `HINCRBY`) — ideal for object representation without full deserialization.
  - **Lists:** Linked lists (`LPUSH`, `RPOP`, `LRANGE`) — ideal for message queues and capped timeline feeds.
  - **Sets:** Unordered unique collections (`SADD`, `SINTER`, `SUNION`, `SDIFF`) — ideal for mutual friends or tag matching.
  - **Sorted Sets (ZSets):** Elements ordered by floating-point score (`ZADD`, `ZRANGEBYSCORE`, `ZRANK`) — ideal for real-time leaderboards.
  - **Bitmaps & HyperLogLogs:** Probabilistic and bit-level analytics (unique DAU tracking in 12 KB).
  - **Streams:** Append-only message log with consumer groups (`XADD`, `XREADGROUP`).

### 2.2 Distributed Caching Design Patterns

| Pattern | Read Path | Write Path | Pros & Cons |
|---|---|---|---|
| **Cache-Aside (Lazy Loading)** | 1. App checks cache.<br>2. On miss, app reads DB and updates cache. | App writes to DB, then deletes/invalidates cache key. | Resilient to cache crashes; lazy loading prevents caching unused data. Stale reads possible if invalidation fails. |
| **Write-Through** | App reads from cache directly. | App writes to cache; cache synchronously writes to DB before acknowledging. | Cache always fresh; higher write latency. |
| **Write-Behind (Write-Back)** | App reads from cache. | App writes to cache; cache asynchronously batches updates to DB. | Extremely fast writes; risk of data loss if cache node dies before flushing. |
| **Refresh-Ahead** | App reads from cache. Cache automatically reloads hot keys before TTL expires. | App writes to DB. | Eliminates read latency on hot keys; complex access prediction. |

### 2.3 Cache Failure Scenarios & Mitigations
- **Cache Stampede / Breakdown:** A popular cached key expires, and 10,000 concurrent requests simultaneously hit the database.
  - *Mitigation:* Mutex locking or probabilistic early recomputation (XFetch algorithm).
- **Cache Avalanche:** Thousands of keys expire at the exact same second.
  - *Mitigation:* Add random jitter to TTLs: $\text{TTL} = \text{BaseTTL} + \text{rand}(0, 300\text{s})$.
- **Cache Penetration:** Requests query non-existent keys (e.g., `id = -9999`), bypassing cache and hitting DB every time.
  - *Mitigation:* Bloom filters or caching null responses with short TTLs.

### 2.4 Distributed Locking (`SET NX EX`)
```shell
# Acquire lock: NX (only if not exists), EX (auto-expire after 30s)
SET lock:order_101 "unique_worker_token" NX EX 30

# Release lock (requires Lua script to verify token before deleting to avoid deleting another worker's lock)
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

---

## 3. Scaling and Suitable Use Cases

### 3.1 Core Concepts (Simplified)
- **Virtual Nodes (vnodes):** Instead of mapping 1 physical server to 1 contiguous range on the hash ring, each physical server hosts $V$ virtual nodes scattered uniformly around the 160-bit hash ring.
- **Benefits:**
  1. Prevents hot-spotting on any single physical machine.
  2. If a physical node fails, its workload is evenly distributed across all surviving cluster nodes rather than overwhelming its immediate neighbor.

### 3.2 Proof — Load Balance Improvement via Vnodes
Let $V$ be the number of virtual nodes per physical machine. The load variance between physical servers drops by a factor of $O(1/\sqrt{V})$:
$$\text{Var}(\text{Load}) \propto \frac{1}{\sqrt{V}}$$
*Intuition:* Averaging many small random slices across the hash ring smooths out disk and memory variance far better than assigning one monolithic partition per server.

### 3.3 Ideal Use Cases
- **Session Management:** Storing user authentication tokens by Session ID (`GET session:token_123`).
- **Real-Time Leaderboards:** Sorted Sets maintaining live player rankings (`ZADD leaderboard 4500 "user_1"`).
- **Rate Limiting & Anti-Abuse:** Atomic counters with TTL (`INCR api_limit:user_id` + `EXPIRE 60`).
- **Shopping Carts:** Fast writes in Riak/Redis that never fail during traffic spikes.

---

## 4. When Not to Use a Key-Value Store — Relationships among Data & Multi-Operation Transactions

### 4.1 Core Concepts (Simplified)
Key-value stores are inefficient for queries based on relationships or non-key attributes (e.g., *"Find all users who purchased Product X and live in California"*).
- **Opaque Blob Limitation:** Because values are opaque blobs indexed solely by key, querying nested properties without knowing the key requires a full cluster scan ($\Omega(n)$ worst-case).
- **Multi-Key Atomicity:** Key-value stores guarantee atomicity on a **single key**. Native multi-key ACID transactions spanning distributed shards require expensive distributed locking (e.g., 2-Phase Locking or Redlock).

### 4.2 Proof — Relationship Search Lower Bound
Without a secondary index mapping attributes to primary keys, locating an unindexed attribute across $n$ items requires examining all $n$ items in the worst case ($\Omega(n)$ search time). Maintaining manual reverse-index keys (e.g., `"product:X:users"`) pushes consistency burdens onto the client application, introducing potential update anomalies.

---

## 5. Graph Databases using Neo4j — Data Model and Cypher Query Features

### 5.1 Core Concepts (Simplified)
- **Property Graph Model:** Consists of:
  - **Nodes:** Entities (e.g., `(:Person)`, `(:Product)`).
  - **Relationships (Edges):** Directed, typed connections linking nodes (e.g., `-[:PURCHASED]->`, `-[:FRIENDS_WITH]->`).
  - **Properties:** Key-value attributes on nodes or relationships (e.g., `{ since: 2024, rating: 5 }`).
  - **Labels:** Tags grouping nodes into operational categories for indexing.
- **Index-Free Adjacency:** Nodes store direct memory pointers (offsets) to their adjacent relationships. Traversing an edge does not perform an index lookup or relational join; it simply dereferences a memory pointer.
- **Cypher Query Language:** Declarative graph query language using visual ASCII-art patterns:
  ```cypher
  (a:Person {name: 'Alice'})-[:FRIEND]->(b:Person)
  ```

### 5.2 Proof — Constant-Time $O(1)$ Traversal per Hop
- **SQL Relational Join:** Traversing a foreign key relationship requires searching a B-tree index, costing $O(\log N)$ per join where $N$ is the total number of rows in the foreign table. A $k$-hop traversal costs $O(k \log N)$.
- **Neo4j Index-Free Adjacency:** Each node stores direct pointers to its relationships. Following a pointer to an adjacent node takes $O(1)$ constant time.
- **Key Result:** A $k$-hop traversal costs $O(\prod_{i=1}^k d_i)$ where $d_i$ is the local degree (number of connections) of the visited nodes, **completely independent of the total graph size $|V|$**.

### 5.3 Worked Example (Cypher Query)
Find friends of Alice's friends who are not already Alice's direct friends (Friend-of-a-Friend Recommendation):
```cypher
MATCH (a:Person {name: "Alice"})-[:FRIEND]->(f:Person)-[:FRIEND]->(fof:Person)
WHERE NOT (a)-[:FRIEND]->(fof) AND fof <> a
RETURN fof.name, count(*) AS mutual_friends
ORDER BY mutual_friends DESC
LIMIT 5;
```

---

## 6. Neo4j Architecture, Causal Clustering & Scaling Limits

### 6.1 Core Concepts (Simplified)
- **Native Graph Storage:** Node and relationship records are stored in dedicated fixed-size record files on disk, enabling direct mathematical offset calculation: $\text{File Offset} = \text{ID} \times \text{Record Size}$.
- **Causal Clustering:**
  - **Core Servers (Raft Consensus):** A small odd number of core instances (e.g., 3 or 5) manage writes using the Raft consensus protocol, ensuring ACID durability (CP focus).
  - **Read Replicas:** Asynchronous follower nodes that replicate graph data to scale read throughput horizontally.
- **The Graph Sharding Challenge:** Hash-partitioning graph nodes across different physical servers breaks Index-Free Adjacency. If a relationship crosses server boundaries, traversal incurs network RPC latency ($~1-5\text{ms}$) instead of in-memory pointer dereferences ($~10\text{ns}$), degrading traversal performance by $10^5\times$.

---

## 7. Suitable Use Cases — Connected Data, Routing, Recommendations & Fraud Detection

### 7.1 Core Use Cases
1. **Social Networks & Organization Charts:** Deeply nested hierarchy traversals.
2. **Fraud Detection Rings:** Identifying circular financial transactions or shared identities (e.g., 5 loan applicants sharing the same phone number and address across 3 hops).
3. **Recommendation Engines:** Collaborative filtering based on path traversal (`(User)-[:BOUGHT]->(Product)<-[:BOUGHT]-(Peer)-[:BOUGHT]->(Rec)`).
4. **Network & IT Infrastructure:** Dependency mapping and blast-radius impact analysis.
5. **Knowledge Graphs & GraphRAG:** Semantic entity networks powering AI retrieval-augmented generation.

### 7.2 Proof — BFS Shortest Path Complexity
Breadth-First Search (BFS) on an unweighted property graph with $|V|$ nodes and $|E|$ edges runs in $O(|V| + |E|)$ time because neighbor lookups are $O(1)$. In SQL, recursive CTE joins scale as $O(|E| \log |E|)$ due to index lookup overhead at each iterative hop.

---

## Practice Problems & Solutions

1. **Vector Clock Comparison:** Given $VC_A = \{n_1: 2, n_2: 0\}$ and $VC_B = \{n_1: 1, n_2: 1\}$, are these clocks in conflict?
   * *Solution:* **Yes.** $VC_A$ has a higher counter for $n_1$, but $VC_B$ has a higher counter for $n_2$. Neither dominates the other, resulting in concurrent **sibling** versions.
2. **Redis Eviction Selection:** An e-commerce app caches product catalog data with fixed TTLs and user session tokens without TTLs. Under memory pressure, we want to evict only expired/temporary product cache entries without dropping active sessions. Which eviction policy should be used?
   * *Solution:* **`volatile-lru`** or **`volatile-ttl`**. These policies restrict eviction exclusively to keys configured with TTLs, protecting permanent session keys.
3. **Traversal Performance:** Why is a 5-hop relationship query in Neo4j orders of magnitude faster than a 5-table self-join in PostgreSQL on a database of 100 million users?
   * *Solution:* Neo4j utilizes **Index-Free Adjacency** (direct pointer dereferencing in $O(1)$ time per edge), whereas PostgreSQL must execute repeated B-tree index searches scaling as $O(\log N)$ against the 100-million row index at every join stage.

---

## Unit II Summary Cheat-Sheet

| Feature / Concept | Key Takeaway |
|---|---|
| **Riak Vector Clocks** | Detect concurrent write conflicts during network partitions. |
| **CRDTs** | Data types that automatically resolve concurrent distributed mutations without application conflicts. |
| **Redis In-Memory** | Single-threaded event loop delivering sub-millisecond data structure operations. |
| **Cache-Aside Pattern** | Application manages cache lookups on miss; invalidates on write. |
| **Distributed Lock** | `SET key token NX EX seconds` with atomic Lua token check on release. |
| **Virtual Nodes (vnodes)** | Reduces cluster load variance by $O(1/\sqrt{V})$ across physical nodes. |
| **Index-Free Adjacency** | Storing direct pointers makes graph traversal speed independent of overall graph size. |
| **Neo4j Causal Clustering** | Raft core servers for ACID writes + read replicas for horizontal read scaling. |
