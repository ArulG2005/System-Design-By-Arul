# Proxy vs Reverse Proxy - Complete Guide

## What is a Proxy?

**Proxy = Middleman**

Think of it like a **personal assistant**:
- You want to buy something
- You don't go directly to the shop
- You send your assistant (proxy)
- Shop doesn't know who you are
- Assistant brings item back to you

---

## Forward Proxy (Regular Proxy)

### What it does:
```
Client ──> Forward Proxy ──> Internet/Server

The proxy acts on behalf of the CLIENT
Server doesn't know the real client
```

### Visual:
```
┌──────────┐         ┌──────────────┐         ┌──────────────┐
│  Client  │ ─────>  │   FORWARD    │ ─────>  │   Server     │
│ (You)    │         │    PROXY     │         │ (Website)    │
│          │ <─────  │              │ <─────  │              │
└──────────┘         └──────────────┘         └──────────────┘

Server sees: Proxy's IP (1.2.3.4)
Server doesn't see: Your real IP (192.168.1.100)
```

### Real Example:
```
You: "I want to access blocked-website.com"
Proxy: "OK, I'll fetch it for you"

Your request:
You (192.168.1.100) → Proxy (1.2.3.4) → blocked-website.com

blocked-website.com sees: Request from 1.2.3.4
blocked-website.com doesn't know: Real user is 192.168.1.100
```

---

## Forward Proxy Use Cases

### 1. Bypass Restrictions
```
Office blocks youtube.com

Direct: You ──X──> YouTube (blocked!)

With Proxy: You ──> Proxy ──> YouTube ✓
```

### 2. Anonymity/Privacy
```
Website wants to track you? Use proxy!

Website sees: Proxy's IP
Website doesn't see: Your real IP
```

### 3. Content Filtering (Corporate)
```
Company wants to block social media:

Employees ──> Corporate Proxy ──> Internet
                   │
                   ├── facebook.com → BLOCKED
                   ├── twitter.com  → BLOCKED
                   └── work.com     → ALLOWED
```

### 4. Caching (Save Bandwidth)
```
100 employees want same file:

Without proxy:
Employee 1 downloads 100MB
Employee 2 downloads 100MB
...
Total: 10GB downloaded from internet

With proxy (cached):
Employee 1 downloads 100MB (proxy caches it)
Employee 2 gets from proxy (0 internet)
...
Total: 100MB downloaded from internet
```

---

## Reverse Proxy

### What it does:
```
Client ──> Reverse Proxy ──> Backend Servers

The proxy acts on behalf of the SERVER
Client doesn't know the real server
```

### Visual:
```
┌──────────┐         ┌──────────────┐         ┌──────────────┐
│  Client  │ ─────>  │   REVERSE    │ ─────>  │  Server 1    │
│ (User)   │         │    PROXY     │    ├──> │  Server 2    │
│          │ <─────  │              │    └──> │  Server 3    │
└──────────┘         └──────────────┘         └──────────────┘

Client sees: Proxy's address (api.youtube.com)
Client doesn't see: Real servers (10.0.0.1, 10.0.0.2, etc.)
```

### Real Example:
```
You visit: youtube.com

What happens:
Your Browser ──> youtube.com (actually a reverse proxy)
                      │
                      ├──> video-server-1 (serves video)
                      ├──> user-server-2 (serves user data)
                      └──> comments-server-3 (serves comments)

You think: You're talking to one server
Reality: Multiple servers behind reverse proxy
```

---

## Reverse Proxy Use Cases

### 1. Load Balancing
```
                    ┌──> Server 1
Client ──> Proxy ───├──> Server 2
                    └──> Server 3

Distributes traffic among servers
```

### 2. SSL Termination
```
Client ──HTTPS──> Reverse Proxy ──HTTP──> Servers

Proxy handles encryption/decryption
Servers don't need SSL certificates
Easier certificate management
```

### 3. Caching
```
Client asks for: GET /trending-videos

First request:
Client ──> Proxy ──> Server (fetches data)
                ↓
        Proxy caches response

Next requests:
Client ──> Proxy (returns cached data)
        (no server hit!)
```

### 4. Security (Hide Servers)
```
Without reverse proxy:
Hackers know: Server at 10.0.0.5:3000

With reverse proxy:
Hackers see: api.youtube.com
Hackers don't know: Internal server IPs
```

