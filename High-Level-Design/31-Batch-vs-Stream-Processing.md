# Batch vs Stream Processing - Complete Guide

## The Laundry Analogy

```
BATCH PROCESSING (Weekly Laundry):
────────────────────────────────────────────────────────────
Monday: Collect clothes in basket
Tuesday: More clothes added
Wednesday: More clothes added
Thursday: More clothes added
Friday: More clothes added
Saturday: WASH EVERYTHING AT ONCE (BATCH)

→ Wait for clothes to accumulate
→ Process all together
→ Efficient use of machine
→ Results only on Saturday


STREAM PROCESSING (Daily Laundry):
────────────────────────────────────────────────────────────
Monday: Wear shirt → Wash shirt → Clean shirt ready
Tuesday: Wear pants → Wash pants → Clean pants ready
Wednesday: Wear socks → Wash socks → Clean socks ready

→ Process items as they arrive
→ Always have clean clothes
→ More machine runs
→ Real-time results
```

---

## Simple Definitions

```
BATCH PROCESSING:
─────────────────
Collect data over time
Process ALL at once
Results after processing completes

Example: "Calculate all user statistics every night at 2 AM"


STREAM PROCESSING:
──────────────────
Process data as it arrives
One event at a time
Results immediately

Example: "Update user statistics in real-time as events happen"
```

---

## Visual Comparison

```
BATCH PROCESSING:
════════════════════════════════════════════════════════════

Time:     ──────────────────────────────────────────────▶

Events:   • • • • • • • • • • • • • • • • • • • • • • • •
          ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓ ↓
Storage:  [════════════════════════════════════════════════]
                           Accumulate

                                                         │
At scheduled time ──────────────────────────────────────►│
                                                         ▼
          ┌─────────────────────────────────────────────────┐
          │              BATCH JOB                          │
          │  Process 1 million records at once              │
          └─────────────────────────────────────────────────┘
                                                         │
                                                         ▼
                                                   [Results]


STREAM PROCESSING:
════════════════════════════════════════════════════════════

Time:     ──────────────────────────────────────────────▶

Events:   • ────────────▶ Process ──▶ Result
            • ──────────▶ Process ──▶ Result
              • ────────▶ Process ──▶ Result
                • ──────▶ Process ──▶ Result
                  • ────▶ Process ──▶ Result
                    • ──▶ Process ──▶ Result
                      • ▶ Process ──▶ Result

Each event processed immediately upon arrival
```

---

## Batch Processing Deep Dive

### Characteristics:
```
┌─────────────────────────────────────────────────────────────┐
│                    BATCH PROCESSING                         │
├─────────────────────────────────────────────────────────────┤
│ Latency:      High (hours to days)                         │
│ Throughput:   Very High                                    │
│ Data Volume:  Massive (TB to PB)                          │
│ Processing:   MapReduce, Spark                             │
│ Storage:      HDFS, Data Lakes                             │
│ Scheduling:   Cron, Airflow, Luigi                         │
│ Use Case:     Analytics, ML Training, ETL                  │
└─────────────────────────────────────────────────────────────┘
```

### Architecture:
```
BATCH PROCESSING ARCHITECTURE:
════════════════════════════════════════════════════════════

  Data Sources                    Data Lake                  
  ────────────                    ─────────                  
  [Web Logs ]  ─────┐     ┌───▶  [HDFS / S3]                
  [Databases]  ─────┼─────┤                                  
  [CSV Files]  ─────┘     └───▶  [Parquet Files]            
                                       │                     
                                       ▼                     
                          ┌────────────────────┐            
                          │   BATCH ENGINE     │            
                          │                    │            
                          │  ┌──────────────┐  │            
                          │  │   Spark      │  │            
                          │  │   Hadoop     │  │            
                          │  │   Flink Batch│  │            
                          │  └──────────────┘  │            
                          │                    │            
                          │  Scheduled daily   │            
                          │  at 2:00 AM        │            
                          └────────────────────┘            
                                       │                     
                                       ▼                     
                          ┌────────────────────┐            
                          │   Data Warehouse   │            
                          │   (Results)        │            
                          │                    │            
                          │  - User analytics  │            
                          │  - Revenue reports │            
                          │  - ML models       │            
                          └────────────────────┘            
```

