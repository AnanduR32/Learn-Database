# Unit III — Advanced MongoDB Application Development with Python & Web Frameworks
### 25CSA642A — NoSQL Databases

This unit moves from MongoDB theoretical foundations to production-grade software development: connecting via PyMongo, executing nuanced CRUD and field modification operations, managing B-tree, Compound, and Geospatial indexes, executing atomic upserts and positional array updates, building multi-stage aggregation pipelines, understanding replica sets/sharding, and integrating MongoDB with Flask and Django.

**Roadmap of this file:**
1. Connecting to MongoDB with Python (PyMongo) & Connection Pooling
2. Nuanced MongoDB Document Operations — Adding, Updating, Removing & Renaming Fields
3. Array Modifications & Positional Update Operators (`$`, `$[]`, `$[elem]`)
4. Query Operators & Filtering
5. Using Indexes with MongoDB — Compound Prefix Rule & Geospatial Indexing
6. Atomic Upserts & Find-and-Modify Operations
7. The Aggregation Framework — Single-Purpose Aggregations & Multi-Stage Pipelines
8. Document Database with Web Frameworks — Django & MongoEngine ODM
9. Document Database with Web Frameworks — Flask & PyMongo REST API

---

## 1. Connecting to MongoDB with Python (PyMongo) & Connection Pooling

### 1.1 Core Concepts (Simplified)
- **PyMongo:** The official thread-safe Python driver for MongoDB.
- **`MongoClient` Connection Pool:** Manages a pool of persistent TCP socket connections and translates Python native dictionaries into BSON (Binary JSON) formats.
- **Connection Pooling Best Practice:** Instantiate `MongoClient` **exactly once** as a global or application singleton at startup. Avoid instantiating `MongoClient` inside request handlers or loops.

### 1.2 Proof — Connection Pooling Amortization
Establishing a new database connection requires TCP 3-way handshakes and TLS cryptographic handshakes, costing a fixed setup latency $c \approx 50-100\text{ms}$.
- **Without Connection Pooling:** Executing $m$ operations establishes $m$ individual connections, paying total setup overhead $m \times c$.
- **With Connection Pooling:** A pre-warmed pool of size $k$ pays the setup cost $k \times c$ once. Subsequent operations check out active sockets in $O(1)$ time. As total operations $m \to \infty$, the amortized connection setup overhead per query approaches **zero**:
$$\lim_{m \to \infty} \frac{k \cdot c + m \cdot t_{\text{exec}}}{m} = t_{\text{exec}}$$

### 1.3 Worked Code Example
```python
from pymongo import MongoClient

# Single client instance initialized once at application startup
client = MongoClient(
    "mongodb://localhost:27017/",
    maxPoolSize=50,
    minPoolSize=10,
    serverSelectionTimeoutMS=5000
)

db = client["ecommerce_db"]
orders_collection = db["orders"]

# Query automatically borrows and returns a socket to the pool
order = orders_collection.find_one({"status": "shipped"})
print(order)
```

---

## 2. Nuanced MongoDB Document Operations — Adding, Updating, Removing & Renaming Fields

### 2.1 Updating Fields & Adding New Columns

MongoDB documents are dynamic. You can add new attributes (columns) or modify existing ones without altering collection-wide schemas.

#### 1. Adding a New Column / Field
Use `$set` with a field name that does not currently exist:
```python
# Adds 'is_verified' and 'joined_date' to a single user document
db.users.update_one(
    {"_id": user_id},
    {"$set": {"is_verified": True, "joined_date": datetime.utcnow()}}
)

# Adds a default 'loyalty_tier' column to ALL documents missing this attribute
db.users.update_many(
    {"loyalty_tier": {"$exists": False}},
    {"$set": {"loyalty_tier": "bronze"}}
)
```

#### 2. Updating a Singular Column / Field
Use `$set` with an existing field name to update only that targeted value:
```python
# Modifies only 'email' and 'status', leaving all other document fields untouched
db.users.update_one(
    {"_id": user_id},
    {"$set": {"email": "alice.new@example.com", "status": "active"}}
)

# Updating nested sub-document fields using dot notation (must be string-quoted)
db.users.update_one(
    {"_id": user_id},
    {"$set": {"profile.address.city": "San Francisco", "profile.preferences.dark_mode": True}}
)
```

