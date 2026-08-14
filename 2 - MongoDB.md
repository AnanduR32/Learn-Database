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

MongoDB provides fine-grained update operators to modify specific fields, add new attributes, alter arrays, and perform atomic arithmetic without replacing the whole document.

#### 1. Adding a New Column / Field
Use `$set` with a field name that does not yet exist in the document:

```javascript
// Adds the 'is_verified' and 'joined_date' fields to a single user
db.users.updateOne(
  { _id: ObjectId("60c72b2f9b1d8b2badbee001") },
  { 
    $set: { 
      is_verified: true, 
      joined_date: new Date() 
    } 
  }
);

// Adds a default 'loyalty_tier' column to ALL documents in the collection
db.users.updateMany(
  { loyalty_tier: { $exists: false } },
  { $set: { loyalty_tier: "bronze" } }
);
```

#### 2. Updating a Singular Column / Field
Use `$set` with an existing field name to update only that specific value:

```javascript
// Updates only the 'email' and 'status' fields, preserving all other fields
db.users.updateOne(
  { _id: ObjectId("60c72b2f9b1d8b2badbee001") },
  { $set: { email: "alice.new@example.com", status: "active" } }
);

// Updating nested embedded document fields using dot notation (must be quoted)
db.users.updateOne(
  { _id: ObjectId("60c72b2f9b1d8b2badbee001") },
  { $set: { "profile.address.city": "San Francisco", "profile.settings.darkMode": true } }
);
```

#### 3. Removing / Dropping a Column (`$unset`)
To delete one or more fields from a document:

```javascript
// Deletes 'temp_token' and 'legacy_id' from the document
db.users.updateOne(
  { _id: ObjectId("60c72b2f9b1d8b2badbee001") },
  { $unset: { temp_token: "", legacy_id: "" } }
);
```

#### 4. Renaming a Column / Field (`$rename`)
```javascript
// Renames 'phone_no' to 'phone_number'
db.users.updateMany(
  {},
  { $rename: { "phone_no": "phone_number" } }
);
```

#### 5. Numeric & Arithmetic Updates (`$inc`, `$mul`, `$min`, `$max`)
```javascript
// Increment / Decrement
db.products.updateOne({ _id: 101 }, { $inc: { stock_count: -1, view_count: 1 } });

// Multiply field value (e.g. 10% discount)
db.products.updateOne({ _id: 101 }, { $mul: { price: 0.90 } });

// $min: Updates only if specified value is LESS than current field value
db.scores.updateOne({ _id: "user_1" }, { $min: { best_lap_time: 54.2 } });

// $max: Updates only if specified value is GREATER than current field value
db.scores.updateOne({ _id: "user_1" }, { $max: { high_score: 950 } });
```

#### 6. Array Updates (`$push`, `$addToSet`, `$pull`, `$pop`)
```javascript
// $push: Appends an element to an array (duplicates allowed)
db.posts.updateOne(
  { _id: 42 },
  { $push: { tags: "mongodb" } }
);

// $push with modifiers ($each, $slice, $sort): Keep only top 5 recent comments
db.posts.updateOne(
  { _id: 42 },
  { 
    $push: { 
      comments: { 
        $each: [{ user: "Bob", text: "Great post!", date: new Date() }],
        $sort: { date: -1 },
        $slice: 5 
      } 
    } 
  }
);

// $addToSet: Adds an element ONLY IF it doesn't already exist (Set behavior)
db.users.updateOne(
  { _id: "user_1" },
  { $addToSet: { roles: "editor" } }
);

// $pull: Removes all matching elements from an array
db.posts.updateOne(
  { _id: 42 },
  { $pull: { tags: "outdated_tag" } }
);

// $pop: Removes the first (-1) or last (1) element of an array
db.posts.updateOne(
  { _id: 42 },
  { $pop: { tags: 1 } }
);
```

#### 7. Positional Array Operators (`$`, `$[]`, `$[<identifier>]`)
```javascript
// 1. Positional operator ($): Updates the FIRST array element that matched the query condition
db.students.updateOne(
  { _id: 1, "grades.subject": "Math" },
  { $set: { "grades.$.score": 95 } }
);

// 2. All positional operator ($[]): Updates ALL elements in the array
db.students.updateOne(
  { _id: 1 },
  { $inc: { "grades.$[].bonus_points": 5 } }
);

// 3. Filtered positional operator ($[<identifier>] + arrayFilters):
// Updates only array elements satisfying a custom filter condition
db.students.updateOne(
  { _id: 1 },
  { $set: { "grades.$[elem].letter": "A+" } },
  { arrayFilters: [{ "elem.score": { $gte: 90 } }] }
);
```

#### 8. Upserts (`upsert: true` & `$setOnInsert`)
Performs an atomic update if the document exists, or creates it if absent:

```javascript
db.analytics.updateOne(
  { page_url: "/home", date: "2026-08-14" },
  {
    $inc: { visits: 1 },
    $setOnInsert: { first_tracked: new Date(), bounce_count: 0 } // Executed ONLY on creation
  },
  { upsert: true }
);
```

