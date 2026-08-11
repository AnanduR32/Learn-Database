# Unit II — Key-Value Databases (Riak) & Graph Databases (Neo4j)
### 25CSA642A — NoSQL Databases

This unit explores Key-Value databases using **Riak** (focusing on consistency, vector clocks, virtual nodes, and transaction limits) and Graph databases using **Neo4j** (focusing on nodes, relationships, index-free adjacency, and scaling).

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

### 1.1 Core Concepts (Simplified)
- **Riak:** A masterless Key-Value store based on Amazon's Dynamo architecture.
- **Data Access:** Data is stored as an opaque byte blob under a specific **Key** inside a **Bucket** (namespace).
- **Consistency Mechanics:** Tuned using the quorum triple $(N, R, W)$ from Unit I.
- **Vector Clocks:** Every object maintains a small history map `{NodeID -> Counter}` to track causal changes across distributed nodes.
  - If Clock 1 dominates Clock 2 (all counters $\ge$ and at least one $>$), Clock 1 is newer (**no conflict**).
  - If neither clock dominates the other, a **sibling conflict** exists. Riak presents both versions to the client or uses a CRDT merge rule to resolve them.

### 1.2 Vector Clock Logic & Proof Sketch
- **Partial Ordering:** The vector clock comparison relation $\preceq$ is mathematically a partial order (reflexive, antisymmetric, transitive).
- **Conflict Detection:** If Node 1 and Node 2 accept updates while disconnected, Node 1 increments its counter `{n1: 3, n2: 1}` while Node 2 increments its counter `{n1: 2, n2: 2}`. 
- Comparing $\{3,1\}$ and $\{2,2\}$ shows neither vector dominates the other. This reliably signals a **concurrent write conflict** created during a network partition.

### 1.3 Worked Example
1. Object created: Clock = `{node1: 1}`.
2. Node 1 updates object: Clock = `{node1: 2}`.
3. Network breaks. Node 1 updates: `{node1: 3, node2: 0}`. Node 2 independently updates: `{node1: 2, node2: 1}`.
4. When network heals, Riak compares `{3,0}` and `{2,1}`. Neither is strictly greater, so Riak creates **siblings** for the application to resolve.

### 1.4 Key Takeaways & Nuances
- Riak treats stored values as unreadable "opaque blobs." The database cannot inspect internal JSON fields unless add-on search modules (like Solr or 2i secondary indexes) are used.
- **CRDTs (Conflict-free Replicated Data Types):** Data structures (like grow-only sets or counters) built into Riak that automatically resolve sibling conflicts without manual application code.

---

## 2. Scaling and Suitable Use Cases

### 2.1 Core Concepts (Simplified)
- **Virtual Nodes (vnodes):** Instead of mapping 1 physical server to 1 large range on the hash ring, each physical server is divided into many **virtual nodes** scattered around the ring.
- **Benefits:**
  1. Smooths out hot-spots across physical servers.
  2. If a physical node fails, its workload is split across many surviving nodes instead of swamping a single neighbor.
- **Ideal Key-Value Use Cases:**
  - **Session Management:** Storing web user login tokens by Session ID.
  - **User Profiles / Preferences:** Reading/writing a complete user blob by User ID.
  - **Shopping Carts:** Fast writes that must always accept items during peak traffic.

### 2.2 Proof — Load Balance Improvement via Vnodes
Assigning $V$ virtual nodes per physical machine causes the load variation (variance) between physical servers to drop by a factor of $O(1/\sqrt{V})$.
- *Intuition:* Averaging many small random slices across the hash ring balances out server disk usage far better than assigning one giant slice per server.

### 2.3 Real-World Application
E-commerce websites use key-value stores for shopping carts because `get(cart_id)` and `put(cart_id, blob)` are $O(1)$ operations that never block checkout during sales.

---

## 3. When Not to Use a Key-Value Store — Relationships among Data

### 3.1 Core Concepts (Simplified)
Key-value stores are inefficient for queries based on relationships or non-key attributes (e.g., *"Find all users who bought Product X"*).
- Because values are opaque blobs indexed only by key, searching inside values requires inspecting every single entry in the database ($\Omega(n)$ worst-case scan).

### 3.2 Proof — Relationship Search Lower Bound
Without a secondary index mapping attributes to keys, finding an unindexed attribute across $n$ items requires examining all $n$ items in the worst case ($\Omega(n)$ complexity).
- Pushing the responsibility onto the application (by maintaining manual lookup keys like `"product:X:users"`) introduces potential update anomalies if keys fall out of sync.

---

## 4. Multi-Operation Transactions, Query by Data, Operations by Sets

### 4.1 Core Concepts (Simplified)
- **Multi-Key Atomicity:** Key-value stores only guarantee atomic operations on a **single key**. Transferring money between two separate key balances (`wallet:A` and `wallet:B`) cannot be executed as a native atomic transaction without added distributed locking.
- **Operations by Sets:** Bulk reporting (e.g., MapReduce across millions of keys) prioritizes high throughput analytics over real-time transaction speed.

### 4.2 Worked Example — Multi-Key Failure
If an application attempts to move $10 from `AccountA` to `AccountB`:
1. `put("AccountA", balanceA - 10)` succeeds.
2. System crashes before `put("AccountB", balanceB + 10)` runs.
3. Money vanishes because the operations were not bound inside a single atomic multi-key transaction.

