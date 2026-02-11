# Producer-Consumer Pattern

## What is Producer-Consumer?

A classic concurrency pattern where:
- **Producers** create data/tasks
- **Consumers** process data/tasks
- They communicate through a **shared buffer/queue**

```
┌──────────────┐                      ┌──────────────┐
│   Producer   │                      │   Consumer   │
│              │                      │              │
│  ┌────────┐  │     ┌──────────┐    │  ┌────────┐  │
│  │ Create │──┼────>│  Queue   │────┼─>│Process │  │
│  │  Data  │  │     │ (Buffer) │    │  │  Data  │  │
│  └────────┘  │     └──────────┘    │  └────────┘  │
└──────────────┘                      └──────────────┘
```

---

## Why Use This Pattern?

1. **Decoupling** - Producers and consumers work independently
2. **Buffering** - Handle speed differences between production/consumption
3. **Scalability** - Add more producers or consumers easily
4. **Load balancing** - Multiple consumers share work

---

## Real-World Examples

- **Web Server**: Requests (produced) → Request Queue → Worker threads (consume)
- **Logging**: Log events (produced) → Log queue → File writer (consume)
- **Order Processing**: Orders (produced) → Order queue → Fulfillment (consume)
- **Message Queues**: Kafka, RabbitMQ, etc.

---

## Implementation 1: Using BlockingQueue (Preferred)

```java
import java.util.concurrent.*;

public class ProducerConsumerBlockingQueue {
    
    public static void main(String[] args) {
        // Bounded queue - blocks when full/empty
        BlockingQueue<String> queue = new LinkedBlockingQueue<>(10);
        
        // Producer
        Thread producer = new Thread(() -> {
            int count = 0;
            while (true) {
                try {
                    String item = "Item-" + count++;
                    queue.put(item);  // Blocks if queue is full
                    System.out.println("Produced: " + item);
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }, "Producer");
        
        // Consumer
        Thread consumer = new Thread(() -> {
            while (true) {
                try {
                    String item = queue.take();  // Blocks if queue is empty
                    System.out.println("Consumed: " + item);
                    Thread.sleep(200);  // Consumer is slower
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }, "Consumer");
        
        producer.start();
        consumer.start();
    }
}
```

---

## BlockingQueue Types

| Queue Type | Description |
|------------|-------------|
| `LinkedBlockingQueue` | Linked list, optionally bounded |
| `ArrayBlockingQueue` | Fixed-size array, bounded |
| `PriorityBlockingQueue` | Priority ordering, unbounded |
| `SynchronousQueue` | Zero capacity, direct handoff |
| `DelayQueue` | Elements available after delay |

### BlockingQueue Methods

| Method | Blocks? | Behavior |
|--------|---------|----------|
| `put(e)` | Yes | Wait for space |
| `take()` | Yes | Wait for element |
| `offer(e)` | No | Returns false if full |
| `poll()` | No | Returns null if empty |
| `offer(e, time, unit)` | Timeout | Wait with timeout |
| `poll(time, unit)` | Timeout | Wait with timeout |

---

## Implementation 2: Using wait/notify

```java
public class ProducerConsumerWaitNotify {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int MAX_SIZE = 5;
    
    public synchronized void produce(int item) throws InterruptedException {
        // Wait while queue is full
        while (queue.size() == MAX_SIZE) {
            System.out.println("Queue full, producer waiting...");
            wait();
        }
        
        queue.offer(item);
        System.out.println("Produced: " + item + ", Queue size: " + queue.size());
        
        // Notify consumers
        notifyAll();
    }
    
    public synchronized int consume() throws InterruptedException {
        // Wait while queue is empty
        while (queue.isEmpty()) {
            System.out.println("Queue empty, consumer waiting...");
            wait();
        }
        
        int item = queue.poll();
        System.out.println("Consumed: " + item + ", Queue size: " + queue.size());
        
        // Notify producers
        notifyAll();
        
        return item;
    }
    
    public static void main(String[] args) {
        ProducerConsumerWaitNotify pc = new ProducerConsumerWaitNotify();
        
        // Producer thread
        new Thread(() -> {
            for (int i = 0; i < 20; i++) {
                try {
                    pc.produce(i);
                    Thread.sleep(50);
                } catch (InterruptedException e) {
                    break;
                }
            }
        }).start();
        
        // Consumer thread
        new Thread(() -> {
            for (int i = 0; i < 20; i++) {
                try {
                    pc.consume();
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    break;
                }
            }
        }).start();
    }
}
```

---

## Implementation 3: Using ReentrantLock and Condition

