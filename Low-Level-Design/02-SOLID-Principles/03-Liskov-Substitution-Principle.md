# Liskov Substitution Principle (LSP)

## The Third SOLID Principle

> **"Objects of a superclass should be replaceable with objects of its subclasses without breaking the application."**
> — Barbara Liskov

---

## What Does This Mean?

If you have a class `Parent` and a class `Child extends Parent`, then anywhere you use `Parent`, you should be able to use `Child` without any problems.

**In simple words**: Children should behave like their parents (at least from the outside).

---

## Simple Analogy

Think of a **car rental company**:
- You book a "4-seater car" (parent class)
- They can give you a Honda, Toyota, or BMW (child classes)
- Any of these will work because they ALL behave like "4-seater cars"

But if they give you a **motorcycle** (violates LSP):
- It's technically a vehicle
- But it can't seat 4 people!
- Your family trip is ruined

---

## The Classic Example: Rectangle and Square

### ❌ BAD: Violating LSP

```java
class Rectangle {
    protected int width;
    protected int height;
    
    public void setWidth(int width) {
        this.width = width;
    }
    
    public void setHeight(int height) {
        this.height = height;
    }
    
    public int getWidth() { return width; }
    public int getHeight() { return height; }
    
    public int getArea() {
        return width * height;
    }
}

// Square IS-A Rectangle mathematically, but...
class Square extends Rectangle {
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width;  // Keep it square!
    }
    
    @Override
    public void setHeight(int height) {
        this.height = height;
        this.width = height;  // Keep it square!
    }
}
```

### The Problem:

```java
void testRectangle(Rectangle rect) {
    rect.setWidth(5);
    rect.setHeight(4);
    
    // For Rectangle: area = 5 * 4 = 20 ✓
    // For Square:    area = 4 * 4 = 16 ✗ (setHeight changed width!)
    
    assert rect.getArea() == 20;  // FAILS for Square!
}

Rectangle rect = new Rectangle();
testRectangle(rect);  // Works! ✓

Rectangle square = new Square();  // LSP violation!
testRectangle(square);  // Fails! ✗
```

The `Square` class changes the expected behavior of `Rectangle`. When we set width and height separately, we expect them to be independent. Square breaks this expectation!

---

### ✅ GOOD: Following LSP

```java
// Don't use inheritance here - use a common interface
interface Shape {
    int getArea();
}

class Rectangle implements Shape {
    private int width;
    private int height;
    
    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }
    
    public int getWidth() { return width; }
    public int getHeight() { return height; }
    
    @Override
    public int getArea() {
        return width * height;
    }
}

class Square implements Shape {
    private int side;
    
    public Square(int side) {
        this.side = side;
    }
    
    public int getSide() { return side; }
    
    @Override
    public int getArea() {
        return side * side;
    }
}

// Now both work correctly!
void printArea(Shape shape) {
    System.out.println("Area: " + shape.getArea());
}

printArea(new Rectangle(5, 4));  // Area: 20
printArea(new Square(5));        // Area: 25
```

---

## Another Example: Bird and Penguin

### ❌ BAD: Violating LSP

```java
class Bird {
    public void fly() {
        System.out.println("Flying high!");
    }
    
    public void eat() {
        System.out.println("Eating...");
    }
}

class Sparrow extends Bird {
    // Works fine - sparrows can fly
}

class Penguin extends Bird {
    @Override
    public void fly() {
        // Penguins can't fly! What do we do?
        throw new UnsupportedOperationException("Penguins can't fly!");
    }
}
```

### The Problem:

```java
void makeBirdFly(Bird bird) {
    bird.fly();  // Expected to work for all birds
}

makeBirdFly(new Sparrow());  // Works! ✓
makeBirdFly(new Penguin());  // EXCEPTION! ✗
```

---

### ✅ GOOD: Following LSP

```java
// Separate flying ability from being a bird
interface Flyable {
    void fly();
}

class Bird {
    public void eat() {
        System.out.println("Eating...");
    }
}

class Sparrow extends Bird implements Flyable {
    @Override
    public void fly() {
        System.out.println("Sparrow flying!");
    }
}

class Penguin extends Bird {
    // No fly method - penguins don't fly!
    
    public void swim() {
        System.out.println("Penguin swimming!");
    }
}

// Now methods are type-safe
void makeFly(Flyable flyable) {
    flyable.fly();
}

void feedBird(Bird bird) {
    bird.eat();
}

makeFly(new Sparrow());  // Works! ✓
// makeFly(new Penguin());  // Compile error - Penguin isn't Flyable

feedBird(new Sparrow());  // Works! ✓
feedBird(new Penguin());  // Works! ✓
```

