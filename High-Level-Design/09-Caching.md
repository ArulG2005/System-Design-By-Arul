# Caching - Complete Guide

## What is Caching?

Think of Caching like a **desk drawer vs filing cabinet**:
- Filing cabinet (database): Has everything, but slow to access
- Desk drawer (cache): Has frequently used items, super fast!

**Simple Definition:**
Caching = Storing copies of frequently accessed data in a faster storage layer, so we don't have to fetch from the slow source every time.

---

## Why Do We Need Caching?

### Without Caching:
```
User requests YouTube homepage:
1. Query database for trending videos   (50ms)
2. Query database for recommended       (100ms)
3. Query database for subscriptions     (30ms)
4. Query database for user info         (20ms)
                                        ________
                                        200ms per request!

1000 users per second = 1000 × 200ms = OVERLOAD! 💥
```

### With Caching:
```
User requests YouTube homepage:
1. Get trending from cache             (1ms)
2. Get recommended from cache          (1ms)
3. Get subscriptions from cache        (1ms)
4. Get user info from cache            (1ms)
                                       ______
                                       4ms per request!

1000 users per second = Easy! ✓
Database load reduced by 99%!
```

---

## What to Cache?

### Cache These (Good Candidates):
```
✓ Data that doesn't change often
  - User profiles
  - Video metadata
  - Static configuration

✓ Expensive to compute
  - Recommendation lists
  - Search results
  - Aggregated statistics

✓ Frequently accessed
  - Trending videos
  - Popular content
  - Session data

✓ Read-heavy workloads
  - Product catalogs
  - News articles
  - Social media feeds
```

### Don't Cache These:
```
✗ Rapidly changing data
  - Real-time stock prices
  - Live game scores

✗ Write-heavy data
  - Logging
  - Analytics events

✗ Sensitive data (be careful!)
  - Passwords
  - Credit card numbers
  - Personal health records

✗ Rarely accessed data
  - Archive data
  - Old records
```

---

## Cache Hierarchy

```
Speed & Cost:

        FASTEST & MOST EXPENSIVE
              ▲
              │    ┌─────────────────┐
              │    │   CPU Cache     │  Nanoseconds
              │    │   (L1, L2, L3)  │
              │    └─────────────────┘
              │    ┌─────────────────┐
              │    │  Application    │  Microseconds
              │    │  Memory Cache   │  (In-process)
              │    └─────────────────┘
              │    ┌─────────────────┐
              │    │  Distributed    │  1-5 milliseconds
              │    │  Cache (Redis)  │
              │    └─────────────────┘
              │    ┌─────────────────┐
              │    │    Database     │  10-100 milliseconds
              │    │                 │
              │    └─────────────────┘
              │    ┌─────────────────┐
              │    │   Disk/SSD      │  Milliseconds
              │    │   Storage       │
              │    └─────────────────┘
              ▼
        SLOWEST & CHEAPEST
```

---

## Types of Caching

### 1. Browser Cache (Client-Side)
```
User's browser stores assets locally

First visit:
Browser → Server: "Give me style.css"
Server → Browser: "Here's style.css" (Cache-Control: max-age=86400)

Next visit (within 24 hours):
Browser: "I already have style.css, use local copy!"
No network request needed!

HTTP Headers:
Cache-Control: max-age=86400, public
ETag: "abc123"
```

### 2. CDN Cache (Edge)
```
Content cached at edge servers near users

                    ┌─────────── CDN Edge (New York)
User in NY ─────────┤           Cached content!
                    │           <1ms latency
                    │
                    │      ┌─── CDN Edge (London)
User in London ─────┼──────┤    Cached content!
                    │      │    <1ms latency
                    │      │
                    │      │    ┌── Origin Server (Singapore)
                    └──────┴────┤   Source of truth
                                │   200ms latency to NY
```

### 3. Application Cache (Server-Side)
```
In-memory cache within application

┌─────────────────────────────────────────┐
│           APPLICATION SERVER            │
│  ┌───────────────────────────────────┐  │
│  │        In-Memory Cache            │  │
│  │   { "user_123": { name: "John" } }│  │
│  │   { "video_abc": { views: 1000 } }│  │
│  └───────────────────────────────────┘  │
│                   ↓                     │
│              If not in cache            │
│                   ↓                     │
│           Query Database                │
└─────────────────────────────────────────┘
```

