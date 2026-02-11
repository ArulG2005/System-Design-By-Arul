# Bloom Filters - Complete Guide

## What is a Bloom Filter?

Think of it like a **bouncer with a guest list**:
- Bouncer can quickly say "Definitely NOT on the list"
- Or "Probably on the list, let me check"
- Never says "definitely on list" when they're not
- Sometimes wrong about "probably on list"

**Simple Definition:**
A Bloom filter is a space-efficient probabilistic data structure that tells you if an element is "probably in set" or "definitely not in set".

---

## Why Bloom Filters?

### The Problem:
```
Check if username exists (1 billion users):

Option 1: Database query
SELECT * FROM users WHERE username = 'john';
→ Slow! Needs disk I/O every time

Option 2: HashSet in memory
Set with 1 billion usernames
→ Needs ~50GB RAM!

Option 3: Bloom Filter
Compact representation
→ Only ~1GB RAM, super fast!
```

### Trade-off:
```
┌────────────────────────────────────────────────────────┐
│ Bloom Filter Properties                                │
├────────────────────────────────────────────────────────┤
│                                                        │
│  "Definitely NOT in set"  → 100% accurate              │
│  "Probably in set"        → May have false positives   │
│                                                        │
│  ✓ NO false negatives (never misses existing items)   │
│  ✗ YES false positives (may say exists when not)      │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## How It Works

### Structure:
```
Bloom Filter = Bit array + Multiple hash functions

Bit Array (initially all 0s):
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5   6   7   8   9

Hash Functions:
h1(x), h2(x), h3(x) → Each returns position in array
```

### Adding an Element:
```
Add "apple":
h1("apple") = 2
h2("apple") = 5
h3("apple") = 9

Set bits at positions 2, 5, 9:
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 0 │ 1 │ 0 │ 0 │ 1 │ 0 │ 0 │ 0 │ 1 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5   6   7   8   9
          ▲           ▲               ▲
```

### Adding Another Element:
```
Add "banana":
h1("banana") = 1
h2("banana") = 5  (same as apple!)
h3("banana") = 7

Set bits at positions 1, 5, 7:
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 1 │ 0 │ 0 │ 1 │ 0 │ 1 │ 0 │ 1 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5   6   7   8   9
      ▲   ▲           ▲       ▲       ▲
```

### Checking Membership:
```
Is "apple" in set?
h1("apple") = 2 → bit[2] = 1 ✓
h2("apple") = 5 → bit[5] = 1 ✓
h3("apple") = 9 → bit[9] = 1 ✓
All bits set → "Probably YES"

Is "cherry" in set?
h1("cherry") = 3 → bit[3] = 0 ✗
→ "Definitely NO" (stop checking)

Is "grape" in set?
h1("grape") = 1 → bit[1] = 1 ✓
h2("grape") = 5 → bit[5] = 1 ✓
h3("grape") = 7 → bit[7] = 1 ✓
All bits set → "Probably YES" (FALSE POSITIVE!)
                (Bits were set by apple and banana)
```

---

## Implementation

### Basic Implementation:
```javascript
class BloomFilter {
    constructor(size, hashCount) {
        this.size = size;
        this.hashCount = hashCount;
        this.bitArray = new Array(size).fill(0);
    }
    
    // Simple hash functions using different seeds
    hash(item, seed) {
        let hash = 0;
        for (let i = 0; i < item.length; i++) {
            hash = (hash * seed + item.charCodeAt(i)) % this.size;
        }
        return Math.abs(hash);
    }
    
    // Get all hash positions for an item
    getHashPositions(item) {
        const positions = [];
        for (let i = 0; i < this.hashCount; i++) {
            positions.push(this.hash(item, i * 31 + 7));
        }
        return positions;
    }
    
    add(item) {
        const positions = this.getHashPositions(item);
        positions.forEach(pos => {
            this.bitArray[pos] = 1;
        });
    }
    
    mightContain(item) {
        const positions = this.getHashPositions(item);
        return positions.every(pos => this.bitArray[pos] === 1);
    }
}

