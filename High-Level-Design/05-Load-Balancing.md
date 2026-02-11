# Load Balancing - Complete Guide

## What is Load Balancing?

Think of Load Balancer like a **traffic police officer at a busy intersection**:
- Many cars (requests) coming
- Multiple roads (servers) available
- Officer (load balancer) directs traffic evenly
- No single road gets jammed

**Simple Definition:**
Load Balancer distributes incoming traffic across multiple servers so no single server gets overwhelmed.

---

## Why Do We Need Load Balancing?

### Without Load Balancing:
```
1 Million Users ────> 1 Server
                         ↓
                    Server crashes! 💥
                    
Problem:
- Server overloaded
- Slow response
- Single point of failure
```

### With Load Balancing:
```
                    ┌─> Server 1 (handles 333K users)
                    │
1 Million Users ───>Load Balancer ──> Server 2 (handles 333K users)
                    │
                    └─> Server 3 (handles 333K users)
                    
Benefits:
- Load distributed evenly
- Fast response
- If one server dies, others continue
```

---

## Visual Architecture

```
                         ┌───────────────┐
                         │  INTERNET     │
                         └───────┬───────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │    LOAD BALANCER       │
                    │    (Single Entry)       │
                    └─────────┬──────────────┘
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
           ▼                  ▼                  ▼
    ┌────────────┐     ┌────────────┐     ┌────────────┐
    │  Server 1  │     │  Server 2  │     │  Server 3  │
    │  (Active)  │     │  (Active)  │     │  (Active)  │
    └────────────┘     └────────────┘     └────────────┘
           │                  │                  │
           └──────────────────┴──────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │    DATABASE     │
                    └─────────────────┘
```

---

## Load Balancing Algorithms

### 1. Round Robin (Simple & Common)
```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1 (back to first)
Request 5 → Server 2
...

Like dealing cards in a circle!
```

**When to use:** All servers are equally powerful

### 2. Weighted Round Robin
```
Server 1 (powerful) → weight: 5
Server 2 (medium)   → weight: 3
Server 3 (weak)     → weight: 2

Every 10 requests:
- Server 1 gets 5
- Server 2 gets 3
- Server 3 gets 2
```

**When to use:** Servers have different capacities

### 3. Least Connections
```
Server 1: 10 active connections
Server 2: 5 active connections  ← Next request goes here!
Server 3: 8 active connections
```

**When to use:** Requests have varying processing times

### 4. Least Response Time
```
Server 1: avg 100ms response
Server 2: avg 50ms response  ← Next request goes here!
Server 3: avg 80ms response
```

**When to use:** Need fastest response

### 5. IP Hash (Sticky Sessions)
```
User IP: 192.168.1.100
Hash(192.168.1.100) → Server 2

Same user always goes to same server!
```

**When to use:** Need session consistency (shopping cart, etc.)

### 6. Random
```
Randomly pick a server for each request

Simple but effective for uniform load
```

---

## Types of Load Balancers

### Layer 4 (Transport Layer) - TCP/UDP
```
Works at: Network level
Looks at: IP address, Port number
Speed: Very fast (hardware level)
Intelligence: Low (no content awareness)

Example:
Client:192.168.1.1:5000 → Server:10.0.0.1:80

Just routes based on IP/Port, doesn't look at HTTP content
```

### Layer 7 (Application Layer) - HTTP/HTTPS
```
Works at: Application level
Looks at: URL, Headers, Cookies, Content
Speed: Slower (more processing)
Intelligence: High (content-aware routing)

Example:
/api/users → User Service
/api/videos → Video Service
/images/* → Static Server
Authorization: Premium → Premium Servers
```

### Comparison:
```
Feature              Layer 4         Layer 7
─────────────────────────────────────────────
Speed                Faster          Slower
Intelligence         Basic           Smart
SSL Termination      No              Yes
Content Routing      No              Yes
Caching              No              Yes
Compression          No              Yes
Cost                 Cheaper         More Expensive
```

---

## Health Checks

Load balancer must know if servers are alive!

### How Health Checks Work:
```
Load Balancer pings each server every 10 seconds:

Server 1: GET /health → 200 OK ✓ (healthy)
Server 2: GET /health → 200 OK ✓ (healthy)
Server 3: GET /health → Timeout ✗ (unhealthy)

Server 3 removed from rotation!

After Server 3 recovers:
Server 3: GET /health → 200 OK ✓
Server 3 added back to rotation!
```

