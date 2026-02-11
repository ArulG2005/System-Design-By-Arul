# SQL vs NoSQL - Complete Guide

## The Core Difference

Think of it like **organizing books**:

**SQL (Library System):**
- Fixed categories (Fiction, Science, History)
- Every book catalogued the same way
- Find by browsing organized shelves
- Relationships: "This author wrote these books"

**NoSQL (Personal Bookshelf):**
- Organize however you want
- Some books have notes, some don't
- Fast to grab what you need
- Can change organization anytime

---

## SQL Databases

### What is SQL?
SQL = **S**tructured **Q**uery **L**anguage

```
Relational Database:
┌──────────────────────────────────────────────────┐
│                      users                        │
├──────────┬──────────┬──────────────┬─────────────┤
│    id    │   name   │    email     │ created_at  │
├──────────┼──────────┼──────────────┼─────────────┤
│    1     │  John    │ john@email   │ 2024-01-01  │
│    2     │  Jane    │ jane@email   │ 2024-01-02  │
└──────────┴──────────┴──────────────┴─────────────┘

Every row has EXACT same columns
Fixed structure (schema)
```

### Key Characteristics

```
1. STRUCTURED (Schema)
   - Define columns before inserting data
   - Every row follows same structure
   
2. RELATIONAL
   - Tables linked via foreign keys
   - JOIN tables to get combined data
   
3. ACID Compliant
   - Atomic: All or nothing
   - Consistent: Data always valid
   - Isolated: Transactions don't interfere
   - Durable: Data persists after commit

4. VERTICAL SCALING
   - Add more CPU/RAM to one server
   - Harder to distribute
```

### Example: YouTube in SQL
```sql
-- Users Table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Videos Table (Foreign Key to users)
CREATE TABLE videos (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    title VARCHAR(200) NOT NULL,
    description TEXT,
    url VARCHAR(500) NOT NULL,
    views INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Comments Table (Foreign Keys to both)
CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    video_id INTEGER REFERENCES videos(id),
    user_id INTEGER REFERENCES users(id),
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Query with JOIN
SELECT 
    v.title,
    u.username,
    COUNT(c.id) as comment_count
FROM videos v
JOIN users u ON v.user_id = u.id
LEFT JOIN comments c ON c.video_id = v.id
GROUP BY v.id, u.username
ORDER BY v.views DESC
LIMIT 10;
```

### Popular SQL Databases
```
┌────────────────┬──────────────────────────────────┐
│   Database     │          Best For                │
├────────────────┼──────────────────────────────────┤
│   PostgreSQL   │  Complex queries, JSON support   │
│   MySQL        │  Web apps, WordPress             │
│   SQLite       │  Embedded, mobile apps           │
│   SQL Server   │  Enterprise, .NET apps           │
│   Oracle       │  Large enterprise                │
└────────────────┴──────────────────────────────────┘
```

---

## NoSQL Databases

### What is NoSQL?
NoSQL = **N**ot **O**nly **SQL**

```
Flexible Structure:
{
    "id": "abc123",
    "name": "John",
    "email": "john@email.com",
    "profile": {           ← Nested object (no separate table!)
        "bio": "Developer",
        "avatar": "url..."
    },
    "tags": ["tech", "gaming"],  ← Array (no separate table!)
    "extra_field": "anything"    ← Can add any field!
}
```

### Types of NoSQL Databases

#### 1. Document Stores
```
Store: JSON-like documents
Example: MongoDB, CouchDB

{
    "_id": "video_abc123",
    "title": "My Video",
    "uploader": {
        "id": "user_456",
        "name": "John"
    },
    "tags": ["gaming", "tutorial"],
    "stats": {
        "views": 10000,
        "likes": 500
    }
}

Good for: Flexible data, nested structures
```

#### 2. Key-Value Stores
```
Store: Simple key → value pairs
Example: Redis, DynamoDB

┌──────────────────┬───────────────────────────┐
│       Key        │          Value            │
├──────────────────┼───────────────────────────┤
│ user:123         │ {name: "John", age: 25}   │
│ session:abc      │ {token: "xyz", exp: 3600} │
│ cache:video:456  │ <serialized video data>   │
└──────────────────┴───────────────────────────┘

Good for: Caching, sessions, simple lookups
```

#### 3. Column-Family Stores
```
Store: Columns grouped together
Example: Cassandra, HBase

Row Key: user_123
┌──────────────────────────────────────────────────┐
│  profile (column family)  │  activity (column family)│
├───────────┬───────────────┼────────────┬─────────────┤
│   name    │    email      │ last_login │  page_views │
├───────────┼───────────────┼────────────┼─────────────┤
│   John    │ john@email    │ 2024-01-15 │    1500     │
└───────────┴───────────────┴────────────┴─────────────┘

Good for: Time-series, analytics, write-heavy
```

