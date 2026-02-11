# Thread Pools and Executors

## Why Thread Pools?

Creating threads is **expensive**:
- Each thread uses memory (~1MB stack)
- OS overhead for thread creation
- Too many threads = context switching overhead
- Unbounded thread creation can crash your app

**Thread Pool Solution**:
- Pre-create a pool of reusable threads
- Submit tasks, pool manages execution
- Limits resource usage
- Improves performance

---

## The Executor Framework

```
┌─────────────────────────────────────────────────────────────┐
│                    Executor Framework                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────┐      ┌──────────────┐     ┌────────────┐  │
│  │   Executor  │◄─────│ExecutorService│◄────│ Executors  │  │
│  │  (interface)│      │  (interface)  │     │  (factory) │  │
│  └──────┬──────┘      └──────────────┘     └────────────┘  │
│         │                    △                              │
│         │                    │                              │
│         ▼                    │                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              ThreadPoolExecutor                      │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐       │   │
│  │  │Thread 1│ │Thread 2│ │Thread 3│ │Thread N│       │   │
│  │  └────────┘ └────────┘ └────────┘ └────────┘       │   │
│  │                                                      │   │
│  │  ┌────────────────────────────────────────────┐    │   │
│  │  │              Task Queue                     │    │   │
│  │  │  [Task] [Task] [Task] [Task] [Task]        │    │   │
│  │  └────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Creating Thread Pools

### 1. Fixed Thread Pool

```java
// Pool with exactly N threads
ExecutorService executor = Executors.newFixedThreadPool(4);

// Submit tasks
executor.submit(() -> System.out.println("Task 1 on " + Thread.currentThread().getName()));
executor.submit(() -> System.out.println("Task 2 on " + Thread.currentThread().getName()));
executor.submit(() -> System.out.println("Task 3 on " + Thread.currentThread().getName()));

// Always shutdown!
executor.shutdown();
```

**Best for**: Known, limited concurrent tasks

### 2. Cached Thread Pool

```java
// Creates threads as needed, reuses idle ones
ExecutorService executor = Executors.newCachedThreadPool();

// Good for many short-lived tasks
for (int i = 0; i < 100; i++) {
    final int taskId = i;
    executor.submit(() -> {
        System.out.println("Task " + taskId);
    });
}

executor.shutdown();
```

**Best for**: Many short tasks with varying load
**Warning**: Can create unlimited threads!

### 3. Single Thread Executor

```java
// Single thread - guarantees sequential execution
ExecutorService executor = Executors.newSingleThreadExecutor();

executor.submit(() -> System.out.println("First"));
executor.submit(() -> System.out.println("Second"));   // Always after First
executor.submit(() -> System.out.println("Third"));    // Always after Second

executor.shutdown();
```

**Best for**: Sequential task processing, event loop

### 4. Scheduled Thread Pool

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// Schedule single execution after delay
scheduler.schedule(() -> {
    System.out.println("Delayed task");
}, 3, TimeUnit.SECONDS);

// Schedule repeated execution
scheduler.scheduleAtFixedRate(() -> {
    System.out.println("Heartbeat: " + System.currentTimeMillis());
}, 0, 1, TimeUnit.SECONDS);  // Initial delay, period

// Schedule with fixed delay between executions
scheduler.scheduleWithFixedDelay(() -> {
    System.out.println("Task with delay after completion");
}, 0, 2, TimeUnit.SECONDS);

// Don't forget to shutdown (after some time)
// scheduler.shutdown();
```

**Best for**: Timers, periodic tasks, scheduled jobs

---

## Custom Thread Pool Configuration

```java
import java.util.concurrent.*;

// Full control over thread pool
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    2,                      // Core pool size (min threads)
    10,                     // Maximum pool size (max threads)
    60L, TimeUnit.SECONDS,  // Keep-alive time for idle threads
    new LinkedBlockingQueue<>(100),  // Work queue (bounded)
    new ThreadFactory() {    // Custom thread factory
        private int count = 0;
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "MyPool-Thread-" + count++);
            t.setDaemon(false);
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy()  // Rejection policy
);

// Use it
executor.submit(() -> System.out.println("Custom pool task"));
executor.shutdown();
```

