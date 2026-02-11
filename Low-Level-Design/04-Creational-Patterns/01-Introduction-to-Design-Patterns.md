# Introduction to Design Patterns

## What are Design Patterns?

**Design Patterns** are proven, reusable solutions to common problems in software design.

They are NOT:
- Finished code you can copy-paste
- Algorithms
- Language-specific features

They ARE:
- Templates for solving problems
- Best practices discovered over years
- Communication tools for developers

---

## History

Design patterns were popularized by the **"Gang of Four" (GoF)** book:
- **"Design Patterns: Elements of Reusable Object-Oriented Software"** (1994)
- Authors: Erich Gamma, Richard Helm, Ralph Johnson, John Vlissides

They documented 23 patterns that are still widely used today.

---

## Why Learn Design Patterns?

### 1. **Don't Reinvent the Wheel**
Smart people already solved these problems. Use their solutions!

### 2. **Common Vocabulary**
"Use the Singleton pattern" is clearer than explaining the whole concept.

### 3. **Better Code Quality**
Patterns promote loose coupling, high cohesion, and flexibility.

### 4. **Interview Success**
LLD interviews heavily test design pattern knowledge.

### 5. **Framework Understanding**
Spring, React, Angular - all use design patterns extensively.

---

## Categories of Design Patterns

The 23 GoF patterns are divided into **3 categories**:

### 1. Creational Patterns
**Purpose**: How objects are CREATED

| Pattern | Description |
|---------|-------------|
| Singleton | Only one instance of a class |
| Factory Method | Create objects without specifying exact class |
| Abstract Factory | Create families of related objects |
| Builder | Construct complex objects step by step |
| Prototype | Clone existing objects |

### 2. Structural Patterns
**Purpose**: How objects are COMPOSED

| Pattern | Description |
|---------|-------------|
| Adapter | Make incompatible interfaces work together |
| Decorator | Add behavior to objects dynamically |
| Facade | Simplified interface to a complex system |
| Composite | Treat individual objects and compositions uniformly |
| Proxy | Placeholder for another object |
| Bridge | Separate abstraction from implementation |
| Flyweight | Share objects to reduce memory |

### 3. Behavioral Patterns
**Purpose**: How objects COMMUNICATE

| Pattern | Description |
|---------|-------------|
| Observer | Notify multiple objects about changes |
| Strategy | Select algorithm at runtime |
| Command | Encapsulate request as an object |
| State | Object behavior changes with state |
| Template Method | Define algorithm skeleton, let subclasses fill in |
| Iterator | Access collection elements sequentially |
| Chain of Responsibility | Pass request along a chain of handlers |
| Mediator | Centralize complex communications |
| Memento | Capture and restore object state |
| Visitor | Add operations to objects without modifying them |

---

## Pattern vs Anti-Pattern

### Pattern (Good)
A proven solution that improves code quality.

### Anti-Pattern (Bad)
A common response to a problem that is ineffective and counterproductive.

**Examples of Anti-Patterns:**
- God Class: One class doing everything
- Spaghetti Code: Unstructured, hard to follow code
- Copy-Paste Programming: Duplicating code instead of reusing
- Golden Hammer: Using one solution for all problems

---

## When to Use Patterns

### DO Use Patterns When:
- You recognize the problem the pattern solves
- The pattern's complexity is justified
- You need the flexibility it provides

### DON'T Use Patterns When:
- The problem is simple enough without them
- You're pattern-hunting (looking for problems to fit patterns)
- It makes code harder to understand

> **"Keep it simple. Don't use a pattern just because you can."**

---

## Pattern Structure

Each pattern is typically described with:

### 1. **Intent**
What problem does it solve?

### 2. **Problem**
When to apply it?

### 3. **Solution**
How does it work?

### 4. **Structure**
UML diagram showing components

### 5. **Participants**
Classes/objects involved

### 6. **Example**
Code implementation

### 7. **Pros and Cons**
Benefits and trade-offs

---

## Simple Example: Before and After Pattern

### Problem: Creating Different Types of Notifications

### ❌ Without Pattern:
```java
class NotificationService {
    public void send(String type, String message) {
        if (type.equals("email")) {
            // 50 lines of email code
        } else if (type.equals("sms")) {
            // 30 lines of SMS code
        } else if (type.equals("push")) {
            // 40 lines of push code
        }
        // Adding new type = modifying this class
    }
}
```

### ✅ With Factory Pattern:
```java
interface Notification {
    void send(String message);
}

class EmailNotification implements Notification {
    public void send(String message) { /* email logic */ }
}

class SMSNotification implements Notification {
    public void send(String message) { /* SMS logic */ }
}

class PushNotification implements Notification {
    public void send(String message) { /* push logic */ }
}

class NotificationFactory {
    public static Notification create(String type) {
        switch (type) {
            case "email": return new EmailNotification();
            case "sms": return new SMSNotification();
            case "push": return new PushNotification();
            default: throw new IllegalArgumentException("Unknown type");
        }
    }
}

class NotificationService {
    public void send(String type, String message) {
        Notification notification = NotificationFactory.create(type);
        notification.send(message);
    }
}
```

---

## Patterns in Real-World Frameworks

| Framework | Pattern | Usage |
|-----------|---------|-------|
| Java Collection | Iterator | for-each loops |
| Spring | Singleton | Beans are singletons by default |
| Spring | Factory | BeanFactory |
| Spring | Proxy | AOP, @Transactional |
| Spring | Template | JdbcTemplate |
| React | Observer | useState, useEffect |
| Node.js | Observer | EventEmitter |
| Java I/O | Decorator | BufferedInputStream wrapping FileInputStream |

---

## Learning Path for Design Patterns

```
Week 1: Creational Patterns
├── Singleton (most common)
├── Factory Method
├── Abstract Factory
├── Builder
└── Prototype

Week 2: Structural Patterns
├── Adapter
├── Decorator
├── Facade
├── Composite
└── Proxy

Week 3: Behavioral Patterns
├── Observer
├── Strategy
├── Command
├── State
└── Template Method

Week 4: Practice
├── Implement patterns from scratch
├── Identify patterns in frameworks
└── Solve LLD problems using patterns
```

---

## How Patterns Relate to SOLID

| SOLID Principle | Related Patterns |
|-----------------|------------------|
| Single Responsibility | Factory, Builder |
| Open/Closed | Strategy, Decorator, Factory |
| Liskov Substitution | Factory, Strategy |
| Interface Segregation | Adapter, Facade |
| Dependency Inversion | Factory, Abstract Factory, Strategy |

---

## Tips for Learning Patterns

1. **Understand the Problem First**
   - What problem does the pattern solve?
   - When would you need it?

2. **Learn the Core Patterns First**
   - Singleton, Factory, Observer, Strategy, Decorator

3. **Implement by Hand**
   - Don't just read - code it yourself!

4. **Find Real Examples**
   - Look at how frameworks use patterns

5. **Don't Force Patterns**
   - Use when they genuinely help

---

## Summary

| Aspect | Description |
|--------|-------------|
| **What** | Reusable solutions to common design problems |
| **Why** | Better code, common vocabulary, interview prep |
| **Types** | Creational, Structural, Behavioral |
| **Key Patterns** | Singleton, Factory, Observer, Strategy, Decorator |
| **Caution** | Don't overuse - keep it simple |

---

**Next: Singleton Design Pattern →**
