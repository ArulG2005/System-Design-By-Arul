# Idempotency - Complete Guide

## What is Idempotency?

Think of it like an **elevator button**:
- Press once → Elevator comes
- Press 5 times → Elevator still comes once
- Same result regardless of how many times you press

**Simple Definition:**
An idempotent operation produces the same result whether executed once or multiple times.

---

## Why Idempotency Matters

### The Problem:
```
User clicks "Pay $100" button

Network Issue:
┌────────┐         ┌────────────┐         ┌────────────┐
│ Client │─────────│  Network   │─────────│   Server   │
│        │  req 1  │  (timeout) │  req 1  │            │
│        │─────────│────────────│─────────│ ✓ Charged  │
│        │         │            │         │            │
│        │  (no response received)        │            │
│        │         │            │         │            │
│        │  retry  │            │  req 2  │            │
│        │─────────│────────────│─────────│ ✓ Charged  │
└────────┘         └────────────┘         └────────────┘
                                           AGAIN! 💀

Result: User charged $200 instead of $100!
```

### With Idempotency:
```
User clicks "Pay $100" button (with idempotency key)

┌────────┐                              ┌────────────┐
│ Client │  req 1 (key: "abc123")       │   Server   │
│        │──────────────────────────────│ ✓ Charged  │
│        │  (no response)               │ Saved key  │
│        │                              │            │
│        │  retry (key: "abc123")       │            │
│        │──────────────────────────────│ Key exists!│
│        │                              │ Return OK  │
└────────┘                              └────────────┘

Result: User charged exactly $100 ✓
```

---

## Idempotent vs Non-Idempotent

### HTTP Methods:
```
IDEMPOTENT (Safe to retry):
┌─────────────────────────────────────────────────────────┐
│ GET    │ Get resource      │ Same resource returned    │
│ PUT    │ Update resource   │ Same final state          │  
│ DELETE │ Delete resource   │ Resource gone             │
│ HEAD   │ Get headers       │ Same headers              │
└─────────────────────────────────────────────────────────┘

NON-IDEMPOTENT (Dangerous to retry):
┌─────────────────────────────────────────────────────────┐
│ POST   │ Create resource   │ New resource each time!   │
│ PATCH  │ Partial update    │ May have cumulative effect│
└─────────────────────────────────────────────────────────┘
```

### Examples:
```
IDEMPOTENT:
───────────
DELETE /users/123      → User gone (retry = still gone) ✓
PUT /users/123 {name: "John"} → Name is John (retry = still John) ✓
GET /videos/abc        → Returns video (retry = same video) ✓

NON-IDEMPOTENT:
───────────────
POST /orders {item: "book"} → Creates order #1
POST /orders {item: "book"} → Creates order #2 (duplicate!)

POST /account/transfer {amount: 100}
→ First call: Balance 1000 → 900
→ Retry: Balance 900 → 800 (WRONG!)
```

---

## Implementing Idempotency

### 1. Idempotency Key (Most Common)

```
Client sends unique key with each request:

POST /payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

{
    "amount": 100,
    "currency": "USD",
    "recipient": "user_123"
}

Server:
1. Check if key exists in database
2. If exists → Return cached response
3. If not → Process and store response with key
```

### Implementation:
```javascript
// Middleware for idempotency
class IdempotencyMiddleware {
    constructor(redis) {
        this.redis = redis;
        this.ttl = 86400; // 24 hours
    }
    
    async handle(req, res, next) {
        const idempotencyKey = req.headers['idempotency-key'];
        
        // Only for non-idempotent methods
        if (!['POST', 'PATCH'].includes(req.method)) {
            return next();
        }
        
        if (!idempotencyKey) {
            return res.status(400).json({
                error: 'Idempotency-Key header required'
            });
        }
        
        const cacheKey = `idempotency:${idempotencyKey}`;
        
        // Check for existing response
        const cached = await this.redis.get(cacheKey);
        if (cached) {
            const response = JSON.parse(cached);
            return res.status(response.status).json(response.body);
        }
        
        // Store that we're processing this key (prevent race conditions)
        const locked = await this.redis.set(
            cacheKey, 
            JSON.stringify({ status: 'processing' }),
            'EX', 60,  // 60 second lock
            'NX'       // Only if not exists
        );
        
        if (!locked) {
            // Another request is processing this key
            return res.status(409).json({
                error: 'Request already in progress'
            });
        }
        
        // Capture response
        const originalJson = res.json.bind(res);
        res.json = async (body) => {
            // Store response for future duplicate requests
            await this.redis.setex(
                cacheKey,
                this.ttl,
                JSON.stringify({
                    status: res.statusCode,
                    body
                })
            );
            return originalJson(body);
        };
        
        next();
    }
}

// Usage
app.use('/api/payments', idempotencyMiddleware.handle.bind(idempotencyMiddleware));

app.post('/api/payments', async (req, res) => {
    const payment = await processPayment(req.body);
    res.json({ success: true, payment_id: payment.id });
});
```

