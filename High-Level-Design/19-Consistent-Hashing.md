# Consistent Hashing - Complete Guide

## What is Consistent Hashing?

Think of it like a **clock**:
- Numbers 1-12 around the circle
- Assign each server a number on the clock
- Assign each data item a number on the clock
- Data goes to the next server clockwise

**Simple Definition:**
A technique to distribute data across servers where adding/removing servers only moves minimal data.

---

## The Problem It Solves

### Traditional Hashing:
```
servers = ["Server0", "Server1", "Server2"]

Which server for "video_123"?
hash("video_123") % 3 = 2 → Server2

Works great... until you add a server!

servers = ["Server0", "Server1", "Server2", "Server3"]  # Added one

hash("video_123") % 4 = 1 → Server1  (DIFFERENT!)

ALL data needs to move! 😱
```

### The Disaster:
```
Before (3 servers):
┌─────────┬─────────┬─────────┐
│ Server0 │ Server1 │ Server2 │
│ 33% data│ 33% data│ 33% data│
└─────────┴─────────┴─────────┘

After adding Server3 (4 servers):
hash(key) % 4 gives different results!

~75% of ALL data moves to wrong server!
Cache misses everywhere, system crashes! 💀
```

---

## Consistent Hashing Solution

### The Hash Ring:
```
Imagine a circle from 0 to 2^32:

                    0
                   ╱│╲
                 ╱  │  ╲
               ╱    │    ╲
        2^32-1      │      2^31/4
              ╲     │     ╱
               ╲    │    ╱
                ╲   │   ╱
                  ╲ │ ╱
                  2^31

Or simplified (0 to 360 degrees):

              0°
              │
              │
    270° ─────●───── 90°
              │
              │
             180°
```

### How It Works:

#### Step 1: Place Servers on Ring
```
hash("Server_A") → 45°
hash("Server_B") → 135°
hash("Server_C") → 270°

              0°
              │
         A ───●──────┐
    270° ─●───┼──────●── 90°
         C    │      │
              │      B
             180°
              │
              ●
```

#### Step 2: Place Data on Ring
```
hash("video_123") → 80°

              0°
              │
         A ───●───x──┐   x = video_123 (80°)
    270° ─●───┼──────●── 90°
         C    │      │
              │      B
             180°

video_123 goes to next server clockwise = B (135°)
```

#### Step 3: Adding a Server
```
Add Server_D at 60°

              0°
              │
         A ───●─D─x──┐   D is new server at 60°
    270° ─●───┼──────●── 90°
         C    │      │
              │      B
             180°

video_123 (80°) now goes to B (unchanged!)

Only data between A (45°) and D (60°) moves!
That's just ~4% of data, not 75%!
```

---

## Visual Example

### Before Adding Server D:
```
Ring: 0 ────────────────────────────────────────► 360°

      │    A          B               C          │
      │    │          │               │          │
      0    45        135             270        360

Data assignment (go clockwise to next server):
- 0°  to  45° → Server A
- 45° to 135° → Server B  ← videos here
- 135° to 270° → Server C
- 270° to 360° → Server A (wraps around)
```

### After Adding Server D at 60°:
```
Ring: 0 ────────────────────────────────────────► 360°

      │    A    D     B               C          │
      │    │    │     │               │          │
      0   45   60    135             270        360

New data assignment:
- 0°  to  45° → Server A
- 45° to  60° → Server D (NEW - only this moves!)
- 60° to 135° → Server B  ← videos still here!
- 135° to 270° → Server C
- 270° to 360° → Server A

Only data from 45°-60° moves = ~4% of data!
```

---

## The Imbalance Problem

### Problem: Uneven Distribution
```
      0°
      │
 A ───●───────────────┐
      │               │
      │               │
      │               │
      │               B 90°
     
Server A: Handles 75% of ring!
Server B: Handles 25% of ring!

Super unbalanced! A will crash!
```

### Solution: Virtual Nodes
```
Each physical server gets MULTIPLE positions on ring:

Server A gets: A1, A2, A3
Server B gets: B1, B2, B3

      0°
      │
 A1 ──●── B1 ──┐
      │       │
 B2 ──●       │
      │       A2
 A3 ──●       │
      │       │
      └── B3 ─┘

Now load is evenly distributed!
More virtual nodes = More even distribution
```

### Virtual Nodes Implementation:
```python
# Each physical server has multiple virtual nodes
virtual_nodes = []
for server in physical_servers:
    for i in range(100):  # 100 virtual nodes per server
        virtual_id = f"{server.name}#vn{i}"
        position = hash(virtual_id) % RING_SIZE
        virtual_nodes.append({
            'position': position,
            'physical_server': server
        })

# Sort by position for efficient lookup
virtual_nodes.sort(key=lambda x: x['position'])
```

