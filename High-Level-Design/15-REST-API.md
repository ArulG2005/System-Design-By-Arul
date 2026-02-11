# REST API - Complete Guide

## What is REST?

REST = **RE**presentational **S**tate **T**ransfer

Think of REST like a **library system**:
- Resources (books) have unique IDs (ISBN)
- Standard actions (borrow, return, request)
- Stateless interactions (each visit independent)

**Simple Definition:**
REST is a set of rules for building web APIs that use HTTP methods to perform operations on resources.

---

## REST Principles

### 1. Client-Server Architecture
```
Client                          Server
┌─────────┐                    ┌─────────┐
│  React  │ ◄───── HTTP ─────► │ Node.js │
│  App    │                    │   API   │
└─────────┘                    └─────────┘

Client and server are independent
Either can be replaced without affecting other
```

### 2. Stateless
```
Bad (Stateful):
Request 1: "I'm John, show my cart"
Server: "OK, remembering you're John..."
Request 2: "Add item to cart"
Server: "Added to John's cart" (remembered from before)

Good (Stateless):
Request 1: "I'm John (token: abc), show my cart"
Server: "Here's John's cart"
Request 2: "I'm John (token: abc), add item to cart"
Server: "Added to John's cart"

Each request contains all info needed!
```

### 3. Uniform Interface
```
Same HTTP methods work for all resources:

GET     → Read
POST    → Create
PUT     → Update (full)
PATCH   → Update (partial)
DELETE  → Remove

Works for /users, /videos, /comments, everything!
```

### 4. Resource-Based
```
Everything is a resource with a unique URL:

/users           → All users
/users/123       → User #123
/users/123/videos → Videos by user #123
/videos/abc      → Video abc
```

### 5. Cacheable
```
HTTP/1.1 200 OK
Cache-Control: max-age=3600
ETag: "abc123"

{
    "data": "..."
}

Clients can cache responses
Headers indicate cache rules
```

### 6. Layered System
```
Client → CDN → Load Balancer → API → Database

Client doesn't know (or care) about layers
Each layer only knows about adjacent layer
```

---

## HTTP Methods

```
┌──────────┬────────────┬─────────────┬───────────────────┐
│  Method  │   CRUD     │  Idempotent │    Description    │
├──────────┼────────────┼─────────────┼───────────────────┤
│  GET     │  Read      │    Yes      │  Retrieve data    │
│  POST    │  Create    │    No       │  Create new       │
│  PUT     │  Update    │    Yes      │  Replace entirely │
│  PATCH   │  Update    │    No       │  Partial update   │
│  DELETE  │  Delete    │    Yes      │  Remove resource  │
│  HEAD    │  Read      │    Yes      │  Get headers only │
│  OPTIONS │  -         │    Yes      │  Get allowed ops  │
└──────────┴────────────┴─────────────┴───────────────────┘

Idempotent = Same request multiple times = Same result
```

---

## URL Design

### Good URL Structure:
```
https://api.youtube.com/v1/videos                # All videos
https://api.youtube.com/v1/videos/abc123         # Specific video
https://api.youtube.com/v1/users/456             # Specific user
https://api.youtube.com/v1/users/456/videos      # Videos by user
https://api.youtube.com/v1/videos/abc123/comments # Comments on video
```

### Rules:
```
✓ Use nouns, not verbs
  Good: /users
  Bad:  /getUsers, /createUser

✓ Use plural nouns
  Good: /videos
  Bad:  /video

✓ Use lowercase
  Good: /users
  Bad:  /Users, /USERS

✓ Use hyphens, not underscores
  Good: /watch-history
  Bad:  /watch_history

✓ Nest for relationships
  Good: /users/123/videos
  Bad:  /user-videos?userId=123

✓ Keep URLs short
  Good: /users/123/videos
  Bad:  /api/v1/resources/users/id/123/sub-resources/videos/list
```

### Query Parameters:
```
Filtering:
GET /videos?category=gaming&status=published

Sorting:
GET /videos?sort=created_at&order=desc

Pagination:
GET /videos?page=2&limit=20
GET /videos?cursor=abc123&limit=20

Searching:
GET /videos?q=funny+cats

Field Selection:
GET /videos?fields=id,title,thumbnail

Combined:
GET /videos?category=gaming&sort=views&order=desc&page=1&limit=10
```

---

## HTTP Status Codes

### 2xx Success:
```
200 OK           → GET success, body contains data
201 Created      → POST success, resource created
202 Accepted     → Request accepted, processing async
204 No Content   → DELETE success, no body returned
```

### 3xx Redirection:
```
301 Moved Permanently → Resource moved, update URL
302 Found            → Temporary redirect
304 Not Modified     → Cached version still valid
```

### 4xx Client Errors:
```
400 Bad Request      → Invalid request syntax/data
401 Unauthorized     → Not authenticated
403 Forbidden        → Authenticated but not authorized
404 Not Found        → Resource doesn't exist
405 Method Not Allowed → HTTP method not supported
409 Conflict         → Resource conflict (e.g., duplicate)
422 Unprocessable Entity → Validation failed
429 Too Many Requests → Rate limit exceeded
```