### 4. Distributed Cache (Redis/Memcached)
```
Separate cache servers, shared by all app servers

┌─────────────┐     ┌─────────────┐
│  App Server │     │  App Server │
│      1      │     │      2      │
└──────┬──────┘     └──────┬──────┘
       │                   │
       └─────────┬─────────┘
                 │
                 ▼
       ┌─────────────────┐
       │   Redis Cache   │
       │  (Shared State) │
       └─────────────────┘
                 │
                 ▼
       ┌─────────────────┐
       │    Database     │
       └─────────────────┘
```

### 5. Database Cache (Query Cache)
```
Database caches query results internally

Query: SELECT * FROM videos WHERE id = 'abc'

First time:
- Parse query
- Execute on disk
- Return result
- Store in query cache

Second time:
- Check query cache
- HIT! Return cached result
- No disk access!
```

---

## Caching Patterns

### 1. Cache-Aside (Lazy Loading)
```
Application manages cache manually

Read:
1. Check cache
2. If HIT → return cached data
3. If MISS → query database
4. Store in cache
5. Return data

┌─────────┐    1. Get    ┌─────────┐
│   App   │ ───────────> │  Cache  │
│         │ <─────────── │         │
│         │    2. HIT?   │         │
│         │              └─────────┘
│         │    3. MISS? Query DB
│         │ ───────────────────────> ┌─────────┐
│         │ <─────────────────────── │   DB    │
│         │    4. Store in cache     └─────────┘
│         │ ───────────> ┌─────────┐
│         │              │  Cache  │
└─────────┘              └─────────┘
```

```javascript
async function getUser(userId) {
    // 1. Check cache
    let user = await cache.get(`user:${userId}`);
    
    if (user) {
        // 2. Cache HIT
        return JSON.parse(user);
    }
    
    // 3. Cache MISS - query database
    user = await db.users.findById(userId);
    
    if (user) {
        // 4. Store in cache for 1 hour
        await cache.set(`user:${userId}`, JSON.stringify(user), 'EX', 3600);
    }
    
    return user;
}
```

**Pros:** Only caches what's needed, simple
**Cons:** First request always slow (cache miss)

### 2. Write-Through
```
Write to cache AND database together

Write:
1. App writes to cache
2. Cache writes to database (synchronous)
3. Both updated before response

┌─────────┐    1. Write   ┌─────────┐    2. Write   ┌─────────┐
│   App   │ ───────────── │  Cache  │ ───────────── │   DB    │
│         │               │         │               │         │
└─────────┘               └─────────┘               └─────────┘
                                3. Both done, respond to app
```

```javascript
async function updateUser(userId, data) {
    // Write to cache first
    await cache.set(`user:${userId}`, JSON.stringify(data));
    
    // Then write to database
    await db.users.update(userId, data);
    
    // Both complete before returning
    return { success: true };
}
```

**Pros:** Cache always has latest data
**Cons:** Slower writes (must update both)

### 3. Write-Behind (Write-Back)
```
Write to cache, update database later (async)

Write:
1. App writes to cache
2. Return immediately (fast!)
3. Background job syncs to database

┌─────────┐    1. Write   ┌─────────┐
│   App   │ ───────────── │  Cache  │
│         │ <───────────  │         │
└─────────┘    2. Done!   │         │
                          │    ↓    │
                          │ 3. Async│
                          │    ↓    │
                          └────┬────┘
                               │
                               ▼
                          ┌─────────┐
                          │   DB    │
                          └─────────┘
```

**Pros:** Very fast writes
**Cons:** Risk of data loss if cache crashes before sync

### 4. Read-Through
```
Cache handles database reads automatically

┌─────────┐    1. Get    ┌─────────────────────────┐
│   App   │ ───────────> │  Cache (with DB access) │
│         │ <─────────── │                         │
└─────────┘              │  Checks itself          │
                         │  If MISS → queries DB   │
                         │  Stores result          │
                         │  Returns to app         │
                         └───────────┬─────────────┘
                                     │
                                     ▼
                               ┌──────────┐
                               │    DB    │
                               └──────────┘
```

