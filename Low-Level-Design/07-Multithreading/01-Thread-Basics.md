# Multithreading and Concurrency in Java

## Why Multithreading Matters for LLD

As an SDE-3, you MUST understand concurrency:
- **Scalability**: Handle multiple users/requests simultaneously
- **Performance**: Utilize multi-core processors
- **Responsiveness**: Keep UI responsive while processing
- **Real systems**: All production systems use threading

---

## What is a Thread?

A **thread** is the smallest unit of execution within a process.

```
┌──────────────────────────────────────────────────┐
│                    PROCESS                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Thread 1 │  │ Thread 2 │  │ Thread 3 │       │
│  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │       │
│  │ │Stack │ │  │ │Stack │ │  │ │Stack │ │       │
│  │ └──────┘ │  │ └──────┘ │  │ └──────┘ │       │
│  └──────────┘  └──────────┘  └──────────┘       │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │            SHARED HEAP MEMORY             │   │
│  │    (Objects, Data Structures, etc.)       │   │
│  └──────────────────────────────────────────┘   │
└──────────────────────────────────────────────────┘
```

---

## Creating Threads

### Method 1: Extend Thread class

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread running: " + getName());
    }
}

// Usage
MyThread thread = new MyThread();
thread.start();  // NOT run()! start() creates new thread
```

### Method 2: Implement Runnable (Preferred)

```java
class MyTask implements Runnable {
    @Override
    public void run() {
        System.out.println("Task running in: " + Thread.currentThread().getName());
    }
}

// Usage
Thread thread = new Thread(new MyTask());
thread.start();

// Or with lambda
Thread thread2 = new Thread(() -> {
    System.out.println("Lambda task in: " + Thread.currentThread().getName());
});
thread2.start();
```

### Method 3: Callable (Returns a value)

```java
import java.util.concurrent.*;

class SumTask implements Callable<Integer> {
    private int[] numbers;
    
    public SumTask(int[] numbers) {
        this.numbers = numbers;
    }
    
    @Override
    public Integer call() throws Exception {
        int sum = 0;
        for (int n : numbers) sum += n;
        return sum;
    }
}

// Usage with ExecutorService
ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(new SumTask(new int[]{1, 2, 3, 4, 5}));

Integer result = future.get();  // Blocks until result ready
System.out.println("Sum: " + result);  // Sum: 15

executor.shutdown();
```

---

## Thread Lifecycle

```
┌─────────────┐
│    NEW      │  Thread created but not started
└──────┬──────┘
       │ start()
       ▼
┌─────────────┐
│  RUNNABLE   │  Ready to run, waiting for CPU
└──────┬──────┘
       │ CPU scheduled
       ▼
┌─────────────┐
│   RUNNING   │  Actually executing
└──────┬──────┘
       │
       ├─────sleep()/wait()────────────┐
       │                               ▼
       │                        ┌─────────────┐
       │                        │   WAITING   │
       │                        │  /BLOCKED   │
       │                        └──────┬──────┘
       │                               │ notify()/time up
       │◄──────────────────────────────┘
       │
       │ run() completes
       ▼
┌─────────────┐
│ TERMINATED  │  Thread finished
└─────────────┘
```

---

## Thread Control Methods

```java
public class ThreadControlDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            for (int i = 0; i < 5; i++) {
                System.out.println("Working... " + i);
                try {
                    Thread.sleep(1000);  // Pause for 1 second
                } catch (InterruptedException e) {
                    System.out.println("Interrupted!");
                    return;
                }
            }
        });
        
        worker.start();
        
        // Main thread waits for worker to finish
        worker.join();  // Blocks until worker completes
        
        System.out.println("Worker finished!");
    }
}
```

### Key Methods:
- `start()` - Begin thread execution
- `sleep(ms)` - Pause current thread
- `join()` - Wait for another thread to complete
- `interrupt()` - Signal thread to stop
- `isAlive()` - Check if thread is still running
- `yield()` - Hint to scheduler to let other threads run

---

## Race Conditions and Thread Safety

### The Problem

```java
// UNSAFE! Race condition! ❌
class Counter {
    private int count = 0;
    
    public void increment() {
        count++;  // NOT atomic! Read-Modify-Write
    }
    
    public int getCount() {
        return count;
    }
}

// Demo
public static void main(String[] args) throws InterruptedException {
    Counter counter = new Counter();
    
    Thread t1 = new Thread(() -> {
        for (int i = 0; i < 10000; i++) counter.increment();
    });
    
    Thread t2 = new Thread(() -> {
        for (int i = 0; i < 10000; i++) counter.increment();
    });
    
    t1.start();
    t2.start();
    t1.join();
    t2.join();
    
    // Expected: 20000, Actual: Random number less than 20000!
    System.out.println("Count: " + counter.getCount());
}
```

### Why Race Condition Happens

```
Thread 1                   Thread 2                   count
────────                   ────────                   ─────
read count (0)             -                          0
-                          read count (0)             0
increment (1)              -                          0
-                          increment (1)              0
write count (1)            -                          1
-                          write count (1)            1  ← Lost update!
```

---

## Solutions to Thread Safety

### 1. Synchronized Methods

```java
class SafeCounter {
    private int count = 0;
    
