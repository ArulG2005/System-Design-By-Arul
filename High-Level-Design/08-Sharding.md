# Sharding - Complete Guide

## What is Sharding?

Think of Sharding like a **pizza party**:
- One pizza (database) feeds few people
- 100 guests? Get multiple pizzas!
- Each guest knows which pizza to go to

**Simple Definition:**
Sharding = Splitting your database into smaller pieces (shards), each stored on different servers.

---

## Why Do We Need Sharding?

### The Problem:
```
YouTube has 1 billion videos

One Server:
┌─────────────────────────┐
│    1 Billion Videos     │
│    - Slow queries       │
│    - Running out of disk│
│    - Can't handle load  │
└─────────────────────────┘

Server says: "I can't handle this alone!" 💀
```

### The Solution - Sharding:
```
Split into multiple servers:

┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│   Shard 1   │ │   Shard 2   │ │   Shard 3   │ │   Shard 4   │
│   250M      │ │   250M      │ │   250M      │ │   250M      │
│   videos    │ │   videos    │ │   videos    │ │   videos    │
└─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘

Each server handles 1/4 of the load!
```

---

## Sharding vs Replication

```
REPLICATION = Copies of SAME data
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Server 1   │ │  Server 2   │ │  Server 3   │
│  All Data   │ │  All Data   │ │  All Data   │
│  (Copy)     │ │  (Copy)     │  (Copy)     │
└─────────────┘ └─────────────┘ └─────────────┘
Purpose: High availability, read scaling

SHARDING = DIFFERENT data on each server
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│  Server 1   │ │  Server 2   │ │  Server 3   │
│  Users A-H  │ │  Users I-P  │ │  Users Q-Z  │
└─────────────┘ └─────────────┘ └─────────────┘
Purpose: Handle massive data, write scaling
```

---

## Sharding Strategies

### 1. Range-Based Sharding
```
Shard by value range:

User IDs 1-1000000     → Shard 1
User IDs 1000001-2000000 → Shard 2
User IDs 2000001-3000000 → Shard 3

OR by alphabet:
Users A-H → Shard 1
Users I-P → Shard 2
Users Q-Z → Shard 3

Query: "Find user #1500000"
System: "That's in Shard 2!"
```

**Pros:**
- Simple to understand
- Range queries are efficient
- Easy to add new shards

**Cons:**
- Can become uneven (everyone named "John" on one shard!)
- Hot spots (newest data on one shard)

### 2. Hash-Based Sharding
```
Use hash function to distribute evenly:

hash(user_id) % num_shards = shard_number

User ID: 12345
hash(12345) = 98765
98765 % 4 = 1 → Shard 1

User ID: 12346
hash(12346) = 45678
45678 % 4 = 2 → Shard 2

Even distribution!
```

**Pros:**
- Even data distribution
- No hot spots
- Works with any key type

**Cons:**
- Range queries are hard (must query all shards)
- Adding shards = reshuffling data

### 3. Directory-Based Sharding
```
Lookup table tells where each key is:

┌──────────────────────────┐
│     DIRECTORY SERVICE    │
├──────────────────────────┤
│ User "John" → Shard 2    │
│ User "Jane" → Shard 1    │
│ User "Bob"  → Shard 3    │
└──────────────────────────┘

Query: "Where is John?"
Directory: "Shard 2"
```

**Pros:**
- Flexible placement
- Easy to rebalance
- Can handle special cases

**Cons:**
- Directory becomes bottleneck
- Extra lookup for every query
- Directory must be highly available

### 4. Geo-Based Sharding
```
Shard by location:

US Users    → US Shard (Virginia)
EU Users    → EU Shard (Frankfurt)
Asia Users  → Asia Shard (Singapore)

Lower latency for users!
```

---

## Shard Key Selection (VERY IMPORTANT!)

### What is a Shard Key?
The field used to determine which shard stores the data.

### Good Shard Key Properties:
```
1. HIGH CARDINALITY
   Bad:  gender (only Male/Female)
   Good: user_id (millions of unique values)

2. EVEN DISTRIBUTION
   Bad:  country (90% in one country)
   Good: hash(user_id) (evenly spread)

3. QUERY PATTERNS
   If you always query by user_id, shard by user_id
   Queries hit single shard = FAST

4. AVOID MONOTONIC
   Bad:  timestamp (all writes to newest shard)
   Good: user_id (writes spread out)
```

