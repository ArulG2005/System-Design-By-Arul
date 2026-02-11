# Change Data Capture (CDC) - Complete Guide

## What is CDC?

Think of it like a **security camera system**:
- Records every change that happens
- You can replay events later
- Other systems can watch the feed
- Nothing gets missed

**Simple Definition:**
CDC captures and broadcasts database changes (inserts, updates, deletes) in real-time to other systems.

---

## The Problem CDC Solves

### Traditional Approach (Polling):
```
Every 5 minutes:
┌──────────────┐      ┌──────────────┐
│   Database   │ ───► │ Search Index │
│              │      │ (Outdated!)  │
└──────────────┘      └──────────────┘

Problems:
- 5 minute delay (stale data)
- Heavy database load (constant queries)
- Missing changes if >1 update between polls
- Complex "where updated_at > last_check"
```

### CDC Approach (Real-time):
```
On every change:
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   Database   │ ───► │     CDC      │ ───► │ Search Index │
│              │      │   Stream     │      │ (Up to date!)│
└──────────────┘      └──────────────┘      └──────────────┘

Benefits:
- Real-time updates (seconds)
- No database polling
- Never miss a change
- Clean separation
```

---

## How CDC Works

### Database Transaction Log:
```
Every database maintains a log of all changes:

PostgreSQL: WAL (Write-Ahead Log)
MySQL: Binlog (Binary Log)
MongoDB: Oplog (Operations Log)

┌────────────────────────────────────────────────────────┐
│                    Transaction Log                      │
├────────────────────────────────────────────────────────┤
│ [001] INSERT INTO users (id, name) VALUES (1, 'John')  │
│ [002] UPDATE users SET name = 'Johnny' WHERE id = 1    │
│ [003] INSERT INTO videos (title) VALUES ('My Video')   │
│ [004] DELETE FROM users WHERE id = 1                   │
└────────────────────────────────────────────────────────┘

CDC reads this log and streams changes!
```

### CDC Pipeline:
```
┌──────────┐    ┌───────────┐    ┌─────────────┐    ┌──────────────┐
│ Database │───►│ CDC Tool  │───►│Message Queue│───►│  Consumers   │
│  (MySQL) │    │ (Debezium)│    │  (Kafka)    │    │              │
└──────────┘    └───────────┘    └─────────────┘    └──────────────┘
                                       │
                      ┌────────────────┼────────────────┐
                      ▼                ▼                ▼
               ┌───────────┐    ┌───────────┐    ┌───────────┐
               │   Search  │    │  Cache    │    │ Analytics │
               │   Index   │    │  Layer    │    │   System  │
               └───────────┘    └───────────┘    └───────────┘
```

---

## CDC Patterns

### 1. Log-Based CDC (Recommended)
```
Read database's transaction log directly

┌──────────────┐
│   Database   │
│  ┌────────┐  │
│  │  WAL   │──┼───► CDC reads log
│  │  Log   │  │
│  └────────┘  │
└──────────────┘

Pros:
✓ No impact on database
✓ Captures all changes
✓ Maintains order

Cons:
✗ Need access to log
✗ Database-specific
```

### 2. Trigger-Based CDC
```sql
-- Create trigger to capture changes

CREATE TABLE users_changelog (
    id SERIAL PRIMARY KEY,
    user_id INTEGER,
    operation VARCHAR(10),
    old_data JSONB,
    new_data JSONB,
    changed_at TIMESTAMP DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION users_cdc_trigger()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO users_changelog (user_id, operation, new_data)
        VALUES (NEW.id, 'INSERT', row_to_json(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO users_changelog (user_id, operation, old_data, new_data)
        VALUES (NEW.id, 'UPDATE', row_to_json(OLD), row_to_json(NEW));
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO users_changelog (user_id, operation, old_data)
        VALUES (OLD.id, 'DELETE', row_to_json(OLD));
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_cdc
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION users_cdc_trigger();

Pros:
✓ Works on any database
✓ Custom logic possible

Cons:
✗ Performance overhead
✗ Table bloat
✗ Trigger complexity
```

### 3. Query-Based CDC (Polling)
```python
# Poll for changes using timestamp

last_check = get_last_check_time()

while True:
    # Get changes since last check
    changes = db.query("""
        SELECT * FROM users 
        WHERE updated_at > %s 
        ORDER BY updated_at
    """, [last_check])
    
    for change in changes:
        process_change(change)
        last_check = change['updated_at']
    
    save_last_check_time(last_check)
    time.sleep(5)  # Check every 5 seconds

Pros:
✓ Simple to implement
✓ No special tools needed

Cons:
✗ Misses deletes
✗ Database load
✗ Latency
```

---

## CDC Event Format

