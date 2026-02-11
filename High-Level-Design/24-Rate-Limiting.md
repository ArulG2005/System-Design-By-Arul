# Rate Limiting - Protecting Your System from Overload

## What is Rate Limiting?

Rate Limiting is like a **bouncer at a club** - it controls how many requests a user can make in a given time.

**Simple Definition:**
```
"You can only make 100 requests per minute. After that, wait!"
```

**Why do we need it?**
- Stop bad users from crashing your server
- Prevent abuse and spam
- Ensure fair usage for everyone
- Protect your wallet (API costs money!)

---

## Real Life Examples

### Example 1: WhatsApp Message Limit
```
WhatsApp limits how many messages you can forward.
Why? To stop spam and fake news spreading.

Without limit: One person forwards to 10,000 people in 1 minute
With limit: One person can forward to 5 people at a time
```

### Example 2: Twitter API
```
Twitter says: "You can only make 300 tweets per 3 hours"

If you try tweet #301:
Response: "Rate limit exceeded. Try again in 2 hours."
```

### Example 3: Login Attempts
```
After 5 wrong password attempts:
"Too many attempts. Try again in 15 minutes."

This stops hackers from guessing passwords!
```

---

## Rate Limiting Algorithms

### 1. Fixed Window Counter

**How it works:**
```
Divide time into fixed windows (like 1 minute each)
Count requests in each window
Reset count when new window starts

Timeline:
|-------- minute 1 --------|-------- minute 2 --------|
|  request count: 0 → 100  |  request count: 0 → 50   |
|  limit: 100              |  limit: 100              |
```

**Example:**
```python
class FixedWindowRateLimiter:
    def __init__(self, limit=100, window_seconds=60):
        self.limit = limit
        self.window_seconds = window_seconds
        self.request_count = {}
        self.window_start = {}
    
    def is_allowed(self, user_id):
        current_time = time.time()
        
        # Check if we're in a new window
        if user_id not in self.window_start:
            self.window_start[user_id] = current_time
            self.request_count[user_id] = 0
        
        # If window expired, reset
        if current_time - self.window_start[user_id] >= self.window_seconds:
            self.window_start[user_id] = current_time
            self.request_count[user_id] = 0
        
        # Check limit
        if self.request_count[user_id] < self.limit:
            self.request_count[user_id] += 1
            return True
        else:
            return False  # Rate limited!

# Usage
limiter = FixedWindowRateLimiter(limit=100, window_seconds=60)

for i in range(105):
    if limiter.is_allowed("user123"):
        print(f"Request {i+1}: Allowed")
    else:
        print(f"Request {i+1}: BLOCKED - Rate Limited!")

# Output:
# Request 1-100: Allowed
# Request 101-105: BLOCKED - Rate Limited!
```

**Problem with Fixed Window:**
```
The "Boundary Attack"

Limit: 100 requests per minute

User makes:
- 100 requests at 0:59 (end of minute 1) ✓
- 100 requests at 1:01 (start of minute 2) ✓

Result: 200 requests in 2 seconds! Not what we wanted!

Timeline:
|--------minute 1--------|--------minute 2--------|
                    100 requests + 100 requests
                         ↑
                    2 seconds gap
```

---

### 2. Sliding Window Log

**How it works:**
```
Store timestamp of each request
Look back exactly 1 minute (or your window)
Count requests in that sliding window

No fixed boundaries = No boundary attack!
```

**Example:**
```python
class SlidingWindowLog:
    def __init__(self, limit=100, window_seconds=60):
        self.limit = limit
        self.window_seconds = window_seconds
        self.request_timestamps = {}  # user_id -> list of timestamps
    
    def is_allowed(self, user_id):
        current_time = time.time()
        
        if user_id not in self.request_timestamps:
            self.request_timestamps[user_id] = []
        
        # Remove old timestamps (outside window)
        window_start = current_time - self.window_seconds
        self.request_timestamps[user_id] = [
            ts for ts in self.request_timestamps[user_id] 
            if ts > window_start
        ]
        
        # Check limit
        if len(self.request_timestamps[user_id]) < self.limit:
            self.request_timestamps[user_id].append(current_time)
            return True
        else:
            return False

# The window "slides" with current time
# Always looking at "last 60 seconds" from NOW
```

