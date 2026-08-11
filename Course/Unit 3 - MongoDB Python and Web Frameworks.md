# Unit III — Advanced MongoDB Application Development with Python & Web Frameworks
### 25CSA642A — NoSQL Databases

This unit moves from MongoDB concepts to hands-on software development: connecting via PyMongo, running CRUD operations and operators, managing B-tree and Geospatial indexes, executing atomic upserts, building aggregation pipelines, understanding replication/sharding, and integrating MongoDB with Flask and Django.

**Roadmap of this file:**
1. Connecting to MongoDB with Python (PyMongo)
2. MongoDB Query Language — Find, Update, Delete Operations
3. MongoDB Query Operators
4. Using Indexes with MongoDB
5. GeoSpatial Indexing
6. Upserts in MongoDB
7. Aggregation Framework, Replication, and Sharding
8. Document Database with Web Frameworks — Django and MongoDB
9. Document Database with Web Frameworks — Flask and MongoDB

---

## 1. Connecting to MongoDB with Python (PyMongo)

### 1.1 Core Concepts (Simplified)
- **PyMongo:** Official Python driver for MongoDB.
- **`MongoClient` Object:** Manages database connection pooling and translates Python dictionaries to/from BSON (Binary JSON).
- **Connection Pooling Best Practice:** Instantiate `MongoClient` **once** at application startup, not inside individual HTTP request handlers or loops.

### 1.2 Proof — Connection Pooling Amortization
Establishing a new network connection requires TCP and TLS handshakes, costing a fixed setup time $c$.
- **Without Pooling:** Executing $m$ operations opens $m$ connections, paying setup cost $m \times c$.
- **With Pooling:** A shared pool of size $k$ pays the setup cost $k \times c$ once. Subsequent operations reuse open sockets at $O(1)$ setup cost. As $m \to \infty$, amortized connection overhead per operation approaches $0$.

### 1.3 Worked Code Example
```python
from pymongo import MongoClient

# Create a single client connection pool once at startup
client = MongoClient("mongodb://localhost:27017/")
db = client["ecommerce"]
orders = db["orders"]

# Operations reuse open pooled connections automatically
shipped_order = orders.find_one({"status": "shipped"})
print(shipped_order)
```

---

## 2. MongoDB Query Language — Find, Update, Delete Operations

### 2.1 Core Concepts (Simplified)
- **CRUD Operations:**
  - **Create:** `insert_one()`, `insert_many()`
  - **Read:** `find_one()`, `find()`
  - **Update:** `update_one()`, `update_many()` with `$set`
  - **Delete:** `delete_one()`, `delete_many()`
- **Partial Updates (`$set`):** Modifies only the specified fields, leaving all other fields intact.

### 2.2 Proof — Commutativity of Disjoint Field Updates
If two users simultaneously update different fields on the same document:
- User 1 applies `{"$set": {"status": "shipped"}}`
- User 2 applies `{"$set": {"tracking_code": "TRK123"}}`
Because MongoDB applies updates at the BSON field level, these operations commute (apply in any order with identical results) without overwriting each other's changes.

### 2.3 Worked Code Example
```python
# Partial Update: Only updates 'status', leaves all other fields intact
orders.update_one({"_id": 1042}, {"$set": {"status": "shipped"}})

# Bulk Delete: Removes all cancelled orders older than cutoff date
orders.delete_many({"status": "cancelled", "createdAt": {"$lt": cutoff_date}})
```

---

## 3. MongoDB Query Operators

### 3.1 Core Concepts (Simplified)
- **Comparison Operators:** `$eq`, `$ne`, `$gt` (>), `$gte` ($\ge$), `$lt` (<), `$lte` ($\le$), `$in`, `$nin`.
- **Logical Operators:** `$and`, `$or`, `$not`, `$nor`.
- **Element Operators:** `$exists`, `$type`.
- **Array Operators:** `$elemMatch` (checks if a single array element matches all criteria), `$all`, `$size`.

