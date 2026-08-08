# Unit III — Advanced MongoDB Application Development with Python & Web Frameworks
### 25CSA642A — NoSQL Databases

This unit moves from MongoDB's conceptual model (Unit I §8) to hands-on application development: connecting via PyMongo, the full query/update/delete language and its operators, indexing (including geospatial), upserts, the aggregation framework with replication/sharding, and integration with Django and Flask.

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

### 1.1 Theory

- **PyMongo** is the official MongoDB driver for Python, exposing a `MongoClient` object that manages a **connection pool** to a MongoDB server/cluster.
- `MongoClient` speaks the MongoDB wire protocol, translating Python dicts to/from BSON.
- A single `MongoClient` instance is thread-safe and intended to be created **once** per application process — it internally maintains a pool of TCP connections reused across operations, rather than opening a new socket per query.

### 1.2 Proof — connection pooling reduces amortized per-operation latency

**Claim:** for $m$ sequential operations against a database, per-operation cost with a reused connection pool of size $\ge 1$ is $O(1)$ amortized network-setup overhead, versus $O(m)$ total TCP-handshake overhead without pooling.

**Proof:**
- A TCP handshake (SYN/SYN-ACK/ACK) plus TLS negotiation costs a fixed latency $c$ independent of the query itself.
- Opening a new connection per operation pays $c$ each time — total overhead $mc$.
- A pool of size $k\ge 1$ amortizes the handshake cost: after the first $k$ connections are established (one-time cost $kc$), each of the remaining $m-k$ operations reuses an already-open connection at $O(1)$ marginal setup cost (no handshake), giving total overhead $kc + O(m-k)\cdot(\text{no handshake})=kc$ — a **constant** independent of $m$ for fixed $k$.
- As $m\to\infty$, amortized per-operation overhead $kc/m \to 0$, whereas the unpooled scheme's amortized overhead stays fixed at $c$ per operation.

This is the formal justification for the PyMongo best practice of instantiating `MongoClient` once (module/application scope), not per-request.

### 1.3 Worked Example

```python
from pymongo import MongoClient

client = MongoClient("mongodb://localhost:27017/")   # one pool, created once
db = client["ecommerce"]
orders = db["orders"]

print(orders.find_one({"status": "shipped"}))
```
Every subsequent `orders.find_one(...)` / `orders.insert_one(...)` call reuses `client`'s internal pool (§1.2) rather than reconnecting.

### 1.4 Nuances

- Creating a new `MongoClient` per request (a common anti-pattern in web frameworks, §8–9) defeats pooling and can exhaust server connection limits under load — directly reintroducing the $O(m)$ overhead §1.2 shows pooling eliminates.
- Connection strings (`mongodb://` or `mongodb+srv://` for Atlas) can encode replica-set members, authentication, and read/write preferences — tying directly into replication concepts (§7).
- PyMongo raises specific exceptions (`ConnectionFailure`, `ServerSelectionTimeoutError`) distinct from query-level errors, reflecting the distributed nature of the target (a single logical database may be several physical replicas, §7).

### 1.5 Real-World Application

Production Python services (Django/Flask, §8–9) instantiate a single shared `MongoClient` at application startup, reused across all incoming HTTP requests for the lifetime of the process — the standard deployment pattern this section's proof formally justifies.

---

## 2. MongoDB Query Language — Find, Update, Delete Operations

### 2.1 Theory

