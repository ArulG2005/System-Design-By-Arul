# Webhooks - Complete Guide

## What is a Webhook?

Think of Webhook like a **doorbell**:
- Without doorbell: Keep checking if someone is at door (polling)
- With doorbell: Door rings when someone arrives (webhook)

**Simple Definition:**
Webhook is a way for one application to send real-time data to another application when something happens.

---

## Webhook vs API (Pull vs Push)

### Traditional API (Polling - YOU ask):
```
You: "Any new orders?"
Server: "No"
(5 seconds later)
You: "Any new orders?"
Server: "No"
(5 seconds later)
You: "Any new orders?"
Server: "Yes! Here's order #123"

Problem: Wasted 100 requests just to get 1 update!
```

### Webhook (Push - SERVER tells you):
```
You: "Tell me when there's a new order"
Server: "OK, I'll call you"
(Server does its thing...)
(New order comes in)
Server: "Hey! New order #123"

Benefit: Only 1 request when something happens!
```

---

## How Webhooks Work

### Step-by-Step:
```
1. You give your URL to the service
   "Call me at https://myapp.com/webhook/payment"

2. Service saves your URL

3. Something happens (payment received)

4. Service sends POST request to your URL:
   POST https://myapp.com/webhook/payment
   {
       "event": "payment.success",
       "data": {
           "amount": 100,
           "customer": "john@example.com"
       }
   }

5. Your server processes the event

6. Your server responds: 200 OK
```

### Visual Flow:
```
┌─────────────┐                    ┌─────────────┐
│   Stripe    │                    │  Your App   │
│  (Payment   │                    │  (Server)   │
│   Gateway)  │                    │             │
└──────┬──────┘                    └──────┬──────┘
       │                                  │
       │  1. Register webhook URL         │
       │<─────────────────────────────────│
       │     "https://myapp.com/webhook"  │
       │                                  │
       │  2. Customer pays $100           │
       │                                  │
       │  3. Payment successful!          │
       │                                  │
       │  4. POST to your webhook URL     │
       │─────────────────────────────────>│
       │  { event: "payment.success" }    │
       │                                  │
       │  5. You process & save to DB     │
       │                                  │
       │  6. Return 200 OK                │
       │<─────────────────────────────────│
       │                                  │
```

---

## Real-World Webhook Examples

### 1. Payment Notifications (Stripe)
```json
{
    "type": "payment_intent.succeeded",
    "data": {
        "object": {
            "id": "pi_123",
            "amount": 10000,
            "currency": "usd",
            "customer": "cus_abc"
        }
    }
}
```

### 2. GitHub - Code Pushed
```json
{
    "action": "push",
    "repository": {
        "name": "my-project",
        "full_name": "john/my-project"
    },
    "commits": [
        {
            "message": "Fix bug",
            "author": "john@example.com"
        }
    ]
}
```

### 3. YouTube - New Video Published
```json
{
    "event": "video.published",
    "video": {
        "id": "abc123",
        "title": "My New Video",
        "channel_id": "UC_xyz"
    },
    "published_at": "2024-01-15T10:30:00Z"
}
```

---

## Building a Webhook Receiver

