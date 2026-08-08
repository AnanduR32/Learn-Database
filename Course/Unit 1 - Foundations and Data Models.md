# Unit I — Relational Foundations, NoSQL Motivations & Data Models
### 25CSA642A — NoSQL Databases

This unit builds the conceptual bridge from relational databases to NoSQL: it reviews why relational systems normalize data, shows *why* that design hits a scaling ceiling, formalizes the CAP theorem and the ACID→BASE relaxation that NoSQL trades for scale, then works through the four NoSQL data-model families (key-value, document, column-family, aggregate-oriented) before landing on MongoDB, Cassandra, and a systematic comparison against relational stores, HBase, and Neo4j.

**Roadmap of this file:**
1. Relational Foundations Review — Queries, Constraints, Normalization, Functional Dependency, Indexing
2. Introduction to NoSQL — Motivations and the Scaling Problem
3. CAP Theorem
4. ACID vs BASE
5. Types of NoSQL Databases — Key-Value and Document Data Models
6. Column-Family (Wide-Column) Stores
7. Aggregate-Oriented Databases
8. NoSQL Databases using MongoDB — Data Model and Queries
9. Column-Oriented NoSQL Databases using Apache Cassandra
10. Comparison of Relational Databases to NoSQL Stores — MongoDB, Cassandra, HBase, Neo4j

---

## 1. Relational Foundations Review — Queries, Constraints, Normalization, Functional Dependency, Indexing

### 1.1 Theory

**The relational model:**
- Data is represented as **relations** (tables): sets of tuples over named attributes.
- Queried declaratively via SQL, which compiles to **relational algebra** operators:
  - selection $\sigma$ — filter rows
  - projection $\pi$ — choose columns
  - join $\bowtie$ — combine tables on a predicate
  - union, difference
- **Constraints** enforce integrity:
  - a **primary key** guarantees entity integrity (unique, non-null identifying attribute(s))
  - a **foreign key** guarantees referential integrity (a value must exist as a primary key elsewhere)
  - `CHECK`/`UNIQUE` enforce domain-specific rules

**Functional dependencies and normalization:**
- A **functional dependency (FD)** $X \to Y$ holds if every pair of tuples agreeing on attributes $X$ also agrees on $Y$ — $X$ *functionally determines* $Y$.
- FDs are the formal basis of **normalization**:
  - **1NF** — atomic attribute values, no repeating groups
  - **2NF** — no non-key attribute depends on only *part* of a composite key
  - **3NF** — no non-key attribute depends *transitively* on the key via another non-key attribute
  - **BCNF** — every determinant of a non-trivial FD is a candidate key (the strictest practical form)

**Indexing:**
- Accelerates lookups. A **B-tree** index organizes keys into a balanced, sorted, multi-way tree.
- A lookup touches only $O(\log_b n)$ nodes instead of scanning all $n$ rows.

### 1.2 Proof — Armstrong's Axioms are sound, and BCNF decomposition is lossless

**Soundness of Armstrong's axioms.** The three inference rules for FDs are:
- *Reflexivity*: $Y\subseteq X \Rightarrow X\to Y$
- *Augmentation*: $X\to Y \Rightarrow XZ\to YZ$
- *Transitivity*: $X\to Y, Y\to Z \Rightarrow X\to Z$

Each follows directly from the definition of FD:
- **Reflexivity** — if $Y\subseteq X$, any two tuples agreeing on all of $X$ automatically agree on the subset $Y$.
- **Augmentation** — if two tuples agree on $XZ$, they agree on $X$, hence (by $X\to Y$) on $Y$; they already agree on $Z$, so they agree on $YZ$.
- **Transitivity** — if two tuples agree on $X$, then (by $X\to Y$) they agree on $Y$, then (by $Y\to Z$) they agree on $Z$.

Since every rule is a direct consequence of the FD definition applied to arbitrary tuple pairs, the axioms only ever derive *true* dependencies — they are **sound** (and, though not shown here, also complete: every true FD is derivable from them).