### 5. Compression
```
Server sends: 1MB response

Proxy compresses: 200KB (gzip)

Client receives: 200KB

Bandwidth saved!
```

### 6. Rate Limiting
```
Client making too many requests:

Request 1 ✓
Request 2 ✓
...
Request 101 → Proxy: "Too many requests!" (blocks)
```

---

## Forward Proxy vs Reverse Proxy - Side by Side

```
                FORWARD PROXY              REVERSE PROXY
                ─────────────              ─────────────

Who uses it?    Client (you)              Server (website)

Purpose         Access internet           Receive internet
                through proxy             traffic

Who's hidden?   CLIENT is hidden          SERVER is hidden
                from server               from client

Sits where?     In front of               In front of
                clients                   servers

Example use     - VPN                     - Load balancing
                - Bypass blocks           - SSL termination
                - Anonymity               - Caching
                - Content filter          - Security


┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  FORWARD PROXY                                              │
│                                                             │
│  [Client A]                                                 │
│  [Client B] ───> [Forward Proxy] ───> [Internet/Servers]   │
│  [Client C]                                                 │
│                                                             │
│  Clients know the proxy                                     │
│  Servers don't know real clients                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  REVERSE PROXY                                              │
│                                                             │
│                                       [Server A]            │
│  [Clients] ───> [Reverse Proxy] ───>  [Server B]            │
│                                       [Server C]            │
│                                                             │
│  Servers know the proxy                                     │
│  Clients don't know real servers                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## NGINX as Reverse Proxy

### Basic Reverse Proxy:
```nginx
# /etc/nginx/nginx.conf

