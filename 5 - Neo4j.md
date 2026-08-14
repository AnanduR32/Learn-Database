# Neo4j

- Leading graph database management system
- Written in Java, with Cypher as its query language
- Stores data as nodes, relationships, and properties, a property graph model
- Fully ACID compliant

Usecases:

- Social networks (User connections, friend-of-friend queries)
- Recommendation engines (Product affinity, content suggestions)
- Fraud detection (Circular transactions, unusual relationship patterns)
- Knowledge graphs (Entity resolution, semantic search)
- Network and IT operations (Infrastructure topology, impact analysis)

## Logical Hierarchy

Neo4j Instance -[1..n]-> Database -[1..n]-> Graph ->> (Node, Relationship, Property, Label)

- A database contains a single graph (Neo4j 4.x+ supports multiple databases)
- **Node**: Represents an entity (person, product, account), analogous to a document/row
- **Relationship**: Directed connection between two nodes with a type and direction
- **Property**: Key-value attributes on nodes and relationships
- **Label**: A tag or role assigned to nodes for grouping and indexing
- **Traversal**: Navigating the graph by following relationships

## Architecture

- **Native graph storage**: Nodes and relationships are stored as fixed-size records with direct pointers
- **Index-free adjacency**: Each node stores direct references to its relationships, traversals don't need global index lookups
- **Page cache**: Memory-mapped file cache for fast read/write
- **Transaction log**: Write-ahead log for crash recovery and replication

### Cluster modes (Neo4j 4.x+)

| Mode | Description |
|------|-------------|
| **Single instance** | Standalone server, full ACID. No replication. |
| **Causal Cluster** | Primary (leader) for writes, secondaries (read replicas) for reads. Automatic failover via Raft consensus. |
| **Read Replicas** | Scale read throughput; eventual consistency for reads. |

- Writes go to the **primary** (leader), which replicates via Raft
- Reads can go to any **secondary** with optional causal chaining (bookmarks) for read-your-writes consistency

## Cypher Query Language

Cypher is Neo4j's declarative graph query language. It uses ASCII-art patterns to express graph structures.

### Creating nodes

```cypher
// Create a node with label and properties
CREATE (n:Person { name: 'Alice', age: 30 })
```

```cypher
// Create multiple nodes
CREATE (a:Person { name: 'Alice' }), (b:Person { name: 'Bob' })
```

### Creating relationships

```cypher
// Create a relationship between existing nodes
MATCH (a:Person { name: 'Alice' })
MATCH (b:Person { name: 'Bob' })
CREATE (a)-[:KNOWS { since: 2024 }]->(b)
```

```cypher
// Create node + relationship in one go
CREATE (a:Person { name: 'Alice' })-[:KNOWS]->(b:Person { name: 'Bob' })
```

### Reading data (MATCH)

```cypher
// Match all nodes with a label
MATCH (n:Person) RETURN n

// Match with property filter
MATCH (n:Person { name: 'Alice' }) RETURN n

// Match with WHERE clause
MATCH (n:Person)
WHERE n.age > 25 AND n.name STARTS WITH 'A'
RETURN n

// Match relationships
MATCH (a:Person { name: 'Alice' })-[:KNOWS]->(friends:Person)
RETURN friends

// Variable-length path (friends-of-friends, up to 2 hops)
MATCH (a:Person { name: 'Alice' })-[:KNOWS*1..2]->(connections)
RETURN connections

// Optional match (like SQL LEFT JOIN)
MATCH (a:Person { name: 'Alice' })
OPTIONAL MATCH (a)-[:KNOWS]->(b:Person)
RETURN a, b

// Aggregation
MATCH (n:Person)
RETURN n.age, count(*) AS count
ORDER BY count DESC

// Limit and skip
MATCH (n:Person)
RETURN n
ORDER BY n.name
SKIP 10
LIMIT 5
```

### Updating data

```cypher
// Set properties
MATCH (n:Person { name: 'Alice' })
SET n.age = 31

// Remove a property
MATCH (n:Person { name: 'Alice' })
REMOVE n.age

// Add a label
MATCH (n:Person { name: 'Alice' })
SET n:Employee
```

### Deleting data

```cypher
// Delete a node (must have no relationships)
MATCH (n:Person { name: 'Alice' })
DELETE n

// Delete a node and all its relationships
MATCH (n:Person { name: 'Alice' })
DETACH DELETE n

// Delete a relationship only
MATCH (a:Person { name: 'Alice' })-[r:KNOWS]->(b:Person { name: 'Bob' })
DELETE r

// Delete all nodes and relationships (clear graph)
MATCH (n)
DETACH DELETE n
```

### MERGE (create if not exists)

```cypher
-- Find or create
MERGE (n:Person { name: 'Alice' })
ON CREATE SET n.created_at = timestamp()
ON MATCH SET n.last_seen = timestamp()
```

### Indexes

```cypher
// Create an index on a label + property (for equality lookups)
CREATE INDEX person_name_index FOR (n:Person) ON (n.name)

// Create a composite index
CREATE INDEX person_name_age_index FOR (n:Person) ON (n.name, n.age)

// Drop index
DROP INDEX person_name_index IF EXISTS
```

### Constraints

```cypher
// Uniqueness constraint (also creates an index)
CREATE CONSTRAINT unique_person_name FOR (n:Person) REQUIRE n.name IS UNIQUE

// Node property existence constraint (Neo4j 4.x+)
CREATE CONSTRAINT person_age_exists FOR (n:Person) REQUIRE n.age IS NOT NULL

// Relationship property existence constraint
CREATE CONSTRAINT knows_since_exists FOR ()-[r:KNOWS]-() REQUIRE r.since IS NOT NULL

// Drop constraint
DROP CONSTRAINT unique_person_name IF EXISTS
```

