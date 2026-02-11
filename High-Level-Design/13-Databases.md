# Databases - Complete Guide

## What is a Database?

Think of Database like a **well-organized filing cabinet**:
- Store information
- Find information quickly
- Keep information safe
- Multiple people can access

---

## Types of Databases

### 1. Relational Databases (SQL)
```
Tables with rows and columns (like Excel)

Users Table:
┌────────┬──────────┬─────────────────────┐
│   id   │   name   │       email         │
├────────┼──────────┼─────────────────────┤
│   1    │  John    │  john@gmail.com     │
│   2    │  Jane    │  jane@gmail.com     │
│   3    │  Bob     │  bob@gmail.com      │
└────────┴──────────┴─────────────────────┘

Examples: PostgreSQL, MySQL, Oracle, SQL Server
```

### 2. Document Databases (NoSQL)
```
Flexible JSON-like documents

{
    "_id": "abc123",
    "name": "John",
    "email": "john@gmail.com",
    "preferences": {
        "theme": "dark",
        "notifications": true
    },
    "tags": ["developer", "premium"]
}

Examples: MongoDB, CouchDB, Firebase
```

### 3. Key-Value Stores
```
Simple key → value pairs

"user:123" → "{name: 'John', email: 'john@gmail.com'}"
"session:abc" → "{user_id: 123, expires: '...'}"
"cache:trending" → "[video1, video2, video3]"

Examples: Redis, DynamoDB, Memcached
```

### 4. Wide-Column Stores
```
Like a spreadsheet, but columns can vary per row

Row Key    | Column1     | Column2      | Column3
─────────────────────────────────────────────────
user:1     | name:John   | email:j@..   |
user:2     | name:Jane   | email:...    | age:25
user:3     | name:Bob    |              | city:NYC

Examples: Cassandra, HBase, ScyllaDB
```

### 5. Graph Databases
```
Nodes and relationships

    (John)──FOLLOWS──▶(Jane)
       │                 │
   LIKES               LIKES
       ▼                 ▼
   (Video1)          (Video2)
       │
    HAS_COMMENT
       ▼
   (Comment1)

Examples: Neo4j, Amazon Neptune, JanusGraph
```

### 6. Time-Series Databases
```
Optimized for time-stamped data

Timestamp           | Metric      | Value
────────────────────────────────────────────
2024-01-01 10:00:00 | cpu_usage   | 45.2
2024-01-01 10:00:01 | cpu_usage   | 46.1
2024-01-01 10:00:02 | cpu_usage   | 44.8

Examples: InfluxDB, TimescaleDB, Prometheus
```

### 7. Search Engines (Text Search)
```
Full-text search on documents

Index: "How to cook pasta"
Search: "cooking pasta" → FOUND!

Examples: Elasticsearch, Algolia, Solr
```

---

## Database Selection Guide

```
┌─────────────────────────────────────────────────────────────────┐
│                   WHICH DATABASE TO USE?                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Need transactions, integrity?                                  │
│   Complex queries with JOINs?                                    │
│   ────────────────────────────▶ PostgreSQL / MySQL              │
│                                                                  │
│   Flexible schema?                                               │
│   JSON-like data?                                                │
│   ────────────────────────────▶ MongoDB                         │
│                                                                  │
│   Super fast reads/writes?                                       │
│   Caching, sessions?                                             │
│   ────────────────────────────▶ Redis                           │
│                                                                  │
│   Massive scale, high write volume?                              │
│   Eventual consistency OK?                                       │
│   ────────────────────────────▶ Cassandra                       │
│                                                                  │
│   Complex relationships?                                         │
│   Social networks, recommendations?                              │
│   ────────────────────────────▶ Neo4j                           │
│                                                                  │
│   Full-text search?                                              │
│   Log analytics?                                                 │
│   ────────────────────────────▶ Elasticsearch                   │
│                                                                  │
│   Metrics, IoT, monitoring?                                      │
│   Time-based queries?                                            │
│   ────────────────────────────▶ InfluxDB / TimescaleDB          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## PostgreSQL Deep Dive (Most Popular SQL)

### Creating Tables:
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE videos (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    views BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    video_id INTEGER REFERENCES videos(id),
    user_id INTEGER REFERENCES users(id),
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Queries:
```sql
-- Get videos with user info
SELECT v.*, u.name as author_name
FROM videos v
JOIN users u ON v.user_id = u.id
WHERE v.views > 1000
ORDER BY v.created_at DESC
LIMIT 10;

-- Count comments per video
SELECT v.title, COUNT(c.id) as comment_count
FROM videos v
LEFT JOIN comments c ON v.id = c.video_id
GROUP BY v.id
ORDER BY comment_count DESC;

