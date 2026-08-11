# Unit I — Relational Foundations, NoSQL Motivations & Data Models
### 25CSA642A — NoSQL Databases

This unit builds the conceptual bridge from relational (SQL) databases to NoSQL. It reviews why traditional databases normalize data, explains why that design hits scaling limits, formalizes the CAP theorem and the ACID vs. BASE trade-offs, and walks through the four NoSQL data models (Key-Value, Document, Column-Family, and Graph) before exploring MongoDB, Cassandra, and database comparisons.

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

### 1.1 Core Concepts (Simplified)

**The Relational Model:**
- Data is organized into **tables** (relations) consisting of rows (tuples) and columns (attributes).
- Queried using **SQL**, which performs table operations like:
  - **Selection ($\sigma$):** Filtering rows (like `WHERE`).
  - **Projection ($\pi$):** Choosing specific columns (like `SELECT col1, col2`).
  - **Join ($\bowtie$):** Combining multiple tables based on matching key columns.

**Constraints:**
- **Primary Key (PK):** A unique identifier for every row (cannot be null).
- **Foreign Key (FK):** A reference column in one table pointing to a Primary Key in another table to maintain relationships.

**Functional Dependencies (FD) & Normalization:**
- A **Functional Dependency ($X \to Y$)** means if two rows have the same value for column $X$, they *must* have the same value for column $Y$ ($X$ determines $Y$).
- **Normalization** rules structure tables to eliminate duplicate data:
  - **1NF:** Every cell contains a single (atomic) value; no multi-value lists.
  - **2NF:** Every non-key column must depend on the *entire* primary key, not just part of a composite key.
  - **3NF:** No non-key column should depend on another non-key column (no indirect/transitive dependencies).
  - **BCNF:** A stricter version of 3NF ensuring every determinant is a candidate key.

**Indexing:**
- Speeds up data lookups using a **B-tree** structure (a balanced search tree).
- Instead of scanning all $n$ rows in a table, an index lookup finds a row in $O(\log n)$ steps.

### 1.2 Mathematical Proof — Armstrong's Axioms & Lossless Join