### 5xx Server Errors:
```
500 Internal Server Error → Server crashed
502 Bad Gateway          → Upstream server error
503 Service Unavailable  → Server overloaded/maintenance
504 Gateway Timeout      → Upstream server timeout
```

---

## Request & Response Structure

### Request:
```http
POST /api/v1/videos HTTP/1.1
Host: api.youtube.com
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
Accept: application/json
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000

{
    "title": "My Awesome Video",
    "description": "This is a great video!",
    "category": "gaming",
    "tags": ["gaming", "tutorial", "tips"]
}
```

### Response:
```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/v1/videos/abc123
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000

{
    "status": "success",
    "data": {
        "id": "abc123",
        "title": "My Awesome Video",
        "description": "This is a great video!",
        "category": "gaming",
        "tags": ["gaming", "tutorial", "tips"],
        "created_at": "2024-01-15T10:30:00Z",
        "updated_at": "2024-01-15T10:30:00Z"
    },
    "links": {
        "self": "/api/v1/videos/abc123",
        "comments": "/api/v1/videos/abc123/comments",
        "likes": "/api/v1/videos/abc123/likes"
    }
}
```

---

## Error Handling

### Standardized Error Response:
```json
{
    "status": "error",
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Validation failed",
        "details": [
            {
                "field": "title",
                "message": "Title is required"
            },
            {
                "field": "category",
                "message": "Invalid category. Must be one of: gaming, music, education"
            }
        ]
    },
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2024-01-15T10:30:00Z"
}
```

### Error Codes:
```javascript
const ERROR_CODES = {
    // Client Errors
    'VALIDATION_ERROR': 400,
    'UNAUTHORIZED': 401,
    'FORBIDDEN': 403,
    'NOT_FOUND': 404,
    'CONFLICT': 409,
    'RATE_LIMITED': 429,
    
    // Server Errors
    'INTERNAL_ERROR': 500,
    'SERVICE_UNAVAILABLE': 503
};
```

---

## REST API Implementation

### Express.js Example:
```javascript
const express = require('express');
const app = express();

// Middleware
app.use(express.json());

// GET all videos
app.get('/api/v1/videos', async (req, res) => {
    try {
        const { page = 1, limit = 10, category } = req.query;
        
        const query = category ? { category } : {};
        const videos = await Video
            .find(query)
            .skip((page - 1) * limit)
            .limit(Number(limit))
            .sort({ created_at: -1 });
        
        const total = await Video.countDocuments(query);
        
        res.json({
            status: 'success',
            data: videos,
            pagination: {
                page: Number(page),
                limit: Number(limit),
                total,
                pages: Math.ceil(total / limit)
            }
        });
    } catch (error) {
        res.status(500).json({
            status: 'error',
            error: { code: 'INTERNAL_ERROR', message: error.message }
        });
    }
});

// GET single video
app.get('/api/v1/videos/:id', async (req, res) => {
    try {
        const video = await Video.findById(req.params.id);
        
        if (!video) {
            return res.status(404).json({
                status: 'error',
                error: { code: 'NOT_FOUND', message: 'Video not found' }
            });
        }
        
        res.json({ status: 'success', data: video });
    } catch (error) {
        res.status(500).json({
            status: 'error',
            error: { code: 'INTERNAL_ERROR', message: error.message }
        });
    }
});

// POST create video
app.post('/api/v1/videos', authenticate, async (req, res) => {
    try {
        const { title, description, category, tags } = req.body;
        
        // Validation
        if (!title) {
            return res.status(400).json({
                status: 'error',
                error: {
                    code: 'VALIDATION_ERROR',
                    details: [{ field: 'title', message: 'Title is required' }]
                }
            });
        }
        
        const video = await Video.create({
            title,
            description,
            category,
            tags,
            user_id: req.user.id
        });
        
        res.status(201).json({
            status: 'success',
            data: video
        });
    } catch (error) {
        res.status(500).json({
            status: 'error',
            error: { code: 'INTERNAL_ERROR', message: error.message }
        });
    }
});

// PUT update video (full replacement)
app.put('/api/v1/videos/:id', authenticate, async (req, res) => {
    try {
        const video = await Video.findOneAndUpdate(
            { _id: req.params.id, user_id: req.user.id },
            req.body,
            { new: true, runValidators: true }
        );
        
        if (!video) {
            return res.status(404).json({
                status: 'error',
                error: { code: 'NOT_FOUND', message: 'Video not found' }
            });
        }
        
        res.json({ status: 'success', data: video });
    } catch (error) {
        res.status(500).json({
            status: 'error',
            error: { code: 'INTERNAL_ERROR', message: error.message }
        });
    }
});

// PATCH partial update
app.patch('/api/v1/videos/:id', authenticate, async (req, res) => {
    try {
        const allowedUpdates = ['title', 'description', 'tags'];
        const updates = Object.keys(req.body)
            .filter(key => allowedUpdates.includes(key))
            .reduce((obj, key) => {
                obj[key] = req.body[key];
                return obj;
            }, {});
        
        const video = await Video.findOneAndUpdate(
            { _id: req.params.id, user_id: req.user.id },
            { $set: updates },
            { new: true }
        );
        
        if (!video) {
            return res.status(404).json({
                status: 'error',
                error: { code: 'NOT_FOUND', message: 'Video not found' }
            });
        }
        
        res.json({ status: 'success', data: video });
    } catch (error) {
        res.status(500).json({
            status: 'error',
            error: { code: 'INTERNAL_ERROR', message: error.message }
        });
    }
});

// DELETE video
app.delete('/api/v1/videos/:id', authenticate, async (req, res) => {
    try {
        const video = await Video.findOneAndDelete({
            _id: req.params.id,
            user_id: req.user.id
        });
        
        if (!video) {
            return res.status(404).json({
                status: 'error',
                error: { code: 'NOT_FOUND', message: 'Video not found' }
            });
        }
        
        res.status(204).send(); // No content
    } catch (error) {
        res.status(500).json({
            status: 'error',
            error: { code: 'INTERNAL_ERROR', message: error.message }
        });
    }
});
```