-- Get user with their video count
SELECT u.*, COUNT(v.id) as video_count
FROM users u
LEFT JOIN videos v ON u.id = v.user_id
GROUP BY u.id;
```

### Indexes (For Speed!):
```sql
-- Create indexes on frequently queried columns
CREATE INDEX idx_videos_user_id ON videos(user_id);
CREATE INDEX idx_videos_created_at ON videos(created_at DESC);
CREATE INDEX idx_comments_video_id ON comments(video_id);

-- Full-text search index
CREATE INDEX idx_videos_title_search ON videos 
USING GIN(to_tsvector('english', title));
```

---

## MongoDB Deep Dive (Most Popular NoSQL)

### Documents:
```javascript
// User document
{
    "_id": ObjectId("..."),
    "name": "John Doe",
    "email": "john@example.com",
    "profile": {
        "bio": "I love making videos!",
        "avatar": "url...",
        "social": {
            "twitter": "@john",
            "instagram": "john_doe"
        }
    },
    "createdAt": ISODate("2024-01-15T10:30:00Z")
}

// Video document
{
    "_id": ObjectId("..."),
    "title": "How to Cook Pasta",
    "description": "Learn cooking...",
    "userId": ObjectId("..."),
    "author": {
        "name": "John Doe",  // Denormalized for fast reads
        "avatar": "url..."
    },
    "stats": {
        "views": 15000,
        "likes": 500,
        "comments": 120
    },
    "tags": ["cooking", "pasta", "italian"],
    "createdAt": ISODate("2024-01-15T10:30:00Z")
}
```

### Queries:
```javascript
// Find videos by user
db.videos.find({ userId: ObjectId("...") });

// Find with multiple conditions
db.videos.find({
    "stats.views": { $gt: 1000 },
    tags: "cooking",
    createdAt: { $gte: ISODate("2024-01-01") }
}).sort({ "stats.views": -1 }).limit(10);

// Aggregation pipeline (like SQL GROUP BY)
db.videos.aggregate([
    { $match: { "stats.views": { $gt: 100 } } },
    { $group: {
        _id: "$userId",
        totalViews: { $sum: "$stats.views" },
        videoCount: { $sum: 1 }
    }},
    { $sort: { totalViews: -1 } },
    { $limit: 10 }
]);

// Text search
db.videos.find({
    $text: { $search: "cooking pasta" }
});
```

### Indexes:
```javascript
// Single field index
db.videos.createIndex({ userId: 1 });

// Compound index
db.videos.createIndex({ userId: 1, createdAt: -1 });

// Text index for search
db.videos.createIndex({ title: "text", description: "text" });

// TTL index (auto-delete old data)
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 86400 });
```

---

## Redis Deep Dive (Cache & More)

### Data Types:
```bash
# Strings
SET user:123:name "John"
GET user:123:name  # "John"

# Expiration
SET session:abc "data" EX 3600  # Expires in 1 hour

# Counters
INCR video:123:views  # Atomic increment
INCRBY video:123:views 10

# Hashes (like objects)
HSET user:123 name "John" email "john@example.com" age 25
HGET user:123 name  # "John"
HGETALL user:123  # {name: "John", email: "...", age: 25}

# Lists (queues)
LPUSH notifications:123 "new comment"
RPOP notifications:123

# Sets (unique items)
SADD user:123:liked video:1 video:2 video:3
SISMEMBER user:123:liked video:2  # true
SMEMBERS user:123:liked  # [video:1, video:2, video:3]

# Sorted Sets (leaderboards)
ZADD trending 1000 video:1 2000 video:2 500 video:3
ZREVRANGE trending 0 9 WITHSCORES  # Top 10

# Pub/Sub
PUBLISH channel:chat "Hello everyone!"
SUBSCRIBE channel:chat
```

### Use Cases:
```
┌─────────────────────────────────────────────────────────────────┐
│                    REDIS USE CASES                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Caching           → Store DB query results, API responses     │
│   Sessions          → Store user session data                   │
│   Rate Limiting     → Track request counts per user             │
│   Leaderboards      → Sorted sets for rankings                  │
│   Real-time         → Pub/Sub for chat, notifications           │
│   Queues            → Lists for job queues                      │
│   Counting          → View counts, likes (atomic increment)     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Database Scaling

### Read Replicas (Read Scaling):
```
                    ┌─────────────┐
   Writes ─────────▶│   Primary   │
                    │  (Master)   │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │ Replication │
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌───────────┐   ┌───────────┐   ┌───────────┐
    │ Replica 1 │   │ Replica 2 │   │ Replica 3 │
    │  (Reads)  │   │  (Reads)  │   │  (Reads)  │
    └───────────┘   └───────────┘   └───────────┘

Reads distributed across replicas!
```

