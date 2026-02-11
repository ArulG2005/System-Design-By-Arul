# Distributed Systems - Complete Guide

## What is a Distributed System?

Think of it like a **pizza chain**:
- Multiple stores (nodes) in different locations
- All stores share orders and inventory info
- If one store closes, others keep serving
- Customers don't care which store handles their order

**Simple Definition:**
A distributed system is a collection of computers that appear to users as a single coherent system.

---

## Why Distributed Systems?

```
Single Computer Problems:
┌───────────────────┐
│  One Server       │
│  - Limited CPU    │
│  - Limited RAM    │
│  - Single failure │
│    = Total outage │
└───────────────────┘

Distributed Solution:
┌─────────┐ ┌─────────┐ ┌─────────┐
│Server 1 │ │Server 2 │ │Server 3 │
│  CPU 1  │ │  CPU 2  │ │  CPU 3  │
│  RAM 1  │ │  RAM 2  │ │  RAM 3  │
└────┬────┘ └────┬────┘ └────┬────┘
     │           │           │
     └───────────┼───────────┘
                 │
         Combined Power!
         Any can fail!
```

### Benefits:
1. **Scalability** - Add more machines as needed
2. **Reliability** - No single point of failure
3. **Performance** - Parallel processing
4. **Geographic** - Servers near users

---

## Core Concepts

### 1. Nodes
```
A node is any computer in the system:

┌─────────────────────────────────────────┐
│            Distributed System            │
├─────────────┬─────────────┬─────────────┤
│   Node 1    │   Node 2    │   Node 3    │
│  (Server)   │  (Server)   │  (Server)   │
│  10.0.0.1   │  10.0.0.2   │  10.0.0.3   │
│  NYC        │  LA         │  London     │
└─────────────┴─────────────┴─────────────┘
```

### 2. Network Communication
```
Nodes talk over network:

Node A                          Node B
  │                               │
  ├────── Request Message ──────►│
  │                               │
  │◄───── Response Message ───────┤
  │                               │

Problems:
- Network can fail
- Messages can be lost
- Messages can be delayed
- Messages can arrive out of order
```

### 3. State
```
Shared State Problem:

Before:
Node A: balance = $100
Node B: balance = $100

User sends: -$30
             ↓
    ┌────────┴────────┐
    ↓                 ↓
  Node A           Node B
  $100-$30         $100-$30
  = $70            = $70

After (if both updated):
Node A: balance = $70 ✓
Node B: balance = $70 ✓

But what if:
- User sends -$30 to Node A
- User sends -$50 to Node B
- Before they sync?

Consistency problem!
```

---

## Fallacies of Distributed Computing

### 8 Wrong Assumptions:
```
1. The network is reliable
   Reality: Packets drop, connections fail

2. Latency is zero
   Reality: Network has delay (ms to seconds)

3. Bandwidth is infinite
   Reality: Limited capacity

4. The network is secure
   Reality: Anyone can intercept

5. Topology doesn't change
   Reality: Nodes join/leave constantly

6. There is one administrator
   Reality: Multiple teams, companies

7. Transport cost is zero
   Reality: Data transfer has cost

8. The network is homogeneous
   Reality: Different hardware, protocols
```

---

## Types of Distributed Systems

### 1. Client-Server
```
┌─────────┐
│ Client  │────┐
└─────────┘    │
               ▼
┌─────────┐  ┌─────────┐
│ Client  │──│ Server  │
└─────────┘  └─────────┘
               ▲
┌─────────┐    │
│ Client  │────┘
└─────────┘

Example: Web applications
```

### 2. Peer-to-Peer (P2P)
```
┌─────────┐     ┌─────────┐
│ Peer 1  │◄───►│ Peer 2  │
└────┬────┘     └────┬────┘
     │               │
     │  ┌─────────┐  │
     └─►│ Peer 3  │◄─┘
        └─────────┘

Everyone is equal
Example: BitTorrent, Bitcoin
```

### 3. Microservices
```
┌─────────┐ ┌─────────┐ ┌─────────┐
│  User   │ │ Video   │ │ Search  │
│ Service │ │ Service │ │ Service │
└────┬────┘ └────┬────┘ └────┬────┘
     │           │           │
     └───────────┼───────────┘
                 │
         Message Bus/API
```

---

## Consistency Models

### Strong Consistency
```
Write: x = 5

        ┌──────────────┐
        │  Write x=5   │
        └──────┬───────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
 Node A     Node B     Node C
  x=5        x=5        x=5
  
All reads immediately return 5
(Slow but accurate)
```

### Eventual Consistency
```
Write: x = 5

        ┌──────────────┐
        │  Write x=5   │
        └──────┬───────┘
               │
               ▼
            Node A
             x=5
               │ (propagates slowly)
    ┌──────────┴──────────┐
    ▼                     ▼
 Node B                Node C
  x=5                   x=? (old value, then 5)
  
Eventually all return 5
(Fast but temporarily inconsistent)
```

### Causal Consistency
```
If A causes B, everyone sees A before B

User posts: "I got a new job!"  (A)
User posts: "Thanks for congratulations!" (B)

B causally depends on A
Everyone sees A before B
```