server {
    listen 80;
    server_name api.myapp.com;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### With Load Balancing:
```nginx
upstream backend_servers {
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
    server 10.0.0.3:3000;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://backend_servers;
    }
}
```

### With SSL Termination:
```nginx
server {
    listen 443 ssl;
    server_name api.myapp.com;
    
    ssl_certificate /etc/ssl/certs/myapp.crt;
    ssl_certificate_key /etc/ssl/private/myapp.key;
    
    location / {
        proxy_pass http://backend_servers;  # HTTP internally
    }
}
```

### With Caching:
```nginx
proxy_cache_path /var/cache/nginx levels=1:2 
                 keys_zone=my_cache:10m 
                 max_size=10g 
                 inactive=60m;

server {
    listen 80;
    
    location / {
        proxy_cache my_cache;
        proxy_cache_valid 200 10m;  # Cache 200 responses for 10 min
        proxy_pass http://backend;
    }
}
```

### With Rate Limiting:
```nginx
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;

server {
    listen 80;
    
    location /api/ {
        limit_req zone=mylimit burst=20;
        proxy_pass http://backend;
    }
}
```

---

## Complete YouTube Clone Reverse Proxy Setup

```nginx
# Main configuration
upstream video_service {
    least_conn;
    server video-1:3000;
    server video-2:3000;
    server video-3:3000;
}

upstream user_service {
    server user-1:3000;
    server user-2:3000;
}

upstream upload_service {
    server upload-1:3000 weight=3;
    server upload-2:3000 weight=2;
}

upstream streaming_service {
    server stream-1:3000;
    server stream-2:3000;
}

# Rate limiting zones
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/s;
limit_req_zone $binary_remote_addr zone=upload_limit:10m rate=1r/s;

# Caching
proxy_cache_path /var/cache/nginx/videos 
                 levels=1:2 
                 keys_zone=video_cache:100m 
                 max_size=50g;

server {
    listen 443 ssl http2;
    server_name youtube-clone.com;
    
    # SSL
    ssl_certificate /etc/ssl/certs/youtube-clone.crt;
    ssl_certificate_key /etc/ssl/private/youtube-clone.key;
    
    # Compression
    gzip on;
    gzip_types application/json text/plain text/css application/javascript;
    
    # Video streaming (cached)
    location /watch/ {
        proxy_cache video_cache;
        proxy_cache_valid 200 1h;
        proxy_pass http://streaming_service;
    }
    
    # Video uploads (rate limited, big files)
    location /upload {
        limit_req zone=upload_limit burst=5;
        client_max_body_size 10G;
        proxy_read_timeout 600;
        proxy_pass http://upload_service;
    }
    
    # API endpoints (rate limited)
    location /api/videos {
        limit_req zone=api_limit burst=50;
        proxy_pass http://video_service;
    }
    
    location /api/users {
        limit_req zone=api_limit burst=50;
        proxy_pass http://user_service;
    }
    
    # Static files (CDN or cached heavily)
    location /static/ {
        proxy_cache video_cache;
        proxy_cache_valid 200 7d;
        add_header Cache-Control "public, max-age=604800";
        proxy_pass http://cdn.youtube-clone.com;
    }
    
    # Health check endpoint
    location /health {
        return 200 'OK';
        add_header Content-Type text/plain;
    }
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name youtube-clone.com;
    return 301 https://$server_name$request_uri;
}
```

---

## Node.js as Reverse Proxy (http-proxy)

```javascript
const express = require('express');
const httpProxy = require('http-proxy');

const app = express();
const proxy = httpProxy.createProxyServer();

// Define service endpoints
const services = {
    videos: 'http://localhost:3001',
    users: 'http://localhost:3002',
    comments: 'http://localhost:3003'
};

// Logging middleware
app.use((req, res, next) => {
    console.log(`${req.method} ${req.url} -> proxying...`);
    next();
});

// Route to video service
app.use('/api/videos', (req, res) => {
    proxy.web(req, res, { target: services.videos });
});

// Route to user service
app.use('/api/users', (req, res) => {
    proxy.web(req, res, { target: services.users });
});

// Route to comments service
app.use('/api/comments', (req, res) => {
    proxy.web(req, res, { target: services.comments });
});

// Handle proxy errors
proxy.on('error', (err, req, res) => {
    res.status(502).json({ error: 'Bad Gateway', message: err.message });
});

app.listen(8080, () => {
    console.log('Reverse proxy running on port 8080');
});
```

---

## Common Proxy Headers

When using reverse proxy, add these headers:

```nginx
# nginx config
location / {
    proxy_pass http://backend;
    
    # Pass real client IP
    proxy_set_header X-Real-IP $remote_addr;
    
    # Pass full proxy chain
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    
    # Pass original host
    proxy_set_header Host $host;
    
    # Pass original protocol (http/https)
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

```javascript
// Access in Node.js app
app.get('/api/data', (req, res) => {
    const clientIP = req.headers['x-real-ip'] || req.ip;
    const originalHost = req.headers['host'];
    const protocol = req.headers['x-forwarded-proto'];
    
    console.log(`Request from ${clientIP} via ${protocol}://${originalHost}`);
});
```

---

## Proxy vs VPN

```
PROXY                           VPN
─────                           ───

Works at:                       Works at:
Application level               Network level
(browser only)                  (entire device)

Encryption:                     Encryption:
Usually none                    Always encrypted
(unless HTTPS proxy)            

Speed:                          Speed:
Faster                          Slower (encryption overhead)

Use case:                       Use case:
Browse anonymously              Secure all internet traffic
Bypass website blocks           Access private networks
```

---

## Interview Questions

**Q: What's the difference between forward proxy and reverse proxy?**
A: Forward proxy hides clients from servers (client-side). Reverse proxy hides servers from clients (server-side).

**Q: Why use a reverse proxy instead of direct server access?**
A: Load balancing, SSL termination, caching, security, rate limiting, and hiding internal server architecture.

**Q: Can NGINX be both?**
A: Yes! NGINX can act as forward proxy, reverse proxy, load balancer, and web server.

**Q: How does a reverse proxy help with security?**
A: Hides internal server IPs, can filter malicious requests, implements rate limiting, and handles SSL centrally.

---

## Quick Summary

```
FORWARD PROXY:
- Sits in front of CLIENTS
- Clients use it to access servers
- Hides client identity from servers
- Use: VPN, bypass blocks, anonymity

REVERSE PROXY:
- Sits in front of SERVERS
- Servers use it to receive traffic
- Hides server identity from clients
- Use: Load balancing, SSL, caching, security

Common Tool: NGINX (can do both!)
```

---

## Mental Model

```
Forward Proxy:  CLIENT → [PROXY] → SERVER
                "Client's agent"

Reverse Proxy:  CLIENT → [PROXY] → SERVER
                "Server's agent"

Same position, different purpose!
```

You now understand Proxy vs Reverse Proxy like a pro! 🚀
