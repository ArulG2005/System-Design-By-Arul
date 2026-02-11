# Message Queues - Complete Guide

## What is a Message Queue?

Think of it like a **restaurant order system**:
- Customer places order (producer)
- Order goes to ticket queue
- Kitchen picks up orders when ready (consumer)
- Customers don't wait at counter for food

**Simple Definition:**
A message queue is a buffer between senders and receivers that stores messages until they can be processed.

---

## Why Message Queues?

### Without Queue (Synchronous):
```
User uploads video → Process video → Respond

User: "I clicked upload 10 minutes ago... still waiting 😤"

┌────────┐     ┌─────────────────────────────────┐     ┌────────┐
│ Client │────►│        Video Processing         │────►│Response│
│        │     │   (10 minutes of waiting!)      │     │        │
└────────┘     └─────────────────────────────────┘     └────────┘
```

### With Queue (Asynchronous):
```
User uploads video → Queue message → Respond immediately!
                          ↓
                   Process in background

User: "Upload submitted! I can do other things 😊"

┌────────┐     ┌───────┐     ┌─────────────────┐
│ Client │────►│ Queue │     │ Video Processor │
│        │     │       │────►│ (Background)    │
└────────┘     └───────┘     └─────────────────┘
     │
     ▼
"Upload accepted!"
(Immediate response)
```

---

## Core Concepts

### 1. Producers & Consumers
```
┌──────────┐          ┌──────────────┐          ┌──────────┐
│ Producer │──────────│    Queue     │──────────│ Consumer │
│          │  sends   │              │  receives│          │
│ (Sender) │ messages │  [msg][msg]  │ messages │(Receiver)│
└──────────┘          └──────────────┘          └──────────┘

Multiple producers → One queue → Multiple consumers
```

### 2. Message
```
{
    "id": "msg_123",
    "type": "video.uploaded",
    "payload": {
        "video_id": "abc123",
        "user_id": "user_456",
        "file_url": "s3://bucket/video.mp4"
    },
    "metadata": {
        "timestamp": "2024-01-15T10:30:00Z",
        "retry_count": 0
    }
}
```

### 3. Topics/Channels
```
                    ┌─────────────────────────────────┐
                    │          Message Broker          │
                    ├─────────────────────────────────┤
                    │                                 │
Producer A ────────►│  Topic: video.upload            │───────► Consumer X
                    │  ┌─────┐ ┌─────┐ ┌─────┐       │
                    │  │msg1 │ │msg2 │ │msg3 │       │
                    │  └─────┘ └─────┘ └─────┘       │
                    │                                 │
Producer B ────────►│  Topic: user.signup             │───────► Consumer Y
                    │  ┌─────┐ ┌─────┐               │
                    │  │msg4 │ │msg5 │               │
                    │  └─────┘ └─────┘               │
                    └─────────────────────────────────┘
```

---

## Messaging Patterns

### 1. Point-to-Point (Queue)
```
One message → One consumer

Producer ────► Queue ────► Consumer A (gets msg 1)
                    ├────► Consumer B (gets msg 2)
                    └────► Consumer C (gets msg 3)

Each message processed by EXACTLY ONE consumer
Load balancing!
```

### 2. Publish-Subscribe (Pub/Sub)
```
One message → All subscribers

                    ┌────► Subscriber A (gets msg)
Producer ───► Topic ├────► Subscriber B (gets msg)
                    └────► Subscriber C (gets msg)

All subscribers get EVERY message
Broadcast!
```

### 3. Request-Reply
```
Producer ───► Request Queue ───► Consumer
    ▲                                │
    │                                ▼
    └──── Reply Queue ◄──────────────┘

Correlation ID links request to response
```

---

## Popular Message Queues

### 1. RabbitMQ
```
Traditional message broker
AMQP protocol
Great for: Request/reply, routing

Features:
- Exchanges & bindings for routing
- Dead letter queues
- Priority queues
- Plugins (delayed messages, sharding)
```

### 2. Apache Kafka
```
Distributed event streaming
High throughput, persistent
Great for: Event sourcing, logs, streaming

Features:
- Persistent storage
- Consumer groups
- Exactly-once semantics
- Partitions for parallelism
```

### 3. AWS SQS
```
Managed queue service
No infrastructure to manage
Great for: AWS applications, simple queuing

Features:
- Standard (at-least-once, out of order)
- FIFO (exactly-once, in order)
- Dead letter queues
- Long polling
```

### 4. Redis Streams
```
Redis-based streaming
Fast, in-memory
Great for: Real-time, simple use cases

Features:
- Consumer groups
- Message persistence
- Blocking reads
```

---

## RabbitMQ Example

