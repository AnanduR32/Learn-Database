# Unit V — Polyglot Persistence, Vector Databases & Advanced Distributed Systems
### 25CSA642A — NoSQL Databases

This unit explores advanced enterprise database architectures: the **Polyglot Persistence** pattern, distributed transaction management via the **Saga Pattern**, **Change Data Capture (CDC)**, modern **Vector Databases & AI Semantic Search** (MongoDB Atlas Vector Search, Redis VSS, Neo4j GraphRAG), database benchmarking with **YCSB (Yahoo! Cloud Serving Benchmark)**, and zero-downtime migration strategies.

**Roadmap of this file:**
1. Polyglot Persistence & Database-per-Service Architecture
2. Distributed Transactions Across NoSQL — Saga Pattern vs. 2PC
3. Change Data Capture (CDC) & Real-Time Event Streaming
4. Vector Databases & AI Semantic Search (Embeddings, HNSW, MongoDB & Redis Vector Search)
5. GraphRAG — Integrating Knowledge Graphs with Vector Embeddings
6. Database Benchmarking with YCSB (Yahoo! Cloud Serving Benchmark)
7. Zero-Downtime Data Migration & Polyglot Synchronization

---

## 1. Polyglot Persistence & Database-per-Service Architecture

### 1.1 Core Concepts (Simplified)
- **Polyglot Persistence:** The practice of using multiple specialized database technologies within the same enterprise architecture, matching each microservice with the storage engine best suited for its data access pattern.
- **Single Model Anti-Pattern:** Forcing a single database (e.g., traditional SQL) to handle full-text search, high-volume sensor telemetry, graph traversals, and in-memory caches leads to performance bottlenecks and operational fragility.

### 1.2 Enterprise Architecture Blueprint

```
                            ┌──────────────────────────────────────────────┐
                            │               API Gateway                    │
                            └──────┬────────────┬────────────┬─────────────┘
                                   │            │            │
             ┌─────────────────────┼────────────┼────────────┼────────────────────┐
             ▼                     ▼            ▼            ▼                    ▼
     [Session Service]     [Catalog Service] [IoT Service] [Social/Fraud Svc] [Order/Billing Svc]
             │                     │            │            │                    │
             ▼                     ▼            ▼            ▼                    ▼
      ┌─────────────┐       ┌─────────────┐┌────────────┐┌──────────────┐   ┌────────────┐
      │ Redis (RAM) │       │ MongoDB     ││ Cassandra  ││ Neo4j (Graph)│   │ PostgreSQL │
      │ Key-Value   │       │ Document    ││ Wide-Column││ Property     │   │ RDBMS ACID │
      └─────────────┘       └─────────────┘└────────────┘└──────────────┘   └────────────┘
```

| Microservice | Selected Database | Primary Justification |
|---|---|---|
| **Session & Auth** | **Redis** | Sub-millisecond read/write latency, automatic TTL token expiration |
| **Product Catalog & CMS** | **MongoDB** | Dynamic, polymorphic document schemas and deep attribute indexing |
| **Telemetry & Time-Series** | **Cassandra** | High-throughput linear write scalability with TWCS compaction |
| **Fraud & Recommendations** | **Neo4j** | $O(1)$ index-free adjacency for multi-hop graph relationship traversals |
| **Billing & Financial Ledger** | **PostgreSQL** | Strict ACID serializability, multi-table foreign keys, and regulatory audits |

---

## 2. Distributed Transactions Across NoSQL — Saga Pattern vs. 2PC

### 2.1 The Distributed Transaction Problem
Relational databases enforce cross-table integrity using Two-Phase Commit (2PC) or distributed locking. In distributed NoSQL microservices spanning distinct physical databases, 2PC creates blocking bottlenecks that violate the CAP theorem.

### 2.2 The Saga Pattern
A **Saga** is a sequence of local transactions across independent databases. Each transaction updates data within a single service; if a step fails, the Saga executes **Compensating Transactions** to undo preceding steps.

```
Happy Path:
  [Order Svc: Create Pending] ──► [Payment Svc: Charge Card] ──► [Inventory Svc: Reserve Item] ──► [Order Svc: Complete]

Failure & Compensation:
  [Order Svc: Create Pending] ──► [Payment Svc: Charge Card] ──► [Inventory Svc: Out of Stock ❌]
                                           │                                    │
                                           ▼                                    │
                                 [Compensate: Refund Card] ◄────────────────────┘
                                           │
                                           ▼
                                 [Compensate: Cancel Order]
```

