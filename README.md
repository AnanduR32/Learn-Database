# NoSQL Databases — Reference Guide & Master Curriculum
### Course Code: 25CSA642A — NoSQL Databases

A complete, production-grade guide and academic course material covering distributed NoSQL systems, theoretical foundations (CAP, BASE, PACELC), core data models, query languages, storage engine mechanics, application development, and modern enterprise architectures.

---

## 1. Structured Course Units

| Unit | File | Core Topics |
|---|---|---|
| **Unit I** | [`Course/Unit 1 - Foundations and Data Models.md`](file:///C:/Dev/Github/Learn-Database/Course/Unit%201%20-%20Foundations%20and%20Data%20Models.md) | Relational limits, Armstrong's Axioms, Amdahl's Law, CAP & PACELC theorems, ACID vs BASE, 4 Data Models, Quorum Math ($R+W>N$). |
| **Unit II** | [`Course/Unit 2 - KeyValue and Graph Databases.md`](file:///C:/Dev/Github/Learn-Database/Course/Unit%202%20-%20KeyValue%20and%20Graph%20Databases.md) | Riak KV (Vector Clocks, CRDTs, vnodes), Redis (Data structures, Cache-Aside, Eviction policies, Distributed locks), Neo4j (Property graphs, Cypher, Index-Free Adjacency, Causal clustering). |
| **Unit III** | [`Course/Unit 3 - MongoDB Python and Web Frameworks.md`](file:///C:/Dev/Github/Learn-Database/Course/Unit%203%20-%20MongoDB%20Python%20and%20Web%20Frameworks.md) | PyMongo connection pooling, document mutations ($set, $unset, $rename, $inc), array positional filters ($[elem]), indexes & ESR rule, Aggregation Pipelines ($match, $group, $unwind, $lookup, $facet), Flask & Django integration. |
| **Unit IV** | [`Course/Unit 4 - Columnar Stores Cassandra and Storage Engines.md`](file:///C:/Dev/Github/Learn-Database/Course/Unit%204%20-%20Columnar%20Stores%20Cassandra%20and%20Storage%20Engines.md) | Wide-column architecture, LSM-trees vs B-Trees, Memtables, SSTables & Tombstones, Bloom filter math, Compaction strategies (STCS, LCS, TWCS), Chebotko query-driven data modeling, Paxos LWT, Cassandra Python driver. |
| **Unit V** | [`Course/Unit 5 - Polyglot Persistence Vector Databases and Advanced Topics.md`](file:///C:/Dev/Github/Learn-Database/Course/Unit%205%20-%20Polyglot%20Persistence%20Vector%20Databases%20and%20Advanced%20Topics.md) | Polyglot Persistence blueprint, Saga pattern vs 2PC, Change Data Capture (CDC via Debezium/Kafka), Vector Databases & HNSW indexing, MongoDB Vector Search & Redis VSS, GraphRAG with Neo4j, YCSB benchmarking. |

---

## 2. Standalone Database Reference Guides

| # | File | Type | Key Coverage |
|---|---|---|---|
| 1 | [`1 - Intro.md`](file:///C:/Dev/Github/Learn-Database/1%20-%20Intro.md) | Overview | NoSQL overview, CAP theorem, BASE, database categories, normalization review |
| 2 | [`2 - MongoDB.md`](file:///C:/Dev/Github/Learn-Database/2%20-%20MongoDB.md) | Document Store | BSON, CRUD operators ($set, $unset, array modifiers, positional filters), full Aggregation Pipelines ($lookup, $facet, $bucket), Indexes, WiredTiger storage engine |
| 3 | [`3 - Cassandra.md`](file:///C:/Dev/Github/Learn-Database/3%20-%20Cassandra.md) | Wide-Column Store | CQL, architecture, tunable consistency, Paxos Lightweight Transactions (LWT), Collections & UDTs, SSTables & compaction |
| 4 | [`4 - Riak.md`](file:///C:/Dev/Github/Learn-Database/4%20-%20Riak.md) | Key-Value Store | HTTP API, CRDTs, vector clocks, tunable consistency (N, R, W, PR, PW, DW), Bitcask & LevelDB backends |
| 4.1 | [`4.1 - Redis.md`](file:///C:/Dev/Github/Learn-Database/4.1%20-%20Redis.md) | In-Memory Store | In-memory data structures (Strings, Hashes, Lists, Sets, ZSets, Streams), persistence (RDB/AOF), Pub/Sub, transactions, evictions |
| 5 | [`5 - Neo4j.md`](file:///C:/Dev/Github/Learn-Database/5%20-%20Neo4j.md) | Graph Database | Labeled Property Graph, Cypher QL, ACID transactions, index-free adjacency, causal clustering |

---

## 3. Topics Covered per Database

- **Logical Hierarchy & Cluster Topologies:** Nodes, Rings, Shards, Replica Sets, Causal Clusters
- **CRUD Operations & Query Languages:** Mongo Shell, CQL, Cypher, Redis CLI, RESTful APIs
- **Document & Field Updates:** Adding/renaming fields, arithmetic updates, positional array filters
- **Aggregation Frameworks:** Multi-stage pipelines, array unwinding, cross-collection joins, faceted search
- **Indexing & Constraints:** Single, compound (ESR rule), multikey, geospatial (`2dsphere`), text, HNSW vector indexes
- **Storage Engines & Compaction:** LSM-Trees vs. B-Trees, WiredTiger, Bitcask, SSTables, Bloom Filters, Compaction (STCS/LCS/TWCS)
- **Consistency Models & Consensus:** Tunable per-query consistency, Vector Clocks, CRDTs, Paxos LWT, Raft
- **System Design & Integration:** Polyglot persistence, Saga pattern, CDC streaming, Flask/Django/Python drivers, YCSB benchmarking
