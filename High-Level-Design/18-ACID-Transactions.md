# ACID Transactions - Complete Guide

## What is ACID?

Think of ACID like a **bank transfer**:
- You send $100 from Account A to Account B
- Money MUST leave A and arrive at B together
- If anything fails, NOTHING should change
- Once done, it's permanent

**ACID = Rules for reliable database transactions**

```
A = Atomicity    → All or Nothing
C = Consistency  → Data stays valid
I = Isolation    → Transactions don't interfere
D = Durability   → Changes are permanent
```

---

## The Problem Without ACID

### Scenario: Transfer $100 from A to B
```
Without ACID:

Step 1: Subtract $100 from Account A
        Account A: $500 → $400 ✓
        
Step 2: ⚡ POWER FAILURE ⚡

Step 3: Add $100 to Account B
        (Never happens!)
        Account B: $200 → $200 ✗

Result:
Account A: $400 (lost $100)
Account B: $200 (never received)
$100 DISAPPEARED! 💀
```

### With ACID:
```
Start Transaction
├── Subtract $100 from A
├── ⚡ POWER FAILURE ⚡
└── ROLLBACK everything!

Result:
Account A: $500 (unchanged)
Account B: $200 (unchanged)
Nothing lost! ✓
```

---

## A - Atomicity

### Definition:
**All or Nothing** - Either ALL operations succeed, or NONE do.

```
Transaction = Single Unit

┌─────────────────────────────────┐
│         TRANSACTION             │
│  ┌─────┐ ┌─────┐ ┌─────┐       │
│  │ Op1 │→│ Op2 │→│ Op3 │       │
│  └─────┘ └─────┘ └─────┘       │
│                                 │
│  All succeed → COMMIT          │
│  Any fails   → ROLLBACK        │
└─────────────────────────────────┘
```

### Example:
```sql
-- YouTube: Subscribe to channel + Add to user's subscriptions

BEGIN TRANSACTION;

-- Operation 1: Increment channel subscriber count
UPDATE channels 
SET subscriber_count = subscriber_count + 1 
WHERE id = 'channel_123';

-- Operation 2: Add subscription record
INSERT INTO subscriptions (user_id, channel_id, created_at)
VALUES ('user_456', 'channel_123', NOW());

-- If both succeed
COMMIT;

-- If any fails, both rollback
-- ROLLBACK;
```

### Code Implementation:
```javascript
async function subscribeToChannel(userId, channelId) {
    const client = await pool.connect();
    
    try {
        await client.query('BEGIN');
        
        // Operation 1
        await client.query(
            'UPDATE channels SET subscriber_count = subscriber_count + 1 WHERE id = $1',
            [channelId]
        );
        
        // Operation 2
        await client.query(
            'INSERT INTO subscriptions (user_id, channel_id) VALUES ($1, $2)',
            [userId, channelId]
        );
        
        // All succeeded - commit
        await client.query('COMMIT');
        return { success: true };
        
    } catch (error) {
        // Something failed - rollback everything
        await client.query('ROLLBACK');
        throw error;
    } finally {
        client.release();
    }
}
```

---

## C - Consistency

### Definition:
**Data always moves from one valid state to another valid state.**

```
Before Transaction:
┌──────────────────────────────┐
│ Account A: $500              │  Total: $700
│ Account B: $200              │  (Valid state)
└──────────────────────────────┘

Transfer $100:
┌──────────────────────────────┐
│ Account A: $400              │  Total: $700
│ Account B: $300              │  (Still valid!)
└──────────────────────────────┘

Constraints are ALWAYS maintained!
```

### Constraints:
```sql
-- YouTube constraints example

CREATE TABLE videos (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id),  -- Must have owner
    title VARCHAR(200) NOT NULL,                    -- Must have title
    views INTEGER DEFAULT 0 CHECK (views >= 0),     -- Can't be negative
    created_at TIMESTAMP DEFAULT NOW()
);

-- Consistency = These rules are NEVER violated
```

### Example Violation Prevention:
```sql
-- Try to set negative views
UPDATE videos SET views = -100 WHERE id = 1;
-- ERROR: Check constraint violation!

-- Try to create video without user
INSERT INTO videos (title) VALUES ('My Video');
-- ERROR: Null constraint violation (user_id required)!

-- Database REFUSES to become inconsistent
```

---

## I - Isolation

### Definition:
**Concurrent transactions don't interfere with each other.**

```
Without Isolation:
Transaction 1                Transaction 2
Read balance: $100          Read balance: $100
Add $50                     Subtract $30
Write: $150                 Write: $70

Final: $70 (We lost the $50!)

With Isolation:
Transaction 1                Transaction 2
Read: $100                  (waiting...)
Add $50
Write: $150
COMMIT                      Read: $150
                            Subtract $30
                            Write: $120
                            COMMIT

Final: $120 ✓
```

### Isolation Levels:

```
┌────────────────────┬────────────┬──────────────┬───────────────┐
│ Isolation Level    │Dirty Reads │Non-Repeatable│ Phantom Reads │
├────────────────────┼────────────┼──────────────┼───────────────┤
│ READ UNCOMMITTED   │    Yes     │     Yes      │     Yes       │
│ READ COMMITTED     │    No      │     Yes      │     Yes       │
│ REPEATABLE READ    │    No      │     No       │     Yes       │
│ SERIALIZABLE       │    No      │     No       │     No        │
└────────────────────┴────────────┴──────────────┴───────────────┘

Higher isolation = More safety, Less performance
Lower isolation = Less safety, More performance
```

### Isolation Problems:

#### 1. Dirty Read
```
Transaction 1                Transaction 2
UPDATE balance = 150         
                            SELECT balance → 150
ROLLBACK (balance = 100)    
                            (Using wrong value 150!)
```

#### 2. Non-Repeatable Read
```
Transaction 1                Transaction 2
SELECT balance → 100         
                            UPDATE balance = 150
                            COMMIT
SELECT balance → 150 (Changed within same transaction!)
```

#### 3. Phantom Read
```
Transaction 1                Transaction 2
SELECT COUNT(*) → 10         
                            INSERT new row
                            COMMIT
SELECT COUNT(*) → 11 (New row appeared!)
```

### Setting Isolation Level:
```sql
-- PostgreSQL
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Your operations here
COMMIT;

-- Per session
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

---

## D - Durability

### Definition:
**Once committed, data survives crashes.**

```
Transaction:
1. Write data
2. COMMIT ← At this point...
3. ⚡ POWER FAILURE ⚡
4. (System restarts)
5. Data is STILL THERE ✓

How?
- Write-Ahead Logging (WAL)
- Data written to disk before commit acknowledged
- Recovery replays logs after crash
```

### Write-Ahead Log (WAL):
```
┌─────────────────────────────────────────────────────────┐
│                     Write-Ahead Log                      │
│  ┌─────────┬─────────┬─────────┬─────────┬─────────┐   │
│  │ Log 1   │ Log 2   │ Log 3   │ Log 4   │ Log 5   │   │
│  │ INSERT  │ UPDATE  │ INSERT  │ DELETE  │ UPDATE  │   │
│  │ user_1  │ user_1  │ video_1 │ user_2  │ video_1 │   │
│  └─────────┴─────────┴─────────┴─────────┴─────────┘   │
│                         ↓                                │
│                Written to disk FIRST                     │
│                         ↓                                │
│                Then data updated                         │
└─────────────────────────────────────────────────────────┘

Crash Recovery:
1. Read WAL from disk
2. Replay committed transactions
3. Rollback uncommitted transactions
4. Database is consistent again!
```

---

## ACID in Practice

### YouTube Transaction Examples:

#### 1. Video Upload with Processing
```sql
BEGIN TRANSACTION;

-- Insert video metadata
INSERT INTO videos (user_id, title, status)
VALUES (123, 'My Video', 'processing')
RETURNING id INTO @video_id;

-- Update user's video count
UPDATE users 
SET video_count = video_count + 1 
WHERE id = 123;

-- Insert initial processing job
INSERT INTO processing_queue (video_id, status)
VALUES (@video_id, 'pending');

COMMIT;
```

#### 2. Comment with Notification
```javascript
async function addComment(videoId, userId, content) {
    const client = await pool.connect();
    
    try {
        await client.query('BEGIN');
        
        // Insert comment
        const result = await client.query(
            `INSERT INTO comments (video_id, user_id, content)
             VALUES ($1, $2, $3) RETURNING id`,
            [videoId, userId, content]
        );
        
        // Update comment count
        await client.query(
            `UPDATE videos SET comment_count = comment_count + 1
             WHERE id = $1`,
            [videoId]
        );
        
        // Get video owner for notification
        const video = await client.query(
            'SELECT user_id FROM videos WHERE id = $1',
            [videoId]
        );
        
        // Create notification
        await client.query(
            `INSERT INTO notifications (user_id, type, content)
             VALUES ($1, 'new_comment', $2)`,
            [video.rows[0].user_id, `New comment on your video`]
        );
        
        await client.query('COMMIT');
        return result.rows[0];
        
    } catch (error) {
        await client.query('ROLLBACK');
        throw error;
    } finally {
        client.release();
    }
}
```

#### 3. Money Transfer (Premium Feature)
```javascript
async function transferCredits(fromUserId, toUserId, amount) {
    const client = await pool.connect();
    
    try {
        await client.query('BEGIN');
        
        // Lock sender row to prevent concurrent updates
        const sender = await client.query(
            'SELECT credits FROM users WHERE id = $1 FOR UPDATE',
            [fromUserId]
        );
        
        // Check sufficient balance
        if (sender.rows[0].credits < amount) {
            throw new Error('Insufficient credits');
        }
        
        // Deduct from sender
        await client.query(
            'UPDATE users SET credits = credits - $1 WHERE id = $2',
            [amount, fromUserId]
        );
        
        // Add to receiver
        await client.query(
            'UPDATE users SET credits = credits + $1 WHERE id = $2',
            [amount, toUserId]
        );
        
        // Log the transfer
        await client.query(
            `INSERT INTO credit_transfers (from_user, to_user, amount)
             VALUES ($1, $2, $3)`,
            [fromUserId, toUserId, amount]
        );
        
        await client.query('COMMIT');
        return { success: true };
        
    } catch (error) {
        await client.query('ROLLBACK');
        throw error;
    } finally {
        client.release();
    }
}
```

---

## ACID vs BASE

### BASE (NoSQL Alternative):
```
B = Basically Available
A = Soft state
E = Eventually consistent