### 2.3 Implementation Flavors:
1. **Choreography (Event-Driven):** Services publish domain events to a message broker (e.g., Apache Kafka / RabbitMQ); other services listen and act autonomously.
2. **Orchestration (Centralized):** A dedicated Saga Orchestrator coordinates transaction execution and dispatches explicit compensation commands on failure.

---

## 3. Change Data Capture (CDC) & Real-Time Event Streaming

### 3.1 Core Concepts (Simplified)
- **Change Data Capture (CDC):** Asynchronously extracting row-level or document-level mutations directly from database transaction logs (e.g., MongoDB Oplog, Cassandra CommitLog) and publishing them to an event stream.
- **Benefits:** Avoids double-writing from application code and prevents database polling.

### 3.2 CDC Architecture with Debezium & Kafka
```
[MongoDB / PostgreSQL] ──► [Debezium Connector] ──► [Kafka Topic] ──► [Elasticsearch / Vector DB]
  (Writes to OpLog)         (Tail log non-blockingly)   (Event stream)    (Synchronized Search Index)
```

```python
# Consuming MongoDB Change Stream natively in Python
with db.orders.watch() as stream:
    for change in stream:
        operation = change["operationType"] # 'insert', 'update', 'replace'
        document = change.get("fullDocument")
        print(f"CDC Event: {operation} on Order ID: {document['_id']}")
```

---

## 4. Vector Databases & AI Semantic Search

### 4.1 Core Concepts (Simplified)
- **Vector Embeddings:** Dense numerical arrays representing semantic meanings generated by machine learning models (e.g., `text-embedding-3-small` $\to 1536$-dimensional vector).
- **Semantic Search:** Locating data points by conceptual meaning rather than keyword matching.

### 4.2 Distance Metrics

| Metric | Formula | Best Use Case |
|---|---|---|
| **Cosine Similarity** | $\cos(\theta) = \frac{\mathbf{A} \cdot \mathbf{B}}{\|\mathbf{A}\| \|\mathbf{B}\|}$ | Text & document semantics (normalized for text length) |
| **Dot Product** | $\mathbf{A} \cdot \mathbf{B} = \sum_{i=1}^n A_i B_i$ | Normalized model embeddings (fastest computation) |
| **Euclidean Distance ($L_2$)** | $d(\mathbf{A}, \mathbf{B}) = \sqrt{\sum_{i=1}^n (A_i - B_i)^2}$ | Physical geospatial features, image similarity |

### 4.3 Approximate Nearest Neighbor (ANN) Indexing — HNSW vs. IVF
- **Flat Index (Exact Search):** Calculates distance against every vector in the dataset ($O(N)$ brute-force; 100% recall, too slow for scale).
- **HNSW (Hierarchical Navigable Small World):** Multi-layered proximity graph where top layers take large spatial hops and bottom layers refine to local neighbors ($O(\log N)$ search time).
- **IVF (Inverted File Index):** Clusters vector space into Voronoi cells, only searching vectors within the closest centroids.

### 4.4 Vector Search in MongoDB & Redis

#### MongoDB Atlas Vector Search Pipeline:
```python
# Querying via MongoDB Aggregation Pipeline with $vectorSearch
pipeline = [
    {
        "$vectorSearch": {
            "index": "vector_index",
            "path": "embedding",
            "queryVector": user_query_embedding, # 1536-dim array
            "numCandidates": 100,
            "limit": 5
        }
    },
    {
        "$project": {
            "_id": 1,
            "title": 1,
            "score": {"$meta": "vectorSearchScore"}
        }
    }
]
results = list(db.articles.aggregate(pipeline))
```

#### Redis Vector Search (RediSearch / Redis Stack):
```shell
# 1. Create HNSW Vector Index
FT.CREATE idx:books ON HASH PREFIX 1 book: SCHEMA title TEXT embedding VECTOR HNSW 6 TYPE FLOAT32 DIM 1536 DISTANCE_METRIC COSINE

# 2. Query KNN (K-Nearest Neighbors)
FT.SEARCH idx:books "*=>[KNN 5 @embedding $vec AS score]" PARAMS 2 vec "\x01\x02..." SORTBY score DIALECT 2
```