### Simple Node.js Example:
```javascript
const express = require('express');
const crypto = require('crypto');
const app = express();

// Raw body needed for signature verification
app.use('/webhook', express.raw({ type: 'application/json' }));

// Webhook secret (from Stripe/GitHub/etc)
const WEBHOOK_SECRET = 'whsec_your_secret_here';

// Verify webhook signature
function verifySignature(payload, signature, secret) {
    const expectedSig = crypto
        .createHmac('sha256', secret)
        .update(payload)
        .digest('hex');
    
    return crypto.timingSafeEqual(
        Buffer.from(signature),
        Buffer.from(expectedSig)
    );
}

// Webhook endpoint
app.post('/webhook/payment', async (req, res) => {
    const signature = req.headers['x-webhook-signature'];
    const payload = req.body;
    
    // 1. Verify signature (VERY IMPORTANT!)
    if (!verifySignature(payload, signature, WEBHOOK_SECRET)) {
        console.log('Invalid signature!');
        return res.status(401).send('Invalid signature');
    }
    
    // 2. Parse the event
    const event = JSON.parse(payload);
    
    // 3. Handle different event types
    switch (event.type) {
        case 'payment.success':
            await handlePaymentSuccess(event.data);
            break;
        case 'payment.failed':
            await handlePaymentFailed(event.data);
            break;
        case 'subscription.cancelled':
            await handleSubscriptionCancelled(event.data);
            break;
        default:
            console.log(`Unhandled event: ${event.type}`);
    }
    
    // 4. Respond quickly with 200
    res.status(200).json({ received: true });
});

async function handlePaymentSuccess(data) {
    // Update database
    await Order.update(
        { payment_status: 'paid' },
        { where: { payment_id: data.payment_id } }
    );
    
    // Send confirmation email
    await sendEmail({
        to: data.customer_email,
        subject: 'Payment Received!',
        body: `Your payment of $${data.amount} was successful.`
    });
}

app.listen(3000);
```

---

## Building a Webhook Sender

### Your app sends webhooks to others:
```javascript
const axios = require('axios');
const crypto = require('crypto');

class WebhookSender {
    constructor(secret) {
        this.secret = secret;
    }
    
    // Generate signature
    generateSignature(payload) {
        return crypto
            .createHmac('sha256', this.secret)
            .update(JSON.stringify(payload))
            .digest('hex');
    }
    
    // Send webhook with retry
    async send(url, event, data, retries = 3) {
        const payload = {
            event,
            data,
            timestamp: new Date().toISOString(),
            id: crypto.randomUUID()
        };
        
        const signature = this.generateSignature(payload);
        
        for (let attempt = 1; attempt <= retries; attempt++) {
            try {
                const response = await axios.post(url, payload, {
                    headers: {
                        'Content-Type': 'application/json',
                        'X-Webhook-Signature': signature,
                        'X-Webhook-ID': payload.id
                    },
                    timeout: 30000 // 30 second timeout
                });
                
                if (response.status === 200) {
                    console.log(`Webhook delivered: ${event}`);
                    return { success: true };
                }
            } catch (error) {
                console.log(`Attempt ${attempt} failed: ${error.message}`);
                
                if (attempt < retries) {
                    // Exponential backoff
                    const delay = Math.pow(2, attempt) * 1000;
                    await this.sleep(delay);
                }
            }
        }
        
        // All retries failed - queue for later
        await this.queueForRetry(url, payload);
        return { success: false, queued: true };
    }
    
    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
    
    async queueForRetry(url, payload) {
        // Save to database for background retry
        await WebhookQueue.create({
            url,
            payload,
            status: 'pending',
            next_retry: new Date(Date.now() + 3600000) // 1 hour
        });
    }
}

// Usage
const webhook = new WebhookSender('your-secret-key');

// When video is uploaded
async function onVideoUploaded(video) {
    // Get all subscribers who want this event
    const subscribers = await WebhookSubscription.find({
        event: 'video.uploaded',
        channel_id: video.channel_id
    });
    
    // Send to all subscribers
    for (const sub of subscribers) {
        await webhook.send(sub.url, 'video.uploaded', {
            video_id: video.id,
            title: video.title,
            channel_id: video.channel_id
        });
    }
}
```

---

## Webhook Registration System