### Sharding (Write Scaling):
```
         ┌─────────────────────────────────────┐
         │          Shard Router               │
         └─────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   ┌─────────┐      ┌─────────┐      ┌─────────┐
   │ Shard 1 │      │ Shard 2 │      │ Shard 3 │
   │Users A-H│      │Users I-P│      │Users Q-Z│
   └─────────┘      └─────────┘      └─────────┘

Writes distributed across shards!
```

---

## Database for YouTube Clone

### Multi-Database Architecture:
```
┌─────────────────────────────────────────────────────────────────┐
│                    YOUTUBE CLONE DATABASES                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   PostgreSQL (Users, Payments)                                   │
│   ├── Strong consistency needed                                  │
│   ├── Transactions for payments                                  │
│   └── Complex user queries                                       │
│                                                                  │
│   MongoDB (Videos, Comments)                                     │
│   ├── Flexible video metadata                                    │
│   ├── Denormalized for reads                                     │
│   └── Easy to shard by video_id                                  │
│                                                                  │
│   Redis (Cache, Sessions, Counters)                              │
│   ├── Video view counts                                          │
│   ├── Session storage                                            │
│   ├── API response caching                                       │
│   └── Rate limiting                                              │
│                                                                  │
│   Cassandra (Analytics, Watch History)                           │
│   ├── High write throughput                                      │
│   ├── Time-series data                                           │
│   └── Massive scale                                              │
│                                                                  │
│   Elasticsearch (Search)                                         │
│   ├── Full-text video search                                     │
│   ├── Auto-complete                                              │
│   └── Filter by category, date                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Schema Design:
```javascript
// PostgreSQL - Users
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE subscriptions (
    user_id UUID REFERENCES users(id),
    channel_id UUID REFERENCES users(id),
    subscribed_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (user_id, channel_id)
);

// MongoDB - Videos
{
    "_id": ObjectId("..."),
    "title": "My Video",
    "description": "...",
    "channelId": "uuid",
    "channelInfo": {  // Denormalized
        "name": "John's Channel",
        "avatar": "url"
    },
    "status": "published",
    "files": {
        "original": "s3://...",
        "360p": "s3://...",
        "720p": "s3://...",
        "1080p": "s3://..."
    },
    "thumbnail": "url",
    "duration": 600,
    "tags": ["gaming", "tutorial"],
    "stats": {
        "views": 10000,
        "likes": 500,
        "dislikes": 10
    }
}

// Redis - Counters & Cache
SET video:abc:views 15000
INCR video:abc:views

SET cache:trending "[...]" EX 300

ZADD channel:xyz:subscribers 1705312200 "user:123"

// Cassandra - Watch History
CREATE TABLE watch_history (
    user_id UUID,
    watched_at TIMESTAMP,
    video_id UUID,
    watch_duration INT,
    PRIMARY KEY (user_id, watched_at)
) WITH CLUSTERING ORDER BY (watched_at DESC);
```

---

## Interview Questions

**Q: When would you use SQL vs NoSQL?**
A: SQL for transactions, complex queries, structured data. NoSQL for flexibility, scale, and simple queries.

**Q: How to scale a database?**
A: First vertical (bigger server). Then read replicas. Then caching. Finally sharding.

**Q: What is normalization vs denormalization?**
A: Normalization: No duplicate data, use JOINs. Denormalization: Duplicate data for faster reads, avoid JOINs.

**Q: How does indexing work?**
A: Creates a data structure (B-tree) for fast lookups. Without index: scan all rows. With index: direct lookup.

**Q: When would you use Redis?**
A: Caching, sessions, real-time features, counters, leaderboards, rate limiting. Anything needing sub-millisecond access.

---

## Quick Summary

```
DATABASE TYPES:
- SQL (PostgreSQL): Transactions, JOINs, structured
- Document (MongoDB): Flexible schema, JSON-like
- Key-Value (Redis): Caching, sessions, super fast
- Wide-Column (Cassandra): Massive writes, time-series
- Graph (Neo4j): Relationships, social networks
- Search (Elasticsearch): Full-text search

SELECTION GUIDE:
- Need transactions? → PostgreSQL
- Need flexibility? → MongoDB
- Need speed? → Redis
- Need massive scale? → Cassandra
- Need search? → Elasticsearch

SCALING:
- Vertical → Bigger server
- Read replicas → More read capacity
- Sharding → More write capacity
```

You now understand Databases like a pro! 🚀