### Standard Event Structure:
```json
{
    "schema": {
        "type": "struct",
        "fields": [...]
    },
    "payload": {
        "before": {
            "id": 1,
            "name": "John",
            "email": "john@old.com"
        },
        "after": {
            "id": 1,
            "name": "John",
            "email": "john@new.com"
        },
        "source": {
            "version": "1.0",
            "connector": "postgresql",
            "name": "my_db",
            "db": "youtube",
            "table": "users"
        },
        "op": "u",
        "ts_ms": 1705312800000
    }
}

op values:
"c" = create (INSERT)
"u" = update
"d" = delete
"r" = read (initial snapshot)
```

---

## Debezium Example

### Architecture:
```
┌──────────────────────────────────────────────────────────────────┐
│                         Kafka Connect                             │
│  ┌────────────────────┐           ┌─────────────────────┐        │
│  │  Debezium MySQL    │           │  Debezium PostgreSQL │        │
│  │  Connector         │           │  Connector           │        │
│  └─────────┬──────────┘           └──────────┬──────────┘        │
└────────────┼─────────────────────────────────┼───────────────────┘
             │                                  │
             ▼                                  ▼
┌───────────────────────────────────────────────────────────────────┐
│                            Kafka                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐│
│  │ mysql.users      │  │ pg.videos        │  │ pg.comments      ││
│  │ topic            │  │ topic            │  │ topic            ││
│  └──────────────────┘  └──────────────────┘  └──────────────────┘│
└───────────────────────────────────────────────────────────────────┘
```

### Configuration:
```json
{
    "name": "youtube-mysql-connector",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "database.hostname": "mysql.youtube.internal",
        "database.port": "3306",
        "database.user": "cdc_user",
        "database.password": "***",
        "database.server.id": "1",
        "database.server.name": "youtube_db",
        "database.include.list": "youtube",
        "table.include.list": "youtube.users,youtube.videos,youtube.comments",
        "database.history.kafka.bootstrap.servers": "kafka:9092",
        "database.history.kafka.topic": "schema-changes.youtube"
    }
}
```

### Consumer:
```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'youtube_db.youtube.videos',
    bootstrap_servers=['kafka:9092'],
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    event = message.value
    payload = event['payload']
    
    if payload['op'] == 'c':  # INSERT
        video = payload['after']
        index_to_elasticsearch(video)
        
    elif payload['op'] == 'u':  # UPDATE
        video = payload['after']
        update_elasticsearch(video)
        
    elif payload['op'] == 'd':  # DELETE
        video_id = payload['before']['id']
        delete_from_elasticsearch(video_id)
```

---

## CDC Use Cases

### 1. Search Index Sync
```
┌──────────┐         ┌───────────┐         ┌───────────────┐
│ PostgreSQL│   CDC   │   Kafka   │ Consumer│ Elasticsearch │
│          │────────►│           │────────►│               │
│ videos   │         │           │         │ videos index  │
└──────────┘         └───────────┘         └───────────────┘

User uploads video → Inserted in DB → CDC captures → Indexed in ES → Searchable!
```

### 2. Cache Invalidation
```python
# CDC Consumer for cache invalidation
def process_change(event):
    table = event['source']['table']
    payload = event['payload']
    
    if table == 'videos':
        if payload['op'] in ['u', 'd']:
            video_id = payload['before']['id']
            redis.delete(f"video:{video_id}")
            redis.delete(f"video:{video_id}:metadata")
            
    elif table == 'users':
        if payload['op'] in ['u', 'd']:
            user_id = payload['before']['id']
            redis.delete(f"user:{user_id}")
            redis.delete(f"user:{user_id}:profile")
```

### 3. Data Warehouse Sync
```
Production DB ──CDC──► Kafka ──────► Data Warehouse
                         │
Real-time events ◄───────┘
                         │
                         ▼
                    Analytics
                    Dashboard
```

### 4. Microservice Communication
```
┌───────────────┐         ┌──────────────────────────────────────┐
│ Video Service │         │              CDC Events              │
│  (Writes DB)  │────────►│                                      │
└───────────────┘         │  video.created                       │
                          │  video.updated                       │
                          │  video.deleted                       │
                          └──────────────┬───────────────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    ▼                    ▼                    ▼
            ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
            │   Search    │      │ Notification│      │  Analytics  │
            │   Service   │      │   Service   │      │   Service   │
            └─────────────┘      └─────────────┘      └─────────────┘
```

### 5. Event Sourcing
```
CDC converts traditional DB into event stream

Instead of: "Current state is X"
You get:    "These events led to state X"

Events:
1. Video created: {title: "My Video"}
2. Video updated: {title: "My Awesome Video"}
3. Video published: {status: "public"}

Can replay events to:
- Rebuild state
- Create new views
- Debug issues
```

