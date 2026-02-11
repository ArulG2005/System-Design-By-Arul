# Caching Strategies - Complete Guide

## What are Caching Strategies?

Think of it like a **library checkout system**:
- **Read-Through**: Librarian gets book for you if not on display
- **Write-Through**: Librarian updates catalog immediately when book returns
- **Write-Behind**: Librarian stacks returns and updates catalog later
- **Cache-Aside**: You check display shelf first, ask librarian if not there

**Simple Definition:**
Caching strategies define HOW and WHEN data moves between cache and database.

---

## Why Different Strategies?

```
Different application needs:
                              
Read-Heavy App         Write-Heavy App        Real-time App
(News website)         (IoT sensors)          (Stock trading)
     │                      │                      │
     ▼                      ▼                      ▼
Read-Through           Write-Behind           Write-Through
+ Cache-Aside          (Batch writes)         (Immediate sync)
```

---

## Strategy 1: Cache-Aside (Lazy Loading)

### How It Works:
```
Application controls EVERYTHING

READ:
┌──────┐     ┌───────┐     ┌──────────┐
│ App  │────►│ Cache │     │ Database │
│      │  1. │ check │     │          │
└──────┘     └───┬───┘     └────┬─────┘
                 │              │
            2. Miss?           │
                 │              │
                 └──────────────▼
                          3. Get from DB
                               │
                               ▼
                 ┌─────────────────────┐
                 │ 4. Store in cache   │
                 │ 5. Return to app    │
                 └─────────────────────┘

WRITE:
App writes to DB directly
App invalidates or updates cache
```

### Code Implementation:
```javascript
class CacheAside {
    constructor(redis, db) {
        this.redis = redis;
        this.db = db;
    }
    
    async get(key) {
        // Step 1: Check cache
        const cached = await this.redis.get(key);
        if (cached) {
            console.log('Cache HIT');
            return JSON.parse(cached);
        }
        
        // Step 2: Cache miss - Get from DB
        console.log('Cache MISS');
        const data = await this.db.find(key);
        
        if (data) {
            // Step 3: Store in cache for next time
            await this.redis.setex(
                key, 
                3600,  // TTL: 1 hour
                JSON.stringify(data)
            );
        }
        
        return data;
    }
    
    async set(key, value) {
        // Write to database
        await this.db.update(key, value);
        
        // Invalidate or update cache
        await this.redis.del(key);  // Invalidate approach
        // OR
        // await this.redis.setex(key, 3600, JSON.stringify(value));  // Update approach
    }
    
    async delete(key) {
        await this.db.delete(key);
        await this.redis.del(key);
    }
}

// Usage
const videoCache = new CacheAside(redis, videoRepository);
const video = await videoCache.get('video:abc123');
```

### Pros & Cons:
```
✓ Simple to understand
✓ Only cache what's needed
✓ Cache failure doesn't break app
✓ Works with any DB

✗ Cache miss = slow first request
✗ Stale data if DB updated directly
✗ Application manages cache logic
```

---

## Strategy 2: Read-Through

### How It Works:
```
Cache handles database reads automatically

READ:
┌──────┐     ┌───────────────────────────────┐     ┌──────────┐
│ App  │────►│          Cache                │────►│ Database │
│      │     │ 1. Check cache               │     │          │
└──────┘     │ 2. If miss, load from DB     │     │          │
             │ 3. Store and return          │     │          │
             └───────────────────────────────┘     └──────────┘

App only talks to cache, NEVER directly to DB for reads
```

### Code Implementation:
```javascript
class ReadThroughCache {
    constructor(redis, db, options = {}) {
        this.redis = redis;
        this.db = db;
        this.ttl = options.ttl || 3600;
    }
    
    async get(key, loader) {
        // Check cache
        const cached = await this.redis.get(key);
        if (cached) {
            return JSON.parse(cached);
        }
        
        // Auto-load from database
        const data = await loader();
        
        if (data) {
            await this.redis.setex(key, this.ttl, JSON.stringify(data));
        }
        
        return data;
    }
}

// Usage - Define loader function
const cache = new ReadThroughCache(redis, db);

async function getVideo(videoId) {
    return cache.get(`video:${videoId}`, async () => {
        // This runs only on cache miss
        return await db.videos.findById(videoId);
    });
}
```