// Usage
const filter = new BloomFilter(1000, 3);

filter.add('apple');
filter.add('banana');
filter.add('cherry');

console.log(filter.mightContain('apple'));   // true (correct)
console.log(filter.mightContain('banana'));  // true (correct)
console.log(filter.mightContain('grape'));   // false (correct) or true (false positive)
console.log(filter.mightContain('xyz'));     // false (definitely not in set)
```

### Optimized with Bit Manipulation:
```javascript
class OptimizedBloomFilter {
    constructor(expectedItems, falsePositiveRate) {
        // Calculate optimal size and hash count
        this.size = this.getOptimalSize(expectedItems, falsePositiveRate);
        this.hashCount = this.getOptimalHashCount(this.size, expectedItems);
        
        // Use typed array for memory efficiency
        this.bitArray = new Uint8Array(Math.ceil(this.size / 8));
    }
    
    getOptimalSize(n, p) {
        // m = -n * ln(p) / (ln(2)^2)
        return Math.ceil(-n * Math.log(p) / (Math.LN2 * Math.LN2));
    }
    
    getOptimalHashCount(m, n) {
        // k = m/n * ln(2)
        return Math.round((m / n) * Math.LN2);
    }
    
    setBit(pos) {
        const byteIndex = Math.floor(pos / 8);
        const bitIndex = pos % 8;
        this.bitArray[byteIndex] |= (1 << bitIndex);
    }
    
    getBit(pos) {
        const byteIndex = Math.floor(pos / 8);
        const bitIndex = pos % 8;
        return (this.bitArray[byteIndex] & (1 << bitIndex)) !== 0;
    }
    
    // Use murmur hash for better distribution
    hash(item, seed) {
        let h = seed;
        for (let i = 0; i < item.length; i++) {
            h ^= item.charCodeAt(i);
            h = Math.imul(h, 0x5bd1e995);
            h ^= h >>> 15;
        }
        return Math.abs(h) % this.size;
    }
    
    add(item) {
        for (let i = 0; i < this.hashCount; i++) {
            const pos = this.hash(item, i);
            this.setBit(pos);
        }
    }
    
    mightContain(item) {
        for (let i = 0; i < this.hashCount; i++) {
            const pos = this.hash(item, i);
            if (!this.getBit(pos)) {
                return false;  // Definitely not in set
            }
        }
        return true;  // Probably in set
    }
}

// Usage: 1 million items, 1% false positive rate
const filter = new OptimizedBloomFilter(1000000, 0.01);
// Uses about 1.2 MB instead of ~50 MB for a HashSet
```

---

## Formulas

### Optimal Parameters:
```
Given:
n = Expected number of items
p = Desired false positive rate (e.g., 0.01 for 1%)

Optimal bit array size (m):
m = -n × ln(p) / (ln(2))²

Optimal number of hash functions (k):
k = (m/n) × ln(2)

Example:
n = 1,000,000 (1 million usernames)
p = 0.01 (1% false positive rate)

m = -1,000,000 × ln(0.01) / (0.693)²
m ≈ 9,585,059 bits ≈ 1.2 MB

k = (9,585,059 / 1,000,000) × 0.693
k ≈ 7 hash functions
```

### False Positive Rate:
```
As filter fills up, false positive rate increases:

p = (1 - e^(-kn/m))^k

Where:
k = number of hash functions
n = number of items added
m = size of bit array
```

---

## Use Cases

### 1. Username/Email Existence Check
```javascript
// Fast check before hitting database
class UserService {
    constructor() {
        this.usernameFilter = new BloomFilter(10000000, 7);
        this.loadExistingUsernames();
    }
    
    async loadExistingUsernames() {
        const usernames = await db.query('SELECT username FROM users');
        usernames.forEach(u => this.usernameFilter.add(u.username));
    }
    