### Architecture:
```
                  ┌─────────────────────────────────────┐
                  │            RabbitMQ                  │
                  │                                      │
Producer ───────► │  Exchange ──► Queue ──► Consumer   │
                  │     │                               │
                  │     └──────► Queue ──► Consumer    │
                  │                                      │
                  └─────────────────────────────────────┘

Exchange types:
- Direct:  Route by exact routing key
- Topic:   Route by pattern (*.video.*)
- Fanout:  Broadcast to all queues
- Headers: Route by message headers
```

### Producer:
```javascript
const amqp = require('amqplib');

class VideoUploadProducer {
    constructor() {
        this.connection = null;
        this.channel = null;
    }
    
    async connect() {
        this.connection = await amqp.connect('amqp://localhost');
        this.channel = await this.connection.createChannel();
        
        // Declare exchange
        await this.channel.assertExchange('video_events', 'topic', {
            durable: true
        });
    }
    
    async publishVideoUploaded(videoId, userId, fileUrl) {
        const message = {
            type: 'video.uploaded',
            payload: {
                video_id: videoId,
                user_id: userId,
                file_url: fileUrl,
                timestamp: new Date().toISOString()
            }
        };
        
        this.channel.publish(
            'video_events',           // exchange
            'video.uploaded',         // routing key
            Buffer.from(JSON.stringify(message)),
            { 
                persistent: true,     // survive broker restart
                messageId: `msg_${Date.now()}`,
                contentType: 'application/json'
            }
        );
        
        console.log('Published video upload event');
    }
}

// Usage
const producer = new VideoUploadProducer();
await producer.connect();
await producer.publishVideoUploaded('vid_123', 'user_456', 's3://bucket/video.mp4');
```

### Consumer:
```javascript
class VideoProcessor {
    async connect() {
        this.connection = await amqp.connect('amqp://localhost');
        this.channel = await this.connection.createChannel();
        
        // Declare queue
        await this.channel.assertQueue('video_processing', {
            durable: true,
            deadLetterExchange: 'video_events_dlx'  // Failed messages go here
        });
        
        // Bind to exchange
        await this.channel.bindQueue(
            'video_processing',
            'video_events',
            'video.uploaded'
        );
        
        // Only process one at a time
        this.channel.prefetch(1);
    }
    
    async startConsuming() {
        this.channel.consume('video_processing', async (msg) => {
            try {
                const content = JSON.parse(msg.content.toString());
                console.log('Processing video:', content.payload.video_id);
                
                // Process video (transcode, generate thumbnails, etc.)
                await this.processVideo(content.payload);
                
                // Acknowledge success
                this.channel.ack(msg);
                console.log('Video processed successfully');
                
            } catch (error) {
                console.error('Processing failed:', error);
                
                // Reject and requeue (or send to DLQ after retries)
                const retried = msg.properties.headers['x-retry-count'] || 0;
                
                if (retried < 3) {
                    // Requeue with retry count
                    this.channel.nack(msg, false, false);
                    await this.republishWithRetry(msg, retried + 1);
                } else {
                    // Send to dead letter queue
                    this.channel.nack(msg, false, false);
                }
            }
        });
    }
    
    async processVideo(payload) {
        // Actual video processing logic
        await transcodeVideo(payload.file_url);
        await generateThumbnails(payload.video_id);
        await updateDatabase(payload.video_id, 'processed');
    }
}
```

---

## Apache Kafka Example

### Architecture:
```
                    ┌─────────────────────────────────────────┐
                    │             Kafka Cluster               │
                    ├─────────────────────────────────────────┤
                    │                                         │
                    │  Topic: video-uploads                   │
                    │  ┌─────────────────────────────────┐   │
                    │  │ Partition 0: [msg1][msg4][msg7] │   │
                    │  │ Partition 1: [msg2][msg5][msg8] │   │
                    │  │ Partition 2: [msg3][msg6][msg9] │   │
                    │  └─────────────────────────────────┘   │
                    │                                         │
Producer ──────────►│                                         │
                    │              Consumer Group A           │
                    │  ┌──────────────────────────────────┐  │
                    │  │ Consumer 1 ◄── Partition 0       │  │
                    │  │ Consumer 2 ◄── Partition 1, 2    │  │
                    │  └──────────────────────────────────┘  │
                    └─────────────────────────────────────────┘

Partitions = Parallelism
Consumer Groups = Independent consumers
```

