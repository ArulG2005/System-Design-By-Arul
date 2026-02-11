# Services (Monolith vs Microservices) - Complete Guide

## What are Services?

Think of Services like **departments in a company**:
- **Monolith**: One big department does everything
- **Microservices**: Specialized departments (HR, Finance, Sales, etc.)

---

## Monolithic Architecture

### Everything in One Place:
```
┌─────────────────────────────────────────────────────────────────┐
│                     MONOLITHIC APPLICATION                       │
│                                                                  │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│  │  User   │  │  Video  │  │ Comment │  │ Payment │             │
│  │ Module  │  │ Module  │  │ Module  │  │ Module  │             │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘             │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    SHARED DATABASE                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ONE codebase, ONE deployment, ONE process                       │
└─────────────────────────────────────────────────────────────────┘
```

### Pros:
```
✓ Simple to develop initially
✓ Easy to test end-to-end
✓ Simple deployment (one thing to deploy)
✓ No network latency between modules
✓ Easy to debug (one process)
✓ Simple transactions (one database)
```

### Cons:
```
✗ Gets harder to maintain as it grows
✗ Small change = redeploy EVERYTHING
✗ One bug can crash entire system
✗ Hard to scale specific parts
✗ Technology lock-in (one language)
✗ Long build/deploy times
✗ Team coordination gets difficult
```

---

## Microservices Architecture

### Separate, Independent Services:
```
┌─────────────────────────────────────────────────────────────────┐
│                    MICROSERVICES                                 │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ User Service│  │Video Service│  │Comment Svc │              │
│  │   Node.js   │  │    Go       │  │   Python   │              │
│  │   ┌─────┐   │  │   ┌─────┐   │  │   ┌─────┐  │              │
│  │   │ DB  │   │  │   │ DB  │   │  │   │ DB  │  │              │
│  │   └─────┘   │  │   └─────┘   │  │   └─────┘  │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│         ▲                 ▲                ▲                     │
│         │                 │                │                     │
│         └─────────────────┼────────────────┘                     │
│                           │                                      │
│               ┌───────────┴───────────┐                          │
│               │      API Gateway      │                          │
│               └───────────────────────┘                          │
│                                                                  │
│  DIFFERENT codebases, SEPARATE deployments                       │
└─────────────────────────────────────────────────────────────────┘
```

### Pros:
```
✓ Scale services independently
✓ Use best technology for each service
✓ Small, focused teams
✓ Independent deployments
✓ Fault isolation (one service crash doesn't kill all)
✓ Easier to understand individual services
```

### Cons:
```
✗ More complex infrastructure
✗ Network latency between services
✗ Distributed transactions are hard
✗ Debugging across services is tricky
✗ Need DevOps expertise
✗ Data consistency challenges
```

---

## When to Use What?

### Use Monolith When:
```
✓ Small team (1-5 developers)
✓ Simple application
✓ Starting a new project (you don't know what you need yet)
✓ Quick time to market needed
✓ Limited DevOps experience

"Start with monolith, split later if needed"
```

### Use Microservices When:
```
✓ Large team (50+ developers)
✓ Different parts need different scaling
✓ Want independent deployments
✓ Multiple technologies needed
✓ Clear domain boundaries exist

"Only when complexity justifies it"
```

---

## How to Split a Monolith

### Step 1: Identify Boundaries (Domain-Driven Design)
```
YouTube Domains:
├── User Management (registration, auth, profiles)
├── Video Management (upload, process, metadata)
├── Streaming (video delivery, CDN)
├── Comments (CRUD, moderation)
├── Subscriptions (follow, notifications)
├── Search (indexing, querying)
├── Recommendations (ML-based suggestions)
├── Analytics (views, engagement)
└── Payments (subscriptions, creator payouts)
```

### Step 2: Extract One Service at a Time
```
Monolith
   │
   ├── Extract User Service (lowest risk)
   │      ↓
   │   Test, stabilize
   │      ↓
   ├── Extract Video Service
   │      ↓
   │   Test, stabilize
   │      ↓
   └── Continue...

Never try to extract all at once!
```

### Step 3: Define APIs Between Services
```javascript
// User Service API
GET  /users/:id          → Get user
POST /users              → Create user
PUT  /users/:id          → Update user

// Video Service API
GET  /videos/:id         → Get video
POST /videos             → Create video
GET  /users/:id/videos   → Get user's videos

// Comment Service API
GET  /videos/:id/comments → Get video comments
POST /videos/:id/comments → Add comment
```

---

## Service Communication

### 1. Synchronous (REST/gRPC)
```
User Service                    Video Service
     │                               │
     │ ──── GET /users/123 ────────> │
     │ <────── Response ──────────── │
     │                               │

Quick response needed!
But: Creates dependency (if Video Service down, caller waits)
```