#### 3. Removing / Dropping a Column (`$unset`)
To completely delete one or more fields from a document:
```python
# Deletes 'temp_token' and 'legacy_id' from the document
db.users.update_one(
    {"_id": user_id},
    {"$unset": {"temp_token": "", "legacy_id": ""}}
)
```

#### 4. Renaming a Field / Column (`$rename`)
```python
# Renames 'phone_no' to 'phone_number' across all matching documents
db.users.update_many(
    {},
    {"$rename": {"phone_no": "phone_number"}}
)
```

#### 5. Numeric & Arithmetic Operations (`$inc`, `$mul`, `$min`, `$max`)
```python
# Atomic Increment / Decrement
db.products.update_one({"_id": product_id}, {"$inc": {"stock": -1, "views": 1}})

# Multiply value (e.g., apply 10% discount)
db.products.update_one({"_id": product_id}, {"$mul": {"price": 0.90}})

# $min: Updates only if specified value is LESS than current field value
db.drivers.update_one({"_id": driver_id}, {"$min": {"fastest_lap": 52.4}})

# $max: Updates only if specified value is GREATER than current field value
db.gamers.update_one({"_id": gamer_id}, {"$max": {"high_score": 12500}})
```

---

## 3. Array Modifications & Positional Update Operators

MongoDB treats arrays as first-class nested structures.

### 3.1 Basic Array Updates (`$push`, `$addToSet`, `$pull`, `$pop`)
```python
# $push: Appends an element (duplicates allowed)
db.posts.update_one({"_id": post_id}, {"$push": {"tags": "nosql"}})

# $addToSet: Adds an element ONLY IF it doesn't already exist (Set behavior)
db.users.update_one({"_id": user_id}, {"$addToSet": {"roles": "admin"}})

# $pull: Removes all occurrences matching the criteria
db.posts.update_one({"_id": post_id}, {"$pull": {"tags": "deprecated"}})
```

### 3.2 Positional Array Operators (`$`, `$[]`, `$[<identifier>]`)
```python
# 1. First Matched Element ($): Updates the first array item matching the query filter
db.students.update_one(
    {"_id": student_id, "grades.subject": "Math"},
    {"$set": {"grades.$.score": 95}}
)

# 2. All Array Elements ($[]): Increments bonus points for every item in the array
db.students.update_one(
    {"_id": student_id},
    {"$inc": {"grades.$[].bonus_points": 5}}
)

# 3. Filtered Elements ($[elem] + array_filters): Updates only elements matching custom condition
db.students.update_one(
    {"_id": student_id},
    {"$set": {"grades.$[elem].letter_grade": "A+"}},
    array_filters=[{"elem.score": {"$gte": 90}}]
)
```

---

## 4. Query Operators & Filtering

- **Comparison Operators:** `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin`.
- **Logical Operators:** `$and`, `$or`, `$not`, `$nor`.
- **Element Operators:** `$exists`, `$type`.
- **Array Operators:** `$elemMatch` (verifies multiple conditions on the *same* array element), `$all`, `$size`.

```python
# Array query: Find orders containing at least one item with category 'electronics' and price > 100
orders.find({
    "status": "completed",
    "items": {
        "$elemMatch": {
            "category": "electronics",
            "price": {"$gt": 100}
        }
    }
})
```

---

## 5. Using Indexes with MongoDB — Compound Prefix Rule & Geospatial Indexing

### 5.1 The Compound Index Prefix Rule
An index on `(A, B, C)` sorts entries lexicographically by $A$ first, then $B$, then $C$.
- ✅ Optimizes queries on `(A)`, `(A, B)`, and `(A, B, C)`.
- ❌ **Cannot** optimize queries filtering on `(B)` or `(C)` alone because the search cannot skip the leading prefix $A$.

### 5.2 Geospatial Indexing (`2dsphere`) & Geohashing
- **`2dsphere` Index:** Supports spherical coordinates (Earth geometry) using GeoJSON Point formats: `{"type": "Point", "coordinates": [longitude, latitude]}`.
- **Proximity Search (`$near`):**
```python
stores.create_index([("location", "2dsphere")])

# Find all stores within 5,000 meters of a coordinate
nearby_stores = stores.find({
    "location": {
        "$near": {
            "$geometry": {"type": "Point", "coordinates": [-73.985, 40.748]},
            "$maxDistance": 5000
        }
    }
})
```