### Advanced: Using Cacheable Library Pattern
```javascript
// Some libraries like Spring Cache do this automatically

// Java-like pseudocode
@Cacheable("videos")
public Video getVideo(String videoId) {
    return videoRepository.findById(videoId);
}

// Automatically:
// 1. Check cache for key "videos::{videoId}"
// 2. If miss, call method
// 3. Store result in cache
// 4. Return result
```

### Pros & Cons:
```
✓ Simple application code
✓ Cache logic centralized
✓ Consistent cache behavior

✗ First request always slow
✗ More complex cache layer
✗ Potential for cache stampede
```

---

## Strategy 3: Write-Through

### How It Works:
```
Cache handles writes synchronously

WRITE:
┌──────┐     ┌───────────────────────────────┐     ┌──────────┐
│ App  │────►│          Cache                │────►│ Database │
│      │     │ 1. Write to cache            │     │          │
└──────┘     │ 2. Write to database         │     │          │
             │ 3. Confirm when BOTH done    │     │          │
             └───────────────────────────────┘     └──────────┘

Data is ALWAYS in sync (cache = database)
```

### Code Implementation:
```javascript
class WriteThroughCache {
    constructor(redis, db, options = {}) {
        this.redis = redis;
        this.db = db;
        this.ttl = options.ttl || 3600;
    }
    
    async get(key) {
        const cached = await this.redis.get(key);
        return cached ? JSON.parse(cached) : null;
    }
    
    async set(key, value) {
        // Write to both cache AND database
        // Both must succeed
        await Promise.all([
            this.redis.setex(key, this.ttl, JSON.stringify(value)),
            this.db.update(key, value)
        ]);
    }
    
    async setWithTransaction(key, value) {
        // More robust: Write DB first, then cache
        try {
            await this.db.update(key, value);
            await this.redis.setex(key, this.ttl, JSON.stringify(value));
        } catch (error) {
            // If cache update fails, invalidate to prevent stale data
            await this.redis.del(key);
            throw error;
        }
    }
}

// Usage
const cache = new WriteThroughCache(redis, db);

async function updateVideo(videoId, data) {
    await cache.set(`video:${videoId}`, data);
    // Both cache and DB are now updated!
}
```

### Pros & Cons:
```
✓ Data always consistent
✓ No stale reads
✓ Good for read-heavy after writes

✗ Write latency is HIGH (cache + DB)
✗ Both systems must be available
✗ Not ideal for write-heavy apps
```

---

## Strategy 4: Write-Behind (Write-Back)

### How It Works:
```
Cache handles writes asynchronously

WRITE:
┌──────┐     ┌───────┐                      ┌──────────┐
│ App  │────►│ Cache │                      │ Database │
│      │     │       │                      │          │
└──────┘     └───┬───┘                      └────┬─────┘
                 │                               │
       1. Write to cache (IMMEDIATE RETURN)     │
                 │                               │
                 └──────── Later ────────────────▼
                    2. Async write to DB

App returns BEFORE database write completes!
```

### Code Implementation:
```javascript
class WriteBehindCache {
    constructor(redis, db, options = {}) {
        this.redis = redis;
        this.db = db;
        this.ttl = options.ttl || 3600;
        this.writeQueue = [];
        this.flushInterval = options.flushInterval || 5000; // 5 seconds
        
        // Start background flush process
        this.startFlusher();
    }
    
    async get(key) {
        const cached = await this.redis.get(key);
        return cached ? JSON.parse(cached) : null;
    }
    
    async set(key, value) {
        // Write to cache immediately
        await this.redis.setex(key, this.ttl, JSON.stringify(value));
        
        // Queue database write
        this.writeQueue.push({ key, value, timestamp: Date.now() });
        
        // Return immediately - DB write happens later!
        return true;
    }
    
    startFlusher() {
        setInterval(async () => {
            if (this.writeQueue.length === 0) return;
            
            // Get batch of writes
            const batch = this.writeQueue.splice(0, 100);
            
            try {
                // Bulk write to database
                await this.db.bulkUpdate(batch);
                console.log(`Flushed ${batch.length} writes to DB`);
            } catch (error) {
                // On failure, add back to queue
                this.writeQueue.unshift(...batch);
                console.error('Flush failed, will retry', error);
            }
        }, this.flushInterval);
    }
}

// Usage
const cache = new WriteBehindCache(redis, db);

async function incrementViews(videoId) {
    await cache.set(`video:${videoId}:views`, views + 1);
    // Returns immediately! DB updates in background
}
```

