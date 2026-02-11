# API Gateway - Complete Guide

## What is an API Gateway?

Think of API Gateway like a **security guard + receptionist at a big building**:
- Checks who you are (Authentication)
- Directs you to the right office (Routing)
- Makes sure you don't come too often (Rate Limiting)
- Keeps record of who came (Logging)

**Simple Definition:**
API Gateway is a single entry point for all your APIs. All requests go through it first.

---

## Without vs With API Gateway

### Without API Gateway (Messy!):
```
Mobile App ──────> User Service
           ──────> Video Service
           ──────> Payment Service
           ──────> Comment Service

Web App ──────> User Service
        ──────> Video Service
        ──────> Payment Service
        ──────> Comment Service

Each client must:
- Know all service addresses
- Handle authentication separately
- Manage rate limiting itself
```

### With API Gateway (Clean!):
```
Mobile App ─────┐
                ├───> API Gateway ───> User Service
Web App ────────┤                 ───> Video Service
                │                 ───> Payment Service
Smart TV App ───┘                 ───> Comment Service

API Gateway handles:
- Single entry point
- Authentication
- Rate limiting
- Request routing
- Load balancing
```

---

## Core Functions of API Gateway

### 1. Request Routing
```
Client Request: GET /users/123

API Gateway thinks:
"Hmm, /users/* requests should go to User Service"

Routes to: http://user-service:3000/users/123
```

### 2. Authentication & Authorization
```
Request comes with token:
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

API Gateway:
1. Validates the token
2. Extracts user info
3. Checks permissions
4. If valid → forwards request
5. If invalid → returns 401 Unauthorized
```

### 3. Rate Limiting
```
User making requests:
Request 1 ✓
Request 2 ✓
Request 3 ✓
...
Request 101 ✗ (429 Too Many Requests)

Rules example:
- Free tier: 100 requests/hour
- Pro tier: 10,000 requests/hour
```

### 4. Load Balancing
```
3 Video Service instances running:
- video-service-1
- video-service-2
- video-service-3

API Gateway distributes:
Request 1 → video-service-1
Request 2 → video-service-2
Request 3 → video-service-3
Request 4 → video-service-1 (round-robin)
```

### 5. Request/Response Transformation
```
Client sends:
{
    "userName": "john"
}

Backend needs:
{
    "user_name": "john"
}

API Gateway transforms camelCase → snake_case
```

### 6. Caching
```
Request: GET /trending-videos

First time:
Client → Gateway → Video Service → Database
                              ↓
                    Gateway caches response

Second time (within cache time):
Client → Gateway (returns cached data)
         ↓
    No backend call! Super fast!
```

---

## API Gateway Patterns

### Pattern 1: Single API Gateway
```
                    ┌─────────────────┐
All Clients ────────> │  API Gateway    │ ────> All Services
                    └─────────────────┘

Good for: Small to medium applications
```

### Pattern 2: Multiple API Gateways (BFF - Backend for Frontend)
```
Mobile App ────> Mobile Gateway ────┐
                                   ├──> Services
Web App ────────> Web Gateway ─────┤
                                   │
TV App ─────────> TV Gateway ──────┘

Why? Each platform has different needs:
- Mobile: Less data (save bandwidth)
- Web: More data (bigger screens)
- TV: Special formatting
```

### Pattern 3: Gateway with Service Mesh
```
                                    ┌─────────────┐
                                    │ Service A   │
API Gateway ───> Service Mesh ─────>│ Service B   │
                (handles internal   │ Service C   │
                 communication)     └─────────────┘
```

---

## Real-World Example: YouTube's API Gateway

```
┌────────────────────────────────────────────────────────────┐
│                      API GATEWAY                           │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  1. Request comes: GET /api/v1/videos/abc123               │
│                                                            │
│  2. Authentication:                                        │
│     - Extract JWT token                                    │
│     - Validate token                                       │
│     - Get user_id: 456                                     │
│                                                            │
│  3. Rate Limit Check:                                      │
│     - User 456 has made 50/100 requests this hour         │
│     - ✓ Allowed                                           │
│                                                            │
│  4. Route Decision:                                        │
│     - /videos/* → Video Service                           │
│                                                            │
│  5. Load Balance:                                          │
│     - video-service-2 has lowest load                      │
│     - Send to video-service-2                             │
│                                                            │
│  6. Forward Request:                                       │
│     - Add headers: X-User-ID: 456                         │
│     - Forward to http://video-service-2/videos/abc123     │
│                                                            │
│  7. Receive Response, send to client                      │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## API Gateway Features Deep Dive

### SSL Termination
```
Client ───HTTPS──> API Gateway ───HTTP──> Internal Services
                        ↓
            Handles all encryption/decryption
            Internal network is secure
            Services don't need SSL certificates
```

### Request Aggregation (Composition)
```
Client wants: User profile + Recent videos + Subscribers count

Without Gateway:
Client makes 3 API calls

With Gateway:
Client makes 1 call → Gateway makes 3 internal calls → Combines response

Single Response:
{
    "user": { "name": "John", "bio": "..." },
    "recent_videos": [...],
    "subscriber_count": 50000
}
```

### Circuit Breaker
```
Video Service is failing...

Request 1 → Timeout (5 seconds wasted)
Request 2 → Timeout (5 seconds wasted)
Request 3 → Timeout (5 seconds wasted)
Request 4 → Timeout (5 seconds wasted)
Request 5 → Timeout (5 seconds wasted)

Circuit Breaker activates:
"Video Service is down, I'll stop sending requests"

Request 6 → Instant error (saved 5 seconds!)
Request 7 → Instant error

