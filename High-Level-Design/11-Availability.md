# Availability - Complete Guide

## What is Availability?

Think of Availability like a **hospital emergency room**:
- ER is ALWAYS open
- Doctors work in shifts
- If one doctor is sick, another takes over
- Never says "We're closed, come back tomorrow"

**Simple Definition:**
Availability = System is operational and accessible when users need it.

---

## Measuring Availability

### The "Nines" System:
```
Availability    Downtime/Year    Downtime/Month    Downtime/Week
────────────────────────────────────────────────────────────────
90%   (1 nine)   36.5 days        3 days            16.8 hours
99%   (2 nines)  3.65 days        7.2 hours         1.68 hours
99.9% (3 nines)  8.76 hours       43.2 minutes      10.1 minutes
99.99% (4 nines) 52.6 minutes     4.32 minutes      1.01 minutes
99.999% (5 nines) 5.26 minutes    25.9 seconds      6.05 seconds
```

### Real-World Targets:
```
Service                  Typical Availability
──────────────────────────────────────────────
Personal blog            99% (acceptable)
E-commerce site          99.9% (important)
Banking system           99.99% (critical)
Cloud infrastructure     99.99%+ (essential)
Emergency services       99.999% (life-critical)
```

---

## Availability Formula

```
            Uptime
Availability = ─────────────────── × 100%
            Uptime + Downtime

Example:
System up for 720 hours in a month
Down for 1 hour

Availability = 720/(720+1) × 100% = 99.86%
```

---

## What Causes Downtime?

### 1. Hardware Failures
```
- Server crash
- Disk failure
- Network card dies
- Power outage

Solution: Redundancy (multiple servers)
```

### 2. Software Bugs
```
- Application crash
- Memory leak
- Deadlock

Solution: Testing, monitoring, auto-restart
```

### 3. Network Issues
```
- ISP outage
- DNS failure
- DDoS attack

Solution: Multiple paths, CDN, DDoS protection
```

### 4. Human Error
```
- Bad deployment
- Misconfiguration
- Accidental deletion

Solution: Automation, testing, backups
```

### 5. Planned Maintenance
```
- Updates
- Scaling
- Migration

Solution: Rolling updates, blue-green deployment
```

---

## High Availability Patterns

### Pattern 1: Redundancy (No Single Point of Failure)
```
BAD - Single Point of Failure:
┌─────────┐     ┌─────────┐     ┌─────────┐
│ Users   │────>│ Server  │────>│   DB    │
└─────────┘     └─────────┘     └─────────┘
                     ↑
              If this dies,
              everything stops!

GOOD - Redundancy:
┌─────────┐     ┌──────────────┐     ┌─────────────┐
│ Users   │────>│Load Balancer │────>│ DB Primary  │
└─────────┘     │   (Active)   │     │    +        │
                │      +       │     │ DB Replica  │
                │   (Standby)  │     └─────────────┘
                └───────┬──────┘
                        │
               ┌────────┴────────┐
               ▼                 ▼
          ┌─────────┐       ┌─────────┐
          │Server 1 │       │Server 2 │
          └─────────┘       └─────────┘

Every component has a backup!
```

### Pattern 2: Active-Passive (Failover)
```
Normal operation:
┌──────────────────────────────────────┐
│                                      │
│   Active Server ◄─── All traffic    │
│        │                             │
│        │ (heartbeat)                 │
│        ▼                             │
│   Passive Server (standby)           │
│                                      │
└──────────────────────────────────────┘

When Active fails:
┌──────────────────────────────────────┐
│                                      │
│   Active Server ✗ (dead)            │
│                                      │
│   Passive Server ──► Becomes Active! │
│        ▲                             │
│        │                             │
│   All traffic                        │
│                                      │
└──────────────────────────────────────┘

Pros: Simple, guaranteed resources
Cons: Passive server sits idle (expensive)
```