### Health Check Implementation:
```javascript
// Simple health endpoint
app.get('/health', (req, res) => {
    res.status(200).json({
        status: 'healthy',
        uptime: process.uptime(),
        timestamp: new Date()
    });
});

// Advanced health check
app.get('/health', async (req, res) => {
    try {
        // Check database connection
        await db.query('SELECT 1');
        
        // Check Redis connection
        await redis.ping();
        
        // Check disk space
        const disk = await checkDiskSpace();
        
        // Check memory
        const memory = process.memoryUsage();
        
        res.status(200).json({
            status: 'healthy',
            checks: {
                database: 'connected',
                redis: 'connected',
                disk: `${disk.freePercent}% free`,
                memory: `${memory.heapUsed / 1024 / 1024}MB used`
            }
        });
    } catch (error) {
        res.status(503).json({
            status: 'unhealthy',
            error: error.message
        });
    }
});
```

---

## Session Persistence (Sticky Sessions)

### Problem:
```
Request 1: User logs in → Server 1 (session stored)
Request 2: User clicks profile → Server 2 (no session!)
           "Please log in again" 😫
```

### Solutions:

#### 1. Sticky Sessions (Load Balancer)
```
User A: Always → Server 1
User B: Always → Server 2
User C: Always → Server 3

How? IP Hash or Cookie-based routing
```

#### 2. Shared Session Store (Better!)
```
                    ┌─> Server 1 ─┐
User ──> Load Balancer ── Server 2 ─├──> Redis (sessions)
                    └─> Server 3 ─┘

All servers read/write sessions to Redis
No sticky sessions needed!
```

```javascript
// Using Redis for sessions
const session = require('express-session');
const RedisStore = require('connect-redis')(session);

app.use(session({
    store: new RedisStore({ client: redisClient }),
    secret: 'session-secret',
    resave: false,
    saveUninitialized: false
}));
```

---

## SSL Termination

### What is it?
```
HTTPS encryption is handled by load balancer, not individual servers

Client ──HTTPS──> Load Balancer ──HTTP──> Servers
                       ↓
            Decrypts SSL here
            Servers don't need SSL certificates
```

### Benefits:
- Servers use less CPU (no encryption work)
- Easier certificate management (one place)
- Can inspect and route based on HTTPS content

---

## Popular Load Balancers

### 1. NGINX
```nginx
# nginx.conf
http {
    upstream backend {
        least_conn;  # Algorithm
        
        server 10.0.0.1:3000 weight=5;
        server 10.0.0.2:3000 weight=3;
        server 10.0.0.3:3000 weight=2;
        
        # Health check
        keepalive 32;
    }
    
    server {
        listen 80;
        
        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /health {
            return 200 'OK';
        }
    }
}
```

### 2. HAProxy
```
# haproxy.cfg
frontend http_front
    bind *:80
    default_backend servers

backend servers
    balance roundrobin
    option httpchk GET /health
    
    server server1 10.0.0.1:3000 check weight 5
    server server2 10.0.0.2:3000 check weight 3
    server server3 10.0.0.3:3000 check weight 2
```

### 3. AWS ELB (Elastic Load Balancer)
```yaml
# CloudFormation
Resources:
  LoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Type: application
      Subnets:
        - subnet-1
        - subnet-2
      SecurityGroups:
        - sg-1

  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Protocol: HTTP
      Port: 80
      HealthCheckPath: /health
      HealthCheckIntervalSeconds: 30
```

### 4. Kubernetes (Built-in)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 3000
```

---

## Load Balancing for YouTube Clone

### Architecture:
```
                           ┌─────────────────┐
                           │   DNS Server    │
                           │ youtube.com     │
                           └────────┬────────┘
                                    │
                           ┌────────▼────────┐
                           │  CDN / Edge     │
                           │ (Static files)  │
                           └────────┬────────┘
                                    │
                    ┌───────────────▼───────────────┐
                    │    Global Load Balancer       │
                    │    (Routes to nearest DC)      │
                    └───────────────┬───────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│   US East     │           │   EU West     │           │   Asia        │
│   Region      │           │   Region      │           │   Region      │
└───────┬───────┘           └───────┬───────┘           └───────┬───────┘
        │                           │                           │
        ▼                           ▼                           ▼
┌───────────────┐           ┌───────────────┐           ┌───────────────┐
│ Regional LB   │           │ Regional LB   │           │ Regional LB   │
└───────┬───────┘           └───────┬───────┘           └───────┬───────┘
        │                           │                           │
   ┌────┴────┐                 ┌────┴────┐                 ┌────┴────┐
   │         │                 │         │                 │         │
   ▼         ▼                 ▼         ▼                 ▼         ▼
┌─────┐   ┌─────┐           ┌─────┐   ┌─────┐           ┌─────┐   ┌─────┐
│ S1  │   │ S2  │           │ S1  │   │ S2  │           │ S1  │   │ S2  │
└─────┘   └─────┘           └─────┘   └─────┘           └─────┘   └─────┘
```

### Layer 7 Routing Rules:
```nginx
# NGINX config for YouTube-like routing