**Problem:**
```
Stores every timestamp = Uses lots of memory!
If limit is 10,000 requests/hour = 10,000 timestamps per user
1 million users = 10 billion timestamps!
```

---

### 3. Sliding Window Counter (Best of Both!)

**How it works:**
```
Combine fixed window counting with sliding window logic
Use weighted average between current and previous window

Less memory than log, more accurate than fixed window!
```

**Example:**
```python
class SlidingWindowCounter:
    def __init__(self, limit=100, window_seconds=60):
        self.limit = limit
        self.window_seconds = window_seconds
        self.current_count = {}
        self.previous_count = {}
        self.window_start = {}
    
    def is_allowed(self, user_id):
        current_time = time.time()
        current_window = int(current_time / self.window_seconds)
        
        # Initialize if new user
        if user_id not in self.window_start:
            self.window_start[user_id] = current_window
            self.current_count[user_id] = 0
            self.previous_count[user_id] = 0
        
        # Check if window changed
        if current_window != self.window_start[user_id]:
            self.previous_count[user_id] = self.current_count[user_id]
            self.current_count[user_id] = 0
            self.window_start[user_id] = current_window
        
        # Calculate position in current window (0 to 1)
        position_in_window = (current_time % self.window_seconds) / self.window_seconds
        
        # Weighted count
        # If we're 30% into current window, use 70% of previous + 100% of current
        weighted_count = (
            self.previous_count[user_id] * (1 - position_in_window) +
            self.current_count[user_id]
        )
        
        if weighted_count < self.limit:
            self.current_count[user_id] += 1
            return True
        else:
            return False

# Example:
# Previous window: 80 requests
# Current window: 30 requests  
# We're 25% into current window
# Weighted = 80 * 0.75 + 30 = 60 + 30 = 90
# If limit is 100, still can make 10 more requests
```

---

### 4. Token Bucket Algorithm

**How it works:**
```
Imagine a bucket that holds tokens
- Bucket fills up with tokens at steady rate
- Each request needs 1 token
- If bucket empty, request denied
- Bucket has max capacity (can't overflow)

Allows short bursts while maintaining average rate!
```

**Visual:**
```
Bucket Capacity: 10 tokens
Fill Rate: 1 token per second

Start: [●●●●●●●●●●] 10 tokens

User makes 5 requests quickly:
[●●●●●○○○○○] 5 tokens left

Wait 3 seconds (3 tokens refill):
[●●●●●●●●○○] 8 tokens

User makes 10 requests:
[○○○○○○○○○○] 0 tokens - last 2 requests DENIED!

Wait 10 seconds:
[●●●●●●●●●●] Full again (capped at 10)
```

**Example:**
```python
import time

class TokenBucket:
    def __init__(self, capacity=10, refill_rate=1):
        self.capacity = capacity        # Max tokens
        self.refill_rate = refill_rate  # Tokens per second
        self.tokens = capacity          # Current tokens
        self.last_refill = time.time()
    
    def _refill(self):
        now = time.time()
        time_passed = now - self.last_refill
        
        # Add tokens based on time passed
        tokens_to_add = time_passed * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now
    
    def is_allowed(self, tokens_needed=1):
        self._refill()
        
        if self.tokens >= tokens_needed:
            self.tokens -= tokens_needed
            return True
        else:
            return False

# Usage
bucket = TokenBucket(capacity=10, refill_rate=2)  # 2 tokens per second

# Burst: Can handle 10 requests immediately
for i in range(12):
    if bucket.is_allowed():
        print(f"Request {i+1}: ✓ Allowed")
    else:
        print(f"Request {i+1}: ✗ Rate Limited")

# Output:
# Request 1-10: ✓ Allowed (burst capacity)
# Request 11-12: ✗ Rate Limited (bucket empty)

time.sleep(2)  # Wait 2 seconds, 4 tokens refill

# Now can make 4 more requests
```

