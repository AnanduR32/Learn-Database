# Unit II — Key-Value Databases (Riak) & Graph Databases (Neo4j)
### 25CSA642A — NoSQL Databases

This unit deepens the key-value model introduced in Unit I §5 using Riak as the exemplar — consistency mechanics, transactional limits, query features, scaling, and canonical use cases — then introduces the graph data model via Neo4j, contrasting its relationship-first design against the aggregate-oriented families of Unit I.

**Roadmap of this file:**
1. Key-Value Store Features — Consistency, Transactions, Query Features, Structure of Data
2. Scaling and Suitable Use Cases (Session Info, User Profiles, Shopping Cart Data)
3. When Not to Use a Key-Value Store — Relationships among Data
4. Multi-Operation Transactions, Query by Data, Operations by Sets
5. Graph Databases using Neo4j — Data Model and Query Features
6. Neo4j Scaling
7. Suitable Use Cases — Connected Data, Routing/Dispatch/Location-Based Services, Recommendation Engines

---

## 1. Key-Value Store Features — Consistency, Transactions, Query Features, Structure of Data

### 1.1 Theory

- Riak is a Dynamo-derived, masterless key-value store where every value is addressed by a key inside a **bucket** (namespace).
- Consistency is governed by the same tunable quorum triple $(N, R, W)$ from Unit I §4, plus a **vector clock** per object to track causal history across concurrent, conflicting writes.
- A vector clock is a map `{node → counter}`; comparing two clocks $VC_1, VC_2$ classifies them as:
  - $VC_1$ **dominates** $VC_2$ (every entry $\ge$, at least one $>$) — no conflict, $VC_1$ is causally newer.
  - **Concurrent** — neither dominates — a genuine conflict (sibling) that the application or Riak's default "last-write-wins"/CRDT merge must resolve.
- Riak's data is fundamentally unstructured (opaque byte blobs), matching Unit I §5's key-value definition — richer query features are optional add-ons (Solr-based search, secondary indexes), not core to the model.

### 1.2 Proof — vector-clock dominance is a valid partial order and correctly detects concurrency

**Claim:** the dominance relation $VC_1 \preceq VC_2 \iff \forall i, VC_1[i] \le VC_2[i]$ is a partial order, and two clocks are "concurrent" (neither dominates) **iff** they descend from a common ancestor via genuinely independent, non-communicating write paths.

**Proof of partial order** (confirmed directly from the pointwise $\le$ definition):
- *Reflexivity* — $VC_1[i]\le VC_1[i]$ trivially for all $i$.
- *Antisymmetry* — if $VC_1\preceq VC_2$ and $VC_2\preceq VC_1$ then $VC_1[i]=VC_2[i]$ for all $i$, so $VC_1=VC_2$.
- *Transitivity* — if $VC_1\preceq VC_2\preceq VC_3$ then $VC_1[i]\le VC_2[i]\le VC_3[i]$ for all $i$, so $VC_1\preceq VC_3$.

These are the three defining properties of a partial order.

**Correctness of concurrency detection:**
- Each node increments only its own counter on a local write and merges (takes the pointwise max) on receiving another clock during replication — an invariant maintained inductively from clock initialization.
- If clock $A$ causally precedes clock $B$ (every write reflected in $A$ happened-before the state captured by $B$), then $B$'s merge history must have absorbed $A$'s counters via non-decreasing updates, so $A\preceq B$ holds by the invariant.
- Conversely, if two writes happen independently — neither's node-history was merged into the other before the second write — some node's counter in $A$ exceeds the corresponding counter in $B$ while another entry does the reverse, so neither dominates: this is the formal definition of a **sibling conflict**, exactly what Unit I §3.2's CAP proof predicts must occur under AP operation during a partition.

### 1.3 Worked Example