### ThreadPoolExecutor Parameters

| Parameter | Description |
|-----------|-------------|
| **corePoolSize** | Min threads always kept alive |
| **maxPoolSize** | Max threads when queue is full |
| **keepAliveTime** | Idle time before extra threads die |
| **workQueue** | Queue for pending tasks |
| **threadFactory** | Creates new threads (naming, daemon) |
| **rejectionPolicy** | What to do when pool is full |

---

## Rejection Policies

When pool is full and queue is full:

```java
// 1. AbortPolicy (default) - throws RejectedExecutionException
new ThreadPoolExecutor.AbortPolicy();

// 2. CallerRunsPolicy - caller thread runs the task
new ThreadPoolExecutor.CallerRunsPolicy();

// 3. DiscardPolicy - silently discard task
new ThreadPoolExecutor.DiscardPolicy();

// 4. DiscardOldestPolicy - discard oldest queued task
new ThreadPoolExecutor.DiscardOldestPolicy();

// 5. Custom policy
RejectedExecutionHandler custom = (runnable, executor) -> {
    System.out.println("Task rejected: " + runnable);
    // Log, save to DB, put in different queue, etc.
};
```

---

## Future and Callable

### Getting Results from Tasks

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

// Callable returns a value
Callable<Integer> task = () -> {
    Thread.sleep(1000);  // Simulate work
    return 42;
};

Future<Integer> future = executor.submit(task);

// Check if done
System.out.println("Is done: " + future.isDone());  // false

// Get result (blocks until complete)
Integer result = future.get();  // Waits...
System.out.println("Result: " + result);  // 42

// Or with timeout
try {
    Integer result2 = future.get(500, TimeUnit.MILLISECONDS);
} catch (TimeoutException e) {
    System.out.println("Task took too long!");
}

// Cancel task
future.cancel(true);  // true = interrupt if running

executor.shutdown();
```

### Submitting Multiple Tasks

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

List<Callable<String>> tasks = Arrays.asList(
    () -> { Thread.sleep(1000); return "Task 1"; },
    () -> { Thread.sleep(500);  return "Task 2"; },
    () -> { Thread.sleep(1500); return "Task 3"; }
);

// InvokeAll - wait for ALL to complete
List<Future<String>> futures = executor.invokeAll(tasks);
for (Future<String> f : futures) {
    System.out.println(f.get());
}

// InvokeAny - return FIRST completed result
String firstResult = executor.invokeAny(tasks);
System.out.println("First: " + firstResult);  // Task 2 (fastest)

executor.shutdown();
```

---

## CompletableFuture (Java 8+)

More powerful than Future - supports chaining, combining, async callbacks.

### Basic Usage

```java
import java.util.concurrent.CompletableFuture;

// Run async
CompletableFuture<Void> future1 = CompletableFuture.runAsync(() -> {
    System.out.println("Running in: " + Thread.currentThread().getName());
});

// Supply async (with result)
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
    return "Hello from async!";
});

System.out.println(future2.get());  // Hello from async!
```

### Chaining Operations

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> "Hello")
    .thenApply(s -> s + " World")          // Transform result
    .thenApply(String::toUpperCase);       // Another transform

System.out.println(future.get());  // HELLO WORLD

// With async methods (runs on different thread)
CompletableFuture<String> future2 = CompletableFuture
    .supplyAsync(() -> "Hello")
    .thenApplyAsync(s -> s + " World")     // Async transform
    .thenApplyAsync(String::toUpperCase);
```

### Combining Futures

```java
// Wait for both, combine results
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "World");

CompletableFuture<String> combined = future1.thenCombine(future2, 
    (s1, s2) -> s1 + " " + s2);
System.out.println(combined.get());  // Hello World