**Real World Use:**
```
API Rate Limits often use Token Bucket:

"100 requests per minute, burst of 20"

Bucket capacity: 20 (burst)
Refill rate: 100/60 = 1.67 tokens per second

- User can make 20 requests instantly (burst)
- Then ~1.67 requests per second sustained
- Average: 100 per minute
```

---

### 5. Leaky Bucket Algorithm

**How it works:**
```
Like a bucket with a hole at bottom
- Requests go into bucket
- Bucket "leaks" at constant rate
- If bucket full, new requests overflow (denied)

Smooths out traffic to constant rate!
```

**Visual:**
```
Bucket Capacity: 5 requests
Leak Rate: 1 request per second (processed)

Requests arrive: ●●●●●●●●●● (10 requests fast)

Bucket: [●●●●●] Full! 
        ●●●●● Overflowed (DENIED)
           ↓ (leaking at 1/sec)
        Output: ● ... ● ... ● ... ● ... ● (steady 1/sec)

Result: 5 requests accepted, 5 denied
Processed at smooth, constant rate
```

**Example:**
```python
import time
from collections import deque

class LeakyBucket:
    def __init__(self, capacity=5, leak_rate=1):
        self.capacity = capacity    # Max queue size
        self.leak_rate = leak_rate  # Requests processed per second
        self.queue = deque()
        self.last_leak = time.time()
    
    def _leak(self):
        now = time.time()
        time_passed = now - self.last_leak
        
        # Calculate how many should have leaked
        leaks = int(time_passed * self.leak_rate)
        
        # Remove leaked items
        for _ in range(min(leaks, len(self.queue))):
            self.queue.popleft()  # Process request
        
        if leaks > 0:
            self.last_leak = now
    
    def is_allowed(self, request):
        self._leak()
        
        if len(self.queue) < self.capacity:
            self.queue.append(request)
            return True
        else:
            return False  # Bucket full, overflow!

# Usage
bucket = LeakyBucket(capacity=5, leak_rate=2)

# 10 requests arrive at once
for i in range(10):
    if bucket.is_allowed(f"request_{i}"):
        print(f"Request {i}: Added to queue")
    else:
        print(f"Request {i}: OVERFLOW - Rejected")

# Output:
# Request 0-4: Added to queue
# Request 5-9: OVERFLOW - Rejected
```

**Token Bucket vs Leaky Bucket:**
```
Token Bucket:
- Allows bursts (good for user experience)
- Variable output rate
- "Save up" capacity for later

Leaky Bucket:
- No bursts allowed
- Constant output rate
- Smoother, more predictable
- Better for protecting backend
```

---

## Algorithm Comparison

```
┌────────────────────┬──────────────┬────────────┬──────────────┐
│ Algorithm          │ Memory       │ Accuracy   │ Burst        │
├────────────────────┼──────────────┼────────────┼──────────────┤
│ Fixed Window       │ Low          │ Low        │ Yes (bad)    │
│ Sliding Log        │ High         │ Perfect    │ No           │
│ Sliding Counter    │ Low          │ Good       │ No           │
│ Token Bucket       │ Low          │ Good       │ Yes (good)   │
│ Leaky Bucket       │ Medium       │ Good       │ No           │
└────────────────────┴──────────────┴────────────┴──────────────┘

Recommendation:
- Simple apps: Fixed Window
- Need accuracy: Sliding Window Counter
- Need burst + limit: Token Bucket
- Need smooth output: Leaky Bucket
```

---

## YouTube Rate Limiting Example

### Scenario: Building YouTube's Rate Limits

```
Different limits for different actions:

1. Video Uploads
   - Regular user: 15 videos per day
   - Verified creator: 100 videos per day
   
2. Comments
   - 10 comments per minute
   - 500 comments per day
   
3. API Requests
   - Free tier: 10,000 requests per day
   - Paid tier: 1,000,000 requests per day
   
4. Search
   - 100 searches per minute
   - Burst: 20 searches allowed quickly
   
5. Video Views (for ads)
   - Same video: Max 5 counted views per day per user
```

### Implementation:

```python
class YouTubeRateLimiter:
    def __init__(self):
        # Different rate limiters for different actions
        self.upload_limiter = {}      # user_id -> count
        self.comment_limiter = {}     # user_id -> TokenBucket
        self.search_limiter = {}      # user_id -> TokenBucket
        self.api_limiter = {}         # api_key -> daily count
        
    def can_upload(self, user_id, is_verified=False):
        daily_limit = 100 if is_verified else 15
        today = date.today()
        
        key = f"{user_id}:{today}"
        if key not in self.upload_limiter:
            self.upload_limiter[key] = 0
        
        if self.upload_limiter[key] < daily_limit:
            self.upload_limiter[key] += 1
            return True, f"Upload {self.upload_limiter[key]}/{daily_limit}"
        else:
            return False, "Daily upload limit reached"
    
    def can_comment(self, user_id):
        if user_id not in self.comment_limiter:
            # 10 tokens, refill 10 per minute
            self.comment_limiter[user_id] = TokenBucket(
                capacity=10, 
                refill_rate=10/60  # 10 per minute
            )
        
        return self.comment_limiter[user_id].is_allowed()
    
    def can_search(self, user_id):
        if user_id not in self.search_limiter:
            # Allow burst of 20, sustained 100/minute
            self.search_limiter[user_id] = TokenBucket(
                capacity=20,          # Burst capacity
                refill_rate=100/60    # 100 per minute
            )
        
        return self.search_limiter[user_id].is_allowed()
    
    def can_use_api(self, api_key, tier="free"):
        daily_limit = 1000000 if tier == "paid" else 10000
        today = date.today()
        
        key = f"{api_key}:{today}"
        if key not in self.api_limiter:
            self.api_limiter[key] = 0
        
        if self.api_limiter[key] < daily_limit:
            self.api_limiter[key] += 1
            remaining = daily_limit - self.api_limiter[key]
            return True, remaining
        else:
            return False, 0

# Usage
yt = YouTubeRateLimiter()

# User trying to spam comments
for i in range(15):
    if yt.can_comment("user123"):
        print(f"Comment {i+1}: Posted")
    else:
        print(f"Comment {i+1}: Rate limited!")

# Output:
# Comment 1-10: Posted
# Comment 11-15: Rate limited!
```

---

## Rate Limiting in Distributed Systems

### Problem: Multiple Servers

```
You have 5 servers behind a load balancer.
Each server has its own rate limiter.
Limit: 100 requests per minute.

User sends:
- 100 requests to Server 1 ✓
- 100 requests to Server 2 ✓
- 100 requests to Server 3 ✓

Total: 300 requests per minute! (Not 100!)
```

### Solution: Centralized Rate Limiting with Redis

```python
import redis
import time

class DistributedRateLimiter:
    def __init__(self, redis_host='localhost'):
        self.redis = redis.Redis(host=redis_host)
    
    def is_allowed_sliding_window(self, user_id, limit=100, window=60):
        """
        Sliding window counter using Redis
        Works across all servers!
        """
        now = time.time()
        window_start = now - window
        key = f"rate_limit:{user_id}"
        
        # Use Redis pipeline for atomic operations
        pipe = self.redis.pipeline()
        
        # Remove old entries
        pipe.zremrangebyscore(key, 0, window_start)
        
        # Count current entries  
        pipe.zcard(key)
        
        # Add current request with timestamp as score
        pipe.zadd(key, {str(now): now})
        
        # Set expiry on key
        pipe.expire(key, window)
        
        results = pipe.execute()
        current_count = results[1]
        
        if current_count < limit:
            return True
        else:
            # Remove the request we just added
            self.redis.zrem(key, str(now))
            return False
    
    def is_allowed_token_bucket(self, user_id, capacity=10, refill_rate=1):
        """
        Token bucket using Redis
        """
        key = f"token_bucket:{user_id}"
        now = time.time()
        
        # Lua script for atomic token bucket
        lua_script = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])
        
        local bucket = redis.call('HMGET', key, 'tokens', 'last_update')
        local tokens = tonumber(bucket[1]) or capacity
        local last_update = tonumber(bucket[2]) or now
        
        -- Refill tokens
        local time_passed = now - last_update
        tokens = math.min(capacity, tokens + time_passed * refill_rate)
        
        local allowed = 0
        if tokens >= 1 then
            tokens = tokens - 1
            allowed = 1
        end
        
        redis.call('HMSET', key, 'tokens', tokens, 'last_update', now)
        redis.call('EXPIRE', key, 60)
        
        return allowed
        """
        
        result = self.redis.eval(lua_script, 1, key, capacity, refill_rate, now)
        return result == 1

# Now all 5 servers share same Redis = same rate limit!
```

