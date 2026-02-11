# Cache Eviction Policies - Complete Guide

## What is Cache Eviction?

Think of it like a **bookshelf with limited space**:
- You can only keep 10 books
- When you get an 11th book, which one do you remove?
- Different strategies for choosing what to remove

**Simple Definition:**
Cache eviction policies determine WHICH data to remove when the cache is full.

---

## Why Eviction is Needed

```
Cache has LIMITED memory:

┌────────────────────────────────────────┐
│             CACHE (1GB limit)           │
├────────────────────────────────────────┤
│ video:1  │ video:2  │ video:3  │ ...   │ 
│   10MB   │   15MB   │   8MB    │       │
└────────────────────────────────────────┘

When full:
┌────────────────────────────────────────┐
│             CACHE (1GB FULL!)           │
├────────────────────────────────────────┤
│ Filled with 1GB of data               │
│ New data arrives!                       │
│                                         │
│ Must remove something to make room!    │
└────────────────────────────────────────┘
```

---

## Common Eviction Policies

### 1. LRU (Least Recently Used)

**Remove what hasn't been used for the longest time**

```
Access order: A → B → C → D → A → B → E (Cache size: 4)

Step 1: Add A     [A]
Step 2: Add B     [A, B]
Step 3: Add C     [A, B, C]
Step 4: Add D     [A, B, C, D]  ← Full!
Step 5: Access A  [B, C, D, A]  ← A moves to end (recently used)
Step 6: Access B  [C, D, A, B]  ← B moves to end
Step 7: Add E     [D, A, B, E]  ← C removed (least recently used)

         Least Recent ◄────────────► Most Recent
```

#### Implementation:
```javascript
class LRUCache {
    constructor(capacity) {
        this.capacity = capacity;
        this.cache = new Map();  // Map maintains insertion order
    }
    
    get(key) {
        if (!this.cache.has(key)) {
            return null;
        }
        
        // Move to end (most recently used)
        const value = this.cache.get(key);
        this.cache.delete(key);
        this.cache.set(key, value);
        
        return value;
    }
    
    put(key, value) {
        // If exists, delete first (to update position)
        if (this.cache.has(key)) {
            this.cache.delete(key);
        }
        
        // If full, remove oldest (first) item
        if (this.cache.size >= this.capacity) {
            const oldest = this.cache.keys().next().value;
            this.cache.delete(oldest);
        }
        
        // Add new item at end
        this.cache.set(key, value);
    }
}

// Usage
const cache = new LRUCache(3);
cache.put('video:1', 'data1');
cache.put('video:2', 'data2');
cache.put('video:3', 'data3');
cache.get('video:1');          // Access video:1
cache.put('video:4', 'data4'); // Evicts video:2 (least recently used)
```

#### Pros & Cons:
```
✓ Simple to understand
✓ Works well for most cases
✓ Recent = likely to be accessed again

✗ Doesn't consider frequency
✗ One-time scans pollute cache
```

---

### 2. LFU (Least Frequently Used)

**Remove what has been used the fewest times**

```
Access counts:
video:A → accessed 100 times
video:B → accessed 5 times
video:C → accessed 50 times
video:D → accessed 3 times  ← EVICT (lowest count)
```

