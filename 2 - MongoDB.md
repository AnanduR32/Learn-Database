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

# Find many
db.collection_name.findMany({ ... })
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
db.collection_name.BulkWrite([{ ... }, ...])
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
    { status: { $eq: "expired" } }
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