### Example - YouTube Analytics:
```python
# BATCH PROCESSING: Daily Video Analytics

from pyspark.sql import SparkSession
from pyspark.sql.functions import *

# Initialize Spark
spark = SparkSession.builder \
    .appName("YouTubeAnalytics") \
    .getOrCreate()

# Read yesterday's view logs (batch of millions of records)
views = spark.read.parquet("s3://youtube-logs/views/date=2024-01-15/")

# Calculate video statistics
video_stats = views \
    .groupBy("video_id") \
    .agg(
        count("*").alias("view_count"),
        countDistinct("user_id").alias("unique_viewers"),
        avg("watch_duration").alias("avg_watch_time"),
        sum("watch_duration").alias("total_watch_time")
    )

# Calculate top videos
top_videos = video_stats \
    .orderBy(desc("view_count")) \
    .limit(100)

# Write results to data warehouse
video_stats.write \
    .mode("overwrite") \
    .partitionBy("date") \
    .parquet("s3://youtube-analytics/video_stats/")

# This job runs once daily at 2 AM
# Processes ~500 million view events
# Takes 2-3 hours to complete
```

### Scheduling with Airflow:
```python
# Airflow DAG for batch processing

from airflow import DAG
from airflow.operators.spark_submit_operator import SparkSubmitOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'youtube-analytics',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'retries': 3,
    'retry_delay': timedelta(minutes=5)
}

dag = DAG(
    'youtube_daily_analytics',
    default_args=default_args,
    schedule_interval='0 2 * * *',  # Run at 2 AM daily
    catchup=False
)

# Task 1: Calculate video stats
video_stats_job = SparkSubmitOperator(
    task_id='calculate_video_stats',
    application='/jobs/video_stats.py',
    conn_id='spark_cluster',
    dag=dag
)

# Task 2: Calculate channel stats
channel_stats_job = SparkSubmitOperator(
    task_id='calculate_channel_stats',
    application='/jobs/channel_stats.py',
    conn_id='spark_cluster',
    dag=dag
)

# Task 3: Generate recommendations
recommendations_job = SparkSubmitOperator(
    task_id='generate_recommendations',
    application='/jobs/recommendations.py',
    conn_id='spark_cluster',
    dag=dag
)

# Dependencies
video_stats_job >> channel_stats_job >> recommendations_job
```

---

## Stream Processing Deep Dive

### Characteristics:
```
┌─────────────────────────────────────────────────────────────┐
│                   STREAM PROCESSING                         │
├─────────────────────────────────────────────────────────────┤
│ Latency:      Low (milliseconds to seconds)                │
│ Throughput:   Medium-High                                  │
│ Data Volume:  Continuous flow                              │
│ Processing:   Kafka Streams, Flink, Storm                  │
│ Storage:      Event Log (Kafka)                            │
│ Triggering:   Event-driven                                 │
│ Use Case:     Real-time alerts, live dashboards            │
└─────────────────────────────────────────────────────────────┘
```

### Architecture:
```
STREAM PROCESSING ARCHITECTURE:
════════════════════════════════════════════════════════════

  Event Sources                                              
  ─────────────                                              
  [User Click]  ──┐                                          
  [Video Play]  ──┼──▶  ┌─────────────────────┐              
  [Comment   ]  ──┤     │                     │              
  [Like      ]  ──┘     │   MESSAGE BROKER    │              
                        │      (Kafka)        │              
                        │                     │              
                        │   Topic: events     │              
                        │   ├─ Partition 0    │              
                        │   ├─ Partition 1    │              
                        │   └─ Partition 2    │              
                        │                     │              
                        └─────────────────────┘              
                                   │                         
                                   ▼                         
                        ┌─────────────────────┐              
                        │  STREAM PROCESSOR   │              
                        │                     │              
                        │  ┌───────────────┐  │              
                        │  │ Kafka Streams │  │              
                        │  │ Apache Flink  │  │              
                        │  │ Spark Stream  │  │              
                        │  └───────────────┘  │              
                        │                     │              
                        │  Running 24/7       │              
                        │  Processing events  │              
                        │  as they arrive     │              
                        └─────────────────────┘              
                                   │                         
            ┌──────────────────────┼──────────────────────┐  
            ▼                      ▼                      ▼  
    ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
    │ Real-time    │      │   Alerts     │      │    Live      │
    │ Dashboard    │      │   Service    │      │   Updates    │
    └──────────────┘      └──────────────┘      └──────────────┘
```