### Bad Shard Key Examples:
```
Shard Key: created_at

January data → Shard 1 (old, never accessed)
February data → Shard 2 (old, never accessed)
December data → Shard 12 (HOT! All writes here)

Result: One shard overloaded! 🔥
```

### Good Shard Key Example:
```
Shard Key: hash(user_id)

All users evenly distributed
Each user's data on one shard
Queries by user_id = Fast (single shard)
```

---

## Sharding Implementation

### MongoDB Sharding:
```javascript
// Enable sharding on database
sh.enableSharding("youtube");

// Shard the videos collection by channel_id
sh.shardCollection("youtube.videos", { channel_id: "hashed" });

// Or range-based
sh.shardCollection("youtube.videos", { channel_id: 1 });

// MongoDB automatically distributes data across shards!
```

### MongoDB Architecture:
```
┌─────────────────────────────────────────────────────────────┐
│                        APPLICATION                          │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       MONGOS (Router)                        │
│              Routes queries to correct shard                 │
└─────────────────────────────┬───────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│    Shard 1      │  │    Shard 2      │  │    Shard 3      │
│  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │
│  │  Primary  │  │  │  │  Primary  │  │  │  │  Primary  │  │
│  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │
│  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │
│  │ Secondary │  │  │  │ Secondary │  │  │  │ Secondary │  │
│  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │
└─────────────────┘  └─────────────────┘  └─────────────────┘

                    ┌─────────────────┐
                    │ Config Servers  │
                    │ (Stores shard   │
                    │  metadata)      │
                    └─────────────────┘
```

### PostgreSQL Sharding (Citus):
```sql
-- Create distributed table
SELECT create_distributed_table('videos', 'channel_id');

-- Insert goes to correct shard automatically
INSERT INTO videos (id, channel_id, title) 
VALUES (1, 'channel_123', 'My Video');

-- Query by shard key = Fast (single shard)
SELECT * FROM videos WHERE channel_id = 'channel_123';

-- Query without shard key = Slower (all shards)
SELECT * FROM videos WHERE title = 'My Video';
```

### Application-Level Sharding:
```javascript
class ShardRouter {
    constructor(shards) {
        this.shards = shards; // Array of database connections
        this.numShards = shards.length;
    }
    
    // Get shard for a given key
    getShard(key) {
        const hash = this.hashFunction(key);
        const shardIndex = hash % this.numShards;
        return this.shards[shardIndex];
    }
    
    hashFunction(key) {
        let hash = 0;
        for (let char of String(key)) {
            hash = ((hash << 5) - hash) + char.charCodeAt(0);
        }
        return Math.abs(hash);
    }
    
    // Insert data
    async insert(key, data) {
        const shard = this.getShard(key);
        return await shard.insert(data);
    }
    
    // Find by shard key
    async findByKey(key) {
        const shard = this.getShard(key);
        return await shard.find({ key });
    }
    
    // Find without shard key (must query all shards!)
    async findAll(query) {
        const results = await Promise.all(
            this.shards.map(shard => shard.find(query))
        );
        return results.flat();
    }
}

// Usage
const router = new ShardRouter([db1, db2, db3, db4]);

// Insert - goes to one shard
await router.insert('user_123', { name: 'John' });

// Find by shard key - queries one shard (fast!)
await router.findByKey('user_123');

// Find by other field - queries ALL shards (slow!)
await router.findAll({ name: 'John' });
```

---

## Cross-Shard Operations (The Challenge!)

### The Problem:
```
User "John" is on Shard 1
Video "abc" is on Shard 3

Query: "Get all videos liked by John"

This requires:
1. Go to Shard 1: Get John's liked video IDs
2. Go to Shard 3 (and others): Get video details
3. Combine results

Expensive! Slow! Avoid if possible.
```

### Solutions:

#### 1. Denormalization
```
Store video info WITH the like:

Likes table (sharded by user_id):
{
    user_id: "john",
    video_id: "abc",
    video_title: "Funny Cat",     // Denormalized!
    video_thumbnail: "url..."      // Denormalized!
}

Now: Get John's likes = Single shard query!
Trade-off: More storage, need to update when video changes
```