---

## 5. GraphRAG — Integrating Knowledge Graphs with Vector Embeddings

### 5.1 The Hallucination Problem in Pure Vector Search
Pure vector search retrieves isolated text chunks that are semantically similar, but lacks structured multi-hop reasoning (e.g., understanding that *Company A* acquired *Company B*, which owns *Patent C*).

### 5.2 The GraphRAG Solution
Combines the semantic power of **Vector Embeddings** with the explicit topological connectivity of **Neo4j Knowledge Graphs**:
1. Vector search identifies relevant starting entity nodes.
2. Cypher graph traversal walks multi-hop relationships to gather connected context.
3. The enriched factual context is supplied to the Large Language Model (LLM) prompt, eliminating hallucinations.

---

## 6. Database Benchmarking with YCSB (Yahoo! Cloud Serving Benchmark)

### 6.1 Core Concepts (Simplified)
- **YCSB:** The industry-standard benchmarking framework used to evaluate performance, throughput, and latency of NoSQL databases.

### 6.2 Standard YCSB Workloads

| Workload | Read/Write Ratio | Access Pattern | Application Example |
|---|---|---|---|
| **Workload A (Update Heavy)** | 50% Read / 50% Update | Zipfian (Power-law distribution) | Session store, live order tracking |
| **Workload B (Read Heavy)** | 95% Read / 5% Update | Zipfian | Photo tagging, product reviews |
| **Workload C (Read Only)** | 100% Read | Zipfian | User profile cache |
| **Workload D (Read Latest)** | 95% Read / 5% Insert | Latest inserted items | Activity stream, status updates |
| **Workload E (Short Ranges)** | 95% Scan / 5% Insert | Uniform range scans | Threaded conversations, logs |
| **Workload F (Read-Modify-Write)** | 50% Read / 50% R-M-W | Zipfian | User activity counters, banking |

---

## 7. Zero-Downtime Data Migration & Polyglot Synchronization

### 7.1 The Dual-Write Migration Strategy
When migrating from an existing SQL database to a distributed NoSQL engine without downtime:

```
Step 1: Deploy Dual-Write Code (Write to SQL primary + NoSQL shadow)
Step 2: Run Offline Historical Backfill (ETL historical records to NoSQL)
Step 3: Verify Data Consistency & Parity via Reconciliation Scripts
Step 4: Switch Reads to NoSQL (Monitor latency and error rates)
Step 5: Deprecate Writes to SQL (Decommission legacy database)
```

---

## Practice Problems & Solutions

1. **Transaction Pattern Choice:** An e-commerce platform processes orders across a MongoDB catalog, a Cassandra clickstream log, and a PostgreSQL payment gateway. Why is Two-Phase Commit (2PC) ill-suited for this setup, and what should be used instead?
   * *Solution:* 2PC requires distributed blocking locks across all databases. A network hiccup or slow node in Cassandra will hold locks on PostgreSQL, destroying system throughput and availability (violating CAP). The **Saga Pattern** with compensating transactions and asynchronous message brokers should be used.
2. **Vector Index Selection:** A real-time recommendation service requires searching 10 million vector embeddings in under 5 milliseconds. Why should HNSW be selected over Flat Brute-Force search?
   * *Solution:* Flat search calculates cosine distance against all 10 million vectors, costing $O(N)$ time (~100-500ms). HNSW constructs a multi-layer graph, traversing it in $O(\log N)$ time (~2-5ms) with high recall accuracy ($> 98\%$).

---

## Unit V Summary Cheat-Sheet

| Concept | Key Takeaway |
|---|---|
| **Polyglot Persistence** | Use the optimal database model per microservice access pattern. |
| **Saga Pattern** | Sequences local transactions across distributed databases with compensating rollbacks. |
| **Change Data Capture** | Streams database mutation logs asynchronously via Debezium/Kafka without app double-writes. |
| **Vector Embeddings** | Numerical representations enabling semantic similarity search via Cosine/Dot Product metrics. |
| **HNSW Index** | Hierarchical navigable small-world graphs providing $O(\log N)$ vector search latency. |
| **YCSB Benchmarking** | Evaluates NoSQL throughput and p95/p99 latency against standard workloads A–F. |
