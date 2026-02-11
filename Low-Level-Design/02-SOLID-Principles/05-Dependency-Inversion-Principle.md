# Dependency Inversion Principle (DIP)

## The Fifth SOLID Principle

> **"High-level modules should not depend on low-level modules. Both should depend on abstractions."**
>
> **"Abstractions should not depend on details. Details should depend on abstractions."**
> — Robert C. Martin

---

## What Does This Mean?

**Traditional approach**: 
- High-level code (business logic) depends on low-level code (database, email, etc.)

**DIP approach**: 
- Both depend on interfaces/abstractions
- Low-level modules implement the interfaces
- High-level modules use the interfaces

Think of it as **inverting** who depends on whom.

---

## Simple Analogy

### Without DIP (Wrong):
Think of a **lamp plugged directly into the wall wiring**:
- Lamp is connected directly to specific wires
- To change the lamp, you need an electrician
- To change wiring, you might break the lamp

### With DIP (Right):
Think of a **lamp with a plug and socket**:
- Lamp has a standard plug (interface)
- Wall has a standard socket (interface)
- Any lamp works with any socket
- Easy to change either one

The **plug/socket is the abstraction** that both depend on!

---

## Real-World Example: Order Processing

### ❌ BAD: Without DIP (High-level depends on Low-level)

```java
// Low-level module: specific database
class MySQLDatabase {
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }
}

// Low-level module: specific email service
class GmailService {
    public void send(String to, String message) {
        System.out.println("Sending via Gmail to " + to + ": " + message);
    }
}

// High-level module: business logic
class OrderService {
    private MySQLDatabase database;  // Depends directly on MySQL!
    private GmailService emailService;  // Depends directly on Gmail!
    
    public OrderService() {
        this.database = new MySQLDatabase();  // Creates its own dependencies!
        this.emailService = new GmailService();
    }
    
    public void createOrder(Order order) {
        // Business logic
        database.save(order.toString());
        emailService.send(order.getCustomerEmail(), "Order confirmed!");
    }
}
```

### Problems:
1. **Tight coupling**: OrderService is tied to MySQL and Gmail
2. **Hard to test**: Can't test without real database/email
3. **Hard to change**: Switching to PostgreSQL means changing OrderService
4. **OrderService creates its own dependencies**: Can't replace them

---

### ✅ GOOD: With DIP (Both depend on Abstractions)

```java
// Step 1: Define abstractions (interfaces)
interface Database {
    void save(String data);
    String find(String id);
}

interface EmailService {
    void send(String to, String message);
}

// Step 2: Low-level modules implement abstractions
class MySQLDatabase implements Database {
    @Override
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }
    
    @Override
    public String find(String id) {
        return "Data from MySQL";
    }
}

class PostgresDatabase implements Database {
    @Override
    public void save(String data) {
        System.out.println("Saving to PostgreSQL: " + data);
    }
    
    @Override
    public String find(String id) {
        return "Data from PostgreSQL";
    }
}

class GmailService implements EmailService {
    @Override
    public void send(String to, String message) {
        System.out.println("Sending via Gmail to " + to);
    }
}

class SendGridService implements EmailService {
    @Override
    public void send(String to, String message) {
        System.out.println("Sending via SendGrid to " + to);
    }
}

// Step 3: High-level module depends on abstractions
class OrderService {
    private Database database;  // Depends on interface!
    private EmailService emailService;  // Depends on interface!
    
    // Dependencies are INJECTED, not created internally
    public OrderService(Database database, EmailService emailService) {
        this.database = database;
        this.emailService = emailService;
    }
    
    public void createOrder(Order order) {
        // Business logic remains the same
        database.save(order.toString());
        emailService.send(order.getCustomerEmail(), "Order confirmed!");
    }
}
```

### Usage:

```java
// Easy to configure with different implementations
Database mysql = new MySQLDatabase();
EmailService gmail = new GmailService();
OrderService orderService = new OrderService(mysql, gmail);

// Easy to switch to different implementations!
Database postgres = new PostgresDatabase();
EmailService sendgrid = new SendGridService();
OrderService orderService2 = new OrderService(postgres, sendgrid);

// Easy to test with mock implementations!
Database mockDb = new MockDatabase();
EmailService mockEmail = new MockEmailService();
OrderService testService = new OrderService(mockDb, mockEmail);
```

---

## Visualization

### Without DIP:
```
    ┌─────────────────┐
    │  OrderService   │  (High-level)
    │  (Business)     │
    └────────┬────────┘
             │ depends on
             ▼
    ┌─────────────────┐
    │  MySQLDatabase  │  (Low-level)
    │  (Concrete)     │
    └─────────────────┘

Problem: High-level depends on Low-level!
```

### With DIP:
```
    ┌─────────────────┐
    │  OrderService   │  (High-level)
    └────────┬────────┘
             │ depends on
             ▼
    ┌─────────────────┐
    │   <<Database>>  │  (Abstraction - Interface)
    └────────▲────────┘
             │ implements
    ┌────────┴────────┐
    │  MySQLDatabase  │  (Low-level)
    └─────────────────┘

Both depend on abstraction!
The dependency is "inverted" - low-level depends on high-level's interface
```

---

## Another Example: Notification System

### ❌ BAD: Violating DIP

```java
class NotificationManager {
    private SlackApi slackApi;
    private TwilioSMS twilioSMS;
    private FirebasePush firebasePush;
    
    public NotificationManager() {
        this.slackApi = new SlackApi("token");
        this.twilioSMS = new TwilioSMS("accountId", "authToken");
        this.firebasePush = new FirebasePush("serverKey");
    }
    
    public void notify(User user, String message, String channel) {
        switch (channel) {
            case "slack":
                slackApi.postMessage(user.getSlackId(), message);
                break;
            case "sms":
                twilioSMS.sendMessage(user.getPhone(), message);
                break;
            case "push":
                firebasePush.send(user.getDeviceToken(), message);
                break;
        }
    }
}
```

### ✅ GOOD: Following DIP

```java
// Abstraction
interface NotificationChannel {
    void send(User user, String message);
    String getChannelType();
}

// Implementations
class SlackNotification implements NotificationChannel {
    private SlackApi slackApi;
    
    public SlackNotification(SlackApi slackApi) {
        this.slackApi = slackApi;
    }
    
    @Override
    public void send(User user, String message) {
        slackApi.postMessage(user.getSlackId(), message);
    }
    
    @Override
    public String getChannelType() {
        return "slack";
    }
}

class SMSNotification implements NotificationChannel {
    private TwilioSMS twilioSMS;
    
    public SMSNotification(TwilioSMS twilioSMS) {
        this.twilioSMS = twilioSMS;
    }
    
    @Override
    public void send(User user, String message) {
        twilioSMS.sendMessage(user.getPhone(), message);
    }
    
    @Override
    public String getChannelType() {
        return "sms";
    }
}

class PushNotification implements NotificationChannel {
    private FirebasePush firebase;
    
    public PushNotification(FirebasePush firebase) {
        this.firebase = firebase;
    }
    
    @Override
    public void send(User user, String message) {
        firebase.send(user.getDeviceToken(), message);
    }
    
    @Override
    public String getChannelType() {
        return "push";
    }
}

// High-level module depends on abstraction
class NotificationManager {
    private Map<String, NotificationChannel> channels;
    
    public NotificationManager(List<NotificationChannel> channelList) {
        this.channels = new HashMap<>();
        for (NotificationChannel channel : channelList) {
            channels.put(channel.getChannelType(), channel);
        }
    }
    
    public void notify(User user, String message, String channelType) {
        NotificationChannel channel = channels.get(channelType);
        if (channel != null) {
            channel.send(user, message);
        }
    }
    
    public void notifyAll(User user, String message) {
        for (NotificationChannel channel : channels.values()) {
            channel.send(user, message);
        }
    }
}
```

