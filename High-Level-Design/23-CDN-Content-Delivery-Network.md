# CDN (Content Delivery Network) - Complete Guide

## What is a CDN?

Think of it like **pizza chains**:
- Instead of one central kitchen serving everyone
- Multiple kitchens (edge servers) in every neighborhood
- Order goes to nearest kitchen
- Faster delivery, less traffic to central kitchen

**Simple Definition:**
A CDN is a network of servers distributed globally that deliver content from locations closest to users.

---

## The Problem CDN Solves

### Without CDN:
```
User in Tokyo                      Origin Server in New York
    │                                      │
    │────── Request (12,000 km) ──────────►│
    │                                      │
    │◄───── Response (12,000 km) ──────────│
    │                                      │

Latency: ~200ms round trip
Every request travels across the world!
```

### With CDN:
```
User in Tokyo          Edge Server in Tokyo     Origin (New York)
    │                          │                      │
    │── Request (50 km) ──────►│                      │
    │                          │                      │
    │◄─ Response (50 km) ──────│                      │
    │                          │                      │

Latency: ~20ms round trip
Content served from nearby edge!
```

---

## How CDN Works

### Basic Flow:
```
1. User requests video from YouTube

2. DNS resolves to nearest CDN edge

3. Edge server checks:
   ├── HIT: Content in cache → Return immediately
   └── MISS: Fetch from origin → Cache → Return

┌─────────────────────────────────────────────────────────────────────┐
│                         CDN Architecture                             │
└─────────────────────────────────────────────────────────────────────┘

                    www.youtube.com
                          │
                          ▼
                   ┌─────────────┐
                   │    DNS      │
                   │  (Anycast)  │
                   └──────┬──────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   ┌─────────┐       ┌─────────┐       ┌─────────┐
   │ Edge    │       │ Edge    │       │ Edge    │
   │ Tokyo   │       │ London  │       │ NYC     │
   └────┬────┘       └────┬────┘       └────┬────┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                          │
                    ┌─────┴─────┐
                    │  Origin   │
                    │  Servers  │
                    └───────────┘
```

### Point of Presence (PoP):
```
A PoP = CDN location with edge servers

Global CDN:
┌────────────────────────────────────────────────────────────────┐
│                                                                 │
│     🌐 North America        🌐 Europe          🌐 Asia         │
│     ├── New York            ├── London        ├── Tokyo        │
│     ├── Los Angeles         ├── Frankfurt     ├── Singapore    │
│     ├── Chicago             ├── Paris         ├── Hong Kong    │
│     └── Miami               └── Amsterdam     └── Sydney       │
│                                                                 │
│                        🌐 South America                        │
│                        └── São Paulo                           │
│                                                                 │
└────────────────────────────────────────────────────────────────┘

More PoPs = Content closer to users = Faster delivery
```

---

## CDN Caching

### What Gets Cached:
```
Static Content (Always cache):
✓ Images (thumbnails, avatars)
✓ Videos (streaming content)
✓ CSS/JavaScript files
✓ Fonts
✓ Static HTML

Dynamic Content (Cache carefully):
⚡ API responses (with short TTL)
⚡ Personalized pages (edge computing)
⚡ Search results (with key variations)

Never Cache:
✗ Payment pages
✗ User-specific data
✗ Real-time data
```

### Cache Headers:
```http
# Origin server sends cache instructions

# Cache for 1 year (static assets)
Cache-Control: public, max-age=31536000, immutable

# Cache for 5 minutes, revalidate after
Cache-Control: public, max-age=300, must-revalidate

# Don't cache at CDN, but OK for browser
Cache-Control: private, max-age=3600

# Never cache
Cache-Control: no-store

# Conditional caching with ETag
ETag: "abc123"
# Client sends: If-None-Match: "abc123"
# Server responds: 304 Not Modified (use cached version)
```

### Cache Key:
```
CDN creates unique key for each cached item:

Default key:
URL = key
https://yt.com/video/abc123 → "video/abc123"

With variations:
URL + Query params
https://yt.com/video/abc123?quality=720p → "video/abc123?quality=720p"
https://yt.com/video/abc123?quality=1080p → "video/abc123?quality=1080p"

Custom key (via Vary header):
Vary: Accept-Encoding, Accept-Language
→ Different cache per encoding (gzip vs br) and language
```