**Lossless-join decomposition.**
- Decomposing $R$ into $R_1, R_2$ is lossless iff $R_1\cap R_2 \to R_1$ or $R_1 \cap R_2 \to R_2$.
- **Why:** naturally joining $R_1 \bowtie R_2$ always reconstructs *at least* the original tuples (any tuple of $R$ projects into matching rows of $R_1,R_2$ that rejoin).
- **The risk:** *spurious* extra tuples from unrelated combinations sharing common $R_1\cap R_2$ values.
- **The guarantee:** if $R_1\cap R_2 \to R_1$, each value of the shared attributes corresponds to *at most one* $R_1$-tuple, so joining it against any matching $R_2$-tuple cannot manufacture a new combination beyond what existed in $R$ — the join exactly reproduces $R$, no more, no less.
- This theorem is what guarantees normalization never silently loses information.

### 1.3 Worked Example

Table `Orders(OrderID, CustomerID, CustomerName, ProductID, ProductName, Qty)` with FDs: `OrderID,ProductID → Qty`; `CustomerID → CustomerName`; `ProductID → ProductName`; `OrderID → CustomerID`. This violates 3NF/BCNF: `CustomerName` depends transitively on `OrderID` via `CustomerID` (a non-key attribute). Decompose into `Orders(OrderID, CustomerID)`, `Customers(CustomerID, CustomerName)`, `OrderItems(OrderID, ProductID, Qty)`, `Products(ProductID, ProductName)`. Each decomposition point shares a determinant (`CustomerID → Customers`, `ProductID → Products`) with the split, satisfying §1.2's lossless-join condition.

### 1.4 Nuances

- Normalization eliminates update/insert/delete anomalies but *increases* the number of joins needed to reconstruct a full record — exactly the cost NoSQL's aggregate model (§5, §7) is designed to avoid for read-heavy, single-entity access patterns.
- Achieving both lossless-join **and** full dependency-preservation simultaneously is not always possible in BCNF; 3NF is the standard practical compromise.
- Every index write-amplifies: each insert/update must maintain every index on the table — the same trade-off resurfaces for NoSQL secondary indexes (Unit III §5).

### 1.5 Real-World Application

Relational schemas remain the standard for transactional systems — banking ledgers, inventory, ERP — where non-redundancy and multi-table consistency outweigh raw write throughput. Query planners exploit indexes to answer point/range lookups over millions of rows in milliseconds, which is the baseline every NoSQL indexing strategy (Unit III) is measured against.

---

## 2. Introduction to NoSQL — Motivations and the Scaling Problem

### 2.1 Theory

- "NoSQL" ("Not Only SQL") denotes databases that relax one or more relational guarantees — fixed schema, native joins, or immediate global consistency — in exchange for horizontal scalability, schema flexibility, and cost-effective handling of high-volume, high-velocity, semi-structured data.
- Two scaling strategies exist:
  - **Vertical scaling** — bigger machine; bounded by hardware ceilings and non-linear cost growth.
  - **Horizontal scaling** — more machines; theoretically unbounded, but requires partitioning data and coordinating across nodes.

### 2.2 Proof — Amdahl's Law bounds vertical/parallel scaling

**Setup:** if a fraction $p$ of a workload is parallelizable and $(1-p)$ is inherently serial, the speedup from $N$ processors is:
$$S(N) = \dfrac{1}{(1-p) + p/N}$$

**Proof:**
- On one processor the total time is $T = (1-p)T + pT$.
- On $N$ processors, the serial part is unchanged but the parallel part divides evenly: time becomes $(1-p)T + pT/N$.
- Speedup is the ratio of times, giving $S(N)$ above.
- As $N\to\infty$, $S(N) \to \dfrac{1}{1-p}$ — a **hard ceiling** independent of how much hardware is added.

**Why this matters:**
- This proves why a single monolithic database server (or naive multi-core parallelism within one machine) plateaus: if 10% of the critical path is serial (one global lock, one write-ahead log), speedup can never exceed $10\times$.
- NoSQL's answer is to eliminate the serial bottleneck itself by **partitioning data** (sharding) across independent nodes, each with its own lock and log.
- $N$ independent shards approach *linear* scaling for shard-local operations, sidestepping Amdahl's ceiling entirely rather than fighting it.

### 2.3 Worked Example