### Producer:
```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
    clientId: 'video-service',
    brokers: ['kafka1:9092', 'kafka2:9092', 'kafka3:9092']
});

const producer = kafka.producer();

async function publishVideoEvent(eventType, data) {
    await producer.connect();
    
    await producer.send({
        topic: 'video-events',
        messages: [{
            key: data.video_id,  // Partition key - same video = same partition
            value: JSON.stringify({
                type: eventType,
                timestamp: Date.now(),
                data
            }),
            headers: {
                'correlation-id': generateCorrelationId()
            }
        }]
    });
}

// Usage
await publishVideoEvent('video.uploaded', {
    video_id: 'vid_123',
    user_id: 'user_456',
    file_url: 's3://bucket/video.mp4'
});
```

### Consumer:
```javascript
const consumer = kafka.consumer({ groupId: 'video-processors' });

async function startProcessing() {
    await consumer.connect();
    await consumer.subscribe({ topic: 'video-events', fromBeginning: false });
    
    await consumer.run({
        eachMessage: async ({ topic, partition, message }) => {
            const event = JSON.parse(message.value.toString());
            
            console.log({
                partition,
                offset: message.offset,
                key: message.key?.toString(),
                type: event.type
            });
            
            switch (event.type) {
                case 'video.uploaded':
                    await processVideoUpload(event.data);
                    break;
                case 'video.deleted':
                    await handleVideoDeletion(event.data);
                    break;
            }
        }
    });
}

// With batch processing for efficiency
await consumer.run({
    eachBatch: async ({ batch, resolveOffset, heartbeat }) => {
        for (const message of batch.messages) {
            await processMessage(message);
            resolveOffset(message.offset);
            await heartbeat();
        }
    }
});
```

---

## Message Delivery Guarantees

### At-Most-Once
```
Send message → Don't confirm → May lose message

Fast but unreliable
Use for: Metrics, logs where loss is OK
```

### At-Least-Once
```
Send message → Confirm → Retry if no confirm

May get duplicates!
Use for: Most use cases (handle duplicates)
```

### Exactly-Once
```
Send message → Confirm → Deduplicate

Best but most complex/slow
Use for: Financial transactions, critical data
```

### Implementation:
```javascript
// At-least-once with idempotency
class IdempotentProcessor {
    constructor(redis) {
        this.redis = redis;
    }
    
    async process(message) {
        const messageId = message.id;
        
        // Check if already processed
        const processed = await this.redis.get(`processed:${messageId}`);
        if (processed) {
            console.log('Already processed, skipping');
            return;
        }
        
        // Process message
        await this.handleMessage(message);
        
        // Mark as processed (with TTL for cleanup)
        await this.redis.setex(`processed:${messageId}`, 86400, 'done');
    }
}
```

---

## Dead Letter Queues

### What is DLQ?
```
Messages that can't be processed go to DLQ

Main Queue                    Dead Letter Queue
┌─────────────────┐          ┌─────────────────┐
│ [msg1] [msg2]   │  failed  │ [bad_msg]       │
│                 │ ───────► │                 │
│ Processing...   │          │ For analysis    │
└─────────────────┘          └─────────────────┘
```

### Implementation:
```javascript
// RabbitMQ DLQ setup
async function setupQueuesWithDLQ() {
    const channel = await connection.createChannel();
    
    // Dead letter exchange
    await channel.assertExchange('dlx', 'direct', { durable: true });
    
    // Dead letter queue
    await channel.assertQueue('video_processing_dlq', {
        durable: true
    });
    await channel.bindQueue('video_processing_dlq', 'dlx', 'video_processing');
    
    // Main queue with DLQ config
    await channel.assertQueue('video_processing', {
        durable: true,
        deadLetterExchange: 'dlx',
        deadLetterRoutingKey: 'video_processing',
        messageTtl: 60000  // Optional: message timeout
    });
}

// DLQ consumer for alerting/analysis
async function monitorDLQ() {
    channel.consume('video_processing_dlq', async (msg) => {
        const failedMessage = JSON.parse(msg.content.toString());
        
        // Alert ops team
        await sendAlert({
            type: 'PROCESSING_FAILURE',
            message: failedMessage,
            originalQueue: 'video_processing',
            failureReason: msg.properties.headers['x-death']
        });
        
        // Log for analysis
        console.error('Dead letter received:', failedMessage);
        
        channel.ack(msg);
    });
}
```

---

## YouTube Message Queue Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      YouTube Event-Driven Architecture                   │
└─────────────────────────────────────────────────────────────────────────┘

