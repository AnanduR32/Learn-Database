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
    CREATE INDEX INDEX_NAME ON TABLE_NAME(COLUME_4);  -- METHOD 3
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

- Requires structure and schema inorder to process data
- ACID properties - performance overhead
- Majority of applications use JSON format for data, which is unsupported
- Distribution - sharding/replicas is slow in response due to the ACID compliance overhead
  - In distributed data, joins are slow due to network overhead
- Additonal complexity for developers to setup

### NoSQL (Not only SQL)

- Non-relational databases
- Generally used in Big Data and realtime web applications
- Allows storage of large volumes of structured, semi-structured and unstructured data
- Allows using flexible schema
- Build to be distributed out of the box
- No concept of constraints and joins, ACID compliance, Group by etc.
  - A document contains all information
  - The built-in routers and config servers maintains, splits, and balances the data across clusters/nodes
- Eventual consistency accepted: Instead of forcing every server to update simultaneously before confirming a write, NoSQL databases often accept the write immediately on one node and replicate it to others asynchronously.

#### CAP Theorem

CAP theorem highlights 3 main aspects of modern distributed systems

- Consistency (C): Every read gets the same data from most recent write
- Availability (A): Every request gets a response, even if it is not the latest data
- Partition Tolerance (P): The system keeps working even if some nodes cannot communicate

In practice, a distributed system can provide only 2 of the 3 guarantees at the same time.

- CA: SQLServer, MySQL, Oracle, MariaDB
- CP: HBase, MongoDB, Big table, Memcache
- AP: Riak, Cassandra, CouchDB

NoSQL databases usually prioritize Availability and Partition Tolerance, accepting eventual consistency.

**ACID properties**: Mandatorily used in FinTech systems

- SQL: MySQL, PostGreSQL, Oracle, SQLite, SQL Server
- NoSQL: Apache CouchDB, IBM DB2

**BASE**: For NoSQL distributed databases

- **BA** (**B**asically **A**vailable): Respond to every request
- **S** (**S**oft State): Data may change over time without requiring additional writes
- **E** (**E**ventually Consistent): The system converges to a consistent state after some time, rather than immediately

eg: MongoDB, Cassandra, Redis, DynamoDB, Couchbase

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
- X (Twitter) uees to manage it's timeline - When a user tweets, X doesn't dynamically fetch that tweet when their followers log in. Instead, they use a Fanout process. The system takes the new tweet and injects it directly into the in-memory Redis Lists of all the user's active followers. (List queues employed)

##### column databases

Stores data in columns and rows optimized for large-scale analytics.
eg: BigTable, Cassandra, Hbase, CosmoDB

Data is oriented in columns instead of rows  
  In column-oriented DBs, individual attributes (fields) are grouped together across all records, rather than keeping full individual records intact together on the disk.

Best for OLAP (in contrast to row-oriented traditional SQL databases which are best for OLTP)

Key mechanisms:

- Has better query performance as the database engine needs to only load the particular columns on which the query is being performed, whereas in row oriented DBs the entire row with all the columns need to be loaded. Thus column databases have better I/O speeds
- Is better at compression, in column DBs the column contains the same datatype and are stored together. The database engine can use algorithms such as Run-Length Encoding or Dictionary Encoding to compress data heavily - saving massive amount of storage space.
- Aggregation functions run at much higher speeds since the columns can be passed in chunks and executed using SIMD (Single Instruction Multiple Data) processing pipelines

Hierarchial representation:  

```json
 Database
    |  
 Keyspace     : outermost container of application data, grouping related data models together  
    |  
Column family : each column family is stored together on disk  
    |  
   Rows       : Each identified by unique row key or partition key  
    |  
  Column      : Each row can have different column count
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

**Cassandra usecases:**

- Spotify data storage, to store metadata on artist, songs user profile parameters.

##### Graph databases

Stores data as connected nodes and relationships.
eg: Neo4j, OrientDB
