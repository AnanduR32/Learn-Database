# Redis

- In-memory key-value data structure store, used as database, cache, and message broker
- Written in C, single-threaded event loop (network) + background threads (persistence)
- All data lives in RAM for sub-millisecond latency

Usecases:

- Caching (Most common use: TTL-based expiry reduces DB load)
- Session storage (Fast read/write with automatic expiry)
- Rate limiting (Atomic counters with INCR + EXPIRE)
- Leaderboards (Sorted sets for ranked data)
- Real-time messaging (Pub/Sub channels, Streams for consumer groups)
- Distributed locks (SET NX / Redlock algorithm)

## Logical Hierarchy

Redis Instance -[1..n]-> Database (logical index, 0-15) ->> Key -> Value (one of many data structures)

- A server has 16 logical databases by default (selectable via `SELECT index`)
- Keys are binary-safe strings
- Values can be one of several data structures (not just strings)
- Keys can have TTL (time-to-live) for automatic expiry

## Data Structures

| Type | Description | Example |
|------|-------------|---------|
| **String** | Binary-safe string (text, serialized JSON, binary). Max 512MB. | `SET name "Alice"` |
| **List** | Linked list of strings. Fast head/tail ops. Queue/stack behavior. | `LPUSH queue task` / `RPOP queue` |
| **Set** | Unordered set of unique strings. O(1) add/remove/check. | `SADD admins "alice"` |
| **Sorted Set (ZSet)** | Ordered by score. Fast range queries. Leaderboards. | `ZADD leaderboard 100 "alice"` |
| **Hash** | Map of string fields to string values. Like a mini-document. | `HSET user:1 name "Alice" age 30` |
| **BitMap** | String as bit array. Bit-level ops. Analytics. | `SETBIT login:2026-07-27 42 1` |
| **HyperLogLog** | Probabilistic cardinality estimator (~0.81% error). | `PFADD unique_visits "alice"` |
| **Stream** | Append-only log with consumer groups. Like Kafka. | `XADD events * type "click"` |
| **GeoSpatial** | Store coordinates, query by radius. | `GEOADD locations 13.36 52.52 "Berlin"` |

## Redis CLI Commands

### Generic

```shell
# Set a key
SET name "Alice"

# Get a key
GET name

# Set with TTL (expires in 60 seconds)
SET session:abc "data" EX 60

# Set only if key does not exist (NX)
SET lock:resource "token" NX EX 30

# Delete a key
DEL name

# Check if key exists
EXISTS name

# Set TTL on existing key
EXPIRE name 60

# Get remaining TTL in seconds
TTL name

# List all keys matching pattern (blocking; avoid in production)
KEYS user:*

# Scan keys iteratively (production-safe)
SCAN 0 MATCH user:* COUNT 100

# Rename key
RENAME name username

# Get type of key
TYPE name
```

### Strings

```shell
# Atomic increment/decrement
INCR counter
INCRBY counter 10
DECR counter

# Append to string
APPEND name " Smith"

# Get substring
GETRANGE name 0 4

# Set multiple keys
MSET name1 "Alice" name2 "Bob"

# Get multiple keys
MGET name1 name2
```

### Lists

```shell
# Push to left/right
LPUSH queue task1
RPUSH queue task2

# Pop from left/right
LPOP queue
RPOP queue

# Get range
LRANGE queue 0 -1

# Get length
LLEN queue

# Trim list to range
LTRIM queue 0 99
```

### Sets

```shell
# Add members
SADD admins "alice" "bob"

# Check membership
SISMEMBER admins "alice"

# Get all members
SMEMBERS admins

# Remove member
SREM admins "bob"

# Union / Intersect / Difference
SUNION set1 set2
SINTER set1 set2
SDIFF set1 set2

# Random member
SRANDMEMBER admins 1
```

### Sorted Sets

```shell
# Add member with score
ZADD leaderboard 100 "alice"  80 "bob"

# Increment score
ZINCRBY leaderboard 10 "alice"

# Get rank (0-based, ascending)
ZRANK leaderboard "alice"

# Get rank with score (descending)
ZREVRANK leaderboard "alice"

# Get range by rank
ZRANGE leaderboard 0 2 WITHSCORES

# Get range by score
ZRANGEBYSCORE leaderboard 80 100

# Get count of members in score range
ZCOUNT leaderboard 80 100
```

### Hashes

