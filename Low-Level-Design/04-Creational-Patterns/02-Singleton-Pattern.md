# Singleton Design Pattern

## Intent

> **Ensure a class has only ONE instance and provide a global point of access to it.**

---

## The Problem

Sometimes you need exactly ONE instance of a class:
- Database connection pool
- Configuration manager
- Logger
- Cache
- Thread pool

**Why only one?**
- Shared resource management
- Consistent state across application
- Prevent resource wastage

---

## Simple Analogy

Think of the **President of a country**:
- There can only be ONE President at a time
- Everyone in the country accesses the SAME President
- You can't create a new President whenever you want

---

## Real-World Examples

1. **Database Connection Pool**: One pool shared by all threads
2. **Logger**: One logger collecting all logs
3. **Configuration**: One config object with all settings
4. **Cache**: One cache shared across application
5. **Print Spooler**: One spooler managing all print jobs

---

## Basic Implementation

### Version 1: Eager Initialization

Instance created when class is loaded.

```java
public class Singleton {
    // Instance created immediately when class loads
    private static final Singleton INSTANCE = new Singleton();
    
    // Private constructor - no one else can create
    private Singleton() {
        System.out.println("Singleton instance created");
    }
    
    // Global access point
    public static Singleton getInstance() {
        return INSTANCE;
    }
    
    public void doSomething() {
        System.out.println("Doing something...");
    }
}

// Usage
Singleton s1 = Singleton.getInstance();
Singleton s2 = Singleton.getInstance();
System.out.println(s1 == s2);  // true - same instance!
```

**Pros:**
- Simple
- Thread-safe automatically
- No synchronization needed

**Cons:**
- Instance created even if never used
- Cannot handle exceptions in constructor

---

### Version 2: Lazy Initialization (NOT Thread-Safe!)

Instance created only when first needed.

```java
public class Singleton {
    private static Singleton instance;
    
    private Singleton() {
        System.out.println("Singleton instance created");
    }
    
    public static Singleton getInstance() {
        if (instance == null) {  // Created only when first called
            instance = new Singleton();
        }
        return instance;
    }
}
```

**⚠️ Problem: NOT thread-safe!**

```java
// Two threads call getInstance() at the same time:
Thread 1: if (instance == null)  -> true
Thread 2: if (instance == null)  -> true (hasn't been created yet)
Thread 1: instance = new Singleton();  // Creates first
Thread 2: instance = new Singleton();  // Creates second! 💥
```

---

### Version 3: Thread-Safe with Synchronized (Slow)

```java
public class Singleton {
    private static Singleton instance;
    
    private Singleton() {}
    
    // synchronized makes it thread-safe but SLOW
    public static synchronized Singleton getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }
        return instance;
    }
}
```

**Cons:**
- Every call pays synchronization cost
- Slow performance (100x slower than non-synchronized)

---

### Version 4: Double-Checked Locking (Recommended)

```java
public class Singleton {
    // volatile ensures visibility across threads
    private static volatile Singleton instance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {  // First check (no locking)
            synchronized (Singleton.class) {
                if (instance == null) {  // Second check (with locking)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

**How it works:**
1. First check without synchronization (fast path)
2. If null, synchronize and check again
3. Only first call pays synchronization cost
4. `volatile` keyword ensures proper visibility

**Why volatile?**
Without `volatile`, one thread might see a partially constructed object due to instruction reordering.

---

### Version 5: Bill Pugh Singleton (Best for Java)

Uses inner static helper class.

```java
public class Singleton {
    
    private Singleton() {}
    
    // Inner class not loaded until getInstance() called
    private static class SingletonHelper {
        private static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstance() {
        return SingletonHelper.INSTANCE;
    }
}
```

**Why is this the best?**
- Lazy loading (inner class loaded only when accessed)
- Thread-safe (JVM handles class loading thread-safely)
- No synchronization overhead
- Simple and clean

---

### Version 6: Enum Singleton (Simplest and Safest)

```java
public enum Singleton {
    INSTANCE;
    
    public void doSomething() {
        System.out.println("Doing something...");
    }
}

// Usage
Singleton.INSTANCE.doSomething();
```

**Pros:**
- Simplest implementation
- Thread-safe by default
- Prevents reflection attacks
- Prevents serialization issues
- Recommended by Joshua Bloch (Effective Java author)

**Cons:**
- Cannot extend other classes
- Less flexibility

---

## Real-World Examples

### Example 1: Logger Singleton

```java
public class Logger {
    private static volatile Logger instance;
    private List<String> logs = new ArrayList<>();
    
    private Logger() {}
    
    public static Logger getInstance() {
        if (instance == null) {
            synchronized (Logger.class) {
                if (instance == null) {
                    instance = new Logger();
                }
            }
        }
        return instance;
    }
    
    public void log(String message) {
        String timestamp = LocalDateTime.now().toString();
        String logEntry = timestamp + ": " + message;
        logs.add(logEntry);
        System.out.println(logEntry);
    }
    
    public void info(String message) {
        log("[INFO] " + message);
    }
    
    public void error(String message) {
        log("[ERROR] " + message);
    }
    
    public List<String> getLogs() {
        return new ArrayList<>(logs);
    }
}

// Usage anywhere in application
Logger.getInstance().info("Application started");
Logger.getInstance().error("Something went wrong");
```

### Example 2: Configuration Manager

```java
public class ConfigManager {
    private static ConfigManager instance;
    private Properties properties;
    
    private ConfigManager() {
        properties = new Properties();
        loadConfig();
    }
    
    private static class Holder {
        private static final ConfigManager INSTANCE = new ConfigManager();
    }
    
    public static ConfigManager getInstance() {
        return Holder.INSTANCE;
    }
    