#### 2. Same Shard Key
```
Design so related data is on same shard:

Videos AND Comments both sharded by video_id

Video "abc" → Shard 2
Comments for "abc" → Shard 2

Query "video + comments" = Single shard!
```

#### 3. Broadcast Queries (Last Resort)
```
Query ALL shards and combine results

┌─────────┐
│  Query  │
└────┬────┘
     │
     ├──→ Shard 1 → Results
     ├──→ Shard 2 → Results
     ├──→ Shard 3 → Results
     ├──→ Shard 4 → Results
     │
     ▼
┌─────────────────┐
│ Combine Results │
└─────────────────┘

Use only when necessary!
```

---

## Rebalancing Shards

### When to Rebalance:
- Some shards getting too big
- Adding new shards
- Removing shards

### Approaches:

#### 1. Fixed Partitioning
```
Create MORE partitions than shards initially:

100 partitions, 4 shards
Each shard gets 25 partitions

Adding 5th shard?
Move some partitions (not data!) to new shard
No data movement within partitions!
```

#### 2. Dynamic Partitioning
```
Split partitions when they get too big:

Shard 1 (too big!)
    ↓
Split into:
Shard 1A + Shard 1B
```

#### 3. Consistent Hashing
```
Only move K/N keys when adding N shards

See separate file on Consistent Hashing!
```

---

## Sharding YouTube Clone

### Video Service Sharding:
```
Shard Key: video_id (hashed)

Why video_id?
- Most queries are by video_id
- Videos are independent
- Evenly distributes writes

┌────────────────────────────────────────────────┐
│                                                │
│  GET /videos/abc123                            │
│       ↓                                        │
│  hash("abc123") % 4 = 2                        │
│       ↓                                        │
│  Route to Shard 2                              │
│       ↓                                        │
│  Get video from Shard 2                        │
│                                                │
└────────────────────────────────────────────────┘
```

### User Service Sharding:
```
Shard Key: user_id (hashed)

Why user_id?
- Queries by user_id are common
- User data is independent
- Login = query by user_id
```

### Comments Service Sharding:
```
Shard Key: video_id (not user_id!)

Why video_id instead of user_id?
- Comments are viewed BY VIDEO
- "Get comments for video X" = One shard!
- "Get all comments by user" = Rare, OK if slow
```

### Channel Subscriptions Sharding:
```
Shard Key: channel_id

"Get subscribers of channel X" = One shard (common)
"Get subscriptions of user Y" = All shards (acceptable)

OR: Dual sharding
- subscriptions_by_channel (sharded by channel_id)
- subscriptions_by_user (sharded by user_id)
Trade-off: More storage, keep in sync
```

---

## Sharding Best Practices

### 1. Start with Right Shard Key
```
Can't easily change later!
Analyze query patterns first
Test with realistic data
```

### 2. Plan for Growth
```
Start with more shards than needed
Easier to merge than split
Consider future query patterns
```

### 3. Monitor Shard Balance
```
Check regularly:
- Data size per shard
- Query load per shard
- Hotspots
```

### 4. Test Cross-Shard Queries
```
Know which queries span shards
Optimize or redesign if too slow
Consider caching for common cross-shard queries
```

---

## Interview Questions

**Q: What is sharding?**
A: Splitting a database horizontally across multiple servers. Each shard holds a portion of the data, determined by a shard key.

**Q: How do you choose a shard key?**
A: Choose a key with high cardinality, even distribution, matches query patterns, and avoids monotonic values.

**Q: What's the difference between sharding and partitioning?**
A: Partitioning splits data within one server. Sharding splits data across multiple servers.

**Q: How to handle cross-shard queries?**
A: Denormalization, same shard keys for related data, or broadcast queries as last resort.

**Q: How to add a new shard?**
A: Use consistent hashing to minimize data movement, or use fixed partitioning strategy.

---

## Quick Summary

```
SHARDING = Horizontal database splitting

Strategies:
- Range: By value range (A-H, I-P, Q-Z)
- Hash: By hash value (even distribution)
- Directory: Lookup table
- Geo: By location

Shard Key Rules:
- High cardinality
- Even distribution
- Match query patterns
- Avoid monotonic

Challenges:
- Cross-shard queries
- Rebalancing
- Transactions across shards
```

You now understand Sharding like a pro! 🚀