### 3.2 Worked Code Example
```python
# Find shipped orders with total > 100, excluding blocked customers
orders.find({
    "status": "shipped",
    "total": {"$gt": 100},
    "customerId": {"$nin": blocked_customer_ids}
})

# Array query: Find orders containing at least one item with qty >= 5
orders.find({"items": {"$elemMatch": {"qty": {"$gte": 5}}}})
```

---

## 4. Using Indexes with MongoDB

### 4.1 Core Concepts (Simplified)
- **Indexes:** B-tree structures built on fields to avoid scanning every document ($O(n) \to O(\log n)$).
- **Index Types:** Single-Field, Compound, Multikey (for array fields), Text, and Hashed (for sharding).
- **The Compound Index Prefix Rule:** An index on `(A, B, C)` can serve queries filtering on:
  - `(A)`
  - `(A, B)`
  - `(A, B, C)`
  - ❌ It **cannot** efficiently serve queries filtering on `(B)` or `(C)` alone because the index is sorted by `A` first.

### 4.2 Proof — Compound Index Prefix Rule
Compound indexes sort entries lexicographically (like a phonebook sorted by `Last_Name, First_Name`).
- Searching by `Last_Name` narrows entries to a contiguous section.
- Searching by `First_Name` alone forces a scan of the entire phonebook because `First_Name` values are scattered across every `Last_Name` section.

### 4.3 Worked Code Example
```python
# Create a compound index on status (ascending) and createdAt (descending)
orders.create_index([("status", 1), ("createdAt", -1)])

# Uses Index (matches prefix 'status')
orders.find({"status": "shipped"}) 

# DOES NOT use index efficiently (skips prefix 'status')
orders.find({"createdAt": {"$gte": start_date}}) 
```

---

## 5. GeoSpatial Indexing

### 5.1 Core Concepts (Simplified)
- **`2dsphere` Index:** Supports spherical map coordinates (GeoJSON points, lines, polygons).
- **Geohashing:** Converts 2D latitude/longitude coordinates into a 1D string by recursively dividing map areas into quadtree grid cells.
- **Proximity Locality:** Nearby physical locations get identical or similar geohash prefixes, allowing standard B-trees to perform fast radius searches (`$near`, `$geoWithin`).

### 5.2 Worked Code Example
```python
# Create geospatial index on 'location' field
stores.create_index([("location", "2dsphere")])

# Find stores within 5,000 meters (5 km) of a point
stores.find({
    "location": {
        "$near": {
            "$geometry": {"type": "Point", "coordinates": [-73.98, 40.75]},
            "$maxDistance": 5000
        }
    }
})
```

---

## 6. Upserts in MongoDB

### 6.1 Core Concepts (Simplified)
- **Upsert (`upsert=True`):** Updates a document if it exists, or inserts a new document if it does not.
- **Atomic Safety:** Combines check-and-insert into a single atomic database operation, eliminating race conditions (Check-Then-Act bugs) under concurrent web requests.

### 6.2 Worked Code Example
```python
orders.update_one(
    {"externalOrderId": "EXT-9981"},
    {
        "$set": {"status": "shipped"},
        "$setOnInsert": {"createdAt": current_time}  # Only set on insert
    },
    upsert=True
)
```

---

## 7. Aggregation Framework, Replication, and Sharding

### 7.1 Core Concepts (Simplified)
- **Aggregation Pipeline:** Passes documents through sequential transformation stages:
  - `$match`: Filters documents (like `WHERE`).
  - `$group`: Aggregates values by key (like `GROUP BY`).
  - `$project`: Reshapes output fields (like `SELECT`).
  - `$sort`: Orders results.
  - `$lookup`: Performs left-outer-joins between collections.
- **Replication (Replica Set):** 1 Primary node (handles writes) + multiple Secondary nodes (replicate data for high availability and read scaling).
- **Sharding:** Distributes large collections across multiple server shards using a **Shard Key** (often hashed to prevent write bottlenecks on a single shard).