---

## Dependency Injection (DI)

DIP often works hand-in-hand with **Dependency Injection**.

### Types of Dependency Injection:

### 1. Constructor Injection (Recommended)

```java
class OrderService {
    private final Database database;
    private final EmailService emailService;
    
    // Dependencies passed via constructor
    public OrderService(Database database, EmailService emailService) {
        this.database = database;
        this.emailService = emailService;
    }
}

// Usage
OrderService service = new OrderService(
    new MySQLDatabase(),
    new GmailService()
);
```

### 2. Setter Injection

```java
class OrderService {
    private Database database;
    private EmailService emailService;
    
    // Dependencies passed via setters
    public void setDatabase(Database database) {
        this.database = database;
    }
    
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
}

// Usage
OrderService service = new OrderService();
service.setDatabase(new MySQLDatabase());
service.setEmailService(new GmailService());
```

### 3. Interface Injection

```java
interface DatabaseInjectable {
    void injectDatabase(Database database);
}

class OrderService implements DatabaseInjectable {
    private Database database;
    
    @Override
    public void injectDatabase(Database database) {
        this.database = database;
    }
}
```

---

## Real-World Patterns Using DIP

### Repository Pattern

```java
// Abstraction
interface UserRepository {
    User findById(int id);
    void save(User user);
    List<User> findAll();
}

// Implementation
class MySQLUserRepository implements UserRepository {
    public User findById(int id) { /* MySQL query */ }
    public void save(User user) { /* MySQL insert */ }
    public List<User> findAll() { /* MySQL query */ }
}

class MongoUserRepository implements UserRepository {
    public User findById(int id) { /* MongoDB query */ }
    public void save(User user) { /* MongoDB insert */ }
    public List<User> findAll() { /* MongoDB query */ }
}

// Service depends on abstraction
class UserService {
    private UserRepository repository;
    
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
    
    public User getUser(int id) {
        return repository.findById(id);
    }
}
```

### Strategy Pattern

```java
// Abstraction
interface PaymentStrategy {
    void pay(double amount);
}

// Implementations
class CreditCardPayment implements PaymentStrategy { /* ... */ }
class PayPalPayment implements PaymentStrategy { /* ... */ }
class BitcoinPayment implements PaymentStrategy { /* ... */ }

// Context depends on abstraction
class ShoppingCart {
    private PaymentStrategy paymentStrategy;
    
    public ShoppingCart(PaymentStrategy paymentStrategy) {
        this.paymentStrategy = paymentStrategy;
    }
    
    public void checkout(double amount) {
        paymentStrategy.pay(amount);
    }
}
```

---

## Testing with DIP

One of the biggest benefits of DIP is **testability**:

```java
// Production code
class OrderService {
    private Database database;
    private EmailService emailService;
    
    public OrderService(Database database, EmailService emailService) {
        this.database = database;
        this.emailService = emailService;
    }
    
    public boolean createOrder(Order order) {
        database.save(order.toString());
        emailService.send(order.getCustomerEmail(), "Confirmed!");
        return true;
    }
}

// Test with mock implementations
class MockDatabase implements Database {
    public List<String> savedData = new ArrayList<>();
    
    @Override
    public void save(String data) {
        savedData.add(data);  // Just store in memory
    }
    
    @Override
    public String find(String id) {
        return "mock data";
    }
}

class MockEmailService implements EmailService {
    public List<String> sentEmails = new ArrayList<>();
    
    @Override
    public void send(String to, String message) {
        sentEmails.add(to + ": " + message);  // Just record
    }
}

// Unit test
class OrderServiceTest {
    @Test
    public void testCreateOrder() {
        // Arrange
        MockDatabase mockDb = new MockDatabase();
        MockEmailService mockEmail = new MockEmailService();
        OrderService service = new OrderService(mockDb, mockEmail);
        Order order = new Order("customer@email.com", 100.0);
        
        // Act
        boolean result = service.createOrder(order);
        
        // Assert
        assertTrue(result);
        assertEquals(1, mockDb.savedData.size());
        assertEquals(1, mockEmail.sentEmails.size());
        assertTrue(mockEmail.sentEmails.get(0).contains("customer@email.com"));
    }
}
```