With $p=0.9$ (90% parallelizable, typical of a single server's query engine): at $N=10$, $S=\frac{1}{0.1+0.09}=5.26\times$; at $N=100$, $S=\frac{1}{0.1+0.009}=9.17\times$; as $N\to\infty$, $S\to10\times$ — diminishing returns. Contrast 10 independently-sharded NoSQL nodes with disjoint key ranges and no shared serial bottleneck: they achieve close to $10\times$ throughput, and an 11th node keeps adding capacity, unlike the flattening curve above.

### 2.4 Nuances

- NoSQL is not "schema-less" in an absolute sense — schema is enforced by the application at read time ("schema-on-read") rather than by the database at write time ("schema-on-write", §1).
- NoSQL sacrifices *something specific* — usually cross-entity joins, multi-record ACID transactions, or immediate global consistency (§3–4) — never "consistency and correctness" wholesale.
- "NoSQL" spans four distinct data-model families (§5–7), each optimized for different access patterns; there is no single NoSQL model.

### 2.5 Real-World Application

Amazon's Dynamo paper (2007) and Google's Bigtable paper (2006) are the two works most directly responsible for the NoSQL movement — both were built because relational engines could not economically scale to Amazon's shopping-cart write volume or Google's web-index storage volume.

---

## 3. CAP Theorem

### 3.1 Theory

The CAP theorem (Brewer; formalized by Gilbert & Lynch, 2002) states that a distributed data store cannot simultaneously guarantee all three of:
- **Consistency (C)** — every read returns the most recent write or an error (linearizability).
- **Availability (A)** — every request receives a non-error response, though not necessarily the latest data.
- **Partition tolerance (P)** — the system keeps operating despite arbitrary message loss/delay between nodes.

Since network partitions are unavoidable in any real distributed system, P is effectively mandatory, making the practical choice **C vs. A during a partition**.

### 3.2 Proof — Gilbert–Lynch impossibility (proof sketch)

**Setup:** two replica nodes $G_1, G_2$ store value $v$, connected by a network capable of partitioning. Suppose, for contradiction, a system guarantees both C and A even under partition.

**Argument:**
- A client writes $v_1$ to $G_1$; the network partitions before $v_1$ replicates to $G_2$.
- A second client then reads from $G_2$.
- By **Availability**, $G_2$ must respond (no error/timeout).
- By **Consistency**, that response must reflect $v_1$.
- But $G_2$ has received no message from $G_1$ (the partition blocks all communication), so it has no way to know $v_1$ occurred — it can only return stale data or guess, violating Consistency.

**Conclusion:** no algorithm can satisfy both C and A the instant a partition occurs; this is a **provable impossibility**, not an engineering shortfall, given that P is required.

### 3.3 Worked Example

Two data-center replicas of a shopping cart experience a mid-checkout partition. **AP** choice (Riak/Dynamo-style): both centers keep accepting cart updates independently; after the partition heals, conflicting versions are reconciled (vector clocks, Unit II §2). **CP** choice (HBase-style): the data center unable to reach a majority region simply refuses reads/writes until the partition heals, guaranteeing no stale cart is ever shown.

### 3.4 Nuances

- CAP is a **worst-case, partition-only** statement — absent a partition, a system can and usually does provide both C and A.
- The **PACELC** extension (Abadi) refines this: *if Partitioned*, choose A or C; *Else* (normal operation), choose Latency or Consistency — capturing the latency/consistency trade-off that persists even without partitions (e.g., synchronous vs. asynchronous replication).
- Choosing AP does not mean *no* consistency — it typically means **eventual consistency** (§4), a weaker but well-defined guarantee, not an absence of one.

### 3.5 Real-World Application

DynamoDB and Riak are classic **AP** systems (a shopping cart must always accept "add to cart," never error). Bigtable/HBase are **CP** (financial-ledger-adjacent workloads must never show two conflicting balances). Knowing which side of CAP a store occupies is the first question when selecting a NoSQL database for a given workload.

---

## 4. ACID vs BASE

### 4.1 Theory

- **ACID** (traditional RDBMS transactions): Atomicity, Consistency, Isolation, Durability — a transaction is all-or-nothing, preserves invariants, is isolated from concurrent transactions, and survives crashes.
- **BASE** (typical of AP NoSQL stores): **B**asically **A**vailable, **S**oft state, **E**ventual consistency — the system stays available for reads/writes almost always; internal state may be provisional; and, absent new writes, all replicas eventually converge.
- Consistency in a replicated store is made **tunable** via quorum parameters:
  - $N$ — replica count
  - $W$ — write acknowledgements required
  - $R$ — read acknowledgements required

### 4.2 Proof — quorum intersection ($R+W>N$) guarantees strong consistency

**Claim:** if $R+W>N$, every read quorum intersects every write quorum, so at least one queried replica holds the latest committed write.

**Proof:**
- A write quorum is any subset of size $W$ from $N$ replicas; a read quorum is any subset of size $R$.
- Two subsets of sizes $W$ and $R$ from a universe of size $N$ must intersect whenever $W+R>N$ — if disjoint, their union would have size $W+R \le N$, contradicting $W+R>N$.
- By this pigeonhole argument, at least one replica belongs to both the write quorum that accepted the latest write and any read quorum, and (assuming version/timestamp comparison on read) that replica's value is returned.
- Hence $R+W>N$ is **sufficient for read-your-writes consistency** on quorum-based stores (Riak, Cassandra, Dynamo).

**Converse:** if $R+W\le N$, disjoint quorums are possible, and a read can miss every replica holding the newest write — exactly how "eventually consistent" (BASE) configurations arise (e.g., $R=W=1, N=3$: fast, but no freshness guarantee).

### 4.3 Worked Example

$N=3$ replicas. Strong-consistency configuration: $W=2, R=2$ ($W+R=4>3$) — every read is guaranteed fresh and tolerates one node failure. Eventually-consistent configuration: $W=1, R=1$ ($W+R=2\le3$) — writes/reads are fast (one acknowledgement needed) but a read can return a stale replica until anti-entropy/read-repair propagates the write to the remaining two.

### 4.4 Nuances

- ACID and BASE are **endpoints of a spectrum**, not a binary choice — many NoSQL stores let $N, R, W$ be dialed per-operation (Cassandra, Riak) rather than fixed globally.
- "Eventual" consistency carries no time bound in the classical definition; real systems add read-repair, hinted handoff, and Merkle-tree anti-entropy to make convergence fast in practice.
- Session-level guarantees (read-your-own-writes, monotonic reads) are commonly layered atop plain eventual consistency (e.g., sticky sessions to one replica) to keep BASE systems usable interactively.

### 4.5 Real-World Application

Banking core systems retain ACID (a transfer must never partially apply). Social-media "like" counters, view counts, and shopping carts use BASE — momentary staleness is an acceptable trade for always-on write availability at massive scale.

---

## 5. Types of NoSQL Databases — Key-Value and Document Data Models

### 5.1 Theory

- The **key-value model** is a distributed hash map — `get(key)`/`put(key, value)` — where the value is an opaque blob the database cannot query internally without secondary indexing; it is the simplest, most horizontally scalable model.
- The **document model** stores semi-structured documents (JSON/BSON/XML) whose internal fields the database *can* query, index, and partially update.
- A document is an **aggregate**: a self-contained unit bundling what a relational schema would spread across several joined tables.

### 5.2 Proof — the Aggregate Atomicity argument

**Claim:** if all data needed for one business operation lives inside a single aggregate, that operation can be made atomic *without* a distributed transaction protocol, because it never crosses a shard boundary.

**Proof sketch:**
- A shard function $h(\text{key})$ maps each aggregate to exactly one physical node (consistent hashing, §9).
- Any write touching only fields *within* one aggregate is, by construction, confined to the single node owning $h(\text{key})$.
- That node applies the write under a local lock/write-ahead log exactly as a single-node database would, yielding atomicity and isolation for free.
- The moment an operation must touch two aggregates hashing to two different nodes, atomicity requires a distributed protocol (two-phase commit, sagas) with the latency/availability costs implied by CAP (§3).

This is the formal justification behind "design aggregate boundaries around transaction boundaries."

### 5.3 Worked Example

Key-value: `put("cart:1042", "<opaque blob>")` — the store cannot answer "which carts contain product X" without scanning every value. Document (MongoDB-style):
```json
{ "_id": 1042, "customer": "Alice",
  "items": [ {"sku": "A1", "qty": 2, "price": 9.99},
             {"sku": "B7", "qty": 1, "price": 24.5} ] }
```
A single `updateOne({_id:1042}, {$push:{items: ...}})` is atomic — it never touches another customer's cart, so no cross-node coordination is required (§5.2 applies directly).

### 5.4 Nuances

- Aggregate design is a **modeling decision with lasting consequences**: over-large aggregates cause write contention and bloated payloads; over-fragmented aggregates reintroduce the multi-entity-transaction problem the model exists to avoid.
- Document databases do support secondary indexes on nested fields (Unit III §5) — the "opaque blob" limitation is specific to pure key-value stores.
- Aggregate design deliberately *reintroduces* redundancy (embedding) for atomicity and read-locality — the opposite of relational normalization's goal of eliminating redundancy (§1).

### 5.5 Real-World Application

Shopping carts, user sessions, and self-describing product catalogs are textbook aggregate use cases — one read/write retrieves everything a page needs in a single round trip, unlike a relational schema requiring several joined tables.

---

## 6. Column-Family (Wide-Column) Stores

### 6.1 Theory

- A **column-family** store (Bigtable/Cassandra/HBase lineage) organizes data as a sparse, distributed, multi-dimensional sorted map, keyed by `(row key, column family, column qualifier, timestamp) → value`.
- Unlike a relational table, rows need not share the same columns, and columns are grouped into families stored contiguously on disk — optimized for **sparse data** and for reading only the columns needed.

### 6.2 Proof — storage efficiency of sparsity

**Claim:** for a table with $R$ rows and $C$ possible columns where each row populates only $k \ll C$ columns, a dense relational representation needs $O(R\times C)$ storage (every cell, including NULLs, is physically accounted for), while a column-family's sparse map needs only $O(R\times k)$.

**Proof:**
- The wide-column model stores a triple $(\text{row}, \text{column}, \text{value})$ only for cells that exist.
- An absent column consumes zero bytes because the key `(row, column)` simply never appears, whereas a fixed-width relational row must reserve space (or a NULL bitmap) for every declared column regardless of population.
- As $k/C \to 0$ (e.g., a web-crawl table with millions of possible "outbound-link" columns but a few dozen populated per page — Bigtable's original use case), the storage-saving factor $C/k$ becomes arbitrarily large.

### 6.3 Worked Example

Cassandra table `sensor_data` (partition key `sensor_id`, clustering column `reading_time`): the row for `sensor_id=42` might have columns `{temp, humidity}` at $t_1$ and `{temp, pressure}` at $t_2$ — no schema violation, no wasted storage for `pressure` at $t_1$ or `humidity` at $t_2$.

### 6.4 Nuances

- Column-family design pushes query patterns into the schema itself — tables are typically designed "query-first" (one denormalized table per query pattern), the opposite of relational "normalize first, query later" (§1).
- Timestamps are first-class (Bigtable/HBase versioning), enabling built-in temporal queries ("value as of time $T$") without extra application logic.
- Unbounded clustering keys (e.g., ever-appending event logs) can grow a row without limit — a known anti-pattern requiring bucketing strategies.

### 6.5 Real-World Application

Time-series/IoT telemetry, and Google's original use case — a sparse web-link graph spanning billions of URLs where any two pages share almost no common outbound-link columns — are canonical wide-column workloads.

---

## 7. Aggregate-Oriented Databases

### 7.1 Theory

- "Aggregate-oriented" (Sadalage & Fowler) is the unifying term for key-value, document, and column-family databases.
- All three treat some **unit of data larger than a single value** as the atomic boundary for storage, replication, and sharding — unlike the relational model's atomic *row* plus cross-table joins.

### 7.2 Proof — aggregates as the natural sharding unit

**Claim:** if the shard function operates on aggregate keys, an operation confined to one aggregate contacts exactly one shard, while an operation spanning aggregates contacts $\ge 2$.

**Proof:**
- By definition, sharding assigns each aggregate wholly to one node via $h(\text{key})\to\text{node}$.
- An operation's node fan-out equals the number of distinct aggregate keys it touches — 1 for a single-aggregate operation, $\geq2$ otherwise, directly from the shard map's definition.
- Since cross-shard operations require coordination protocols with strictly worse latency/availability characteristics than single-shard operations (§3–4), **minimizing cross-aggregate operations is provably equivalent to minimizing coordination overhead**.

This is the formal reason aggregate-oriented modeling effort focuses on matching aggregate boundaries to query/transaction patterns (§5.2).

### 7.3 Worked Example

An e-commerce system with aggregate-per-order (order + line items + shipping address embedded): "get order #500" is a single-shard read; "sum revenue across all orders" is a cross-shard scatter-gather — cheap operations are exactly those the aggregate boundary was designed around.

### 7.4 Nuances

- Aggregate orientation is *why* NoSQL databases generally lack general-purpose server-side joins — a join is, by definition, a cross-aggregate operation.
- The "right" aggregate boundary is workload-dependent: the same entities (customer, order) might form one aggregate in an order-management app and separate aggregates in an analytics-heavy app.
- Graph databases (Unit II) are the explicit exception to aggregate orientation — they optimize for *traversing* relationships between fine-grained entities rather than avoiding them.

### 7.5 Real-World Application

Every major cloud-native NoSQL offering (DynamoDB, Cosmos DB, MongoDB Atlas, managed Cassandra) bills and partitions by aggregate/partition key — cost and performance planning for these services is inseparable from correct aggregate design.

---

## 8. NoSQL Databases using MongoDB — Data Model and Queries

### 8.1 Theory

- MongoDB is a **document** database storing BSON (binary JSON) documents inside schema-flexible **collections** (loosely analogous to tables).
- Queries are BSON filter documents evaluated against a collection, e.g. `db.orders.find({status: "shipped"})` — conceptually a relational-algebra selection $\sigma$ restricted to a single collection, with no native cross-collection join (mirroring §7.4).

### 8.2 Proof — index lookup complexity

- MongoDB's default index type is a B-tree; a lookup on an indexed field over $n$ documents costs $O(\log_b n)$ comparisons ($b$ = branching factor), versus $O(n)$ for a full collection scan (`COLLSCAN`).
- This follows the standard B-tree height bound: a B-tree of branching factor $b$ storing $n$ keys has worst-case height $h=\lceil\log_b n\rceil$, since each level holds at most $O(b^{\text{level}})$ keys, giving $n \le O(b^h) \Rightarrow h \ge \log_b n$.
- Each level costs one page access, bounding total lookup cost by $O(\log_b n)$ (developed fully in Unit III §5).

### 8.3 Worked Example

```
db.orders.find({ "items.sku": "A1", status: "shipped" })
```
returns every order document with a line item `sku="A1"` and `status="shipped"` — a single-collection predicate scan (index-assisted if `items.sku` is indexed), with no join needed because line items are embedded (§5).

### 8.4 Nuances

- MongoDB's `$lookup` aggregation stage provides a left-outer-join-like operation for the rare cross-collection case, but it is comparatively expensive versus embedding — reinforcing §7's aggregate-boundary guidance.
- Schema flexibility means a document missing a field is not rejected at write time by default; validation must be declared explicitly (JSON Schema validators) or enforced by the application (schema-on-read, §2.4).
- Full query-language depth (operators, indexing, geospatial queries, upserts) with hands-on Python examples is developed in Unit III.

### 8.5 Real-World Application

Content-management systems, product catalogs, and mobile-app backends favor MongoDB's document model because data naturally arrives as nested JSON from client applications, avoiding an object-relational mapping translation layer.

---

## 9. Column-Oriented NoSQL Databases using Apache Cassandra

### 9.1 Theory

- Cassandra combines Bigtable's wide-column data model (§6) with Dynamo's decentralized, masterless, consistent-hashing architecture (no single point of failure).
- Every node owns a range of the hash ring (via a **partitioner**, typically Murmur3), and rows are placed by hashing the partition key.

### 9.2 Proof — consistent hashing's load-balance guarantee

**Claim:** with consistent hashing over a ring of $N$ (pseudo-)uniformly placed nodes, adding or removing one node remaps only an expected $O(K/N)$ of $K$ keys, versus $O(K)$ for naive modulo hashing $h(\text{key}) \bmod N$.

**Proof sketch:**
- Consistent hashing assigns each key to the first node clockwise from its hash position.
- Removing node $X$ affects only keys whose hash falls in the arc between $X$'s predecessor and $X$ — by uniform placement, this arc holds an expected fraction $1/N$ of the ring, i.e., $O(K/N)$ keys, all moving to exactly one neighboring node.
- Contrast modulo hashing: changing $N\to N-1$ changes $\text{key}\bmod N$ for nearly every key, forcing an $O(K)$ full reshuffle.

This is the structural reason (with **virtual nodes** smoothing load further) Cassandra clusters can resize online without a full data migration.

### 9.3 Worked Example

Ring positions (mod $360°$) for 4 nodes at $0°,90°,180°,270°$. A key hashing to $100°$ maps to the node at $180°$ (first clockwise). Removing the node at $180°$ remaps only keys in $(90°,180°]$ (one quarter of the ring, $K/4$ keys) — to the node at $270°$ — while all other keys are untouched.

### 9.4 Nuances

- Replication factor $RF$ places $RF-1$ additional replicas at the next $RF-1$ distinct clockwise nodes; combined with tunable per-query consistency levels (ONE, QUORUM, ALL), this directly applies the $N,R,W$ quorum math of §4.
- CQL is SQL-like syntactically but model-restricted: queries must specify (or derive) the partition key — arbitrary `WHERE` clauses on non-indexed, non-partition columns are disallowed by default, since they would require a full-cluster scatter-gather.
- The gossip protocol (peer-to-peer state dissemination) removes the coordinator single point of failure that master-based systems (HBase) retain, at the cost of eventual (not immediate) cluster-membership consistency.

### 9.5 Real-World Application

Cassandra powers write-heavy, globally-distributed workloads — Netflix's telemetry pipeline and large-scale messaging backends rely on its masterless, linearly-scalable write path.

---

## 10. Comparison of Relational Databases to NoSQL Stores — MongoDB, Cassandra, HBase, Neo4j

### 10.1 Theory

| Dimension | RDBMS | MongoDB | Cassandra | HBase | Neo4j |
|---|---|---|---|---|---|
| Data model | Tables/rows | Documents | Wide-column | Wide-column | Property graph |
| Schema | Fixed (schema-on-write) | Flexible | Flexible per-partition | Flexible | Flexible (labeled nodes/edges) |
| Joins | Native relational algebra | `$lookup` (limited) | None (denormalize) | None | Native traversal |
| CAP leaning | CP (single-node CA) | CP-leaning, tunable | AP-leaning, tunable | CP | CP (causal clustering) |
| Query language | SQL | MQL (BSON queries) | CQL | Java API / scans | Cypher |

### 10.2 Proof — the query-expressiveness trade-off

- Relational algebra ($\sigma,\pi,\bowtie,\cup,-,\times$) is **relationally complete**: any first-order-logic query over the schema is composable from these operators (Codd's theorem).
- Document/column-family query languages deliberately omit general $\bowtie$, because unrestricted joins require unbounded cross-shard coordination (§7.2).
- Graph query languages (Cypher) specialize the opposite way: they make bounded-depth **traversal** along explicit relationship edges a first-class, optimized primitive, because a property graph physically stores relationships as direct pointers — traversing a stored edge is $O(1)$ (index-free adjacency), versus a relational join's $O(n\log n)$ or worse over unindexed foreign keys.

### 10.3 Worked Example

"Friends of friends": relationally, a self-join `Friends ⋈ Friends` costs $O(n\log n)$ (or better with an index per hop, compounding with each additional hop). In Neo4j, `MATCH (a:Person)-[:FRIEND]->()-[:FRIEND]->(b) RETURN b` walks stored relationship pointers directly from `a`, costing time proportional only to `a`'s actual connection count at each hop, independent of total graph size.

### 10.4 Nuances

- No system dominates — the comparison table encodes deliberate trade-offs; selection is a function of dominant access pattern (point lookups → key-value/document; wide scans/time-series → column-family; multi-hop relationships → graph; ad hoc joins/strong global consistency → relational).
- Polyglot persistence — using multiple database types within one system, each for its strength — is standard modern practice, not an edge case.
- HBase's dependence on an external coordination service (ZooKeeper) and a single active master per region shows "NoSQL" does not imply "no coordination point" — HBase is deliberately CP, trading availability for HDFS-grade consistency.

### 10.5 Real-World Application

A modern e-commerce platform might use PostgreSQL for order/payment ledgers (ACID), MongoDB for the product catalog (flexible schema), Cassandra for clickstream/session logs (write-heavy, AP), and Neo4j for "customers who bought this also bought" recommendations (graph traversal) — one company, four data models, each matched to its query pattern.

---

## Practice Problems (Exam-Style)

**P1 — CAP scenario.** A shopping-cart service is deployed across two datacenters and the link between them drops. Should it prioritize C or A? Justify.

*Solution:* **Choose AP.** A cart must always accept adds/removes (§4's BASE argument) — rejecting a write because the other datacenter is unreachable would lose the sale. Reconcile the two divergent cart copies later (vector clocks / CRDT merge, Unit II §1), rather than blocking writes during the partition.

**P2 — Quorum design.** With $N=5$ replicas, pick $(R,W)$ for strong read-your-writes consistency, and show why $(R,W)=(1,1)$ fails.

*Solution:* Need $R+W>N=5$. **$(R,W)=(3,3)$** works: $3+3=6>5$, guaranteeing every read quorum overlaps every write quorum by at least one replica. $(1,1)$ gives $1+1=2\le5$ — a read can hit a replica that never saw the latest write, returning stale data (§4's quorum math).

**P3 — Normalization / functional dependency.** Table `Orders(OrderID, CustomerName, CustomerAddress, ProductID, ProductName, Price)` has FDs `OrderID→CustomerName, CustomerAddress` and `ProductID→ProductName, Price`. Decompose to 3NF.

*Solution:* `CustomerAddress` depends on `CustomerName`, not the whole key `OrderID` — a partial/transitive dependency. Decompose into: `Customers(CustomerID, CustomerName, CustomerAddress)`, `Products(ProductID, ProductName, Price)`, `Orders(OrderID, CustomerID, ProductID)` — each non-key attribute now depends only on its own table's full key (§1's normalization argument).

**P4 — Embed vs. reference.** A blog post can accumulate 10,000+ comments. Should comments be embedded in the post document?

*Solution:* **Reference, don't embed.** MongoDB documents cap at 16MB (§5's aggregate-boundary theory); an unbounded, ever-growing comment array risks hitting that limit and makes every post fetch drag along all comments. Store comments in a separate collection referencing `postID`, paginated on read — trading single-query embedding convenience for bounded document size.

**P5 — Cassandra table design.** Design a table for sensor readings queried by "last 24 hours for device X."

*Solution:* `PRIMARY KEY ((device_id), timestamp)`. `device_id` as the **partition key** spreads writes evenly across the ring (§9's consistent-hashing); `timestamp` as the **clustering key** sorts each partition so a range query `WHERE device_id=X AND timestamp > ?` is a fast sequential scan within one partition — no cross-partition scatter-gather.

**P6 — Database selection.** A financial ledger needs multi-row ACID transactions but has modest write volume. Which model?

*Solution:* **Relational** (or a document store with explicit multi-document transaction support). Per §3–4, ledger consistency requirements (no partial transfers) outweigh the horizontal-scale benefits AP stores offer — the workload doesn't need the throughput that would justify trading away strong consistency.

---

## Unit I Summary

| Concept | One-line takeaway |
|---|---|
| Relational review | Normalization removes redundancy at the cost of joins |
| NoSQL motivations | Horizontal scale-out escapes Amdahl's-law-bounded vertical scaling |
| CAP theorem | Partition tolerance is mandatory; C and A cannot both hold during a partition |
| ACID vs BASE | Quorum math ($R+W>N$) tunes the consistency/availability trade-off |
| Key-value / document | Aggregates make single-document atomicity free; cross-aggregate ops need coordination |
| Column-family | Sparse storage: pay only for populated cells |
| Aggregate-oriented | Shard-by-aggregate minimizes cross-node coordination |
| MongoDB | Document model with B-tree-indexed, join-light queries |
| Cassandra | Consistent hashing gives $O(K/N)$ rebalancing on node change |
| Relational vs NoSQL | Joins vs. index-free adjacency: relational completeness vs. traversal specialization |