    async isUsernameTaken(username) {
        // Quick bloom filter check
        if (!this.usernameFilter.mightContain(username)) {
            return false;  // Definitely available!
        }
        
        // Might be taken, verify with database
        return await db.query(
            'SELECT 1 FROM users WHERE username = ?',
            [username]
        ).then(r => r.length > 0);
    }
}
```

### 2. Cache Miss Optimization
```javascript
// Prevent cache stampede for non-existent keys
class SmartCache {
    constructor() {
        this.cache = new Redis();
        this.existsFilter = new BloomFilter(1000000, 7);
    }
    
    async get(key) {
        // Check bloom filter first
        if (!this.existsFilter.mightContain(key)) {
            return null;  // Definitely doesn't exist
        }
        
        // Might exist, check cache
        return await this.cache.get(key);
    }
    
    async set(key, value) {
        await this.cache.set(key, value);
        this.existsFilter.add(key);
    }
}
```

### 3. Web Crawler URL Deduplication
```javascript
// Track visited URLs without storing all of them
class WebCrawler {
    constructor() {
        this.visitedFilter = new BloomFilter(100000000, 10);
        this.queue = [];
    }
    
    async crawl(startUrl) {
        this.queue.push(startUrl);
        
        while (this.queue.length > 0) {
            const url = this.queue.shift();
            
            // Skip if probably already visited
            if (this.visitedFilter.mightContain(url)) {
                continue;
            }
            
            // Mark as visited
            this.visitedFilter.add(url);
            
            // Crawl page and add new URLs
            const links = await this.fetchLinks(url);
            links.forEach(link => {
                if (!this.visitedFilter.mightContain(link)) {
                    this.queue.push(link);
                }
            });
        }
    }
}
```

### 4. Spell Checker
```javascript
// Fast dictionary lookup
class SpellChecker {
    constructor() {
        this.dictionaryFilter = new BloomFilter(500000, 7);
        this.loadDictionary();
    }
    
    loadDictionary() {
        const words = fs.readFileSync('dictionary.txt', 'utf8').split('\n');
        words.forEach(word => this.dictionaryFilter.add(word.toLowerCase()));
    }
    
    isValidWord(word) {
        return this.dictionaryFilter.mightContain(word.toLowerCase());
        // False positives OK - we'd rather not flag valid words
    }
}
```

---

## YouTube Use Cases

```
┌─────────────────────────────────────────────────────────────────┐
│                YouTube Bloom Filter Usage                        │
└─────────────────────────────────────────────────────────────────┘

1. Watched Video History
   ┌─────────────────────────────────────────────┐
   │ User's watched video IDs in Bloom Filter    │
   │                                             │
   │ "Has user watched video X?"                 │
   │ - NO → Show "New" badge                     │
   │ - Probably YES → Check DB to confirm        │
   │                                             │
   │ Saves millions of DB queries!               │
   └─────────────────────────────────────────────┘

2. Recommendation Deduplication
   ┌─────────────────────────────────────────────┐
   │ "Don't recommend videos user already saw"   │
   │                                             │
   │ Bloom filter of watched + dismissed videos  │
   │ Filter recommendations against it           │
   └─────────────────────────────────────────────┘

3. Spam Comment Detection
   ┌─────────────────────────────────────────────┐
   │ Known spam phrases in Bloom filter          │
   │                                             │
   │ Quick first-pass check before expensive     │
   │ ML-based spam detection                     │
   └─────────────────────────────────────────────┘

4. Username Availability
   ┌─────────────────────────────────────────────┐
   │ All usernames in Bloom filter               │
   │                                             │
   │ Instant "Username available!" for new names │
   │ "Might be taken" triggers DB check          │
   └─────────────────────────────────────────────┘
```

### Implementation:
```javascript
class YouTubeBloomFilters {
    constructor() {
        // 10M videos per user history, 0.1% false positive
        this.watchedFilters = new Map();  // userId → BloomFilter
        
        // 100M total usernames, 0.01% false positive  
        this.usernameFilter = new BloomFilter(100000000, 10);
    }
    
    // Watched history
    getUserWatchedFilter(userId) {
        if (!this.watchedFilters.has(userId)) {
            this.watchedFilters.set(userId, new BloomFilter(10000, 7));
        }
        return this.watchedFilters.get(userId);
    }
    