### Pattern 3: Active-Active
```
┌──────────────────────────────────────┐
│                                      │
│        ┌────────────────┐            │
│        │  Load Balancer │            │
│        └────────┬───────┘            │
│                 │                    │
│     ┌───────────┴───────────┐        │
│     ▼                       ▼        │
│ ┌─────────┐           ┌─────────┐    │
│ │Server 1 │           │Server 2 │    │
│ │(Active) │           │(Active) │    │
│ │  50%    │           │  50%    │    │
│ └─────────┘           └─────────┘    │
│                                      │
└──────────────────────────────────────┘

If one fails:
┌──────────────────────────────────────┐
│                                      │
│        ┌────────────────┐            │
│        │  Load Balancer │            │
│        └────────┬───────┘            │
│                 │                    │
│                 ▼                    │
│ ┌─────────┐           ┌─────────┐    │
│ │Server 1 │           │Server 2 │    │
│ │(Active) │           │  DEAD   │    │
│ │  100%   │           │    ✗    │    │
│ └─────────┘           └─────────┘    │
│                                      │
└──────────────────────────────────────┘

Pros: Better resource utilization, handles more traffic
Cons: More complex, need to handle state
```

### Pattern 4: Multi-Region
```
┌─────────────────────────────────────────────────────────┐
│                      Global DNS                         │
│              (Routes to nearest/healthiest)             │
└─────────────────────────┬───────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   ┌─────────┐       ┌─────────┐       ┌─────────┐
   │ US-EAST │       │ EU-WEST │       │  ASIA   │
   │ Region  │       │ Region  │       │ Region  │
   └─────────┘       └─────────┘       └─────────┘

US-EAST goes down?
Traffic routes to EU-WEST and ASIA
Users don't even notice!
```

---

## Health Checks

### How to Know Something is Down?

```javascript
// Simple health check endpoint
app.get('/health', async (req, res) => {
    try {
        // Check database
        await db.query('SELECT 1');
        
        // Check cache
        await redis.ping();
        
        // Check external services
        await checkExternalAPI();
        
        res.status(200).json({ status: 'healthy' });
    } catch (error) {
        res.status(503).json({ 
            status: 'unhealthy',
            error: error.message 
        });
    }
});
```

### Health Check Types:
```
1. Liveness Check: "Is the process running?"
   - Simple ping
   - Process exists

2. Readiness Check: "Can it handle requests?"
   - Database connected
   - Dependencies healthy
   - Warmed up

3. Startup Check: "Is it done starting?"
   - Initialization complete
   - Migrations done
```

### Kubernetes Health Checks:
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    livenessProbe:
      httpGet:
        path: /health/live
        port: 3000
      initialDelaySeconds: 10
      periodSeconds: 5
      
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 3
```

---

## Failover Strategies

### 1. DNS Failover
```
Normal:
youtube.com → 1.2.3.4 (Primary)

Primary down:
youtube.com → 5.6.7.8 (Secondary)

Pros: Simple
Cons: DNS propagation delay (minutes)
```

### 2. Load Balancer Failover
```
Load balancer health checks all servers:

Server 1: ✓ Healthy
Server 2: ✓ Healthy  
Server 3: ✗ Unhealthy → Removed from pool

Instant! (seconds)
```

### 3. Database Failover
```
                 ┌─── App Server
                 ▼
┌─────────────────────────────────────────┐
│              Connection Pool            │
└─────────────────────┬───────────────────┘
                      │
           ┌──────────┴──────────┐
           ▼                     ▼
    ┌─────────────┐       ┌─────────────┐
    │   Primary   │──────>│   Replica   │
    │  (Writes)   │ sync  │  (Reads)    │
    └─────────────┘       └─────────────┘

Primary fails?
Replica promoted to Primary!

Automatic with tools like:
- PostgreSQL with Patroni
- MySQL with orchestrator
- MongoDB replica sets
```

---

## Zero Downtime Deployments

### 1. Rolling Update
```
Current: v1 v1 v1 v1 (4 pods)

Step 1:  v1 v1 v1 v2 (replace 1)
Step 2:  v1 v1 v2 v2 (replace 1 more)
Step 3:  v1 v2 v2 v2 (replace 1 more)
Step 4:  v2 v2 v2 v2 (done!)