### 7.2 Worked Code Example (Aggregation Pipeline)
Calculate total revenue by category, sorted highest first:
```python
pipeline = [
    {"$match": {"createdAt": {"$gte": start_date}}},
    {"$group": {"_id": "$category", "total_sales": {"$sum": "$amount"}}},
    {"$sort": {"total_sales": -1}}
]
results = list(orders.aggregate(pipeline))
```

---

## 8. Document Database with Web Frameworks — Django & MongoDB

### 8.1 Core Concepts (Simplified)
- Django's default ORM is built for SQL databases (tables, foreign keys).
- Connecting Django to MongoDB uses an **Object-Document Mapper (ODM)** like MongoEngine, requiring developers to choose between:
  1. **Embedding Documents:** Fast single-query reads, but shares document lifecycle.
  2. **Referencing Documents:** Independent document lifecycles, but requires application-level query joins.

### 8.2 Worked Code Example (MongoEngine)
```python
from mongoengine import Document, EmbeddedDocument, StringField, ListField, EmbeddedDocumentField

class LineItem(EmbeddedDocument):
    sku = StringField()
    qty = StringField()

class Order(Document):
    customer = StringField()
    items = ListField(EmbeddedDocumentField(LineItem))  # Embedded documents

# Fetching an order gets items in a single round-trip query
order = Order.objects(customer="Alice").first()
```

---

## 9. Document Database with Web Frameworks — Flask & MongoDB

### 9.1 Core Concepts (Simplified)
- Flask is lightweight and unopinionated, making it easy to use with MongoDB.
- Best practice: Initialize a single `MongoClient` object at application startup and share it across HTTP route functions.

### 9.2 Worked Code Example (Flask Application)
```python
from flask import Flask, request, jsonify
from pymongo import MongoClient
from bson import ObjectId

app = Flask(__name__)

# Initialize client once at application startup
client = MongoClient("mongodb://localhost:27017/")
notes_collection = client["noteapp"]["notes"]

@app.route("/notes", methods=["POST"])
def create_note():
    data = request.json
    result = notes_collection.insert_one({"title": data["title"], "body": data["body"]})
    return jsonify({"id": str(result.inserted_id)}), 201

@app.route("/notes/<note_id>", methods=["GET"])
def get_note(note_id):
    note = notes_collection.find_one({"_id": ObjectId(note_id)})
    if note:
        return jsonify({"title": note["title"], "body": note["body"]})
    return jsonify({"error": "Not found"}), 404
```

---

## Practice Problems & Solutions

1. **Query Operators:** Write a PyMongo query to find products with `category` in `["electronics", "toys"]` and `price >= 50`.
   * *Solution:* `db.products.find({"category": {"$in": ["electronics", "toys"]}, "price": {"$gte": 50}})`
2. **Index Usage:** Does index `{"a": 1, "b": 1, "c": 1}` optimize a query filtering on `{"b": 2, "c": 3}`?
   * *Solution:* **No.** It skips leading field `"a"`, violating the compound index prefix rule.
3. **Atomic Counter:** Increment page views for path `"/home"`, creating it if it doesn't exist.
   * *Solution:* `db.pageviews.update_one({"page": "/home"}, {"$inc": {"views": 1}}, upsert=True)`

---

## Unit III Summary Cheat-Sheet

| Concept | Key Takeaway |
|---|---|
| **`MongoClient`** | Create once at app startup to reuse the connection pool across HTTP requests. |
| **Partial Updates (`$set`)** | Updates targeted fields safely without overwriting concurrent edits on other fields. |
| **Index Prefix Rule** | Queries must filter on leading fields of a compound index to utilize it. |
| **Geohashing** | Maps 2D coordinates to 1D strings so standard B-trees can run spatial proximity queries. |
| **Upserts** | Prevents race conditions by atomically updating or inserting in a single operation. |
| **Aggregation Pipelines** | Multi-stage document processing (`$match` $\to$ `$group` $\to$ `$sort`). |
| **Flask Integration** | Share a single `MongoClient` instance globally across route handlers. |
