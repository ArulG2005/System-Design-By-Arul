# Data Sharding & Partitioning - Complete Guide

## What's the Difference?

### Partitioning = Dividing data within ONE database
```
┌─────────────────────────────────────────────────┐
│                 SINGLE DATABASE                  │
│                                                  │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐     │
│  │Partition 1│ │Partition 2│ │Partition 3│     │
│  │ Jan-Apr   │ │ May-Aug   │ │ Sep-Dec   │     │
│  └───────────┘ └───────────┘ └───────────┘     │
│                                                  │
│  Same machine, different storage partitions      │
└─────────────────────────────────────────────────┘
```

### Sharding = Dividing data across MULTIPLE databases
```
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│   SERVER 1    │ │   SERVER 2    │ │   SERVER 3    │
│   (Shard A)   │ │   (Shard B)   │ │   (Shard C)   │
│               │ │               │ │               │
│  Users A-H    │ │  Users I-P    │ │  Users Q-Z    │
└───────────────┘ └───────────────┘ └───────────────┘

Different machines, different databases!
```

---

## Partitioning Types

### 1. Horizontal Partitioning (Row-based)
```
Original Table (1M rows):
┌────────┬──────────┬───────────┐
│   id   │   name   │  created  │
├────────┼──────────┼───────────┤
│   1    │  John    │  Jan 5    │
│   2    │  Jane    │  May 10   │
│   ...  │  ...     │  ...      │
│ 1M     │  Bob     │  Dec 20   │
└────────┴──────────┴───────────┘

After Horizontal Partitioning:
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  Partition Q1   │ │  Partition Q2   │ │  Partition Q3   │
│  (Jan-Mar)      │ │  (Apr-Jun)      │ │  (Jul-Sep)      │
│  250K rows      │ │  250K rows      │ │  250K rows      │
└─────────────────┘ └─────────────────┘ └─────────────────┘

Rows split by date range!
```

### 2. Vertical Partitioning (Column-based)
```
Original Table:
┌────────┬───────┬────────────────┬──────────────┐
│   id   │ name  │  profile_pic   │ preferences  │
├────────┼───────┼────────────────┼──────────────┤
│   1    │ John  │  (large blob)  │ {json...}    │
└────────┴───────┴────────────────┴──────────────┘

After Vertical Partitioning:
┌────────┬───────┐    ┌────────┬────────────────┐
│   id   │ name  │    │   id   │  profile_pic   │
├────────┼───────┤    ├────────┼────────────────┤
│   1    │ John  │    │   1    │  (large blob)  │
└────────┴───────┘    └────────┴────────────────┘
  (Hot data - fast)     (Cold data - slower)

Columns split by access pattern!
```

---

## Partitioning Strategies

### 1. Range Partitioning
```sql
-- PostgreSQL Example
CREATE TABLE videos (
    id UUID,
    title VARCHAR(255),
    created_at TIMESTAMP
) PARTITION BY RANGE (created_at);

CREATE TABLE videos_2023 PARTITION OF videos
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE videos_2024 PARTITION OF videos
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Query "videos from 2024" only scans videos_2024 partition!
```

```
Good for: Time-series data, logs, analytics
Problem: Uneven distribution (newest partition gets all writes)
```

### 2. List Partitioning
```sql
CREATE TABLE users (
    id UUID,
    name VARCHAR(255),
    country VARCHAR(2)
) PARTITION BY LIST (country);

CREATE TABLE users_us PARTITION OF users
    FOR VALUES IN ('US');

CREATE TABLE users_eu PARTITION OF users
    FOR VALUES IN ('DE', 'FR', 'UK', 'ES', 'IT');

CREATE TABLE users_asia PARTITION OF users
    FOR VALUES IN ('IN', 'JP', 'CN', 'KR');
```

```
Good for: Geographic data, categorical data
Problem: Need to know all possible values
```

### 3. Hash Partitioning
```sql
CREATE TABLE comments (
    id UUID,
    video_id UUID,
    content TEXT
) PARTITION BY HASH (video_id);

CREATE TABLE comments_p0 PARTITION OF comments
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE comments_p1 PARTITION OF comments
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE comments_p2 PARTITION OF comments
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE comments_p3 PARTITION OF comments
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

```
Good for: Even distribution, unknown values
Problem: Range queries need all partitions
```

---

## Sharding in Detail

### Shard Key Selection

The MOST IMPORTANT decision! Cannot change easily later.

```
Good Shard Key Properties:
───────────────────────────────────────────────
1. High Cardinality
   Bad:  country (only ~200 values)
   Good: user_id (millions of values)

2. Even Distribution
   Bad:  created_at (recent data hot)
   Good: hash(user_id) (uniform spread)

3. Query Isolation
   Bad:  If you always query by video_id but shard by user_id
   Good: Shard by the field you query most

4. Write Distribution
   Bad:  All new users on one shard
   Good: New users spread across shards