    public synchronized void increment() {
        count++;
    }
    
    public synchronized int getCount() {
        return count;
    }
}
```

### 2. Synchronized Blocks

```java
class SafeCounter {
    private int count = 0;
    private final Object lock = new Object();
    
    public void increment() {
        synchronized (lock) {
            count++;
        }
    }
    
    public int getCount() {
        synchronized (lock) {
            return count;
        }
    }
}
```

### 3. Atomic Classes

```java
import java.util.concurrent.atomic.AtomicInteger;

class SafeCounter {
    private AtomicInteger count = new AtomicInteger(0);
    
    public void increment() {
        count.incrementAndGet();  // Atomic operation
    }
    
    public int getCount() {
        return count.get();
    }
}
```

### 4. ReentrantLock

```java
import java.util.concurrent.locks.ReentrantLock;

class SafeCounter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();
    
    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();  // Always unlock in finally!
        }
    }
    
    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## Comparison of Synchronization Methods

| Method | Pros | Cons |
|--------|------|------|
| `synchronized` | Simple, automatic unlock | Less flexible |
| `ReentrantLock` | tryLock, timeout, fairness | Manual unlock required |
| `AtomicInteger` | Lock-free, fast | Only for single variables |
| `volatile` | Simple visibility | No atomicity |

---

## The volatile Keyword

Ensures visibility across threads, but NOT atomicity:

```java
class VisibilityDemo {
    private volatile boolean running = true;  // Changes visible to all threads
    
    public void start() {
        new Thread(() -> {
            while (running) {  // Will see updates from other threads
                // do work
            }
            System.out.println("Stopped!");
        }).start();
    }
    
    public void stop() {
        running = false;  // Immediately visible to other threads
    }
}
```

### When to use volatile:
- Simple flags (stop signals)
- One writer, multiple readers
- NOT for compound operations (check-then-act, read-modify-write)

---

## Common Multithreading Problems

### 1. Deadlock

```java
// DEADLOCK! ❌
class DeadlockExample {
    private final Object lock1 = new Object();
    private final Object lock2 = new Object();
    
    public void method1() {
        synchronized (lock1) {
            System.out.println("Holding lock1...");
            synchronized (lock2) {  // Waiting for lock2
                System.out.println("Holding lock1 and lock2");
            }
        }
    }
    
    public void method2() {
        synchronized (lock2) {  // Holding lock2
            System.out.println("Holding lock2...");
            synchronized (lock1) {  // Waiting for lock1 - DEADLOCK!
                System.out.println("Holding lock2 and lock1");
            }
        }
    }
}
```

### Deadlock Prevention:
1. **Lock ordering** - Always acquire locks in same order
2. **Lock timeout** - Use tryLock with timeout
3. **Avoid nested locks** - Keep critical sections small

```java
// FIXED: Consistent lock ordering
public void method1() {
    synchronized (lock1) {
        synchronized (lock2) {
            // ...
        }
    }
}

public void method2() {
    synchronized (lock1) {  // Same order!
        synchronized (lock2) {
            // ...
        }
    }
}
```

### 2. Starvation

A thread never gets CPU time because other threads monopolize it.

### 3. Livelock

Threads respond to each other but make no progress.

---

## Thread-Safe Collections

### Regular Collections - NOT Thread Safe!

```java
// NOT thread-safe ❌
List<String> list = new ArrayList<>();
Map<String, Integer> map = new HashMap<>();
```

### Thread-Safe Alternatives

```java
// Option 1: Synchronized wrappers (old way)
List<String> syncList = Collections.synchronizedList(new ArrayList<>());

// Option 2: Concurrent collections (preferred) ✓
List<String> list = new CopyOnWriteArrayList<>();
Map<String, Integer> map = new ConcurrentHashMap<>();
Queue<String> queue = new ConcurrentLinkedQueue<>();
BlockingQueue<String> blockingQueue = new LinkedBlockingQueue<>();
```

### ConcurrentHashMap Example

```java
ConcurrentHashMap<String, Integer> wordCount = new ConcurrentHashMap<>();

// Thread-safe operations
wordCount.putIfAbsent("hello", 0);
wordCount.compute("hello", (key, val) -> val + 1);
wordCount.merge("hello", 1, Integer::sum);  // Most elegant

// Atomic check-then-act
wordCount.computeIfAbsent("world", key -> 0);
```

---

## Summary: Thread Basics

| Concept | Key Point |
|---------|-----------|
| **Thread Creation** | Use Runnable/Callable, not extend Thread |
| **Race Condition** | Multiple threads + shared mutable state |
| **synchronized** | One thread at a time |
| **volatile** | Visibility only, not atomicity |
| **Atomic classes** | Lock-free atomic operations |
| **Deadlock** | Circular wait for locks |
| **Thread-safe collections** | Use java.util.concurrent |

---

## Best Practices

1. **Prefer immutable objects** - They're automatically thread-safe
2. **Minimize shared mutable state** - Less sharing = less problems
3. **Use higher-level concurrency utilities** - Executors, concurrent collections
4. **Keep synchronized blocks small** - Reduce contention
5. **Document thread-safety** - Make it clear in API
6. **Prefer Runnable to Thread** - Separation of task and execution

---

**Next: Thread Pools and Executors →**