---

## Implementation

### Python Implementation:
```python
import hashlib
import bisect

class ConsistentHash:
    def __init__(self, nodes=None, virtual_nodes=100):
        self.virtual_nodes = virtual_nodes
        self.ring = {}  # position → node
        self.sorted_keys = []  # sorted positions
        
        if nodes:
            for node in nodes:
                self.add_node(node)
    
    def _hash(self, key):
        """Hash a key to a position on ring (0 to 2^32-1)"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16) % (2**32)
    
    def add_node(self, node):
        """Add a node with virtual nodes"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}#vn{i}"
            position = self._hash(virtual_key)
            self.ring[position] = node
            bisect.insort(self.sorted_keys, position)
    
    def remove_node(self, node):
        """Remove a node and its virtual nodes"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}#vn{i}"
            position = self._hash(virtual_key)
            del self.ring[position]
            self.sorted_keys.remove(position)
    
    def get_node(self, key):
        """Get the node responsible for this key"""
        if not self.ring:
            return None
        
        position = self._hash(key)
        
        # Find first node position >= key position
        idx = bisect.bisect(self.sorted_keys, position)
        
        # Wrap around to first node if at end
        if idx == len(self.sorted_keys):
            idx = 0
        
        return self.ring[self.sorted_keys[idx]]

# Usage
ch = ConsistentHash(['Server_A', 'Server_B', 'Server_C'])

# Get server for a video
server = ch.get_node('video_123')
print(f"video_123 → {server}")

# Add a new server
ch.add_node('Server_D')

# Most mappings stay the same!
server = ch.get_node('video_123')
print(f"video_123 → {server}")  # Probably same server
```

### JavaScript Implementation:
```javascript
const crypto = require('crypto');

class ConsistentHash {
    constructor(nodes = [], virtualNodes = 100) {
        this.virtualNodes = virtualNodes;
        this.ring = new Map();
        this.sortedKeys = [];
        
        nodes.forEach(node => this.addNode(node));
    }
    
    hash(key) {
        return parseInt(
            crypto.createHash('md5').update(key).digest('hex').slice(0, 8),
            16
        );
    }
    
    addNode(node) {
        for (let i = 0; i < this.virtualNodes; i++) {
            const virtualKey = `${node}#vn${i}`;
            const position = this.hash(virtualKey);
            this.ring.set(position, node);
            this.sortedKeys.push(position);
        }
        this.sortedKeys.sort((a, b) => a - b);
    }
    
    removeNode(node) {
        for (let i = 0; i < this.virtualNodes; i++) {
            const virtualKey = `${node}#vn${i}`;
            const position = this.hash(virtualKey);
            this.ring.delete(position);
            const idx = this.sortedKeys.indexOf(position);
            if (idx > -1) this.sortedKeys.splice(idx, 1);
        }
    }
    
    getNode(key) {
        if (this.ring.size === 0) return null;
        
        const position = this.hash(key);
        
        // Binary search for first position >= key position
        let idx = this.binarySearch(position);
        
        // Wrap around
        if (idx >= this.sortedKeys.length) idx = 0;
        
        return this.ring.get(this.sortedKeys[idx]);
    }
    
    binarySearch(target) {
        let left = 0;
        let right = this.sortedKeys.length;
        
        while (left < right) {
            const mid = Math.floor((left + right) / 2);
            if (this.sortedKeys[mid] < target) {
                left = mid + 1;
            } else {
                right = mid;
            }
        }
        return left;
    }
}

// Usage
const ch = new ConsistentHash(['ServerA', 'ServerB', 'ServerC']);

console.log('video_123 →', ch.getNode('video_123'));
console.log('video_456 →', ch.getNode('video_456'));

ch.addNode('ServerD');
console.log('After adding ServerD:');
console.log('video_123 →', ch.getNode('video_123'));
```

---

## Replication with Consistent Hashing

### Store Data on Multiple Nodes:
```
Replication Factor = 3

For key "video_123":
1. Hash to position 80°
2. Find next 3 nodes clockwise

              0°
              │
         A ───●─D───x─┐   x = video_123
    270° ─●───┼───────●── 90°
         C    │       │
              │       B
             180°