#### 9. Atomic Find and Modify (`findOneAndUpdate`, `findOneAndReplace`, `findOneAndDelete`)
Returns the modified or original document atomically (useful for distributed job queues / ticket reservations):

```javascript
const updatedTask = db.task_queue.findOneAndUpdate(
  { status: "pending" },
  { $set: { status: "processing", worker_id: "worker_01", started_at: new Date() } },
  { 
    sort: { priority: -1 }, 
    returnDocument: "after" // "after" returns updated document; "before" returns original
  }
);
```

#### 10. Document Replacement (`replaceOne`)
Completely replaces the whole document except for the immutable `_id` field:

```javascript
db.users.replaceOne(
  { _id: ObjectId("60c72b2f9b1d8b2badbee001") },
  { name: "Alice Cooper", email: "alice@example.com", status: "active" }
);
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

---

## MongoDB Aggregation Framework

The Aggregation Framework processes documents through an execution pipeline, transforming and summarizing collections into calculated results.

```
Input Collection ──► [$match] ──► [$group] ──► [$project] ──► [$sort] ──► Aggregated Output
```

### 1. Single-Purpose Aggregation Methods

For quick, simple counts or unique value extraction without building a full multi-stage pipeline:

```javascript
// 1. Estimated count (Fast O(1) read from collection metadata; approximate)
db.orders.estimatedDocumentCount();

// 2. Exact count with query criteria
db.orders.countDocuments({ status: "shipped", total: { $gte: 100 } });