#### Implementation:
```javascript
class LFUCache {
    constructor(capacity) {
        this.capacity = capacity;
        this.cache = new Map();      // key → {value, freq}
        this.freqMap = new Map();    // freq → Set of keys
        this.minFreq = 0;
    }
    
    get(key) {
        if (!this.cache.has(key)) {
            return null;
        }
        
        const item = this.cache.get(key);
        this.updateFrequency(key, item);
        
        return item.value;
    }
    
    updateFrequency(key, item) {
        const oldFreq = item.freq;
        const newFreq = oldFreq + 1;
        
        // Remove from old frequency set
        this.freqMap.get(oldFreq).delete(key);
        if (this.freqMap.get(oldFreq).size === 0) {
            this.freqMap.delete(oldFreq);
            if (this.minFreq === oldFreq) {
                this.minFreq = newFreq;
            }
        }
        
        // Add to new frequency set
        if (!this.freqMap.has(newFreq)) {
            this.freqMap.set(newFreq, new Set());
        }
        this.freqMap.get(newFreq).add(key);
        
        // Update item frequency
        item.freq = newFreq;
    }
    
    put(key, value) {
        if (this.capacity === 0) return;
        
        if (this.cache.has(key)) {
            const item = this.cache.get(key);
            item.value = value;
            this.updateFrequency(key, item);
            return;
        }
        
        // Evict if full
        if (this.cache.size >= this.capacity) {
            const keysWithMinFreq = this.freqMap.get(this.minFreq);
            const evictKey = keysWithMinFreq.values().next().value;
            keysWithMinFreq.delete(evictKey);
            this.cache.delete(evictKey);
        }
        
        // Add new item with frequency 1
        this.cache.set(key, { value, freq: 1 });
        if (!this.freqMap.has(1)) {
            this.freqMap.set(1, new Set());
        }
        this.freqMap.get(1).add(key);
        this.minFreq = 1;
    }
}
```

#### Pros & Cons:
```
✓ Keeps frequently accessed items
✓ Good for hot/cold data patterns

✗ Doesn't adapt to changing patterns
✗ New items may be evicted quickly
✗ More complex to implement
```

---

### 3. FIFO (First In, First Out)

**Remove the oldest item (by insertion time)**

```
Insertion order: A → B → C → D → E (Cache size: 4)

Step 1: Add A     [A]
Step 2: Add B     [A, B]
Step 3: Add C     [A, B, C]
Step 4: Add D     [A, B, C, D]  ← Full!
Step 5: Add E     [B, C, D, E]  ← A removed (first in)

Access doesn't matter! Only insertion order.
```

#### Implementation:
```javascript
class FIFOCache {
    constructor(capacity) {
        this.capacity = capacity;
        this.cache = new Map();
    }
    
    get(key) {
        return this.cache.get(key) || null;
    }
    
    put(key, value) {
        // If exists, just update value (don't change position)
        if (this.cache.has(key)) {
            this.cache.set(key, value);
            return;
        }
        
        // If full, remove first item
        if (this.cache.size >= this.capacity) {
            const oldest = this.cache.keys().next().value;
            this.cache.delete(oldest);
        }
        
        this.cache.set(key, value);
    }
}
```

#### Pros & Cons:
```
✓ Simplest implementation
✓ Predictable behavior
✓ Low overhead

✗ Ignores access patterns
✗ May evict hot data
```

---

### 4. Random Replacement

**Remove a random item**

```
Cache: [A, B, C, D] ← Full!
Add E → Remove random (say B)
Cache: [A, C, D, E]
```

#### Implementation:
```javascript
class RandomCache {
    constructor(capacity) {
        this.capacity = capacity;
        this.cache = new Map();
        this.keys = [];
    }
    
    get(key) {
        return this.cache.get(key) || null;
    }
    
    put(key, value) {
        if (this.cache.has(key)) {
            this.cache.set(key, value);
            return;
        }
        
        if (this.cache.size >= this.capacity) {
            // Remove random key
            const randomIndex = Math.floor(Math.random() * this.keys.length);
            const evictKey = this.keys[randomIndex];
            this.cache.delete(evictKey);
            this.keys.splice(randomIndex, 1);
        }
        
        this.cache.set(key, value);
        this.keys.push(key);
    }
}
```

#### Pros & Cons:
```
✓ Very simple
✓ No tracking overhead
✓ Surprisingly effective sometimes

✗ Unpredictable
✗ May evict hot data
```

---

### 5. TTL (Time To Live)

**Remove items after they expire**

```
Set item with TTL:
video:A expires in 60 seconds
video:B expires in 300 seconds

After 60 seconds:
video:A → EXPIRED → Removed
video:B → Still valid
```