---

## Replication

### Why Replicate?
```
Original → Copy → Copy → Copy

Benefits:
1. High availability (one dies, others work)
2. Better performance (read from nearest)
3. Data safety (backup)
```

### Replication Strategies

#### 1. Primary-Replica (Master-Slave)
```
         ┌──────────┐
         │ Primary  │ ◄── All Writes
         │  (Master)│
         └────┬─────┘
              │ Sync
    ┌─────────┼─────────┐
    ▼         ▼         ▼
┌──────┐  ┌──────┐  ┌──────┐
│Replica│  │Replica│  │Replica│ ◄── Reads
└──────┘  └──────┘  └──────┘
```

#### 2. Multi-Primary (Master-Master)
```
┌──────────┐     ┌──────────┐
│ Primary 1│◄───►│ Primary 2│
│(Writes OK)│     │(Writes OK)│
└──────────┘     └──────────┘

Both accept writes
Must handle conflicts!
```

#### 3. Leaderless
```
     Client
       │
    ┌──┼──┬──┐
    ▼  ▼  ▼  ▼
   N1  N2 N3 N4

Write to R nodes
Read from R nodes
Quorum: R + W > N
```

---

## Partitioning (Sharding)

### Horizontal Partitioning
```
Users Table → Split by user_id

     Shard 1          Shard 2          Shard 3
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│ Users 1-1M  │  │ Users 1M-2M │  │ Users 2M+   │
└─────────────┘  └─────────────┘  └─────────────┘
```

### Consistent Hashing
```
        0
       ╱│╲
     ╱  │  ╲
   ╱    │    ╲
 270────●────90
   ╲    │    ╱
     ╲  │  ╱
       ╲│╱
       180

Nodes placed on ring
Data hashed to ring
Goes to next node clockwise
```

---

## Consensus

### The Problem
```
Node A: "Let's set x = 5"
Node B: "Let's set x = 7"
Node C: "What should x be?"

How do we agree?
```

### Raft Consensus Algorithm
```
1. Leader Election:
   - Nodes vote for leader
   - Leader makes decisions
   
2. Log Replication:
   - Leader gets request
   - Leader writes to log
   - Leader sends to followers
   - Majority confirm → committed

┌────────┐     ┌──────────┐     ┌──────────┐
│ Leader │────►│ Follower │────►│ Follower │
│        │     │          │     │          │
│ Log:   │     │ Log:     │     │ Log:     │
│ [x=5]  │     │ [x=5]    │     │ [x=5]    │
└────────┘     └──────────┘     └──────────┘
```

### Paxos
```
Three Roles:
- Proposers: Propose values
- Acceptors: Accept/reject proposals
- Learners: Learn decided value

Phase 1: Prepare
Phase 2: Accept
Phase 3: Learn

(Complex but proven correct)
```

---

## Failure Handling

### Types of Failures
```
1. Crash Failure
   Node stops working completely
   
2. Omission Failure
   Node fails to send/receive messages
   
3. Timing Failure
   Node responds too slow
   
4. Byzantine Failure
   Node behaves maliciously/randomly
```

### Handling Failures
```javascript
// Retry with exponential backoff
async function reliableCall(fn, maxRetries = 3) {
    for (let i = 0; i < maxRetries; i++) {
        try {
            return await fn();
        } catch (error) {
            const delay = Math.pow(2, i) * 1000; // 1s, 2s, 4s
            await sleep(delay);
        }
    }
    throw new Error('Max retries exceeded');
}

// Circuit breaker
class CircuitBreaker {
    constructor(threshold = 5) {
        this.failures = 0;
        this.threshold = threshold;
        this.state = 'CLOSED';
        this.nextTry = Date.now();
    }
    
    async call(fn) {
        if (this.state === 'OPEN') {
            if (Date.now() < this.nextTry) {
                throw new Error('Circuit is OPEN');
            }
            this.state = 'HALF-OPEN';
        }
        
        try {
            const result = await fn();
            this.onSuccess();
            return result;
        } catch (error) {
            this.onFailure();
            throw error;
        }
    }
    
    onSuccess() {
        this.failures = 0;
        this.state = 'CLOSED';
    }
    
    onFailure() {
        this.failures++;
        if (this.failures >= this.threshold) {
            this.state = 'OPEN';
            this.nextTry = Date.now() + 30000; // 30 seconds
        }
    }
}
```

---

## Time in Distributed Systems

### The Problem
```
Node A: Clock says 10:00:00
Node B: Clock says 10:00:02
Node C: Clock says 09:59:58

2 second drift!
Whose clock is right?
```

### Solutions

#### 1. NTP (Network Time Protocol)
```
All nodes sync with time server
Still has millisecond drift
Not perfect for ordering events
```

#### 2. Logical Clocks (Lamport)
```
Events ordered by logical time, not real time

Node A: Event 1 (time=1), Event 4 (time=4)
Node B: Event 2 (time=2), Event 3 (time=3)

Send message: include your clock
Receive message: max(your_clock, msg_clock) + 1
```