upstream video_servers {
    least_conn;
    server video-1:3000;
    server video-2:3000;
    server video-3:3000;
}

upstream user_servers {
    least_conn;
    server user-1:3000;
    server user-2:3000;
}

upstream upload_servers {
    least_conn;
    server upload-1:3000;  # Dedicated upload servers
    server upload-2:3000;
}

server {
    listen 80;
    
    # Video streaming
    location /watch {
        proxy_pass http://video_servers;
    }
    
    # Video uploads (bigger servers)
    location /upload {
        proxy_pass http://upload_servers;
        client_max_body_size 10G;  # Allow big files
    }
    
    # User operations
    location /api/users {
        proxy_pass http://user_servers;
    }
    
    # Static content from CDN
    location /static {
        proxy_pass http://cdn.youtube-clone.com;
    }
}
```

---

## High Availability Setup

### Single Load Balancer = Single Point of Failure!

### Solution: Multiple Load Balancers
```
                    ┌─────────────────┐
                    │   Virtual IP    │
                    │  (Floating IP)  │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
     ┌─────────────────┐           ┌─────────────────┐
     │  Load Balancer  │           │  Load Balancer  │
     │    (Active)     │◀─────────▶│   (Standby)     │
     │                 │ heartbeat │                 │
     └────────┬────────┘           └────────┬────────┘
              │                             │
              └──────────────┬──────────────┘
                             │
              ┌──────────────┴──────────────┐
              │              │              │
              ▼              ▼              ▼
          Server 1       Server 2       Server 3

If Active LB dies:
- Standby detects (no heartbeat)
- Standby takes over Virtual IP
- Users don't notice anything!
```

---

## Auto Scaling with Load Balancer

```
Normal load: 3 servers
                    
Request spike detected!
Load Balancer: "Servers at 80% capacity!"
                    ↓
Auto-scaling triggers:
- Spin up Server 4
- Spin up Server 5
- Add to load balancer pool
                    
Traffic decreases...
- Remove Server 5
- Remove Server 4
- Back to 3 servers
```

### AWS Auto Scaling:
```yaml
AutoScalingGroup:
  Type: AWS::AutoScaling::AutoScalingGroup
  Properties:
    MinSize: 2
    MaxSize: 10
    DesiredCapacity: 3
    TargetGroupARNs:
      - !Ref TargetGroup
    
ScalingPolicy:
  Type: AWS::AutoScaling::ScalingPolicy
  Properties:
    PolicyType: TargetTrackingScaling
    TargetTrackingConfiguration:
      TargetValue: 70.0
      PredefinedMetricSpecification:
        PredefinedMetricType: ASGAverageCPUUtilization
```

---

## Load Balancing Best Practices

### 1. Always Use Health Checks
```nginx
upstream backend {
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
    
    # Check every 5 seconds
    # 3 failures = mark unhealthy
    # 2 successes = mark healthy again
}
```

### 2. Enable Connection Draining
```
When removing a server:
- Don't cut existing connections immediately
- Let ongoing requests complete
- Stop sending NEW requests
- Wait (30 seconds typical)
- Then remove server
```

### 3. Use Proper Timeouts
```nginx
proxy_connect_timeout 5s;   # Time to establish connection
proxy_send_timeout 10s;     # Time to send request
proxy_read_timeout 60s;     # Time to wait for response
```

### 4. Log Everything
```nginx
log_format detailed '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'upstream: $upstream_addr '
                    'response_time: $upstream_response_time';
```

---

## Interview Questions

**Q: What's the difference between L4 and L7 load balancing?**
A: L4 routes based on IP/Port (faster, dumber). L7 routes based on content like URL, headers, cookies (slower, smarter).

**Q: How to handle session with multiple servers?**
A: Use sticky sessions OR store sessions in shared storage like Redis (preferred).

**Q: What if load balancer itself fails?**
A: Use multiple load balancers with floating IP (Active-Passive or Active-Active).

**Q: What algorithm would you use for a video streaming service?**
A: Least Connections - because streaming connections are long-lived and we want to balance active connections.

---

## Quick Summary

```
Load Balancer = Traffic distributor

Algorithms:
- Round Robin: Take turns
- Weighted: Based on capacity
- Least Connections: Least busy first
- IP Hash: Same user → Same server

Types:
- L4: Fast, based on IP/Port
- L7: Smart, based on content

Must Have:
- Health checks
- SSL termination
- High availability (multiple LBs)
- Auto-scaling integration
```

You now understand Load Balancing like a pro! 🚀