### Let users register webhooks:
```javascript
// Database model
const WebhookSubscription = {
    id: 'uuid',
    user_id: 'user who created',
    url: 'https://their-server.com/webhook',
    events: ['video.uploaded', 'comment.created'],
    secret: 'randomly generated',
    is_active: true,
    created_at: 'timestamp'
};

// API to register webhook
app.post('/api/webhooks', authenticate, async (req, res) => {
    const { url, events } = req.body;
    
    // Validate URL
    if (!isValidUrl(url)) {
        return res.status(400).json({ error: 'Invalid URL' });
    }
    
    // Validate events
    const validEvents = ['video.uploaded', 'video.deleted', 
                         'comment.created', 'subscriber.new'];
    const invalidEvents = events.filter(e => !validEvents.includes(e));
    if (invalidEvents.length > 0) {
        return res.status(400).json({ 
            error: `Invalid events: ${invalidEvents.join(', ')}` 
        });
    }
    
    // Generate secret for signature verification
    const secret = crypto.randomBytes(32).toString('hex');
    
    // Create subscription
    const subscription = await WebhookSubscription.create({
        user_id: req.user.id,
        url,
        events,
        secret,
        is_active: true
    });
    
    res.status(201).json({
        id: subscription.id,
        url: subscription.url,
        events: subscription.events,
        secret: secret,  // Show only once!
        message: 'Save this secret! It won\'t be shown again.'
    });
});

// API to list webhooks
app.get('/api/webhooks', authenticate, async (req, res) => {
    const webhooks = await WebhookSubscription.find({
        user_id: req.user.id
    });
    
    res.json({
        webhooks: webhooks.map(w => ({
            id: w.id,
            url: w.url,
            events: w.events,
            is_active: w.is_active,
            created_at: w.created_at
        }))
    });
});

// API to delete webhook
app.delete('/api/webhooks/:id', authenticate, async (req, res) => {
    await WebhookSubscription.deleteOne({
        id: req.params.id,
        user_id: req.user.id
    });
    
    res.json({ message: 'Webhook deleted' });
});
```

---

## Webhook Security

### 1. Signature Verification (MUST DO!)
```
Why? Anyone can send POST request to your webhook URL!

Without verification:
Hacker: POST /webhook { "event": "payment.success", "amount": 1000000 }
Your app: "Yay! Rich! Give them premium!"  😱

With verification:
Hacker: POST /webhook { fake data }
Your app: "Signature doesn't match. Rejected!" ✓
```

### 2. Use HTTPS
```
HTTP:  Data visible to attackers
HTTPS: Data encrypted ✓
```

### 3. Verify Source IP (Optional)
```javascript
const allowedIPs = ['192.168.1.1', '10.0.0.1'];

app.post('/webhook', (req, res) => {
    const clientIP = req.ip;
    
    if (!allowedIPs.includes(clientIP)) {
        return res.status(403).send('IP not allowed');
    }
    
    // Process webhook...
});
```

### 4. Idempotency (Handle Duplicates)
```javascript
// Same webhook might be sent multiple times
// Use webhook ID to detect duplicates

app.post('/webhook', async (req, res) => {
    const webhookId = req.headers['x-webhook-id'];
    
    // Check if already processed
    const exists = await ProcessedWebhook.findOne({ webhook_id: webhookId });
    
    if (exists) {
        console.log('Duplicate webhook, ignoring');
        return res.status(200).json({ received: true });
    }
    
    // Process webhook
    await processWebhook(req.body);
    
    // Mark as processed
    await ProcessedWebhook.create({ webhook_id: webhookId });
    
    res.status(200).json({ received: true });
});
```

---

## Webhook Best Practices

### 1. Respond Quickly (< 5 seconds)
```javascript
// Bad - Long processing blocks response
app.post('/webhook', async (req, res) => {
    await processOrder();        // 10 seconds
    await sendEmail();           // 5 seconds
    await updateInventory();     // 3 seconds
    res.json({ received: true }); // Too late! Sender thinks it failed
});

// Good - Respond immediately, process later
app.post('/webhook', async (req, res) => {
    // Respond immediately
    res.json({ received: true });
    
    // Process in background
    processWebhookAsync(req.body);
});

// Or use a queue
app.post('/webhook', async (req, res) => {
    await queue.add('process-webhook', req.body);
    res.json({ received: true });
});
```

### 2. Implement Retry Logic (Sender Side)
```javascript
// Retry with exponential backoff
const retryDelays = [
    1000,    // 1 second
    5000,    // 5 seconds
    30000,   // 30 seconds
    300000,  // 5 minutes
    3600000  // 1 hour
];

async function sendWithRetry(url, payload, attempt = 0) {
    try {
        await axios.post(url, payload);
    } catch (error) {
        if (attempt < retryDelays.length) {
            await sleep(retryDelays[attempt]);
            return sendWithRetry(url, payload, attempt + 1);
        }
        throw new Error('All retries exhausted');
    }
}
```