```java
import java.util.concurrent.locks.*;

public class ProducerConsumerReentrantLock<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    
    private final Lock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    
    public ProducerConsumerReentrantLock(int capacity) {
        this.capacity = capacity;
    }
    
    public void produce(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();  // Wait for space
            }
            queue.offer(item);
            System.out.println("Produced: " + item);
            notEmpty.signal();  // Signal consumer
        } finally {
            lock.unlock();
        }
    }
    
    public T consume() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();  // Wait for item
            }
            T item = queue.poll();
            System.out.println("Consumed: " + item);
            notFull.signal();  // Signal producer
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## Multiple Producers and Consumers

```java
public class MultiProducerConsumer {
    
    public static void main(String[] args) {
        BlockingQueue<Task> taskQueue = new LinkedBlockingQueue<>(100);
        
        // Multiple producers
        for (int i = 0; i < 3; i++) {
            final int producerId = i;
            new Thread(() -> {
                while (true) {
                    try {
                        Task task = new Task("Task from P" + producerId);
                        taskQueue.put(task);
                        System.out.println("P" + producerId + " produced: " + task);
                        Thread.sleep((long) (Math.random() * 100));
                    } catch (InterruptedException e) {
                        break;
                    }
                }
            }, "Producer-" + i).start();
        }
        
        // Multiple consumers
        for (int i = 0; i < 5; i++) {
            final int consumerId = i;
            new Thread(() -> {
                while (true) {
                    try {
                        Task task = taskQueue.take();
                        System.out.println("C" + consumerId + " processing: " + task);
                        task.process();
                    } catch (InterruptedException e) {
                        break;
                    }
                }
            }, "Consumer-" + i).start();
        }
    }
}

class Task {
    private String name;
    
    public Task(String name) {
        this.name = name;
    }
    
    public void process() {
        // Simulate processing
        try {
            Thread.sleep((long) (Math.random() * 200));
        } catch (InterruptedException e) {}
    }
    
    @Override
    public String toString() {
        return name;
    }
}
```

---

## Using ExecutorService (Best in Production)

```java
public class ExecutorProducerConsumer {
    
    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Runnable> taskQueue = new LinkedBlockingQueue<>(100);
        
        // Consumer pool - processes tasks
        ExecutorService consumerPool = new ThreadPoolExecutor(
            5, 10,                          // core/max threads
            60, TimeUnit.SECONDS,           // keep-alive
            taskQueue,                       // work queue
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
        
        // Producer - submits tasks
        ExecutorService producerPool = Executors.newFixedThreadPool(3);
        
        for (int i = 0; i < 3; i++) {
            final int producerId = i;
            producerPool.submit(() -> {
                for (int j = 0; j < 100; j++) {
                    final int taskId = j;
                    consumerPool.submit(() -> {
                        String name = "P" + producerId + "-Task" + taskId;
                        System.out.println(Thread.currentThread().getName() 
                            + " processing " + name);
                        try {
                            Thread.sleep(50);
                        } catch (InterruptedException e) {}
                    });
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {}
                }
            });
        }
        
        // Shutdown
        producerPool.shutdown();
        producerPool.awaitTermination(1, TimeUnit.MINUTES);
        consumerPool.shutdown();
        consumerPool.awaitTermination(1, TimeUnit.MINUTES);
    }
}
```

---

## Real-World Example: Log Processing System

```java
public class LogProcessor {
    private final BlockingQueue<LogEntry> logQueue;
    private final ExecutorService writers;
    private volatile boolean running = true;
    
    public LogProcessor(int queueSize, int writerCount) {
        this.logQueue = new LinkedBlockingQueue<>(queueSize);
        this.writers = Executors.newFixedThreadPool(writerCount);
        
        // Start writer threads
        for (int i = 0; i < writerCount; i++) {
            writers.submit(this::writeLoop);
        }
    }
    
    // Called by application threads (producers)
    public void log(String level, String message) {
        LogEntry entry = new LogEntry(level, message, System.currentTimeMillis());
        
        // Non-blocking offer with fallback
        if (!logQueue.offer(entry)) {
            // Queue full - could drop, write to stderr, or block
            System.err.println("Log queue full, dropping: " + entry);
        }
    }
    