---

## 6. Atomic Upserts & Find-and-Modify Operations

### 6.1 Upserts (`upsert=True` & `$setOnInsert`)
Eliminates Check-Then-Act race conditions by performing update-or-insert atomically:
```python
# Updates counter if document exists; initializes creation fields if newly inserted
analytics.update_one(
    {"page": "/dashboard", "date": "2026-08-14"},
    {
        "$inc": {"views": 1},
        "$setOnInsert": {"created_at": datetime.utcnow(), "initial_referrer": "direct"}
    },
    upsert=True
)
```

### 6.2 Atomic Find-and-Modify (`find_one_and_update`)
Useful for distributed task workers checking out pending jobs atomically:
```python
job = task_queue.find_one_and_update(
    {"status": "queued"},
    {"$set": {"status": "in_progress", "worker_id": "worker_42", "started_at": datetime.utcnow()}},
    sort=[("priority", -1)],
    return_document=pymongo.ReturnDocument.AFTER
)
```

---

## 7. The Aggregation Framework — Single-Purpose Aggregations & Multi-Stage Pipelines

### 7.1 Single-Purpose Aggregation Methods
```python
# Fast metadata count
approx_count = orders.estimated_document_count()

# Exact filtered count
exact_count = orders.count_documents({"status": "shipped"})

# Distinct values for an attribute
unique_countries = users.distinct("profile.address.country", {"status": "active"})
```

### 7.2 Aggregation Pipeline Architecture & RAM Constraints
- **Pipeline Processing:** Documents stream through sequential stages ($[\text{Stage}_1] \to [\text{Stage}_2] \to \dots$).
- **RAM Limits:** Each stage is restricted to **100 MB of RAM** by default. Pass `allowDiskUse=True` to spill large sort/group operations to temporary disk files.

### 7.3 Complete Pipeline Stages Summary

| Stage | Purpose | SQL Analogy |
|---|---|---|
| `$match` | Filters documents passing into subsequent stages | `WHERE` |
| `$project` | Reshapes documents, selects/excludes fields, creates computed expressions | `SELECT` |
| `$group` | Groups documents by key and computes accumulators (`$sum`, `$avg`, `$min`, `$max`) | `GROUP BY` |
| `$sort` | Orders documents | `ORDER BY` |
| `$limit` / `$skip` | Pagination control | `LIMIT` / `OFFSET` |
| `$unwind` | Deconstructs an array field into individual documents per array element | *Flatten / Table-Valued Function* |
| `$lookup` | Performs left outer joins to another collection | `LEFT OUTER JOIN` |
| `$addFields` | Appends computed fields without dropping existing fields | *Computed columns* |
| `$facet` | Processes multiple independent sub-pipelines in parallel | *Multi-dimensional reporting* |
| `$out` / `$merge` | Writes pipeline results into a target collection | `INSERT INTO ... SELECT` |

### 7.4 Worked Aggregation Pipeline Example
```python
pipeline = [
    # 1. Filter completed orders from 2026
    {"$match": {"status": "completed", "year": 2026}},

    # 2. Deconstruct line items array
    {"$unwind": "$items"},

    # 3. Filter only high-value items
    {"$match": {"items.price": {"$gte": 50}}},

    # 4. Group by category and compute total sales and average item quantity
    {
        "$group": {
            "_id": "$items.category",
            "total_revenue": {"$sum": {"$multiply": ["$items.price", "$items.qty"]}},
            "avg_qty": {"$avg": "$items.qty"},
            "item_count": {"$count": {}}
        }
    },

    # 5. Sort highest revenue first
    {"$sort": {"total_revenue": -1}},

    # 6. Limit to top 5 categories
    {"$limit": 5}
]

results = list(orders.aggregate(pipeline, allowDiskUse=True))
```

---

## 8. Document Database with Web Frameworks — Django & MongoEngine ODM