Traffic never stops!
```

### 2. Blue-Green Deployment
```
Current (Blue): Running v1, serving traffic
New (Green): Deploy v2, test it

         ┌─────────────────┐
         │  Load Balancer  │
         └────────┬────────┘
                  │ 100%
                  ▼
    ┌─────────────────────────┐
    │   BLUE (v1) - Production│
    └─────────────────────────┘
    
    ┌─────────────────────────┐
    │   GREEN (v2) - Staging  │ ← Testing
    └─────────────────────────┘

Switch when ready:
         ┌─────────────────┐
         │  Load Balancer  │
         └────────┬────────┘
                  │ 100%
                  ▼
    ┌─────────────────────────┐
    │   GREEN (v2) - Now Live!│
    └─────────────────────────┘

Rollback = Switch back to Blue instantly!
```

### 3. Canary Deployment
```
Gradually shift traffic:

Step 1:  95% to v1, 5% to v2  (test with 5%)
Step 2:  80% to v1, 20% to v2 (looking good!)
Step 3:  50% to v1, 50% to v2 (more confidence)
Step 4:  0% to v1, 100% to v2 (full rollout)

If v2 has errors → Roll back immediately
```

---

## Circuit Breaker Pattern

### Problem:
```
Your Service → Failing Service (timeout 30s)
                     ↓
              Every request waits 30s
              Users angry! 😠
```

### Solution: Circuit Breaker
```javascript
class CircuitBreaker {
    constructor(options = {}) {
        this.failureThreshold = options.failureThreshold || 5;
        this.resetTimeout = options.resetTimeout || 30000;
        this.failureCount = 0;
        this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
        this.lastFailure = null;
    }
    
    async call(fn) {
        if (this.state === 'OPEN') {
            // Check if we should try again
            if (Date.now() - this.lastFailure > this.resetTimeout) {
                this.state = 'HALF_OPEN';
            } else {
                throw new Error('Circuit breaker is OPEN');
            }
        }
        
        try {
            const result = await fn();
            this.onSuccess();
            return result;
        } catch (error) {
            this.onFailure();
            throw error;
        }
    }
    
    onSuccess() {
        this.failureCount = 0;
        this.state = 'CLOSED';
    }
    
    onFailure() {
        this.failureCount++;
        this.lastFailure = Date.now();
        
        if (this.failureCount >= this.failureThreshold) {
            this.state = 'OPEN';
            console.log('Circuit breaker OPENED!');
        }
    }
}

// Usage
const breaker = new CircuitBreaker({ failureThreshold: 3 });

async function callExternalService() {
    return breaker.call(async () => {
        return await fetch('https://external-service.com/api');
    });
}
```

### States:
```
CLOSED: Normal operation, requests go through
        ↓ (failures exceed threshold)
OPEN:   Requests fail immediately (fast failure)
        ↓ (after timeout period)
HALF_OPEN: Allow one request to test
        ↓ (success → CLOSED, failure → OPEN)
```

---

## Graceful Degradation

### Provide Reduced Functionality Instead of Failing

```javascript
async function getVideoRecommendations(userId) {
    try {
        // Try ML recommendation service
        return await mlService.getPersonalized(userId);
    } catch (error) {
        console.error('ML service failed, using fallback');
        
        try {
            // Fallback: Get popular videos
            return await getPopularVideos();
        } catch (error) {
            console.error('Popular videos failed, using cache');
            
            // Last resort: Return cached generic list
            return await getStaticFallbackList();
        }
    }
}
```

### YouTube Example:
```
Normal: Personalized recommendations + comments + likes + full UI

Degraded (some services down):
- No personalized recommendations → Show trending
- Comments service down → Hide comments section
- Like count unavailable → Show video anyway
- HD not available → Show SD version