ACID                          BASE
─────                         ────
Strong consistency            Eventual consistency
Pessimistic (lock)            Optimistic (no lock)
Complex transactions          Simple operations
Single database               Distributed systems
Lower availability            Higher availability
```

### When to Use:
```
ACID (SQL):
- Banking/payments
- Inventory management
- Order processing
- User authentication
- Anything where data MUST be accurate

BASE (NoSQL):
- Social media feeds
- View counts
- Like counts
- Analytics
- Anything where slight delay is OK
```

---

## Transaction Patterns

### 1. Savepoints
```sql
BEGIN;

INSERT INTO videos (...) VALUES (...);
SAVEPOINT after_video;

INSERT INTO tags (...) VALUES (...);
-- Oops, something wrong with tags

ROLLBACK TO after_video;
-- Video insert is kept!

INSERT INTO tags (...) VALUES (...);  -- Try again
COMMIT;
```

### 2. Optimistic Locking
```javascript
// Read with version
const video = await query(
    'SELECT id, title, version FROM videos WHERE id = $1',
    [videoId]
);

// Update only if version matches
const result = await query(
    `UPDATE videos 
     SET title = $1, version = version + 1 
     WHERE id = $2 AND version = $3`,
    [newTitle, videoId, video.version]
);

if (result.rowCount === 0) {
    throw new Error('Video was modified by another transaction');
}
```

### 3. Pessimistic Locking
```sql
BEGIN;

-- Lock the row
SELECT * FROM accounts WHERE id = 123 FOR UPDATE;

-- Now this row is locked
-- Other transactions wait

UPDATE accounts SET balance = balance - 100 WHERE id = 123;

COMMIT;
-- Lock released
```

---

## Distributed Transactions

### Problem:
```
Service A (PostgreSQL)        Service B (PostgreSQL)
        │                              │
        │    Transfer money            │
        │    A → B                     │
        │                              │
        ├─ Deduct $100 ────────────────│
        │                              ├─ Add $100
        │                              │
        ▼                              ▼
   COMMIT?                         COMMIT?
   
If one commits and other fails?
```

### Two-Phase Commit (2PC):
```
Coordinator: "Prepare to commit?"
Service A: "Ready!"
Service B: "Ready!"

Coordinator: "COMMIT!"
Service A: *commits*
Service B: *commits*

If anyone not ready → Everyone ROLLBACK
```

### Saga Pattern (Better for Microservices):
```
Step 1: Deduct $100 from A (with compensation: Add $100)
Step 2: Add $100 to B (with compensation: Deduct $100)

If Step 2 fails:
- Execute compensation for Step 1
- A gets $100 back
```

---

## Interview Questions

**Q: What is ACID?**
A: Properties ensuring reliable transactions:
- Atomicity: All or nothing
- Consistency: Data stays valid
- Isolation: Transactions don't interfere
- Durability: Commits survive crashes

**Q: Explain isolation levels.**
A: READ UNCOMMITTED (dirty reads), READ COMMITTED (no dirty reads), REPEATABLE READ (same reads), SERIALIZABLE (no phantoms). Higher = safer but slower.

**Q: How does durability work?**
A: Write-Ahead Logging (WAL). Changes logged to disk before committed. Crash recovery replays logs.

**Q: ACID vs BASE?**
A: ACID: Strong consistency, for critical data. BASE: Eventual consistency, for scalability. Use ACID for payments, BASE for view counts.

**Q: How to handle distributed transactions?**
A: 2PC for strong consistency, Saga pattern for microservices with compensation logic.

---

## Quick Summary

```
ACID PROPERTIES:
─────────────────
Atomicity   → All or nothing
Consistency → Valid state to valid state
Isolation   → Transactions don't interfere
Durability  → Committed data survives crashes

ISOLATION LEVELS:
─────────────────
READ UNCOMMITTED → Fastest, dirty reads
READ COMMITTED   → No dirty reads (default)
REPEATABLE READ  → Same data within transaction
SERIALIZABLE     → Safest, slowest

IMPLEMENTATION:
───────────────
BEGIN TRANSACTION
→ Operations
→ COMMIT or ROLLBACK

DISTRIBUTED:
────────────
2PC: Prepare → Commit
Saga: Forward → Compensate on failure

USE ACID FOR:
─────────────
✓ Money transfers
✓ Order processing
✓ Inventory updates
✓ User data changes
✓ Anything critical
```

You now understand ACID transactions! 💳