```shell
# Set field
HSET user:1 name "Alice"
HSET user:1 age 30 email "alice@example.com"

# Get field
HGET user:1 name

# Get all fields and values
HGETALL user:1

# Get multiple fields
HMGET user:1 name email

# Increment field (atomic)
HINCRBY user:1 age 1

# Delete field
HDEL user:1 email

# Get all field names
HKEYS user:1

# Get all values
HVALS user:1
```

### Streams

```shell
# Add entry
XADD events * type "click" user_id 42 page "home"

# Get length
XLEN events

# Read from start
XRANGE events - +

# Read new entries (blocking)
XREAD BLOCK 0 STREAMS events $

# Create consumer group
XGROUP CREATE events mygroup $

# Read as consumer group
XREADGROUP GROUP mygroup consumer1 COUNT 10 BLOCK 1000 STREAMS events >
```

### Pub/Sub

```shell
# Subscribe to channel (blocking)
SUBSCRIBE notifications

# Publish message
PUBLISH notifications "Hello subscribers"
```

## Persistence

Redis is in-memory but offers optional persistence:

| Option | Description | When to use |
|--------|-------------|-------------|
| **RDB** (snapshot) | Point-in-time dump to disk every N seconds / M changes. Compact file. | Cache use cases, data loss acceptable |
| **AOF** (append-only) | Logs every write operation. Replay on restart. Durable. | Durability critical, slower rewrites |
| **RDB + AOF** | Both enabled. AOF used on restart (more complete). | Production data persistence |
| **None** | No persistence. Pure cache. | Ephemeral data, session with DB source of truth |

```shell
# Trigger RDB snapshot
SAVE     # synchronous (blocking)
BGSAVE   # background fork (non-blocking)

# Rewrite AOF (compact log)
BGREWRITEAOF
```

Configuration (`redis.conf`):

```conf
# RDB: save every 60s if at least 1000 keys changed
save 60 1000

# AOF: fsync every second
appendonly yes
appendfsync everysec
```

## Transactions

Redis transactions use `MULTI` / `EXEC` / `DISCARD`.

```shell
MULTI
SET balance:alice 100
SET balance:bob 200
EXEC
```

- `MULTI` queues commands
- `EXEC` executes all atomically (no interleaving from other clients)
- `DISCARD` discards the queue
- **No rollback**: if a command in the queue fails, others still execute (syntax errors cancel all, runtime errors let others proceed)

```shell
MULTI
SET key1 "value1"
INCR key1      # runtime error (key1 is a string, not integer)
SET key2 "value2"
EXEC
# key1 and key2 are both set; INCR fails silently
```

### Lua scripting for atomicity

```lua
-- Atomic transfer: decrement Alice, increment Bob
EVAL "redis.call('DECRBY', KEYS[1], ARGV[1]); redis.call('INCRBY', KEYS[2], ARGV[1])" 2 balance:alice balance:bob 50
```

Lua scripts run atomically; no other commands execute during the script.

## Replication and High Availability

### Replication (master-replica)

```shell
# Replica connects to master
REPLICAOF 192.168.1.10 6379
```

- Asynchronous replication by default
- Replicas can accept reads (scale read throughput)
- `WAIT` command blocks until N replicas acknowledge write

### Sentinel (high availability)

```
Redis Sentinel monitors masters/replicas and performs automatic failover.
```

### Redis Cluster (sharding)

```
Redis Cluster partitions data across 16384 hash slots across multiple nodes.
```

- Automatic sharding and rebalancing
- Partial availability during partitions (majority side stays up)
- No cross-slot transactions or multi-key operations

## Eviction Policies

When `maxmemory` is reached, Redis evicts keys:

| Policy | Behavior |
|--------|----------|
| `noeviction` | Return errors on writes (default) |
| `allkeys-lru` | Evict least recently used keys |
| `allkeys-lfu` | Evict least frequently used keys |
| `volatile-lru` | Evict LRU among keys with TTL set |
| `volatile-lfu` | Evict LFU among keys with TTL set |
| `allkeys-random` | Evict random key |
| `volatile-random` | Evict random key with TTL set |
| `volatile-ttl` | Evict key with shortest remaining TTL |

```conf
maxmemory 4gb
maxmemory-policy allkeys-lru
```

## Supported datatypes (summary)

- String
- List
- Set
- Sorted Set (ZSet)
- Hash
- BitMap
- HyperLogLog
- Stream
- GeoSpatial (via Geo commands on ZSet encoding)
- JSON (via RedisJSON module)
- Graph (via RedisGraph module)
- Time Series (via RedisTimeSeries module)
- Bloom / Cuckoo Filters (via RedisBloom module)