    // Writer thread loop (consumer)
    private void writeLoop() {
        while (running || !logQueue.isEmpty()) {
            try {
                LogEntry entry = logQueue.poll(100, TimeUnit.MILLISECONDS);
                if (entry != null) {
                    writeToFile(entry);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }
    
    private void writeToFile(LogEntry entry) {
        // Write to log file
        System.out.println(String.format("[%s] %s: %s",
            new java.util.Date(entry.timestamp),
            entry.level,
            entry.message));
    }
    
    public void shutdown() {
        running = false;
        writers.shutdown();
        try {
            writers.awaitTermination(30, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            writers.shutdownNow();
        }
    }
    
    static class LogEntry {
        final String level;
        final String message;
        final long timestamp;
        
        LogEntry(String level, String message, long timestamp) {
            this.level = level;
            this.message = message;
            this.timestamp = timestamp;
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        LogProcessor processor = new LogProcessor(1000, 2);
        
        // Simulate multiple threads logging
        ExecutorService app = Executors.newFixedThreadPool(10);
        for (int i = 0; i < 100; i++) {
            final int msgId = i;
            app.submit(() -> {
                processor.log("INFO", "Message " + msgId);
            });
        }
        
        app.shutdown();
        app.awaitTermination(10, TimeUnit.SECONDS);
        
        Thread.sleep(1000);  // Let logs flush
        processor.shutdown();
    }
}
```

---

## Real-World Example: Order Processing Pipeline

```java
public class OrderPipeline {
    
    // Stage queues
    private final BlockingQueue<Order> validationQueue = new LinkedBlockingQueue<>(100);
    private final BlockingQueue<Order> paymentQueue = new LinkedBlockingQueue<>(100);
    private final BlockingQueue<Order> fulfillmentQueue = new LinkedBlockingQueue<>(100);
    
    private final ExecutorService validators;
    private final ExecutorService paymentProcessors;
    private final ExecutorService fulfillers;
    
    public OrderPipeline() {
        validators = Executors.newFixedThreadPool(2);
        paymentProcessors = Executors.newFixedThreadPool(3);
        fulfillers = Executors.newFixedThreadPool(5);
        
        // Start pipeline stages
        startValidators(2);
        startPaymentProcessors(3);
        startFulfillers(5);
    }
    
    // Entry point - submit order
    public void submitOrder(Order order) throws InterruptedException {
        validationQueue.put(order);
        System.out.println("Order " + order.id + " submitted");
    }
    
    private void startValidators(int count) {
        for (int i = 0; i < count; i++) {
            validators.submit(() -> {
                while (!Thread.interrupted()) {
                    try {
                        Order order = validationQueue.take();
                        if (validateOrder(order)) {
                            order.status = "VALIDATED";
                            paymentQueue.put(order);
                            System.out.println("Order " + order.id + " validated");
                        } else {
                            order.status = "REJECTED";
                            System.out.println("Order " + order.id + " rejected");
                        }
                    } catch (InterruptedException e) {
                        break;
                    }
                }
            });
        }
    }
    
    private void startPaymentProcessors(int count) {
        for (int i = 0; i < count; i++) {
            paymentProcessors.submit(() -> {
                while (!Thread.interrupted()) {
                    try {
                        Order order = paymentQueue.take();
                        if (processPayment(order)) {
                            order.status = "PAID";
                            fulfillmentQueue.put(order);
                            System.out.println("Order " + order.id + " paid");
                        } else {
                            order.status = "PAYMENT_FAILED";
                            System.out.println("Order " + order.id + " payment failed");
                        }
                    } catch (InterruptedException e) {
                        break;
                    }
                }
            });
        }
    }
    
    private void startFulfillers(int count) {
        for (int i = 0; i < count; i++) {
            fulfillers.submit(() -> {
                while (!Thread.interrupted()) {
                    try {
                        Order order = fulfillmentQueue.take();
                        fulfillOrder(order);
                        order.status = "COMPLETED";
                        System.out.println("Order " + order.id + " completed!");
                    } catch (InterruptedException e) {
                        break;
                    }
                }
            });
        }
    }
    
    // Simulated stages
    private boolean validateOrder(Order order) {
        sleep(50);
        return order.amount > 0;
    }
    
    private boolean processPayment(Order order) {
        sleep(100);
        return Math.random() > 0.1;  // 90% success
    }
    
    private void fulfillOrder(Order order) {
        sleep(200);
    }
    
    private void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) {}
    }
    
    public void shutdown() {
        validators.shutdownNow();
        paymentProcessors.shutdownNow();
        fulfillers.shutdownNow();
    }
    
    static class Order {
        String id;
        double amount;
        String status;
        
        Order(String id, double amount) {
            this.id = id;
            this.amount = amount;
            this.status = "NEW";
        }
    }
    
    public static void main(String[] args) throws InterruptedException {
        OrderPipeline pipeline = new OrderPipeline();
        
        // Submit orders
        for (int i = 0; i < 20; i++) {
            pipeline.submitOrder(new Order("ORD-" + i, 100.0 + i));
        }
        
        Thread.sleep(5000);
        pipeline.shutdown();
    }
}
```

---

## Summary

| Approach | Pros | Cons |
|----------|------|------|
| `BlockingQueue` | Simple, thread-safe, flexible | Less control |
| `wait/notify` | Fine control | Error-prone, harder to get right |
| `ReentrantLock` | More features, conditions | More code |
| `ExecutorService` | Production-ready, scalable | Abstraction overhead |

### Best Practices:
1. Use `BlockingQueue` - it's designed for this
2. Handle `InterruptedException` properly
3. Consider bounded queues to prevent memory issues
4. Use ExecutorService for real applications
5. Implement graceful shutdown

---

**Next: Concurrent Collections →**