### Example - YouTube Real-time View Count:
```javascript
// STREAM PROCESSING: Real-time View Counter (Kafka Streams style)

const { Kafka } = require('kafkajs');
const redis = require('redis');

const kafka = new Kafka({ brokers: ['kafka:9092'] });
const consumer = kafka.consumer({ groupId: 'view-counter' });
const redisClient = redis.createClient();

async function processViewEvents() {
    await consumer.connect();
    await consumer.subscribe({ topic: 'video-views', fromBeginning: false });
    
    await consumer.run({
        eachMessage: async ({ topic, partition, message }) => {
            const event = JSON.parse(message.value);
            
            // Process each view event immediately
            const { videoId, userId, timestamp } = event;
            
            // Increment view count in Redis (real-time)
            await redisClient.incr(`video:${videoId}:views`);
            
            // Update unique viewers (HyperLogLog for memory efficiency)
            await redisClient.pfAdd(`video:${videoId}:unique_viewers`, userId);
            
            // Update per-minute view count (for trending)
            const minute = Math.floor(timestamp / 60000);
            await redisClient.incr(`video:${videoId}:views:${minute}`);
            
            console.log(`Processed view for video ${videoId}`);
        }
    });
}

// Run continuously 24/7
processViewEvents();
```

### Apache Flink Example:
```java
// Stream Processing with Apache Flink

public class YouTubeViewProcessor {
    public static void main(String[] args) throws Exception {
        
        StreamExecutionEnvironment env = 
            StreamExecutionEnvironment.getExecutionEnvironment();
        
        // Connect to Kafka
        FlinkKafkaConsumer<ViewEvent> consumer = new FlinkKafkaConsumer<>(
            "video-views",
            new ViewEventSchema(),
            kafkaProperties
        );
        
        DataStream<ViewEvent> viewStream = env.addSource(consumer);
        
        // Real-time aggregation: views per video per minute
        DataStream<VideoStats> videoStats = viewStream
            .keyBy(event -> event.getVideoId())
            .window(TumblingEventTimeWindows.of(Time.minutes(1)))
            .aggregate(new ViewCountAggregator());
        
        // Detect trending videos (more than 10K views in 1 minute)
        DataStream<Alert> trendingAlerts = videoStats
            .filter(stats -> stats.getViewCount() > 10000)
            .map(stats -> new Alert("TRENDING", stats.getVideoId()));
        
        // Output to different sinks
        videoStats.addSink(new RedisSink<>(redisConfig));
        trendingAlerts.addSink(new KafkaSink<>("alerts-topic"));
        
        env.execute("YouTube Real-time Analytics");
    }
}
```

---

## Comparison Table

```
┌──────────────────┬────────────────────┬────────────────────┐
│ Aspect           │ Batch Processing   │ Stream Processing  │
├──────────────────┼────────────────────┼────────────────────┤
│ Latency          │ Hours to days      │ Milliseconds-secs  │
├──────────────────┼────────────────────┼────────────────────┤
│ Data Size        │ Large (TB/PB)      │ Continuous flow    │
├──────────────────┼────────────────────┼────────────────────┤
│ Processing       │ Scheduled          │ Continuous 24/7    │
├──────────────────┼────────────────────┼────────────────────┤
│ Completeness     │ Full dataset       │ Incomplete (now)   │
├──────────────────┼────────────────────┼────────────────────┤
│ Accuracy         │ Exact results      │ Approximate OK     │
├──────────────────┼────────────────────┼────────────────────┤
│ State            │ Stateless (mostly) │ Stateful           │
├──────────────────┼────────────────────┼────────────────────┤
│ Recovery         │ Restart job        │ Checkpointing      │
├──────────────────┼────────────────────┼────────────────────┤
│ Infrastructure   │ Hadoop, Spark      │ Kafka, Flink       │
├──────────────────┼────────────────────┼────────────────────┤
│ Cost             │ Pay for job time   │ Pay for uptime     │
├──────────────────┼────────────────────┼────────────────────┤
│ Complexity       │ Simpler            │ More complex       │
└──────────────────┴────────────────────┴────────────────────┘
```

---

## Lambda Architecture

```
LAMBDA ARCHITECTURE:
═══════════════════════════════════════════════════════════

Combines BATCH and STREAM processing!

                         ┌────────────────────────────────┐
                         │         Data Source            │
                         │    (All incoming events)       │
                         └────────────────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    ▼                   ▼                   ▼
         ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
         │   BATCH LAYER   │  │  SPEED LAYER    │  │  SERVING LAYER  │
         │   (Accuracy)    │  │  (Low Latency)  │  │   (Queries)     │
         └─────────────────┘  └─────────────────┘  └─────────────────┘
                │                       │                   │
                ▼                       ▼                   │
         ┌─────────────┐        ┌─────────────┐            │
         │  Data Lake  │        │   Stream    │            │
         │  (Raw data) │        │  Processor  │            │
         └─────────────┘        └─────────────┘            │
                │                       │                   │
                ▼                       ▼                   │
         ┌─────────────┐        ┌─────────────┐            │
         │ Batch Views │        │ Real-time   │            │
         │ (Accurate)  │        │   Views     │            │
         └─────────────┘        │ (Fast)      │            │
                │               └─────────────┘            │
                │                       │                   │
                └───────────────┬───────┘                   │
                                ▼                           │
                    ┌─────────────────────┐                │
                    │   MERGED VIEW       │◀───────────────┘
                    │                     │
                    │ batch + real-time   │
                    │ = complete picture  │
                    └─────────────────────┘


Use Case: YouTube View Count

SPEED LAYER (Real-time):
- Counts views happening NOW
- Updates every second
- Might miss a few events (99.9% accurate)

BATCH LAYER (Nightly):
- Recalculates ALL views from logs
- 100% accurate
- Replaces speed layer results

MERGED VIEW:
- During day: batch_count + realtime_count
- After batch: batch_count (exact)
```