#### 4. Graph Databases
```
Store: Nodes and relationships
Example: Neo4j, Amazon Neptune

    (John)──FOLLOWS──►(Jane)
       │                 │
    LIKES             UPLOADED
       │                 │
       ▼                 ▼
   (Video 1)◄──RELATED──(Video 2)

Good for: Social networks, recommendations
```

---

## Head-to-Head Comparison

### Schema
```
SQL:
┌──────────────────────────────────────┐
│ MUST define schema before inserting  │
│ ALTER TABLE to change structure      │
│ Rigid, predictable                   │
└──────────────────────────────────────┘

NoSQL:
┌──────────────────────────────────────┐
│ Schema-less or flexible schema       │
│ Just insert different data           │
│ Flexible, adaptable                  │
└──────────────────────────────────────┘
```

### Relationships
```
SQL:
Users table ←─ Foreign Key ─→ Videos table
              JOIN to combine

NoSQL:
User document contains embedded videos
OR: Store user_id in video, query separately
No native JOINs (denormalized data)
```

### Scaling
```
SQL (Vertical):
┌─────────────────┐
│  One Big Server │
│  More CPU       │
│  More RAM       │
│  More Storage   │
└─────────────────┘
Limited by hardware

NoSQL (Horizontal):
┌─────────┐ ┌─────────┐ ┌─────────┐
│ Server 1│ │ Server 2│ │ Server 3│
└─────────┘ └─────────┘ └─────────┘
Add more servers (sharding built-in)
```

### Consistency vs Performance
```
SQL:
- Strong consistency (ACID)
- Slower writes (must ensure consistency)
- Good: Banking, inventory

NoSQL:
- Eventual consistency (BASE)
- Faster writes (acknowledge before sync)
- Good: Social media, logs
```

---

## When to Use What

### Use SQL When:
```
✓ Data has clear structure
✓ Relationships between data are important
✓ Need complex queries (JOINs, aggregations)
✓ ACID transactions required
✓ Data integrity is critical

Examples:
- Banking/Financial systems
- E-commerce (orders, inventory)
- ERP systems
- Traditional web apps
```

### Use NoSQL When:
```
✓ Schema changes frequently
✓ Massive scale needed
✓ High write throughput
✓ Flexible data structures
✓ Real-time analytics

Examples:
- Social media feeds
- IoT sensor data
- Real-time analytics
- Content management
- Caching
```

### YouTube Example:
```
SQL (PostgreSQL):
- User accounts
- Subscriptions
- Payment/billing
- Video metadata (core)

NoSQL (MongoDB):
- Video metadata (extended)
- User preferences
- Recommendations

NoSQL (Redis):
- Session cache
- View counts
- Rate limiting

NoSQL (Cassandra):
- Analytics data
- View history
- Time-series metrics

NoSQL (Neo4j):
- User relationships
- Video recommendations
- "Users who watched this also..."
```

---

## Code Examples

### SQL (PostgreSQL)
```javascript
// Using node-postgres
const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Create tables
await pool.query(`
    CREATE TABLE IF NOT EXISTS videos (
        id SERIAL PRIMARY KEY,
        title VARCHAR(200) NOT NULL,
        user_id INTEGER REFERENCES users(id),
        views INTEGER DEFAULT 0
    )
`);

// Insert with transaction
const client = await pool.connect();
try {
    await client.query('BEGIN');
    
    const userResult = await client.query(
        'INSERT INTO users (username, email) VALUES ($1, $2) RETURNING id',
        ['john', 'john@email.com']
    );
    
    await client.query(
        'INSERT INTO videos (title, user_id) VALUES ($1, $2)',
        ['My Video', userResult.rows[0].id]
    );
    
    await client.query('COMMIT');
} catch (e) {
    await client.query('ROLLBACK');
    throw e;
} finally {
    client.release();
}

// Complex query
const result = await pool.query(`
    SELECT 
        v.title,
        u.username,
        v.views,
        COUNT(c.id) as comments
    FROM videos v
    JOIN users u ON v.user_id = u.id
    LEFT JOIN comments c ON c.video_id = v.id
    WHERE v.views > 1000
    GROUP BY v.id, u.username
    ORDER BY v.views DESC
    LIMIT 10
`);
```