### 2. Asynchronous (Message Queue)
```
User Service          Queue           Video Service
     │                  │                   │
     │ ── Push msg ──>  │                   │
     │ (returns fast!)  │ ── Pull msg ──>   │
     │                  │                   │
     │                  │                   │ (processes)
     │                  │ <── Ack ──────    │

No waiting! Fire and forget.
But: No immediate response
```

### When to Use What:
```
Synchronous (REST/gRPC):
- Need immediate response
- Query operations
- Simple request/response

Asynchronous (Queues):
- Background processing
- Event notifications
- Heavy operations
- Loose coupling needed
```

---

## Service Discovery

### How do services find each other?
```
Without Service Discovery:
Video Service hardcodes: "User Service at 192.168.1.5:3000"
IP changes? Everything breaks!

With Service Discovery:
┌─────────────────────────────────────────────┐
│           SERVICE REGISTRY                   │
│  ┌─────────────────────────────────────┐    │
│  │ user-service: 192.168.1.5:3000      │    │
│  │               192.168.1.6:3000      │    │
│  │ video-service: 192.168.1.10:3000    │    │
│  │                192.168.1.11:3000    │    │
│  └─────────────────────────────────────┘    │
└─────────────────────────────────────────────┘

Video Service asks: "Where is user-service?"
Registry responds: "192.168.1.5:3000 and 192.168.1.6:3000"
```

### Tools:
```
- Kubernetes (built-in DNS)
- Consul
- etcd
- AWS Cloud Map
```

---

## API Gateway Pattern

### Single Entry Point for All Services:
```
                    ┌────────────────────────────┐
   Client  ────────>│       API GATEWAY          │
                    │                            │
                    │  - Authentication          │
                    │  - Rate limiting           │
                    │  - Request routing         │
                    │  - Load balancing          │
                    │  - Response aggregation    │
                    │                            │
                    └────────────┬───────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         ▼                       ▼                       ▼
   ┌───────────┐          ┌───────────┐          ┌───────────┐
   │   User    │          │   Video   │          │  Comment  │
   │  Service  │          │  Service  │          │  Service  │
   └───────────┘          └───────────┘          └───────────┘
```

---

## Database Per Service

### Each Service Has Its Own Data:
```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│ User Service│     │Video Service│     │Comment Svc │
│ ┌─────────┐ │     │ ┌─────────┐ │     │ ┌─────────┐ │
│ │PostgreSQL│ │     │ │ MongoDB │ │     │ │ Cassandra│ │
│ │  Users   │ │     │ │ Videos  │ │     │ │ Comments│ │
│ └─────────┘ │     └─────────┘ │     │ └─────────┘ │
└─────────────┘     └─────────────┘     └─────────────┘

Why own database?
- No tight coupling
- Can use best DB for each service
- Independent scaling
```

### The Challenge: Joins Across Services
```
Old way (monolith):
SELECT * FROM users 
JOIN videos ON users.id = videos.user_id;

New way (microservices):
1. Call User Service for user data
2. Call Video Service for video data
3. Combine in code

Or use API Composition/CQRS pattern
```

---

## Saga Pattern (Distributed Transactions)

### Problem:
```
User wants to upload video:
1. Create video record (Video Service)
2. Deduct storage quota (User Service)
3. Start processing (Transcoding Service)

If step 3 fails, need to undo steps 1 and 2!
```

### Solution: Saga
```
Choreography (Event-based):
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Video     │     │    User     │     │ Transcoding │
│  Service    │     │   Service   │     │  Service    │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       │ VideoCreated      │                   │
       │──────────────────>│                   │
       │                   │ QuotaDeducted     │
       │                   │──────────────────>│
       │                   │                   │
       │                   │                   │ (Fails!)
       │                   │                   │
       │                   │ TranscodingFailed │
       │                   │<──────────────────│
       │ RestoreQuota      │                   │
       │<──────────────────│                   │
       │                   │                   │
       │ Delete video      │                   │
       │ (compensate)      │                   │

Each step has a compensating action!
```

---

## Microservices for YouTube

