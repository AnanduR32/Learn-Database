# NoSql

Follows BASE properties

- Working on basis of CAP theorm - Availability, Partition Tolerance - No consistency overhead as it sacrificed for availability and partition tolerance

## Categories

- Document Store - MongoDB, CouchDB
- Graph Database - Neo4J, OrientDB
- Key-value Stores/Database - Redis, Dynamo, Riak
- Columnar Database - BigTable, HBase, Cassandra

## Hierarchical overview

Data - inter-related -> Database - managed by -> DBMS

Types of DBMS system:

- Hierarchical
- Network
- RDBMS

### SQL (RDBMS)

Used for storing manipulating and retrieving data

Uses,

- Data Definition Language (DDL) - to create/modify schema
  - create, drop, alter, truncate
- Data Query Language (DQL) - to query/retrieve data
  - select
- Data Manipulation Language (DML) - to insert/modify data
  - insert, update, delete
- Data Control Language (DCL) - access grant and revoke
  - grant, revoke
- Transaction Control Language (TCL) - commit, rollback, savepoint
  - commit, rollback, savepoint

#### Components

- tuples/record
- attribute
- schema
- keys
- table

#### Constraints

- Not null, primary key, foreign key, default, check
- Can be applied during creation or alter existing
- named constraints can be created using 'CONSTRAINT' keyword

#### Normalization

Normalize data to organise data and reduce redundancy thereby optimizing distribution of data and thus query performance

Mainly to remove anomalies, and for good structured data.

Anomalies include:

- Insertion anomaly: Department data being part of the student table, the management can't insert department information without a student data to be inserted alongwith.
- Deletion anomaly: Deleting the last student's record, removes the information on the existance of the department too
- Update anomaly: If head of department information is to be changed, all the student records needs to be updated.

Types

- 1NF - The relation/field contains atomic values
- 2NF - All non-key attributes are functionally dependent on PK
- 3NF - No transitive dependency between attributes and PK
- 4NF - No multivalued dependency
- 5NF - No join dependency, joins should be lossless

#### Indexing

To speed up query (provides rapid random lookups)

- Clustered index - via Primary Key or Unique

- Creating index:
  
  ```sql
    CREATE TABLE TABLE_NAME (
      COLUMN_1 PRIMARY KEY,                           -- METHOD 1
      ...
      INDEX (COLUMN_2, COLUMN_3)                      -- METHOD 2
    )
    CREATE INDEX INDEX_NAME ON TABLE_NAME(COLUMN_4);  -- METHOD 3
  ```

- Dropping index

  ```sql
    ALTER TABLE TABLE_NAME DROP INDEX INDEX_NAME;
  ```

- Renaming index

  ```sql
    ALTER TABLE TABLE_NAME RENAME INDEX INDEX_NAME TO NEW_INDEX_NAME;
  ```

- Show indexes

  ```sql
    SHOW INDEX FROM TABLE_NAME;
  ```

#### Limitations

- Requires structure and schema in order to process data
- ACID properties - performance and coordination overhead in distributed environments
- Legacy relational engines were not designed natively for nested JSON/hierarchical document trees
- Distribution - sharding and cross-node replication face latency overhead to maintain strict serializability
  - In distributed data, cross-node joins are slow due to network overhead
- Additional complexity for developers to migrate schemas continuously

### NoSQL (Not only SQL)

- Non-relational, distributed-first databases
- Generally used in Big Data, real-time web applications, IoT, and high-throughput systems
- Allows storage of large volumes of structured, semi-structured (JSON/BSON), and unstructured data
- Employs dynamic/flexible schemas (schema-on-read rather than schema-on-write)
- Built to scale horizontally across commodity clusters out of the box
- Design philosophy favors denormalization (aggregates) over relational joins:
  - An aggregate/document embeds related information to allow single-node atomic reads/writes
  - Built-in partitioners, routers, and cluster coordinators manage data balancing automatically
- Eventual consistency accepted by default: Writes can be acknowledged quickly on replica subsets and propagated asynchronously, though modern systems provide tunable consistency levels.
- **Modern Nuance**: While early NoSQL avoided ACID entirely, modern NoSQL provides selective ACID transactions (e.g., MongoDB multi-document ACID transactions, Cassandra Lightweight Transactions via Paxos, Neo4j full ACID), pipeline joins (e.g., MongoDB `$lookup`), and schema validation rules (`$jsonSchema`).

#### CAP Theorem

CAP theorem highlights 3 main guarantees of distributed systems:

- **Consistency (C)**: Every read receives the most recent write or an error (Linearizability)
- **Availability (A)**: Every non-failing node returns a response for every request, without guaranteeing it contains the most recent write
- **Partition Tolerance (P)**: The system continues to operate despite arbitrary network message loss or delays between nodes

In practice, because physical networks cannot guarantee 100% reliable communication (**Partition Tolerance is mandatory**), distributed systems must trade off between Consistency and Availability during a partition:

- **CP (Consistency + Partition Tolerance)**: MongoDB (default primary writes), Apache HBase, Google Cloud Bigtable
- **AP (Availability + Partition Tolerance)**: Apache Cassandra (tunable), Riak KV, CouchDB, Amazon DynamoDB (default)
- **CA (Consistency + Availability)**: Traditional single-node RDBMS (MySQL, PostgreSQL, Oracle) operate in CA mode because there is no distributed network partition; however, across a distributed cluster, true CA is impossible when partitions occur.
- **PACELC Extension**: **I**f **P**artitioned $\to$ choose **A** or **C**; **E**lse $\to$ choose **L**atency or **C**onsistency.

**ACID properties**: Essential for transactional systems (e.g., banking ledgers)
- SQL: PostgreSQL, MySQL, Oracle, SQLite, SQL Server
- NoSQL with native ACID: Neo4j (graph ACID), MongoDB (multi-document ACID since 4.0/4.2), CouchDB

**BASE**: Architectural model for highly available NoSQL distributed databases
- **BA** (**B**asically **A**vailable): Distributed system remains operational for requests despite node failures
- **S** (**S**oft State): System state may change over time due to background replica convergence without explicit writes
- **E** (**E**ventually Consistent): In the absence of new mutations, all replicas converge to identical values over time

eg: Cassandra, Riak, Redis, DynamoDB, Couchbase

#### Types

##### Document Databases

A.k.a Document Store. It stores data as flexible JSON-like documents (JSON, BSON, XML).
eg: MongoDB, Firebase, ElasticSearch

Stores and retrieves data as key value pari, but value is a document, which is basic unit of data, which are grouped into collections.

Provides ability to index and query deep into documents, eg: Getting documents of all students who have cgpa > 8 or marks in a subject > 90

```js
db.students.find({
  "academics.semesters.subjects": {
    $elemMatch: { 
      "code": "MCA201",
       "marks": { 
        "$gt": 90 
      } 
    }
  }
})
```

Hierarchial overview:
Document Database -> Collections (Tables) -> Documents

eg:

```json
{
  "_id": "STU20260401",
  "name": "Anandu",
  "email": "anandu@example.com",
  "joined_year": 2025,
  "status": "active",
  
  "academics": {
    "current_cgpa": 8.2,
    "semesters": [
      {
        "semester_num": 1,
        "gpa": 8.2,
        "subjects": [
          { "code": "MCA101", "name": "Data Structures", "marks": 82 },
          { "code": "MCA102", "name": "Database Management Systems", "marks": 82 }
        ]
      }
    ]
  }
}
```

> 16MB limit per document in MongoDB.

General use-cases:

- Profile management - Different users have completely different profiles depending on their accounts, settings, preferences, and activity
- CMS systems - Handle diverse types of media, layouts, and revisions
- Blogging platforms - Same as previous
- real-time analytics - Real-time analytics involves capturing millions of incoming events (like user clicks, application logs, or temperature readings from IoT sensors) and calculating metrics on the fly.
- e-commerce applications - In a store selling various items, say laptops and shirts - A laptop has parameters like RAM, CPU, Storage, and Battery. A t-shirt has Size, Color, Fabric, and Gender.

MongoDb use-cases:

- Aadhar project uses mongoDB to store demographic and biometric data of 1.2billion+ Indians
- MetLife allows company to manage their customer, insured policy and transaction details
- eBay uses MongoDB to make search suggestion feature, to find relevant products in less time.

##### Key-Value databases

Stores data as simple key-value pairs for fast lookups.
eg: Redis, DynamoDB, Riak, Aerospike, Oracle NoSQL

Available operations:

- Put(key, value) - Insert or update a value for a key
- Get(key) - Retrieve the value associated with a key
- Delete(key) - Remove a key-value pair
- Update(key, value) - Modify an existing value
- Exists(key) - Check whether a key is present
- Increment/Decrement - Perform atomic counters on numeric values

Schema: **tablename:primary_key:attribute_name**:*value*
key:   tablename:primary_key:attribute_name
value: value

Datatypes supported:

- string: simple
- set: supports all set operations
- sorted set (zset) in which the unordered set is sorted by assigning every element in value to a numeric score.
- list: Linked list of strings, supports queue behaviour
- streams: append only log, to build message brokers and activity feeds, behaving similar to Kafka

General use-cases:

- Session management, user session details, preferences
- Caching
- Storing personal data on specific domain. eg: Premium customers having their shard key stored
- Product recommendations, storing personalized list for individual customers
- Customized ad delivery to users based on their profile
- Supports atomic operations like INCR and EXPIRE, it is the industry standard for counting API requests in real time to stop DDoS attacks or API abuse.
- Set property can be used to find mutual friends in social media platforms
- zSets property can be used for tracking leaderboards

Redis use-cases:

- Pinterest uses Redis heavily for graph data and list management. They use Redis to track user relationships (who follows whom) and boards using Redis data structures like Sets and Sorted Sets.
- X (Twitter) uses Redis to manage its timeline - When a user tweets, X doesn't dynamically fetch that tweet when their followers log in. Instead, they use a Fanout process. The system takes the new tweet and injects it directly into the in-memory Redis Lists of all the user's active followers. (List queues employed)

##### Column-Family / Wide-Column databases

Stores data in columns and rows optimized for massive scale, sparse matrices, and write-heavy telemetry.
eg: Apache Cassandra, ScyllaDB, Google Cloud Bigtable, Apache HBase, Azure Cosmos DB

Data is oriented around column families rather than rigid tabular rows:
  In wide-column stores, each row key maps to an arbitrary number of dynamic, timestamped columns. Space is only allocated for populated values.

Best for high-throughput write workloads and time-series analytics (in contrast to traditional row-oriented RDBMS designed for localized transactional CRUD).

Key mechanisms:

- Has better query performance for targeted columns, avoiding scanning unrequested attributes across wide records.
- Is better at compression: columns of uniform data types can use algorithms such as Run-Length Encoding (RLE) or Dictionary Encoding, saving massive disk space.
- Employs Log-Structured Merge (LSM) trees for append-only sequential writes to disk, delivering lightning-fast write performance.

Hierarchical representation:  

```json
 Database (Cluster)
    |  
 Keyspace     : Outermost container of application data, defining replication factor and strategy  
    |  
 Table / Column family : Stored together on disk (SSTables)  
    |  
   Rows       : Each identified by a unique Partition Key (determines node location) and optional Clustering Keys (determines on-disk sorting)  
    |  
   Columns    : Smallest unit of data (Name, Value, Timestamp)
```

eg:  

```json
[Column Family: UserProfiles]
  |
  +---> [Row Key: STU_101] 
  |       |
  |       +---> Column 1: { Name: "name",    Value: "Anandu",   Timestamp: 17180000 }
  |       +---> Column 2: { Name: "role",    Value: "Dev",      Timestamp: 17180005 }
  |       +---> Column 3: { Name: "skills",  Value: ".NET, C#", Timestamp: 17180010 }
  |
  +---> [Row Key: STU_102]
          |
          +---> Column 1: { Name: "name",    Value: "Sarah",    Timestamp: 17180020 }
          +---> Column 2: { Name: "country", Value: "USA",      Timestamp: 17180025 }
          // Notice: No "role" or "skills" columns here. Entirely different column count!
```

**Cassandra use cases:**

- Spotify: User playlists, listening history, and recommendation metadata.
- Netflix: Global viewing history and bookmark state synchronized across multi-region datacenters.
- IoT & Telemetry: Sensor time-series telemetry streams using composite time-window keys.

##### Graph databases

Stores data as connected nodes, relationships (edges), and properties, implementing the **Labeled Property Graph** model.
eg: Neo4j, Amazon Neptune, Memgraph, OrientDB

Key mechanisms:

- **Index-Free Adjacency**: Each node stores direct memory pointers to adjacent relationships. Traversing relationships is $O(1)$ constant time per hop, completely independent of the total graph size.
- **Declarative Graph Queries**: Cypher query language uses visual ASCII-art pattern matching `(a)-[:RELATION]->(b)` to express complex graph traversals.
- **ACID Compliance**: Full ACID transaction support ensures strict consistency during relationship and property mutations.

General use cases:

- Social Networks: Friend-of-a-friend discovery, mutual connections, follower graph.
- Recommendation Engines: Collaborative filtering based on shared interests and purchase paths.
- Fraud Detection: Uncovering circular transactions, synthetic identity rings, and shared financial attributes.
- Knowledge Graphs & GraphRAG: Semantic entity networks for AI retrieval-augmented generation.