    markVideoWatched(userId, videoId) {
        this.getUserWatchedFilter(userId).add(videoId);
    }
    
    hasUserProbablyWatched(userId, videoId) {
        return this.getUserWatchedFilter(userId).mightContain(videoId);
    }
    
    // Filter recommendations
    filterRecommendations(userId, recommendations) {
        const watchedFilter = this.getUserWatchedFilter(userId);
        
        return recommendations.filter(video => {
            // Keep if definitely not watched
            return !watchedFilter.mightContain(video.id);
        });
    }
}
```

---

## Counting Bloom Filters

### Problem with Standard Bloom Filter:
```
Can't delete elements!

Why? Setting bit to 0 might affect other elements:

apple sets bits: 2, 5, 9
banana sets bits: 1, 5, 7   (5 is shared!)

Delete apple → set bits 2, 5, 9 to 0
Now banana lookup fails! (bit 5 is 0)
```

### Solution: Counting Bloom Filter
```
Use counters instead of bits:

┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 1 │ 1 │ 0 │ 0 │ 2 │ 0 │ 1 │ 0 │ 1 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5   6   7   8   9
                      ▲
                 Count = 2 (shared by 2 items)

Delete = Decrement counters
Add = Increment counters
Check = All counters > 0?
```

```javascript
class CountingBloomFilter {
    constructor(size, hashCount) {
        this.size = size;
        this.hashCount = hashCount;
        this.counters = new Uint8Array(size);  // Max count 255
    }
    
    add(item) {
        for (let i = 0; i < this.hashCount; i++) {
            const pos = this.hash(item, i);
            if (this.counters[pos] < 255) {
                this.counters[pos]++;
            }
        }
    }
    
    remove(item) {
        // Only remove if probably exists
        if (!this.mightContain(item)) return false;
        
        for (let i = 0; i < this.hashCount; i++) {
            const pos = this.hash(item, i);
            if (this.counters[pos] > 0) {
                this.counters[pos]--;
            }
        }
        return true;
    }
    
    mightContain(item) {
        for (let i = 0; i < this.hashCount; i++) {
            const pos = this.hash(item, i);
            if (this.counters[pos] === 0) {
                return false;
            }
        }
        return true;
    }
}
```

---

## Interview Questions

**Q: What is a Bloom filter?**
A: A space-efficient probabilistic data structure that can tell you "definitely not in set" or "probably in set". Has false positives but no false negatives.

**Q: When would you use a Bloom filter?**
A: When you need fast membership checking with limited memory: username existence, cache miss prevention, URL deduplication, spam detection.

**Q: Can you delete from a Bloom filter?**
A: Not from standard Bloom filter (bits shared between items). Use Counting Bloom Filter for deletions.

**Q: How do you size a Bloom filter?**
A: Based on expected items (n) and desired false positive rate (p). m = -n × ln(p) / (ln(2))². More space = lower false positives.

**Q: What's the trade-off?**
A: Space efficiency vs accuracy. Smaller filter = more false positives. Can't get exact membership, only probabilistic.

---

## Quick Summary

```
BLOOM FILTER:
─────────────
Probabilistic data structure
"Definitely NOT" or "Probably YES"

HOW IT WORKS:
─────────────
- Bit array + multiple hash functions
- Add: Set bits at hash positions
- Check: All bits set? Probably exists

PROPERTIES:
───────────
✓ No false negatives (100% accurate for "not exists")
✗ Has false positives (may say exists when not)
✓ Very space efficient
✓ O(k) add/lookup (k = hash functions)

PARAMETERS:
───────────
m = bit array size
k = number of hash functions
n = number of items
p = false positive rate

FORMULAS:
─────────
m = -n × ln(p) / (ln(2))²
k = (m/n) × ln(2)

USE CASES:
──────────
- Username/email existence
- Cache miss prevention
- URL deduplication
- Spell checkers
- Recommendation filtering
```

You now understand Bloom filters! 🌸