// Wait for first completed
CompletableFuture<String> fastest = CompletableFuture.anyOf(future1, future2)
    .thenApply(obj -> (String) obj);

// Wait for all
CompletableFuture<Void> all = CompletableFuture.allOf(future1, future2);
all.join();  // Wait for all to complete
```

### Error Handling

```java
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> {
        if (Math.random() > 0.5) {
            throw new RuntimeException("Oops!");
        }
        return "Success";
    })
    .exceptionally(ex -> {
        System.out.println("Error: " + ex.getMessage());
        return "Default Value";  // Recovery value
    })
    .thenApply(s -> s + " processed");

// Or with handle (gets both result and exception)
CompletableFuture<String> future2 = CompletableFuture
    .supplyAsync(() -> "Result")
    .handle((result, ex) -> {
        if (ex != null) {
            return "Error: " + ex.getMessage();
        }
        return result;
    });
```

---

## Real-World Example: Parallel API Calls

```java
public class ParallelApiCalls {
    private ExecutorService executor = Executors.newFixedThreadPool(10);
    
    public void fetchUserData(String userId) {
        // Fetch multiple APIs in parallel
        CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(
            () -> fetchUser(userId), executor);
            
        CompletableFuture<List<Order>> ordersFuture = CompletableFuture.supplyAsync(
            () -> fetchOrders(userId), executor);
            
        CompletableFuture<List<Review>> reviewsFuture = CompletableFuture.supplyAsync(
            () -> fetchReviews(userId), executor);
        
        // Combine all results
        CompletableFuture<UserProfile> profileFuture = userFuture
            .thenCombine(ordersFuture, (user, orders) -> {
                user.setOrders(orders);
                return user;
            })
            .thenCombine(reviewsFuture, (user, reviews) -> {
                user.setReviews(reviews);
                return new UserProfile(user);
            });
        
        // Handle result
        profileFuture.thenAccept(profile -> {
            System.out.println("User profile loaded: " + profile);
        }).exceptionally(ex -> {
            System.out.println("Failed to load profile: " + ex.getMessage());
            return null;
        });
    }
    
    // Simulated API calls
    private User fetchUser(String id) { /* API call */ return new User(); }
    private List<Order> fetchOrders(String id) { /* API call */ return new ArrayList<>(); }
    private List<Review> fetchReviews(String id) { /* API call */ return new ArrayList<>(); }
}
```

---

## Shutdown Best Practices

```java
public void shutdownExecutor(ExecutorService executor) {
    executor.shutdown();  // Stop accepting new tasks
    
    try {
        // Wait for existing tasks to complete
        if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
            // Force shutdown if tasks don't complete
            executor.shutdownNow();
            
            // Wait again
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                System.err.println("Executor did not terminate");
            }
        }
    } catch (InterruptedException e) {
        // Re-cancel if interrupted
        executor.shutdownNow();
        Thread.currentThread().interrupt();
    }
}
```

---

## Choosing the Right Pool

| Scenario | Pool Type |
|----------|-----------|
| Known fixed concurrency | `newFixedThreadPool(n)` |
| Many short tasks | `newCachedThreadPool()` |
| Sequential processing | `newSingleThreadExecutor()` |
| Scheduled/periodic tasks | `newScheduledThreadPool(n)` |
| Fine-grained control | `ThreadPoolExecutor` |
| CPU-bound tasks | Pool size = CPU cores |
| I/O-bound tasks | Pool size = CPU cores * 2 or more |

---

## Summary

| Concept | Description |
|---------|-------------|
| **Thread Pool** | Reuses threads, limits resource usage |
| **ExecutorService** | Submit tasks, get Futures |
| **Fixed Pool** | Constant number of threads |
| **Cached Pool** | Dynamic threads, reuse idle |
| **Scheduled Pool** | Delayed and periodic tasks |
| **Future** | Handle for async result |
| **CompletableFuture** | Chainable, composable async |
| **Shutdown** | Always shutdown gracefully |

---

**Next: Synchronization and Locks →**