---

## CDN Features

### 1. Load Balancing
```
                    ┌─────────────────┐
                    │   CDN Edge      │
                    │   Load Balancer │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           ▼                 ▼                 ▼
     ┌─────────┐       ┌─────────┐       ┌─────────┐
     │ Origin 1│       │ Origin 2│       │ Origin 3│
     └─────────┘       └─────────┘       └─────────┘

Distributes origin requests across multiple servers
```

### 2. DDoS Protection
```
Attack traffic:
100M malicious requests/sec
           │
           ▼
    ┌──────────────┐
    │    CDN       │
    │ DDoS Shield  │
    │ (filters)    │
    └──────┬───────┘
           │
           ▼
    1000 legitimate requests/sec → Origin

CDN absorbs attack, only legitimate traffic passes
```

### 3. SSL/TLS Termination
```
User ←─── HTTPS ───► CDN Edge ←─── HTTP ───► Origin

Benefits:
- Edge handles CPU-heavy encryption
- Faster handshakes (closer to user)
- Centralized certificate management
```

### 4. Compression
```
Original file: 1MB JavaScript

CDN applies:
├── Gzip:   300KB (70% reduction)
└── Brotli: 250KB (75% reduction)

Served compressed, browser decompresses
```

### 5. Image Optimization
```
Original: 5MB high-res image

CDN transforms on-the-fly:
├── Mobile:  100KB (resized, compressed)
├── Tablet:  500KB (medium quality)
├── Desktop: 1MB (high quality)
└── WebP:    50% smaller than JPEG

URL parameters:
/image.jpg?width=300&format=webp&quality=80
```

---

## Video Streaming with CDN

### Adaptive Bitrate Streaming:
```
Video encoded at multiple qualities:
├── 240p  (400 Kbps)
├── 480p  (1 Mbps)
├── 720p  (2.5 Mbps)
├── 1080p (5 Mbps)
└── 4K    (15 Mbps)

Video split into small chunks:
video_240p_segment_001.ts
video_240p_segment_002.ts
video_720p_segment_001.ts
video_720p_segment_002.ts
...

User's player requests chunks based on bandwidth:
Fast connection → 1080p chunks
Slow connection → 480p chunks
Connection drops → Switch to lower quality mid-video
```

### HLS (HTTP Live Streaming):
```
Master playlist (m3u8):
┌─────────────────────────────────────────┐
│ #EXTM3U                                 │
│ #EXT-X-STREAM-INF:BANDWIDTH=400000      │
│ 240p/playlist.m3u8                      │
│ #EXT-X-STREAM-INF:BANDWIDTH=1000000     │
│ 480p/playlist.m3u8                      │
│ #EXT-X-STREAM-INF:BANDWIDTH=2500000     │
│ 720p/playlist.m3u8                      │
│ #EXT-X-STREAM-INF:BANDWIDTH=5000000     │
│ 1080p/playlist.m3u8                     │
└─────────────────────────────────────────┘

All segments cached at CDN edge!
```

---

## CDN Providers

### Major Players:
```
┌───────────────────┬──────────────────────────────────────────┐
│ Provider          │ Best For                                  │
├───────────────────┼──────────────────────────────────────────┤
│ Cloudflare        │ Free tier, DDoS protection, Workers       │
│ AWS CloudFront    │ AWS integration, Lambda@Edge              │
│ Akamai            │ Enterprise, largest network               │
│ Fastly            │ Real-time purging, edge computing         │
│ Google Cloud CDN  │ GCP integration, YouTube's tech           │
│ Azure CDN         │ Microsoft integration                     │
│ Bunny.net         │ Cost-effective, simple                    │
└───────────────────┴──────────────────────────────────────────┘
```

---

## CDN Configuration