Object with vector clock $VC_A=\{n1:2, n2:1\}$ updated concurrently on two partitioned nodes: node 1 produces $VC_1=\{n1:3, n2:1\}$ (dominates $VC_A$, standard sequential update); node 2 independently produces $VC_2=\{n1:2, n2:2\}$. Comparing $VC_1$ vs $VC_2$: neither $\{3,1\}\le\{2,2\}$ nor $\{2,2\}\le\{3,1\}$ holds entrywise — concurrent siblings — Riak returns both to the client (or auto-merges via a CRDT counter/set type) for resolution.

### 1.4 Nuances

- Vector clocks grow with the number of distinct actors that have written an object; Riak prunes old entries, trading perfect causal history for bounded metadata size.
- Riak's built-in CRDT types (counters, sets, maps) sidestep manual sibling resolution for common patterns by defining a deterministic, associative, commutative merge function — a stronger guarantee than generic last-write-wins.
- "Structure of data" in Riak is opaque by default; any structure (JSON) is a convention the application imposes, not something the core store parses (contrast MongoDB, Unit I §8).

### 1.5 Real-World Application

Riak is used where uptime during partitions is non-negotiable — e.g., ad-tech bid caches and session stores where returning slightly stale data beats returning an error.

---

## 2. Scaling and Suitable Use Cases

### 2.1 Theory

- Riak scales identically to Cassandra's consistent-hashing ring (Unit I §9.2): keys hash onto a ring of **virtual nodes (vnodes)**, each physical node owning several vnodes to smooth load distribution beyond what a single hash point per node would achieve.
- Canonical use cases exploit the model's strengths:
  - **Session storage** — short-lived, key-addressed, no cross-record query needed.
  - **User profiles/preferences** — self-contained per-user blob.
  - **Shopping cart data** — write-heavy, must always accept writes (an AP requirement per Unit I §3.5).

### 2.2 Proof — virtual nodes reduce load-imbalance variance

**Claim:** with $V$ vnodes per physical node distributed across $N$ physical nodes, the variance of load (keys assigned) per physical node decreases roughly as $O(1/V)$ relative to the single-token-per-node scheme of Unit I §9.2.

**Proof sketch:**
- With one hash token per node, each physical node's share of the ring is a single random arc-length draw from a uniform partition of the ring into $N$ pieces — the coefficient of variation of $N$ i.i.d. uniform-spacing arc lengths is $O(1)$, i.e., substantial imbalance is likely for small $N$.
- Assigning $V$ independent random tokens per physical node instead makes each node's total load the **sum of $V$ i.i.d. arc-length draws**.
- By the Central Limit Theorem, the sum of $V$ i.i.d. random variables has standard deviation scaling as $O(\sqrt{V})$ while the mean scales as $O(V)$, so the *relative* variability (coefficient of variation) shrinks as $O(1/\sqrt{V})$.

This is the quantitative justification for vnodes beyond the qualitative "smooths hot spots" claim, and it composes directly with the $O(K/N)$ rebalancing-cost proof from Unit I §9.2 (each vnode migrates independently on membership change).

### 2.3 Worked Example

A 4-node Riak cluster with 64 vnodes total (16 per physical node): losing one physical node redistributes only that node's 16 vnodes (each an independent, small unit) across the remaining 3 nodes — smaller, more evenly-sized migration units than one 25%-of-ring transfer, reducing peak load spikes on any single receiving node during rebalancing.

### 2.4 Nuances