#### 3. Vector Clocks
```
Track causality more precisely

Node A: [1,0,0] → [2,0,0]
Node B: [0,1,0] → [2,2,0] (after msg from A)
Node C: [0,0,1] → [2,2,2] (after msg from B)
```

---

## Distributed Transactions

### Two-Phase Commit (2PC)
```
Phase 1: Prepare
Coordinator: "Everyone ready to commit?"
Node A: "Yes"
Node B: "Yes"
Node C: "Yes"

Phase 2: Commit
Coordinator: "All commit!"
Node A: *commits*
Node B: *commits*
Node C: *commits*

If any says "No" → All abort
```

```javascript
// Coordinator
class TwoPhaseCommit {
    async execute(participants, transaction) {
        // Phase 1: Prepare
        const votes = await Promise.all(
            participants.map(p => p.prepare(transaction))
        );
        
        const allReady = votes.every(v => v === 'YES');
        
        // Phase 2: Commit or Abort
        if (allReady) {
            await Promise.all(
                participants.map(p => p.commit(transaction))
            );
            return 'COMMITTED';
        } else {
            await Promise.all(
                participants.map(p => p.abort(transaction))
            );
            return 'ABORTED';
        }
    }
}
```

### Saga Pattern
```
For long-running distributed transactions:

T1 → T2 → T3 → T4 (success)

If T3 fails:
T1 → T2 → T3 (fail)
          ↓
      C2 ← C1 (compensate backwards)

Each step has a compensation step
```

---

## YouTube as Distributed System

```
                           ┌─────────────────────────────────┐
                           │         Load Balancer           │
                           └────────────────┬────────────────┘
                                            │
              ┌─────────────────────────────┼─────────────────────────────┐
              │                             │                             │
              ▼                             ▼                             ▼
       ┌──────────────┐              ┌──────────────┐              ┌──────────────┐
       │ API Server 1 │              │ API Server 2 │              │ API Server N │
       │  (Region A)  │              │  (Region B)  │              │  (Region C)  │
       └──────┬───────┘              └──────┬───────┘              └──────┬───────┘
              │                             │                             │
              └─────────────────────────────┼─────────────────────────────┘
                                            │
       ┌────────────────────────────────────┼────────────────────────────────────┐
       │                                    │                                    │
       ▼                                    ▼                                    ▼
┌──────────────┐                    ┌──────────────┐                    ┌──────────────┐
│ Video Service│                    │ User Service │                    │Search Service│
│   Cluster    │                    │    Cluster   │                    │    Cluster   │
└──────┬───────┘                    └──────┬───────┘                    └──────┬───────┘
       │                                    │                                    │
       ▼                                    ▼                                    ▼
┌──────────────┐                    ┌──────────────┐                    ┌──────────────┐
│  Video DB    │                    │   User DB    │                    │ Search Index │
│  (Sharded)   │                    │  (Replicated)│                    │(Elasticsearch)│
└──────────────┘                    └──────────────┘                    └──────────────┘

       │
       ▼
┌──────────────────────────┐
│   Global CDN             │
│  ┌────┐ ┌────┐ ┌────┐   │
│  │Edge│ │Edge│ │Edge│   │
│  │ NY │ │ LA │ │Tokyo│  │
│  └────┘ └────┘ └────┘   │
└──────────────────────────┘
```

### Distribution Decisions:
```
Video Storage:
- Sharded by video_id
- Replicated across regions
- CDN for delivery

User Data:
- Sharded by user_id
- Primary-replica for reads

Comments:
- Sharded by video_id
- Eventually consistent OK

Search:
- Distributed Elasticsearch
- Near real-time indexing

View Counts:
- Approximate counting
- Eventually consistent
- Aggregated periodically
```

---

## Interview Questions

**Q: What is a distributed system?**
A: Multiple computers working together appearing as one system. Benefits: scalability, reliability, performance.

**Q: What is CAP theorem?**
A: Can only have 2 of 3: Consistency, Availability, Partition Tolerance. During network partition, choose C or A.

**Q: How do nodes communicate?**
A: Over network using messages (HTTP, gRPC, message queues). Must handle failures, delays, ordering.

**Q: How do you handle failures?**
A: Replication, retries with backoff, circuit breakers, health checks, graceful degradation.

**Q: What is consensus?**
A: Agreement among nodes on a value. Algorithms: Raft, Paxos. Essential for consistency.

---

## Quick Summary

```
DISTRIBUTED SYSTEM COMPONENTS:
- Nodes: Individual computers
- Network: Communication between nodes
- State: Shared data problem

KEY CHALLENGES:
- Network failures
- Clock synchronization
- Consistency vs Availability
- Consensus

SOLUTIONS:
- Replication: Copy data
- Partitioning: Split data
- Consensus: Agree on values
- Failure handling: Retries, circuit breakers

CONSISTENCY MODELS:
- Strong: All see same value immediately
- Eventual: All see same value... eventually
- Causal: Cause before effect

REMEMBER:
- Networks fail
- Latency exists
- Clocks drift
- Machines crash
- Design for failure!
```

You now understand distributed systems! 🌐