---

## Rate Limit Headers

When you hit a rate limit, API should tell you:

```
HTTP/1.1 429 Too Many Requests
Content-Type: application/json

Headers:
X-RateLimit-Limit: 100          # Your limit
X-RateLimit-Remaining: 0         # Requests left
X-RateLimit-Reset: 1609459200    # When limit resets (Unix timestamp)
Retry-After: 45                  # Seconds to wait

Body:
{
    "error": "rate_limit_exceeded",
    "message": "Too many requests. Please retry after 45 seconds.",
    "retry_after": 45
}
```

**Implementation:**
```python
from flask import Flask, request, jsonify
import time

app = Flask(__name__)
limiter = DistributedRateLimiter()

@app.route('/api/videos')
def get_videos():
    user_id = request.headers.get('X-User-ID')
    
    # Check rate limit
    allowed, remaining, reset_time = limiter.check_with_info(user_id)
    
    # Always include rate limit headers
    headers = {
        'X-RateLimit-Limit': '100',
        'X-RateLimit-Remaining': str(remaining),
        'X-RateLimit-Reset': str(reset_time)
    }
    
    if not allowed:
        retry_after = reset_time - int(time.time())
        headers['Retry-After'] = str(retry_after)
        
        return jsonify({
            'error': 'rate_limit_exceeded',
            'message': f'Too many requests. Retry after {retry_after} seconds',
            'retry_after': retry_after
        }), 429, headers
    
    # Process normal request
    videos = get_videos_from_db()
    return jsonify(videos), 200, headers
```

---

## Rate Limiting Strategies

### 1. Per User Rate Limiting
```
Each user gets their own limit
User A: 100 requests (independent of User B)
User B: 100 requests

Best for: Fair usage, preventing abuse
```

### 2. Per IP Rate Limiting
```
Limit by IP address
Same IP: All requests share limit

Problem: People behind same network (office, college)
         share IP = unfair!

Best for: Unauthenticated endpoints, DDoS protection
```

### 3. Per API Key Rate Limiting
```
Each API key gets a limit
Same user, different keys = different limits

Best for: B2B APIs, developer platforms
```

### 4. Per Endpoint Rate Limiting
```
Different limits for different endpoints

/api/search: 100/minute (expensive)
/api/profile: 1000/minute (cheap)
/api/upload: 10/hour (very expensive)

Best for: Protecting expensive operations
```

### 5. Global Rate Limiting
```
Total requests to entire system
If system can handle 1M requests/sec
Reject any request beyond that

Best for: System protection, preventing overload
```

---

## YouTube System Design with Rate Limiting

```
┌─────────────────────────────────────────────────────────────────┐
│                     YOUTUBE RATE LIMITING                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   User Request                                                   │
│        ↓                                                         │
│   ┌─────────────┐                                               │
│   │ API Gateway │ ← Global Rate Limit (DDoS Protection)         │
│   └─────────────┘                                               │
│        ↓                                                         │
│   ┌─────────────┐                                               │
│   │ Auth + ID   │ ← Identify User/API Key                       │
│   └─────────────┘                                               │
│        ↓                                                         │
│   ┌─────────────────────────────────────────────────┐           │
│   │           Rate Limit Check (Redis)               │           │
│   │   ┌─────────────┬──────────────┬─────────────┐  │           │
│   │   │ Per User    │ Per IP       │ Per Endpoint│  │           │
│   │   │ Limits      │ Limits       │ Limits      │  │           │
│   │   └─────────────┴──────────────┴─────────────┘  │           │
│   └─────────────────────────────────────────────────┘           │
│        ↓                                                         │
│   ┌─────────────┐        ┌─────────────┐                        │
│   │   Allowed   │   OR   │   Blocked   │                        │
│   │   Continue  │        │   429 Error │                        │
│   └─────────────┘        └─────────────┘                        │
│        ↓                                                         │
│   ┌─────────────────────────────────────────────────┐           │
│   │              Backend Services                    │           │
│   │   Video │ Comment │ Upload │ Search │ Stream    │           │
│   └─────────────────────────────────────────────────┘           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Common Mistakes to Avoid

### Mistake 1: Rate limiting only on one server
```
WRONG: Each server has its own counter
RIGHT: Use Redis or similar for distributed counting
```

### Mistake 2: Not returning proper headers
```
WRONG: Just return 429 error
RIGHT: Include Retry-After, X-RateLimit-Remaining headers
       So client knows when to retry