Django's built-in ORM is designed for SQL tables. Integrating Django with MongoDB uses an Object-Document Mapper (ODM) such as **MongoEngine**:

```python
from mongoengine import Document, EmbeddedDocument, StringField, DecimalField, ListField, EmbeddedDocumentField

class OrderItem(EmbeddedDocument):
    sku = StringField(required=True)
    quantity = DecimalField(default=1)
    price = DecimalField(required=True)

class CustomerOrder(Document):
    customer_id = StringField(required=True)
    status = StringField(default="pending")
    items = ListField(EmbeddedDocumentField(OrderItem)) # Embedded sub-documents

    meta = {
        'collection': 'customer_orders',
        'indexes': ['customer_id', 'status']
    }

# Querying via MongoEngine
order = CustomerOrder.objects(customer_id="CUST-100", status="pending").first()
```

---

## 9. Document Database with Web Frameworks — Flask & PyMongo REST API

Flask provides a lightweight, thread-safe environment to expose MongoDB collections via RESTful endpoints:

```python
from flask import Flask, request, jsonify
from pymongo import MongoClient
from bson import ObjectId
import datetime

app = Flask(__name__)

# Initialize single global connection pool
mongo_client = MongoClient("mongodb://localhost:27017/", maxPoolSize=25)
notes_collection = mongo_client["notes_db"]["notes"]

@app.route("/api/notes", methods=["POST"])
def create_note():
    payload = request.get_json()
    new_note = {
        "title": payload["title"],
        "body": payload["body"],
        "tags": payload.get("tags", []),
        "created_at": datetime.datetime.utcnow()
    }
    result = notes_collection.insert_one(new_note)
    return jsonify({"id": str(result.inserted_id), "status": "created"}), 201

@app.route("/api/notes/<note_id>", methods=["PATCH"])
def update_note_field(note_id):
    payload = request.get_json()
    # Supports dynamic single/multi field updates via $set
    result = notes_collection.update_one(
        {"_id": ObjectId(note_id)},
        {"$set": payload}
    )
    if result.matched_count == 0:
        return jsonify({"error": "Note not found"}), 404
    return jsonify({"status": "updated"}), 200

if __name__ == "__main__":
    app.run(port=5000)
```

---

## Practice Problems & Solutions

1. **Positional Update:** A document contains `scores: [85, 92, 78, 95]`. Write a PyMongo command to increment all scores below 80 by 5 points.
   * *Solution:*
     ```python
     db.students.update_one(
         {"_id": student_id},
         {"$inc": {"scores.$[elem]": 5}},
         array_filters=[{"elem": {"$lt": 80}}]
     )
     ```
2. **Aggregation Stage Ordering:** Why should `$match` be positioned before `$unwind` in an aggregation pipeline?
   * *Solution:* Placing `$match` first filters out non-relevant documents before array expansion. Unwinding prior to filtering unnecessarily duplicates millions of array sub-documents into memory, causing high CPU/RAM overhead and cache thrashing.
3. **Compound Index Prefix:** Given index `{"country": 1, "city": 1, "postal_code": 1}`, will it accelerate a query filtering on `{"city": "Austin", "postal_code": "78701"}`?
   * *Solution:* **No.** The query skips the leading prefix attribute `"country"`, violating the B-tree compound index prefix rule and resulting in a full collection scan (`COLLSCAN`).

---

## Unit III Summary Cheat-Sheet

| Concept | Key Takeaway |
|---|---|
| **`MongoClient` Singleton** | Create once at app startup to amortize TCP/TLS handshake latencies across all threads. |
| **Adding / Updating Columns** | Use `$set` for singular/multiple fields; dot-notation for nested documents. |
| **Removing Columns** | Use `$unset: {"field_name": ""}`. |
| **Positional Operators** | `$` (first matched), `$[]` (all elements), `$[elem]` + `array_filters` (conditional elements). |
| **Atomic Upserts** | `upsert=True` with `$setOnInsert` ensures race-condition-free check-and-insert. |
| **Aggregation Pipelines** | Multi-stage streaming transformations with 100MB RAM stage limit (`allowDiskUse=True`). |
| **`$unwind` & `$lookup`** | `$unwind` flattens array elements into discrete documents; `$lookup` performs left outer joins. |