---

## Kappa Architecture

```
KAPPA ARCHITECTURE:
═══════════════════════════════════════════════════════════

Stream processing ONLY! No batch layer.

                         ┌────────────────────────────────┐
                         │         Data Source            │
                         │    (All events as stream)      │
                         └────────────────────────────────┘
                                        │
                                        ▼
                         ┌────────────────────────────────┐
                         │        KAFKA (Log)             │
                         │   Immutable, ordered events     │
                         │   Retained for replay          │
                         └────────────────────────────────┘
                                        │
                                        ▼
                         ┌────────────────────────────────┐
                         │     STREAM PROCESSOR           │
                         │   (Single processing layer)    │
                         │                                │
                         │   For reprocessing:            │
                         │   1. Deploy new processor v2   │
                         │   2. Replay from beginning     │
                         │   3. New results replace old   │
                         └────────────────────────────────┘
                                        │
                                        ▼
                         ┌────────────────────────────────┐
                         │       SERVING LAYER            │
                         └────────────────────────────────┘


Benefits:
- Simpler than Lambda (one codebase)
- Same logic for real-time and reprocessing
- Easier to maintain

Tradeoff:
- Reprocessing can be slower than batch
- Requires keeping event log longer
```

---

## YouTube Use Cases

```
┌─────────────────────────────────────────────────────────────────┐
│              YOUTUBE PROCESSING USE CASES                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BATCH PROCESSING:                                              │
│  ─────────────────                                              │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Daily Analytics Reports                                   │ │
│  │ - Total views per video (yesterday)                      │ │
│  │ - Channel growth metrics                                 │ │
│  │ - Revenue calculations                                   │ │
│  │ - Runs at: 2 AM daily                                    │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Recommendation Model Training                             │ │
│  │ - Process last 30 days of watch history                  │ │
│  │ - Train ML models                                        │ │
│  │ - Runs at: Weekly on Sunday                              │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Content Moderation Backlog                               │ │
│  │ - Re-scan older videos with new rules                    │ │
│  │ - Process flagged content queue                          │ │
│  │ - Runs at: Continuous batch jobs                         │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  STREAM PROCESSING:                                             │
│  ──────────────────                                             │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Real-time View Counter                                    │ │
│  │ - Update view count as users watch                       │ │
│  │ - Power "watching now" feature                           │ │
│  │ - Latency: < 1 second                                    │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Trending Detection                                        │ │
│  │ - Detect videos going viral                              │ │
│  │ - Alert content team                                     │ │
│  │ - Update trending page                                   │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Live Chat & Comments                                      │ │
│  │ - Process chat messages in real-time                     │ │
│  │ - Spam detection                                         │ │
│  │ - Sentiment analysis                                     │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Fraud Detection                                          │ │
│  │ - Detect view bot patterns                               │ │
│  │ - Block suspicious activity                              │ │
│  │ - Real-time decision making                              │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Complete Implementation:
```python
# HYBRID APPROACH: Batch + Stream

# ═══════════════════════════════════════════════════════
# BATCH: Daily Video Statistics (runs at 2 AM)
# ═══════════════════════════════════════════════════════

def daily_video_stats_batch():
    spark = SparkSession.builder.appName("DailyStats").getOrCreate()
    
    # Read all yesterday's events
    yesterday = (date.today() - timedelta(days=1)).strftime("%Y-%m-%d")
    events = spark.read.parquet(f"s3://youtube-events/date={yesterday}")
    
    # Calculate exact statistics
    video_stats = events \
        .filter(col("event_type") == "view") \
        .groupBy("video_id") \
        .agg(
            count("*").alias("total_views"),
            countDistinct("user_id").alias("unique_viewers"),
            sum("watch_time").alias("total_watch_time"),
            avg("watch_time").alias("avg_watch_time")
        )
    
    # Write to database (this is the source of truth)
    video_stats.write \
        .mode("overwrite") \
        .jdbc(
            url="jdbc:postgresql://analytics-db:5432/youtube",
            table="video_daily_stats",
            properties={"driver": "org.postgresql.Driver"}
        )