### NoSQL (MongoDB)
```javascript
// Using mongoose
const mongoose = require('mongoose');

// Define schema (optional but recommended)
const videoSchema = new mongoose.Schema({
    title: String,
    uploader: {
        userId: mongoose.Types.ObjectId,
        username: String  // Denormalized!
    },
    views: { type: Number, default: 0 },
    tags: [String],
    comments: [{
        userId: mongoose.Types.ObjectId,
        username: String,
        content: String,
        createdAt: Date
    }],
    metadata: mongoose.Schema.Types.Mixed  // Any structure!
});

const Video = mongoose.model('Video', videoSchema);

// Insert (no transaction needed for single doc)
const video = await Video.create({
    title: 'My Video',
    uploader: {
        userId: user._id,
        username: user.username
    },
    tags: ['gaming', 'tutorial'],
    metadata: {
        resolution: '1080p',
        duration: 300,
        customField: 'anything'
    }
});

// Query
const popularGamingVideos = await Video.find({
    tags: 'gaming',
    views: { $gt: 1000 }
})
.sort({ views: -1 })
.limit(10);

// Aggregation
const tagStats = await Video.aggregate([
    { $unwind: '$tags' },
    { $group: { 
        _id: '$tags',
        count: { $sum: 1 },
        totalViews: { $sum: '$views' }
    }},
    { $sort: { count: -1 } }
]);
```

### Key-Value (Redis)
```javascript
const Redis = require('ioredis');
const redis = new Redis();

// Simple key-value
await redis.set('user:123', JSON.stringify({ name: 'John' }));
const user = JSON.parse(await redis.get('user:123'));

// With expiration (caching)
await redis.setex('session:abc', 3600, 'user_123');

// Counters
await redis.incr('video:456:views');

// Leaderboard
await redis.zadd('leaderboard', 100, 'video:a');
await redis.zadd('leaderboard', 200, 'video:b');
const top10 = await redis.zrevrange('leaderboard', 0, 9, 'WITHSCORES');
```

---

## Comparison Table

```
┌─────────────────────┬────────────────────┬───────────────────────┐
│     Feature         │        SQL         │        NoSQL          │
├─────────────────────┼────────────────────┼───────────────────────┤
│ Data Model          │ Tables & Rows      │ Documents/Key-Value   │
│ Schema              │ Fixed              │ Flexible              │
│ Relationships       │ JOINs              │ Embedded/References   │
│ Scaling             │ Vertical           │ Horizontal            │
│ Consistency         │ Strong (ACID)      │ Eventual (BASE)       │
│ Query Language      │ SQL                │ Varies                │
│ Transactions        │ Full support       │ Limited               │
│ Best For            │ Complex relations  │ Flexible/Scale        │
│ Examples            │ PostgreSQL, MySQL  │ MongoDB, Redis        │
└─────────────────────┴────────────────────┴───────────────────────┘
```

---

## Hybrid Approach (Best Practice)

```
YouTube Architecture:

┌─────────────────────────────────────────────────────────────┐
│                      Application Layer                       │
└───────────────────────────┬─────────────────────────────────┘
                            │
  ┌─────────────────────────┼─────────────────────────────────┐
  │                         │                                 │
  ▼                         ▼                                 ▼
┌───────────────┐    ┌───────────────┐    ┌───────────────────┐
│  PostgreSQL   │    │    MongoDB    │    │      Redis        │
│               │    │               │    │                   │
│ Users         │    │ Video content │    │ Caching           │
│ Payments      │    │ Comments      │    │ Sessions          │
│ Subscriptions │    │ Playlists     │    │ Rate limiting     │
│ Billing       │    │ Preferences   │    │ View counts       │
└───────────────┘    └───────────────┘    └───────────────────┘

Use the RIGHT database for each use case!
```

---

## Interview Questions

**Q: When would you use SQL over NoSQL?**
A: Complex relationships, need JOINs, ACID transactions required, data is structured. Example: banking, e-commerce orders.

**Q: When would you use NoSQL?**
A: Flexible schema, massive scale, high write throughput, hierarchical data. Example: social feeds, IoT data.

**Q: Can you use both?**
A: Yes! Most modern systems use polyglot persistence - different databases for different needs.

**Q: What is eventual consistency?**
A: Data will be consistent eventually, but reads might return stale data temporarily. Acceptable for likes, views.

**Q: How does NoSQL scale better?**
A: Built for horizontal scaling (sharding). Data distributed across nodes. No complex JOINs across servers.

---

## Quick Summary

```
SQL:
- Structured, relational data
- ACID transactions
- Complex queries with JOINs
- Vertical scaling
- Strong consistency

NoSQL:
- Flexible, denormalized data  
- BASE (eventual consistency)
- Simple queries
- Horizontal scaling
- High performance

CHOOSE SQL:
- Banking, payments
- Order management
- Inventory
- Complex relationships

CHOOSE NoSQL:
- Social media
- Real-time analytics
- Caching
- IoT/Logs
- Flexible content

BEST PRACTICE:
Use both! Pick the right tool for each job.
```

You now understand SQL vs NoSQL! 📊