After 30 seconds, try again...
If service is back → Resume normal operation
```

---

## Popular API Gateways

### 1. Kong
```yaml
# Kong configuration example
services:
  - name: video-service
    url: http://video-service:3000
    routes:
      - name: video-route
        paths:
          - /videos
    plugins:
      - name: rate-limiting
        config:
          minute: 100
      - name: jwt
```

### 2. AWS API Gateway
```yaml
# AWS SAM template
Resources:
  MyApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Auth:
        DefaultAuthorizer: MyCognitoAuthorizer
```

### 3. NGINX
```nginx
# NGINX as API Gateway
upstream video_service {
    server video-service-1:3000;
    server video-service-2:3000;
    server video-service-3:3000;
}

server {
    listen 80;
    
    location /api/videos {
        proxy_pass http://video_service;
        
        # Rate limiting
        limit_req zone=api_limit burst=20;
        
        # Add headers
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 4. Express Gateway (Node.js)
```javascript
// gateway.config.yml
http:
  port: 8080
apiEndpoints:
  videos:
    path: '/videos/*'
serviceEndpoints:
  videoService:
    url: 'http://localhost:3001'
pipelines:
  videoPipeline:
    apiEndpoints:
      - videos
    policies:
      - jwt:
      - rate-limit:
          - action:
              max: 100
              windowMs: 60000
      - proxy:
          - action:
              serviceEndpoint: videoService
```

---

## Building API Gateway for YouTube Clone

### Architecture:
```
                         ┌──────────────────┐
                         │   API Gateway    │
                         │   (Port 8080)    │
                         └────────┬─────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
        ▼                         ▼                         ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│ User Service  │       │ Video Service │       │Comment Service│
│  (Port 3001)  │       │  (Port 3002)  │       │  (Port 3003)  │
└───────────────┘       └───────────────┘       └───────────────┘
```

### Simple Implementation (Node.js):
```javascript
const express = require('express');
const httpProxy = require('http-proxy');
const rateLimit = require('express-rate-limit');
const jwt = require('jsonwebtoken');

const app = express();
const proxy = httpProxy.createProxyServer();

// Service URLs
const services = {
    users: 'http://localhost:3001',
    videos: 'http://localhost:3002',
    comments: 'http://localhost:3003'
};

// Rate Limiting
const limiter = rateLimit({
    windowMs: 60 * 1000, // 1 minute
    max: 100, // 100 requests per minute
    message: { error: 'Too many requests, slow down!' }
});
app.use(limiter);

// JWT Authentication Middleware
const authenticate = (req, res, next) => {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
        return res.status(401).json({ error: 'No token provided' });
    }
    
    try {
        const decoded = jwt.verify(token, 'secret-key');
        req.user = decoded;
        next();
    } catch (error) {
        res.status(401).json({ error: 'Invalid token' });
    }
};

// Routing
app.use('/api/users', authenticate, (req, res) => {
    proxy.web(req, res, { target: services.users });
});

app.use('/api/videos', authenticate, (req, res) => {
    proxy.web(req, res, { target: services.videos });
});

app.use('/api/comments', authenticate, (req, res) => {
    proxy.web(req, res, { target: services.comments });
});

// Health check (no auth needed)
app.get('/health', (req, res) => {
    res.json({ status: 'API Gateway is running' });
});

// Error handling
proxy.on('error', (err, req, res) => {
    res.status(503).json({ error: 'Service unavailable' });
});

app.listen(8080, () => {
    console.log('API Gateway running on port 8080');
});
```

---

## API Gateway Best Practices

### 1. Keep Gateway Lightweight
```
DO:
- Authentication
- Rate limiting
- Routing
- Logging

DON'T:
- Complex business logic
- Heavy data processing
- Database operations
```

### 2. Implement Timeouts
```javascript
const timeout = (ms) => {
    return (req, res, next) => {
        res.setTimeout(ms, () => {
            res.status(504).json({ error: 'Gateway timeout' });
        });
        next();
    };
};

app.use(timeout(30000)); // 30 second timeout
```

### 3. Add Request ID for Tracing
```javascript
const { v4: uuid } = require('uuid');

app.use((req, res, next) => {
    req.requestId = uuid();
    res.setHeader('X-Request-ID', req.requestId);
    next();
});
```

### 4. Implement Health Checks
```javascript
app.get('/health', async (req, res) => {
    const health = {
        gateway: 'healthy',
        services: {}
    };
    
    for (const [name, url] of Object.entries(services)) {
        try {
            await fetch(`${url}/health`);
            health.services[name] = 'healthy';
        } catch {
            health.services[name] = 'unhealthy';
        }
    }
    
    res.json(health);
});
```

---

## Common Interview Questions

**Q: Why use API Gateway?**
A: Single entry point, centralized authentication, rate limiting, load balancing, logging, and easier client integration.

**Q: API Gateway vs Load Balancer?**
A: Load Balancer only distributes traffic. API Gateway does that PLUS authentication, rate limiting, routing, transformation, etc.

**Q: What is BFF pattern?**
A: Backend For Frontend - Different API Gateways for different clients (mobile, web, TV) to serve optimized responses.

**Q: How to handle Gateway failure?**
A: Run multiple gateway instances behind a load balancer. Use health checks and auto-scaling.

---

## Quick Summary

```
API Gateway = Security Guard + Receptionist + Traffic Controller

Main Jobs:
1. Route requests to correct service
2. Authenticate users
3. Limit request rate
4. Balance load
5. Transform requests/responses
6. Cache responses
7. Log everything

Popular Tools:
- Kong
- AWS API Gateway
- NGINX
- Traefik
- Express Gateway
```

You now understand API Gateway like a pro! 🚀
