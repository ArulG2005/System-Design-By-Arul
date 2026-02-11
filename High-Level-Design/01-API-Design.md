# API Design - Complete Guide

## What is an API?

API = Application Programming Interface

Think of API like a **waiter in a restaurant**:
- You (client) want food
- Kitchen (server) makes food
- Waiter (API) takes your order and brings food back

API is the middleman that lets two software talk to each other.

---

## Why Do We Need APIs?

```
Without API:
Your App ----X---- Database (Direct access = Security risk!)

With API:
Your App ----> API ----> Database (Safe and controlled!)
```

**Real Example:**
- Instagram app doesn't directly access Instagram's database
- It talks through Instagram's API
- API decides what you can and cannot do

---

## Types of APIs

### 1. REST API (Most Common)
- Uses HTTP methods (GET, POST, PUT, DELETE)
- Stateless (each request is independent)
- Returns JSON data

### 2. GraphQL
- Client asks exactly what data it needs
- Single endpoint
- More flexible than REST

### 3. gRPC
- Very fast
- Uses Protocol Buffers
- Good for microservices

### 4. SOAP
- Old style
- Uses XML
- Very strict rules

---

## REST API Design - The Main Course

### HTTP Methods (CRUD Operations)

| Method | Purpose | Example |
|--------|---------|---------|
| GET | Read data | Get user profile |
| POST | Create new data | Create new user |
| PUT | Update entire data | Update whole profile |
| PATCH | Update part of data | Change only email |
| DELETE | Remove data | Delete account |

---

## URL Design Rules

### Good URL Design:
```
GET    /users              → Get all users
GET    /users/123          → Get user with ID 123
POST   /users              → Create new user
PUT    /users/123          → Update user 123
DELETE /users/123          → Delete user 123


GET    /users/123/orders   → Get orders of user 123
GET    /users/123/orders/5 → Get order 5 of user 123
```

### Bad URL Design:
```
GET /getUser              ❌ (verb in URL)
GET /user/delete/123      ❌ (use DELETE method instead)
GET /Users                ❌ (use lowercase)
GET /user_orders          ❌ (use hyphens, not underscores)
```

---

## Request and Response Structure

### Request Example:
```http
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...

{
    "name": "John Doe",
    "email": "john@example.com",
    "age": 25
}
```

### Response Example:
```json
{
    "status": "success",
    "data": {
        "id": 123,
        "name": "John Doe",
        "email": "john@example.com",
        "age": 25,
        "created_at": "2024-01-15T10:30:00Z"
    },
    "message": "User created successfully"
}
```

---

## HTTP Status Codes

### Success Codes (2xx):
| Code | Meaning | When to Use |
|------|---------|-------------|
| 200 | OK | GET successful |
| 201 | Created | POST successful |
| 204 | No Content | DELETE successful |

### Client Error (4xx):
| Code | Meaning | When to Use |
|------|---------|-------------|
| 400 | Bad Request | Invalid data sent |
| 401 | Unauthorized | Not logged in |
| 403 | Forbidden | No permission |
| 404 | Not Found | Resource doesn't exist |
| 429 | Too Many Requests | Rate limit exceeded |

### Server Error (5xx):
| Code | Meaning | When to Use |
|------|---------|-------------|
| 500 | Internal Server Error | Server crashed |
| 502 | Bad Gateway | Proxy error |
| 503 | Service Unavailable | Server overloaded |

---

## Pagination - Handling Large Data

When you have 1 million users, you can't send all at once!

### Offset-Based Pagination:
```
GET /users?page=2&limit=10

Response:
{
    "data": [...10 users...],
    "pagination": {
        "current_page": 2,
        "per_page": 10,
        "total_pages": 100,
        "total_items": 1000
    }
}
```

### Cursor-Based Pagination (Better for large data):
```
GET /users?cursor=abc123&limit=10

Response:
{
    "data": [...10 users...],
    "next_cursor": "def456",
    "has_more": true
}
```

---

## Filtering, Sorting, and Searching

```
# Filtering
GET /products?category=electronics&price_min=100&price_max=500

# Sorting
GET /products?sort=price&order=desc

# Searching
GET /products?search=laptop

# Combined
GET /products?category=electronics&sort=price&order=asc&search=laptop&page=1&limit=20
```

---

## Versioning Your API

When you update API, old apps might break. Solution? Versioning!