### 2. Database-Level Idempotency

```javascript
// Using unique constraints
async function createOrder(userId, orderId, items) {
    try {
        // orderId is client-generated UUID
        await db.query(`
            INSERT INTO orders (id, user_id, items, created_at)
            VALUES ($1, $2, $3, NOW())
            ON CONFLICT (id) DO NOTHING
            RETURNING *
        `, [orderId, userId, JSON.stringify(items)]);
        
        // Get the order (whether just created or existing)
        const order = await db.query(
            'SELECT * FROM orders WHERE id = $1',
            [orderId]
        );
        
        return order.rows[0];
        
    } catch (error) {
        if (error.code === '23505') { // Unique violation
            // Return existing order
            const order = await db.query(
                'SELECT * FROM orders WHERE id = $1',
                [orderId]
            );
            return order.rows[0];
        }
        throw error;
    }
}
```

### 3. Natural Idempotency Keys

```javascript
// Use business logic for idempotency
async function likeVideo(userId, videoId) {
    // This is naturally idempotent!
    // User can only like a video once
    
    await db.query(`
        INSERT INTO likes (user_id, video_id, created_at)
        VALUES ($1, $2, NOW())
        ON CONFLICT (user_id, video_id) DO NOTHING
    `, [userId, videoId]);
    
    return { success: true };
}

// Subscription is naturally idempotent
async function subscribe(userId, channelId) {
    await db.query(`
        INSERT INTO subscriptions (user_id, channel_id)
        VALUES ($1, $2)
        ON CONFLICT (user_id, channel_id) DO NOTHING
    `, [userId, channelId]);
    
    return { success: true };
}
```

---

## Idempotency Patterns

### Pattern 1: Optimistic Locking with Version
```javascript
// Each update requires current version
async function updateVideo(videoId, updates, expectedVersion) {
    const result = await db.query(`
        UPDATE videos 
        SET title = $1, description = $2, version = version + 1
        WHERE id = $3 AND version = $4
        RETURNING *
    `, [updates.title, updates.description, videoId, expectedVersion]);
    
    if (result.rows.length === 0) {
        throw new Error('Version mismatch - video was modified');
    }
    
    return result.rows[0];
}

// Retry-safe: if same update sent twice with same version,
// first succeeds, second fails version check
```

### Pattern 2: State Machine
```javascript
// Only valid state transitions allowed
const ORDER_STATES = {
    PENDING: ['CONFIRMED', 'CANCELLED'],
    CONFIRMED: ['SHIPPED', 'CANCELLED'],
    SHIPPED: ['DELIVERED'],
    DELIVERED: [],
    CANCELLED: []
};

async function updateOrderState(orderId, newState) {
    const order = await db.query(
        'SELECT state FROM orders WHERE id = $1',
        [orderId]
    );
    
    const currentState = order.rows[0].state;
    const validTransitions = ORDER_STATES[currentState];
    
    if (!validTransitions.includes(newState)) {
        // Idempotent: if already in desired state, return success
        if (currentState === newState) {
            return { success: true, message: 'Already in this state' };
        }
        throw new Error(`Invalid transition: ${currentState} → ${newState}`);
    }
    
    await db.query(
        'UPDATE orders SET state = $1 WHERE id = $2 AND state = $3',
        [newState, orderId, currentState]
    );
    
    return { success: true };
}
```

