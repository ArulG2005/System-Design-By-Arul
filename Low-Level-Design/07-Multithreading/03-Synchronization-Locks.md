# Synchronization and Locks

## Why Synchronization?

When multiple threads access **shared mutable state**, we need to ensure:
1. **Mutual Exclusion** - Only one thread accesses critical section at a time
2. **Visibility** - Changes made by one thread are visible to others
3. **Ordering** - Operations happen in expected order

---

## The synchronized Keyword

### Synchronized Methods

```java
public class Counter {
    private int count = 0;
    
    // Only ONE thread can execute this at a time
    public synchronized void increment() {
        count++;
    }
    
    public synchronized int getCount() {
        return count;
    }
}
```

**How it works**:
- Each object has an **intrinsic lock** (monitor)
- `synchronized` method acquires lock on `this`
- Other threads wait until lock is released

### Synchronized Static Methods

```java
public class Config {
    private static String setting;
    
    // Lock is on the CLASS object (Config.class)
    public static synchronized void setSetting(String s) {
        setting = s;
    }
    
    public static synchronized String getSetting() {
        return setting;
    }
}
```

### Synchronized Blocks

```java
public class BankAccount {
    private double balance;
    private final Object lock = new Object();  // Explicit lock object
    
    public void deposit(double amount) {
        synchronized (lock) {  // Lock on specific object
            balance += amount;
        }
    }
    
    public void withdraw(double amount) {
        synchronized (lock) {
            if (balance >= amount) {
                balance -= amount;
            }
        }
    }
    
    // Can use 'this' as lock (same as synchronized method)
    public void transfer(BankAccount target, double amount) {
        synchronized (this) {
            if (balance >= amount) {
                balance -= amount;
                target.deposit(amount);  // Could cause DEADLOCK!
            }
        }
    }
}
```

---

## ReentrantLock

More flexible than `synchronized`:

```java
import java.util.concurrent.locks.ReentrantLock;

public class SafeCounter {
    private int count = 0;
    private final ReentrantLock lock = new ReentrantLock();
    
    public void increment() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();  // ALWAYS unlock in finally!
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

### ReentrantLock Features

#### 1. Try Lock (Non-blocking)

```java
ReentrantLock lock = new ReentrantLock();

if (lock.tryLock()) {  // Returns immediately
    try {
        // Do work
    } finally {
        lock.unlock();
    }
} else {
    // Lock not available, do something else
    System.out.println("Could not acquire lock");
}

// With timeout
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try {
        // Do work
    } finally {
        lock.unlock();
    }
} else {
    System.out.println("Timeout waiting for lock");
}
```

#### 2. Interruptible Lock

```java
try {
    lock.lockInterruptibly();  // Can be interrupted while waiting
    try {
        // Do work
    } finally {
        lock.unlock();
    }
} catch (InterruptedException e) {
    System.out.println("Interrupted while waiting for lock");
}
```

#### 3. Fair Lock

```java
// Threads acquire lock in FIFO order
ReentrantLock fairLock = new ReentrantLock(true);  // true = fair

// Default is non-fair (better performance, but possible starvation)
ReentrantLock unfairLock = new ReentrantLock(false);
```

#### 4. Lock Information

```java
lock.getHoldCount();      // Times current thread holds lock
lock.isHeldByCurrentThread();  // Does current thread hold it?
lock.isLocked();          // Is lock held by any thread?
lock.getQueueLength();    // Threads waiting for lock
```

---

## ReadWriteLock

Optimizes for read-heavy scenarios:
- Multiple readers can read simultaneously
- Writers get exclusive access

```java
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class Cache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    
    public V get(K key) {
        lock.readLock().lock();  // Multiple threads can read
        try {
            return cache.get(key);
        } finally {
            lock.readLock().unlock();
        }
    }
    
    public void put(K key, V value) {
        lock.writeLock().lock();  // Exclusive access
        try {
            cache.put(key, value);
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    public void clear() {
        lock.writeLock().lock();
        try {
            cache.clear();
        } finally {
            lock.writeLock().unlock();
        }
    }
}
```

---

## StampedLock (Java 8+)

Even more optimized, supports optimistic reading:

```java
import java.util.concurrent.locks.StampedLock;

public class Point {
    private double x, y;
    private final StampedLock lock = new StampedLock();
    
    public void move(double deltaX, double deltaY) {
        long stamp = lock.writeLock();
        try {
            x += deltaX;
            y += deltaY;
        } finally {
            lock.unlockWrite(stamp);
        }
    }
    
    // Optimistic read - no locking unless write happened
    public double distanceFromOrigin() {
        long stamp = lock.tryOptimisticRead();  // No blocking!
        double currentX = x;
        double currentY = y;
        
        if (!lock.validate(stamp)) {  // Check if write occurred
            // Fall back to read lock
            stamp = lock.readLock();
            try {
                currentX = x;
                currentY = y;
            } finally {
                lock.unlockRead(stamp);
            }
        }
        return Math.sqrt(currentX * currentX + currentY * currentY);
    }
}
```

---

## Condition Variables

For wait/notify with ReentrantLock:

```java
import java.util.concurrent.locks.*;

public class BoundedBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    
    public BoundedBuffer(int capacity) {
        this.capacity = capacity;
    }
    
    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();  // Wait until not full
            }
            queue.offer(item);
            notEmpty.signal();  // Signal that buffer is not empty
        } finally {
            lock.unlock();
        }
    }
    
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();  // Wait until not empty
            }
            T item = queue.poll();
            notFull.signal();  // Signal that buffer is not full
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

---

## Semaphore

Controls access to limited resources:

```java
import java.util.concurrent.Semaphore;

public class ConnectionPool {
    private final Semaphore semaphore;
    private final List<Connection> connections;
    
    public ConnectionPool(int size) {
        this.semaphore = new Semaphore(size);  // Max concurrent access
        this.connections = new ArrayList<>();
        for (int i = 0; i < size; i++) {
            connections.add(createConnection());
        }
    }
    
    public Connection acquire() throws InterruptedException {
        semaphore.acquire();  // Blocks if no permits available
        return getAvailableConnection();
    }
    
    public void release(Connection conn) {
        returnConnectionToPool(conn);
        semaphore.release();  // Release permit
    }
    
    // Non-blocking version
    public Connection tryAcquire(long timeout, TimeUnit unit) 
            throws InterruptedException {
        if (semaphore.tryAcquire(timeout, unit)) {
            return getAvailableConnection();
        }
        return null;  // No connection available
    }
}
```

---

## CountDownLatch

Wait for N operations to complete:

```java
import java.util.concurrent.CountDownLatch;

public class ParallelInitializer {
    public void initialize() throws InterruptedException {
        int serviceCount = 3;
        CountDownLatch latch = new CountDownLatch(serviceCount);
        
        // Start services in parallel
        new Thread(() -> {
            initDatabase();
            latch.countDown();  // Signal completion
        }).start();
        
        new Thread(() -> {
            initCache();
            latch.countDown();
        }).start();
        
        new Thread(() -> {
            initMessageQueue();
            latch.countDown();
        }).start();
        
        // Wait for all to complete
        latch.await();  // Blocks until count reaches 0
        
        System.out.println("All services initialized!");
    }
    
    // Or with timeout
    public void initializeWithTimeout() throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(3);
        // ... start threads ...
        
        boolean completed = latch.await(30, TimeUnit.SECONDS);
        if (!completed) {
            System.out.println("Initialization timed out!");
        }
    }
}
```

---

## CyclicBarrier

Threads wait for each other at a barrier:

```java
import java.util.concurrent.CyclicBarrier;

public class ParallelProcessor {
    public void process(int[][] matrix) throws Exception {
        int numThreads = 4;
        
        // All threads wait at barrier before proceeding
        CyclicBarrier barrier = new CyclicBarrier(numThreads, () -> {
            // Runs when all threads reach barrier
            System.out.println("All threads reached barrier - combining results");
        });
        
        for (int i = 0; i < numThreads; i++) {
            final int threadId = i;
            new Thread(() -> {
                try {
                    // Phase 1: Process my portion
                    processRows(matrix, threadId);
                    System.out.println("Thread " + threadId + " finished phase 1");
                    barrier.await();  // Wait for others
                    
                    // Phase 2: All threads continue together
                    computeResults(threadId);
                    System.out.println("Thread " + threadId + " finished phase 2");
                    barrier.await();  // Can reuse barrier!
                    
                    // Phase 3
                    finalizeResults(threadId);
                    
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

### CountDownLatch vs CyclicBarrier

| CountDownLatch | CyclicBarrier |
|----------------|---------------|
| One-time use | Reusable |
| Threads count down | Threads wait for each other |
| Wait for N events | N threads synchronize |
| Any code can countdown | Only barrier threads |

---

## Phaser (Advanced)

Dynamic participant count, multiple phases:

```java
import java.util.concurrent.Phaser;

public class PhaserExample {
    public void processInPhases(List<Task> tasks) {
        Phaser phaser = new Phaser(1);  // Register self
        
        for (Task task : tasks) {
            phaser.register();  // Register participant
            new Thread(() -> {
                try {
                    // Phase 0
                    task.prepare();
                    phaser.arriveAndAwaitAdvance();
                    
                    // Phase 1
                    task.process();
                    phaser.arriveAndAwaitAdvance();
                    
                    // Phase 2
                    task.complete();
                    
                } finally {
                    phaser.arriveAndDeregister();  // Done
                }
            }).start();
        }
        
        // Main thread waits for all phases
        phaser.arriveAndDeregister();
    }
}
```

---

## Avoiding Deadlocks

### Rule 1: Lock Ordering

```java
// ALWAYS lock in same order
class Account {
    private final int id;
    private double balance;
    
    public static void transfer(Account from, Account to, double amount) {
        // Order by ID to prevent deadlock
        Account first = from.id < to.id ? from : to;
        Account second = from.id < to.id ? to : from;
        
        synchronized (first) {
            synchronized (second) {
                from.balance -= amount;
                to.balance += amount;
            }
        }
    }
}
```

### Rule 2: Use tryLock with Timeout

```java
public boolean transferWithTryLock(Account from, Account to, double amount) {
    while (true) {
        if (from.lock.tryLock()) {
            try {
                if (to.lock.tryLock()) {
                    try {
                        from.balance -= amount;
                        to.balance += amount;
                        return true;
                    } finally {
                        to.lock.unlock();
                    }
                }
            } finally {
                from.lock.unlock();
            }
        }
        // Back off and retry
        Thread.sleep(random.nextInt(100));
    }
}
```

### Rule 3: Avoid Nested Locks

Keep critical sections small, avoid calling external code while holding locks.

---

## Summary

| Mechanism | Use Case |
|-----------|----------|
| **synchronized** | Simple mutual exclusion |
| **ReentrantLock** | Advanced features, tryLock |
| **ReadWriteLock** | Read-heavy workloads |
| **StampedLock** | Optimistic reads |
| **Semaphore** | Limited resource access |
| **CountDownLatch** | Wait for N events (one-time) |
| **CyclicBarrier** | Threads synchronize (reusable) |
| **Condition** | wait/notify with locks |

---

**Next: Producer-Consumer Pattern →**
