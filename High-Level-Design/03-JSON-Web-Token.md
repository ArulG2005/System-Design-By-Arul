# JSON Web Token (JWT) - Complete Guide

## What is JWT?

JWT = JSON Web Token

Think of JWT like a **movie ticket**:
- You buy ticket (login)
- Ticket has your details (name, seat, movie)
- You show ticket to enter (authentication)
- Guard doesn't call box office to verify (stateless)
- Ticket works until movie ends (expiration)

**Simple Definition:**
JWT is a secure way to send information between two parties that can be verified and trusted.

---

## Why JWT?

### Old Way (Sessions):
```
1. User logs in
2. Server creates session in database
3. Server sends session ID in cookie
4. Every request: Server checks database for session
5. Problem: Database hit on EVERY request!

Server 1 ──> Session DB <── Server 2
            (bottleneck!)
```

### New Way (JWT):
```
1. User logs in
2. Server creates JWT (stores nothing!)
3. Server sends JWT to client
4. Every request: Client sends JWT
5. Server verifies JWT signature (no database!)

Server 1 (verifies JWT independently)
Server 2 (verifies JWT independently)
Server 3 (verifies JWT independently)
```

---

## JWT Structure

JWT has 3 parts separated by dots (.)

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoiMTIzIiwibmFtZSI6IkpvaG4iLCJpYXQiOjE2NDIyMzQwMDB9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

Part 1: Header (Red)
Part 2: Payload (Purple)  
Part 3: Signature (Blue)
```

### Part 1: Header
```json
{
    "alg": "HS256",    // Algorithm used to sign
    "typ": "JWT"       // Type of token
}
```
Base64 encoded → `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9`

### Part 2: Payload (Claims)
```json
{
    "user_id": "123",
    "name": "John Doe",
    "email": "john@example.com",
    "role": "admin",
    "iat": 1642234000,     // Issued at
    "exp": 1642237600      // Expires at
}
```
Base64 encoded → `eyJ1c2VyX2lkIjoiMTIzIiwibmFtZ...`

### Part 3: Signature
```
HMACSHA256(
    base64(header) + "." + base64(payload),
    secret_key
)
```
This makes JWT tamper-proof!

---

## How JWT Works - Step by Step

### Login Process:
```
1. User sends: POST /login
   {
       "email": "john@example.com",
       "password": "secret123"
   }

2. Server checks credentials in database
   ✓ Valid!

3. Server creates JWT:
   {
       "user_id": "123",
       "role": "user",
       "exp": "1 hour from now"
   }

4. Server signs JWT with SECRET KEY

5. Server sends JWT to client:
   {
       "token": "eyJhbGciOiJIUzI1NiIs...",
       "expires_in": 3600
   }

6. Client stores JWT (localStorage/cookie)
```

### Making Authenticated Requests:
```
1. Client sends request:
   GET /api/my-profile
   Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

2. Server extracts JWT from header

3. Server verifies signature using SECRET KEY
   ✓ Signature valid!

4. Server checks expiration
   ✓ Not expired!

5. Server extracts user_id from payload

6. Server returns user data
```

---

## JWT Claims Explained

### Standard Claims:
```json
{
    "iss": "https://myapp.com",     // Issuer - who created token
    "sub": "user123",               // Subject - who token is about
    "aud": "https://api.myapp.com", // Audience - who should accept
    "exp": 1642237600,              // Expiration - when token dies
    "nbf": 1642234000,              // Not Before - token not valid before
    "iat": 1642234000,              // Issued At - when created
    "jti": "unique-token-id"        // JWT ID - unique identifier
}
```

### Custom Claims (Your Data):
```json
{
    "user_id": "123",
    "email": "john@example.com",
    "role": "admin",
    "permissions": ["read", "write", "delete"],
    "subscription": "premium"
}
```

---

## JWT Implementation

### Node.js Example:

```javascript
const jwt = require('jsonwebtoken');

const SECRET_KEY = 'your-super-secret-key-keep-it-safe';

// Generate JWT (Login)
function generateToken(user) {
    const payload = {
        user_id: user.id,
        email: user.email,
        role: user.role
    };
    
    const options = {
        expiresIn: '1h',        // Expires in 1 hour
        issuer: 'my-app'
    };
    
    return jwt.sign(payload, SECRET_KEY, options);
}