video_123 stored on: B, C, A (next 3 clockwise)
```

### Implementation:
```python
def get_nodes(self, key, replicas=3):
    """Get multiple nodes for replication"""
    if len(self.ring) < replicas:
        return list(set(self.ring.values()))
    
    position = self._hash(key)
    idx = bisect.bisect(self.sorted_keys, position)
    
    nodes = []
    seen = set()
    
    while len(nodes) < replicas:
        if idx >= len(self.sorted_keys):
            idx = 0
        
        node = self.ring[self.sorted_keys[idx]]
        
        # Skip virtual nodes of same physical server
        if node not in seen:
            nodes.append(node)
            seen.add(node)
        
        idx += 1
    
    return nodes
```

---

## YouTube with Consistent Hashing

### Video Cache Distribution:
```
YouTube Video Cache:

                    0
                   ╱│╲
         Cache1  ╱  │  ╲  Cache2
               ╱    │    ╲
              ╲     │     ╱
     Cache4    ╲    │    ╱  Cache3
                  ╲ │ ╱
                   ╲│╱
                   180

User requests video "abc123":
1. hash("abc123") → position on ring
2. Find cache server (clockwise)
3. Get from that cache (or origin if miss)

Adding new cache server:
- Only ~25% of videos need to move
- Hot videos stay in place
- No cache stampede!
```

### User Session Distribution:
```javascript
class SessionStore {
    constructor() {
        this.hashRing = new ConsistentHash([
            'redis-1.internal',
            'redis-2.internal',
            'redis-3.internal'
        ]);
    }
    
    async getSession(sessionId) {
        const server = this.hashRing.getNode(sessionId);
        const redis = this.getRedisClient(server);
        return await redis.get(`session:${sessionId}`);
    }
    
    async setSession(sessionId, data) {
        const server = this.hashRing.getNode(sessionId);
        const redis = this.getRedisClient(server);
        await redis.setex(`session:${sessionId}`, 3600, JSON.stringify(data));
    }
    
    addServer(serverAddress) {
        this.hashRing.addNode(serverAddress);
        // Only sessions that hash to new server's range move!
    }
}
```

---

## Consistent Hashing vs Alternatives

### Modulo Hashing:
```
hash(key) % num_servers

Problems:
- Adding server: ~(N-1)/N data moves
- 3→4 servers: 75% data moves
- 10→11 servers: 91% data moves
```

### Consistent Hashing:
```
Place on ring, go clockwise

Benefits:
- Adding server: ~1/N data moves
- 3→4 servers: 25% data moves
- 10→11 servers: 9% data moves
```

### Rendezvous Hashing:
```
For each key, compute hash with each server
Pick server with highest hash

Benefits:
- Also minimal data movement
- Simpler than ring

Drawback:
- O(N) computation per lookup
```

---

## Real-World Uses

### 1. Amazon DynamoDB
```
Partition key hashed to ring
Data distributed across partitions
Virtual nodes for balance
```

### 2. Apache Cassandra
```
Cluster ring topology
Each node owns token range
Consistent hashing for distribution
```

### 3. Discord
```
User connections distributed
Server failures minimal impact
```

### 4. Memcached/Redis Cluster
```
Cache keys distributed
Adding nodes doesn't flush all cache
```

---

## Interview Questions

**Q: What is consistent hashing?**
A: Technique to distribute data where adding/removing servers only moves K/N data (K=keys, N=servers) instead of most data.

**Q: How does it work?**
A: Hash servers and keys to a ring. Key goes to first server clockwise from its position.

**Q: What are virtual nodes?**
A: Multiple positions per physical server on ring. Ensures even distribution and handles heterogeneous servers.

**Q: Why use it instead of modulo?**
A: Modulo hashing moves ~75% data when adding 1 server to 3. Consistent hashing moves ~25%.

**Q: Where is it used?**
A: DynamoDB, Cassandra, Redis Cluster, CDNs, load balancers, distributed caches.

---

## Quick Summary

```
CONSISTENT HASHING:
───────────────────
- Hash ring (0 to 2^32)
- Servers placed on ring
- Keys placed on ring
- Key → Next server clockwise

BENEFITS:
─────────
- Add server: Only ~1/N data moves
- Remove server: Only its data redistributes
- No cache stampede
- Smooth scaling

VIRTUAL NODES:
──────────────
- Multiple positions per server
- Even distribution
- Handle different server capacities

FORMULA:
────────
Traditional: hash(key) % N servers
Consistent:  hash(key) → ring position → next server

DATA MOVEMENT COMPARISON:
─────────────────────────
Traditional (3→4 servers): ~75% moves
Consistent (3→4 servers):  ~25% moves

USE FOR:
────────
- Distributed caches
- Database sharding
- Load balancing
- CDN routing
```

You now understand consistent hashing! 💍
