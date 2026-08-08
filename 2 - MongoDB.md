# MongoDB

- BSON Document based NoSQL database.
- Cross-platform written in C++

Usecases:

- Web analytics (Flexible schema allows new metrics to be added on the fly)
- E-commerce applications (Product, Order details)
- Event logging (Events can be sharded on basis of name of application or event type)
- Content management systems

## Logical Hierarchy

MongoDb server Instance -[1..n]-> Database -[1..n]-> Collection -[1..n]-> Document (JSON/BSON)

- Server instance runs the core MongoDB daemon
- Each database has it's own set of files with specific permissions
- A collection is equivalent to a table
- Documents are not restricted to schema and can be different within same collection

## MongoDB cluster

Group of server instances working together to ensure data durability, availability, and scaling.

Available Configurations:

- Replica Set: high availability and redundancy
- Sharded Cluster: horizontal scale out, to spread the data across machine nodes

Components:

- Shards: holds data
- Config servers: Stores structural metadata to route requests to specifc Shards
- Mongos routers: Lightweight routing service that checks the config server and handles data retrival from the specifc Shard

Methods for sharding

> Sharding is generally done using a Shard Key present in every document

There are two primary methods to split data:

- Ranged sharding: Dividing data into continuous ranges based on shard key
- Hashed sharding: Computes MD5 hash of shard key value to distribute data uniformly without creating hotspots (As seen in ranged sharding were all new writes will be clubbed together)

### Read Concern / Write Concern

MongoDB offers tunable consistency per operation, similar to Cassandra's consistency levels.

**Read Concern** (controls data freshness):

| Level | Description |
|-------|-------------|
| `local` | Default. Returns the most recent data on the primary. Stale reads possible on secondaries. |
| `available` | Same as `local` but for sharded clusters; may return orphaned documents. |
| `majority` | Returns data that has been acknowledged by the majority of replica set members. Guarantees no rollback. |
| `linearizable` | Strongest. Ensures reads return the most recent write acknowledged by a majority. Slower. |
| `snapshot` | Returns data from a point-in-time snapshot (used with transactions). |

**Write Concern** (controls durability acknowledgment):

| Level | Description |
|-------|-------------|
| `w: 1` | Default. Acknowledged by the primary only. |
| `w: majority` | Acknowledged by majority of replica set members. |
| `w: <n>` | Acknowledged by n members. |
| `w: 0` | Fire-and-forget. No acknowledgment. Fastest, no durability guarantee. |
| `j: true` | Forces journal (WiredTiger) commit to disk before acknowledgment. |
| `wtimeout: <ms>` | Timeout if write concern is not met; operation fails without rollback. |

```shell
# Write with majority write concern
db.collection_name.insertOne(
  { name: "Alice" },
  { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
);

# Read with majority read concern
db.collection_name.find({ name: "Alice" }).readConcern("majority");
```

> `R + W > N` (MongoDB equivalent): Use `w: majority` writes and `readConcern("majority")` reads to ensure strong consistency.

### Transactions and Rollback

MongoDB supports **multi-document ACID transactions** (since 4.0 for replica sets, 4.2 for sharded clusters).

```shell
session = db.getMongo().startSession();
session.startTransaction();

try {
  session.getDatabase("db1").collection("accounts").updateOne(
    { _id: "alice" },
    { $inc: { balance: -100 } }
  );
  session.getDatabase("db1").collection("accounts").updateOne(
    { _id: "bob" },
    { $inc: { balance: 100 } }
  );
  session.commitTransaction();
} catch (e) {
  session.abortTransaction();  // Rollback, undoes all changes
}
```

**Rollback behavior outside transactions:**

- If a **replica set primary election** occurs and an old primary rejoins with un-replicated writes, those writes are **rolled back**
- Rolled-back data is written to a `rollback/` directory; an admin must manually apply it if needed
- Use `w: majority` write concern to minimise rollback risk

### Storage Engine (WiredTiger)

MongoDB uses **WiredTiger** as its default storage engine (since 3.2).

**Write path:**

```
Write → Journal (disk) → Cache (memory) → Periodic Checkpoint → Data Files (disk, B-tree)
```

- **Journal**: Write-ahead log for crash recovery. Commits every 100ms by default (configurable via `storage.journal.commitIntervalMs`, range 1–500ms).
- **Cache**: In-memory B-tree cache (default 50% of RAM - 1GB). Reads and writes go through this cache.
- **Checkpoint**: Every 60 seconds, the cache is synced to disk as a consistent snapshot.
- **Data Files**: B-tree structured on disk (not LSM). No tombstones, updates are in-place with journal protection.