#### Implementation:
```javascript
class TTLCache {
    constructor() {
        this.cache = new Map();
    }
    
    get(key) {
        const item = this.cache.get(key);
        if (!item) return null;
        
        // Check if expired
        if (Date.now() > item.expiry) {
            this.cache.delete(key);
            return null;
        }
        
        return item.value;
    }
    
    put(key, value, ttlSeconds) {
        this.cache.set(key, {
            value,
            expiry: Date.now() + (ttlSeconds * 1000)
        });
    }
    
    // Cleanup expired items (run periodically)
    cleanup() {
        const now = Date.now();
        for (const [key, item] of this.cache) {
            if (now > item.expiry) {
                this.cache.delete(key);
            }
        }
    }
}

// Usage
const cache = new TTLCache();
cache.put('session:123', userData, 3600);  // Expires in 1 hour
```

#### Pros & Cons:
```
✓ Automatic staleness prevention
✓ Good for time-sensitive data
✓ Predictable expiration

✗ Not based on usage patterns
✗ May evict frequently used items
```

---

### 6. LRU-K (K References)

**Track last K accesses, not just most recent**

```
LRU-K where K=2:
Track second-to-last access time

Item A: Last access: 10:00, 2nd last: 9:00
Item B: Last access: 10:05, 2nd last: 8:00  ← Evict (older 2nd access)
Item C: Last access: 9:30, 2nd last: 9:30

Protects against scan patterns!
```

---

### 7. ARC (Adaptive Replacement Cache)

**Dynamically balance between LRU and LFU**

```
Maintains 4 lists:
T1: Recent items (LRU)
T2: Frequent items (LFU-like)
B1: Ghost entries from T1 (recently evicted)
B2: Ghost entries from T2 (recently evicted)

Adapts based on hit/miss patterns:
- If B1 hit → Increase T1 size (favor recency)
- If B2 hit → Increase T2 size (favor frequency)
```

---

## Comparison Table

```
┌───────────────┬────────────────┬─────────────────┬──────────────────┐
│   Policy      │   Best For     │   Complexity    │    Hit Rate      │
├───────────────┼────────────────┼─────────────────┼──────────────────┤
│   LRU         │ General use    │   Medium        │     Good         │
│   LFU         │ Hot/cold data  │   High          │     Better       │
│   FIFO        │ Simple cases   │   Low           │     Fair         │
│   Random      │ Unknown patterns│  Very Low      │     Fair         │
│   TTL         │ Time-sensitive │   Low           │     Varies       │
│   ARC         │ Adaptive needs │   Very High     │     Excellent    │
└───────────────┴────────────────┴─────────────────┴──────────────────┘
```

---

## Redis Eviction Policies

### Available Policies:
```
# redis.conf

# noeviction: Return error when memory full
maxmemory-policy noeviction

# allkeys-lru: LRU among ALL keys
maxmemory-policy allkeys-lru

# volatile-lru: LRU among keys WITH expiry
maxmemory-policy volatile-lru

# allkeys-lfu: LFU among ALL keys
maxmemory-policy allkeys-lfu

# volatile-lfu: LFU among keys WITH expiry
maxmemory-policy volatile-lfu

# allkeys-random: Random eviction
maxmemory-policy allkeys-random

# volatile-random: Random among keys WITH expiry
maxmemory-policy volatile-random

# volatile-ttl: Evict keys with shortest TTL
maxmemory-policy volatile-ttl
```

### Configuration:
```bash
# Set max memory
CONFIG SET maxmemory 1gb

# Set eviction policy
CONFIG SET maxmemory-policy allkeys-lru

# Check current policy
CONFIG GET maxmemory-policy
```

---

## YouTube Cache Eviction

### Strategy by Cache Type:

```
┌─────────────────────────────────────────────────────────────────┐
│                  YouTube Caching System                          │
└─────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│                      Video Cache                               │
│                                                                 │
│  Policy: LFU                                                   │
│  - Popular videos stay cached                                  │
│  - Viral videos get high frequency                             │
│  - Old, rarely watched videos evicted                          │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│                      Session Cache                             │
│                                                                 │
│  Policy: TTL (1 hour) + LRU                                    │
│  - Sessions expire after inactivity                            │
│  - Recent sessions prioritized                                 │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│                      Search Cache                              │
│                                                                 │
│  Policy: LRU + TTL (5 minutes)                                 │
│  - Fresh results important                                     │
│  - Recent searches more valuable                               │
└───────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│                   Recommendation Cache                         │
│                                                                 │
│  Policy: TTL (1 hour) + LFU                                    │
│  - Recommendations refresh periodically                        │
│  - Frequent users get priority                                 │
└───────────────────────────────────────────────────────────────┘
```