    private void loadConfig() {
        try (InputStream input = getClass().getClassLoader()
                .getResourceAsStream("config.properties")) {
            if (input != null) {
                properties.load(input);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
    public String get(String key) {
        return properties.getProperty(key);
    }
    
    public String get(String key, String defaultValue) {
        return properties.getProperty(key, defaultValue);
    }
    
    public int getInt(String key, int defaultValue) {
        String value = properties.getProperty(key);
        return value != null ? Integer.parseInt(value) : defaultValue;
    }
}

// Usage
String dbUrl = ConfigManager.getInstance().get("database.url");
int maxConnections = ConfigManager.getInstance().getInt("pool.size", 10);
```

### Example 3: Database Connection Pool

```java
public class ConnectionPool {
    private static volatile ConnectionPool instance;
    private List<Connection> availableConnections;
    private List<Connection> usedConnections;
    private static final int MAX_POOL_SIZE = 10;
    
    private ConnectionPool() {
        availableConnections = new ArrayList<>();
        usedConnections = new ArrayList<>();
        initializePool();
    }
    
    private void initializePool() {
        for (int i = 0; i < MAX_POOL_SIZE; i++) {
            availableConnections.add(createConnection());
        }
    }
    
    private Connection createConnection() {
        // Create actual database connection
        try {
            return DriverManager.getConnection(
                "jdbc:mysql://localhost/db", "user", "password"
            );
        } catch (SQLException e) {
            throw new RuntimeException(e);
        }
    }
    
    public static ConnectionPool getInstance() {
        if (instance == null) {
            synchronized (ConnectionPool.class) {
                if (instance == null) {
                    instance = new ConnectionPool();
                }
            }
        }
        return instance;
    }
    
    public synchronized Connection getConnection() {
        if (availableConnections.isEmpty()) {
            throw new RuntimeException("No connections available");
        }
        Connection conn = availableConnections.remove(
            availableConnections.size() - 1
        );
        usedConnections.add(conn);
        return conn;
    }
    
    public synchronized void releaseConnection(Connection conn) {
        usedConnections.remove(conn);
        availableConnections.add(conn);
    }
    
    public int getAvailableCount() {
        return availableConnections.size();
    }
}

// Usage
Connection conn = ConnectionPool.getInstance().getConnection();
try {
    // Use connection
} finally {
    ConnectionPool.getInstance().releaseConnection(conn);
}
```

---

## Breaking Singleton (And How to Prevent)

### 1. Reflection Attack

```java
// Breaking singleton using reflection
Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
constructor.setAccessible(true);
Singleton instance2 = constructor.newInstance();  // New instance created!
```

**Prevention:**
```java
private Singleton() {
    if (instance != null) {
        throw new RuntimeException("Use getInstance() instead!");
    }
}
```

### 2. Serialization Attack

```java
// If singleton is serializable, deserialization creates new instance
ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("singleton.ser"));
out.writeObject(singleton);

ObjectInputStream in = new ObjectInputStream(new FileInputStream("singleton.ser"));
Singleton instance2 = (Singleton) in.readObject();  // New instance!
```

**Prevention:**
```java
public class Singleton implements Serializable {
    private static final Singleton INSTANCE = new Singleton();
    
    protected Object readResolve() {
        return INSTANCE;  // Return existing instance
    }
}
```

### 3. Cloning Attack

**Prevention:**
```java
@Override
protected Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException("Cannot clone Singleton!");
}
```

---

## Singleton vs Static Class

| Aspect | Singleton | Static Class |
|--------|-----------|--------------|
| Can implement interfaces | ✓ Yes | ✗ No |
| Can be passed as parameter | ✓ Yes | ✗ No |
| Lazy initialization | ✓ Yes | ✗ No |
| Can extend other classes | ✓ Yes | ✗ No |
| Can be serialized | ✓ Yes | ✗ No |
| Testing/Mocking | ✓ Easier | ✗ Harder |

**Use Singleton when you need OOP features. Use static class for pure utility functions.**

---

## When to Use Singleton

### ✅ Good Use Cases:
- Logger
- Configuration manager
- Connection pool
- Cache manager
- Device drivers (printer spooler)

### ❌ Bad Use Cases:
- When you might need multiple instances later
- When it introduces tight coupling
- When it makes testing difficult
- As a way to avoid dependency injection

---

## Singleton vs Dependency Injection

Modern frameworks prefer DI over Singleton:

```java
// Instead of Singleton
public class UserService {
    private final Logger logger = Logger.getInstance();
}

// Use Dependency Injection
public class UserService {
    private final Logger logger;
    
    public UserService(Logger logger) {
        this.logger = logger;
    }
}
```

**Why DI is better:**
- Easier to test (inject mocks)
- Explicit dependencies
- Less coupling
- Framework manages lifecycle

---

## UML Diagram

```
┌──────────────────────────────────┐
│           Singleton              │
├──────────────────────────────────┤
│ - instance: Singleton {static}   │
├──────────────────────────────────┤
│ - Singleton()                    │
│ + getInstance(): Singleton       │
│ + doSomething(): void            │
└──────────────────────────────────┘
```

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Intent** | One instance, global access |
| **Key Parts** | Private constructor, static instance, public getter |
| **Best Java Impl** | Bill Pugh (inner class) or Enum |
| **Thread Safety** | Use volatile + double-check or inner class |
| **Use When** | Shared resources (logger, config, pool) |
| **Avoid When** | Tight coupling, testing problems |

### Remember:
- Private constructor prevents external instantiation
- Static method provides global access
- Be careful with thread safety
- Consider DI frameworks instead

---

**Next: Factory Method Pattern →**