// Verify JWT (Middleware)
function verifyToken(req, res, next) {
    // Get token from header
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
        return res.status(401).json({ error: 'No token provided' });
    }
    
    const token = authHeader.split(' ')[1];
    
    try {
        // Verify and decode
        const decoded = jwt.verify(token, SECRET_KEY);
        req.user = decoded;
        next();
    } catch (error) {
        if (error.name === 'TokenExpiredError') {
            return res.status(401).json({ error: 'Token expired' });
        }
        return res.status(401).json({ error: 'Invalid token' });
    }
}

// Usage
app.post('/login', async (req, res) => {
    const { email, password } = req.body;
    
    // Check credentials
    const user = await User.findOne({ email });
    if (!user || !await bcrypt.compare(password, user.password)) {
        return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    // Generate token
    const token = generateToken(user);
    
    res.json({
        token,
        expires_in: 3600,
        user: {
            id: user.id,
            email: user.email,
            name: user.name
        }
    });
});

// Protected route
app.get('/api/profile', verifyToken, (req, res) => {
    // req.user contains decoded JWT payload
    res.json({
        user_id: req.user.user_id,
        email: req.user.email
    });
});
```

---

## Access Token vs Refresh Token

### Problem:
```
- Short-lived token (15 min) = User logs in every 15 minutes 😫
- Long-lived token (30 days) = Risky if token is stolen 😰
```

### Solution: Use Both!
```
Access Token:
- Short-lived (15 minutes)
- Used for API requests
- Stored in memory

Refresh Token:
- Long-lived (7-30 days)
- Only used to get new access token
- Stored in httpOnly cookie
```

### Flow:
```
1. Login → Get access token + refresh token

2. Use access token for requests
   GET /api/videos
   Authorization: Bearer <access_token>

3. Access token expires (15 min later)
   API returns: 401 Unauthorized

4. Client uses refresh token to get new access token
   POST /refresh
   Cookie: refresh_token=xyz

5. Server validates refresh token
   Returns new access token

6. Continue using new access token
```

### Implementation:
```javascript
// Login - Issue both tokens
app.post('/login', async (req, res) => {
    const user = await validateCredentials(req.body);
    
    // Access token - short lived
    const accessToken = jwt.sign(
        { user_id: user.id },
        ACCESS_SECRET,
        { expiresIn: '15m' }
    );
    
    // Refresh token - long lived
    const refreshToken = jwt.sign(
        { user_id: user.id },
        REFRESH_SECRET,
        { expiresIn: '7d' }
    );
    
    // Store refresh token in database (for revocation)
    await saveRefreshToken(user.id, refreshToken);
    
    // Send refresh token as httpOnly cookie
    res.cookie('refresh_token', refreshToken, {
        httpOnly: true,
        secure: true,
        sameSite: 'strict',
        maxAge: 7 * 24 * 60 * 60 * 1000 // 7 days
    });
    
    res.json({ accessToken });
});

// Refresh endpoint
app.post('/refresh', async (req, res) => {
    const refreshToken = req.cookies.refresh_token;
    
    if (!refreshToken) {
        return res.status(401).json({ error: 'No refresh token' });
    }
    
    try {
        // Verify refresh token
        const decoded = jwt.verify(refreshToken, REFRESH_SECRET);
        
        // Check if token is in database (not revoked)
        const isValid = await isRefreshTokenValid(decoded.user_id, refreshToken);
        if (!isValid) {
            return res.status(401).json({ error: 'Token revoked' });
        }
        
        // Issue new access token
        const accessToken = jwt.sign(
            { user_id: decoded.user_id },
            ACCESS_SECRET,
            { expiresIn: '15m' }
        );
        
        res.json({ accessToken });
    } catch (error) {
        res.status(401).json({ error: 'Invalid refresh token' });
    }
});

// Logout - Revoke refresh token
app.post('/logout', async (req, res) => {
    const refreshToken = req.cookies.refresh_token;
    
    // Remove from database
    await revokeRefreshToken(refreshToken);
    
    // Clear cookie
    res.clearCookie('refresh_token');
    
    res.json({ message: 'Logged out' });
});
```

---

## JWT Security Best Practices

### 1. Use Strong Secret Keys
```javascript
// Bad
const SECRET = 'secret123';

// Good - Use long random string
const SECRET = 'a8f5f167f44f4964e6c998dee827110c5b5...(64+ chars)';

// Best - Use environment variable
const SECRET = process.env.JWT_SECRET;
```

### 2. Always Use HTTPS
```
HTTP:  Token can be intercepted! 😱
HTTPS: Token is encrypted in transit ✓
```

### 3. Set Short Expiration
```javascript
// Access token: 15 minutes
jwt.sign(payload, secret, { expiresIn: '15m' });

// Refresh token: 7 days
jwt.sign(payload, secret, { expiresIn: '7d' });
```

### 4. Don't Store Sensitive Data
```javascript
// Bad - Anyone can decode and see!
{
    "password": "secret123",
    "credit_card": "1234-5678-9012"
}

// Good
{
    "user_id": "123",
    "role": "user"
}
```

### 5. Use Proper Algorithm
```javascript
// Bad - No signature
{ "alg": "none" }

// Good
{ "alg": "HS256" }  // HMAC with SHA-256

// Better (for distributed systems)
{ "alg": "RS256" }  // RSA with SHA-256
```

### 6. Validate All Claims
```javascript
jwt.verify(token, SECRET, {
    algorithms: ['HS256'],      // Prevent algorithm switching
    issuer: 'my-app',           // Verify issuer
    audience: 'my-api'          // Verify audience
});
```

---

## JWT for YouTube Clone

### Token Structure:
```json
{
    "user_id": "user_abc123",
    "email": "creator@youtube.com",
    "channel_id": "channel_xyz789",
    "role": "creator",
    "subscription": "premium",
    "permissions": [
        "upload_video",
        "delete_own_video",
        "monetize"
    ],
    "iat": 1642234000,
    "exp": 1642237600
}
```

### Auth Middleware:
```javascript
// Check if user can upload videos
const canUpload = (req, res, next) => {
    if (!req.user.permissions.includes('upload_video')) {
        return res.status(403).json({ 
            error: 'You need a channel to upload videos' 
        });
    }
    next();
};

// Check if user owns the video
const ownsVideo = async (req, res, next) => {
    const video = await Video.findById(req.params.videoId);
    
    if (video.channel_id !== req.user.channel_id) {
        return res.status(403).json({ 
            error: 'You can only edit your own videos' 
        });
    }
    next();
};

// Routes
app.post('/api/videos', verifyToken, canUpload, uploadVideo);
app.delete('/api/videos/:videoId', verifyToken, ownsVideo, deleteVideo);
```

---

## Common JWT Attacks & Prevention

### 1. Token Stealing (XSS)
```
Attack: Hacker injects script, steals token from localStorage

Prevention:
- Store in httpOnly cookie (can't be accessed by JavaScript)
- Use Content Security Policy (CSP)
- Sanitize all inputs
```

### 2. Algorithm Confusion
```
Attack: Change algorithm to 'none' or use public key as HMAC secret

Prevention:
jwt.verify(token, SECRET, {
    algorithms: ['HS256']  // Only allow specific algorithm
});
```

### 3. Brute Force
```
Attack: Try to guess the secret key

Prevention:
- Use 256+ bit random secret
- Use asymmetric keys (RS256)
```

---

## JWT vs Sessions - When to Use What?

### Use JWT When:
```
✓ Multiple servers (microservices)
✓ Mobile app authentication
✓ Third-party API access
✓ Stateless architecture
✓ Cross-domain authentication
```

### Use Sessions When:
```
✓ Single server application
✓ Need to revoke access immediately
✓ Sensitive applications (banking)
✓ Server-rendered websites
```

---

## Quick Reference

### Generate Token:
```javascript
jwt.sign({ user_id: '123' }, SECRET, { expiresIn: '1h' });
```

### Verify Token:
```javascript
const decoded = jwt.verify(token, SECRET);
```

### Decode Without Verify:
```javascript
const decoded = jwt.decode(token);  // Don't use for authentication!
```

---

## Interview Questions

**Q: How is JWT secure if anyone can decode it?**
A: JWT is not encrypted, it's signed. Anyone can read it, but they can't modify it because they don't have the secret key to create a valid signature.

**Q: How to revoke a JWT?**
A: Use short expiration + refresh tokens stored in database. Delete refresh token from database to revoke.

**Q: Where to store JWT?**
A: Access token in memory, refresh token in httpOnly cookie.

**Q: JWT vs OAuth?**
A: JWT is a token format. OAuth is an authorization protocol that can use JWT as its token format.

---

You now understand JWT like a pro! 🚀