**Compression:**

- Data files: snappy (default) or zlib/zstd
- Journal: snappy (default)

**Key differences from Cassandra LSM:**

- MongoDB updates **in-place** on checkpoint (no read amplification)
- No tombstones (deletes remove data immediately)
- No compaction required in the Cassandra sense, WiredTiger periodically reorganises B-tree pages

## MongoDB shell

### Creating database

Will only manifest once atleast one document is inserted

```shell
use database_name
```

### Check selected database

```shell
db
```

### List all available databases

```shell
show dbs
```

### Drop database

```shell
db.dropDatabase()
```

### Creating collections

```shell
db.createCollection("collection_name")
```

### Drop collection

```shell
db.collection_name.drop()
```

### List all collections in selected db

```shell
show collections
```

### Finding documents

```shell
db.collection_name.find({ ... })

# Pretty formatting
db.collection_name.find({ ... }).pretty()

# Limiting search
db.collection_name.find({ ... }).limit(limit_value:int)

# Skipping/offsetting search
db.collection_name.find({ ... }).skip(offset_value:int)

# Find one
db.collection_name.findOne({ ... })

# find() returns a cursor over all matching documents (no findMany in MongoDB shell)
db.collection_name.find({ ... })
```

### Inserting documents

Automatically creates the collection if not existing

```shell
# Deprecated
db.collection_name.insert({ ... })

# Insert one
db.collection_name.insertOne({ ... })

# Insert many
db.collection_name.insertMany([{ ... }, ...])

# Bulk write
db.collection_name.bulkWrite([{ ... }, ...])
```

### Updating documents

```shell
# Deprecated
db.collection_name.update({ ... }, { ... })

# Update one
db.collection_name.updateOne({ ... }, { $set: ... })

# Update many
db.collection_name.updateMany({ ... }, { $set: ... })
```

### Deleting documents

```shell
# Deprecated
db.collection_name.remove({ ... })

# Delete one
db.collection_name.deleteOne({ ... })

# Delete many
db.collection_name.deleteMany([{ ... }, ...])
```

### TTL (Time-To-Live)

Auto-delete documents after a specified time using a TTL index on a date field.

```shell
# Create TTL index: documents expire 86400 seconds (1 day) after createdAt
db.collection_name.createIndex({ createdAt: 1 }, { expireAfterSeconds: 86400 })
```

```shell
# Insert a document; it will auto-delete after 86400 seconds
db.collection_name.insertOne({ data: "temp", createdAt: new Date() })
```

> The TTL index runs every 60 seconds. Documents may persist briefly past expiry.

### Counter field

No dedicated counter type; use `$inc` to atomically increment/decrement numeric fields.

```shell
# Increment
db.collection_name.updateOne({ _id: docId }, { $inc: { view_count: 1 } })

# Decrement
db.collection_name.updateOne({ _id: docId }, { $inc: { view_count: -1 } })

# Increment by arbitrary value
db.collection_name.updateOne({ _id: docId }, { $inc: { score: 50 } })
```

### Supported datatypes for values

- String
- Integer
- Boolean
- Double
- Arrays
- Date
- Timestamp
- Object (embedding/nesting documents)
- Object ID
- Binary data
- Null
- Symbol
- Code (Javascript code)

### Filtering conditions

- $eq : equals  

    ```json
    { status: { $eq: "expired" } }
    ```

- $ne : not equal
  
    ```json
    { status: { $ne: "active" } }
    ```

- $or : or

    ```json
    { $or: [ { balance: 0 }, { status: "suspended" } ] }
    ```

- $and : and

    ```json
    { $and: [ { score: { $lt: 50 } }, { attempts: { $gt: 3 } } ] }
    ```

- $in : in

    ```json
    { category: { $in: ["trial", "guest", "banned"] } }
    ```

- $nin : not in

    ```json
    { role: { $nin: ["admin", "owner"] } }
    ```

- $gte or $gt : greater than or equals, or greater than

    ```json
    { age: { $gte: 65 } }
    ```

- $lte or $lt : lesser than or equals, or lesser than

    ```json
    { age: { $lt: 65 } }
    ```

- $not : inverts query

    ```json
    { price: { $not: { $gt: 100 } } }
    ```

- $exists : check existence

    ```json
    { legacy_id: { $exists: true } }
    ```

- $type : find specific BSON datatype

    ```json
    { phone: { $not: { $type : "long"} } }
    ```

> For nested document querying using '.' to drill in, keep query param in double quotes.