### Production Version with Kafka:
```javascript
class WriteBehindWithKafka {
    constructor(redis, producer) {
        this.redis = redis;
        this.producer = producer;
    }
    
    async set(key, value) {
        // 1. Write to cache
        await this.redis.setex(key, 3600, JSON.stringify(value));
        
        // 2. Send to Kafka for async DB write
        await this.producer.send({
            topic: 'db-writes',
            messages: [{
                key,
                value: JSON.stringify({ key, value, timestamp: Date.now() })
            }]
        });
    }
}

// Separate consumer writes to DB
async function dbWriter() {
    await consumer.run({
        eachBatch: async ({ batch }) => {
            const writes = batch.messages.map(m => JSON.parse(m.value));
            await db.bulkUpdate(writes);
        }
    });
}
```

### Pros & Cons:
```
✓ Fastest writes (async)
✓ Reduces DB load (batching)
✓ Great for write-heavy apps

✗ Data can be lost if cache fails
✗ Complex failure handling
✗ Inconsistency window
```

---

## Strategy 5: Refresh-Ahead

### How It Works:
```
Proactively refresh cache before expiry

┌──────┐         ┌───────┐
│ App  │────────►│ Cache │
│      │         │       │
└──────┘         └───┬───┘
                     │
                     ▼
            Is data near expiry?
                (80% of TTL)
                     │
            ┌────────┴────────┐
            ▼                 ▼
          Yes                No
            │                 │
   Async refresh          Return
   in background          cached data
```

### Code Implementation:
```javascript
class RefreshAheadCache {
    constructor(redis, db, options = {}) {
        this.redis = redis;
        this.db = db;
        this.ttl = options.ttl || 3600;
        this.refreshThreshold = options.refreshThreshold || 0.8; // 80%
        this.refreshing = new Set(); // Track ongoing refreshes
    }
    
    async get(key, loader) {
        const cached = await this.redis.get(key);
        
        if (cached) {
            const data = JSON.parse(cached);
            
            // Check if should refresh
            const ttl = await this.redis.ttl(key);
            const threshold = this.ttl * (1 - this.refreshThreshold);
            
            if (ttl < threshold && !this.refreshing.has(key)) {
                // Trigger async refresh (don't await!)
                this.refreshAsync(key, loader);
            }
            
            return data;
        }
        
        // Cache miss - load synchronously
        return await this.loadAndCache(key, loader);
    }
    
    async refreshAsync(key, loader) {
        this.refreshing.add(key);
        
        try {
            await this.loadAndCache(key, loader);
            console.log(`Refreshed ${key} proactively`);
        } catch (error) {
            console.error(`Failed to refresh ${key}`, error);
        } finally {
            this.refreshing.delete(key);
        }
    }
    
    async loadAndCache(key, loader) {
        const data = await loader();
        if (data) {
            await this.redis.setex(key, this.ttl, JSON.stringify(data));
        }
        return data;
    }
}

// Usage
const cache = new RefreshAheadCache(redis, db, {
    ttl: 3600,
    refreshThreshold: 0.8  // Refresh when 80% expired
});

const video = await cache.get('video:abc', () => db.findVideo('abc'));
```

### Pros & Cons:
```
✓ Never experience cache miss (for hot data)
✓ Always fast responses
✓ Reduces cache stampede

✗ Complex implementation
✗ Wasteful for cold data
✗ More cache/DB operations
```

---

## Strategy Comparison