- Vnode count is a tuning knob: too few reintroduces imbalance (§2.2's $O(1/\sqrt V)$ argument weakens); too many increases per-node bookkeeping overhead.
- Session/profile/cart use cases share a common shape: single-key access, no need for ad hoc queries across keys — exactly the workload where key-value's $O(1)$ hash lookup dominates a B-tree-indexed document store's $O(\log n)$ (Unit I §8.2).
- Shopping-cart AP behavior directly reuses the vector-clock conflict model of §1 — concurrent add-to-cart from two devices become siblings, merged (often via a CRDT set) rather than one silently overwriting the other.

### 2.5 Real-World Application

Web-application session stores (login state, shopping-cart contents) and user-preference stores at scale are Riak's textbook deployments — each is single-key-addressed and demands the store never reject a write.

---

## 3. When Not to Use a Key-Value Store — Relationships among Data

### 3.1 Theory

- A pure key-value store cannot efficiently answer queries that require traversing **relationships among data** — "find all orders for customer X" or "find friends of friends" — because the value is opaque and the only access path is by exact key.
- Any such query degenerates to a full scan or requires the application to maintain its own secondary index (e.g., a second key `"customer:X:orders" → [orderKey1, orderKey2, ...]`), pushing relational-style responsibility back onto the application layer.

### 3.2 Proof — key-value lookup cost is $\Theta(1)$ only for exact-key access; relationship queries cost $\Omega(n)$ without an auxiliary index

**Claim:** for a key-value store with $n$ total keys, answering "list all values whose (hidden, unindexed) field $f$ equals $v$" requires examining $\Omega(n)$ keys in the worst case.

**Proof:**
- The store indexes only the key itself (a hash table or similar structure mapping key → value in $O(1)$ expected time); it has no data structure correlating field $f$'s value to key identity.
- Absent such a structure, the adversary can place the single value with $f=v$ at any of the $n$ positions.
- Any algorithm that has not yet inspected a given key cannot rule out that key holding the answer, so a correct algorithm must inspect all $n$ keys in the worst case (a standard decision-tree/adversary lower-bound argument) — cost $\Omega(n)$, in contrast to $O(\log_b n)$ for a genuinely indexed field (Unit I §8.2) or $O(1)$ for a stored graph edge (Unit I §10.2).

This formally justifies "key-value stores are unsuitable when queries require relationships": the model provides no sub-linear path to relationship data unless the application manufactures one via denormalized secondary keys.

### 3.3 Worked Example

Modeling a social graph as key-value (`user:42 → {name, ...}`) with no relationship structure: "who are user 42's mutual friends with user 7?" requires either scanning every user's data (the $\Omega(n)$ bound above) or maintaining redundant adjacency-list keys (`friends:42 → [7, 13, 99]`) that must be manually kept consistent on every follow/unfollow — exactly the coordination burden a graph database (§5) removes by storing adjacency natively.

### 3.4 Nuances

- This limitation is intrinsic to the *pure* key-value model, not a Riak-specific flaw — Riak's Solr-based search add-on partially compensates by indexing document content, at the cost of re-introducing schema-aware machinery.
- The workaround (application-maintained secondary indexes) reintroduces exactly the update-anomaly risk normalization was designed to prevent (Unit I §1.4) — two keys can drift out of sync if not updated atomically.
- Recognizing this boundary is the practical skill being tested: choosing key-value for single-entity lookup workloads and graph/document/column stores for relationship-heavy or ad hoc-query workloads.

### 3.5 Real-World Application

Systems that started as key-value stores for session data but grew "find related records" requirements (e.g., fraud detection needing "accounts sharing a device fingerprint") are the classic trigger for migrating that sub-workload to a graph database (§5) rather than forcing joins onto a key-value store.

---

## 4. Multi-Operation Transactions, Query by Data, Operations by Sets

### 4.1 Theory

- **Multi-operation transactions** across multiple keys are, by Unit I §5.2/§7.2's aggregate-atomicity argument, not natively supported by key-value stores — each `put`/`get` is atomic only for its own key.
- **Query by data** (as opposed to query by key) requires either a full scan or an auxiliary index (Riak Search/Solr integration, or 2i — secondary indexes on tagged fields).
- **Operations by sets** refers to bulk/batch operations (map-reduce style aggregation across many keys) rather than a single-key `get`.

### 4.2 Proof — the impossibility of general cross-key atomicity without coordination (specialization of §3.2/Unit I §5.2)

**Claim:** a key-value store offering only per-key atomicity cannot guarantee that a multi-key operation $\{put(k_1,v_1), put(k_2,v_2)\}$ is all-or-nothing unless keys $k_1,k_2$ are guaranteed to reside on the same physical node **and** the store adds an explicit cross-key locking/logging mechanism.

**Proof:**
- By the sharding definition (Unit I §7.2), $h(k_1)$ and $h(k_2)$ may map to different nodes.
- If a node hosting $k_1$ commits its write and then a crash or partition prevents the write to $k_2$'s node, the system is left in a state where exactly one of the two writes is visible — violating atomicity by definition (a partial effect is observable).
- No amount of per-key durability guarantees prevents this, since durability at each node is a local property that says nothing about the *joint* visibility of two independent commits.

This is precisely why "multi-operation transactions" require either confining all keys to one aggregate/partition (§1, Unit I §5.2) or layering a distributed transaction protocol (2PC, Paxos-based commit) with its associated CAP-theorem costs (Unit I §3).

### 4.3 Worked Example

Transferring "credit" represented as two separate key-value entries `wallet:A → 100` and `wallet:B → 50`: a naive `put(wallet:A, 90)` followed by `put(wallet:B, 60)` can fail between the two calls, leaving $10$ units created from nothing (visible partial state) — precisely the failure mode §4.2 proves is possible without added coordination. A relational or single-aggregate document model confines both balances to one atomically-updated unit (Unit I §5.3) and avoids the issue entirely.

### 4.4 Nuances

- "Operations by sets" (batch/map-reduce queries) trade per-operation latency for throughput — they scan/aggregate across many keys in a distributed, parallel fashion (analogous to a distributed `GROUP BY`), which is efficient for analytics but unsuitable for real-time single-record transactions.
- Riak's 2i (secondary index) feature allows tagging a key with indexed fields for range/exact-match queries without a full scan, narrowing (but not eliminating) the $\Omega(n)$ bound of §3.2 to the size of the matching index bucket.
- These limitations are the direct motivation for the document (Unit I §5, §8) and column-family (Unit I §6, §9) models when an application's dominant pattern is "query by attribute" rather than "query by exact key."

### 4.5 Real-World Application

Analytics pipelines built atop Riak commonly use its map-reduce query interface for batch reporting (e.g., "count objects in bucket X by tag"), while keeping real-time application logic strictly to single-key `get`/`put` to stay within the store's atomicity guarantees.

---

## 5. Graph Databases using Neo4j — Data Model and Query Features

### 5.1 Theory

- A **property graph** consists of **nodes** (entities, each with labels and key-value properties) and **relationships** (directed, typed edges, each also carrying properties) connecting nodes.
- Unlike every model in Unit I (which is aggregate-oriented — §7), the graph model makes relationships **first-class, physically stored** citizens via **index-free adjacency**: each node stores direct pointers to its incident relationships, so traversal does not require a lookup/join at all.
- Neo4j's query language, **Cypher**, expresses traversals declaratively via ASCII-art-like patterns: `(a)-[:REL]->(b)`.

### 5.2 Proof — index-free adjacency gives $O(1)$-per-hop traversal, independent of total graph size $|V|$

**Claim:** given a starting node $a$ with degree $d(a)$, retrieving all of $a$'s neighbors costs $O(d(a))$, **not** $O(\log|V|)$ or $O(|V|)$ — i.e., traversal cost depends only on local degree, never on the total number of nodes in the graph.

**Proof:**
- Each node in Neo4j's native storage holds a direct in-memory-pointer-equivalent (a fixed-size record offset) to its first relationship.
- Each relationship record holds pointers to the next/previous relationship for both its start and end node, forming a doubly-linked list per node.
- Retrieving neighbors means following $d(a)$ pointer hops, each $O(1)$ (direct record-offset dereference, no search structure involved) — total cost $O(d(a))$.
- This is fundamentally different from a relational join, which (absent a foreign-key index) must search among all $|V|$ candidate rows for matches (Unit I §10.2's $O(n\log n)$-or-worse bound), or even an *indexed* join, which still costs $O(\log|V|)$ per lookup via a B-tree (Unit I §1.1).
- Chaining $k$ hops of index-free adjacency costs $O(\sum_{i=1}^k d(a_i))$ — independent of $|V|$ — whereas chaining $k$ relational joins compounds $k$ separate $O(\log|V|)$ (or worse) lookups.

This proves the multi-hop query advantage claimed qualitatively in Unit I §10.2/10.3 with a concrete cost model.

### 5.3 Worked Example

Cypher for "friends of friends of Alice, excluding direct friends":
```
MATCH (a:Person {name:"Alice"})-[:FRIEND]->()-[:FRIEND]->(fof)
WHERE NOT (a)-[:FRIEND]->(fof) AND fof <> a
RETURN DISTINCT fof.name
```
Each `-[:FRIEND]->` hop is a pointer-chase costing $O(d)$ per node visited (§5.2) — for a graph of a million people but average friend-count 150, this query touches on the order of $150^2=22{,}500$ relationship traversals, not a fraction of a million-row join.

### 5.4 Nuances

- Index-free adjacency accelerates *traversal*, not arbitrary attribute lookup — finding the *starting* node `Person {name:"Alice"}` still needs a conventional index (Neo4j maintains B-tree-like indexes on labeled properties for this reason), so graph databases combine both index types.
- Graphs deliberately abandon the aggregate-boundary optimization of Unit I §7 — cross-entity operations (traversal) are the *primary* supported access pattern, not an expensive edge case to avoid.
- Cypher's declarative pattern-matching lets the query planner choose traversal direction/order, but poorly bounded patterns (unbounded-depth `*` traversals) can still be expensive — degree explosion in dense graphs is analogous to (but structurally different from) a cross-shard scatter-gather (Unit I §7.2).

### 5.5 Real-World Application

Fraud-ring detection (accounts connected via shared devices/addresses within $k$ hops) and knowledge graphs (Wikidata-style entity relationships) are canonical graph workloads where the query *is* a multi-hop traversal, making index-free adjacency's constant-per-hop cost decisive.

---

## 6. Neo4j Scaling

### 6.1 Theory

- Neo4j scales primarily via **causal clustering**: one or more **core servers** (participating in a Raft consensus group for writes, guaranteeing linearizable, CP-leaning consistency per Unit I §3) plus **read replicas** that asynchronously receive updates to scale read throughput horizontally.
- Unlike Cassandra/Riak's sharding-by-key model (Unit I §9, Unit II §2), Neo4j historically does not natively shard a single graph across nodes by default — full graph replicas plus read-scaling is the primary lever, reflecting graph queries' need for arbitrary cross-entity traversal that sharding would fragment.

### 6.2 Proof — why hash-partitioning a graph breaks the $O(1)$-per-hop guarantee of §5.2

**Claim:** if a graph's nodes are hash-partitioned across $N$ shards, a traversal has probability $\ge 1-\frac{1}{N}$ of crossing a shard boundary at each hop (for $N$ large, roughly uniform hashing, and no special co-location strategy), reintroducing a network round-trip (and its coordination cost, Unit I §7.2) at almost every step.

**Proof:**
- Under uniform hash partitioning, an edge's two endpoints are independently assigned to one of $N$ shards.
- The probability both endpoints land on the *same* shard is $1/N$ (matching the second endpoint's independent draw), so the probability of a cross-shard edge is $1-1/N$.
- Real traversal queries chain many hops, and each cross-shard hop requires a network call rather than the local pointer-dereference of §5.2.

Naive hash-partitioning would therefore degrade the very $O(d(a))$ guarantee that makes graph databases valuable — the formal reason Neo4j favors full-graph replication (or careful relationship-aware partitioning schemes) over naive key-hash sharding, unlike the aggregate-oriented stores of Unit I where cross-shard operations are already assumed rare by design (§7.3).

### 6.3 Worked Example

A causal cluster with 3 core servers (Raft majority = 2) and 5 read replicas: a write (e.g., creating a `FRIEND` relationship) commits once 2 of 3 cores acknowledge (a direct application of the quorum-intersection proof, Unit I §4.2, with $N=3, W=2$); read replicas asynchronously catch up, so a read immediately after a write on a *different* replica may briefly lag (an eventual-consistency window, Unit I §4.4) unless the driver requests **causal consistency** (bookmark-based read-your-writes).

### 6.4 Nuances

- Causal clustering's read/write split mirrors the general leader/follower replication pattern seen across NoSQL systems, but Neo4j's Raft-based core is explicitly CP (Unit I §3.4) rather than AP — a write-availability trade-off distinct from Riak/Cassandra's typical AP default.
- "Fabric" (Neo4j's federated/sharding layer in newer versions) allows composing multiple graph databases for horizontal scale, but cross-database traversal reintroduces the cross-shard cost of §6.2 — graph sharding remains fundamentally harder than aggregate-store sharding.
- Read-replica scaling assumes the read workload dominates — heavy write workloads still bottleneck on the core Raft group's consensus latency.

### 6.5 Real-World Application

Large recommendation and knowledge-graph deployments typically scale Neo4j vertically (larger core-server memory, since traversal performance benefits from keeping the graph resident in memory) combined with read replicas for query fan-out, rather than horizontal key-sharding.

---

## 7. Suitable Use Cases — Connected Data, Routing/Dispatch/Location-Based Services, Recommendation Engines

### 7.1 Theory

- Graph databases excel whenever the **query itself is a relationship traversal**:
  - **Connected data** — social networks, org charts.
  - **Routing/dispatch/location-based services** — shortest-path and nearest-neighbor-along-a-network queries.
  - **Recommendation engines** — collaborative filtering expressed as graph traversal, e.g., "customers who bought X also bought".
- Each maps directly to §5.2's $O(d)$-per-hop cost model rather than the aggregate/index-lookup model of Unit I.

### 7.2 Proof — shortest-path (Dijkstra/BFS) complexity on index-free adjacency vs. relational adjacency

**Claim:**
- BFS shortest-path on a graph with $|V|$ nodes and $|E|$ edges runs in $O(|V|+|E|)$ time using index-free adjacency (each node's neighbor list accessed in $O(1)$ per neighbor, §5.2) — matching the standard graph-algorithms textbook bound.
- The equivalent computation over a relational adjacency table (`edges(from, to)`) without a native adjacency structure costs $O(|V|\cdot\log|E|)$ or worse per BFS layer (one indexed join per frontier expansion, Unit I §1.1's B-tree bound), since each "get neighbors of node $v$" operation is a separate indexed query rather than a pointer dereference.

**Proof:**
- BFS visits each node once ($O(|V|)$ enqueue/dequeue operations) and examines each edge at most twice (once from each endpoint, $O(|E|)$ total edge examinations) — the standard BFS complexity argument, valid here because index-free adjacency provides true $O(1)$ neighbor access per edge examined (§5.2), matching the algorithm's assumed cost model exactly.
- A relational adjacency table instead pays $O(\log|E|)$ *per edge examined* (a B-tree index lookup to find rows where `from = v`), inflating total cost to $O(|E|\log|E|)$ — asymptotically worse, and empirically far worse once join overhead and page-cache misses are included.

### 7.3 Worked Example

Dispatch routing: `MATCH p = shortestPath((start:Location)-[:ROAD*..10]-(end:Location)) RETURN p` finds a route within 10 hops by BFS-style expansion over stored road-segment relationships, each hop an $O(1)$ pointer traversal (§7.2) — directly usable for real-time dispatch/ETA systems where relational recomputation of joins at each hop would be too slow for interactive use.

### 7.4 Nuances

- "Location-based services" here refers to network/graph proximity (hops along a road/connection graph), distinct from geospatial coordinate indexing (`$geoNear`/R-trees), which is a document-database feature developed in Unit III.
- Recommendation engines modeled as graphs (`(user)-[:BOUGHT]->(product)<-[:BOUGHT]-(otherUser)-[:BOUGHT]->(recommendation)`) express collaborative filtering as a 3-hop traversal — a direct application of §7.2's BFS-cost advantage over computing the equivalent via repeated relational self-joins.
- Not every "connected" dataset benefits from a graph database — if the dominant query is still single-entity lookup with occasional relationship access, a document store with reference fields (Unit I §5.4) may suffice at lower operational complexity.

### 7.5 Real-World Application

Ride-hailing dispatch (nearest available driver along the road network), telecom network-outage impact analysis (which downstream nodes are affected), and e-commerce "customers also bought" recommendation panels are all production Neo4j-class workloads built directly on §7.2's shortest-path/traversal cost advantage.

---

## Practice Problems (Exam-Style)

**P1 — Vector clock conflict detection.** From $VC_A=\{n1:1,n2:0\}$, two nodes independently produce $VC_B=\{n1:2,n2:0\}$ and $VC_C=\{n1:1,n2:1\}$. Are $B$ and $C$ concurrent?

*Solution:* Compare entrywise: $B=(2,0)$ vs $C=(1,1)$ — neither dominates (B is bigger in slot 1, C is bigger in slot 2). **Concurrent siblings** (§1's partial-order proof) — Riak returns both for resolution.

**P2 — Vnode redistribution.** A 5-node cluster has 100 vnodes (20 per node). One node fails. How are its vnodes redistributed?

*Solution:* The failed node's 20 vnodes are redistributed across the remaining 4 nodes, 5 vnodes each — each surviving node's load rises from 20→25 vnodes, a small, evenly-spread increase rather than one node absorbing the whole failed node's range (§2's $O(1/\sqrt V)$ variance argument).

**P3 — Riak suitability scenario.** Is Riak appropriate for (a) a shopping cart, (b) a bank account balance?

*Solution:* (a) **Yes** — single-key, write-heavy, must-never-reject (§2's AP use case). (b) **No** — balance updates need strict consistency and often multi-key transactions (debit one account, credit another), which Riak's model explicitly does not guarantee (§4's cross-key atomicity limitation).

**P4 — Cypher query.** Find friends-of-friends of "Alice" who are not already her direct friends.

*Solution:*
```cypher
MATCH (a:Person {name:"Alice"})-[:FRIEND]->()-[:FRIEND]->(fof)
WHERE NOT (a)-[:FRIEND]->(fof) AND fof <> a
RETURN DISTINCT fof.name
```
Each `-[:FRIEND]->` hop is an $O(1)$ index-free-adjacency traversal (§5.2) — no join cost regardless of total graph size.

**P5 — Traversal cost comparison.** A social graph has a path Alice→Bob→Carol→Dave (3 hops). Compare Neo4j traversal cost to the equivalent relational self-join query.

*Solution:* Neo4j: 3 pointer hops, $O(1)$ each → $O(3)$ total (§7.2). Relational: a 3-hop path needs a 3-way self-join on a `friends(user_id, friend_id)` table, each join costing $O(n\log n)$ or worse without perfect indexing — cost grows with total table size, not just path length, which is exactly the asymptotic gap §7.2's proof formalizes.

---

## Unit II Summary

| Concept | One-line takeaway |
|---|---|
| Riak key-value features | Vector clocks form a provable partial order that correctly flags concurrent sibling writes |
| Scaling (vnodes) | Virtual nodes reduce load-imbalance variance by a provable $O(1/\sqrt V)$ factor |
| Suitable use cases | Session/profile/cart data fit single-key access; AP behavior reuses vector-clock conflict handling |
| When not to use key-value | Relationship queries cost $\Omega(n)$ without an auxiliary index — a proven lower bound |
| Multi-op transactions | Cross-key atomicity is impossible without confinement to one aggregate or added coordination |
| Neo4j data model | Index-free adjacency gives $O(1)$-per-hop traversal, independent of total graph size |
| Neo4j scaling | Hash-sharding a graph breaks the $O(1)$-per-hop guarantee — causal clustering (replicas) is preferred |
| Connected data / routing / recommendations | BFS on index-free adjacency is $O(|V|+|E|)$, provably better than relational adjacency-table joins |