- CRUD in MongoDB:
  - **Create** — `insert_one`/`insert_many`
  - **Read** — `find_one`/`find` with a BSON filter document (Unit I §8.1's $\sigma$ selection)
  - **Update** — `update_one`/`update_many`, applying a modification document with operators like `$set`
  - **Delete** — `delete_one`/`delete_many`
- Updates are, by default, **partial** — only the specified fields change; the rest of the document is untouched, unlike a relational `UPDATE` that logically rewrites the full row.

### 2.2 Proof — partial-field update preserves untouched-field invariants under concurrent writers

**Claim:** if two concurrent updates $U_1=\{\$set:\{a: x\}\}$ and $U_2=\{\$set:\{b: y\}\}$ target disjoint field sets on the same document, applying them in either order yields the same final document (they **commute**), unlike two relational `UPDATE ... SET *` statements that overwrite the entire row and would lose one writer's change under a naive last-writer-wins row replacement.

**Proof:**
- MongoDB's storage engine applies an update as a targeted mutation to specific BSON paths within the document's on-disk representation.
- Mathematically, the document is a partial function `fields → values`, and `$set` on field $a$ is the function-update operator $f' = f[a\mapsto x]$ (identical to $f$ except at $a$).
- Composing two such updates on disjoint domains, $f[a\mapsto x][b\mapsto y] = f[b\mapsto y][a\mapsto x]$, because updating $f$ at $a$ does not touch the value returned at $b$ and vice versa — this is exactly the mathematical definition of function-update commutativity for disjoint keys.
- Each individual `update_one` call is additionally atomic at the single-document level (Unit I §5.2's aggregate-atomicity argument), so no torn/partial write is observable.

This proves why `$set`-based partial updates are safe under concurrent writers targeting different fields, without requiring the application to re-read-and-rewrite the entire document.

### 2.3 Worked Example

```python
orders.update_one({"_id": 1042}, {"$set": {"status": "shipped"}})
orders.delete_many({"status": "cancelled", "createdAt": {"$lt": cutoff}})
```
The `update_one` call touches only `status`, leaving `items`, `customer`, etc. untouched (§2.2); `delete_many` removes every document matching the compound filter in one atomic-per-document sweep.

### 2.4 Nuances

- `update_one` matches and updates only the **first** matching document; `update_many` applies to all matches — a common source of bugs when the wrong variant is used against a non-unique filter.
- Without an explicit `$set` (i.e., passing a plain replacement document), `update_one`/`replace_one` performs a **full-document replace**, losing the commutativity property of §2.2 — an important operator distinction.
- Deletes are permanent by default (no built-in soft-delete/undo); production systems commonly add an application-level `deletedAt` field and filter it out, rather than relying on `$delete` semantics for reversibility.

### 2.5 Real-World Application

Order-status pipelines (`pending → shipped → delivered`) rely on `update_one` with `$set` to advance state without re-sending the entire order document over the wire, minimizing both bandwidth and the risk of clobbering concurrently-written fields (e.g., a customer editing their shipping address while the warehouse marks the order shipped).

---

## 3. MongoDB Query Operators

### 3.1 Theory

- Query operators extend simple equality matching into a small predicate algebra:
  - **Comparison** — `$eq, $ne, $gt, $gte, $lt, $lte, $in, $nin`
  - **Logical** — `$and, $or, $not, $nor`
  - **Element** — `$exists, $type`
  - **Array** — `$all, $elemMatch, $size`
- Together these operators let a BSON filter document express any conjunction/disjunction of field predicates — a restricted but composable analogue of a relational `WHERE` clause's boolean expression tree.

### 3.2 Proof — the operator set is closed under composition and expressively equivalent to propositional predicate logic over field comparisons

**Claim:** any Boolean combination (AND/OR/NOT) of atomic field-comparison predicates (each of the form "field $\theta$ value" for $\theta\in\{=,\ne,<,\le,>,\ge\}$) can be expressed as a single BSON filter document using only `$and`, `$or`, `$not`, and the comparison operators.

**Proof by structural induction on the Boolean expression tree:**
- *Base case:* an atomic predicate "field $\theta$ value" maps directly to `{field: {$θ: value}}` — a leaf filter document.
- *Inductive step:* assume sub-expressions $P, Q$ already have equivalent filter documents $F_P, F_Q$. Then $P \wedge Q$ maps to `{$and: [F_P, F_Q]}`, $P \vee Q$ maps to `{$or: [F_P, F_Q]}`, and $\neg P$ maps to `{$not: F_P}` (or a De Morgan rewrite pushing negation to the leaves, e.g. $\neg(P\wedge Q) \equiv \neg P \vee \neg Q$, both of which remain expressible by the inductive hypothesis).
- Since every Boolean formula over atomic comparisons can be built from AND/OR/NOT applied finitely many times to atoms (by definition of Boolean expression trees), and each construction step is shown expressible above, induction gives that **every** such formula has an equivalent MongoDB filter document.

This proves MongoDB's query operators are not an ad hoc convenience set but a complete predicate calculus over field comparisons — the same completeness property relational algebra's selection $\sigma$ has over its own predicate language (Unit I §1.1), scoped to a single collection (no cross-collection join, per Unit I §8.1).

### 3.3 Worked Example

"Shipped orders over $100 that are not from a blocked customer list":
```python
orders.find({
  "$and": [
    {"status": "shipped"},
    {"total": {"$gt": 100}},
    {"customerId": {"$nin": blocked_ids}}
  ]
})
```
Array operator example — orders containing a line item with quantity ≥ 5:
```python
orders.find({"items": {"$elemMatch": {"qty": {"$gte": 5}}}})
```
`$elemMatch` is necessary (rather than `{"items.qty": {"$gte": 5}}`) when the condition must hold on a **single** array element rather than independently across different elements.

### 3.4 Nuances

- `$or` at the top level cannot use an index as efficiently as an equivalent `$in` on a single field — query planners handle `$in` as a single indexed multi-point lookup, while general `$or` may require evaluating and merging separate index scans (relevant to §4's indexing discussion).
- Implicit AND (multiple fields in one filter document, no explicit `$and`) is semantically identical to explicit `$and` per §3.2 but more idiomatic and typically better optimized by the query planner.
- Array-field matching without `$elemMatch` matches if *any* array element satisfies each condition *independently*, which is a subtly different (weaker) predicate than "one single element satisfies all conditions" — a common source of query bugs.

### 3.5 Real-World Application

Faceted search/filtering UIs (price range + category + in-stock) compile directly to compound `$and` filters combining comparison and equality operators, letting the query planner select a compound index (§4) matching the filter shape.

---

## 4. Using Indexes with MongoDB

### 4.1 Theory

- Indexes are B-tree (Unit I §8.2) structures maintained on one or more fields to avoid full collection scans.
- Primary types: **single-field**, **compound** (multiple fields, ordered), **multikey** (automatically created when indexing an array field — one index entry per array element), and **text** indexes.
- Compound index field **order** matters: an index on `(a, b)` can serve queries filtering on `a` alone or on `a` and `b` together, but not efficiently on `b` alone.

### 4.2 Proof — the "prefix rule" for compound index usability

**Claim:** a compound B-tree index built on fields $(f_1, f_2, \ldots, f_k)$ (in that sort order) can be used to efficiently answer any query whose equality/range predicates form a **prefix** $(f_1, \ldots, f_j)$ for $j\le k$, but cannot in general provide a sub-linear lookup for a predicate on $f_j$ alone when $j>1$.

**Proof:**
- A compound B-tree index sorts entries lexicographically by $(f_1, f_2, \ldots, f_k)$ — entries are ordered first by $f_1$, then, within equal $f_1$, by $f_2$, and so on (identical in principle to a dictionary sorted first by surname, then first name).
- A query fixing $f_1=v_1$ can binary-search to the contiguous range of entries with that $f_1$ value in $O(\log_b n)$ (Unit I §8.2's B-tree bound), then optionally continue narrowing by $f_2$ within that range — because entries with the same $f_1$ are contiguous and internally sorted by $f_2$.
- But a query fixing only $f_2=v_2$ (skipping $f_1$) cannot binary search: entries with $f_2=v_2$ are scattered across every $f_1$-group (since sort order is dominated by $f_1$ first), so locating them requires scanning within *each* $f_1$-group — degenerating toward $O(n)$ overall, no better than an unindexed scan.

This proves the prefix rule from first principles of lexicographic sort order, not merely as an empirical MongoDB quirk.

### 4.3 Worked Example

Index `{status: 1, createdAt: -1}` efficiently serves `find({status: "shipped"})` (uses the $f_1$-prefix, §4.2) and `find({status: "shipped", createdAt: {"$gte": t}})` (uses both fields — status equality narrows the range, then createdAt's descending sort order is directly usable). It does **not** efficiently serve `find({createdAt: {"$gte": t}})` alone (skips the $f_1$ prefix) — that query would need its own leading index on `createdAt`.

### 4.4 Nuances

- `explain()` reveals whether a query used an index scan (`IXSCAN`) or fell back to `COLLSCAN` — the practical tool for verifying the prefix rule holds for a given query shape.
- Every index write-amplifies inserts/updates (maintaining the B-tree on each write) — directly the same trade-off proven in Unit I §1.4 for relational indexes; over-indexing a write-heavy collection can hurt more than it helps.
- Multikey indexes on array fields have restrictions (e.g., only one array field per compound index) because indexing the Cartesian product of two arrays would explode index size combinatorially.
- **Text indexes** tokenize string fields for keyword search (a specialized, non-B-tree inverted-index structure) rather than exact/range B-tree matching; **hashed indexes** apply a hash function to the field before insertion into the B-tree, sacrificing range-query support (hashing destroys ordering) in exchange for perfectly uniform key distribution — the same uniformity goal as consistent hashing (Unit I §9.2), which is why MongoDB recommends hashed indexes as shard keys (§7.4).
- `drop_index()`/`drop_indexes()` remove an index's write-amplification cost when a query pattern is retired — index lifecycle management (create for a known query shape, drop when unused) is as important as initial index design, verifiable via `explain()` and index-usage statistics.
- **Atlas** (MongoDB's managed cloud service) and **Compass** (its GUI client) provide the same CRUD/index/aggregation operations covered in §1–7 through a hosted deployment and visual interface respectively — useful for exploration and administration, but they wrap the identical underlying wire protocol and query semantics PyMongo uses programmatically.

### 4.5 Real-World Application

Designing compound indexes to match the **exact prefix** of an application's most frequent query shapes (e.g., `{tenantId: 1, status: 1, createdAt: -1}` for a multi-tenant SaaS dashboard) is standard MongoDB performance practice, directly informed by §4.2's prefix-rule proof.

---

## 5. GeoSpatial Indexing

### 5.1 Theory

- MongoDB supports **2dsphere** indexes (for GeoJSON points/lines/polygons on an Earth-like sphere) and legacy **2d** indexes (flat-plane coordinates), enabling queries like `$near` (nearest points to a location), `$geoWithin` (points inside a shape), and `$geoIntersects`.
- Internally, 2dsphere indexes use a **geohash**-like technique that encodes 2D coordinates into a single sortable string/integer by interleaving bits of latitude and longitude, mapping proximity in 2D space to proximity in the resulting 1D sort order — enabling reuse of the ordinary B-tree machinery (Unit I §8.2) for spatial range queries.

### 5.2 Proof — geohashing approximately preserves spatial locality, enabling B-tree-indexed proximity search

**Claim:** two points whose geohash strings share a common prefix of length $\ell$ lie within a bounding cell whose area shrinks geometrically as $\ell$ increases — specifically, each additional interleaved bit pair halves the cell's width and height, so cell area shrinks by a factor of 4 per two additional bits.

**Proof:**
- Geohashing recursively bisects the coordinate space: the first bit distinguishes which half of the longitude range a point falls in, the second bit which half of the latitude range, and so on, alternating — this is a **quadtree** in disguise.
- Each two-bit increment selects one of 4 sub-quadrants of the current cell, and the sub-quadrant's area is exactly $1/4$ of the parent cell's area by construction (bisecting both dimensions).
- Two points sharing a length-$\ell$ geohash prefix have, by this recursive construction, both been assigned to the *same* cell at recursion depth $\ell$ — hence both lie within that cell's bounding box, whose area is $O(4^{-\ell/2})$ of the original space.

This proves shared-prefix geohash strings *imply* spatial proximity (bounded by the cell), which is precisely why sorting geohashes lexicographically (a B-tree's native operation) clusters nearby points together, letting `$near`/`$geoWithin` be answered via ordinary indexed range scans rather than an $O(n)$ scan comparing every point's true distance.

### 5.3 Worked Example

```python
db.stores.create_index([("location", "2dsphere")])
db.stores.find({
  "location": {"$near": {"$geometry": {"type": "Point", "coordinates": [lng, lat]}, "$maxDistance": 5000}}
})
```
finds stores within 5 km, using the geohash-ordered B-tree to prune the search to nearby cells (§5.2) rather than computing true geodesic distance for every stored point.

### 5.4 Nuances

- Geohash prefix-sharing is a **sufficient but approximate** locality signal — points near a cell *boundary* can be geographically close while having very different geohash prefixes (the classic "boundary problem"); production geospatial engines query neighboring cells too, to compensate.
- `2dsphere` accounts for Earth's curvature (spherical geometry); `2d` assumes a flat plane and is only appropriate for small-scale or non-geographic 2D data.
- Geospatial indexes compose with other compound index fields in MongoDB (e.g., indexing `location` alongside `category`), subject to MongoDB's specific restrictions on which index types can combine.

### 5.5 Real-World Application

"Find the nearest 10 open restaurants within 2 km" (ride-hailing driver matching, retail store-locators, delivery-radius checks) are the canonical `$near` query pattern, all resting on §5.2's geohash locality proof for sub-linear performance at scale.

---

## 6. Upserts in MongoDB

### 6.1 Theory

- An **upsert** (`update_one(filter, update, upsert=True)`) performs an update if a matching document exists, or **inserts** a new document (derived from the filter plus update) if none does.
- This combines insert-or-update into one atomic operation, avoiding the classic "check-then-act" race condition of a separate `find` followed by conditional `insert`/`update`.

### 6.2 Proof — upsert atomicity eliminates the check-then-act race condition

**Claim:** a non-atomic "check-then-act" pattern (`if not find_one(filter): insert_one(doc)`) run concurrently by two clients can produce two documents violating an intended uniqueness invariant, while an `update_one(..., upsert=True)` cannot.

**Proof:**
- In the non-atomic pattern, there exists an interleaving where client 1 executes `find_one` (returns nothing), then before client 1's `insert_one` executes, client 2 also executes `find_one` (also returns nothing, since client 1 hasn't inserted yet) — both clients then proceed to `insert_one`, producing two documents where only one was intended.
- This is a standard **time-of-check-to-time-of-use (TOCTOU)** race, provable by exhibiting this interleaving as a valid schedule under any non-atomic execution model.
- An upsert instead performs the check-and-insert-or-update as a **single operation** at the storage-engine level, under the same per-document atomicity guarantee already established for single-document writes (Unit I §5.2).
- The storage engine serializes concurrent upserts targeting the same filter (typically via a unique index on the filter's key field, which rejects/retries the second concurrent insert attempt at the index level) so that no interleaving can produce two documents — the operation is atomic *by construction*, not merely "usually fine in practice."

This is why upserts, backed by a unique index, are the recommended pattern for "insert-or-update" logic instead of manual check-then-act code.

### 6.3 Worked Example

```python
orders.update_one(
  {"externalOrderId": "EXT-9981"},
  {"$set": {"status": "shipped"}, "$setOnInsert": {"createdAt": now}},
  upsert=True
)
```
If no document has `externalOrderId="EXT-9981"`, one is created with `status` and `createdAt` set; if it exists, only `status` is updated (`$setOnInsert` fields are ignored on a pure update) — safe even if two webhook deliveries for the same external order race each other, provided a unique index exists on `externalOrderId` (§6.2).

### 6.4 Nuances

- Upsert safety against races depends on a **unique index** on the filter fields; without one, two concurrent upserts with a *non-unique*, multi-field filter can still both perceive "no match" and both insert, because the storage engine's atomicity guarantee is enforced via that index, not the upsert keyword alone.
- `$setOnInsert` lets an upsert distinguish "fields to set only on creation" from "fields to set on every write" (`$set`) — a common pattern for tracking `createdAt` vs. `updatedAt`.
- Upserts interact with sharding (§7): the shard key must typically be part of the filter, since the upsert must resolve to exactly one shard before the storage engine can apply its atomic check-and-write.

### 6.5 Real-World Application

Idempotent webhook/event ingestion (payment-gateway callbacks, IoT device pings) is the textbook upsert use case — the same event may be delivered more than once, and upserting on a unique event ID guarantees exactly-once effective processing without manual deduplication logic.

---

## 7. Aggregation Framework, Replication, and Sharding

### 7.1 Theory

- The **aggregation framework** processes documents through a **pipeline** of stages (`$match`, `$group`, `$project`, `$sort`, `$lookup`, `$unwind`, ...), each stage's output feeding the next — conceptually a composition of relational-algebra-like operators (Unit I §1.1) chained together, but scoped per-collection.
- **Replication** (a **replica set**: one primary, multiple secondaries) provides durability and read-scaling via asynchronous oplog-based replication, with an election protocol promoting a new primary on failure.
- **Sharding** partitions a collection across multiple replica sets (shards) by a **shard key**, using a config-server-tracked routing layer (`mongos`) to direct operations to the correct shard(s) — the same aggregate-sharding principle formalized in Unit I §7.2.

### 7.2 Proof — pipeline composition preserves the relational-completeness argument of Unit I §10.2, restricted per-collection

**Claim:** any $\sigma$ (select), $\pi$ (project), or grouped-aggregate relational-algebra query over a *single* collection is expressible as a finite aggregation pipeline, but a true $\bowtie$ (join) across two large, differently-sharded collections cannot be executed as efficiently as a single-shard operation.

**Proof (expressiveness):**
- `$match` implements $\sigma$ directly (its filter document is the same predicate language proven complete in §3.2).
- `$project` implements $\pi$ (arbitrary field selection/computation).
- `$group` implements relational `GROUP BY`/aggregate functions (SUM, COUNT, AVG map directly to accumulator expressions like `$sum`, `$avg`).
- Since pipeline stages compose sequentially (each stage's output is a well-formed document stream consumable by the next), and $\sigma, \pi, \text{GROUP BY}$ together with sequencing can express any single-collection relational query (a standard result from relational algebra), the aggregation pipeline is expressively sufficient for single-collection analytics.

**Proof (join cost):**
- `$lookup` implements a left-outer-join-like operation, but per Unit I §7.2's aggregate-sharding argument, if the "local" and "foreign" collections are sharded independently, `$lookup` must, in the worst case, contact every shard holding candidate foreign documents for each local document examined.
- This is an operation whose cost scales with the number of shards touched, unlike a single-shard `$match`/`$group` pipeline which (with a shard-key-aligned filter) can be routed to one shard.

This formally extends Unit I §7.4's qualitative claim ("joins are inherently cross-aggregate, hence expensive") with a concrete pipeline-cost argument.

### 7.3 Worked Example

Revenue by status, most recent first:
```python
pipeline = [
  {"$match": {"createdAt": {"$gte": start}}},
  {"$group": {"_id": "$status", "total": {"$sum": "$amount"}, "count": {"$sum": 1}}},
  {"$sort": {"total": -1}}
]
list(orders.aggregate(pipeline))
```
`$match` first (pushing the filter as early as possible, mirroring relational query-optimizer predicate pushdown) minimizes documents entering the more expensive `$group` stage — the same optimization principle as filtering before joining in SQL.

Replica-set read/write example — reading from a secondary for analytics load-shedding:
```python
client = MongoClient(host, readPreference="secondaryPreferred")
```
directly applies Unit I §4's $N,R,W$ tunable-consistency model: reading from a secondary risks a small replication-lag staleness window (an eventual-consistency read) in exchange for offloading the primary.

### 7.4 Nuances

- `$lookup` is best reserved for small "foreign" collections or reference/lookup tables — for large, independently-growing collections, embedding (Unit I §5) or application-side multi-query joins remain preferable, consistent with §7.2's cost proof.
- Replica-set elections use a Raft-like majority-vote protocol; write concern `"majority"` ties directly to the quorum-intersection proof of Unit I §4.2 (a write is only acknowledged once a majority of replica-set members have it, guaranteeing it survives a single-node failure).
- Choosing a good **shard key** is critical: a low-cardinality or monotonically increasing shard key (e.g., a timestamp) creates a "hot shard" bottleneck, the same load-imbalance failure mode Unit II §2.2 showed vnodes/consistent-hashing are designed to avoid — MongoDB mitigates via hashed shard keys for this reason.

### 7.5 Real-World Application

Business dashboards computing daily/monthly rollups (`$group` by date-truncated field) run directly against a replica-set secondary to avoid loading the primary that serves live application traffic, while high-write-volume, horizontally-scaled collections (event logs, IoT telemetry) rely on sharding by a well-chosen (often hashed) key to avoid the hot-shard problem.

---

## 8. Document Database with Web Frameworks — Django and MongoDB

### 8.1 Theory

- Django's default ORM targets relational databases; using MongoDB with Django requires either:
  - a MongoDB-aware ODM (Object-Document Mapper, e.g., **djongo** or **MongoEngine**) that translates Django model syntax into PyMongo calls, or
  - bypassing the ORM and calling PyMongo directly within Django views.
- Either approach must reconcile Django's implicit "one row = one model instance, joins via ForeignKey" assumption with MongoDB's aggregate/embedding model (Unit I §5, §7).

### 8.2 Proof — impedance mismatch between Django's relational ORM assumptions and MongoDB's aggregate model is structural, not incidental

**Claim:** any ODM mapping Django's `ForeignKey`/`ManyToManyField` relational-reference semantics onto a document store must choose between:
- (a) embedding the referenced object (denormalizing, breaking Django's assumption that a related object has independent identity/lifecycle), or
- (b) storing a reference and performing an *application-level* join (an extra query round-trip Django's ORM would normally hide via SQL `JOIN`).

No third option preserves both single-round-trip fetches and independent referenced-object lifecycle, since these two properties are exactly what Unit I §7.2's aggregate-boundary proof shows are in tension (embedding buys atomicity/single-round-trip at the cost of shared lifecycle; referencing buys independent lifecycle at the cost of a second round-trip).

**Proof:** this is a direct corollary of §7.2/Unit I §7.2 — a `ForeignKey` in Django models a cross-entity relationship, which by definition spans two aggregates in document-model terms. Unit I §7.2 already proved cross-aggregate access requires either a second operation (reference) or collapsing the boundary (embedding), so no mapping layer can avoid this trade-off — an ODM can only make the choice look syntactically like a normal Django `ForeignKey` traversal while performing an extra query underneath.

### 8.3 Worked Example

MongoEngine model with an embedded document (Unit I §5.3's pattern applied to Django):
```python
from mongoengine import Document, EmbeddedDocument, EmbeddedDocumentField, StringField, ListField

class LineItem(EmbeddedDocument):
    sku = StringField()
    qty = StringField()

class Order(Document):
    customer = StringField()
    items = ListField(EmbeddedDocumentField(LineItem))
```
`Order.objects(customer="Alice")` issues a single MongoDB query returning fully-populated `items` — no separate query needed, because `items` is embedded (§8.2's option (a)) rather than referenced.

### 8.4 Nuances

- Django's built-in admin site, migrations framework, and `ForeignKey` cascade-delete behavior assume a relational backend; MongoDB ODMs reimplement subsets of this functionality with document-model semantics, and full feature parity is not guaranteed.
- CRUD via Django views changes only at the model/query layer (`Order.objects.create(...)` vs. raw PyMongo `insert_one`) — URL routing, views, and templates remain framework-standard, since the impedance mismatch of §8.2 is isolated to the data-access layer.
- Some teams bypass an ODM entirely, using PyMongo directly inside Django views/services for full control over embedding vs. referencing decisions, at the cost of losing Django-model conveniences (form generation, admin integration).

### 8.5 Real-World Application

Content-heavy Django sites (CMS-backed blogs, product catalogs with highly variable per-category attributes) commonly pair Django's routing/templating/auth machinery with MongoDB via an ODM specifically because catalog schemas vary by category — a poor fit for a fixed relational schema, a natural fit for MongoDB's schema flexibility (Unit I §2.4).

---

## 9. Document Database with Web Frameworks — Flask and MongoDB

### 9.1 Theory

- Flask, being a minimal/unopinionated framework, has no built-in ORM assumption.
- The common pattern (via **Flask-PyMongo** or direct PyMongo usage) attaches a single shared `MongoClient`/database handle to the Flask application object at startup, then accesses it within route handlers via Flask's application/request context.
- This sidesteps §8.2's ODM impedance-mismatch problem entirely by not attempting to impose relational-style model semantics.

### 9.2 Proof — request-scoped access to a startup-created client preserves §1.2's pooling guarantee under concurrent requests

**Claim:** if a `MongoClient` is created once at Flask application startup and shared (not recreated) across concurrently-handled requests, then $m$ concurrent requests each issuing one query incur the same $O(1)$-amortized connection overhead proven in §1.2, rather than $O(m)$.

**Proof:**
- This is a direct application of §1.2 with "sequential operations" generalized to "concurrent operations sharing one pool."
- `MongoClient`'s internal connection pool is explicitly documented and designed to be **thread-safe**, meaning concurrent calls from different request-handling threads/greenlets each borrow an available connection from the shared pool (or briefly wait/expand up to a max pool size) rather than each independently performing a TCP handshake.
- Since the pool is created exactly once (at startup, outside the per-request code path), the $kc$ one-time setup cost from §1.2 is paid once for the application's lifetime, not once per request — giving the same amortized-to-zero overhead as the sequential case, now under concurrency.

This is why Flask (and Django) integration guides universally instruct creating the client at application/module scope, never inside a route handler.

### 9.3 Worked Example — Note application (per syllabus)

```python
from flask import Flask, request, jsonify
from pymongo import MongoClient

app = Flask(__name__)
client = MongoClient("mongodb://localhost:27017/")   # created once, §9.2
notes = client["noteapp"]["notes"]

@app.route("/notes", methods=["POST"])
def create_note():
    data = request.json
    result = notes.insert_one({"title": data["title"], "body": data["body"]})
    return jsonify({"id": str(result.inserted_id)}), 201

@app.route("/notes/<note_id>", methods=["GET"])
def get_note(note_id):
    from bson import ObjectId
    note = notes.find_one({"_id": ObjectId(note_id)})
    return jsonify({"title": note["title"], "body": note["body"]}) if note else ("Not found", 404)
```
Each route handler reuses the module-level `notes` collection handle, never constructing a new `MongoClient` per request (§9.2).

### 9.4 Nuances

- `ObjectId` (MongoDB's default 12-byte identifier) must be explicitly converted to/from `str` at the HTTP/JSON boundary, since raw `ObjectId` is not JSON-serializable — a routine but easy-to-forget integration detail.
- Flask's lack of an ORM means schema validation (if desired) must be handled explicitly — either via MongoDB's own JSON Schema validators (Unit I §8.4) or an application-level library (e.g., `marshmallow`, `pydantic`) — there is no framework-enforced schema by default.
- Flask's development server is single-threaded by default; production deployment (via a WSGI server with multiple workers/threads) is what actually exercises the concurrent-pooling guarantee of §9.2 at scale.

### 9.5 Real-World Application

Lightweight microservice APIs and small full-stack apps (a note-taking app, a to-do list, an internal admin tool) favor Flask + MongoDB specifically for minimal boilerplate — no migrations, no rigid model classes — trading Django's batteries-included structure for direct control over the document shape, appropriate when the schema is simple or rapidly evolving.

---

## Practice Problems (Exam-Style)

**P1 — PyMongo query.** Find the 5 most recent shipped orders over $100.

*Solution:*
```python
orders.find({"status": "shipped", "price": {"$gt": 100}}) \
      .sort("date", -1).limit(5)
```
The compound filter (§3) narrows by two conditions before `.sort()`/`.limit()` truncate the result set — an index on `{status:1, price:1, date:-1}` would let MongoDB satisfy all three clauses without a separate sort pass (§4's compound-index rule).

**P2 — Query operators.** Find products in `["electronics","toys"]` where price ≥ 50 OR discount is true.

*Solution:*
```python
products.find({
  "category": {"$in": ["electronics", "toys"]},
  "$or": [{"price": {"$gte": 50}}, {"discount": True}]
})
```
`$in` expands to an OR over the listed values; the outer `$or` combines with the implicit AND from having both keys in one filter document (§3's predicate-calculus completeness).

**P3 — Compound index prefix rule.** Given index `{a:1, b:1, c:1}`, which of these queries use it efficiently: (i) filter on `{a,b}`, (ii) filter on `{b,c}`, (iii) filter on `{a,c}`?

*Solution:* (i) **Fully efficient** — matches the index prefix `a,b`. (ii) **Not usable** — skips the leading field `a`, so the B-tree can't be used at all (§4's prefix rule). (iii) **Partially usable** — only the `a` portion of the index narrows the scan; `c` is then filtered in memory since `b` was skipped.

**P4 — GeoSpatial query.** Find restaurants within 2 km of a point `[-73.98, 40.75]`.

*Solution:*
```python
restaurants.find({
  "location": {"$near": {
    "$geometry": {"type": "Point", "coordinates": [-73.98, 40.75]},
    "$maxDistance": 2000
  }}
})
```
Requires a `2dsphere` index on `location`; `$maxDistance` is in meters. The index's geohash-based quadtree (§5) prunes the search to nearby cells instead of scanning every document.

**P5 — Atomic upsert.** Increment a page-view counter, creating the document if it doesn't exist.

*Solution:*
```python
pageviews.update_one(
  {"page": "/home"},
  {"$inc": {"views": 1}},
  upsert=True
)
```
Atomic single-call upsert (§6) avoids the check-then-act race condition a manual "find, then insert-or-update" sequence would have under concurrent requests.

**P6 — Aggregation pipeline.** Compute total revenue per category, highest first.

*Solution:*
```python
orders.aggregate([
  {"$group": {"_id": "$category", "total": {"$sum": {"$multiply": ["$price", "$qty"]}}}},
  {"$sort": {"total": -1}}
])
```
`$group` + `$sum` implement relational `GROUP BY`/`SUM` per collection (§7's relational-completeness argument); no cross-collection join is needed since category is embedded per order.

---

## Unit III Summary

| Concept | One-line takeaway |
|---|---|
| PyMongo connection | A single pooled `MongoClient` amortizes TCP/TLS handshake cost to near-zero per operation |
| Find/Update/Delete | `$set`-based partial updates commute on disjoint fields, unlike full-row relational rewrites |
| Query operators | AND/OR/NOT over comparisons form a provably complete predicate calculus per collection |
| Indexes | The compound-index prefix rule follows directly from B-tree lexicographic sort order |
| GeoSpatial indexing | Geohashing is a quadtree that provably preserves spatial locality in a sortable string |
| Upserts | Atomic upsert-with-unique-index eliminates the check-then-act race condition by construction |
| Aggregation/Replication/Sharding | Pipelines are relationally complete per-collection; joins/hot-shards inherit Unit I's cross-aggregate cost proofs |
| Django + MongoDB | ORM/ODM impedance mismatch is a structural corollary of the embed-vs-reference trade-off |
| Flask + MongoDB | Framework-level pooling guarantees extend cleanly from sequential to concurrent request handling |