---

## Rules for LSP

### Rule 1: Method Signatures

**Subtypes must honor the parent's method signatures.**

```java
// Parent
class Animal {
    public void makeSound() { }
}

// ✅ Good - same signature
class Dog extends Animal {
    @Override
    public void makeSound() {
        System.out.println("Woof!");
    }
}

// ❌ Bad - throws unexpected exception
class Fish extends Animal {
    @Override
    public void makeSound() {
        throw new UnsupportedOperationException("Fish don't make sounds!");
    }
}
```

---

### Rule 2: Preconditions Cannot Be Strengthened

**Subclass's methods should not require MORE than the parent.**

```java
class PaymentProcessor {
    public void process(double amount) {
        // Accepts any positive amount
        if (amount <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        // Process...
    }
}

// ❌ Bad - adds more restrictions (strengthens precondition)
class PremiumPaymentProcessor extends PaymentProcessor {
    @Override
    public void process(double amount) {
        // Now requires amount > 100! This is MORE restrictive!
        if (amount <= 100) {
            throw new IllegalArgumentException("Amount must be > 100");
        }
        super.process(amount);
    }
}

// This worked with PaymentProcessor but fails with PremiumPaymentProcessor
PaymentProcessor processor = new PremiumPaymentProcessor();
processor.process(50);  // Fails! But 50 is valid for parent class!
```

---

### Rule 3: Postconditions Cannot Be Weakened

**Subclass's methods should guarantee at least what the parent guarantees.**

```java
class BankAccount {
    protected double balance;
    
    // Postcondition: balance is always updated correctly
    public void withdraw(double amount) {
        if (amount > balance) {
            throw new InsufficientFundsException();
        }
        balance -= amount;
        // Guaranteed: balance is now reduced by exactly 'amount'
    }
}

// ❌ Bad - weakens the postcondition
class SpecialAccount extends BankAccount {
    @Override
    public void withdraw(double amount) {
        // Sometimes doesn't update balance! (weakens postcondition)
        if (Math.random() > 0.5) {
            balance -= amount;
        }
        // NOT guaranteed: balance might or might not be updated!
    }
}
```

---

### Rule 4: Invariants Must Be Preserved

**Subclass must maintain the same constraints as the parent.**

```java
class PositiveNumber {
    protected int value;
    
    // Invariant: value is ALWAYS positive
    public PositiveNumber(int value) {
        if (value <= 0) {
            throw new IllegalArgumentException("Must be positive");
        }
        this.value = value;
    }
    
    public void setValue(int value) {
        if (value <= 0) {
            throw new IllegalArgumentException("Must be positive");
        }
        this.value = value;
    }
}

// ❌ Bad - breaks the invariant
class AnyNumber extends PositiveNumber {
    public AnyNumber(int value) {
        super(1);  // Trick the parent
        this.value = value;  // Allow any value, including negative!
    }
    
    @Override
    public void setValue(int value) {
        this.value = value;  // No validation!
    }
}

// This breaks expectations
PositiveNumber num = new AnyNumber(-5);
// Now we have a "PositiveNumber" with value -5!
```

---

## More Real-World Examples

### Example 1: File Reading

```java
// ❌ Bad
class FileReader {
    public String readFile(String path) {
        // Reads file and returns content
        return Files.readString(Path.of(path));
    }
}

class EncryptedFileReader extends FileReader {
    @Override
    public String readFile(String path) {
        // Returns encrypted content instead of decrypted!
        // Violates LSP - caller expects readable content
        return getEncryptedBytes(path);
    }
}

// ✅ Good
interface ContentReader {
    String read(String path);
}

class FileReader implements ContentReader {
    public String read(String path) {
        return Files.readString(Path.of(path));
    }
}

class EncryptedFileReader implements ContentReader {
    private String encryptionKey;
    
    public String read(String path) {
        // Returns DECRYPTED content - same behavior expected by interface
        String encrypted = Files.readString(Path.of(path));
        return decrypt(encrypted, encryptionKey);
    }
}
```

### Example 2: Cache System