### Method 1: URL Versioning (Most Common)
```
GET /v1/users
GET /v2/users
```

### Method 2: Header Versioning
```
GET /users
Header: API-Version: 2
```

### Method 3: Query Parameter
```
GET /users?version=2
```

---

## Error Handling

### Good Error Response:
```json
{
    "status": "error",
    "error": {
        "code": "VALIDATION_ERROR",
        "message": "Invalid email format",
        "details": [
            {
                "field": "email",
                "message": "Must be a valid email address"
            }
        ]
    },
    "timestamp": "2024-01-15T10:30:00Z",
    "request_id": "req_abc123"
}
```

---

## Rate Limiting

Protect your API from abuse:

```
Headers in Response:
X-RateLimit-Limit: 100        (max requests allowed)
X-RateLimit-Remaining: 95     (requests left)
X-RateLimit-Reset: 1642234567 (when limit resets)
```

---

## Authentication vs Authorization

### Authentication = WHO are you?
- Login with username/password
- Get a token

### Authorization = WHAT can you do?
- Admin can delete users
- Regular user cannot

---

## Building YouTube-like API

### Endpoints Design:
```
# Videos
GET    /videos                    → List all videos
GET    /videos/:id                → Get single video
POST   /videos                    → Upload video
PUT    /videos/:id                → Update video
DELETE /videos/:id                → Delete video

# User's Videos
GET    /users/:id/videos          → Get user's videos
GET    /users/:id/liked-videos    → Get liked videos

# Comments
GET    /videos/:id/comments       → Get comments
POST   /videos/:id/comments       → Add comment
DELETE /comments/:id              → Delete comment

# Subscriptions
POST   /users/:id/subscribe       → Subscribe to channel
DELETE /users/:id/subscribe       → Unsubscribe
GET    /users/:id/subscribers     → Get subscriber list

# Search
GET    /search?q=funny+cats&type=video
```

### Video Upload API Example:
```
POST /videos
Headers:
  Authorization: Bearer <token>
  Content-Type: multipart/form-data

Body:
  - file: video.mp4
  - title: "My Cat Video"
  - description: "Funny cat moments"
  - tags: ["cats", "funny", "pets"]
  - visibility: "public"

Response (201 Created):
{
    "data": {
        "id": "vid_abc123",
        "title": "My Cat Video",
        "status": "processing",
        "upload_progress": "complete",
        "processing_progress": "0%",
        "estimated_time": "5 minutes"
    }
}
```

---

## Best Practices Summary

1. ✅ Use nouns, not verbs in URLs
2. ✅ Use HTTP methods correctly
3. ✅ Use proper status codes
4. ✅ Version your API
5. ✅ Implement pagination
6. ✅ Handle errors gracefully
7. ✅ Use HTTPS always
8. ✅ Implement rate limiting
9. ✅ Document your API
10. ✅ Use consistent naming (snake_case or camelCase)

---

## Quick Code Example (Node.js/Express)

```javascript
const express = require('express');
const app = express();

// Get all videos
app.get('/api/v1/videos', async (req, res) => {
    try {
        const { page = 1, limit = 10 } = req.query;
        const videos = await Video.find()
            .skip((page - 1) * limit)
            .limit(limit);
        
        res.status(200).json({
            status: 'success',
            data: videos,
            pagination: {
                page: parseInt(page),
                limit: parseInt(limit)
            }
        });
    } catch (error) {
        res.status(500).json({
            status: 'error',
            message: 'Failed to fetch videos'
        });
    }
});

// Create video
app.post('/api/v1/videos', authenticate, async (req, res) => {
    try {
        const video = await Video.create({
            ...req.body,
            user_id: req.user.id
        });
        
        res.status(201).json({
            status: 'success',
            data: video
        });
    } catch (error) {
        res.status(400).json({
            status: 'error',
            message: error.message
        });
    }
});
```

---

## Interview Questions

**Q: What makes a good API?**
A: Easy to understand, consistent, well-documented, secure, versioned, and follows REST principles.

**Q: REST vs GraphQL?**
A: REST has multiple endpoints and fixed responses. GraphQL has one endpoint and client chooses what data it wants.

**Q: How to handle API breaking changes?**
A: Use versioning. Keep old version running while releasing new version.

---

You are now ready to design APIs like a pro! 🚀