### Pattern 3: Transactional Outbox
```javascript
// Ensure message is sent exactly once
async function transferMoney(fromAccount, toAccount, amount, transferId) {
    await db.transaction(async (tx) => {
        // Check if already processed
        const existing = await tx.query(
            'SELECT * FROM transfers WHERE id = $1',
            [transferId]
        );
        
        if (existing.rows.length > 0) {
            return existing.rows[0]; // Already done
        }
        
        // Deduct from source
        await tx.query(
            'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
            [amount, fromAccount]
        );
        
        // Add to destination
        await tx.query(
            'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
            [amount, toAccount]
        );
        
        // Record transfer
        await tx.query(
            'INSERT INTO transfers (id, from_account, to_account, amount) VALUES ($1, $2, $3, $4)',
            [transferId, fromAccount, toAccount, amount]
        );
        
        // Add to outbox for notification
        await tx.query(
            'INSERT INTO outbox (id, type, payload) VALUES ($1, $2, $3)',
            [transferId, 'transfer.completed', JSON.stringify({ fromAccount, toAccount, amount })]
        );
    });
}
```

---

## API Design for Idempotency

### Request Headers:
```http
POST /api/v1/payments HTTP/1.1
Host: api.youtube.com
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
X-Request-ID: req_abc123

{
    "amount": 999,
    "currency": "USD",
    "payment_method": "card_xyz"
}
```

### Response Headers:
```http
HTTP/1.1 200 OK
Content-Type: application/json
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
X-Request-ID: req_abc123
X-Idempotent-Replayed: true

{
    "payment_id": "pay_123",
    "status": "completed"
}
```

### Stripe's Approach:
```javascript
const stripe = require('stripe')('sk_test_...');

// Same idempotency key = same result
const payment = await stripe.paymentIntents.create({
    amount: 2000,
    currency: 'usd',
}, {
    idempotencyKey: 'order_123_payment'
});

// Retry with same key returns cached result
const retryPayment = await stripe.paymentIntents.create({
    amount: 2000,
    currency: 'usd',
}, {
    idempotencyKey: 'order_123_payment'
});

// payment.id === retryPayment.id (same payment!)
```

---

## YouTube Idempotency Examples

```
┌─────────────────────────────────────────────────────────────────┐
│                YouTube Idempotency Use Cases                     │
└─────────────────────────────────────────────────────────────────┘

1. Video Upload
   ┌────────────────────────────────────────────────────────────┐
   │ User uploads video, network fails after server receives   │
   │                                                           │
   │ Without idempotency:                                      │
   │ User retries → Two copies of same video! 😱               │
   │                                                           │
   │ With idempotency:                                         │
   │ Upload includes unique upload_id                          │
   │ Retry → Server recognizes, returns existing video         │
   └────────────────────────────────────────────────────────────┘

2. Subscribe/Unsubscribe
   ┌────────────────────────────────────────────────────────────┐
   │ POST /subscribe (userId, channelId)                       │
   │                                                           │
   │ Naturally idempotent:                                     │
   │ Unique constraint on (user_id, channel_id)                │
   │ Multiple clicks → Still just one subscription             │
   └────────────────────────────────────────────────────────────┘

3. Like Video
   ┌────────────────────────────────────────────────────────────┐
   │ POST /videos/abc/like                                     │
   │                                                           │
   │ Naturally idempotent:                                     │
   │ User can only like once                                   │
   │ Clicking like 100 times = 1 like                          │
   └────────────────────────────────────────────────────────────┘

4. Premium Purchase
   ┌────────────────────────────────────────────────────────────┐
   │ POST /premium/subscribe                                   │
   │ Idempotency-Key: purchase_user123_2024jan                 │
   │                                                           │
   │ Critical for payments!                                    │
   │ Retry → Return existing subscription, don't charge again  │
   └────────────────────────────────────────────────────────────┘
```

### Implementation:
```javascript
// Video Upload with Idempotency
class VideoUploadService {
    async uploadVideo(userId, uploadId, file) {
        // Check if upload already exists
        const existing = await db.query(
            'SELECT * FROM video_uploads WHERE upload_id = $1',
            [uploadId]
        );
        
        if (existing.rows.length > 0) {
            // Return existing video
            return {
                video_id: existing.rows[0].video_id,
                status: existing.rows[0].status,
                message: 'Upload already processed'
            };
        }
        
        // Create upload record first (claim the idempotency key)
        const video = await db.transaction(async (tx) => {
            // Insert with conflict handling
            await tx.query(`
                INSERT INTO video_uploads (upload_id, user_id, status)
                VALUES ($1, $2, 'processing')
                ON CONFLICT (upload_id) DO NOTHING
            `, [uploadId, userId]);
            
            // Create video record
            const result = await tx.query(`
                INSERT INTO videos (user_id, upload_id, status)
                VALUES ($1, $2, 'processing')
                RETURNING id
            `, [userId, uploadId]);
            
            return result.rows[0];
        });
        
        // Process in background
        await this.queueVideoProcessing(video.id, file);
        
        return {
            video_id: video.id,
            status: 'processing',
            message: 'Upload started'
        };
    }
}

// Subscribe (Naturally Idempotent)
class SubscriptionService {
    async subscribe(userId, channelId) {
        await db.query(`
            INSERT INTO subscriptions (user_id, channel_id, created_at)
            VALUES ($1, $2, NOW())
            ON CONFLICT (user_id, channel_id) 
            DO UPDATE SET updated_at = NOW()
        `, [userId, channelId]);
        
        // Update subscriber count idempotently
        await db.query(`
            UPDATE channels 
            SET subscriber_count = (
                SELECT COUNT(*) FROM subscriptions WHERE channel_id = $1
            )
            WHERE id = $1
        `, [channelId]);
        
        return { subscribed: true };
    }
}
```