```

### Mistake 3: Same limit for all endpoints
```
WRONG: 100/min for everything
RIGHT: /search = 50/min (expensive)
       /profile = 500/min (cheap)
       /payment = 10/min (critical)
```

### Mistake 4: Blocking legitimate bursty traffic
```
WRONG: Strict limit of 1 request/second
RIGHT: Token Bucket with burst capacity
       Allow 20 quick requests, average 60/minute
```

### Mistake 5: Not having bypass for internal services
```
WRONG: Rate limit everyone including your own services
RIGHT: Whitelist internal IPs or use service tokens
```

---

## Interview Questions & Answers

**Q1: What is rate limiting?**
```
Rate limiting controls how many requests a user or client can 
make within a time period. It protects systems from abuse, 
ensures fair usage, and prevents system overload.
```

**Q2: Explain Token Bucket algorithm.**
```
Token Bucket has a bucket that holds tokens (max capacity).
Tokens refill at constant rate. Each request consumes a token.
If bucket empty, request denied. This allows controlled bursts
while maintaining average rate.
```

**Q3: How to implement rate limiting in distributed system?**
```
Use centralized storage like Redis shared by all servers.
Store counters/timestamps in Redis with atomic operations.
All servers check same Redis = consistent rate limiting.
```

**Q4: Token Bucket vs Leaky Bucket?**
```
Token Bucket: Allows bursts, variable output rate, 
              good for user experience.
              
Leaky Bucket: No bursts, constant output rate,
              good for protecting backend systems.
```

**Q5: What HTTP status code for rate limiting?**
```
429 Too Many Requests

Include headers:
- Retry-After: seconds until can retry
- X-RateLimit-Limit: max requests allowed
- X-RateLimit-Remaining: requests left
- X-RateLimit-Reset: when limit resets
```

---

## Summary Cheat Sheet

```
┌──────────────────────────────────────────────────────────────┐
│                 RATE LIMITING CHEAT SHEET                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  PURPOSE: Control request rate to prevent abuse/overload    │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  ALGORITHMS:                                                 │
│                                                              │
│  Fixed Window    - Simple, boundary problem                 │
│  Sliding Log     - Accurate, high memory                    │
│  Sliding Counter - Best balance (usually best choice)       │
│  Token Bucket    - Allows bursts (APIs)                     │
│  Leaky Bucket    - Smooth output (backends)                 │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  DISTRIBUTED: Use Redis for shared state                    │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  RESPONSE CODE: 429 Too Many Requests                       │
│  HEADERS: Retry-After, X-RateLimit-*                        │
│                                                              │
├──────────────────────────────────────────────────────────────┤
│  BEST PRACTICES:                                            │
│  ✓ Different limits for different endpoints                 │
│  ✓ Proper error messages with retry info                    │
│  ✓ Centralized rate limiting (Redis)                        │
│  ✓ Whitelist internal services                              │
│  ✓ Use Token Bucket for better UX                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## What's Next?

Now you understand Rate Limiting! Next learn:
1. **API Gateway** - Rate limiting at gateway level
2. **DDoS Protection** - Advanced rate limiting for attacks
3. **Circuit Breaker** - Protecting downstream services