### Cloudflare Example:
```javascript
// Cloudflare Worker (edge computing)
addEventListener('fetch', event => {
    event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
    const url = new URL(request.url);
    
    // Custom cache key
    const cacheKey = new Request(url.toString(), request);
    const cache = caches.default;
    
    // Check cache
    let response = await cache.match(cacheKey);
    
    if (!response) {
        // Cache miss - fetch from origin
        response = await fetch(request);
        
        // Clone response for caching
        const responseToCache = response.clone();
        
        // Add custom cache headers
        const headers = new Headers(responseToCache.headers);
        headers.set('Cache-Control', 'public, max-age=3600');
        
        // Store in cache
        event.waitUntil(
            cache.put(cacheKey, new Response(responseToCache.body, {
                status: responseToCache.status,
                headers
            }))
        );
    }
    
    return response;
}
```

### AWS CloudFront:
```yaml
# CloudFormation template
Resources:
  Distribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Origins:
          - Id: S3Origin
            DomainName: videos.s3.amazonaws.com
            S3OriginConfig:
              OriginAccessIdentity: ""
        DefaultCacheBehavior:
          TargetOriginId: S3Origin
          ViewerProtocolPolicy: redirect-to-https
          CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6
          Compress: true
        PriceClass: PriceClass_All
        Enabled: true
```

### NGINX as CDN:
```nginx
# nginx.conf for caching proxy

http {
    # Cache configuration
    proxy_cache_path /var/cache/nginx levels=1:2 
                     keys_zone=cdn_cache:100m 
                     max_size=10g 
                     inactive=60m;
    
    server {
        listen 80;
        server_name cdn.youtube.local;
        
        location /videos/ {
            proxy_pass http://origin-server;
            proxy_cache cdn_cache;
            proxy_cache_valid 200 1d;
            proxy_cache_valid 404 1m;
            proxy_cache_use_stale error timeout updating;
            
            # Add cache status header
            add_header X-Cache-Status $upstream_cache_status;
        }
        
        location /images/ {
            proxy_pass http://origin-server;
            proxy_cache cdn_cache;
            proxy_cache_valid 200 7d;
            
            # Image optimization
            image_filter resize 800 -;
            image_filter_jpeg_quality 85;
        }
    }
}
```

---

## Cache Invalidation

### Purge Methods:
```
1. URL Purge:
   DELETE /purge/videos/abc123
   → Only that specific URL cleared

2. Tag/Key Purge:
   POST /purge
   { "tags": ["user:123", "category:gaming"] }
   → All items with those tags cleared

3. Prefix Purge:
   POST /purge
   { "prefix": "/videos/live/*" }
   → All URLs matching prefix cleared

4. Full Purge:
   POST /purge/all
   → Entire cache cleared (use rarely!)
```

### Implementation:
```javascript
class CDNPurger {
    constructor(apiKey, zoneId) {
        this.apiKey = apiKey;
        this.zoneId = zoneId;
        this.baseUrl = 'https://api.cloudflare.com/client/v4';
    }
    
    async purgeUrls(urls) {
        const response = await fetch(
            `${this.baseUrl}/zones/${this.zoneId}/purge_cache`,
            {
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${this.apiKey}`,
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({ files: urls })
            }
        );
        return response.json();
    }
    
    async purgeByTag(tags) {
        const response = await fetch(
            `${this.baseUrl}/zones/${this.zoneId}/purge_cache`,
            {
                method: 'POST',
                headers: {
                    'Authorization': `Bearer ${this.apiKey}`,
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({ tags })
            }
        );
        return response.json();
    }
}

// Usage
const purger = new CDNPurger(process.env.CF_API_KEY, process.env.CF_ZONE_ID);

// When video metadata updates
await purger.purgeUrls([
    'https://youtube.com/video/abc123',
    'https://youtube.com/api/video/abc123'
]);