```

### Sharding Example for YouTube

```javascript
// Shard key decision for different collections

// Videos: Shard by video_id
// Why? Most queries are "GET video by ID"
db.adminCommand({
    shardCollection: "youtube.videos",
    key: { video_id: "hashed" }
});

// Comments: Shard by video_id (NOT user_id)
// Why? "Get comments for video" is common query
db.adminCommand({
    shardCollection: "youtube.comments",
    key: { video_id: "hashed" }
});

// Watch History: Shard by user_id
// Why? "Get my watch history" is the main query
db.adminCommand({
    shardCollection: "youtube.watch_history",
    key: { user_id: "hashed" }
});

// Subscriptions: Compound shard key
// Why? Need both "my subscriptions" and "channel subscribers"
db.adminCommand({
    shardCollection: "youtube.subscriptions",
    key: { channel_id: 1, user_id: 1 }
});
```

---

## Cross-Shard Queries

### The Challenge:
```
Query: "Get all videos with >1M views"

Without shard key in query:
┌─────────────┐          ┌─────────────┐
│   Shard 1   │          │   Shard 2   │
│   Query...  │          │   Query...  │
└──────┬──────┘          └──────┬──────┘
       │                        │
       └────────────┬───────────┘
                    │
            ┌───────▼───────┐
            │    COMBINE    │
            │   RESULTS     │
            └───────────────┘

SLOW! Must scan all shards (scatter-gather)
```

### Solutions:

#### 1. Design Queries to Include Shard Key
```javascript
// Bad: Scatter-gather query
db.videos.find({ views: { $gt: 1000000 } });

// Good: Single shard query (include shard key)
db.videos.find({ 
    channel_id: "abc123",
    views: { $gt: 1000000 } 
});
```

#### 2. Denormalization
```javascript
// Store data redundantly to avoid cross-shard joins

// Video document includes channel info
{
    video_id: "xyz",
    title: "My Video",
    channel_id: "abc",
    channel_name: "John's Channel",  // Denormalized!
    channel_avatar: "url..."          // Denormalized!
}

// Now don't need to fetch from channel shard
```

#### 3. Global/Reference Tables
```javascript
// Small tables replicated across all shards

// Categories table (small, rarely changes)
// Copied to every shard
{
    id: "gaming",
    name: "Gaming",
    icon: "🎮"
}

// No need to cross-shard query for categories
```

---

## Rebalancing Shards

### When Data Becomes Uneven:
```
Before:
Shard 1: 100GB  ███████████
Shard 2: 500GB  ████████████████████████████████████████████████████
Shard 3: 150GB  ███████████████

Shard 2 is overloaded!

After Rebalancing:
Shard 1: 250GB  █████████████████████████
Shard 2: 250GB  █████████████████████████
Shard 3: 250GB  █████████████████████████
```

### Rebalancing Strategies:

#### 1. Virtual Shards (Chunks)
```
4 Physical Shards, 64 Virtual Chunks

Shard 1: Chunks 1-16
Shard 2: Chunks 17-32
Shard 3: Chunks 33-48
Shard 4: Chunks 49-64

Adding Shard 5?
Just move some chunks:
Shard 1: Chunks 1-12
Shard 2: Chunks 17-29
Shard 3: Chunks 33-45
Shard 4: Chunks 49-60
Shard 5: Chunks 13-16, 30-32, 46-48, 61-64

No data within chunks needs to move!
```

#### 2. Consistent Hashing
```
See Consistent Hashing file for details!

Key benefit: Adding/removing shard only moves
K/N of data (K=total keys, N=shards)
```

---

## Sharding Architectures

### 1. Application-Level Sharding
```javascript
class ShardRouter {
    constructor(shards) {
        this.shards = shards; // Map of shard connections
    }
    
    getShardForUser(userId) {
        // Simple hash-based routing
        const hash = this.hashCode(userId);
        const shardIndex = hash % this.shards.length;
        return this.shards[shardIndex];
    }
    
    async findUser(userId) {
        const shard = this.getShardForUser(userId);
        return await shard.users.findOne({ _id: userId });
    }
    
    async findAllUsers(criteria) {
        // Must query all shards!
        const results = await Promise.all(
            this.shards.map(shard => 
                shard.users.find(criteria).toArray()
            )
        );
        return results.flat();
    }
}
```

**Pros:** Full control, no middleware cost
**Cons:** Complex application logic, tight coupling

### 2. Proxy-Level Sharding
```
┌─────────────┐
│ Application │
└──────┬──────┘
       │
┌──────▼──────┐
│   PROXY     │ ← Shard routing logic here
│ (Vitess/    │
│  ProxySQL)  │
└──────┬──────┘
       │
┌──────┼──────┐
▼      ▼      ▼
DB1   DB2   DB3
```

**Pros:** Transparent to application
**Cons:** Added latency, another component to manage

### 3. Database-Level Sharding
```
MongoDB with Auto-sharding:

┌─────────────┐
│ Application │
└──────┬──────┘
       │
┌──────▼──────┐
│   mongos    │ ← Query router
│  (router)   │
└──────┬──────┘
       │
┌──────┼──────┐──────┐
▼      ▼      ▼      ▼
Shard1 Shard2 Shard3 Config
                     Servers

Database handles everything!
```

**Pros:** Built-in, well-tested
**Cons:** Vendor lock-in, specific to database

---

## Implementation Example

### PostgreSQL Partitioning:
```sql
-- Create partitioned table
CREATE TABLE watch_history (
    id UUID DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    video_id UUID NOT NULL,
    watched_at TIMESTAMP NOT NULL,
    watch_duration INTEGER
) PARTITION BY RANGE (watched_at);

-- Create monthly partitions
CREATE TABLE watch_history_2024_01 
    PARTITION OF watch_history
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE watch_history_2024_02 
    PARTITION OF watch_history
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Indexes on each partition
CREATE INDEX ON watch_history_2024_01 (user_id, watched_at DESC);
CREATE INDEX ON watch_history_2024_02 (user_id, watched_at DESC);

-- Query automatically uses correct partition
EXPLAIN SELECT * FROM watch_history 
WHERE watched_at >= '2024-01-15' 
  AND watched_at < '2024-01-20';
-- Only scans watch_history_2024_01!
```

### MongoDB Sharding:
```javascript
// Enable sharding on database
sh.enableSharding("youtube");

// Create index on shard key
db.videos.createIndex({ channel_id: "hashed" });

// Shard the collection
sh.shardCollection("youtube.videos", { channel_id: "hashed" });

// Check shard distribution
db.videos.getShardDistribution();

// Output:
// Shard shard0: 33.33% (1000000 docs)
// Shard shard1: 33.33% (1000000 docs)
// Shard shard2: 33.34% (1000001 docs)
```

### Cassandra Partitioning:
```sql
-- Create table with partition key
CREATE TABLE videos (
    channel_id UUID,
    video_id UUID,
    title TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY ((channel_id), created_at, video_id)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- channel_id is partition key
-- Each channel's videos on same partition
-- Querying one channel = one partition (fast!)

-- Query pattern
SELECT * FROM videos 
WHERE channel_id = 'abc123'
ORDER BY created_at DESC
LIMIT 10;
```

---

## Best Practices

### 1. Plan Shard Key Before You Start
```
Questions to ask:
- What's my most common query?
- What field has highest cardinality?
- How will data grow?
- What queries must be fast?
```

### 2. Start with Enough Shards
```
Bad: Start with 2 shards, need 10 later
     Massive rebalancing needed!

Good: Start with 8-16 shards
      Room to grow, easier rebalancing
```

### 3. Monitor Distribution
```javascript
// Check for hot shards
db.stats().shards.forEach(shard => {
    console.log(`${shard.name}: ${shard.dataSize}GB`);
});

// Alert if any shard > 2x average
```

### 4. Automate Partition Management
```sql
-- Auto-create future partitions
CREATE OR REPLACE FUNCTION create_partition_if_not_exists()
RETURNS TRIGGER AS $$
BEGIN
    -- Create partition for next month if doesn't exist
    EXECUTE format(
        'CREATE TABLE IF NOT EXISTS watch_history_%s 
         PARTITION OF watch_history FOR VALUES FROM (%L) TO (%L)',
        to_char(NEW.watched_at, 'YYYY_MM'),
        date_trunc('month', NEW.watched_at),
        date_trunc('month', NEW.watched_at) + interval '1 month'
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

---

## Interview Questions

**Q: Difference between partitioning and sharding?**
A: Partitioning splits data within one database. Sharding splits data across multiple database servers.

**Q: How to choose a shard key?**
A: High cardinality, even distribution, matches query patterns, avoids monotonic increase.

**Q: What about cross-shard joins?**
A: Avoid them! Denormalize data, use application-level joins, or restructure queries to include shard key.

**Q: How to add a new shard?**
A: Use virtual shards/chunks, consistent hashing, or database's built-in rebalancing. Minimize data movement.

**Q: What if shard key was wrong choice?**
A: Very expensive to change. Options: dual-write migration, complete rebuild. Prevention is key!

---

## Quick Summary

```
PARTITIONING:
- Within one DB server
- Improves query speed
- Types: Range, List, Hash
- PostgreSQL: Built-in support

SHARDING:
- Across multiple DB servers
- Improves scalability
- Need careful shard key selection
- MongoDB: Built-in support

KEY DECISIONS:
- Shard key (most important!)
- Sharding strategy (hash vs range)
- Cross-shard query handling
- Rebalancing strategy

COMMON PATTERNS:
- Shard by user_id (for user data)
- Shard by video_id (for videos)
- Partition by time (for logs/history)
```

You now understand Data Sharding & Partitioning like a pro! 🚀