### Supported datatypes

- Integer
- Float
- String
- Boolean
- Byte array
- Point (spatial)
- Date / Time / LocalTime / DateTime / LocalDateTime / Duration
- List (ordered collection)
- Map (key-value)
- Node / Relationship (reference types)

### Filtering conditions

- **Equality**: `n.name = 'Alice'`
- **Comparison**: `n.age > 25`, `n.age >= 18`, `n.age < 65`, `n.age <= 65`
- **String matching**: `n.name STARTS WITH 'A'`, `n.name ENDS WITH 'z'`, `n.name CONTAINS 'li'`
- **Regular expression**: `n.name =~ 'Ali.*'`
- **IN**: `n.name IN ['Alice', 'Bob']`
- **IS NULL / IS NOT NULL**: `n.age IS NULL`, `n.age IS NOT NULL`
- **Range**: `n.age > 25 AND n.age < 50`
- **Logical**: `AND`, `OR`, `NOT`, `XOR`
- **List membership**: `'admin' IN n.roles`
- **Property existence**: `n.email IS NOT NULL` (or pattern existence `EXISTS { MATCH (n)-[:KNOWS]->() }`)

### Advanced pattern matching

```cypher
-- Shortest path
MATCH p = shortestPath((a:Person { name: 'Alice' })-[:KNOWS*]-(b:Person { name: 'Dave' }))
RETURN p

-- Find all paths (up to 5 hops)
MATCH p = (a:Person { name: 'Alice' })-[:KNOWS*1..5]-(b:Person { name: 'Dave' })
RETURN p

-- Pattern comprehension (return list of related nodes)
MATCH (a:Person { name: 'Alice' })
RETURN [(a)-[:KNOWS]->(friend) | friend.name] AS friends

-- Conditional path filtering
MATCH (a:Person { name: 'Alice' })
MATCH path = (a)-[*1..3]-(connected)
WHERE ALL(n in nodes(path) WHERE n.age > 18)
RETURN connected
```

## Transactions

Neo4j supports **full ACID transactions** (commit and rollback).

```cypher
:begin
  MATCH (a:Person { name: 'Alice' })
  SET a.balance = a.balance - 100;
  MATCH (b:Person { name: 'Bob' })
  SET b.balance = b.balance + 100;
:commit
```

```cypher
-- Rollback on error
:begin
  MATCH (a:Person { name: 'Alice' })
  SET a.balance = a.balance - 100;
  MATCH (b:Person { name: 'Bob' })
  SET b.balance = b.balance + 100;
:rollback
```

From application code (Java/JS):

```js
const session = driver.session();
const tx = session.beginTransaction();
try {
  await tx.run('MATCH (a:Person {name: $name}) SET a.balance = a.balance - $amt', { name: 'Alice', amt: 100 });
  await tx.run('MATCH (b:Person {name: $name}) SET b.balance = b.balance + $amt', { name: 'Bob', amt: 100 });
  await tx.commit();
} catch (e) {
  await tx.rollback();
}
```

## Storage Engine

Native graph storage with fixed-size records:

```
Write → Transaction Log (WAL on disk) → Page Cache (memory) → Data Files (disk, fixed-size records)
```

- **Record Store**: Nodes and relationships stored as fixed-length records (9 bytes for node, 14 bytes for relationship) with direct file offsets as pointers, no index required for adjacency traversal
- **Property Store**: Variable-length key-value pairs stored separately, referenced by records
- **Label Store**: Bit-set of labels per node, enables fast label scans
- **Page Cache**: Memory-mapped file cache (configurable, default 50% of available RAM)
- **Transaction Log (WAL)**: Append-only log for crash recovery. On commit, WAL is flushed to disk before applying to page cache.
- **Checkpoint**: Periodically, dirty pages in the page cache are flushed to the data files

> **Index-free adjacency**: A node's relationships are stored as a doubly-linked list with direct pointers to the target node. Traversal from Alice to her friends doesn't query an index, it follows pointers. This is what makes graph traversals O(1) per hop regardless of graph size.

## CQL limitations (Cypher vs SQL)

| Feature | Supported? | Notes |
|---------|-----------|-------|
| `AND` / `OR` / `NOT` | Yes | Full logical operator support |
| `JOIN` | No | Relationships replace joins |
| `GROUP BY` | Yes | Via aggregation with `WITH` clause |
| `HAVING` | Yes | After `WITH` + aggregation |
| Subqueries | Yes | Subqueries with `CALL { ... }` (Neo4j 4.x+) |
| `UNION` | Yes | `UNION` and `UNION ALL` supported |
| Aggregates | Yes | `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `COLLECT`, `STDEV` |
| `DISTINCT` | Yes | `RETURN DISTINCT` |
| `ORDER BY` | Yes | With `ASC` / `DESC` and `NULLS FIRST` / `NULLS LAST` |
| `LIMIT` / `SKIP` | Yes | Pagination support |
| `LIKE` | No | Use `CONTAINS`, `STARTS WITH`, `ENDS WITH`, or regex `=~` |
| `BETWEEN` | No | Use `>=` and `<=` |
| `IN` | Yes | List membership |
| `IS NULL` | Yes | `IS NULL` / `IS NOT NULL` |
| Schema | Schema-less | No rigid schema, labels and properties are flexible |
| ACID | Yes | Full ACID compliance. Unlike Cassandra/MongoDB, Neo4j supports commit and rollback. |