// When user updates profile
await purger.purgeByTag(['user:456']);
```

---

## YouTube CDN Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         YouTube Global CDN                               │
└─────────────────────────────────────────────────────────────────────────┘

User Request: "Watch video abc123"
                │
                ▼
        ┌───────────────┐
        │     DNS       │
        │   (Anycast)   │
        └───────┬───────┘
                │ Resolves to nearest edge
                ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        Edge PoP (User's Region)                         │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐            │
│  │   L1 Cache   │    │   L1 Cache   │    │   L1 Cache   │            │
│  │  (Hot data)  │    │  (Hot data)  │    │  (Hot data)  │            │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘            │
│         │                   │                   │                     │
│         └───────────────────┼───────────────────┘                     │
│                             │                                          │
│                    ┌────────┴────────┐                                │
│                    │    L2 Cache     │                                │
│                    │   (Regional)    │                                │
│                    └────────┬────────┘                                │
│                             │                                          │
└─────────────────────────────┼──────────────────────────────────────────┘
                              │ Cache miss
                              ▼
                    ┌──────────────────┐
                    │  Origin Shield   │
                    │  (Super PoP)     │
                    └────────┬─────────┘
                             │ Shield miss
                             ▼
                    ┌──────────────────┐
                    │  Origin Servers  │
                    │  (Video Storage) │
                    └──────────────────┘

Caching Strategy:
- L1: Hot videos (viral, trending) - Seconds latency
- L2: Warm videos (popular) - ~50ms latency
- Shield: All requested videos - ~100ms latency
- Origin: All videos - ~200ms latency

Hit rates:
- L1 hit: 60% of requests
- L2 hit: 30% of requests
- Shield hit: 8% of requests
- Origin: 2% of requests
```

---

## Edge Computing

### Run Code at the Edge:
```javascript
// Cloudflare Worker - Personalization at edge
addEventListener('fetch', event => {
    event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
    const url = new URL(request.url);
    
    // Get user's country from CDN
    const country = request.headers.get('CF-IPCountry');
    
    // Customize response based on location
    if (url.pathname === '/api/trending') {
        // Fetch trending videos for user's country
        const trending = await fetch(
            `https://origin.youtube.com/api/trending?country=${country}`
        );
        
        // Cache by country
        const response = new Response(trending.body, trending);
        response.headers.set('Cache-Control', 'max-age=300');
        response.headers.set('Vary', 'CF-IPCountry');
        
        return response;
    }
    
    return fetch(request);
}
```

### Use Cases:
```
1. A/B Testing at Edge
   - Route users to different versions
   - No origin round-trip

2. Authentication at Edge
   - Validate JWT at edge
   - Block unauthorized before origin

3. Image Transformation
   - Resize/format on the fly
   - Device-specific optimization

4. Geolocation Routing
   - Country-specific content
   - Compliance (GDPR, etc.)
```

---

## Interview Questions

**Q: What is a CDN?**
A: Network of globally distributed servers that cache and deliver content from locations closest to users, reducing latency and origin load.

**Q: How does CDN caching work?**
A: User requests content, DNS routes to nearest edge server. If cached (HIT), return immediately. If not (MISS), fetch from origin, cache, return.

**Q: What is cache invalidation in CDN?**
A: Removing or updating cached content before it expires. Methods: URL purge, tag purge, prefix purge, or full purge.

**Q: How would you design video streaming with CDN?**
A: Use adaptive bitrate streaming (HLS/DASH), encode at multiple qualities, split into small chunks, cache all chunks at edge, client requests based on bandwidth.

**Q: What is edge computing?**
A: Running code at CDN edge servers, close to users. Used for personalization, A/B testing, authentication without origin round-trips.

---

## Quick Summary

```
CDN (Content Delivery Network):
───────────────────────────────
- Distributed network of edge servers
- Caches content close to users
- Reduces latency and origin load

HOW IT WORKS:
─────────────
User → DNS → Nearest Edge → Cache HIT? → Return
                 │
                 └── MISS → Origin → Cache → Return

CACHE CONTROL:
──────────────
Cache-Control: public, max-age=3600
Cache-Control: no-cache (revalidate)
Cache-Control: no-store (never cache)

FEATURES:
─────────
- Static/dynamic caching
- DDoS protection
- SSL termination
- Compression
- Image optimization
- Edge computing

VIDEO STREAMING:
────────────────
- HLS/DASH adaptive streaming
- Multiple quality encodings
- Small chunk segments
- Client adapts to bandwidth

INVALIDATION:
─────────────
- URL purge (specific)
- Tag purge (grouped)
- Prefix purge (pattern)
- Full purge (everything)
```

You now understand CDN! 🌐