---

## HATEOAS (Hypermedia)

### Include Links for Discoverability:
```json
{
    "status": "success",
    "data": {
        "id": "abc123",
        "title": "My Video"
    },
    "links": {
        "self": "/api/v1/videos/abc123",
        "comments": "/api/v1/videos/abc123/comments",
        "likes": "/api/v1/videos/abc123/likes",
        "channel": "/api/v1/channels/xyz789",
        "related": "/api/v1/videos?related_to=abc123"
    },
    "actions": {
        "like": { "method": "POST", "href": "/api/v1/videos/abc123/likes" },
        "comment": { "method": "POST", "href": "/api/v1/videos/abc123/comments" },
        "share": { "method": "POST", "href": "/api/v1/videos/abc123/share" }
    }
}
```

---

## Versioning

### URL Versioning (Most Common):
```
/api/v1/videos
/api/v2/videos
```

### Header Versioning:
```http
GET /api/videos
Accept-Version: v2
```

### Content Negotiation:
```http
GET /api/videos
Accept: application/vnd.youtube.v2+json
```

---

## REST for YouTube Clone

### Complete API Design:
```
# Authentication
POST   /api/v1/auth/register
POST   /api/v1/auth/login
POST   /api/v1/auth/logout
POST   /api/v1/auth/refresh

# Users
GET    /api/v1/users/:id
PUT    /api/v1/users/:id
DELETE /api/v1/users/:id
GET    /api/v1/users/:id/videos
GET    /api/v1/users/:id/playlists
GET    /api/v1/users/:id/subscriptions
GET    /api/v1/users/:id/subscribers

# Videos
GET    /api/v1/videos
POST   /api/v1/videos
GET    /api/v1/videos/:id
PUT    /api/v1/videos/:id
PATCH  /api/v1/videos/:id
DELETE /api/v1/videos/:id
GET    /api/v1/videos/:id/comments
POST   /api/v1/videos/:id/comments
POST   /api/v1/videos/:id/like
DELETE /api/v1/videos/:id/like
POST   /api/v1/videos/:id/view

# Comments
GET    /api/v1/comments/:id
PUT    /api/v1/comments/:id
DELETE /api/v1/comments/:id
GET    /api/v1/comments/:id/replies
POST   /api/v1/comments/:id/replies

# Channels
GET    /api/v1/channels/:id
PUT    /api/v1/channels/:id
POST   /api/v1/channels/:id/subscribe
DELETE /api/v1/channels/:id/subscribe

# Search
GET    /api/v1/search?q=funny+cats&type=video

# Playlists
GET    /api/v1/playlists
POST   /api/v1/playlists
GET    /api/v1/playlists/:id
PUT    /api/v1/playlists/:id
DELETE /api/v1/playlists/:id
POST   /api/v1/playlists/:id/videos
DELETE /api/v1/playlists/:id/videos/:videoId
```

---

## Interview Questions

**Q: What is REST?**
A: An architectural style for web APIs using HTTP methods on resources. Key principles: stateless, uniform interface, resource-based, cacheable.

**Q: PUT vs PATCH?**
A: PUT replaces entire resource. PATCH updates only specified fields.

**Q: What makes an API RESTful?**
A: Uses HTTP methods correctly, stateless, has uniform interface, resources identified by URLs, supports caching.

**Q: How do you handle errors in REST?**
A: Use appropriate HTTP status codes (4xx for client, 5xx for server errors) with structured JSON error responses.

**Q: REST vs GraphQL?**
A: REST: multiple endpoints, fixed responses. GraphQL: single endpoint, client chooses fields. REST simpler, GraphQL more flexible.

---

## Quick Summary

```
REST PRINCIPLES:
- Client-Server separation
- Stateless requests
- Uniform interface (HTTP methods)
- Resource-based URLs
- Cacheable responses

HTTP METHODS:
- GET: Read
- POST: Create
- PUT: Full update
- PATCH: Partial update
- DELETE: Remove

URL RULES:
- Nouns, not verbs
- Plural
- Lowercase
- Hyphens

STATUS CODES:
- 2xx: Success
- 4xx: Client error
- 5xx: Server error
```

You now understand REST like a pro! 🚀