### Architecture:
```
┌─────────────────────────────────────────────────────────────────────┐
│                          API GATEWAY                                 │
│            (Kong / AWS API Gateway / NGINX)                         │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
    ┌──────────────┬──────────────┼──────────────┬──────────────┐
    ▼              ▼              ▼              ▼              ▼
┌────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌────────┐
│  User  │   │  Video   │   │ Comment  │   │  Search  │   │ Payment│
│Service │   │ Service  │   │ Service  │   │ Service  │   │Service │
│        │   │          │   │          │   │          │   │        │
│Postgres│   │ MongoDB  │   │Cassandra │   │Elastic   │   │Postgres│
└────────┘   └──────────┘   └──────────┘   │Search    │   └────────┘
                                           └──────────┘
    │              │              │              │              │
    └──────────────┴──────────────┴──────────────┴──────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │      MESSAGE QUEUE        │
                    │    (Kafka / RabbitMQ)     │
                    └───────────────────────────┘
                                  │
    ┌──────────────────────────────┴──────────────────────────────┐
    │                                                              │
    ▼                              ▼                               ▼
┌──────────┐                ┌──────────────┐               ┌───────────┐
│Transcode │                │Notification │               │ Analytics │
│ Worker   │                │  Service    │               │  Service  │
│          │                │             │               │           │
│(CPU heavy)               │ (Email/Push) │               │ClickHouse│
└──────────┘                └──────────────┘               └───────────┘
```

### Service Responsibilities:
```javascript
// User Service
{
    responsibilities: [
        'User registration and authentication',
        'Profile management',
        'Subscription tracking',
        'OAuth integrations'
    ],
    database: 'PostgreSQL',
    exposes: '/api/v1/users/*'
}

// Video Service
{
    responsibilities: [
        'Video metadata CRUD',
        'Video upload initiation',
        'View count updates'
    ],
    database: 'MongoDB',
    exposes: '/api/v1/videos/*',
    publishes: ['video.created', 'video.deleted']
}

// Transcoding Service
{
    responsibilities: [
        'Convert videos to multiple resolutions',
        'Generate thumbnails',
        'Extract video info'
    ],
    subscribes: ['video.created'],
    publishes: ['video.processed'],
    type: 'Worker (no HTTP API)'
}

// Comment Service
{
    responsibilities: [
        'Comment CRUD',
        'Reply threading',
        'Comment moderation'
    ],
    database: 'Cassandra',
    exposes: '/api/v1/comments/*'
}

// Search Service
{
    responsibilities: [
        'Index videos',
        'Handle search queries',
        'Auto-complete'
    ],
    database: 'Elasticsearch',
    subscribes: ['video.created', 'video.updated'],
    exposes: '/api/v1/search/*'
}
```

---

## Service Example Implementation

### User Service (Node.js):
```javascript
const express = require('express');
const app = express();

// User endpoints
app.get('/users/:id', async (req, res) => {
    const user = await db.users.findById(req.params.id);
    res.json(user);
});

app.post('/users', async (req, res) => {
    const user = await db.users.create(req.body);
    
    // Publish event for other services
    await kafka.publish('user.created', user);
    
    res.status(201).json(user);
});

// Health check
app.get('/health', (req, res) => {
    res.json({ status: 'healthy' });
});

app.listen(3001);
```

### Video Service (Go):
```go
package main

import (
    "github.com/gin-gonic/gin"
)

func main() {
    r := gin.Default()
    
    r.GET("/videos/:id", getVideo)
    r.POST("/videos", createVideo)
    r.GET("/health", healthCheck)
    
    r.Run(":3002")
}

func createVideo(c *gin.Context) {
    var video Video
    c.BindJSON(&video)
    
    // Save to MongoDB
    db.Videos.Insert(video)
    
    // Publish event for transcoding
    kafka.Publish("video.created", video)
    
    c.JSON(201, video)
}
```

---

## Interview Questions

**Q: What are microservices?**
A: An architecture style where an application is built as a collection of small, independent services that communicate over APIs. Each service is focused on a specific business capability.

**Q: When should you use microservices over monolith?**
A: When you have large teams, need independent deployments, different scaling requirements, or want technology flexibility. Start with monolith for new projects.

**Q: How do microservices communicate?**
A: Synchronously via REST/gRPC for request-response, or asynchronously via message queues (Kafka, RabbitMQ) for events.

**Q: What is the hardest part of microservices?**
A: Distributed transactions, data consistency, debugging across services, and operational complexity.

**Q: How to handle transactions across services?**
A: Saga pattern - sequence of local transactions with compensating actions for rollback.

---

## Quick Summary

```
MONOLITH:
- One codebase, one deployment
- Simple but doesn't scale well
- Good for small teams/projects

MICROSERVICES:
- Many services, many deployments
- Complex but scales well
- Good for large teams/projects

Key Patterns:
- API Gateway (single entry)
- Service Discovery (find services)
- Database per service (isolation)
- Saga (distributed transactions)

Communication:
- Sync: REST, gRPC
- Async: Message queues

Rule: Start monolith, split when needed!
```

You now understand Services like a pro! 🚀