---

## 5. Graph Databases using Neo4j — Data Model and Query Features

### 5.1 Core Concepts (Simplified)
- **Property Graph Model:** Consists of:
  - **Nodes:** Entities (e.g., `Person`, `Product`).
  - **Relationships (Edges):** Directional connections linking nodes (e.g., `:FRIENDS_WITH`, `:PURCHASED`).
  - **Properties:** Key-value attributes on nodes or edges (e.g., `name: "Alice"`, `since: 2022`).
- **Index-Free Adjacency:** Nodes store direct memory pointers to their neighbor nodes. Traversing a relationship does not perform a table search or join; it simply follows a pointer.
- **Cypher:** Neo4j's declarative query language using visual pattern syntax: `(a:Person)-[:FRIENDS_WITH]->(b:Person)`.

### 5.2 Proof — Constant-Time $O(1)$ Traversal per Hop
- **SQL Join Traversal:** Finding connections requires searching foreign keys in an index, costing $O(\log N)$ per lookup where $N$ is total table size.
- **Neo4j Traversal:** Each node holds direct pointers to its relationships. Following a pointer to a neighbor costs $O(1)$ constant time per edge.
- **Key Result:** Traversing $k$ hops costs time proportional only to the local degree (number of connections) of the visited nodes, **completely independent of the total graph size $|V|$**.

### 5.3 Worked Example (Cypher Query)
Find friends of Alice's friends who are not already Alice's direct friends:
```cypher
MATCH (a:Person {name: "Alice"})-[:FRIEND]->()-[:FRIEND]->(fof)
WHERE NOT (a)-[:FRIEND]->(fof) AND fof <> a
RETURN DISTINCT fof.name
```
Each `-[:FRIEND]->` step follows direct memory pointers ($O(1)$ per edge), making this fast even in a graph with 100 million total users.

---

## 6. Neo4j Scaling

### 6.1 Core Concepts (Simplified)
- **Causal Clustering:** Neo4j scales using:
  - **Core Servers:** A small group of nodes running the Raft consensus protocol to manage write operations securely (CP focus).
  - **Read Replicas:** Asynchronous follower nodes that replicate graph data to handle massive read traffic.
- **Why Graph Sharding is Hard:** Hash-partitioning graph nodes across different servers breaks Index-Free Adjacency. If an edge crosses between two physical servers, traversal requires a network round-trip instead of an in-memory pointer chase.

---

## 7. Suitable Use Cases — Connected Data, Routing, & Recommendations

### 7.1 Core Use Cases
1. **Social Networks & Org Charts:** Deeply nested relationship lookups.
2. **Fraud Detection:** Identifying suspicious loops (e.g., 5 accounts sharing the same phone number and credit card within 3 hops).
3. **Recommendation Engines:** Collaborative filtering (`(User)-[:BOUGHT]->(Product)<-[:BOUGHT]-(OtherUser)-[:BOUGHT]->(Recommendation)`).
4. **Shortest-Path Routing:** Calculating network or road paths using Dijkstra/BFS algorithms.

### 7.2 Proof — BFS Shortest Path Complexity
Running Breadth-First Search (BFS) for shortest path on a property graph with $|V|$ nodes and $|E|$ edges runs in $O(|V| + |E|)$ time because neighbor lookups are $O(1)$. In SQL, the equivalent recursive query costs $O(|E| \log |E|)$ due to index lookup overhead at each step.

---

## Practice Problems & Solutions

1. **Vector Clock Comparison:** Given $VC_A = \{n1: 2, n2: 0\}$ and $VC_B = \{n1: 1, n2: 1\}$, are these clocks in conflict?
   * *Solution:* **Yes.** $VC_A$ is greater in slot 1, but $VC_B$ is greater in slot 2. Neither dominates, creating concurrent **sibling** versions.
2. **Virtual Node Redistribution:** A 4-node cluster has 80 vnodes (20 per server). If 1 server fails, how is its load distributed?
   * *Solution:* Its 20 vnodes are evenly divided among the remaining 3 servers ($\approx 6-7$ vnodes each), spreading the extra load evenly.
3. **Traversal Cost:** Why is a 4-hop query in Neo4j faster than a 4-table join in SQL on a huge dataset?
   * *Solution:* Neo4j uses **Index-Free Adjacency** (direct pointers), executing each hop in $O(1)$ constant time based on local node degree, whereas SQL must repeatedly search B-tree indexes scaling with total table size.

---

## Unit II Summary Cheat-Sheet

| Concept | Key Takeaway |
|---|---|
| **Vector Clocks** | Detect concurrent write conflicts during network partitions. |
| **Virtual Nodes** | Smooth out load balance across servers and prevent single-server overload on failure. |
| **Key-Value Limit** | Searching inside values without a key requires an $O(n)$ full database scan. |
| **Index-Free Adjacency** | Storing direct pointers makes graph traversal speed independent of overall graph size. |
| **Neo4j Scaling** | Uses Causal Clustering (Raft core + Read Replicas); avoids naive cross-server graph sharding. |
| **Graph Use Cases** | Fraud detection, recommendation engines, social graphs, and network routing. |