// 3. Distinct values for a specific field
db.users.distinct("profile.address.country", { status: "active" });
```

### 2. Aggregation Pipeline Architecture & Nuances

- **Pipeline Concept**: Documents stream through stages sequentially. The output of stage $N$ becomes the input for stage $N+1$.
- **RAM Limits & Spilling to Disk**: Each stage is limited to **100 MB of RAM** by default. To process large datasets without out-of-memory errors, enable `allowDiskUse`:
  ```javascript
  db.orders.aggregate(pipeline, { allowDiskUse: true });
  ```
- **Index Optimization**:
  - A `$match` or `$sort` placed at the **very start** of the pipeline can utilize indexes (`IXSCAN`).
  - Once a `$group`, `$project`, or `$unwind` stage executes, subsequent stages process documents in-memory and cannot use collection indexes.

---

### 3. Core Pipeline Stages & Syntax

#### Stage 1: `$match` (Filter Documents)
Filters documents to pass only those matching criteria (equivalent to SQL `WHERE`).
```javascript
{ $match: { status: "completed", order_date: { $gte: ISODate("2026-01-01") } } }
```

#### Stage 2: `$project` (Reshape & Compute)
Selects, excludes, or computes new fields (equivalent to SQL `SELECT`).
```javascript
{ 
  $project: {
    _id: 0,
    order_id: "$_id",
    customer_name: 1,
    discounted_total: { $multiply: ["$total", 0.90] }
  }
}
```

#### Stage 3: `$group` (Aggregate by Key)
Groups documents by a specified identifier and computes aggregate metrics (equivalent to SQL `GROUP BY`).
- `_id: null` groups the entire collection into a single summary document.
- Accumulator operators: `$sum`, `$avg`, `$min`, `$max`, `$first`, `$last`, `$push`, `$addToSet`, `$count`.

```javascript
{
  $group: {
    _id: "$category",                         // Grouping Key
    total_revenue: { $sum: "$price" },         // Sum of price
    avg_rating: { $avg: "$rating" },          // Average rating
    product_count: { $count: {} },             // Number of items in group
    unique_brands: { $addToSet: "$brand" }    // Set of distinct brands
  }
}
```

#### Stage 4: `$sort`, `$limit`, and `$skip` (Ordering & Pagination)
```javascript
{ $sort: { total_revenue: -1, product_count: 1 } },
{ $skip: 10 },
{ $limit: 5 }
```

#### Stage 5: `$unwind` (Deconstruct Arrays)
Splits a document with an array of $n$ elements into $n$ separate documents:
```javascript
// Before unwind: { _id: 1, items: ["apple", "banana"] }
// After unwind:  { _id: 1, items: "apple" }, { _id: 1, items: "banana" }
{ 
  $unwind: { 
    path: "$items", 
    preserveNullAndEmptyArrays: true // Retains documents with empty/missing arrays
  } 
}
```

#### Stage 6: `$lookup` (Left Outer Join)
Performs an equality join between two collections:
```javascript
{
  $lookup: {
    from: "customers",            // Target collection to join
    localField: "customer_id",    // Field in the orders collection
    foreignField: "_id",          // Matching field in customers collection
    as: "customer_details"        // Result array field name
  }
}
```

*Uncorrelated Subquery Join (with inner pipeline)*:
```javascript
{
  $lookup: {
    from: "inventory",
    let: { order_item: "$item_sku" },
    pipeline: [
      { $match: { $expr: { $and: [{ $eq: ["$sku", "$$order_item"] }, { $gt: ["$stock", 0] }] } } }
    ],
    as: "available_inventory"
  }
}
```

#### Stage 7: `$addFields` / `$set` (Add Computed Attributes)
Appends new computed fields while retaining all existing document fields (unlike `$project` which excludes fields by default):
```javascript
{
  $addFields: {
    is_high_value: { $gt: ["$total_revenue", 10000] },
    report_generated: new Date()
  }
}
```

#### Stage 8: `$facet` (Multi-Faceted Categorization)
Executes multiple independent aggregation sub-pipelines in parallel on the same incoming stream:
```javascript
{
  $facet: {
    "price_breakdown": [
      { $bucket: { groupBy: "$price", boundaries: [0, 50, 200, 1000], default: "Premium" } }
    ],
    "top_categories": [
      { $group: { _id: "$category", count: { $sum: 1 } } },
      { $sort: { count: -1 } },
      { $limit: 3 }
    ]
  }
}
```

#### Stage 9: Output Stages (`$out` and `$merge`)
- `$out`: Writes the entire aggregation output to a new collection, replacing it completely.
  ```javascript
  { $out: "monthly_revenue_summary" }
  ```
- `$merge`: Incrementally updates or inserts aggregation results into an existing collection (on-demand materialized views).
  ```javascript
  { 
    $merge: { 
      into: "live_analytics", 
      on: "_id", 
      whenMatched: "replace", 
      whenNotMatched: "insert" 
    } 
  }
  ```

---

### 4. Comprehensive Aggregation Example

**Problem**: Find the top 3 customers who spent the most on completed electronics orders in 2026, joining customer details and calculating their average order value:

```javascript
db.orders.aggregate([
  // 1. Filter completed orders placed in 2026
  {
    $match: {
      status: "completed",
      order_date: {
        $gte: ISODate("2026-01-01T00:00:00Z"),
        $lt: ISODate("2027-01-01T00:00:00Z")
      }
    }
  },

  // 2. Unwind items array to inspect individual line items
  { $unwind: "$items" },

  // 3. Filter only electronics items
  { $match: { "items.category": "electronics" } },

  // 4. Group by customer to compute total spend and order count
  {
    $group: {
      _id: "$customer_id",
      total_spent: { $sum: { $multiply: ["$items.price", "$items.quantity"] } },
      items_bought: { $sum: "$items.quantity" },
      orders_count: { $addToSet: "$_id" }
    }
  },

  // 5. Join customer profile from 'users' collection
  {
    $lookup: {
      from: "users",
      localField: "_id",
      foreignField: "_id",
      as: "user_info"
    }
  },

  // 6. Flatten user info array
  { $unwind: "$user_info" },

  // 7. Shape the final output
  {
    $project: {
      _id: 0,
      customer_id: "$_id",
      name: "$user_info.name",
      email: "$user_info.email",
      total_spent: 1,
      items_bought: 1,
      order_count: { $size: "$orders_count" }
    }
  },

  // 8. Sort highest spenders first and limit to top 3
  { $sort: { total_spent: -1 } },
  { $limit: 3 }
]);
```

---

## Indexes & Query Optimization in MongoDB

### Index Types

| Index Type | Syntax | Best Use Case |
|---|---|---|
| **Single Field** | `db.coll.createIndex({ email: 1 })` | Direct equality and range lookups on 1 field |
| **Compound** | `db.coll.createIndex({ status: 1, date: -1 })` | Queries filtering on prefix subsets (Equality, Sort, Range - ESR rule) |
| **Multikey** | `db.coll.createIndex({ tags: 1 })` | Indexing array elements for `$in`, `$elemMatch` |
| **Text** | `db.coll.createIndex({ bio: "text" })` | Full-text search with tokenization (`$text: { $search: "term" }`) |
| **Geospatial (2dsphere)** | `db.coll.createIndex({ location: "2dsphere" })` | Earth-spherical proximity queries (`$near`, `$geoWithin`) |
| **Hashed** | `db.coll.createIndex({ user_id: "hashed" })` | Even data distribution across Sharded Clusters |
| **Wildcard** | `db.coll.createIndex({ "custom_attributes.$**": 1 })` | Dynamically indexing unknown arbitrary user-defined JSON keys |
| **Partial** | `db.coll.createIndex({ email: 1 }, { partialFilterExpression: { status: "active" } })` | Saves RAM by only indexing subset of active documents |

### Query Profiling with `explain("executionStats")`

Inspect query execution plans to verify if queries use index scans (`IXSCAN`) or full collection scans (`COLLSCAN`):

```javascript
db.orders.find({ status: "shipped", total: { $gt: 100 } }).explain("executionStats");
```

Key metrics to evaluate:
- `stage`: `IXSCAN` (Good, using index) vs `COLLSCAN` (Bad, scanning entire disk collection).
- `totalDocsExamined`: Total documents read from disk.
- `nReturned`: Number of matching documents returned.
- **Ideal Index Efficiency**: $\frac{\text{totalDocsExamined}}{\text{nReturned}} \approx 1$.