┌────────────────┐                 ┌────────────────────────────────────┐
│  Upload Service│                 │            Kafka Cluster           │
│                │                 │                                    │
│  POST /upload  │───────────────►│  Topic: video-uploads              │
└────────────────┘                 │  ┌────────────────────────────┐   │
                                    │  │ P0 │ P1 │ P2 │ P3 │ P4 │   │   │
                                    │  └────────────────────────────┘   │
                                    │                                    │
                                    │  Topic: video-processing          │
                                    │  Topic: notifications             │
                                    │  Topic: analytics                 │
                                    └───────────────────┬────────────────┘
                                                        │
               ┌────────────────────────────────────────┼────────────────────┐
               │                                        │                    │
               ▼                                        ▼                    ▼
    ┌────────────────────┐              ┌────────────────────┐    ┌──────────────┐
    │  Transcode Service │              │ Thumbnail Service  │    │ Notification │
    │  (Consumer Group)  │              │  (Consumer Group)  │    │   Service    │
    │                    │              │                    │    │              │
    │  ┌──────┐ ┌──────┐│              │  ┌──────┐ ┌──────┐│    │  ┌──────┐   │
    │  │W1    │ │W2    ││              │  │W1    │ │W2    ││    │  │W1    │   │
    │  └──────┘ └──────┘│              │  └──────┘ └──────┘│    │  └──────┘   │
    └────────────────────┘              └────────────────────┘    └──────────────┘
               │                                  │
               ▼                                  ▼
    ┌────────────────────┐              ┌────────────────────┐
    │   Video Storage    │              │  Thumbnail Storage │
    │      (S3)          │              │       (S3)         │
    └────────────────────┘              └────────────────────┘
```

### Event Flow:
```javascript
// 1. Upload Service publishes event
async function handleVideoUpload(file, userId) {
    // Store raw file
    const fileUrl = await s3.upload(file);
    
    // Create video record
    const video = await db.videos.create({
        userId,
        status: 'processing',
        originalUrl: fileUrl
    });
    
    // Publish event
    await kafka.send('video-uploads', {
        key: video.id,
        value: {
            type: 'video.uploaded',
            videoId: video.id,
            userId,
            fileUrl,
            timestamp: Date.now()
        }
    });
    
    return { videoId: video.id, status: 'processing' };
}

// 2. Multiple consumers process independently
// Transcode Consumer
async function handleTranscode(event) {
    const { videoId, fileUrl } = event;
    
    const qualities = ['1080p', '720p', '480p', '360p'];
    
    for (const quality of qualities) {
        const outputUrl = await ffmpeg.transcode(fileUrl, quality);
        await db.videoQualities.create({ videoId, quality, url: outputUrl });
        
        // Publish progress
        await kafka.send('video-processing', {
            type: 'quality.ready',
            videoId,
            quality
        });
    }
    
    await db.videos.update(videoId, { status: 'ready' });
    await kafka.send('video-processing', {
        type: 'video.ready',
        videoId
    });
}

// Thumbnail Consumer
async function handleThumbnail(event) {
    const { videoId, fileUrl } = event;
    const thumbnails = await generateThumbnails(fileUrl, [0, 25, 50, 75]);
    await db.thumbnails.createMany(videoId, thumbnails);
}

// Notification Consumer
kafka.consume('video-processing', async (event) => {
    if (event.type === 'video.ready') {
        const video = await db.videos.find(event.videoId);
        await sendPushNotification(video.userId, 'Your video is ready!');
        await sendEmail(video.userId, 'Video published', video);
    }
});
```

---

## Interview Questions

**Q: What is a message queue?**
A: A buffer between producers and consumers that stores messages for asynchronous processing. Decouples services and enables scaling.

**Q: When would you use a message queue?**
A: Long-running tasks (video processing), decoupling services, handling traffic spikes, event-driven architecture.

**Q: Kafka vs RabbitMQ?**
A: Kafka: High throughput, log-based, replay messages, streaming. RabbitMQ: Traditional broker, complex routing, lower latency for low volume.

**Q: What is a dead letter queue?**
A: Queue for messages that failed processing after retries. Used for debugging and manual intervention.

**Q: Delivery guarantees?**
A: At-most-once (may lose), At-least-once (may duplicate), Exactly-once (complex, slow). Most use at-least-once with idempotent consumers.

---

## Quick Summary

```
MESSAGE QUEUES:
───────────────
Async communication between services
Producers → Queue → Consumers

PATTERNS:
─────────
Point-to-Point: One consumer processes each message
Pub/Sub: All subscribers get every message
Request-Reply: Async request/response

POPULAR QUEUES:
───────────────
RabbitMQ:  Traditional broker, routing, low latency
Kafka:     Streaming, high throughput, replay
SQS:       Managed, serverless, AWS native
Redis:     Fast, simple, in-memory

DELIVERY:
─────────
At-most-once:  May lose messages
At-least-once: May duplicate (use idempotency)
Exactly-once:  Complex but reliable

BEST PRACTICES:
───────────────
- Idempotent consumers
- Dead letter queues
- Message retries with backoff
- Correlation IDs for tracing
```

You now understand message queues! 📬