**Soundness of Armstrong's Axioms:**
The three basic rules for deriving functional dependencies are:
1. **Reflexivity:** If $Y$ is a subset of $X$, then $X \to Y$. (If you know a person's Full Name & DOB, you automatically know their DOB).
2. **Augmentation:** If $X \to Y$, then $XZ \to YZ$. (Adding attribute $Z$ to both sides maintains the dependency).
3. **Transitivity:** If $X \to Y$ and $Y \to Z$, then $X \to Z$. (If Student_ID determines Major, and Major determines Department_Head, then Student_ID determines Department_Head).

*Proof Intuition:* Each rule logically follows directly from the definition of a functional dependency. Therefore, any dependency derived using these rules is guaranteed to be true (**sound**).

**Lossless-Join Decomposition:**
When splitting a big table $R$ into two smaller tables $R_1$ and $R_2$, we must ensure no data or false combinations are created when rejoining them ($R_1 \bowtie R_2 = R$).
- **Rule:** The split is lossless if the shared column(s) ($R_1 \cap R_2$) functionally determine either $R_1$ or $R_2$ (i.e., the shared column is a primary key in at least one of the new tables).

### 1.3 Worked Example

Consider an unnormalized table: `Orders(OrderID, CustomerID, CustomerName, ProductID, ProductName, Qty)`.
- **Dependencies:**
  - `OrderID, ProductID` $\to$ `Qty`
  - `CustomerID` $\to$ `CustomerName`
  - `ProductID` $\to$ `ProductName`
  - `OrderID` $\to$ `CustomerID`
- **Problem:** `CustomerName` depends on `CustomerID` (violates 3NF/BCNF).
- **Normalized Solution:** Split into 4 clean tables:
  1. `Customers(CustomerID, CustomerName)`
  2. `Products(ProductID, ProductName)`
  3. `Orders(OrderID, CustomerID)`
  4. `OrderItems(OrderID, ProductID, Qty)`

### 1.4 Key Takeaways & Nuances
- Normalization prevents data anomalies (accidental deletions/inconsistencies) but increases the number of `JOIN` operations needed to read full records.
- Every index speeds up reads, but slows down writes (`INSERT`/`UPDATE`) because the B-tree must be updated on every change.

### 1.5 Real-World Application
Relational databases (PostgreSQL, MySQL) are standard for transactional applications like banking and inventory where strict non-redundancy and multi-table integrity are essential.

---

## 2. Introduction to NoSQL — Motivations and the Scaling Problem

### 2.1 Core Concepts (Simplified)
- **NoSQL ("Not Only SQL"):** Databases designed for horizontal scaling, flexible schemas, and high write/read speeds, relaxing traditional SQL guarantees (like joins or strict consistency).
- **Scaling Strategies:**
  - **Vertical Scaling ("Scale Up"):** Adding more RAM/CPUs to a single server. It quickly hits hardware limits and gets exponentially expensive.
  - **Horizontal Scaling ("Scale Out"):** Adding more low-cost server machines (nodes) into a cluster.

### 2.2 Proof — Amdahl's Law Limits Single-Machine Scaling
Amdahl's Law calculates the maximum speedup $S(N)$ obtained when using $N$ parallel processors for a workload where a fraction $p$ can be parallelized, leaving $(1-p)$ serial:

$$S(N) = \frac{1}{(1-p) + \frac{p}{N}}$$

As $N \to \infty$, the maximum theoretical speedup reaches a hard limit of $\frac{1}{1-p}$.
- *Intuition:* If 10% of a database operation must run sequentially (like waiting for a single global lock or write-ahead log), performance can **never exceed 10x**, no matter how powerful the machine is.
- *NoSQL Solution:* NoSQL eliminates global locks by **partitioning (sharding)** data across independent machines, allowing linear horizontal scaling.

### 2.3 Worked Example
If a query engine is 90% parallelizable ($p=0.9$):
- With 10 CPUs: Speedup $= 5.26\times$
- With 100 CPUs: Speedup $= 9.17\times$
- Theoretical maximum speedup $= 10\times$
By contrast, 10 independent NoSQL shards with non-overlapping data key ranges achieve nearly $10\times$ capacity, and an 11th shard adds another $\sim 1\times$ increment.

### 2.4 Key Takeaways & Nuances
- NoSQL is not "schema-less"; it uses **schema-on-read** (the application code interprets data when reading it) instead of **schema-on-write** (the database enforcing table constraints on insert).

### 2.5 Real-World Application
Amazon built DynamoDB and Google built Bigtable because standard SQL engines could not economically scale to global e-commerce shopping carts or web indexing.

---

## 3. CAP Theorem

### 3.1 Core Concepts (Simplified)
In a distributed database running across a network of machines, you can only guarantee **2 out of 3** properties at the same time:
1. **Consistency (C):** Every read returns the most recent write or an error (Linearizability).
2. **Availability (A):** Every request receives a non-error response (though it might contain slightly stale data).
3. **Partition Tolerance (P):** The system continues operating despite network communication failures/delays between nodes.

Since physical networks inevitably experience cable drops or delays (**P is non-negotiable**), distributed databases must choose between **Consistency (CP)** or **Availability (AP)** during a network partition.

### 3.2 Proof — Gilbert & Lynch Impossibility Proof Sketch
Suppose a network breaks, separating Node 1 and Node 2.
1. A user writes new value $V_1$ to Node 1.
2. Because of the network partition, Node 1 cannot send $V_1$ to Node 2.
3. Another user reads from Node 2.
4. If Node 2 responds immediately (for **Availability**), it returns stale data, violating **Consistency**.
5. If Node 2 waits or errors out to maintain **Consistency**, it sacrifices **Availability**.
*Conclusion:* A system cannot guarantee both C and A during a network partition.

### 3.3 Worked Example
During a datacenter network failure:
- **AP Choice (Riak / DynamoDB):** Shopping cart accepts items on both disconnected nodes; conflicting cart versions are merged later when the network recovers.
- **CP Choice (HBase / MongoDB):** The disconnected node refuses reads/writes to prevent showing or recording out-of-date financial balances.

### 3.4 Key Takeaways & Nuances
- **PACELC Theorem:** Refines CAP by stating: **If Partitioned (P)**, choose between **A** and **C**; **Else (E)** (normal operations), choose between **Latency (L)** and **Consistency (C)**.

---

## 4. ACID vs BASE

### 4.1 Core Concepts (Simplified)
- **ACID (SQL Defaults):**
  - **Atomicity:** All operations in a transaction succeed, or all roll back.
  - **Consistency:** Transactions preserve schema constraints and invariants.
  - **Isolation:** Concurrent transactions don't interfere with each other.
  - **Durability:** Committed changes survive system crashes.
- **BASE (NoSQL Defaults):**
  - **Basically Available:** The system stays online for reads/writes despite failures.
  - **Soft state:** Data may change over time without explicit user action (due to background syncs).
  - **Eventual consistency:** Given enough time without new updates, all nodes will converge to identical values.

### 4.2 Proof — Quorum Math ($R + W > N$)
In a cluster with $N$ total replicas for a key:
- $W$ = Number of nodes that must confirm a write operation.
- $R$ = Number of nodes queried during a read operation.

**Theorem:** If $R + W > N$, every read set overlaps with at least one node from the latest write set.
- *Proof (Pigeonhole Principle):* If you choose $W$ nodes out of $N$ for writing, and $R$ nodes out of $N$ for reading, and $R + W > N$, the two subsets *must* share at least 1 overlapping node. That node holds the latest write timestamp, guaranteeing **Strong Read-Your-Writes Consistency**.
- If $R + W \le N$, reads might hit nodes that haven't received the write yet (**Eventual Consistency**).

### 4.3 Worked Example
With $N=3$ replicas:
- **Strong Consistency:** $W=2, R=2$ ($2+2 = 4 > 3$). Every read hits at least 1 updated replica and tolerates 1 node failure.
- **Eventual Consistency:** $W=1, R=1$ ($1+1 = 2 \le 3$). Extremely fast writes/reads, but reads can temporarily return old data.

---

## 5. Types of NoSQL Databases — Key-Value and Document Models

### 5.1 Core Concepts (Simplified)
- **Key-Value Model:** A massive distributed hash map (`get(key)` / `put(key, value)`). The database treats the value as an unreadable (opaque) byte blob.
- **Document Model:** Stores structured records (JSON, BSON, XML). The database can inspect internal fields, index them, and run partial updates.
- **Aggregate Unit:** A self-contained document holding an entity and its nested details, avoiding the need for multi-table relational joins.

### 5.2 Proof — Single-Aggregate Atomicity
If all data for a business transaction (e.g., an Order and its Line Items) lives inside a single document/aggregate:
- The database routes the document to a single server based on its Key's hash: $h(\text{key}) \to \text{Node}$.
- Updates within that single document are completed locally using local locks/logs without requiring complex distributed multi-node commit protocols.

### 5.3 Worked Example
Instead of joining `Orders`, `OrderItems`, and `Customers` tables, a MongoDB document embeds everything together:
```json
{
  "_id": 1042,
  "customer": "Alice",
  "items": [
    {"sku": "A1", "qty": 2, "price": 9.99},
    {"sku": "B7", "qty": 1, "price": 24.50}
  ]
}
```
Updating an item quantity in this single document is atomic and requires zero cross-server coordination.

---

## 6. Column-Family (Wide-Column) Stores

### 6.1 Core Concepts (Simplified)
- Used by Apache Cassandra and Bigtable.
- Organizes data into dynamic rows where each row key maps to column families and timestamped column values: `(row_key, column_family, column_qualifier, timestamp) -> value`.
- Unlike SQL, rows in the same table do not need to have the same columns. Space is consumed only for populated cells (**sparse data**).

### 6.2 Proof — Sparse Storage Efficiency
In SQL, a table with $R$ rows and $C$ columns uses $O(R \times C)$ memory allocations (reserving space or NULL bitmasks for unpopulated cells).
In Wide-Column stores, only existing triples `(row, column, value)` are stored, consuming $O(R \times k)$ space where $k$ is the average number of populated columns per row. When $k \ll C$, storage savings are massive.

---

## 7. Aggregate-Oriented Databases

### 7.1 Core Concepts (Simplified)
- "Aggregate-Oriented" is the umbrella term for Key-Value, Document, and Column-Family stores.
- They group related pieces of data together into an **aggregate** (the primary unit for storage, sharding, and atomic updates), avoiding multi-table joins.

### 7.2 Sharding Guarantee
Because data is sharded by aggregate key ($h(\text{key}) \to \text{Node}$), any operation confined to a single aggregate contacts exactly **1 node**. Operations spanning multiple aggregates must fan out to $\ge 2$ nodes, incurring network latency.

---

## 8. NoSQL Databases using MongoDB — Data Model & Queries

### 8.1 Core Concepts (Simplified)
- MongoDB stores data as **BSON** (Binary JSON) documents inside **collections**.
- Queries filter collections using JSON criteria: `db.orders.find({ status: "shipped" })`.

### 8.2 Index Lookup Complexity
Searching documents sequentially costs $O(n)$ time (Full Collection Scan). Creating a B-tree index on a field reduces point/range lookup time to $O(\log n)$.

---

## 9. Column-Oriented NoSQL Databases using Apache Cassandra

### 9.1 Core Concepts (Simplified)
- Cassandra uses a masterless, peer-to-peer ring architecture with no single point of failure.
- A **partitioner** (Murmur3 hash) maps row keys onto positions along a circular ring to assign data to nodes.

### 9.2 Proof — Consistent Hashing Rebalancing
In naive modulo hashing ($h(\text{key}) \bmod N$), changing the node count from $N$ to $N+1$ forces almost **100% of keys ($O(K)$)** to move to new nodes.
Under **Consistent Hashing**, adding or removing 1 node only remaps an expected fraction of $\frac{1}{N}$ keys ($O(K/N)$) to adjacent nodes on the ring, enabling seamless scaling without cluster-wide data re-shuffling.

---

## 10. Comparison of Relational Databases to NoSQL Stores

| Feature | RDBMS (SQL) | MongoDB | Cassandra | Neo4j |
|---|---|---|---|---|
| **Data Model** | Tables & Rows | BSON Documents | Wide-Column | Property Graph |
| **Schema** | Fixed (Schema-on-write) | Flexible | Flexible per partition | Flexible |
| **Joins** | Native SQL `JOIN` | Limited (`$lookup`) | None (Denormalize) | Native Pointer Traversal |
| **CAP Focus** | CP (Single-node CA) | CP-leaning | AP-leaning (Tunable) | CP |
| **Primary Use** | Financial Ledgers / ERP | Web Catalogs / Content | Global Event Logs / IoT | Social & Fraud Networks |

---

## Practice Problems & Solutions

1. **CAP Choice:** A mobile cart app loses connection to its primary datacenter. Should it prioritize C or A?
   * *Solution:* Prioritize **Availability (AP)**. Allow customers to add items to their local cart, and resolve conflicting cart versions once connection is restored.
2. **Quorum Calculation:** For $N=5$ replicas, pick $(R, W)$ values for strong consistency.
   * *Solution:* Choose **$R=3, W=3$**. Since $R + W = 6 > 5$, read and write quorums are guaranteed to overlap.
3. **Index Selection:** Given index `{status: 1, date: -1}`, does it optimize `find({date: "2026-01-01"})`?
   * *Solution:* **No.** The query skips the prefix field (`status`), violating the B-tree compound index prefix rule.

---

## Unit I Summary Cheat-Sheet

| Concept | Key Takeaway |
|---|---|
| **Normalization** | Eliminates duplicate data but makes reads slower due to `JOIN`s. |
| **Amdahl's Law** | Proves single-server parallel speedup is capped by serial code sections. |
| **CAP Theorem** | When network partitions occur, you must pick between Consistency or Availability. |
| **Quorum Formula** | $R + W > N$ guarantees you always read fresh data. |
| **Consistent Hashing** | Adding/removing nodes remaps only $O(K/N)$ keys instead of re-shuffling everything. |