### 3. Log Everything
```javascript
app.post('/webhook', async (req, res) => {
    const logEntry = {
        webhook_id: req.headers['x-webhook-id'],
        event: req.body.event,
        received_at: new Date(),
        ip: req.ip,
        headers: req.headers,
        body: req.body
    };
    
    await WebhookLog.create(logEntry);
    
    // Process...
});
```

---

## Webhooks for YouTube Clone

### Events You Might Send:
```javascript
const WEBHOOK_EVENTS = {
    // Video events
    'video.uploaded': 'When a new video is uploaded',
    'video.processed': 'When video processing is complete',
    'video.published': 'When video goes live',
    'video.deleted': 'When video is removed',
    
    // Channel events
    'subscriber.new': 'New subscriber',
    'subscriber.lost': 'Someone unsubscribed',
    'milestone.reached': 'Subscriber milestone (1K, 10K, etc)',
    
    // Engagement events
    'comment.new': 'New comment on video',
    'like.milestone': 'Like milestone reached',
    
    // Monetization
    'payment.received': 'Payment received',
    'payout.processed': 'Creator payout sent'
};
```

### Implementation:
```javascript
// When video finishes processing
async function onVideoProcessed(video) {
    const event = {
        type: 'video.processed',
        data: {
            video_id: video.id,
            title: video.title,
            duration: video.duration,
            thumbnail_url: video.thumbnail_url,
            resolutions: ['360p', '720p', '1080p'],
            processed_at: new Date().toISOString()
        }
    };
    
    // Find all webhooks subscribed to this event
    const webhooks = await WebhookSubscription.find({
        channel_id: video.channel_id,
        events: 'video.processed',
        is_active: true
    });
    
    // Send to all
    for (const webhook of webhooks) {
        await webhookQueue.add('send-webhook', {
            url: webhook.url,
            event: event,
            secret: webhook.secret
        });
    }
}

// Queue processor (runs in background)
webhookQueue.process('send-webhook', async (job) => {
    const { url, event, secret } = job.data;
    
    const signature = generateSignature(event, secret);
    
    try {
        await axios.post(url, event, {
            headers: {
                'Content-Type': 'application/json',
                'X-Webhook-Signature': signature,
                'X-Webhook-ID': event.id,
                'X-Webhook-Timestamp': Date.now()
            },
            timeout: 30000
        });
        
        await WebhookDelivery.create({
            webhook_id: event.id,
            status: 'delivered',
            response_code: 200
        });
    } catch (error) {
        await WebhookDelivery.create({
            webhook_id: event.id,
            status: 'failed',
            error: error.message
        });
        
        throw error; // Re-throw for retry
    }
});
```

---

## Webhook Testing Tools

### 1. RequestBin / Webhook.site
```
Get temporary URL to receive webhooks for testing:
https://webhook.site/unique-id

See all incoming requests in browser
```

### 2. ngrok (Expose local server)
```bash
# Your local server runs on port 3000
ngrok http 3000

# Get public URL
https://abc123.ngrok.io/webhook

# Use this URL to receive webhooks on your local machine
```

### 3. Stripe CLI (For Stripe webhooks)
```bash
stripe listen --forward-to localhost:3000/webhook
```

---

## Interview Questions

**Q: Webhook vs API?**
A: API = You ask for data (pull). Webhook = Server tells you when data changes (push).

**Q: How to secure webhooks?**
A: Use HTTPS, verify signatures, validate IP addresses, and implement idempotency.

**Q: What if webhook delivery fails?**
A: Implement retry with exponential backoff. Queue failed webhooks for later retry.

**Q: How to handle duplicate webhooks?**
A: Use webhook ID to detect and skip duplicates (idempotency).

---

## Quick Summary

```
Webhook = Reverse API

Instead of:
You → Server (asking repeatedly)

It's:
Server → You (notifying when events happen)

Key Components:
1. Webhook URL (where to send)
2. Events (what triggers webhook)
3. Payload (data sent)
4. Signature (security)
5. Retry logic (reliability)
```

You now understand Webhooks like a pro! 🚀