---

## YouTube CDC Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           YouTube Platform                               │
└─────────────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   Users DB      │  │   Videos DB     │  │   Comments DB   │
│   (PostgreSQL)  │  │   (PostgreSQL)  │  │   (PostgreSQL)  │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
         │         ┌──────────┴──────────┐         │
         │         │                     │         │
         ▼         ▼                     ▼         ▼
    ┌─────────────────────────────────────────────────────┐
    │               Debezium CDC Connectors               │
    └───────────────────────────┬─────────────────────────┘
                                │
                                ▼
    ┌─────────────────────────────────────────────────────┐
    │                    Kafka Topics                      │
    │  ┌───────────┐  ┌───────────┐  ┌───────────────┐   │
    │  │ db.users  │  │ db.videos │  │ db.comments   │   │
    │  └───────────┘  └───────────┘  └───────────────┘   │
    └───────────────────────────┬─────────────────────────┘
                                │
       ┌────────────────────────┼────────────────────────┐
       ▼                        ▼                        ▼
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│Elasticsearch │        │    Redis     │        │   BigQuery   │
│  (Search)    │        │   (Cache)    │        │ (Analytics)  │
└──────────────┘        └──────────────┘        └──────────────┘
       │                        │                        │
       ▼                        ▼                        ▼
  Video search          Cache invalidation       View analytics
  appears instantly     stays fresh              Real-time stats
```

---

## Code Example: Full CDC Pipeline

### Producer (Debezium Config):
```yaml
# docker-compose.yml
version: '3'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    
  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      
  connect:
    image: debezium/connect:latest
    depends_on:
      - kafka
    ports:
      - "8083:8083"
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
```

### Consumer (Node.js):
```javascript
const { Kafka } = require('kafkajs');
const { Client } = require('@elastic/elasticsearch');

const kafka = new Kafka({
    clientId: 'cdc-consumer',
    brokers: ['kafka:9092']
});

const consumer = kafka.consumer({ groupId: 'search-indexer' });
const elastic = new Client({ node: 'http://elasticsearch:9200' });

async function run() {
    await consumer.connect();
    await consumer.subscribe({ topic: 'youtube.videos', fromBeginning: false });
    
    await consumer.run({
        eachMessage: async ({ topic, partition, message }) => {
            const event = JSON.parse(message.value.toString());
            const { op, before, after } = event.payload;
            
            switch (op) {
                case 'c': // CREATE
                    await elastic.index({
                        index: 'videos',
                        id: after.id.toString(),
                        body: {
                            title: after.title,
                            description: after.description,
                            user_id: after.user_id,
                            created_at: after.created_at
                        }
                    });
                    console.log(`Indexed video ${after.id}`);
                    break;
                    
                case 'u': // UPDATE
                    await elastic.update({
                        index: 'videos',
                        id: after.id.toString(),
                        body: {
                            doc: {
                                title: after.title,
                                description: after.description
                            }
                        }
                    });
                    console.log(`Updated video ${after.id}`);
                    break;
                    
                case 'd': // DELETE
                    await elastic.delete({
                        index: 'videos',
                        id: before.id.toString()
                    });
                    console.log(`Deleted video ${before.id}`);
                    break;
            }
        }
    });
}

run().catch(console.error);
```

---

## Interview Questions

**Q: What is CDC?**
A: Change Data Capture - technique to track and stream database changes (inserts, updates, deletes) in real-time to other systems.

**Q: How does log-based CDC work?**
A: Reads the database's transaction log (WAL in Postgres, binlog in MySQL) without impacting database performance.

**Q: CDC vs polling?**
A: CDC: real-time, no DB load, catches all changes. Polling: delayed, loads DB, might miss changes between polls.

**Q: What is Debezium?**
A: Open-source CDC platform that streams changes from databases to Kafka using log-based capture.

**Q: Use cases for CDC?**
A: Search indexing, cache invalidation, data warehouse sync, microservice events, audit logging.

---

## Quick Summary

```
CDC (Change Data Capture):
─────────────────────────
Captures database changes in real-time
Streams to other systems

HOW IT WORKS:
─────────────
Database → Transaction Log → CDC Tool → Message Queue → Consumers

CDC PATTERNS:
─────────────
1. Log-based (best): Read transaction log
2. Trigger-based: Database triggers
3. Query-based: Polling with timestamps

EVENT FORMAT:
─────────────
{
  "op": "c/u/d",
  "before": {...},
  "after": {...},
  "source": {...}
}

USE CASES:
──────────
- Search index sync
- Cache invalidation
- Data warehouse sync
- Microservice events
- Audit logging

TOOLS:
──────
- Debezium (open-source)
- AWS DMS
- Google Datastream
- Confluent CDC
```

You now understand CDC! 📡