---

## DIP in Frameworks

### Spring Framework

```java
// Abstraction
interface MessageService {
    void send(String message);
}

// Implementation
@Service
class EmailMessageService implements MessageService {
    @Override
    public void send(String message) {
        System.out.println("Email: " + message);
    }
}

// High-level module
@Service
class NotificationService {
    private final MessageService messageService;
    
    @Autowired  // Spring injects the dependency
    public NotificationService(MessageService messageService) {
        this.messageService = messageService;
    }
    
    public void notifyUser(String message) {
        messageService.send(message);
    }
}
```

### Angular (TypeScript)

```typescript
// Abstraction
interface LoggerService {
  log(message: string): void;
}

// Implementation
@Injectable()
class ConsoleLoggerService implements LoggerService {
  log(message: string): void {
    console.log(message);
  }
}

// High-level module
@Injectable()
class UserService {
  constructor(private logger: LoggerService) {}
  
  createUser(name: string): void {
    this.logger.log(`Creating user: ${name}`);
  }
}
```

---

## Common DIP Mistakes

### Mistake 1: Interface Near Implementation

```java
// ❌ Bad: Interface in same package as implementation
// com.myapp.mysql/
//   ├── DatabaseInterface.java
//   └── MySQLDatabase.java

// The interface is "owned" by the low-level module
```

```java
// ✅ Good: Interface near high-level module
// com.myapp.core/
//   └── Database.java (interface)
// com.myapp.mysql/
//   └── MySQLDatabase.java (implementation)

// High-level module "owns" the interface
```

### Mistake 2: Depending on Concrete Class Through Variable

```java
// ❌ Bad: Constructor takes interface but stores concrete
class OrderService {
    private MySQLDatabase database;  // Concrete type!
    
    public OrderService(Database database) {
        this.database = (MySQLDatabase) database;  // Cast!
    }
}
```

```java
// ✅ Good: Always use interface type
class OrderService {
    private Database database;  // Interface type
    
    public OrderService(Database database) {
        this.database = database;  // No cast
    }
}
```

---

## Summary

| Aspect | Description |
|--------|-------------|
| **Principle** | Depend on abstractions, not concretions |
| **How** | Use interfaces between high and low-level modules |
| **Benefit** | Loose coupling, easier testing, flexibility |
| **Related** | Dependency Injection, Repository Pattern, Strategy Pattern |

### The Two Rules:
1. **High-level modules** should not depend on **low-level modules**
   - Both should depend on **abstractions**

2. **Abstractions** should not depend on **details**
   - **Details** should depend on **abstractions**

### Key Practices:
- Define interfaces for dependencies
- Pass dependencies through constructor (Dependency Injection)
- High-level modules "own" the interfaces
- Low-level modules implement the interfaces

---

## SOLID Principles Summary

| Principle | One-Line Description |
|-----------|---------------------|
| **S**RP | One class, one responsibility |
| **O**CP | Open for extension, closed for modification |
| **L**SP | Subclasses must be substitutable for base classes |
| **I**SP | Many small interfaces > one fat interface |
| **D**IP | Depend on abstractions, not concretions |

### How They Work Together:
- **SRP** → Small, focused classes
- **OCP** → Extensible without modification
- **LSP** → Proper inheritance hierarchies
- **ISP** → Focused interfaces
- **DIP** → Loosely coupled modules

**Next: UML - Unified Modeling Language →**