---

## Best Practices

### 1. Key Generation
```javascript
// Client generates idempotency key
function generateIdempotencyKey(action, params) {
    // Deterministic key based on action and parameters
    const data = `${action}:${JSON.stringify(params)}`;
    return crypto.createHash('sha256').update(data).digest('hex');
}

// For payments: Use order ID
const key = `payment:order_${orderId}`;

// For uploads: Use file hash
const key = `upload:${userId}:${fileHash}`;

// General: UUID per user action
const key = crypto.randomUUID();
```

### 2. TTL for Keys
```javascript
// Keys should expire after reasonable time
const IDEMPOTENCY_TTL = 24 * 60 * 60; // 24 hours

// Allow retry within 24 hours
// After that, assume new request
```

### 3. Error Handling
```javascript
async function processWithIdempotency(key, processor) {
    const existing = await redis.get(`idempotency:${key}`);
    
    if (existing) {
        const cached = JSON.parse(existing);
        
        if (cached.status === 'processing') {
            // Still processing, return conflict
            throw new ConflictError('Request in progress');
        }
        
        if (cached.status === 'error') {
            // Previous attempt failed, allow retry
            // (Or return the error, depending on use case)
        }
        
        return cached.result;
    }
    
    try {
        // Mark as processing
        await redis.setex(`idempotency:${key}`, 60, 
            JSON.stringify({ status: 'processing' }));
        
        // Execute
        const result = await processor();
        
        // Store success
        await redis.setex(`idempotency:${key}`, 86400,
            JSON.stringify({ status: 'success', result }));
        
        return result;
        
    } catch (error) {
        // Store error (or delete key for retry)
        await redis.setex(`idempotency:${key}`, 86400,
            JSON.stringify({ status: 'error', error: error.message }));
        
        throw error;
    }
}
```

---

## Interview Questions

**Q: What is idempotency?**
A: An operation is idempotent if executing it multiple times has the same effect as executing once. Critical for handling retries safely.

**Q: Which HTTP methods are idempotent?**
A: GET, PUT, DELETE, HEAD are idempotent. POST and PATCH are not (by default).

**Q: How do you implement idempotency for payments?**
A: Use idempotency keys - client sends unique key, server stores result. Retry with same key returns cached response without reprocessing.

**Q: What happens to the idempotency key after success?**
A: Store for TTL (24-48 hours typically). After TTL expires, same key creates new request. During TTL, returns cached response.

**Q: Natural idempotency vs explicit keys?**
A: Natural: Use unique constraints (like user_id + video_id for likes). Explicit: Client-provided keys for operations without natural uniqueness.

---

## Quick Summary

```
IDEMPOTENCY:
────────────
Same operation multiple times = Same result
Essential for handling retries safely

IDEMPOTENT OPERATIONS:
──────────────────────
GET, PUT, DELETE, HEAD → Safe to retry
POST, PATCH → Need explicit handling

IMPLEMENTATION METHODS:
───────────────────────
1. Idempotency Keys: Client-sent unique ID
2. Database Constraints: ON CONFLICT DO NOTHING
3. Version Checking: Optimistic locking
4. State Machines: Only valid transitions

IDEMPOTENCY KEY FLOW:
─────────────────────
1. Client sends request + unique key
2. Server checks if key exists
3. Exists → Return cached response
4. Not exists → Process, store result with key
5. Retry → Returns same cached response

BEST PRACTICES:
───────────────
- Keys expire after 24-48 hours
- Handle "processing" state
- Use deterministic keys when possible
- Natural idempotency > explicit keys
```

You now understand idempotency! 🔄