# ═══════════════════════════════════════════════════════
# STREAM: Real-time View Counter (runs 24/7)
# ═══════════════════════════════════════════════════════

async def realtime_view_counter():
    """Process view events as they happen"""
    
    consumer = kafka.consumer('view-events')
    
    async for event in consumer:
        video_id = event['video_id']
        user_id = event['user_id']
        
        # Update Redis counters (for real-time display)
        pipeline = redis.pipeline()
        
        # Today's view count
        today = date.today().strftime("%Y-%m-%d")
        pipeline.incr(f"views:{video_id}:{today}")
        
        # Current hour (for trending)
        hour_key = datetime.now().strftime("%Y-%m-%d-%H")
        pipeline.incr(f"hourly:{video_id}:{hour_key}")
        pipeline.expire(f"hourly:{video_id}:{hour_key}", 86400)
        
        # Unique viewers (HyperLogLog)
        pipeline.pfadd(f"unique:{video_id}:{today}", user_id)
        
        await pipeline.execute()
        
        # Check for trending
        hourly_views = await redis.get(f"hourly:{video_id}:{hour_key}")
        if int(hourly_views) > 10000:
            await notify_trending(video_id, hourly_views)

# ═══════════════════════════════════════════════════════
# API: Get View Count (combines batch + stream)
# ═══════════════════════════════════════════════════════

async def get_view_count(video_id):
    """Return total views = batch (historical) + stream (today)"""
    
    # Get historical count from batch results (database)
    historical = await db.fetchval(
        "SELECT SUM(total_views) FROM video_daily_stats WHERE video_id = $1",
        video_id
    )
    
    # Get today's count from stream (Redis)
    today = date.today().strftime("%Y-%m-%d")
    today_views = await redis.get(f"views:{video_id}:{today}")
    
    total = (historical or 0) + int(today_views or 0)
    
    return total
```

---

## When to Use What

```
USE BATCH WHEN:
───────────────
✓ Data completeness required (all users, all events)
✓ Complex calculations (ML training, aggregations)
✓ Historical analysis
✓ Latency not critical (hours/days OK)
✓ Cost-sensitive (only pay for job time)

USE STREAM WHEN:
────────────────
✓ Real-time response needed
✓ Fraud/anomaly detection
✓ Live dashboards
✓ Event-driven actions
✓ Continuous monitoring
✓ User-facing metrics (view counts)

USE BOTH (Lambda/Kappa) WHEN:
─────────────────────────────
✓ Need real-time AND accuracy
✓ Reprocessing required (fix bugs)
✓ Combine historical + live data
```

---

## Interview Questions

**Q: What's the difference between batch and stream processing?**
A: Batch processes accumulated data at scheduled intervals (high latency, exact results). Stream processes events as they arrive (low latency, may be approximate).

**Q: When would you use stream processing?**
A: Real-time dashboards, fraud detection, live view counts, trending detection, notifications - anywhere low latency is critical.

**Q: What is Lambda Architecture?**
A: Combines batch (accuracy) and stream (speed) layers. Batch provides exact historical data, stream provides real-time approximations. Merged view gives complete picture.

**Q: How do you handle late-arriving events in streaming?**
A: Use watermarks (allow late events within window), windowing strategies (tumbling, sliding, session), and mechanisms to update past aggregations.

**Q: What's the difference between Lambda and Kappa architecture?**
A: Lambda has separate batch and stream layers. Kappa uses stream processing only - replays events from log for reprocessing. Kappa is simpler but reprocessing can be slower.

---

## Quick Summary

```
BATCH PROCESSING:
─────────────────
- Scheduled (daily, hourly)
- Processes accumulated data
- Exact, complete results
- Higher latency
- Hadoop, Spark, Hive
- Use for: analytics, ML, reports

STREAM PROCESSING:
──────────────────
- Continuous (24/7)
- Processes events as they arrive
- Approximate, real-time results
- Very low latency
- Kafka, Flink, Spark Streaming
- Use for: live dashboards, fraud detection

LAMBDA ARCHITECTURE:
────────────────────
- Batch + Stream layers
- Speed layer for real-time
- Batch layer for accuracy
- Serving layer merges both

KAPPA ARCHITECTURE:
───────────────────
- Stream only
- Replay from event log to reprocess
- Simpler, single codebase
```

You now understand Batch vs Stream Processing! 📊