```
┌─────────────────┬──────────┬──────────┬──────────┬───────────┐
│    Strategy     │  Reads   │  Writes  │ Latency  │   Best For│
├─────────────────┼──────────┼──────────┼──────────┼───────────┤
│ Cache-Aside     │  Medium  │   Low    │  Medium  │  General  │
│ Read-Through    │   Fast   │    -     │   Low    │  Read-heavy│
│ Write-Through   │   Fast   │   Slow   │   High   │  Consistency│
│ Write-Behind    │   Fast   │   Fast   │   Low    │  Write-heavy│
│ Refresh-Ahead   │   Fast   │    -     │   Low    │  Hot data  │
└─────────────────┴──────────┴──────────┴──────────┴───────────┘
```

---

## YouTube Caching Strategy

### Mixed Strategy Approach:
```
┌─────────────────────────────────────────────────────────────┐
│                     YouTube Platform                         │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ Video Metadata│   │  View Counts  │   │    Search     │
│               │   │               │   │    Results    │
│ Read-Through  │   │ Write-Behind  │   │ Refresh-Ahead │
│ + Cache-Aside │   │ (async batch) │   │ (pre-compute) │
└───────────────┘   └───────────────┘   └───────────────┘

Video Metadata:
- High read ratio
- Rarely changes
- Use Read-Through for auto-loading
- Cache-Aside for explicit invalidation

View Counts:
- Extremely write-heavy
- Slight delay OK
- Write-Behind batches updates

Search Results:
- Popular searches pre-cached
- Refresh before expiry
- Refresh-Ahead for hot queries
```

### Implementation:
```javascript
class YouTubeCacheService {
    constructor(redis, db) {
        this.videoCache = new ReadThroughCache(redis, db);
        this.viewCounter = new WriteBehindCache(redis, db);
        this.searchCache = new RefreshAheadCache(redis, db);
    }
    
    // Video metadata - Read-Through
    async getVideo(videoId) {
        return this.videoCache.get(
            `video:${videoId}`,
            () => this.db.videos.findById(videoId)
        );
    }
    
    // View counts - Write-Behind
    async incrementViews(videoId) {
        const key = `views:${videoId}`;
        await this.redis.incr(key);  // Fast increment
        this.viewCounter.queueSync(videoId);  // Async DB sync
    }
    
    // Search - Refresh-Ahead
    async search(query) {
        return this.searchCache.get(
            `search:${query}`,
            () => this.searchEngine.search(query),
            { refreshThreshold: 0.7 }  // Refresh at 70% TTL
        );
    }
    
    // Explicit invalidation on updates
    async updateVideo(videoId, data) {
        await this.db.videos.update(videoId, data);
        await this.redis.del(`video:${videoId}`);
    }
}
```

---

## Interview Questions

**Q: What is Cache-Aside?**
A: Application checks cache, on miss loads from DB and populates cache. Simple but application manages logic.

**Q: Write-Through vs Write-Behind?**
A: Write-Through: sync write to both cache and DB (slow but consistent). Write-Behind: async DB write (fast but can lose data).

**Q: When to use Refresh-Ahead?**
A: For hot data that's frequently accessed. Prevents cache miss by refreshing before expiry.

**Q: How to handle cache-DB inconsistency?**
A: TTL limits staleness, CDC for real-time sync, or write-through for immediate consistency.

**Q: Best strategy for YouTube?**
A: Mix: Read-Through for video metadata, Write-Behind for view counts, Refresh-Ahead for trending content.

---

## Quick Summary

```
CACHING STRATEGIES:
───────────────────

CACHE-ASIDE (Lazy Loading):
- App checks cache → miss → load from DB → cache it
- Simple, flexible, but cache miss is slow

READ-THROUGH:
- Cache auto-loads from DB on miss
- Simpler app code, cache handles loading

WRITE-THROUGH:
- Write to cache AND DB synchronously
- Always consistent, but slow writes

WRITE-BEHIND (Write-Back):
- Write to cache, async DB write later
- Fast writes, but risk of data loss

REFRESH-AHEAD:
- Proactively refresh before expiry
- Always fast, good for hot data

CHOOSE BASED ON:
────────────────
- Read vs Write ratio
- Consistency requirements
- Latency tolerance
- Data importance
```

You now understand caching strategies! 🎯