### Implementation:
```javascript
class YouTubeCacheManager {
    constructor() {
        // Video metadata - LFU (keep popular videos)
        this.videoCache = new Redis({
            keyPrefix: 'video:',
            maxmemory: '2gb',
            maxmemoryPolicy: 'allkeys-lfu'
        });
        
        // User sessions - TTL with LRU
        this.sessionCache = new Redis({
            keyPrefix: 'session:',
            maxmemory: '1gb',
            maxmemoryPolicy: 'volatile-lru'
        });
        
        // Search results - Short TTL
        this.searchCache = new Redis({
            keyPrefix: 'search:',
            maxmemory: '512mb',
            maxmemoryPolicy: 'volatile-ttl'
        });
    }
    
    async cacheVideo(videoId, data) {
        // LFU - accessed often = stays
        await this.videoCache.set(videoId, JSON.stringify(data));
    }
    
    async cacheSession(sessionId, data) {
        // TTL + LRU - expires and old sessions evicted
        await this.sessionCache.setex(
            sessionId,
            3600,  // 1 hour TTL
            JSON.stringify(data)
        );
    }
    
    async cacheSearch(query, results) {
        // Short TTL - fresh results
        await this.searchCache.setex(
            query,
            300,  // 5 minute TTL
            JSON.stringify(results)
        );
    }
}
```

---

## Choosing the Right Policy

### Decision Tree:
```
                    What's your use case?
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    Time-sensitive?    Hot/cold data?    General use?
          │                │                │
          ▼                ▼                ▼
        TTL              LFU              LRU
          │                │                │
          ▼                ▼                ▼
    Sessions          Popular items    Most cases
    API responses     Trending data    Good default
    News articles     User profiles    Simple apps
```

### Best Practices:
```python
POLICY_RECOMMENDATIONS = {
    # Time-sensitive data
    'sessions': 'volatile-lru + TTL',
    'api_cache': 'volatile-ttl',
    'news': 'volatile-ttl',
    
    # Popularity-based
    'video_metadata': 'allkeys-lfu',
    'user_profiles': 'allkeys-lfu',
    'product_catalog': 'allkeys-lfu',
    
    # General purpose
    'general': 'allkeys-lru',
    'database_cache': 'allkeys-lru',
    
    # Mixed workloads
    'adaptive': 'Use ARC or LRU-K',
}
```

---

## Interview Questions

**Q: What is cache eviction?**
A: The process of removing data from cache when it's full to make room for new data.

**Q: LRU vs LFU?**
A: LRU evicts least recently used. LFU evicts least frequently used. LRU is simpler, LFU better for hot data.

**Q: What policy would you use for a video platform?**
A: LFU for video content (popular stays cached), TTL+LRU for sessions, volatile-ttl for search results.

**Q: What is cache stampede and how does eviction relate?**
A: When many requests hit for evicted data simultaneously. Mitigate with staggered TTLs or background refresh.

**Q: How does Redis implement LRU?**
A: Approximated LRU - samples random keys and evicts the least recently used among samples (efficient).

---

## Quick Summary

```
EVICTION POLICIES:
──────────────────

LRU (Least Recently Used):
- Evicts what hasn't been touched longest
- Good default for most cases

LFU (Least Frequently Used):
- Evicts what's accessed least often
- Good for hot/cold data patterns

FIFO (First In First Out):
- Evicts oldest by insertion time
- Simple, ignores access patterns

TTL (Time To Live):
- Evicts after time expires
- Good for time-sensitive data

Random:
- Evicts random item
- Simple, unpredictable

REDIS POLICIES:
───────────────
allkeys-lru    → LRU on all keys
volatile-lru   → LRU on keys with TTL
allkeys-lfu    → LFU on all keys
volatile-ttl   → Shortest TTL first

CHOOSE BASED ON:
────────────────
Time-sensitive → TTL
Popular data   → LFU
General use    → LRU
Simple needs   → FIFO
```

You now understand cache eviction! 🗑️