User sees reduced experience, not error page!
```

---

## High Availability for YouTube Clone

### Architecture:
```
┌─────────────────────────────────────────────────────────────────┐
│                         GLOBAL DNS                               │
│                    (Route 53 / CloudFlare)                       │
│                 Geo-routing + Health checks                      │
└───────────────────────────┬─────────────────────────────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         ▼                  ▼                  ▼
    ┌─────────┐        ┌─────────┐        ┌─────────┐
    │ US-EAST │        │ EU-WEST │        │  ASIA   │
    └────┬────┘        └────┬────┘        └────┬────┘
         │                  │                  │
         │    Each region has:                 │
         │                                     │
         │    ┌────────────────────────────┐  │
         │    │   CDN Edge Servers         │  │
         │    │   (Cached video content)   │  │
         │    └────────────────────────────┘  │
         │                │                    │
         │    ┌────────────────────────────┐  │
         │    │   Load Balancer (HA pair)  │  │
         │    └────────────────────────────┘  │
         │                │                    │
         │    ┌───────────┴───────────┐       │
         │    ▼                       ▼       │
         │  ┌──────┐               ┌──────┐   │
         │  │App 1 │...           │App N │   │
         │  └──────┘               └──────┘   │
         │         │                          │
         │    ┌────────────────────────────┐  │
         │    │   Redis Cluster (HA)       │  │
         │    └────────────────────────────┘  │
         │                │                    │
         │    ┌────────────────────────────┐  │
         │    │   Database Cluster         │  │
         │    │   Primary + Replicas       │  │
         │    │   Auto-failover            │  │
         │    └────────────────────────────┘  │
         │                                     │
         └─────────────────────────────────────┘

Cross-region replication for disaster recovery
```

### Availability Calculation:
```
Component Availabilities:
- Load Balancer: 99.99% (with HA pair)
- App Servers: 99.9% (multiple instances)
- Database: 99.99% (with replicas)
- Cache: 99.99% (Redis cluster)

Combined (series):
99.99% × 99.9% × 99.99% × 99.99% = 99.87%

With 3 regions (parallel):
1 - (0.0013)^3 = 99.9999998%

Three nines to eight nines with multi-region!
```

---

## Monitoring and Alerting

```javascript
// Key metrics to monitor
const healthMetrics = {
    // Availability indicators
    error_rate: {
        threshold: 1,   // Alert if > 1%
        critical: 5     // Page if > 5%
    },
    
    latency_p99: {
        threshold: 500, // Alert if > 500ms
        critical: 2000  // Page if > 2s
    },
    
    // Saturation
    cpu_usage: {
        threshold: 70,
        critical: 90
    },
    
    memory_usage: {
        threshold: 80,
        critical: 95
    },
    
    // Health check
    health_check_failures: {
        threshold: 1,
        critical: 3
    }
};

// Alert rules
if (error_rate > 1%) {
    sendSlackAlert('High error rate detected');
}

if (error_rate > 5%) {
    pageOnCall('CRITICAL: Error rate above 5%');
}
```

---

## Interview Questions

**Q: What is high availability?**
A: System's ability to remain operational and accessible. Measured in "nines" - 99.9% means ~8.7 hours downtime per year.

**Q: What's the difference between Active-Active and Active-Passive?**
A: Active-Active: All nodes handle traffic simultaneously. Active-Passive: Standby nodes only activate when primary fails.

**Q: How do you achieve zero downtime deployments?**
A: Rolling updates, blue-green deployments, or canary releases. Each gradually shifts traffic to new version while keeping old version running.

**Q: What is a circuit breaker pattern?**
A: A pattern that prevents cascading failures by stopping requests to a failing service. Opens after threshold failures, allows retry after timeout.

**Q: How do you design for 99.99% availability?**
A: No single points of failure, multiple regions, auto-failover, health checks, redundancy at every layer, graceful degradation.

---

## Quick Summary

```
AVAILABILITY = System uptime percentage

The Nines:
- 99% = 3.65 days downtime/year
- 99.9% = 8.76 hours downtime/year
- 99.99% = 52.6 minutes downtime/year

Patterns:
- Redundancy (no SPOF)
- Active-Passive (failover)
- Active-Active (load sharing)
- Multi-region (geographic)

Key Practices:
- Health checks
- Auto-failover
- Zero-downtime deployments
- Circuit breakers
- Graceful degradation
- Monitoring & alerting

Formula:
Availability = Uptime / (Uptime + Downtime)
```

You now understand Availability like a pro! 🚀