```java
// ✅ Good - All caches behave the same way
interface Cache<K, V> {
    void put(K key, V value);
    V get(K key);
    void remove(K key);
}

class InMemoryCache<K, V> implements Cache<K, V> {
    private Map<K, V> map = new HashMap<>();
    
    public void put(K key, V value) { map.put(key, value); }
    public V get(K key) { return map.get(key); }
    public void remove(K key) { map.remove(key); }
}

class RedisCache<K, V> implements Cache<K, V> {
    public void put(K key, V value) { /* Redis logic */ }
    public V get(K key) { /* Redis logic */ }
    public void remove(K key) { /* Redis logic */ }
}

class LRUCache<K, V> implements Cache<K, V> {
    public void put(K key, V value) { /* LRU logic */ }
    public V get(K key) { /* LRU logic */ }
    public void remove(K key) { /* LRU logic */ }
}

// Can use any cache interchangeably!
void useCache(Cache<String, User> cache) {
    cache.put("user1", new User("John"));
    User user = cache.get("user1");
}
```

### Example 3: Database Connection

```java
// ✅ Good - All databases behave consistently
interface Database {
    void connect();
    void disconnect();
    List<Map<String, Object>> query(String sql);
    int execute(String sql);
}

class MySQLDatabase implements Database {
    public void connect() { /* MySQL connection */ }
    public void disconnect() { /* MySQL disconnect */ }
    public List<Map<String, Object>> query(String sql) { /* MySQL query */ }
    public int execute(String sql) { /* MySQL execute */ }
}

class PostgresDatabase implements Database {
    public void connect() { /* Postgres connection */ }
    public void disconnect() { /* Postgres disconnect */ }
    public List<Map<String, Object>> query(String sql) { /* Postgres query */ }
    public int execute(String sql) { /* Postgres execute */ }
}

// Works with ANY database
class UserRepository {
    private Database database;
    
    public UserRepository(Database database) {
        this.database = database;
    }
    
    public List<User> findAll() {
        database.connect();
        List<Map<String, Object>> rows = database.query("SELECT * FROM users");
        database.disconnect();
        return mapToUsers(rows);
    }
}
```

---

## How to Identify LSP Violations

### Red Flags 🚩

1. **Throwing unexpected exceptions in overridden methods**
```java
@Override
public void doSomething() {
    throw new UnsupportedOperationException();  // 🚩
}
```

2. **Empty implementations**
```java
@Override
public void fly() {
    // Do nothing - penguin doesn't fly  // 🚩
}
```

3. **Type checking before doing something**
```java
if (bird instanceof Penguin) {
    // Handle specially  // 🚩
} else {
    bird.fly();
}
```

4. **Documentation says "don't call this method"**
```java
/**
 * Don't call this method for Penguin!  // 🚩
 */
public void fly() { }
```

---

## LSP Design Tips

### 1. Use "Is-Substitutable-For" Instead of "Is-A"

Don't think: "Is a Square a Rectangle?" (mathematically yes)
Think: "Can a Square substitute for a Rectangle in my code?" (often no)

### 2. Design for Behavior, Not Just Properties

```java
// Bad thinking: Penguin IS-A Bird
// Good thinking: Penguin BEHAVES-LIKE what Bird interface promises?
```

### 3. Favor Composition Over Inheritance

```java
// Instead of inheriting flight behavior
class Bird {
    private FlightBehavior flightBehavior;
    
    public Bird(FlightBehavior flightBehavior) {
        this.flightBehavior = flightBehavior;
    }
    
    public void performFlight() {
        if (flightBehavior != null) {
            flightBehavior.fly();
        }
    }
}

// Sparrow gets flying behavior
Bird sparrow = new Bird(new FlyWithWings());

// Penguin gets no flying behavior
Bird penguin = new Bird(null);
```

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Principle** | Subclasses must be substitutable for their base class |
| **Key Test** | Replace parent with child - does everything still work? |
| **Common Violations** | Throwing exceptions, empty methods, type checking |
| **Solution** | Better class hierarchy, use interfaces appropriately |

### Remember:
- Child classes must fully honor the contract of the parent
- If you override a method, it should do what the parent promised
- Don't use inheritance just because there's an "IS-A" relationship
- Ask: "Can I substitute X for Y everywhere without breaking anything?"

---

**Next: Interface Segregation Principle (ISP) →**