**Pros:** Simple app code, cache handles everything
**Cons:** Need smart cache system

### 5. Refresh-Ahead
```
Proactively refresh cache before expiry

Cache entry expires in 60 seconds
At 50 seconds (80% of TTL):
- Background: Refresh from database
- User: Still gets cached (fast!)
- When it would expire, already refreshed!

Never see cache miss for hot data!
```

---

## Redis - The Most Popular Cache

### Basic Operations:
```javascript
const Redis = require('ioredis');
const redis = new Redis();

// String operations
await redis.set('key', 'value');
await redis.set('key', 'value', 'EX', 3600); // Expires in 1 hour
const value = await redis.get('key');

// JSON (need to serialize)
await redis.set('user:123', JSON.stringify({ name: 'John' }));
const user = JSON.parse(await redis.get('user:123'));

// Hash (better for objects)
await redis.hset('user:123', 'name', 'John', 'age', 30);
const name = await redis.hget('user:123', 'name');
const all = await redis.hgetall('user:123');

// List (for queues, recent items)
await redis.lpush('recent:videos', 'video1', 'video2');
const recent = await redis.lrange('recent:videos', 0, 9); // Get 10

// Set (unique items)
await redis.sadd('liked:user:123', 'video1', 'video2');
const isLiked = await redis.sismember('liked:user:123', 'video1');

// Sorted Set (leaderboards, rankings)
await redis.zadd('trending', 1000, 'video1', 2000, 'video2');
const top10 = await redis.zrevrange('trending', 0, 9, 'WITHSCORES');
```

### Cache Pattern with Redis:
```javascript
class CacheService {
    constructor(redis, db) {
        this.redis = redis;
        this.db = db;
    }
    
    async get(key, fetchFn, ttl = 3600) {
        // Try cache first
        const cached = await this.redis.get(key);
        
        if (cached) {
            return JSON.parse(cached);
        }
        
        // Cache miss - fetch from source
        const data = await fetchFn();
        
        if (data) {
            await this.redis.set(key, JSON.stringify(data), 'EX', ttl);
        }
        
        return data;
    }
    
    async invalidate(key) {
        await this.redis.del(key);
    }
    
    async invalidatePattern(pattern) {
        const keys = await this.redis.keys(pattern);
        if (keys.length > 0) {
            await this.redis.del(...keys);
        }
    }
}

// Usage
const cache = new CacheService(redis, db);

// Get user (cache-aside pattern)
const user = await cache.get(
    `user:${userId}`,
    () => db.users.findById(userId),
    3600 // 1 hour TTL
);

// Invalidate when user updates
await cache.invalidate(`user:${userId}`);
```

---

## Cache Invalidation

**"There are only two hard things in Computer Science: cache invalidation and naming things."**

### Strategies:

#### 1. Time-Based (TTL)
```
Set expiration time, cache auto-deletes

redis.set('trending', data, 'EX', 300);  // 5 minutes

Pros: Simple, automatic
Cons: Might show stale data until expires
```

#### 2. Event-Based
```
Invalidate when data changes

// When user updates profile
await db.users.update(userId, newData);
await cache.del(`user:${userId}`);

Pros: Always fresh
Cons: Must track all places to invalidate
```

#### 3. Version-Based
```
Include version in cache key

// v1
cache.set('user:123:v1', userData);

// After update, use v2
cache.set('user:123:v2', newUserData);

Old key expires naturally
```

#### 4. Tag-Based
```
Group related caches with tags

video:123 → tags: ['channel:456', 'category:gaming']
video:124 → tags: ['channel:456', 'category:music']

When channel:456 updates:
Invalidate all caches tagged with 'channel:456'
```

---

## Caching for YouTube Clone

### What to Cache:

```javascript
// 1. Video Metadata (High read, low change)
const video = await cache.get(
    `video:${videoId}`,
    () => db.videos.findById(videoId),
    3600 * 24  // 24 hours
);

// 2. Trending Videos (Computed, changes periodically)
const trending = await cache.get(
    'trending:videos',
    () => computeTrendingVideos(),
    300  // 5 minutes
);

// 3. User Profile (Medium change frequency)
const profile = await cache.get(
    `profile:${userId}`,
    () => db.users.getProfile(userId),
    3600  // 1 hour
);

// 4. Channel Subscriber Count (Changes often, but OK if slightly stale)
const subCount = await cache.get(
    `channel:${channelId}:subs`,
    () => db.subscriptions.count(channelId),
    60  // 1 minute
);

// 5. Video View Count (Use Redis INCR for real-time)
await redis.incr(`views:${videoId}`);
// Periodically sync to database
```

### Architecture:
```
┌─────────────────────────────────────────────────────────────────┐
│                         USER REQUEST                             │
│                    GET /watch?v=abc123                           │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                         CDN LAYER                                │
│              Video files, thumbnails, static assets              │
│                    HIT? → Return immediately                     │
└───────────────────────────┬─────────────────────────────────────┘
                            │ MISS
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      APPLICATION CACHE                           │
│                     (Redis Cluster)                              │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Video     │  │   User      │  │  Trending   │              │
│  │  Metadata   │  │  Sessions   │  │   Lists     │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│                    HIT? → Return                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │ MISS
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                        DATABASE                                  │
│                   (Primary + Replicas)                          │
│         Fetch data → Store in cache → Return to user            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Cache Stampede Prevention

### The Problem:
```
Popular key expires
1000 requests come in simultaneously
All see cache MISS
All query database at once
Database overwhelmed! 💥
```

### Solutions:

#### 1. Locking
```javascript
async function getWithLock(key, fetchFn) {
    let data = await redis.get(key);
    
    if (data) return JSON.parse(data);
    
    // Try to acquire lock
    const lockKey = `lock:${key}`;
    const acquired = await redis.set(lockKey, '1', 'NX', 'EX', 10);
    
    if (acquired) {
        // I got the lock, fetch and cache
        data = await fetchFn();
        await redis.set(key, JSON.stringify(data), 'EX', 3600);
        await redis.del(lockKey);
        return data;
    } else {
        // Someone else is fetching, wait and retry
        await sleep(100);
        return getWithLock(key, fetchFn);
    }
}
```

#### 2. Early Expiration (Probabilistic)
```javascript
async function getWithEarlyRefresh(key, fetchFn, ttl) {
    const result = await redis.get(key);
    
    if (result) {
        const { data, expiry } = JSON.parse(result);
        const remaining = expiry - Date.now();
        
        // Probabilistically refresh if approaching expiry
        if (remaining < ttl * 0.1 && Math.random() < 0.1) {
            // 10% chance to refresh in last 10% of TTL
            refreshInBackground(key, fetchFn, ttl);
        }
        
        return data;
    }
    
    return fetchAndCache(key, fetchFn, ttl);
}
```

---

## Interview Questions

**Q: What is caching?**
A: Storing frequently accessed data in faster storage to reduce load on the primary data source and improve response times.

**Q: What's the difference between cache-aside and write-through?**
A: Cache-aside: App manages cache manually (read populates, write invalidates). Write-through: Writes go through cache to database together.

**Q: How do you handle cache invalidation?**
A: Use TTL for automatic expiry, event-based invalidation when data changes, or version/tag-based strategies for complex scenarios.

**Q: What is cache stampede and how to prevent it?**
A: When popular cache expires and many requests hit database simultaneously. Prevent with locking, early refresh, or probabilistic expiration.

**Q: When should you NOT use caching?**
A: Rapidly changing data, write-heavy workloads, rarely accessed data, or when consistency is critical.

---

## Quick Summary

```
CACHING = Fast storage for frequently accessed data

Types:
- Browser cache (client)
- CDN cache (edge)
- Application cache (in-memory)
- Distributed cache (Redis)
- Database cache (query cache)

Patterns:
- Cache-Aside: App checks cache, then DB
- Write-Through: Write both cache + DB
- Write-Behind: Write cache, async to DB
- Read-Through: Cache auto-fetches from DB

Invalidation:
- TTL (time-based)
- Event-based (on data change)
- Version-based (new key per version)

Popular Tools:
- Redis
- Memcached
- CDN (CloudFlare, CloudFront)
```

You now understand Caching like a pro! 🚀
